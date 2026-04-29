# Proyek-BDA-A-Kelompok-2
## Infrastruktur
Proyek ini berjalan di atas Docker dengan layanan:
- **PostgreSQL** ---> Sebagai sumber data transaksional.
- **MinIO** ---> Sebagai Object Storage (Data Lake).
- **MinIO Client (MC)** ---> Untuk konfigurasi otomatis bucket.
- **Apache Spark (PySpark)** ---> Untuk transformasi Bronze → Silver.
## Penyiapan Infrastruktur - Ingestion (Bronze)
1. Clone repositori
  ```
  git clone https://github.com/CantikaZahnaBrilliantoPutri/Proyek-BDA-A-Kelompok-2.git
  ```
2. Persiapan Infrastruktur
  Pastikan Docker Desktop sudah berjalan, kemudian buka terminal di folder proyek dan jalankan:
  ```
  docker compose up -d
  ```
3. Buat file .env dengan menjalankan
  ```
  cp .env.example .env
  ```
4. Buat virtual environment
  ```
  python -m venv venv
  venv\Scripts\activate
  ```
5. Instalansi Library Python
  ```
  pip install -r requirements.txt
  ```
6. Sebelum melakukan ingestion, pastikan database postgres sudah terisi data (`count` tidak 0)
  ```
  docker exec -it postgres-kelompok2 psql -U postgres -d postgres -c "SELECT COUNT(*) FROM stock_move;"
  ```
7. Jalankan script python untuk memindahkan data ke Data Lake, yaitu
  ```
  python scripts/ingest_to_datalake.py
  ```
8. Data Lake MinIO
  Setelah script dijalankan, data akan tersimpan di bucket `datalake-kelompok2` dengan struktur berikut:
  ```
  raw/
  ├── stock_transactions.csv (dari Postgres)
  ├── grocery-inventory.csv (dari Local CSV)
  └── suppliers_info.json (dari Local JSON)
  ```
## Data Cleaning & Pre-Processing (Silver)
### Deskripsi Umum Silver Layer
Silver layer bertujuan untuk mengubah data raw (bronze) menjadi data yang lebih bersih, konsisten, dan siap dianalisis/diolah lanjut. Proses ini dijalankan menggunakan `PySpark`.

#### Input (Bronze)
Data dibaca dari MinIO bucket `datalake-kelompok2` pada folder `raw/`:
```
raw/stock_transactions.csv
raw/grocery-inventory.csv
raw/suppliers_info.json
```

#### Transformasi yang dilakukan
1. **Standarisasi/Normalisasi nama kolom** agar konsisten dan mudah digunakan untuk analisis
    - Mengubah nama kolom menjadi huruf kecil (lowercase)
    - Menghapus spasi di awal dan akhir
    - Mengganti karakter pemisah seperti spasi / (-) menjadi underscore (_)

2. **Trimming kolom bertipe string**
    - Semua kolom string di-trim untuk menghilangkan whitespace yang tidak perlu

3. **Parsing kolom tanggal/waktu**
    - Kolom yang namanya mengandung kata date, `time`, atau `created` akan diubah menjadi tipe timestamp

4. **Penanganan nilai `null`**
    - Pada dataset inventory, kolom harga akan dicast menjadi numeric, nilai `null` akan diisi menggunakan median (pendekatan *percentile_approx*), lalu dilakukan *Deduplication* (menghapus data duplikat)
    - Pada dataset transaksi, kolom `quantity` akan diubah tipe datanya menjadi double. Jika menghasilkan nilai `null`, maka nilainya akan di-set menjadi `0.0`

5. **Data yang duplikat dihapus**
    - jika ada kolom `id`, deduplikasi berdasarkan `id`
    - jika tidak ada, deduplikasi berdasarkan seluruh baris
    
6. **Pembuatan ID transaksi jika tidak tersedia**
    - Pada dataset transaksi, jika kolom `transaction_id` tidak ada, maka dibuat otomatis menggunakan `uuid()`

#### Output (Silver)
Hasil disimpan kembali ke MinIO dalam format Parquet pada folder `silver/`
```
  silver/
  ├── stock_transactions/
  │   ├── _SUCCESS
  │   └── part-00000-***.snappy.parquet
  └── grocery_inventory/
  │   ├── _SUCCESS
  │   └── part-00000-***.snappy.parquet
  └── suppliers/
      ├── _SUCCESS
      └── part-00000-***.snappy.parquet
  ```
