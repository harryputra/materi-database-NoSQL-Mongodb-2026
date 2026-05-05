# PERTEMUAN 6 – MONGODB DENGAN PYTHON (PYMONGO)

**Disertai Simulasi Data Sensor Melalui MQTT untuk Aplikasi IoT Industri**

---

## CAPAIAN PEMBELAJARAN PERTEMUAN 6
Setelah menyelesaikan pertemuan keenam ini, mahasiswa akan memiliki kemampuan yang komprehensif dalam mengintegrasikan bahasa pemrograman Python dengan sistem basis data MongoDB menggunakan driver resmi PyMongo, serta membangun simulasi aliran data sensor yang mencerminkan arsitektur Internet of Things di dunia industri. Mahasiswa akan mampu menginstal dan mengonfigurasi seluruh pustaka pendukung—yakni `pymongo`, `pandas`, `python-dotenv`, `matplotlib`, dan `paho-mqtt`—di dalam lingkungan virtual Python yang terisolasi, sehingga proyek tetap bersih dan dependensinya terkendali. Mahasiswa juga akan terampil menghubungkan aplikasi Python ke server MongoDB, baik yang berjalan di mesin lokal maupun di platform cloud seperti MongoDB Atlas, dengan memanfaatkan connection string yang disimpan secara aman di dalam file environment. Setelah koneksi terbangun, mahasiswa akan melaksanakan operasi CRUD secara langsung dari kode Python; mereka akan menyisipkan dokumen sensor, melakukan pencarian dengan berbagai kondisi, memperbarui field tertentu menggunakan operator update, serta menghapus data berdasarkan kriteria waktu. Selanjutnya, mahasiswa akan menyusun query kompleks dan aggregation pipeline melalui PyMongo untuk melakukan pengelompokan, penyaringan, dan perhitungan statistik di sisi server, yang hasilnya kemudian dialirkan ke dalam pandas DataFrame. Di dalam ekosistem pandas, mahasiswa akan menjalankan analisis data industri—menampilkan statistik deskriptif, meresampling data time series, dan membuat visualisasi informatif dengan matplotlib. Secara paralel, mahasiswa akan membangun skrip otomatis yang menghasilkan data sensor tiruan secara periodik; data tersebut dapat langsung disisipkan ke MongoDB maupun dikirim melalui protokol MQTT menggunakan broker, meniru arus informasi dari perangkat sensor di lapangan. Dalam setiap skrip, mahasiswa akan menerapkan penanganan error berbasis `try-except` dan mencatat kejadian penting ke dalam berkas log, sehingga aplikasi lebih tangguh dan mudah ditelusuri. Akhirnya, mahasiswa akan memahami dan mempraktikkan best practice koneksi database, seperti penggunaan connection pooling, penyimpanan kredensial di environment variable, dan pengelolaan koneksi melalui context manager.

## ALAT DAN BAHAN
Untuk dapat mengikuti seluruh rangkaian praktikum ini, mahasiswa perlu menyiapkan sejumlah perangkat lunak dan konfigurasi. Komputer harus telah terpasang Python dengan versi minimal 3.8, meskipun sangat disarankan menggunakan versi 3.10 ke atas untuk memastikan kompatibilitas dan dukungan fitur terbaru. Bersama Python, pastikan pengelola paket `pip` sudah tersedia dan dapat digunakan melalui terminal. Lingkungan kerja akan diisolasi dengan virtual environment, bisa menggunakan `venv` bawaan Python atau `conda` bagi yang terbiasa dengan distribusi Anaconda.

Pustaka Python yang akan diinstal mencakup lima komponen. Pertama, `pymongo` versi 4.x sebagai driver resmi yang menjembatani Python dan MongoDB. Kedua, `pandas` yang menjadi andalan untuk manipulasi dan analisis data tabular. Ketiga, `python-dotenv` yang akan membaca konfigurasi sensitif dari file `.env`. Keempat, `matplotlib` untuk menghasilkan grafik dan visualisasi data. Kelima, `paho-mqtt` yang memungkinkan Python berkomunikasi melalui protokol MQTT, protokol ringan yang banyak digunakan di dunia IoT. Seluruh pustaka ini akan dipasang dalam lingkungan virtual yang sudah diaktifkan.

Di sisi server basis data, mahasiswa harus memiliki MongoDB yang berjalan di lingkungan lokal. Pastikan MongoDB server telah diinstal dan aktif, biasanya dapat diakses melalui `localhost` pada port `27017`. Untuk memverifikasi data secara visual, MongoDB Compass—aplikasi GUI resmi—akan sangat membantu. Sementara itu, untuk simulasi MQTT, mahasiswa memerlukan sebuah MQTT broker. Dua pilihan disediakan: menggunakan broker publik `test.mosquitto.org` yang tidak memerlukan instalasi, atau menginstal dan menjalankan broker lokal Mosquitto. Terakhir, mahasiswa akan menulis kode menggunakan IDE atau editor teks seperti Visual Studio Code, PyCharm, atau editor lainnya, serta mengeksekusi perintah melalui terminal atau command prompt.

## URAIAN MATERI 

### 1. Mengapa Integrasi Python–MongoDB Menjadi Pilar Industri Modern
Revolusi Industri 4.0 telah mengubah lanskap manufaktur dan otomasi. Di pabrik-pabrik modern, setiap mesin, sensor, dan sistem kontrol menghasilkan aliran data yang konstan—data suhu, getaran, tekanan, log produksi, dan parameter kualitas. Agar data ini tidak sekadar menguap, diperlukan sebuah sistem yang mampu mengumpulkan, menyimpan, dan menganalisisnya secara efisien. Di sinilah Python dan MongoDB berpadu. Python, dengan sintaksis yang bersih dan ekosistem pustaka yang sangat kaya, telah menjadi bahasa de facto untuk otomasi, analisis data, dan pengembangan IoT. MongoDB, di sisi lain, adalah basis data dokumen NoSQL yang menyimpan data dalam format BSON yang fleksibel, sangat mirip dengan dictionary Python. Ia tidak memerlukan skema yang kaku, sehingga sangat cocok untuk menampung data semi-terstruktur yang lahir dari sensor yang berbeda-beda spesifikasinya. Integrasi antara keduanya melalui PyMongo membuka jalan bagi para insinyur untuk membangun solusi nyata: dari otomatisasi pencatatan data mesin, analisis real-time dan historis, hingga pembuatan dashboard sederhana untuk pemantauan kinerja pabrik. Dengan menambahkan protokol MQTT, mahasiswa juga akan memahami bagaimana data mengalir dari perangkat lapangan menuju pusat data, sebuah arsitektur yang menjadi tulang punggung Internet of Things industri.

### 2. Arsitektur Integrasi: Dari Kode Python Hingga ke Koleksi MongoDB
Secara arsitektural, hubungan antara aplikasi Python dan server MongoDB dijembatani oleh sebuah objek bernama `MongoClient`. Objek ini bertindak sebagai gerbang tunggal; ketika dibuat, ia sudah menyediakan sebuah kumpulan koneksi (connection pool) yang siap digunakan. Default-nya, pool ini dapat menampung hingga 100 koneksi, cukup untuk sebagian besar aplikasi. Dalam praktik yang baik, kita hanya membuat satu instance `MongoClient` untuk seluruh aplikasi, kemudian membagikannya ke modul-modul lain. Dari `MongoClient` inilah kita mendapatkan akses ke database dan koleksi. Setiap operasi—menyisipkan, mencari, memperbarui, atau menghapus—dikirim melalui driver PyMongo ke server, yang kemudian mengembalikan hasil dalam bentuk Cursor atau objek lain. Cursor bersifat lazy; ia tidak langsung mengambil seluruh dokumen, melainkan mengambil per batch, sehingga hemat memori. Hasil dari Cursor dapat dengan mudah dikonversi menjadi list, diiterasi, atau langsung dibungkus ke dalam pandas DataFrame untuk analisis lebih lanjut.

