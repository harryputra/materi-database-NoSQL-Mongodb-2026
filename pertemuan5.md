# PERTEMUAN 5 – AGGREGATION FRAMEWORK
**Program Studi:** Teknologi Rekayasa Informatika Industri (TRIN)
**Alokasi Waktu:** 7.30-16.00 (praktikum)

---

## A. CAPAIAN PEMBELAJARAN PERTEMUAN 5
Setelah mengikuti perkuliahan ini, mahasiswa mampu:
1. Menjelaskan konsep aggregation pipeline dan cara kerjanya.
2. Menggunakan stage-stage utama: `$match`, `$group`, `$sort`, `$project`, `$unwind`, `$lookup`, `$addFields`, `$bucket`, `$facet`.
3. Menerapkan akumulator (`$sum`, `$avg`, `$min`, `$max`, `$push`, `$addToSet`, `$first`, `$last`) untuk menghasilkan ringkasan data industri.
4. Memanfaatkan operator ekspresi (`$dateToString`, `$hour`, `$cond`, `$ifNull`, `$multiply`, `$divide`) untuk transformasi data di dalam pipeline.
5. Menggabungkan data dari beberapa koleksi menggunakan `$lookup` (join).
6. Merancang pipeline agregasi untuk menjawab pertanyaan bisnis nyata: total produksi per mesin, rata‑rata suhu per jam, reject rate, OEE, distribusi nilai sensor.
7. Mengoptimasi pipeline dengan menempatkan `$match` dan `$limit` di awal, serta memproyeksikan field yang diperlukan saja.

---

## B. ALAT DAN BAHAN
- Server MongoDB aktif.
- MongoDB Shell (`mongosh`).
- MongoDB Compass (sangat membantu untuk membangun pipeline secara visual dan melihat hasil tiap stage).
- Dataset: koleksi `log_produksi`, `sensor`, `maintenance` dari pertemuan sebelumnya (atau disediakan dataset baru). Minimal 1000 dokumen untuk agregasi bermakna.
- Script Python (opsional) untuk menambah volume data.

---

## C. URAIAN MATERI

### 1. Konsep Aggregation Pipeline
Aggregation framework adalah alat utama di MongoDB untuk memproses, mengubah, dan menganalisis data. Data mengalir melalui serangkaian **stage**, di mana output dari satu stage menjadi input stage berikutnya.

```javascript
db.collection.aggregate( [ stage1, stage2, ... ] )
```

Setiap stage adalah dokumen yang mendeskripsikan operasi. MongoDB mengeksekusi pipeline secara berurutan, dan dapat mengoptimasi dengan menggabungkan `$match` dan `$sort` dengan indeks di awal pipeline.

**Keuntungan agregasi:**
- Semua pemrosesan terjadi di server, mengurangi data yang dikirim ke klien.
- Ekspresif dan lebih cepat dibandingkan banyak query di aplikasi.
- Mendukung operasi kompleks (group, unwind, join) dalam satu perintah.

### 2. Stage dan Contoh Detail (dengan konteks industri)

#### a. `$match`
Memfilter dokumen, sama dengan query filter. Letakkan **paling awal** untuk mengurangi dokumen yang diproses selanjutnya, terutama jika didukung indeks.
```javascript
{ $match: { mesin: "M05", waktu: { $gte: ISODate("2026-04-10") } } }
```

#### b. `$project`
Memilih field yang akan diteruskan, menambahkan field baru hasil komputasi, atau menghilangkan field. Bisa juga merubah struktur.
```javascript
{ $project: { _id: 0, mesin: 1, suhu: 1, tahun: { $year: "$timestamp" } } }
```
Setelah `$project`, hanya field yang disebutkan yang lolos.

#### c. `$addFields` (alias `$set`)
Menambahkan field baru tanpa menghilangkan field yang sudah ada.
```javascript
{ $addFields: { reject_rate: { $divide: ["$reject", "$jumlah"] } } }
```
Ini berguna untuk menambahkan hasil kalkulasi.

#### d. `$group`
Stage paling penting. Mengelompokkan dokumen berdasarkan `_id` dan menghitung agregat menggunakan **akumulator**.
- `$sum` – menjumlahkan nilai.
- `$avg` – rata‑rata.
- `$min`, `$max` – nilai minimum/maksimum.
- `$push` – membuat array seluruh nilai.
- `$addToSet` – membuat array nilai unik.
- `$first`, `$last` – mengambil nilai pertama/terakhir dalam kelompok (perlu `$sort` sebelumnya).
- `$count` – jumlah dokumen dalam grup (sama dengan `{ $sum: 1 }`).

**Contoh:**
```javascript
{ $group: {
    _id: "$mesin",
    total_produksi: { $sum: "$jumlah" },
    rata_reject: { $avg: "$reject" }
}}
```
`_id` bisa berupa ekspresi, misal `{ mesin: "$mesin", shift: "$shift" }` untuk grup komposit.

#### e. `$sort`
Mengurutkan dokumen berdasarkan field. Arah: 1 (ascending), -1 (descending).
```javascript
{ $sort: { total_produksi: -1 } }
```
Jika diletakkan setelah `$group`, mengurutkan hasil agregasi. Bisa juga sebelum `$group` untuk memengaruhi `$first`/`$last`.

#### f. `$unwind`
Memecah field array menjadi beberapa dokumen, satu untuk tiap elemen. Setiap dokumen baru menggantikan array dengan nilai elemen.
```javascript
// Sebelum: { _id: 1, sensor: ["temp", "press"] }
// Sesudah $unwind "$sensor":
// { _id: 1, sensor: "temp" }
// { _id: 1, sensor: "press" }
```
Jika array kosong, dokumen hilang. Gunakan `preserveNullAndEmptyArrays: true` untuk mempertahankannya.

