# Panem Cafeteria — Daily Sales Demand Forecasting


> Predict the top 5 food products sold per branch, per day, so each location can plan its daily stock in advance.

---

## Business Problem

**Panem** is a café and bakery chain with 7 branches in Monterrey, Nuevo León, Mexico. Every morning, each branch needs to decide how much of each product to prepare or order. Over- or under-stocking results in waste or lost sales.

The goal of this project is to build a **daily demand forecasting system** that predicts the quantity sold of each branch's top 5 products for the upcoming day — giving the operations team a data-driven basis for their daily stock decisions.

---

## Project Structure

```
ChallengeAI/
│
├── data/
│   ├── raw/                              # Original POS CSV exports — DO NOT MODIFY
│   │   ├── detail_Panem-Carreta_*.csv    # 186+ files, one per branch × date range
│   │   └── ...
│   │
│   ├── processed/                        # ML-ready branch files (output of pipeline)
│   │   ├── branch_carreta.csv
│   │   ├── branch_credi_club.csv
│   │   ├── branch_hospital_zambrano.csv
│   │   ├── branch_hotel_kavia.csv
│   │   ├── branch_plaza_nativa.csv
│   │   ├── branch_plaza_qin.csv
│   │   ├── branch_punto_valle.csv
│   │   ├── top5_per_branch.csv           # Final model input: top 5 products only
│   │   └── top5_summary.csv             # Reference: branch × top 5 ranking table
│   │
│   ├── weather/
│   │   ├── Clima Monterrey.csv          # Raw Monterrey weather station data
│   │   └── Clima_limpio.csv             # Cleaned: date + tavg only
│   │
│   ├── intermediate/                     # Large pipeline artifacts (git LFS tracked)
│   │   ├── df_complete_186_files.csv     # All raw CSVs concatenated (~3.76M rows)
│   │   ├── complete data with weather.csv # Sales + weather merged
│   │   └── datanomodifier.csv            # Primary input for analysis (modifiers removed)
│   │
│   └── geospatial/
│       └── 19-NL.geojson                # Nuevo León state geographic boundaries
│
├── notebooks/                            # All analysis and pipeline notebooks
│   ├── 00_data_pipeline.ipynb
│   ├── 01_data_cleaning.ipynb
│   ├── 02_eda_sales.ipynb
│   ├── 03_eda_weather_temporal.ipynb
│   ├── 04_eda_hourly_patterns.ipynb
│   ├── 05_visualizations.ipynb
│   ├── 06_feature_engineering.ipynb
│   └── 07_top_products.ipynb
│
├── models/
│   ├── arima/                            # ARIMA/SARIMA model code (to be built)
│   ├── prophet/                          # Facebook Prophet model code (to be built)
│   └── xgboost/                          # XGBoost model code (to be built)
│
├── CLAUDE.md                             # AI assistant context and project guidelines
└── README.md                             # This file
```

---

## Branches

| Branch | Processed file | Rows |
|--------|---------------|------|
| Panem - Punto Valle | `branch_punto_valle.csv` | ~63,000 |
| Panem - Plaza QIN | `branch_plaza_qin.csv` | ~55,000 |
| Panem - Hospital Zambrano | `branch_hospital_zambrano.csv` | ~44,000 |
| Panem - Hotel Kavia | `branch_hotel_kavia.csv` | ~42,000 |
| Panem - Carreta | `branch_carreta.csv` | ~31,000 |
| Panem - Plaza Nativa | `branch_plaza_nativa.csv` | ~19,000 |
| Panem - Credi Club | `branch_credi_club.csv` | ~5,800 |

---

## Notebooks

### Run order

```
00 → 01 → 02
          03
          04
          05
          06 → 07
```

Notebooks 02–05 (EDA) can run in any order after 01. Notebook 07 must run after 06.

---

### `00_data_pipeline.ipynb` — ETL Pipeline

**What it does:** Pure data extraction and loading — no analysis, no graphs.

| Step | Input | Output |
|------|-------|--------|
| Concat raw CSVs | 186 files in `data/raw/` | `data/intermediate/df_complete_186_files.csv` |
| Clean weather | `data/weather/Clima Monterrey.csv` | `data/weather/Clima_limpio.csv` |
| Merge weather | Both files above | `data/intermediate/complete data with weather.csv` |
| Remove modifiers + add features | Merged file | `data/intermediate/datanomodifier.csv` |

