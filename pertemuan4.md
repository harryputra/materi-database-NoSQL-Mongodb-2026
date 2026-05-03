# PERTEMUAN 4 – INDEXING DAN OPTIMASI QUERY
**Program Studi:** Teknologi Rekayasa Informatika Industri (TRIN)
**Alokasi Waktu:** 1 × 150 menit (praktikum)

---

## A. CAPAIAN PEMBELAJARAN PERTEMUAN 4
Setelah pertemuan ini mahasiswa mampu:
1. Menjelaskan fungsi indeks dan dampaknya terhadap performa query.
2. Membuat berbagai jenis indeks: *single field*, *compound*, *multikey*, *text*, dan *TTL*.
3. Menganalisis rencana eksekusi query menggunakan `explain()` pada level `queryPlanner`, `executionStats`, dan `allPlansExecution`.
4. Memilih dan merancang indeks yang sesuai untuk beban kerja industri (time‑series sensor, laporan produksi, pencarian teks pada catatan maintenance).
5. Memahami konsep *covered query* dan *index intersection* untuk optimasi lanjut.
6. Mengelola indeks (melihat, menghapus) serta menyadari *trade‑off* antara kecepatan baca dan tulis.

---

## B. ALAT DAN BAHAN
- Server MongoDB aktif.
- MongoDB Shell (`mongosh`).
- MongoDB Compass (untuk melihat indeks dan explain plan secara visual).
- Dataset besar: minimal 100.000 dokumen simulasi data sensor (dapat di‑generate oleh script Python/shell yang disediakan dosen atau dibuat sendiri).
- Tools opsional: `mgenerate` (untuk data dummy), atau script Node.js/Python.

---

## C. URAIAN MATERI SUPER LENGKAP

### 1. Pentingnya Indeks dalam Basis Data Industri
- Tanpa indeks, MongoDB melakukan *collection scan* (COLLSCAN), membaca semua dokumen satu per satu. Pada 100.000 dokumen, satu query bisa memakan ratusan milidetik.
- Dengan indeks yang tepat, query dapat langsung menuju dokumen yang relevan (*index scan*/IXSCAN) dalam orde milidetik.
- Di lingkungan industri, data tumbuh cepat (sensor tiap detik). Indeks yang baik menjadi kunci agar dashboard monitoring, laporan, dan alarm tetap responsif.

### 2. Konsep Dasar Indeks
- **Indeks** adalah struktur data (B‑tree) yang menyimpan subset field dari koleksi secara terurut, memungkinkan pencarian cepat.
- Setiap indeks mempercepat pembacaan, tetapi memperlambat penulisan (insert, update, delete) karena indeks harus diperbarui.
- Indeks disimpan di RAM jika memungkinkan. Pastikan RAM server cukup menampung *working set* (indeks + data yang sering diakses).

#### a. Struktur B‑tree
MongoDB menggunakan B‑tree yang menyeimbangkan diri. Setiap node berisi key dan pointer. Pencarian O(log n).

#### b. Default Index `_id`
Setiap koleksi memiliki indeks unik pada field `_id` secara otomatis. Indeks ini tidak bisa dihapus.

### 3. Jenis‑Jenis Indeks

#### a. Single Field Index
Indeks pada satu field, ascending (1) atau descending (-1). Arah penting untuk *compound index* dan sorting.
```javascript
db.sensor.createIndex({ suhu: 1 })
```

#### b. Compound Index
Indeks pada beberapa field. Urutan field sangat penting: indeks dapat mendukung query pada prefix (awalan) field tersebut.
```javascript
db.sensor.createIndex({ mesin: 1, timestamp: -1 })
```
Indeks ini mendukung query:
- `{ mesin: "M001" }`
- `{ mesin: "M001", timestamp: { $gte: ... } }`
- `{ mesin: "M001" }` dengan sort `{ timestamp: -1 }`

Tidak mendukung optimal query pada `timestamp` saja (tanpa `mesin`), kecuali dengan *index intersection* (terbatas).

