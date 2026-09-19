Predicting Diesel Prices at French Fuel Stations Using XGBoost




Project overview



This project develops a supervised Machine Learning model to predict the diesel price at French fuel stations. 
The flagship algorithm is XGBoost Regressor, as required by the project brief. A lightly configured Random Forest is included as a reference baseline model.

The project addresses the following research question:
Can the diesel price at a French fuel station be predicted from its location, station characteristics and prior-day oil market and geopolitical conditions, using only information available before the prediction date?

The task is a regression problem because the target is a continuous value expressed in euros per litre. Unlike a purely cross-sectional model, this version follows a temporal prediction approach: the model is trained on early-2025 observations and evaluated on a later, unseen period, so that features are restricted to information that would genuinely have been available before the prediction date.




Phase 1 — Topic selection and data acquisition


The selected topic is diesel-price prediction in France over 2025. Two station-level source files are used:
    • prices_2025.csv — one row per station, fuel type and timestamp, giving the recorded price (Station ID, Fuel, Date, Price).
    • stations_2025.csv — one row per station, giving station characteristics (Station ID, Location Type, Postal Code, Latitude, Longitude, 24/7 Payment Terminal).
    
Two external datasets are added to enrich the station data:
    • RBRTEd.xls — daily Brent crude oil price (USD per barrel), sheet "Data 1".
    • data_gpr_daily_recent.xls — daily Geopolitical Risk Index (GPRD), sheet "Sheet1".

The raw source files are stored in:
data/raw/prices_2025.csv
data/raw/stations_2025.csv
data/raw/RBRTEd.xls
data/raw/data_gpr_daily_recent.xls



Phase 2 — Data cleaning


The cleaning workflow is applied to the price and station tables independently before they are combined:
    • removes price records with a missing fuel type, date or price, since they cannot be used for price analysis;
    • converts postal codes to 5-character strings to preserve leading zeros;
    • converts the Date column to a proper datetime type;
    • checks for exact duplicate rows and duplicate station–fuel–date combinations, and removes isolated inconsistent records after inspecting their temporal and cross-sectional context (for example, a single conflicting E10 observation and an isolated €3.00/L diesel outlier);
    • fills missing "24/7 Payment Terminal" values with 0, since the source only flags stations where the service is available;
    • derives a Department code from the first two digits of the postal code for exploratory geographic analysis.
The cleaned data remain in memory for the exploratory and modeling notebook; no separate translated export is produced at this stage.



Phase 3 — Exploratory Data Analysis (EDA)


The EDA covers:
    • the price distribution of each fuel type (diesel/Gazole, SP95, SP98, E10, E85, GPLc);
    • the daily median price evolution of each fuel type over 2025;
    • the correlation between fuel types, showing that diesel moves closely with E10, SP98 and SP95;
    • diesel price differences by station Location Type (highway stations priced notably higher than road stations);
    • diesel price differences by 24/7 payment terminal availability (a small effect);
    • diesel price differences by department (roughly €0.20/L spread between the cheapest and most expensive areas).
These findings motivate the choice of predictors used later: station location type, geographic coordinates, and temporal features.



Phase 4 — External data enrichment


Two external indicators are added to the diesel observations:
    • Brent crude oil price — reindexed to a complete daily calendar for 2025, with non-trading days (weekends, holidays) forward-filled from the last available price.
    • Geopolitical Risk Index (GPR) — the daily GPRD series is used as-is, since it already has complete 2025 coverage.
Both series are visualized over the year: Brent shows an overall downward trend with short-term rebounds, while GPR is more volatile, with several spikes around mid-year.



Phase 5 — Feature engineering


The modeling dataset is restricted to diesel (Gazole) observations, merged with station characteristics and the two external indicators on a normalized daily date key. From this base, the following features are built:
    • temporal features: Month, Week, Day of Week and Day of Year;
    • a binary Is Highway indicator derived from Location Type (A = highway, R = road);
    • one-day-lagged external features, Brent Lag 1 and GPR Lag 1, so that only information available before the prediction date is used — same-day Brent and GPR values are excluded from the model to avoid temporal leakage.