**Key decisions made here:**
- Only `date` and `tavg` are kept from weather data
- `cold_or_warm` column created: warm if `tavg >= 25°C`, cold otherwise
- `day_part` column created: morning (6–12), afternoon (12–19), night (19–6)
- `is_modifier=True` rows removed — modifier rows track customizations, not base product sales

---

### `01_data_cleaning.ipynb` — Data Cleaning

**What it does:** Loads `datanomodifier.csv` and applies all standardization steps.

- Fix datetime columns to proper `datetime64` type
- Standardize `is_modifier` to boolean
- Remove beverage groups (not the forecasting target)
- Map legacy branch names to canonical names (e.g., "Panem - Hotel Kavia N" → "Panem - Hotel Kavia")
- Fix product name typos (e.g., "SANDWITCH" → "SANDWICH")
- Report shape and missing values

**Output:** A clean `df` dataframe used as the base for all analysis notebooks.

---

### `02_eda_sales.ipynb` — Sales Analysis

**What it does:** Answers the core business question — what sells and where?

- Total sales by branch (quantity + % share)
- Sales by channel: dine-in (Restaurant) vs takeout (Para llevar)
- Top 20 items overall by units sold
- Top 20 items per branch by units sold
- Top 20 items per branch by revenue (MXN)

**Why this matters:** Identifies the candidate top-5 products per branch and reveals channel mix differences between locations.

---

### `03_eda_weather_temporal.ipynb` — Weather & Temporal Patterns

**What it does:** Analyzes how external factors affect demand.

- Top items on cold vs warm days (overall and per branch)
- Top items by day of week (Monday → Sunday)
- Top items by day part (morning / afternoon / night)
- Monthly sales bar charts per branch
- Average ticket value by year (overall and per branch)
- Box plots: cold vs warm ticket values with bootstrap confidence intervals

**Why this matters:** Temperature, day of week, and time of day are all features in the forecasting model. This notebook shows us they're actually useful.

---

### `04_eda_hourly_patterns.ipynb` — Hourly Patterns & Holidays

**What it does:** Deep dive into hour-by-hour behavior and holiday effects.

- Creates holiday columns (Mexican federal holidays 2022–2026, classified as `Festivo Oficial` or `Puente`)
- Hour × day-of-week heatmap across all branches combined
- Same heatmap per branch and per temperature group (cold/warm)
- Top day + hour combinations ranked table

**Why this matters:** The hour × day heatmap directly answers "when do we need to have stock ready?" — it's one of the most operationally useful outputs in the project.

---

### `05_visualizations.ipynb` — Interactive Visualizations

**What it does:** Builds all interactive Plotly charts for presentations and stakeholder sharing.

- Sankey diagrams: branch → top 5 products (overall, cold days, warm days)
- Sankey: day parts → top products
- Line charts: monthly quantity trend for top 5 items per branch
- Top 3 food groups per branch × day of week

**Why Sankey diagrams?** They visually show which products dominate at each branch, and the flow widths make it instantly clear which differences between cold/warm days matter.

---

### `06_feature_engineering.ipynb` — Feature Engineering

**What it does:** Transforms the clean data into ML-ready branch datasets.

1. Adds holiday columns
2. Applies 90% volume rule — keeps only top 83 products
3. Removes non-commercial items (SUBSIDIO TEC, PAN DE MUERTO)
4. Keeps only completed sales (`action == "Venta"`)
5. Drops 36 columns not needed for ML
6. Aggregates to daily level: one row per (branch, date, product)
7. Computes 9 rolling window features (1, 3, 7, 15, 30, 60, 90, 180, 365 days)
8. Exports one CSV per branch to `data/processed/`

**The rolling window features are the most important step.** They are the auto-regressive features that let the model learn "if this product sold a lot recently, it will probably sell well tomorrow." The `shift(1)` ensures today's quantity is not leaked into today's features.

---

### `07_top_products.ipynb` — Top Products

