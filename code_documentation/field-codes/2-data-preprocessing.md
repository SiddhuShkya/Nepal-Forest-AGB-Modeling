# Code Documentation: 2-data-preprocessing.ipynb

## Overview

This notebook performs essential data cleaning and preprocessing operations on the raw field inventory data. It removes unnecessary columns, handles missing values, eliminates duplicates, and decomposes the hierarchical plot ID structure into separate components for easier analysis.

**Location:** `notebooks/Field-Data/2-data-preprocessing.ipynb`

---

## Purpose

- Clean the raw dataset by removing unnecessary columns
- Handle missing values and duplicate records
- Decompose composite plot IDs into separate hierarchical components
- Prepare data for downstream analysis and visualization
- Create a preprocessed dataset ready for further processing

---

## Dependencies

### Libraries Used

```python
import pandas as pd
```

**pandas** - For data manipulation, cleaning, and string operations

---

## Input Data

### File Path
```python
FIELD_DATA_PATH = "../../data/field-data/nepal_forest_agb.csv"
```

### Input Dataset Characteristics
- **Rows:** 2,038 (initial)
- **Columns:** 5
- **Format:** CSV

### Input Variables
| Variable | Description |
|----------|-------------|
| `plot_id` | Composite hierarchical identifier |
| `lon` | Longitude (decimal degrees) |
| `lat` | Latitude (decimal degrees) |
| `AGB_tha` | Aboveground Biomass (tonnes/ha) |
| `SOC_tha` | Soil Organic Carbon (tonnes/ha) |

---

## Output Data

### File Path
```python
PREPROCESSED_DATA_PATH = "../../data/field-data/preprocessed.csv"
```

### Output Dataset Characteristics
- **Rows:** 2,009 (after cleaning)
- **Columns:** 7 (after ID decomposition)
- **Format:** CSV
- **Index:** Not saved (index=False)

### Output Variables
| Variable | Description |
|----------|-------------|
| `id` | Original composite plot_id (renamed from plot_id) |
| `lon` | Longitude (decimal degrees) |
| `lat` | Latitude (decimal degrees) |
| `AGB_tha` | Aboveground Biomass (tonnes/ha) |
| `cluster_id` | Cluster identifier (extracted from plot_id) |
| `plot_id` | Plot identifier within cluster (extracted) |
| `subplot_id` | Subplot identifier within plot (extracted) |

---

## Code Structure

### Cell 0-2: Setup
**Markdown:** "Importing Necessary Dependencies" and "Load our data"

Imports pandas and defines input/output file paths.

### Cell 3: Define File Paths
```python
FIELD_DATA_PATH = "../../data/field-data/nepal_forest_agb.csv"
PREPROCESSED_DATA_PATH = "../../data/field-data/preprocessed.csv"
```

Defines both input (raw data) and output (preprocessed data) file paths.

### Cell 4: Load Initial Data
```python
df = pd.read_csv(FIELD_DATA_PATH)
print(f"rows = {df.shape[0]}\ncolumns = {df.shape[1]}")
df.head()
```

**Operations:**
1. Loads the raw CSV file
2. Prints initial dimensions
3. Displays first rows for inspection

**Output:**
- Initial shape: 2,038 rows × 5 columns

### Cell 5: Basic Data Cleaning Section
**Markdown:** "Basic Data Cleaning"

### Cell 6: Data Cleaning Operations
```python
# Dropping unnecessary columns
df.drop(columns=['SOC_tha'], axis=1, inplace=True)
# Drop null rows
df.dropna(inplace=True)
# Drop Duplicate rows
df.drop_duplicates(inplace=True)

print(f"rows = {df.shape[0]}\ncolumns = {df.shape[1]}")
```

**Step-by-Step Operations:**

1. **Remove SOC Column**
   ```python
   df.drop(columns=['SOC_tha'], axis=1, inplace=True)
   ```
   - Removes Soil Organic Carbon column (focus on AGB only)
   - `inplace=True` modifies the DataFrame directly
   - Result: 5 columns → 4 columns

2. **Remove Missing Values**
   ```python
   df.dropna(inplace=True)
   ```
   - Removes any rows with null/NaN values
   - Ensures complete data records
   - Result: 2,038 rows → ~2,009 rows (29 rows removed)

3. **Remove Duplicates**
   ```python
   df.drop_duplicates(inplace=True)
   ```
   - Removes exact duplicate rows
   - Ensures unique subplot records
   - Result: Maintains ~2,009 rows (no duplicates found)

**Output:**
- Cleaned shape: 2,009 rows × 4 columns

### Cell 7: Plot ID Decomposition Section
**Markdown:** "plot_id (id) -> cluster_id, plot_id, subplot_id"

### Cell 8: Decompose Hierarchical ID
```python
df = df.rename(columns={"plot_id": "id"})
df[["cluster_id", "plot_id", "subplot_id"]] = df["id"].str.split("-", expand=True)
df.head()
```

**Step-by-Step Operations:**

1. **Rename Original Column**
   ```python
   df = df.rename(columns={"plot_id": "id"})
   ```
   - Renames `plot_id` to `id` to preserve original composite ID
   - Original values retained for reference

