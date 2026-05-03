# 2026-04-13: PVE self-fence, 9-hour alerting blackout, full remediation

**Date:** 2026-04-13, 08:55 PDT
**Time-to-detect:** ~9 hours (no alerts; noticed by user during normal use)
**Time-to-mitigate:** Same day after detection
**Time-to-remediate (the alerting chain):** 2026-04-18 (5 days of follow-up work, 8 PRs)
**Severity:** S1. Lab service degraded for hours, alert pipeline silently down for the entire window.

This one is the foundation incident for a lot of the structural work the lab has now. It exposed three different single points of failure in the pipeline that is *supposed* to tell me when things are broken, and remediating those took the rest of the week.

## Symptom

Around 17:30 PDT, while working on something else, I noticed the trading dashboard was unreachable. A `kubectl get pod -A` showed multiple pods in `Pending`, all on nodes whose kubelets were not reporting. `pvecm status` from the surviving Proxmox node showed only one peer.

`pve` (one of the two Proxmox hosts) had self-fenced at 08:55:04. Seven of eight `onboot=1` VMs failed to start back up after the reboot. Several K8s nodes were among them. The cluster ran on whatever was left for nine hours, no alert ever reached me.

## Three independent failures that compounded

### Failure 1: Watchdog routed to `null`

The Alertmanager `Watchdog` alert is the deadman: it fires constantly and is meant to be received by an external service that pages **when it stops arriving**. The config had `receiver: null` for it. So the alert fired, was discarded, and there was no external observer to notice the silence. The whole point of a deadman switch was missing.

### Failure 2: Single-replica `discord-alert-proxy` on a downed node

The Discord notification proxy was a 1-replica Deployment. The replica happened to be running on `k8cluster1`, which was on the downed PVE host. With no replica running, every Alertmanager send failed. There was no podAntiAffinity, no PDB, and no priorityClassName, so when the node came back the controller had no urgency to schedule it.

### Failure 3: The proxy installed dependencies at runtime

The proxy deployment ran a base `python:3.12-slim` image and `pip install`-ed `fastapi`, `uvicorn`, `httpx` from an inline ConfigMap entrypoint. So even when the pod *did* eventually start, it depended on PyPI being reachable from inside the cluster at startup. Effectively: a monitoring component whose recovery depended on outbound internet from the lab. PyPI being available was something I never checked because nothing in the alert path advertised it as a dependency.

## Root cause of the original PVE outage

`pve-guests.service` (the Proxmox systemd unit that starts onboot VMs) fires ~13 seconds after the kernel hands off. The `freenas-proxmox` plugin I use to resolve iSCSI extent paths calls the TrueNAS REST API at `https://10.10.12.2/api/v2.0/system/info`. If TrueNAS's API daemon is not yet serving when this call happens, the plugin fails fast with no retry, the systemd unit declares failure for the affected VMs, and they sit in a stopped state.

After a cold reboot of pve, TrueNAS was still booting and not yet serving the API. The plugin failed for every VM that depended on iSCSI extents (the K8s nodes among them). pve-guests gave up. pve was up but the VMs that mattered were not.

## Remediation across the cluster (8 PRs, 2026-04-13 to 2026-04-18)

### Alerting chain

