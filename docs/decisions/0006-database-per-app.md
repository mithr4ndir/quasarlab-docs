# ADR 0006: Database-per-app on the shared Postgres VM

**Status:** Accepted
**Date:** 2026-04-26 (formalized when claude-bridge needed its own database)

## Context

The lab has one Postgres VM at `192.168.1.123` running Postgres 16, with several apps depending on it: Grafana, claude-bridge, Authentik, and one or two smaller things. The question for any new app is: **does it get its own database, or its own schema inside an existing database?**

## Decision

**Database-per-app.** Each application gets its own Postgres database, owned by an app-specific role. Privileges within the database are managed by the app's own migrations, not by the Ansible role that creates the database.

Concretely, claude-bridge owns the `claude_bridge` database. Grafana owns `grafana`. Authentik owns `authentik`. Two roles per app: a runtime role (`<app>_app`) with limited grants, and a deploy-time role (`<app>_migrate`) that owns the database and is the one that runs DDL.

## Considered

- **Schema-per-app inside a single shared database.** Cheaper Postgres footprint, simpler `pg_hba.conf`, but couples backup, restore, and major-version upgrade across all apps. Hard "no" once you have ever had to restore one app to a known-good point in time.
- **Database server per app.** Hard "yes" at scale, hard "no" at homelab scale. Resource overhead per server is unjustified.

Schema-per-app would be the right call inside a multi-tenant SaaS where the apps are all instances of the same product. This lab is not that.

## Consequences

- **Backups, restore, and migrations are per-app.** The nightly `pg_dump` script (`pg_backup.sh.j2` in the postgres role) iterates databases, so adding a new database is one entry in the role defaults, not a code change.
- **`pg_hba.conf` has narrow rules per (db, user)** above the broad K8s pod CIDR fallback. This makes the auth surface explicit and easy to audit. The order matters: narrow rules first (so they match before the fallback), broad rules last.
- **Privilege grants stay in app migrations, not in Ansible.** The Ansible role creates the database and the two roles. Granting `INSERT` on a specific table to the runtime role is the migration's job; Ansible should not know what tables exist. This keeps the app's DB-level privileges versioned alongside its schema.
- **REVOKE CONNECT FROM PUBLIC, not FROM the user.** Postgres grants `CONNECT` to `PUBLIC` implicitly on every database. `REVOKE CONNECT ON DATABASE x FROM <user>` is a no-op because the privilege comes from `PUBLIC`. To deny cross-database CONNECT, REVOKE FROM PUBLIC and explicitly GRANT to the legitimate users. I do this for system DBs (`postgres`, `template1`) so an app role cannot use them as a lateral surface; I have not yet rolled it through to every app DB because it requires coordinated GRANT-to-owner before REVOKE-from-PUBLIC, and that is per-app finicky.
- **`community.postgresql` collection version matters.** The lab's pinned version supports `no_password_changes: true` (boolean) but **not** `update_password: on_create` (enum, added in a later collection version). Both have the same intent (do not rewrite the password when the user already exists). Use the boolean form until the collection version is bumped. Intentional rotations go through a dedicated rotate playbook.

## Related

- [Architecture: external Postgres VM](../architecture/overview.md) for the high-level rationale.
- [Runbook: Postgres rejecting K8s pods](../runbooks/pg-hba-rejecting-k8s-pods.md) for the `pg_hba.conf` failure mode this ADR's structure makes possible to fix narrowly.
- [ADR 0002 External Postgres](0002-external-postgres.md) for the upstream "why a VM at all" decision this builds on.