#### g. `$lookup` (Join)
Melakukan left outer join dengan koleksi lain.
```javascript
{ $lookup: {
    from: "mesin_info",      // koleksi asing
    localField: "mesin",     // field di koleksi saat ini
    foreignField: "_id",     // field di koleksi asing
    as: "detail_mesin"      // nama field hasil
}}
```
Hasilnya adalah array `detail_mesin`. Untuk mengambil elemen pertama, gunakan `$unwind` atau `$first`.

#### h. `$bucket` dan `$bucketAuto`
Mengelompokkan dokumen ke dalam interval (bucket).
```javascript
{ $bucket: {
    groupBy: "$suhu",
    boundaries: [ 0, 50, 70, 90, 100 ],
    default: ">100",
    output: { count: { $sum: 1 } }
}}
```
Berguna untuk histogram suhu, analisis distribusi kinerja.

#### i. `$facet`
Menjalankan beberapa pipeline secara paralel pada input yang sama dan menghasilkan satu dokumen dengan beberapa array hasil.
```javascript
{ $facet: {
    "total_per_mesin": [ ... ],
    "reject_tertinggi": [ ... ]
}}
```
Memberikan banyak laporan dalam satu query. Cocok untuk dashboard.

### 3. Ekspresi dalam Aggregation
Ekspresi digunakan di dalam `$project`, `$group`, `$addFields`, dll. Beberapa yang penting:

**String & Date:**
- `$concat`, `$toUpper`, `$substr`
- `$dateToString: { format: "%Y-%m-%d", date: "$timestamp" }`
- `$year`, `$month`, `$dayOfMonth`, `$hour`, `$minute`, `$week`

**Aritmatika:**
- `$add`, `$subtract`, `$multiply`, `$divide`, `$mod`

**Logika:**
- `$cond: { if: { $gte: ["$suhu", 85] }, then: "tinggi", else: "normal" }`
- `$ifNull: ["$field", "default"]`
- `$and`, `$or`, `$not`

**Akumulator khusus `$group`:**
- `$accumulator` (custom JavaScript, hindari kecuali perlu)
- `$mergeObjects` (menggabungkan dokumen)

### 4. Optimasi Aggregation Pipeline
- **Posisi `$match`**: Tempatkan `$match` sedini mungkin.
- **`$limit` awal**: Jika hanya butuh beberapa dokumen teratas, batasi sebelum operasi mahal.
- **Proyeksi awal**: `$project` untuk membuang field yang tidak diperlukan bisa mengurangi ukuran dokumen.
- **Indeks**: `$match` dan `$sort` di awal pipeline dapat memanfaatkan indeks.
- **Hindari `$unwind` besar-besaran**: Jika array besar, pertimbangkan untuk menyimpan data secara terpisah (reference) agar tidak perlu unwind.
- **Gunakan `$facet` dengan hati-hati**: karena menjalankan banyak pipeline, bisa berat.
- **Pantau dengan `explain()`**: `aggregate(...).explain("executionStats")` untuk melihat statistik dan indeks yang digunakan.

### 5. Pipeline Transformatif dalam Industri
Beberapa pola analisis:
- **Rata‑rata bergerak (moving avg):** Butuh window function, tidak ada stage bawaan. Bisa dilakukan dengan `$lookup` pada koleksi sendiri atau dengan aplikasi.
- **Pivot table:** Gunakan `$group` dengan multiple akumulator. Untuk pivot dinamis bisa menggunakan `$facet`.
- **Hierarki grouping:** Group bertingkat bisa dilakukan dengan beberapa `$group`, atau dengan sub-dokumen `_id`.

**Contoh menghitung OEE (Overall Equipment Effectiveness) sederhana** akan dipraktikkan dalam studi kasus di bawah.

### 6. Penggunaan `$merge` dan `$out`
- `$out`: menulis hasil agregasi ke koleksi baru (menimpa jika ada). Berguna untuk laporan materialized.
- `$merge`: lebih fleksibel, dapat menambah, mengganti, atau memperbarui koleksi target berdasarkan kondisi.

```javascript
{ $merge: { into: "laporan_harian", on: "_id", whenMatched: "replace", whenNotMatched: "insert" } }
```

### 7. Validasi Skema dengan `$jsonSchema` (Opsional)
Aggregation juga bisa digunakan untuk memvalidasi atau membersihkan data.

---


## D. KEGIATAN PRAKTIKUM (120 MENIT)

# TUTORIAL TERBIMBING STUDI KASUS AGREGASI
**Analisis Efektivitas Produksi – Menghitung OEE (Overall Equipment Effectiveness)**

---

## A. DESKRIPSI STUDI KASUS
Sebuah pabrik elektronik ingin mengevaluasi kinerja mesin‑mesin produksinya menggunakan metrik **OEE (Overall Equipment Effectiveness)**. Data produksi harian dicatat dalam koleksi `produksi_harian` dengan struktur:

| Field                   | Tipe    | Keterangan                                 |
|-------------------------|---------|--------------------------------------------|
| `mesin`                 | string  | Kode mesin (M01 – M10)                     |
| `tanggal`               | Date    | Tanggal produksi                           |
| `shift`                 | int     | Shift (1,2,3)                              |
| `target`                | int     | Target produksi (unit)                     |
| `actual_ok`             | int     | Hasil baik aktual                          |
| `actual_reject`         | int     | Hasil cacat                                |
| `durasi_operasi_menit`  | int     | Waktu operasi aktual (menit)               |
| `durasi_tersedia_menit` | int     | Waktu tersedia (menit)                     |

