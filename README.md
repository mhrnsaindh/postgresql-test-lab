# PostgreSQL High Availability on Google Cloud — Manual & Automated (Patroni)

A hands-on portfolio project building PostgreSQL High Availability (HA) on Google Compute Engine — first fully **manual**, then **automated with Patroni + etcd** — to understand exactly what HA orchestration tools solve under the hood.

## Why this project

Most tutorials jump straight to Patroni. This project deliberately does it the hard way first: manual streaming replication, manual failover, manual switchover — hitting (and fixing) real issues like split-brain risk, replication slot conflicts, and listener misconfiguration — before automating it. The goal was to actually understand *why* each Patroni config value exists, not just copy-paste a working setup.

## Architecture

- 3x Google Compute Engine VMs (Ubuntu, PostgreSQL 18)
- Streaming replication (1 primary + 2 standbys)
- Manual failover/switchover procedures
- Automated failover via **Patroni** + **etcd** (3-node consensus)
- Optional HAProxy for client connection routing

## Contents

| Guide | Description |
|---|---|
| [`postgresql-ha-manual.md`](./postgresql-ha-manual.md) | Manual streaming replication setup, manual failover, and manual switchover — no automation |
| [`postgresql-ha-switchover-pg-node1.md`](./postgresql-ha-switchover-pg-node1.md) | Planned switchover walkthrough: restoring the original primary after a failover exercise |
| [`postgresql-ha-patroni-gcp.md`](./postgresql-ha-patroni-gcp.md) | Full automated HA setup using Patroni + etcd on the same 3-VM architecture |

## What's covered

- Setting up streaming replication with `pg_basebackup` and replication slots
- Manual primary promotion (`pg_ctl promote` / `pg_promote()`)
- Repointing standbys to a new primary
- Rejoining a failed primary safely (avoiding split-brain)
- Planned switchover vs. unplanned failover
- Automating all of the above with Patroni's leader election (etcd) and REST API health checks
- Testing with a reproducible 100-row dummy dataset

## Key lessons

- A node's role (primary/standby) is defined by its local config, not by which VM it originally was — any node can become primary.
- Restarting a failed primary without reconfiguring it first risks running two primaries at once (split-brain).
- Replication slots must be uniquely named per standby and cleaned up carefully during rebuilds.
- Patroni essentially automates the exact manual sequence documented here: health checks, leader election, promotion, and standby re-pointing.

## Tech stack

`PostgreSQL 18` · `Patroni` · `etcd` · `HAProxy` · `Google Compute Engine` · `Ubuntu`

---

📄 Full write-ups linked above. Feedback and suggestions welcome — this is an ongoing learning project.