- **Watchdog → Healthchecks.io** (PR #115). The `Watchdog` alert is now routed to a Healthchecks.io check via an ExternalSecret-supplied URL. Healthchecks.io emails (and SMS, in this config) when it stops getting pings. External observer of internal silence, finally.
- **`discord-alert-proxy` made highly available** (PRs #117, #118). Two replicas, soft podAntiAffinity, PodDisruptionBudget with `minAvailable: 1`, `priorityClassName: system-cluster-critical`. The pod that pages me cannot be the pod most likely to be evicted.
- **Image baked, dependencies pinned** (PRs #119, #120, #121, #122). Source moved into the repo at `infrastructure/monitoring/discord-alert-proxy/`. Multi-stage Dockerfile, non-root UID 10001, requirements pinned. Built in CI, pushed to `ghcr.io/mithr4ndir/discord-alert-proxy`, deployment refers to the image by 12-character SHA tag, not `latest`. Dependabot covers pip + docker + github-actions ecosystems. Weekly scheduled rebuild keeps the base image fresh; Trivy gates the build on HIGH/CRITICAL CVEs.
- **PveGuestDown alert** improved (PR #117). Tag-based silencing via an anchored `(.*;)?(no-alert|maintenance)(;.*)?` regex on the VM tag string, plus a `PveGuestMaintenanceTagStale` safety alert at 24 hours so a tag I forgot to remove eventually pages.

### Proxmox-side

- **`wait-truenas-api.sh` ExecStartPre** wired into pve-guests.service via drop-in (`roles/pve/freenas_iscsi`). Polls the TrueNAS REST endpoint every 2 seconds for up to 300 seconds. Accepts 2xx, 401, and 403 (the API daemon being up but auth-rejecting is "ready"). Rejects 5xx (daemon up but malfunctioning). `TimeoutStartSec=infinity` preserves the upstream pve-guests timeout. Ansible deploys the script and the drop-in.
- **QDevice NSS database** rebuilt by hand: corosync-qdevice on pve had had a corrupted `nssdb` since the 04-13 reboot, so the cluster had been running at 2/3 votes for a week and I hadn't noticed. Copied the working nssdb from pve2.
- **Both PVE nodes upgraded** 8.4.x to 8.4.18.
- **SSH host-key policy** in `ansible.cfg` set to `accept-new` so brand-new hosts can be reached without manual `ssh-keyscan`, but a host-key change on a known host still refuses (TOFU, not blanket trust).
- **`update-metrics.timer` runs every 10 minutes** so `node_reboot_required` gauge stays fresh; was stale after reboots before this change.

### Validation

Real reboots of both pve and pve2 on 2026-04-14. The TrueNAS-readiness wait logged "reachable (HTTP 401) after 9 attempts" in both cases. Every onboot VM started cleanly. ~10 min recovery on pve, ~18 min on pve2. The whole pipeline was tested, including the Healthchecks.io deadman: I let the cluster idle for 11 minutes (longer than the HC.io alert threshold) to confirm it would page if Watchdog stopped, then re-armed.

## What this incident is really about

It is easy to treat this as "PVE had a bad reboot." The real lesson is structural: **don't host the alerter on the thing it monitors.** Each of the three failures was a different instance of the same bug.

- The Watchdog routed to `null` meant the alerter was hosted on its own honesty: I trusted the config to be right, with no external observer.
- The single-replica proxy meant the alerter was hosted on the cluster it monitors. Cluster down, alerter down.
- The runtime `pip install` meant the alerter was hosted on PyPI's availability at the worst possible time.

After the remediation, every leg of the alert path has an external observer or external pin:

- Watchdog has an external observer (Healthchecks.io).
- `discord-alert-proxy` has external pinning (image baked at GHCR, deployed by SHA).
- `discord-alert-proxy` has internal redundancy (HA, PDB, priority).

There is still one node left to chase: the alert pipeline still runs *inside* the K8s cluster, so a full cluster-down event will still go silent. That is the active question behind ["public read-only Grafana"](../decisions/index.md) and the off-site Uptime Kuma decision.

## Follow-ups

- [x] Alert chain hardened (8 PRs above).
- [x] Boot-race fixed (`wait-truenas-api.sh`).
- [x] QDevice NSS rebuilt; PVE upgrades applied.
- [x] Reboot-required gauge timer refresh.
- [ ] Off-site external probe (Uptime Kuma off-site) so a full cluster-down still pages. Tracked under [decisions/index.md](../decisions/index.md).
- [ ] CI step that walks every `image:` reference and HEAD-requests the registry, so a future "supply chain rot" failure does not need to be discovered by accident again. Same idea, different scope.

## What this incident is a good example of

- A symptom-vs-cause split. "Trading dashboard is down" looked like an app failure; the real failure was the alerting chain that should have told me 9 hours earlier.
- Three independent SPOFs in the same path. Fixing only the loudest one would have left the other two waiting to re-fire.
- Choosing **structural** fixes (HA, deadman, baked image) over **tweak** fixes (bump retry count, increase replica memory). The fix list looks heavier upfront but doesn't require the same diagnosis a year later.
