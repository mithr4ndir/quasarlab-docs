# 2026-09-12: 1Password quota burn, two reads for every SSH session

**Date:** 2026-09-12 (spikes), root cause found and fixed 2026-09-13
**Severity:** S3. The spikes took the account cap to 60%. My own investigation
later took it to 92%. Nothing ran dry.

Every SSH session into command-center1 (the host that runs Ansible, and one of
its own targets) cost two 1Password reads, because `~/.bashrc` ran an uncached
`op read` for every command, interactive or not. A run of the command center
playbook opens dozens of SSH sessions, so each run against that host cost 88 to
324 reads. Tracing Ansible could not see it: the reads ran in shells that `sshd`
started, outside Ansible's process tree, even though both were on the same machine.

This is the third 1Password quota page on this site, after
[2026-04-18](2026-04-18-1password-rate-limit.md) and
[2026-05-02](2026-05-02-op-ratelimit-recurrence.md). Each found and fixed a real
consumer. None found this one.

## Symptom

The 50% quota alert, at `info` severity then, was firing again. At 23:00 UTC:

```
TYPE       ACTION        LIMIT    USED    REMAINING    RESET
account    read_write    1000     599     401          3 hours from now
```

The cap is 1000 reads per 24-hour window, shared by every token on the account.
On this account the window rolls over once a day, at about 02:51 UTC. My target
is to stay under half, so 599 was a failure, just not an outage.

The day's spend was not a drip. It was three bursts (first 5-minute collector
sample showing each jump):

| Time (UTC) | Delta | Running total |
|---|---|---|
| 04:15 | +160 | 249 |
| 05:15 | +160 | 413 |
| 14:30 | +94 | 549 |
| everything else | small steps of +2, several per hour | 185 |

Three bursts were 414 of 599 reads, about 70% of the day.

## First hypotheses, and what made me revise them

### The cache should have made this impossible