**Aturan prefix:** Indeks `(a, b, c)` mendukung query `(a)`, `(a,b)`, `(a,b,c)`, tapi tidak `(b)` atau `(c)` saja. Namun bisa digunakan untuk sort pada `(a,b)` atau `(a,b,c)`.

#### c. Multikey Index
Digunakan untuk field yang berisi array. MongoDB membuat satu entri indeks untuk setiap elemen array.
```javascript
db.mesin.createIndex({ "tags": 1 })
```
Jika dokumen memiliki `tags: ["CNC", "presisi", "berat"]`, indeks akan memiliki tiga key terpisah. Query `tags: "CNC"` akan menggunakan indeks ini.

**Catatan:** Anda tidak bisa membuat *compound index* dengan dua field array secara bersamaan (mencegah ledakan kombinatorial). Satu indeks hanya boleh mengandung paling banyak satu field array.

#### d. Text Index
Untuk pencarian teks lengkap (full‑text search) pada string.
```javascript
db.maintenance.createIndex({ keterangan: "text", teknisi: "text" })
db.maintenance.find({ $text: { $search: "bocor oli" } })
```
- Text index bersifat *case‑insensitive* dan mendukung stemming (untuk bahasa tertentu, perlu mengatur `default_language`).
- Satu koleksi hanya boleh memiliki **satu** text index, tetapi bisa mencakup banyak field.
- Query dengan `$text` akan mengurutkan hasil berdasarkan skor relevansi (`textScore`).

#### e. TTL (Time‑To‑Live) Index
Indeks khusus yang otomatis menghapus dokumen setelah waktu tertentu. Sangat berguna untuk data kadaluarsa industri (log sementara, session).
```javascript
db.sensor.createIndex({ timestamp: 1 }, { expireAfterSeconds: 2592000 }) // 30 hari
```
- Field harus bertipe Date.
- MongoDB menjalankan thread penghapus setiap 60 detik.
- Cocok untuk membersihkan data sensor yang sudah tidak diperlukan.

#### f. Unique Index
Menjamin nilai field unik di seluruh koleksi. Selain `_id`, kita bisa membuatnya untuk field lain.
```javascript
db.mesin.createIndex({ kode_mesin: 1 }, { unique: true })
```
Berguna untuk memastikan tidak ada duplikasi kode mesin, nomor batch, dll.

#### g. Sparse Index
Hanya mengindeks dokumen yang memiliki field tersebut (tidak termasuk yang field‑nya tidak ada).
```javascript
db.log_produksi.createIndex({ catatan: 1 }, { sparse: true })
```
Berguna untuk menghemat ruang jika banyak dokumen tidak memiliki field opsional.

#### h. Partial Index
Hanya mengindeks subset dokumen yang memenuhi kondisi filter tertentu.
```javascript
db.log_produksi.createIndex(
  { mesin: 1 },
  { partialFilterExpression: { reject: { $gt: 0 } } }
)
```
Indeks ini hanya akan berisi data produksi yang memiliki cacat. Lebih kecil dan cepat.

#### i. Wildcard Index
Mengindeks semua field atau field dengan pola tertentu. Berguna untuk koleksi yang skemanya sangat dinamis.
```javascript
db.sensor.createIndex({ "$**": 1 })
```
Hati‑hati karena bisa besar dan lambat untuk penulisan.

### 4. Opsi Pembuatan Indeks
- `background: true` (versi lama, sekarang tidak perlu) – di MongoDB 4.2+ pembangunan indeks default di background.
- `unique: true`
- `sparse: true`
- `name: "custom_name"` – memberi nama indeks sendiri.
- `expireAfterSeconds` – untuk TTL.
- `partialFilterExpression`
- `collation` – untuk pengaturan bahasa.

### 5. Menganalisis Performa dengan `explain()`
Metode `explain()` memberikan detail rencana eksekusi query. Tiga mode:
- `queryPlanner` (default): menampilkan rencana yang dipilih tanpa eksekusi.
- `executionStats`: mengeksekusi query dan mengumpulkan statistik.
- `allPlansExecution`: mengeksekusi semua rencana kandidat.

