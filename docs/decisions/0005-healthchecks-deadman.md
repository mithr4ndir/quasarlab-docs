# ADR 0005: External deadman switch via Healthchecks.io

**Status:** Accepted
**Date:** 2026-04-15 (post-2026-04-13 incident)

## Context

Alertmanager ships with a built-in alert called `Watchdog` that fires every cycle, by design. The intent is that you route it to an external observer that pages you when the pings **stop arriving**. If the entire alerting pipeline goes down (Alertmanager, Prometheus, the cluster, the network, the lab's power), the external observer is the only thing that notices.

In the pre-incident config, `Watchdog` was routed to `null`. So Alertmanager fired the alert into the bit bucket, and there was no external observer. During the [2026-04-13 alerting blackout](../incidents/2026-04-13-alerting-blackout-cascade.md), that meant a 9-hour silence with no page.

## Decision

Route `Watchdog` to a Healthchecks.io check via Alertmanager's webhook receiver. The Healthchecks.io URL is supplied to the cluster as an ExternalSecret, the URL itself stored as a 1Password Secure Note (item id `zr7mrafo64zpf7usc2ah5nso2a` in the Infrastructure vault).

Configuration:

- Healthchecks.io ping schedule: every minute, with a 10-minute grace period (Alertmanager group_wait + group_interval headroom).
- HC.io email notifications **on**. SMS notifications optional but currently on.
- The receiver in Alertmanager has `send_resolved: false` so it sends pings only on alerts firing, not on resolution. This avoids HC.io interpreting a resolution-only burst as a successful ping when Watchdog itself has been silenced.

## Considered

- **Self-hosted deadman (e.g. my own polling service on a friend's VPS).** More work to maintain, same external-vantage property. Defer.
- **Push to Discord directly via a separate webhook with no proxy.** Discord is where I already live, but Discord is also subject to the same "alert chain inside the cluster" failure mode if the proxy is the only path. HC.io is genuinely external, with email + SMS that do not depend on any of the lab's infrastructure.
- **Pager service like PagerDuty or Opsgenie.** Heavier than the lab needs, contractually, monetarily, and operationally.

## Consequences

- **The single most load-bearing observability link in the lab is now external.** Healthchecks.io itself going down is a real failure mode, but it is a failure mode I cannot do anything about, and the worst case is "I don't get paged for *this specific minute window*", not "I never get paged."
- **The HC.io URL is a credential.** It must be stored in 1Password and pulled via ExternalSecret. Not in Git, not in plaintext.
- **The grace period is a real tuning parameter.** Too short and routine Alertmanager group-window jitter pages me; too long and a real outage delays the page. 10 minutes works for the lab's setup.
- **`send_resolved: false` is non-obvious but important.** Otherwise Alertmanager's resolution traffic interleaves with the "is alive" pings in a way that confuses the deadman.

## Validation

After this ADR was implemented, I ran a planned 11-minute Alertmanager-down window to confirm the HC.io email arrived. It did, within a minute of the grace period elapsing. The pipeline now tests itself any time I do planned maintenance on Alertmanager.

## Correction and update, 2026-09-20

**This ADR was accurate about the design and wrong about the deployment, and
the gap was invisible for five months.**

On [2026-09-19](../incidents/2026-09-19-nfsv4-callback-deadlock.md) the
alerting pipeline was dead for about eleven hours and no page arrived. The
cause was not that the deadman failed. It was configured so loosely that
eleven hours was well inside normal:

| | Documented above | Actually deployed |
|---|---|---|
| Ping cadence | every minute | every 2 minutes |
| **Time before it alerts** | **10 minutes** | **48 hours** |
| Notification channel | email | email only, to a secondary inbox |

The pings really were arriving every couple of minutes, and the check really
did go red. The **alerting** threshold was 48 hours, so an eleven-hour total
alerting outage never came close to triggering it.

This is the same failure this lab keeps meeting in new costumes. The Wazuh
SIEM was `active` for four months while indexing nothing. A drive-error guard
was vacuous because `grep -c ... || echo 0` yields `"0\n0"`. Here, a
two-minute heartbeat that cannot alert for two days looks like tight
monitoring on every dashboard and is worth nothing for any outage shorter
than a weekend. **Check cadence is not detection time.**

### An unresolved contradiction, left visible on purpose

The Validation section above records a planned 11-minute Alertmanager-down
window that produced an email "within a minute of the grace period elapsing".
That is **not compatible** with a 48-hour threshold. Either the threshold was
10 minutes when that test ran and was widened later, or the validation did
not demonstrate what it recorded.

Which one it was is not established, and it is left here rather than quietly
tidied away, because either answer carries a lesson worth more than the
tidying. If it drifted: **a validation is a measurement of one moment, not a
property of the system**, and nothing re-checked it for five months. If it
never worked: a green test result was recorded without confirming what
actually caused the email.

### What it is now

Corrected by the owner on 2026-09-20:

- Heartbeat still checked every 2 minutes.
- **Alert evaluation every 30 minutes, alerting after two consecutive bad
  checks.** Worst-case detection is therefore about an hour, rather than 48.
- **Notifications now go to Discord as well as email.**

The Discord addition matters as much as the timing. The email was going to a
secondary inbox the owner does not read, so even a correctly-timed alert would
have been delivered and not seen. A notification nobody reads is the same
outcome as a notification that never fires, and it fails the same way:
silently, and only during an incident.

An hour is a deliberate trade against flapping, not an attempt to reach the
10 minutes this ADR originally claimed. It is the right shape: layer 3 exists
to catch "the whole pipeline is gone", which is not a condition that resolves
itself in twenty minutes.

### Follow-up

Layer 3 is still the only leg outside the house, and it has still never been
proven to page under the new configuration. Test it deliberately: stop
Alertmanager for a bit over an hour and confirm the **Discord** message
arrives. Until someone does that, treat this ADR's coverage as designed
rather than demonstrated, which is exactly the trap it just fell into.

## Related

- [2026-04-13 alerting blackout](../incidents/2026-04-13-alerting-blackout-cascade.md) is the originating event.
- [ADR 0004 Bake monitoring images](0004-bake-monitoring-images.md) covers the inside-the-cluster half of the same defense-in-depth picture.
