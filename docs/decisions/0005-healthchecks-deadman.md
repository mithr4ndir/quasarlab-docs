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

## Related

- [2026-04-13 alerting blackout](../incidents/2026-04-13-alerting-blackout-cascade.md) is the originating event.
- [ADR 0004 Bake monitoring images](0004-bake-monitoring-images.md) covers the inside-the-cluster half of the same defense-in-depth picture.
