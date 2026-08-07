# PostgreSQL High Availability with Patroni on Google Cloud (Compute Engine)

This guide walks through building a 3-node PostgreSQL High Availability (HA) cluster using **Patroni** + **etcd** on **3 Google Compute Engine VMs**, following standard PostgreSQL HA manual practices.

---

## 1. Architecture Overview

```
                ┌─────────────────────────────┐
                │        Client / App         │
                └───────────────┬─────────────┘
                                │
                     (connects via HAProxy / Patroni REST API)
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                        │
 ┌──────▼──────┐        ┌───────▼──────┐         ┌───────▼──────┐
 │  pg-node1   │        │   pg-node2    │         │   pg-node3   │
 │ PostgreSQL  │◄──────►│  PostgreSQL   │◄───────►│  PostgreSQL  │
 │  Patroni    │  repl  │   Patroni     │  repl   │   Patroni    │
 │   etcd      │        │    etcd       │         │    etcd      │
 └─────────────┘        └───────────────┘         └──────────────┘
   LEADER (initial)        REPLICA                    REPLICA
```

- **3 VMs**, each running: **PostgreSQL**, **Patroni** (HA orchestrator), and **etcd** (distributed consensus store — 3 nodes gives quorum).
- Patroni monitors PostgreSQL health on each node and automatically promotes a replica to leader if the current leader fails.
- etcd stores cluster state/leader lock so all nodes agree on who the current leader is.

| Node | Hostname | Role (initial) | Internal IP (example) |
|------|----------|-----------------|------------------------|
| VM 1 | pg-node1 | Leader | 10.128.0.10 |
| VM 2 | pg-node2 | Replica | 10.128.0.11 |
| VM 3 | pg-node3 | Replica | 10.128.0.12 |

---

## 2. Prerequisites

- A Google Cloud project with billing enabled
- `gcloud` CLI installed and authenticated (`gcloud init`)
- Basic familiarity with Linux/SSH

---

## 3. Step 1 — Create 3 VMs on Google Compute Engine

```bash
gcloud compute instances create pg-node1 pg-node2 pg-node3 \
  --zone=asia-southeast2-a \
  --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --tags=postgres-ha \
  --network=default \
  --subnet=default
```

> Adjust `--zone` to your preferred region. `e2-medium` (2 vCPU / 4GB RAM) is enough for a lab/demo setup.

Get the internal IPs assigned to each VM:

```bash
gcloud compute instances list --filter="name~'pg-node'" \
  --format="table(name,networkInterfaces[0].networkIP)"
```

Note down the 3 internal IPs — you'll need them for `/etc/hosts` and Patroni/etcd config.

---

## 4. Step 2 — Configure Firewall Rules

Allow the ports needed between nodes: PostgreSQL (5432), Patroni REST API (8008), etcd client/peer (2379/2380), and HAProxy (5000, optional).

```bash
gcloud compute firewall-rules create allow-postgres-ha \
  --network=default \
  --allow=tcp:22,tcp:5432,tcp:8008,tcp:2379,tcp:2380,tcp:5000 \
  --source-tags=postgres-ha \
  --target-tags=postgres-ha \
  --description="Allow PostgreSQL/Patroni/etcd traffic between HA nodes"
```

---

## 5. Step 3 — Set Up Hostnames on All Nodes

SSH into **each** of the 3 VMs:

```bash
gcloud compute ssh pg-node1 --zone=asia-southeast2-a
```

On **all 3 nodes**, edit `/etc/hosts` (use the real internal IPs from Step 1):

```bash
sudo tee -a /etc/hosts <<EOF
10.128.0.10 pg-node1
10.128.0.11 pg-node2
10.128.0.12 pg-node3
EOF
```

---

## 6. Step 4 — Install Dependencies (run on all 3 nodes)

### 6.1 Install PostgreSQL 16

