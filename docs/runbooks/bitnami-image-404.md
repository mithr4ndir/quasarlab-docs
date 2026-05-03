# Runbook: Bitnami image 404 / `ImagePullBackOff`

## Symptom

A pod, Job, or initContainer is stuck in `ImagePullBackOff`. The reason in `kubectl describe` reads:

```text
rpc error: code = NotFound desc = failed to pull and unpack image
"docker.io/bitnami/<image>:<tag>": failed to resolve image
```

Or the workflow log says `manifest unknown` for any `bitnami/*:<tag>` reference.

## Why

Bitnami quietly retired their public Docker Hub images in 2025. Every classic `bitnami/<image>:<tag>` path now 404s. There is a continuation namespace at `bitnamilegacy/<image>:<tag>` that still resolves today, but Bitnami has flagged that as EOL-bound, so it is a kicking-the-can fix.

This typically catches you on:

- Helm charts whose `image.repository` defaults to `bitnami/*`.
- One-shot Kubernetes Jobs and PostSync hooks that use `bitnami/kubectl` to template ConfigMaps or run cluster-bootstrap tasks.
- Older Ansible roles or `docker-compose` files that pin a Bitnami image as a base.
- `Dockerfile` `FROM bitnami/*` lines.

In the lab, the first sighting was a PostSync hook in `apps/media/jellyfin-endpoints-hook.yaml` that sat in `ImagePullBackOff` for ~67 minutes (284 retries) before being noticed during an unrelated investigation.

## Triage

Find every Bitnami reference across the relevant repos:

```bash
# k8s-argocd
grep -rn "image: bitnami" --include="*.yaml" .
grep -rn "bitnami/"       --include="*.yaml" --include="Chart.*" .

# ansible-quasarlab
grep -rn "bitnami/" --include="*.yml" --include="*.yaml" --include="Dockerfile*" .
```

Helm chart `values.yaml` is the sneaky one. Many community charts default `image.repository: bitnami/<thing>` and you only know if you read the chart.

## Fix

Pick the replacement based on what you needed Bitnami for. **Pin to a specific patch version**, never `latest`, never just the minor.

| Need | Replacement |
|------|-------------|
| `kubectl` with a shell (heredocs in Jobs, wrapper scripts) | `alpine/kubectl:<cluster-version>` (matches your cluster minor) |
| `kubectl` distroless (exec only, no shell) | `cgr.dev/chainguard/kubectl:latest` |
| Postgres, Redis, MongoDB, RabbitMQ, etc | The official Docker Hub image (`postgres:<tag>`, `redis:<tag>`), or a Chainguard image (`cgr.dev/chainguard/postgres`), or a vendor-maintained image |
| Helm chart that defaults to `bitnami/*` | Override `image.repository` in your values.yaml |

For the lab's specific incident, the fix was `bitnami/kubectl:1.31` to `alpine/kubectl:1.33.4` (matches the cluster minor exactly, has `/bin/sh` for the heredoc the Job uses).

## Prevention

A periodic CI step that walks every `image:` reference in the repo and `HEAD`-requests its registry catches this entire class of rot before production does. Cheap to run, deterministic, fails loud. Tracked as a follow-up.

In the meantime, the Trivy scan + Dependabot wiring on the few container images I build (e.g. `discord-alert-proxy`) catches CVEs and tag drift on those specifically. Standalone `image:` references in YAML do not get scanned unless they are explicitly enrolled.

## Patterns worth knowing

- **"Free public base image from a big vendor" is not a permanent dependency.** Treat them like transitive packages and audit them.
- **Pin patch, not minor or `latest`.** If you pin minor, a tag-drift CVE can land via `:latest` resolution. If you pin patch, your scanner flags the move and you decide.
- **`ImagePullBackOff` + 404 looks like network at first glance.** Always check the registry directly (`docker pull` or `crane manifest`) before chasing CNI / DNS.
