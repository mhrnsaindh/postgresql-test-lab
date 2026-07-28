# PostgreSQL Learning Guide & Hands-On Lab (Ubuntu on Google Cloud)

A practical path from "I know SQL a little" to "I can run, query, back up, replicate, and keep PostgreSQL highly available." Every section has a short concept explanation followed by commands you actually run over SSH on a GCE Ubuntu VM.

---

## 0. Lab Setup — Google Cloud VM

You'll want **2 VMs** eventually (for replication/HA practice), but start with 1.

### 0.1 Create the VM(s)

From Cloud Shell or your local `gcloud` CLI:

```bash
gcloud compute instances create pg-node1 \
  --zone=asia-southeast2-a \
  --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --tags=postgres-lab

# second node for replication practice (do this later, when you reach Module 4)
gcloud compute instances create pg-node2 \
  --zone=asia-southeast2-a \
  --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --tags=postgres-lab
```

Open the firewall for Postgres (5432) between your lab VMs only — don't expose it to the internet:

```bash
gcloud compute firewall-rules create allow-postgres-lab \
  --network=default \
  --allow=tcp:5432 \
  --source-tags=postgres-lab \
  --target-tags=postgres-lab
```

SSH in:

```bash
gcloud compute ssh pg-node1 --zone=asia-southeast2-a
```

### 0.2 Install PostgreSQL

```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib postgresql-client
sudo systemctl status postgresql
psql --version
```

Key files/paths you'll touch constantly (Ubuntu puts these under version-numbered dirs, e.g. `16`):

```bash
sudo -u postgres psql -c "SHOW config_file;"   # postgresql.conf location
sudo -u postgres psql -c "SHOW hba_file;"      # pg_hba.conf location
sudo -u postgres psql -c "SHOW data_directory;"
```

Typically:
- Config: `/etc/postgresql/16/main/postgresql.conf`
- Auth rules: `/etc/postgresql/16/main/pg_hba.conf`
- Data: `/var/lib/postgresql/16/main`

Set a password for the `postgres` superuser and create a working role/database:

```bash
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'labpass123';"
sudo -u postgres createuser --interactive --pwprompt learner
sudo -u postgres createdb labdb -O learner
```

---

## Module 1 — PostgreSQL Fundamentals & Data

**Concepts to learn:**
- Architecture: postmaster process, backend processes per connection, shared buffers, WAL (Write-Ahead Log)
- Cluster vs database vs schema vs table (a "cluster" = one data directory, can host many databases)
- Data types: numeric, text, date/time, boolean, `JSON`/`JSONB`, arrays, `UUID`, `ENUM`
- Constraints: `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, `NOT NULL`
- MVCC (Multi-Version Concurrency Control) — how Postgres handles concurrent reads/writes without locking everything

**Lab exercise:**

```bash
psql -U learner -d labdb -h 127.0.0.1 -W
```

```sql
-- create a small schema
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    amount NUMERIC(10,2) NOT NULL CHECK (amount >= 0),
    status TEXT DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT now()
);

-- insert sample data
INSERT INTO customers (name, email, metadata) VALUES
 ('Alice', 'alice@example.com', '{"tier": "gold"}'),
 ('Budi', 'budi@example.com', '{"tier": "silver"}');

INSERT INTO orders (customer_id, amount, status) VALUES
 (1, 150.00, 'paid'),
 (1, 45.50, 'pending'),
 (2, 300.00, 'paid');

-- inspect
\d customers
\d orders
SELECT * FROM customers;
```

**Exercise:** explore JSONB querying:
```sql
SELECT name, metadata->>'tier' AS tier FROM customers WHERE metadata->>'tier' = 'gold';
```

---

## Module 2 — Querying (Beginner → Advanced)

**Concepts:**
- Basic `SELECT`, filtering, sorting, `LIMIT`/`OFFSET`
- Joins: `INNER`, `LEFT`, `RIGHT`, `FULL`, self-joins
- Aggregation: `GROUP BY`, `HAVING`, window functions (`OVER`, `PARTITION BY`)
- Subqueries & CTEs (`WITH`)
- Indexes: B-tree, GIN (for JSONB/arrays), partial and expression indexes
- `EXPLAIN` / `EXPLAIN ANALYZE` for reading query plans

**Lab exercises:**

```sql
-- joins
SELECT c.name, o.amount, o.status
FROM customers c
JOIN orders o ON o.customer_id = c.id
ORDER BY o.amount DESC;

