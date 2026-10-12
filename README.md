<h1 align="center">AGBD Estimation with STAC API</h1>

<p align="center">
  <img src="assets/dataset_distribution.png" alt="Deskripsi gambar" width="800">
</p>

Estimasi *Aboveground Biomass Density* (AGBD, Mg/ha) berbasis citra satelit. Data diakses langsung melalui STAC API, diolah menjadi fitur spasial, lalu diprediksi dengan model Random Forest terlatih untuk menghasilkan peta AGBD beresolusi 30 m.

## Gambaran Proyek

Proyek ini terdiri dari tiga alur kerja utama:

1. **Pembuatan Dataset** (`create_dataset.ipynb`):
   - Mengambil citra satelit (Sentinel-2, Sentinel-1, GLO-30 DEM, Dynamic World) via STAC API dan footprint observasi GEDI L4A via NASA Earthdata.
   - Melakukan reproyeksi ke grid 30 m (UTM), komposit temporal median, ekstraksi indeks vegetasi, tekstur GLCM, dan probabilitas tutupan lahan.
   - Melakukan sampling nilai fitur raster pada setiap titik footprint GEDI valid dan mengekspor dataset tabular untuk training.
   - **Lokasi Penyimpanan Dataset:** Hasil dataset training disimpan di `data/collections/dataset_gedi_valid.csv` (dapat disalin ke `data/table/dataset.csv`). File checkpoint raster sementara di `data/collections/` otomatis dibersihkan setelah dataset berhasil diekspor.
2. **Pemodelan Machine Learning** (`modelling.ipynb`):
   - Memuat dataset tabular (`data/table/dataset.csv` atau `data/collections/dataset_gedi_valid.csv`).
   - Melakukan *Exploratory Data Analysis* (EDA), analisis korelasi fitur, dan visualisasi distribusi AGBD.
   - Melatih model *Random Forest Regressor* dengan target AGBD dari GEDI L4A menggunakan 10-fold cross validation.
   - Menyimpan model terlatih sebagai `models/randomforest.pkl`.
3. **Prediksi Spasial & Inferensi** (`inference.ipynb`):
   - Mengunduh data citra satelit untuk seluruh area studi (AOI) pada periode target via STAC API.
   - Mengekstrak fitur raster *wall-to-wall* pada resolusi 30 m.
   - Menjalankan inferensi pada setiap piksel menggunakan model terlatih (`models/randomforest.pkl`).
   - Menerapkan *masking* non-vegetasi menggunakan probabilitas tutupan lahan Dynamic World.
   - Menyimpan peta estimasi AGBD berformat GeoTIFF dan visualisasinya.

### Sumber Data

| Dataset | Sumber | Penggunaan |
| --- | --- | --- |
| Sentinel-2 L2A | Microsoft Planetary Computer | Band reflektansi dan indeks vegetasi |
| Sentinel-1 RTC | Microsoft Planetary Computer | Backscatter VV/VH dan tekstur GLCM |
| Copernicus GLO-30 | Microsoft Planetary Computer | Elevasi, slope, dan aspect |
| GEDI L4A dan L2A | NASA Earthdata (CMR STAC) | Referensi AGBD dan tinggi kanopi (RH98) |
| Dynamic World | Model pretrained (TensorFlow) pada Sentinel-2 | Probabilitas 9 kelas tutupan lahan |

### Alur Prediksi

```
STAC API → Unduh & clip AOI → Fitur (S2, S1, DEM, DW) → Random Forest → Masking vegetasi → Peta AGBD (GeoTIFF)
```

- **Akuisisi:** pencarian scene berdasarkan AOI, rentang waktu, dan tutupan awan, lalu disimpan lokal sebagai GeoTIFF/CSV.
- **Preprocessing:** reproyeksi semua raster ke grid 30 m (UTM 48S), komposit median, indeks vegetasi (DVI, EVI, GNDVI, NDVI, RVI), tekstur GLCM, dan turunan topografi.
- **Inference:** prediksi AGBD pada piksel valid, lalu area non-vegetasi (berdasarkan Dynamic World) diberi nilai 0.
- **Output:** peta AGBD berformat GeoTIFF dan visualisasi peta.