> Karena output ditulis oleh Spark, masing-masing folder berisi beberapa file part-*.parquet dan marker _SUCCESS

### Cara Menjalankan Proyek
1. Jalankan kode berikut untuk memastikan semua service sudah siap dan semua requirement sudah terinstall:
  ```
  docker compose up -d
  venv\Scripts\activate
  pip install -r requirements.txt
  ```
2. Pastikan data sudah di-ingest ke bucket `datalake-kelompok2` di MiniO. Buka [localhost:9000](http://localhost:9001/), pastikan sudah ada folder `raw` di dalam bucket. Jika belum, jalankan ingestion terlebih dahulu
3. Masuk ke dalam container `spark-processor` dan buka shell bash dengan menjalankan kode:
  ```
  docker exec -it spark-processor bash
  ```
4. Setelah masuk ke dalam container (tampilan CLI menjadi `root@<container_id>:/app#`), jalankan kode berikut untuk memulai data cleaning dan pre-processing:
  ```
  spark-submit /app/scripts/silver_pyspark.py
  ```
5. Setelah muncul baris `s3a-file-system metrics system shutdown complete`, proses telah selesai. Buka/refresh MiniO ([localhost:9000](http://localhost:9001/)), hasil processing tahap silver dapat dilihat di folder `silver`.

```python?code_reference&code_event_index=1
# Definisi konten README dalam format Markdown
readme_content = """# Proyek-BDA-A-Kelompok-2

## Infrastruktur
Proyek ini berjalan di atas Docker dengan layanan:
- **PostgreSQL** ---> Sebagai sumber data transaksional.
- **MinIO** ---> Sebagai Object Storage (Data Lake).
- **MinIO Client (MC)** ---> Untuk konfigurasi otomatis bucket.
- **Apache Spark (PySpark)** ---> Untuk transformasi Bronze → Silver → Gold.

## Penyiapan Infrastruktur - Ingestion (Bronze)
1. **Clone repositori**
   ```bash
   git clone [https://github.com/CantikaZahnaBrilliantoPutri/Proyek-BDA-A-Kelompok-2.git](https://github.com/CantikaZahnaBrilliantoPutri/Proyek-BDA-A-Kelompok-2.git)
   ```
2. **Persiapan Infrastruktur**
   Pastikan Docker Desktop sudah berjalan, kemudian buka terminal di folder proyek dan jalankan:
   ```bash
   docker compose up -d
   ```
3. **Buat file .env** dengan menjalankan:
   ```bash
   cp .env.example .env
   ```
4. **Buat virtual environment**
   ```bash
   python -m venv venv
   # Windows
   venv\\Scripts\\activate
   # Linux/Mac
   source venv/bin/activate
   ```
5. **Instalansi Library Python**
   ```bash
   pip install -r requirements.txt
   ```
6. **Cek Database**
   Sebelum melakukan ingestion, pastikan database postgres sudah terisi data:
   ```bash
   docker exec -it postgres-kelompok2 psql -U postgres -d postgres -c "SELECT COUNT(*) FROM stock_move;"
   ```
7. **Jalankan Ingestion**
   Pindahkan data ke Data Lake (Bronze Layer):
   ```bash
   python scripts/ingest_to_datalake.py
   ```

## Data Cleaning & Pre-Processing (Silver)
### Deskripsi Umum Silver Layer
Silver layer bertujuan untuk mengubah data raw (bronze) menjadi data yang lebih bersih dan konsisten menggunakan **PySpark**.

#### Transformasi Utama:
- Standarisasi nama kolom (lowercase, trim, replacement space ke underscore).
- Parsing tipe data (Date/Timestamp, Numeric casting).
- Penanganan nilai `null` menggunakan median (untuk harga) atau default value (0.0 untuk quantity).
- Deduplikasi data berdasarkan ID atau row-level.

#### Cara Menjalankan Silver:
1. Masuk ke container Spark:
   ```bash
   docker exec -it spark-processor bash
   ```
2. Jalankan spark-submit:
   ```bash
   spark-submit /app/scripts/silver_pyspark.py
   ```

## Data Aggregation & Business Insights (Gold)
### Deskripsi Umum Gold Layer
Gold layer adalah tahap akhir di mana data dari Silver layer digabungkan dan diagregasi untuk menghasilkan tabel yang siap untuk keperluan Business Intelligence dan Analytics.

#### Transformasi yang Dilakukan:
1. **Join Multi-Dataset**: Menggabungkan data transaksi stok dengan info produk dan supplier.
2. **Aggregated Stock Performance**: Menghitung total stok masuk, stok keluar, dan saldo akhir per produk.
3. **Supplier Efficiency Metrics**: Menganalisis jumlah item yang disuplai oleh masing-masing vendor.
4. **Inventory Status Categorization**: Penentuan label stok seperti 'Low Stock', 'Healthy', atau 'Overstock'.

#### Output (Gold):
Hasil disimpan dalam format **Parquet** di:
- `gold/inventory_performance_report/`
- `gold/supplier_efficiency_metrics/`

#### Cara Menjalankan Gold:
1. Pastikan masih berada di dalam container `spark-processor`.
2. Jalankan spark-submit untuk tahap Gold:
   ```bash
   spark-submit /app/scripts/gold_pyspark.py
   ```

## Business Recommendations
Berdasarkan data yang diproses di Gold Layer, sistem ini memberikan wawasan untuk:
- **Restocking Strategy**: Identifikasi produk yang masuk kategori 'Low Stock' untuk segera dipesan ulang.
- **Supplier Performance Review**: Mengevaluasi supplier mana yang paling aktif berkontribusi pada ketersediaan inventaris.
- **Loss Prevention**: Memantau anomali pada transaksi keluar yang tidak wajar.

#### Cara Menghasilkan Laporan Rekomendasi:
Keluar dari container spark (`exit`), lalu jalankan script report di lingkungan lokal:
```bash
python scripts/generate_business_report.py
```

## Ringkasan Alur Eksekusi
1. `docker compose up -d` (Setup Infra)
2. `python scripts/ingest_to_datalake.py` (Ingestion - Bronze)
3. `docker exec -it spark-processor spark-submit /app/scripts/silver_pyspark.py` (Cleaning - Silver)
4. `docker exec -it spark-processor spark-submit /app/scripts/gold_pyspark.py` (Aggregation - Gold)
5. `python scripts/generate_business_report.py` (Business Insight)
"""

with open("README.md", "w") as f:
    f.write(readme_content)

## Data Aggregation & Business Insights (Gold)
### Deskripsi Umum Gold Layer
Gold layer adalah tahap akhir pemrosesan data di mana data yang sudah bersih di Silver Layer diagregasi untuk membentuk tabel yang siap dianalisis.

#### Transformasi Utama:
1. **Join Multi-Dataset**: Menggabungkan data transaksi stok, inventaris, dan informasi supplier.
2. **Agregasi Performa**: Menghitung total stok masuk/keluar dan saldo akhir per produk.
3. **Metrik Supplier**: Menghitung jumlah item dan kontribusi tiap supplier.
4. **Business Logic**: Klasifikasi status stok (e.g., *Low Stock*, *Healthy*, *Overstock*).

#### Cara Menjalankan Gold:
1. Di dalam container `spark-processor`, jalankan:
   ```bash
   spark-submit /app/scripts/gold_pyspark.py
   ```
2. Hasil akan tersimpan di MinIO pada folder `gold/` dalam format Parquet.

## Business Recommendations
Hasil dari Gold Layer digunakan untuk memberikan rekomendasi strategis:
- **Restocking**: Memberikan daftar produk yang harus segera dipesan ulang.
- **Supplier Review**: Menentukan supplier mana yang memiliki ketersediaan barang paling stabil.
- **Inventory Turnaround**: Memberikan insight mengenai produk yang paling cepat terjual.

#### Cara Menghasilkan Laporan Rekomendasi:
Keluar dari container (`exit`) dan jalankan di terminal lokal:
```bash
python scripts/generate_business_report.py
```

## Ringkasan Alur Eksekusi Keseluruhan
1. `docker compose up -d` (Setup Infra)
2. `python scripts/ingest_to_datalake.py` (Bronze)
3. `docker exec -it spark-processor spark-submit /app/scripts/silver_pyspark.py` (Silver)
4. `docker exec -it spark-processor spark-submit /app/scripts/gold_pyspark.py` (Gold)
5. `python scripts/generate_business_report.py` (Insights)
```