Secrets are read through a file-backed cache with a TTL, the Proxmox inventory
token moved into ansible-vault in
[ansible-quasarlab#129](https://github.com/mithr4ndir/ansible-quasarlab/pull/129),
and the hourly Ansible timers had been measured at a handful of reads per fire.

### Ruled out by measurement

Each of these had caused quota burn before, so I measured each one.

- **The hourly timers.** They fired every hour all day and cost about 5 reads an
  hour, not bursts of 160. The cache held 12 secrets and was working.
- **External Secrets Operator.** Only 7 ExternalSecrets exist, its pods had
  restarted at 03:08 rather than 04:15, and comparing token fingerprints, without
  printing either token, showed ESO uses a **different** service account. The
  hourly token-level counter on command-center1 peaked in exactly the burst
  hours. The burn was on this host's own token.
- **Someone running Ansible by hand.** ARA, which records runs from the timer
  wrappers and from login shells, showed 262 runs in 24 hours, all from the pinned
  automation checkout.

So the burn was this host's own token, in bursts, and not the timers, ESO, or any
recorded run. The collector could say how much quota was left, but not who spent it.

### Attribution, then an accidental reproduction

Rather than patch the likeliest suspect again, I built an `op` wrapper that
records redacted invocation metadata and exports a per-consumer counter
([ansible-quasarlab#151](https://github.com/mithr4ndir/ansible-quasarlab/pull/151)).
Review caught that its argument redaction was a denylist that let field
assignments like `notesPlain=` through, so it became an allowlist before anything
was deployed ([#153](https://github.com/mithr4ndir/ansible-quasarlab/pull/153)).

The wrapper is installed by `cmd_center.yml`. A one-host `--check` run of that
playbook, bracketed by the free quota status call, moved the counter from 607 to
695: **88 reads for a dry run**. Three more identical runs cost about 88 each.

### 88 reads, zero `op` processes

I got the next step wrong twice.

- **Dynamic inventory**, the cause of an earlier incident. Bracketing
  `ansible-inventory --list` on its own cost 0 reads. #129 had fixed it.
- **The `onepassword_cli` role**, which makes an uncached call. The play failed on
  its first role (a `kubectl` package held with `apt-mark hold`) and never reached
  it. Still 88.

Then I traced it. An `op` wrapper added to the controller's own `PATH` logged zero
calls during an 88-read run. `strace -f -e trace=execve` on the whole playbook
found zero `op` execs and 44 execs of `/usr/bin/ssh`. 88 reads, 44 SSH
sessions: exactly 2 each.

## Real root cause

One SSH session, bracketed by the free quota call:

```bash
before=$(op service-account ratelimit | awk '/account/{print $4}')
ssh command-center1 true
after=$(op service-account ratelimit | awk '/account/{print $4}')
echo "cost: $((after - before)) reads"
# cost: 2 reads
```

The top of `~/.bashrc` exported the Proxmox API token with an uncached `op read`,
and a comment explained that it had been placed there on purpose:

```bash
# 1Password / Proxmox token for Ansible (must be before non-interactive guard)
if [ -z "${PROXMOX_TOKEN_SECRET:-}" ] && [ -n "${OP_SERVICE_ACCOUNT_TOKEN:-}" ] \
   && command -v op >/dev/null 2>&1; then
    export PROXMOX_TOKEN_SECRET="$(op read 'op://<vault>/<item>/<field>' 2>/dev/null || true)"
fi

# If not running interactively, don't do anything
case $- in
    *i*) ;;
      *) return;;
esac
```

Bash reads `~/.bashrc` even for a non-interactive command when `sshd` starts it,
so this ran for every command Ansible sent over SSH. Measured on its own, a single
`op read` of a named reference costs 2 account reads. Ansible multiplexes many
sessions over one SSH connection, but each session starts a new shell, and each
shell paid again. A later dry run that got 28 tasks deep instead of 8 cost
**324 reads**, consistent with 162 sessions at 2 each.

A root-owned sibling, `/etc/profile.d/op-ansible-env.sh`, did the same thing for
login shells, which the plain commands Ansible sends over SSH do not start.
Neither file was in the Ansible repo.

One thing this does **not** explain: what caused the bursts at 04:15, 05:15 and
14:30 on 2026-09-12. They were on this host's token, but no recorded run in those
hours differs from any other hour. If they were SSH sessions, the fix removed their
cost. If not, the attribution wrapper should name them next time.

### Why earlier investigations missed it

- **Tracing Ansible could not see it.** The reads ran in shells that `sshd`
  started, not in Ansible's process tree, even though Ansible was connecting to
  the host it runs on. Every tool I pointed at Ansible truthfully reported that it
  never called `op`.
- **Nothing owned the files.** No playbook managed `~/.bashrc` or the profile
  script, so no code review ever saw them.
- **The vault migration never knew about them.** #129 replaced the wrapper
  scripts' `op read` path with a vault helper and closed the inventory bypass. A
  second copy of the same export lived in shell startup files nobody had listed.
  The migration landed on 2026-05-02; the shell copy kept running for four more
  months.

The earlier pages found real consumers (the ESO retry loop, uncached reads in
playbooks, the inventory plugin) and fixed them. I never measured how much of their
burn this read caused. One unpublished recurrence was blamed on hand-run
`ansible-playbook`, which was likely the right trigger and the wrong mechanism:
those runs were expensive because every SSH session they opened paid this toll.

## Blast radius

- **Every SSH session into command-center1** cost 2 reads, whether it came from
  Ansible, a script, or a person.
- **Every `cmd_center.yml` run** cost 88 to 324 reads depending on how far it got.
  Other playbooks that SSH into the host paid in proportion: the hourly drip of
  about 5 reads went to zero after the fix, while the timers kept running.
- **My own investigation** spent roughly 1,000 reads across two daily windows and
  took the counter to 917. Most of that was full playbook dry runs. The `ssh true`
  test that settled it cost 2.
- **The last apply tripped the kill switch by accident.** The wrappers scan
  playbook output for "Too many requests", and the `--diff` of the newly installed
  kill switch script contained that phrase in a comment. That paused the Ansible
  timers until the quota check cleared it, with 1Password refusing nothing
  ([#160](https://github.com/mithr4ndir/ansible-quasarlab/issues/160)).

### Found along the way

Fixing this meant getting `cmd_center.yml` to finish a full run, which it had not
done since 2026-03-16 (every completion since then was limited to a single tag).
That exposed three unrelated problems, each now fixed:

- **HashiCorp rotated its apt signing key around 2026-09-10.** From 2026-09-09
  23:21 UTC, `apt-get update` failed on command-center1, so `vm_baseline.yml`,
  `monitoring.yml` and `wazuh.yml` failed on that host on every timer run for
  almost four days. The key is now pinned by fingerprint, so the next rotation
  fails loudly ([#157](https://github.com/mithr4ndir/ansible-quasarlab/pull/157)).
- **A retired Elastic apt repository was still configured on 9 hosts.** Its
  removal had been added to the two roles for the three hosts where it was noticed
  (wazuh, pve and pve2), and the other 9 never run either role. It now lives in a role every host runs, and
  was removed from all 9 on 2026-09-13
  ([#157](https://github.com/mithr4ndir/ansible-quasarlab/pull/157)).
- **Three user-level systemd tasks were broken.** The play runs with
  `become: yes`, so facts are gathered as root and `ansible_user_uid` is `0`. The
  tasks built `/run/user/0/bus` from it, and `| default(1000)` never fired because
  the fact was defined, just wrong. The play aborted there, unnoticed because
  nobody ran it, and the spec-workflow dashboard had only ever been started by hand
  ([#158](https://github.com/mithr4ndir/ansible-quasarlab/pull/158)).

## Fix

[ansible-quasarlab#155](https://github.com/mithr4ndir/ansible-quasarlab/pull/155)
brings both shell startup files under Ansible management:

- Neither calls `op` any more. A task fails the play if an `op read` reappears in
  `~/.bashrc`.
- The token is decrypted from ansible-vault through the same helper the timer
  wrappers use, and only in interactive shells. The shells Ansible opens over SSH
  never needed it.
- `scripts/run-cmd-center.sh` gives manual runs of this playbook the same guard
  rails as the timers.

#151 also locked the secret cache against concurrent misses and raised its TTL to
48 hours.

Same commands, before and after:

| Measurement | Before | After |
|---|---|---|
| `ssh command-center1 true` | 2 reads | **0** |
| `cmd_center.yml --check` | 324 reads | **0** |
| Full `cmd_center.yml` run | none completed since 2026-03-16 | **0 reads**, 123 ok, 0 failed |

Detection, so the next burst is a query rather than an investigation:

- **Attribution.** The `op` wrapper exports
  `onepassword_op_invocations_total{consumer,caller,unit,subcommand}`, confirmed
  scraped by Prometheus. It went live after the fix, so it did not find this one;
  it exists for the next.
- **Burn rate.** A new alert fires on reads per hour rather than on remaining
  quota ([k8s-argocd#200](https://github.com/mithr4ndir/k8s-argocd/pull/200), with
  a rollover edge case fixed in
  [#202](https://github.com/mithr4ndir/k8s-argocd/pull/202)). Replayed against the
  day's data, it fires at 04:16, the first sample after the 04:15 burst, at 202
  reads in the trailing hour, and peaks at 324 once the 05:15 burst lands. The 50%
  threshold was not crossed until 14:30.
- **The 50% alert is now a warning, not info**, so crossing my own target is no
  longer quiet.

### What I did not change, and why

- **No full apply while the cap was nearly spent.** With about 140 reads left and a
  full run measured at 324, I applied only the shell startup tasks
  (`--tags shell_env`, which cost 60 while fixing itself and left 83), confirmed
  SSH was free, and only then ran the rest.
- **Kept credential values out of telemetry.** I rejected capturing full command
  lines, which would have copied secret arguments into logs. The wrapper keeps
  useful metadata and redacts arguments through an allowlist.
- **Kept scheduling separate.** A reconciliation schedule for configuration that
  touches the cluster deserves its own decision, even though the lack of one is
  how this host drifted.

## Follow-ups

- Stop the kill switch matching text in `--diff` and task output, and confirm the
  timers resume once it clears
  ([#160](https://github.com/mithr4ndir/ansible-quasarlab/issues/160)).
- Identify what caused the three bursts, using the attribution metric the next
  time one happens.
- Decide on a reconciliation schedule for `cmd_center.yml`.
- `cmd_center.yml --check` still fails for reasons unrelated to this incident,
  which makes dry runs less trustworthy:
  [#156](https://github.com/mithr4ndir/ansible-quasarlab/issues/156) and
  [#159](https://github.com/mithr4ndir/ansible-quasarlab/issues/159).

## What this incident is a good example of

- **Measure one operation, not a whole run.** Bracketing a single
  `ssh command-center1 true` with the free quota call turned months of inference
  into arithmetic: 2 reads per session, 44 sessions, 88 reads.
- **A clean trace only covers the tree you traced.** `strace -f` and a wrapper on
  the controller's `PATH` were both correct. The reads ran in shells that `sshd`
  started on the same machine, outside the process tree I was watching.
- **A replacement is finished when every copy of the old thing is gone.** The vault
  migration shipped, but a second copy of the old export lived in files no
  playbook owned, so nothing was ever going to find or remove it. The
  decommissioning checklist I use for VMs applies to configuration too.
- **A fix in the wrong place is not a fleet fix.** The Elastic repo removal was
  merged into roles most hosts never run, and this playbook had not completed a
  full run in six months. Both looked finished from the pull request page.
