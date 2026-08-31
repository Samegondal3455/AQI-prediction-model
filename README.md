# Sargodha AQI Prediction Model

A 3-day Air Quality Index (AQI) forecasting system for Sargodha, Punjab. The project retrieves historical weather and air-quality data from Open-Meteo, creates daily time-series features, trains one Gradient Boosting model per forecast horizon, stores improved models in the Hopsworks Model Registry, and serves live predictions through Streamlit.

## What The System Does

- Fetches historical weather and AQI data for Sargodha.
- Aggregates hourly data into daily records.
- Builds lag, rolling-window, calendar, and future-weather features.
- Trains separate models for tomorrow, two days ahead, and three days ahead.
- Compares new metrics with existing Hopsworks model versions.
- Uploads a new model version only when the new model improves the registered model.
- Fetches live weather, AQI, and forecast weather in the Streamlit app.
- Displays 3-day predictions and air-quality alerts.

## Architecture

```text
Open-Meteo Archive APIs
        |
        v
retrain.py: raw hourly data
        |
        v
Daily aggregation + feature engineering
        |
        v
Three GradientBoostingRegressor models
        |
        +--> local retrain_artifacts/
        |
        +--> Hopsworks Model Registry
                    |
                    v
              app.py / Streamlit
                    |
                    v
          Live 3-day AQI forecast and alerts
```

## Repository Structure

```text
.
├── app.py                          # Streamlit application
├── retrain.py                      # Data, feature, training, and registry pipeline
├── requirements.txt                # Application dependencies
├── requirements-retrain.txt        # Pinned retraining dependencies
├── new.ipynb                       # Notebook used during model development
└── .github/workflows/retrain.yml   # Scheduled and manual GitHub Actions workflow
```

Generated files are intentionally ignored by Git:

- `sargodha_raw_data_3yrs (5).csv`: downloaded historical raw dataset.
- `sargodha_features_daily_v2.csv`: generated daily feature dataset.
- `retrain_artifacts/`: local model and feature artifacts.
- `.env`: local secrets file.

## Requirements

- Python 3.11 or a compatible Python 3 version.
- Internet access for Open-Meteo APIs.
- A Hopsworks account and API key for training/uploading models and running the app predictions.
- Hopsworks model versions expected by the current app:
  - `sargodha_aqi_gbr_day1`, version `4`
  - `sargodha_aqi_gbr_day2`, version `3`
  - `sargodha_aqi_gbr_day3`, version `3`

## Local Installation

Create and activate a virtual environment:

```bash
python3.11 -m venv .venv
source .venv/bin/activate
```

Install dependencies for the Streamlit app:

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

For retraining, use the pinned retraining dependencies instead of relying on unpinned application packages:

```bash
python -m pip install -r requirements-retrain.txt
```

Check the installation:

```bash
python -c "import hopsworks, joblib, numpy, pandas, requests, sklearn; print('Dependencies loaded successfully')"
python -m pip check
```

## Hopsworks Configuration

The retraining script reads `HOPSWORKS_API_KEY` from the environment or from a local `.env` file. Do not commit the key.

Option 1, environment variable:

```bash
export HOPSWORKS_API_KEY="your-hopsworks-api-key"
```

Option 2, local `.env` file in the project root:

```dotenv
HOPSWORKS_API_KEY=your-hopsworks-api-key
```

The application accepts the Hopsworks key through the password field in the Streamlit sidebar, supports `.env` values, and also reads the key from Streamlit Community Cloud secrets via `st.secrets`. The key is used to log in to the `colab` project at `eu-west.cloud.hopsworks.ai`.

### Streamlit Community Cloud setup

In Streamlit Community Cloud, add the secret in the app dashboard (Settings → Secrets):

```toml
HOPSWORKS_API_KEY = "your_hopsworks_api_key_here"
```

A working local/example file can also be created at `.streamlit/secrets.toml` or `.streamlit/secrets.toml.example`.

## Run The Streamlit Application

```bash
streamlit run app.py
```

Open the URL printed by Streamlit, normally `http://localhost:8501`.

Application flow:

1. Enter the Hopsworks API key in the sidebar.
2. Confirm that Day 1, Day 2, and Day 3 models load successfully.
3. Review live weather and AQI values.
4. Click `Run 3-Day AQI Forecast`.
5. The app fetches the latest 45 days of daily history and up to 5 days of forecast weather.
6. Predictions and an AQI alert are displayed.

The app uses fallback values for the live current-weather cards when an API request fails. Forecast execution still reports an error when required historical or model data is unavailable.

## Run Retraining Locally

From the project root:

```bash
python retrain.py
```

Retraining flow:

