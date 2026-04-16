# CLAUDE.md — Panem Cafeteria Sales Forecasting

## Project Identity

**Company:** Panem (café/bakery chain, Monterrey, Mexico)  
**Goal:** Daily demand forecasting of the top 5 food products per branch so owners can plan daily stock.  
**Scope:** 7 branches, ~83 retained products, daily-level predictions.  
**Language:** Spanish product names, branch names, and day names throughout the data.

---

## Directory Structure

```
ChallengeAI/
├── data/
│   ├── raw/                        ← 186+ original POS CSV exports (DO NOT MODIFY)
│   │   └── Borrador complete data.ipynb  ← archived original notebook (reference only)
│   ├── processed/                  ← ML-ready branch files (output of notebook 06)
│   │   ├── branch_carreta.csv
│   │   ├── branch_credi_club.csv
│   │   ├── branch_hospital_zambrano.csv
│   │   ├── branch_hotel_kavia.csv
│   │   ├── branch_plaza_nativa.csv
│   │   ├── branch_plaza_qin.csv
│   │   ├── branch_punto_valle.csv
│   │   ├── top5_per_branch.csv     ← output of notebook 07 (model input)
│   │   └── top5_summary.csv        ← output of notebook 07 (reference table)
│   ├── weather/
│   │   ├── Clima Monterrey.csv     ← raw Monterrey weather station data
│   │   └── Clima_limpio.csv        ← cleaned weather: date + tavg only
│   ├── intermediate/               ← large pipeline artifacts (git LFS tracked)
│   │   ├── df_complete_186_files.csv   ← all 186 raw CSVs concatenated
│   │   ├── complete data with weather.csv  ← sales + weather merged
│   │   └── datanomodifier.csv      ← PRIMARY INPUT for all analysis notebooks
│   └── geospatial/
│       └── 19-NL.geojson           ← Nuevo León state boundaries
│
├── notebooks/                      ← ALL analysis and preparation notebooks
│   ├── 00_data_pipeline.ipynb      ← ETL: load raw CSVs, clean weather, merge, remove modifiers
│   ├── 01_data_cleaning.ipynb      ← Clean types, remove beverages, standardize names
│   ├── 02_eda_sales.ipynb          ← Sales by branch, channel, top products by qty and revenue
│   ├── 03_eda_weather_temporal.ipynb  ← Temperature, day-of-week, day-part, monthly charts
│   ├── 04_eda_hourly_patterns.ipynb   ← Hour×day heatmaps, holiday columns
│   ├── 05_visualizations.ipynb     ← Interactive Sankey + trend line charts (Plotly)
│   ├── 06_feature_engineering.ipynb  ← Daily aggregation, rolling windows, branch CSV export
│   └── 07_top_products.ipynb       ← Identify top 5 per branch, export model-ready files
│
├── models/
│   ├── arima/                      ← ARIMA/SARIMA model code (to be built)
│   ├── prophet/                    ← Facebook Prophet model code (to be built)
│   └── xgboost/                    ← XGBoost model code (to be built)
│
├── CLAUDE.md                       ← This file
└── README.md                       ← Project documentation
```

---

## Notebook Run Order

Must run in sequence — each notebook depends on the previous:

```
00_data_pipeline → 01_data_cleaning → 02_eda_sales
                                    → 03_eda_weather_temporal
                                    → 04_eda_hourly_patterns
                                    → 05_visualizations
                                    → 06_feature_engineering → 07_top_products
```

Notebooks 02–05 can run in any order after 01. Notebook 07 must run after 06.

---

## Clean Data Schema (data/processed/branch_*.csv)

One row per `(sucursal, operating_date, item)`.

| Column | Type | Description |
|--------|------|-------------|
| `sucursal` | str | Branch name |
| `operating_date` | date | Sale date |
| `item` | str | Product name (uppercase Spanish) |
| `quantity` | float | **Target variable** — units sold that day |
| `day_name` | str | Day of week in Spanish |
| `week_number` | int | ISO week |
| `tavg` | float | Average temperature °C |
| `cold_or_warm` | str | "warm" (≥25°C) or "cold" (<25°C) |
| `holiday_type` | str | "Festivo Oficial", "Puente", or "No holiday" |
| `holidays` | bool | True if holiday |
| `month` | float | Month (1–12) |
| `qty_roll_1..365` | float | Rolling window features (9 columns) |

---

## Deliberate Exclusions — Do Not Re-introduce

1. **Beverages** — `CAFE Y BEBIDAS CALIENTES` and `JUGOS Y BEBIDAS FRIAS` removed by design
2. **Modifier rows** — `is_modifier=True` rows integrated and removed
3. **Seasonal items** — "PAN DE MUERTO" removed (extreme seasonality)
4. **Non-commercial items** — "SUBSIDIO TEC" removed (employee subsidy, not demand)
5. **Long-tail products** — only top 83 by cumulative volume (90% rule)

---

## Modeling Context

### Target Variable
`quantity` — total units sold per (branch, date, product)

### Model Scope
Top 5 products per branch per day. The top-5 list is defined in `data/processed/top5_summary.csv` and `data/processed/top5_per_branch.csv`.

### Planned Approaches
- **ARIMA/SARIMA** → `models/arima/` — univariate time-series baseline
- **Facebook Prophet** → `models/prophet/` — handles seasonality + holidays
- **XGBoost** → `models/xgboost/` — tabular regression with rolling lags + all features

### Key Features for Modeling
1. Lagged quantity: `qty_roll_1` through `qty_roll_365`
2. Temporal: `day_name`, `week_number`, `month`
3. Weather: `tavg`, `cold_or_warm`
4. Holidays: `holidays`, `holiday_type`
5. Branch identity: `sucursal`

### Evaluation
- Time-aware train/validation split — **never shuffle**, always split chronologically
- Metrics: MAE, RMSE, MAPE per product per branch
- Be careful with MAPE on zero or near-zero days

---

## Hard Rules

- Do not modify anything in `data/raw/` — original source files
- Do not shuffle data for time-series splits
- Do not re-introduce beverages, modifiers, or excluded items
- Do not modify `data/processed/` files manually — only notebook 06 or 07 should write there
- All paths in notebooks use `PROJECT_ROOT = os.path.abspath(os.path.join(os.getcwd(), ".."))` from the `notebooks/` directory

---

## Environment

- Python / Jupyter Notebooks
- Libraries: pandas, numpy, matplotlib, seaborn, plotly, statsmodels, prophet, xgboost, scikit-learn
- All product/branch names in **Spanish uppercase**
- Day names in **Spanish**: lunes, martes, miércoles, jueves, viernes, sábado, domingo
- Currency: **Mexican Pesos (MXN)**
- Location: **Monterrey, Nuevo León, Mexico**
- Data range: **2022-01-01 through ~2026-02-28**
