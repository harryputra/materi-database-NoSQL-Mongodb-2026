# PERTEMUAN 2 – OPERASI CRUD LANJUT DAN OPERATOR QUERY
**Program Studi:** Teknologi Rekayasa Informatika Industri (TRIN)
**Alokasi Waktu:** 1 × 150 menit (praktikum)

---

## A. CAPAIAN PEMBELAJARAN PERTEMUAN 2
Setelah pertemuan ini mahasiswa mampu:
1. Melakukan penyisipan dokumen secara massal (`insertMany`) dengan opsi *write concern*.
2. Menyusun query yang kompleks menggunakan proyeksi dan operator perbandingan, logika, elemen, dan array.
3. Melakukan pembaruan data menggunakan berbagai operator *update* (`$set`, `$inc`, `$push`, `$pull`, dll.) serta *upsert*.
4. Menghapus dokumen dengan filter yang tepat (`deleteOne`, `deleteMany`).
5. Menerjemahkan kebutuhan bisnis industri (lacak produksi, maintenance, alarm) ke dalam operasi CRUD MongoDB yang efisien.
6. Membedakan penggunaan operator query yang tepat untuk data historis sensor dan log produksi.

---

## B. ALAT DAN BAHAN
- **Server MongoDB** versi 7.0+ (terinstal di pertemuan 1) dan berjalan sebagai service.
- **MongoDB Shell (`mongosh`)** terbaru.
- **MongoDB Compass** (opsional, untuk verifikasi visual).
- **Dataset simulasi**: file JSON `log_produksi.json` berisi 100+ dokumen produksi harian (disediakan oleh dosen, atau mahasiswa bisa generate sendiri melalui script).
- **Terminal / Command Prompt**.

Pastikan server MongoDB berjalan:
```bash
sudo systemctl status mongod   # Linux
# atau cek Services di Windows
```

---

## C. URAIAN MATERI SUPER LENGKAP

### 1. Operasi Insert Mendalam

#### a. `insertOne()` Rekapitulasi
```javascript
db.collection.insertOne({ field: value, ... })
```
Menyisipkan satu dokumen. Jika field `_id` tidak disediakan, MongoDB membuatkan ObjectId otomatis.

#### b. `insertMany()` – Batch Insert
```javascript
db.log_produksi.insertMany([
  { mesin: "M001", produk: "Baut 10mm", jumlah: 500, reject: 2, waktu: new Date("2026-05-04T08:00:00Z") },
  { mesin: "M001", produk: "Baut 10mm", jumlah: 480, reject: 5, waktu: new Date("2026-05-04T09:00:00Z") },
  { mesin: "M002", produk: "Mur 8mm",   jumlah: 1000, reject: 8, waktu: new Date("2026-05-04T08:00:00Z") }
]);
```
Keuntungan:
- Satu kali perjalanan jaringan (round-trip) untuk banyak dokumen.
- Kecepatan jauh lebih tinggi dibanding memanggil `insertOne` berulang.

**Write Concern** (opsi) menentukan seberapa banyak anggota replica set yang harus mengakui penulisan sebelum operasi dianggap berhasil.
```javascript
db.collection.insertMany(docs, { writeConcern: { w: "majority", wtimeout: 5000 } })
```
- `w: 1` – hanya primary yang acknowledge (cepat, risiko data loss).
- `w: "majority"` – mayoritas anggota harus acknowledge (lebih aman).
- `wtimeout` – batas waktu tunggu acknowledge (ms).

**Catatan untuk industri:** Untuk data sensor yang bisa di-resend, `w:1` cukup; untuk transaksi penting (misal pergantian shift), gunakan `w:"majority"`.

**Ordered vs Unordered Insert:**
```javascript
db.collection.insertMany(docs, { ordered: false })
```
- `ordered: true` (default): berhenti pada error pertama.
- `ordered: false`: melanjutkan meski ada error, melaporkan semua error di akhir. Cocok untuk batch besar dimana beberapa dokumen mungkin duplikat `_id` tetapi sisanya harus tetap masuk.

#### c. Mengimpor File JSON dengan `mongoimport`
Sebagai alternatif shell, gunakan terminal:
```bash
mongoimport --db produksi --collection log_produksi --file log_produksi.json --jsonArray
```
Flag `--jsonArray` jika file berisi array JSON. Tanpa flag, setiap baris dianggap satu dokumen (format JSON Lines).

### 2. Query dan Proyeksi Kompleks