```bash
sudo apt update
sudo apt install -y wget gnupg2 curl lsb-release

sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

sudo apt update
sudo apt install -y postgresql-16 postgresql-contrib-16

# Stop the default service — Patroni will manage PostgreSQL's lifecycle instead
sudo systemctl stop postgresql
sudo systemctl disable postgresql
```

### 6.2 Install etcd

```bash
ETCD_VER=v3.5.15
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd.tar.gz
sudo tar xzvf /tmp/etcd.tar.gz -C /tmp
sudo mv /tmp/etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/
sudo mkdir -p /var/lib/etcd /etc/etcd

latest version
ETCD_VER=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest | grep '"tag_name"' | cut -d '"' -f4)
echo "Using etcd version: $ETCD_VER"
```

### 6.3 Install Patroni

```bash
sudo apt install -y python3-pip python3-dev libpq-dev
sudo pip3 install patroni[etcd3] psycopg2-binary
```

---

## 7. Step 5 — Configure etcd Cluster

On **each node**, create `/etc/etcd/etcd.conf.yml` (change `name` and `initial-advertise-peer-urls` per node — example below is for `pg-node1`):

```yaml
name: pg-node1
data-dir: /var/lib/etcd
listen-peer-urls: http://0.0.0.0:2380
listen-client-urls: http://0.0.0.0:2379
initial-advertise-peer-urls: http://pg-node1:2380
advertise-client-urls: http://pg-node1:2379
initial-cluster: pg-node1=http://pg-node1:2380,pg-node2=http://pg-node2:2380,pg-node3=http://pg-node3:2380
initial-cluster-token: postgres-ha-etcd-cluster
initial-cluster-state: new
```

> Repeat on `pg-node2` and `pg-node3`, only changing `name`, `initial-advertise-peer-urls`, and `advertise-client-urls` to match that node's hostname.

Create the systemd service (all 3 nodes):

```bash
sudo tee /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd distributed key-value store
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd --config-file /etc/etcd/etcd.conf.yml
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now etcd
```

Verify cluster health (run on any node once all 3 are started):

```bash
ETCDCTL_API=3 etcdctl --endpoints=http://pg-node1:2379,http://pg-node2:2379,http://pg-node3:2379 endpoint health
```

---

## 8. Step 6 — Configure Patroni

Create `/etc/patroni.yml` on **each node**. Example for `pg-node1` (change `name` and `connect_address`/`listen` IPs on each node accordingly):

```yaml
scope: postgres-ha-cluster
namespace: /db/
name: pg-node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: pg-node1:8008

etcd3:
  hosts: pg-node1:2379,pg-node2:2379,pg-node3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        wal_keep_size: 128MB

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator 10.128.0.0/20 md5
    - host all all 10.128.0.0/20 md5
    - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: pg-node1:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  authentication:
    replication:
      username: replicator
      password: "replicator_pass"
    superuser:
      username: postgres
      password: "postgres_pass"
  parameters:
    unix_socket_directories: '/var/run/postgresql'

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

> ⚠️ Replace `replicator_pass` / `postgres_pass` with strong passwords, and adjust the `10.128.0.0/20` CIDR to match your VPC subnet range.

Fix ownership and create systemd service (all 3 nodes):

```bash
sudo mkdir -p /var/lib/postgresql/16/main
sudo chown -R postgres:postgres /var/lib/postgresql/16/main
sudo mv /etc/patroni.yml /etc/patroni.yml
sudo chown postgres:postgres /etc/patroni.yml

