# Technical Documentation

## Architecture Overview

This ETL pipeline follows a modular, functional design pattern:

```
┌─────────────────┐
│   Input Data    │
│  (JSON, CSV)    │
└────────┬────────┘
         │
    ┌────▼────────────┐
    │  Extract Layer  │
    │ (Normalize raw) │
    └────┬────────────┘
         │
    ┌────▼──────────────┐
    │ Transform Layer   │
    │ (Clean, filter)   │
    └────┬──────────────┘
         │
    ┌────▼────────────┐
    │   Load Layer    │
    │ (Save output)   │
    └────┬────────────┘
         │
    ┌────▼────────────┐
    │ Output Files    │
    │ (CSV, Parquet)  │
    └─────────────────┘
```

---

## Data Schema

### Input: electricity_sales.csv
**Source Format**: CSV (Comma-Separated Values)

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| period | str | ISO date (YYYY-MM format) | "2023-01" |
| sectorName | str | Electricity sector | "residential" |
| stateid | str | US state code | "CA" |
| price | float | Price per unit | 145.32 |
| price-units | str | Currency and unit | "USD/MWh" |

**MWh** = Megawatt-hour (standard electricity unit)

---

### Input: electricity_capability_nested.json
**Source Format**: Nested JSON (hierarchical structure)

**Example Structure:**
```json
[
  {
    "region": "West Coast",
    "states": [
      {
        "code": "CA",
        "capacity": {
          "solar": 45000,
          "wind": 28000,
          "hydro": 22000
        }
      }
    ]
  }
]
```

**After `pd.json_normalize()`:**
```
region     | states.code | states.capacity.solar | states.capacity.wind
West Coast | CA          | 45000                 | 28000
```

---

### Output: loaded__electricity_sales.csv
**Format**: CSV

| Column | Type | Notes |
|--------|------|-------|
| year | str | Extracted from period field |
| month | str | Extracted from period field (zero-padded) |
| stateid | str | Preserved from input |
| price | float | Cleaned (nulls removed) |
| price-units | str | Preserved from input |

**Filters Applied:**
- ✅ Only `residential` and `transportation` sectors
- ✅ Rows with non-null `price` values

---

### Output: loaded__electricity_capability.parquet
**Format**: Apache Parquet (columnar, compressed)

**Content**: Flattened JSON data with all nested keys converted to column names

**Why Parquet?**
- 📊 Efficient for large datasets (compression ratio ~10x)
- ⚡ Fast columnar queries
- 🔒 Schema preservation
- Compatible with: Pandas, PySpark, Pandas Big Query, etc.

---

## Implementation Details

### Extract Phase

#### `extract_tabular_data()`

```python
def extract_tabular_data(file_path: str):
    if file_path.endswith(".csv"):
        return pd.read_csv(file_path)
    elif file_path.endswith(".parquet"):
        return pd.read_parquet(file_path)
    else:
        raise Exception("Warning: Invalid file extension...")
```

**Key Points:**
- ✅ Polymorphic: Handles multiple formats
- ✅ Pandas does all heavy lifting (parsing, type inference)
- ⚠️ No schema validation (relies on file integrity)

---

#### `extract_json_data()`

```python
def extract_json_data(file_path):
    with open(file_path, "r") as json_file:
        raw_data = json.load(json_file)
    return pd.json_normalize(raw_data)
```

**How `pd.json_normalize()` works:**

Input:
```json
[{"user": {"name": "Alice", "age": 30}, "score": 95}]
```

Output (flattened):
```
user.name | user.age | score
Alice     | 30       | 95
```

**Config options** (can be enhanced):
```python
pd.json_normalize(
    raw_data,
    sep='_',  # Separator for nested keys (default: '.')
    max_level=2  # Limit nesting depth
)
```

---

### Transform Phase

#### `transform_electricity_sales_data()`

**Step 1: Null removal**
```python
raw_data.dropna(subset=["price"], inplace=True)
```
- Removes rows where `price` is NaN
- In-place modification for efficiency

**Step 2: Sector filtering**
```python
cleaned_df = raw_data.loc[
    raw_data["sectorName"].isin(["residential", "transportation"]), 
    :
]
```
- Boolean indexing: keeps only matching rows
- `.isin()` method is more efficient than multiple `==` conditions

