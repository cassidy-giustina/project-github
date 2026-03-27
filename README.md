# Electricity Sales & Capability ETL Pipeline

A Python-based ETL (Extract, Transform, Load) pipeline for processing and analyzing electricity sales and capability data. This project normalizes nested JSON data, transforms tabular electricity sales records, and outputs cleaned datasets for analysis.

## 📋 Project Overview

This notebook implements a complete data pipeline with the following workflow:

```
Raw JSON (nested) ──┐
                     ├──→ Transform ──→ Load (Output Files)
Raw CSV ────────────┘
```

### Data Availability Note
- This pipeline runs with the project files listed below.
- If some files are missing locally, download/copy them into the project folder before running the notebook.

### Key Features
- ✅ **Flexible file parsing**: Supports CSV, Parquet, and nested JSON formats
- ✅ **Data normalization**: Flattens hierarchical JSON into tabular format using `pd.json_normalize()`
- ✅ **Intelligent filtering**: Isolates residential and transportation electricity sectors
- ✅ **Date parsing**: Extracts year and month from period fields for temporal analysis
- ✅ **Data validation**: Removes records with missing prices and validates file types
- ✅ **Multiple output formats**: Saves results as CSV or Parquet

---

## 📁 Project Structure

```
Electricity Sales & Capability ETL Pipeline/
├── poweringdata.ipynb                          # Main ETL notebook
├── electricity_capability_nested.json          # Input: Nested JSON data (capability info)
├── electricity_sales.csv                       # Input: Tabular CSV data (sales records)
├── loaded__electricity_sales.csv               # Output: Cleaned sales data
├── loaded__electricity_capability.parquet      # Output: Normalized capability data (generated after running pipeline)
└── README.md                                   # This file
```

---

## 🔧 Functions & Workflow

### 1. **extract_tabular_data(file_path: str) → pd.DataFrame**
Reads tabular data from CSV or Parquet files.

**Usage:**
```python
df = extract_tabular_data("electricity_sales.csv")
```

**Supported formats:**
- `.csv` — Comma-separated values
- `.parquet` — Apache Parquet columnar format

**Error handling:**
- Raises `Exception` if file extension is neither CSV nor Parquet

---

### 2. **extract_json_data(file_path: str) → pd.DataFrame**
Reads and normalizes nested JSON data into a flat DataFrame.

**Usage:**
```python
df = extract_json_data("electricity_capability_nested.json")
```

**How it works:**
1. Loads JSON file into memory using Python's `json` library
2. Flattens nested structures using `pd.json_normalize()`
3. Returns DataFrame with all nested keys converted to column names

**Why it matters:**
- Converts hierarchical data into queryable tabular format
- Handles complex nested structures automatically

---

### 3. **transform_electricity_sales_data(raw_data: pd.DataFrame) → pd.DataFrame**
Cleans and reshapes electricity sales data for analysis.

**Transformation steps:**
1. **Remove nulls**: Drops rows with missing `price` values
2. **Filter sectors**: Keeps only `residential` and `transportation` sectors
3. **Parse dates**: Extracts `year` and `month` from `period` field (format: `YYYY-MM`)
4. **Select columns**: Returns only: `year`, `month`, `stateid`, `price`, `price-units`

**Usage:**
```python
cleaned_df = transform_electricity_sales_data(raw_electricity_sales_df)
```

**Output columns:**
| Column | Type | Example |
|--------|------|---------|
| year | str | "2023" |
| month | str | "01" |
| stateid | str | "CA" |
| price | float | 145.32 |
| price-units | str | "USD/MWh" |

---

### 4. **load(dataframe: pd.DataFrame, file_path: str)**
Saves a DataFrame to CSV or Parquet format.

**Usage:**
```python
load(cleaned_df, "output_data.csv")
load(cleaned_df, "output_data.parquet")
```

**Supported formats:**
- `.csv` — Human-readable, widely compatible
- `.parquet` — Compressed, faster for large datasets

**Error handling:**
- Raises `Exception` if file extension is invalid

---

## 🚀 Running the Pipeline

### Prerequisites
```bash
pip install pandas
```

Make sure the required files are present in the project folder before running the notebook:
- `electricity_sales.csv`
- `electricity_capability_nested.json`

### Step 1: Open the Notebook
Open `poweringdata.ipynb` in Jupyter or VS Code

### Step 2: Run All Cells
Execute cells in order:
1. **Cell 1** - Import libraries
2. **Cell 2** - Define `extract_tabular_data()`
3. **Cell 3** - Define `extract_json_data()`
4. **Cell 4** - Define `transform_electricity_sales_data()`
5. **Cell 5** - Define `load()`
6. **Cell 6** - Execute the full pipeline

### Step 3: Verify Output
After running the pipeline, check for these output files:
- ✅ `loaded__electricity_sales.csv` — Cleaned sales data
- ✅ `loaded__electricity_capability.parquet` — Normalized capability data (if generated in your run)

---

## 📊 Data Flow Example

**Input (raw_electricity_sales)**
```
period    | sectorName    | stateid | price | price-units
2023-01   | residential   | CA      | 145.2 | USD/MWh
2023-01   | transportation| TX      | 98.5  | USD/MWh
2023-01   | industrial    | NY      | 120.3 | USD/MWh
```

**After Transform**
```
year | month | stateid | price | price-units
2023 | 01    | CA      | 145.2 | USD/MWh
2023 | 01    | TX      | 98.5  | USD/MWh
```
*(Industrial sector filtered out; dates parsed)*

---

## 🔍 Use Cases

- **Utilities Analysis**: Analyze electricity pricing trends across sectors and states
- **Market Research**: Compare residential vs. transportation sector electricity costs
- **Data Warehousing**: Prepare data for BI dashboards (Tableau, Power BI)
- **Time Series Analysis**: Study price changes over months and years
- **Capacity Planning**: Use capability data for infrastructure decisions

---

## 📈 Next Steps

1. **Exploratory Analysis**: Add visualization cells to plot price trends
2. **Statistical Analysis**: Calculate averages, correlations by sector/state
3. **Forecasting**: Build models to predict future electricity prices
4. **Automation**: Schedule this pipeline to run on new data monthly
5. **Database Integration**: Load results into SQL database for querying

---

## 📝 License

This project is provided as-is for educational and analytical purposes.


**Last Updated**: March 27, 2026