sudo tee /etc/systemd/system/patroni.service <<EOF
[Unit]
Description=Patroni PostgreSQL HA
After=etcd.service
Requires=etcd.service

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now patroni
```

---

## 9. Step 7 — Verify the Cluster

Install `patronictl` helper (already included with the `patroni` pip package) and check status:

```bash
patronictl -c /etc/patroni.yml list
```

Expected output (example):

```
+ Cluster: postgres-ha-cluster (uninitialized) -----+---------+---------+----+-----------+
| Member   | Host                | Role    | State   | TL | Lag in MB |
+----------+---------------------+---------+---------+----+-----------+
| pg-node1 | pg-node1:5432       | Leader  | running |  1 |           |
| pg-node2 | pg-node2:5432       | Replica | running |  1 |         0 |
| pg-node3 | pg-node3:5432       | Replica | running |  1 |         0 |
+----------+---------------------+---------+---------+----+-----------+
```

---

## 10. Step 8 — (Optional) HAProxy for Client Connection Routing

Since the leader can change during failover, apps should connect through something that always points to the current leader. Install HAProxy on each node (or a dedicated 4th node if you have one):

```bash
sudo apt install -y haproxy
```

`/etc/haproxy/haproxy.cfg` (append):

```
listen postgres_leader
    bind *:5000
    option httpchk GET /master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-node1 pg-node1:5432 maxconn 100 check port 8008
    server pg-node2 pg-node2:5432 maxconn 100 check port 8008
    server pg-node3 pg-node3:5432 maxconn 100 check port 8008
```

```bash
sudo systemctl restart haproxy
```

Applications now connect to `<any-node>:5000` and HAProxy automatically routes to whichever node Patroni currently reports as `/master`.

---

## 11. Step 9 — Test Automatic Failover

On the current leader node (e.g. `pg-node1`):

```bash
sudo systemctl stop patroni
```

On another node, watch Patroni promote a replica automatically:

```bash
watch patronictl -c /etc/patroni.yml list
```

Within a few seconds you should see one of `pg-node2` / `pg-node3` promoted to `Leader`. Restart Patroni on `pg-node1` — it will rejoin the cluster as a replica automatically:

```bash
sudo systemctl start patroni
```

---

## 12. Step 10 — Dummy Data (100 Records)

Connect to the current leader (find it via `patronictl list`) and create a sample table populated with 100 dummy rows.

```bash
psql -h pg-node1 -U postgres -d postgres
```

```sql
-- Create sample database & table
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

-- Generate 100 dummy rows deterministically
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

-- Verify: should return 100
SELECT COUNT(*) FROM employees;

-- Preview the data
SELECT * FROM employees ORDER BY id LIMIT 20;
```

This produces exactly **100 realistic dummy employee records** (unique emails, varied departments, salaries, and hire dates) — enough to visibly test replication.

### Verify Replication to the Other Nodes

On a **replica** node (e.g. `pg-node2`):

```bash
psql -h pg-node2 -U postgres -d hademo -c "SELECT COUNT(*) FROM employees;"
```

You should see the same count of **100** rows, confirming streaming replication is working. Any writes on the leader propagate automatically — replicas are read-only until a failover promotes one of them.

---

## 13. Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| `patronictl list` shows no cluster | etcd not reachable | Check `systemctl status etcd`, firewall rules on port 2379/2380 |
| Patroni won't start PostgreSQL | Wrong `data_dir` ownership | `chown -R postgres:postgres /var/lib/postgresql/16/main` |
| Replica stuck in "starting" | Replication user/password mismatch | Verify `pg_hba.conf` entries and `replicator` credentials in `patroni.yml` |
| Failover doesn't happen | `nofailover: true` tag set | Check the `tags` section of `patroni.yml` on each node |
| Split-brain / two leaders | etcd cluster not achieving quorum | Ensure exactly 3 (odd number) etcd members are healthy |

---

## 14. References

- Patroni documentation: https://patroni.readthedocs.io
- PostgreSQL High Availability, Load Balancing, and Replication documentation: https://www.postgresql.org/docs/current/high-availability.html
- etcd documentation: https://etcd.io/docs/

---

## Repository Structure Suggestion

```
postgresql-ha-patroni-gcp/
├── README.md                  # this file
├── config/
│   ├── etcd.conf.yml
│   ├── patroni.yml
│   └── haproxy.cfg
├── scripts/
│   └── seed_dummy_data.sql
└── diagrams/
    └── architecture.png
```
