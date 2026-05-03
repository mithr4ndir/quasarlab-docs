# ADR 0001: MetalLB in L2 mode, not BGP

**Status:** Accepted
**Date:** 2025-08

## Context

The cluster needs LoadBalancer Services. The home network is a single L2 segment behind one consumer router that does not speak BGP, and I do not want to introduce a peering router just to satisfy a homelab.

## Decision

Run MetalLB in **L2 mode**, with a single `IPAddressPool` covering `192.168.1.225-240` and a matching `L2Advertisement` selecting the whole pool. All three K8s nodes act as candidate speakers, MetalLB picks one per service IP via leader election, and that speaker answers ARP for the IP.

## Considered

- **BGP mode.** Cleaner failover, no ARP gratuitous-broadcast quirks, but requires a BGP-capable router or a software router (FRR/MikroTik) added as another moving part. Wrong tradeoff for one user.
- **Cloud-provider-style controllers (kube-vip, PureLB).** Less mature for L2 on the same scale, and L2 with kube-vip is just MetalLB-shaped without the polish.
- **NodePort + manual reverse-proxy.** Works, but loses one of the two reasons I run a cluster (declarative LB IPs).

## Consequences

- **One node owns each IP at any time.** Failover is ARP-driven and takes a few seconds. Acceptable for a lab.
- **One ARP-reply per IP.** If a switch ever does not learn the MAC quickly, one client can have stale neighbor cache. Has not happened, but worth knowing.
- **Endpoint-based withdrawal is load-bearing.** When a service has no Ready endpoints, MetalLB withdraws the announcement, which means the LB IP stops answering ARP. This is the right behavior, but if you do not know it, it looks like MetalLB is broken (see [the 2026-05-03 incident](../incidents/2026-05-03-grafana-metallb-pg_hba.md)).
- **No cross-subnet failover.** Every node has to be on the same L2 segment as the LB pool. True today, future me may want to revisit if I add a second site.
