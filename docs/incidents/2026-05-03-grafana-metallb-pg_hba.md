# 2026-05-03: Grafana down, MetalLB "not advertising," one underlying cause

**Date:** 2026-05-03, ~03:00 UTC
**Duration to mitigation:** about 30 minutes from "something is wrong" to both services healthy.
**Severity:** S2. Lab service degraded, no data loss, alerting still functioning.

## Symptom

I noticed Grafana was unreachable. A first look at the cluster showed:

- The Grafana pod in `monitoring` was `2/3 Running, CrashLoopBackOff` with 10 restarts.
- The MetalLB LoadBalancer IPs for Grafana (`192.168.1.229`) and the claude-bridge HITL service (`192.168.1.235`) were not responding to ARP, so even the working endpoints were unreachable from the LAN.

My initial framing was "two issues:" Grafana is broken, and MetalLB is misbehaving on two of the three speakers. That framing turned out to be wrong.

## Investigation

### Grafana container is what is crashing, not the pod

`2/3` was containers ready, not pods. The pod has three containers: `grafana`, `grafana-sc-dashboard` (k8s-sidecar), and `grafana-sc-datasources`. The two sidecars were healthy. The crashing one was the Grafana process itself.

Previous-container logs gave the actual error:

```text
logger=sqlstore level=info msg="Connecting to DB" dbtype=postgres
Error: pq: no pg_hba.conf entry for host "192.168.1.90", user "grafana", database "grafana", no encryption
```

Two facts in that line:

- The host being rejected, `192.168.1.90`, is `k8cluster1`, the **node** where the pod runs, not the pod IP.
- "no encryption" means the connection is plaintext while Postgres expected TLS.

So the Grafana pod is hitting Postgres with the **node IP** as the source. That made sense once I remembered Calico SNATs pod traffic that egresses the cluster.

### MetalLB withdrawals are correlated, not independent

Speaker logs from `k8cluster1` showed:

```text
2026-05-03T03:00:22Z  service=monitoring/kube-prometheus-stack-grafana
                      withdrawing service announcement
                      ip=192.168.1.229 reason=notOwner
2026-05-03T03:00:54Z  service=automation/claude-bridge
                      withdrawing service announcement
                      ip=192.168.1.235 reason=notOwner
```

Same minute Grafana started crashing. Same downstream signature for `claude-bridge`. The `claude-bridge` pod was `0/1 Running` with `/readyz` returning 503, and its `Ready` condition flipped to `False` at exactly `03:00:54Z`. Both apps lost readiness within 32 seconds of each other.

### Why MetalLB stopped advertising

In MetalLB L2, the elected speaker for a service IP must have at least one ready endpoint of that service available. When all endpoints go NotReady, the speaker withdraws (`reason: notOwner`). With only one replica each for Grafana and claude-bridge and both of those replicas NotReady, no speaker on any node could pick the IP back up.

So MetalLB was working correctly. It was withdrawing IPs because there was nothing to send traffic to.

### Why both apps lost readiness at the same time

Grafana (`reloader.stakater.com/auto: "true"`) was clearly restarted by Reloader after a ConfigMap or Secret change. claude-bridge had not restarted, but its `/readyz` started failing at the same moment, which suggested a shared dependency.

The shared dependency was Postgres. The user confirmed: "I think there were updates." Calico had been updated as part of that wave, which changed pod-to-external-LAN egress to be SNATed where it had previously not been (or had been allowed implicitly). The masquerade SNAT made traffic from the pods arrive at Postgres with the **node IP** as source, not the pod IP.

`pg_hba.conf` on the Postgres VM allowed:

```text
host  all  all  127.0.0.1/32      scram-sha-256
host  all  all  10.244.0.0/16     scram-sha-256   # pod CIDR, no longer reachable due to SNAT
host  all  all  192.168.1.88/32   scram-sha-256   # legacy, decommissioned host
host  grafana grafana 192.168.1.121/32 scram-sha-256  # legacy host
```

Nothing for `.89`, `.90`, or `.91`, the actual K8s node IPs that traffic was now arriving from. So Postgres rejected every connection. Grafana could not start, claude-bridge could not pass readiness, and MetalLB correctly withdrew the announcements once endpoints went NotReady.

