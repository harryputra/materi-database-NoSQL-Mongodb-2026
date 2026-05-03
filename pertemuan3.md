# PERTEMUAN 3 – PEMODELAN DATA MONGODB UNTUK DOMAIN INDUSTRI
**Program Studi:** Teknologi Rekayasa Informatika Industri (TRIN)
**Alokasi Waktu:** 1 × 150 menit (praktikum)

---

## A. CAPAIAN PEMBELAJARAN PERTEMUAN 3
Setelah mengikuti perkuliahan ini, mahasiswa mampu:
1. Menjelaskan karakteristik data di industri yang memengaruhi pemodelan NoSQL.
2. Membedakan pendekatan **embedded** dan **referencing** serta menganalisis kelebihan/kekurangan masing-masing.
3. Memilih pendekatan yang tepat berdasarkan kebutuhan query, ukuran data, dan hubungan antar entitas.
4. Merancang skema dokumen MongoDB untuk kasus nyata: data sensor, hierarki pabrik, bill of material (BOM), dan log produksi.
5. Mengimplementasikan beberapa pola desain (embedded array, manual reference, `$lookup`).
6. Menerapkan teknik normalisasi/denormalisasi yang sesuai dengan prinsip performa dan konsistensi.

---

## B. ALAT DAN BAHAN
- Server MongoDB aktif (versi 7.0+).
- MongoDB Shell (`mongosh`).
- MongoDB Compass (untuk visualisasi skema).
- Dataset dummy/imsak untuk mensimulasikan data pabrik (akan dibuat selama praktikum).
- Kertas/buffer digital untuk membuat diagram koleksi dan hubungan.

---

## C. URAIAN MATERI SUPER LENGKAP

### 1. Mengapa Pemodelan Data di MongoDB Berbeda dengan SQL?
- **Tidak ada relasi formal** seperti foreign key, sehingga kita memutuskan apakah menyimpan data terkait dalam satu dokumen (embed) atau memisahkannya (reference).
- **Pertanyaan utama**: “Bagaimana data akan diakses dan dimanipulasi?”
- **Prinsip utama**: *Data that is accessed together should be stored together.*
- **Dampak pada performa**: embedding mengurangi join, namun dapat membuat dokumen besar (batas 16 MB) dan update berat. Referencing membutuhkan join (`$lookup`) yang mahal.

### 2. Konsep Embedded Document vs Reference

#### a. Embedded Document (Denormalisasi)
- Data yang berhubungan dimasukkan langsung ke dalam dokumen utama.
- **Contoh**: Satu dokumen `mesin` menyimpan array `sensor_history` yang berisi 100 data terbaru.
- **Kelebihan**:
  - Pembacaan (read) sekali jalan, tanpa join.
  - Ideal jika data sering diakses bersama.
  - Operasi atomik pada satu dokumen (transaksi built-in).
- **Kekurangan**:
  - Dokumen bisa membesar, mendekati batas 16 MB.
  - Update data yang di-embed redundan jika digunakan di banyak dokumen.
  - Tidak efisien jika data yang di-embed sering berubah secara independen.

#### b. Reference (Normalisasi)
- Data disimpan di koleksi terpisah, dihubungkan menggunakan field `_id` atau key buatan sendiri.
- **Contoh**: Koleksi `machines` dan `sensors` terpisah. Dokumen sensor menyimpan `machine_id`.
- **Kelebihan**:
  - Mengurangi duplikasi data.
  - Data dapat berubah secara independen tanpa memengaruhi dokumen lain.
  - Mudah untuk one-to-many tak terbatas (misal jutaan data sensor per mesin).
- **Kekurangan**:
  - Membutuhkan `$lookup` untuk menggabungkan, yang lebih lambat dari single read.
  - Kompleksitas query bertambah.

### 3. Aturan Praktis Pemilihan (Rule of Thumb)

| Kondisi                                            | Gunakan Embedding | Gunakan Reference |
|----------------------------------------------------|-------------------|-------------------|
| Hubungan one-to-one atau one-to-few                | ✅ Ya             | -                 |
| One-to-many, tapi data anak sering diakses bersama induk dan jumlahnya terbatas (≤ ratusan) | ✅ Ya             | -                 |
| One-to-many, data anak jumlahnya besar (ribuan+) atau tumbuh terus | -                 | ✅ Ya             |
| Many-to-many                                       | ❌ Hindari        | ✅ Ya             |
| Data anak sering diakses terpisah tanpa induk      | -                 | ✅ Ya             |
| Data anak sangat sering berubah (update)           | -                 | ✅ Ya             |
| Butuh konsistensi tinggi antar data terkait        | ✅ Ya (atomik)    | - (transaksi multi-dokumen bisa, tapi mahal) |

