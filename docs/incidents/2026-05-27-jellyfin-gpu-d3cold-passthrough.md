# 2026-05-27: jellyfin GPU D3cold passthrough hang and K620 fallback crash-loop

**Date:** 2026-05-27
**Severity:** S3. Single-service user-visible failure (jellyfin transcoding broken), no data loss, no other services affected. Watch chain stayed online throughout.

## Symptom

Reported as "transcoding fails." Jellyfin itself answered HTTP 200 on both the direct address and the public proxy in under 30 ms, and the web UI loaded fine. So the failure was specifically in the playback path: stream starts, dies in the first minute, retry, same thing.

## What I found, in order

### 1. The 2080 Ti was gone from pve2

pve2's `lspci -s 0a:` returned nothing. The RTX 2080 Ti, which is normally `[10de:1e07]` on `0000:0a:00.0` (plus 3 sibling functions for HDMI audio, USB-C controller, UCSI), had dropped off the PCI bus completely. Vfio-pci was still loaded but bound to zero devices.

dmesg captured the exact moment the card died, earlier in the day at 21:25:53:

```
vfio-pci 0000:0a:00.0: no suspend buffer for LTR; ASPM issues possible after resume
vfio-pci 0000:0a:00.0: Unable to change power state from D3cold to D0, device inaccessible
vfio-pci 0000:0a:00.0: timed out waiting for pending transaction; performing function level reset anyway
```

The VM had been stopped or had its guest reset, the GPU dropped to D3cold, and the host could not bring it back up. The kernel attempted Function Level Reset, that timed out, and after that the device was unreachable.

I tried the documented soft-recovery escalations from sysfs:

```
echo 1 > /sys/bus/pci/devices/0000:00:03.1/reset_subordinate   # hot reset of parent bridge
echo 1 > /sys/bus/pci/rescan                                   # bus rescan
```

Neither brought it back. The parent bridge's `reset_method` file said `pm` only, no `bus`, no `acs`, no `slot`. So the only available reset mechanism is PCI power-management D-state cycling, and that is exactly the path the card just failed on. There is nothing more to try in software. The card needs the chassis power cycled.

### 2. HA had already failed over to pve, onto the K620

This was actually correct behavior. The failover hookscript (`/var/lib/vz/snippets/gpu-failover.sh`) detected the start was happening on `pve` instead of `pve2`, rewrote VM 115's hostpci entries from the 2080 Ti's BDFs (`0000:0a:00.*`) to the K620's (`0000:03:00.*`), and the VM came up with the K620 attached.

The K620 is a Quadro Maxwell card. It has H.264 NVENC. That is the entire list. It cannot decode HEVC in hardware, it has no AV1 anything, no HDR tone-mapping support. So any source that needed HEVC NVDEC or 10-bit was going to either fall back to CPU decode or fail.

That was a degraded but partly-working state. But playback was failing even on H.264 sources, which means the K620 itself was not the immediate cause.

### 3. The in-VM watchdog was killing every transcode

`gpu-watchdog.service` inside the jellyfin VM was crash-looping. The journal showed 99 cumulative failures within a few hours, in a recognizable cycle:

```
00:48:54 Exhausted 2 soft recovery attempts, waiting for host PCI reset
00:50:54 GPU failure detected (attempt 1/2), total failures: 96
00:53:54 Soft recovery failed
00:54:54 GPU failure detected (attempt 2/2), total failures: 97
00:57:54 Soft recovery failed
01:07:51 GPU failure detected (attempt 1/2), total failures: 99
01:07:54 Stopped gpu-watchdog.service
```

The watchdog's recovery path is: stop jellyfin, `killall -9 ffmpeg`, `rmmod` the nvidia stack, `modprobe nvidia` again, retry `nvidia-smi`. That logic was written for the 2080 Ti's failure modes. On the K620 fallback it false-positived (most plausibly because `nvidia-smi` was timing out under sustained NVENC load on a 2 GB Maxwell card with a 570-series driver), and every false-positive killed the in-flight transcode.

So the user-visible "transcoding fails" was the watchdog: ffmpeg got SIGTERM (`FFmpeg exited with code 143`, where 143 is 128 + SIGTERM 15) about 60 seconds into every play, by the watchdog, regardless of whether the stream itself was healthy.

## What I did

In order, with the destructive steps gated on explicit confirmation:

### Stop the SIGTERM loop first, restore service to the K620

The watchdog unit file (`/etc/systemd/system/gpu-watchdog.service`) was Ansible-managed, so `systemctl mask` refused. Worked around by renaming the unit file out of systemd's search path, then `daemon-reload`. After that, h264 transcodes succeeded on the K620, which was enough to confirm the diagnosis.

