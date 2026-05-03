# PERTEMUAN 8 – PROYEK AKHIR: STUDI KASUS INDUSTRI TERINTEGRASI
**Program Studi:** Teknologi Rekayasa Informatika Industri (TRIN)
**Alokasi Waktu:** 1 × 150 menit (praktikum) + pengerjaan mandiri

---

## A. CAPAIAN PEMBELAJARAN PERTEMUAN 8
Setelah menyelesaikan pertemuan ini dan proyek akhir, mahasiswa mampu:
1. Menganalisis kebutuhan penyimpanan dan pengolahan data pada skenario industri nyata.
2. Merancang model data MongoDB (embedded/reference) yang optimal untuk kebutuhan query dan pertumbuhan data.
3. Mengimplementasikan skema, memasukkan data (dummy/generator), serta membuat indeks yang mendukung performa.
4. Membangun pipeline agregasi yang menjawab pertanyaan bisnis spesifik (KPI, laporan, analisis).
5. Mengintegrasikan MongoDB dengan Python untuk otomasi input data atau analisis lanjut.
6. Menerapkan prinsip replikasi dan backup untuk menjamin ketersediaan data.
7. Mengevaluasi performa query dengan `explain()` dan melakukan optimasi.
8. Mendokumentasikan seluruh proses dan mempresentasikan solusi secara profesional.

---

## B. ALAT DAN BAHAN
- **Infrastruktur:**
  - Server MongoDB lokal (disarankan replica set minimal 1 node, jika memungkinkan 3 node untuk nilai tambah).
  - MongoDB Shell (`mongosh`) dan Compass.
  - Python 3.10+ dengan PyMongo, pandas, matplotlib (jika diperlukan).
- **Data:**
  - Dataset dummy yang dibuat menggunakan script Python atau `mgenerate`.
  - Atau dataset publik yang relevan (misal dari Kaggle predictive maintenance).
- **Dokumentasi:**
  - Template laporan proyek (disediakan dosen).
  - Draw.io atau alat diagram untuk ERD/skema.
- **Manajemen Proyek:**
  - Git repository untuk menyimpan kode dan laporan.
- **Presentasi:**
  - Slide deck (PowerPoint/Google Slides) untuk presentasi 5–7 menit.

---

## C. URAIAN MATERI SUPER LENGKAP

### 1. Metodologi Pengerjaan Proyek
Proyek ini mengikuti siklus sederhana **Analyze – Design – Build – Test – Demonstrate**. Setiap fase dijelaskan di bawah.

#### a. Analisis Kebutuhan (Requirements Analysis)
- Pelajari skenario industri yang dipilih.
- Identifikasi entitas utama, hubungan, volume data, frekuensi baca/tulis.
- Tentukan pertanyaan bisnis yang harus dijawab (misal: “mesin mana yang paling sering mengalami reject?”, “berapakah OEE per lini?”, “kapan stok part harus diisi ulang?”).

#### b. Perancangan Model Data
- Pilih pendekatan embedding atau referencing berdasarkan akses data.
- Buat diagram koleksi dan contoh dokumen.
- Tentukan indeks yang diperlukan berdasarkan query yang akan dijalankan.
- Dokumentasikan alasan desain (trade-off).

#### c. Implementasi
- Buat database dan koleksi di MongoDB.
- Generate data dummy (minimal 1000 dokumen, disarankan >5000 untuk menunjukkan performa).
- Buat indeks.
- Bangun pipeline agregasi untuk menjawab pertanyaan bisnis (minimal 3 pipeline berbeda).
- (Opsional) Buat script Python untuk input data otomatis atau analisis.

#### d. Pengujian dan Optimasi
- Gunakan `explain("executionStats")` pada query utama, catat hasil sebelum dan sesudah indeks.
- Lakukan optimasi pipeline agregasi (pindahkan `$match` ke awal, proyeksi minimal).
- Jika ada replikasi, uji failover dan baca dari secondary.

#### e. Dokumentasi dan Presentasi
- Susun laporan lengkap (format akan dijelaskan di bagian Tugas).
- Siapkan demo singkat: jalankan query, tampilkan hasil, tunjukkan explain plan.

