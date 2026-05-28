# Runbook: GPU passthrough D3cold recovery (jellyfin / pve2 / RTX 2080 Ti)

When the RTX 2080 Ti on pve2 hangs at the PCI level and HA fails VM 115 (jellyfin) over to pve onto the K620 fallback. This is the path through the [2026-05-27 incident](../incidents/2026-05-27-jellyfin-gpu-d3cold-passthrough.md), expressed as a checklist.

## Symptoms that match this runbook

- Jellyfin web UI works (HTTP 200) but playback dies in the first minute, repeatedly.
- `qm config 115` on pve shows `hostpci0: 0000:03:00.0,pcie=1,x-vga=0` (K620 BDFs) instead of `0000:0a:00.0` (2080 Ti BDFs). VM is running on `pve`, not `pve2`.
- `ha-manager status` shows `service vm:115 (pve, started)` and the pve2 LRM is `idle`.
- `lspci -s 0a:` on pve2 returns nothing.

If those all hold, this is the same failure. Proceed.

## Diagnosis (1 minute)

```bash
# 1) Confirm the card is gone from the bus
ssh root@192.168.1.11 'lspci -nn | grep -i nvidia'
# Expected on a healthy host: 4 lines for 0a:00.0 through 0a:00.3
# Failure mode: empty output

# 2) Confirm what the kernel said
ssh root@192.168.1.11 'dmesg -T | grep -iE "0000:0a|nvidia|vfio" | tail -30'
# Look for: "Unable to change power state from D3cold to D0, device inaccessible"

# 3) Check what reset methods are actually available
ssh root@192.168.1.11 'cat /sys/bus/pci/devices/0000:00:03.1/reset_method'
# If output is "pm" only (no "slot" or "bus" or "acs"), only a cold reboot will recover the card.
```

## Hot-side mitigation (no reboot, restores service on K620)

While planning the cold reboot, stop the watchdog inside VM 115 so transcodes stop getting SIGTERMed on the K620 fallback path. This was fixed structurally in ansible-quasarlab#133 (the watchdog is now K620-aware and enters PASSIVE mode automatically), so on any host that has run the playbook since 2026-05-27 this step is **not needed**. Check first:

```bash
ssh root@192.168.1.11 'qm guest exec 115 -- /bin/bash -c "
  grep -E \"EXPECTED_GPU|PASSIVE\" /usr/local/bin/gpu-watchdog.sh
"'
# If you see EXPECTED_GPU= and PASSIVE= lines, the K620-aware watchdog is deployed.
# Service will be active but in PASSIVE mode; no manual intervention needed.
# If you do NOT see those lines, the host has the old watchdog. Disable it temporarily:

ssh root@192.168.1.11 'qm guest exec 115 -- /bin/bash -c "
  systemctl stop gpu-watchdog
  mv /etc/systemd/system/gpu-watchdog.service \
     /etc/systemd/system/gpu-watchdog.service.disabled-\$(date +%F)
  systemctl daemon-reload
"'
# Reverse: rename the file back + daemon-reload + enable --now.
```

## Cold reboot of pve2

Blast radius: command-center1, k8cluster2, wazuh, authentik. Pick a window. Notify if anyone else uses these.

### Prep before reboot (so one reboot does both jobs)

```bash
ssh root@192.168.1.11 '
# Backup grub first
cp /etc/default/grub /etc/default/grub.bak.$(date +%F)

# Confirm pcie_aspm=off and vfio-pci.ids= are in pve_grub_cmdline.
# If the host has been Ansibled since 2026-05-27 they should already be present.
grep pcie_aspm /etc/default/grub || echo MISSING_PCIE_ASPM
grep vfio-pci.ids /etc/default/grub || echo MISSING_VFIO_IDS

# If MISSING_PCIE_ASPM appears, add it (idempotent):
sed -i "/^GRUB_CMDLINE_LINUX_DEFAULT=/ { /pcie_aspm=off/! s/\"$/ pcie_aspm=off\"/ }" /etc/default/grub
update-grub

# Make sure the host-side watchdog will be running after boot
systemctl enable gpu-vm-monitor.timer
'
```

### Reboot

```bash
ssh root@192.168.1.11 'systemctl reboot'
```

Wait roughly 3 to 5 minutes. SSH back in and verify:

```bash
ssh root@192.168.1.11 '
echo "uptime:"; uptime
echo "cmdline:"; cat /proc/cmdline | grep -oE "pcie_aspm=off|vfio-pci.ids=[^ ]+"
echo "gpu enumerated:"; lspci -nn | grep -i nvidia | wc -l   # expect 4
echo "vfio bindings:"; ls /sys/bus/pci/drivers/vfio-pci/ | grep "^0000:0a" | wc -l   # expect 4
echo "gpu-vm-monitor.timer:"; systemctl is-active gpu-vm-monitor.timer
echo "ha:"; ha-manager status
'
```

If `lspci` shows zero nvidia devices after a clean boot, escalate (the card may have hardware-failed, or there is a BIOS-level problem).

