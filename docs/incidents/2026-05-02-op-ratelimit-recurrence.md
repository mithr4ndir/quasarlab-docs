# 2026-05-02: 1Password rate-limit recurrence, the dynamic-inventory bypass

**Date:** 2026-05-02
**Severity:** S2. Same blast radius as the [2026-04-18 incident](2026-04-18-1password-rate-limit.md), worse because it was a recurrence the structural fixes had been supposed to prevent.

## Symptom

The 1Password Grafana panel showed `account read_write USED: 1000/1000`, REMAINING 0, RESET 5 hours from now. ESO had silently been retrying for the entire afternoon. The user noticed because their "1pass quota dashboard, no data" prompt and the alerting was getting noisy.

## What I expected

After the 04-18 work, I expected the steady-state ansible consumption to be ~40 to 90 reads/day, which is 4-9% of the daily cap. So either the structural fixes were not in effect, or there was a code path the cache was not catching, or a rogue caller was running.

## What I found

Two issues compounding, plus one that turned out to be its own bug.

### Issue 1: the 04-18 mitigation got reverted

`ansible-proxmox.timer` and `ansible-security.timer` were both **enabled and firing** when I looked. The 04-18 runbook explicitly said to keep them off until secrets are vaulted. I do not have a clean explanation for how they came back on; the most charitable theory is that an unattended re-run of the bootstrap play put them back. The lesson is: "I disabled it once" is not durable. If the proper fix is structural, the operational mitigation needs an Ansible task that holds the disabled state, not just a one-shot `systemctl disable`.

### Issue 2: `op_ratelimit_collector` had never actually been applied

The role was referenced in `playbooks/cmd_center.yml`, the role files existed in the repo, but the **systemd unit, the script, and the textfile under `/var/lib/node_exporter/textfiles/op_quota.prom` did not exist on the host.** The dashboard panels for `onepassword_ratelimit_*` had no data source even when the cap was healthy. The role had been added to the playbook but the playbook had not been re-applied since the role was added, and there is no separate "install collectors" play or scheduled drift check.

This is a class of bug: **role-in-playbook plus playbook-not-applied is silent drift**. There is no detection unless something else trips first.

### Issue 3 (separate): the collector script over-gated on the killswitch

The original `op-quota-collector.sh.j2` exited early with `success=0 reason=killswitch` if the kill-switch was active. This was a defensive over-application: per the 04-18 work, `op service-account ratelimit` is a **free** control-plane call. The killswitch gate was blinding the dashboard during exactly the incidents the collector exists to observe. Patched on the host and in the role template to skip the kill-switch gate on the rate-limit query specifically (still source the lib so the file can trip the lock if it sees a bad response, protecting other callers that *do* make billable calls).

## What I did, very carefully

The temptation during a rate-limit incident is to debug interactively, which means more `op` calls, which extends the rolling 24h window. Every interactive call was a self-inflicted delay.

So I installed the role outputs **manually** from the local repo working tree, with no further `op` calls and without re-running ansible (which itself would re-resolve the dynamic Proxmox inventory and burn more quota even while the cap was already at 0):

```bash
# 1. Stop the timers, durably this time:
sudo systemctl stop ansible-proxmox.timer ansible-security.timer
sudo systemctl disable --now ansible-proxmox.timer ansible-security.timer

# 2. Install the role's flat files by hand:
sudo mkdir -p /usr/local/lib/op-quota-collector
sudo install -o root -g root -m 0755 \
  ansible-quasarlab/roles/op_ratelimit_collector/files/parse.py \
  /usr/local/lib/op-quota-collector/parse.py
sudo install -o root -g root -m 0755 \
  ansible-quasarlab/roles/op_ratelimit_collector/files/op-killswitch.sh \
  /usr/local/lib/op-quota-collector/op-killswitch.sh

# 3. Render op-quota-collector.sh.j2 manually, with the killswitch gate
#    dropped (Issue 3), and install it + service + timer:
# (rendered files written by hand into ./rendered_*)

sudo install -o root -g root -m 0755 ./rendered_collector.sh \
  /usr/local/bin/op-quota-collector.sh
sudo install -o root -g root -m 0644 ./rendered_service \
  /etc/systemd/system/op-quota-collector.service
sudo install -o root -g root -m 0644 ./rendered_timer \
  /etc/systemd/system/op-quota-collector.timer
sudo systemctl daemon-reload
sudo systemctl enable --now op-quota-collector.timer

# 4. Prime once so /metrics has fresh values immediately:
sudo -u ladino /usr/local/bin/op-quota-collector.sh
```

Verified `/metrics` exposed `onepassword_ratelimit_*` gauges immediately after. The dashboard, which had been blank, started showing the live cap state.

## A side fix that landed in the same session

The `apt list --upgradable` count on hosts was showing 4 pending upgrades while `unattended_upgrades_pending_security` was 0. Cause: the four upgrades were all from third-party repos (Kubernetes, HashiCorp, Vector, Wazuh) that do not carry a `-security` pocket suffix in their `Release` files, so they are correctly excluded from the security-only gauge. Added `apt_upgrades_pending_total` as a separate gauge so the MOTD count and the Prometheus metric agree, with the security-only gauge preserved as a stricter signal. Same role, `unattended_upgrades`.

## Open follow-ups

Local-only at the time of the incident; not yet branched, not yet PR'd:

1. PR for the `op-quota-collector.sh.j2` killswitch-bypass fix.
2. PR for the `update-metrics.sh.j2` `apt_upgrades_pending_total` gauge plus a Grafana panel update on the relevant dashboard.
3. Phase 1 of the 2026-04-22 secrets-IaC rollout (`ansible-quasarlab#124`) is still **the** structural root-cause fix. Until it lands, the proxmox/security ansible timers cannot be safely re-enabled.
4. After the cap recovered, the local `1p-killswitch` lock would still be active until 24 hours after the trip. Remove manually once `op service-account ratelimit` shows REMAINING > 0.

## What this incident is a good example of

- **Recurrence is the truth-teller.** The 04-18 work was the right diagnosis but the wrong scope: it caught the ESO retry storm and the explicit `op read` calls in the playbook body, but missed the fork-time dynamic inventory call that bypasses the env-cache entirely.
- **"I disabled the timer" is not durable.** Operational mitigations need a code expression (a task that asserts disabled) or they will silently come back.
- **Role files vs role applied is an invisible gap.** Adding a role to a playbook does not run the playbook. There is no "show me the diff between role expected files and host actual state" command in vanilla Ansible; you have to build it. Adding that as a CI drift check is one of the more valuable things I could do next.
- **Killswitches need a call-class concept.** The shared kill-switch was written assuming all `op` calls are billable; the assumption is wrong for the control-plane subset (`service-account ratelimit`, possibly others). Future shared killswitches should accept a `--allow-free-calls` opt-out for this exact reason.
