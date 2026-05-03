# Incidents

Real failures, walked from symptom to fix. These are the most useful pages on this site, both for me (so I do not redo the work) and for anyone evaluating how I think about systems.

Format for every incident:

1. **Symptom** as I first noticed it.
2. **First hypothesis** and what made me revise it.
3. **Real root cause**.
4. **Blast radius**, including downstream effects that looked like separate issues.
5. **Fix**, including what I did **not** change and why.
6. **Follow-ups** that survived the incident.

## Index, newest first

- [2026-05-03 Grafana down + MetalLB withdrawing IPs (one issue, not two)](2026-05-03-grafana-metallb-pg_hba.md)
- [2026-05-02 1Password rate-limit recurrence, the dynamic-inventory bypass](2026-05-02-op-ratelimit-recurrence.md)
- [2026-04-19 Bitnami public images quietly disappeared](2026-04-19-bitnami-images-removed.md)
- [2026-04-18 1Password daily rate-limit, the per-account bucket](2026-04-18-1password-rate-limit.md)
- [2026-04-18 etcd bloat, control-plane instability](2026-04-18-etcd-bloat-rca.md)
- [2026-04-13 PVE self-fence, 9-hour alerting blackout, full remediation](2026-04-13-alerting-blackout-cascade.md)
- [2026-04-07 Grafana SQLite to Postgres migration, bigint vs boolean](2026-04-07-grafana-pgloader-migration.md)
