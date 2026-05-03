# Runbook: Jellyfin database corruption from ungraceful shutdown

## Symptom

After a node reboot, evict, manual `kubectl delete pod`, or any forced restart **while a stream was active**, Jellyfin comes back with one or more of:

- `library.db: malformed disk image` in the logs.
- Web UI loads but the library is empty, or specific items are gone.
- Plays appear to start but immediately fail with "Playback error" or stop after a few seconds.
- Background scan errors continuously, recreating thumbnails.
- `sqlite3 /config/data/library.db "PRAGMA integrity_check;"` returns anything other than `ok`.

SQLite is the proximate cause: Jellyfin keeps several SQLite databases (`library.db`, `playback_reporting.db`, sometimes `jellyfin.db` depending on version) under `/config/data/`, and SQLite is intolerant of being killed mid-write. A streaming session means writes are happening every few seconds (progress, transcode state, watch position), so the window for corruption is wide.

## Why this happens to *me* in particular

Three contributing factors in the lab:

1. **`emptyDir` and `hostPath`-style volumes.** Anything that doesn't survive a pod move loses transactional state mid-write.
2. **`terminationGracePeriodSeconds` defaults to 30s.** Jellyfin can take longer than that to flush, finalize transcodes, and close DB handles. SIGKILL at the 30s mark catches it mid-fsync.
3. **Eager rolling updates.** `strategy: RollingUpdate` with `maxSurge=0, maxUnavailable=1` will start tearing down the old pod before the new one is Ready. Same risk as a forced delete.

## Triage

Confirm corruption versus other failure modes:

```bash
# Get the live pod
POD=$(kubectl -n media get pod -l app.kubernetes.io/name=jellyfin -o jsonpath='{.items[0].metadata.name}')

# Last 200 lines of Jellyfin's own log (errors usually mention library.db by name)
kubectl -n media logs $POD --tail=200 | grep -iE "malformed|corrupt|locked|database disk image"

# Run SQLite's built-in integrity check on the live DB
kubectl -n media exec $POD -- sqlite3 /config/data/library.db "PRAGMA integrity_check;"
kubectl -n media exec $POD -- sqlite3 /config/data/library.db "PRAGMA quick_check;"

# How big are the .db-wal and .db-shm sidecar files? Large sidecars = uncommitted WAL.
kubectl -n media exec $POD -- ls -la /config/data/ | grep -E "\.db(-wal|-shm)?$"
```

`integrity_check` returning anything other than `ok` is dispositive. If it returns a list of corrupt indices or pages, recovery is possible (next section). If it segfaults or the DB cannot be opened at all, you are restoring from backup.

## Fix

### Step 0: stop the bleed

Scale Jellyfin to zero before doing anything else. Every minute the pod is up, it's writing to the corrupt DB and making the recovery diff worse.

```bash
kubectl -n media scale deploy jellyfin --replicas=0
kubectl -n media wait pod -l app.kubernetes.io/name=jellyfin --for=delete --timeout=120s
```

### Step 1: take a forensic copy

Before touching anything, make a byte-for-byte copy of the DB and its WAL/SHM files. Mount the PVC into a debug pod or copy off-cluster. Keep this copy until you are sure the recovery worked and the new DB is good. This is your only undo.

### Step 2: try the in-place repair

For "soft" corruption (PRAGMA integrity_check returns a list of broken indices, page count finite):

```bash
# From inside a debug pod that has /config mounted at the same path:
cd /config/data
sqlite3 library.db ".recover" | sqlite3 library.db.recovered
mv library.db library.db.broken
mv library.db.recovered library.db
rm -f library.db-wal library.db-shm
```

`.recover` reads what it can salvage and emits a fresh SQL dump that is then loaded into a new database file. It is the most reliable in-place repair. Indices and FTS tables are rebuilt by Jellyfin on next startup.

### Step 3: when in-place repair fails, restore from backup

The lab takes nightly `pg_dump`-style backups of Jellyfin's `/config` to TrueNAS. Restore the most recent good copy:

```bash
# Identify the latest snapshot
zfs list -t snapshot -o name,creation tank/files/jellyfin | tail
# Or, if backups are tarballs:
ls -la /mnt/tank/backups/jellyfin/ | tail

# Restore the .db files from snapshot, leave media files alone
# (run from the truenas shell or a host that has the share mounted)
```

Be selective: you only want to overwrite `/config/data/*.db*` and `/config/metadata/`. Leaving `/config/cache/` alone is fine. Do **not** restore the watch-history database from too far back if avoidable, users notice.

### Step 4: bring it back

```bash
kubectl -n media scale deploy jellyfin --replicas=1
kubectl -n media wait pod -l app.kubernetes.io/name=jellyfin --for=condition=Ready --timeout=180s
```

Tail the log for the first scan cycle. Expect re-indexing, re-thumbnailing, and on the first stream a brief delay while Jellyfin rebuilds the position cache.

## Prevention

Apply all of these. Each is cheap individually and they compound.

- **Bump the grace period.** `terminationGracePeriodSeconds: 120` on the Deployment. Jellyfin gets two full minutes to flush before SIGKILL.
- **Configure SQLite for safer crashes.** Jellyfin honors a few env vars; the most useful is keeping WAL mode (default) and ensuring `synchronous=FULL` instead of `NORMAL`. Slower writes, much smaller corruption window.
- **Drain the stream before shutdown.** A `preStop` hook that hits the Jellyfin admin API to stop sessions, then `sleep 10`:

  ```yaml
  lifecycle:
    preStop:
      exec:
        command:
        - /bin/sh
        - -c
        - |
          curl -s -X POST -H "X-Emby-Token: ${JF_API_TOKEN}" \
            http://127.0.0.1:8096/Sessions || true
          sleep 10
  ```

- **One replica only, with `Recreate` strategy.** `RollingUpdate` for a stateful pod sharing a PVC will run two pods concurrently against the same SQLite file, which is its own corruption mode.
- **Schedule planned restarts** at low-traffic windows. A Discord-active-sessions check (the metric is already exposed) can gate a `kubectl rollout restart` so you never rotate during a play.
- **Long term: move off SQLite.** Jellyfin's experimental MariaDB / Postgres plugin is the durable answer. The lab already runs Postgres on `192.168.1.123` and has spare capacity. The migration is a known, finite task and would close this whole class of failure.

## Related

- [Jellyfin transcoding / ffmpeg troubleshooting](jellyfin-transcoding-ffmpeg.md) for stream-side issues that are *not* DB corruption.
- The [pg_hba runbook](pg-hba-rejecting-k8s-pods.md) is the equivalent class of failure for the Postgres-backed apps; the lesson "external state requires explicit care across pod restarts" is the same.
