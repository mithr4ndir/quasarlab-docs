# Runbook: Jellyfin database corruption from ungraceful shutdown

> Jellyfin runs as a native systemd service (`jellyfin.service`) on VM 115, not in Kubernetes or Docker (it moved off both in March 2026). All paths and commands below are for the native install. Reach the VM by SSH (`ssh ladino@192.168.1.170`) or from a Proxmox host with `qm guest exec 115 -- <cmd>`.

## Symptom

After the VM was stopped or reset **while a stream was active**, Jellyfin comes back with one or more of:

- `jellyfin.db: malformed disk image` (or `database disk image is malformed`) in the log.
- Web UI loads but the library is empty, or specific items are gone.
- Plays appear to start but immediately fail with "Playback error" or stop after a few seconds.
- Background scan errors continuously, recreating thumbnails.
- `sqlite3 /var/lib/jellyfin/data/jellyfin.db "PRAGMA integrity_check;"` returns anything other than `ok`.

Jellyfin 10.11 consolidated everything into a single SQLite database at `/var/lib/jellyfin/data/jellyfin.db` (the old per-domain files like `library.db` are legacy and now 0 bytes, ignore them). Backend confirmed in `/etc/jellyfin/database.xml`: `DatabaseType = Jellyfin-SQLite`, `LockingBehavior = NoLock`. SQLite is intolerant of being killed mid-write, and a streaming session writes every few seconds (progress, transcode state, watch position), so the corruption window is wide.

## Why this happens in this setup

The K8s-era causes (emptyDir, pod eviction, rolling updates) are gone. The triggers now are all VM and GPU lifecycle events:

1. **HA failover.** When the RTX 2080 Ti hangs (see the [GPU D3cold runbook](gpu-passthrough-d3cold-recovery.md)), HA stops VM 115 and restarts it on the other node. If a stream was active, that stop can catch SQLite mid-write.
2. **gpu-watchdog stopping Jellyfin.** The in-VM watchdog's recovery path does `systemctl stop jellyfin` + `killall -9 ffmpeg` when it detects a GPU failure. That is a graceful SIGTERM to Jellyfin, but with `TimeoutStopUSec=15s` it can still SIGKILL before the DB flushes under load. (The watchdog is now K620-aware and runs in PASSIVE mode on the fallback node, which removes the spurious-stop vector that caused the 2026-05-27 thrash.)
3. **`qm stop` / cold reboot during migration or maintenance.** Stopping the VM out from under an active stream.
4. **15-second stop timeout.** `systemctl show jellyfin -p TimeoutStopUSec` is `15s`. Jellyfin under transcode load can need longer to finalize and close DB handles.

## Triage

Confirm corruption versus other failure modes. `sqlite3` is installed in the guest.

```bash
# Reach the VM (either works)
ssh ladino@192.168.1.170
# or:  ssh root@<pve-host> 'qm guest exec 115 -- /bin/bash -c "<cmd>"'

# Last 200 lines of Jellyfin's own log (errors usually name the db)
tail -200 /var/log/jellyfin/jellyfin$(date +%Y%m%d).log | grep -iE "malformed|corrupt|locked|database disk image"

# SQLite integrity checks on the live DB
sqlite3 /var/lib/jellyfin/data/jellyfin.db "PRAGMA quick_check;"
sqlite3 /var/lib/jellyfin/data/jellyfin.db "PRAGMA integrity_check;"

# Sidecar files: large -wal/-shm means uncommitted WAL waiting to be checkpointed
ls -la /var/lib/jellyfin/data/ | grep -E "jellyfin\.db(-wal|-shm)?$"
```

`integrity_check` returning anything other than `ok` is dispositive. A finite list of corrupt indices or pages means recovery is possible (next section). If the DB cannot be opened at all, restore from backup.

## Fix

### Step 0: stop the bleed

```bash
sudo systemctl stop jellyfin
# Confirm nothing is still holding the DB
sudo fuser -v /var/lib/jellyfin/data/jellyfin.db 2>/dev/null || echo "no holders"
```

Every minute Jellyfin stays up, it writes to the corrupt DB and widens the recovery diff.

### Step 1: take a forensic copy

```bash
cd /var/lib/jellyfin/data
sudo cp -a jellyfin.db jellyfin.db.corrupt-$(date +%F-%H%M)
sudo cp -a jellyfin.db-wal jellyfin.db-wal.corrupt-$(date +%F-%H%M) 2>/dev/null || true
sudo cp -a jellyfin.db-shm jellyfin.db-shm.corrupt-$(date +%F-%H%M) 2>/dev/null || true
```

Keep this copy until you are sure recovery worked. It is your only undo.

### Step 2: try the in-place repair

For soft corruption (integrity_check returns a list, the file still opens):

