# 🐘 PostgreSQL Deep-Dive Roadmap (GCP VM Labs + HA/Failover + DB Engineer Cert Prep)

A hands-on roadmap to go from solid PostgreSQL fundamentals → manual HA/failover → Patroni-based HA → preparing for the **Google Cloud Professional Cloud Database Engineer** certification.

> 💡 Best used together with a Google Cloud VM (Compute Engine) so every step is a real hands-on lab, not just theory.

---

## ✅ Step 0: Prerequisites

- [ ] Comfortable with Linux CLI (systemctl, journalctl, ssh, package managers)
- [ ] A GCP project with billing/credits active
- [ ] Basic SQL knowledge
- [ ] Completed (or in parallel with) the [GCP beginner roadmap](./google-cloud-beginner-roadmap.md)

---

## ✅ Step 1: PostgreSQL Fundamentals (Theory + Local Practice)

- [ ] Install PostgreSQL on a Compute Engine VM (Debian/Ubuntu)
- [ ] Understand PostgreSQL architecture: postmaster, backend processes, shared buffers, WAL
- [ ] Databases, schemas, roles & privileges
- [ ] `psql` basics — connecting, `\d`, `\l`, `\du`
- [ ] Data types, indexes (B-tree, GIN, GiST), constraints
- [ ] `EXPLAIN` / `EXPLAIN ANALYZE` — query planning basics
- [ ] Backup & restore: `pg_dump`, `pg_dumpall`, `pg_restore`
- [ ] `pg_hba.conf` and `postgresql.conf` — authentication & tuning basics

