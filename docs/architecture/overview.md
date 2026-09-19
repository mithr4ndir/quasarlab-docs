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
    k8cluster2      192.168.1.89  ── pve2
    k8cluster3      192.168.1.91  ── pve2
                       │  pod->external = SNAT to node IP
                       ▼
  Postgres 16 VM      192.168.1.123  ── pve
  Uptime Kuma VM      192.168.1.129  ── pve  (out-of-cluster monitor)
  TrueNAS             192.168.1.15   (iSCSI: VM disks, NFS: K8s PVCs + media)
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
| k8cluster2 | 192.168.1.89 | K8s worker / control plane (on pve2, with k8cluster3) |
| k8cluster3 | 192.168.1.91 | K8s worker / control plane |
| postgresql | 192.168.1.123 | Postgres 16, shared backend for Grafana, claude-bridge, and other stateful apps |
| uptime-kuma | 192.168.1.129 | Out-of-cluster monitor, live since 2026-09-19 (see [ADR 0007](../decisions/0007-uptime-kuma-external-monitor.md)). On **pve**, deliberately not on pve2 with the etcd majority |
| truenas | 192.168.1.15 | TrueNAS. VM disks over iSCSI, Kubernetes PVCs and media over NFS. See [Storage](#storage) |
| NPM | LAN | nginx-proxy-manager, TLS termination and routing for `*.herro.me` |

## Storage

Undocumented here until 2026-09-19, which is part of why the
[NFSv4 deadlock](../incidents/2026-09-19-nfsv4-callback-deadlock.md) was hard
to reason about while it was happening.

Everything stateful in the lab lands on one appliance, **TrueNAS at
192.168.1.15**, over **two independent services**. That separation is the
single most important fact about lab storage, because it is what decides how
bad any given NAS problem is.

| Service | Serves | Consumers |
|---|---|---|
| **iSCSI** (ZFS-over-iSCSI) | VM disks | Every VM in the lab, via Proxmox storage `truenas-iscsi` |
| **NFS** | Kubernetes PVCs (`/mnt/tank/k8s`), media (`/mnt/tank/media`) | `k8s-nfs` StorageClass, Jellyfin and the *arr stack |

**They fail independently.** On 2026-09-19 NFSv4 wedged completely for eleven
hours while iSCSI kept serving, so every VM stayed up and only Kubernetes
workloads with NFS PVCs were affected. That is the only reason the recovery
was possible: the VMs that did the recovery were themselves running off the
broken appliance, over the service that still worked.

### The path

- Storage traffic does **not** use the LAN. There is a direct **10G**
  NIC-to-NAS link on `10.10.12.0/24` (`enp11s0` on pve, `enp3s0` on pve2),
  which is also the Proxmox migration network.
- iSCSI portal `10.10.12.2`, target
  `iqn.2005-10.org.freenas.ctl:proxmox-cluster`, backed by
  `tank/proxmox_zfs_vms` with a 16k block size.
- The Kubernetes StorageClass `k8s-nfs` currently mounts with
  **`nfsvers=4.1`**. Moving it to NFSv3 is the pending fix from the incident
  (k8s-argocd#213): v3 is stateless, with no delegations, callbacks or
  sessions, so that entire deadlock class disappears.

### The pool

`tank` is four mirrored vdevs on SSDs, with a separate log (SLOG) and cache
(L2ARC) device, roughly 14.7 TB raw.

!!! warning "Not all the drives are trustworthy"
    Three Inland SSDs from one bad batch repeatedly drop off the SATA bus
    under sustained writes, and `mirror-3` pairs two of them. This is why
    bulk operations against the pool are rate limited (see the
    [NAS reboot runbook](../runbooks/nas-reboot-evacuate-vm-disks.md)) and
    why `dmesg` on the NAS is part of triage for anything storage-shaped.

### Node-local storage, and one trap

Each Proxmox node also has local LVM-thin storage, which is what VMs are
evacuated onto when the NAS needs a reboot:

| Node | Storages |
|---|---|
| pve | `local`, `SSD1`, `SSD2`, `isos` |
| pve2 | `nvme_1tb`, `ssd_1`, `ssd_2` |

!!! danger "SSD1 and ssd_1 also hold the etcd data disks"
    They are QLC. Putting bulk sequential writes there drove etcd WAL fsync
    from 74 ms to **7,627 ms** and took the Kubernetes control plane down for
    about sixteen minutes on 2026-09-19.

    Nothing in Proxmox marks these as latency-critical. It shows free space.
    The constraint exists only as operator knowledge until
    terraform-quasarlab#24 codifies the etcd disks.

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
