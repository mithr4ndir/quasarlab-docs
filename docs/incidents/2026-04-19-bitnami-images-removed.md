# 2026-04-19: Bitnami public images quietly disappeared

**Date:** 2026-04-19
**Severity:** S3. Single Job stuck in `ImagePullBackOff` for ~67 minutes (284 retries). No user-visible impact, caught during an unrelated investigation.

## Symptom

While poking at something else in `apps/media/`, I noticed a PostSync hook (`jellyfin-endpoints-hook.yaml`) had been failing image pulls for over an hour with:

```text
rpc error: code = NotFound desc = failed to pull and unpack image
"docker.io/bitnami/kubectl:1.31": failed to resolve image:
docker.io/bitnami/kubectl:1.31: not found
```

Tag, registry, exact spelling all looked correct.

## What happened

Bitnami quietly retired their public Docker Hub images sometime in 2025. Every classic `bitnami/<image>:<tag>` path now 404s on resolution. There is a continuation namespace at `bitnamilegacy/<image>:<tag>` that still resolves today, but Bitnami has flagged it as EOL-bound, so it is a kicking-the-can fix rather than a real one.

The hook itself was harmless: a `Job` that templated some Service Endpoints after Argo synced the Jellyfin app. With the image gone, the Job sat in retry, the post-sync wave never completed, but the running Jellyfin pod kept serving so nothing was visible to users. This is the worst kind of quiet failure: it would have made a future ArgoCD sync look stuck or misleadingly succeed.

## Blast-radius audit

After fixing the immediate hook, I grepped the relevant repos for every Bitnami reference. The same retirement affects:

- Kubernetes Jobs, CronJobs, and initContainers across `apps/*` and `infrastructure/*`.
- Ansible roles and `docker-compose` files in `playbooks/*` and `roles/*` that pull `bitnami/*`.
- Helm chart `values.yaml` that defaults `image.repository` to `bitnami/*` (this is the sneaky one; many community charts default to it and you only know if you read the chart).
- `Dockerfile` `FROM bitnami/*` base images.

```bash
# k8s-argocd
grep -rn "image: bitnami" --include="*.yaml" .
grep -rn "bitnami/" --include="*.yaml" --include="Chart.*" .

# ansible-quasarlab
grep -rn "bitnami/" --include="*.yml" --include="*.yaml" --include="Dockerfile*" .
```

Only the one Job was affected in this case. Audit is now part of [Runbook: Bitnami image 404](../runbooks/bitnami-image-404.md).

## Replacement policy

| Need | Pick |
|------|------|
| `kubectl` with a shell (heredocs in Jobs) | `alpine/kubectl:<cluster-version>` |
| `kubectl` distroless (exec only, no shell) | `cgr.dev/chainguard/kubectl:latest` |
| Postgres, Redis, MongoDB, RabbitMQ, etc. | Docker Hub official, Chainguard, or a vendor-maintained image |
| Helm chart that defaults to `bitnami/*` | Override `image.repository` in values.yaml |

**Pin to a specific patch version**, never `latest`, never just the minor. Trivy + Dependabot will then flag tag drift as a finding instead of letting it slide silently.

## Fix

PR #143: `bitnami/kubectl:1.31` → `alpine/kubectl:1.33.4` in the Jellyfin endpoints hook. Matches the cluster minor exactly, has `/bin/sh` for the heredoc the Job uses, and is maintained by an active project.

## Why this caught me

This was a **silent rot** failure: the resource never changed, the registry path was unchanged, but the *world* changed underneath. The Trivy + Dependabot setup I built for `discord-alert-proxy` (during the [2026-04-13 alerting cascade](2026-04-13-alerting-blackout-cascade.md) remediation) catches this class of problem automatically for images I **build**. Standalone `image:` references in third-party manifests do not get scanned unless I wire them in explicitly.

## Follow-ups

- [x] Replace the affected Bitnami pull in the Jellyfin hook (PR #143).
- [x] Runbook for the next time this happens with a different image.
- [ ] CI step that walks every `image:` reference across `k8s-argocd` and HEAD-requests its registry. Cheap, deterministic, fails loud on 404. Tracked.
- [ ] Periodic audit of base images in `Dockerfile` files in my own repos (`discord-alert-proxy`, `claude-bridge`, `sky-explorer`). Dependabot catches CVEs; it does not catch "vendor pulled the rug."

## What this incident is a good example of

- **"Free public base image from a big vendor" is not a permanent dependency.** Treat them like transitive packages and audit them.
- **Silent failures are worse than loud ones.** A broken Job in retry is invisible until you happen to look. Add alerts on `kube_job_failed` for production-relevant Jobs, not just user-facing workloads.
- **Pin patch, not minor.** A tag drift via `:latest` resolution can land a CVE or, as here, a complete vendor change. Patch-pinning makes the scanner do its job.
