# PostgreSQL High Availability — Manual Setup (Streaming Replication + Manual Failover)

This guide is the **"learn it manually first"** companion to the Patroni guide. Here you'll build the same 3-VM PostgreSQL cluster on Google Compute Engine, but configure **streaming replication and failover completely by hand** — no Patroni, no etcd, no automation. This is the best way to actually understand what Patroni automates for you later.

---

## 1. Architecture Overview

```
 ┌─────────────┐        async streaming        ┌───────────────┐
 │  pg-node1   │ ─────────────────────────────► │   pg-node2    │
 │  PRIMARY    │                                 │   STANDBY 1   │
 └──────┬──────┘                                 └───────────────┘
        │             async streaming
        └───────────────────────────────────────► ┌───────────────┐
                                                    │   pg-node3    │
                                                    │   STANDBY 2   │
                                                    └───────────────┘
```

- **pg-node1** — Primary (accepts writes)
- **pg-node2 / pg-node3** — Standbys (read-only, replicate from primary via WAL streaming)
- Failover here is **manual**: if the primary dies, *you* decide which standby becomes the new primary and *you* run the commands to promote it and repoint the others.

| Node | Hostname | Role | Internal IP (example) |
|------|----------|------|------------------------|
| VM 1 | pg-node1 | Primary | 10.128.0.10 |
| VM 2 | pg-node2 | Standby | 10.128.0.11 |
| VM 3 | pg-node3 | Standby | 10.128.0.12 |

> Reuse the same 3 VMs / firewall rules from the Patroni guide if you already created them — just make sure PostgreSQL is stopped and not currently managed by Patroni before starting this exercise (`sudo systemctl stop patroni etcd` and `sudo systemctl disable patroni etcd` if applicable).

---

## 2. Prerequisites

- 3 Compute Engine VMs (Ubuntu 22.04) with internal networking, as created in the previous guide
- PostgreSQL 16 installed on all 3 nodes
- `/etc/hosts` entries on all 3 nodes pointing to each other (see previous guide, Step 5)
- Firewall allowing port `5432` between the 3 nodes

If you haven't installed PostgreSQL yet, on **all 3 nodes**:

```bash
sudo apt update
sudo apt install -y wget gnupg2 lsb-release
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt update
sudo apt install postgresql postgresql-contrib -y
sudo systemctl enable postgresql
```

---

## 3. Step 1 — Configure the Primary (pg-node1)

### 3.1 Create a replication role

```bash
sudo -u postgres psql
```

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replicator_pass';
\q
```

### 3.2 Allow standbys to connect (`pg_hba.conf`)

```bash
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

Add at the bottom (adjust IPs/CIDR to match your VPC subnet):

```
host    replication     replicator      10.128.0.11/32          md5
host    replication     replicator      10.128.0.12/32          md5
host    all             all             10.128.0.0/20           md5
```

### 3.3 Let PostgreSQL listen on the network (`postgresql.conf`)

```bash
sudo nano /etc/postgresql/16/main/postgresql.conf
```

Set/uncomment:

```
listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 512MB
hot_standby = on
```

### 3.4 Create physical replication slots (one per standby — recommended so the primary retains WAL each standby needs)

```bash
sudo -u postgres psql
```

```sql
SELECT pg_create_physical_replication_slot('pg_node2_slot');
SELECT pg_create_physical_replication_slot('pg_node3_slot');
\q
```

### 3.5 Restart PostgreSQL

```bash
sudo systemctl restart postgresql
```

---

## 4. Step 2 — Clone the Primary onto Each Standby

## before do this step first setting the network:

## 1. cat /etc/hosts
## 2. sudo tee -a /etc/hosts <<EOF
   10.128.0.10 pg-node1  (use your actually internal IP)
   10.128.0.11 pg-node2  (use your actually internal IP)
   10.128.0.12 pg-node3  (use your actually internal IP)
   EOF ##
##  Why /etc/hosts needs to be set on every single machine?
   /etc/hosts is a local file — it only affects name resolution on the machine it's stored on. It's not a shared or centralized service like DNS; each VM has its own
   private copy, and they don't sync with each other automatically. 

   

On **pg-node2** and **pg-node3**, stop PostgreSQL and wipe the empty data directory, then clone the primary using `pg_basebackup`:

```bash
sudo systemctl stop postgresql
sudo rm -rf /var/lib/postgresql/16/main/*

sudo -u postgres pg_basebackup \
  -h pg-node1 \
  -D /var/lib/postgresql/16/main \
  -U replicator \
  -P -v -R -X stream -C \
  -S pg_node2_slot   # use pg_node3_slot when running this on pg-node3
```

Flags explained:

