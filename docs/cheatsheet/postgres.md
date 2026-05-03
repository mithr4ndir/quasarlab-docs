# Postgres

## What's running

```bash
pg_lsclusters                              # all local clusters with port and status
sudo -u postgres psql -tAc "SELECT version();"
sudo -u postgres psql -tAc "SHOW config_file; SHOW hba_file;"
```

## Reload config without dropping sessions

```bash
sudo systemctl reload postgresql@16-main   # OS-level reload
# or, from inside Postgres:
sudo -u postgres psql -tAc "SELECT pg_reload_conf();"
```

`reload` only re-reads things marked "sighup" in `postgresql.conf`. Anything tagged "postmaster" needs a real restart. `pg_hba.conf` is sighup, so reload is enough.

## Who is connected, and what are they running

```sql
-- Active queries:
SELECT pid, usename, datname, application_name, state, wait_event_type, wait_event,
       now() - query_start AS runtime, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY runtime DESC NULLS LAST;

-- Connection counts by user/db:
SELECT usename, datname, COUNT(*) FROM pg_stat_activity GROUP BY 1,2 ORDER BY 3 DESC;

-- Locks held that are blocking other queries:
SELECT blocked.pid AS blocked_pid, blocking.pid AS blocking_pid,
       blocked.query AS blocked_query, blocking.query AS blocking_query
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

## psql shortcuts

```text
\l            list databases
\du           list roles
\dn           list schemas
\dt+          tables with size
\d+ <table>   table definition with size and stats
\df+ <fn>     function definition
\timing on    show query duration
\x            expanded output (key:value per row)
\e            edit the previous query in $EDITOR
\watch 5      re-run the previous query every 5 seconds
```

## pg_hba.conf

Order matters. Postgres uses the **first matching** line, no fall-through. Common shape:

```text
# TYPE    DATABASE   USER       ADDRESS             METHOD
local     all        postgres                       peer
local     all        all                            scram-sha-256
host      all        all        127.0.0.1/32        scram-sha-256
host      all        all        ::1/128             scram-sha-256
hostssl   <db>       <user>     <cidr>              scram-sha-256
host      all        all        <node-cidrs>        scram-sha-256
```

Things to remember:

- `host` matches both SSL and non-SSL connections, `hostssl` matches only SSL, `hostnossl` only non-SSL.
- `scram-sha-256` is the modern default. `md5` is legacy and should be migrated.
- Allowed source must be the IP **as Postgres sees it**, after any SNAT. From a K8s pod, that is the **node IP**, not the pod IP.

### What is currently loaded vs what is in the file

`pg_hba.conf` on disk is the source. `pg_hba_file_rules` is the live rule table that actually decides auth. They diverge if there is a syntax error (a bad rule is silently dropped on reload).

```sql
-- Active rules, in evaluation order, with any errors:
SELECT line_number, type, database, user_name, address, auth_method, error
FROM pg_hba_file_rules
ORDER BY line_number;

-- Just the broken ones:
SELECT line_number, error FROM pg_hba_file_rules WHERE error IS NOT NULL;
```

If `error` is non-NULL on a rule, that rule is not in effect. Reloading does not save you from a typo; the typo'd rule just gets dropped and traffic that depended on it starts failing.

## Schema operations from real migrations

### Convert a column to boolean (the Grafana migration pattern)

When a tool like `pgloader` lands integer columns where the application expects boolean, the conversion has to be explicit and per-column.

```sql
-- For columns with a default, drop the default first:
ALTER TABLE <table> ALTER COLUMN <col> DROP DEFAULT;

-- Convert via a USING expression that is unambiguous:
ALTER TABLE <table> ALTER COLUMN <col>
  TYPE boolean USING (<col>::int::boolean);

-- Restore a default:
ALTER TABLE <table> ALTER COLUMN <col> SET DEFAULT false;
```

Always work on a `backup_<table>` copy first if the data matters. See [the Grafana migration incident](../incidents/2026-04-07-grafana-pgloader-migration.md) for the case study.

### Reserved words in table or column names

Postgres has more reserved words than you would guess. `user` is the one that bites people; you have to double-quote it forever.

```sql
SELECT id FROM "user" WHERE login = 'admin';
ALTER TABLE "user" ADD COLUMN something boolean DEFAULT false;
```

If you have a choice when designing a schema, just do not name a table `user`. If you inherit one (Grafana does), the quoting is non-negotiable.

### Find columns of a given type, fast

Useful when surveying a freshly-migrated DB for type mismatches.

```sql
-- Every bigint column in the public schema, with table and column name:
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_schema = 'public'
  AND data_type = 'bigint'
ORDER BY table_name, column_name;
```

## Backup and restore

```bash
# Logical, single-database, custom format (best for selective restores):
sudo -u postgres pg_dump -Fc -d <db> -f /backup/<db>.dump

# Restore to a fresh DB:
sudo -u postgres createdb <db_new>
sudo -u postgres pg_restore -d <db_new> /backup/<db>.dump

# Verify a dump file before trusting it (lists the TOC, no restore):
pg_restore -l /backup/<db>.dump | head
```

For real backups in this lab, `pgBackRest` runs nightly and ships to TrueNAS. The above is for ad-hoc copies and quick restores during incidents.

## Things I reach for when something is wrong

| Question | Answer |
|----------|--------|
| Is Postgres rejecting connections? | `journalctl -u postgresql@16-main --since "30 min ago" \| grep -i "no pg_hba"` |
| What rule denied this connection? | The error message includes user, db, and source IP. Match against `pg_hba.conf` top to bottom. |
| Did a config change actually load? | `SELECT pg_reload_conf();` and check `pg_hba_file_rules` for syntax errors. |
| Are there long-running queries? | `pg_stat_activity` filtered by `state != 'idle'`. |
| Is something blocking? | `pg_blocking_pids()` join above. |
