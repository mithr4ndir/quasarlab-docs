# ADR 0009: What the two SN850X should actually do

**Status:** Proposed. Supersedes [ADR 0008](0008-storage-tiering.md) steps 3
and 4, and amends step 2.
**Date:** 2026-10-07
**Measurements taken:** 2026-10-07, read-only, against `192.168.1.15`,
`192.168.1.10` and the Kubernetes API.

## The question

The NAS holds two 4 TB WD_BLACK SN850X. One is `tank`'s SLOG, one is idle
since the L2ARC was removed on 2026-09-20. The question put to this ADR was
open: "maybe a new datastore for something else". [ADR 0008](0008-storage-tiering.md)
already proposed an answer, but it was written on 2026-09-20, several of its
load-bearing facts have changed, and exactly one of its four steps has been
executed.

This ADR re-derives the answer from measurements taken today. The short
version: **ADR 0008's end state is still right, its reasoning for two of the
four steps is wrong, and it omits the one constraint that determines the
whole ordering.**

## What changed since ADR 0008

| | 2026-09-20 (ADR 0008) | 2026-10-07 (today) |
|---|---|---|
| L2ARC | 3.4 TB, `l2_hdr_size` 20.6 GiB, 43% of ARC | **Gone.** `l2_size`, `l2_asize`, `l2_hdr_size`, `l2_hits` all `0` |
| VM disk backups | `tank/backups`, 158 G of vzdump, on the pool it protects | **PBS 3.4.9 on `pbs1`**, hourly, all VMs, `--verify-new`, 24/7/4/3, restore drill passed. `tank/backups` destroyed (terraform-quasarlab#26) |
| SLOG assessment | "11.5 MiB used of 3.64 TB", "not earning it" | **Earning it.** 49 write IOPS and 3.70 MiB/s sustained at 92us. See [Finding 1](#finding-1-the-slog-is-earning-its-place) |
| ARC hit rate | 99.27% | 99.08% lifetime, metadata **99.99%** |
| `tank` health | 3 drives had dropped off the bus | ONLINE, 0 errors, **0 `ata` resets in 16 days** |
| PVE | 8.4.21, EOL 2026-08-31 | Unchanged, still EOL, plugin still held at 2.4.0-1 |

Two of those changes matter enough to move the decision. PBS existing changes
the cost of every destructive step below. The SLOG measurement reverses ADR
0008 step 3.

## The constraint ADR 0008 never states

`nvme0n1p1` **is** `tank`'s log vdev right now (`blkid` reports
`LABEL="tank" TYPE="zfs_member"`, and its `PARTUUID`
`a1bfbe13-bfc0-4e7c-92e4-c28c963225e6` is the device under `logs` in
`zpool status`). `nvme1n1p1` is a member of nothing: `zdb -l` still finds a
stale **L2ARC device header** on it carrying `pool_guid`
`7043569411961946042` (which is `tank`) and `state: 4`, but `blkid` reports no
pool label.

So **the two drives cannot both join a new mirror until the SLOG is retired.**
ADR 0008 lists "create `fast`" as step 2 and "retire the SLOG" as step 3, in
that order, which is not executable. The steps are coupled and the dependency
runs the other way. Everything in the [runbook](../runbooks/nvme-tiering-migration.md)
follows from this.

## Finding 1: the SLOG is earning its place

ADR 0008 retired the SLOG on the grounds that it held 11.5 MiB of a 3.64 TB
device. That is a measurement of **allocation**, and it says nothing about
utility. ZIL blocks are freed on every txg commit, so near-zero allocation is
the correct and expected steady state for a healthy SLOG. A SLOG that had
filled up would be the pathology.

The metrics that do measure utility are IOPS, bandwidth and latency. All three
say it is working.

| Measurement | Log vdev | `tank` data vdevs | Source |
|---|---|---|---|
| Write IOPS, since boot (16d) | **49/s** | 211/s across 4 mirrors | `zpool iostat -v tank` |
| Write bandwidth, since boot | **3.70 MiB/s** | 8.78 MiB/s | same |
| Write IOPS, live 60s sample | **95/s** | 151/s | `zpool iostat -v -l tank 60 2` |
| Write `disk_wait` | **88us** | 346us to 593us | same |
| Write `total_wait` | **92us** | 1ms to 2ms | same |
| Lifetime written | **76.0 TB** | n/a | `smartctl -a /dev/nvme0` |
| Lifetime read | **4.16 GB** | n/a | same |
| Percentage Used | 2% | n/a | same |

Read the first two rows together: **42% of the bytes written to `tank`'s data
vdevs pass through the ZIL first.** That is not an async-dominated workload.

The 76.0 TB written against 4.16 GB read is the signature of a correctly
functioning SLOG. It is write-only in normal operation and is read back only
during an unclean import, so 4.16 GB over 14,654 power-on hours means the pool
has essentially never needed it for recovery. That is the SLOG working, not
the SLOG idle.

### Why there is sync traffic at all, given `sync=standard`

`sync=standard` is set everywhere (`local` on `tank/proxmox_zfs_vms`, default
elsewhere) and `logbias=latency` everywhere. Under `sync=standard` the ZIL
only sees writes the client explicitly asks to be durable. Two things are
asking, constantly:

1. **SCST advertises a volatile cache.** All 68 exported devices report
   `nv_cache = 0` under `/sys/kernel/scst_tgt/devices/*/nv_cache`, so the
   target honours `SYNCHRONIZE_CACHE` and FUA. Every guest `fsync` (etcd WAL,
   ext4 journal commits, the *arr SQLite databases) becomes a real ZFS sync
   write.
2. **Every NFS export is `sync`.** `exportfs -v` shows `sync` on
   `/mnt/tank/media`, `/mnt/tank/isos`, `/mnt/tank/k8s`, `/mnt/tank/terraform`
   and `/mnt/tank/files`.

So ADR 0008 step 3's premise, that `tank`'s workload is "async-dominated", is
measurably false **while VM disks are still on `tank`**. It becomes true only
after they leave, which is why step 3 cannot be justified on its own terms and
has to be justified as a consequence of step 2.

### What *is* true about the SLOG

It is absurdly oversized. Peak allocation observed across three samples was
77.5 MiB. At the current 3.70 MiB/s and a 5-second txg window, even a hundred
times the observed working set fits in well under 32 GiB. And wear is a
non-issue: 76.0 TB is about 3% of the SN850X 4 TB endurance rating, SMART
agrees at 2% used, and at the current burn rate (roughly 312 GiB/day) the
remaining endurance is on the order of 19 years. **The role is worth keeping.
Dedicating 3.64 TB to it is not.**

## Finding 2: the NVMe is 89 GiB too small to protect `mirror-3`

The obvious cheap mitigation for the `mirror-3` problem (both remaining
bad-batch drives in one vdev, so one vdev failing loses the pool) is to attach
the idle NVMe as a third mirror leg. One command, one resilver, reversible by
`zpool detach`, and `mirror-3` would then survive losing both bad drives.

**It is physically impossible.** `zpool attach` and `zpool replace` both
require the incoming device to be at least as large as the vdev's `asize`.

| vdev | `asize` (bytes) | Members |
|---|---|---|
| mirror-0 | 4,000,780,386,304 | WD Blue `24230KD00111`, WD Blue `24230KD00129` |
| mirror-1 | 4,000,780,386,304 | WD Blue `24230KD00005`, Inland `IB24AK0004S00013` |
| mirror-2 | **4,096,799,539,200** | Inland `IB24AJ0004S00183`, Inland `IB24AJ0004S00059` |
| mirror-3 | **4,096,799,539,200** | Inland `IB24AK0004S00064`, Inland `IB24AK0004S00004` |
| SN850X partition | 4,000,780,386,304 | `nvme0n1p1` / `nvme1n1p1` |

The SN850X is **96,019,152,896 bytes (89.4 GiB) smaller** than `mirror-3`'s
`asize`. The Inland drives are physically larger than both the WD Blues and
the NVMe, and `mirror-0`/`mirror-1` are capped at the WD Blue size.

So the NVMe fits `mirror-0` and `mirror-1` exactly, and cannot go into
`mirror-2` or `mirror-3` at all. **The one vdev where a third leg would
prevent pool loss is the one vdev the drive cannot join.** Attaching it to
`mirror-1` instead would protect the third bad drive
(`IB24AK0004S00013`), which is real but not the pool-losing case, at the cost
of consuming the drive permanently.

!!! note "A useful side result: the bad batch is identifiable by serial prefix"
    `smartctl -i` reports only `Inland SATA SSD` as the model for all five.
    The discriminator is the serial prefix: **`IB24AK` is the bad batch**
    (three drives: `S00013` in `mirror-1`, `S00064` and `S00004` in
    `mirror-3`) and **`IB24AJ` is the good batch** (`S00183` and `S00059`,
    both in `mirror-2`). This confirms ADR 0008's claim about `mirror-3`
    independently, by serial.

## Finding 3: capacity on `fast` is tighter than ADR 0008 implies

ADR 0008 says the hot data is "about 700 GB on 3.6 TB usable, **19% full**,
with room to grow several times over". That is true of *consumption* and
misleading about *commitment*.

| Measurement | Value |
|---|---|
| zvols under `tank/proxmox_zfs_vms` | 61 |
| **Provisioned (sum of `volsize`)** | **2785.1 GiB** |
| Used (sum of `used`) | 663.8 GiB |
| Referenced (sum of `referenced`) | 506.0 GiB |
| In snapshots | 157.8 GiB across **2233 snapshots** |
| Thin ratio | 4.2x |

2785.1 GiB provisioned against a 3.64 TiB mirror is **75% committed before a
single snapshot**, and ZFS write performance degrades past roughly 80%. On
`tank` this overcommit is harmless because 9.66 T is free. On `fast` it is
not.

`refquota=4T` is set on `tank/proxmox_zfs_vms` and **does not bound the
zvols**. Verified directly: every child zvol reports `AVAIL 9.53T`, the whole
pool's free space, not the 4.00 T the parent reports. `quota` is unset.
`zvol_enforce_quotas` is `1`, so a real `quota` *will* be honoured.

**`fast` must get a `quota`, not a `refquota`.** ADR 0008 does not mention
capacity bounding at all.

## Finding 4: `tank/k8s` cannot ride along with the zvols

ADR 0008 step 2 moves `proxmox_zfs_vms` (664 G) **and** `k8s` (45.9 G) to
`fast` in one step. The second half is a different and much harder change.

The Kubernetes PVCs are provisioned by `nfs-subdir-external-provisioner`
behind StorageClass `k8s-nfs`, and the dataset path is baked into every
PersistentVolume. Verified on `pvc-c1e41304-2466-49fa-a8d4-64fdbe5af80d`
(Loki's 50 Gi volume):

```yaml
spec:
  nfs:
    path: /mnt/tank/k8s/monitoring/storage-loki-0
    server: 192.168.1.15
```

`spec.nfs.path` is immutable on a PV. Moving the dataset to `/mnt/fast/k8s`
invalidates roughly twenty bound PVs, each of which would have to be deleted
and recreated with the new path and re-bound by `volumeName`, plus a Helm
values change for the provisioner's `NFS_PATH`. The failure mode is pods stuck
mounting, which is a different blast radius from a zvol move.

Two further points argue for leaving it alone: those PVs target
**`192.168.1.15`, the 1 G LAN interface**, not `10.10.12.2` on the 10 G
storage network, so NVMe backing is partly wasted behind the transport. And
`tank/k8s` is 45.9 G, so the capacity argument is negligible.

**`tank/k8s` stays on `tank`.** It is a separate decision for a separate ADR.

## Candidate uses, evaluated

| Option | Problem it solves | Cost | Blast radius | Survives one device loss | Verdict |
|---|---|---|---|---|---|
| **A.** `fast` mirror of both, VM zvols only | Takes 61 VM disks off a pool whose `mirror-3` pairs both bad drives | SLOG must be retired first; new PVE storage def; 2233 snapshots lost; needs a `quota` | All VMs in one 2-way mirror, same appliance | Yes | **Recommended, amended as G** |
| **B.** PBS datastore on the NAS | A bigger PBS datastore than 600 G | Re-creates a decision already made | NAS dies, VM disks *and* their backups die together | Yes (pool), no (appliance) | **Rejected** |
| **C.** Mirrored `special` vdev on `tank` | Accelerates the whole pool without splitting it | Adds a third failure domain to a pool that already has a bad vdev; not evacuable | Whole pool, and now one more way to lose it | Yes (if mirrored) | **Rejected, better reasons than ADR 0008 gave** |
| **D.** Keep `nvme0` as SLOG, repurpose `nvme1` alone | Preserves the measured 92us sync path | Leaves all 61 VM disks on the bad SATA pool | Unchanged, which is the problem | n/a | **Rejected as an end state** |
| **E.** Single-drive `fast`, no mirror | Can start today without touching the SLOG | Every VM disk on one non-redundant device | Total lab loss on one NVMe failure | **No** | **Rejected** |
| **F.** Do nothing | Nothing. Costs nothing. | `mirror-3` stays a single-vdev pool-loss risk with no alerting (#175) | Unchanged | n/a | **Rejected, on narrowed grounds** |
| **G.** Partition both: 32 GiB SLOG slice + `fast` mirror on the remainder | A, plus keeps the SLOG option alive | 64 GiB of 7.3 TiB raw, unused until needed | Same as A | Yes | **Recommended** |
| **H.** `nvme1` as a third leg of `mirror-3` | Would remove the pool-loss risk in one reversible command | n/a | n/a | n/a | **Impossible.** 89.4 GiB too small, [Finding 2](#finding-2-the-nvme-is-89-gib-too-small-to-protect-mirror-3) |
| **I.** Re-add a right-sized L2ARC | Would serve reads that miss ARC | ARC misses are 98.6% satisfied already and metadata misses are 0.008% of metadata reads | Low | Yes | **Rejected, settled by ADR 0008 and still true** |

### Notes on the rejections

**B, PBS on the NAS.** `tank/backups` (261 GiB of vzdump archives) was
deliberately destroyed in terraform-quasarlab#26 because backups must not live
on the pool they protect. Putting a PBS datastore on NVMe in a *different
pool* on the *same appliance* does not change the failure domain that
motivated that decision: the NAS is a single appliance and a SPOF, and it also
hosts the cluster QDevice and command-center1's cloud-init drive (#199). If
the appliance dies, the VM disks and their backups die in the same event. The
correct second copy is `pbs2`, which is already staged as vm120 on **pve2**
with a 600 G disk on `ssd_2` (verified in `qm config 120`), on different
hardware. This is the one option that actively re-litigates a settled
decision, and it should not be reopened on capacity grounds.

**C, `special` vdev.** ADR 0008 gave two reasons and one of them is wrong. It
said a `special` vdev "is not optional once written: losing it loses the pool,
so it inherits the blast radius". A *mirrored* `special` survives one device
loss exactly as the proposed `fast` mirror does, so that argument proves too
much. It also said 3.6 TB of `special` for a 5 TB pool is "the same mistake as
the current L2ARC in a new hat". **That analogy is wrong.** The L2ARC's harm
was `l2_hdr_size`, RAM overhead that scales with device size, which is why
3.4 TB of L2ARC ate 20.6 GiB of a 62 GiB ARC. A `special` vdev has no RAM tax
at all. Oversized `special` is merely wasteful, not harmful.

Reject it anyway, for two reasons ADR 0008 did not have:

- **The read benefit is near zero.** `demand_metadata_hits` is 1,281,629,175
  against `demand_metadata_misses` of 96,646: metadata is served from ARC
  **99.99%** of the time. Misses are overwhelmingly demand *data*
  (6,154,391). A metadata `special` vdev accelerates reads that are already
  free.
- **To get a real benefit you would need `special_small_blocks=16K`**, which,
  given `volblocksize`/`recordsize` of 16 K on `proxmox_zfs_vms`, would route
  essentially all VM data to the `special` vdev. At that point you have built
  `fast` inside `tank`, keeping the single-pool blast radius and gaining no
  ability to survive losing `tank`. That is the opposite of the goal.

**D, keep the SLOG and repurpose one drive.** This is the option that respects
Finding 1, and it is the cheapest. It is still wrong, because the two risks
are not comparable. The SLOG is worth roughly 1 ms per sync write. The
bad-batch pairing in `mirror-3` is worth the entire pool. Option G gets both.

**E, single-drive `fast`.** Tempting because `nvme1` is free today, so no SLOG
retirement is needed and the migration can start immediately. Rejected: it
puts every VM disk in the lab on one device with no redundancy for the
duration of the migration *and* the subsequent resilver. PBS with a passed
restore drill makes that survivable rather than reckless, but it trades a
bounded performance cost for an unbounded data-loss cost. Never trade
redundancy for latency.

**F, do nothing.** More defensible today than on 2026-09-20, and that should
be said plainly. `tank` is ONLINE with zero read, write and checksum errors,
"No known data errors", 34% full, 25% fragmented, zero `ata` link resets in 16
days of uptime, a 99.08% ARC hit rate and a SLOG absorbing sync writes at
92us. **There is no performance problem to fix.** PBS landing on 2026-10-06
also lowered the cost of the risk materialising. Rejected anyway, but note the
narrowed justification: the reason to act is `mirror-3`, not speed. Any
argument for this work that leans on latency is dishonest.

## Decision

**Adopt option G.** The end state is ADR 0008's: a mirror of the two SN850X
holding the VM zvols, taking every VM disk off the SATA pool. Three
amendments.

1. **Partition both drives up front:** a 32 GiB `p1` and the remainder as
   `p2`. Build `fast` as a mirror of `nvme0n1p2` + `nvme1n1p2`. Leave both
   `p1` slices **unused** until measurement says `tank` needs a log again.

    The reason is mechanical: you cannot carve a partition off a drive that is
    already a whole-disk vdev member without destroying the pool. Finding 1
    proves the SLOG role is real, so the option has to be preserved at
    creation time or not at all. The cost is 64 GiB of 7.3 TiB raw. The cost
    of skipping it is redoing the entire migration. ZFS on partitions is
    already how this box works: `boot-pool` is `nvme2n1p3` and today's SLOG is
    itself `nvme0n1p1`, not a whole disk.

2. **Set `quota` on `fast/proxmox_zfs_vms`**, not `refquota`. See
   [Finding 3](#finding-3-capacity-on-fast-is-tighter-than-adr-0008-implies).
   Suggested value 2.5 T, which bounds the 2785 GiB of thin commitment below
   the 80% line and still leaves room.

3. **Leave `tank/k8s` where it is.** See
   [Finding 4](#finding-4-tankk8s-cannot-ride-along-with-the-zvols).

**And drop ADR 0008 step 4, the `tank` rebalance.** Do not execute it. The
rebalance exists only to stop `mirror-3` pairing two bad drives, but the RMA
(#193) *replaces* those drives. Rebalancing now means three resilvers on bad
drives, then three more when the replacements land: six instead of three, on
drives whose failure trigger is still not established. Worse, a rebalance by
detach-and-attach runs a resilver on a vdev that is deliberately degraded,
which is precisely the scenario ADR 0008's own consequences section names as
"how a pool is lost". A straight `zpool replace` after the RMA keeps the old
drive in the vdev until the resilver completes, and never degrades anything.

**Step 4 was the right diagnosis with the wrong remedy. The remedy is the
RMA.**

### Ordering, and why

Full detail in the [runbook](../runbooks/nvme-tiering-migration.md). The
constraints:

| | Must happen | Before | Because |
|---|---|---|---|
| 0 | ZFS pool health alerting (#175) | **everything** | Three drives failed and Prometheus stayed silent. Resilvers and topology changes on an appliance with no alerting is how a recoverable event became the 4.5 hour wedge on 2026-09-20 |
| 1 | `pbs1` to `pbs2` sync job | any topology change | Cheapest risk reduction available, pure addition, no lab risk. Roles are already open in ansible-quasarlab#215 |
| 2 | RMA #193 and `zpool replace` | the rebalance (which is now cancelled) | Replacing the bad drives dissolves the `mirror-3` problem without a rebalance |
| 3 | PVE 9 upgrade and `freenas-proxmox` 3.0.0 | creating the `fast` storage definition | `fast` needs a new ZFS-over-iSCSI storage definition interpreted by a plugin that is apt-held at 2.4.0-1 (confirmed `hi` in `apt-mark showhold`) and known to fail the 8 to 9 dist-upgrade through its dpkg trigger. Defining `fast` against 2.4.0 means validating the migration twice. Do the plugin work once, then define `fast` once |
| 4 | SLOG removal from `tank` | creating `fast` | The coupling. Both drives are needed for the mirror |

Note that steps 2 and 3 have **unknown timing** and the `mirror-3` risk is
live in the meantime. There is no clean interim mitigation available:
[Finding 2](#finding-2-the-nvme-is-89-gib-too-small-to-protect-mirror-3) rules
out the obvious one on physics. What is available is steps 0 and 1, which
convert "the pool might die silently" into "the pool dies loudly and we
restore from two independent copies". Do those now, this week, and treat them
as the mitigation.

### The exposure this creates, stated plainly

Between `zpool remove tank <log>` and the last VM leaving `tank`, every guest
`fsync` on `tank` lands on the SATA mirrors at 346us to 593us `disk_wait`
instead of 88us, and the ZIL stops being a separate device so sync writes
contend with data writes. Roughly **4x to 6x worse per sync write**. The thing
that cares is etcd WAL fsync, whose baseline was already about 74 ms, and 2 of
3 etcd members are on pve2.

The alternative ordering (option E, migrate to a single-device `fast` first so
the SLOG keeps working throughout) removes that regression by removing
redundancy instead. **Take the latency hit.** Mitigate it by migrating the
sync-sensitive VMs first: etcd members, then anything with a database. The
regression is bounded, measurable, and undone at any moment by
`zpool add tank log`.

!!! danger "`zpool remove` of a log vdev is not the operation that wedged the lab"
    On 2026-09-20 a `zpool remove` of the **cache** vdev, pitched as "instant
    and reversible", aggravated an overloaded NAS into a 4.5 hour wedge and a
    cold power cycle. That removal had 20.6 GiB of `l2_hdr_size` to tear out
    of ARC. Removing a **log** vdev has no ARC work to do at all: it quiesces
    the ZIL, flushes outstanding sync writes to the data vdevs, and detaches.
    Mechanically it is a much smaller operation.

    That is not permission to run it casually. It is still a pool topology
    change on an appliance with a demonstrated capacity to wedge under load.
    **"Reversible" and "safe to run right now" are different claims**, and
    ADR 0008 collapsed them. Run it in a window, with alerting live
    (step 0) and PBS verified (step 1), and with `zpool iostat` open.

## The SLOG question, answered

**Yes, the SLOG is earning something today, and it is quantifiable.** 49 write
IOPS and 3.70 MiB/s sustained over 16 days of uptime, 95 IOPS in a live
sample, acknowledged at 92us against 1 ms to 2 ms `total_wait` on the SATA
mirrors. 42% of the bytes reaching `tank`'s data vdevs pass through the ZIL
first. Full detail in [Finding 1](#finding-1-the-slog-is-earning-its-place).

What **cannot** be quantified read-only is the end-to-end user-visible benefit:
how much of that 1 ms to 2 ms difference a guest actually observes, after SCST
queueing, the 10 G network and QEMU's libiscsi path. Measuring that needs an
A/B with the SLOG removed, which is a mutating change and is not done here.
**Marked unverified.** The device-level latency gap is real and measured; the
guest-level consequence is inferred.

The honest summary: the role is worth keeping, a 3.64 TB device for it is not,
and after the VM disks leave `tank` the sync load drops to NFS-`sync` traffic
on `media`, `isos` and `k8s` only, at which point a log device may genuinely
stop earning its place. That is why recommendation G **keeps the option and
defers the judgement** rather than retiring the role on today's evidence or
ADR 0008's.

## The ARC question, answered

**Yes, cap it. `zfs_arc_max = 51539607552` (48 GiB).** But this is a guardrail,
not a fix, and the case is weaker than it looks.

Measured today:

| Metric | Value | |
|---|---|---|
| `zfs_arc_max` | `0` | TrueNAS default, not tuned |
| `c_max` | 66,086,522,880 | **61.55 GiB of 62 GiB host RAM** |
| `c` (target) | 51,840,468,464 | 48.28 GiB |
| `size` (actual) | 50,307,580,376 | 46.86 GiB |
| `c_min` | 2,098,758,272 | 1.95 GiB |
| `memory_available_bytes` | 6,345,863,552 | 5.91 GiB |
| `arc_no_grow` | `0` | not currently under pressure |
| `evict_skip` | 8,161 | negligible |
| Lifetime hit rate | 1,718,164,113 / 15,882,958 | **99.08%** |

The argument for capping: `c_max` permits ARC to take 61.55 of 62 GiB,
leaving under half a gigabyte for the OS, SCST with its 68 exported devices,
`nfsd`, `middlewared`, the `ix-apps` Docker stack and the cluster QDevice VM
that runs on this appliance. That only works because ARC shrinks under
pressure, and ARC shrinking is reactive and lags the pressure that triggers
it.

**48 GiB costs nothing measured.** ARC has self-tuned to a 48.28 GiB target
and a 46.86 GiB actual size, so the cap pins it where it already sits and
removes only the headroom to balloon under some future workload. The 99.08%
hit rate is the hit rate *at this size*.

The argument against, which should be recorded rather than buried: **the
specific RAM pressure that contributed to 2026-09-20 was `l2_hdr_size` at
20.6 GiB, and it is already gone.** `l2_hdr_size` reads `0` today. So this
recommendation does not fix an observed problem. It is a cheap ceiling against
a repeat by some other route.

**Confidence: high that 48 GiB is harmless, medium that it is necessary.** Set
it, in Ansible rather than by hand, and do not credit it with anything.

## What in ADR 0008 is now stale or wrong

| ADR 0008 | Status |
|---|---|
| Step 1, destroy the L2ARC | **Executed 2026-09-20.** Correct call, correct reasoning. `l2_hdr_size` is now 0 |
| Step 1's "**Instant, reversible, no migration**" | **Wrong, and it cost 4.5 hours.** The removal wedged an already-loaded NAS and forced a cold power cycle. The claim was true of the command's semantics and false of its behaviour under load. ADR 0008 also calls it "the highest value per unit of risk of anything here", which was right about the value and wrong about the risk |
| Step 1's predicted "returns 20.6 GiB to ARC" | **Not delivered as predicted.** ARC `size` is 46.86 GiB today against the 48.2 GiB recorded on 2026-09-20, so it did not grow by 20 GiB. ARC's target had already absorbed the headroom. The hit rate also moved from 99.27% to 99.08%, so the predicted improvement did not appear either. Neither is a problem; both are predictions recorded as outcomes-in-waiting that did not come true |
| Step 2, create `fast` and move hot data | **Right end state, three amendments.** Capacity bounding missing (Finding 3), `tank/k8s` cannot ride along (Finding 4), and the SLOG coupling is unstated |
| Step 2's "about 700 GB on 3.6 TB usable, 19% full" | **Misleading.** True of consumption, but 2785.1 GiB is provisioned across 61 thin zvols, which is 75% of the mirror before snapshots |
| Step 3, retire the SLOG | **Premise refuted.** "Async-dominated" is false while VM disks are on `tank`. See Finding 1 |
| Step 3's "11.5 MiB used of 3.64 TB" as evidence of no utility | **Category error.** Measures allocation, not utility. Near-zero allocation is correct SLOG behaviour |
| Step 4, rebalance `tank` mirrors | **Should be cancelled, not executed.** The RMA supersedes it. See [Decision](#decision) |
| `special` vdev rejection, reason 1 ("losing it loses the pool") | **Weak.** A mirrored `special` survives one device loss, same as the proposed `fast` |
| `special` vdev rejection, reason 2 ("the L2ARC mistake in a new hat") | **Wrong analogy.** The L2ARC's harm was RAM headers scaling with device size. A `special` vdev has no RAM tax |
| Dataset table: `backups` 158 G | **Gone.** `tank/backups` destroyed in terraform-quasarlab#26 |
| Dataset table: `media` 4.21 T, `proxmox_zfs_vms` 656 G, `k8s` 44 G | Now 4.27 T, 664 G, 45.9 G. Drift only |
| "three Inland SSDs pending RMA, two of them paired in the same mirror" | **Confirmed independently by serial.** `IB24AK` prefix, `S00064` and `S00004` both in `mirror-3` |

One meta-point. ADR 0008's own Validation section says "Record the results
here rather than assuming them" and warns about "a plan recorded as though it
were an outcome". Step 1 was then executed and its three predicted outcomes
(20.6 GiB returned to ARC, hit rate rises, done in seconds with no disruption)
were none of them recorded, and none of them happened as written. **The
warning was correct and was not followed.** That is the strongest argument for
keeping the validation table in this ADR short, specific and actually filled
in.

## Consequences

- **`fast` becomes a single failure domain for every VM disk.** Two-way
  mirror, survives one device. Not a regression: `tank` is that failure domain
  today and contains three drives that drop off the SATA bus. But it is also
  not an improvement in cardinality, only in component quality (2% worn
  SN850X, 100% spare, zero media errors).
- **The NAS remains a SPOF and this ADR does not touch that.** Same appliance,
  same QDevice-on-the-NAS problem, same command-center1 cloud-init dependency
  (#199). Splitting one pool into two reduces the *pool* blast radius, not the
  *appliance* blast radius. Anyone reading this as "the lab now survives the
  NAS" has misread it.
- **2233 snapshots holding 157.8 GiB are lost** in a PVE `Move Disk`, which
  copies current state only. This is acceptable **now** in a way it was not
  when ADR 0008 was written: before 2026-10-06 those hourly snapshots were the
  only recovery mechanism for VM disks, and now PBS is, hourly, with 24/7/4/3
  retention and a restore verified against `nginx2` (hostname, kernel, root
  filesystem, package count and `/etc` md5 all identical). **PBS landing is
  what makes this step affordable.**
- **`zfs send -R` is not an alternative for preserving them.** The
  `freenas-proxmox` plugin creates the SCST extents through the TrueNAS API, so
  hand-replicated zvols would arrive with no iSCSI target mapping. Use
  `Move Disk` and accept the snapshot loss.
- **A degraded sync-write path on `tank` for the duration of the migration.**
  Bounded, measurable, reversible. See
  [the exposure](#the-exposure-this-creates-stated-plainly).
- **Two pools, two capacity numbers, a second ZFS-over-iSCSI storage
  definition** against a third-party plugin on a distribution that is already
  EOL. Accepted, and the reason the PVE 9 upgrade is sequenced first.
- **`media` stays on SSD** and `tank/k8s` stays on SATA. Both unchanged from
  ADR 0008's position on `media`, and a deliberate reversal on `k8s`.
- **No live migration anywhere in this.** pve is Intel and pve2 is AMD, so
  every VM move is shutdown-and-start. `Move Disk` itself is online, but any
  step that needs a VM on the other node is not.

## Validation

Nothing here is validated. These are measurements and a plan. Record results
in this section as they land, and leave the unfilled rows visibly unfilled.

| # | Claim to test | How | Result |
|---|---|---|---|
| 1 | ZFS pool health alerting actually fires | Offline a drive in a scratch pool, or `zinject`, and confirm Discord | *not done* |
| 2 | `pbs2` holds a complete independent copy | Restore one VM from `pbs2`, not `pbs1`, to a scratch VMID and diff as the `nginx2` drill did | *not done* |
| 3 | Removing the SLOG degrades sync writes by the predicted 4x to 6x | `zpool iostat -v -l tank 60` before and after; record etcd WAL fsync p99 from Prometheus | *not done* |
| 4 | `fast` outperforms `tank` for VM workloads | etcd WAL fsync p99 and Uptime Kuma NFS probe latency, against today's baseline | *not done* |
| 5 | The `quota` on `fast/proxmox_zfs_vms` is **not** vacuous | Read `AVAIL` on a **child zvol**, not the parent. It must report the quota, not the pool free space | *not done* |
| 6 | Capping ARC at 48 GiB does not move the hit rate | `arcstats` hits/misses over a week either side | *not done* |
| 7 | `tank` no longer needs a log device | After the VMs leave, `zpool iostat -v tank` log-vdev IOPS should collapse toward NFS-`sync` traffic only | *not done* |

Row 5 is the one most likely to be skipped and is the one that has already
caught this lab out once: `refquota` on the parent reads as a ceiling in the
Proxmox UI and bounds nothing.

## Confidence, and what could not be verified

**High confidence**, measured directly and cited above: the SLOG's IOPS,
bandwidth, latency and lifetime figures; the 89.4 GiB size shortfall against
`mirror-3`; the drive serial to vdev mapping and the `IB24AK`/`IB24AJ` batch
split; `refquota` being vacuous for child zvols; the 2785.1 GiB thin
commitment; SCST `nv_cache=0`; every NFS export being `sync`; the ARC figures;
`nvme1n1p1` being a member of no pool; the PV path immutability problem.

**Inferred, not measured:**

- **The guest-visible benefit of the SLOG.** Device-level latency is measured.
  How much of it a VM sees through SCST, 10 GbE and libiscsi is not. Requires
  a mutating A/B.
- **Resilver duration for the RMA replacements.** The only data point is
  "13.8 G in 11:05 on 2026-09-16", which is too small to extrapolate from.
- **That PVE 9 plus plugin 3.0.0 will accept a second ZFS-over-iSCSI storage
  definition cleanly.** Taken from upstream ADR-009 and not tested here.
- **Whether `tank` still wants a log device after the VMs leave.** That is
  validation row 7 and the reason the 32 GiB slices exist.

**One premise in the brief that needs a footnote:** the uptime was given as "2
weeks 2 days (since 2026-09-20 15:37:46)". `uptime` on the NAS at 17:39 on
2026-10-07 reports `16 days, 2:01`, which puts the boot at roughly
**2026-09-21 15:38**, one day later than stated. The 16-day figure is right
and nothing in this ADR depends on the date, but the 2026-09-20 incident and
the NAS boot are not the same moment.

## Related

- [ADR 0008](0008-storage-tiering.md), the prior this ADR amends.
- [Runbook: NVMe tiering migration](../runbooks/nvme-tiering-migration.md), the sequenced plan.
- [Runbook: tiered NAS storage migration](../runbooks/nas-storage-tiering-migration.md), ADR 0008's runbook. Stages 3 and 4 there are superseded by this ADR.
- [Runbook: NAS reboot with zero VM downtime](../runbooks/nas-reboot-evacuate-vm-disks.md), the same live `Move Disk` technique.
- [2026-09-20 UGREEN IT8613 watchdog](../incidents/2026-09-20-ugreen-it8613-watchdog.md).
- [2026-09-19 NFSv4 callback deadlock](../incidents/2026-09-19-nfsv4-callback-deadlock.md), which made the single-pool coupling visible.
- ansible-quasarlab#193 (RMA), #174 (`mirror-3` pairing), #175 (no ZFS health alerting), #199 (NAS SPOF), #215 (PBS roles).
- terraform-quasarlab#26, which destroyed `tank/backups`.
- [Storage architecture](../architecture/overview.md#storage).
