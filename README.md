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
```bash

Create a .env file in the project root based on .env.example and fill in:
- HOPSWORKS_PROJECT
- HOPSWORKS_HOST
- HOPSWORKS_API_KEY
- AQICN_API_KEY
AQICN sensor info (AQICN_COUNTRY, AQICN_CITY, AQICN_STREET, AQICN_URL, etc.)

### 3.2 Run the notebooks
