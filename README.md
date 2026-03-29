# Datathon-Team133 

Objective: Develop a model to predict intraday contact center inbound call volume at the portfolio level. Forecast data shouldaccount for 24/7 servicing in 30-minute intervals for each day in a month. Historical data at daily & interval levels hasbeen provided for 4 portfolios of varying sizes within the Diversified & Value platform.

This repository contains our end-to-end workflow for the datathon submission: cleaning the raw Excel workbook, standardizing the interval timestamps, filling missing values with a SARIMAX/Kalman-based approach, and generating forecasts for the submission template with XGBoost.

## Authors
- **Vinay Krishnan** — vinayek2@illinois.edu
- **Rahiya Rasheed** — rrashe2@illinois.edu

## Project Overview
Our workflow is organized in three main stages:

1. **UTC/time processing** to turn the interval sheets into a complete 30-minute timestamp series.
2. **Missing-value imputation** using SARIMAX with Kalman smoothing, with interpolation/forward-fill as fallback.
3. **Forecasting/modeling** using engineered time features and XGBoost to produce the final submission CSV.

## Data Used
The main input file is:

- `Data for Datathon (Revised).xlsx`

This workbook includes:
- `A - Daily`, `B - Daily`, `C - Daily`, `D - Daily`
- `A - Interval`, `B - Interval`, `C - Interval`, `D - Interval`
- `Daily Staffing`
- `Definitions`

The interval sheets contain fields such as:
- `Month`
- `Day`
- `Interval`
- `Service Level`
- `Call Volume`
- `Abandoned Calls`
- `Abandoned Rate`
- `CCT`

The daily sheets contain fields such as:
- `Date`
- `Call Volume`
- `CCT`
- `Service Level`
- `Abandon Rate`

The forecast template file is:
- `template_forecast_v00.csv`

## Repository Files
- `UTCProcessing.ipynb` — notebook for timestamp cleaning and interval completion
- `utcprocessing.py` — script version of the UTC/timestamp preprocessing logic
- `impute_all_sheets.ipynb` — notebook for missing-value imputation
- `impute_all_sheets.py` — script version of the imputation pipeline
- `Model.ipynb` — notebook that combines preprocessing, imputation, feature creation, model training, and final forecast generation
- `template_forecast_v00.csv` — submission template used for prediction formatting
- `DatathonTeam133CodeNotebooks.zip` — packaged notebook/code archive

## Environment Setup
This project was originally developed in **Google Colab**, but it can also be run locally in Jupyter.

### 1. Clone the repository
```bash
git clone https://github.com/vinayek2/Datathon-Information.git
cd Datathon-Information
```

### 2. Create a Python environment
Using `venv`:

```bash
python -m venv .venv
source .venv/bin/activate
```

On Windows PowerShell:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

### 3. Install dependencies
```bash
pip install pandas numpy statsmodels scikit-learn xgboost openpyxl jupyter
```

## Recommended Run Order
If you want to reproduce the workflow step by step, run the files in this order:

1. `UTCProcessing.ipynb` or `utcprocessing.py`
2. `impute_all_sheets.ipynb` or `impute_all_sheets.py`
3. `Model.ipynb`

---

## File-by-File Breakdown

### 1) `UTCProcessing.ipynb`
This notebook cleans and standardizes the interval data.

#### What it does
- Loads the Excel workbook and reads all interval/daily sheets
- Converts month names to month numbers where needed
- Builds a full timestamp from `Month + Day + Interval`
- Localizes timestamps to **UTC**
- Creates a complete 30-minute sequence with `pd.date_range(...)`
- Left joins the original data onto the complete timeline
- Rebuilds `Interval`, `Month`, and `Day` from the completed timestamp range
- Verifies that each day contains exactly **48** half-hour intervals

#### Main function used
```python
process_interval_data(df, year=2025)
```

#### How to run it in Jupyter
Open the notebook and run all cells.

#### How to run the script version
```bash
python utcprocessing.py
```

#### Important note
The notebook version contains explicit file paths like:
```python
/content/Data for Datathon (Revised).xlsx
```
or in Colab:
```python
/content/drive/MyDrive/Colab Notebooks/Data for Datathon (Revised).xlsx
```
If you run locally, update the path so it points to your copy of the Excel file.

