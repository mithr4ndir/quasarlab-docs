# Runbook: 1Password rate-limit hit

## Symptom

One or more of:

- ESO ExternalSecret resources stuck in `SyncedFalse`, with status messages like "rate limited" or "Too many requests".
- An ansible-playbook run fails partway through with `op read` returning `Too many requests` to stderr.
- A cron-driven script (Prometheus target sync, vault-pass.sh, the bridge token refresh) starts erroring at random points in the day.
- The 1Password quota Grafana panel suddenly shows `read_write USED: 1000 / 1000`, REMAINING 0.
- Discord goes quiet on the `#ops` channel because `discord-alert-proxy` cannot pull its webhook URL secret to fan out.

## What you are actually hitting

1Password service accounts have **two independent** rate limits, and confusing them is the #1 reason I wasted time during the 2026-04-18 incident:

| Limit | Period | Scope | Personal / Families value |
|-------|--------|-------|---------------------------|
| Per-token read | 1 hour | per service account token | 1,000 |
| Per-token write | 1 hour | per service account token | 100 |
| **Account read+write** | **24 hours** | **per 1Password account, all SAs combined** | **1,000** |

The big one for me is the **daily account cap**. Creating a second service account does **not** double your daily quota. The bucket is the account.

## Triage

Always start with the live status:

```bash
op service-account ratelimit
```

This is a free control-plane call. It does not consume quota and works even when you are at 1000/1000. Sample output during the 2026-04-18 incident:

```text
TYPE       ACTION        LIMIT    USED    REMAINING    RESET
token      write         100      0       100          N/A
token      read          1000     0       1000         N/A
account    read_write    1000     1000    0            5 hours from now
```

The "RESET" column tells you when the rolling 24h window will free up the quota. The CLI's other error messages (`Try again in  seconds`) truncate the number; do not rely on them.

Then identify who is consuming quota:

```bash
# ESO controller pods, recent reconcile activity:
kubectl -n external-secrets logs -l app.kubernetes.io/name=external-secrets --tail=200 \
  | grep -iE "1password|onepassword|rate"

# Process tree on command-center1: who is still calling op?
ssh command-center1 'ps -eo pid,ppid,cmd --forest | grep -E "(op |op-|ansible-playbook|run-)" | grep -v grep'

# The Prometheus collector exports onepassword_ratelimit_remaining; trend over hours/days:
# (open the Grafana 1P quota dashboard, or)
curl -s 'http://192.168.1.230:9090/api/v1/query?query=onepassword_ratelimit_remaining' | jq .
```

## Stop the bleed

Do all three of these in order, even if you think you know which one is at fault. The retry loops compound and the daily window only resets once.

```bash
# 1. Scale ESO to zero. Its retry loop fires every ~6 min when a sync fails
#    and is NOT configurable. This is the single biggest amplifier when the
#    cap has already been hit.
kubectl -n external-secrets scale deploy external-secrets --replicas=0
kubectl -n external-secrets scale deploy external-secrets-cert-controller --replicas=0
kubectl -n external-secrets scale deploy external-secrets-webhook --replicas=0

# 2. Disable the ansible timers on command-center1.
ssh command-center1 'sudo systemctl stop ansible-proxmox.timer ansible-security.timer'
ssh command-center1 'sudo systemctl disable --now ansible-proxmox.timer ansible-security.timer'

# 3. Trip the local kill-switch so any straggler script bails fast.
ssh command-center1 'scripts/op-killswitch-status.sh trip'
```

## Wait it out

The 24h account cap is the long pole. From `RESET: N hours from now`, that is when it lifts. Do not poll. Do not "just check if it cleared yet." Every probe call was free above the limit, but every actual `op read` you trigger will reset the rolling window.

After the cap recovers:

```bash
# Verify
op service-account ratelimit

# Clear the kill-switch
ssh command-center1 'scripts/op-killswitch-status.sh clear'

# Bring ESO back
kubectl -n external-secrets scale deploy external-secrets --replicas=1
kubectl -n external-secrets scale deploy external-secrets-cert-controller --replicas=1
kubectl -n external-secrets scale deploy external-secrets-webhook --replicas=1

# Force a sync on the watchdog secret first; if that succeeds the rest will follow.
kubectl annotate externalsecret <name> -n <ns> force-sync=$(date +%s) --overwrite
```

Re-enable the ansible timers **only after** confirming Phase 1 of the secrets-IaC rollout has shipped. The 2026-05-02 incident was caused by re-enabling the timers while the dynamic Proxmox inventory still calls `op read` per fork.

## Why creating another service account does not help

This is the single most common wrong intuition during the incident. New SAs get a fresh **hourly** budget but share the **daily** account budget. I tried this on 2026-04-18, created `eso-op-retrieval`, swapped ESO to it, and its very first `op vault list` call returned "Too many requests." The bucket is the account.

The only way to raise the daily ceiling is to upgrade the 1Password tier:

| Tier | Daily account cap |
|------|-------------------|
| Personal / Families | 1,000 |
| Teams | 5,000 |
| Business | 50,000 |

## Long-term mitigations already in place

- **Kill switch** at `/var/lib/ansible-quasarlab/1p-killswitch`, 24h TTL. All ansible wrappers honor it. Trips automatically when any script observes "Too many requests" in stderr.
- **Secret cache** at `/var/lib/ansible-quasarlab/secrets/`, 12h TTL, populated once per playbook run via `scripts/lib/op-secret-cache.sh`.
- **ESO refresh interval** bumped from 1h to 24h on every ExternalSecret. The retry loop is unaffected by this and is the unsolved part.
- **Quota collector** running on `command-center1` (`op-quota-collector.timer`, every 5 min, exports `onepassword_ratelimit_*` gauges). Alerts at 50% / 80% / 95% of the daily cap.
- **Operational rule for Claude:** one `op` call per session, only when explicitly requested by the user. Never loop or re-probe.

## Outstanding work

The dynamic Proxmox inventory still calls `op read` per fork at inventory resolution time, bypassing the secret cache. This is the structural cause of the 04-18 and 05-02 recurrences. The proper fix is to vault the Proxmox API token in `group_vars/vault.yml` so ansible reads it once at play start. Tracked as Phase 1 of the 04-22 secrets-IaC rollout.

## Patterns worth knowing

- **Two limits, two scopes.** Hourly is per-token; daily is per-account. Confusing them costs hours.
- **`op service-account ratelimit` is free.** It is the one call you can always make. Use it before doing anything else.
- **The retry loop is the amplifier.** ESO and any ansible re-fork can burn a clean budget in under an hour. Stop the retries, then wait.
- **Don't probe to "see if it cleared."** Every probe is a real call, and every real call resets the rolling window.
