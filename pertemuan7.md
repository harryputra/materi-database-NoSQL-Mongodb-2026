# PERTEMUAN 7 – REPLIKASI, SHARDING, DAN MANAJEMEN DATA
**Program Studi:** Teknologi Rekayasa Informatika Industri (TRIN)
**Alokasi Waktu:** 1 × 150 menit (praktikum)

---

## A. CAPAIAN PEMBELAJARAN PERTEMUAN 7
Setelah pertemuan ini mahasiswa mampu:
1. Menjelaskan konsep replikasi dan arsitektur replica set untuk ketersediaan tinggi (high availability).
2. Mengonfigurasi replica set 3 node di lingkungan lokal dan mengamati proses failover.
3. Menerapkan write concern, read concern, dan read preference sesuai kebutuhan industri.
4. Memahami arsitektur sharded cluster (config server, mongos, shard) untuk skalabilitas horizontal.
5. Menentukan shard key yang tepat berdasarkan pola akses data industri.
6. Melakukan backup dan restore database menggunakan `mongodump`/`mongorestore` serta `mongoexport`/`mongoimport`.
7. Merancang strategi backup otomatis dan disaster recovery untuk sistem basis data pabrik.

---

## B. ALAT DAN BAHAN
- **MongoDB Server** (mongod) yang terinstal (versi 7.0+).
- **MongoDB Shell** (`mongosh`).
- **Terminal** (minimal 3 tab/window untuk menjalankan beberapa proses).
- **Direktori penyimpanan data terpisah** untuk tiap node replika.
- **MongoDB Compass** (untuk visualisasi status replikasi, opsional).
- **Hak akses** untuk membuka port 27017–27019 (localhost).
- **Docker** (opsional, alternatif untuk menjalankan multi‑node dengan mudah).
- Koneksi internet (jika menggunakan MongoDB Atlas untuk demo sharding).

---

## C. URAIAN MATERI SUPER LENGKAP

### 1. Replikasi di MongoDB

#### a. Mengapa Replikasi Penting?
- **High Availability:** Jika server utama (primary) mati, secondary otomatis menggantikan tanpa downtime.
- **Disaster Recovery:** Data disalin ke beberapa node; jika satu rusak, data tetap aman.
- **Read Scalability:** Beban baca dapat didistribusikan ke secondary.
- **Dedicated Backup:** Secondary bisa dijadikan sumber backup tanpa memengaruhi primary.
- **Data Locality:** Replika dapat ditempatkan di lokasi berbeda untuk mendekatkan data ke pengguna.

Di pabrik: server di lantai produksi tidak boleh berhenti saat terjadi pemadaman atau kerusakan hardware.

#### b. Arsitektur Replica Set
- **Primary:** Satu‑satunya node yang menerima operasi tulis. Semua perubahan direkam di **oplog** (operation log).
- **Secondary:** Menyalin oplog dari primary secara asinkron dan menerapkan operasi yang sama. Bisa melayani pembacaan jika diizinkan.
- **Arbiter (opsional):** Node tanpa data, hanya berpartisipasi dalam pemilihan primary. Digunakan untuk memecah kebuntuan (jumlah ganjil) tanpa biaya penyimpanan.
- **Number of members:** Idealnya ganjil (minimal 3) agar dapat mencapai quorum.

**Failover Otomatis:**
Jika primary gagal, replica set mengadakan **election** untuk memilih primary baru. Node dengan priority tertinggi dan data terbaru (oplog tertinggi) terpilih. Waktu failover biasanya 10–30 detik.

**Oplog:**
- Capped collection di setiap node replika.
- Menyimpan urutan operasi yang mengubah data.
- Secondary menggunakan oplog untuk menyinkronkan.
- Ukuran oplog default 5% dari disk, bisa dikonfigurasi.

#### c. Konfigurasi Dasar Replica Set
1. Jalankan minimal 3 instance `mongod` dengan opsi `--replSet` yang sama, port dan dbpath berbeda.
2. Dari `mongosh` ke salah satu node, inisialisasi:
   ```javascript
   rs.initiate({
     _id: "rs0",
     members: [
       { _id: 0, host: "localhost:27017" },
       { _id: 1, host: "localhost:27018" },
       { _id: 2, host: "localhost:27019" }
     ]
   })
   ```
3. Setelah beberapa saat, primary akan terpilih. Cek dengan `rs.status()` atau `rs.isMaster()`.

#### d. Read Preference
Menentukan dari mana klien membaca data:
- `primary` (default): hanya primary, konsistensi kuat.
- `primaryPreferred`: primary jika tersedia, kalau tidak secondary.
- `secondary`: hanya secondary.
- `secondaryPreferred`: secondary jika tersedia, kalau tidak primary.
- `nearest`: node dengan latensi terendah.

