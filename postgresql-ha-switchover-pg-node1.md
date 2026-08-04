# Restoring pg-node1 as Primary (Planned Switchover)

This guide covers a **planned switchover** — reconfiguring the cluster back to its original topology (`pg-node1` = primary, `pg-node2` + `pg-node3` = standbys) after a manual failover exercise promoted `pg-node2` to primary.

> **Failover vs. Switchover**
> - **Failover** = *unplanned*. The primary dies, you promote a standby reactively.
> - **Switchover** = *planned*. Everything is healthy and running; you deliberately swap roles in a controlled, safe order (this guide).
>
> Switchover is safer because you control the timing — you can stop writes, confirm zero replication lag, then swap, instead of racing against an unknown amount of lost data like in a real failover.

---

## 1. Current State vs. Target State

| Node | Current role | Target role |
|---|---|---|
| pg-node1 | stopped / stale old primary | **Primary** |
| pg-node2 | Primary (promoted earlier) | Standby |
| pg-node3 | Standby (of pg-node2) | Standby (of pg-node1) |

**We cannot just start pg-node1 and call it primary again** — its data has diverged from pg-node2 since the earlier failover. It must first be re-cloned as a standby, allowed to fully catch up, and only then promoted in a controlled way. Skipping this creates **split-brain** (two primaries writing independently).

---

## 2. Prerequisites

- `/etc/hosts` on all 3 nodes already contains entries for `pg-node1`, `pg-node2`, `pg-node3` (internal IPs)
- `pg-node2` is currently a healthy, running primary
- `pg-node3` is currently a healthy standby of `pg-node2`
- You have the `replicator` role password from earlier setup

---

## 3. Step 1 — Rejoin pg-node1 as a Standby of pg-node2

### 3.1 Start the VM (if stopped)

```bash
gcloud compute instances start pg-node1 --zone=us-central1-a
```

### 3.2 SSH into pg-node1 and wipe its stale data

```bash
sudo systemctl stop postgresql
sudo rm -rf /var/lib/postgresql/18/main/*
```

### 3.3 Make sure pg-node2 (current primary) allows pg-node1 to replicate

On **pg-node2**:

```bash
sudo grep replicat /etc/postgresql/18/main/pg_hba.conf
```

If there's no line covering pg-node1's IP, add one:

```bash
sudo nano /etc/postgresql/18/main/pg_hba.conf
```

