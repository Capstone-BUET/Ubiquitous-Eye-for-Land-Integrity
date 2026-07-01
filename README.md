# 🛰️ Ubiquitous-Eye-for-Land-Integrity

## 📁 Project Structure
```
Ubiquitous-Eye-for-Land-Integrity/
│
├── Frontend                          # React frontend (Vite)
│   ├── public
│   ├── src
│   │   ├── assets
│   │   ├── components
│   │   │   ├── LayerSelection.css
│   │   │   ├── LayerSelection.jsx
│   │   │   ├── MapView.css
│   │   │   └── MapView.jsx
│   │   ├── App.css
│   │   ├── App.jsx
│   │   ├── index.css
│   │   └── main.jsx
│   ├── .dockerignore
│   ├── .env
│   ├── .env.example
│   ├── .gitignore
│   ├── BACKEND_GUIDE.md
│   ├── eslint.config.js
│   ├── index.html
│   ├── package-lock.json
│   ├── package.json
│   ├── README.md
│   └── vite.config.js
│
├── inference                         # ML inference engine
│   ├── capstone_model_v2             # Trained model files
│   │   ├── cart.joblib
│   │   ├── catboost.joblib
│   │   ├── label_encoder.joblib
│   │   ├── lightgbm.joblib
│   │   ├── scaler.joblib
│   │   └── xgboost.joblib
│   └── inference.py
│
├── server                            # Backend API server
│   ├── GRANULE                       # Sentinel-2 raw scene data
│   │   └── L2A_T45QYE_...
│   │       └── IMG_DATA
│   │           └── T45QYE_*_SCL_20m.jp2
│   ├── inference
│   │   ├── capstone_model_v2
│   │   └── inference.py
│   ├── __init__.py
│   ├── .dockerignore
│   ├── api_server.py
│   ├── bimonthly_composite.py
│   ├── Dockerfile
│   ├── requirements.txt
│   └── scl_to_csv.py
│
├── .env
├── .env.example
├── bimonthly_composite.py
├── docker-compose.yml
├── Dockerfile
├── frontend.nginx.conf
├── planet.py
└── README.md
```

## 📓 Kaggle Notebooks & Training Data

```
Kaggle Workspace
│
├── Landsat                                         # Raw USGS Landsat 8 scenes
│   ├── LC08_L2SP_137043_20131106_20200912_02_T1
│   ├── LC08_L2SP_137043_20131224_20200912_02_T1
│   └── LC08_L2SP_137043_20140125_20200912_02_T1
│
├── Copernicus                                      # Raw Sentinel-2 scenes
│   ├── S2A_MSIL2A_20201112T043031_N0500_R133_T45QZG_20230324T113157.SAFE
│   ├── S2A_MSIL2A_20201112T043031_N0500_R133_T45QZG_20230410T140649.SAFE
│   ├── S2A_MSIL2A_20201122T043111_N0500_R133_T45QZG_20230411T134214.SAFE
│   └── ...
│
├── Landsat_filter                                  # Landsat band extraction
│   ├── landsat.ipynb
│   ├── input
│   │   └── Landsat dataset
│   └── output
│       └── landsat8_10m_all_bands.csv               # 2.22 GB
│
├── Data_Preparation                                # Sentinel-2 band extraction → Parquet
│   ├── data-preparation.ipynb
│   ├── input
│   │   ├── Copernicus dataset
│   │   └── Filtered Sentinel-2 dataset
│   └── output
│       └── filtered_dataset
│           ├── data1.parquet
│           ├── data2.parquet
│           └── ...
│
├── Fork-of-capstone-model                          # Multi-model training
│   ├── fork-of-capstone-model.ipynb
│   ├── input
│   │   ├── Filtered Sentinel-2 Dataset
│   │   └── Datav1.csv
│   └── Output_model
│       ├── cart.joblib
│       ├── catboost.joblib
│       ├── label_encoder.joblib
│       ├── lightgbm.joblib
│       ├── scaler.joblib
│       └── xgboost.joblib
│
└── Inference_Code                                  # Run predictions on Landsat
    ├── inference-code-d35339.ipynb
    ├── input
    │   ├── landsat8_10m_all_bands.csv               # 2.22 GB (from Landsat_filter)
    │   └── Fork-of-capstone-model
    │       ├── catboost_info
    │       │   ├── catboost_training.json
    │       │   ├── learn
    │       │   │   └── events.out.tfevents
    │       │   ├── learn_error.tsv
    │       │   └── time_left.tsv
    │       └── saved_models
    │           ├── cart.joblib
    │           ├── catboost.joblib
    │           ├── label_encoder.joblib
    │           ├── lightgbm.joblib
    │           ├── scaler.joblib
    │           └── xgboost.joblib
    └── output
        └── landsat_prediction.parquet
```


## 🔄 Pipeline Overview

