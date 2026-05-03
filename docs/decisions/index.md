# Decisions

Short notes on tradeoffs taken in the lab. Each ADR follows the standard shape:

- **Context** — what was true at the time.
- **Decision** — what I picked.
- **Considered** — what else I looked at and why I did not pick it.
- **Consequences** — what this commits me to, including the parts that are not great.

## Index

- [0001 MetalLB L2 not BGP](0001-metallb-l2.md)
- [0002 External Postgres VM](0002-external-postgres.md)
- [0003 ArgoCD not Flux](0003-argocd-not-flux.md)
- [0004 Bake images for the monitoring chain](0004-bake-monitoring-images.md)
- [0005 External deadman switch via Healthchecks.io](0005-healthchecks-deadman.md)
- [0006 Database-per-app on the shared Postgres VM](0006-database-per-app.md)

## Active questions (not yet ADR'd)

- **Uptime Kuma on a VM, in-cluster, or off-site.** A VM was provisioned but never started ([incident reference](../incidents/2026-05-03-grafana-metallb-pg_hba.md)). The right move is probably to run it off-site for external vantage. To be ADR'd before the next round of monitoring changes.
- **Public read-only Grafana dashboard.** The operational UIs (ArgoCD, homepage, Prometheus, Alertmanager) are intentionally LAN-only. A single curated read-only Grafana dashboard exposed through NPM, with the rest of Grafana login-gated, would be the smallest blast-radius way to give an external viewer a real-time look at the lab. Open: which dashboard, what to redact, whether to put it behind Cloudflare Access for additional gating.
- **Calico vs Cilium.** Today the lab runs Calico. Cilium would buy me eBPF-based observability and policy, at the cost of one more thing to debug. Defer.
- **`hostssl + ssl_mode=require` for all Postgres clients.** Already a follow-up from the 2026-05-03 incident.