| Flag | Meaning |
|---|---|
| `-D` | target data directory |
| `-U replicator` | replication user created in Step 1 |
| `-P -v` | show progress |
| `-X stream` | stream WAL during the backup (avoids gaps) |
| `-R` | **auto-generates `standby.signal` and `primary_conninfo`** in `postgresql.auto.conf` — this is what makes the node start up as a standby |
| `-C -S <slot>` | create and use the replication slot on the primary |

You'll be prompted for the `replicator` password (`replicator_pass`).

Start PostgreSQL on the standby:

```bash
sudo systemctl start postgresql
```

Repeat identically on **pg-node3** (using `pg_node3_slot`).

---

## 5. Step 3 — Verify Replication

On the **primary** (pg-node1):

```sql
SELECT client_addr, state, sync_state, replay_lag FROM pg_stat_replication;
```

Expected: 2 rows, one per standby, `state = streaming`.

On a **standby** (pg-node2 or pg-node3):

```sql
SELECT pg_is_in_recovery();  -- should return 't' (true)
```

```bash
cat /var/lib/postgresql/16/main/postgresql.auto.conf   # should show primary_conninfo
ls /var/lib/postgresql/16/main/standby.signal           # file should exist
```

---

## 6. Step 4 — Test Replication with Dummy Data

Use the same dummy dataset from the Patroni guide to confirm replication is actually working end-to-end.

On the **primary**:

```sql
CREATE DATABASE hademo;
\c hademo

CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL,
    department VARCHAR(50) NOT NULL,
    salary NUMERIC(10,2) NOT NULL,
    hire_date DATE NOT NULL
);

INSERT INTO employees (full_name, email, department, salary, hire_date)
SELECT
    fn || ' ' || ln AS full_name,
    lower(fn || '.' || ln || i::text || '@example.com') AS email,
    dept AS department,
    (4000 + (i * 137) % 6000)::numeric(10,2) AS salary,
    (DATE '2018-01-01' + ((i * 11) % 2500) * INTERVAL '1 day')::date AS hire_date
FROM generate_series(1, 100) AS s(i)
CROSS JOIN LATERAL (
    SELECT
        (ARRAY['James','Michael','Robert','Maria','Linda','Patricia','John','David',
               'Susan','Karen','Andi','Budi','Citra','Dewi','Eka','Fajar','Gita','Hadi',
               'Indah','Joko'])[1 + (i % 20)] AS fn,
        (ARRAY['Smith','Johnson','Williams','Brown','Jones','Garcia','Miller','Davis',
               'Rodriguez','Martinez','Wijaya','Santoso','Pratama','Kurniawan','Saputra',
               'Hidayat','Setiawan','Gunawan','Utami','Wibowo'])[1 + ((i * 3) % 20)] AS ln
) AS names
CROSS JOIN LATERAL (
    SELECT (ARRAY['Engineering','Sales','Marketing','Finance','HR',
                  'Operations','Support','Product','IT','Legal'])[1 + (i % 10)] AS dept
) AS d;

SELECT COUNT(*) FROM employees;  -- 100
```

On **both standbys**:

```bash
psql -U postgres -d hademo -c "SELECT COUNT(*) FROM employees;"
```

You should see **100** on both — confirming the data streamed from the primary automatically. Try inserting one more row on the primary and re-check the count on the standbys after a second or two.

---

## 7. Step 5 — Manual Failover (the core of this exercise)

This is what Patroni does automatically. Here, you do it by hand.

### 7.1 Simulate a primary failure

On **pg-node1**:

```bash
sudo systemctl stop postgresql
```

### 7.2 Choose which standby becomes the new primary

Pick the standby with the **least replication lag** (ideally 0). Check on each standby before promoting:

```sql
SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();
```

The one that is most caught-up (or check `pg_stat_replication.replay_lag` on the primary beforehand, if it was still reachable) should be promoted. Let's say **pg-node2** is chosen.

### 7.3 Promote pg-node2 to primary

```bash
sudo -u postgres /usr/lib/postgresql/16/bin/pg_ctl promote -D /var/lib/postgresql/16/main
```

Or, from `psql` on pg-node2:

```sql
SELECT pg_promote();
```

Verify it's no longer in recovery:

```sql
SELECT pg_is_in_recovery();  -- should now return 'f' (false)
```

`pg-node2` is now a writable primary.

### 7.4 Repoint the remaining standby (pg-node3) to the new primary

On **pg-node3**:

```bash
sudo systemctl stop postgresql
```

Edit the replication connection info so it follows the new primary instead of the old one:

```bash
sudo -u postgres psql -c "ALTER SYSTEM SET primary_conninfo = 'host=pg-node2 port=5432 user=replicator password=replicator_pass';"
```

(Or edit `postgresql.auto.conf` directly and update the `primary_conninfo` line.)

