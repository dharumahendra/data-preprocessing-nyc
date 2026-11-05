# Preprocessing Data: Lubang Jalan NYC × Cuaca

Tugas mata kuliah **Rekayasa Data** (Semester 5): pipeline preprocessing lengkap
(cleaning, integration, reduction) pada data perbaikan lubang jalan Kota New York
yang digabungkan dengan data cuaca harian.

**Pertanyaan analisis:** bagaimana kondisi cuaca (hujan, salju, siklus beku-cair)
berkaitan dengan jumlah laporan lubang jalan dan kecepatan perbaikannya di tiap
borough NYC?

---

## Dataset

| Dataset | File | Level data | Sumber | Lisensi |
|---|---|---|---|---|
| Street Pothole Work Orders – Closed | `street_pothole_work_orders.csv.gz` | 1 baris = 1 work order (410.488 baris, 15 kolom) | NYC Department of Transportation via [NYC Open Data](https://data.cityofnewyork.us/Transportation/Street-Pothole-Work-Orders-Closed-Dataset-/x9wy-ing4) ([metadata resmi](https://www.nyc.gov/html/dot/downloads/pdf/street-potholes-metadata.pdf)) | NYC Open Data Terms of Use |
| Cuaca harian NYC 2018–2024 | `weather_nyc.csv` | 1 baris = 1 hari | [Open-Meteo Historical Weather API](https://open-meteo.com/en/docs/historical-weather-api) (reanalisis ERA5) | CC-BY-4.0, [doi:10.5281/zenodo.7970649](https://doi.org/10.5281/zenodo.7970649) |
| Referensi verifikasi cuaca | `reference/noaa_centralpark_2018_2024.csv` | 1 baris = 1 hari | [NOAA NCEI GHCN-Daily](https://www.ncei.noaa.gov/access/services/data/v1), stasiun Central Park (`USW00094728`) | Public domain |

Data pothole disimpan dalam format `.csv.gz` agar muat di bawah batas 100 MB GitHub.
Pandas membacanya langsung tanpa perlu diekstrak.

---

## Alur Pipeline

Notebook dijalankan **berurutan**, karena tiap notebook memakai output notebook sebelumnya.

```
street_pothole_work_orders.csv.gz ─┐
weather_nyc.csv ───────────────────┼─► data_cleaning.ipynb ─► data_integration.ipynb ─► data_reduction.ipynb
reference/noaa_centralpark_*.csv ──┘        │                        │                         │
                                     pothole_clean.parquet    integrated_daily.parquet    pca_result, data mingguan,
                                     weather_clean.csv        (hari × borough)            sampel stratified
```

| Notebook | Isi |
|---|---|
| [`data_cleaning.ipynb`](data_cleaning.ipynb) | Profiling dan issue register (21 masalah kualitas data), perbaikan tipe data, imputasi berbasis redundansi (`OFT` → nama jalan), missing value, duplikat, feature engineering, deteksi outlier (rule-based, IQR, Z-score, MAD, Log-IQR, Isolation Forest), dan verifikasi cuaca ke stasiun NOAA |
| [`data_integration.ipynb`](data_integration.ipynb) | Entity identification, penyamaan cakupan waktu (2018–2024), agregasi work order → hari × borough, merge dengan cuaca, analisis redundansi (Pearson, Spearman, kovarians, VIF, chi-square / Cramér's V) |
| [`data_reduction.ipynb`](data_reduction.ipynb) | Low-variance filter, winsorizing dan transformasi log, standardisasi, PCA (elbow, Kaiser, kumulatif 90%), interpretasi loadings, biplot, rekonstruksi, serta numerosity reduction (agregasi mingguan dan stratified sampling) |

### Ringkasan perubahan data

| Tahap | Unit baris | Baris | Kolom / fitur |
|---|---|---:|---:|
| Raw | work order (2010–2026) | 410.488 | 15 |
| Cleaning | work order (2010–2026) | 400.961 | 27 |
| Integration | hari × borough (2018–2024) | 12.650 | 28 |
| Buang atribut redundan | hari × borough | 12.650 | 21 |
| Low-variance filter | hari × borough | 12.650 | 20 |
| PCA (≥ 90% varians) | hari × borough | 12.650 | 14 |
| Numerosity (mingguan) | minggu × borough | 1.815 | 21 |

---

## Temuan Utama

**Cleaning**
- Data mentah tidak siap pakai: 65,5% `SpecLoc` dan 39,3% `HouseNum` kosong, 8.853 laporan ganda,
  574 perbaikan "selesai sebelum dilaporkan", dan segmen "BODY OF WATER" sepanjang 140.473 ft.
  Setelah cleaning, **97,7% data dipertahankan**.
- Redundansi internal dimanfaatkan untuk validasi: `Boro` 100% konsisten dengan prefix `DefNum`,
  dan kode jalan di `OFT` memetakan 1:1 ke nama jalan sehingga bisa dipakai untuk imputasi.
- Metode outlier harus sesuai bentuk distribusi. Pada durasi perbaikan (skewness 182), IQR biasa
  menandai 10,9% data dan MAD 19,8%, sedangkan **Log-IQR hanya 1,3%**.
- Outlier tidak selalu error. Hujan 187,2 mm pada 2022-10-04 terbukti salah setelah dicek ke
  stasiun NOAA (47 mm), sedangkan Badai Ida (111,5 mm) adalah kejadian nyata.

**Integration**
- Dari 28 fitur, 7 redundan (|r| > 0,85) dan dibuang. VIF menemukan kolinearitas yang tidak terlihat
  dari korelasi berpasangan (presipitasi = hujan + salju). Setelah pembuangan, semua VIF < 10.
- Sumber pelapor dan instansi penangan sangat terkait (Cramér's V = 0,74).

**Reduction**
- PCA mempertahankan **91,4% varians dengan 14 dari 20 dimensi**. Reduksinya moderat karena fitur
  redundan sudah dibuang di tahap integrasi.
- Komponen utama dapat diinterpretasi: **PC1 = musim dingin dan beku-cair**,
  **PC2 = cara pencatatan lokasi**, **PC3 = hujan**.

**Insight**
- Laporan lubang jalan sangat musiman: rata-rata tertinggi di musim semi (±15 laporan/borough/hari)
  dibanding musim gugur (±7).
- Korelasi laporan mingguan dengan jumlah hari beku-cair 3 minggu sebelumnya sekitar 0,57, tetapi
  sebagian besar merupakan efek musim. Di dalam musim dingin saja korelasinya ±0,3, dan hujan saja
  hampir tidak berpengaruh.

**Keterbatasan:** hasil berupa asosiasi, bukan sebab-akibat; cuaca berasal dari satu titik untuk
seluruh NYC; data cuaca adalah data model (hujan badai lokal cenderung di bawah nilai sebenarnya);
arti sebagian kode `InitBy`/`Source` tidak terdokumentasi resmi.

---

## Cara Menjalankan

Membutuhkan **Python 3.12**.

```bash
git clone https://github.com/dharumahendra/data-preprocessing-nyc.git
cd data-preprocessing-nyc

python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate

pip install -r requirements.txt
jupyter notebook
```

Lalu jalankan notebook secara berurutan: `data_cleaning` → `data_integration` → `data_reduction`.
Folder `output/` (data hasil) dan `figures/` (grafik) akan dibuat otomatis oleh notebook.

> **Catatan:** jangan membuka lalu menyimpan ulang CSV hasil di Excel. Excel hanya menyimpan
> 15 digit sehingga kolom `OFT` (18 digit) akan rusak. Tutup juga file CSV yang sedang dibuka
> di Excel sebelum menjalankan ulang notebook, karena file yang terkunci membuat proses simpan gagal.

---

## Struktur Repository

```
.
├── data_cleaning.ipynb
├── data_integration.ipynb
├── data_reduction.ipynb
├── street_pothole_work_orders.csv.gz   # data mentah NYC DOT (terkompresi)
├── weather_nyc.csv                     # data mentah cuaca Open-Meteo
├── reference/
│   └── noaa_centralpark_2018_2024.csv  # rujukan verifikasi presipitasi
├── requirements.txt
└── README.md
```

## Library

pandas, numpy, scipy, scikit-learn, statsmodels, matplotlib, seaborn, missingno, pyarrow
(versi lengkap di [`requirements.txt`](requirements.txt)).