Ketika MQTT ikut terlibat, arsitekturnya menjadi lebih realistis. Bayangkan puluhan sensor suhu terpasang di mesin-mesin produksi. Alih-alih menulis langsung ke database, setiap sensor akan mengirimkan datanya ke sebuah entitas bernama MQTT broker melalui protokol publish/subscribe. Sensor bertindak sebagai publisher, mengirim data ke topik tertentu, misalnya `pabrik/sensor/suhu`. Di sisi lain, sebuah aplikasi Python yang bertindak sebagai subscriber akan mendengarkan topik yang sama. Setiap kali broker menerima pesan, ia akan mendistribusikannya ke semua subscriber yang sedang terhubung. Aplikasi subscriber kemudian memproses payload (yang biasanya berformat JSON), melakukan parsing, dan baru kemudian menyimpannya ke MongoDB. Pola ini memisahkan produsen data dari konsumen data dengan sangat bersih, memungkinkan skalabilitas dan keandalan yang tinggi.

### 3. Menyiapkan Lingkungan Kerja yang Sesungguhnya
Sebelum kita menulis sebaris kode pun, penting untuk menyiapkan lingkungan kerja yang terisolasi. Membiarkan proyek menggunakan instalasi Python global akan menyebabkan bentrokan versi pustaka, terutama ketika kita menangani banyak proyek dengan kebutuhan yang berbeda. Virtual environment adalah solusinya. Dengan perintah `python -m venv venv_mongo`, Python akan menciptakan sebuah folder bernama `venv_mongo` yang berisi salinan interpreter Python dan direktori khusus untuk pustaka. Setelah environment ini diaktifkan—melalui `source venv_mongo/bin/activate` di Linux/macOS atau `venv_mongo\Scripts\activate` di Windows—setiap perintah `pip install` akan memasang pustaka di dalam environment tersebut, bukan secara global. Inilah saatnya untuk memasang kelima pustaka yang diperlukan: `pip install pymongo pandas python-dotenv matplotlib paho-mqtt`. Proses ini akan mengunduh driver PyMongo, pustaka data science pandas, manajer environment dotenv, visualisasi matplotlib, dan klien MQTT paho-mqtt. Setelah selesai, kita bisa melanjutkan dengan membuat file `.env`. File ini hanyalah teks biasa yang berisi pasangan kunci dan nilai, misalnya `MONGO_URI=mongodb://localhost:27017` dan `DB_NAME=praktikum6`. Dengan menempatkan URI dan nama database di sini, kita memisahkan konfigurasi dari logika aplikasi, sebuah praktik yang sangat dianjurkan karena memudahkan pemindahan aplikasi ke server lain atau penggunaan kredensial yang berbeda tanpa harus mengubah kode sumber. Jangan lupa untuk menambahkan file `.env` ke dalam `.gitignore` agar tidak ikut terunggah ke repositori publik.

### 4. Membuka Gerbang Koneksi dengan PyMongo
Koneksi pertama Anda adalah momen yang menentukan. Buatlah sebuah file bernama `koneksi.py`. Di dalam file ini, mulailah dengan mengimpor modul yang diperlukan: `os`, `load_dotenv` dari `dotenv`, dan `MongoClient` dari `pymongo`. Panggil `load_dotenv()` agar seluruh variabel di file `.env` termuat ke dalam `os.environ`. Kemudian ambil URI dari environment dengan `os.getenv("MONGO_URI")`. Selanjutnya, ciptakan instance `MongoClient` dengan parameter URI tersebut dan tambahan `serverSelectionTimeoutMS=5000`. Parameter timeout ini sangat penting agar program tidak menggantung tanpa batas jika server MongoDB tidak dapat dijangkau. Ia memberi tahu driver untuk menunggu paling lama 5 detik saat mencoba memilih server; jika gagal, akan dilempar pengecualian `ServerSelectionTimeoutError`.

Untuk memastikan semuanya berfungsi, kirimkan perintah `ping` ke server. Ini dilakukan dengan `client.admin.command('ping')`. Karena operasi ini bisa gagal, bungkuslah di dalam blok `try-except`. Di dalam `try`, cetak pesan “Koneksi MongoDB berhasil!”; di dalam `except`, tangkap `Exception` dan cetak pesan kesalahan yang terjadi. Terakhir, apapun yang terjadi, pastikan koneksi ditutup dengan `client.close()` di blok `finally`, atau gunakan context manager `with MongoClient(...) as client:` yang secara otomatis akan menutup koneksi saat blok selesai. Dari sini, kita memperoleh referensi ke database dengan `client["praktikum6"]` dan ke koleksi dengan `db["sensor"]`. Struktur tiga tingkat ini—client, database, koleksi—akan selalu menjadi fondasi setiap operasi database yang kita lakukan.

### 5. Menguasai Operasi CRUD Langsung dari Python
Untuk menghidupkan data, kita akan bekerja dengan koleksi bernama `sensor`. Setiap dokumen di dalamnya akan merepresentasikan satu kali pembacaan sensor dari mesin produksi. Struktur tipikalnya akan mencakup `mesin` (string, seperti "CNC-01"), `suhu` (float, dalam derajat Celsius), `getaran` (float, dalam satuan tertentu), `timestamp` (datetime UTC), dan `status` (string, misal "normal" atau "maintenance").

Operasi penciptaan data dilakukan dengan dua fungsi: `insert_one` dan `insert_many`. Ketika kita memiliki satu buah dictionary Python yang mewakili dokumen, kita panggil `collection.insert_one(doc)`. Hasil panggilan ini adalah objek `InsertOneResult` yang memiliki atribut `inserted_id`, sebuah ObjectId unik yang dihasilkan oleh MongoDB. Sebagai contoh, kita dapat membangun dictionary `doc` dengan field-field di atas, mengisi `timestamp` menggunakan `datetime.utcnow()`, lalu menyisipkannya. Jika kita ingin menyisipkan banyak data sekaligus—katakanlah 10 data sensor—kita kumpulkan semua dictionary itu ke dalam sebuah list, lalu panggil `collection.insert_many(list_of_dicts)`. Hasilnya adalah objek `InsertManyResult` dengan atribut `inserted_ids` yang berisi list ObjectId. Mengapa `insert_many` lebih baik daripada `insert_one` di dalam loop? Karena `insert_many` hanya mengirim satu perintah ke server untuk seluruh batch, mengurangi overhead jaringan dan meningkatkan throughput secara signifikan.

Untuk membaca data, PyMongo menyediakan `find` dan `find_one`. `find` mengembalikan objek Cursor yang dapat diiterasi. Kita bisa memberikan filter sebagai argumen pertama, misalnya `{"mesin": "CNC-01"}` untuk mengambil semua dokumen dengan mesin tersebut. Argumen kedua adalah projection, misalnya `{"_id": 0, "suhu": 1, "timestamp": 1}` yang berarti kita hanya ingin field `suhu` dan `timestamp` tanpa `_id`. Cursor juga mendukung method chaining seperti `.sort("timestamp", -1)` untuk mengurutkan menurun, dan `.limit(10)` untuk membatasi jumlah hasil. Karena Cursor hanya bisa diiterasi sekali, jika kita perlu menyimpan seluruh hasil, kita bungkus dengan `list(cursor)`. `find_one` bekerja serupa, tetapi langsung mengembalikan satu dokumen atau `None` jika tidak ditemukan.

