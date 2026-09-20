# Migrate to tiered NAS storage

The execution plan for [ADR 0008](../decisions/0008-storage-tiering.md): move
VM disks and Kubernetes PVCs onto an NVMe mirror, leave bulk data on SATA,
and stop wasting two SN850X on a SLOG that holds 11 MiB and an L2ARC that
costs more ARC than it returns.

**Four stages, in this order**, cheapest and most reversible first. Each is
independently useful. Stop after any of them.

| Stage | Risk | Downtime | Reversible |
|---|---|---|---|
| 1. Remove L2ARC | Very low | None | Seconds |
| 2. Build `fast`, move hot data | Medium | None, live moves | Yes, move back |
| 3. Retire SLOG | Low | None | Re-add a partition |
| 4. Rebalance `tank` mirrors | **Highest** | None, but resilvers | Per-drive |

!!! danger "Stage 4 is the one that can lose data"
    Resilvering is sustained write load, and sustained write load is exactly
    what makes the bad-batch drives drop off the SATA bus. Do one vdev at a
    time, watch `dmesg`, and never start it while another stage is in flight.

## Before anything

Record the baseline, because "it feels faster" is not a result:

```bash
# ARC and L2ARC
awk '/^(size|hits|misses|l2_hdr_size|l2_hits|l2_misses)/ {print $1, $3}' \
    /proc/spl/kstat/zfs/arcstats

# Drive error baseline. Note the number.
sudo dmesg | grep -ciE 'COMRESET|hard resetting|failed command'

# Pool latency
sudo zpool iostat -l tank 5 3
```

On 2026-09-20 the baseline was: ARC 48.2 GiB with a 99.27% hit rate,
`l2_hdr_size` 20.6 GiB, zero drive errors since the reboot, NFS probe about
18 ms idle, etcd WAL fsync about 74 ms.

## Stage 1: remove the L2ARC

The best change here. No migration, no downtime, undone in seconds.

```bash
sudo zpool remove tank 2f3c4144-9531-4f3f-be53-85f4186b0584   # nvme1n1p1
sudo zpool status tank        # cache section should be gone
```

Verify it did what it was supposed to:

```bash
awk '/^(size|l2_hdr_size|hits|misses)/ {print $1, $3}' /proc/spl/kstat/zfs/arcstats
```

`l2_hdr_size` should be 0 and ARC `size` should climb by roughly 20 GiB as
the headers are freed. Give it a few minutes and some real traffic.

**Check the hit rate does not fall.** It should not: ARC was already serving
99.27% of reads and it just got 20 GiB bigger. If it drops noticeably,
something in the assumption was wrong and re-adding the cache device is one
command.

## Stage 2: build `fast` and move the hot data

### 2a. Create the pool

`nvme0n1` still holds the SLOG at this point, so free it first or do stage 3
before this one. Either order works; the SLOG is not load-bearing.

```bash
sudo zpool remove tank a1bfbe13-bfc0-4e7c-92e4-c28c963225e6   # nvme0n1p1
sudo zpool create -o ashift=12 fast mirror /dev/nvme0n1 /dev/nvme1n1
sudo zpool status fast
```

Match the dataset properties that `tank/proxmox_zfs_vms` uses today
(`zfs get all tank/proxmox_zfs_vms`), in particular `volblocksize` for zvols
and `compression`. Do not let defaults silently change the geometry.

### 2b. Add the new Proxmox storage

`fast` needs its own ZFS-over-iSCSI storage definition alongside
`truenas-iscsi`, pointing at `fast/proxmox_zfs_vms` on the same portal
(`10.10.12.2`) and target. Do not edit the existing definition: both must
coexist while disks move between them.

### 2c. Move the VM disks, live, one at a time

Exactly the procedure from the
[NAS reboot runbook](nas-reboot-evacuate-vm-disks.md), with a different
target:

```bash
pvesh create "/nodes/$node/qemu/$id/move_disk" \
    --disk scsi0 \
    --storage truenas-nvme \
    --delete 1 \
    --bwlimit 204800
```

The same rules apply and they are not optional:

- **Serialise.** One move at a time across the whole cluster.
- **Keep the 200 MiB/s throttle.** Reads still come off the SATA pool with
  the bad drives in it.
- **Extract the UPID properly** on cross-node calls:
  `grep -oE 'UPID:[A-Za-z0-9:@._-]+' | tail -1`. `tail -1` alone returns a
  log line and makes a successful move look failed.
- **Watch etcd.** `etcd_disk_wal_fsync_duration_seconds` above 50 ms means
  stop. The k8s nodes are the ones to move most carefully.

Expect elevated NFS latency throughout. On 2026-09-19 a comparable migration
took the Uptime Kuma NFS probe from an 18 ms baseline to a 170 ms average
with 1.2 s peaks, and it cleared the moment the lane finished. That is cost,
not fault.

### 2d. Move the Kubernetes PVC dataset

`tank/k8s` is 44 GB and is served over NFS, not iSCSI, so this is a
`zfs send | zfs recv` rather than a disk move. It needs the workloads
quiesced, because the NFS export path changes.

Scale the consumers down **through Git**, not `kubectl scale`, or ArgoCD
self-heal will put them straight back. See the *arr SQLite runbook for why
that matters.

## Stage 3: retire the SLOG

If stage 2a has not already removed it:

```bash
sudo zpool remove tank a1bfbe13-bfc0-4e7c-92e4-c28c963225e6
```

`tank` now carries media, backups and ISOs, which is async-dominated work. If
sync write latency ever becomes a real complaint, add a **32 GB partition**,
not a whole device.

## Stage 4: rebalance the `tank` mirrors

The goal: no mirror holds two VE1R9204 drives. Target layout in
[ADR 0008](../decisions/0008-storage-tiering.md#4-rebalance-tank-so-no-mirror-pairs-two-bad-drives).

!!! danger "Identify drives by serial and PARTUUID, never by sd letter"
    On this machine `sdb` sits on `ata1` and `sda` on `ata2`. Alphabetical
    order is actively misleading, not merely unstable.

    ```bash
    sudo zpool status tank -L
    readlink -f /dev/disk/by-partuuid/<uuid>     # -> /dev/sdX
    readlink -f /sys/block/sdX                   # -> .../ataN/...
    ```

    On the DXP8800, LED `diskN` = kernel `ataN` = the Nth tray from the left.

    On 2026-09-16 bay 5 was lit, **bay 3 was pulled twice**, bay 3 held the
    only live drive in `mirror-3`, and that vdev briefly had zero working
    disks. Confirm which port actually changed after any reseat:

    ```bash
    journalctl -k | grep -E 'ata[0-9]+: SATA link (up|down)|detaching|Attached SCSI disk'
    ```

Do one vdev at a time with `zpool replace`, and between each:

```bash
sudo zpool status tank           # wait for resilver to finish completely
sudo dmesg | grep -ciE 'COMRESET|hard resetting|failed command'
```

**If the error count rises above the baseline, stop.** A bad drive dropping
off mid-resilver on a vdev that is already degraded is how a pool is lost.

Best done **after** the RMA replacements arrive (ansible-quasarlab#193), so
the bad drives are being removed rather than shuffled. If the RMA is slow,
the rebalance is still worth doing on its own, because today `mirror-3` has
no reliable member at all.

## When it is done

Update [ADR 0008's Validation section](../decisions/0008-storage-tiering.md#validation)
with what actually happened, not what was expected:

1. ARC size and hit rate after removing L2ARC.
2. NFS probe latency and etcd WAL fsync after the move, against the 18 ms and
   74 ms baselines.
3. `zpool status` showing no mirror with two VE1R9204 serials.

## Rollback

| Stage | Undo |
|---|---|
| 1 | `zpool add tank cache /dev/nvme1n1` |
| 2 | `qm disk move` back to `truenas-iscsi`, same procedure |
| 3 | `zpool add tank log <partition>` |
| 4 | `zpool replace` back, one vdev at a time |

Nothing here is one-way except a destroyed pool, which is why `fast` is
created new rather than by reshaping `tank`.
