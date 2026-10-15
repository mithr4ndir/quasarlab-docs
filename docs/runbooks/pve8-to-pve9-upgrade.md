# Runbook: Proxmox VE 8 to 9 upgrade

Status: **planned, not started.** Facts below were measured on 2026-10-07.

## Why now

Both hypervisors run `pve-manager/8.4.21` on Debian bookworm, kernel
`6.8.12-43-pve`. Proxmox VE 8 reached end of life on **2026-08-31**.

### PVE 9 buys this lab almost nothing in features

Worth stating plainly, because it changes how much urgency this deserves.
Checked against the 9.0, 9.1 and 9.2 feature lists on 2026-10-15.

Does not apply here:

| Feature | Why it does not land |
|---|---|
| Snapshots as volume chains (9.0) | Targets LVM and shared storage with no native snapshots. Ours is ZFS-over-iSCSI, which snapshots natively already. |
| ZFS 2.3.4 / 2.4 | PVE root is LVM (`/dev/mapper/pve-root`) and our ZFS lives on the NAS, so PVE's ZFS version is irrelevant. |
| LXC containers from OCI images (9.1) | `pve8to9` reports no containers on either node. |
| GlusterFS and cgroupv1 dropped | Not used. |

Applies in theory, blocked by our own constraints:

- **CRS dynamic load balancing (9.2)** is unusable. `cpu: host` across an Intel
  pve and an AMD pve2 blocks live migration, and seven guests are pinned by
  local lvmthin.
- **HA affinity and negative affinity rules** map onto a real problem, both
  spare etcd members sitting on pve2, but are defeated by that same pinning.
  They only pay off after Phase A puts those disks on shared storage.
- **`disarm-ha` / `arm-ha` (9.2)** would make the *next* upgrade safer, not
  this one.

Costs rather than benefits: `VM.Monitor` is removed, which breaks the
`AnsibleInventory` role; the release notes warn of "Potential changes in
network interface names", a live risk given the 10G direct NIC-to-NAS link; and
AppArmor 4. The one plausible small win is **extended metrics collection**
(9.0), worth a look given how much we lean on Prometheus, but not a reason to
upgrade.

### The actual case is that the kernel and QEMU stopped being patched

Measured 2026-10-15. The cleanest evidence is the repo metadata itself, fetched
live from the remote so a stale local cache cannot fake it:

```
remote bookworm InRelease:  Date: Wed, 26 Aug 2026 17:56:10 +0000
remote trixie   InRelease:  Date: Wed, 14 Oct 2026 17:22:48 +0000
```

**The PVE 8 channel stopped being rebuilt on 2026-08-26**, five days before the
announced EOL, and has not moved in the seven weeks since. The PVE 9 channel
was rebuilt yesterday. On both nodes, installed equals candidate for
`pve-manager` (8.4.21) and `pve-qemu-kvm` (9.2.0-8), with zero PVE updates
pending.

One correction worth recording, because the obvious version of this claim is
wrong. The kernel is **not** stuck at 6.8 for lack of alternatives: the 8.4
repo still offers opt-in `proxmox-kernel-6.11` (`6.11.11-2`) and
`proxmox-kernel-6.14` (`6.14.11-9~bpo12+1`) for bookworm. They are frozen along
with everything else in that channel, but 6.14.11 is a far newer baseline than
the running `6.8.12-43`. **If we defer PVE 9, opting into
`proxmox-kernel-6.14` is a cheap way to close most of the kernel gap** without
any of the Phase C risk. It does nothing for QEMU.

Debian underneath is **still covered**: bookworm moved to LTS, with security
support to 2028-06-30, though LTS is narrower than regular support. The catch
is that `proxmox-kernel-*` and `pve-qemu-kvm` are **Proxmox-built packages, not
Debian ones**, so Debian LTS does nothing for them. `pve-qemu-kvm` is the one
with no remaining mitigation at all, and QEMU is the component that matters
most for a guest-to-host escape.

That is not abstract here. The 2026-08-20 security sweep flagged RCON port
25575 WAN-exposed with `src=any`, and palworld (`106`) is a guest on pve2. An
internet-reachable guest in front of a frozen QEMU is the one chain where this
actually bites. **Closing that firewall rule buys more real security, sooner,
than this entire upgrade does.** Do that first, regardless.

### Decision: the phases are decoupled

Agreed 2026-10-15. **Phase A is approved and proceeds on its own merits**,
whether or not PVE 9 ever happens, because it:

- retires the cleartext TrueNAS root password in `storage.cfg` (see below),
- gets us off the Cloudsmith repo, which caps at `3.0.0-1` and is being retired
  upstream,
- and is the mandatory prerequisite if we do later want PVE 9.

Phase B and Phase C are **not** approved yet. Revisit them once Phase A is done
and the plugin is no longer the blocker.