```javascript
db.sensor.find({ mesin: "M001" }).explain("executionStats")
```
**Bagian penting:**
- `winningPlan.stage`: `IXSCAN` (pakai indeks) atau `COLLSCAN`.
- `executionTimeMillis`: waktu eksekusi.
- `totalDocsExamined` vs `nReturned`: jika `totalDocsExamined` jauh lebih besar, indeks mungkin tidak optimal atau ada filter tambahan.
- `totalKeysExamined`: jumlah key indeks yang diperiksa.

**Covered Query**: Jika semua field yang diminta query ada di dalam indeks, MongoDB tidak perlu membaca dokumen asli (`totalDocsExamined: 0`). Ini sangat cepat.
```javascript
db.sensor.find({ mesin: "M001" }, { _id: 0, mesin: 1, timestamp: 1 }).explain("executionStats")
```
Jika indeks mencakup `mesin` dan `timestamp`, dan proyeksi hanya meminta keduanya, maka akan menjadi covered query.

**Index Intersection**: MongoDB dapat menggunakan lebih dari satu indeks untuk satu query, lalu menginterseksi hasilnya. Namun biasanya satu compound index lebih baik.

### 6. Strategi Indeks untuk Data Time‑Series (Industri)
Data sensor umumnya di‑query berdasarkan:
- Rentang waktu (`timestamp`) untuk seluruh mesin.
- Mesin tertentu + rentang waktu.
- Nilai sensor tertentu (misal suhu > 80) dalam waktu terakhir.

**Rekomendasi:**
- Indeks compound `(mesin, timestamp)` untuk query per mesin.
- Indeks `(timestamp)` untuk query global.
- Jika sering query `(timestamp, mesin)`, pertimbangkan compound `(timestamp, mesin)`.
- Gunakan TTL untuk menghapus data lama otomatis.

**Contoh indeks untuk koleksi `sensor`:**
```javascript
db.sensor.createIndex({ mesin: 1, timestamp: -1 })
db.sensor.createIndex({ timestamp: -1 })
```

### 7. Indeks pada Embedded Document
MongoDB mendukung indeks pada field di dalam dokumen bersarang.
```javascript
db.mesin.createIndex({ "spesifikasi.max_rpm": 1 })
db.mesin.find({ "spesifikasi.max_rpm": { $gte: 10000 } })
```
Ini akan menggunakan indeks tersebut.

### 8. Memantau Ukuran dan Pengaruh Indeks
- `db.collection.getIndexes()` melihat semua indeks.
- `db.collection.totalIndexSize()` ukuran total indeks.
- `db.collection.stats()` statistik koleksi termasuk indeks.
- Hapus indeks yang tidak terpakai dengan `dropIndex("nama")`.

### 9. Trade‑off dan Best Practices
- Setiap insert/update/delete harus memperbarui indeks. Terlalu banyak indeks akan menurunkan throughput tulis.
- Di sistem sensor dengan laju tulis sangat tinggi, batasi indeks seminimal mungkin untuk mempertahankan kecepatan.
- Pantau *slow queries* dengan MongoDB Profiler (`db.setProfilingLevel(1, 100)`) untuk mengidentifikasi query yang perlu diindeks.
- Gunakan indeks *partial* untuk mengurangi ukuran jika data memiliki subset yang jarang diakses.
- Untuk koleksi besar, bangun indeks saat off‑peak.

---

## D. KEGIATAN PRAKTIKUM (120 MENIT)

### Fase 1: Persiapan Dataset Besar (20 menit)
1. Buka shell dan gunakan (atau buat) database `praktikum4`.
2. Buat koleksi `sensor_mentah` dan isi dengan 100.000 dokumen. Dosen menyediakan script Python sederhana (`generate_sensor.py`) yang akan dijalankan mahasiswa. Script ini menggunakan PyMongo untuk insertMany dalam batch. (Jika tidak memungkinkan, mahasiswa dapat mengimpor file JSON yang disediakan).
3. Pastikan dataset memiliki field: `mesin` (10 mesin berbeda), `suhu` (50–100), `getaran` (0.1–2.0), `timestamp` (acak dalam 30 hari terakhir).
4. Cek jumlah dokumen: `db.sensor_mentah.countDocuments()`.

