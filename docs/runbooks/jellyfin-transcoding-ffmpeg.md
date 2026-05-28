# Runbook: Jellyfin transcoding and ffmpeg

> Jellyfin runs as a native systemd service (`jellyfin.service`) on VM 115 with an RTX 2080 Ti passed through from the Proxmox host, not in Kubernetes or Docker (it moved off both in March 2026). Reach the VM by SSH (`ssh ladino@192.168.1.170`) or from a Proxmox host with `qm guest exec 115 -- <cmd>`. NVENC is the hardware accel path; there is no Intel QSV / VAAPI here.

When a play "doesn't work," the cause is almost always one of: a codec mismatch the client cannot direct-play, a transcoding failure on the GPU, the GPU watchdog killing the transcode, or insufficient bandwidth. ffmpeg does the work, so most diagnostics start by reading what ffmpeg said.

## Symptoms and what they usually mean

| Symptom | Likely cause |
|---------|--------------|
| "Playback error" immediately, before any frame | Profile mismatch (codec, container, level), or transcode failed to start |
| Plays a few seconds then stops, repeatedly | Transcode falling behind, **or the gpu-watchdog SIGTERMing ffmpeg** (see below) |
| Audio works, video black or green | Hardware decode pipeline failing, silent CPU fallback dying on the time budget |
| Stream choppy on local LAN | Transcoding used where direct-play was possible; check the decision log |
| VM CPU pegged during a single stream | Software transcode where NVENC was expected (driver gone, or HEVC source on a card that cannot decode it) |
| `Hardware is lacking required capabilities` in ffmpeg log | The card cannot decode this codec. Almost always the **K620 failover path** (Maxwell cannot do HEVC/AV1/10-bit) |
| Multiple streams all stutter together | Concurrent transcode count exceeds what the GPU can sustain |

## Triage

### Step 1: read the ffmpeg log for the failed session

Jellyfin writes every ffmpeg invocation to `/var/log/jellyfin/`. Files are `FFmpeg.Transcode-<timestamp>_<sessionid>_<hash>.log`.

```bash
# Most recent transcode log
ls -t /var/log/jellyfin/FFmpeg.Transcode-*.log | head -1

# Tail it
tail -100 "$(ls -t /var/log/jellyfin/FFmpeg.Transcode-*.log | head -1)"
```

Things to grep for:

- `Hardware is lacking required capabilities` / `Failed setup for format cuda`: the GPU cannot decode this source codec in hardware. On the 2080 Ti this should not happen for H.264/HEVC. **If you see it, you are almost certainly on the K620 failover node** (Maxwell: H.264 NVENC only, no HEVC/AV1/10-bit). Check `nvidia-smi` (next step) and see the [GPU D3cold runbook](gpu-passthrough-d3cold-recovery.md).
- `exited with code 143`: SIGTERM (128 + 15). Something killed ffmpeg. The usual culprit is the **gpu-watchdog** stopping Jellyfin and `killall -9 ffmpeg` on a (real or false-positive) GPU failure. Check `journalctl -u gpu-watchdog`.
- `Cannot load libnvcuvid` / `OpenEncodeSessionEx failed` / `no capable devices found`: NVENC/NVDEC path broken. Usually the nvidia driver in the guest is not talking to the card (`nvidia-smi` fails), which after a migration is often the [HA cmdline-cache quirk](gpu-passthrough-d3cold-recovery.md) (GPU not actually attached to the QEMU process).
- `Invalid data found when processing input`: bad source file or a truncated NFS read. `ffprobe` the file directly.

### Step 2: confirm the GPU is actually there and healthy

This is the difference between a server that handles several streams and one that handles none. It is also a frequent silent regression after a migration, reboot, or driver update.

```bash
# Always wrap nvidia-smi in timeout. A hung GPU hangs nvidia-smi forever.
timeout 10 nvidia-smi --query-gpu=name,driver_version,utilization.gpu,memory.used --format=csv
# Expected on the primary node: "NVIDIA GeForce RTX 2080 Ti, 570.xxx, ..."
# "Quadro K620" means you are on the failover node and HEVC/AV1/HDR will not transcode.
# "NVIDIA-SMI has failed..." or "No NVIDIA GPU found" means the card is not attached to the guest.

# What does ffmpeg think it can do?
/usr/lib/jellyfin-ffmpeg/ffmpeg -hide_banner -hwaccels        # expect: cuda (among others)
/usr/lib/jellyfin-ffmpeg/ffmpeg -hide_banner -encoders | grep -E "nvenc"
# Expect h264_nvenc + hevc_nvenc (+ av1_nvenc on the 2080 Ti). Only h264_nvenc on the K620.
```

If `nvidia-smi` fails inside the VM, the problem is below Jellyfin (passthrough / driver), not Jellyfin itself. Go to the [GPU D3cold runbook](gpu-passthrough-d3cold-recovery.md). In particular, after an HA-driven migration, verify the card is in the running QEMU process on the host:

```bash
# On the Proxmox host running VM 115
QEMU_PID=$(cat /run/qemu-server/115.pid)
tr "\0" "\n" < /proc/$QEMU_PID/cmdline | grep -c vfio-pci    # expect 4; if 0, qm stop 115 && qm start 115
```

### Step 3: is the watchdog killing the transcode?