1. Checks for `HOPSWORKS_API_KEY`.
2. Logs into the Hopsworks `colab` project.
3. Loads `sargodha_raw_data_3yrs (5).csv` when present.
4. Otherwise downloads approximately three years of data from Open-Meteo and saves the CSV.
5. Resamples hourly records to daily records.
6. Adds calendar features, AQI/PM2.5/temperature lags, rolling AQI statistics, future weather features, and three target columns.
7. Splits the data chronologically into 80% training and 20% test sets.
8. Trains three `GradientBoostingRegressor` models.
9. Calculates RMSE, MAE, and R2 for each horizon.
10. Compares the new metrics with all existing versions of the corresponding Hopsworks model.
11. Uploads `model.pkl` and `features.pkl` only when the new metrics are better.

Model names:

```text
sargodha_aqi_gbr_day1
sargodha_aqi_gbr_day2
sargodha_aqi_gbr_day3
```

A successful run may print either `new model version(s) registered` or `No model versions were registered because no improvement was found.` The second message is not a failure; it means the existing registered models performed better.

## GitHub Actions Retraining

The workflow is defined in `.github/workflows/retrain.yml` and supports:

- Daily schedule at `02:00 UTC`.
- Manual execution through the GitHub Actions `Run workflow` button.

Before running it, add these repository secrets under **Settings > Secrets and variables > Actions**:

```text
HOPSWORKS_API_KEY
GMAIL_USERNAME
GMAIL_APP_PASSWORD
```

`HOPSWORKS_API_KEY` is required by the retraining job. The Gmail values are required by the notification job, which runs after the retraining job whether it succeeds or fails.

The workflow installs `requirements-retrain.txt`, verifies the imports, and then runs:

```bash
python retrain.py
```

If the Actions page shows no runs, open the workflow and use `Run workflow` on the `main` branch. If the button is unavailable, confirm that the workflow file on GitHub contains `workflow_dispatch: {}` and that you have permission to run workflows.

## Data Sources

- Historical weather: Open-Meteo Archive API.
- Historical AQI and pollutants: Open-Meteo Air Quality API.
- Live weather and forecast weather: Open-Meteo Forecast API.

Sargodha coordinates used by the project:

```text
Latitude: 32.0836
Longitude: 72.6711
```

## Feature and Prediction Contract

The retraining pipeline saves the exact feature list used for each model in `features.pkl`. The Streamlit app loads that list and constructs the input vector in the same order.

Features include:

- Daily weather and pollutant aggregates.
- Calendar features and cyclical seasonal features.
- AQI, PM2.5, and temperature lag values.
- AQI rolling mean, maximum, minimum, and standard deviation over 3, 7, 14, and 30 days.
- Future daily temperature, humidity, and wind values for the relevant horizon.

The application and retraining pipeline must remain aligned. If feature names or feature construction changes, retrain and register new model versions before using the application.

## Troubleshooting

### `HOPSWORKS_API_KEY is required in the environment`

Set the environment variable or create `.env` in the project root. In GitHub Actions, add `HOPSWORKS_API_KEY` as a repository secret.

### `Could not open requirements file: req.txt`

The correct file names in this repository are `requirements.txt` and `requirements-retrain.txt`. The retraining workflow must install:

```bash
python -m pip install -r requirements-retrain.txt
```

### Dependencies import locally but fail in Actions

Use the pinned file for retraining:

```bash
python -m pip install -r requirements-retrain.txt
```

Avoid mixing a globally installed Hopsworks version with the pinned retraining environment. Check the active versions with:

```bash
python -m pip show numpy pandas scikit-learn hopsworks
python -m pip check
```

### No model versions are registered

This can be a normal result. The pipeline uploads only when the new model improves the existing model according to RMSE/R2 comparison. Check the printed metrics and the Hopsworks Model Registry.

### Models do not load in Streamlit

Confirm the API key, Hopsworks project, model names, and model versions. Each downloaded artifact must contain a model file and `features.pkl`.

### Open-Meteo request fails

Check internet access, API availability, date ranges, and the returned API response. The retraining script retries transient HTTP failures, but it cannot recover from a permanently unavailable API or invalid response.

## Validation Commands

```bash
python -m py_compile app.py retrain.py
python -m pip check
python -c "import hopsworks, joblib, numpy, pandas, requests, sklearn; print('Dependencies loaded successfully')"
```

A real end-to-end retraining run additionally requires a valid Hopsworks API key and network access.

## Security Notes

- Never commit `.env`, Hopsworks API keys, Gmail passwords, or downloaded private artifacts.
- Use a Gmail App Password for the notification workflow, not the normal Gmail account password.
- Rotate exposed keys immediately.
- Keep API keys in GitHub Actions secrets or local environment variables only.
