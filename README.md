# ID2223 Lab 1 – Air Quality Forecasting (Paris, Boulevard Peripherique Est)

This repo contains our solution for ID2223 Lab 1 (Scalable ML & DL).  
The goal is to build an end-to-end **serverless air quality forecasting system** for a specific AQICN sensor:

- **Location:** Paris, Boulevard Peripherique Est  
- **Target:** daily PM2.5 concentration  
- **Feature store:** Hopsworks Feature Store  
- **Model:** XGBoost regression  
- **Dashboard:** GitHub Pages  

Public dashboard:  
<https://lawlv.github.io/id2223-lab1-airquality/air-quality/>

---

## 1. System overview

The system consists of four main pipelines:

1. **Backfill feature pipeline** (`1_air_quality_feature_backfill.ipynb`)  
   - Reads historical PM2.5 CSV from AQICN (`data/air-quality-data.csv`)  
   - Downloads historical weather data from Open-Meteo  
   - Creates two Feature Groups in Hopsworks:  
     - `air_quality` – PM2.5 label + location info  
     - `weather` – daily weather features  
   - Registers sensor metadata as a Hopsworks secret (`SENSOR_LOCATION_JSON`)

2. **Daily feature pipeline** (`2_air_quality_feature_pipeline.ipynb`)  
   - Fetches **today’s PM2.5** from AQICN API  
   - Fetches **today’s noon weather forecast** from Open-Meteo  
   - Appends one new row to both Feature Groups:
     - `air_quality`  
     - `weather`  

3. **Training pipeline** (`3_air_quality_training_pipeline.ipynb`)  
   - Builds a Feature View by joining `air_quality` and `weather` on `city`  
   - Splits data into train/test based on date (train `< 2025-05-01`)  
   - Trains an XGBoost model to predict PM2.5  
   - Evaluates the model (MSE, R², hindcast plots)  
   - Registers the model in the Hopsworks Model Registry as  
     `air_quality_xgboost_model` (baseline v1, improved v2 – see Section 6)

4. **Batch inference & monitoring** (`4_air_quality_batch_inference.ipynb`)  
   - Loads **version 1** of `air_quality_xgboost_model` from the Model Registry  
   - Reads future weather forecasts and produces PM2.5 forecasts  
   - Stores predictions in Feature Group `aq_predictions`  
   - Generates plots:
     - `pm25_forecast.png` – future PM2.5 forecast (multi-day)  
     - `pm25_hindcast_1day.png` – **1-day-ahead hindcast**  
       (for each day T, show prediction made on T−1 vs actual PM2.5 on T)  
   - Copies these images to `docs/air-quality/assets/img/` so the dashboard can show them

---

## 2. Hopsworks setup

To run the pipelines you need:

1. A Hopsworks project, e.g. `id2223_lab1_yilai`.  
2. A personal API key with Data Scientist permissions.  
3. A local `.env` file in the project root (based on `.env.example`) containing:

   - `HOPSWORKS_API_KEY`  
   - `HOPSWORKS_PROJECT`  
   - `HOPSWORKS_HOST`  
   - `AQICN_API_KEY`  
   - AQICN sensor info (`AQICN_COUNTRY`, `AQICN_CITY`, `AQICN_STREET`, `AQICN_URL`, etc.)

On GitHub Actions, the Hopsworks API key is provided via a repository secret:

- `HOPSWORKS_API_KEY` → used in `.github/workflows/air-quality-daily.yml`.

---

## 3. How to run locally

### 3.1 Environment setup

```bash
# 1. Create & activate conda environment
conda create -n id2223lab1 python=3.11
conda activate id2223lab1

# 2. Install dependencies
pip install -r requirements.txt
```
Create a `.env` file in the project root based on `.env.example` and fill in:

- HOPSWORKS_PROJECT  
- HOPSWORKS_HOST  
- HOPSWORKS_API_KEY  
- AQICN_API_KEY  
- AQICN_COUNTRY / AQICN_CITY / AQICN_STREET / AQICN_URL  

### 3.2 Run the notebooks

Start Jupyter Lab:
```bash
jupyter lab
```
Then run the notebooks in this order (kernel: `id2223lab1`):

1. `notebooks/airquality/1_air_quality_feature_backfill.ipynb`  
   - Backfills historical `air_quality` and `weather` Feature Groups.

2. `notebooks/airquality/2_air_quality_feature_pipeline.ipynb`  
   - Inserts today’s PM2.5 and weather into the Feature Store.

3. `notebooks/airquality/3_air_quality_training_pipeline.ipynb`  
   - Reads features from the Feature Store via Feature View.  
   - Trains the XGBoost model.  
   - Registers the model in the Hopsworks Model Registry as  
     `air_quality_xgboost_model` (v1 baseline, v2 with lag features).

