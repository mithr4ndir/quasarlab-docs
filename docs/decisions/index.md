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
- [0007 Uptime Kuma as the out-of-cluster monitor](0007-uptime-kuma-external-monitor.md)

## Active questions (not yet ADR'd)

!!! note "Answered 2026-09-19, and not the way this list predicted"
    **Uptime Kuma on a VM, in-cluster, or off-site** is now
    [ADR 0007](0007-uptime-kuma-external-monitor.md): an **on-LAN VM**
    (vm117), not off-site. This list said "the right move is probably to run
    it off-site for external vantage", and the ADR argues the opposite for
    this lab. Off-site cannot reach LAN-only endpoints without opening a
    tunnel through the perimeter, the failure modes it uniquely catches are
    already covered by the Healthchecks.io layer, and a monitor that cannot
    be repaired without the thing it monitors is a poor monitor of last
    resort.

    It is worth recording **how this entry was resolved**. It was not decided;
    it was overtaken. The VM sat provisioned and never started while this
    line described it as a pending choice, and on 2026-09-19 an
    [NFSv4 deadlock](../incidents/2026-09-19-nfsv4-callback-deadlock.md) left
    the lab with no alerting for eleven hours. A known monitoring gap parked
    as an "active question" is a decision to accept the risk. It just does
    not feel like one while it sits in a list.

Still open:

- **Did the Healthchecks.io deadman actually fire on 2026-09-19?**
  [ADR 0005](0005-healthchecks-deadman.md) should have produced an email
  within about ten minutes of 07:05, and nothing reached the owner for
  eleven hours. Whether it fired and was missed, or never fired, is
  unresolved and is the most valuable open question on this list. A deadman
  switch that does not wake you is worse than none, because the design
  leans on it while it contributes nothing.
- **Public read-only Grafana dashboard.** The operational UIs (ArgoCD, homepage, Prometheus, Alertmanager) are intentionally LAN-only. A single curated read-only Grafana dashboard exposed through NPM, with the rest of Grafana login-gated, would be the smallest blast-radius way to give an external viewer a real-time look at the lab. Open: which dashboard, what to redact, whether to put it behind Cloudflare Access for additional gating.
- **Calico vs Cilium.** Today the lab runs Calico. Cilium would buy me eBPF-based observability and policy, at the cost of one more thing to debug. Defer.
- **`hostssl + ssl_mode=require` for all Postgres clients.** Already a follow-up from the 2026-05-03 incident.