Operasi update menggunakan `update_one` atau `update_many`, masing-masing menerima filter dan dokumen update yang berisi operator. Operator yang sering dipakai adalah `$set` untuk menetapkan nilai field, `$inc` untuk menambah nilai numerik, dan `$unset` untuk menghapus field. Contoh kasus: kita ingin menambah jumlah reject pada mesin CNC-01 dan mengubah statusnya menjadi "diperbaiki". Maka kita tulis: `collection.update_one({"mesin": "CNC-01"}, {"$inc": {"reject_count": 1}, "$set": {"status": "diperbaiki"}})`. Opsi `upsert=True` dapat diberikan; jika tidak ada dokumen yang cocok, dokumen baru akan disisipkan. Hasil update memberi tahu berapa banyak dokumen yang cocok (`matched_count`) dan berapa yang benar-benar diubah (`modified_count`).

Untuk menghapus, kita memiliki `delete_one` dan `delete_many`. Filter yang sama digunakan untuk menandai dokumen mana yang akan dihapus. Misalnya, untuk membersihkan data yang sudah sangat lama, kita bisa menulis `collection.delete_many({"timestamp": {"$lt": datetime(2026, 1, 1)}})`. Semua operasi ini, jika tidak ditangani, dapat melemparkan pengecualian yang akan kita pelajari pada bagian error handling.

### 6. Mengelola Cursor dan Pemrosesan Batch Secara Bijak
Saat berhadapan dengan volume data yang besar, memahami perilaku Cursor sangatlah penting. Cursor tidak langsung menarik semua dokumen dari server; ia bekerja secara batch. Secara default, satu batch berisi 101 dokumen. Ketika kita mulai mengiterasi Cursor, PyMongo akan mengambil batch pertama. Setelah batch itu habis, ia akan secara otomatis meminta batch berikutnya dari server. Proses ini terus berlanjut hingga tidak ada lagi dokumen yang memenuhi filter. Kita bisa mengubah ukuran batch dengan `batch_size(500)`. Mengapa ini penting? Karena jika kita memproses jutaan dokumen, menarik semuanya sekaligus ke dalam memori bisa menyebabkan aplikasi kehabisan RAM. Dengan Cursor, kita memproses secara streaming, lebih hemat sumber daya. Namun, perlu diingat bahwa setelah iterasi selesai, Cursor tidak dapat diulang. Apabila kita membutuhkan data yang sama untuk dua keperluan berbeda, solusinya adalah mengonversi Cursor menjadi list dengan `list(cursor)` sebelum iterasi pertama, atau menjalankan query ulang. Untuk pemantauan pada dataset besar, kita bisa menyisipkan pencetakan status setiap beberapa ribu dokumen agar kita tahu proses berjalan normal.

### 7. Kekuatan Agregasi: Menganalisis Data di Sisi Server
Aggregation pipeline adalah fitur MongoDB yang memungkinkan kita mendorong logika pemrosesan data ke server database. Ini sangat menguntungkan karena mengurangi jumlah data yang harus dikirim melalui jaringan dan memanfaatkan mesin database yang sudah dioptimalkan untuk komputasi semacam itu. Di PyMongo, pipeline agregasi dieksekusi dengan `collection.aggregate(pipeline)`, di mana `pipeline` adalah sebuah list yang setiap elemennya adalah sebuah stage (dictionary). Stage-stage tersebut akan dijalankan secara berurutan, output dari satu stage menjadi input stage berikutnya.

Sebagai contoh, kita ingin mengetahui suhu maksimum dan rata-rata getaran untuk setiap mesin, tetapi hanya untuk data yang suhunya di atas 80 derajat. Pertama, kita gunakan stage `$match` dengan filter `{"suhu": {"$gt": 80}}` untuk menyaring dokumen. Kedua, stage `$group` dengan `_id: "$mesin"` untuk mengelompokkan berdasarkan mesin, lalu di dalamnya kita hitung `max_suhu: {"$max": "$suhu"}`, `avg_getaran: {"$avg": "$getaran"}`, dan `count: {"$sum": 1}`. Terakhir, stage `$sort` dengan `{"max_suhu": -1}` untuk mengurutkan dari suhu tertinggi. Metode `aggregate` mengembalikan Cursor, sehingga kita bisa mencetak hasilnya dengan loop `for`. Hasil yang muncul akan langsung berupa ringkasan per mesin, tanpa kita perlu menarik seluruh data mentah ke Python. Pemahaman yang kuat tentang agregasi akan memungkinkan mahasiswa membangun laporan dan analisis kompleks dengan kode yang ringkas dan efisien.

### 8. Perkawinan PyMongo dan pandas: Dari Dokumen ke Insight
Salah satu keunggulan terbesar Python adalah kemampuannya untuk menjembatani database dengan analisis data. Setelah kita memiliki data di MongoDB, kita dapat menariknya ke dalam pandas DataFrame hanya dalam dua langkah. Pertama, ambil hasil query menggunakan `find` atau `aggregate`, lalu konversi menjadi list. Kedua, panggil `pd.DataFrame(list_of_dicts)`. Seketika, data dokumen kita berubah menjadi tabel dua dimensi dengan kolom-kolom yang sesuai dengan field dokumen. Field `_id` yang berupa ObjectId mungkin tidak kita perlukan; kita bisa menghapusnya dengan `df.drop(columns=['_id'], inplace=True)`. Kolom `timestamp` yang semula bertipe `datetime` Python harus kita konversi menggunakan `pd.to_datetime(df['timestamp'])` agar pandas mengenalinya sebagai data temporal. Setelah itu, kita bisa menetapkan kolom tersebut sebagai index dengan `df.set_index('timestamp', inplace=True)`, membuka jalan bagi operasi time series seperti resampling. Perintah `df.resample('5T').mean()` akan menghitung rata-rata setiap lima menit. Fungsi `describe()` memberikan gambaran statistik deskriptif secara instan: count, mean, std, min, max, dan kuartil. Dengan DataFrame ini, kita dapat melakukan analisis eksploratif, membersihkan data, hingga menyiapkannya untuk model machine learning.

### 9. Menghidupkan Data dengan Visualisasi Matplotlib
Angka dan tabel seringkali sulit dicerna. Visualisasi menerjemahkan data ke dalam bentuk yang langsung dipahami oleh otak manusia. Matplotlib adalah pustaka fondasi untuk membuat grafik di Python. Setelah kita memiliki DataFrame dengan index datetime, kita bisa langsung memanggil method `.plot()` pada kolom yang diinginkan. Contohnya, `df['suhu'].plot()` akan menghasilkan grafik garis dari kolom suhu sepanjang waktu. Kita bisa menambahkan judul dengan parameter `title`, label sumbu dengan `plt.ylabel()`, dan grid dengan `plt.grid(True)`. Untuk menyimpan grafik sebagai file gambar, gunakan `plt.savefig('suhu_plot.png')`. Jika skrip dijalankan pada server tanpa layar (headless), kita perlu mengaktifkan backend non-interaktif dengan menambahkan `import matplotlib; matplotlib.use('Agg')` di bagian paling atas sebelum mengimpor `pyplot`. Dengan cara ini, grafik tetap digambar dan disimpan tanpa mencoba membuka jendela tampilan. Kemampuan ini sangat berguna untuk otomatisasi laporan; skrip bisa berjalan di tengah malam, menghasilkan grafik, dan menyimpannya untuk ditinjau keesokan harinya.