```
Data Sources
─────────────────────────────────────────────────
  [USGS Landsat 8]          [Copernicus Sentinel-2]
  Level-2 .TIF              Level-2A .SAFE
─────────────────────────────────────────────────
                       │
                       ▼
          ┌────────────────────────┐
          │   Step 1               │
          │   Band Reading &       │
          │   Normalization        │
          │   (rasterio / numpy)   │
          └───────────┬─────────── ┘
                      │
                      ▼
          ┌────────────────────────┐
          │   Step 2               │
          │   Index Computation    │
          │   NDVI, EVI, MNDWI,    │
          │   NDBI, BSI            │
          └───────────┬─────────── ┘
                      │
                      ▼
          ┌────────────────────────┐
          │   Step 3               │
          │   SCL Filter           │
          │   Keep ClassID 4,5,6   │
          │   Vegetation/Soil/     │
          │   Water                │
          └───────────┬─────────── ┘
                      │
                      ▼
          ┌────────────────────────┐
          │   Filtered Parquet     │
          │   Dataset (~65M rows)  │
          └───────────┬─────────── ┘
                      │
                      ▼
          ┌────────────────────────┐
          │   Step 4               │
          │   ML Training          │
          │   20% test, 3-Fold CV  │
          └───────────┬─────────── ┘
                      │
        ┌─────────────┼─────────────┐─────────────┐
        ▼             ▼             ▼             ▼
  [XGBoost]    [LightGBM ✅]  [CatBoost]     [CART]
   89.63%        89.67%        89.22%        89.00%
```


## 🗂️ Data Sources

### 1. USGS Landsat 8 — Level-2 Surface Reflectance

| Property | Value |
|----------|-------|
| Sensor | Landsat 8 OLI/TIRS |
| Product | Level-2 Science Product (L2SP) |
| Path/Row | 137/043 |
| Format | GeoTIFF (.tif), JSON metadata, XML angle files |
| Source | [USGS EarthExplorer](https://earthexplorer.usgs.gov/) |

### 2. Copernicus Sentinel-2 — Level-2A

| Property | Value |
|----------|-------|
| Satellites | Sentinel-2A, 2B, 2C |
| Product | MSIL2A (Bottom-of-Atmosphere reflectance) |
| Coverage | 12 tiles across Bangladesh (Rangpur, Meherpur, Jessore, Pabna, Naogoan, Dinajpur, Bogura, Mymensingh, Dhaka, Chattogram, Feni, and more) |
| Acquisitions | 2017 – 2026 |
| Format | .jp2 (JPEG 2000) bands inside .SAFE directory |
| Source | [Copernicus Data Space](https://dataspace.copernicus.eu/) |

---

## 🔬 Feature Engineering

### 1. Sentinel-2 Bands Used

| Band | Name | Resolution | Wavelength |
|------|------|------------|------------|
| B01 | Coastal Aerosol | 20 m | 443 nm |
| B02 | Blue | 10 m | 490 nm |
| B03 | Green | 10 m | 560 nm |
| B04 | Red | 10 m | 665 nm |
| B08 | Near-Infrared (NIR) | 10 m | 842 nm |
| B11 | SWIR-1 | 20 m | 1610 nm |
| B12 | SWIR-2 | 20 m | 2190 nm |
| SCL | Scene Classification Layer | 20 m | — (label) |

### 2. Spectral Indices Derived

| Index | Formula | Captures |
|-------|---------|----------|
| NDVI | (B08 − B04) / (B08 + B04) | Vegetation density |
| EVI | 2.5 × (B08 − B04) / (B08 + 6×B04 − 7.5×B02 + 1) | Enhanced vegetation |
| MNDWI | (B03 − B11) / (B03 + B11) | Open water bodies |
| NDBI | (B11 − B08) / (B11 + B08) | Built-up / urban areas |
| BSI | ((B11+B04) − (B08+B02)) / ((B11+B04) + (B08+B02)) | Bare soil |

### 3. Target Classes (SCL-based)

| ClassID | Label | Training Rows |
|---------|-------|---------------|
| 4 | Vegetation | 17,083,674 |
| 5 | Bare Soil | 41,406,079 |
| 6 | Water | 6,488,640 |

---

## 🤖 Model Training

### 1. Configuration

| Parameter | Value |
|-----------|-------|
| Total dataset | ~64.9M rows |
| Sample fraction | 5% per file (stratified) |
| Feature columns | B04, B03, B08, B02, B01, B12, B11, EVI, NDBI, MNDWI, BSI, NDVI |
| Train / Test split | 80% / 20% |
| Cross-validation | 3-Fold |
| Scaler | StandardScaler |
| Hardware | NVIDIA H100 GPU (Kaggle) |

### 2. Models & Results

| Model | Accuracy | 3-Fold CV | Vegetation F1 | Bare Soil F1 | Water F1 |
|-------|----------|-----------|---------------|--------------|----------|
| **LightGBM** | **89.67%** ✅ | 89.66% ± 0.01% | 0.85 | 0.92 | 0.82 |
| XGBoost | 89.63% | 89.60% ± 0.02% | 0.85 | 0.92 | 0.82 |
| CatBoost | 89.22% | 89.21% ± 0.02% | 0.85 | 0.92 | 0.82 |
| CART | 89.00% | 88.99% ± 0.02% | 0.84 | 0.92 | 0.82 |
