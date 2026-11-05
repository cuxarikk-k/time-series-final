# time-series-final
# MATH840 Final project: 48-hour electricity generation forecast

Mariana Konovalenko

The notebook forecasts the 48 hours after the end of `train.csv` (2026-10-27 01:00 to 2026-10-29 00:00), hour by hour,
with a 95% prediction interval, and writes the submission file `KONOVALENKO_project.csv`.

## Files

| File | What it is |
|---|---|
| `_project.ipynb` | the full pipeline: data, benchmark, model, validation, diagnostics, submission |
| `train.csv` | the history with the actual weather (given) |
| `weather_forecast.csv` | the weather forecast for the 48 hours (given) |
| `session1_project.csv` | my Session 1 submission, used only to draw it on the comparison plot (optional) |
| `requirements.txt` | the Python packages |
| `REPORT_FILE` | the report |
| `SLIDES_FILE` | the slides for Session 2 |

## How to run

### In Google Colab

1. Open `_project.ipynb` in Colab (File → Upload notebook, or open it from GitHub).
2. Run all cells (Runtime → Run all).
3. When the upload dialog appears, choose `train.csv` and `weather_forecast.csv` together.
   To get the green Session 1 line on the comparison plot, also upload `session1_project.csv` from the Files panel.
4. The last cell checks the format, saves `KONOVALENKO_project.csv` and downloads it.

### Locally

```bash
git clone REPO_URL
cd REPO_NAME
pip install -r requirements.txt
jupyter notebook _project.ipynb
```

Keep the CSV files next to the notebook and run all cells. The upload step is skipped when the files are already there.

The whole notebook takes about 3–5 minutes on a CPU, almost all of it in the validation loop
(20 windows × several LightGBM fits). The models use a fixed `random_state`, so the file comes out the same on every run
with the same package versions. Tested with Python 3.13, numpy 2.5, pandas 3.0, lightgbm 4.7, matplotlib 3.11.

## Method in short

- **Point forecast:** the average of two LightGBM models. One predicts `y` itself. The other predicts `y` divided by its mean over
  the previous 7 days, because the level of the series grows strongly over time. Features are the weather of the hour, the hour
  and day of year, and lags of `y`. The first day of the horizon has its own models that can use the last 24 hours;
  the second day uses lags of 48 hours and more.
- **95% interval:** the average of two intervals calibrated on the last 42 days before the forecast, separately for day and night,
  in units of the 7-day level: empirical quantiles of the point model's residuals, and conformalised quantile regression (CQR).
  Two scale factors are chosen on the validation windows.
- **Validation:** 20 windows of 48 hours, one every 4 days before the end of the history, compared with the seasonal naive benchmark.

| 20 validation windows | RMSE | MAPE | Winkler | coverage |
|---|---|---|---|---|
| seasonal naive | 273.4 | 127.3 | 1998 | 0.78 |
| final model | 207.7 | 85.9 | 1061 | 0.95 |

## Use of LLMs

I used Claude (Anthropic) as an assistant throughout the project:

- **Code.** Claude wrote most of the model code from my instructions: the LightGBM features and models, the validation loop,
  the interval calibration, the residual diagnostics and the plots. I ran it in Colab, checked the outputs and asked for changes.
- **Analysis.** Claude ran exploratory analysis on the same two data files (the trend in the level, how wind and cloud cover relate
  to generation) and comparison experiments that informed the final choice: point-model variants, interval methods,
  a test with noise added to the weather, and the Session 1 model re-scored on the same 20 windows.
- **Checking.** Claude checked the submission files and that the notebook runs from a clean folder.

The decisions on what to submit, the interpretation of the results and the report are my own.
