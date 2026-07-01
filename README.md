# 🛰️ Ubiquitous-Eye-for-Land-Integrity

## 📁 Project Structure

\```
satellite-land-classification/
│
├── data/
│   ├── raw/
│   │   ├── landsat/
│   │   │   └── LC08_L2SP_137043_*/
│   │   └── sentinel2/
│   │       └── S2A_MSIL2A_*/
│   │           └── GRANULE/
│   │               └── IMG_DATA/
│   │                   ├── R10m/
│   │                   └── R20m/
│   └── processed/
│       └── filtered-sentinel2-dataset/
│           ├── data1.parquet
│           ├── data2.parquet
│           ├── data3.parquet
│           ├── data4.parquet
│           ├── data5.parquet
│           ├── data6.parquet
│           ├── data7.parquet
│           ├── data8.parquet
│           ├── data9.parquet
│           ├── data10.parquet
│           ├── data11.parquet
│           └── data12.parquet
├── notebooks/
│   ├── data-preparation.ipynb
│   └── capstone-model.ipynb
├── saved_models/
│   ├── xgboost.joblib
│   ├── catboost.joblib
│   ├── lightgbm.joblib
│   ├── cart.joblib
│   ├── label_encoder.joblib
│   └── scaler.joblib
└── README.md
\```

---

## 🔄 Pipeline Overview

\```
┌─────────────────────┐     ┌──────────────────────┐
│  USGS Landsat 8     │     │ Copernicus Sentinel-2 │
│  (Level-2 .TIF)     │     │ (Level-2A .SAFE)      │
└────────┬────────────┘     └──────────┬────────────┘
         │   Band data (.jp2/.tif)     │
         └──────────────┬──────────────┘
                        ▼
           ┌────────────────────────┐
           │  Step 1: Band Reading  │
           │  & Normalization       │
           └────────────┬───────────┘
                        ▼
           ┌────────────────────────┐
           │  Step 2: Index         │
           │  Computation           │
           │  NDVI, EVI, MNDWI,     │
           │  NDBI, BSI             │
           └────────────┬───────────┘
                        ▼
           ┌────────────────────────┐
           │  Step 3: SCL Filter    │
           │  Keep ClassID 4,5,6    │
           └────────────┬───────────┘
                        ▼
           ┌────────────────────────┐
           │  Filtered Parquet      │
           │  Dataset (~65M rows)   │
           └────────────┬───────────┘
                        ▼
           ┌────────────────────────┐
           │  Step 4: ML Training   │
           │  (20% test, 3-Fold CV) │
           └────────────┬───────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   XGBoost         LightGBM ✅      CatBoost        CART
   89.63%          89.67%           89.22%          89.00%
\```

---

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
