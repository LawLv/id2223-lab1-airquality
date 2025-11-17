# ID2223 Lab 1 – Air Quality Forecasting (Paris, Boulevard Peripherique Est)

This repo contains my solution for **ID2223 Lab 1 (Scalable ML & DL)**.
The goal is to build an end-to-end **serverless air quality forecasting system**
for a specific AQICN sensor:

- **Location**: Paris, Boulevard Peripherique Est  
- **Target**: daily PM2.5 concentration  
- **Feature store**: Hopsworks Feature Store  
- **Model**: XGBoost regression  
- **Dashboard**: GitHub Pages

Public dashboard:  
👉 https://lawlv.github.io/id2223-lab1-airquality/air-quality/

---

## 1. System overview

The system consists of the following pipelines:

1. **Backfill feature pipeline** (`1_air_quality_feature_backfill.ipynb`)
   - Reads historical PM2.5 CSV from AQICN (`data/air-quality-data.csv`)
   - Downloads historical weather data from Open-Meteo
   - Creates two Feature Groups in Hopsworks:
     - `air_quality` (pm25 label + location info)
     - `weather` (daily weather features)
   - Registers sensor metadata as Hopsworks secrets (`SENSOR_LOCATION_JSON`)

2. **Daily feature pipeline** (`2_air_quality_feature_pipeline.ipynb`)
   - Fetches **today's** PM2.5 from AQICN
   - Fetches **today's noon weather forecast** from Open-Meteo
   - Appends one row to:
     - `air_quality` Feature Group
     - `weather` Feature Group

3. **Training pipeline** (`3_air_quality_training_pipeline.ipynb`)
   - Reads joined features from the Feature Store
   - Splits data into train/test based on date (train < 2025-05-01)
   - Trains an XGBoost model to predict PM2.5
   - Evaluates the model (RMSE, hindcast plots)
   - Registers the model in **Hopsworks Model Registry**  
     as `air_quality_xgboost_model` (version 1)

4. **Batch inference & monitoring** (`4_air_quality_batch_inference.ipynb`)
   - Uses the latest model from the Model Registry
   - Reads future weather forecasts and produces **PM2.5 forecasts**
   - Stores predictions in a Feature Group `aq_predictions`
   - Generates plots:
     - `pm25_forecast.png` – future PM2.5 forecast
     - `pm25_hindcast_1day.png` – 1-day-ahead hindcast (prediction vs actual)
   - Copies these images to `docs/air-quality/assets/img/`  
     so that the dashboard can display them

---

## 2. Hopsworks setup

In order to run the pipelines, you need:

1. A Hopsworks project, e.g. `id2223_lab1_yilai`.
2. A personal API key with Data Scientist permissions.
3. A local `.env` file in the project root (based on `.env.example`) containing:
   - `HOPSWORKS_API_KEY`
   - `HOPSWORKS_PROJECT`
   - `HOPSWORKS_HOST`
   - `AQICN_API_KEY`
   - AQICN sensor info (`AQICN_CITY`, etc.)

On GitHub Actions, the API key is provided via repository secret:

- `HOPSWORKS_API_KEY` → used in the workflow
  `.github/workflows/air-quality-daily.yml`.

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

- `HOPSWORKS_PROJECT`
- `HOPSWORKS_HOST`
- `HOPSWORKS_API_KEY`
- `AQICN_API_KEY`
- AQICN sensor info (`AQICN_COUNTRY`, `AQICN_CITY`, `AQICN_STREET`, etc.)

### 3.2 Run the notebooks

Start Jupyter Lab:

```bash
jupyter lab
```

Then run the notebooks in this order (kernel: `Python (id2223lab1)`):

1. `notebooks/airquality/1_air_quality_feature_backfill.ipynb`  
   - Backfills historical `air_quality` and `weather` Feature Groups.

2. `notebooks/airquality/2_air_quality_feature_pipeline.ipynb`  
   - Inserts **today’s** PM2.5 and weather into the Feature Store.

3. `notebooks/airquality/3_air_quality_training_pipeline.ipynb`  
   - Reads features from the Feature Store, trains the XGBoost model,  
     and registers it in the Hopsworks Model Registry as  
     `air_quality_xgboost_model` (version 1).

4. `notebooks/airquality/4_air_quality_batch_inference.ipynb`  
   - Loads the latest registered model, generates future forecasts and  
     hindcast plots, writes predictions to `aq_predictions`, and  
     saves images into `docs/air-quality/assets/img/`.

---

## 4. Serverless scheduling (GitHub Actions)

The **daily feature pipeline** is automated via GitHub Actions.

- Workflow file:  
  `.github/workflows/air-quality-daily.yml`

- It performs:
  1. Checkout this repository.
  2. Install Python dependencies for the air quality notebooks.
  3. Execute `notebooks/airquality/2_air_quality_feature_pipeline.ipynb`.

- Triggers:
  - `workflow_dispatch` – manual trigger from the Actions tab.
  - A `cron` schedule in the YAML file – runs the pipeline once per day.

### 4.1 GitHub secret

The workflow authenticates with Hopsworks using a repository secret:

- **Name:** `HOPSWORKS_API_KEY`  
- **Value:** the same API key used in the local `.env`.

In the workflow, it is exposed as an environment variable:

```yaml
env:
  HOPSWORKS_API_KEY: ${{ secrets.HOPSWORKS_API_KEY }}
```

---

## 5. Dashboard

The dashboard is a static site served by **GitHub Pages** from the `docs` folder.

- Root docs directory: `docs/`
- Air quality page: `docs/air-quality/index.md`
- Images generated by the batch inference notebook:
  - `docs/air-quality/assets/img/pm25_forecast.png`
  - `docs/air-quality/assets/img/pm25_hindcast_1day.png`

GitHub Pages configuration:

- Source: `Deploy from a branch`
- Branch: `main`
- Folder: `/docs`

Public dashboard URL:

> https://lawlv.github.io/id2223-lab1-airquality/air-quality/