---

### 2) `impute_all_sheets.ipynb`
This notebook fills missing numeric values in each sheet.

#### What it does
- Defines which identifier columns should never be imputed
- Chooses an ARIMA order using:
  - ADF test for differencing (`d`)
  - AIC search for `p` and `q`
- Fits a **SARIMAX** model to each numeric column with missing values
- Uses **Kalman smoothing** to estimate missing observations
- Falls back to interpolation and forward/backward fill if the model fails
- Clips filled values to the observed min/max range for that column
- Writes cleaned CSV files for each sheet

#### Main functions used
```python
choose_arima_order(series, max_p=4, max_q=2, max_d=2)
kalman_fill(series, order=None)
impute_one_sheet(sheet_name, df, verbose=True)
impute_all_sheets(datasets)
```

#### How to run it in Jupyter
Open the notebook, make sure your input dataframes are loaded, then run all cells.

#### How to run the script version
```bash
python impute_all_sheets.py
```

#### Important note
The script ends with a reminder:
```python
Load your Excel file into a dictionary first, then run impute_all_sheets(datasets).
```
So before calling `impute_all_sheets(datasets)`, you should create a dictionary like:

```python
datasets = {
    "A - Interval": df_interval1,
    "B - Interval": df_interval2,
    "C - Interval": df_interval3,
    "D - Interval": df_interval4,
    "A - Daily": df_daily1,
    "B - Daily": df_daily2,
    "C - Daily": df_daily3,
    "D - Daily": df_daily4,
}
```

Expected outputs are CSV files such as:
- `A_Interval.csv`
- `B_Interval.csv`
- `C_Interval.csv`
- `D_Interval.csv`
- `A_Daily.csv`
- `B_Daily.csv`
- `C_Daily.csv`
- `D_Daily.csv`

---

### 3) `Model.ipynb`
This notebook is the main modeling pipeline.

#### What it does
- Mounts Google Drive in Colab and adds a script path
- Reuses preprocessing logic from the timestamp-cleaning stage
- Reuses the imputation functions from the SARIMAX/Kalman stage
- Creates merged training data from interval and daily sheets
- Builds time-based features such as:
  - hour
  - minute
  - day of week
  - weekend flag
  - lag values
  - rolling averages
- Trains **XGBoost regressors**
- Predicts values for the forecast template
- Computes abandoned rate from predicted abandoned calls and offered calls
- Exports the final submission file

#### Main functions used
```python
process_interval_data(df, year=2025)
process_datetime_daily(df)
choose_arima_order(series, max_p=4, max_q=2, max_d=2)
kalman_fill(series, order=None)
impute_one_sheet(sheet_name, df, verbose=True)
impute_all_sheets(datasets)
create_features(df, target_col)
```

#### Key libraries used
- `pandas`
- `numpy`
- `statsmodels`
- `scikit-learn`
- `xgboost`

#### How to run it in Jupyter/Colab
Open `Model.ipynb` and run all cells in order.

#### Expected input files
- `Data for Datathon (Revised).xlsx`
- `template_forecast_v00.csv`

#### Final output file
```bash
Final_August_XGBoost_Forecast_Fixed_v2.csv
```

---

## Typical Commands to Run the Project

### Option 1: Notebook workflow
Run these notebooks in order:
1. `UTCProcessing.ipynb`
2. `impute_all_sheets.ipynb`
3. `Model.ipynb`

### Option 2: Python script workflow where possible
```bash
python utcprocessing.py
python impute_all_sheets.py
```
Then open and run:
```bash
jupyter notebook Model.ipynb
```

## Notes on Paths
A few notebook cells use Google Colab / Google Drive paths such as:
- `/content/...`
- `/content/drive/MyDrive/...`

If you run this outside Colab, update those paths to match your local machine.

## Video Demo
Add your project/demo video link here:

```text
https://mediaspace.illinois.edu/media/t/1_5gvua3rc
```

## Summary
In simple terms, this repo follows the same pipeline we used during the datathon:
- clean the timestamp structure,
- fill the missing values,
- engineer time-based features,
- train XGBoost models,
- and generate the final forecast CSV.

We kept the code split by stage so it is easier to understand, debug, and rerun without having to rebuild everything from scratch each time.
