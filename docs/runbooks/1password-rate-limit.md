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

The "RESET" column tells you when the 24-hour window rolls over and frees the quota (on this account, about 02:51 UTC). The CLI's other error messages (`Try again in  seconds`) truncate the number; do not rely on them.

Then find who is spending it. Run everything below on command-center1.

**1. Attribution.** `op` calls made through the logging wrapper are counted per consumer:

```bash
# Top consumers, last hour
curl -s 'http://192.168.1.230:9090/api/v1/query' \
  --data-urlencode 'query=topk(10, sum by (consumer, subcommand) (increase(onepassword_op_invocations_total[1h])))' | jq .

# Raw records, newest last
tail -n 50 /var/log/op-shim/op-invocations.log | jq -c '{ts, consumer, chain, subcommand}'
```

**2. Burst not explained by attribution?** Reconcile against the account and token counters, then measure one suspect operation against the free status call:

```bash
before=$(op service-account ratelimit | awk '/account/{print $4}')
ssh command-center1 true          # or: one playbook task, one script, one sync
after=$(op service-account ratelimit | awk '/account/{print $4}')
echo "cost: $((after - before)) reads"   # expect 0
```

A clean `strace -f` or process list does not clear SSH-driven work: shells that `sshd` starts are outside the caller's process tree, even on the same host ([2026-09-12](../incidents/2026-09-12-op-quota-shell-profile.md)).

**3. Other places to look:**

```bash
# ESO controller, recent reconcile activity
kubectl -n external-secrets logs -l app.kubernetes.io/name=external-secrets --tail=200 \
  | grep -iE "1password|onepassword|rate"

# Long-running callers, names only (a snapshot: misses short calls)
ps -eo pid,ppid,comm --forest | grep -E "op|ansible|run-" | grep -v grep

# Shell startup files must never call op: every SSH command would pay for it.
# Lists matching files only, and ignores comment lines.
grep -lE '^[^#]*\bop[[:space:]]+(read|inject|item)\b' ~/.bashrc ~/.profile /etc/profile.d/*.sh 2>/dev/null

# Quota trend (or the Grafana 1P quota dashboard)
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

The 24h account cap is the long pole. From `RESET: N hours from now`, that is when it lifts. Do not poll. Do not "just check if it cleared yet." Every probe call was free above the limit, but every actual `op read` you trigger spends quota you do not have.

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

The kill switch clears itself once a free quota read shows more than 200 remaining. That does **not** restart timers you disabled in "Stop the bleed". Once attribution shows no unexpected consumer, re-enable them explicitly:

```bash
ssh command-center1 'sudo systemctl enable --now ansible-proxmox.timer ansible-security.timer'
ssh command-center1 'systemctl list-timers "ansible-*"'
```

## Why creating another service account does not help

This is the single most common wrong intuition during the incident. New SAs get a fresh **hourly** budget but share the **daily** account budget. I tried this on 2026-04-18, created `eso-op-retrieval`, swapped ESO to it, and its very first `op vault list` call returned "Too many requests." The bucket is the account.

The only way to raise the daily ceiling is to upgrade the 1Password tier:

| Tier | Daily account cap |
|------|-------------------|
| Personal / Families | 1,000 |
| Teams | 5,000 |
| Business | 50,000 |

## Long-term mitigations already in place

- **Kill switch** at `/var/lib/ansible-quasarlab/1p-killswitch`. All ansible wrappers honor it. It trips when a wrapper finds "Too many requests" anywhere in a run's captured output (which can false-trip on `--diff` text, see ansible-quasarlab#160), and clears once a free quota read shows more than 200 remaining (24h at most).
- **Secret cache:** restricted local files, 48h TTL, locked against concurrent misses, populated once per playbook run via `scripts/lib/op-secret-cache.sh`.
- **Attribution wrapper** on command-center1 exports `onepassword_op_invocations_total{consumer,caller,unit,subcommand}`.
- **Shell startup files:** `/etc/profile.d/op-ansible-env.sh` is templated without `op`, and a task fails the play if `op read` reappears in `~/.bashrc`.
- **ESO refresh interval** bumped from 1h to 24h on every ExternalSecret. The retry loop is unaffected by this and is the unsolved part.
- **Quota collector** running on `command-center1` (`op-quota-collector.timer`, every 5 min, exports `onepassword_ratelimit_*` gauges). Alerts at 50% (warning) / 80% / 95% of the daily cap, plus a burn-rate alert on reads per hour.
- **Operational rule for Claude:** one `op` call per session, only when explicitly requested by the user. Never loop or re-probe.

## Outstanding work

The dynamic inventory bypass was fixed in ansible-quasarlab#129 (measured at 0 reads on 2026-09-13), and the shell startup read in [2026-09-12](../incidents/2026-09-12-op-quota-shell-profile.md). Still open: the source of the 2026-09-12 bursts, and the kill switch output scan false-tripping on `--diff` text (ansible-quasarlab#160).

## Patterns worth knowing

- **Two limits, two scopes.** Hourly is per-token; daily is per-account. Confusing them costs hours.
- **`op service-account ratelimit` is free.** It is the one call you can always make. Use it before doing anything else.
- **The retry loop is the amplifier.** ESO and any ansible re-fork can burn a clean budget in under an hour. Stop the retries, then wait.
- **Don't probe with real reads to "see if it cleared."** Every `op read` spends quota you do not have. The status call `op service-account ratelimit` is free, so use that instead.
