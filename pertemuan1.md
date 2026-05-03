# PERTEMUAN 1 – PENGENALAN NOSQL, MONGODB, DAN INSTALASI LINGKUNGAN KERJA
**Program Studi:** Teknologi Rekayasa Informatika Industri (TRIN)
**Alokasi Waktu:** 1 × 150 menit (praktikum)

---

## A. CAPAIAN PEMBELAJARAN PERTEMUAN 1
Setelah mengikuti perkuliahan ini, mahasiswa mampu:
1. Menjelaskan keterbatasan basis data relasional dalam konteks data industri modern.
2. Memahami karakteristik basis data NoSQL serta perbedaannya dengan SQL.
3. Menjelaskan posisi MongoDB sebagai document store dan konsep dasar arsitekturnya.
4. Melakukan instalasi MongoDB Community Edition, MongoDB Shell (`mongosh`), dan MongoDB Compass pada sistem operasi (Windows, Linux, atau macOS) secara mandiri.
5. Menjalankan perintah dasar: membuat database, membuat koleksi, menyisipkan dokumen, serta menampilkan dokumen.
6. Menerjemahkan kebutuhan penyimpanan data sensor sederhana ke dalam struktur dokumen MongoDB.

---

## B. ALAT DAN BAHAN
### Perangkat Keras
- PC/Laptop dengan RAM minimal 4 GB, ruang disk kosong 5 GB.
- Sistem Operasi: Windows 10/11 64-bit, Ubuntu 20.04/22.04 LTS 64-bit, atau macOS 11+.

### Perangkat Lunak (disediakan oleh mahasiswa/panduan)
- **MongoDB Community Server** versi 7.0 atau lebih baru (gratis).
- **MongoDB Shell (`mongosh`)** versi terbaru.
- **MongoDB Compass** (GUI, opsional namun disarankan).
- **Text Editor**: Visual Studio Code, Sublime Text, atau Notepad++.
- **Browser web** untuk akses dokumentasi dan unduhan.
- **Terminal/Command Prompt** yang mendukung perintah shell.

### Koneksi Internet
- Diperlukan untuk mengunduh installer. Setelah instalasi, semua praktik berjalan secara lokal (offline).

---

## C. URAIAN MATERI SUPER LENGKAP

### 1. Mengapa NoSQL untuk Dunia Industri?
#### a. Karakteristik Data di Industri Modern
- **Volume:** Ratusan sensor pada mesin produksi menghasilkan ribuan titik data per detik.
- **Variety:** Data terstruktur (tabel produk), semi‑terstruktur (log JSON dari PLC), dan tidak terstruktur (gambar hasil inspeksi, catatan teknisi).
- **Velocity:** Data streaming real‑time (suhu, getaran, tekanan) harus langsung tersimpan tanpa hambatan skema rigid.
- **Variability:** Format data sensor bisa berubah sewaktu‑waktu (penambahan sensor baru, perubahan parameter).

#### b. Keterbatasan SQL dalam Konteks Tersebut
- **Skema kaku:** Perubahan struktur tabel (ALTER TABLE) memerlukan downtime dan kompleksitas tinggi pada sistem yang sudah berjalan.
- **Skalabilitas vertikal** yang mahal. Scale‑out (horizontal) sulit dilakukan secara native.
- **Object‑Relational Impedance Mismatch:** Data dari aplikasi (berbentuk objek) harus dipecah menjadi baris dan kolom, menambah overhead kode.
- **Join yang kompleks** menjadi lambat pada volume data sangat besar.

#### c. Solusi NoSQL
NoSQL (Not Only SQL) menawarkan:
- **Schema‑less / Dynamic schema:** Dokumen dapat memiliki struktur berbeda dalam satu koleksi.
- **Skalabilitas horizontal** melalui sharding otomatis.
- **Model data yang lebih sesuai** dengan representasi objek aplikasi (dokumen JSON).

### 2. Teori Fundamental NoSQL

#### a. Teorema CAP (Consistency, Availability, Partition Tolerance)
Teorema CAP menyatakan bahwa dalam sistem terdistribusi, kita hanya dapat memilih dua dari tiga properti secara bersamaan. MongoDB secara default mengutamakan **Consistency** dan **Partition Tolerance** (CP), namun dapat dikonfigurasi untuk prioritas **Availability** (AP) dengan mengatur *read preference* dan *write concern*.