## The finding that reshapes this project

The obvious plan ("clear the `pve8to9` blockers, then dist-upgrade") is wrong,
because the TrueNAS storage plugin gates everything and it cannot make the jump
in one step.

- **All 24 guests, every VM and every template, have at least one disk on
  `truenas-iscsi`.** Not one guest is local-only. The plugin is on the critical
  path for the whole lab.
- The installed plugin is `freenas-proxmox 2.4.0-1`, held, on both nodes. Its
  patch set is `patches/ZFSPlugin/{7,8,8.4}.patch`. **There is no `9.patch`**,
  confirmed by reading `dpkg -L freenas-proxmox` on the host, so its dpkg
  trigger fails on PVE 9.
- Upstream `docs/upgrade-paths.md` at tag `v4.0.1` states plainly that
  **v2.x to v4.x is not a supported path**: it "will require v3 config first".
- v2 to v3 is a breaking change, and upstream's own table answers
  "In-place upgrade?" with **"No, VM disks must be moved"**. The auth model
  changes to an API key, the iSCSI model changes, and the package is renamed
  `freenas-proxmox` to `truenas-proxmox`.
- The currently configured Cloudsmith repo
  (`dl.cloudsmith.io/public/ksatechnologies/truenas-proxmox`) **tops out at
  3.0.0-1** and will never serve v4. Cloudsmith is being retired upstream in
  favour of GitHub Pages dist tracks (`main` for v2, `v3`, `v4`).

So the plugin migration is the long pole, it comes first, and it is a separate
piece of work from the hypervisor upgrade.

## Measured preconditions

| Fact | pve (192.168.1.10) | pve2 (192.168.1.11) |
|---|---|---|
| Version | 8.4.21 / bookworm | 8.4.21 / bookworm |
| CPU vendor | Intel (`intel-microcode` wanted) | AMD (`amd64-microcode` wanted) |
| Root fs free | 851 GiB of 911 GiB | **26 GiB of 94 GiB (72% used)** |
| `pve8to9` result | 2 FAIL, 7 WARN | 3 FAIL, 5 WARN |
| Running guests | 10 | 6 |
| Boot | UEFI, `proxmox-boot-uuids` absent | UEFI, `proxmox-boot-uuids` absent |

Cluster `rocketfuel` is 2 nodes plus a QDevice, 3 votes, quorate. Only
`vm:115` (jellyfin) is HA managed, in group `gpu-vms`.

### Live migration is not available to us

Nearly every guest is configured `cpu: host`, and pve is Intel while pve2 is
AMD. A `cpu: host` guest cannot be live migrated across that vendor boundary.
Because all disks are on shared `truenas-iscsi`, *offline* migration is cheap
(config move, no data copy), so the pattern per node is: shut the guest down,
offline migrate, start it on the other node.

### Seven guests are pinned to their node by local lvmthin disks

These have disks on node-local `lvmthin` and cannot move at all without a data
copy. They go down with their host.

| Node | Pinned guests |
|---|---|
| pve | `103 pbs1`, `110 k8cluster1`, `117 uptime-kuma` |
| pve2 | `105 command-center1`, `109 k8cluster2`, `111 k8cluster3`, `120 pbs2` |

Three consequences worth internalising before scheduling anything:

1. **Upgrading pve2 takes down the Kubernetes control plane.** `k8cluster2` and
   `k8cluster3` are 2 of 3 etcd members and both are pinned to pve2, so they
   cannot be evacuated first. etcd loses quorum.
2. **Upgrading pve2 takes down `command-center1`**, which is the Ansible control
   node, the agent session host, and the Prometheus target sync. Plan for
   working from somewhere else during that window.
3. **Upgrading pve takes down `pbs1`**, which holds the backups you would roll
   back from. Do not start pve's upgrade while pbs1 is your only copy. `pbs2`
   exists but its sync job is still open work.

### `pve8to9` failures, by node

**Both nodes:** custom role `AnsibleInventory` uses the to-be-dropped
`VM.Monitor` privilege (`/etc/pve/user.cfg`). This is the role the Ansible
dynamic Proxmox inventory authenticates with, so if it breaks, the automation
that runs this lab breaks. It already carries `Sys.Audit`; the replacement for
guest agent access is the new `VM.GuestAgent.*` family.

**pve:** `FAIL: Hyper-converged Ceph 18 Reef is to old for upgrade!` This one
looks alarming and is not. The Ceph install is **vestigial and holds nothing**:

```
mon: 1 daemons, quorum pve      osd: 0 osds: 0 up, 0 in
pools: 0 pools, 0 pgs           objects: 0 objects, 0 B
```