Make sure a replication slot for pg-node3 exists on the **new primary** (pg-node2):

```sql
SELECT pg_create_physical_replication_slot('pg_node3_slot');
```

Update `primary_slot_name` too if you're using slots:

```bash
sudo -u postgres psql -c "ALTER SYSTEM SET primary_slot_name = 'pg_node3_slot';"
```

Start pg-node3 back up:

```bash
sudo systemctl start postgresql
```

Confirm it's streaming from pg-node2:

```sql
-- on pg-node2 (new primary)
SELECT client_addr, state FROM pg_stat_replication;
```

### 7.5 (Later) Bring the old primary back as a standby

The old `pg-node1` **cannot simply be restarted** as-is — its timeline has diverged from the new primary's, since it may have accepted writes the new primary never saw (or its WAL history now conflicts). Two options:

**Option A — Full re-clone (simplest, safest for learning):**

```bash
# on pg-node1
sudo systemctl stop postgresql
sudo rm -rf /var/lib/postgresql/16/main/*
sudo -u postgres pg_basebackup -h pg-node2 -D /var/lib/postgresql/16/main \
  -U replicator -P -v -R -X stream -C -S pg_node1_slot
sudo systemctl start postgresql
```

**Option B — `pg_rewind` (faster, avoids a full re-copy, only works if `wal_log_hints = on` or checksums are enabled and no WAL needed for rewind was recycled):**

```bash
sudo systemctl stop postgresql
sudo -u postgres /usr/lib/postgresql/16/bin/pg_rewind \
  --target-pgdata=/var/lib/postgresql/16/main \
  --source-server="host=pg-node2 user=replicator password=replicator_pass dbname=postgres"
# then create standby.signal and set primary_conninfo manually, same as pg_basebackup -R would
sudo systemctl start postgresql
```

---

## 8. Step 6 — Client Connection Routing (also manual)

Without Patroni + HAProxy health checks, your application needs to know which node is currently the primary. Two simple manual approaches while learning:

1. **libpq multi-host connection string** (PostgreSQL 10+), which tries hosts in order and uses `target_session_attrs=read-write` to automatically pick whichever one is currently writable:

   ```
   postgresql://app_user:app_pass@pg-node1,pg-node2,pg-node3:5432/hademo?target_session_attrs=read-write
   ```

   This is the closest you can get to "automatic" client failover without any HA orchestrator — libpq itself tries each host and connects to whichever accepts writes.

2. **Manual DNS/hosts update** — after promoting a new primary, update a shared DNS record (or `/etc/hosts` on app servers) to point `pg-primary.internal` at the new primary's IP, and have your app always connect to that name.

---

## 9. What You Just Learned (and What Patroni Automates)

| Manual step you just did | What Patroni does for you automatically |
|---|---|
| Watching for primary failure | Continuous health checks via REST API |
| Deciding which standby to promote | Leader election via etcd, picks the most caught-up node |
| Running `pg_ctl promote` / `pg_promote()` | Automatic promotion |
| Editing `primary_conninfo` on remaining standbys | Automatic re-pointing of replicas |
| Re-cloning or `pg_rewind`-ing the old primary | Automatic rejoin as a replica |
| Updating client connection target | HAProxy + Patroni REST API `/master` health check |

Now that you've done all of this by hand, the Patroni guide should make a lot more sense — every YAML setting in `patroni.yml` maps directly to one of the manual steps above.

---

## 10. Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| `pg_basebackup` hangs or fails | Firewall blocking 5432, or `pg_hba.conf` not reloaded | `sudo systemctl reload postgresql`, check firewall rule |
| Standby won't start, no `standby.signal` | Forgot `-R` flag on `pg_basebackup` | Manually create `touch standby.signal` and set `primary_conninfo` via `ALTER SYSTEM` |
| `pg_stat_replication` empty on primary | Replication slot not created, or standby not connecting | Check `sudo -u postgres psql -c "\dRs+"` on primary for lag/slot state |
| Old primary won't rejoin after failover | Diverged WAL timelines | Use Option A (re-clone) or Option B (`pg_rewind`) from Step 5.5 |
| `pg_rewind` fails with "target server needs to use either data checksums or wal_log_hints" | Cluster wasn't initialized with checksums | Re-`initdb` with `--data-checksums`, or just re-clone with `pg_basebackup` instead |

---

## 11. References

- PostgreSQL Streaming Replication docs: https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION
- `pg_basebackup`: https://www.postgresql.org/docs/current/app-pgbasebackup.html
- `pg_rewind`: https://www.postgresql.org/docs/current/app-pgrewind.html
- High Availability, Load Balancing, and Replication: https://www.postgresql.org/docs/current/high-availability.html
