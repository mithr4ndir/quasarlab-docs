# NVMe tiering migration

The execution plan for [ADR 0009](../decisions/0009-nvme-repurposing.md):
retire `tank`'s oversized SLOG, build `fast` as a mirror of the two SN850X,
and move 61 VM zvols onto it. This supersedes stages 3 and 4 of the
[ADR 0008 runbook](nas-storage-tiering-migration.md).

!!! warning "Read this first: the two drives are coupled"
    `nvme0n1p1` **is** `tank`'s log vdev today. `nvme1n1p1` is free. They
    cannot both join a mirror until the SLOG is retired, so the SLOG removal
    is a **prerequisite** of building `fast`, not a follow-up. ADR 0008 has
    these in the opposite order and that order is not executable.

## Ordering constraints

Nothing below is optional and nothing below reorders safely.

```
Step 0  ZFS health alerting (#175)        ─┐
Step 1  pbs1 -> pbs2 sync job (#215)       ├─ do these now, this week
                                           │  they are the interim mitigation
Step 2  RMA #193, zpool replace x3        ─┤  blocked on vendor
Step 3  PVE 9 + freenas-proxmox 3.0.0     ─┤  blocked on step 0 and 1
Step 4  remove tank's SLOG                ─┤  irreversible-ish window opens
Step 5  partition both drives, build fast ─┤
Step 6  migrate 61 zvols, set quota       ─┤
Step 7  decide on the 32 GiB SLOG slice   ─┘  blocked on step 6 measurement

CANCELLED: ADR 0008 step 4, the tank mirror rebalance. Step 2 replaces it.
NOT IN SCOPE: moving tank/k8s. Immutable PV paths. Separate change.
```

| Step | Blocks on | Reversible | Lab exposed |
|---|---|---|---|
| 0 Alerting | nothing | n/a, pure addition | no |
| 1 PBS sync | nothing | n/a, pure addition | no |
| 2 RMA replace | vendor, step 0 | per-drive, resilver | **yes, resilver load on bad drives** |
| 3 PVE 9 | steps 0, 1 | snapshot/rollback per node | **yes, node reboots** |
| 4 Remove SLOG | step 3 | `zpool add tank log` | **yes, sync latency 4-6x worse until step 6 completes** |
| 5 Build `fast` | step 4 | `zpool destroy fast` | no |
| 6 Migrate zvols | step 5 | `Move Disk` back, per VM | **yes, 2233 snapshots lost irreversibly** |
| 7 SLOG slice | step 6 | `zpool remove` | no |

**The only irreversible thing in this plan is the snapshot loss in step 6.**
Everything else can be undone, at a cost. That does not make the rest safe to
run casually: see the warning on step 4.

## Before anything: record the baseline

"It feels faster" is not a result. Capture this and keep the output.

```bash
# NAS
ssh truenas_admin@192.168.1.15 '
  sudo zpool status tank
  sudo zpool list -v -p tank
  sudo zpool iostat -v -l tank 60 2
  sudo zfs list -r -t volume -H -p -o used tank/proxmox_zfs_vms |
    awk "{s+=\$1} END {printf \"%.1f GiB used\n\", s/1073741824}"
  sudo zfs list -r -t snapshot -H -o name tank/proxmox_zfs_vms | wc -l
  awk "/^(size|c|c_max|hits|misses|l2_hdr_size)/ {print \$1, \$3}" \
    /proc/spl/kstat/zfs/arcstats
  sudo dmesg | grep -ciE "COMRESET|hard resetting|failed command"
'
```

Baseline measured 2026-10-07 at 17:39, 16 days 2 hours of uptime:

