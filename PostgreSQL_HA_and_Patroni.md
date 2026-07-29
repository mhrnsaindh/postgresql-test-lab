# Praktikum PostgreSQL High Availability (HA) di Ubuntu

## Topologi

| VM | Hostname | Peran |
|---|---|---|
| VM1 | pg-master | Primary |
| VM2 | pg-standby | Standby |
| VM3 | pg-patroni | etcd + Patroni |

---

# Bagian 1 - Native PostgreSQL Streaming Replication

## 1. Install PostgreSQL (VM1 & VM2)

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
psql --version
```

## 2. Buat User Replication (VM1)

```bash
sudo -u postgres psql
```

```sql
CREATE ROLE replicator LOGIN REPLICATION PASSWORD '123456';
\du
```

## 3. Konfigurasi postgresql.conf

Edit:

```bash
sudo nano /etc/postgresql/16/main/postgresql.conf
```

Tambahkan/Ubah:

```conf
listen_addresses='*'
wal_level=replica
max_wal_senders=10
max_replication_slots=10
hot_standby=on
wal_keep_size=512MB
```

## 4. Konfigurasi pg_hba.conf

```conf
host replication replicator 10.10.0.3/32 md5
```

(Lab)

```conf
host replication replicator 0.0.0.0/0 md5
```

## 5. Restart PostgreSQL

```bash
sudo systemctl restart postgresql
```

## 6. Buat Database Uji

```bash
createdb perusahaan
psql perusahaan
```

```sql
CREATE TABLE pegawai(
 id SERIAL PRIMARY KEY,
 nama TEXT,
 divisi TEXT
);

INSERT INTO pegawai(nama,divisi)
VALUES
('Andi','IT'),
('Budi','Finance'),
('Rina','HR');

SELECT * FROM pegawai;
```

## 7. Konfigurasi Standby (VM2)

```bash
sudo systemctl stop postgresql
sudo rm -rf /var/lib/postgresql/16/main/*
```

```bash
sudo -u postgres pg_basebackup \
-h 10.10.0.2 \
-D /var/lib/postgresql/16/main \
-U replicator \
-P \
-R
```

Start:

```bash
sudo systemctl start postgresql
```

Cek:

```sql
SELECT pg_is_in_recovery();
```

Output:

```
t
```

## 8. Uji Replikasi

Primary:

```sql
INSERT INTO pegawai(nama,divisi)
VALUES ('Doni','IT');
```

Standby:

```sql
SELECT * FROM pegawai;
```

## 9. Simulasi Failover Manual

Matikan Primary

```bash
sudo systemctl stop postgresql
```

Promote Standby

```bash
sudo -u postgres pg_ctl promote \
-D /var/lib/postgresql/16/main
```

atau

```sql
SELECT pg_promote();
```

Verifikasi

```sql
SELECT pg_is_in_recovery();
```

Output

```
false
```

Tes

```sql
INSERT INTO pegawai VALUES(DEFAULT,'Susi','IT');
```

---

# Bagian 2 - Patroni + etcd

## 1. Install PostgreSQL

```bash
sudo apt install postgresql -y
```

## 2. Install Python

```bash
sudo apt install python3-pip -y
```

## 3. Install Patroni

```bash
pip3 install patroni
```

## 4. Install etcd (VM3)

```bash
sudo apt install etcd -y
sudo systemctl enable etcd
sudo systemctl start etcd
systemctl status etcd
```

## 5. Konfigurasi patroni.yml (VM1)

```yaml
scope: pgcluster
name: pg-master

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.10.0.2:8008

etcd:
  host: 10.10.0.4:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.10.0.2:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  authentication:
    superuser:
      username: postgres
      password: postgres
    replication:
      username: replicator
      password: 123456
```

VM2 sama, hanya ubah `name` dan `connect_address`.

## 6. Jalankan Patroni

```bash
patroni /etc/patroni.yml
```

atau sebagai service.

## 7. Cek Cluster

```bash
patronictl list
```

Contoh:

```
+ Cluster: pgcluster +

pg-master   Leader
pg-standby  Replica
```

## 8. Simulasi Failover

```bash
sudo systemctl stop patroni
```

atau

```bash
sudo systemctl stop postgresql
```

Verifikasi:

```bash
patronictl list
```

Replica akan otomatis menjadi Leader.

---

# Perbandingan

|Fitur|Native|Patroni|
|---|---|---|
|Streaming Replication|✅|✅|
|Manual Failover|✅|❌|
|Automatic Failover|❌|✅|
|Leader Election|❌|✅|
|Auto Rejoin|❌|✅|
|Production Ready|Terbatas|Ya|