### 10. Meniru Dunia Nyata: Data Generator Langsung ke MongoDB
Sebelum kita memiliki akses ke sensor sungguhan, simulasi adalah langkah awal yang penting. Kita dapat menulis sebuah skrip Python sederhana yang berperan sebagai generator data sensor. Skrip ini akan berjalan dalam loop tak terbatas. Di setiap iterasi, ia memilih secara acak salah satu dari lima mesin (misalnya `CNC-01` hingga `CNC-05`), menghasilkan suhu acak antara 60 hingga 100 derajat, getaran acak antara 0.1 hingga 0.5, dan mencatat timestamp saat itu menggunakan `datetime.utcnow()`. Dictionary ini kemudian langsung disisipkan ke MongoDB dengan `insert_one`. Setelah itu, skrip berhenti sejenak (sleep) selama dua detik sebelum mengulangi lagi. Dengan menjalankan skrip ini, koleksi `sensor` akan terisi secara konstan. Kita bisa membuka MongoDB Compass, menekan tombol refresh, dan menyaksikan dokumen baru bermunculan setiap dua detik. Ini memberikan pengalaman visual yang sangat kuat tentang bagaimana data real-time terakumulasi.

### 11. Melangkah ke Arsitektur IoT: Simulasi Data Sensor Melalui MQTT
Di dunia nyata, data sensor tidak dikirim langsung ke database oleh sensor itu sendiri, karena sensor seringkali berupa perangkat kecil dengan sumber daya terbatas yang hanya mampu mengirim data melalui protokol ringan. MQTT hadir sebagai protokol publish/subscribe yang mengandalkan broker sebagai perantara. Dalam simulasi ini, kita akan membagi peran menjadi dua: publisher dan subscriber.

Publisher bertindak seolah-olah ia adalah sensor. Dengan menggunakan pustaka `paho-mqtt`, publisher membuat sebuah client, menghubungkannya ke broker (misalnya `test.mosquitto.org` pada port 1883), lalu masuk ke dalam loop. Di setiap iterasi, ia membangun dictionary data sensor, mengonversinya menjadi string JSON menggunakan `json.dumps()`, dan menerbitkannya ke topik `pabrik/sensor/suhu` melalui `client.publish(topik, payload)`. Setelah itu, ia menunggu dua detik. Yang penting di sini adalah bahwa publisher tidak tahu dan tidak peduli siapa yang akan menerima data; ia hanya mengirim ke broker.

Subscriber adalah pihak yang lebih cerdas. Ia juga membuat client MQTT, tetapi kali ini ia mendefinisikan dua fungsi callback: `on_connect` dan `on_message`. Callback `on_connect` akan dipanggil ketika koneksi ke broker berhasil terjalin; di dalamnya, subscriber memanggil `client.subscribe("pabrik/sensor/suhu")` untuk mendaftarkan diri pada topik yang sama. Callback `on_message` dipanggil setiap kali broker mengirimkan pesan baru. Di dalam fungsi ini, kita mendekode payload dari bytes menjadi string, lalu mengubahnya kembali menjadi dictionary Python dengan `json.loads`. Field `timestamp` yang dikirim sebagai string ISO kita konversi menjadi objek `datetime` sejati. Setelah itu, dictionary tersebut kita sisipkan ke MongoDB menggunakan `insert_one`. Subscriber kemudian berjalan tanpa henti dengan `client.loop_forever()`. Untuk meningkatkan keandalan, kita dapat mengaktifkan mekanisme reconnect otomatis dengan `client.reconnect_delay_set(min_delay=1, max_delay=120)`. Arsitektur ini mencerminkan pemisahan tanggung jawab yang sering diterapkan di industri: perangkat lapangan hanya bertugas mengirim, sementara server pusat mengumpulkan dan menyimpan.

### 12. Menulis Kode yang Tangguh dengan Error Handling dan Logging
Dalam aplikasi nyata, kegagalan adalah keniscayaan. Server MongoDB bisa saja direstart, koneksi jaringan terputus, atau payload MQTT bisa saja korup. Agar aplikasi tidak langsung mati ketika menghadapi situasi ini, setiap operasi yang berpotensi gagal harus dibungkus dalam blok `try-except`. PyMongo menyediakan kelas pengecualian `PyMongoError` yang menjadi induk dari hampir semua error yang mungkin terjadi. Dengan menangkap `PyMongoError`, kita dapat mencatat pesan kesalahan yang informatif tanpa menghentikan proses secara keseluruhan.

Lebih jauh, alih-alih hanya mencetak ke layar, kita dapat menggunakan modul `logging` Python. Logging memungkinkan kita mengarahkan pesan ke beberapa output sekaligus: ke konsol untuk pemantauan langsung, dan ke file untuk audit di kemudian hari. Konfigurasi logging dasar dilakukan dengan `logging.basicConfig()`, menentukan level (misalnya `INFO`), format pesan yang mencakup timestamp, dan daftar handler. Setelah dikonfigurasi, kita tinggal memanggil `logging.info("Data berhasil disisipkan")` atau `logging.error("Gagal menyimpan data", exc_info=True)`. Dengan jejak log yang rapi, kita bisa mendiagnosis masalah tanpa harus membaca ulang seluruh kode atau bergantung pada traceback mentah.

### 13. Best Practice Koneksi Database yang Tidak Boleh Diabaikan
Beberapa kebiasaan baik akan menyelamatkan kita dari masalah di kemudian hari. Pertama, jangan pernah membuat `MongoClient` baru setiap kali Anda butuh database. Buat satu instance di awal aplikasi (misalnya di modul `db.py`) dan impor ke mana pun diperlukan. Ini karena setiap `MongoClient` membawa connection pool sendiri; membuat banyak instance akan menghabiskan sumber daya. Kedua, pastikan setiap koneksi ditutup. Jika menggunakan context manager (`with MongoClient(...) as client:`), penutupan terjadi otomatis. Jika tidak, panggil `client.close()` di blok `finally`. Ketiga, selalu berikan timeout dengan `serverSelectionTimeoutMS` agar aplikasi tidak hang. Keempat, pisahkan konfigurasi dari kode menggunakan environment variable. Kelima, jangan pernah mengekspos URI yang mengandung kredensial ke publik; file `.env` wajib masuk `.gitignore`.

### 14. Menjaga Keamanan Kredensial
File `.env` hanyalah berkas teks dengan format `KEY=VALUE`. Di Python, `python-dotenv` akan membacanya dan menyuntikkan variabel-variabel tersebut ke `os.environ`. Ini adalah lapisan keamanan minimal yang sangat efektif selama tahap pengembangan. Namun, ingatlah bahwa di environment production, kredensial seharusnya dikelola oleh layanan secret manager yang sebenarnya, seperti AWS Secrets Manager atau HashiCorp Vault. Untuk kebutuhan belajar, `.env` sudah sangat mencukupi. Intinya: jangan pernah menuliskan password atau connection string langsung di dalam kode.

### 15. Panduan Troubleshooting Ketika Sesuatu Tidak Berjalan
Server MongoDB tidak bisa dijangkau? Pesan `ServerSelectionTimeoutError` biasanya muncul. Solusinya: periksa apakah MongoDB benar-benar berjalan dengan mengetik `mongosh` di terminal. Jika bisa masuk, berarti server aktif; jika tidak, nyalakan service. Periksa juga konfigurasi `bindIp` di file `mongod.conf`, pastikan ia mengizinkan koneksi dari `127.0.0.1`. Di sisi MQTT, jika subscriber tidak menerima pesan, pastikan nama topik yang di-subscribe persis sama—termasuk huruf besar/kecil—dengan yang di-publish. Jika muncul `JSONDecodeError`, artinya payload yang diterima bukan JSON yang valid; ini bisa terjadi jika publisher mengirim data yang salah format. Dengan menambahkan logging pada setiap tahap, kita bisa melacak di mana letak masalahnya.

---

