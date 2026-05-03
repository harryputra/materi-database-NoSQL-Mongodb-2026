# PERTEMUAN 6 – MONGODB DENGAN PYTHON (PYMONGO)
**Program Studi:** Teknologi Rekayasa Informatika Industri (TRIN)
**Alokasi Waktu:** 1 × 150 menit (praktikum)

---

## A. CAPAIAN PEMBELAJARAN PERTEMUAN 6
Setelah pertemuan ini mahasiswa mampu:
1. Menginstal dan mengkonfigurasi PyMongo serta library pendukung (pandas, python-dotenv) di lingkungan Python.
2. Menghubungkan aplikasi Python ke server MongoDB (lokal dan Atlas) menggunakan connection string.
3. Melakukan operasi CRUD (Create, Read, Update, Delete) langsung dari kode Python.
4. Menjalankan query kompleks dan aggregation pipeline melalui PyMongo.
5. Mengonversi hasil query menjadi pandas DataFrame untuk analisis data industri (statistik deskriptif, visualisasi sederhana).
6. Membangun skrip otomatis untuk menghasilkan data sensor tiruan secara periodik dan menyimpannya ke MongoDB.
7. Menerapkan penanganan error (`try-except`) dan logging dasar pada aplikasi integrasi.
8. Memahami best practice dalam koneksi database (connection pooling, environment variable, context manager).

---

## B. ALAT DAN BAHAN
- **Python** versi 3.8 ke atas (disarankan 3.10+).
- **pip** (package installer for Python) dan virtual environment (venv atau conda).
- Library Python:
  - `pymongo` (minimal versi 4.x)
  - `pandas`
  - `python-dotenv` (opsional, untuk manajemen kredensial)
  - `matplotlib` (opsional, untuk visualisasi)
- **MongoDB Server** lokal yang sedang berjalan.
- **MongoDB Compass** (untuk verifikasi data).
- **IDE/Text Editor**: VS Code, PyCharm, atau lainnya.
- **Terminal / Command Prompt**.

---

## C. URAIAN MATERI SUPER LENGKAP

### 1. Pendahuluan: Mengapa Integrasi Python-MongoDB di Industri?
Python adalah bahasa utama dalam otomasi, data science, dan pengembangan IoT di industri. MongoDB dengan fleksibilitas skema-nya sangat cocok untuk menangani data semi-terstruktur dari sensor, log, dan sistem produksi. Integrasi PyMongo memungkinkan:
- Otomasi pengumpulan data dari mesin/PLC ke database.
- Analisis data real-time dan historis menggunakan ekosistem Python (pandas, NumPy, matplotlib).
- Pembuatan dashboard sederhana atau laporan otomatis.
- Prototyping cepat aplikasi industri tanpa overhead relasional.

### 2. Instalasi dan Konfigurasi Lingkungan

#### a. Membuat Virtual Environment
```bash
python -m venv venv_mongo
source venv_mongo/bin/activate   # Linux/macOS
venv_mongo\Scripts\activate      # Windows
```

#### b. Instalasi Library
```bash
pip install pymongo pandas python-dotenv matplotlib
```
- `pymongo`: driver resmi.
- `pandas`: analisis data.
- `python-dotenv`: membaca variabel dari file `.env`.
- `matplotlib`: plotting.

#### c. Menyimpan Konfigurasi di `.env` (best practice)
Buat file `.env`:
```
MONGO_URI=mongodb://localhost:27017
DB_NAME=industri
```
Kemudian di Python:
```python
import os
from dotenv import load_dotenv
load_dotenv()
MONGO_URI = os.getenv("MONGO_URI")
DB_NAME = os.getenv("DB_NAME")
```

### 3. Koneksi ke MongoDB dengan PyMongo

#### a. Dasar Koneksi
```python
from pymongo import MongoClient
client = MongoClient("mongodb://localhost:27017/")
# atau gunakan MONGO_URI
db = client["nama_database"]
collection = db["nama_koleksi"]
```

**MongoClient** secara default menggunakan connection pooling (maks 100 koneksi). Disarankan membuat satu instance `MongoClient` untuk seluruh aplikasi (misal di modul terpisah) dan menggunakannya kembali.

#### b. Menguji Koneksi
```python
try:
    client.admin.command('ping')
    print("Koneksi berhasil!")
except Exception as e:
    print("Gagal:", e)
```

### 4. CRUD Operations via PyMongo

#### a. Insert
- `insert_one(document)` -> mengembalikan `InsertOneResult` dengan `inserted_id`.
- `insert_many(list_of_docs)` -> mengembalikan `InsertManyResult`.

**Contoh insert data sensor:**
```python
sensor_data = {
    "mesin": "CNC-01",
    "suhu": 72.5,
    "getaran": 0.13,
    "timestamp": datetime.utcnow()
}
result = collection.insert_one(sensor_data)
print(result.inserted_id)
```

Batch insert:
```python
docs = [
    {"mesin": "CNC-01", "suhu": 71.0, "timestamp": datetime.utcnow()},
    {"mesin": "CNC-02", "suhu": 85.3, "timestamp": datetime.utcnow()}
]
result = collection.insert_many(docs)
```