**Manajemen membutuhkan laporan:**
1. **OEE** setiap mesin per bulan.
   Rumus OEE = Availability × Performance × Quality
   - **Availability** = `durasi_operasi / durasi_tersedia`
   - **Performance** = `(actual_ok + actual_reject) / (target × (durasi_operasi / durasi_tersedia))`
     *(Di sini kita sederhanakan performance = actual_ok / target, karena data aktual sudah memperhitungkan kecepatan)*
   - **Quality** = `actual_ok / (actual_ok + actual_reject)`

   Namun agar lebih mudah dipahami, kita gunakan pendekatan praktis:
   **OEE = (actual_ok / target) × (durasi_operasi / durasi_tersedia) × (actual_ok / (actual_ok + actual_reject))**

2. Mesin dengan OEE di bawah **80%** pada bulan tertentu.
3. **Distribusi jumlah produksi** (actual_ok) per shift dalam bentuk bucket.

---

## B. PERSIAPAN DATA

### B.1. Buat Koleksi dan Generate Data Dummy
Kita akan membuat 1000 dokumen untuk 10 mesin, periode 3 bulan (April–Juni 2026), 3 shift per hari.

**Script Python (disarankan) – `generate_produksi_harian.py`**:
```python
from pymongo import MongoClient
from datetime import datetime, timedelta
import random

client = MongoClient('mongodb://localhost:27017')
db = client['studi_kasus_oee']
collection = db['produksi_harian']
collection.drop()  # Mulai bersih

mesin_list = [f"M{i:02d}" for i in range(1, 11)]
start_date = datetime(2026, 4, 1)
docs = []

for i in range(1000):
    # Acak hari dalam 90 hari
    day_offset = random.randint(0, 90)
    tanggal = start_date + timedelta(days=day_offset)
    shift = random.randint(1, 3)
    mesin = random.choice(mesin_list)
    target = random.randint(200, 500)
    actual_ok = int(target * random.uniform(0.6, 1.0))  # Kadang di bawah target
    actual_reject = int(actual_ok * random.uniform(0, 0.15))  # reject sampai 15%
    durasi_tersedia = 480  # 8 jam
    durasi_operasi = int(durasi_tersedia * random.uniform(0.7, 1.0))

    doc = {
        "mesin": mesin,
        "tanggal": tanggal,
        "shift": shift,
        "target": target,
        "actual_ok": actual_ok,
        "actual_reject": actual_reject,
        "durasi_operasi_menit": durasi_operasi,
        "durasi_tersedia_menit": durasi_tersedia
    }
    docs.append(doc)

    if len(docs) >= 500:
        collection.insert_many(docs)
        docs.clear()

if docs:
    collection.insert_many(docs)

print("Data berhasil dibuat. Total:", collection.count_documents({}))
```

**Alternatif langsung di `mongosh` (JavaScript):**
```javascript
use studi_kasus_oee
db.produksi_harian.drop()
var docs = [];
var mesinList = []; for (let i=1;i<=10;i++) mesinList.push("M"+i.toString().padStart(2,'0'));
var start = new Date(2026,3,1); // April
for (let i=0; i<1000; i++) {
    var tgl = new Date(start.getTime() + Math.floor(Math.random()*90)*86400000);
    var shift = Math.floor(Math.random()*3)+1;
    var target = Math.floor(Math.random()*301)+200;
    var actual_ok = Math.floor(target * (0.6 + Math.random()*0.4));
    var actual_reject = Math.floor(actual_ok * Math.random()*0.15);
    var durasi_tersedia = 480;
    var durasi_operasi = Math.floor(durasi_tersedia * (0.7 + Math.random()*0.3));
    docs.push({
        mesin: mesinList[Math.floor(Math.random()*10)],
        tanggal: tgl,
        shift: shift,
        target: target,
        actual_ok: actual_ok,
        actual_reject: actual_reject,
        durasi_operasi_menit: durasi_operasi,
        durasi_tersedia_menit: durasi_tersedia
    });
    if (docs.length === 500) { db.produksi_harian.insertMany(docs); docs = []; }
}
if (docs.length) db.produksi_harian.insertMany(docs);
```

Setelah data siap, verifikasi:
```javascript
use studi_kasus_oee
db.produksi_harian.countDocuments()
db.produksi_harian.findOne()
```

---

## C. MENGHITUNG OEE PER MESIN PER BULAN

### C.1. Rumus dan Stage
Kita akan menggunakan aggregation pipeline:

1. `$addFields` – menambahkan field perhitungan parsial:
   - `availability` = `durasi_operasi_menit / durasi_tersedia_menit`
   - `performance` = `actual_ok / target`   (asumsi sederhana)
   - `quality` = `actual_ok / (actual_ok + actual_reject)`

2. `$group` – mengelompokkan berdasarkan `mesin` dan **bulan** (diekstrak dari `tanggal`):
   - Gunakan `_id: { mesin: "$mesin", bulan: { $month: "$tanggal" }, tahun: { $year: "$tanggal" } }`
   - Hitung rata‑rata availability, performance, quality (atau lebih tepat, hitung ulang dari total agar tidak bias).
     **Pendekatan akurat:** Jumlahkan total `actual_ok`, `actual_reject`, `target`, `durasi_operasi`, `durasi_tersedia` per kelompok, lalu hitung OEE dari total‑total tersebut. Ini lebih representatif.

   **Langkahnya:**
   - `$group` dengan akumulator:
     - `total_ok` : `$sum: "$actual_ok"`
     - `total_reject` : `$sum: "$actual_reject"`
     - `total_target` : `$sum: "$target"`
     - `total_durasi_operasi` : `$sum: "$durasi_operasi_menit"`
     - `total_durasi_tersedia` : `$sum: "$durasi_tersedia_menit"`

