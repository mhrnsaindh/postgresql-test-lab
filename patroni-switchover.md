# Switchover with Patroni (`patronictl switchover`)

After testing failover by stopping Patroni on the leader node, you may want to restore the original node as leader once it rejoins as a healthy replica. Unlike the manual HA setup — which requires re-cloning, editing `pg_hba.conf`, and repointing standbys by hand — Patroni handles this with a single command.

## When to use this

- The original leader (e.g. `pg-node1`) was stopped to simulate a failure, another node was automatically promoted (e.g. `pg-node2`), and `pg-node1` has since rejoined the cluster as a healthy replica with zero lag.
- You now want to move leadership back to `pg-node1` in a controlled, safe way.

## Step 1 — Confirm current cluster state

```bash
patronictl -c /etc/patroni.yml list
```

Example output before switchover:

```
+ Cluster: postgres-ha-cluster ----+-----+------------+-----+
| Member   | Host     | Role    | State   | TL | Lag |
+----------+----------+---------+---------+----+-----+
| pg-node1 | pg-node1 | Replica | running |  2 |   0 |
| pg-node2 | pg-node2 | Leader  | running |  2 |     |
| pg-node3 | pg-node3 | Replica | running |  2 |   0 |
+----------+----------+---------+---------+----+-----+
```

Confirm `pg-node1`'s `Lag` is `0` before proceeding — this ensures no data loss during the switchover.

## Step 2 — Run the switchover (interactive)

```bash
patronictl -c /etc/patroni.yml switchover
```

Patroni will prompt you through the process:

```
Master [pg-node2]:                       # press Enter to confirm current leader
Candidate ['pg-node1', 'pg-node3'] []:   # type: pg-node1
When should the switchover take place (e.g. 2026-08-08T12:00 )  [now]:   # press Enter for "now"
Are you sure you want to switchover cluster postgres-ha-cluster, demoting current master pg-node2? [y/N]: y
```

## Step 3 — What happens automatically

1. Patroni stops accepting new writes on the current leader (`pg-node2`)
2. Waits for the target candidate (`pg-node1`) to fully catch up (zero lag)
3. Promotes `pg-node1` to leader
4. Demotes `pg-node2` to a replica and automatically reconfigures it to follow the new leader
5. `pg-node3` automatically continues streaming from whichever node is now the leader — no manual repointing needed

## Step 4 — Verify

```bash
patronictl -c /etc/patroni.yml list
```

Expected result:

```
+ Cluster: postgres-ha-cluster ----+-----+------------+-----+
| Member   | Host     | Role    | State   | TL | Lag |
+----------+----------+---------+---------+----+-----+
| pg-node1 | pg-node1 | Leader  | running |  3 |     |
| pg-node2 | pg-node2 | Replica | running |  3 |   0 |
| pg-node3 | pg-node3 | Replica | running |  3 |   0 |
+----------+----------+---------+---------+----+-----+
```

## Non-interactive version (for scripts/automation)

```bash
patronictl -c /etc/patroni.yml switchover --master pg-node2 --candidate pg-node1 --force
```

## Why this matters (compared to manual HA)

| Task | Manual HA (by hand) | Patroni |
|---|---|---|
| Detect current leader / lag | Manual `pg_stat_replication` query | `patronictl list` |
| Promote candidate | `pg_ctl promote` / `pg_promote()` | Automatic |
| Demote old leader | Manual stop + full re-clone with `pg_basebackup` | Automatic |
| Repoint other replicas | Manual `primary_conninfo` edit on each replica | Automatic |
| Update `pg_hba.conf` / replication slots | Manual, per node | Automatic |
| Total commands needed | ~15–20 across multiple nodes | 1 command |

This is the clearest practical demonstration of what Patroni automates versus the manual HA process documented earlier in this repo.