**Implikasi untuk industri:**
- Jika sistem monitoring tidak boleh menampilkan data kadaluarsa → pilih CP.
- Jika downtime tidak dapat ditoleransi meski data mungkin belum terupdate → pilih AP.

#### b. Model Konsistensi BASE (Basically Available, Soft state, Eventually consistent)
Berbeda dengan ACID pada SQL, banyak sistem NoSQL mengadopsi BASE. MongoDB mendukung transaksi ACID multi‑dokumen sejak versi 4.0, tetapi fleksibilitasnya tetap mengizinkan pendekatan BASE untuk kinerja tinggi.

### 3. Klasifikasi Basis Data NoSQL
1. **Key-Value Store** (Redis, DynamoDB) – cocok untuk caching, session.
2. **Document Store** (MongoDB, CouchDB) – fokus pada dokumen semi‑terstruktur (JSON/BSON).
3. **Wide Column Store** (Cassandra, HBase) – untuk time-series dengan kolom dinamis.
4. **Graph Database** (Neo4j) – data yang sangat terhubung (supply chain, BOM).

**MongoDB** dipilih karena:
- Model dokumen yang intuitif bagi pengembang.
- Kemampuan query yang kaya (agregasi, indeks sekunder).
- Ekosistem yang matang dan digunakan luas di industri (Bosch, Forbes, Cisco).

### 4. Arsitektur MongoDB
MongoDB adalah basis data berorientasi dokumen yang menyimpan data dalam **BSON** (Binary JSON).

**Komponen utama:**
- **mongod**: Daemon server yang menangani permintaan data, manajemen indeks, dan operasi latar belakang.
- **mongos**: Router untuk sharded cluster (topik lanjut).
- **MongoDB Shell (`mongosh`)**: CLI untuk berinteraksi dengan server.
- **MongoDB Compass**: GUI untuk visualisasi data dan pembuatan query.

**Struktur data:**
- **Database**: Wadah tertinggi, berisi koleksi.
- **Collection**: Setara tabel, namun tanpa skema tetap.
- **Document**: Unit data terkecil, berupa pasangan key‑value dalam format BSON.
- **Field**: Key di dalam dokumen.

**Perbandingan terminologi:**

| SQL          | MongoDB           | Keterangan |
|--------------|-------------------|------------|
| Database     | Database          | Sama       |
| Table        | Collection        | Kumpulan dokumen |
| Row          | Document (BSON)   | Data individual |
| Column       | Field             | Key dalam dokumen |
| Primary Key  | `_id` (ObjectId)  | Unik dan otomatis |
| Join         | `$lookup` / embedding | Pendekatan berbeda |

### 5. JSON dan BSON
- **JSON**: Format teks ringan, mudah dibaca manusia, standar pertukaran data.
- **BSON**: Binary JSON; mendukung tipe data tambahan (Date, ObjectId, Int32, Double, Binary data) dan lebih efisien untuk parsing dan penyimpanan.

**Contoh dokumen sensor:**
```json
{
  "_id": ObjectId("645af1e2b3c4d5e6f7a8b9c0"),
  "id_mesin": "CNC-15",
  "suhu_celsius": 78.5,
  "getaran_mm_s": 0.12,
  "timestamp": ISODate("2026-05-04T08:30:00Z"),
  "status": "normal",
  "lokasi": {
    "lantai": 2,
    "area": "perakitan"
  }
}
```

### 6. ObjectId
- Otomatis dibuat untuk field `_id` jika tidak didefinisikan.
- Panjang 12 byte: timestamp (4), machine identifier (5), process id (3), counter (3).
- Menjamin keunikan global bahkan dalam sistem terdistribusi.

### 7. Instalasi MongoDB (Panduan Langkah demi Langkah Super Detail)

**Peringatan Umum:**
- Pastikan tidak ada service MongoDB berjalan sebelumnya.
- Hak akses administrator diperlukan.
- Port default 27017 harus bebas. Gunakan `netstat -an | findstr 27017` (Windows) atau `sudo lsof -i :27017` (Linux/macOS) untuk memeriksa.