3. `$project` – hitung OEE akhir:
   `OEE = (total_ok / total_target) * (total_durasi_operasi / total_durasi_tersedia) * (total_ok / (total_ok + total_reject))`

4. `$sort` – urutkan berdasarkan OEE atau mesin.

### C.2. Pipeline Lengkap
```javascript
db.produksi_harian.aggregate([
  // Tahap 1: Tidak perlu $match karena ingin semua data
  // Tahap 2: Group by mesin dan bulan
  { $group: {
      _id: {
        mesin: "$mesin",
        bulan: { $month: "$tanggal" },
        tahun: { $year: "$tanggal" }
      },
      total_target: { $sum: "$target" },
      total_ok: { $sum: "$actual_ok" },
      total_reject: { $sum: "$actual_reject" },
      total_op: { $sum: "$durasi_operasi_menit" },
      total_avail: { $sum: "$durasi_tersedia_menit" }
  }},
  // Tahap 3: Hitung OEE
  { $project: {
      _id: 0,
      mesin: "$_id.mesin",
      bulan: "$_id.bulan",
      tahun: "$_id.tahun",
      total_ok: 1,
      total_target: 1,
      total_reject: 1,
      availability: { $divide: ["$total_op", "$total_avail"] },
      performance: { $divide: ["$total_ok", "$total_target"] },
      quality: { $divide: ["$total_ok", { $add: ["$total_ok", "$total_reject"] } ] },
      OEE: {
        $multiply: [
          { $divide: ["$total_ok", "$total_target"] },
          { $divide: ["$total_op", "$total_avail"] },
          { $divide: ["$total_ok", { $add: ["$total_ok", "$total_reject"] } ] }
        ]
      }
  }},
  // Tahap 4: Urutkan berdasarkan OEE terendah
  { $sort: { OEE: 1 } }
])
```

**Penjelasan:**
- `$group` mengumpulkan semua data produksi untuk setiap mesin di tiap bulan. Total‑total ini memastikan perhitungan OEE agregat akurat.
- `$project` menciptakan field `availability`, `performance`, `quality`, dan `OEE`. Ekspresi `$divide` dan `$multiply` menghasilkan nilai numerik.
- Hasilnya adalah dokumen per mesin per bulan dengan OEE.

**Contoh Output:**
```json
{ "mesin": "M05", "bulan": 4, "tahun": 2026, "OEE": 0.72, "availability": 0.88, "performance": 0.82, "quality": 0.99 }
{ "mesin": "M08", "bulan": 5, "tahun": 2026, "OEE": 0.65, ... }
...
```

---

## D. MENEMUKAN MESIN DENGAN OEE DI BAWAH 80%
Kita bisa menambahkan `$match` langsung setelah `$project` pada pipeline di atas.
```javascript
db.produksi_harian.aggregate([
  { $group: { ... } },  // sama seperti sebelumnya
  { $project: { OEE: ..., mesin: ... } },
  { $match: { OEE: { $lt: 0.80 } } },
  { $sort: { OEE: 1 } }
])
```
**Perhatian:** `$match` setelah `$project` dapat memfilter dokumen hasil agregasi. Letakkan setelah field `OEE` terdefinisi.

---

## E. DISTRIBUSI JUMLAH PRODUKSI PER SHIFT DALAM BUCKET
Manajemen ingin melihat sebaran jumlah produksi (actual_ok) per shift, misal dikelompokkan menjadi bucket: rendah (0-200), sedang (200-400), tinggi (400+). Gunakan `$bucket`.

```javascript
db.produksi_harian.aggregate([
  { $bucket: {
      groupBy: "$actual_ok",
      boundaries: [0, 200, 300, 400, 600], // 0-199, 200-299, 300-399, 400-600
      default: "di atas 600",
      output: {
        count: { $sum: 1 },
        rata_reject: { $avg: "$actual_reject" }
      }
  }}
])
```
Namun ini belum dipisah per shift. Untuk distribusi per shift, kita bisa `$group` atau `$facet`. Misal kita ingin tiga bucket terpisah untuk setiap shift:

```javascript
db.produksi_harian.aggregate([
  { $group: {
      _id: "$shift",
      data: { $push: "$actual_ok" }  // kumpulkan nilai
  }},
  // Kemudian di aplikasi atau gunakan operator $bucket di sub-pipeline (tidak bisa langsung karena $bucket butuh array dokumen)
])
```
**Alternatif:** Gunakan `$facet` untuk menjalankan pipeline bucket paralel per shift:
```javascript
db.produksi_harian.aggregate([
  { $facet: {
      "shift1": [
        { $match: { shift: 1 } },
        { $bucket: { groupBy: "$actual_ok", boundaries: [0,200,300,400,600], default: "600+", output: { count: {$sum:1} } } }
      ],
      "shift2": [
        { $match: { shift: 2 } },
        { $bucket: { groupBy: "$actual_ok", boundaries: [0,200,300,400,600], default: "600+", output: { count: {$sum:1} } } }
      ],
      "shift3": [
        { $match: { shift: 3 } },
        { $bucket: { groupBy: "$actual_ok", boundaries: [0,200,300,400,600], default: "600+", output: { count: {$sum:1} } } }
      ]
  }}
])
```
Hasilnya satu dokumen berisi tiga array, masing‑masing dengan distribusi bucket untuk shift tertentu.