-- aggregation
SELECT c.name, SUM(o.amount) AS total_spent, COUNT(*) AS order_count
FROM customers c
JOIN orders o ON o.customer_id = c.id
GROUP BY c.name
HAVING SUM(o.amount) > 100;

-- window function: running total per customer
SELECT c.name, o.amount,
       SUM(o.amount) OVER (PARTITION BY c.id ORDER BY o.created_at) AS running_total
FROM customers c JOIN orders o ON o.customer_id = c.id;

-- CTE
WITH big_spenders AS (
  SELECT customer_id, SUM(amount) AS total
  FROM orders GROUP BY customer_id HAVING SUM(amount) > 200
)
SELECT c.name, b.total FROM big_spenders b JOIN customers c ON c.id = b.customer_id;
```

**Indexing & query plans:**
```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_customers_metadata ON customers USING GIN (metadata);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 1;
```
Read the output: look for `Seq Scan` (slow, full table read) vs `Index Scan` (fast), and compare cost/actual time before and after adding the index.

---

## Module 3 — Backup: Hot vs Cold, Logical vs Physical

This is the part people mix up most, so here's the map:

| Type | What it means | Tool | DB stays up? |
|---|---|---|---|
| **Cold backup** | Stop Postgres, copy the data directory | `systemctl stop postgresql` + `cp`/`tar` | No |
| **Hot backup (physical)** | Copy data directory *while running*, consistent via WAL | `pg_basebackup` | Yes |
| **Logical backup** | Export SQL statements / data, not raw files | `pg_dump`, `pg_dumpall` | Yes |

### 3.1 Logical backup — `pg_dump` (most common day-to-day)

```bash
# single database, custom format (compressed, supports parallel restore)
pg_dump -U postgres -h 127.0.0.1 -Fc labdb -f /tmp/labdb.dump

# plain SQL (human-readable, good for small DBs / version control)
pg_dump -U postgres -h 127.0.0.1 labdb > /tmp/labdb.sql

# all databases + roles (cluster-wide)
pg_dumpall -U postgres -h 127.0.0.1 > /tmp/full_cluster.sql
```

Restore:
```bash
createdb -U postgres labdb_restore
pg_restore -U postgres -d labdb_restore /tmp/labdb.dump

