# Runbook: Jellyfin transcoding and ffmpeg

When a play "doesn't work," the cause is almost always one of: codec mismatch the client cannot direct-play, transcoding failure inside the pod, or insufficient bandwidth between server and client. ffmpeg is the part doing the work, so most diagnostics start by reading what ffmpeg said.

## Symptoms and what they usually mean

| Symptom | Likely cause |
|---------|--------------|
| "Playback error" immediately, before any video frame | Profile mismatch (codec, container, level), or transcode failed to start |
| Plays for a few seconds then stops | Transcode segment generation falling behind, or stream killed by a session limit |
| Audio works, video black or green | Hardware accel pipeline failing, falling back to CPU silently then dying on time budget |
| Stream choppy on local LAN | Transcode being used unnecessarily; check direct-play eligibility |
| Server pod CPU pegged at 100% during a single stream | Transcoding in software when hardware accel was expected to be available |
| Multiple streams all stutter together | Concurrent transcode count exceeded what the GPU/CPU can sustain |

## Triage

### Step 1: read the ffmpeg log for the failed session

Jellyfin writes every ffmpeg invocation to `/config/log/`. The file name is timestamped and includes the session ID.

```bash
POD=$(kubectl -n media get pod -l app.kubernetes.io/name=jellyfin -o jsonpath='{.items[0].metadata.name}')

# Most recent transcode log
kubectl -n media exec $POD -- ls -t /config/log/ffmpeg-transcode-*.txt | head -1

# Tail the last one
kubectl -n media exec $POD -- sh -c 'tail -200 $(ls -t /config/log/ffmpeg-transcode-*.txt | head -1)'
```

Things to grep for:

- `Failed to open codec` or `No such codec`: missing codec, almost always because the container build of jellyfin-ffmpeg lost a codec or the input has a codec the build does not include (Dolby Vision profile 8, certain DTS variants).
- `vaapi_open_internal: Failed to initialize VAAPI device`: Intel QSV / VAAPI hardware accel cannot reach the GPU. Almost always permissions on `/dev/dri/renderD128` inside the pod.
- `Cannot load nvcuda.dll` or `nvenc not loaded`: NVENC path broken. Usually the wrong driver inside the pod versus the host, or no `nvidia.com/gpu` resource on the Deployment.
- `Decoder ... not allowed`: Jellyfin's profile is rejecting the source codec for transcode. Codec license / build flag.
- `Invalid data found when processing input`: source file is bad, or the network mount returning truncated data. Try `ffprobe` on the file directly.

### Step 2: confirm hardware accel is wired up

Hardware-accelerated transcoding is the difference between a server that handles 4 streams and one that handles 1. It is also a frequent silent regression after image upgrades.

For Intel QSV / VAAPI:

```bash
# Does the pod see the iGPU device?
kubectl -n media exec $POD -- ls -la /dev/dri/

# Expected: card0 + renderD128, both readable by the jellyfin user (UID 1000 typically)
# If renderD128 has mode 0660 root:render and the pod runs as 1000, transcode will fail.

# What does ffmpeg think it can do?
kubectl -n media exec $POD -- /usr/lib/jellyfin-ffmpeg/ffmpeg -hide_banner -hwaccels
# Expected output includes: vaapi, qsv (Intel), or cuda/nvenc (NVIDIA)
```

For NVIDIA / NVENC:

```bash
kubectl -n media exec $POD -- nvidia-smi   # should list the GPU
kubectl -n media describe pod $POD | grep -i nvidia.com/gpu
# Expected: "nvidia.com/gpu: 1" in resources
```

If the device is missing inside the pod, the issue is at the Deployment manifest level (not Jellyfin itself):

- VAAPI: needs `/dev/dri` device added via `volumes` + `volumeMounts` and `securityContext.runAsGroup` set to the host's `render` group GID, OR `securityContext.privileged: true` (heavier).
- NVENC: needs `nvidia.com/gpu: 1` in resources, NVIDIA device plugin running on the node, and `runtimeClassName: nvidia` if the cluster uses runtime classes.