#### a. Windows
1. **Unduh Installer:**
   - Buka https://www.mongodb.com/try/download/community
   - Pilih Version: 7.0.x, Platform: Windows, Package: MSI.
   - Klik Download.
2. **Jalankan Installer:**
   - Klik kanan file `.msi`, pilih *Run as Administrator*.
   - Klik Next, centang *I accept the terms*.
   - **Pilih Setup Type:** Pilih **Complete** (menginstal semua fitur).
   - **Service Configuration:**
     - Centang *Install MongoD as a Service*.
     - Pilih *Run service as Network Service user* (default).
     - Service Name: MongoDB.
     - Data Directory: biarkan default (`C:\Program Files\MongoDB\Server\7.0\data`).
     - Log Directory: biarkan default.
   - Klik Next, lalu Install.
   - Tunggu hingga selesai.
3. **Tambahkan ke System PATH:**
   - Buka File Explorer, salin path `C:\Program Files\MongoDB\Server\7.0\bin`.
   - Tekan Windows + Pause/Break → Advanced system settings → Environment Variables.
   - Di System variables, cari Path → Edit → New → tempel path tersebut → OK.
4. **Verifikasi mongod:**
   - Buka Command Prompt (Admin) dan jalankan:
     ```
     "C:\Program Files\MongoDB\Server\7.0\bin\mongod.exe" --version
     ```
   - Harus muncul versi.
5. **Mulai Service:**
   - Buka Services (services.msc), cari MongoDB, pastikan status Running. Jika tidak, klik Start.
6. **Unduh dan Instal `mongosh`:**
   - Buka https://www.mongodb.com/try/download/shell
   - Pilih Platform: Windows 64-bit, download ZIP.
   - Ekstrak ZIP, salin folder `mongosh-...\bin` ke PATH yang sama seperti langkah 3.
   - Verifikasi: buka CMD baru, ketik `mongosh --version`.

#### b. Linux (Ubuntu 22.04 LTS)
1. **Impor kunci publik MongoDB:**
   ```bash
   wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-7.0.gpg
   ```
2. **Tambahkan repositori:**
   ```bash
   echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
   ```
3. **Update dan instal:**
   ```bash
   sudo apt update
   sudo apt install -y mongodb-org
   ```
4. **Jalankan dan enable service:**
   ```bash
   sudo systemctl start mongod
   sudo systemctl enable mongod
   sudo systemctl status mongod  # pastikan active (running)
   ```
5. **Unduh dan instal `mongosh`:** (gunakan paket .deb dari situs resmi atau melalui apt jika tersedia)
   ```bash
   wget https://downloads.mongodb.com/compass/mongosh-2.1.0-linux-x64.tgz
   tar -xvzf mongosh-2.1.0-linux-x64.tgz
   sudo cp mongosh-2.1.0-linux-x64/bin/mongosh /usr/local/bin/
   ```

#### c. macOS
1. **Homebrew (disarankan):**
   ```bash
   brew tap mongodb/brew
   brew install mongodb-community@7.0
   brew services start mongodb-community@7.0
   ```
2. **Verifikasi:**
   ```bash
   mongosh --version
   ```
3. **Jika tanpa Homebrew**, unduh Community Server dari situs MongoDB, ekstrak, tambahkan bin ke PATH.

### 8. Menggunakan MongoDB Shell (`mongosh`)
- Masuk shell:
  ```bash
  mongosh
  ```
  Output akan menampilkan versi shell, server, dan URL koneksi.
- `mongosh` adalah shell JavaScript modern, mendukung sintaks ES6.

### 9. Perintah Dasar (Praktik Terbimbing)
```javascript
// Melihat database yang ada
show dbs

// Pindah ke database tertentu (jika belum ada, akan dibuat saat data disisipkan)
use industri_db

// Menampilkan database yang sedang digunakan
db

// Membuat koleksi secara eksplisit (opsional)
db.createCollection("sensor")

// Melihat semua koleksi dalam database
show collections

// Menyisipkan satu dokumen ke koleksi "sensor"
db.sensor.insertOne({
  id_mesin: "CNC-01",
  suhu: 75.2,
  timestamp: new Date(),
  status: "aktif"
})

// Menampilkan semua dokumen (diformat rapi)
db.sensor.find().pretty()

// Menyisipkan banyak dokumen
db.sensor.insertMany([
  { id_mesin: "CNC-01", suhu: 76.1, timestamp: new Date() },
  { id_mesin: "CNC-02", suhu: 82.3, timestamp: new Date() }
])

// Filter sederhana
db.sensor.find({ id_mesin: "CNC-01" }).pretty()

// Menghapus koleksi (hati-hati)
db.sensor.drop()
```