**Cara lain – lebih sederhana:** Jika hanya ingin jumlah per shift, gunakan `$group` langsung:
```javascript
db.produksi_harian.aggregate([
  { $group: { _id: "$shift", total_produksi: { $sum: "$actual_ok" } } }
])
```

---

## F. OPTIMASI DAN ANALISIS
- **Indeks:** Buat indeks compound untuk `tanggal` dan `mesin` agar query OEE lebih cepat.
  ```javascript
  db.produksi_harian.createIndex({ mesin: 1, tanggal: 1 })
  ```
- **`explain()`:** Jalankan pipeline OEE dengan `.explain("executionStats")` dan perhatikan apakah indeks terpakai (walaupun `$group` tanpa `$match` mungkin tetap melakukan COLLSCAN, namun dengan indeks bisa mempercepat pengelompokan jika ada filter tanggal).
- **Penyempurnaan OEE:** Rumus di atas menggunakan rasio `actual_ok / target` untuk performance. Di dunia nyata, performance dihitung dari (Total Parts / (Run Time × Ideal Speed)). Dengan data yang ada, kita bisa menyesuaikan asumsi.

---

## G. LATIHAN TAMBAHAN
1. **Tren Bulanan:** Modifikasi pipeline agar menampilkan OEE per bulan untuk semua mesin, lalu urutkan bulan.
2. **Mesin Terburuk:** Tampilkan 3 mesin dengan OEE terendah di bulan Juni 2026 saja (gunakan `$match` di awal untuk filter bulan).
3. **Kinerja Harian:** Hitung OEE per mesin per hari (gunakan `$dateToString` untuk ekstrak tanggal).
4. **Simulasi Perbaikan:** Jika quality harus minimal 95%, tampilkan mesin yang tidak memenuhi kriteria tersebut.

---

## H. PENUTUP
Dengan tutorial ini, Anda telah mempraktikkan:
- `$addFields` untuk menambah field perhitungan.
- `$group` dengan akumulator untuk agregat total.
- `$project` untuk kalkulasi metrik bisnis.
- `$match` untuk filter hasil agregasi.
- `$facet` untuk multi‑pipeline.
- `$bucket` untuk distribusi.

Gunakan pengetahuan ini untuk membangun laporan analitik industri langsung di MongoDB tanpa perlu menarik data ke aplikasi eksternal. Selamat mencoba!
---


# PANDUAN TUGAS MANDIRI PERTEMUAN 5
**Mata Kuliah:** Praktikum Basis Data NoSQL – MongoDB
**Topik:** Aggregation Framework
**Tugas:** Membangun Pipeline Agregasi untuk Dashboard Monitoring Kualitas Pabrik

---

## A. TUJUAN TUGAS
Melalui tugas mandiri ini, mahasiswa akan:
1. Terampil membuat dataset simulasi yang mencerminkan data inspeksi kualitas di pabrik.
2. Mampu merancang pipeline agregasi multi‑stage menggunakan `$group`, `$lookup`, `$match`, `$project`, `$sort`, `$addFields`, dan `$facet`.
3. Dapat mengoptimasi pipeline dengan menempatkan `$match` di awal dan memanfaatkan indeks.
4. Mampu menginterpretasikan hasil agregasi untuk pengambilan keputusan mutu.

---

## B. DESKRIPSI STUDI KASUS
Sebuah pabrik komponen otomotif memiliki sistem pencatatan hasil inspeksi. Setiap batch produksi diperiksa oleh inspektor, dan hasilnya dicatat dalam koleksi **`inspeksi`**. Manajemen memiliki target maksimal cacat yang boleh terjadi per batch, yang disimpan di koleksi **`target_kualitas`**.

Kebutuhan dashboard mutu:
1. **Persentase cacat** tiap batch (rasio cacat terhadap jumlah diperiksa).
2. **Status batch** (OK atau NOT OK) dengan membandingkan persentase cacat terhadap target.
3. **Jumlah batch NOT OK per mesin per minggu** untuk melihat mesin paling bermasalah.
4. **Top 3 mesin** dengan jumlah NOT OK terbanyak dalam satu bulan.
5. **Ringkasan bulanan** yang menampilkan total produksi, rata‑rata cacat, dan mesin terburuk dalam satu query (`$facet`).

---

## C. LANGKAH PENGERJAAN

### Langkah 1 – Membuat Database dan Koleksi
Buka `mongosh` dan jalankan:
```javascript
use tugas5_<NIM>   // ganti dengan NPM Anda
```
Contoh: `use tugas5_2206054321`

### Langkah 2 – Generate Data Dummy (minimal 1000 dokumen untuk `inspeksi`, 50 untuk `target_kualitas`)
Gunakan script Python (`generate_inspeksi.py`) di bawah. Pastikan `pymongo` terinstal.