### Step 3: check the user's playback profile

Sometimes ffmpeg is fine and the problem is upstream: Jellyfin is choosing to transcode where it could direct-play, or refusing direct-stream because the client claims unsupported.

```bash
# Tail the main Jellyfin log during a known-bad play
kubectl -n media logs $POD -f | grep -iE "stream|playback|transcoding decision|videocodec|audiocodec"
```

The key line is "Transcoding decision: ...". It will say the chosen reason ("Container is not supported", "Audio codec is not supported", "Bitrate is too high"). Most of the time this is correct and the answer is "fix the client" (e.g. the iOS native player that lies about HEVC support). Sometimes it is a misconfigured custom profile in Dashboard, easy to fix.

## Fix patterns

### Hardware accel broken after image bump

```yaml
# Make sure the Deployment has these for VAAPI / Intel QSV:
spec:
  template:
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        supplementalGroups: [44, 109]   # video, render groups on Debian-based images
      containers:
      - name: jellyfin
        resources:
          limits:
            gpu.intel.com/i915: 1       # if using the Intel device plugin
        volumeMounts:
        - mountPath: /dev/dri
          name: dri
      volumes:
      - name: dri
        hostPath:
          path: /dev/dri
          type: Directory
```

Verify the host's `render` group GID matches one of the supplementalGroups. On Debian 12 it is 109; older may be 110.

### Transcoding works but the box is melting

Cap concurrent transcodes. Dashboard > Playback > "Maximum number of concurrent stream transcodes" set conservatively (1 to 2 for a small iGPU, 4 to 6 for an Arc / dedicated GPU). Better to refuse a play than to deliver four broken ones.

### Direct-play forced where transcoding is wanted

A specific client-profile override in `/config/data/dlna/profiles/users/` may be forcing direct play and failing on the wall clock instead of falling back. Move the profile aside, restart Jellyfin, retest with the default profile.

### Source file is corrupt

```bash
# Run from the pod (uses jellyfin-ffmpeg, not system ffmpeg)
kubectl -n media exec $POD -- /usr/lib/jellyfin-ffmpeg/ffprobe -v error \
  -show_streams -show_format \
  "/media/<path-to-the-bad-file>"
```

If `ffprobe` errors out, the file is the problem, not Jellyfin. Re-grab from the *arr stack or restore from snapshot.

## Telemetry to add (if not present)

The lab's Prometheus stack already scrapes pod CPU and memory. Useful Jellyfin-specific signals if you want them:

- `jellyfin_active_sessions` from the Jellyfin Prometheus plugin.
- `jellyfin_transcoding_sessions` count.
- A `nvidia-smi` exporter (or DCGM) if running NVENC; track GPU utilization and encoder utilization separately. The lab has had GPU temp via `node_hwmon` (see [homepage temperatures panel](../architecture/monitoring.md)) but no direct NVENC saturation metric.

A "transcoding active for >X min" alert is more useful than "Jellyfin is up", because Jellyfin can be up and Ready while every stream silently fails.

## Patterns worth knowing

- **Read the ffmpeg log first, every time.** It is verbose and intimidating, but the actual error is in there in plain text and saves hours of guessing.
- **Hardware accel is a silent regression magnet.** Every image rebuild, every kernel upgrade, every CSI/device-plugin change can break it without a clear failure. Add a synthetic transcode check (a tiny file ffmpeg-transcoded on a CronJob) that pages on failure.
- **Refuse, don't degrade.** Capping concurrent transcodes preserves quality of service. Letting a small box accept 6 streams and serve 6 broken ones is worse than serving 2 well.
- **SQLite + active stream + abrupt restart = corruption.** Treat it as a hard rule. See the [DB corruption runbook](jellyfin-db-corruption.md).