## Root cause

**One cause, three symptoms.** A networking-layer change (Calico update, masquerading flipped on for the LAN destination) caused pod-to-Postgres traffic to be SNATed to node IPs that `pg_hba.conf` had never been told about. Postgres rejected the connections, the dependent apps lost readiness, MetalLB correctly withdrew their LoadBalancer IPs.

## Fix

Additive change, on the Postgres VM only. Backup first.

```bash
ssh 192.168.1.123 "
  TS=\$(date +%Y%m%d-%H%M%S)
  sudo cp -a /etc/postgresql/16/main/pg_hba.conf \
            /etc/postgresql/16/main/pg_hba.conf.bak-\${TS}
  sudo tee -a /etc/postgresql/16/main/pg_hba.conf > /dev/null <<EOF

# K8s cluster nodes (k8cluster1=.90 k8cluster2=.89 k8cluster3=.91)
host    all  all  192.168.1.89/32   scram-sha-256
host    all  all  192.168.1.90/32   scram-sha-256
host    all  all  192.168.1.91/32   scram-sha-256
EOF
  sudo systemctl reload postgresql@16-main
"
```

`reload` rather than `restart` so existing sessions are not killed. Then bounce the two affected pods to skip the CrashLoopBackOff backoff window:

```bash
kubectl delete pod -n monitoring  -l app.kubernetes.io/name=grafana
kubectl delete pod -n automation  -l app=claude-bridge
```

Wait for both to become Ready, verify the LB IPs answer:

```bash
curl -s -o /dev/null -w "grafana %{http_code}\n"     http://192.168.1.229/api/health
curl -s -o /dev/null -w "claude-bridge %{http_code}\n" http://192.168.1.235:8080/readyz
```

Both `200`. MetalLB re-elected speakers automatically as soon as endpoints became Ready, no MetalLB intervention needed.

## What I did not change, and why

- **MetalLB**. It was behaving correctly. Touching it would have been chasing a symptom.
- **Grafana ssl_mode**. Long-term I want `hostssl + ssl_mode=require` so credentials do not traverse the LAN in plaintext, but flipping that during the incident would have introduced a second variable. Tracked as a follow-up.
- **Calico config**. The new SNAT behavior is fine. Updating `pg_hba.conf` is the simpler change and fits the way other LAN hosts are already authorized.

## Blast radius

Internal only. Falco, Prometheus, Alertmanager, the Discord pipeline, Loki, and the *arr stack were unaffected because they do not depend on the external Postgres.

The incident did **not** trigger a Discord alert, which is its own follow-up: I have no alert on "claude-bridge readiness has been failing for N minutes," and the Grafana CrashLoopBackOff alert fired but I missed it because I was not watching the channel during the rollout window.

## Follow-ups

- [x] Fix `pg_hba.conf` on the live Postgres VM.
- [ ] Persist the `pg_hba.conf` change in `ansible-quasarlab` so an Ansible run does not drop it.
- [ ] Identify which package update at ~03:00 UTC flipped the SNAT behavior. `dpkg.log` and `journalctl --since "03:00 UTC"` on the K8s nodes.
- [ ] Move Grafana and claude-bridge to `ssl_mode=require`, switch the matching `pg_hba.conf` entries to `hostssl`.
- [ ] Add a "long-running readiness failure" Alertmanager rule so a quiet readiness regression actually pages.
- [ ] Add an external-vantage probe (Uptime Kuma off-site) so the next time the entire cluster is silent for a reason, **something** still tells me. See the [Uptime Kuma decision note](../decisions/index.md).

## What this incident is a good example of

- A two-symptom problem that looks like two bugs but is one cause.
- Reading the **withdrawal reason** in MetalLB speaker logs (`notOwner`) instead of assuming MetalLB itself was broken.
- Correlating timestamps across systems (Postgres, MetalLB speakers, Reloader, pod conditions) to find the trigger window.
- Choosing the **smallest, most reversible** fix that closes the ticket, then capturing the rest as follow-ups instead of trying to fix everything at 3am.