**What it does:** Identifies the final top-5 products per branch and produces model-ready files.

1. Loads all 7 branch CSVs
2. Ranks each product by total cumulative volume per branch
3. Shows the top-10 ranking for review
4. Filters dataset to only the top-5 products per branch
5. Checks date coverage and history length per product
6. Exports:
   - `data/processed/top5_per_branch.csv` — full dataset, top-5 products only
   - `data/processed/top5_summary.csv` — the 7×5 reference ranking table

---

## Clean Data Schema

Each file in `data/processed/branch_*.csv` has 20 columns:

| Column | Type | Description |
|--------|------|-------------|
| `sucursal` | str | Branch name |
| `operating_date` | date | Sale date |
| `item` | str | Product name (uppercase Spanish) |
| `quantity` | float | **Target variable** — units sold that day |
| `day_name` | str | Day of week in Spanish (lunes–domingo) |
| `week_number` | int | ISO week number |
| `tavg` | float | Average temperature °C |
| `cold_or_warm` | str | "warm" (≥25°C) or "cold" (<25°C) |
| `holiday_type` | str | "Festivo Oficial", "Puente", or "No holiday" |
| `holidays` | bool | True if a holiday date |
| `month` | float | Month number (1–12) |
| `qty_roll_1` | float | Quantity sold the previous day |
| `qty_roll_3` | float | Rolling sum over past 3 days |
| `qty_roll_7` | float | Rolling sum over past 7 days |
| `qty_roll_15` | float | Rolling sum over past 15 days |
| `qty_roll_30` | float | Rolling sum over past 30 days |
| `qty_roll_60` | float | Rolling sum over past 60 days |
| `qty_roll_90` | float | Rolling sum over past 90 days |
| `qty_roll_180` | float | Rolling sum over past 180 days |
| `qty_roll_365` | float | Rolling sum over past 365 days |

---

## Modeling Strategy

### Target
Predict `quantity` for each of the **top 5 products per branch** for the **next day**.

### Train / Validation Split
**Chronological only** — never shuffle time-series data. Validate on the most recent N days.

### Planned Approaches

#### ARIMA / SARIMA — `models/arima/`
- Classical univariate time-series baseline
- One model per (branch, product) pair — 35 models total
- Captures autocorrelation, trend, and weekly/annual seasonality
- No external features (temperature, holidays) — serves as baseline

#### Facebook Prophet — `models/prophet/`
- Handles multiple seasonality components (weekly, monthly, yearly)
- Built-in holiday support — can directly use the holiday columns
- Accepts `tavg` as an additional regressor
- Strengths: robust to missing data, automatic seasonality detection

#### XGBoost — `models/xgboost/`
- Tabular regression using all engineered features
- Can train a global model (all branches together) or per-branch
- Feature groups: lagged quantity (`qty_roll_*`), temporal, weather, holidays
- Strengths: handles non-linear interactions, fast training and inference

### Evaluation Metrics
- **MAE** — Mean Absolute Error (primary metric, in units)
- **RMSE** — Root Mean Squared Error (penalizes large errors)
- **MAPE** — Mean Absolute Percentage Error (use with caution — undefined on zero-sales days)

---

## Key Design Decisions

| Decision | Reasoning |
|----------|-----------|
| Beverages excluded | Not the forecasting target; different supply chain logic |
| 25°C warm threshold | Matches Monterrey's seasonal divide in product preference |
| 90% volume rule (top 83 products) | Long-tail SKUs have insufficient data to model reliably |
| Rolling windows start with `shift(1)` | Prevents data leakage — today's sale cannot predict itself |
| Daily aggregation level | Matching the granularity of the stock planning decision |
| Per-branch model files | Each branch has distinct product mix and customer patterns |

---

## Notes

- All product and branch names are in **Spanish uppercase**
- Day names are in **Spanish**: lunes, martes, miércoles, jueves, viernes, sábado, domingo
- Currency is **Mexican Pesos (MXN)**
- City: **Monterrey, Nuevo León, Mexico**
- Data range: **January 2022 – February 2026** (4+ years)
- Branch Credi Club has ~5,800 rows vs ~63,000 for Punto Valle — models for Credi Club may need different treatment
