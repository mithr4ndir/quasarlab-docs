# ADR 0008: Tier the NAS storage, NVMe for VMs and SATA for bulk

**Status:** Accepted
**Date:** 2026-09-20

## Context

The NAS has eight 4 TB SATA SSDs, two 4 TB WD_BLACK SN850X NVMe, and 62 GB of
RAM. Every byte of lab state lives here, over iSCSI (VM disks) and NFS
(Kubernetes PVCs and media).

The allocation is backwards. The two fastest devices do the least useful
work, and everything latency-critical sits on the slowest and least reliable
drives.

### What is actually on the pool

| Dataset | Used | Latency-sensitive |
|---|---|---|
| `media` | **4.21 T** | No. Sequential, cache-friendly |
| `proxmox_zfs_vms` | 656 G | **Yes.** Every VM disk, etcd included |
| `backups` | 158 G | No |
| `isos` | 59 G | No |
| `k8s` (NFS PVCs) | 44 G | **Yes.** Prometheus, Loki, the *arr SQLite |

`media` is **82% of the data and the least demanding**. Everything that
genuinely needs IOPS totals about **700 GB**.

### What the NVMe is doing instead

Both SN850X are consumed, and neither is earning it.

**SLOG (`nvme0n1`, serial 24160W805392): 11.5 MiB used of 3.64 TB.** A SLOG
holds a few seconds of sync writes. Even saturating 10 GbE, that is tens of
gigabytes. This is 0.0003% utilisation of a 4 TB device.

**L2ARC (`nvme1n1`, serial 23524A803293): actively harmful.** Measured
2026-09-20 from `/proc/spl/kstat/zfs/arcstats`:

| Metric | Value |
|---|---|
| ARC size / `c_max` | 48.2 GiB / 61.5 GiB |
| **`l2_hdr_size`** | **20.6 GiB** |
| **ARC spent on L2ARC headers** | **43%** |
| ARC hit rate | **99.27%** (41,813,114 hits / 306,187 misses) |
| L2ARC hit rate, of ARC misses | 44.4% |
| **L2ARC share of all reads served** | **0.295%** |
| L2ARC size vs RAM | 3.4 TB vs 62 GB, about **55x** |

ARC already serves **99.27%** of reads out of RAM. L2ARC only ever sees the
0.73% that miss, and catches 44% of those, so it serves roughly **three
reads in a thousand**. For that it consumes **43% of the RAM cache** in
header overhead.

The usual guidance is that L2ARC should not exceed 5 to 10 times RAM. This is
55 times. It is not a cache, it is a tax on the cache that works.

### Why this surfaced now