## KEGIATAN PRAKTIKUM TERBIMBING (120 MENIT)
Pada sesi praktikum ini, Anda akan dipandu langkah demi langkah untuk membangun berbagai komponen integrasi Python-MongoDB. Setiap langkah disertai penjelasan mengapa langkah tersebut harus dilakukan, sehingga Anda tidak sekadar menulis kode, melainkan memahami filosofi di baliknya.

### Fase 1: Setup Lingkungan (15 menit)
**Apa yang harus dikerjakan?**
Anda akan membuat folder proyek baru, mengaktifkan virtual environment, menginstal pustaka, dan menyiapkan file konfigurasi.

**Langkah-langkah dan kode yang harus ditulis:**
1. Buka terminal, arahkan ke direktori kerja Anda, lalu buat folder `praktikum6` dan masuk ke dalamnya.
   Mengapa? Karena mengorganisir file dalam satu folder memudahkan navigasi dan pengumpulan tugas.

2. Jalankan `python -m venv venv` untuk membuat virtual environment, lalu aktifkan.
   Mengapa? Virtual environment mengisolasi dependensi proyek ini dari proyek lain, mencegah konflik versi pustaka.

3. Setelah environment aktif, jalankan perintah:
   `pip install pymongo pandas python-dotenv matplotlib paho-mqtt`
   Mengapa? Inilah pustaka-pustaka yang akan kita butuhkan sepanjang praktikum. `pymongo` untuk akses database, `pandas` dan `matplotlib` untuk analisis, `dotenv` untuk konfigurasi, dan `paho-mqtt` untuk komunikasi MQTT.

4. Buat file bernama `.env` di root folder dan isi dengan:
   ```
   MONGO_URI=mongodb://localhost:27017
   DB_NAME=praktikum6
   ```
   Mengapa? File ini akan memisahkan konfigurasi sensitif dari kode. Dengan memanggil `load_dotenv()` nanti, variabel-variabel ini akan tersedia di seluruh aplikasi tanpa kita hardcode.

5. Buat file `koneksi.py` dan tulis kode berikut:
   ```python
   import os
   from dotenv import load_dotenv
   from pymongo import MongoClient

   load_dotenv()
   client = MongoClient(os.getenv("MONGO_URI"), serverSelectionTimeoutMS=5000)

   try:
       client.admin.command('ping')
       print("Koneksi MongoDB berhasil!")
   except Exception as e:
       print("Gagal terhubung:", e)
   finally:
       client.close()
   ```
   **Mengapa kita menulis kode seperti ini?**
   `load_dotenv()` membaca file `.env`. `os.getenv()` mengambil URI sehingga kode tidak mengandung alamat server secara langsung. Parameter `serverSelectionTimeoutMS` mencegah program menggantung jika server mati. Blok `try-except` menangani kegagalan koneksi dengan elegan, dan `client.close()` memastikan sumber daya dibebaskan. Jalankan skrip ini. Jika muncul "Koneksi MongoDB berhasil!", Anda siap melanjutkan.

### Fase 2: Operasi Dasar CRUD (25 menit)
**Apa yang harus dikerjakan?**
Anda akan menulis skrip `crud_sensor.py` yang melakukan insert banyak data, pencarian dengan filter, update, dan delete, sambil memverifikasi setiap langkah di Compass.

**Langkah-langkah dan kode:**
1. Buat file `crud_sensor.py`. Import library yang dibutuhkan: `os`, `load_dotenv`, `MongoClient`, `datetime`, `timedelta`, dan `random`.
   Mengapa? Kita akan menggunakan datetime untuk timestamp dan random untuk variasi data.

2. Muat `.env`, buat koneksi ke database `praktikum6` dan koleksi `sensor`.
   ```python
   from dotenv import load_dotenv
   from pymongo import MongoClient
   from datetime import datetime, timedelta
   import random, os

   load_dotenv()
   client = MongoClient(os.getenv("MONGO_URI"))
   db = client[os.getenv("DB_NAME")]
   collection = db["sensor"]
   ```
   Mengapa? Pola ini adalah fondasi; setelah satu kali setup, kita bisa menggunakan `collection` di mana saja di file ini.

3. **Insert many**: Bangun list of dictionary berisi 10 data sensor dengan data bervariasi.
   ```python
   data = []
   for i in range(10):
       doc = {
           "mesin": f"CNC-{random.randint(1,3):02d}",
           "suhu": round(random.uniform(60, 100), 2),
           "getaran": round(random.uniform(0.1, 0.5), 2),
           "timestamp": datetime.utcnow() - timedelta(minutes=i*5),
           "status": "normal"
       }
       data.append(doc)
   result = collection.insert_many(data)
   print(f"Tersimpan {len(result.inserted_ids)} dokumen")
   ```
   **Mengapa?** Kita menggunakan `insert_many` untuk efisiensi; 10 data dimasukkan dalam satu perintah. Loop `for i` menciptakan timestamp yang mundur 5 menit setiap langkah agar data tidak semuanya pada waktu yang sama, lebih realistis. `random.randint` dan `random.uniform` memberi variasi mesin dan nilai sensor.

4. **Read**: Query mesin CNC-01, tampilkan hanya suhu dan timestamp, urutkan terbaru, batasi 5.
   ```python
   cursor = collection.find(
       {"mesin": "CNC-01"},
       {"_id": 0, "suhu": 1, "timestamp": 1}
   ).sort("timestamp", -1).limit(5)
   print("Data CNC-01 terbaru:")
   for doc in cursor:
       print(doc)
   ```
   **Mengapa?** Kita membatasi field yang dikembalikan agar output bersih. Sorting descending memastikan kita melihat data terbaru. Limit 5 mencegah kebanjiran data.

5. **Update**: Ubah status menjadi "maintenance" untuk semua mesin yang suhunya di atas 90.
   ```python
   update_result = collection.update_many(
       {"suhu": {"$gt": 90}},
       {"$set": {"status": "maintenance"}}
   )
   print(f"Update: {update_result.modified_count} dokumen diubah")
   ```
   **Mengapa?** Filter `$gt: 90` menangkap suhu kritis. Kita menggunakan `update_many` karena mungkin ada beberapa dokumen yang memenuhi. `$set` mengganti field `status` tanpa menghapus field lain.

6. **Delete**: Hapus semua data yang lebih tua dari 1 jam yang lalu.
   ```python
   satu_jam_lalu = datetime.utcnow() - timedelta(hours=1)
   delete_result = collection.delete_many({"timestamp": {"$lt": satu_jam_lalu}})
   print(f"Delete: {delete_result.deleted_count} dokumen dihapus")
   ```
   **Mengapa?** Simulasi pembersihan data usang. Kita menghitung waktu satu jam lalu dan menggunakan `$lt` (less than) sebagai filter.

7. Jangan lupa menutup koneksi: `client.close()`. Buka MongoDB Compass, refresh koleksi `sensor`, dan verifikasi bahwa data telah berubah sesuai operasi yang dijalankan.

### Fase 3: Agregasi via Python (20 menit)
**Apa yang harus dikerjakan?**
Buat file `agregasi.py` yang menghitung rata-rata suhu per mesin, getaran maksimum, dan jumlah data, lalu mengurutkannya.

