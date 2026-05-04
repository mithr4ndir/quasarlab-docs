# Architecture overview

A small homelab that runs like a real one, with GitOps, observability, and a controlled blast radius.

## Layout

```text
Internet
  Cloudflare (DNS, proxy for select names)
    │
    ▼
LAN 192.168.1.0/24
  NPM (TLS termination, reverse proxy)
    │
    ▼
  MetalLB pool 192.168.1.225-240
    │
    ▼
  Kubernetes cluster (kubeadm, Calico, ArgoCD)
    apiserver VIP   192.168.1.20:6443
    k8cluster1      192.168.1.90  ── pve1
    k8cluster2      192.168.1.89  ── pve1
    k8cluster3      192.168.1.91  ── pve2
                       │  pod->external = SNAT to node IP
                       ▼
  Postgres 16 VM      192.168.1.123  ── pve2
  Uptime Kuma VM      192.168.1.129  ── pve2 (idle, see decisions)
  Proxmox pve1        192.168.1.10
  Proxmox pve2        192.168.1.11
```

A higher-fidelity diagram and screenshots live in [Visuals](visuals.md).

## Hosts and roles

| Host | IP | Role |
|------|----|------|
| pve1 | 192.168.1.10 | Proxmox node, hosts most VMs |
| pve2 | 192.168.1.11 | Proxmox node, hosts the rest |
| K8s API | 192.168.1.20 | kube-apiserver VIP |
| k8cluster1 | 192.168.1.90 | K8s worker / control plane |
| k8cluster2 | 192.168.1.89 | K8s worker / control plane |
| k8cluster3 | 192.168.1.91 | K8s worker / control plane |
| postgresql | 192.168.1.123 | Postgres 16, shared backend for Grafana, claude-bridge, and other stateful apps |
| uptime-kuma | 192.168.1.129 | Black-box monitoring VM (currently idle, see [decisions](../decisions/index.md)) |
| NPM | LAN | nginx-proxy-manager, TLS termination and routing for `*.herro.me` |

## Kubernetes cluster

- 3 nodes, kubeadm-built, single control-plane endpoint at `192.168.1.20:6443`.
- Calico CNI, pod CIDR `10.244.0.0/16`.
- MetalLB in **L2 mode** announcing from the pool `192.168.1.225-240` (see [ADR 0001](../decisions/0001-metallb-l2.md)).
- ArgoCD reconciles the cluster from Git. App-of-apps pattern, source repo `k8s-argocd`.
- Stakater Reloader watches ConfigMaps and Secrets and rolls Deployments when they change.
- External Secrets Operator pulls credentials from outside the cluster so secrets never live in Git.

## What runs where

| Namespace | What | Notes |
|-----------|------|-------|
| `argocd` | ArgoCD HA + Redis | HA mode, redis-ha-haproxy |
| `monitoring` | kube-prometheus-stack, Loki, Vector, Alertmanager, Grafana | Discord notification proxy lives here |
| `metallb-system` | MetalLB controller and speakers | L2Advertisement covers the whole pool |
| `external-secrets` | ESO controllers | Backed by a secret store outside the cluster |
| `reloader` | Stakater Reloader | `reloader.stakater.com/auto: "true"` triggers rollouts |
| `media` | *arr stack, Jellyfin, qBittorrent, Jellyseerr, NZBGet | `192.168.1.226` shared LB IP |
| `automation` | claude-bridge HITL Discord bridge | LB at `192.168.1.235` |
| `trading` | Trading dashboard + API | NQ-bias-engine |
| `science` | Dask cluster, lsdb workloads | LSDB (large scale astronomy databases) |
| `dashboard` | Homepage | Lab landing page |
| `security` | Falco runtime detection | Discord webhook for alerts |

The full layout is in [k8s-argocd](https://github.com/mithr4ndir/k8s-argocd).

## Why this shape

A few decisions worth calling out, with longer rationale in [Decisions](../decisions/index.md):

- **External Postgres VM** instead of in-cluster CloudNativePG. Stateful data is the most expensive thing to lose, and I wanted backups, restore drills, and major-version upgrades to be boring even if K8s broke. ([ADR 0002](../decisions/0002-external-postgres.md))
- **MetalLB L2 instead of BGP**, because the home network is a single L2 segment and there is no router I trust to peer with. ([ADR 0001](../decisions/0001-metallb-l2.md))
- **ArgoCD instead of Flux**, mostly for the UI when explaining the lab to other humans, and the App-of-Apps pattern fits how I think. ([ADR 0003](../decisions/0003-argocd-not-flux.md))
- **Discord as the alert sink** because it is where I already live. A small custom proxy translates Alertmanager and Falco webhooks into channel-appropriate messages.
