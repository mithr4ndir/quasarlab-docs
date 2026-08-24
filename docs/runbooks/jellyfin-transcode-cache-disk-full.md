# Runbook: Jellyfin transcode cache filling the root filesystem

> Jellyfin runs as a native systemd service on VM 115 (`ssh ladino@192.168.1.170`). Root is a 51 GB disk shared by the OS, `jellyfin.db`, `/var/log` **and** the transcode cache. From [2026-08-24](../incidents/2026-08-24-jellyfin-transcode-disk-exhaustion.md).

## Symptom you would actually see

- `DiskFillingFast` or `DiskSpaceLow` for `instance=jellyfin`, `mountpoint=/`.
- Several people streaming, **CPU almost idle**, disk filling fast anyway.
- `/var/cache/jellyfin/transcodes` holding thousands of `<32-hex><n>.mp4` files.
- Worst case: streams die together and Jellyfin logs SQLite write errors.

!!! danger "Do not restart Jellyfin to reclaim space while people are watching"
    Restarting mid-stream is how you turn a disk problem into
    [database corruption](jellyfin-db-corruption.md). Free space first; restart later.

## Quick triage

```bash
ssh ladino@192.168.1.170
df -h /
sudo du -sh /var/cache/jellyfin/transcodes
sudo ls /var/cache/jellyfin/transcodes | wc -l

# Fill rate: is this minutes away or hours?
A=$(df --output=avail / | tail -1); sleep 30; B=$(df --output=avail / | tail -1)
echo "$(( (A-B) / 1024 )) MB in 30s"
```

Confirm it is remux, not re-encode. `-codec:v:0 copy` means the video is being
stream-copied and CPU will look fine:

```bash
pgrep -af ffmpeg | head
```

Who is actually watching (Jellyfin's own session IPs are the proxy, not clients):

```bash
K=$(sudo cat /etc/jellyfin/monitoring-api-key)
curl -sf -H "Authorization: MediaBrowser Token=$K" http://localhost:8096/Sessions \
  | python3 -c 'import json,sys; print(len([s for s in json.load(sys.stdin) if s.get("NowPlayingItem")]), "playing")'
```

## Fix

### 1. Reclaim space without touching live playback

HLS segments below the slowest playhead are consumed and safe to delete. Find
the lowest position, convert to a segment index (`pos / 6`), and leave a buffer
of ~100 segments.

```bash
# Lowest playhead across active sessions, in seconds
K=$(sudo cat /etc/jellyfin/monitoring-api-key)
curl -sf -H "Authorization: MediaBrowser Token=$K" http://localhost:8096/Sessions \
  | python3 -c 'import json,sys; S=[s for s in json.load(sys.stdin) if s.get("NowPlayingItem")]; print(min((s.get("PlayState",{}).get("PositionTicks") or 0)//10000000 for s in S))'
```

With the lowest playhead at 4861s: `4861/6 ≈ 810`, so cutting below **700**
leaves an 11-minute buffer. Set `CUTOFF` accordingly:

```bash
sudo bash -c 'CUTOFF=700; n=0; bytes=0
for f in /var/cache/jellyfin/transcodes/*.mp4; do
  b=$(basename "$f")
  if [[ "$b" =~ ^[0-9a-f]{32}([0-9]+)\.mp4$ ]] && (( BASH_REMATCH[1] < CUTOFF )); then
    sz=$(stat -c %s "$f"); rm -f "$f" && { n=$((n+1)); bytes=$((bytes+sz)); }
  fi
done
echo "deleted $n segments, freed $((bytes/1024/1024)) MB"'
```

The `-1.mp4` init segments are skipped by that pattern deliberately. Deleting one
breaks its session.

Then confirm nobody dropped:

```bash
curl -sf -H "Authorization: MediaBrowser Token=$K" http://localhost:8096/Sessions \
  | python3 -c 'import json,sys; print(len([s for s in json.load(sys.stdin) if s.get("NowPlayingItem")]), "still playing")'
```

### 2. Once everyone has stopped

Jellyfin reclaims its own cache at session end, so verify before doing more:

```bash
df -h / && sudo du -sh /var/cache/jellyfin/transcodes
```

## Follow-up so it does not bite again

`EnableSegmentDeletion=true` and `SegmentKeepSeconds=720` are set in
`roles/jellyfin/templates/encoding.xml.j2` (ansible#143). With deletion off,
each session writes the **whole title** to disk and holds it to the end — six
sessions on one 96-minute film was 33 GB.

```bash
sudo grep -E 'SegmentDeletion|SegmentKeep' /etc/jellyfin/encoding.xml
```

If that shows `false`, the config drifted. Re-run `playbooks/jellyfin.yml`
**when nobody is streaming** — it notifies a Jellyfin restart.

### Why it fills even with plenty of CPU headroom

A session shows as "Transcode" whenever *anything* is converted, including a
container remux with `IsVideoDirect=true`. Video stream-copy is nearly free on
CPU and writes at the source video bitrate. Common triggers here:

- `AnamorphicVideoNotSupported` — non-square pixels (one file had SAR `481:480`).
- `SecondaryAudioNotSupported` — more than one audio track flagged `default`.
- `AudioCodecNotSupported` — browsers rejecting E-AC3.

Browsers cannot play MKV at all, so browser clients always remux. Native clients
(Jellyfin Media Player, Moonfin on webOS) direct-play the same file and use
**zero** transcode cache. For a watch party, that is a bigger lever than any
server setting.

### Related

- [Jellyfin database corruption](jellyfin-db-corruption.md) — what a mid-stream restart costs
- [Jellyfin transcoding and ffmpeg](jellyfin-transcoding-ffmpeg.md)
- [2026-08-24 incident](../incidents/2026-08-24-jellyfin-transcode-disk-exhaustion.md)
