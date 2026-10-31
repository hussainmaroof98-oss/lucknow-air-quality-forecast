# lucknow-air-quality-forecast
# Lucknow PM2.5 Forecasting (Gomti Nagar station)

## Goal
Forecast hourly PM2.5 (fine dust) at Gomti Nagar, Lucknow using past pollution
and weather data. This is a supervised regression problem built to practise the
full ML life cycle.

## Data
- Source: OpenAQ API v3 (free API key), station "Gomti Nagar, Lucknow - UPPCB" (location 7140)
- Period: 18 Feb 2025 to end of Sep 2026, hourly. Timestamps are in UTC (India = UTC + 5:30)
- Variables: pm25 (target), pm10, no2, o3, temperature, humidity, wind_speed, wind_direction
- Tools: Python, pandas, numpy, matplotlib, OpenAQ Python SDK, Google Colab

## Folder structure
- data/raw: files exactly as downloaded (never edited)
- data/processed: cleaned file (gomti_nagar_hourly_clean.csv)
- notebooks: Colab notebooks
(The API key is stored in Colab Secrets and is never committed.)

## What has been done

### 1. Data collection
Found all Lucknow stations with the OpenAQ API, checked which sensors had recent,
continuous history, and chose Gomti Nagar (highest coverage, with its own weather
sensors). Downloaded hourly data for 8 variables and saved each as raw CSV.

### 2. Data understanding
- 13,034 rows were present, but a full hourly timeline has 14,111 hours, so
  1,077 hours had no row at all (the station was offline). Plain null counts cannot see these.
- PM2.5 is right-skewed: median 44.5, mean 50.7, 75% of hours under 68.3.
- Found 284 separate gaps in PM2.5; the longest is 242 hours (about 10 days).

### 3. Missing values
- Built the full hourly timeline and reindexed, so every missing hour became a visible blank.
- Mean/median/mode fill was rejected for this data: a global value put inside a
  pollution episode creates fake readings. Time-series data should be filled from neighbouring hours.
- Used linear interpolation (equal steps between the values before and after the gap),
  only for gaps of 6 hours or less. Longer gaps are left blank, because filling them
  would invent data.
- wind_direction is not interpolated (it is an angle: 350 and 10 degrees are close).

### 4. Outliers and sensor faults
Not every high value is wrong (real winter episodes are kept), so each suspicious
peak was checked with:
- build-up and fall: real episodes rise and fall over several hours; faults jump in and out
- repeated identical values (a frozen sensor)
- PM2.5 should never be higher than PM10 (PM2.5 is a subset of PM10)
- other pollutant and weather columns in the same hours
- comparison with three other Lucknow stations (Lalbagh, Talkatora, Ambedkar University)

Removed windows (UTC), 28 hours in total:

| Window | What was seen | Decision |
|---|---|---|
| 2026-08-02 05:30 to 13:30 | PM2.5 406 to 683, PM10 stuck at 686, other stations 12 to 49 | Fault |
| 2026-09-25 06:30 | one hour of 672 among near-zero readings | Fault |
| 2025-05-26 14:30 to 23:30 | stuck near 384 for 10 hours | Fault |
| 2025-08-01 02:30 | lone 478 after 6 blank hours | Fault |
| 2025-09-24 02:30 to 04:30 | 454, blank, 584; neighbours normal | Fault |
| 2026-05-30 12:30 to 13:30 | 484 and 447; three stations calm | Fault |
| 2025-09-23 18:30 to 19:30 | 112 and 251, only one neighbour also spiked | Removed as unconfirmed (judgement call) |

Kept as real: the evening episode of 27 Apr 2025 (up to 359), because it rose and fell
gradually and other stations rose at the same hours.

### 5. Order of cleaning (lesson learned)
Remove bad readings first, then fill short gaps. Doing it the other way round filled
a gap using two fake readings next to it.

### 6. Result
Blank PM2.5 hours: 1,330 raw, 1,356 after fault removal, 894 after filling short gaps
(about 6% of the timeline). Output file: data/processed/gomti_nagar_hourly_clean.csv
with Indian-time columns (time_ist, hour_ist).

## Limitations and open items
- Only about 19 months of data, so just one full winter.
- Long gaps (up to 242 hours) are left blank.
- 16 hours with PM2.5 higher than PM10 are not yet handled.
- Neighbour stations also contain bad readings, so the comparison is evidence, not proof.
- Two removals (23 Sep 2025 and 30 May 2026) are judgement calls.
- Scaling and encoding are not done yet.

## Next steps
EDA on the clean data, time features (hour, month, lags, rolling averages),
time-based train/test split, scaling (fit on training data only), multiple regression
baseline, Ridge/Lasso, then tree models and evaluation (MAE, RMSE, R2).