**Kode dan penjelasan:**
```python
import os
from dotenv import load_dotenv
from pymongo import MongoClient

load_dotenv()
client = MongoClient(os.getenv("MONGO_URI"))
db = client[os.getenv("DB_NAME")]
collection = db["sensor"]

pipeline = [
    {"$group": {
        "_id": "$mesin",
        "rata_suhu": {"$avg": "$suhu"},
        "max_getaran": {"$max": "$getaran"},
        "count": {"$sum": 1}
    }},
    {"$sort": {"rata_suhu": -1}}
]

results = collection.aggregate(pipeline)
print("Rata-rata suhu per mesin:")
for doc in results:
    print(f"{doc['_id']} -> rata:{doc['rata_suhu']:.2f}°C, max getaran:{doc['max_getaran']}, jumlah data:{doc['count']}")

client.close()
```
**Mengapa kita menulis pipeline seperti ini?** Stage `$group` mengelompokkan dokumen berdasarkan field `mesin` yang dinyatakan sebagai `_id` di dokumen output. `$avg` dan `$max` adalah operator akumulator yang menghitung agregasi numerik. `$sum: 1` menghitung berapa banyak dokumen di setiap kelompok. Stage `$sort` mengurutkan hasil berdasarkan `rata_suhu` secara menurun. Hasil agregasi ini memberikan gambaran performa termal setiap mesin secara ringkas. Dengan memproses di server, kita menghemat transfer data dan memanfaatkan optimasi MongoDB.

### Fase 4: Analisis dengan pandas (30 menit)
**Apa yang harus dikerjakan?**
Buat file `analisis.py` yang mengambil seluruh data dari koleksi `sensor`, memuatnya ke DataFrame, melakukan analisis deskriptif, resampling, plotting, dan ekspor ke CSV.

**Langkah-langkah dan kode:**
1. Import library yang dibutuhkan: `pandas as pd`, `matplotlib.pyplot as plt`, serta modul koneksi yang sama.
2. Ambil data dari MongoDB:
   ```python
   cursor = collection.find({}, {"_id": 0})
   df = pd.DataFrame(list(cursor))
   ```
   **Mengapa?** `list(cursor)` mengonversi seluruh hasil query menjadi list of dicts, yang langsung diterima oleh DataFrame. Menyertakan `{"_id": 0}` pada projection agar kolom `_id` tidak ikut, sehingga DataFrame lebih bersih.

3. Konversi timestamp dan jadikan index:
   ```python
   df['timestamp'] = pd.to_datetime(df['timestamp'])
   df.set_index('timestamp', inplace=True)
   ```
   **Mengapa?** `pd.to_datetime()` mengubah string atau objek datetime Python menjadi tipe `datetime64` pandas yang kaya fitur. Dengan index datetime, kita bisa melakukan operasi time series.

4. Tampilkan statistik deskriptif: `print(df.describe())`.
   **Mengapa?** `describe()` memberikan ringkasan numerik instan: count, mean, std, min, max, dan kuartil. Ini sangat membantu untuk memahami distribusi data sensor.

5. Resampling:
   ```python
   resampled = df.resample('10T').mean()
   print(resampled.head())
   ```
   **Mengapa?** Resample mengelompokkan data ke dalam interval 10 menit (`'10T'`) dan menghitung rata-rata. Ini mengurangi noise dan menunjukkan tren suhu dalam jangka waktu yang lebih halus.

6. Plot grafik:
   ```python
   plt.figure(figsize=(10,5))
   df['suhu'].plot(title='Suhu dari Waktu ke Waktu')
   plt.ylabel('Suhu (°C)')
   plt.grid(True)
   plt.savefig('suhu_plot.png')
   plt.show()
   ```
   **Mengapa?** Grafik garis memberi visualisasi temporal. `savefig` menyimpan grafik ke file sehingga bisa dilampirkan di laporan. Jika berjalan di server tanpa GUI, gunakan `matplotlib.use('Agg')` sebelum import pyplot.

7. Ekspor DataFrame agregasi ke CSV: `resampled.to_csv('agregasi.csv')`.
   **Mengapa?** Format CSV mudah dibuka di Excel atau diolah lebih lanjut. Dengan ini, analisis kita dapat dibagikan kepada pihak lain.

### Fase 5: Data Generator Langsung (15 menit)
**Apa yang harus dikerjakan?**
Buat `data_generator.py` yang berjalan dalam infinite loop, menyisipkan data sensor acak setiap 2 detik.

**Kode:**
```python
import time, random
from datetime import datetime
from pymongo import MongoClient
import os
from dotenv import load_dotenv

load_dotenv()
client = MongoClient(os.getenv("MONGO_URI"))
db = client[os.getenv("DB_NAME")]
collection = db["sensor"]

while True:
    doc = {
        "mesin": f"CNC-{random.randint(1,5):02d}",
        "suhu": round(random.uniform(60, 100), 2),
        "getaran": round(random.uniform(0.1, 0.5), 2),
        "timestamp": datetime.utcnow()
    }
    collection.insert_one(doc)
    print(f"[{datetime.utcnow()}] Inserted: {doc['mesin']} - {doc['suhu']}°C")
    time.sleep(2)
```
**Mengapa?** Skrip ini sangat penting untuk mengisi database dengan data yang terus bertambah, mensimulasikan lingkungan produksi nyata. `time.sleep(2)` mengatur interval pengiriman. Jalankan skrip ini, lalu buka Compass, refresh, dan saksikan dokumen muncul setiap 2 detik. Hentikan dengan `Ctrl+C`.

### Fase 5B (Lanjutan): Simulasi Data Sensor Melalui MQTT (30 menit)
**Apa yang harus dikerjakan?**
Anda akan menjalankan dua skrip di dua terminal terpisah: satu sebagai publisher (sensor), satu sebagai subscriber (penyimpan). Pastikan broker MQTT tersedia (gunakan `test.mosquitto.org` atau broker lokal Mosquitto).

**Terminal 1: Publisher (`mqtt_publisher.py`)**
```python
import json, time, random
import paho.mqtt.client as mqtt
from datetime import datetime

BROKER = "test.mosquitto.org"
PORT = 1883
TOPIC = "pabrik/sensor/suhu"

client = mqtt.Client()
client.connect(BROKER, PORT, 60)

while True:
    data = {
        "mesin": f"CNC-{random.randint(1,5):02d}",
        "suhu": round(random.uniform(60, 100), 2),
        "getaran": round(random.uniform(0.1, 0.5), 2),
        "timestamp": datetime.utcnow().isoformat()
    }
    payload = json.dumps(data)
    client.publish(TOPIC, payload)
    print(f"[PUB] {payload}")
    time.sleep(2)
```
**Mengapa?** Kita mengubah dictionary Python menjadi string JSON karena MQTT mengirimkan pesan dalam bentuk bytes/string. `isoformat()` mengonversi datetime menjadi string yang standar dan mudah di-parse kembali.

**Terminal 2: Subscriber (`mqtt_subscriber.py`)**
```python
import json, os
import paho.mqtt.client as mqtt
from pymongo import MongoClient
from datetime import datetime
from dotenv import load_dotenv

load_dotenv()
MONGO_URI = os.getenv("MONGO_URI")
DB_NAME = os.getenv("DB_NAME")

mongo_client = MongoClient(MONGO_URI)
db = mongo_client[DB_NAME]
collection = db["sensor"]

def on_connect(client, userdata, flags, rc):
    print("Terhubung ke MQTT broker")
    client.subscribe("pabrik/sensor/suhu")

def on_message(client, userdata, msg):
    try:
        payload = json.loads(msg.payload.decode())
        payload['timestamp'] = datetime.fromisoformat(payload['timestamp'])
        result = collection.insert_one(payload)
        print(f"[MONGO] Tersimpan {result.inserted_id} - {payload['mesin']} {payload['suhu']}°C")
    except Exception as e:
        print("Error:", e)

mqtt_client = mqtt.Client()
mqtt_client.on_connect = on_connect
mqtt_client.on_message = on_message
mqtt_client.connect("test.mosquitto.org", 1883, 60)
mqtt_client.loop_forever()
```
**Mengapa callback digunakan?** Arsitektur MQTT bersifat event-driven. Kita tidak bisa secara manual memeriksa apakah ada pesan; kita harus mendaftarkan fungsi yang akan dipanggil secara otomatis ketika event terjadi. `on_connect` men-subscribe topik; `on_message` memproses payload. `loop_forever()` menjaga koneksi dan memonitor pesan secara asynchronous.