#### a. Proyeksi
Mengambil hanya field tertentu.
```javascript
db.log_produksi.find({}, { mesin: 1, reject: 1, _id: 0 })
```
- `1` = include, `0` = exclude.
- Tidak boleh campur include dan exclude kecuali `_id`.

**Proyeksi pada sub-dokumen:**
Misal dokumen punya `lokasi: { lantai: 2, area: "perakitan" }`.
```javascript
db.mesin.find({}, { "lokasi.lantai": 1 })
```

#### b. Operator Perbandingan
- `$eq` : sama dengan (default, jarang ditulis eksplisit)
- `$ne` : tidak sama dengan
- `$gt` : lebih besar dari (>)
- `$gte` : lebih besar atau sama dengan (>=)
- `$lt` : kurang dari (<)
- `$lte` : kurang atau sama dengan (<=)
- `$in` : nilai ada di dalam array
- `$nin` : nilai tidak ada di dalam array

**Contoh:**
```javascript
// Produk dengan jumlah cacat lebih dari 5
db.log_produksi.find({ reject: { $gt: 5 } })

// Mesin M001 atau M002
db.log_produksi.find({ mesin: { $in: ["M001", "M002"] } })
```

**Query Rentang Waktu:**
```javascript
// Data antara jam 8 dan 12 pada tanggal 4 Mei 2026
db.sensor.find({
  timestamp: {
    $gte: ISODate("2026-05-04T08:00:00Z"),
    $lt:  ISODate("2026-05-04T12:00:00Z")
  }
})
```

#### c. Operator Logika
- `$and` : semua kondisi terpenuhi
- `$or` : minimal satu kondisi terpenuhi
- `$not` : negasi kondisi
- `$nor` : semua kondisi tidak terpenuhi

**Contoh kasus industri – inspeksi kualitas:**
```javascript
// Produk yang reject > 5 DAN mesin M002, ATAU reject > 10 pada mesin manapun
db.log_produksi.find({
  $or: [
    { $and: [ { reject: { $gt: 5 } }, { mesin: "M002" } ] },
    { reject: { $gt: 10 } }
  ]
})
```

**Peringatan:** `$and` sering tidak diperlukan karena MongoDB secara implisit meng-AND kondisi jika diletakkan dalam satu objek. Gunakan `$and` jika perlu mengulangi field yang sama.
```javascript
// Salah: { reject: { $gt: 5 }, reject: { $lt: 10 } } -> field konflik
// Benar:
db.log_produksi.find({ $and: [ { reject: {$gt: 5} }, { reject: {$lt: 10} } ] })
```

#### d. Operator Elemen
- `$exists` : memeriksa keberadaan field
- `$type` : memeriksa tipe data field (menggunakan kode tipe atau alias string)

```javascript
// Dokumen yang memiliki field 'catatan'
db.log_produksi.find({ catatan: { $exists: true } })

// Field 'biaya' bertipe integer atau double
db.maintenance.find({ biaya: { $type: ["int", "double"] } })
```
**Kode tipe BSON penting:** `"string"`, `"number"`, `"bool"`, `"date"`, `"null"`, `"array"`, `"object"`.
Kode numerik: 2 = string, 16 = int32, 1 = double, 9 = date, dsb.

#### e. Operator Array (Pengantar)
- `$size` : array dengan jumlah elemen tertentu
- `$all` : array berisi semua nilai yang disebutkan (tanpa peduli urutan)
- `$elemMatch` : setidaknya satu elemen memenuhi beberapa kondisi

Contoh data mesin dengan array:
```json
{
  "_id": "M001",
  "tags": ["CNC", "berat", "presisi"]
}
```
```javascript
db.mesin.find({ tags: { $all: ["CNC", "berat"] } }) // tags harus memiliki kedua nilai
db.mesin.find({ tags: { $size: 3 } })
```

Untuk dokumen dengan array objek (misal `sensor_data: [{sensor:"temp", nilai:75}, ...]`), kita akan dalami di pertemuan agregasi.

### 3. Operasi Update Mendalam

#### a. Operator Field (Update)

**`$set`** – mengatur nilai suatu field (membuat jika belum ada)
```javascript
db.mesin.updateOne(
  { _id: "M001" },
  { $set: { status: "maintenance", lokasi: "Lantai 3" } }
)
```