**Script Python:**
```python
from pymongo import MongoClient
from datetime import datetime, timedelta
import random

# Ganti dengan NIM Anda
NIM = "2206054321"
client = MongoClient('mongodb://localhost:27017')
db = client[f'tugas5_{NIM}']
db.inspeksi.drop()
db.target_kualitas.drop()

mesin_list = [f"M{i:02d}" for i in range(1, 11)]
batch_ids = [f"B{str(i).zfill(4)}" for i in range(1, 2001)]
inspektor_list = ["Andi", "Budi", "Citra", "Dian", "Eka"]
jenis_cacat_options = ["gores", "retak", "bengkok", "warna", "dimensi", "pori"]

# Generate data inspeksi (1500 dokumen)
docs = []
start_date = datetime(2026, 4, 1, 6, 0)
for i in range(1500):
    batch = random.choice(batch_ids)
    mesin = random.choice(mesin_list)
    tanggal = start_date + timedelta(minutes=random.randint(0, 60*24*90))  # 3 bulan
    shift = 1 if tanggal.hour < 14 else (2 if tanggal.hour < 22 else 3)
    jumlah = random.randint(100, 500)
    reject = int(jumlah * random.uniform(0, 0.10))  # 0-10% cacat
    jenis = random.sample(jenis_cacat_options, k=random.randint(1, 3))
    doc = {
        "batch_id": batch,
        "mesin": mesin,
        "tanggal": tanggal,
        "shift": shift,
        "inspektor": random.choice(inspektor_list),
        "jumlah_diperiksa": jumlah,
        "cacat_ditemukan": reject,
        "jenis_cacat": jenis
    }
    docs.append(doc)
    if len(docs) >= 500:
        db.inspeksi.insert_many(docs)
        docs.clear()
if docs:
    db.inspeksi.insert_many(docs)

print("Data inspeksi selesai. Total:", db.inspeksi.count_documents({}))

# Generate target kualitas (setiap batch punya target cacat maksimal 5% dari rata-rata ukuran batch)
targets = []
for i in range(1, 2001):
    batch = f"B{str(i).zfill(4)}"
    # target cacat acak 1-15
    target_cacat = random.randint(1, 15)
    targets.append({"batch_id": batch, "target_maks_cacat": target_cacat})
    if len(targets) >= 500:
        db.target_kualitas.insert_many(targets)
        targets.clear()
if targets:
    db.target_kualitas.insert_many(targets)

print("Data target kualitas selesai. Total:", db.target_kualitas.count_documents({}))
```

**Penting:** Jika tidak ada Python, Anda bisa membuat data di `mongosh` dengan perulangan (lihat metode alternatif di modul sebelumnya).

### Langkah 3 – Memastikan Data Siap
Di `mongosh`:
```javascript
use tugas5_<NIM>
db.inspeksi.countDocuments()   // harus >= 1000
db.target_kualitas.countDocuments()  // harus >= 50
db.inspeksi.findOne()
db.target_kualitas.findOne()
```
Buat indeks yang akan mempercepat agregasi:
```javascript
db.inspeksi.createIndex({ batch_id: 1 })
db.inspeksi.createIndex({ mesin: 1, tanggal: 1 })
db.target_kualitas.createIndex({ batch_id: 1 })
```

### Langkah 4 – Pipeline 1: Persentase Cacat per Batch
Tampilkan `batch_id`, `mesin`, `jumlah_diperiksa`, `cacat_ditemukan`, dan `persen_cacat` (cacat_ditemukan / jumlah_diperiksa * 100). Urutkan berdasarkan persentase tertinggi.

**Pipeline:**
```javascript
db.inspeksi.aggregate([
  { $addFields: {
      persen_cacat: { $multiply: [ { $divide: ["$cacat_ditemukan", "$jumlah_diperiksa"] }, 100 ] }
  }},
  { $sort: { persen_cacat: -1 } }
]).limit(10) // lihat 10 teratas saja
```

### Langkah 5 – Pipeline 2: Gabung dengan Target dan Tentukan Status Batch
Gabungkan koleksi `inspeksi` dengan `target_kualitas` berdasarkan `batch_id`, lalu hitung persentase cacat dan bandingkan dengan `target_maks_cacat`. Jika `persen_cacat > target_maks_cacat` → status "NOT OK", sebaliknya "OK". Tampilkan juga selisihnya.

**Pipeline:**
```javascript
db.inspeksi.aggregate([
  { $lookup: {
      from: "target_kualitas",
      localField: "batch_id",
      foreignField: "batch_id",
      as: "target_info"
  }},
  { $unwind: "$target_info" },
  { $addFields: {
      persen_cacat: { $multiply: [ { $divide: ["$cacat_ditemukan", "$jumlah_diperiksa"] }, 100 ] },
      target_maks: "$target_info.target_maks_cacat"
  }},
  { $addFields: {
      status: { $cond: { if: { $gt: ["$persen_cacat", "$target_maks"] }, then: "NOT OK", else: "OK" } }
  }},
  { $project: {
      _id: 0,
      batch_id: 1,
      mesin: 1,
      persen_cacat: 1,
      target_maks: 1,
      status: 1,
      selisih: { $subtract: ["$persen_cacat", "$target_maks"] }
  }},
  { $sort: { selisih: -1 } }
]).limit(10)
```

### Langkah 6 – Pipeline 3: Jumlah Batch NOT OK per Mesin per Minggu
Ekstrak minggu dari `tanggal` menggunakan `$week`, lalu filter hanya batch dengan status NOT OK, kemudian group per mesin dan minggu.

Kita bisa langsung melanjutkan dari pipeline sebelumnya, tapi agar efisien kita bisa menulis ulang dengan `$match` di awal (hanya batch NOT OK). Tapi karena status adalah hasil kalkulasi, kita perlu menghitung dulu, lalu `$match`.

