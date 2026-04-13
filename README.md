# Understanding the Role of Weather and Pit Strategy in Formula 1 Race Outcomes

## Project Overview
Formula 1 race outcomes are often seen as being mainly determined by car pace and qualifying performance. However, race-day conditions may also influence the final result. This project investigates whether race outcomes are explained mostly by starting grid position, or whether weather conditions and pit strategy also play a significant role.

The project combines Formula 1 race data with historical weather data and analyzes their relationship with race outcomes. The main focus is on measurable strategy variables such as pit stop timing, pit stop frequency, tire stint structure, and race-day weather conditions.

## Motivation
Formula 1 provides a good setting for studying how multiple factors interact in a competitive environment. Starting position is clearly important, but race-day uncertainty may create opportunities for drivers to gain or lose places. This project aims to quantify the relative importance of starting position, weather, and pit strategy in explaining Formula 1 race results.

## Research Questions
1. Is starting grid position strongly associated with finishing position?
2. Do wet or mixed-weather races lead to greater position changes than dry races?
3. Are pit strategy variables such as pit stop timing, number of pit stops, and stint structure associated with positions gained or lost during a race?

## Data Sources
This project uses two public data sources:

- **OpenF1 API**  
  https://openf1.org/docs/

- **Open-Meteo Historical Weather API**  
  https://open-meteo.com/en/docs/historical-weather-api

## Data Collection
Data is collected in Python through API requests.

From the **OpenF1 API**, the project uses race-related endpoints:
- `sessions`
- `starting_grid` (with fallback to qualifying `session_result` when the grid endpoint is unavailable)
- `session_result`
- `pit`
- `stints`
- `drivers`

From the **Open-Meteo Historical Weather API**, the project uses hourly historical weather variables:
- temperature
- relative humidity
- precipitation
- wind speed
- weather code

Circuit coordinates are resolved through a manual coordinate override table for all Formula 1 venues, with the Open-Meteo geocoding API as a fallback. The two datasets are merged by race session and circuit location.

## Unit of Analysis
The unit of analysis is **each driver in each race**.

## Scope of the Dataset
The project covers the **2023, 2024, and 2025 Formula 1 seasons**, producing approximately **1,300 driver-race observations** after cleaning and merging.

## Main Variables

### Race and grid variables
- `grid_position`
- `finishing_position`
- `positions_gained` (derived)
- `abs_position_change` (derived)
- `dnf`, `dns`, `dsq` flags
- `grid_source` (records whether grid came from `starting_grid` or from the qualifying fallback)

### Pit strategy variables
- `n_pit_stops`
- `first_pit_lap`
- `avg_pit_lap`
- `avg_pit_duration`
- `avg_pit_lane_duration`

### Stint and tire variables
- `n_stints`
- `avg_stint_length`
- `max_stint_length`
- `n_compounds_used`
- `tire_compounds_used`
- `first_stint_compound`

### Weather variables (race-level, hourly aggregates over the race window)
- `temperature_mean`
- `humidity_mean`
- `precipitation_sum`
- `precipitation_max`
- `wind_speed_mean`
- `weather_code_mode`
- `weather_class` (derived: `dry`, `mixed`, `wet`)

### Derived variables
- `positions_gained = grid_position - finishing_position`
- `abs_position_change = abs(grid_position - finishing_position)`

### Weather classification logic
Race weather is classified into one of three categories from hourly ERA5 reanalysis precipitation data over the race window:
- `dry`  – total precipitation below 1.5 mm or maximum hourly precipitation below 0.4 mm (removes drizzle and reanalysis noise)
- `mixed` – exactly one hour with ≥ 0.5 mm of precipitation
- `wet` – two or more hours with ≥ 0.5 mm of precipitation

These thresholds were tuned after inspecting known wet and mixed races (e.g., Zandvoort 2023, Interlagos 2024) to reduce false positives from ERA5 noise.

## Project Workflow
The project follows the full data science pipeline:

1. Data collection
2. Data cleaning and preprocessing
3. Exploratory data analysis (EDA)
4. Hypothesis testing
5. Machine learning modeling (Stage 3)
6. Interpretation and final reporting

## Stage 2 Scope
For the **14 April** milestone, the focus is on:

- data collection
- data cleaning
- exploratory data analysis (EDA)
- hypothesis testing

## Hypotheses
The project tests the following hypotheses:

### H1
**Starting grid position is positively associated with finishing position.**

### H2
**Wet or mixed-weather races lead to greater absolute position changes than dry races.**

### H3
**Pit strategy variables such as number of pit stops, first pit lap, and average stint length are significantly associated with positions gained or lost during the race.**

## Analytical Decisions