```bash
systemctl is-active gpu-watchdog
journalctl -u gpu-watchdog --since "20 min ago" --no-pager | tail -20
# "Detected expected GPU: NVIDIA GeForce RTX 2080 Ti" = active recovery mode (normal on primary node)
# "entering PASSIVE mode" = on the fallback K620, watchdog will NOT kill transcodes (correct)
# Repeated "GPU failure detected ... Stopping Jellyfin" = the watchdog is the problem
```

If the watchdog is crash-looping on the primary node, the GPU is genuinely sick: go to the [GPU D3cold runbook](gpu-passthrough-d3cold-recovery.md). On the fallback node it should be PASSIVE and harmless; if it is not, the host has an old pre-2026-05-27 watchdog and needs `roles/jellyfin` re-applied.

### Step 4: check the playback decision

Sometimes ffmpeg is fine and Jellyfin is choosing to transcode where it could direct-play, or refusing direct-stream because the client lied about support.

```bash
journalctl -u jellyfin -f | grep -iE "stream|playback|transcod|videocodec|audiocodec"
```

The key line is the transcoding decision and its reason ("Container is not supported", "Audio codec is not supported", "Bitrate is too high"). Most of the time it is correct and the fix is client-side (e.g. an app that misreports HEVC support). Occasionally it is a misconfigured custom profile.

## Fix patterns

### Wrong GPU (on the K620 failover node)

If `nvidia-smi` shows a Quadro K620, the 2080 Ti hung and HA failed VM 115 over. H.264 SDR content will still transcode; HEVC/AV1/HDR will not. This is the [GPU D3cold runbook](gpu-passthrough-d3cold-recovery.md): recover the 2080 Ti (cold reboot pve2) and migrate VM 115 back.

### Hardware accel config

Encoder settings live in `/etc/jellyfin/encoding.xml` (managed by the `roles/jellyfin` Ansible role; edit there, not live). Current known-good values:

```xml
<HardwareAccelerationType>nvenc</HardwareAccelerationType>
<EnableHardwareEncoding>true</EnableHardwareEncoding>
<EnableTonemapping>false</EnableTonemapping>            <!-- HDR tonemapping crashes this card; keep off -->
<EnableEnhancedNvdecDecoder>false</EnableEnhancedNvdecDecoder>  <!-- 4K DV HDR can OOM 11GB VRAM -->
```

`EnableTonemapping` and enhanced NVDEC are deliberately off (they have crashed or OOM'd the GPU historically). Do not flip them on without testing under 4K HDR load.

### Verify the full pipeline with a synthetic transcode

```bash
# HEVC NVDEC + h264 NVENC. Exits 0 on the 2080 Ti, FAILS with
# "Hardware is lacking required capabilities" on the K620.
SAMPLE=$(find /mnt/media/library -type f -name "*.mkv" -size -500M 2>/dev/null | head -1)
timeout 30 /usr/lib/jellyfin-ffmpeg/ffmpeg -hide_banner -loglevel error \
  -hwaccel cuda -hwaccel_output_format cuda -c:v hevc_cuvid \
  -i "$SAMPLE" -t 10 -c:v h264_nvenc -preset p1 -b:v 5M -f null - && echo OK
```

### Transcoding works but the box is melting

Cap concurrent transcodes in Dashboard > Playback. The 2080 Ti handles several 1080p streams comfortably; still set a ceiling so it refuses rather than degrades. Better to reject one play than serve several broken ones.

### Source file is corrupt

```bash
/usr/lib/jellyfin-ffmpeg/ffprobe -v error -show_streams -show_format "/mnt/media/<path>"
```

If `ffprobe` errors, the file is the problem. Re-grab from the *arr stack or restore from a NAS snapshot.

## Telemetry

Already wired in the lab (textfile collectors on the VM, scraped by Prometheus):

- `gpu-metrics.timer` writes `nvidia_*` (utilization, encoder/decoder load, VRAM, temp) every 30s.
- `jellyfin-metrics.timer` writes session/transcode counts every 60s.
- `gpu_watchdog.prom` (`nvidia_gpu_failures_total`, soft/hard recovery counters) written by the watchdog.

A "transcoding active for >X min with rising failures" signal is more useful than "Jellyfin is up", because Jellyfin can be Ready while every stream silently fails.

## Patterns worth knowing

- **Read the ffmpeg log first, every time.** The actual error is in there in plain text.
- **`exited with code 143` is not an ffmpeg bug, it is SIGTERM.** Something killed it. On this host that is almost always the gpu-watchdog. Chase the watchdog, not the codec.
- **Wrap `nvidia-smi` in `timeout`.** A hung GPU hangs it forever and wedges anything that calls it.
- **`Hardware is lacking required capabilities` means wrong card, not wrong config.** You are on the K620. Recover the 2080 Ti.
- **ICMP to the VM is blocked (UFW); use HTTP for liveness.** `curl http://192.168.1.170:8096/health`, not `ping`.
- **SQLite plus an abrupt stop equals corruption.** Never restart Jellyfin mid-stream. See the [DB corruption runbook](jellyfin-db-corruption.md).

## Related

- [GPU passthrough D3cold recovery](gpu-passthrough-d3cold-recovery.md)
- [Jellyfin DB corruption](jellyfin-db-corruption.md)
- [2026-05-27 incident](../incidents/2026-05-27-jellyfin-gpu-d3cold-passthrough.md)