**Tambahan industri:**
- Data sensor real-time: array embedded untuk buffer terbaru (misal 100 data), sisanya di koleksi terpisah (reference) untuk histori panjang.
- Konfigurasi mesin (jarang berubah) → embed ke mesin.
- Transaksi maintenance: referensi ke sparepart karena stok sparepart dipakai banyak mesin.

### 4. Pola Umum dalam Pemodelan MongoDB untuk Industri

#### a. Pola One-to-One
Contoh: Satu mesin memiliki satu spesifikasi teknis.
```json
// Mesin
{
  "_id": "M001",
  "nama": "CNC Milling",
  "spesifikasi": {
    "max_rpm": 12000,
    "daya_kw": 15,
    "berat_kg": 2500
  }
}
```
Embed karena spesifikasi tidak akan diakses terpisah, jarang berubah.

#### b. Pola One-to-Few (Embedded Array)
Contoh: Setiap mesin memiliki beberapa alarm aktif (biasanya sedikit).
```json
{
  "_id": "M001",
  "alarm": [
    { "kode": "A1", "deskripsi": "suhu tinggi", "timestamp": ISODate("...") },
    { "kode": "A2", "deskripsi": "tekanan rendah", "timestamp": ISODate("...") }
  ]
}
```
Jumlah alarm per mesin kecil (<50), sering diakses bersama data mesin.

#### c. Pola One-to-Many (Reference)
Contoh: Satu mesin menghasilkan ribuan data sensor per hari.
```javascript
// Koleksi machines
{ "_id": "M001", "nama": "CNC 15", "lokasi": "Lantai 2" }

// Koleksi sensor_data
{ "_id": ObjectId(...), "machine_id": "M001", "suhu": 78.5, "ts": ISODate(...) }
```
Reference karena volume besar dan sering di-query terpisah (laporan harian, tren).

#### d. Pola Many-to-Many
Contoh: Produk dan komponen (Bill of Material). Satu produk punya banyak komponen, satu komponen dipakai banyak produk.
Gunakan koleksi ketiga (junction):
```javascript
// products
{ "_id": "PROD1", "nama": "Mesin X" }
// components
{ "_id": "COMP1", "nama": "Bearing A", "stok": 200 }
// product_components (junction)
{ "product_id": "PROD1", "component_id": "COMP1", "jumlah": 4 }
```
Bisa juga dengan embed array of references di salah satu sisi, hati-hati dengan duplikasi.

#### e. Pola Tree (Hierarki)
Contoh: Struktur organisasi pabrik → lantai → area → mesin.
Beberapa pendekatan:
- **Parent reference**: setiap node menyimpan `parent_id`.
- **Child reference**: setiap node menyimpan array `children_ids`.
- **Materialized path**: menyimpan path string, misal `"pabrik/lantai2/areaA"`.
- **Nested sets** (kompleks).

Untuk MongoDB, disarankan parent reference untuk fleksibilitas:
```json
{
  "_id": "mesin5",
  "nama": "CNC Router",
  "parent": "areaA",
  "type": "machine"
}
```

### 5. Praktik Query dengan Referensi: `$lookup`
`$lookup` melakukan left outer join dengan koleksi lain dalam aggregation pipeline.
```javascript
db.machines.aggregate([
  {
    $lookup: {
      from: "sensor_data",
      localField: "_id",
      foreignField: "machine_id",
      as: "data_sensor"
    }
  }
])
```
**Catatan**: Hindari `$lookup` pada data sangat besar jika bisa di-embed.

### 6. Validasi Skema (Opsional tapi Penting)
Mulai MongoDB 3.2, kita dapat menerapkan **JSON Schema Validation** untuk menjaga kualitas data walaupun skema fleksibel.
```javascript
db.createCollection("mesin", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nama", "lokasi"],
      properties: {
        nama: { bsonType: "string", description: "nama mesin wajib string" },
        lokasi: { bsonType: "string" }
      }
    }
  }
})
```
Untuk industri, ini membantu mencegah data kotor (misal field suhu bertipe string).

### 7. Studi Kasus Rancangan Skema

#### a. Sistem Monitoring Mesin (Real-Time)
Kebutuhan:
- Tampilkan data terkini (suhu, getaran) untuk dashboard.
- Simpan histori selama 1 tahun untuk analisis.
- 50 mesin, setiap mesin 5 sensor, data setiap detik.

**Solusi:**
- Koleksi `machines` menyimpan data statis mesin + **embedded array “latest_data”** 100 detik terakhir.
- Koleksi `sensor_histories` untuk data historis dengan referensi `machine_id` dan `timestamp`.
- Data terkini diambil dari `machines` (single read), histori menggunakan agregasi `sensor_histories`.

