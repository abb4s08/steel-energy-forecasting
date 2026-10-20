# Steel Plant Energy Consumption Forecasting

Forecast the electricity used by a steel plant in the **next 15-minute interval** from current plant telemetry and recent consumption history.

The project ships a ready-to-run, weighted three-model regression ensemble. It is designed to help operations teams anticipate high-demand intervals and take action before tariff thresholds are exceeded.

## Results

The final model was evaluated on a chronological holdout set, so validation respects the time order of the observations.

| Model | MAE (kWh) | RMSE (kWh) |
| --- | ---: | ---: |
| Random Forest baseline | 4.609 | 9.507 |
| **Three-model ensemble** | **3.843** | **8.116** |

Lower values are better. The ensemble improves MAE by **16.6%** and RMSE by **14.6%** relative to the Random Forest baseline.

## Model

The final prediction is a weighted blend of three models:

| Component | Target | Weight |
| --- | --- | ---: |
| CatBoost Regressor | Next-interval energy usage | 48.75% |
| XGBoost Regressor | Next-interval energy usage | 26.25% |
| CatBoost Regressor | Change in energy usage | 25.00% |

The change model's prediction is added to the current `Usage_kWh` value before blending. Combining direct-usage and usage-change forecasts makes the model more responsive to short-term changes in plant demand.

## Dataset

`train.csv` contains sequential 15-minute readings. The target is `next_interval_Usage_kWh`.

| Field | Description |
| --- | --- |
| `date` | Timestamp of the current reading (`DD/MM/YYYY HH:MM`) |
| `Usage_kWh` | Electricity used in the current interval |
| Reactive power columns | Lagging and leading reactive load |
| Power factor columns | Lagging and leading power factor |
| `CO2(tCO2)` | Carbon-emission measurement |
| `NSM` | Seconds elapsed since midnight |
| `WeekStatus`, `Day_of_week`, `Load_Type` | Operating-calendar and load-tier categories |
| `next_interval_Usage_kWh` | Energy usage in the following 15-minute interval (training target) |

## Features

In addition to the raw telemetry, the inference pipeline creates:

- Calendar features: hour, minute, month, day of week, and weekend flag
- Cyclical time features using sine/cosine transforms of `NSM`
- Total reactive power and the difference between power factors
- Consumption lags from 1 interval through 672 intervals (one week)
- Rolling means and standard deviations over 4, 16, 96, and 672 intervals
- One- and four-interval changes in consumption

Because the longest lag is 672 readings, forecasting requires **at least 672 preceding 15-minute observations**. The included history-context file supplies this for the exported model package.

## Repository layout

```text
.
├── train.csv                                  # Labeled historical data
├── steel_energy_forecasting.ipynb             # Notebook workflow for model inference
├── Steel_Energy_Colab.ipynb                   # Google Colab inference notebook
├── Winner_to_Steal_PC/                        # Portable trained-model package
│   ├── inference.py                           # Command-line prediction script
│   ├── requirements.txt                       # Runtime dependencies
│   ├── steel_catboost_energy.cbm              # Direct CatBoost model
│   ├── steel_xgboost_energy.json              # Direct XGBoost model
│   ├── steel_change_catboost_energy.cbm       # Change-target CatBoost model
│   └── steel_history_context.csv              # Historical context for lag features
├── timing_package/                            # Copy of package used for timing tests
└── output/pdf/                                # Project summary PDF
```

## Quick start

### 1. Set up Python

Use Python 3.10–3.12. From the repository root:

```powershell
cd Winner_to_Steal_PC
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

### 2. Prepare input data

Create a `test.csv` with these columns, in chronological order:

```text
date, Usage_kWh, Lagging_Current_Reactive.Power_kVarh,
Leading_Current_Reactive_Power_kVarh, CO2(tCO2),
Lagging_Current_Power_Factor, Leading_Current_Power_Factor, NSM,
WeekStatus, Day_of_week, Load_Type
```

`test.csv` must include the current `Usage_kWh` value, but it must **not** require `next_interval_Usage_kWh`. Dates may be day-first (for example, `01/01/2018 00:15`).

### 3. Run a forecast

```powershell
python inference.py --input test.csv --output predictions.csv
```

The output contains one row per input record:

```text
forecast_for,predicted_next_interval_Usage_kWh
2018-01-01 00:30:00,3.84
```

`forecast_for` is the input timestamp plus 15 minutes.

## Google Colab

Open `Steel_Energy_Colab.ipynb` from the public GitHub repository in Google Colab. Its first cell clones the repository and extracts `Winner_to_Steal_PC.zip` automatically—**no Google Drive upload is required**. The notebook then installs the dependencies, loads the exported artefacts, validates engineered inputs, and runs the same weighted ensemble.

Keep the repository public and make sure it contains `Steel_Energy_Colab.ipynb`, `Winner_to_Steal_PC.zip`, and `train.csv`. To score new data, upload only the new input CSV to the temporary Colab session.

## Notes and limitations

- Keep rows sorted chronologically. Lag and rolling features depend on correct time order.
- The supplied history context is model-specific. For production forecasting, refresh the context with the most recent verified 672 readings before scoring a new batch.
- Reported metrics are local chronological-holdout results; performance on a hidden or future test period may differ.
- This repository contains trained artefacts and inference workflows. Re-training code and full experiment tracking are not included.

## Dependencies

`pandas`, `numpy`, `scikit-learn==1.6.1`, `catboost`, `xgboost`, and `joblib`.

## License

No license has been specified. Add a license file before publishing if you want others to reuse or modify the project.
