# 1Password and ExternalSecrets

The lab pulls every cluster-side secret from 1Password through ExternalSecrets Operator. Ansible-side secrets come through the same vault via the `op` CLI plus a small caching wrapper. This page is the operational toolkit for both.

The structural background is in [Runbook: 1Password rate-limit hit](../runbooks/1password-rate-limit.md) and the [2026-04-18 incident](../incidents/2026-04-18-1password-rate-limit.md). The TL;DR is: **two limits, two scopes**. Hourly is per-token, daily is per-account. Confusing them costs hours.

## The free probe

This is the most useful command on the page. It does **not** consume quota and works even when you are at 1000/1000.

```bash
op service-account ratelimit
```

Sample output during an active rate-limit:

```text
TYPE       ACTION        LIMIT    USED    REMAINING    RESET
token      write         100      0       100          N/A
token      read          1000     0       1000         N/A
account    read_write    1000     1000    0            5 hours from now
```

Always run this first when something feels rate-limited. Do not guess from `op read` error messages; they truncate the retry seconds and do not say which limit you hit.

## Reading and writing secrets

```bash
# Read a single field. The cache wrapper exports these as env vars at
# the top of an ansible run, so playbook tasks should read env, not call op.
op read 'op://<vault>/<item>/<field>'

# Read with cache (the lab's wrapper):
. scripts/lib/op-secret-cache.sh
load_cached_secrets   # populates env vars for everything the playbook needs

# Then in the playbook:
# - name: Use the secret
#   shell: do-thing-with "{{ lookup('env', 'OP_PROXMOX_API_TOKEN') }}"
#   no_log: true

# Inspect an item without reading any secret value:
op item get '<item-name>' --vault <vault> --format json | jq '.fields[] | {label, type}'

# Create a new item (write tokens are scoped separately, see service-account section):
op item create --vault <vault> --category 'Secure Note' --title '<title>' \
  notesPlain[text]='<content>'
```

## ExternalSecrets day-to-day

```bash
# What's the state of every ExternalSecret?
kubectl get externalsecret -A \
  -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,STATUS:.status.conditions[0].reason,SYNCED:.status.conditions[0].status,LASTSYNC:.status.refreshTime

# Force a single one to re-sync now (outside the refreshInterval):
kubectl annotate externalsecret <name> -n <ns> \
  force-sync=$(date +%s) --overwrite

# Show the rendered Secret produced by an ExternalSecret:
kubectl get secret -n <ns> <secret-name> -o json \
  | jq -r '.data | to_entries[] | "\(.key): \(.value | @base64d)"'

# Look at ESO's own logs (where retry storms show up):
kubectl -n external-secrets logs -l app.kubernetes.io/name=external-secrets \
  --tail=200 | grep -iE "1password|onepassword|rate|retry"
```

## Stop-the-bleed during a rate-limit

When the daily account cap is exhausted, every ESO retry and every ansible fork that calls `op` extends the rolling 24h window. The order matters: stop the amplifier first, then the cron callers, then the manual probes.

```bash
# 1. Scale ESO to zero. The retry loop is non-configurable and is the
#    primary amplifier.
kubectl -n external-secrets scale deploy external-secrets               --replicas=0
kubectl -n external-secrets scale deploy external-secrets-cert-controller --replicas=0
kubectl -n external-secrets scale deploy external-secrets-webhook       --replicas=0

# 2. Disable ansible timers on command-center1.
ssh command-center1 'sudo systemctl stop    ansible-proxmox.timer ansible-security.timer'
ssh command-center1 'sudo systemctl disable ansible-proxmox.timer ansible-security.timer'

# 3. Trip the local kill-switch so any straggler script bails fast.
ssh command-center1 'scripts/op-killswitch-status.sh trip'

# 4. Wait. Do not poll. Every poll resets the rolling window.
#    The reset time is in the `op service-account ratelimit` output.

# 5. After the cap recovers, restore in reverse:
ssh command-center1 'scripts/op-killswitch-status.sh clear'
kubectl -n external-secrets scale deploy external-secrets               --replicas=1
kubectl -n external-secrets scale deploy external-secrets-cert-controller --replicas=1
kubectl -n external-secrets scale deploy external-secrets-webhook       --replicas=1
# Re-enabling the ansible timers is a separate decision; do not flip
# them back on until Phase 1 of the secrets-IaC rollout has shipped.
```

## Kill-switch operations

`/var/lib/ansible-quasarlab/1p-killswitch` with a 24h TTL. All ansible wrappers honor it. Trips automatically when any wrapper observes "Too many requests" in stderr.

```bash
# What's the current state?
scripts/op-killswitch-status.sh status

# Trip manually (e.g. as part of incident response):
scripts/op-killswitch-status.sh trip

# Clear once the cap has recovered:
scripts/op-killswitch-status.sh clear
```

The Prometheus gauge `onepassword_killswitch_active` is alertable; `> 0` for more than a few minutes is worth a page during business hours.

## Service-account hygiene

```bash
# Which service accounts exist? (run with the user op token, not an SA)
op service-account list

# Inspect one (read-only, free):
op service-account get <name>

# Token files live under ~/.config/op/. Permissions matter:
ls -la ~/.config/op/service-account-token
# expected: 0600, owned by the user that runs ansible
```

Reminder: the **daily** rate-limit bucket is **per 1Password account**. Creating a new SA does not give you new daily quota. The hourly buckets are per-token, so a new SA helps with hourly limits only.

## Tier upgrade table (in case you decide it is time)

| Tier | Daily account cap | Notes |
|------|-------------------|-------|
| Personal / Families | 1,000 | Quasarlab is here |
| Teams | 5,000 | 5x bump |
| Business | 50,000 | 50x bump |

## Patterns worth knowing

- **`op service-account ratelimit` is free.** Use it before doing anything else.
- **The daily limit is per account.** A new SA does not double your daily quota.
- **Retry loops are the amplifier.** Scaling ESO to 0 is the single biggest mitigation when the daily cap is hit.
- **Cache at the wrapper, not in the playbook.** The lab's `op-secret-cache.sh` populates env vars once per run; playbook tasks should read env, not call `op`.
- **Cache-bypassers are easy to miss.** Dynamic inventory plugins, custom lookup plugins, and pre-tasks that call `op` directly all bypass a wrapper-level cache. Audit with: `grep -rn '\bop\b.*read\|\bop\b.*item get' --include="*.yml" --include="*.py" --include="*.sh" .`
- **`onepassword_ratelimit_*` Prometheus gauges** come from the `op-quota-collector` timer on `command-center1`. Alert at 50%, 80%, and 95% of the daily cap, not just at 100%.