A correlation analysis against the target shows Is Highway (0.47) and Brent Lag 1 (0.33) as the strongest linear associations, and that Month, Week and Day of Year are highly collinear (0.95–1.00). Month and Week are therefore dropped, keeping Day of Year.
The final predictor set is: Latitude, Longitude, 24/7 Payment Terminal, Is Highway, Day of Week, Day of Year, Brent Lag 1, GPR Lag 1. The target Diesel Price is never included as a feature.



Phase 6 — Chronological train/test split and model development


Because the goal is to predict future diesel prices rather than interpolate within a single snapshot, the data are split chronologically instead of randomly:
    • January–September 2025: development (model training during tuning);
    • October 2025: validation (used to compare models and select hyperparameters);
    • November–December 2025: final test period, kept completely unseen until the last evaluation.
The modeling workflow follows the required progression:
    • a Random Forest Regressor (50 trees, max depth 15) provides a baseline, trained on the development period and scored on October;
    • an initial XGBoost model (200 trees) is trained on the same development period and already outperforms the Random Forest baseline on October;
    • Randomized Search explores five XGBoost hyperparameters (n_estimators, learning_rate, subsample, colsample_bytree, reg_lambda) using a predefined chronological train/validation split (Jan–Sep vs. October) rather than standard k-fold cross-validation, since the data are temporal;
    • Grid Search refines the region identified by Randomized Search;
    • Manual Search tests a few targeted configurations around the Grid Search optimum, adjusting the number of trees, feature sampling and regularization.
The selected configuration is n_estimators = 1200, learning_rate = 0.1, subsample = 0.9, colsample_bytree = 0.6, reg_lambda = 3, reaching a validation MAE of about 0.0374 €/L on October.



Phase 7 — Final model and results


The tuned XGBoost model is retrained on the full January–October training period and evaluated once on the unseen November–December test period, using:

    • R², the proportion of target variance explained by the model;
    • MAE, the mean absolute error in euros per litre;
    • RMSE, which penalizes larger errors more strongly.
    
Results:

    • Random Forest baseline (October validation): MAE 0.0487 €/L, RMSE 0.0614 €/L, R² 0.509;
    • Initial XGBoost (October validation): MAE 0.0380 €/L, RMSE 0.0502 €/L, R² 0.672;
    • Final tuned XGBoost (November–December test): MAE 0.0507 €/L, RMSE 0.0639 €/L, R² 0.596;
    • Final tuned XGBoost (training set, for comparison): MAE 0.0303 €/L, R² 0.835.
    
The gap between training and test performance points to some overfitting, but the test period also shows a genuine shift in diesel prices — the monthly median rose from €1.602/L in October to €1.679/L in November before falling back to €1.599/L in December — which also limits achievable test performance.

Feature importance places Is Highway far ahead of the other predictors (about 53% of total importance), followed by Day of Year and Brent Lag 1; latitude and longitude also contribute meaningfully, while GPR Lag 1 and Day of Week matter comparatively little. These are predictive contributions within the model, not causal effects.



Data leakage and scope limitation


The target Diesel Price is never used as an input feature. Same-day Brent and GPR values are deliberately excluded and replaced with one-day lags, and the train/validation/test split is strictly chronological, so no future information reaches the model at training or tuning time.
The dataset covers a single year (2025), so the model captures within-year seasonal and cross-sectional patterns rather than multi-year trends. A future version could incorporate several years of data, additional lagged market features, and station-level historical price features.


Deliverables

Deliverable
Location
Project documentation
README.md
Modeling notebook
Projet_ML_Tina_2.ipynb
Raw price data
data/raw/prices_2025.csv
Raw station data
data/raw/stations_2025.csv
Raw Brent crude oil data
data/raw/RBRTEd.xls
Raw GPR data
data/raw/data_gpr_daily_recent.xls


The Google Slides presentation should contain the 11-slide structure required by the brief: title, project overview, two data-preparation and feature-engineering slides, three model-building and tuning slides, two key-findings slides, future work, and closing. Add the presentation link here once the deck has been created.


Reproducibility

Install the required packages:
pip install -r requirements.txt
Open and run the notebook:
Projet_ML_Final_Version.ipynb


Coding practices

The notebook uses descriptive English variable names, explicit comments for important transformations, logically ordered sections (loading, cleaning, EDA, external enrichment, feature engineering, chronological train/test split, baseline and XGBoost modeling, tuning, final evaluation, feature importance, conclusion), and a fixed random seed (17) for reproducibility.
