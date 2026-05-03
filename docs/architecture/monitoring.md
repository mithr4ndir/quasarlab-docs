# Monitoring

The stack is conventional: Prometheus pulls metrics, Loki/Vector handle logs, Alertmanager fans out alerts, Grafana glues the views together. The only custom piece is the Discord proxy.

## Components

| Component | Where | Notes |
|-----------|-------|-------|
| `kube-prometheus-stack` | `monitoring` ns | Prometheus, Alertmanager (HA, 2 replicas), Grafana, kube-state-metrics, prometheus-operator |
| Loki | `monitoring/loki-0` | Single-binary mode, sufficient for one user |
| Vector | DaemonSet + aggregator StatefulSet | Agents on every node, ship to Loki |
| Discord proxy | `monitoring/discord-alert-proxy` | Tiny HTTP service, takes Alertmanager and Falco webhooks, posts to channels |
| Falco | `security/falco` | Runtime threat detection on the cluster, alerts via Discord |

External LB IPs let me reach things from outside the cluster without port-forwarding:

| Service | LB IP |
|---------|-------|
| Grafana | `192.168.1.229` |
| Prometheus | `192.168.1.230` |
| Loki | `192.168.1.231` |
| Vector aggregator | `192.168.1.232` |
| Alertmanager | `192.168.1.233` |

## Alert routing

Alerts go through Alertmanager into the Discord proxy. The proxy splits them by route:

- Infrastructure (NodeDown, KubePodCrashLooping, MemoryPressure) into `#ops`.
- Media stack failures into `#media`.
- Falco runtime events into `#security`.
- Trading and science workloads into their own channels.

The proxy also filters routine churn. Restarts and scrape blips that resolve in under 60 seconds get suppressed so the Discord channels stay scannable.

## What this stack covers and what it does not

What it does well:

- Pull-based metrics from anything that exposes `/metrics`, including the apps I write.
- Cluster health (node, pod, kubelet, scheduler, etcd, apiserver).
- Logs centralized off the nodes.
- Alerts that I will see, because Discord is where I already am.

What it does not cover, and where Uptime Kuma was supposed to fill in (see [the incident commentary on Kuma](../decisions/index.md)):

- **External vantage.** Everything alerting lives inside the cluster. If the cluster, the home internet, or NPM goes down, the pipeline is silent.
- **Heartbeats.** "Did this cron job actually run." Backups, scheduled labctl runs, GitHub Actions.
- **Public status page.** Discord channels are not a status page.
- **Third-party uptime.** GHCR, the upstream DNS provider, etc.

## Observability lessons captured here

- Alertmanager grouping matters. A flapping condition on three nodes should be one Discord message, not three. `group_by`, `group_wait`, `group_interval`.
- Prometheus is great at "things that have metrics," not "things that should have run." Heartbeats need a different tool.
- "Pod is Ready" is a load-bearing signal. The MetalLB / pg_hba / Grafana cascade in [this incident](../incidents/2026-05-03-grafana-metallb-pg_hba.md) was kicked off by a single readiness probe failing, which then had blast radius all the way to the LoadBalancer IP not answering ARP.