4. `notebooks/airquality/4_air_quality_batch_inference.ipynb`  
   - Loads v1 of the registered model.  
   - Generates future forecasts and 1-day-ahead hindcast plots.  
   - Writes predictions to `aq_predictions`.  
   - Saves images into `docs/air-quality/assets/img/`.

---

## 4. Serverless scheduling (GitHub Actions)

The daily feature pipeline is automated with **GitHub Actions**.

Workflow file: `.github/workflows/air-quality-daily.yml`

It performs:

1. Check out this repository.  
2. Install Python dependencies.  
3. Execute the daily feature pipeline notebook.

The workflow uses a repository secret:

- HOPSWORKS_API_KEY → same key as in the local `.env`.

In the workflow it is exposed as:

env:  
  HOPSWORKS_API_KEY: ${{ secrets.HOPSWORKS_API_KEY }}

Triggers:

- Manual trigger (`workflow_dispatch`).  
- Cron schedule (runs once per day).

Result: the `air_quality` and `weather` Feature Groups are updated **every day automatically**.

---

## 5. Dashboard (GitHub Pages)

The dashboard is a static site served by GitHub Pages from the `docs` folder.

- Air quality page: `docs/air-quality/index.md`  
- Images generated by the batch inference notebook:
  - `docs/air-quality/assets/img/pm25_forecast.png`  
    - Multi-day **future PM2.5 forecast** (computed when notebook 4 runs).  
  - `docs/air-quality/assets/img/pm25_hindcast_1day.png`  
    - **1-day-ahead hindcast**: for each day T, the prediction made on day T−1  
      (using T’s weather forecast) vs the actual PM2.5 on day T.

GitHub Pages is configured to:

- Source: branch `main`  
- Folder: `/docs`

Public dashboard URL:

https://lawlv.github.io/id2223-lab1-airquality/air-quality/

---

## 6. Model improvement – lag features (C-grade)

For the C-grade extension, I improved the model by adding **lagged PM2.5 features**:

- pm25_lag_1 – PM2.5 one day before  
- pm25_lag_2 – PM2.5 two days before  
- pm25_lag_3 – PM2.5 three days before  

These features are added in the training notebook after the initial train/test split.  
The idea: PM2.5 has strong **temporal correlation** (today depends on the last few days).

### 6.1 Lag feature code (main idea)

After `feature_view.train_test_split(...)`, I recombine and add lags:

```bash
# Combine train and test with labels
train_df = X_train.copy()
train_df["pm25"] = y_train.values

test_df = X_test.copy()
test_df["pm25"] = y_test.values

full_df = pd.concat([train_df, test_df], ignore_index=True)
full_df["date"] = pd.to_datetime(full_df["date"])

# Sort by date
full_df = full_df.sort_values(["date"]).reset_index(drop=True)

# Add lagged features: previous 1/2/3 days of PM2.5
full_df["pm25_lag_1"] = full_df["pm25"].shift(1)
full_df["pm25_lag_2"] = full_df["pm25"].shift(2)
full_df["pm25_lag_3"] = full_df["pm25"].shift(3)

# Drop rows where lag values are NaN
full_df = full_df.dropna(
    subset=["pm25_lag_1", "pm25_lag_2", "pm25_lag_3"]
).reset_index(drop=True)

# Re-split by date (same test_start)
full_df["date_for_split"] = full_df["date"].dt.date
test_start_date = test_start.date()

train_mask = full_df["date_for_split"] < test_start_date
train_df = full_df[train_mask].copy()
test_df = full_df[~train_mask].copy()

X_train = train_df.drop(columns=["pm25", "date_for_split"])
y_train = train_df["pm25"]

X_test = test_df.drop(columns=["pm25", "date_for_split"])
y_test = test_df["pm25"]

# Final feature matrices (exclude date from features)
X_features = X_train.drop(columns=["date"])
X_test_features = X_test.drop(columns=["date"])
```

Then I retrain the XGBoost model with these extended features and register it as  
`air_quality_xgboost_model` version 2.

### 6.2 Results (v1 vs v2)

- Baseline model (v1, no lag):
  - R² ≈ −1.418  
  - MSE ≈ 344.7  

- Improved model (v2, with pm25_lag_1/2/3):
  - R² ≈ 0.0657  
  - MSE ≈ 136.25  

So:

- MSE decreases by about **60%**.  
- R² moves from negative (worse than predicting the mean) to slightly positive.  

The batch inference notebook (4) still uses v1 for daily forecasts.  
Version 2 is used for **offline evaluation and comparison**, which satisfies the C-grade requirement.
