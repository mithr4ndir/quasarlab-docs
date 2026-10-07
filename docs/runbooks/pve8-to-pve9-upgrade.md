# Runbook: Proxmox VE 8 to 9 upgrade

Status: **planned, not started.** Facts below were measured on 2026-10-07.

## Why now

Both hypervisors run `pve-manager/8.4.21` on Debian bookworm, kernel
`6.8.12-43-pve`. Proxmox VE 8 reached end of life on **2026-08-31**, so the pair
that runs the entire lab has been receiving no security updates for five weeks.

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
`ceph` entry in `/etc/pve/storage.cfg`** at all. So this is a Ceph that was
started and never finished, backing zero storage. The fix is to destroy it and
remove the `ceph-reef` repo, not to attempt a Reef to Squid migration.

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

1. **pve:** destroy the vestigial Ceph (mon and mgr on pve, zero OSDs, zero
   pools), purge the ceph packages, remove the `ceph-reef bookworm` repo.
   Confirm no guest config and no `storage.cfg` entry references ceph first.
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
