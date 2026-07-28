# PostgreSQL Restore Menggunakan pg_basebackup

## Apakah Restore `pg_basebackup` Harus Stop Service PostgreSQL?

**Jawaban: Ya.**

Restore dari `pg_basebackup` (physical backup) biasanya harus dilakukan ketika PostgreSQL **berhenti (stop service)**.

Alasannya karena `pg_basebackup` melakukan backup pada level **data directory PostgreSQL**, bukan hanya isi database.

---

# 1. Perbedaan Cara Restore

## Logical Backup (`pg_dump`)

Logical backup bekerja pada level objek database.

Contoh:

```bash
pg_dump -U postgres -Fc coba -f coba.backup

Physical Backup (pg_basebackup)

Physical backup bekerja dengan menyalin seluruh data directory PostgreSQL.

Contoh:

pg_basebackup \
-D /backup/basebackup \
-F p \
-X stream \
-P

Hasil:

/backup/basebackup

├── base/
├── global/
├── pg_wal/
├── pg_xact/
├── postgresql.conf
└── ...

Restore dilakukan dengan mengganti isi data directory PostgreSQL.

2. Kenapa Harus Stop PostgreSQL?

Data PostgreSQL berada di:

/var/lib/postgresql/16/main

Isi folder:

main/

├── base/
├── global/
├── pg_wal/
├── pg_xact/
├── postgresql.conf
└── ...

Folder tersebut sedang digunakan oleh PostgreSQL.

Saat PostgreSQL berjalan:

PostgreSQL RUNNING

        |
        |
        v

Data directory sedang digunakan

Jika kita menimpa folder tersebut ketika PostgreSQL aktif:

Backup

   |
   v

/var/lib/postgresql/16/main

Risikonya:

File sedang digunakan oleh PostgreSQL
Data menjadi tidak konsisten
PostgreSQL gagal start
Permission menjadi bermasalah

Karena itu restore physical backup dilakukan saat PostgreSQL berhenti.

3. Alur Restore pg_basebackup

Misalnya hasil backup berada di:

/backup/basebackup
Step 1 - Stop PostgreSQL
sudo systemctl stop postgresql

Cek status:

sudo systemctl status postgresql

Output:

inactive (dead)
Step 2 - Backup Data Directory Lama (Opsional)

Sebelum mengganti data directory, lebih aman rename folder lama.

mv /var/lib/postgresql/16/main \
/var/lib/postgresql/16/main_old
Step 3 - Copy Hasil Backup

Copy hasil pg_basebackup:

cp -a /backup/basebackup \
/var/lib/postgresql/16/main

atau:

rm -rf /var/lib/postgresql/16/main/*

cp -a /backup/basebackup/* \
/var/lib/postgresql/16/main/
Step 4 - Perbaiki Permission

Pastikan owner adalah user PostgreSQL.

chown -R postgres:postgres \
/var/lib/postgresql/16/main
Step 5 - Start PostgreSQL
sudo systemctl start postgresql
Step 6 - Verifikasi

Login:

sudo -u postgres psql

Cek database:

\l

Cek tabel:

\c coba

\dt
4. Apa yang Direstore oleh pg_basebackup?

Berbeda dengan pg_dump, pg_basebackup melakukan restore seluruh cluster PostgreSQL.

Contoh server:

Database:
- coba
- production
- testing

User:
- postgres
- app_user
- backup_user

Hasil pg_basebackup:

Backup

├── Database coba
├── Database production
├── Database testing
├── User postgres
├── User app_user
└── User backup_user

Ketika direstore:

Database coba        ✅
Database production  ✅
Database testing     ✅
User PostgreSQL      ✅
Permission           ✅

Semua akan kembali.

5. Hubungan pg_basebackup dengan WAL

pg_basebackup menggunakan WAL agar hasil backup tetap konsisten.

Ketika backup berjalan:

Client

   |
   v

INSERT / UPDATE / DELETE

   |
   v

PostgreSQL

   |
   +------> WAL
   |
   +------> Data Directory

Perubahan yang terjadi selama backup akan dicatat dalam WAL.

Saat restore:

Base Backup

     +

WAL

     |

     v

Recovery Process

     |

     v

Database Konsisten

PostgreSQL akan membaca WAL dan melakukan replay transaksi yang diperlukan.

6. Perbandingan Restore
Backup	Restore perlu stop PostgreSQL?	Level Backup
pg_dump	Tidak wajib	Logical
pg_restore	Tidak wajib	Logical
pg_basebackup	Ya	Physical
Cold Backup	Ya	Physical
7. Kesimpulan
pg_dump melakukan backup pada level database/object.
pg_basebackup melakukan backup pada level physical server.
Restore pg_basebackup membutuhkan PostgreSQL berhenti karena mengganti data directory.
pg_basebackup mengembalikan seluruh cluster PostgreSQL, bukan hanya satu database.
WAL berperan menjaga konsistensi backup dan membantu proses recovery saat PostgreSQL berjalan kembali.