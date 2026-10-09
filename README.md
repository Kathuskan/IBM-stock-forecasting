# IBM Stock Price Forecasting with Databricks

A learning project using Databricks, PySpark, and Python to investigate IBM stock prices and evaluate one-trading-day-ahead forecasts.

The main finding: the previous-close naïve forecast outperformed three small ARIMA candidates on validation. A historical-quantile interval method did not maintain its intended 90% coverage on the final test period.

## Objective

After each trading day closes, predict the next recorded trading day's closing price using only information available at the forecast origin. Evaluate both point accuracy and prediction-interval coverage.

## Data

- Reported source: Yahoo Finance; downloaded CSV.
- File: `IBM_stock_history.csv`.
- Coverage: 2 January 1962–27 August 2026.
- Records: 16,272.
- Columns: Date, Open, High, Low, Close, Adj Close, Volume, Dividends, Stock Splits.
- Databricks table: `workspace.default.ibm_stock_raw`.
- Target: the supplied `Close` series.

Exact download settings and price-adjustment conventions remain unconfirmed. Returns calculated from `Close` are not asserted to be dividend-inclusive total returns. The raw dataset is not included in this repository.

## Repository structure

```text
ibm-stock-forecasting/
├── README.md
├── .gitignore
└── notebooks/
    └── 01_explore_ibm_stock.ipynb
```

## Analytical workflow

1. Inspect column types, dates, missing values, duplicates, and price consistency.
2. Define chronological training, validation, and test periods.
3. Explore price, log price, log returns, and rolling volatility.
4. Assess stationarity using ADF and KPSS tests.
5. Inspect ACF/PACF of returns and ACF of squared returns.
6. Establish the previous-close naïve benchmark.
7. Compare ARIMA(1,1,0), ARIMA(0,1,1), and ARIMA(1,1,1) on log prices.
8. Examine year-specific accuracy and forecast-error diagnostics.
9. Evaluate rolling historical-quantile intervals.
10. Freeze the selected procedure, evaluate the final test period, and audit extreme errors.

No clear seasonal structure was established from the inspected plots. Seasonal differencing was not imposed.

## Evaluation design

| Period | Target dates |
|---|---|
| Training | Before 1 January 2023 |
| Validation | 2023–2024 |
| Final test | 1 January 2025 onward, through the dataset endpoint |

ARIMA parameters were estimated on training-period log prices without drift and held fixed during validation. Model states were updated after each actual observation became available. Exponentiated log forecasts were evaluated as model-implied median price forecasts. MAE was the primary metric; RMSE was secondary.

Each forecast preceded its target observation. Earlier validation or test outcomes could inform later forecasts, matching sequential daily use. No final-test data informed initial model selection.

## Validation results

Results cover 502 forecasts across 2023–2024. Errors are in the supplied price units.

| Model | MAE | RMSE |
|---|---:|---:|
| Naïve: previous close | **1.4826** | **2.2696** |
| ARIMA(1,1,0) | 1.4843 | 2.2704 |
| ARIMA(0,1,1) | 1.4844 | 2.2705 |
| ARIMA(1,1,1) | 1.4867 | 2.2727 |

The naïve forecast also had the lowest MAE and RMSE in each validation year separately. Differences were small and were not established as statistically distinguishable. Ljung–Box tests on naïve validation errors did not reject zero autocorrelation through lags 10 or 20.

## Prediction intervals

For each target day, the method used the preceding 252 log returns. Their 5th and 95th percentiles were exponentiated and multiplied by the previous close to obtain candidate 90% prediction limits. The target day's return was excluded.

| Period | Coverage | Average width |
|---|---:|---:|
| Validation overall | 90.64% | 6.1859 |
| 2023 | 96.00% | 5.4804 |
| 2024 | 85.32% | 6.8858 |

Near-nominal overall coverage concealed substantial variation between years.

## Final test results

Only the validation-selected naïve point forecast and the frozen interval procedure were evaluated on the final test period.

| Period | Forecasts | MAE | RMSE | Interval coverage | Average width |
|---|---:|---:|---:|---:|---:|
| Overall: 2 Jan 2025–27 Aug 2026 | 414 | 4.4037 | 7.2174 | 83.33% | 13.0875 |
| 2025 | 250 | 3.3719 | 4.9187 | 87.20% | 11.8991 |
| 2026 through 27 August | 164 | 5.9766 | 9.7271 | 77.44% | 14.8991 |

The intervals undercovered despite becoming wider. The ten largest errors contributed approximately 54.49% of total squared error. Extreme observations were retained; no observations were removed merely to improve scores.

## Run in Databricks

1. Obtain the source CSV separately. Matching the reported results requires the same data snapshot and adjustment conventions.
2. Import it as `workspace.default.ibm_stock_raw`, or update the table reference in the notebook.
3. Import `notebooks/01_explore_ibm_stock.ipynb` into Databricks.
4. Attach compatible Python compute with PySpark, pandas, NumPy, Matplotlib, and statsmodels available.
5. Run cells in order. The notebook uses `result_object=False` in ADF/KPSS calls, requiring a statsmodels version supporting that argument, such as 0.15.0.

The original Delta table is read without being overwritten. The single-stock dataset is collected into pandas for statistical analysis and plotting; this approach is not intended for datasets too large for driver memory.

Exact runtime and dependency versions were not captured in the reported experiment. Record them when reproducing it.

## Limitations and further work

- Results are conditional on source-data accuracy and unconfirmed adjustment conventions.
- The study covers one stock and one chronological evaluation arrangement.
- ARIMA comparisons used full-history training, no drift, and fixed parameters.
- Passing statistical diagnostics does not prove independence or forecastability.
- The interval method cannot be described as reliably calibrated to 90% coverage.
- Price-unit error comparisons across periods also reflect changes in price level.
- The test period has now been examined. Revisions informed by its outcomes require fresh evaluation data for an independent assessment.

Possible further experiments include rolling training windows, scheduled parameter refitting, and volatility-aware intervals. These were not implemented in the reported experiment.