### 2. Indikator Keberhasilan Proyek (Rubrik Detail)
Proyek dinilai berdasarkan:
- **Kesesuaian Skema (25%):** Apakah model data mendukung semua kebutuhan query? Apakah pilihan embed/reference tepat? Apakah ada justifikasi?
- **Indeks dan Performa (20%):** Apakah indeks digunakan dengan benar? Apakah ada bukti perbaikan performa (explain)? Apakah ada covered query?
- **Kompleksitas Agregasi (20%):** Apakah pipeline menjawab pertanyaan bisnis dengan benar? Apakah menggunakan stage yang tepat (`$group`, `$lookup`, `$unwind`, dll.)?
- **Integrasi dan Otomasi (15%):** Apakah ada script Python? Apakah koneksi aman? Apakah data dummy dihasilkan secara realistis?
- **Dokumentasi dan Presentasi (20%):** Apakah laporan terstruktur? Apakah diagram jelas? Apakah presentasi meyakinkan?

### 3. Panduan Teknis Praktis

**a. Skrip Pembangkit Data Dummy (Python)**
```python
from pymongo import MongoClient
from datetime import datetime, timedelta
import random

client = MongoClient('mongodb://localhost:27017/')
db = client.proyek_industri

# Generate data sensor
mesin_list = ['M' + str(i).zfill(3) for i in range(1, 11)]
start = datetime(2026, 5, 1)
for i in range(5000):
    doc = {
        'mesin': random.choice(mesin_list),
        'suhu': round(random.uniform(60, 100), 2),
        'getaran': round(random.uniform(0.1, 0.8), 2),
        'timestamp': start + timedelta(minutes=i)
    }
    db.sensor.insert_one(doc)
```
Pastikan melakukan batch insert untuk efisiensi.
**b. Template Pipeline Agregasi yang Efisien**
- Selalu filter dengan `$match` menggunakan rentang waktu atau mesin.
- Jika perlu join, lakukan setelah `$match` untuk mengurangi data yang di-join.
- Gunakan `$project` untuk menghapus field besar sebelum `$group`.

**c. Checklist Evaluasi Mandiri**
- [ ] Apakah setiap koleksi memiliki `_id` yang unik?
- [ ] Apakah field yang sering dicari sudah diindeks?
- [ ] Apakah semua stage agregasi diperlukan?
- [ ] Apakah dokumen tidak melebihi 16 MB?
- [ ] Apakah backup sudah dilakukan?
- [ ] Apakah script Python menangani error koneksi?

### 4. Contoh Skenario Proyek (Studi Kasus)
Berikut tiga pilihan yang dapat dipilih oleh kelompok (atau ditentukan dosen):

#### Kasus 1: Sistem Monitoring Kualitas Lingkungan Pabrik
- **Deskripsi:** 20 sensor suhu, kelembaban, dan CO2 di seluruh pabrik mengirim data setiap menit. Manajemen ingin dashboard real-time suhu tertinggi, rata-rata per jam, dan alarm jika CO2 > 1000 ppm.
- **Koleksi:** `sensor_readings` (reference `sensor_id` ke `sensors`).
- **Pertanyaan:**
  1. 5 sensor dengan suhu rata-rata tertinggi dalam 24 jam terakhir.
  2. Jumlah alarm (CO2 > 1000) per jam.
  3. Rata-rata kelembaban per area (gunakan lookup).

#### Kasus 2: Sistem Manajemen Perawatan Prediktif
- **Deskripsi:** 10 mesin memiliki jadwal perawatan rutin dan riwayat kerusakan. Setiap kerusakan dicatat dengan part yang diganti. Teknisi perlu tahu mesin yang akan jatuh tempo perawatan, part yang sering rusak, dan biaya perawatan bulanan.
- **Koleksi:** `machines`, `maintenance_schedules`, `breakdown_logs` (reference).
- **Pertanyaan:**
  1. Mesin yang jadwal perawatannya dalam 7 hari ke depan.
  2. Top 3 part dengan frekuensi penggantian tertinggi.
  3. Total biaya perawatan per mesin per bulan (agregasi + lookup).