`HEALTH_WARN` is only `OSD count 0 < osd_pool_default_size 3`. There is **no
`ceph` entry in `/etc/pve/storage.cfg`**, no `rbd:` or `cephfs:` storage, no
guest config that mentions ceph, and no device claimed by `ceph-volume`.
`/etc/pve/ceph.conf` is dated **2024-10-01**, so this has been dead weight for
over two years. It was started and never finished. The fix is a cleanup, not a
Reef to Squid migration.

### Do not purge all the ceph packages {#ceph-dependency-trap}

Measured 2026-10-15, and this is the part that will bite you.
`pve-qemu-kvm` and `libpve-storage-perl` link against the Ceph client
libraries, so `apt purge` of the obvious `ceph*` glob cascades into PVE itself.
`apt-get purge -s` of every ceph package on pve reports it would also purge:

```
pve-ha-manager   qemu-server   pve-qemu-kvm
libpve-guest-common-perl   libpve-storage-perl
```

That removes the ability to run virtual machines at all. **Keep**
`ceph-common`, `libcephfs2`, `python3-cephfs`, `python3-ceph-common`,
`python3-ceph-argparse` and `ceph-fuse`.

Only the server side daemons can go. On pve that is exactly these eight, and
`apt-get purge -s` confirms nothing else is touched:

```
ceph  ceph-base  ceph-mds  ceph-mgr  ceph-mgr-modules-core
ceph-mon  ceph-osd  ceph-volume
```

**pve2 needs no package work.** It has no server side ceph at all, only the six
client packages listed above as keepers, which PVE core depends on.

A note on verifying this yourself: `apt-get purge -s` prints `Purg`, not
`Remv`. A danger check that greps only for `^Remv` matches nothing and hands
you a false all clear. Grep for both, and confirm the check can actually fail
by looking at the package list it prints.

### Leave the ceph apt repo alone until the upgrade

`/etc/apt/sources.list.d/ceph.list` (`ceph-reef bookworm`) is present on
**both** nodes, and it is where the kept `ceph-common` comes from. Deleting it
orphans a package that `pve-qemu-kvm` depends on. At Phase C, re-point it at
the PVE 9 Ceph repo (squid) rather than removing it.

**pve2:** two more failures, both straightforward.

- `systemd-boot` meta-package installed. Documented PVE 8 to 9 hazard, see the
  upstream `#sd-boot-warning` note. Remove the meta-package.
- Mixed repo suites: `/etc/apt/sources.list.d/microsoft.list` pins **bullseye**
  while the base is bookworm. Two releases stale and a hard blocker.

### Warnings worth acting on, not just noting

- **pve: `grub-efi-amd64` meta-package is not installed on a UEFI system**, so
  "new grub versions will not be installed to `/boot/efi`". On a hypervisor,
  shipping a new kernel that never reaches the ESP is how you get a host that
  does not come back. Fix this before any kernel change.
- Both nodes: removable bootloader at `/boot/efi/EFI/BOOT/BOOTX64.efi` is not
  maintained by the GRUB packages.
- pve2: dkms modules present (CUDA repo is configured here), which the checker
  flags as upgrade risk.
- pve still carries `pve-kernel-5.13 7.1-9`, leftover from PVE 7.
- LVM guest volumes still have autoactivation enabled, which PVE 9 changes the
  default for. Upstream ships
  `/usr/share/pve-manager/migrations/pve-lvm-disable-autoactivation`.

## Security finding, separate from the upgrade

`/etc/pve/storage.cfg` holds the TrueNAS **root** password in cleartext
(`freenas_user root` with a plaintext `freenas_password`). That file lives in
pmxcfs, so it is replicated to both nodes and captured in every backup of
`/etc`. Treat the credential as compromised and rotate it.

The good news: the v2 to v3 plugin migration replaces username and password
auth with an API key, so Phase A fixes this as a side effect rather than
needing its own project. Scope the new key to what the plugin needs instead of
reusing root.

## Phase A: plugin 2.4.0 to v3.x, on PVE 8, no hypervisor change

Independently valuable: it retires the cleartext root password, gets off the
retiring Cloudsmith repo, and is the mandatory prerequisite for v4. Upstream
says the disk moves are **live**, so this is a long grind rather than an outage.

1. Read upstream `docs/migrating-from-v2.md` end to end first. Do not
   `apt upgrade` the plugin from the Cloudsmith install, which would walk
   straight into the breaking change.
2. Create a scoped TrueNAS API key. Do not reuse root.
3. Swap the apt source from Cloudsmith to the GitHub Pages `v3` dist track,
   with the key at `/etc/apt/keyrings/truenas-proxmox.gpg`.
4. Install `truenas-proxmox` on **both** nodes.
5. Add a **new** v3 storage entry alongside the existing v2 `truenas-iscsi`.
   Both run in parallel during the migration.