### DNF handling
Drivers who did not finish (DNF), did not start (DNS), or were disqualified (DSQ) are excluded from the analytical dataset. Their recorded finishing positions reflect mechanical failure, crashes, or penalties rather than race pace and pit strategy, which would bias the tests of all three hypotheses. DNF observations are retained in the cleaned dataset (`f1_driver_race_dataset_cleaned.csv`) for descriptive purposes, and a DNF-rate-by-weather table is produced as supporting context.

### Directional tests
H1 and H2 are framed as directional hypotheses, so they are tested with one-sided tests (positive direction for H1, and wet/mixed greater than dry for H2). H3 uses two-sided Spearman correlations because individual pit strategy variables are not directionally specified, combined with a multivariate regression.

## Statistical Methods

### For H1
- **Spearman correlation** (one-sided, positive) between `grid_position` and `finishing_position`

### For H2
- **Mann–Whitney U test** (one-sided) comparing `abs_position_change` between dry races and wet/mixed races
- `abs_position_change` is a discrete, right-skewed count-like variable, so the nonparametric test is used as the primary test
- Cohen's *d* reported as an effect-size summary

### For H3
- **Spearman correlations** between each pit strategy variable and `positions_gained`, with **Holm correction** for multiple comparisons
- **OLS regression** with HC3 robust standard errors as the multivariate test:

  `finishing_position ~ grid_position + n_pit_stops + first_pit_lap + avg_stint_length + n_compounds_used + C(weather_class)`

  The dependent variable is `finishing_position` (rather than `positions_gained`) to avoid the mechanical dependency that would arise from including `grid_position` as a regressor when `positions_gained = grid_position − finishing_position`. The regression evaluates whether pit strategy variables retain explanatory power after controlling for starting position and weather.

## Exploratory Data Analysis (EDA)
The EDA stage includes visual and descriptive analysis of the dataset:

- distribution of grid position and finishing position
- distribution of positions gained or lost
- scatter plot of grid vs finishing position with a reference line
- boxplot of absolute position change by weather class
- scatter plots of pit strategy variables vs positions gained (pit stops, first pit lap, average stint length)
- summary statistics for strategy and weather variables
- Spearman correlation matrix visualized as a heatmap

## Reproducibility
To reproduce the analysis:

1. Clone the repository
2. Install the required Python packages from `requirements.txt`
3. Run the notebooks in order:
   - `01_data_collection.ipynb`
   - `02_data_cleaning_eda_hypothesis_tests.ipynb`
   - `03_ml_modeling.ipynb` (Stage 3)

Raw data is saved to `data/raw/`, processed data to `data/processed/`, figures to `outputs/figures/`, and tables to `outputs/tables/`.

## Technologies Used
- Python
- pandas
- numpy
- requests
- matplotlib
- scipy
- statsmodels
- scikit-learn (for Stage 3)

## Expected Findings
The project expects to show that starting position explains a large part of race outcomes, while weather conditions and pit strategy also affect how much drivers gain or lose positions during a race.

## Limitations
Possible limitations of the project include:

- ERA5 reanalysis weather data approximates race-day conditions at an hourly resolution and may not capture brief local showers or track-surface conditions
- the `weather_class` classification is based on precipitation only and does not directly capture track wetness, tire choice, or safety-car-driven strategy shifts
- some strategy decisions are influenced by factors not directly observed in the dataset (team radio calls, undercut/overcut decisions, incidents)
- race incidents such as safety cars, red flags, and retirements affect both pit strategy and finishing position in ways that are not fully modeled
- DNFs are excluded from the analytical dataset to isolate the effect of pace and strategy, which means the analysis does not capture the reliability dimension of race outcomes
- the available OpenF1 data starts from the 2023 season only, which limits historical depth

## Academic Integrity and AI Use Disclosure
All external data sources used in this project are publicly available and are properly cited.

AI assistance was used in a limited and transparent way for:

- refining the wording of the project description and hypotheses
- organizing the README and analysis plan
- reviewing and debugging the data collection pipeline (identifying a stale OpenF1 field name, tuning the weather classification thresholds against known wet and mixed races, and improving output formatting)
- reviewing the statistical methodology (flagging the need to exclude DNF observations, using one-sided tests for directional hypotheses, applying Holm correction for multiple comparisons in H3, and changing the regression dependent variable to avoid a mechanical dependency with `grid_position`)

All coding was written and executed by me, and all data collection decisions, data cleaning, analysis, interpretation, and submission preparation were reviewed and finalized by me. AI suggestions were evaluated and accepted only when they were consistent with the project goals and the underlying data.

## Author
Deniz Temel  
DSA 210 – 35683 - 2026
