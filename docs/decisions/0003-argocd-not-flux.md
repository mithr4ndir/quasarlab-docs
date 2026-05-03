# ADR 0003: ArgoCD, not Flux

**Status:** Accepted
**Date:** 2025-08

## Context

The cluster needs a GitOps engine to reconcile manifests from `k8s-argocd`. The two real choices are ArgoCD and Flux.

## Decision

ArgoCD, deployed in HA mode in the `argocd` namespace, fronted by `argocd-redis-ha-haproxy` for Redis HA. Apps are organized App-of-Apps with a single `root` Application that points at the Argo apps directory. Sync policy is automated with `prune=true, selfHeal=true` for everything except a small allowlist (cert-manager CRDs, the Postgres operator) where I want a human in the loop.

## Considered

- **Flux v2.** Smaller footprint, friendlier CRD model, no UI by default. The "no UI" part is a feature for operators and a bug for hobbyists who want to show people what is going on.
- **Bare `kubectl apply` in CI.** Works, but loses drift detection and self-heal.
- **Spinnaker, JenkinsX.** Wrong scale entirely.

## Consequences

- **The UI is a teaching aid.** When walking someone through the lab, "open ArgoCD and watch the diff" is faster than `flux get kustomizations`.
- **App-of-Apps imposes a directory structure.** Every new application is one Argo `Application` resource pointing at a path. New folks reading the repo can find any app in two clicks.
- **Sync waves and hooks become important.** Ordering across apps (CRDs before workloads, Postgres operator before databases) lives in `argocd.argoproj.io/sync-wave` annotations. Documented in the repo, not here.
- **ArgoCD itself is pet-shaped.** The HA mode is fine, but the cluster cannot recover if ArgoCD is down. I keep a `kubectl apply -f bootstrap/` directory that can re-create ArgoCD from scratch in case the namespace is lost.