#### b. Read (Query)
- `find(filter, projection)` -> mengembalikan `Cursor` (iterable).
- `find_one(filter)` -> satu dokumen atau `None`.

**Contoh:**
```python
# Semua dokumen mesin CNC-01, hanya tampilkan suhu dan timestamp
cursor = collection.find(
    {"mesin": "CNC-01"},
    {"_id": 0, "suhu": 1, "timestamp": 1}
).sort("timestamp", -1).limit(10)

for doc in cursor:
    print(doc)
```

**Hati-hati:** Cursor harus dikonsumsi segera atau ditutup. Gunakan `list(cursor)` untuk mengambil semua atau loop.

#### c. Update
- `update_one(filter, update)` -> mengembalikan `UpdateResult`.
- `update_many(filter, update)`.
- Operasi update menggunakan operator yang sama seperti di shell.

```python
# Menambah jumlah reject satu mesin
collection.update_one(
    {"mesin": "CNC-01"},
    {"$inc": {"reject_count": 1}, "$set": {"status": "diperbaiki"}}
)
```

Upsert:
```python
collection.update_one(
    {"mesin": "CNC-05"},
    {"$set": {"suhu": 80.0, "timestamp": datetime.utcnow()}},
    upsert=True
)
```

#### d. Delete
- `delete_one(filter)`
- `delete_many(filter)`

```python
collection.delete_many({"timestamp": {"$lt": datetime(2026,1,1)}})
```

### 5. Bekerja dengan Cursor dan Batch Processing
Cursor bersifat *lazy*, mengambil data dari server dalam batch (default 101 dokumen per batch). Untuk memproses dokumen dalam jumlah besar, gunakan `batch_size()` dan loop, atau gunakan `for doc in cursor`.

```python
cursor = collection.find().batch_size(500)
for doc in cursor:
    process(doc)
```

### 6. Agregasi dengan PyMongo
Menggunakan `aggregate(pipeline)` yang mengembalikan cursor.
```python
pipeline = [
    {"$match": {"suhu": {"$gt": 80}}},
    {"$group": {"_id": "$mesin", "max_suhu": {"$max": "$suhu"}}}
]
results = collection.aggregate(pipeline)
for r in results:
    print(r)
```
Sama seperti shell, pipeline adalah list of dicts.

### 7. Integrasi dengan pandas
Mengonversi hasil query atau agregasi menjadi DataFrame untuk analisis lebih lanjut.

```python
import pandas as pd

cursor = collection.find({"mesin": "CNC-01"})
df = pd.DataFrame(list(cursor))
print(df.head())
print(df.describe())
```

**Catatan:** Field `_id` akan ikut sebagai kolom. Bisa di-drop: `df.drop(columns=['_id'], inplace=True)`.

#### Analisis data sensor:
- Konversi timestamp ke datetime.
- Resampling data (jika banyak data).
- Plotting:
```python
import matplotlib.pyplot as plt
df['timestamp'] = pd.to_datetime(df['timestamp'])
df.set_index('timestamp', inplace=True)
df['suhu'].plot()
plt.show()
```

### 8. Simulasi Data Generator untuk IoT Industri
Contoh skrip yang menghasilkan data sensor secara periodik:
```python
import time
import random
from datetime import datetime

while True:
    doc = {
        "mesin": f"CNC-{random.randint(1,5):02d}",
        "suhu": round(random.uniform(60, 100), 2),
        "getaran": round(random.uniform(0.1, 0.5), 2),
        "timestamp": datetime.utcnow()
    }
    collection.insert_one(doc)
    print("Inserted:", doc)
    time.sleep(2)  # setiap 2 detik
```

### 9. Penanganan Error dan Logging
```python
import logging
logging.basicConfig(level=logging.INFO)

try:
    collection.insert_one(doc)
    logging.info("Data inserted")
except Exception as e:
    logging.error(f"Insert gagal: {e}")
```

### 10. Manajemen Koneksi yang Baik
- Gunakan satu `MongoClient` per proses (biasanya di modul global).
- Pada aplikasi web (Flask/Django), buat client saat startup dan buat pool.
- Untuk skrip sederhana, gunakan `with` statement atau close manual.

```python
client = MongoClient(MONGO_URI)
try:
    # operasi
finally:
    client.close()
```

Atau gunakan context manager dengan `pymongo.MongoClient` (mulai PyMongo 4.x mendukung):
```python
with MongoClient(MONGO_URI) as client:
    db = client.test
    # operasi
```

### 11. Dokumentasi dan Troubleshooting
- Pastikan server MongoDB menerima koneksi di port yang benar.
- Gunakan `localhost:27017` jika server lokal.
- Jika menggunakan Atlas, dapatkan connection string dan pastikan IP whitelist.
- Cek versi PyMongo: `pip show pymongo`.

---

## D. KEGIATAN PRAKTIKUM (120 MENIT)

### Fase 1: Setup Lingkungan (15 menit)
1. Mahasiswa membuat virtual environment dan menginstal library.
2. Membuat file `.env` berisi URI MongoDB lokal.
3. Membuat file Python `koneksi.py` dan menjalankan uji koneksi.