6. Move disks with Proxmox Move Disk, one guest at a time, guests stay running.
   24 guests, so expect this to span days. Verify each guest boots and its data
   is intact before moving the next.
7. Remove the old v2 storage entry and purge `freenas-proxmox` only once
   nothing references it.
8. Rotate the old TrueNAS root password once the plaintext entry is gone.

Gate before leaving Phase A: `grep -c truenas-iscsi` across all guest configs
returns 0, and a restore test from pbs1 still succeeds.

## Phase B: clear the host blockers, still on PVE 8

Each of these is small and independently verifiable, and none of them require
the upgrade to proceed. Doing them first makes `pve8to9` clean, which is the
real go or no-go signal.

1. **pve:** remove the vestigial Ceph. Stop and destroy the mgr and the mon
   (zero OSDs and zero pools, so there is nothing to drain), then purge **only
   the eight server side packages** listed in
   [the dependency trap above](#ceph-dependency-trap). Keep `ceph-common` and
   the client libraries, and leave `ceph.list` in place. Then remove
   `/etc/pve/ceph.conf`, the `/etc/ceph` symlink and keyring, and
   `/var/lib/ceph/{mon,mgr}/ceph-pve` (20 MiB). pve2 needs no package work.
   Re-run `pve8to9` to confirm the FAIL clears. Note `pveceph purge` behaviour
   on a last remaining mon has not been validated here, so check it before
   relying on it.
2. **pve:** `apt install grub-efi-amd64`, then set
   `grub2/force_efi_extra_removable` and reinstall grub so the ESP is actually
   maintained. Verify a kernel install lands in `/boot/efi`.
3. **pve2:** remove the `systemd-boot` meta-package.
4. **pve2:** fix or remove the bullseye `microsoft.list`, and drop the
   duplicate `pve bookworm pve-no-subscription` line.
5. **Both:** adapt the `AnsibleInventory` role off `VM.Monitor` onto the
   `VM.GuestAgent.*` equivalents. Then prove the Ansible dynamic inventory
   still authenticates, without burning 1Password quota.
6. **pve2:** reclaim root fs space. 26 GiB is thin for a dist-upgrade.
7. **Both:** install the matching microcode package, run the LVM
   autoactivation migration script, remove the PVE 7 era `pve-kernel-5.13`.
8. Re-run `pve8to9 --full` on both until clean.

## Phase C: the upgrade itself

Only start when Phase A and Phase B are both closed and `pve8to9` is clean.

Order: **pve2 first**, because pve holds pbs1 and the HA master, so you want
your backup server and HA master on the already-proven node last. Accept that
upgrading pve2 means a Kubernetes control plane outage and losing
`command-center1`.

Per node:

1. Confirm a completed, restore-tested PBS backup of every guest on that node.
2. Set HA for `vm:115` to `ignored` so HA does not try to relocate a GPU
   passthrough guest onto a node without the GPU.
3. Offline migrate everything that *can* move to the other node. Shut down the
   pinned guests listed above.
4. Switch apt suites bookworm to trixie, `apt dist-upgrade`, reboot.
5. Verify the node boots, rejoins the cluster with quorum, and the QDevice
   still votes.
6. Move the plugin to the `v4` dist track and install `truenas-proxmox` v4.x.
   v4.0.0 adds WebSocket JSON-RPC for TrueNAS SCALE 25.04+, auto-detected per
   host, with no `storage.cfg` change required either way.
7. Verify storage comes back and guests start, before touching the other node.

## Rollback

- Per guest: restore from pbs1. Proven by a real restore on 2026-10-06
  (nginx2, `/etc` md5 plus 687 packages identical).
- Per node: there is no in-place downgrade from a Debian dist-upgrade. The
  rollback for a failed node is rebuild and restore, which is why the Phase C
  gate is a verified backup of every guest on that node, not just a recent job.
- Because the two nodes are upgraded one at a time, a failure on pve2 leaves pve
  on 8.4.21 still serving. Mixed 8 and 9 in one cluster is supported for the
  duration of a rolling upgrade, not as a resting state.

## Open questions

- Exact TrueNAS version. Kernel is `6.6.44-production+truenas` built
  2025-08-06, which is the SCALE 24.10 / 25.04 family. Confirm the exact
  release, since it decides whether v4 uses WebSocket or REST.
- Whether the v3 to v4 step is in-place. Upstream `upgrade-paths.md` still says
  "TBD" for v3 to v4 even at tag `v4.0.1`, while the v4.0.0 release notes say no
  `storage.cfg` changes are required. Resolve against `CHANGELOG.md` before
  Phase C step 6.
- Which dkms modules are on pve2, and whether they survive a trixie kernel.
- Whether `pbs2` should be brought into service before Phase C so that pve's
  upgrade is not gated on pbs1 being up.