2. **Split Composite ID**
   ```python
   df[["cluster_id", "plot_id", "subplot_id"]] = df["id"].str.split("-", expand=True)
   ```
   - Uses pandas string `split()` method
   - Splits on `-` delimiter
   - `expand=True` creates separate columns for each component
   - Creates three new columns from one composite ID

**Example Transformation:**
```
Before:
id: "184-35-2"

After:
id: "184-35-2"
cluster_id: "184"
plot_id: "35"
subplot_id: "2"
```

**Output DataFrame Structure:**
- 7 columns total: `id`, `lon`, `lat`, `AGB_tha`, `cluster_id`, `plot_id`, `subplot_id`
- All 2,009 rows preserved

### Cell 9: Save Section
**Markdown:** "Save the preprocessed data"

### Cell 10: Export Preprocessed Data
```python
df.to_csv(PREPROCESSED_DATA_PATH, index=False)
```

**Operations:**
1. Exports DataFrame to CSV format
2. `index=False` excludes the pandas index column
3. Saves to `data/field-data/preprocessed.csv`

---

## Data Transformations Summary

### Row Count Changes
| Stage | Rows | Change |
|-------|------|--------|
| Initial Load | 2,038 | - |
| After dropna() | ~2,009 | -29 rows |
| After drop_duplicates() | 2,009 | 0 rows (no duplicates) |
| Final Output | 2,009 | - |

### Column Changes
| Stage | Columns | Change |
|-------|---------|--------|
| Initial Load | 5 | - |
| After drop SOC_tha | 4 | -1 column |
| After ID split | 7 | +3 columns |
| Final Output | 7 | - |

### Removed Records
- **29 rows** removed due to missing values (null/NaN)
- **No duplicate rows** found in the dataset
- All removed records likely had missing AGB values or coordinates

---

## Key Processing Decisions

### 1. SOC Column Removal
**Reason:** Focus on Aboveground Biomass (AGB) only for this project  
**Impact:** Reduces dataset complexity, focuses analysis on primary variable  
**Alternative:** Could be kept if SOC modeling is needed

### 2. Complete Row Removal for Missing Values
**Reason:** Ensures data quality - incomplete records cannot be used for modeling  
**Method:** `dropna()` without specifying columns removes rows with ANY missing values  
**Impact:** 29 records lost (1.4% of data)  
**Alternative:** Could use column-specific dropping if some columns can be missing

### 3. ID Decomposition Strategy
**Reason:** Separate components enable grouping and hierarchical analysis  
**Method:** String splitting on delimiter preserves all information  
**Impact:** Original ID preserved in `id` column, components available separately  
**Alternative:** Could use regex for more complex patterns

---

## Data Quality Checks

### Implemented Checks
1. ✅ Missing value removal (complete rows)
2. ✅ Duplicate detection and removal
3. ✅ Column structure validation (automatic during processing)

### Recommended Additional Checks
- Verify coordinate ranges (Nepal: ~80-88°E, 26-31°N)
- Check AGB value ranges (should be positive, realistic ranges)
- Validate ID structure consistency (all should have 3 components)
- Check for coordinate outliers

---

## Usage Notes

### When to Run
- **Second step** in the Field Data pipeline (after data understanding)
- Run after `1-data-understanding.ipynb`
- Required before visualization or data preparation notebooks

### Prerequisites
- Raw data file must exist: `data/field-data/nepal_forest_agb.csv`
- pandas library installed

### Output Requirements
- Output directory: `data/field-data/` (must exist or be created)
- Sufficient disk space for CSV file (~100-200 KB)

### Next Steps
After running this notebook, proceed to:
- **3-data-visualization.ipynb** - Visualize preprocessed data
- **4-data-preparation.ipynb** - Aggregate to plot level (requires this preprocessing)

---

## Technical Details

### String Operations
- Uses pandas `.str` accessor for vectorized string operations
- `.split("-", expand=True)` creates multiple columns in one operation
- More efficient than iterating through rows

### Memory Management
- `inplace=True` operations modify DataFrame in place (saves memory)
- Final CSV export creates a new file (doesn't modify original)

### Data Types
After preprocessing:
- `id`: String (object)
- `lon`, `lat`: Float64
- `AGB_tha`: Float64
- `cluster_id`, `plot_id`, `subplot_id`: String (object)

---

## Error Handling

### Potential Issues
1. **File not found:** Verify `FIELD_DATA_PATH` is correct
2. **ID format inconsistent:** If split fails, some plot_ids may not follow `cluster-plot-subplot` format
3. **All data removed:** If all rows have missing values, output will be empty

### Validation Recommendations
- Check row count before and after processing
- Verify ID split success (all new columns should have values)
- Inspect sample rows to confirm transformations

---

## Related Notebooks

- **1-data-understanding.ipynb** - Provides context for preprocessing decisions
- **3-data-visualization.ipynb** - Visualizes the cleaned data
- **4-data-preparation.ipynb** - Uses decomposed IDs for grouping

---

## Performance Considerations

### Execution Time
- Typically runs in < 1 second for ~2,000 rows
- CSV read/write operations are the slowest steps
- String splitting is vectorized and fast

### Scalability
- Handles datasets up to millions of rows efficiently
- Memory usage proportional to dataset size
- Processing time scales linearly with row count