📚 Free resources:
- [PostgreSQL Official Docs](https://www.postgresql.org/docs/)
- [PGExercises](https://pgexercises.com/) — interactive SQL practice
- [postgresqltutorial.com](https://www.postgresqltutorial.com/)

---

## ✅ Step 2: WAL, Replication & Backup Concepts

- [ ] Understand **WAL (Write-Ahead Logging)**
- [ ] Physical vs logical replication
- [ ] Streaming replication (async vs sync)
- [ ] Replication slots
- [ ] `pg_basebackup`
- [ ] Point-in-time recovery (PITR)
- [ ] Tools: **pgBackRest**, **Barman**, **WAL-G**

---

## ✅ Step 3: Lab — Manual HA & Failover on GCP VMs

Build this on **2–3 Compute Engine VMs** (1 primary, 1–2 replicas):

1. [ ] Provision 3 VMs in the same VPC/subnet (e.g. `pg-primary`, `pg-replica1`, `pg-replica2`)
2. [ ] Install PostgreSQL on all nodes
3. [ ] Configure **streaming replication** (primary → replica) using `pg_basebackup`
4. [ ] Verify replication with `pg_stat_replication` on primary and `pg_stat_wal_receiver` on replica
5. [ ] Simulate a **primary failure** (stop the service / VM)
6. [ ] Perform a **manual failover**: promote a replica with `pg_ctl promote` or `pg_promote()`
7. [ ] Repoint your application / connection string to the new primary
8. [ ] Rebuild the old primary as a new replica (rejoin the cluster)
9. [ ] (Optional) Add a **floating/virtual IP** using `keepalived` to reduce app reconnection pain

🎯 Goal: understand *why* manual failover is risky (data loss risk, human delay, split-brain) — this motivates Step 4.

---

## ✅ Step 4: Lab — Automated HA with Patroni

1. [ ] Install **Patroni** + **etcd** (or Consul/ZooKeeper) as the Distributed Configuration Store (DCS) on your GCP VMs
2. [ ] Configure a 3-node **etcd cluster** for consensus
3. [ ] Configure Patroni on each PostgreSQL node (`patroni.yml`)
4. [ ] Start the cluster and confirm leader election: `patronictl list`
5. [ ] Simulate a primary node failure (kill VM / stop service) and observe **automatic failover**
6. [ ] Add **HAProxy** in front of the cluster to route traffic to the current leader automatically
7. [ ] Test **switchover** (planned, zero/low-downtime): `patronictl switchover`
8. [ ] Explore Patroni's REST API health checks (`/leader`, `/replica`, `/health`)
9. [ ] (Optional) Compare with **pg_auto_failover** as an alternative tool

📚 Resources:
- [Patroni Official Docs](https://patroni.readthedocs.io/)
- [Patroni GitHub repo](https://github.com/patroni/patroni) (examples, Docker Compose demo)
- [pg_auto_failover docs](https://pg-auto-failover.readthedocs.io/)

---

## ✅ Step 5: Bring It Into GCP-Managed Services

Now compare your manual/Patroni HA experience against **Google's managed HA**:

- [ ] Deploy a **Cloud SQL for PostgreSQL** instance with HA (regional) enabled
- [ ] Trigger a **manual failover** in Cloud SQL and observe the differences vs. your DIY setup
- [ ] Explore **read replicas** and **cross-region replicas** in Cloud SQL
- [ ] Explore **AlloyDB for PostgreSQL** — its HA model, read pools, and columnar engine
- [ ] Compare: self-managed HA (Patroni) vs. Cloud SQL HA vs. AlloyDB HA — write a short doc for yourself on trade-offs (cost, control, complexity)

---

## ✅ Step 6: Advanced PostgreSQL Topics (Cert-Relevant)

- [ ] Connection pooling: **PgBouncer**
- [ ] Partitioning (range, list, hash)
- [ ] Vacuum, autovacuum tuning, bloat management
- [ ] Monitoring: `pg_stat_statements`, Cloud Monitoring integration
- [ ] Security: SSL/TLS connections, Cloud SQL IAM auth, encryption at rest/in transit
- [ ] Migration tools: **Database Migration Service (DMS)**, `pgloader`, logical replication-based migration
- [ ] Disaster recovery planning (RPO/RTO)

---

## ✅ Step 7: Google Cloud Professional Database Engineer — Exam Prep Map

The exam (as of 2026) covers 4 domains — map your study accordingly:

| Domain | Weight | What to focus on |
|---|---|---|
| Design scalable & HA cloud database solutions | 30–35% | HA/DR patterns, choosing Cloud SQL vs AlloyDB vs Spanner vs Bigtable vs Firestore |
| Manage a solution spanning multiple database solutions | 20–25% | Monitoring, performance tuning, cost optimization, security/IAM |
| Migrate data solutions | 15–20% | DMS, homogeneous/heterogeneous migration, minimal-downtime strategies |
| Deploy scalable & HA databases in Google Cloud | 25–30% | Hands-on provisioning of Cloud SQL, AlloyDB, replicas, failover config |

📌 Notes:
- No official exam code; 2 hours, ~50 questions, $200 USD, valid 2 years
- Recommended: 5 years overall DB/IT experience, 2 years hands-on GCP DB experience (guideline, not a hard requirement)
- Official exam guide: https://services.google.com/fh/files/misc/professional_cloud_database_engineer_exam_guide_english.pdf

---

## ✅ Step 8: Recommended Labs (Cloud Skills Boost + Other Platforms)

### On Google Cloud Skills Boost (use your credits here)
- [ ] **"Google Cloud SQL for PostgreSQL"** individual labs
- [ ] **Quest: "Cloud SQL for PostgreSQL"**
- [ ] **Quest: "Database Migration"** (using DMS)
- [ ] **"Explore AlloyDB for PostgreSQL"** lab
- [ ] **"Manage PostgreSQL databases on Compute Engine"** lab
- [ ] **Professional Cloud Database Engineer** learning path (official)

### On Other Platforms
- [ ] **Katacoda-style / KillerCoda** — free interactive Patroni/etcd scenarios (search "PostgreSQL HA" on killercoda.com)
- [ ] **Percona Blog & Labs** — deep technical write-ups on Patroni, pgBackRest, replication
- [ ] **Crunchy Data Blog** — excellent PostgreSQL HA and Kubernetes (via `postgres-operator`) content
- [ ] **A Cloud Guru / Pluralsight** — PostgreSQL & GCP database courses
- [ ] **Udemy** — "Google Professional Cloud Database Engineer" practice exam courses
- [ ] **GitHub — patroni/patroni** `examples/` folder — spin up a local Docker Compose HA demo without any cloud cost
- [ ] **PostgreSQL Exercises (pgexercises.com)** — for SQL fluency
- [ ] **CNCF/Kubernetes route (optional, advanced):** try the **Zalando Postgres Operator** or **CloudNativePG** on a GKE cluster to see HA managed via Kubernetes

---

## 📝 Progress Tracker

| Step | Status |
|---|---|
| Step 0: Prerequisites | ⬜ |
| Step 1: PostgreSQL Fundamentals | ⬜ |
| Step 2: WAL, Replication & Backup Concepts | ⬜ |
| Step 3: Manual HA & Failover Lab | ⬜ |
| Step 4: Patroni Automated HA Lab | ⬜ |
| Step 5: Cloud SQL / AlloyDB Managed HA | ⬜ |
| Step 6: Advanced PostgreSQL Topics | ⬜ |
| Step 7: Certification Exam Domain Mapping | ⬜ |
| Step 8: Additional Labs | ⬜ |

---

*Fork this, track your progress, and pair it with your `google-cloud-beginner-roadmap.md` for a full learning path on GitHub.*