#### Kasus 3: Sistem Pelacakan Produksi dan Cacat
- **Deskripsi:** Setiap shift, setiap mesin menghasilkan batch produk dengan jumlah ok dan reject. Inspektor mencatat jenis cacat (array). Manajemen ingin reject rate per mesin, per produk, dan tren bulanan.
- **Koleksi:** `production_batches`, `products` (reference).
- **Pertanyaan:**
  1. Reject rate (dalam %) per mesin minggu ini.
  2. Jenis cacat paling sering muncul di suatu produk.
  3. Perbandingan total produksi antar shift.

---

## D. KEGIATAN PRAKTIKUM (120 MENIT) – **Sprint Proyek di Kelas**
Waktu digunakan untuk memulai proyek, dengan bimbingan dosen/asisten.

### Fase 1: Penjelasan & Pembagian Kelompok (10 menit)
- Dosen menjelaskan skenario dan ekspektasi.
- Mahasiswa membentuk kelompok (2-3 orang), memilih skenario.

### Fase 2: Analisis & Desain Awal (20 menit)
- Diskusi internal: tentukan kebutuhan query, buat skema awal di kertas/whiteboard.
- Tuliskan 3 pertanyaan bisnis yang akan dijawab.
- Konsultasi singkat dengan dosen.

### Fase 3: Setup Awal & Data Dummy (30 menit)
- Buat database proyek.
- Tulis script Python sederhana untuk generate minimal 2000 dokumen (sebagai starter, sisanya dilanjutkan di luar).
- Insert data ke koleksi.
- Buat indeks awal berdasarkan desain.

### Fase 4: Implementasi Pipeline Pertama (30 menit)
- Tulis pipeline agregasi untuk satu pertanyaan bisnis.
- Jalankan dan verifikasi hasil.
- Gunakan `explain()` untuk melihat statistik awal.

### Fase 5: Diskusi Kendala & Penyesuaian (20 menit)
- Setiap kelompok melaporkan kemajuan.
- Bahas masalah umum (misal format tanggal, array unwind).
- Dosen memberikan tips.

### Fase 6: Penutup & Tugas Lanjutan (10 menit)
- Instruksi untuk menyelesaikan proyek di luar kelas.
- Deadline pengumpulan.
- Sesi tanya jawab.

---

## E. LATIHAN MANDIRI (Self-Assessment & Pengembangan)
Sebelum mengumpulkan proyek, lakukan evaluasi internal dengan pertanyaan:
1. Apakah model data sudah mendukung semua query tanpa join berlebihan?
2. Apakah ada indeks yang tidak terpakai? Cek dengan `db.collection.aggregate([ { $indexStats: {} } ])`.
3. Apakah pipeline agregasi bisa dioptimasi lebih lanjut? (coba ubah urutan stage, atau gunakan `$limit`).
4. Apakah Anda sudah mencoba simulasi beban dengan banyak data (misal 10.000 dokumen)? Bagaimana pengaruhnya?
5. Apakah Anda sudah menyiapkan backup dan restore dari database proyek?
6. Apakah script Python memiliki dokumentasi singkat dan penanganan error?

---

## F. STUDI KASUS (Pilihan Proyek yang Diperdalam)
Untuk memandu lebih rinci, berikut ekspansi salah satu skenario: **Sistem Pelacakan Produksi dan Cacat**.
- **Latar Belakang:** Pabrik tekstil memiliki 6 mesin weaving, beroperasi 3 shift. Setiap 30 menit, operator mencatat hasil: panjang kain (meter), jumlah cacat, jenis cacat (putus benang, noda, dll). Data ini tersimpan di MongoDB. Pihak QC ingin laporan otomatis.
- **Tantangan:** Data historis 6 bulan dengan total 100.000 record. Laporan harian dan mingguan harus cepat.
- **Tugas Spesifik:**
  1. Desain model data (disarankan koleksi `produksi` dengan embed array cacat).
  2. Buat indeks compound `{mesin:1, tanggal:1}`.
  3. Agregasi: reject rate per mesin per minggu, distribusi jenis cacat per bulan, total produksi per shift per hari.
  4. Script Python untuk mengambil data mingguan dan mengekspor ke CSV.
  5. Backup rutin menggunakan crontab.
- Luaran yang diharapkan: laporan lengkap, kode, dan presentasi.

---

