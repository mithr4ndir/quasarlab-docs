# Ansible

The lab uses `ansible-quasarlab` for VM-level configuration. Inventory is `inventory.proxmox.yml` (dynamic from Proxmox API) plus `inventory.static.ini` for things outside Proxmox.

## Day-to-day commands

```bash
# Inventory sanity check:
ansible-inventory --graph
ansible-inventory --list --limit <group>

# Ping every host (does it answer over SSH?):
ansible all -m ping

# Run an ad-hoc command:
ansible <group> -m shell -a 'systemctl is-active postgresql@16-main' -b

# Gather facts for one host:
ansible <host> -m setup
ansible <host> -m setup -a 'filter=ansible_distribution*'
```

## Playbook runs, with safety

```bash
# Always check first when touching a real host:
ansible-playbook -i inventory.proxmox.yml playbook.yml \
  --limit <host> --check --diff

# Only run a subset of tags:
ansible-playbook ... --tags "configure,reload" --skip-tags "destructive"

# Step through tasks one at a time (interactive):
ansible-playbook ... --step
```

`--check --diff` together is the gold pattern: dry-run mode plus a unified diff of every file the play would change.

## Vault

```bash
ansible-vault create   group_vars/all/vault.yml
ansible-vault edit     group_vars/all/vault.yml
ansible-vault encrypt_string 'secret-value' --name 'my_var'  # inline encryption
ansible-vault rekey    group_vars/all/vault.yml              # change password
```

The vault password lives in a `pass`/`gopass` entry locally and is referenced via `ansible.cfg`'s `vault_password_file = ./bin/vault-pass.sh`.

## Roles I lean on

- `common/os/debian` — base hardening, NTP, unattended upgrades, non-root user.
- `common/vm_baseline`: the every-VM role. Baseline packages, qemu-guest-agent, chrony, sysctl tuning, UTC, the journald size cap and vacuum, and since 2026-09-19 the operator SSH `authorized_keys` ([ansible-quasarlab#187](https://github.com/mithr4ndir/ansible-quasarlab/pull/187), [#188](https://github.com/mithr4ndir/ansible-quasarlab/pull/188), [#189](https://github.com/mithr4ndir/ansible-quasarlab/pull/189)). Keys used to be seeded only by Terraform cloud-init at build time, which means nothing converged them afterwards: a host built before a key existed could never be given that key by re-running playbooks, no matter how many times you ran them. uptime-kuma, musicbot and jellyfin were all missing the ed25519 key for exactly that reason. The list is additive (`vm_baseline_authorized_keys_exclusive: false`) so it never fights cloud-init or locks out a key added out of band. Two non-obvious traps in the implementation are documented in the role itself, at [`roles/common/vm_baseline`](https://github.com/mithr4ndir/ansible-quasarlab/tree/main/roles/common/vm_baseline), next to the tests that hold them down.
- `k8s/docker`: installs Debian's `docker.io`, and is used by `playbooks/k8s_init.yml` only. On Debian 12 that package is 20.10.24 with **no compose plugin**, which is why Uptime Kuma does not use it: `roles/uptime_kuma/tasks/docker.yml` installs Docker CE from Docker's own apt repo, with the signing key pinned by full fingerprint.
- `monitoring/node_exporter` — Prometheus node_exporter on every VM.
- `monitoring/filebeat` (now `vector`) — log shipping to the in-cluster aggregator.

## Things I reach for when a play has gone wrong

| Question | Command |
|----------|---------|
| Why did this task fail? | Re-run with `-vvvv` for SSH-level detail. |
| What state did it leave the host in? | `ansible <host> -m shell -a 'cat /etc/<file>' -b` |
| Is something in the play wrong, or the inventory? | `--check --diff --limit <host>` and read the diff. |
| Has this playbook ever succeeded? | `git log --follow playbooks/<file>` and check CI. |

## Scheduled runs deploy `main`, not your checkout

Two timers on cmd_center1 run at different cadences (`ansible-proxmox` hourly,
`ansible-security` every 30 minutes). Both run
from a **dedicated automation checkout**, force-synced to its remote ref before
every run:

```bash
/var/lib/ansible-quasarlab/repo           # pinned to origin/main
/var/lib/ansible-quasarlab/observability  # pinned to origin/master
```

```bash
# What is automation actually running?
git -C /var/lib/ansible-quasarlab/repo log --oneline -1

# Confirm the units point at the automation checkout, not ~/code
systemctl cat ansible-proxmox.service | grep -E 'ExecStart|WorkingDirectory'

# One-time bootstrap after a rebuild or a unit change
ansible-playbook playbooks/cmd_center.yml --tags ansible_timers --diff
```

!!! warning "Unmerged work is not deployed. Undoing it is a separate job."
    Because runs pin to `origin/main`, a change only sticks once merged.

    But the next run only reconciles what `main` **declares**. A templated file
    comes back; a purged package, a deleted directory or a service restart does
    not, because those tasks do not exist in `main` to reverse them. After an
    accidental deploy, read the `changed:` lines in the run log and clean up the
    irreversible parts by hand.

`~/code/ansible-quasarlab` is the **operator** tree. Keep any branch checked out
there; timers do not read it. Manual `ansible-playbook` runs *do*, so a manual
run can deploy uncommitted work. Never hand-edit the automation checkout: the
next run discards it.

This exists because the timers used to run straight out of `~/code` and, on
2026-08-24, [deployed an unmerged branch to the
fleet](../incidents/2026-08-24-jellyfin-transcode-disk-exhaustion.md).

## Integration with the rest of the lab

- Postgres lives on a VM and is owned by Ansible. Manual edits during incidents (like the [pg_hba fix](../runbooks/pg-hba-rejecting-k8s-pods.md)) must be back-ported to the role the same day.
- Uptime Kuma is deployed by `playbooks/uptime-kuma.yml` through `scripts/run-uptime-kuma.sh`, and vm117 lives in `inventory.static.ini` deliberately: the monitor of last resort has to be deployable with the Proxmox API, the dynamic inventory and the Proxmox token all unavailable. The wrapper is operator-run, never on a timer, and passes an allowlist of `ansible-playbook` options rather than a denylist, because abbreviations (`--lim`) and clustered short flags (`-vi x.yml`) can otherwise smuggle in another inventory. The earlier labctl run left the VM provisioned but never started: it used `k8s/docker`, so its final `docker compose up -d` failed with `docker: 'compose' is not a docker command`. See [ADR 0007](../decisions/0007-uptime-kuma-external-monitor.md) and the [2026-09-19 incident](../incidents/2026-09-19-nfsv4-callback-deadlock.md).
- Anything K8s-related lives in ArgoCD, not Ansible. Ansible bootstraps the K8s VMs only.
- Alerting on the timers themselves: `AnsibleRunStale`, `AnsiblePlaybookFailed`, and `AnsibleRepoSyncFailed` (critical: a run refused to start because it could not pin its checkout, so nothing is being enforced anywhere).