**Pipeline:**
```javascript
db.inspeksi.aggregate([
  { $lookup: {
      from: "target_kualitas",
      localField: "batch_id",
      foreignField: "batch_id",
      as: "target_info"
  }},
  { $unwind: "$target_info" },
  { $addFields: {
      persen_cacat: { $multiply: [ { $divide: ["$cacat_ditemukan", "$jumlah_diperiksa"] }, 100 ] }
  }},
  { $addFields: {
      status: { $cond: { if: { $gt: ["$persen_cacat", "$target_info.target_maks_cacat"] }, then: "NOT OK", else: "OK" } }
  }},
  { $match: { status: "NOT OK" } },
  { $group: {
      _id: {
        mesin: "$mesin",
        minggu: { $week: "$tanggal" },
        tahun: { $year: "$tanggal" }
      },
      jumlah_NOT_OK: { $sum: 1 }
  }},
  { $sort: { "_id.tahun": 1, "_id.minggu": 1, "_id.mesin": 1 } }
])
```
**Catatan:** `$week` mengembalikan angka 0-53. Tahun diperlukan agar minggu tidak campur.

### Langkah 7 – Pipeline 4: Top 3 Mesin dengan Jumlah NOT OK Terbanyak (dalam satu bulan tertentu)
Misal untuk bulan Mei 2026. Gunakan `$match` di awal untuk filter bulan, lalu lakukan join dan penentuan status, lalu group per mesin dan hitung total NOT OK, sort dan limit 3.

```javascript
db.inspeksi.aggregate([
  { $match: {
      tanggal: { $gte: ISODate("2026-05-01"), $lt: ISODate("2026-06-01") }
  }},
  { $lookup: {
      from: "target_kualitas",
      localField: "batch_id",
      foreignField: "batch_id",
      as: "target_info"
  }},
  { $unwind: "$target_info" },
  { $addFields: {
      persen_cacat: { $multiply: [ { $divide: ["$cacat_ditemukan", "$jumlah_diperiksa"] }, 100 ] }
  }},
  { $addFields: {
      status: { $cond: { if: { $gt: ["$persen_cacat", "$target_info.target_maks_cacat"] }, then: "NOT OK", else: "OK" } }
  }},
  { $match: { status: "NOT OK" } },
  { $group: {
      _id: "$mesin",
      total_NOT_OK: { $sum: 1 }
  }},
  { $sort: { total_NOT_OK: -1 } },
  { $limit: 3 }
])
```

### Langkah 8 – Pipeline 5 (Bonus/Opsional): Ringkasan Bulanan dengan `$facet`
Gunakan `$facet` untuk memperoleh tiga informasi dalam satu query:
- Total produksi (jumlah batch) per bulan.
- Rata‑rata persentase cacat per bulan.
- Mesin dengan jumlah NOT OK terbanyak per bulan.

Kita akan fokus pada satu bulan tertentu (misal Juni 2026) untuk kemudahan. Bisa juga tanpa filter bulan agar mencakup semua data, tetapi hasilnya akan lebih kompleks. Contoh (dengan filter Juni):

```javascript
db.inspeksi.aggregate([
  { $match: { tanggal: { $gte: ISODate("2026-06-01"), $lt: ISODate("2026-07-01") } } },
  { $lookup: {
      from: "target_kualitas",
      localField: "batch_id",
      foreignField: "batch_id",
      as: "target_info"
  }},
  { $unwind: "$target_info" },
  { $addFields: {
      persen_cacat: { $multiply: [ { $divide: ["$cacat_ditemukan", "$jumlah_diperiksa"] }, 100 ] },
      status: { $cond: { if: { $gt: [ { $multiply: [ { $divide: ["$cacat_ditemukan", "$jumlah_diperiksa"] }, 100 ] }, "$target_info.target_maks_cacat" ] }, then: "NOT OK", else: "OK" } }
  }},
  { $facet: {
      "total_batch": [
        { $count: "count" }
      ],
      "rata_cacat": [
        { $group: { _id: null, avg_cacat: { $avg: "$persen_cacat" } } }
      ],
      "mesin_terburuk": [
        { $match: { status: "NOT OK" } },
        { $group: { _id: "$mesin", jumlah: { $sum: 1 } } },
        { $sort: { jumlah: -1 } },
        { $limit: 3 }
      ]
  }}
])
```
**Catatan:** Di dalam `$facet`, setiap pipeline berjalan pada input yang sama (hasil `$project` sebelumnya).

---

## D. FORMAT LAPORAN
Kumpulkan laporan dalam bentuk **PDF** dengan struktur:
1. **Halaman Sampul** (judul, NIM, nama, kelas)
2. **Pendahuluan** (1 paragraf tentang tujuan)
3. **Desain Data** (diagram koleksi, contoh dokumen)
4. **Pipeline dan Hasil**
   Untuk setiap pipeline (1 s/d 5) sertakan:
   - Kode pipeline lengkap
   - Tangkapan layar hasil (10 data pertama)
   - Penjelasan singkat setiap stage
5. **Analisis dan Kesimpulan** (temuan dari data, misal mesin mana yang perlu perbaikan)
6. **Lampiran**
   - Script pembangkit data (jika menggunakan Python)
   - Semua perintah dalam satu file `.txt` atau `.js`

**Pengumpulan:** Unggah PDF di LMS paling lambat sebelum pertemuan 6.

---

## E. KRITERIA PENILAIAN
| Kriteria                       | Bobot | Indikator |
|--------------------------------|-------|-----------|
| Kebenaran pipeline (output)    | 30%   | Semua pipeline menghasilkan data sesuai spesifikasi |
| Kompleksitas & ketepatan stage | 20%   | Pemilihan stage tepat, tidak ada stage berlebih |
| Optimasi (indeks & $match)     | 15%   | Ada `$match` awal, indeks dibuat dan dipakai |
| Penggunaan $lookup & $facet    | 15%   | Join berhasil, facet berjalan |
| Dokumentasi & kerapian laporan | 20%   | Screenshot jelas, penjelasan runut, kode rapi |