## G. TUGAS TERSTRUKTUR (PROYEK AKHIR)
### A. Deskripsi Umum
Setiap kelompok (2-3 orang) harus mengembangkan solusi basis data MongoDB untuk salah satu studi kasus industri yang telah ditentukan. Proyek mencakup seluruh siklus hidup: analisis, desain, implementasi, pengujian, dokumentasi, dan presentasi.

### B. Deliverables yang Dikumpulkan
1. **Laporan Proyek (PDF)** dengan struktur:
   - Halaman Judul (judul proyek, nama kelompok, NIM, mata kuliah)
   - Abstrak / Ringkasan Eksekutif
   - Bab 1: Pendahuluan (latar belakang, masalah, tujuan)
   - Bab 2: Analisis Kebutuhan (entitas, query, volume)
   - Bab 3: Perancangan Model Data (diagram koleksi, contoh dokumen, justifikasi)
   - Bab 4: Implementasi (skrip pembuatan indeks, kode Python jika ada, pipeline agregasi utama)
   - Bab 5: Pengujian dan Optimasi (hasil explain sebelum/sesudah indeks, pembahasan)
   - Bab 6: Kesimpulan dan Saran
   - Lampiran: semua kode sumber, capture hasil query
2. **Repositori Git** (link) berisi:
   - Script Python dan/atau file JavaScript untuk shell.
   - File export data (opsional).
   - README.md (cara menjalankan).
3. **Presentasi** (slide) maksimal 10 slide, untuk dipresentasikan di waktu yang ditentukan.

### C. Ketentuan Teknis
- Database harus memiliki minimal 2 koleksi dengan hubungan reference atau embedded.
- Data dummy harus mencerminkan volume realistis (minimal 5000 dokumen di koleksi utama).
- Harus menggunakan minimal 3 stage agregasi berbeda dalam pipeline final.
- Wajib menyertakan bukti penggunaan `explain()`.
- Opsional (nilai tambah): implementasi replika set atau TTL index.

### D. Jadwal dan Tenggat
- Pertemuan 8 (hari ini): kick-off, diskusi, mulai implementasi awal.
- Tenggat pengumpulan laporan dan kode: **1 minggu setelah pertemuan 8** pukul 23:59 melalui LMS.
- Presentasi: dijadwalkan pada sesi Ujian Akhir Semester (waktu diinformasikan kemudian).

### E. Rubrik Penilaian Proyek
| Kriteria                            | Sangat Baik (5) | Baik (4) | Cukup (3) | Kurang (<3) |
|-------------------------------------|-----------------|----------|-----------|-------------|
| Kesesuaian dan inovasi model data   |                 |          |           |             |
| Implementasi indeks & bukti optimasi|                 |          |           |             |
| Pipeline agregasi (kebenaran & kompleksitas) |          |          |           |             |
| Integrasi Python (jika ada)         |                 |          |           |             |
| Dokumentasi & kode yang rapi        |                 |          |           |             |
| Presentasi & demonstrasi            |                 |          |           |             |
**Total Bobot:** masing-masing proporsional (lihat rincian di bawah)
- Model data (25%)
- Indeks & performa (20%)
- Agregasi (20%)
- Integrasi/otomasi (15%)
- Dokumentasi & presentasi (20%)

### F. Catatan Penting
- Plagiarisme akan ditindak tegas.
- Jika menggunakan referensi eksternal (dataset, library), cantumkan sumbernya.
- Setiap anggota kelompok harus memahami semua bagian proyek, karena akan ditanyai saat presentasi.

---

## H. REFERENSI PERTEMUAN 8
1. MongoDB. *MongoDB Architecture Guide*. https://www.mongodb.com/collateral/mongodb-architecture-guide
2. MongoDB University. *M100: MongoDB for SQL Pros* (perbandingan).
3. Celko, J. (2014). *Joe Celko's Complete Guide to NoSQL*. Morgan Kaufmann.
4. Fowler, M. *NoSQL Distilled*. Addison-Wesley.

---

**Pesan Dosen:**
Proyek ini adalah puncak dari semua pembelajaran Anda. Manfaatkan semua keterampilan yang telah diasah. Dunia industri membutuhkan solusi nyata, dan ini kesempatan Anda untuk membuktikan kemampuan. Jangan ragu berkreasi dan bertanya. Semangat!