```bash
cd /var/lib/jellyfin/data
sudo -u jellyfin sqlite3 jellyfin.db ".recover" | sudo -u jellyfin sqlite3 jellyfin.db.recovered
sudo mv jellyfin.db jellyfin.db.broken
sudo mv jellyfin.db.recovered jellyfin.db
sudo rm -f jellyfin.db-wal jellyfin.db-shm
sudo chown jellyfin:jellyfin jellyfin.db
# Verify the recovered file
sudo -u jellyfin sqlite3 jellyfin.db "PRAGMA integrity_check;"
```

`.recover` salvages what it can into a fresh SQL dump and reloads it into a new database. It is the most reliable in-place repair. Run everything as the `jellyfin` user so ownership stays correct.

### Step 3: when in-place repair fails, restore from backup

There is **no automated app-level Jellyfin backup** in the lab today (gap, see Prevention). Two restore sources exist:

1. **Ad-hoc snapshots** in `/var/lib/jellyfin/backups/` (manual `.db` copies taken before past maintenance). Check dates:
   ```bash
   ls -la /var/lib/jellyfin/backups/
   # Restore (stop jellyfin first):
   sudo -u jellyfin cp -a /var/lib/jellyfin/backups/<chosen>.db /var/lib/jellyfin/data/jellyfin.db
   sudo rm -f /var/lib/jellyfin/data/jellyfin.db-wal /var/lib/jellyfin/data/jellyfin.db-shm
   ```
2. **TrueNAS ZFS snapshot of the VM disk.** VM 115's disk is an iSCSI zvol at `tank/proxmox_zfs_vms/vm-115-disk-0`. A snapshot rollback is whole-disk and crash-consistent, so it is the heavy hammer (it reverts the entire VM filesystem, not just the DB). Use only if the DB file itself is unrecoverable and the `backups/` copies are too old.
   ```bash
   # On TrueNAS
   zfs list -t snapshot -o name,creation tank/proxmox_zfs_vms/vm-115-disk-0 | tail
   ```

Prefer a recent `backups/` copy over a zvol rollback when possible, restoring watch history from too far back is user-visible.

### Step 4: bring it back

```bash
sudo systemctl start jellyfin
sleep 5
curl -sS -o /dev/null -w "HTTP %{http_code}\n" http://127.0.0.1:8096/health
journalctl -u jellyfin -f | grep -iE "scan|startup|error"
```

Expect re-indexing and re-thumbnailing on the first scan, and a brief delay on the first stream while the position cache rebuilds.

## Prevention

- **Never restart Jellyfin while a stream is active.** Hard rule. SQLite plus an abrupt stop equals corruption. Check first:
  ```bash
  pgrep -af ffmpeg | grep -v pgrep   # active transcodes
  ss -tn state established sport = 8096 | tail -n +2   # connected clients
  ```
- **The gpu-watchdog is a corruption vector by design.** It stops Jellyfin on GPU failure. The 2026-05-27 K620-awareness fix (PASSIVE mode on the fallback node) removed the spurious-stop loop, but a real 2080 Ti failure will still stop Jellyfin. Accept this as the lesser evil versus a wedged GPU, but know it is the most likely trigger.
- **Consider bumping the stop timeout.** `TimeoutStopUSec=15s` is tight for a loaded transcoder. A systemd drop-in raising it to `120s` gives Jellyfin time to flush before SIGKILL. (Would go in the `roles/jellyfin` Ansible role, not edited live.)
- **`LockingBehavior` is `NoLock`.** That is Jellyfin's default for single-process SQLite and is fine here, but it means zero protection if two Jellyfin processes ever touch the file (do not run a second instance against the same data dir).
- **Long term: external database.** Jellyfin 10.11's `/etc/jellyfin/database.xml` supports pluggable backends. Postgres support is still experimental upstream, but the lab already runs Postgres (see the [database-per-app decision](../decisions/0006-database-per-app.md)). When upstream marks it stable, migrating closes this entire class of failure. Track it; do not jump early while it is experimental.
- **Add a scheduled backup.** The biggest current gap. A nightly `sqlite3 jellyfin.db ".backup"` to `/var/lib/jellyfin/backups/` (which lives on the zvol and rides along with TrueNAS snapshots) plus retention would give a real restore point. Belongs in the `roles/jellyfin` role.

## Related

- [GPU passthrough D3cold recovery](gpu-passthrough-d3cold-recovery.md): the most common trigger for the ungraceful stops that cause this.
- [Jellyfin transcoding / ffmpeg](jellyfin-transcoding-ffmpeg.md): stream-side issues that are *not* DB corruption.
- [2026-05-27 incident](../incidents/2026-05-27-jellyfin-gpu-d3cold-passthrough.md): where the watchdog-stops-Jellyfin interaction was diagnosed.