### 10. MongoDB Compass (GUI)
- Menyambungkan ke `mongodb://localhost:27017`.
- Memudahkan melihat dokumen, membuat query dengan builder, dan memonitor performa.
- Untuk saat ini, cukup digunakan untuk memverifikasi data yang diinsert melalui shell.

---

## D. KEGIATAN PRAKTIKUM (120 MENIT)
### Fase 1: Diskusi Interaktif dan Brainstorming (15 menit)
Dosen menampilkan data riil dari satu stasiun kerja produksi (dapat berupa CSV dengan kolom: `mesin`, `waktu`, `suhu`, `getaran`, `operator`). Pertanyaan pemantik:
- “Bayangkan kita punya 100 mesin, setiap detik mengirim data. Desain tabel SQL seperti apa yang paling efisien?”
- “Apa yang terjadi jika besok kita tambah sensor kelembaban pada sebagian mesin saja?”
Mahasiswa mendiskusikan kendala dan menyadari kelemahan skema kaku.

### Fase 2: Instalasi Terpandu (45 menit)
Asisten/dosen memandu langkah instalasi sesuai OS mayoritas. Mahasiswa mengikuti pada perangkat masing‑masing. Setiap keberhasilan diverifikasi dengan menjalankan `mongosh --version` dan `mongosh`.

**Checkpoint:**
- [ ] MongoDB Community Server terinstal dan service berjalan.
- [ ] `mongosh` dapat masuk tanpa error.
- [ ] Compass dapat terhubung (opsional).

### Fase 3: Eksplorasi Shell Dasar (30 menit)
Mahasiswa melakukan secara mandiri (dengan panduan lembar kerja):

1. Masuk ke shell.
2. Buat database `praktikum1`.
3. Buat koleksi `machines`.
4. Sisipkan 3 dokumen dengan struktur:
   - `nama_mesin` (string),
   - `tipe` (CNC / Bubut / Milling),
   - `tahun_instalasi` (integer),
   - `lokasi` (dokumen embedded: `{ lantai: int, area: string }`).
5. Gunakan `find().pretty()` untuk menampilkan semua.
6. Coba filter hanya mesin dengan tipe “CNC”.
7. Ubah satu dokumen menggunakan `updateOne` dengan `$set` (menambahkan field `status: "maintenance"`).
8. Hapus salah satu dokumen dengan `deleteOne`.

### Fase 4: Pengenalan MongoDB Compass (30 menit)
- Buka Compass, koneksi ke localhost.
- Navigasi ke database `praktikum1`, lihat koleksi `machines`.
- Perhatikan tampilan Table View vs JSON View.
- Gunakan fitur “Insert Document” untuk menambah dokumen baru via GUI.
- Kembali ke shell, verifikasi dokumen baru tersebut muncul.

---

## E. LATIHAN MANDIRI
**Instruksi:** Kerjakan dengan teliti, simpan semua perintah dalam file teks.

1. Database: Buat database `latihan_industri`.
2. Koleksi: Di dalamnya, buat koleksi `sensor_suhu`.
3. Insert: Masukkan 5 dokumen yang merepresentasikan pembacaan suhu dari 3 mesin berbeda (`M001`, `M002`, `M003`) pada jam berbeda. Gunakan `new Date()` untuk timestamp, tetapi atur jam berbeda secara manual dengan `new Date("2026-05-04T08:00:00")` dsb.
4. Query:
   - Tampilkan semua dokumen milik `M001`.
   - Tampilkan dokumen yang suhunya di atas 80 derajat ( `suhu: {$gt: 80}` ).
   - Tampilkan hanya field `mesin` dan `suhu` (proyeksi).