### Fase 2: Operasi Dasar (25 menit)
Mahasiswa menulis script `crud_sensor.py`:
- Buat koneksi ke database `praktikum6` dan koleksi `sensor`.
- Insert 10 data sensor manual (gunakan `insert_many`).
- Query data dengan `find` dan cetak.
- Update suhu mesin tertentu.
- Hapus data yang lebih dari 1 hari (simulasi).

### Fase 3: Agregasi via Python (20 menit)
Script `agregasi.py`:
- Gunakan `aggregate` untuk menghitung rata-rata suhu per mesin.
- Cetak hasilnya.

### Fase 4: Analisis dengan pandas (30 menit)
Script `analisis.py`:
- Ambil 100 data terbaru dari koleksi `sensor` (atau gunakan data yang sudah ada).
- Muat ke DataFrame.
- Tampilkan statistik deskriptif (`df.describe()`).
- Buat plot sederhana (gunakan opsi tanpa GUI jika tidak tersedia, atau simpan ke file).
- Simpan hasil agregasi ke file CSV.

### Fase 5: Data Generator & Monitoring (20 menit)
Script `data_generator.py`:
- Loop tak terbatas (bisa dihentikan dengan KeyboardInterrupt) yang menyisipkan data sensor setiap 1 detik.
- Buka MongoDB Compass, lihat dokumen bertambah secara real-time.

### Fase 6: Error Handling & Logging (10 menit)
Tambahkan try-except di setiap operasi, dan logging basic.

---

## E. LATIHAN MANDIRI
1. Buat database `latihan6` dan koleksi `maintenance`.
2. Tulis script Python yang:
   - Mengimpor data maintenance dari file CSV (sediakan file contoh dengan kolom: mesin, tanggal, biaya, teknisi) menggunakan `pandas.read_csv()` lalu `insert_many`.
   - Mencari semua maintenance yang biayanya > 1.000.000 dan menampilkannya dalam DataFrame.
   - Meng-update teknisi untuk satu record tertentu.
   - Menghitung total biaya per bulan menggunakan agregasi, lalu mencetak hasilnya.
3. Simpan script dan screenshot hasil.

---

## F. STUDI KASUS: DASHBOARD SEDERHANA MONITORING SUHU MESIN
**Latar Belakang:** Pabrik memiliki 5 mesin yang mengirim data suhu setiap 5 detik melalui Python script ke MongoDB koleksi `suhu_mesin`. Manajer ingin melihat:
- Suhu terkini setiap mesin.
- Suhu rata-rata dalam 1 jam terakhir.
- Alarm jika suhu > 90.

**Tugas Anda:**
- Buat script Python `monitor.py` yang:
  1. Mengambil data suhu 1 jam terakhir dari MongoDB (gunakan filter timestamp).
  2. Menghitung rata-rata suhu per mesin.
  3. Mendeteksi mesin yang suhunya pernah > 90.
  4. Menampilkan hasil di konsol dan menyimpan ke file log.
- Sertakan penanganan error jika MongoDB tidak dapat dihubungi.
- Dokumentasikan.

---

## G. TUGAS TERSTRUKTUR
### Deskripsi
Membangun aplikasi Python sederhana untuk input data produksi dan laporan.

### Spesifikasi
1. Buat database `tugas6_<NIM>`.
2. Buat koleksi `produksi` dengan field: `batch`, `mesin`, `jumlah`, `reject`, `tanggal` (Date).
3. Kembangkan script Python `app_produksi.py` dengan fitur:
   - Input data produksi baru (input dari keyboard atau baca dari CSV).
   - Menampilkan semua data produksi untuk mesin tertentu (input pengguna).
   - Menghitung reject rate (reject/jumlah*100) per batch menggunakan agregasi dan menampilkan yang reject rate > 5%.
   - Membuat file CSV laporan produksi bulanan (gunakan `pandas` dan agregasi untuk total per bulan).
4. Gunakan environment variable untuk connection string.
5. Implementasikan logging ke file `app.log`.
6. Buat dokumentasi singkat (README) cara menjalankan.

### Rubrik Penilaian
| Kriteria                        | Bobot |
|---------------------------------|-------|
| Operasi CRUD & query            | 25%   |
| Agregasi dan analisis           | 25%   |
| Struktur kode & error handling  | 20%   |
| Integrasi pandas & file output  | 15%   |
| Dokumentasi & logging           | 15%   |

### Tenggat
Sebelum pertemuan ke-7, unggah di LMS (source code + laporan).

---

## H. REFERENSI PERTEMUAN 6
1. PyMongo Documentation. https://pymongo.readthedocs.io/
2. MongoDB. *MongoDB Python Developer Path*. https://learn.mongodb.com/
3. pandas documentation. https://pandas.pydata.org/docs/
4. Lutz, M. (2013). *Learning Python*. O'Reilly.

---

**Pesan Dosen:**
Python + MongoDB adalah pasangan kuat di industri 4.0. Kuasai integrasinya, dan Anda bisa membangun solusi nyata dari pengumpulan data hingga analisis cerdas. Selamat coding!
