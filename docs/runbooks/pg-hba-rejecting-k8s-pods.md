# Runbook: Postgres rejecting K8s pods

## Symptom

App pods fail to connect to the external Postgres VM (`192.168.1.123`). Pod logs show:

```text
pq: no pg_hba.conf entry for host "192.168.1.X", user "<app>", database "<app>", no encryption
```

The `192.168.1.X` in the error is a **K8s node IP**, not a pod IP. Multiple apps that share the same Postgres backend lose readiness at roughly the same time.

## Triage

Confirm it is the auth layer, not Postgres being down:

```bash
# Is Postgres reachable?
nc -vz 192.168.1.123 5432

# Same TCP probe with no extra tools:
timeout 3 bash -c 'cat < /dev/tcp/192.168.1.123/5432' ; echo "exit=$?"

# Postgres logs from the VM:
ssh 192.168.1.123 'sudo journalctl -u postgresql@16-main --since "30 min ago" | tail -50'
```

Look for `no pg_hba.conf entry` lines. The source IP in those lines is the IP the connection is **arriving from** at Postgres, after Calico SNAT.

## Fix

Allow the K8s node IPs explicitly. Backup before editing.

```bash
ssh 192.168.1.123 "
  TS=\$(date +%Y%m%d-%H%M%S)
  sudo cp -a /etc/postgresql/16/main/pg_hba.conf \
            /etc/postgresql/16/main/pg_hba.conf.bak-\${TS}

  sudo tee -a /etc/postgresql/16/main/pg_hba.conf > /dev/null <<EOF

# K8s cluster nodes
host    all  all  192.168.1.89/32   scram-sha-256
host    all  all  192.168.1.90/32   scram-sha-256
host    all  all  192.168.1.91/32   scram-sha-256
EOF

  sudo systemctl reload postgresql@16-main
"
```

Bounce affected pods so they retry instead of waiting for the CrashLoopBackOff window:

```bash
kubectl delete pod -n monitoring -l app.kubernetes.io/name=grafana
kubectl delete pod -n automation -l app=claude-bridge
```

Verify:

```bash
kubectl wait pod -n monitoring -l app.kubernetes.io/name=grafana --for=condition=Ready --timeout=120s
kubectl wait pod -n automation -l app=claude-bridge          --for=condition=Ready --timeout=120s
```

## Follow-up

- **Persist in Ansible** (`ansible-quasarlab/labctl-runs/postgres/`) so a config run does not revert it.
- **Move toward `hostssl` + `ssl_mode=require`** in app configs so the LAN hop is encrypted. Then change the rules above from `host` to `hostssl`.
- **Track what introduced the SNAT.** If the old behavior had pod IPs reaching Postgres directly, a Calico upgrade or `natOutgoing` change is the most likely cause. Check `dpkg.log` and Calico release notes for the upgrade window.

## Why this happens at all

Calico masquerades pod traffic that egresses the cluster, so external hosts see the **node IP** as source. Allowing only the pod CIDR (`10.244.0.0/16`) in `pg_hba.conf` will not work, because by the time the packet reaches Postgres the source has already been rewritten.
