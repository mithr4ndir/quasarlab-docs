# 2026-04-07: Grafana SQLite to Postgres migration, the bigint-versus-boolean schema mismatch

**Date:** 2026-04-07
**Severity:** S3. Migration was planned, the rollback to SQLite was always available, but the new K8s-resident Grafana sat in `CrashLoopBackOff` for hours while I figured out why.

This is a migration story rather than an incident story, but it shows up here because most of a day was spent untangling a single class of error that pgloader produced silently and that Grafana's Postgres dialect surfaced loudly.

## Context

Grafana had been running on a dedicated VM (`192.168.1.121`, VMID 103) with a SQLite backend since the lab's first incarnation. Migration target: an in-cluster Grafana via `kube-prometheus-stack`, with the database moved to the existing Postgres VM at `192.168.1.123`. LB IP: `192.168.1.229`.

I used `pgloader` to migrate the SQLite database to Postgres. It produced a clean conversion log, no errors. Schema looked plausible. I cut the new Grafana over to the migrated DB and it crashed on first start.

## Symptom

`kubectl logs` of the Grafana container showed three distinct error patterns, all in the SQL-store layer at startup:

```text
pq: invalid input syntax for type bigint: "false"
pq: invalid input syntax for type bigint: "true"
pq: operator does not exist: bigint = boolean
```

The first two appeared during data-source provisioning and SSO settings load. The third appeared during the folder service migration. Pod went CrashLoopBackOff before the HTTP listener even came up.

## Root cause

SQLite is dynamically typed and stores booleans as plain `INTEGER` columns containing `0` or `1`. pgloader saw `INTEGER` and mapped it to Postgres `bigint` (`int8`) for every such column, which is technically correct for any integer SQLite column.

But Grafana's Postgres dialect expects a **`boolean`** column type for the columns it knows are boolean. When it tries to read or write `true` / `false`, Postgres receives a string for an integer column and rejects it. When it tries to compare a column it thinks is boolean against a `boolean` literal, Postgres refuses the comparison entirely.

So the actual values in the rows were fine (just `0` and `1` in `bigint` columns). The **column types** were wrong.

## Fix

Convert the affected columns from `bigint` to `boolean`, table by table.

Pattern, applied per column:

```sql
-- For columns that have a default, drop the default first:
ALTER TABLE <table> ALTER COLUMN <col> DROP DEFAULT;

-- Convert the type:
ALTER TABLE <table> ALTER COLUMN <col> TYPE boolean USING (<col>::int::boolean);

-- Restore a sensible default:
ALTER TABLE <table> ALTER COLUMN <col> SET DEFAULT false;
```

Total scope: 42 boolean-like columns across 28 tables. The full list of tables touched, in case I ever do this again or somebody else does a similar migration:

`dashboard`, `data_source`, `sso_setting`, `"user"`, `alert`, `alert_notification`, `alert_rule`, `alert_rule_version`, `alert_configuration`, `alert_configuration_history`, `api_key`, `correlation`, `dashboard_public`, `dashboard_snapshot`, `data_keys`, `plugin_setting`, `role`, `team`, `team_member`, `temp_user`, `user_auth_token`, `migration_log`, `secret_data_key`, `secret_keeper`, `secret_migration_log`, `secret_secure_value`, `resource_migration_log`, `unifiedstorage_migration_log`.

(The double-quoted `"user"` is necessary because `user` is a reserved word in Postgres.)

For safety, every table got a `backup_<table>` copy before the conversion ran. After Grafana came up cleanly and ran for a few days without surfacing any data anomalies, the backups were dropped.

## Validation

After the schema changes, Grafana started cleanly, dashboards rendered, users could log in, alert rules evaluated. Two follow-up items remained:

1. Alert rules surfaced "data source not found" errors. Cause: data sources had been provisioned against UIDs that did not match the new K8s deployment. Re-provisioning fixed this once the schema layer was correct.
2. The `neocat-cal-heatmap-panel` Grafana plugin was rejected by Grafana 12.4.1 (Angular plugin support was removed in 12.x). Cosmetic only, not blocking. Removed from the plugin list in the Helm values.

## Cleanup

- Old VM `103` on `pve1` destroyed once the new in-cluster Grafana was confirmed stable. Per the [decommissioning checklist](../runbooks/index.md), I:
  - Removed the VM from Prometheus static targets and the Ansible inventory.
  - Removed the host_vars entry.
  - Updated NPM upstream from `192.168.1.121` to `192.168.1.229` (the new LoadBalancer IP).
  - Confirmed no lingering DNS records pointed at `.121`.
- Backup tables (`backup_*`) dropped from the `grafana` Postgres database.

## What this incident is a good example of

- **A migration tool's defaults are not your migration plan.** pgloader's column-type inference was technically correct for SQLite-as-typed-storage, but wrong for Grafana-as-application. Read the schema diff after, even when the tool reports zero errors.
- **The error messages tell the truth.** `invalid input syntax for type bigint: "false"` is the database telling you, very precisely, that it sees a string going into an integer column. The temptation is to chase Grafana config; the answer is in the database log.
- **Have a rollback in your hand the entire time.** The old VM stayed up, untouched, until the new in-cluster Grafana was confirmed stable for several days. The migration was risky-feeling, but never one mistake away from data loss.
