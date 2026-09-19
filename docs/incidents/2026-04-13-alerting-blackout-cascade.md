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
- [x] External probe so a full cluster-down still pages. Shipped 2026-09-19 as [Uptime Kuma on vm117](../decisions/0007-uptime-kuma-external-monitor.md), five months later and only after the gap named here cost another eleven hours. It is **on the LAN and on this Proxmox cluster**, not off-site, which is a deliberate reversal of what this line asked for. See the postscript.
- [ ] Coverage for the failure modes an on-LAN probe cannot see (house internet, power, the whole Proxmox cluster). Healthchecks.io (ADR 0005) is the only leg outside the house, and 2026-09-19 left its effectiveness unestablished.
- [ ] CI step that walks every `image:` reference and HEAD-requests the registry, so a future "supply chain rot" failure does not need to be discovered by accident again. Same idea, different scope.

## What this incident is a good example of

- A symptom-vs-cause split. "Trading dashboard is down" looked like an app failure; the real failure was the alerting chain that should have told me 9 hours earlier.
- Three independent SPOFs in the same path. Fixing only the loudest one would have left the other two waiting to re-fire.
- Choosing **structural** fixes (HA, deadman, baked image) over **tweak** fixes (bump retry count, increase replica memory). The fix list looks heavier upfront but doesn't require the same diagnosis a year later.

## Postscript, 2026-09-19: the one node left to chase came due

This page ends by naming the gap it did not close: "the alert pipeline still runs *inside* the K8s cluster, so a full cluster-down event will still go silent." On 2026-09-19 that is exactly what happened. An NFSv4 callback deadlock on the NAS wedged Prometheus and left Loki unable to ingest, and the lab ran blind for about eleven hours with nothing alerting. The full write-up is [2026-09-19 NFSv4 callback deadlock](2026-09-19-nfsv4-callback-deadlock.md).

Two corrections to what I wrote above, in order of how wrong they were.

**"After the remediation, every leg of the alert path has an external observer or external pin" was too strong.** The three legs I listed were real fixes, but all three protect the *delivery* of an alert. None of them observes whether Prometheus is still evaluating rules. Healthchecks.io was supposed to be the backstop for exactly that, and on 2026-09-19 it did not produce a response for eleven hours. Whether it fired and was missed, or never fired at all, is still not established; that is the open question at the top of the 2026-09-19 page and it should not be treated as settled.

**"Off-site Uptime Kuma" was the wrong shape, and I only found that out by doing it.** What shipped is an on-LAN VM, vm117, running Uptime Kuma outside the Kubernetes cluster, with eight monitors covering Prometheus, Alertmanager, Grafana, Loki, the three kube-apiservers, and an NFS read probe. It posts to Discord `#alerts` directly rather than through `discord-alert-proxy`, because the proxy runs in the cluster being watched. Its disk is deliberately not on NFS. The reasoning for on-LAN over off-site is in [ADR 0007](../decisions/0007-uptime-kuma-external-monitor.md): an off-site probe cannot reach the LAN-only endpoints without opening a tunnel through the perimeter, and a monitor you cannot repair without the thing it monitors is a poor monitor of last resort.

So the gap is narrower now, not closed. Kuma covers cluster-down. It does not cover house-internet-down, power-down, or the loss of the Proxmox cluster it shares with everything it watches. ADR 0007 also records the part nothing enforces: vm117's hypervisor placement is currently the right way round by luck rather than by constraint.

The lesson this page taught in April holds, and it needed restating five months later in a second language: **don't host the alerter on the thing it monitors.** In April that meant the Discord proxy. In September it meant Prometheus itself.
