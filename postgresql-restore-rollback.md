# PostgreSQL Backup, Restore, WAL, dan Rollback

## 1. Perbedaan Rollback dan Restore

Sering kali Rollback dianggap sama dengan Restore, padahal keduanya berbeda.

| Rollback | Restore |
|----------|----------|
| Membatalkan transaksi yang belum di-COMMIT | Mengembalikan database dari file backup |
| Tidak membutuhkan file backup | Membutuhkan file backup |
| Menggunakan mekanisme Transaction PostgreSQL | Menggunakan pg_restore / pg_basebackup |
| Cepat | Bergantung ukuran backup |

---

# 2. Praktik Rollback

## Kondisi Awal

```sql
SELECT id,name,score
FROM dummy_users;
```

Output

| id | name | score |
|----|------|------:|
|1|User_1|50|
|2|User_2|80|
|3|User_3|90|

---

## Step 1 - Mulai Transaction

```sql
BEGIN;
```

Output

```
BEGIN
```

---

## Step 2 - Update Data

```sql
UPDATE dummy_users
SET score=100
WHERE id=1;
```

Lihat hasilnya

```sql
SELECT id,name,score
FROM dummy_users;
```

Output

|id|name|score|
|--|----|------:|
|1|User_1|100|
|2|User_2|80|
|3|User_3|90|

Perubahan ini **belum permanen**.

---

## Step 3 - Rollback

```sql
ROLLBACK;
```

Lihat kembali

```sql
SELECT id,name,score
FROM dummy_users;
```

Output

|id|name|score|
|--|----|------:|
|1|User_1|50|
|2|User_2|80|
|3|User_3|90|

Data kembali seperti semula.

---

# 3. Praktik COMMIT

Mulai transaction

```sql
BEGIN;
```

Update

```sql
UPDATE dummy_users
SET score=100
WHERE id=1;
```

Simpan perubahan

```sql
COMMIT;
```

Sekarang data menjadi permanen.

Jika menjalankan

```sql
ROLLBACK;
```

akan muncul

```
WARNING:
there is no transaction in progress
```

Karena transaksi sudah selesai.

---

# 4. Praktik Restore

Misalnya sebelumnya sudah melakukan backup.

```bash
pg_dump -U postgres -Fc coba -f coba.backup
```

Hari berikutnya database berubah.

```sql
DELETE FROM dummy_users;
```

Semua data hilang.

Restore dilakukan menggunakan

```bash
dropdb coba

createdb coba

pg_restore -U postgres -d coba coba.backup
```

Masuk kembali ke database

```bash
psql -U postgres -d coba
```

Cek data

```sql
SELECT * FROM dummy_users;
```

Semua data kembali seperti saat backup dibuat.

---

# 5. Apa itu WAL?

WAL adalah singkatan dari

> **Write Ahead Log**

Artinya

> PostgreSQL akan menulis log perubahan terlebih dahulu sebelum mengubah data asli.

Urutannya

```
INSERT

↓

Tulis ke WAL

↓

COMMIT

↓

Update Data File
```

---

## Contoh

Misalnya menjalankan

```sql
UPDATE dummy_users
SET score=500
WHERE id=2;
```

Yang terjadi

```
Client

↓

UPDATE

↓

WAL

↓

Data File
```

---

# 6. Fungsi WAL

WAL digunakan untuk

- Crash Recovery
- Streaming Replication
- Hot Backup
- Point In Time Recovery (PITR)

---

# 7. Crash Recovery

Misalnya

```sql
BEGIN;

UPDATE dummy_users
SET score=500
WHERE id=2;

COMMIT;
```

Tiba-tiba server mati.

Saat PostgreSQL hidup kembali

```
Startup

↓

Membaca WAL

↓

Replay WAL

↓

Database Konsisten
```

Karena perubahan sudah tercatat di WAL.

---

# 8. Hot Backup

Hot Backup adalah backup ketika PostgreSQL masih berjalan.

```
Client

↓

INSERT

↓

Primary PostgreSQL

↓

WAL

↓

Copy Data

↓

Backup
```

Database tetap bisa digunakan oleh user.

Contoh

```bash
pg_basebackup \
-D /backup/basebackup \
-F p \
-X stream \
-P
```

Keterangan

| Parameter | Fungsi |
|-----------|---------|
| -D | Folder tujuan backup |
| -F p | Format plain |
| -X stream | Ikut mengambil WAL |
| -P | Menampilkan progress |

---

# 9. Restore Hot Backup

Misalnya hasil backup berada di

```
/backup/basebackup
```

Stop PostgreSQL

```bash
sudo systemctl stop postgresql
```

Hapus data lama

```bash
rm -rf /var/lib/postgresql/16/main/*
```

Copy hasil backup

```bash
cp -r /backup/basebackup/* /var/lib/postgresql/16/main/
```

Jalankan PostgreSQL

```bash
sudo systemctl start postgresql
```

Saat startup PostgreSQL akan melakukan

```
Replay WAL

↓

Database Konsisten
```

---

# 10. Cold Backup

Cold Backup dilakukan ketika PostgreSQL dimatikan.

Stop service

```bash
sudo systemctl stop postgresql
```

Copy seluruh data directory

```bash
cp -a /var/lib/postgresql/16/main /backup/cold_backup
```

atau

```bash
tar -czvf cold_backup.tar.gz /var/lib/postgresql/16/main
```

Start kembali

```bash
sudo systemctl start postgresql
```

Karena database dimatikan, tidak ada transaksi baru selama proses backup sehingga hasil copy sudah konsisten.

---

# 11. Restore Cold Backup

Stop PostgreSQL

```bash
sudo systemctl stop postgresql
```

Hapus data lama

```bash
rm -rf /var/lib/postgresql/16/main/*
```

Copy kembali hasil backup

```bash
cp -a /backup/cold_backup/* /var/lib/postgresql/16/main/
```

Perbaiki permission

```bash
chown -R postgres:postgres /var/lib/postgresql/16/main
```

Start PostgreSQL

```bash
sudo systemctl start postgresql
```

---

# 12. Streaming Replication

Yang dikirim dari Primary ke Standby **bukan database**, tetapi WAL.

```
Client

↓

INSERT

↓

Primary

↓

Generate WAL

↓

Streaming WAL

↓

Standby

↓

Replay WAL

↓

Standby ikut berubah
```

---

# 13. Ringkasan

| Fitur | Menggunakan WAL |
|--------|-----------------|
| Rollback | ❌ |
| COMMIT | ✅ |
| Crash Recovery | ✅ |
| Hot Backup | ✅ |
| Cold Backup | ❌ (tidak membutuhkan WAL tambahan selama proses backup) |
| Streaming Replication | ✅ |
| PITR | ✅ |

---

# Kesimpulan

- **Rollback** digunakan untuk membatalkan transaksi yang belum di-COMMIT.
- **Restore** digunakan untuk mengembalikan database dari file backup.
- **WAL** adalah log perubahan yang ditulis sebelum data asli diubah.
- **Hot Backup** memanfaatkan WAL agar backup tetap konsisten saat database masih online.
- **Cold Backup** dilakukan ketika database dimatikan sehingga tidak memerlukan WAL tambahan selama proses backup.
- **Streaming Replication** mengirim WAL dari Primary ke Standby.
- **PITR (Point In Time Recovery)** menggunakan WAL untuk mengembalikan database ke waktu tertentu.