## Struktur Proyek

```
agbd-with-stac-api/
├── assets/                 # Gambar hasil visualisasi (peta AGBD)
├── data/
│   ├── collections/        # Dataset training (dataset_gedi_valid.csv) & cache unduhan sementara
│   ├── table/              # Dataset tabular untuk pelatihan (dataset.csv)
│   ├── vector/             # Shapefile batas administrasi AOI (.shp, .shx, .dbf)
│   ├── sentinel02/         # Hasil unduhan Sentinel-2 (GeoTIFF)
│   ├── sentinel1/          # Hasil unduhan Sentinel-1 (GeoTIFF)
│   ├── glo30/              # Hasil unduhan DEM GLO-30 (GeoTIFF)
│   ├── gedil4a/            # Footprint GEDI L4A (CSV)
│   ├── dynamic_world/      # Probabilitas tutupan lahan (GeoTIFF)
│   ├── preprocessed/       # Cache fitur Sentinel-1 (.npz)
│   └── inference/          # Peta AGBD hasil prediksi (GeoTIFF)
├── models/
│   ├── randomforest.pkl    # Model Random Forest terlatih
│   └── dynamicword/        # Model Dynamic World (TensorFlow SavedModel)
├── create_dataset.ipynb    # Akuisisi data, ekstraksi fitur raster & pembuatan dataset tabular GEDI
├── modelling.ipynb         # EDA, pelatihan, validasi silang, dan evaluasi model Random Forest
├── inference.ipynb         # Inferensi spasial wall-to-wall pada AOI dan ekspor peta GeoTIFF
├── pyproject.toml          # Dependensi proyek
├── uv.lock                 # Versi dependensi terkunci
├── .python-version         # Versi Python
└── README.md
```

> Folder di dalam `data/` dibuat otomatis oleh notebook saat proses unduh/ekstraksi dijalankan.

## Persiapan

**Prasyarat:** Python 3.13 atau lebih baru, [uv](https://docs.astral.sh/uv/), dan akun [NASA Earthdata](https://urs.earthdata.nasa.gov/) untuk mengakses data GEDI.

```bash
git clone https://github.com/pan-don/agbd-with-stac-api.git
cd agbd-with-stac-api
uv sync
```

Buat file `.env` di root proyek:

```env
USERNAME_NASA=<username_earthdata>
PASSWORD_NASA=<password_earthdata>
```

## Cara Penggunaan

1. **Pembuatan Dataset (`create_dataset.ipynb`)**:
   Jalankan untuk mengunduh citra satelit pada batas AOI, memfilter footprint GEDI yang valid, mengekstrak fitur spasial, dan menghasilkan dataset training di `data/collections/dataset_gedi_valid.csv`.
2. **Pelatihan Model (`modelling.ipynb`)**:
   Jalankan untuk menganalisis fitur, melatih model Random Forest Regressor menggunakan 10-fold cross validation, mengevaluasi performa (RMSE, MAE, R²), dan menyimpan model ke `models/randomforest.pkl`.
3. **Prediksi Spasial & Pembuatan Peta (`inference.ipynb`)**:
   Jalankan secara berurutan untuk mengekstrak fitur seluruh grid AOI pada periode target, menjalankan model Random Forest, menerapkan masking non-vegetasi, dan mengekspor peta AGBD ke `data/inference/`.

Area studi, periode, dan resolusi dapat disesuaikan pada bagian *Config* di masing-masing notebook.

## Teknologi

Python, STAC (`pystac-client`, `planetary-computer`), `rasterio`, `geopandas`, `scikit-learn`, `scikit-image`, `TensorFlow`, `h5py`, `matplotlib`, dan `seaborn`.