The [2026-09-19 NFSv4 deadlock](../incidents/2026-09-19-nfsv4-callback-deadlock.md)
put every VM disk in the lab on a single SATA pool that also contains
**three Inland SSDs pending RMA** for repeated SATA bus dropouts, two of them
paired in the same mirror (ansible-quasarlab#193). The incident made the
coupling obvious: one appliance, one pool, one class of failure.

## Decision

Tier the storage by what the data actually needs.

### 1. Destroy the L2ARC

`zpool remove tank <cache device>`. Instant, reversible, no migration.
Returns **20.6 GiB to ARC**, which should push an already 99.27% hit rate
higher. This is the highest value per unit of risk of anything here and is
done first.

### 2. Create `fast`, a mirror of the two SN850X, and move the hot data there

`proxmox_zfs_vms` (656 G) and `k8s` (44 G) move to `fast`. About 700 GB on
3.6 TB usable, **19% full**, with room to grow several times over.

This takes every VM disk and every Kubernetes PVC **off the SATA drives
entirely**, which also takes them off the three RMA-bound Inlands.

### 3. Retire the SLOG

Once `tank` holds only media, backups and ISOs, its workload is
async-dominated and a dedicated log device earns little. The `fast` pool does
not want a SLOG made of the same class of device it is already built from.

If a SLOG is wanted later, a 32 GB partition is generous. Not 4 TB.

### 4. Rebalance `tank` so no mirror pairs two bad drives

Today `mirror-3` holds **both** remaining bad-batch VE1R9204 drives
(`IB24AK0004S00064` and `IB24AK0004S00004`), while `mirror-0` holds two
reliable WD Blues. That is the worst possible arrangement: redundancy is
doubled where it is not needed and absent where it is.

With three WD Blue, three bad Inland and two good Inland (VE1R9004):

| vdev | Proposed |
|---|---|
| mirror-0 | WD Blue + bad Inland |
| mirror-1 | WD Blue + bad Inland |
| mirror-2 | WD Blue + bad Inland |
| mirror-3 | good Inland + good Inland |

Every vdev then survives the loss of any single bad-batch drive. Today
`mirror-3` does not.

## Considered

- **Leave it alone.** The pool is healthy and 34% full. Rejected: "healthy"
  is exactly what the SATA drives looked like while two of them were dead,
  and the L2ARC is measurably costing more than it returns today.
- **Add the NVMe as a `special` vdev on `tank`** (metadata and small blocks).
  Genuinely attractive, and it would accelerate the whole pool without
  splitting it. Rejected for two reasons. A `special` vdev is not optional
  once written: losing it loses the pool, so it inherits the blast radius
  rather than reducing it. And 3.6 TB of special for a 5 TB pool is wildly
  oversized, which is the same mistake as the current L2ARC in a new hat.
- **Keep L2ARC but shrink it** to roughly 500 GB. Defensible on paper, but at
  a 99.27% ARC hit rate the ceiling on any L2ARC benefit is the remaining
  0.73%. Spending a 4 TB NVMe to chase that, while VM disks sit on failing
  SATA, is the wrong allocation at any size.
- **Move media to spinning disks and keep everything SSD-backed for VMs.**
  No spare bays and no spinning disks. Revisit only if the chassis grows.

## Consequences

- **`fast` becomes a single failure domain for every VM.** Two-way mirror, so
  one device can fail. This is not a regression: `tank` is that failure
  domain today, and two SN850X are far more trustworthy than five Inlands of
  which three are RMA candidates.
- **`tank` going down stops mattering as much.** With no VM disks on it, a
  repeat of 2026-09-19 would cost media and backups access, not the whole
  lab. That is the strongest argument in this ADR.
- **Two pools to manage**, two capacity numbers to watch, and a new Proxmox
  storage definition. Accepted.
- **Media stays on SSD**, which is arguably a waste of SSD. It is the
  hardware that exists, and freeing those bays is not a goal.
- **The migration is not free.** About 700 GB of zvols move between pools,
  live, one VM at a time. The procedure is already proven: it is what
  rebooted the NAS with zero VM downtime on 2026-09-19, and the same
  throttle and etcd cautions apply. See the
  [runbook](../runbooks/nas-storage-tiering-migration.md).
- **The rebalance requires resilvering**, which puts real load on drives that
  have failed before. The trigger for those failures is **not established**
  (see the [failure analysis](../incidents/2026-09-19-nfsv4-callback-deadlock.md#about-the-drive-failures-trigger-unknown)),
  so this is not "avoid writes and it will be fine". It is "these drives fail
  unpredictably, so do one vdev at a time and watch `dmesg`, because a
  dropout during a resilver on an already-degraded vdev is how a pool is
  lost".

## Validation

None of this is validated yet. It is a plan based on measurements, and the
measurements are the only part currently proven.

What would count as validation, in order:

1. After removing L2ARC, `l2_hdr_size` is gone and ARC `size` grows by
   roughly 20 GiB. Confirm the ARC hit rate does not fall.
2. After the move, NFS probe latency from Uptime Kuma and etcd WAL fsync both
   improve, measured against tonight's baseline of about 18 ms idle NFS reads
   and 74 ms etcd fsync.
3. After the rebalance, `zpool status` shows no mirror with two VE1R9204
   serials.

Record the results here rather than assuming them. The lab has been caught
several times by a plan recorded as though it were an outcome, most recently
in [ADR 0005](0005-healthchecks-deadman.md), whose documented 10-minute grace
period turned out to be 48 hours in production.

## Related

- [2026-09-19 NFSv4 callback deadlock](../incidents/2026-09-19-nfsv4-callback-deadlock.md), which put all of this in one pool and made the coupling visible.
- [Runbook: tiering migration](../runbooks/nas-storage-tiering-migration.md).
- [Runbook: NAS reboot with zero VM downtime](../runbooks/nas-reboot-evacuate-vm-disks.md), the same live disk-move technique.
- ansible-quasarlab#193, the RMA and the `mirror-3` pairing.
- [Storage architecture](../architecture/overview.md#storage).