### Fase 2: Query Tanpa Indeks & Analisis (20 menit)
1. Jalankan query tanpa indeks:
   ```javascript
   db.sensor_mentah.find({ suhu: { $gt: 90 } }).explain("executionStats")
   ```
   Catat `executionTimeMillis`, `totalDocsExamined`, `stage`.
2. Query dengan mesin tertentu:
   ```javascript
   db.sensor_mentah.find({ mesin: "M01" }).explain("executionStats")
   ```
3. Diskusikan hasil: COLLSCAN, waktu tinggi.

### Fase 3: Membuat Indeks dan Menganalisis Perubahan (40 menit)
1. Buat indeks pada `mesin`:
   ```javascript
   db.sensor_mentah.createIndex({ mesin: 1 })
   ```
   Ulangi query mesin M01, bandingkan `executionStats`. Harapannya kini `IXSCAN`.
2. Buat indeks pada `suhu` ascending, coba query `suhu > 90` lagi.
3. Buat **compound index** `(mesin, timestamp)`:
   ```javascript
   db.sensor_mentah.createIndex({ mesin: 1, timestamp: -1 })
   ```
   Lakukan query:
   ```javascript
   db.sensor_mentah.find({ mesin: "M02", timestamp: { $gte: ISODate("2026-04-20") } }).explain("executionStats")
   ```
   Amati penggunaan indeks dan waktu.
4. Uji **prefix** rule:
   - Query hanya `{ mesin: "M02" }` → harus pakai indeks.
   - Query hanya `{ timestamp: ... }` → mungkin COLLSCAN jika tidak ada indeks timestamp. Diskusikan mengapa.
5. Buat indeks tambahan `{ timestamp: -1 }` untuk query global.
6. Buat **TTL index** pada `timestamp` dengan masa 1 jam (3600 detik) untuk demonstrasi. Lihat dokumen yang lebih dari 1 jam akan hilang setelah beberapa saat (untuk keperluan praktik, buat saja lalu hapus indeksnya setelah dipahami).
7. Buat **text index** pada koleksi `maintenance` (bisa buat baru dengan field `deskripsi`) dan lakukan pencarian teks.

### Fase 4: Covered Query (15 menit)
1. Buat indeks compound `(mesin, suhu)`.
2. Lakukan query dengan proyeksi hanya `mesin` dan `suhu`, tanpa `_id`:
   ```javascript
   db.sensor_mentah.find({ mesin: "M03" }, { _id: 0, mesin: 1, suhu: 1 }).explain("executionStats")
   ```
   Periksa `totalDocsExamined` = 0, stage `PROJECTION_COVERED`.
3. Diskusikan manfaatnya untuk dashboard yang hanya butuh field tertentu.

### Fase 5: Melihat dan Menghapus Indeks (10 menit)
- `db.sensor_mentah.getIndexes()`
- `db.sensor_mentah.totalIndexSize()`
- Hapus indeks yang tidak perlu: `db.sensor_mentah.dropIndex("mesin_1")`

### Fase 6: Evaluasi & Diskusi (15 menit)
Bahas:
- Berapa jumlah indeks ideal untuk koleksi sensor dengan 10.000 tulis/detik?
- Kapan menggunakan *partial index*?
- Bagaimana menganalisis slow query dari log?

---

## E. LATIHAN MANDIRI
1. Gunakan database `latihan4` dan koleksi `data_log` yang berisi 10.000 dokumen dengan field: `device`, `event_code`, `severity`, `timestamp`, `message` (string).
2. Lakukan query tanpa indeks dan catat statistiknya.
3. Buat indeks yang sesuai untuk query:
   - `{ device: "...", timestamp: { $gte: ... } }`
   - `{ severity: "critical" }`
   - Pencarian teks pada `message` dengan kata kunci "failure".
4. Uji dengan `explain("executionStats")` dan bandingkan.
5. Lakukan eksperimen kecil: Tambahkan 1000 dokumen baru, ukur waktu insert sebelum dan sesudah indeks dihapus (gunakan `Date.now()` di shell atau script).
6. Tulis laporan singkat hasil pengamatan.