**`$unset`** – menghapus field
```javascript
db.mesin.updateOne({ _id: "M001" }, { $unset: { catatan_sementara: "" } })
```
Nilai yang diberikan tidak penting (bisa string kosong, 1, null).

**`$inc`** – menambah nilai numerik (bisa negatif untuk mengurangi)
```javascript
// Naikkan jumlah reject sebanyak 2
db.log_produksi.updateOne({ _id: ObjectId("...") }, { $inc: { reject: 2 } })

// Kurangi stok komponen
db.inventory.updateOne({ part: "bearing" }, { $inc: { stok: -5 } })
```

**`$mul`** – mengalikan nilai numerik
```javascript
db.mesin.updateOne({ _id: "M001" }, { $mul: { max_rpm: 1.1 } }) // naik 10%
```

**`$rename`** – mengubah nama field
```javascript
db.mesin.updateMany({}, { $rename: { "nama_mesin": "nama" } })
```

#### b. Operator Array

**`$push`** – menambahkan elemen ke akhir array
```javascript
db.mesin.updateOne(
  { _id: "M001" },
  { $push: { log_error: "overheat" } }
)
```
Jika field belum ada, array akan dibuat.

**`$push` dengan modifier `$each` dan `$slice`:**
```javascript
// Tambah beberapa data sensor dan batasi array hanya 100 terbaru
db.mesin.updateOne(
  { _id: "M001" },
  { $push: { history_sensor: { $each: [ {s: "temp", v:75}, {s: "vib", v:0.1} ], $slice: -100 } }
)
```
`$slice: -100` menjaga array maksimal 100 elemen terakhir.

**`$addToSet`** – menambahkan elemen jika belum ada (unik)
```javascript
db.mesin.updateOne(
  { _id: "M001" },
  { $addToSet: { tags: "presisi" } }
)
```

**`$pull`** – menghapus elemen yang cocok dengan kondisi
```javascript
// Hapus tag "berat"
db.mesin.updateOne({ _id: "M001" }, { $pull: { tags: "berat" } })

// Hapus log error yang mengandung "overheat"
db.mesin.updateOne({ _id: "M001" }, { $pull: { log_error: { $regex: "overheat" } } })
```

**`$pop`** – menghapus elemen pertama (-1) atau terakhir (1)
```javascript
db.mesin.updateOne({ _id: "M001" }, { $pop: { history_sensor: 1 } }) // hapus terakhir
```

**`$push` vs `$addToSet` pada data industri:**
- `$push` untuk log berurutan (kita peduli waktu dan jumlah).
- `$addToSet` untuk himpunan (misal daftar part yang pernah dipasang).

#### c. Upsert
Upsert = update jika ditemukan, insert jika tidak. Digunakan untuk *create or update*.
```javascript
db.mesin.updateOne(
  { _id: "M005" },  // filter
  { $set: { nama: "CNC Router", lokasi: "Lantai 2" } },
  { upsert: true }
)
```
Jika M005 tidak ada, dokumen baru dibuat dengan `_id: "M005"` dan field yang di-set.

**Update banyak dokumen sekaligus:**
```javascript
db.log_produksi.updateMany(
  { mesin: "M001" },
  { $inc: { jumlah: -3 } }  // koreksi stok semua
)
```

### 4. Operasi Penghapusan

- `deleteOne(filter)`: menghapus satu dokumen pertama yang cocok.
- `deleteMany(filter)`: menghapus semua dokumen yang cocok.
- `deleteMany({})`: menghapus semua dokumen dalam koleksi (hati-hati!).

Contoh industri:
```javascript
// Hapus log yang reject >= 20 (data mungkin salah input)
db.log_produksi.deleteMany({ reject: { $gte: 20 } })

// Hapus dokumen maintenance yang tanggalnya sudah lewat lebih dari 30 hari
db.maintenance.deleteMany({ tanggal: { $lt: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) } })
```

Untuk menghapus koleksi: `db.log_produksi.drop()`.

### 5. Bekerja dengan Tipe Data Khusus
- **Date:** Gunakan `ISODate()` atau `new Date()` di shell. Untuk query rentang, pastikan timezone.
- **ObjectId:** Dapat digunakan untuk filter unik, misal `find({ _id: ObjectId("...") })`.
- **Null:** Field bernilai null atau tidak ada. `{ field: null }` akan mencocokkan field yang bernilai null atau tidak ada. Gunakan `{ field: { $type: 10 } }` khusus null.