**Industri:** Laporan shift bisa diambil dari secondary untuk mengurangi beban primary. Monitoring real‑time mungkin butuh `primary` agar data terbaru.

#### e. Write Concern
Menentukan seberapa banyak node yang harus mengakui tulis sebelum operasi dianggap berhasil.
- `{ w: 1 }` – hanya primary (cepat, risiko data loss).
- `{ w: "majority" }` – mayoritas anggota voting (aman, delay sedikit).
- `{ w: <number> }` – spesifik jumlah node.
- `j: true` – memaksa journal commit (lebih aman).
- `wtimeout` – batas waktu tunggu.

**Contoh di PyMongo:**
```python
db.sensor.insert_one(doc, write_concern=WriteConcern(w="majority"))
```

#### f. Read Concern
Menentukan tingkat isolasi untuk data yang dibaca:
- `local`: data apa adanya di node (bisa belum direplikasi ke mayoritas = rollback possible).
- `available`: mirip `local` pada non‑sharded.
- `majority`: data yang sudah diakui mayoritas (aman, tidak akan kena rollback).
- `linearizable`: paling ketat, menunggu semua operasi tulis saat itu selesai.

Untuk data kritis (misal penggantian stok sparepart) gunakan `majority`.

### 2. Sharding (Skalabilitas Horizontal)

#### a. Konsep
Sharding membagi koleksi menjadi beberapa **chunk** yang didistribusikan ke beberapa **shard**. Setiap shard adalah replica set tersendiri. Ini memungkinkan database menangani volume data dan beban tulis yang sangat besar melebihi kapasitas satu server.

**Komponen:**
- **Shard:** Tempat penyimpanan subset data (biasanya replica set).
- **Config Server:** Menyimpan metadata cluster dan mapping chunk. (Wajib replica set, 3 node di produksi).
- **mongos:** Router yang menerima query dari aplikasi, meneruskannya ke shard yang tepat berdasarkan shard key.

#### b. Shard Key
Field (atau field‑field) yang digunakan untuk mendistribusikan dokumen. Pilihan shard key sangat kritis:
- **Range‑based sharding:** Dokumen dikelompokkan berdasarkan rentang nilai shard key (misal mesin A‑M di shard1, N‑Z di shard2).
- **Hash‑based sharding:** Fungsi hash diterapkan pada shard key, sehingga distribusi merata (menghindari hotspot).

**Ciri shard key baik:**
- **Cardinality tinggi:** Banyak nilai unik (misal `sensor_id` bukan `status`).
- **Distribusi merata:** Tidak ada satu nilai yang mendominasi (jangan `lokasi` jika semua sensor di satu tempat).
- **Mendukung query utama:** Query yang tidak menyertakan shard key akan menyebar ke semua shard (scatter‑gather, lambat).

**Contoh industri:**
Koleksi `sensor_data` dengan miliaran dokumen. Shard key: `{ mesin: 1, timestamp: 1 }` (compound) memberikan lokalitas data per mesin dan waktu.

#### c. Operasi Sharding Dasar
1. Aktifkan sharding pada database:
   ```javascript
   sh.enableSharding("produksi")
   ```
2. Shard koleksi:
   ```javascript
   sh.shardCollection("produksi.sensor", { mesin: "hashed" })
   ```
   atau range:
   ```javascript
   sh.shardCollection("produksi.sensor", { mesin: 1, timestamp: 1 })
   ```

#### d. Balancing
Balancer berjalan di background untuk memindahkan chunk antar shard agar distribusinya merata. Di lingkungan produksi sebaiknya dijadwalkan di jam sepi.

### 3. Backup dan Restore

#### a. `mongodump` / `mongorestore`
- Backup biner (BSON) dan metadata koleksi.
- `mongodump --db=industri --out=/backup/` menghasilkan folder per database.
- `mongorestore --db=industri /backup/industri/` mengembalikan.
- Bisa dilakukan pada primary atau secondary (gunakan `--readPreference=secondary` untuk mengurangi beban primary).

**Command lengkap:**
```bash
mongodump --uri="mongodb://localhost:27017" --db=praktikum7 --out=/home/user/backup/
```
- Opsi `--gzip` untuk kompresi.
- Opsi `--archive` untuk backup streaming.

#### b. `mongoexport` / `mongoimport`
- Eksport/impor dalam format JSON atau CSV, cocok untuk pertukaran dengan sistem lain.
- Tidak menyimpan tipe data BSON spesifik (tanggal jadi string), kurang akurat.
- `mongoexport --db=industri --collection=sensor --out=sensor.json`

#### c. Snapshot File System
- Menghentikan sementara penulisan (`db.fsyncLock()`), mengambil snapshot volume (LVM, SAN), lalu `db.fsyncUnlock()`.
- Metode paling cepat untuk database besar, namun perlu akses sistem operasi.

