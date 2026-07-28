# Module 6 — Monitoring PostgreSQL with Prometheus & Grafana

Continuation of the PostgreSQL learning guide. This module wires up real-time metrics and dashboards for your `pg-node1` (and optionally `pg-node2`) VM.

**Architecture you're building:**

```
PostgreSQL  --(metrics)-->  postgres_exporter  --(scraped by)-->  Prometheus  --(queried by)-->  Grafana
   :5432                        :9187                                :9090                          :3000
```

- **postgres_exporter** — translates Postgres internal stats (`pg_stat_activity`, `pg_stat_replication`, etc.) into a format Prometheus understands
- **Prometheus** — pulls ("scrapes") metrics on a schedule and stores them as a time series
- **Grafana** — queries Prometheus and renders dashboards/alerts

You can run all three on `pg-node1` for a lab, or on a separate `monitoring` VM (closer to real-world practice, since you don't want monitoring competing with the DB for resources). I'll show the single-VM version first and note the separate-VM tweak.

---

## 6.1 Install postgres_exporter (on `pg-node1`)

```bash
cd /tmp
PG_EXPORTER_VERSION=0.15.0
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v${PG_EXPORTER_VERSION}/postgres_exporter-${PG_EXPORTER_VERSION}.linux-amd64.tar.gz
tar xvf postgres_exporter-${PG_EXPORTER_VERSION}.linux-amd64.tar.gz
sudo mv postgres_exporter-${PG_EXPORTER_VERSION}.linux-amd64/postgres_exporter /usr/local/bin/
```

Create a monitoring role in Postgres (least-privilege, don't use the superuser here):

```bash
sudo -u postgres psql <<'EOF'
CREATE ROLE monitoring WITH LOGIN PASSWORD 'monpass123';
GRANT pg_monitor TO monitoring;
EOF
```
`pg_monitor` is a built-in role (since PG10) that grants read access to all the stats views without full superuser rights.

Create a systemd service so it survives reboots:

```bash
sudo tee /etc/systemd/system/postgres_exporter.service > /dev/null <<'EOF'
[Unit]
Description=Postgres Exporter
After=network.target

[Service]
Environment=DATA_SOURCE_NAME=postgresql://monitoring:monpass123@127.0.0.1:5432/labdb?sslmode=disable
ExecStart=/usr/local/bin/postgres_exporter
Restart=always
User=postgres

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now postgres_exporter
sudo systemctl status postgres_exporter
```

Verify it's emitting metrics:
```bash
curl http://127.0.0.1:9187/metrics | head -30
```
You should see lines like `pg_up 1`, `pg_stat_database_numbackends`, etc.

---

## 6.2 Install node_exporter (host-level metrics: CPU, RAM, disk)

Useful alongside postgres_exporter since DB problems are often actually disk-I/O or memory problems.

```bash
cd /tmp
NODE_EXPORTER_VERSION=1.8.2
wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
tar xvf node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
sudo mv node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter /usr/local/bin/

sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<'EOF'
[Unit]
Description=Node Exporter
After=network.target

[Service]
ExecStart=/usr/local/bin/node_exporter
Restart=always
User=nobody

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
curl http://127.0.0.1:9100/metrics | head -10
```

---

## 6.3 Install Prometheus

```bash
cd /tmp
PROM_VERSION=2.53.1
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz
tar xvf prometheus-${PROM_VERSION}.linux-amd64.tar.gz
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo mv prometheus-${PROM_VERSION}.linux-amd64/prometheus /usr/local/bin/
sudo mv prometheus-${PROM_VERSION}.linux-amd64/promtool /usr/local/bin/
sudo mv prometheus-${PROM_VERSION}.linux-amd64/consoles /etc/prometheus/
sudo mv prometheus-${PROM_VERSION}.linux-amd64/console_libraries /etc/prometheus/
```

Config file — this is where you tell Prometheus what to scrape:

```bash
sudo tee /etc/prometheus/prometheus.yml > /dev/null <<'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'postgres'
    static_configs:
      - targets: ['127.0.0.1:9187']

  - job_name: 'node'
    static_configs:
      - targets: ['127.0.0.1:9100']

  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9090']
EOF
```

If you're monitoring `pg-node2` as well, add another target under the `postgres` job pointing to `<pg-node2-ip>:9187` (after installing postgres_exporter there too).

Systemd service:
```bash
sudo useradd --no-create-home --shell /bin/false prometheus
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

sudo tee /etc/systemd/system/prometheus.service > /dev/null <<'EOF'
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

Check the targets page: `http://<pg-node1-external-ip>:9090/targets` — all three jobs should show **UP**.

**You'll need a firewall rule to reach the web UIs from your laptop:**
```bash
gcloud compute firewall-rules create allow-monitoring-ui \
  --network=default \
  --allow=tcp:9090,tcp:3000 \
  --source-ranges=<your-laptop-public-ip>/32 \
  --target-tags=postgres-lab
```
Find your public IP with `curl ifconfig.me` if you don't know it. Scoping to your own IP (not `0.0.0.0/0`) keeps the lab from being open to the whole internet.

---

## 6.4 Install Grafana

```bash
sudo apt-get install -y apt-transport-https software-properties-common
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana

sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```

Open `http://<pg-node1-external-ip>:3000` — default login is `admin` / `admin` (you'll be forced to change it on first login).

### Connect Grafana to Prometheus
1. Left menu → **Connections → Data sources → Add data source**
2. Choose **Prometheus**
3. URL: `http://127.0.0.1:9090`
4. **Save & test** — should show "Successfully queried the Prometheus API"

### Import a ready-made PostgreSQL dashboard
Grafana.com hosts community dashboards. Easiest path:
1. Left menu → **Dashboards → New → Import**
2. Enter dashboard ID **9628** (a well-known "PostgreSQL Database" dashboard built for postgres_exporter) or **1860** for the Node Exporter Full host dashboard
3. Select your Prometheus data source → **Import**

You'll immediately see panels for active connections, transactions/sec, cache hit ratio, database size, replication lag, etc.

---

## 6.5 Exercises — generate load and watch it happen

**Generate query load** using `pgbench` (built into `postgresql-contrib`):
```bash
sudo -u postgres createdb pgbench_db -O learner
pgbench -i -s 10 -U postgres -h 127.0.0.1 pgbench_db     # initializes test tables, scale factor 10
pgbench -c 10 -j 2 -T 60 -U postgres -h 127.0.0.1 pgbench_db   # 10 clients, 60 seconds
```
Watch the Grafana dashboard while this runs — TPS, active connections, and cache hit ratio should visibly spike.

**Watch replication lag** (if `pg-node2` is set up from Module 4): the imported dashboard should have a replication lag panel; alternatively query directly:
```promql
pg_replication_lag
```
in Prometheus's own query UI (`http://<ip>:9090/graph`).

**Simulate a problem and observe it:**
```bash
# open a long-running idle transaction to watch "idle in transaction" connections climb
psql -U postgres -h 127.0.0.1 -d labdb -c "BEGIN; SELECT pg_sleep(120);"
```
Check the dashboard's connection-state breakdown panel while this runs.

---

## 6.6 Alerting (optional next step)

Once dashboards feel familiar, the natural next step is **Grafana Alerting** or **Prometheus Alertmanager** — e.g., alert if:
- `pg_up == 0` (Postgres is down)
- `pg_replication_lag > 30` (replica falling behind)
- connections approaching `max_connections`
- disk usage (`node_filesystem_avail_bytes`) below a threshold

I can build this out as Module 7 with concrete alert rules and a routing setup (e.g., to email or Slack) when you're ready — it's a good final piece once you're comfortable reading the dashboards.

---

## Quick Reference

```bash
sudo systemctl {status|restart} postgres_exporter node_exporter prometheus grafana-server
curl http://127.0.0.1:9187/metrics   # raw postgres_exporter output
curl http://127.0.0.1:9100/metrics   # raw node_exporter output
```

Useful PromQL to try in the Prometheus UI:
```promql
pg_up
rate(pg_stat_database_xact_commit[5m])
pg_stat_activity_count
node_filesystem_avail_bytes{mountpoint="/"}
```