---

## F. STUDI KASUS: OPTIMASI SISTEM MONITORING KUALITAS UDARA PABRIK
**Deskripsi:**
Sebuah pabrik kimia memiliki 30 sensor kualitas udara yang mengirim data setiap 10 detik. Data disimpan di koleksi `air_quality` dengan struktur:
- `sensor_id` (string)
- `co2_ppm` (int)
- `voc_ppb` (int)
- `pm25` (double)
- `timestamp` (Date)
- `lokasi` (subdokumen: `{ gedung: string, lantai: int }`)
- `catatan` (string, optional)

**Kebutuhan query yang harus cepat:**
1. Dashboard menampilkan data terbaru setiap sensor (query per `sensor_id` sorted by `timestamp` desc limit 1).
2. Laporan rata‑rata harian per gedung (aggregasi nanti, tapi perlu filter rentang waktu dan mungkin `lokasi.gedung`).
3. Alarm jika `pm25 > 150` dalam 5 menit terakhir.
4. Pencarian sensor yang memiliki catatan mengandung "kalibrasi" (teks).

**Tugas Anda:**
1. Analisis kebutuhan dan tentukan indeks yang harus dibuat.
2. Buat koleksi dan masukkan 20.000 data simulasi (bisa dengan script yang dimodifikasi).
3. Implementasikan indeks yang menurut Anda optimal.
4. Uji setiap skenario query dengan `explain` dan catat hasilnya (sebelum vs sesudah indeks).
5. Sampaikan rekomendasi final dalam laporan.
6. Sertakan perintah yang digunakan dan screenshot.

---

## G. TUGAS TERSTRUKTUR
### Deskripsi
Melakukan audit dan optimasi indeks pada dataset produksi.

### Spesifikasi
1. Gunakan koleksi `log_produksi` dari pertemuan sebelumnya, atau buat baru dengan 50.000 dokumen. Struktur: `mesin`, `produk`, `jumlah_ok`, `jumlah_reject`, `waktu` (Date), `shift` (1,2,3), `operator` (string).
2. Identifikasi 3 query yang sering dijalankan (buat sendiri skenarionya, misal: "total reject per mesin pada shift 1", "data mesin M01 dalam 7 hari terakhir", "pencarian operator dengan nama tertentu").
3. Sebelum membuat indeks baru, catat `executionStats` untuk setiap query.
4. Rancang dan buat indeks yang diperlukan. Jangan membuat indeks berlebihan, pilih yang paling efektif berdasarkan aturan prefix.
5. Setelah indeks dibuat, uji kembali query dan bandingkan.
6. Buat juga satu *partial index* untuk mengindeks hanya data dengan `jumlah_reject > 0`, karena sebagian besar query analisis cacat hanya butuh itu.
7. Buat analisis perbandingan dalam tabel: Query, Waktu sebelum (ms), Waktu sesudah (ms), Indeks yang digunakan, Stage, Docs Examined.
8. Tulis laporan (PDF) serta lampirkan semua perintah dan output.

### Rubrik Penilaian
| Kriteria                        | Bobot |
|---------------------------------|-------|
| Pemahaman kebutuhan indeks      | 20%   |
| Pembuatan indeks yang tepat     | 25%   |
| Penggunaan explain & analisis   | 25%   |
| Partial index & inovasi         | 15%   |
| Dokumentasi dan laporan         | 15%   |

### Tenggat
Sebelum pertemuan ke-5, unggah di LMS.

---

## H. REFERENSI PERTEMUAN 4
1. MongoDB, Inc. *Indexes*. https://docs.mongodb.com/manual/indexes/
2. MongoDB University. *M201: MongoDB Performance* (Free Course).
3. Murphy, T. (2021). *Practical MongoDB Performance*. Apress.
4. MongoDB. *Explain Results*. https://docs.mongodb.com/manual/reference/explain-results/

---

**Pesan Dosen:**
Indeks adalah pedang bermata dua. Gunakan dengan bijak, ukur selalu, dan jangan lupa bahwa setiap indeks memakan ruang dan memperlambat penulisan. Selamat mengoptimalkan!