### Then plan the actual recovery

Cold-rebooting pve2 has blast radius: it hosts command-center1 (this admin host), k8cluster2, wazuh, and authentik. I will not just kick it without telling the user, even though I knew the GPU would not come back any other way.

Confirmed go-ahead, then prepared for one reboot rather than two: added `pcie_aspm=off` to `/etc/default/grub` on pve2 (backed up the original), ran `update-grub`, and `systemctl enable gpu-vm-monitor.timer` so the host-side watchdog would actually be running after boot (it had been deployed by Ansible but ended up `inactive` somehow).

`pcie_aspm=off` is the standard mitigation for the D3cold class of failure on consumer NVIDIA cards under vfio. Not a guarantee, but the canonical first try, low risk relative to the alternative escalations.

Then `systemctl reboot` and waited for pve2 to come back.

### After the reboot

pve2 came up in 3 minutes. `lspci` showed all four 2080 Ti functions enumerated cleanly. vfio-pci was bound to all four. `pcie_aspm=off` was active in `/proc/cmdline`. `gpu-vm-monitor.timer` was active and enabled.

VM 115 was still where HA left it (on pve, K620 path) because the HA group is configured with `nofailback`, which is deliberate (don't yo-yo on transient failures).

### Migrate VM 115 back to pve2

`qm migrate` refuses to migrate any VM that has `hostpci` entries. The documented workaround is to strip them from the config file directly (`qm set` deadlocks inside hookscript context), migrate, and let the failover hookscript re-add the right BDFs for the destination node on start.

```bash
qm shutdown 115
sed -i '/^hostpci/d' /etc/pve/qemu-server/115.conf
qm migrate 115 pve2
```

That all worked. Hookscript log on pve2 showed the four 0a:00.* entries being re-added at pre-start. QEMU started. HA reported the VM running.

### Then nothing inside the VM worked

The guest's `lspci` showed only the emulated Bochs VGA (`[1234:1111]`). No NVIDIA functions at all. The host had vfio-pci bound to the 2080 Ti, the hookscript had added the hostpci entries, `qm showcmd 115` returned the correct `-device vfio-pci,host=0000:0a:00.0,...` strings. But the actually-running QEMU process did not have any of those flags.

```bash
# qm config 115 includes the hostpci lines (good)
# qm showcmd 115 includes the -device vfio-pci flags (good)
# but:
tr "\0" "\n" < /proc/$(cat /run/qemu-server/115.pid)/cmdline | grep -c vfio-pci
# 0
```

Plain `qm stop 115 && qm start 115` (not via HA) launched QEMU correctly with the vfio devices, and the guest came up with all four 2080 Ti functions visible, drivers loading, `nvidia-smi` reporting the card.

That suggests the HA-driven start path used a cached cmdline from before the hookscript edited the config, but I did not chase it further during the incident. Captured as a follow-up.

### Verify, then restore the watchdog

```bash
# inside VM 115
mv /etc/systemd/system/gpu-watchdog.service.disabled-2026-05-27 \
   /etc/systemd/system/gpu-watchdog.service
systemctl daemon-reload
systemctl enable --now gpu-watchdog
```

Service came up `active`, journal showed `Detected expected GPU: NVIDIA GeForce RTX 2080 Ti` (after I shipped the watchdog change described in the IaC section below).

Transcode test as final sign-off:

```bash
ffmpeg -hwaccel cuda -hwaccel_output_format cuda -c:v hevc_cuvid \
  -i <some_hevc_file.mkv> -t 15 -c:v h264_nvenc -preset p1 -b:v 5M -f null -
# exit 0
```

This exact command had failed on the K620 with `Hardware is lacking required capabilities`. On the 2080 Ti it ran clean.

## IaC follow-up the same night (PR ansible-quasarlab#133)

Three things had been touched manually on the live host during the recovery: GRUB cmdline, the host watchdog timer, and the in-VM watchdog unit file. Per the lab's prime directive of "everything in code", all of these had to go back into Ansible, or the next playbook run would either revert them or be inconsistent with what was on disk.

Two changes landed:

### `roles/jellyfin`: make the watchdog GPU-aware

New default `gpu_watchdog_expected_gpu_substring`, default `"GeForce RTX 2080 Ti"`. At startup the script probes `nvidia-smi` (with retries, because the driver can take a moment to come up), and if the detected card does not contain that substring it enters PASSIVE mode: still probes and emits Prometheus metrics, but never restarts services or reloads modules.

The destructive recovery path is preserved exactly when the expected card IS present. Only the fallback path is gated.

### `roles/pve/gpu_passthrough`: harden the GRUB cmdline

The default `pve_grub_cmdline` was `"quiet {iommu_type}_iommu=on iommu=pt"`, which is missing two things that were live on pve2 by hand:

- `vfio-pci.ids=...`: belt-and-suspenders binding alongside `/etc/modprobe.d/vfio.conf`. Was on pve2's cmdline by hand. The next Ansible run would have stripped it.
- `pcie_aspm=off`: the D3cold mitigation, added during this incident.

Split into `pve_grub_cmdline_base` plus `pve_grub_cmdline_passthrough_extra`, with the extra applied only when `pve_gpu_passthrough` is true. Non-passthrough nodes are unaffected.

Verified `changed=0` on a real apply against pve2 (the rendered cmdline matches what was already there). The IaC also produces the correct cmdline on a fresh node.

The other piece, `gpu-vm-monitor.timer`, turned out to already be `systemd.state=started, enabled=true` in `roles/pve/hookscripts/tasks/main.yml`. The reason it was inactive on pve2 was almost certainly that the playbook had not been re-run since whatever event disabled it. Re-running the playbook brought it to the desired state. No code change needed.

## What was the smoking gun

Two stacked guns, one per failure layer:

For "why the 2080 Ti keeps dying": the dmesg line `vfio-pci 0000:0a:00.0: Unable to change power state from D3cold to D0, device inaccessible`, plus `/sys/bus/pci/devices/0000:00:03.1/reset_method` being `pm` only. Together those say: the card cannot wake from D3cold, and the kernel has no working slot or bus reset available, so soft recovery is impossible. Cold boot only.

For "why transcoding was actively failing today, on the K620 path": the gpu-watchdog journal showing `total failures: 99` cycling through soft-recovery attempts, paired with the user-visible `FFmpeg exited with code 143` (SIGTERM = 128 + 15). The watchdog was the agent of failure, not the GPU.

Bonus surprise: the HA-driven start launching QEMU without any `-device vfio-pci` flags despite the config and `qm showcmd` both saying it should. `tr "\0" "\n" < /proc/<pid>/cmdline | grep -c vfio-pci = 0` is a clean one-line test for that condition. A plain `qm stop && qm start` worked around it.

## Patterns worth keeping

- **The pcie reset taxonomy is real.** Before chasing soft recoveries, read `/sys/bus/pci/devices/<bridge>/reset_method`. If it only has `pm`, the kernel cannot do anything beyond power-cycling D-states, and that is exactly the path that just failed. Cold boot the host.
- **A watchdog tuned for one card model is hostile on a different model.** The K620 fallback path was not even broken in itself, it was being killed every 60s by recovery logic written for a different card. Gate destructive recovery on actual hardware identity.
- **`qm migrate` refuses hostpci VMs; the failover hookscript expects to add hostpci at start time.** Strip the lines from the config file directly with `sed`, do not use `qm set` (it deadlocks against the hookscript's own config lock).
- **HA on a hostpci VM has a cmdline-cache quirk.** If a guest comes up after HA-driven migration and reports no GPU, `tr "\0" "\n" < /proc/<qemu_pid>/cmdline | grep -c vfio-pci`. If it returns 0, plain `qm stop && qm start` fixes it.
- **ICMP is not a liveness check for the jellyfin VM.** Ping fails (UFW), but TCP and HTTP work fine. `curl http://192.168.1.170:8096/health` is the right probe.
- **`nvidia-smi` must always be wrapped with `timeout`.** A hung GPU hangs nvidia-smi indefinitely otherwise, and anything that calls it becomes a wedge in the recovery path.

## Open follow-ups

- The HA-cached-cmdline quirk is captured but not investigated. A guard would be a post-start hookscript that diffs `qm showcmd <vmid>` against `/proc/<pid>/cmdline` and screams if hostpci is missing.
- The K620 passive-mode change was deployed and the active-path is verified live, but the passive path itself was code-reviewed rather than exercised. Live test deferred because it requires another forced HA failover.
- If `pcie_aspm=off` alone does not hold the card under sustained load, the next escalation is `vfio-pci disable_idle_d3=1` as a module option. The structural fix is a workstation-class card with proper FLR.
- The K8s-flavored `runbooks/jellyfin-transcoding-ffmpeg.md` page is stale. Jellyfin moved to a VM in March. That page should be re-aimed at the native-on-VM install or split into a separate runbook.