#### b. Bill of Material (BOM)
- Produk “Mesin Bubut” terdiri dari sub-assembly “Spindle”, “Meja”, dll.
- Setiap sub-assembly punya komponen leaf (baut, bearing).

Pendekatan:
- Koleksi `assemblies` dengan field `parent_id` (null untuk root).
- Tiap dokumen punya `type: "assembly"` atau `type: "component"`.
- Query rekursif di aplikasi atau gunakan materialized path untuk query tree.

### 8. Best Practices
- Mulailah dengan embed, lalu pisah jika ditemukan masalah performa atau batasan ukuran.
- Jangan takut duplikasi data jika mempercepat pembacaan (data sensor terkini).
- Gunakan indexing untuk mendukung query reference (`machine_id` harus di‑index).
- Pantau ukuran dokumen dengan `Object.bsonsize(db.machines.findOne())`.
- Rencanakan pertumbuhan data: jika sensor bertambah 1 juta per hari, pasti reference.

---

## D. KEGIATAN PRAKTIKUM (120 MENIT)

### Fase 1: Diskusi Pemodelan (15 menit)
Dosen memberikan dua skenario singkat (misal “sistem absensi karyawan” dan “log error mesin”). Mahasiswa berpasangan menentukan apakah harus embed atau reference beserta alasan. Perwakilan menyampaikan.

### Fase 2: Embedded vs Reference Hands-On (45 menit)
**A. Embedded (15 menit)**
1. Buat database `praktikum3`.
2. Buat koleksi `machines_embedded` dengan 3 dokumen mesin, masing-masing memiliki array `data_terkini` (5 data sensor suhu, getaran).
   ```javascript
   db.machines_embedded.insertOne({
     _id: "M01",
     nama: "CNC Milling",
     lokasi: "Lantai 1",
     data_terkini: [
       { suhu: 72.1, getaran: 0.12, ts: new Date() },
       { suhu: 72.3, getaran: 0.13, ts: new Date(Date.now() - 1000) }
     ]
   })
   ```
3. Lakukan query:
   - Ambil data mesin M01 lengkap.
   - Ambil hanya suhu terbaru M01 (gunakan `$slice`). `db.machines_embedded.find({_id:"M01"}, { data_terkini: { $slice: -1 } })`

**B. Reference (30 menit)**
1. Buat koleksi `machines_ref` (hanya `_id` dan `nama`).
2. Buat koleksi `sensor_ref` dengan banyak data (20 dokumen) menyimpan `machine_id`, `suhu`, `getaran`, `ts`.
3. Insert data mesin dan sensor.
4. Jalankan agregasi `$lookup` untuk menampilkan data mesin beserta semua sensor miliknya.
5. Bandingkan:
   - Jumlah langkah dan query yang diperlukan.
   - Gunakan `explain()` untuk melihat `executionStats` kedua pendekatan.
   - Bahas mana yang lebih cepat untuk satu mesin, bagaimana jika banyak sensor (1000+).

### Fase 3: Simulasi Update pada Embedded vs Reference (20 menit)
1. Pada `machines_embedded`, tambahkan data suhu baru ke `data_terkini` menggunakan `$push` dengan `$slice` -100.
2. Pada `sensor_ref`, tambahkan satu dokumen baru ke koleksi `sensor_ref`.
3. Bandingkan operasi update: embedded harus memodifikasi dokumen besar (jika sudah banyak data), reference hanya insert kecil.
4. Cek ukuran dokumen dengan `Object.bsonsize`.

### Fase 4: Pola Tree dengan Parent Reference (20 menit)
1. Buat koleksi `area_pabrik`:
   - Pabrik (root, parent: null)
   - Lantai 1 (parent: id pabrik)
   - Area A (parent: id lantai1)
   - Mesin M01 (parent: areaA)
2. Query:
   - Cari semua anak langsung dari Lantai 1.
   - Bagaimana mendapatkan seluruh subtree? (Akan dilanjutkan di pertemuan agregasi atau aplikasi).

### Fase 5: Diskusi Studi Kasus Tambahan (20 menit)
Mahasiswa diberi deskripsi “Sistem Manajemen Kualitas: setiap batch produksi diperiksa oleh beberapa inspektor, setiap inspektor bisa menangani banyak batch.” Minta mereka menggambar skema di papan tulis, lalu didiskusikan.

---

## E. LATIHAN MANDIRI
**Instruksi:** Kerjakan dengan shell, simpan perintah dan screenshot.

