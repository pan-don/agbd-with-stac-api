<h1 align="center">AGBD Estimation with STAC API</h1>

Estimasi *Aboveground Biomass Density* (AGBD, Mg/ha) berbasis citra satelit. Data diakses langsung melalui STAC API, diolah menjadi fitur spasial, lalu diprediksi dengan model Random Forest terlatih untuk menghasilkan peta AGBD beresolusi 30 m.

## Gambaran Proyek

Proyek ini terdiri dari dua alur kerja:

1. **Pemodelan** (`agbd_modelling.ipynb`): melatih model Random Forest Regressor dengan target AGBD dari GEDI L4A dan fitur dari citra satelit, divalidasi dengan 10-fold cross validation, lalu disimpan sebagai `models/randomforest.pkl`.
2. **Prediksi spasial** (`notebook.ipynb`): mengambil data satelit lewat STAC API, mengekstrak fitur, menjalankan model terlatih pada setiap piksel, dan menyimpan peta AGBD sebagai GeoTIFF.

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
│   ├── table/              # Dataset tabular untuk pelatihan (dataset.csv)
│   ├── sentinel02/         # Hasil unduhan Sentinel-2 (GeoTIFF)
│   ├── sentinel1/          # Hasil unduhan Sentinel-1 (GeoTIFF)
│   ├── glo30/              # Hasil unduhan DEM GLO-30 (GeoTIFF)
│   ├── gedil4a/            # Footprint GEDI L4A (CSV)
│   ├── dynamic_world/      # Probabilitas tutupan lahan (GeoTIFF)
│   ├── preprocessed/       # Cache fitur Sentinel-1
│   └── inference/          # Peta AGBD hasil prediksi (GeoTIFF)
├── models/
│   ├── randomforest.pkl    # Model Random Forest terlatih
│   └── dynamicword/        # Model Dynamic World (TensorFlow SavedModel)
├── agbd_modelling.ipynb    # Pelatihan dan evaluasi model
├── notebook.ipynb          # Akuisisi data, preprocessing, dan inference
├── pyproject.toml          # Dependensi proyek
├── uv.lock                 # Versi dependensi terkunci
├── .python-version         # Versi Python
└── README.md
```

> Folder di dalam `data/` dibuat otomatis oleh notebook saat proses unduh dijalankan.

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

1. Jalankan `agbd_modelling.ipynb` untuk melatih model dan menyimpan `models/randomforest.pkl` (lewati jika model sudah tersedia).
2. Jalankan `notebook.ipynb` berurutan, dari konfigurasi AOI hingga penyimpanan peta di bagian *Save Map*.

Area studi, periode, dan resolusi dapat diubah pada bagian *Config* di `notebook.ipynb`.

## Teknologi

Python, STAC (`pystac-client`, `planetary-computer`), `rasterio`, `geopandas`, `scikit-learn`, `scikit-image`, `TensorFlow`, `h5py`, `matplotlib`, dan `seaborn`.