5. Eksplorasi: Jalankan `db.sensor_suhu.stats()` dan catat ukuran data.

**Format Laporan Latihan:**
- Screenshot hasil setiap langkah.
- Penjelasan singkat mengapa MongoDB tidak memerlukan definisi kolom terlebih dahulu.

---

## F. STUDI KASUS: SISTEM MONITORING GETARAN MESIN
**Latar Belakang:** Sebuah pabrik pengecoran logam memiliki 10 mesin cetak. Setiap mesin dilengkapi sensor getaran yang mengirim data setiap 5 menit. Data yang perlu disimpan: ID mesin, nilai getaran (mm/s), frekuensi dominan (Hz), suhu bearing (°C), waktu pengukuran. Kadang teknisi menambahkan catatan jika ada anomali.

**Tugas Anda (dikerjakan di dalam kelas sisa waktu atau sebagai pekerjaan rumah terarah):**
1. Rancang struktur dokumen yang sesuai untuk menyimpan data di atas.
2. Tentukan database dan nama koleksi yang relevan.
3. Sisipkan 3 data simulasi untuk mesin `M-CETAK-01` dengan kondisi normal.
4. Sisipkan 1 data anomali, dengan tambahan field `catatan: "getaran tinggi, bearing perlu dicek"` yang tidak ada pada dokumen normal.
5. Query untuk menampilkan:
   - Semua data `M-CETAK-01`.
   - Data dengan getaran > 10 mm/s.
6. Apa keuntungan skema fleksibel MongoDB dalam kasus penambahan catatan di atas?

**Hasil dikumpulkan dalam bentuk laporan singkat PDF** (format: pendahuluan, desain dokumen, perintah, output screenshot, analisis).

---

## G. TUGAS TERSTRUKTUR
### Deskripsi
Mahasiswa membuat database pabrik mini secara mandiri. Tugas ini mengintegrasikan semua keterampilan pertemuan 1.

### Spesifikasi
1. Buat database `pabrik_<NIM>` (misal `pabrik_123456`).
2. Di dalamnya buat dua koleksi:
   - `mesin`: berisi data statis 3 mesin (kode, nama, tipe, lokasi_lantai).
   - `log_produksi`: berisi 10 dokumen produksi yang melibatkan mesin‑mesin tersebut (kode_mesin, tanggal, produk, jumlah_baik, jumlah_cacat, operator).
3. Setidaknya satu dokumen di `log_produksi` memiliki field tambahan `keterangan` yang menjelaskan sebab cacat.
4. Tunjukkan query untuk:
   - Menampilkan semua mesin.
   - Menampilkan log produksi mesin dengan kode tertentu.
   - Menampilkan log yang jumlah cacatnya > 2.
5. Gunakan `insertOne`, `insertMany`, dan `find` dengan filter.

### Pengumpulan
- File teks berisi semua perintah yang digunakan.
- Screenshot shell yang menunjukkan output.
- Berikan komentar singkat pada setiap langkah.

### Rubrik Penilaian
| Kriteria | Bobot |
|----------|-------|
| Pembuatan database & koleksi sesuai instruksi | 20% |
| Struktur dokumen sesuai dan variatif | 25% |
| Keberhasilan query dengan filter | 30% |
| Dokumentasi dan komentar | 15% |
| Kreativitas (field tambahan, variasi data) | 10% |

### Tenggat
Sebelum pertemuan ke-2 dimulai, diunggah di LMS.

---

## H. DAFTAR PUSTAKA DAN REFERENSI
1. Bradshaw, S., Brazil, E., & Chodorow, K. (2019). *MongoDB: The Definitive Guide* (3rd ed.). O’Reilly Media.
2. MongoDB, Inc. (2025). *MongoDB Manual*. https://docs.mongodb.com/manual/
3. MongoDB University. (2025). *M001: MongoDB Basics* (Free Online Course).
4. Edlich, S., et al. (2020). *NoSQL: Database Technologies*. Springer.

---

**Catatan untuk Mahasiswa:**
Pastikan MongoDB tetap terinstal dengan baik karena akan digunakan di seluruh pertemuan. Jika mengalami kendala selama instalasi mandiri, segera hubungi asisten melalui forum diskusi LMS. Selamat menjelajah dunia NoSQL!
