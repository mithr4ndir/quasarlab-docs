# Quasarlab Docs

Architecture, runbooks, incidents, and command references for my homelab.

## Why these docs exist

Two reasons. The first is selfish: when something breaks at 3am, I want to read what past me already figured out instead of rediscovering it. The second is to demonstrate, in public, how I think about systems: how I lay them out, where the failure modes are, what the tradeoffs were, and how I respond when things go sideways.

## How to read it

If you only have a few minutes:

1. Skim [Architecture / Overview](architecture/overview.md) for the lay of the land.
2. Read one [incident report](incidents/index.md). They each tell a story from symptom to root cause to fix.

If you have longer:

- The [runbooks](runbooks/index.md) are the response side of those incidents, distilled.
- The [decisions](decisions/index.md) explain why the lab is the shape it is.
- The [cheatsheet](cheatsheet/index.md) is grep-bait for me at 3am.

## What runs in the lab

- Three-node Kubernetes cluster on Proxmox, Calico CNI, MetalLB L2 for LoadBalancer IPs.
- ArgoCD as the GitOps engine for everything in cluster.
- kube-prometheus-stack for metrics, Loki + Vector for logs, Alertmanager into Discord via a small custom proxy.
- External Postgres VM for stateful workloads that I want to keep portable.
- Ansible for VM-level configuration, Terraform for provider-managed resources.
- Mostly homegrown apps plus the usual *arr stack, Jellyfin, ArgoCD, and a HITL Discord bridge for agentic workflows.

The full picture lives under [Architecture](architecture/overview.md).
