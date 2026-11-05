# Report: 48-hour electricity generation forecast
## Summary (3–4 sentences)

- Task: hourly forecast for 2026-10-27 01:00 – 10-29 00:00, with a 95% interval; benchmark: seasonal naive.
- Final model: two LightGBM models (absolute and level-normalised target), separate models for day 1 and day 2,
  interval = average of residual quantiles and CQR, both scaled by the recent level.
- On 20 validation windows: RMSE 208 vs 273, MAPE 86 vs 127, Winkler 1061 vs 1998, coverage 0.95 vs 0.78.
- My Tuesday model beat snaive on the 20 validation windows, but on the public leaderboard it lost on all three metrics and placed 29th. The main reason was the interval, which was too wide: Winkler 917 vs 521 for the benchmark, while RMSE and MAPE were only slightly worse. On the same 20 validation windows, my final model beats my first model on all three metrics (RMSE 219 → 208, MAPE 96 → 86, Winkler 1190 → 1061), and its interval is almost half as wide (mean width 917 → 513).

## 1. What is in the series?

**Facts** (EDA cell):
- 33 480 hourly values, 2023-01-01 → 2026-10-27, no gaps; `y > 0` everywhere (min 0.7), so MAPE is defined.
- Daily cycle: ~170 at night, ~650 at 12–13h (mean by hour).
- The level is not stable. Monthly mean ~150–200 in 2023–2024, ~250–410 in 2025, up to ~900 in Aug–Sep 2026,
  675 in Oct 2026, and the last week of October drops to ~250–400 per day.
- Night generation is not zero and grows with wind: since April 2026, night mean ~150 at 0 m/s, ~310 at 3 m/s, ~440 at 5 m/s.
- `cloud_cover` behaves the opposite way to its description: at 11–14h, mean y ~960 at 0 and ~2000 at 1.
  The series is disguised, so I take the relation from the data and do not interpret it physically.
- Correlations with y: daylight 0.40, humidity −0.38, wind 0.30, temperature 0.28, cloud 0.16.
- Strong persistence: the daily mean has autocorrelation 0.89 (lag 1) and 0.83 (lag 2). This is why seasonal naive is hard to beat.

**How it shaped the model:**
- Growing level → trees cannot extrapolate it, so one model predicts `y / (mean of the last 7 days)` and multiplies back.
- Persistence → lags of y matter most. Day 1 may use the last 24 hours (lag 24), day 2 only lag ≥ 48, so each day has its own models.
- Wind drives the night level → weather features stay in.
- The most significant finding is that the series level fluctuates considerably: in September, generation was three times higher than in late October. This negatively affected both the forecast and the interval. Tree-based models (like LightGBM) cannot predict values ​​higher than those observed during training. Furthermore, the interval—calibrated based on the "high" levels seen in September—ended up being too wide for the "low" levels in October. For this reason, the final model performs all calculations relative to the level observed over the last week.

## 2. How did you check the forecast before submitting it?

- The template window (last 48 hours) alone is misleading: naive RMSE 120 there vs 273 on average over 20 windows.
- Session 1: 10 windows, one every 5 days. Final: 20 windows of 48 hours, one every 4 days (Aug–Oct 2026),
  each starting at the same hour as the real forecast.
- No leakage: every feature uses only y known at the forecast origin; models and interval calibration use only data before the origin.
- Results on the same 20 windows:

| | RMSE | MAPE | Winkler | coverage |
|---|---|---|---|---|
| seasonal naive | 273.4 | 127.3 | 1998 | 0.78 |
| Session 1 model | 218.9 | 95.6 | 1190 | 0.87 |
| final model | 207.7 | 85.9 | 1061 | 0.95 |

- Day 2 only (the private part): the final model beats naive in 16/20 windows on RMSE, 15/20 on Winkler, 13/20 on MAPE.
- Public leaderboard (first 24 hidden hours), Session 1 file: RMSE 122.1 vs 115.7, MAPE 86.5 vs 84.3, Winkler 917 vs 521.
  [write: what this taught you; that one calm day is a single window, and why you did not tune to it]
- Weather forecast: With actual weather RMSE 200; with noise added to the weather 229; without weather 227; naive 250.
  So the forecast helps only if it is accurate. [write: your decision and reason]
- Limitations: validation uses the actual weather (optimistic); the two interval scales are chosen on the same windows.

## 3. How good is your interval?

- Session 1: quantile LightGBM + CQR in absolute units, 28-day calibration, scaled by 0.8.
  It was calibrated on September, when the level was ~3× higher, so in late October it was too wide:
  mean width 917 on day 1, which was exactly the public Winkler (coverage 100%).
- Final: two intervals in units of the 7-day level, calibrated on the 42 days before the origin, separately for day and night:
  1. empirical 2.5% / 97.5% quantiles of the point model's residuals;
  2. CQR: quantile LightGBM (2.5%, 97.5%) widened by the 95% quantile of its calibration errors.
  The final interval is their average, with scales `s_res`, `s_cqr` chosen on a grid.
- Before scaling (s = 1): Winkler 1100, coverage 0.92. After: Winkler 1061, coverage 0.953 (`s_res` 0.8, `s_cqr` 1.4).
- The optimum is flat: the top 8 settings give Winkler 1061–1071 → not tuned to noise.
- By group (diagnostics table): coverage 0.93 by day, 0.97 at night; mean width ~840 on day 1, ~920 on day 2.
- Mean width on the forecast day 1 fell from 917 (Session 1) to 513.
- [write: is the interval good enough, where it is weakest]

## 4. What is left, and what did not work?

**Residuals of the final model** (diagnostics cell):
- Bias: under-forecast by day (+40 on average, most at 8–14h), over-forecast in the evening and at night (−20 to −50 at 18–23h).
- RMSE by hour: ~300 at the daytime peak, ~160 at night.
- By horizon: the gain over naive is mostly at night and in the evening (~130–190 vs ~250–330);
  at the daytime peaks (10–15 and 34–40 hours ahead) the model is close to naive.
- Autocorrelation: 0.77 at lag 1, back to ~0 by lag 8, ~0.28 at lag 24 → errors come in whole days, not single hours.
- [write: what this means: the model does not know the level of the next day in advance; the daytime peaks are what is left to improve]

**Tried and dropped:**

| approach | result | why dropped |
|---|---|---|
| lags ≥ 48 for the whole horizon (Session 1) | first day cannot see the last day | separate day-1 model: skill 0.803 → 0.793 |
| blend with seasonal naive (final model) | RMSE 205 → 208, MAPE 120 → 135 on day 2 | worse on average |
| log target (Session 1) | MAPE 79 → 60, RMSE 249 → 302 | RMSE too much worse |
| separate interval scale for day and night (Session 1) | skill 0.803 → 0.802 | no gain |
| train only on the last year | RMSE 204 → 204 | no gain |
| no weather features | RMSE 200 → 227 | weather helps |
| absolute-unit interval (Session 1) | too wide after a level change | replaced with level-scaled interval |

- [write: what you would do next: e.g. a better daily level estimate, a model for the daytime peak]

## What I would do differently

- [write: 2–3 points, e.g. validate on many windows from the start; scale the interval with the level from the start]
