# ADR 0004: Bake images for the monitoring chain, never `pip install` at runtime

**Status:** Accepted
**Date:** 2026-04-15 (post-2026-04-13 incident)

## Context

The original `discord-alert-proxy` shipped as a `python:3.12-slim` base image plus a `pip install fastapi uvicorn httpx` step in the entrypoint, with the proxy code itself loaded from a ConfigMap. It worked, but it had three problems that all surfaced during the [2026-04-13 alerting blackout](../incidents/2026-04-13-alerting-blackout-cascade.md):

1. Pod startup depended on PyPI being reachable from the cluster at the moment the pod started. PyPI being down at the wrong moment would have silenced the alert chain.
2. Restart-time was ~30 seconds for the pip install plus container startup. Combined with single-replica + no PDB, this was load-bearing latency.
3. There was no supply-chain pin. The proxy resolved its dependencies at runtime against whatever PyPI was serving.

For the lab's primary alert path, none of those is acceptable.

## Decision

Every component in the monitoring / alerting chain is shipped as a baked container image that:

- Has all dependencies installed at build time.
- Pins every transitive dependency in `requirements.txt` (or the language-equivalent lockfile).
- Is built in CI (GitHub Actions), pushed to GHCR, and referenced by **12-character SHA tag**, not `latest`.
- Has Trivy scanning gating the build on HIGH/CRITICAL CVEs.
- Has Dependabot watching the `pip`, `docker`, and `github-actions` ecosystems with weekly checks.
- Has a weekly scheduled rebuild via cron in the workflow, so CVEs in the base image (`python:3.12-slim` Debian layer) get picked up even when nothing in the source has changed.

This applies to: `discord-alert-proxy`, `claude-bridge` (HITL Discord bridge), and any future monitoring or alerting component.

## Considered

- **Helm chart with init container that runs pip install.** Same runtime-PyPI dependency, just behind one more abstraction layer.
- **Use a third-party Alertmanager-to-Discord receiver image.** No clear leader as of 2026, and I want some custom translation logic anyway. Vendor risk replaces the runtime risk.
- **Run the proxy outside Kubernetes on the alerter VM.** Cleanly avoids the cluster-dependency problem but adds a second deployment surface to maintain. Not currently worth the cost; the HA + PDB + priority-class story inside the cluster is good enough.

## Consequences

- **Image size matters less than I thought.** A pinned `python:3.12-slim` build of the proxy is around 80 MB. Fine.
- **Tag-by-SHA means the manifest changes on every build.** That is the point: the image change is reviewable in Git, and Reloader is not needed because the SHA bump is the manifest bump.
- **Weekly scheduled rebuild is non-optional.** Otherwise the base-image CVE story is on the day you happen to push a code change. The cron in the workflow makes this independent of source changes.
- **Trivy + Dependabot only cover the images I *build*.** They do not cover standalone `image:` references in third-party manifests (e.g. Helm chart defaults). That gap is what bit me in the [2026-04-19 Bitnami removal](../incidents/2026-04-19-bitnami-images-removed.md). A separate "registry-walk in CI" check is tracked as follow-up work.
- **GHCR public packages, no `imagePullSecret` plumbing.** Internal images stay public for now. The moment any of them needs to be private, the deployment grows an `ExternalSecret` for a GHCR PAT, which is a known and documented step but worth flagging.

## Related

- [2026-04-13 alerting blackout](../incidents/2026-04-13-alerting-blackout-cascade.md) is the original justification.
- [2026-04-19 Bitnami images removed](../incidents/2026-04-19-bitnami-images-removed.md) shows the limit of this ADR: it covers images I own, not images I consume.