```
host    replication     replicator      10.128.0.4/32          md5
```
(replace with pg-node1's actual internal IP)

```bash
sudo systemctl reload postgresql
```

## Using the `-C` Option with `pg_basebackup`

The `-C` option should **only be used the very first time** you create a standby server, when the **replication slot does not yet exist** on the primary server.

After the replication slot has already been created, **do not use `-C` again**. Instead, simply reference the existing slot using the `-S <slot_name>` option.

> **Rule of Thumb**
>
> - **First standby creation** → Use `-C`
> - **Re-cloning or rebuilding an existing standby** → **Omit `-C`** and use only `-S <slot_name>`

### Verify Whether the Replication Slot Exists

If you're unsure whether the replication slot already exists, check it first on the primary server:

```sql
SELECT slot_name
FROM pg_replication_slots;
```

### Decision Guide

| Replication Slot Status | Action |
|--------------------------|--------|
| **Slot not listed** | ✅ Use `-C` to create the replication slot. |
| **Slot already listed** | ✅ Omit `-C` and use only `-S <slot_name>`. |
### 3.4 Clone from pg-node2 onto pg-node1

Back on **pg-node1**:

```bash
sudo -u postgres pg_basebackup \
  -h pg-node1 \
  -D /var/lib/postgresql/18/main \
  -U replicator \
  -P -v -R -X stream \
  -S pg_node2_slot

sudo systemctl start postgresql
```

### 3.5 Verify pg-node1 is now a standby, catching up

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
# expect: t
```

On **pg-node2** (primary), confirm it sees pg-node1 streaming:

```sql
SELECT client_addr, state, sync_state, replay_lag FROM pg_stat_replication;
```

Wait until `replay_lag` for pg-node1 is `0` (or very close to it) before continuing — this ensures no data will be lost when you promote it.

---

## 4. Step 2 — Perform the Switchover

### 4.1 Stop application writes (important!)

Since this is a **planned** switchover, pause anything writing to the database (your app, or just don't run more `INSERT`s manually) so pg-node1 can fully catch up to zero lag before you cut over.

Re-check lag one more time on pg-node2:

```sql
SELECT client_addr, replay_lag FROM pg_stat_replication WHERE client_addr = '<pg-node1-ip>';
```

Confirm it's `0` or `NULL` (meaning fully caught up).

### 4.2 Promote pg-node1

On **pg-node1**:

```bash
sudo -u postgres /usr/lib/postgresql/18/bin/pg_ctl promote -D /var/lib/postgresql/18/main
```

Verify:

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
# expect: f  (now a primary)
```

`pg-node1` is now a writable primary again.

---

## 5. Step 3 — Demote pg-node2 into a Standby

Now that pg-node1 is primary, pg-node2 must stop being a primary and become a standby instead. Since pg-node2 has been accepting writes independently, the safest approach is a full re-clone (same logic as before).

### 5.1 On pg-node1 (new primary), allow pg-node2 to replicate

```bash
sudo nano /etc/postgresql/18/main/pg_hba.conf
```

```
host    replication     replicator      10.128.0.5/32          md5
```
(pg-node2's IP)

```bash
sudo systemctl reload postgresql
```

Create a replication slot for it:

```sql
SELECT pg_create_physical_replication_slot('pg_node2_slot');
```

### 5.2 On pg-node2, wipe and re-clone from pg-node1

```bash
sudo systemctl stop postgresql
sudo rm -rf /var/lib/postgresql/18/main/*

sudo -u postgres pg_basebackup \
  -h pg-node1 \
  -D /var/lib/postgresql/18/main \
  -U replicator \
  -P -v -R -X stream -C \
  -S pg_node2_slot

sudo systemctl start postgresql
```

### 5.3 Verify

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
# expect: t
```

---

## 6. Step 4 — Repoint pg-node3 to Follow pg-node1

pg-node3 is currently streaming from pg-node2 (which is no longer the primary). Repoint it.

### 6.1 On pg-node1, allow pg-node3 to replicate + create its slot

```bash
sudo nano /etc/postgresql/18/main/pg_hba.conf
```

```
host    replication     replicator      10.128.0.7/32          md5
```
(pg-node3's IP)

```bash
sudo systemctl reload postgresql
```

```sql
SELECT pg_create_physical_replication_slot('pg_node3_slot');
```

### 6.2 On pg-node3, update its connection info and restart

```bash
sudo systemctl stop postgresql

sudo -u postgres psql -c "ALTER SYSTEM SET primary_conninfo = 'host=pg-node1 port=5432 user=replicator password=replicator_pass';"
sudo -u postgres psql -c "ALTER SYSTEM SET primary_slot_name = 'pg_node3_slot';"
```

> Note: `ALTER SYSTEM` requires the server to be running to execute — if it's already stopped, edit `/var/lib/postgresql/18/main/postgresql.auto.conf` directly instead, updating the `primary_conninfo` and `primary_slot_name` lines by hand.

```bash
sudo systemctl start postgresql
```

### 6.3 Verify

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
# expect: t
```

---

## 7. Step 5 — Final Verification

On **pg-node1** (primary), confirm both standbys are streaming:

```sql
SELECT client_addr, state, sync_state, replay_lag FROM pg_stat_replication;
```

Expected: 2 rows — one for pg-node2, one for pg-node3, both `state = streaming`.

Test with data — insert on pg-node1:

```sql
\c hademo
INSERT INTO employees (full_name, email, department, salary, hire_date)
VALUES ('Test Switchover', 'test.switchover@example.com', 'IT', 5000, CURRENT_DATE);
```

Check it appears on both standbys:

```bash
# on pg-node2 and pg-node3
psql -U postgres -d hademo -c "SELECT * FROM employees WHERE email = 'test.switchover@example.com';"
```

---

## 8. Clean Up Stale Replication Slots

Old, unused replication slots silently consume WAL disk space on whichever node still has them. Check each node for leftover slots that no longer have an active standby attached:

```sql
SELECT slot_name, active FROM pg_replication_slots;
```

Drop any slot where `active = f` and you know it's no longer needed:

```sql
SELECT pg_drop_replication_slot('slot_name_here');
```

---

## 9. Final Topology

| Node | Role |
|---|---|
| pg-node1 | **Primary** |
| pg-node2 | Standby |
| pg-node3 | Standby |

You're back to the original topology — cluster fully healthy, all 3 nodes reconciled onto pg-node1's timeline.

---

## 10. Key Lessons for Documentation

- **A node's role is defined by config files (`standby.signal`, `primary_conninfo`), not by which VM was "originally" the primary.** Any node can be primary; the cluster doesn't care about original naming.
- **Never blindly restart a demoted/failed primary without reconfiguring it first** — always re-clone or `pg_rewind` it before starting PostgreSQL, or you risk split-brain.
- **Switchover (planned) is safer than failover (unplanned)** because you can verify zero replication lag before cutting over — no data loss.
- **Every role change requires updating `pg_hba.conf`, replication slots, and `primary_conninfo` on the affected nodes** — this is exactly what Patroni automates for you.
