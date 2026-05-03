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
- `k8s/docker` — Docker engine for VM-resident services (uptime-kuma, registry, etc.).
- `monitoring/node_exporter` — Prometheus node_exporter on every VM.
- `monitoring/filebeat` (now `vector`) — log shipping to the in-cluster aggregator.

## Things I reach for when a play has gone wrong

| Question | Command |
|----------|---------|
| Why did this task fail? | Re-run with `-vvvv` for SSH-level detail. |
| What state did it leave the host in? | `ansible <host> -m shell -a 'cat /etc/<file>' -b` |
| Is something in the play wrong, or the inventory? | `--check --diff --limit <host>` and read the diff. |
| Has this playbook ever succeeded? | `git log --follow playbooks/<file>` and check CI. |

## Integration with the rest of the lab

- Postgres lives on a VM and is owned by Ansible. Manual edits during incidents (like the [pg_hba fix](../runbooks/pg-hba-rejecting-k8s-pods.md)) must be back-ported to the role the same day.
- Uptime Kuma's VM had a stub playbook that never completed end-to-end, which is how the VM ended up idle for 18 days. ([decision pending](../decisions/index.md))
- Anything K8s-related lives in ArgoCD, not Ansible. Ansible bootstraps the K8s VMs only.