#### d. Backup Otomatis dengan Cron / Task Scheduler
Contoh script shell:
```bash
#!/bin/bash
BACKUP_DIR="/backup/mongodb/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR
mongodump --uri="mongodb://localhost:27017" --out=$BACKUP_DIR
find /backup/mongodb/ -type d -mtime +7 -exec rm -rf {} \;  # hapus backup >7 hari
```
Jalankan via cron harian.

### 4. Praktik Terbaik di Industri
- Selalu gunakan replica set untuk produksi, minimal 3 node.
- Simpan oplog cukup besar untuk toleransi jeda replikasi.
- Gunakan write concern `majority` untuk data yang tidak boleh hilang.
- Backup reguler, uji restore secara berkala.
- Sharding hanya jika diperlukan; jangan terlalu cepat.
- Pantau replikasi lag dan operasi balancing.

---

## D. KEGIATAN PRAKTIKUM (120 MENIT)

### Fase 1: Membangun Replica Set Lokal (45 menit)
Asisten/dosen memandu langkah‑langkah:

1. **Buat direktori data:**
   ```bash
   mkdir -p ~/data/rs0-0 ~/data/rs0-1 ~/data/rs0-2
   ```
2. **Jalankan tiga proses mongod** (gunakan 3 terminal):
   ```bash
   mongod --replSet rs0 --port 27017 --dbpath ~/data/rs0-0 --bind_ip localhost --logpath ~/data/rs0-0.log --fork
   mongod --replSet rs0 --port 27018 --dbpath ~/data/rs0-1 --bind_ip localhost --logpath ~/data/rs0-1.log --fork
   mongod --replSet rs0 --port 27019 --dbpath ~/data/rs0-2 --bind_ip localhost --logpath ~/data/rs0-2.log --fork
   ```
   (Di Windows, gunakan Command Prompt terpisah, abaikan `--fork`, atau gunakan `start`).

3. **Koneksi dan inisialisasi replica set:**
   Buka `mongosh` ke port 27017:
   ```javascript
   rs.initiate({
     _id: "rs0",
     members: [
       { _id: 0, host: "localhost:27017" },
       { _id: 1, host: "localhost:27018" },
       { _id: 2, host: "localhost:27019" }
     ]
   })
   ```

4. **Cek status:** `rs.status()` perhatikan `stateStr` (PRIMARY/SECONDARY).

5. **Insert data di primary:**
   ```javascript
   use rs_test
   db.data.insertOne({ info: "test replikasi", waktu: new Date() })
   ```

6. **Baca dari secondary:**
   - Koneksi ke secondary (misal port 27018).
   - Jalankan `rs.secondaryOk()` (atau di `mongosh` gunakan `db.data.find().readPref("secondary")`).
   - Tampilkan data, pastikan tersalin.

7. **Simulasi failover:**
   - Hentikan proses primary (Ctrl+C).
   - Amati perubahan `rs.status()` di salah satu shell yang tersisa.
   - Primary baru akan terpilih setelah beberapa detik.
   - Lakukan insert lagi untuk membuktikan operasi tulis berjalan.

8. **Bersihkan:** Matikan semua proses mongod (jangan di‑fork, gunakan `kill` atau `Ctrl+C`). Hapus direktori data jika tidak diperlukan.

### Fase 2: Backup dan Restore (30 menit)
1. Gunakan database `praktikum7` yang berisi koleksi `sensor` (bisa dibuat dadakan dengan 100 dokumen).
2. Lakukan backup dengan `mongodump`:
   ```bash
   mongodump --db=praktikum7 --out=/tmp/backup_praktikum7
   ```
   Periksa isi folder backup: file `.bson` dan `.metadata.json`.
3. Hapus koleksi `sensor`:
   ```javascript
   db.sensor.drop()
   ```
4. Restore:
   ```bash
   mongorestore --db=praktikum7 /tmp/backup_praktikum7/praktikum7
   ```
5. Verifikasi data kembali di shell.
6. Coba juga `mongoexport` dan `mongoimport` untuk format JSON:
   ```bash
   mongoexport --db=praktikum7 --collection=sensor --out=sensor_export.json
   mongoimport --db=praktikum7 --collection=sensor_import --file=sensor_export.json
   ```

### Fase 3: Pengenalan Sharding (30 menit)
Karena kompleksitas, dosen memberikan demonstrasi langsung atau menggunakan environment Docker. Jika memungkinkan, mahasiswa mengikuti langkah ini (disediakan VM atau script):