1. Buat database `latihan3`.
2. Buat dua pendekatan untuk sistem peminjaman alat ukur di pabrik:
   - **Embedded**: Koleksi `karyawan` dengan array `pinjaman_alat` (alat, tanggal_pinjam, tanggal_kembali).
   - **Reference**: Koleksi `karyawan` dan `pinjaman` terpisah.
3. Masukkan 3 karyawan dan 5 peminjaman total.
4. Untuk tiap pendekatan, lakukan query:
   - Tampilkan semua peminjaman oleh karyawan “Budi”.
   - Tambahkan peminjaman baru.
   - Update tanggal_kembali pada satu peminjaman.
5. Analisis: Pada jumlah peminjaman banyak (>50 per karyawan), pendekatan mana yang lebih baik? Mengapa?

---

## F. STUDI KASUS: SISTEM INVENTORI DAN PEMASANGAN PART
**Latar Belakang:** Sebuah pabrik perakitan memiliki banyak mesin yang menggunakan berbagai part (bearing, seal, belt). Setiap mesin bisa memiliki 10–50 part terpasang. Teknisi ingin mencatat part apa saja yang terpasang di setiap mesin, serta melacak jumlah stok part di gudang. Kebutuhan:
- Melihat daftar part yang terpasang pada mesin tertentu.
- Melihat stok terkini suatu part.
- Mengurangi stok part saat dipasang.
- Part bisa dipasang di banyak mesin, stok part berubah sering.

**Tugas:**
1. Rancang dua skema minimal: (a) embedded part di mesin, (b) reference dengan koleksi part terpisah dan junction.
2. Implementasikan di MongoDB, masukkan data 3 mesin dan 5 jenis part.
3. Tunjukkan operasi:
   - Pasang part ke mesin (update).
   - Kurangi stok part (update).
   - Query menampilkan semua mesin beserta part-nya (gunakan `$lookup` jika reference).
   - Query menampilkan part dengan stok di bawah 10.
4. Evaluasi: Hitung waktu eksekusi dengan `explain()`. Tulis kesimpulan dalam 1 paragraf.
5. Format laporan: PDF dengan desain skema, perintah, hasil, analisis.

---

## G. TUGAS TERSTRUKTUR
### Deskripsi
Merancang basis data untuk **sistem manajemen shift dan produksi harian** dengan pendekatan yang dipilih sendiri secara argumentatif.

### Spesifikasi
- Buat database `tugas3_<NIM>`.
- Anda harus menyimpan data:
  - **Mesin**: 5 mesin (ID, nama, tipe).
  - **Shift**: ada 3 shift per hari (pagi, siang, malam).
  - **Hasil produksi**: setiap shift, setiap mesin menghasilkan sejumlah produk dengan jumlah ok dan reject, operator bertugas.
- Kebutuhan query:
  - Total produksi per mesin hari ini.
  - Semua hasil produksi shift pagi untuk inspeksi.
  - Update ok/reject pada suatu entri produksi.
- Pilih embedding atau referencing untuk menyimpan hasil produksi, apakah di-embed ke mesin, ke shift, atau koleksi sendiri. Jelaskan alasan dalam laporan.

### Tugas Detail:
1. Buat diagram koleksi dan contoh dokumen (bisa gambar tangan difoto / draw.io).
2. Implementasikan di MongoDB (berikan perintah).
3. Masukkan data: 5 mesin, 3 shift, 2 hari data produksi (total 30 entri produksi).
4. Jalankan query yang dibutuhkan, termasuk update.
5. Berikan analisis performa sederhana dengan `explain()` pada query utama.
6. Tulis dalam laporan PDF.

### Rubrik Penilaian
| Kriteria                              | Bobot |
|---------------------------------------|-------|
| Kualitas diagram dan desain skema     | 20%   |
| Ketepatan pilihan (embed vs ref)      | 25%   |
| Implementasi & data                   | 20%   |
| Query dan update berhasil             | 20%   |
| Analisis performa & argumentasi       | 15%   |

### Tenggat
Sebelum pertemuan ke-4, unggah di LMS.

---

## H. REFERENSI PERTEMUAN 3
1. MongoDB. *Data Modeling Introduction*. https://docs.mongodb.com/manual/core/data-modeling-introduction/
2. MongoDB University. *M320: Data Modeling* (Free Course).
3. Banker, K. (2020). *MongoDB in Action* (2nd ed.). Manning Publications.
4. Giamas, A. (2021). *MongoDB Data Modeling & Schema Design*. Technics Publications.

---

**Pesan Dosen:**
Pemodelan data adalah seni. Tidak ada jawaban mutlak benar/salah, hanya ada desain yang cocok untuk kebutuhan Anda. Selalu uji dengan volume data realistis dan beban query. Selamat bereksplorasi!