# or from plain SQL
psql -U postgres -d labdb_restore -f /tmp/labdb.sql
```

**Exercise:** drop a table, restore just that data, verify row counts match.

### 3.2 Cold backup (simple, but requires downtime)

```bash
sudo systemctl stop postgresql
sudo tar -czf /tmp/pg_cold_backup.tar.gz /var/lib/postgresql/16/main
sudo systemctl start postgresql
```
Good for a lab VM, bad for production (any downtime-sensitive service).

### 3.3 Hot physical backup — `pg_basebackup`

This is what real HA/replication setups use as their base. It requires `wal_level = replica` (default is usually fine) and a replication-privileged role.

```bash
# create a replication role
sudo -u postgres psql -c "CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replpass123';"
```

Edit `pg_hba.conf` to allow replication connections (add near the top of the rules):
```
host    replication     replicator      0.0.0.0/0               md5
```
Then reload:
```bash
sudo systemctl reload postgresql
```

Take a hot backup while the server is live:
```bash
mkdir -p /tmp/basebackup
pg_basebackup -h 127.0.0.1 -U replicator -D /tmp/basebackup -Fp -Xs -P
```
`-Xs` streams the WAL needed to make the backup consistent; `-P` shows progress. This base backup is exactly what streaming replicas are built from — which leads into Module 4.

**Exercise:** compare pg_dump vs pg_basebackup restore times on your lab data, and note that pg_basebackup gives you the whole cluster (all databases + roles) while pg_dump is per-database.

---

## Module 4 — Replication (needs `pg_node1` + `pg_node2`)

**Concepts:**
- **Streaming replication**: primary ships WAL records to a standby in near real time
- **Physical replication**: byte-for-byte copy (whole cluster) — what we set up below
- **Logical replication**: replicate specific tables via `CREATE PUBLICATION` / `CREATE SUBSCRIPTION`, cross-version capable, selective
- **Synchronous vs asynchronous**: sync guarantees zero data loss on failover but adds latency; async is faster but can lose the last few transactions

### 4.1 Streaming replication setup

On `pg-node1` (primary), edit `/etc/postgresql/16/main/postgresql.conf`:
```
listen_addresses = '*'
wal_level = replica
max_wal_senders = 5
```

Update `pg_hba.conf` to allow node2's IP:
```
host    replication     replicator      <pg-node2-internal-ip>/32      md5
```

```bash
sudo systemctl restart postgresql
```

On `pg-node2` (standby):
```bash
sudo systemctl stop postgresql
sudo rm -rf /var/lib/postgresql/16/main/*
sudo -u postgres pg_basebackup -h <pg-node1-internal-ip> -U replicator \
  -D /var/lib/postgresql/16/main -Fp -Xs -P -R
sudo systemctl start postgresql
```
The `-R` flag auto-generates `standby.signal` and the primary connection info — that's what puts node2 into standby/recovery mode.

**Verify:**
```bash
# on primary
sudo -u postgres psql -c "SELECT client_addr, state, sync_state FROM pg_stat_replication;"

# on standby
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"   # should return t
```

**Exercise:** insert a row on `pg-node1`, `SELECT` it on `pg-node2` and confirm it appears (replication lag is usually sub-second on a LAN).

### 4.2 Logical replication (selective, table-level)

On primary: `wal_level = logical`, then:
```sql
CREATE PUBLICATION my_pub FOR TABLE customers, orders;
```
On subscriber (a separate, independently-writable database):
```sql
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=<primary-ip> dbname=labdb user=replicator password=replpass123'
  PUBLICATION my_pub;
```

---

## Module 5 — High Availability (HA)

**Concepts:**
- Replication alone isn't HA — you also need **automatic failure detection + failover + client redirection**
- Common building blocks:
  - **Patroni** — the most widely used modern HA manager; uses etcd/Consul/ZooKeeper as a distributed consensus store to elect a leader
  - **repmgr** — simpler, Postgres-native tool for managing replication + manual/automatic failover
  - **pgpool-II / HAProxy / PgBouncer** — connection pooling and routing (send writes to primary, reads to replicas)
  - **Virtual IP / DNS failover** — how clients actually get redirected after a failover
- Key metrics: RPO (Recovery Point Objective — how much data you can afford to lose) and RTO (Recovery Time Objective — how fast you must recover)

**Suggested lab progression** (this is a bigger undertaking — good for once you're comfortable with Modules 1–4):
1. Manually promote `pg-node2` to primary and test client reconnection:
   ```bash
   sudo -u postgres pg_ctl promote -D /var/lib/postgresql/16/main
   ```
2. Install `repmgr` and let it manage the primary/standby relationship + do a *managed* failover.
3. Once comfortable, try Patroni with a 3-node etcd cluster for automatic failover — this mirrors what's used in production.

I can build out a dedicated Patroni or repmgr lab guide with exact commands when you're ready for that stage — it's substantial enough to be its own module.

---

## Suggested Order of Attack

1. Module 1 & 2 (data model + querying) — spend the most time here, it's the foundation
2. Module 3 (backups) — do this before touching replication, since `pg_basebackup` is reused there
3. Module 4 (replication) — spin up the second VM
4. Module 5 (HA) — once replication feels natural

## Quick Reference Cheat Sheet

```bash
sudo systemctl {start|stop|restart|status} postgresql
sudo -u postgres psql                       # connect as superuser locally
psql -U user -d db -h host -W               # connect with password prompt
\l          # list databases
\c dbname   # connect to a database
\dt         # list tables
\d table    # describe table
\du         # list roles
\timing     # toggle query timing
```

---

Want me to turn Module 5 into a full Patroni lab (with etcd cluster setup), or write practice query exercises with a bigger sample dataset to drill joins/window functions/indexing further?