### 6. Best Practices di Industri
- Selalu sertakan timestamp di dokumen untuk tracking.
- Batch insert dengan `insertMany` untuk efisiensi.
- Jangan update dokumen besar secara keseluruhan, gunakan operator spesifik.
- Manfaatkan `upsert` untuk *idempotent* data ingestion.
- Hindari `deleteMany({})` di produksi tanpa filter ketat.

---

## D. KEGIATAN PRAKTIKUM (120 MENIT)

### Fase 1: Persiapan Dataset (15 menit)
1. Jalankan MongoDB daemon.
2. Buka terminal, masuk ke shell: `mongosh`.
3. Unduh file `log_produksi.json` yang disediakan (berisi 100+ dokumen).
4. Import dengan `mongoimport`:
   ```bash
   mongoimport --db praktikum2 --collection log_produksi --file log_produksi.json --jsonArray
   ```
5. Cek di shell: `use praktikum2`, `db.log_produksi.countDocuments()`.

**Jika mongoimport tidak tersedia**, mahasiswa bisa menyalin isi file dan menjalankan `insertMany` di shell.

### Fase 2: Eksplorasi Insert & Query Dasar (20 menit)
1. Sisipkan `insertMany` dengan 10 dokumen manual yang mencakup beberapa mesin (`M01`, `M02`, `M03`) dan variasi jam.
2. Lakukan query:
   - Semua dokumen dengan proyeksi hanya `mesin`, `produk`, `reject`.
   - Dokumen dengan `reject > 3`.
   - Dokumen `mesin: M03` **atau** `reject == 0`.
   - Dokumen yang memiliki field `operator` (gunakan `$exists`).
3. Gunakan `$in` untuk mencari produk "Baut 10mm" dan "Mur 8mm" sekaligus.

### Fase 3: Update Mendalam (35 menit)
1. **$set dan $unset:**
   - Pilih satu dokumen, tambahkan field ``status``: ``"checked"``.
   - Hapus field `operator` pada dokumen yang memilikinya.
2. **$inc:**
   - Naikkan semua `reject` mesin `M01` sebesar 1.
3. **$push pada array:**
   - Buat koleksi `mesin` dengan 3 dokumen yang punya field `history_error` berupa array kosong.
   - Tambahkan string error ke `history_error` mesin tertentu dengan `$push`.
   - Gunakan `$each` untuk menambah sekaligus 3 error.
4. **$pull:**
   - Hapus elemen tertentu dari array `history_error`.
5. **Upsert:**
   - Coba update mesin dengan kode `M10` dengan `$set` dan `{ upsert: true }`. Verifikasi dokumen baru tercipta.

### Fase 4: Delete & Cleanup (15 menit)
1. Hapus dokumen yang `reject`-nya di atas 10 dengan `deleteMany`.
2. Hapus dokumen yang tidak memiliki field `produk` (`$exists: false`).
3. Coba `deleteOne` pada salah satu dokumen dengan `_id` spesifik.

### Fase 5: Bekerja dengan Tipe Data (15 menit)
1. Query dokumen berdasarkan rentang tanggal.
   ```javascript
   db.log_produksi.find({ waktu: { $gte: ISODate("2026-05-04T08:00:00Z"), $lte: ISODate("2026-05-04T12:00:00Z") } })
   ```
2. Cari dokumen yang memiliki field `biaya` bertipe number.
   ```javascript
   db.maintenance.find({ biaya: { $type: "number" } })
   ```
   (Koleksi maintenance bisa dibuat dadakan dengan beberapa field biaya bertipe string untuk melihat beda.)

### Fase 6: Diskusi dan Refleksi (20 menit)
Diskusikan:
- Kapan `$inc` lebih baik daripada membaca lalu menulis ulang nilai?
- Mengapa `$push` dengan `$slice` penting untuk histori sensor?
- Bagaimana cara aman menghapus data lama (data retention policy)?

---

## E. LATIHAN MANDIRI
**Kerjakan di shell, simpan perintah.**

1. Gunakan database `latihan2_<NIM>`.
2. Buat koleksi `alat` berisi 10 dokumen alat berat dengan field: `kode`, `nama`, `tipe`, `tahun`, `jam_operasi` (number), `komponen` (array string part).
3. Lakukan:
   - Insert 10 dokumen dengan variasi.
   - Update jam_operasi alat tertentu menggunakan `$inc` (tambah 50 jam).
   - Tambahkan komponen "filter oli" ke semua alat yang belum memiliki komponen tersebut (`$addToSet`).
   - Hapus alat yang tahunnya < 2010 (`$lt`).
   - Cari alat yang memiliki komponen "bearing" **atau** "pompa" (`$in` pada array).
   - Tarik semua alat dengan proyeksi tanpa `_id`.