**Step 3: Date parsing**
```python
cleaned_df["year"] = cleaned_df["period"].str[0:4]
cleaned_df["month"] = cleaned_df["period"].str[5:]
```
- String slicing (fast): extracts year (chars 0-3) and month (chars 5-6)
- Format assumed: `YYYY-MM` (e.g., "2023-01")
- Alternative: `pd.to_datetime()` for more robust parsing

**Step 4: Column selection**
```python
cleaned_df = cleaned_df.loc[:, ["year", "month", "stateid", "price", "price-units"]]
```
- Explicit column reordering
- Drops all other columns (memory efficient)

---

### Load Phase

#### `load()`

```python
def load(dataframe: pd.DataFrame, file_path: str):
    if file_path.endswith(".csv"):
        dataframe.to_csv(file_path)
    elif file_path.endswith(".parquet"):
        dataframe.to_parquet(file_path)
    else:
        raise Exception(f"Warning: {file_path} is not a valid file type...")
```

**CSV Options** (can be customized):
```python
dataframe.to_csv(file_path, index=False)  # Skip row numbers
```

**Parquet Options** (can be customized):
```python
dataframe.to_parquet(file_path, engine='pyarrow', compression='snappy')
```

---

## Performance Considerations

### Memory Usage
- **DataFrame in RAM**: Entire dataset must fit in available RAM
- **Optimization**: Stream processing (for very large files):
  ```python
  df_chunks = pd.read_csv(file_path, chunksize=10000)
  for chunk in df_chunks:
      # Process chunk
  ```

### Speed Optimizations
| Operation | Current | Optimized |
|-----------|---------|-----------|
| Extract CSV | `pd.read_csv()` | `pd.read_parquet()` (10-100x faster) |
| Extract JSON | `json.load()` + `json_normalize()` | Use `orient='table'` if available |
| Filter | `.loc[]` with `.isin()` | Pre-filter during read with `usecols` |
| Save | `to_csv()` | `to_parquet()` (compression included) |

### Scalability Limits
- **Current**: Files up to ~1-2GB (depending on system RAM)
- **Large-scale**: Use PySpark, Dask, or Apache Beam

---

## Error Handling Strategy

### Current Approach
- **Minimal**: Only catches invalid file extensions
- **Risk**: Silent failures if data is malformed (e.g., wrong column names)

### Recommended Enhancements

```python
def transform_electricity_sales_data(raw_data: pd.DataFrame):
    # Validate required columns
    required_cols = ["price", "sectorName", "period"]
    missing = [col for col in required_cols if col not in raw_data.columns]
    if missing:
        raise ValueError(f"Missing columns: {missing}")
    
    # Validate data types
    if not pd.api.types.is_numeric_dtype(raw_data["price"]):
        raise TypeError("'price' column must be numeric")
    
    # Proceed with transformation
    ...
```

---

## Testing Strategy

### Unit Tests
```python
def test_extract_csv():
    df = extract_tabular_data("test_data.csv")
    assert len(df) > 0
    assert "price" in df.columns

def test_transform_filters():
    df = transform_electricity_sales_data(raw_data)
    assert not df["sectorName"].isin(["residential", "transportation"]).all()
```

### Integration Tests
```python
def test_full_pipeline():
    # Run all functions end-to-end
    raw_cap = extract_json_data("electricity_capability_nested.json")
    raw_sales = extract_tabular_data("electricity_sales.csv")
    clean_sales = transform_electricity_sales_data(raw_sales)
    load(clean_sales, "test_output.csv")
    assert os.path.exists("test_output.csv")
```

---

## Future Enhancements

1. **Schema Validation**: Add Pydantic or Great Expectations
2. **Logging**: Add structured logging for debugging
3. **Configuration**: Move hardcoded values to config file
4. **Parallelization**: Process multiple files concurrently
5. **Database Output**: Support direct load to PostgreSQL/BigQuery
6. **Scheduling**: Add Apache Airflow integration
7. **Monitoring**: Add data quality checks post-transform
8. **API**: Expose ETL as REST endpoint

---

## Version History

**v1.0** (March 27, 2026)
- Initial pipeline implementation
- Support for CSV, Parquet, JSON
- Basic data cleaning and filtering