| Metric | Value |
|---|---|
| `tank` | 14.7 T size, 5.03 T alloc, 9.66 T free, 25% frag, 34% cap, ONLINE, 0 errors |
| Log vdev write | **49 IOPS, 3.70 MiB/s, 88us `disk_wait`, 92us `total_wait`** |
| Data vdev write | 211 IOPS, 8.78 MiB/s, 346-593us `disk_wait`, 1-2ms `total_wait` |
| ARC | `size` 46.86 GiB, `c` 48.28 GiB, `c_max` 61.55 GiB, hit rate 99.08% |
| `l2_hdr_size` | 0 (L2ARC removed 2026-09-20) |
| zvols | 61, **2785.1 GiB provisioned**, 663.8 GiB used, 506.0 GiB referenced |
| Snapshots | **2233**, holding 157.8 GiB |
| `ata` resets since boot | **0** |
| etcd WAL fsync | about 74 ms (from ADR 0008, **re-measure, this is stale**) |

## Step 0: ZFS pool health alerting

**Blocking. Do not skip. Do not reorder.**

Three drives dropped off the SATA bus and Prometheus stayed silent
(ansible-quasarlab#175). Every step from 2 onward puts load on drives that
have failed before. Running those with no alerting is how 2026-09-20 turned a
recoverable event into a 4.5 hour wedge and a cold power cycle.

What must exist before step 2:

- A metric carrying `zpool status` health per pool, scraped into Prometheus.
- An alert on anything other than `ONLINE`, and separately on non-zero
  read/write/checksum counters.
- An alert on resilver in progress, so a resilver nobody started is visible.
- A Discord route that has been **tested**, not assumed.

!!! danger "Verify the series exists before trusting the rule"
    This lab has shipped vacuous alert rules before: `node_exporter`
    `--collector.systemd.unit-include` allowlists made rules for unlisted
    units silently unfirable and hid three real outages. Query the metric in
    Prometheus and confirm it returns data **for `tank` specifically** before
    believing the rule covers anything.

**Rollback point:** none needed, pure addition.

## Step 1: `pbs1` to `pbs2` sync job

**Blocking. Cheapest risk reduction available.**

`pbs1` (vm103 on pve) runs PBS 3.4.9 with datastore `lab` on a 600 G disk on
pve's local `SSD2`, hourly, all VMs, excluding vm9000, `--verify-new`,
retention 24/7/4/3. A restore drill passed: `nginx2` restored to a scratch
VMID with hostname, kernel, root filesystem, package count and `/etc` md5 all
identical to the live host.

`pbs2` (vm120 on pve2) is **staged but empty**: `qm config 120` confirms
`scsi0: ssd_1:vm-120-disk-0,size=32G` and `scsi1: ssd_2:vm-120-disk-0,size=600G`,
but PBS is not installed and the sync job does not exist.

1. Merge ansible-quasarlab#215 and install PBS on `pbs2`.
2. Create a remote + sync job pulling datastore `lab` from `pbs1`.
3. Run a verify job on `pbs2`'s datastore.
4. **Restore one VM from `pbs2`, not from `pbs1`**, to a scratch VMID, and
   diff it the way the `nginx2` drill did.

Step 4 is the point of the exercise. A sync job that has never been restored
from is a sync job you are guessing about.

!!! note "Why this is on the critical path and not a nice-to-have"
    Step 6 irreversibly discards 2233 snapshots. PBS is what replaces them as
    the recovery mechanism for VM disks. One copy of the replacement, on a
    600 G local disk on one node, is thin cover for discarding the other
    mechanism entirely.

**Rollback point:** none needed, pure addition.

## Step 2: RMA #193, then `zpool replace`

**Blocked on the vendor. Blocked on step 0.**

Three drives are RMA-pending, identifiable by serial prefix. Verified mapping
as of 2026-10-07:

| vdev | Member | Serial | Batch |
|---|---|---|---|
| mirror-0 | `/dev/sdg1` | WD Blue `24230KD00111` | good |
| mirror-0 | `/dev/sdh1` | WD Blue `24230KD00129` | good |
| mirror-1 | `/dev/sdf1` | WD Blue `24230KD00005` | good |
| mirror-1 | `/dev/sdc1` | Inland `IB24AK0004S00013` | **bad** |
| mirror-2 | `/dev/sda1` | Inland `IB24AJ0004S00183` | good |
| mirror-2 | `/dev/sde1` | Inland `IB24AJ0004S00059` | good |
| mirror-3 | `/dev/sdd1` | Inland `IB24AK0004S00064` | **bad** |
| mirror-3 | `/dev/sdb1` | Inland `IB24AK0004S00004` | **bad** |

`smartctl -i` reports only `Inland SATA SSD` for all five, so **the serial
prefix is the discriminator**: `IB24AK` is the bad batch, `IB24AJ` is not.

!!! danger "Identify the physical tray by serial, never by `sd` letter"
    `sd` letters are not stable across boots. On the DXP8800 the LED `diskN`
    maps to `ataN`, which maps to the Nth tray left to right, and **not** to
    `sd` letters. Confirm which tray someone actually pulled from `ata` link
    down events in `dmesg`, not from the letter.

Procedure, **one drive at a time**:

1. Confirm `zpool status tank` is ONLINE with zero errors. If it is not, stop.
2. `zpool replace tank <bad-partuuid> <new-partuuid>`.
3. Watch the resilver to completion. Watch `dmesg` for `COMRESET` and
   `hard resetting link` throughout.
4. Confirm ONLINE and zero errors before touching the next drive.

Do `mirror-3` first (both members bad), then `mirror-1`.

!!! note "`replace`, not detach-and-attach"
    `zpool replace` keeps the old drive in the vdev until the resilver
    completes, so the vdev never runs degraded. A rebalance by detach then
    attach runs the resilver on a deliberately degraded vdev, which is
    precisely the scenario ADR 0008's own consequences section calls "how a
    pool is lost". This is why **ADR 0008 step 4 is cancelled**: the RMA
    dissolves the `mirror-3` pairing without ever degrading a vdev, and
    rebalancing first would mean six resilvers on bad drives instead of three.

**Rollback point:** a `zpool replace` can be cancelled mid-resilver with
`zpool detach` of the incoming device. The vdev returns to its prior
membership.

**Lab exposed:** yes. Resilver reads put sustained load on drives whose
failure trigger is **not established**, so "wait for a quiet moment" is not a
mitigation. One vdev at a time, alerting live, PBS verified.

## Step 3: PVE 9 and `freenas-proxmox` 3.0.0

**Blocked on steps 0 and 1. Blocks step 5.**

Both nodes run `pve-manager/8.4.21`, EOL since 2026-08-31. The ZFS-over-iSCSI
plugin is held:

```
$ apt-mark showhold
freenas-proxmox
$ dpkg -l freenas-proxmox
hi  freenas-proxmox  2.4.0-1  all
```

2.4.0-1 **fails** the PVE 8 to 9 dist-upgrade through its dpkg trigger, and per
upstream ADR-009 only plugin v3.0.0 loads on PVE 8.4 (`api()` 11 against
APIVER 11).

**Why this is sequenced before `fast` and not after.** `fast` needs a second
`zfs:` storage definition, interpreted by this plugin. Defining it against
2.4.0 means validating the whole migration against a plugin that must then be
replaced, and revalidating afterwards. Do the plugin work once, then define
`fast` once.

Order within the step:

1. Upgrade the plugin to 3.0.0 on PVE 8.4 first, while the existing
   `truenas-iscsi` definition is the only one in play. Confirm VMs still
   start, stop and snapshot.
2. Then dist-upgrade to PVE 9, one node at a time.

!!! warning "etcd and the node order"
    2 of 3 etcd members (`k8cluster2`, `k8cluster3`) are both on **pve2**, and
    their disks are node-local, so they are **not clean live migrations**.
    Rebooting pve2 kills the Kubernetes control plane quorum. Upgrade **pve
    first**, confirm the cluster is healthy, then pve2.

    Live migration between the nodes is impossible regardless (Intel pve
    against AMD pve2), so every VM relocation here is shutdown-and-start.

**Rollback point:** per node, a pre-upgrade snapshot or a known-good kernel
entry. Keep the old plugin `.deb` on disk.

**Lab exposed:** yes, node reboots, control plane quorum on the pve2 reboot.

## Step 4: remove `tank`'s SLOG

**Blocked on step 3. This opens the exposure window.**

```bash
# The device: PARTUUID a1bfbe13-bfc0-4e7c-92e4-c28c963225e6 = nvme0n1p1
sudo zpool remove tank a1bfbe13-bfc0-4e7c-92e4-c28c963225e6
```

!!! danger "This is a pool topology change on an appliance that has wedged"
    On 2026-09-20 a `zpool remove` of the **cache** vdev, pitched in ADR 0008
    as "instant, reversible, no migration", aggravated an already-overloaded
    NAS into a 4.5 hour wedge and forced a cold power cycle.

    Removing a **log** vdev is mechanically a much smaller operation: that
    removal had 20.6 GiB of `l2_hdr_size` to tear out of ARC, and a log
    removal has no ARC work at all. It quiesces the ZIL, flushes outstanding
    sync writes to the data vdevs, and detaches.

    **That is not permission to run it casually.** "Reversible" and "safe to
    run right now" are different claims and ADR 0008 collapsed them. Run it
    in a window, at low load, with alerting live, PBS verified, and
    `zpool iostat -v -l tank 5` open in another pane.

    Never run a pool topology change mid-incident.

### The exposure this opens

From this moment until the last VM leaves `tank` in step 6, every guest
`fsync` on `tank` lands on the SATA mirrors. Measured difference:

| | With SLOG | Without (predicted) |
|---|---|---|
| Sync write `disk_wait` | 88us | 346-593us |
| Sync write `total_wait` | 92us | 1-2ms |

Roughly **4x to 6x worse per sync write**, and sync writes now contend with
data writes on the same devices. 42% of the bytes reaching `tank`'s data
vdevs currently pass through the ZIL first, so this is not a corner case.

The thing that cares most is **etcd WAL fsync**, baseline about 74 ms, with 2
of 3 members on pve2. Re-measure it immediately after the removal and again
after step 6.

!!! note "Why take this hit rather than avoid it"
    The alternative ordering is to build `fast` as a single-device pool on the
    already-free `nvme1n1`, migrate everything, then remove the SLOG and
    attach `nvme0n1p2` to make it a mirror. That keeps the 92us sync path for
    the whole migration.

    It also puts **every VM disk in the lab on one non-redundant device** for
    the duration of the migration plus the subsequent resilver. That trades a
    bounded, measurable, reversible performance cost for an unbounded
    data-loss cost. Take the latency hit.

**Rollback point:** `zpool add tank log a1bfbe13-...` restores it in seconds,
at any time before step 5 repartitions the drive. **After step 5 the partition
is gone and this rollback no longer exists**, which is why step 5 creates the
32 GiB replacement slices.

## Step 5: partition both drives and build `fast`

**Blocked on step 4.**

Partition **both** SN850X identically: a 32 GiB `p1` and the remainder as
`p2`. Build the mirror on the `p2` pair. **Leave both `p1` slices unused.**

```
nvme0n1  ->  p1  32 GiB   (unused, reserved for a log vdev)
             p2  ~3.61 TiB -> fast mirror leg
nvme1n1  ->  p1  32 GiB   (unused, reserved)
             p2  ~3.61 TiB -> fast mirror leg
```

!!! warning "This decision cannot be deferred"
    You cannot carve a partition off a drive that is already a whole-disk vdev
    member without destroying the pool. The SLOG role is
    [measurably real](../decisions/0009-nvme-repurposing.md#finding-1-the-slog-is-earning-its-place),
    so the option has to be preserved now or not at all. The cost is 64 GiB
    of 7.3 TiB raw. The cost of skipping it is redoing this entire migration.

    ZFS on partitions is already how this box works: `boot-pool` is
    `nvme2n1p3`, and today's SLOG is itself `nvme0n1p1`, not a whole disk.

Then create the pool with `ashift=12` to match the existing vdevs (verified:
every `tank` vdev is `ashift: 12`), and a `proxmox_zfs_vms` child with the
same properties the current one has: `volblocksize`/`recordsize` 16 K,
`compression=lz4`, `sync=standard`, `logbias=latency`.

### Set a `quota`, not a `refquota`

```
quota on fast/proxmox_zfs_vms = 2.5T
```

!!! danger "`refquota` on the parent bounds nothing"
    `tank/proxmox_zfs_vms` has `refquota=4T` and it is **vacuous**. Verified:
    the parent reports `AVAIL 4.00T` while **every child zvol reports
    `AVAIL 9.53T`**, the whole pool's free space. `refquota` excludes
    descendants. Proxmox displays it as the storage ceiling anyway.

    61 zvols commit **2785.1 GiB** thin against 663.8 GiB actually used. On a
    3.64 TiB mirror that is 75% committed before a single snapshot, and ZFS
    degrades past roughly 80%. On `tank` the overcommit was harmless because
    9.66 T was free. On `fast` it is not.

    `zvol_enforce_quotas` is `1`, so a real `quota` **will** be honoured.
    **Verify it by reading `AVAIL` on a child zvol, not on the parent.**

**Rollback point:** `zpool destroy fast` while it is empty. The `p1` slices
mean step 4's rollback (`zpool add tank log <p1>`) is available again from
here on.

## Step 6: migrate the zvols

**Blocked on step 5. Contains the only irreversible action in this plan.**

Add the `fast` storage definition in `/etc/pve/storage.cfg`, mirroring the
existing one but pointing at the new pool. For reference, the current
definition:

```
zfs: truenas-iscsi
	blocksize 16k
	iscsiprovider freenas
	pool tank/proxmox_zfs_vms
	portal 10.10.12.2
	target iqn.2005-10.org.freenas.ctl:proxmox-cluster
	content images
	sparse 1
```

Then move disks with the PVE **`Move Disk`** action, one VM at a time. This is
the same technique that rebooted the NAS with zero VM downtime on 2026-09-19;
the [existing runbook](nas-reboot-evacuate-vm-disks.md) covers the throttle
and the etcd cautions and they apply unchanged.

### Migration order

Sync-sensitive first, so the workloads that suffer most from step 4's
regression spend the least time on the SLOG-less pool:

1. **etcd members.** `k8cluster2` and `k8cluster3` are both on pve2. One at a
   time, confirming quorum between each.
2. The Postgres VM, and anything else with a database.
3. `command-center1`, noting that its cloud-init drive is on the NAS (#199).
4. Everything else.
5. **The QDevice VM last**, or plan around it: it provides the third corosync
   vote and it runs on the NAS.

### The irreversible part

!!! danger "`Move Disk` discards snapshots. 2233 of them, holding 157.8 GiB."
    `Move Disk` copies current state only. PVE-level snapshots do not survive
    it and neither do the hourly TrueNAS periodic snapshots.

    **This is acceptable now and was not when ADR 0008 was written.** Before
    2026-10-06 those snapshots were the only recovery mechanism for VM disks.
    PBS now is: hourly, all VMs, `--verify-new`, 24/7/4/3, with a restore
    verified against `nginx2`. **Step 1 existing is what makes this step
    affordable.** If step 1 is not done, do not run step 6.

    Confirm a current PBS backup for each VM immediately before moving it.

!!! note "`zfs send -R` is not an alternative"
    It would preserve all 2233 snapshots, but the `freenas-proxmox` plugin
    creates the SCST extents through the TrueNAS API. Hand-replicated zvols
    arrive with **no iSCSI target mapping**, so PVE cannot attach them. Use
    `Move Disk` and accept the loss.

**Rollback point:** per VM, `Move Disk` back to `truenas-iscsi`. The snapshots
are already gone by then and do not come back.

### Not in scope: `tank/k8s`

ADR 0008 step 2 moves `tank/k8s` (45.9 G) alongside the zvols. **Do not.**

Every Kubernetes PV has the dataset path baked into an immutable field.
Verified on Loki's 50 Gi volume:

```yaml
spec:
  nfs:
    path: /mnt/tank/k8s/monitoring/storage-loki-0
    server: 192.168.1.15
```

`spec.nfs.path` is immutable on a PV. Moving the dataset invalidates roughly
twenty bound PVs, each needing deletion and recreation with `volumeName`
re-binding, plus a Helm values change for `nfs-subdir-external-provisioner`'s
`NFS_PATH`. Different work, different failure mode (pods stuck mounting), and
it should not ride along with a zvol migration. Those PVs also target
`192.168.1.15`, the **1 G LAN** interface, not `10.10.12.2` on the 10 G
storage network, so NVMe backing would be partly wasted behind the transport.

## Step 7: decide on the 32 GiB SLOG slice

**Blocked on step 6, and on measurement.**

With the VM disks gone, `tank`'s sync load should collapse to NFS-`sync`
traffic on `media`, `isos`, `k8s`, `terraform` and `files` only. Measure it:

```bash
sudo zpool iostat -v -l tank 60 5
```

- **If log-vdev-equivalent sync traffic is negligible**, leave the `p1` slices
  unused. `tank` genuinely does not want a log device any more, and *that* is
  when ADR 0008 step 3's conclusion becomes correct, for reasons it did not
  give.
- **If NFS write latency regressed**, `zpool add tank log <nvme0n1p1>`.

!!! note "A log slice on a `fast` mirror member is acceptable"
    If `nvme0n1` then fails, `fast` goes degraded (survivable, it is a mirror)
    and `tank` loses its log (survivable, the pool imports and only in-flight
    sync writes from a simultaneous crash are at risk). The two pools share a
    physical device, which is worth knowing, but neither loses data from it.

    A single-device log is not the hazard it was before ZFS supported log
    device removal and pool import without one.

**Rollback point:** `zpool remove tank <p1>` at any time.

## Also do, independently of all of the above

**Cap `zfs_arc_max` at 48 GiB** (`51539607552` bytes), in Ansible, not by
hand.

`zfs_arc_max` is `0` today, so `c_max` is 66,086,522,880 bytes, **61.55 GiB of
62 GiB host RAM**. ARC has self-tuned to a 48.28 GiB target and 46.86 GiB
actual, so 48 GiB pins it where it already sits and removes only the headroom
to balloon and starve SCST, `nfsd`, `middlewared`, `ix-apps` and the QDevice
VM.

!!! note "Do not credit this with fixing anything"
    The specific RAM pressure that contributed to 2026-09-20 was
    `l2_hdr_size` at 20.6 GiB and it is **already gone** (`l2_hdr_size` reads
    `0`). `arc_no_grow` is `0` and `evict_skip` is 8,161, so ARC is not under
    pressure today. This is a cheap ceiling against a future workload, not a
    remedy for an observed problem. High confidence it is harmless, medium
    confidence it is necessary.

## Follow-up

Fill in the [validation table in ADR 0009](../decisions/0009-nvme-repurposing.md#validation)
as each step lands, and leave the unfilled rows visibly unfilled.

ADR 0008 warned against "a plan recorded as though it were an outcome", then
had step 1 executed with all three of its predicted outcomes unrecorded and
none of them true as written. The warning was right and was not followed.

The row most likely to be skipped is the `quota` check, and it is the one that
has already caught this lab out: **read `AVAIL` on a child zvol, not on the
parent.**
