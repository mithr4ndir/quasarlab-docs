# 2026-04-18: 1Password daily rate-limit, the per-account bucket nobody reads about until they hit it

**Date:** 2026-04-18
**Severity:** S2. ESO sync paralyzed for hours, several Discord-routed alerts delayed, no data loss.

## Symptom

- ExternalSecret resources stuck in `SyncedFalse` across the cluster.
- Ansible playbook runs failing partway through with `op read` returning `Too many requests`.
- A brand-new service account I had just created (`eso-op-retrieval`, intended to give ESO its own quota) returned "Too many requests" on its very first `op vault list`. That last bit was the surprising part.

## What I thought it was

Hourly per-token rate limit hit by a noisy ansible cron. Plan: create a separate service account for ESO so it has its own bucket. Implement, swap the K8s `onepassword-service-account-token` secret to the new SA, watch ESO recover.

That plan failed because the new SA's first call also rate-limited. Which meant my mental model was wrong.

## What it actually was

1Password service accounts have **two independent** rate limits, and they have very different scopes:

| Limit | Period | Scope |
|-------|--------|-------|
| Per-token read | 1 hour | per service account token |
| Per-token write | 1 hour | per service account token |
| **Account read+write** | **24 hours** | **per 1Password account, all SAs combined** |

The hourly limits are per-token, so creating a new SA does give you a fresh hourly budget. But the daily limit is **per account**, shared across every service account on it. The lab is on the Personal/Families tier, which has a 1,000 daily account cap. The `eso-op-retrieval` SA inherited the same exhausted bucket as the rest of the lab.

This is documented in 1Password's official rate-limit page if you read carefully, but the docs lead with the per-token numbers and the per-account cap is mentioned as a footnote-feeling line.

## What consumed the daily cap

Estimated mix during the incident window:

- Ansible timers at 25 to 30 reads/hr (before any caching work) ≈ ~700 reads/day baseline.
- ESO's controller-runtime retry loop, which fires every ~6 minutes when a sync fails, **regardless** of `refreshInterval`. Once the rate limit was tripped, ESO entered a retry storm at ~240 reads/hr, burning the remaining quota in roughly 4 hours.
- An interactive session creating items and probing fields: ~100 calls.

Total: well over 1,000 in 24 hours. Daily cap pinned for a full rolling-24h reset window.

## Triage

The single most useful command is the one I did not learn about until halfway through:

```bash
op service-account ratelimit
```

This is a **free** control-plane call. It does not consume quota and works even when you are at 1000/1000. Sample output during the incident:

```text
TYPE       ACTION        LIMIT    USED    REMAINING    RESET
token      write         100      0       100          N/A
token      read          1000     0       1000         N/A
account    read_write    1000     1000    0            5 hours from now
```

That single line told me everything: the token-level budgets were healthy, the account-level budget was exhausted, and the rolling window resets in 5 hours. No more guessing, no more probing.

## Stop the bleed

Three things, in order, all of which had to happen together to actually let the cap recover:

```bash
# 1. Scale ESO to zero. Its retry loop is the amplifier; the loop fires
#    every ~6 minutes when sync fails, with no backoff and not configurable.
kubectl -n external-secrets scale deploy external-secrets --replicas=0

# 2. Disable the ansible timers on command-center1 so they stop draining
#    the moment the cap recovers.
ssh command-center1 'sudo systemctl stop ansible-proxmox.timer ansible-security.timer'

# 3. Trip the local kill-switch so any straggler script bails fast
#    instead of doing one more retry against the same dry bucket.
ssh command-center1 'scripts/op-killswitch-status.sh trip'
```

Then wait. The 24h account window does not honor any "I promise to stop, can you reset early" wishes; you wait for it to roll forward.

## Structural fixes (PRs #104, #105, #106, #131)

The recurrence on 2026-05-02 is the proof that the structural fixes were not enough; there is one structural piece still outstanding (the dynamic Proxmox inventory bypass). But the four PRs from this incident materially reduced the blast radius:

### Kill switch (PR #104)

`/var/lib/ansible-quasarlab/1p-killswitch` is a lock file with a 24h TTL. All ansible wrappers check it before any `op` call. If tripped, they exit 0 silently. Any wrapper that observes "Too many requests" in stderr also trips it automatically, so one failure protects every subsequent caller for the rest of the day.

Also exposed as a Prometheus metric: `onepassword_killswitch_active`, alertable.

### Secret cache (PRs #105, #106)

`scripts/lib/op-secret-cache.sh` adds a file-backed cache at `/var/lib/ansible-quasarlab/secrets/` (mode 0700, files mode 0600). 12-hour TTL, stale-on-failure fallback. Wrappers call `load_cached_secrets` once at the top of each playbook to populate every needed secret as an env var. Playbook tasks that previously did `command: op read ...` switched to `lookup('env', X)` with `assert: that: lookup('env', X) | length > 0` for fail-fast behavior on empty values.

This cuts ansible's baseline from ~25/hr to 0 on cache hits, ~8/day on cache misses. Effectively eliminated as a quota consumer for the steady-state case.

### ESO refresh interval (PR #131)

All four ExternalSecret resources bumped from 1h to 24h `refreshInterval`. **Important:** this does **not** stop the retry storm on failure, because the controller-runtime retry loop is internal to ESO and ignores `refreshInterval` for retries. The only working defense against that loop, today, is scaling ESO to 0 when the cap is hit.

### Operational rules for the AI side of the lab

A separate set of guardrails landed in personal memory:

- One `op` call per session maximum, only when explicitly requested.
- Never loop, never re-probe to "check if cleared."
- When rate-limited: stop all `op`, pause ansible timers, scale ESO to 0, wait an hour-plus of total quiet.

This was as important as the code fixes. A "helpful" `op service-account ratelimit` polling loop will not consume billable quota, but a "helpful" `op vault list` retry will.

## What this incident is a good example of

- **Read the docs end-to-end the first time you bind to a new SaaS rate limit.** Per-token vs per-account scope is not the kind of thing you want to discover by hitting it.
- **Retry loops are the amplifier in any rate-limit incident.** Most quotas are survivable if every consumer backs off cleanly. ESO's non-configurable 6-minute retry turned a soft hourly nudge into an exhausted daily cap in 4 hours.
- **A free observability call is gold.** `op service-account ratelimit` was the single most valuable command I learned that day.

## Postscript: the recurrence

This incident has a sequel. On 2026-05-02 the same daily cap was exhausted again, by a different code path: the dynamic Proxmox inventory plugin calls `op read` per fork at inventory resolution time, which the secret cache cannot intercept because it loads before the cache is on the path. That story is in [2026-05-02 op rate-limit recurrence](2026-05-02-op-ratelimit-recurrence.md).