---

## F. TIPS & TROUBLESHOOTING
- **Error “$lookup membutuhkan indeks”?** MongoDB menyarankan indeks pada `foreignField`, tapi tidak wajib. Jika ingin cepat, tambahkan indeks.
- **`$week` menghasilkan angka 0?** Ya, minggu pertama tahun yang hanya memiliki beberapa hari bisa jadi 0. Gunakan kombinasi dengan tahun.
- **Data kosong di `$facet`?** Pastikan semua field yang direferensikan sudah didefinisikan sebelum `$facet`. Gunakan `$addFields` untuk menambah `persen_cacat` dan `status` sebelum `$facet`.
- **Ingin cek stage per stage?** Gunakan MongoDB Compass, tambahkan stage satu per satu, amati output antara.


---

## E. LATIHAN MANDIRI
1. Gunakan database `latihan5`, koleksi `data_sensor` dengan 5000 dokumen (field: `sensor_id`, `nilai`, `timestamp`, `category`).
2. Tulis pipeline untuk:
   - Menampilkan rata‑rata nilai per `sensor_id` pada 7 hari terakhir.
   - Menampilkan sensor yang memiliki nilai di atas 90 minimal 3 kali (gunakan `$group` dengan `$push` atau `$sum` kondisional).
   - Menambahkan field `jam` dan menghitung nilai rata‑rata per jam per sensor.
3. Buat koleksi `sensor_info` lalu join untuk menampilkan nama sensor dalam hasil.
4. Dokumentasikan setiap pipeline, jelaskan setiap stage.

---

## F. STUDI KASUS: ANALISIS EFEKTIVITAS PRODUKSI
**Latar Belakang:** Pabrik elektronik mencatat produksi di koleksi `produksi_harian` dengan struktur:
- `mesin`, `tanggal`, `shift`, `target` (target produksi),
- `actual_ok` (hasil baik), `actual_reject` (hasil cacat),
- `durasi_operasi_menit`, `durasi_tersedia_menit`.

Manajemen ingin:
1. Menghitung OEE secara kasar = (actual_ok / target) * (durasi_operasi / durasi_tersedia) * (actual_ok / (actual_ok+actual_reject)).
2. Menampilkan OEE per mesin per bulan.
3. Menemukan mesin dengan OEE di bawah 80% pada bulan tertentu.
4. Distribusi jumlah produksi per shift dalam bentuk bucket.

**Tugas:**
- Buat koleksi dan isi data simulasi minimal 200 dokumen.
- Tulis pipeline agregasi untuk menghitung OEE per mesin, per bulan. (Gunakan `$group` dengan `_id` { mesin, bulan: { $month: "$tanggal" } }).
- Tampilkan hanya mesin yang OEE < 0.8.
- Sajikan hasil dalam laporan, serta jelaskan setiap stage.

---

## G. TUGAS TERSTRUKTUR
### Deskripsi
Membangun pipeline agregasi untuk dashboard monitoring kualitas pabrik.

### Spesifikasi
1. Gunakan database `tugas5_<NIM>`. Buat koleksi `inspeksi` (min 1000 dok) dengan field:
   - `batch_id`, `tanggal` (Date), `mesin`, `inspektor`, `jumlah_diperiksa`, `cacat_ditemukan`, `jenis_cacat` (array string).
2. Buat koleksi `target_kualitas` berisi `batch_id`, `target_maks_cacat`.
3. Buat pipeline untuk:
   - Menghitung persentase cacat per batch.
   - Gabungkan dengan target kualitas menggunakan `$lookup`, lalu tandai batch dengan status "OK" atau "NOT OK" jika persentase melebihi target.
   - Hitung jumlah batch NOT OK per mesin per minggu (gunakan `$week`).
   - Tampilkan top 3 mesin dengan jumlah NOT OK terbanyak.
   - (Opsional) Gunakan `$facet` untuk memberikan ringkasan bulanan: total produksi, rata-rata cacat, dan mesin terburuk dalam satu query.
4. Ikuti prinsip optimasi: `$match` awal, indeks pada `tanggal` dan `mesin`, proyeksi minimal.
5. Sertakan perintah dan hasil di laporan PDF, serta interpretasi.

### Rubrik Penilaian
| Kriteria                        | Bobot |
|---------------------------------|-------|
| Kompleksitas & ketepatan pipeline | 30%   |
| Penggunaan $lookup & ekspresi   | 20%   |
| Optimasi dan indeks             | 20%   |
| Dokumentasi dan analisis        | 20%   |
| Kreativitas (facet, bucket)     | 10%   |

### Tenggat
Sebelum pertemuan ke-6, unggah di LMS.

---

## H. REFERENSI PERTEMUAN 5
1. MongoDB, Inc. *Aggregation Pipeline*. https://docs.mongodb.com/manual/aggregation/
2. MongoDB University. *M121: The MongoDB Aggregation Framework* (Free Course).
3. Hows, D., Membrey, P., Plugge, E. (2020). *MongoDB Basics*. Apress.
4. Copeland, R. (2020). *MongoDB Applied Design Patterns*. O'Reilly.

---

**Pesan Dosen:**
Aggregation framework adalah jantung analisis data di MongoDB. Kuasai, dan Anda dapat mengubah data mentah menjadi wawasan industri berharga. Jangan lupa selalu mengukur kinerja dengan `explain`. Selamat belajar!