1. Jalankan config server replica set (3 node) di port 27019‑27021 (singkat, cukup 1 config server untuk latihan menggunakan `mongod --configsvr`).
2. Jalankan 1 shard (mongod biasa) di port 27022.
3. Jalankan mongos di port 27017 yang menunjuk ke config server.
4. Tambahkan shard ke cluster melalui mongos:
   ```javascript
   sh.addShard("localhost:27022")
   ```
5. Aktifkan sharding database dan koleksi:
   ```javascript
   sh.enableSharding("demo_industri")
   sh.shardCollection("demo_industri.sensor_large", { mesin: "hashed" })
   ```
6. Masukkan banyak dokumen (5000) melalui mongos dan amati distribusi dengan `sh.status()`.

Jika lingkungan terbatas, mahasiswa dapat menggunakan MongoDB Atlas free tier (M0) untuk mencoba fitur sharding (dengan batasan). Atau bisa dengan video interaktif dan diskusi.

### Fase 4: Diskusi dan Penutup (15 menit)
- Kapan pabrik membutuhkan sharding?
- Bagaimana strategi backup harian yang aman tanpa downtime?
- Tanya jawab.

---

## E. LATIHAN MANDIRI
1. Gunakan replica set yang sudah dibuat (jika masih ada) atau buat ulang miniatur dengan 2 node + arbiter. Dokumentasikan langkah inisialisasi dan uji failover.
2. Buat script backup otomatis untuk database `latihan7` dan simpan di direktori dengan tanggal. Jadwalkan menggunakan cron (Linux) atau Task Scheduler (Windows) dan catat log hasil.
3. Jelaskan perbedaan antara `mongodump` dan snapshot filesystem, serta kapan masing‑masing cocok untuk data sensor berukuran 500 GB.

---

## F. STUDI KASUS: HIGH AVAILABILITY UNTUK LANTAI PRODUKSI
**Latar Belakang:**
Sebuah pabrik besar memiliki 4 lantai, masing‑masing lantai memiliki server lokal yang mengumpulkan data sensor. Manajemen ingin agar data tetap tersedia jika satu server mati, dan seluruh data dapat diakses secara terpusat di kantor pusat. Volume data per lantai sekitar 50 GB/bulan, terus bertambah.

**Tugas:**
1. Rancang arsitektur replikasi untuk setiap lantai. Berapa node minimal yang disarankan?
2. Apakah diperlukan sharding untuk menggabungkan data ke pusat? Jika ya, usulkan shard key.
3. Buat strategi backup: jenis backup, frekuensi, penyimpanan.
4. Evaluasi kegagalan: skenario disaster (server lantai 2 rusak total). Bagaimana data dipulihkan?
5. Tuangkan dalam laporan 2–3 halaman, sertakan diagram.

---

## G. TUGAS TERSTRUKTUR
### Deskripsi
Mengimplementasikan replikasi dan backup pada database praktikum.

### Spesifikasi
1. Buat database `pabrik_<NIM>` dengan koleksi `produksi` yang berisi data produksi 3 bulan (minimal 1000 dokumen).
2. Siapkan replica set dengan 2 node data dan 1 arbiter (jika memungkinkan) atau 3 node data.
3. Lakukan backup penuh menggunakan `mongodump` dan juga backup logis dengan `mongoexport` dalam format CSV.
4. Simulasikan skenario kehilangan data: drop database, lalu restore dari backup.
5. Buktikan bahwa data berhasil dikembalikan dengan query count.
6. (Opsional) Buat script Python yang otomatis melakukan backup dan kirim notifikasi (cetak) jika sukses/gagal.
7. Tulis laporan langkah‑langkah, tangkapan layar, dan pelajaran yang didapat.

### Rubrik Penilaian
| Kriteria                        | Bobot |
|---------------------------------|-------|
| Konfigurasi replica set         | 30%   |
| Implementasi backup & restore   | 25%   |
| Verifikasi integritas data      | 15%   |
| Otomasi / script                | 15%   |
| Dokumentasi dan analisis        | 15%   |

### Tenggat
Sebelum pertemuan ke‑8, unggah di LMS.

---

## H. REFERENSI PERTEMUAN 7
1. MongoDB. *Replication*. https://docs.mongodb.com/manual/replication/
2. MongoDB. *Sharding*. https://docs.mongodb.com/manual/sharding/
3. MongoDB University. *M103: Basic Cluster Administration* (Free Course).
4. MongoDB. *Backup and Restore*. https://docs.mongodb.com/manual/core/backups/
5. Copeland, R. (2020). *MongoDB Applied Design Patterns*. O'Reilly.

---

**Pesan Dosen:**
Replikasi dan sharding adalah fondasi infrastruktur data modern yang handal. Kemampuan backup adalah asuransi terbaik. Pahami dengan baik karena di dunia industri nyata, Anda akan sering berhadapan dengan masalah ketersediaan dan pemulihan data. Selamat belajar!