4. Dokumentasikan setiap langkah.

---

## F. STUDI KASUS: SISTEM TRACKING MAINTENANCE MESIN
**Deskripsi:** Sebuah pabrik memiliki jadwal maintenance untuk setiap mesin. Data maintenance meliputi: `kode_mesin`, `jenis_perawatan`, `tanggal_jadwal`, `teknisi`, `biaya`, `status` (terjadwal/selesai/ditunda), dan array `parts_diganti` (sparepart yang digunakan). Saat teknisi menyelesaikan perawatan, mereka meng-update status dan menambah array `parts_diganti`. Terkadang jadwal baru dimasukkan dengan `upsert` jika belum ada.

**Tugas:**
1. Buat database `studi_kasus_2` dan koleksi `maintenance`.
2. Masukkan 5 jadwal awal (status "terjadwal").
3. Update satu jadwal menjadi "selesai", tambahkan biaya aktual (field baru), dan push beberapa part ke `parts_diganti`.
4. Buat query:
   a. Semua jadwal yang statusnya "terjadwal" dan tanggal_jadwal antara hari ini dan 7 hari ke depan.
   b. Semua maintenance yang menggunakan part "bearing" (cari di array `parts_diganti`).
   c. Maintenance yang tidak memiliki field `teknisi` (mungkin data kurang).
5. Hapus semua jadwal yang statusnya "ditunda" dan sudah lewat 30 hari.
6. Sertakan perintah dan output screenshot.

**Analisis:** Jelaskan keuntungan menggunakan array `parts_diganti` daripada menyimpan part di koleksi terpisah untuk kasus ini.

---

## G. TUGAS TERSTRUKTUR
### Deskripsi
Membangun basis data sederhana untuk *quality control* produksi.

### Spesifikasi
1. Buat database `qc_<NIM>`.
2. Buat koleksi `inspeksi` dengan 20 dokumen. Setiap dokumen:
   - `nomor_batch` (string),
   - `mesin` (string),
   - `produk` (string),
   - `jumlah_produksi` (number),
   - `jumlah_cacat` (number),
   - `jenis_cacat` (array string, misal `["gores", "retak"]`),
   - `inspektor` (string),
   - `waktu_inspeksi` (Date).
3. Sisipkan data dengan variasi yang mencakup setidaknya 3 mesin, 4 produk.
4. Tunjukkan operasi:
   a. Insert 20 dokumen sekaligus.
   b. Update batch tertentu: jika `jumlah_cacat > 10`, set field `status` menjadi "reject" dan `$inc` jumlah_cacat menjadi 0 (reset) sambil menambah catatan.
   c. Tambahkan jenis cacat baru ke batch yang sudah ada dengan `$addToSet`.
   d. Tarik semua batch dari mesin `M-A` dengan `jumlah_cacat > 5` dan tampilkan hanya `nomor_batch`, `jenis_cacat`, dan `inspektor`.
   e. Hapus batch yang `jumlah_produksi`-nya kurang dari 50.
5. Simpan semua perintah dalam file teks, beri komentar.

### Komponen Penilaian
| Kriteria                         | Bobot |
|----------------------------------|-------|
| Insert (struktur, variasi data)  | 20%   |
| Update dengan operator tepat     | 25%   |
| Query dengan filter & proyeksi   | 25%   |
| Delete dengan kondisi logis      | 15%   |
| Dokumentasi perintah & komentar  | 15%   |

### Tenggat
Dikumpulkan di LMS sebelum pertemuan ke-3, format PDF + file teks perintah.

---

## H. REFERENSI PERTEMUAN 2
1. MongoDB, Inc. *MongoDB CRUD Operations*. https://docs.mongodb.com/manual/crud/
2. MongoDB University. *M001: MongoDB Basics* (Chapter 2: CRUD).
3. Bradshaw, S., et al. *MongoDB: The Definitive Guide*, O'Reilly.

---

**Catatan Penting:**
Pastikan Anda memahami setiap operator dan kapan menggunakannya. Praktikkan langsung di shell. Jika ada error, baca pesan error dan periksa sintaks. Jangan ragu bertanya di forum LMS.