Jalankan kedua skrip. Di terminal publisher, Anda akan melihat pesan yang dikirim; di terminal subscriber, data disimpan ke MongoDB. Verifikasi di Compass.

### Fase 6: Error Handling dan Logging (10 menit)
**Apa yang harus dikerjakan?**
Tambahkan blok `try-except` pada operasi-operasi kritis di setiap skrip, dan konfigurasikan logging ke file dan konsol.

**Contoh penambahan logging pada `crud_sensor.py` (di awal file):**
```python
import logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler("crud_sensor.log"),
        logging.StreamHandler()
    ]
)
```
Kemudian bungkus `insert_many` dengan try-except:
```python
try:
    result = collection.insert_many(data)
    logging.info(f"Insert berhasil, {len(result.inserted_ids)} dokumen")
except Exception as e:
    logging.error(f"Insert gagal: {e}")
```
**Mengapa?** Dengan logging, Anda memiliki jejak kronologis kejadian. Ketika aplikasi berjalan di latar belakang, Anda bisa memeriksa file log untuk mendiagnosis masalah alih-alih hanya mengandalkan print di konsol.

---

## LATIHAN MANDIRI
Pada bagian ini, mahasiswa diberikan instruksi tugas yang harus diselesaikan secara mandiri tanpa panduan langkah demi langkah. Mahasiswa diharapkan mampu merancang sendiri solusi berdasarkan materi yang telah dipelajari.

### Latihan 1: Impor Data Maintenance dari CSV
**Instruksi:**
- Anda harus membuat sebuah database baru bernama `latihan6` dan di dalamnya terdapat koleksi `maintenance`.
- Siapkan sebuah file CSV bernama `maintenance.csv` dengan kolom: `mesin`, `tanggal`, `biaya`, `teknisi`. Isi file tersebut dengan beberapa baris data fiktif, di mana `tanggal` menggunakan format `YYYY-MM-DD`.
- Tulis skrip Python bernama `import_maintenance.py` yang melakukan hal berikut:
   - Membaca file CSV menggunakan `pandas.read_csv()`.
   - Mengonversi kolom `tanggal` menjadi tipe datetime.
   - Menyisipkan seluruh data dari DataFrame ke dalam koleksi `maintenance` menggunakan satu perintah `insert_many`. Untuk itu, Anda harus mengubah DataFrame menjadi list of dictionary terlebih dahulu.
- Tulis skrip kedua bernama `query_maintenance.py` yang melakukan:
   - Mencari semua dokumen dengan biaya lebih dari 1.000.000, menampilkannya dalam bentuk DataFrame.
   - Melakukan update: pada dokumen yang memiliki `mesin = "CNC-01"` dan `biaya = 1200000`, ganti nilai `teknisi` menjadi "Dewi".
   - Menghitung total biaya per bulan menggunakan aggregation pipeline. Karena `tanggal` bertipe date, Anda perlu mengekstrak bulan dan tahun, misalnya dengan operator `$dateToString` atau `$substr`. Tampilkan hasilnya dalam bentuk tabel di konsol atau DataFrame.

**Mengapa latihan ini diberikan?** Mahasiswa harus membuktikan bahwa mereka mampu melakukan impor data dari sumber eksternal, memanipulasi dokumen di MongoDB, dan menyusun pipeline agregasi yang sedikit lebih kompleks. Tidak adanya step-by-step memaksa mahasiswa untuk mengingat kembali dan merangkai sendiri potongan-potongan pengetahuan yang sudah diberikan.

### Latihan 2: MQTT Data Produksi (Opsional)
**Instruksi:**
- Pastikan MQTT broker dapat diakses (lokal atau publik).
- Buat dua skrip:
   - `mqtt_pub_latihan.py`: skrip publisher yang mengirimkan data produksi (fields: `batch`, `mesin`, `jumlah`, `reject`) ke topik `pabrik/produksi` setiap 3 detik. Data `batch` dan `mesin` diacak, `jumlah` antara 100-500, `reject` antara 0-50.
   - `mqtt_sub_latihan.py`: skrip subscriber yang menerima pesan dari topik tersebut, menambahkan field `timestamp` saat pesan diterima, lalu menyimpan ke MongoDB di koleksi `produksi_mqtt` (database bebas). Di subscriber, hitung reject rate (`reject/jumlah * 100`); jika hasilnya lebih dari 5%, cetak peringatan dan simpan field `peringatan: true` ke dokumen.
- Jalankan kedua skrip, verifikasi data di Compass, dan dokumentasikan dengan screenshot.

**Mengapa latihan ini diberikan?** Mahasiswa berlatih mengintegrasikan MQTT dengan bisnis logic sederhana, merefleksikan kebutuhan nyata di mana data yang masuk perlu divalidasi atau diberi label sebelum disimpan.

---

## STUDI KASUS: DASHBOARD MONITORING SUHU MESIN
Studi kasus ini adalah tugas terstruktur yang harus dikerjakan secara individu. Mahasiswa berperan sebagai engineer yang diminta membuat dashboard monitoring berbasis konsol untuk manajer pabrik.

**Instruksi:**
- Gunakan database `studi_kasus` dan koleksi `suhu_mesin`.
- Tulis sebuah skrip Python bernama `monitor.py` yang melakukan:
   1. Mengambil data dari koleksi `suhu_mesin` yang timestamp-nya berada dalam 1 jam terakhir. (Filter: `timestamp` di atas `datetime.utcnow() - timedelta(hours=1)`).
   2. Menghitung rata-rata suhu per mesin menggunakan aggregation pipeline.
   3. Mendeteksi mesin yang pernah mencatatkan suhu di atas 90 derajat dalam periode tersebut.
   4. Menampilkan di konsol:
      - Tabel yang menunjukkan suhu terkini untuk setiap mesin (ambil satu dokumen terbaru per mesin, bisa dengan `find_one` disort descending untuk setiap mesin, atau gunakan trik agregasi).
      - Daftar mesin yang memicu alarm beserta suhu maksimumnya.
      - Tabel rata-rata suhu per mesin.
   5. Setiap kali alarm terdeteksi, catat waktu kejadian, nama mesin, dan suhunya ke file `alarm.log` dengan format yang rapi.
   6. Menangani error koneksi database dengan baik; jika MongoDB tidak bisa dihubungi, tampilkan pesan bahwa sistem sedang offline, bukan traceback.
   7. Sertakan dokumentasi cara menjalankan di `README.txt`.

**Mengapa studi kasus ini?** Mahasiswa dituntut untuk mengintegrasikan hampir semua keterampilan yang dipelajari: koneksi, query dengan filter waktu, agregasi, penelusuran data per kelompok, penanganan error, dan logging. Kasus ini sangat realistis dan sering dijumpai di industri.

---

## TUGAS TERSTRUKTUR (TUGAS MANDIRI)
Tugas ini dikerjakan secara individu dan memiliki bobot penilaian yang signifikan. Mahasiswa akan membangun aplikasi input data produksi dan laporan.

