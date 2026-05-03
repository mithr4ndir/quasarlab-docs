# ADR 0002: Postgres on a dedicated VM, not in-cluster

**Status:** Accepted
**Date:** 2025-08

## Context

A handful of in-cluster apps need a relational store: Grafana, the claude-bridge HITL service, the trading dashboard, and a few hobby projects. I want to keep stateful data outside the K8s blast radius and make backups, restore drills, and major-version upgrades boring.

## Decision

A single dedicated VM running Postgres 16 (Debian 12, PGDG packages, `192.168.1.123`). Each app gets its own role, database, and `pg_hba.conf` rule. Backups via `pgBackRest` to a TrueNAS dataset, with a periodic restore drill into a throwaway VM.

## Considered

- **CloudNativePG (CNPG).** Strong operator, real HA, in-cluster. Rejected for two reasons: I trust myself more with `pg_dump` and `pgBackRest` against a familiar VM than I trust myself to debug an in-cluster cluster at 2am, and CNPG cannot help if K8s itself is the problem (which it has been).
- **Stolon, Zalando-postgres-operator.** Same shape as CNPG, less momentum, more legwork for me.
- **A managed Postgres (RDS, Supabase).** Defeats the purpose of a homelab and adds external dependency.

## Consequences

- **The data layer survives a cluster rebuild.** This is the main win. I have rebuilt the K8s cluster more than once and no app data was at risk.
- **`pg_hba.conf` becomes a load-bearing piece of network policy.** When CNI behavior around SNAT changes, this file has to know. The 2026-05-03 incident is the obvious example.
- **No automatic HA.** A single Postgres VM means an OS-level failure on `192.168.1.123` takes the dependent apps down. Acceptable for a lab. If I ever care, the upgrade path is to add a synchronous replica on a second Proxmox node and front it with Patroni or pgpool. Not now.
- **Ansible owns the box.** All changes (`pg_hba.conf`, `postgresql.conf`, package versions) belong in `ansible-quasarlab/labctl-runs/postgres`. Manual edits during incidents are allowed but must be back-ported the same day.