## Migrate VM 115 back to pve2

The HA group is `nofailback`, so the VM stays on pve until you move it manually.

```bash
ssh root@192.168.1.10 '
# 1. Gracefully stop the VM (preserves jellyfin SQLite integrity if a stream is mid-write)
qm shutdown 115

# 2. Wait for stopped
until qm status 115 | grep -q stopped; do sleep 3; done

# 3. Strip hostpci lines directly from the config file.
#    DO NOT use qm set: it deadlocks in hookscript context.
cp /etc/pve/qemu-server/115.conf /etc/pve/qemu-server/115.conf.bak.$(date +%F)
sed -i "/^hostpci[0-9]*:/d" /etc/pve/qemu-server/115.conf

# 4. Migrate. HA wraps qm migrate.
qm migrate 115 pve2

# If HA leaves the service in request_stop after migrate, kick it:
ha-manager set vm:115 --state started
'
```

The failover hookscript on pve2 will re-add the four hostpci entries with the 2080 Ti BDFs (`0000:0a:00.0` through `0000:0a:00.3`) on start.

## Handle the HA cmdline cache quirk

After the first HA-driven start on pve2, the QEMU process may have launched WITHOUT the vfio devices in its actual cmdline, even though the config and `qm showcmd 115` look right. Always check, always fix it the same way:

```bash
ssh root@192.168.1.11 '
QEMU_PID=$(cat /run/qemu-server/115.pid)
VFIO_COUNT=$(tr "\0" "\n" < /proc/$QEMU_PID/cmdline | grep -c vfio-pci)
echo "vfio devices in running QEMU cmdline: $VFIO_COUNT (expect 4)"

if [ "$VFIO_COUNT" -eq 0 ]; then
  echo "Cmdline cache quirk hit. Bouncing the VM."
  qm stop 115
  qm start 115
fi
'
```

After this, `qm guest exec 115 -- nvidia-smi` should return the 2080 Ti.

## Verify

```bash
ssh root@192.168.1.11 'qm guest exec 115 -- /bin/bash -c "
  echo == nvidia ==
  nvidia-smi --query-gpu=name,driver_version --format=csv

  echo == watchdog ==
  systemctl is-active gpu-watchdog
  journalctl -u gpu-watchdog --since \"1 min ago\" --no-pager | grep -E \"Detected|PASSIVE\" | tail -3

  echo == hevc nvdec + h264 nvenc transcode (would FAIL on K620) ==
  SAMPLE=\$(find /mnt/media/library -type f -name \"*.mkv\" -size -500M 2>/dev/null | head -1)
  timeout 30 /usr/lib/jellyfin-ffmpeg/ffmpeg -hide_banner -loglevel error \
    -hwaccel cuda -hwaccel_output_format cuda -c:v hevc_cuvid \
    -i \"\$SAMPLE\" -t 10 -c:v h264_nvenc -preset p1 -b:v 5M -f null - && echo HEVC_OK
"'

curl -sS -o /dev/null -w "HTTP %{http_code} %{time_total}s\n" http://192.168.1.170:8096/health
curl -sSk -o /dev/null -w "Proxy HTTP %{http_code} %{time_total}s\n" https://jelly.herro.me/health
```

Pass criteria:

- `nvidia-smi` returns "NVIDIA GeForce RTX 2080 Ti"
- Watchdog journal shows `Detected expected GPU: NVIDIA GeForce RTX 2080 Ti` (active mode, not PASSIVE)
- HEVC test prints `HEVC_OK`
- Both curls return HTTP 200

If anything fails, do not restore the watchdog rename (if you did the hot-side mitigation step). Capture the state and stop.

## Cleanup

```bash
# Backup files from this recovery
ssh root@192.168.1.11 'rm /etc/default/grub.bak.* 2>/dev/null'
ssh root@192.168.1.10 'rm /etc/pve/qemu-server/115.conf.bak.* 2>/dev/null'
```

## Why all this is needed (one-paragraph summary)

The 2080 Ti is a consumer Turing card. Consumer NVIDIA cards under vfio passthrough have an unreliable D3cold to D0 wake path. When the VM stops or migrates, the card drops to D3cold; on the AMD Matisse chipset on pve2, the parent PCIe bridge exposes only `pm` as a reset method, which is the same mechanism that just failed. So once the card hangs there is no software path back, only a host power cycle. `pcie_aspm=off` reduces the rate at which the card enters problematic low-power states. The host-side `gpu-vm-monitor.timer` and the in-VM `gpu-watchdog.service` together attempt automatic recovery short of a reboot. The structural fix is a workstation-class NVIDIA card (A-series or higher) with proper FLR support. Until that hardware swap, this runbook is the recovery procedure.

## Related

- [2026-05-27 incident retro](../incidents/2026-05-27-jellyfin-gpu-d3cold-passthrough.md)
- [Jellyfin DB corruption runbook](jellyfin-db-corruption.md) (relevant because cold-stopping Jellyfin mid-stream can corrupt the SQLite DB)