**Instruksi:**
1. Buat database dengan nama `tugas6_<NIM>` dan koleksi `produksi`. Dokumen di koleksi ini memiliki field: `batch` (string), `mesin` (string), `jumlah` (int), `reject` (int), `tanggal` (datetime).
2. Kembangkan sebuah skrip Python bernama `app_produksi.py` yang berjalan dalam mode interaktif berbasis menu teks. Menu yang harus tersedia:
   - **1. Input data produksi baru**: pengguna memasukkan batch, mesin, jumlah, reject, dan tanggal melalui keyboard. Data disimpan ke MongoDB.
   - **2. Tampilkan data produksi per mesin**: pengguna memasukkan nama mesin, lalu sistem menampilkan semua data produksi untuk mesin tersebut dalam bentuk tabel (bisa print DataFrame atau format manual).
   - **3. Hitung reject rate**: sistem menjalankan agregasi untuk menghitung reject rate (`reject/jumlah*100`) per batch, lalu hanya menampilkan batch yang reject rate-nya di atas 5%.
   - **4. Ekspor laporan bulanan**: pengguna memasukkan bulan dan tahun (format MM-YYYY). Sistem menghitung total jumlah dan total reject per mesin untuk bulan tersebut menggunakan agregasi, lalu menyimpan hasilnya ke file CSV dengan nama `laporan_MM-YYYY.csv`.
   - **5. Keluar**.
3. Gunakan environment variable untuk connection string (`.env`).
4. Implementasikan logging ke file `app.log`; setiap operasi besar (insert, query, error) wajib tercatat dengan timestamp.
5. Struktur kode harus modular, dengan fungsi-fungsi terpisah untuk setiap pilihan menu.
6. Sertakan dokumentasi singkat cara menjalankan program di file `README.md` atau `README.txt`.

**Mengapa tugas ini diberikan?** Tugas ini menilai kemampuan mahasiswa dalam merancang aplikasi CLI lengkap, mengimplementasikan CRUD, agregasi, integrasi pandas, penanganan error, logging, dan dokumentasi. Semua aspek inilah yang akan dibutuhkan di dunia kerja sebagai developer Python yang berhubungan dengan data.

---

## TUGAS KELOMPOK (3-4 ORANG)
Tugas kelompok akan membawa mahasiswa ke dalam proyek kolaboratif yang menyerupai sistem pemantauan kualitas udara pabrik dengan arsitektur IoT berbasis MQTT. Mahasiswa dibagi dalam kelompok yang terdiri dari 3 hingga 4 orang dengan pembagian peran sebagai berikut.

### Pembagian Peran dan Tugas Setiap Anggota
- **Anggota 1: MQTT Publisher Simulasi**
  Membuat skrip `pub_udara.py` yang mensimulasikan 10 sensor udara. Setiap sensor diidentifikasi dengan `sensor_id` dan `lokasi` (contoh: "Sensor-A", "Lantai 1"). Skrip ini mengirim data ke topik `pabrik/udara` setiap 10 detik. Format data yang dikirim: `sensor_id`, `lokasi`, `pm25`, `pm10`, `co2`, `suhu`, `kelembaban`, `timestamp` (dalam string ISO). Variasi data mengikuti rentang realistis, namun dengan probabilitas 5%, nilai `pm25` dibuat di atas 150 µg/m³ untuk menyimulasikan kondisi berbahaya.

- **Anggota 2: MQTT Subscriber dan Penyimpanan**
  Membuat skrip `sub_udara.py` yang menerima pesan dari topik `pabrik/udara`, melakukan parsing, dan menyimpan ke MongoDB di database `kelompok6` koleksi `kualitas_udara`. Subscriber juga harus mendeteksi anomali: jika `pm25 > 150`, ia harus langsung menyisipkan dokumen peringatan ke koleksi `alert` dengan field `sensor_id`, `lokasi`, `pm25`, `timestamp`, dan `pesan: "PM2.5 berbahaya!"`. Selain itu, cetak notifikasi mencolok di konsol. Pastikan subscriber memiliki error handling yang baik.

- **Anggota 3: Analisis dan Dashboard**
  Membuat skrip `analisis_dashboard.py` yang tidak terkait langsung dengan aliran real-time, tetapi bekerja secara on-demand. Skrip ini mengambil data 24 jam terakhir dari koleksi `kualitas_udara`, menghitung rata-rata `pm25` per jam untuk setiap `lokasi` menggunakan agregasi, lalu memplot grafik perbandingan antar lokasi dengan matplotlib. Grafik disimpan sebagai `dashboard_udara.png`. Skrip ini juga mengekspor data 24 jam terakhir ke file CSV `data_24jam.csv`. Sertakan komentar atau docstring yang jelas.

- **Anggota 4: Status Terkini dan Dokumentasi**
   Membuat skrip `status_udara.py` yang dapat dijalankan kapan saja untuk menampilkan status terkini setiap sensor. Ada dua pendekatan yang bisa dipilih:
   - Pendekatan 1: Skrip berlangganan ke topik MQTT dan setiap kali ada pesan masuk, ia memperbarui tampilan status di konsol menjadi "Normal" (pm25 ≤ 50), "Waspada" (50 < pm25 ≤ 150), atau "Bahaya" (pm25 > 150) untuk sensor yang bersangkutan.
   - Pendekatan 2: Skrip melakukan query ke MongoDB untuk setiap sensor, mencari dokumen terbaru berdasarkan `sensor_id`, lalu menampilkan statusnya.
   Anggota 4 juga bertanggung jawab mengkoordinasikan dokumentasi kelompok: membuat file `README.md` yang menjelaskan cara menjalankan seluruh sistem (dengan urutan yang jelas), menyusun laporan kelompok dalam format PDF yang berisi flowchart sistem, pembagian kerja, tangkapan layar output, dan penjelasan singkat setiap komponen kode. Presentasi singkat (10 menit) juga disiapkan untuk pertemuan ke-7.

### Studi Kasus yang Jelas
**Latar Belakang:** Sebuah pabrik kimia memiliki 10 sensor kualitas udara yang tersebar di berbagai lokasi. Sensor mengirim data melalui MQTT ke pusat monitoring. Manajemen ingin sistem yang mampu mendeteksi polusi udara berbahaya secara real-time dan menyediakan laporan historis untuk evaluasi lingkungan kerja. Kelompok Anda bertugas membangun sistem tersebut.

### Rubrik Penilaian Kelompok
- **Fungsionalitas (30%)**: publisher, subscriber, analisis, dan status berjalan sesuai spesifikasi.
- **Penerapan PyMongo & pandas (25%)**: penggunaan MongoDB dan pandas tepat guna dan efisien.
- **Kualitas kode & error handling (20%)**: kode rapi, modular, terdokumentasi, dan tahan terhadap kesalahan.
- **Visualisasi & output (15%)**: grafik bermakna, file CSV terbaca, notifikasi jelas.
- **Dokumentasi & presentasi (10%)**: laporan lengkap, README informatif, presentasi terstruktur.

---

## REFERENSI
1. PyMongo Documentation. https://pymongo.readthedocs.io/
2. MongoDB. *MongoDB Python Developer Path*. https://learn.mongodb.com/
3. pandas documentation. https://pandas.pydata.org/docs/
4. Matplotlib documentation. https://matplotlib.org/stable/contents.html
5. Eclipse Paho MQTT Python. https://www.eclipse.org/paho/clients/python/
6. HiveMQ. *MQTT Essentials*. https://www.hivemq.com/mqtt-essentials/
7. Mosquitto Broker. https://mosquitto.org/

---

**Pesan Dosen:**
Perjalanan Anda menguasai integrasi Python dan MongoDB baru saja mencapai puncak dalam pertemuan ini. Bukan hanya teori, Anda sekarang telah menyentuh sisi paling praktis dari rekayasa data industri: dari sepotong kode yang mengirim suhu mesin, melintasi broker MQTT, mendarat di database NoSQL, lalu bertransformasi menjadi grafik yang siap disajikan ke manajemen. Tetaplah berkarya, jangan takut melakukan kesalahan, karena di setiap error yang berhasil Anda atasi, di sanalah pengetahuan sejati tumbuh. Selamat membangun solusi cerdas, selamat menjadi bagian dari revolusi industri digital.
