# Code Documentation: 4-data-preparation.ipynb

## Overview

This notebook aggregates subplot-level field data to plot-level data, creating the final ground-truth dataset optimized for satellite imagery matching. It computes plot-level coordinates, area-weighted AGB averages, and preserves metadata for quality assessment.

**Location:** `notebooks/Field-Data/4-data-preparation.ipynb`

---

## Purpose

- Aggregate subplot measurements to plot level
- Compute plot-level coordinates (mean of subplot locations)
- Calculate area-weighted average AGB per plot
- Create plot-level identifiers for satellite data matching
- Generate final ground-truth dataset for machine learning

**Critical Rationale:**
Satellite imagery (Sentinel-2: 10m, Landsat-8: 30m pixels) cannot reliably match individual subplots due to mixed pixels and GPS uncertainty. Plot-level aggregation provides the optimal scale for satellite matching.

---

## Dependencies

### Libraries Used

```python
import pandas as pd
import numpy as np
```

**Libraries:**
- **pandas** - Data manipulation and grouping operations
- **numpy** - Numerical calculations (weighted averages, sums)

---

## Input Data

### File Path
```python
PREPROCESSED_DATA_PATH = "../../data/field-data/preprocessed.csv"
```

### Input Dataset Characteristics
- **Rows:** 2,009 (subplot-level records)
- **Columns:** 7
- **Format:** CSV

### Required Variables
| Variable | Description | Usage |
|----------|-------------|-------|
| `cluster_id` | Cluster identifier | Grouping |
| `plot_id` | Plot identifier within cluster | Grouping |
| `lon` | Longitude (decimal degrees) | Coordinate aggregation |
| `lat` | Latitude (decimal degrees) | Coordinate aggregation |
| `AGB_tha` | Aboveground Biomass (tonnes/ha) | Weighted averaging |
| `subplot_id` | Subplot identifier | Counting |

---

## Output Data

### File Path
```python
FINAL_PLOT_DATA_PATH = "../../data/field-data/final.csv"
```

### Output Dataset Characteristics
- **Rows:** 524 (plot-level records)
- **Columns:** 5
- **Format:** CSV
- **Index:** Not saved (index=False)

### Output Variables
| Variable | Description | Calculation Method |
|----------|-------------|-------------------|
| `plotId` | Composite plot identifier | `cluster_id-plot_id` |
| `plot_lon` | Plot longitude (decimal degrees) | Mean of subplot longitudes |
| `plot_lat` | Plot latitude (decimal degrees) | Mean of subplot latitudes |
| `plot_agb_mean_t_ha` | Plot AGB average (tonnes/ha) | Area-weighted average |
| `plot_total_agb_t` | Total plot AGB (tonnes) | Sum of (AGB × area) |
| `subplot_count` | Number of subplots per plot | Count of subplots |

**Note:** Output column name may vary (e.g., `plot_agb_mean_tha` vs `plot_agb_mean_t_ha`)

---

## Code Structure

### Cell 0-2: Setup
**Markdown:** "Importing Necessary Dependencies"

Imports pandas and numpy, defines file paths.

### Cell 2: Define File Paths
```python
PREPROCESSED_DATA_PATH = "../../data/field-data/preprocessed.csv"
FINAL_PLOT_DATA_PATH = "../../data/field-data/final.csv"
```

### Cell 3: Load Preprocessed Data
```python
df = pd.read_csv(PREPROCESSED_DATA_PATH)
print(f"rows = {df.shape[0]}\ncolumns = {df.shape[1]}")
df.head()
```

**Operations:**
- Loads preprocessed subplot-level data
- Prints dimensions for verification
- Displays first rows

**Output:**
- 2,009 rows × 7 columns

### Cell 4: Data Preparation Section
**Markdown:** Explanation of why plot-level aggregation is necessary:
- Satellite imagery spatial resolution (10m-30m)
- Mixed pixel problem with subplot-level matching
- GPS uncertainty affects small areas

### Cell 5: Plot-Level Aggregation
```python
df["cluster_id"] = df["cluster_id"].astype(str).str.strip()
df["plot_id"] = df["plot_id"].astype(str).str.strip()
df["subplot_area_ha"] = 0.075  

plot_level_df = (
    df.groupby(["cluster_id", "plot_id"], dropna=False)
    .apply(
        lambda x: pd.Series(
            {
                "plot_lon": x["lon"].mean(),
                "plot_lat": x["lat"].mean(),
                "plot_agb_mean_t_ha": np.average(
                    x["AGB_tha"], weights=x["subplot_area_ha"]
                ),
                "plot_total_agb_t": np.sum(x["AGB_tha"] * x["subplot_area_ha"]),
                "subplot_count": len(x),
            }
        ),
        include_groups=False,
    )
    .reset_index()
)

plot_level_df["plotId"] = plot_level_df["cluster_id"] + "-" + plot_level_df["plot_id"]
plot_level_df.drop(columns=["cluster_id", "plot_id"], inplace=True)
plot_level_df.head()
```

**Step-by-Step Breakdown:**

1. **Data Type Conversion and Cleaning**
   ```python
   df["cluster_id"] = df["cluster_id"].astype(str).str.strip()
   df["plot_id"] = df["plot_id"].astype(str).str.strip()
   ```
   - Converts IDs to string type (ensures consistent grouping)
   - Strips whitespace (prevents grouping errors)
   - Critical for reliable groupby operations

2. **Subplot Area Assignment**
   ```python
   df["subplot_area_ha"] = 0.075
   ```
   - Assumes each subplot = 0.075 hectares = 750 m²
   - Used for area-weighted averaging
   - Standard subplot size in forest inventory protocols

3. **Grouping Operation**
   ```python
   df.groupby(["cluster_id", "plot_id"], dropna=False)
   ```
   - Groups by cluster and plot identifiers
   - `dropna=False` preserves groups even with missing values
   - Creates groups for each unique plot

4. **Aggregation Function**
   ```python
   .apply(
       lambda x: pd.Series({
           "plot_lon": x["lon"].mean(),
           "plot_lat": x["lat"].mean(),
           "plot_agb_mean_t_ha": np.average(
               x["AGB_tha"], weights=x["subplot_area_ha"]
           ),
           "plot_total_agb_t": np.sum(x["AGB_tha"] * x["subplot_area_ha"]),
           "subplot_count": len(x),
       }),
       include_groups=False,
   )
   ```
   
   **Calculated Metrics:**
   
   a. **Plot Coordinates**
      ```python
      "plot_lon": x["lon"].mean()
      "plot_lat": x["lat"].mean()
      ```
      - Mean longitude/latitude of all subplots in the plot
      - Represents plot centroid
      - Suitable for satellite pixel extraction
   
   b. **Area-Weighted AGB Average**
      ```python
      "plot_agb_mean_t_ha": np.average(
          x["AGB_tha"], weights=x["subplot_area_ha"]
      )
      ```
      - **Weighted average** considering subplot areas
      - Formula: Σ(AGB_i × Area_i) / Σ(Area_i)
      - More accurate than simple mean if areas differ
      - Result in tonnes per hectare
   
   c. **Total Plot AGB**
      ```python
      "plot_total_agb_t": np.sum(x["AGB_tha"] * x["subplot_area_ha"])
      ```
      - Sum of (AGB per hectare × subplot area)
      - Gives total biomass in tonnes for entire plot
      - Formula: Σ(AGB_tha_i × Area_ha_i)
   
   d. **Subplot Count**
      ```python
      "subplot_count": len(x)
      ```
      - Number of subplots within the plot
      - Quality indicator (more subplots = more reliable estimate)
      - Used for sample weighting in modeling

5. **DataFrame Reset and Structure**
   ```python
   .reset_index()
   ```
   - Converts groupby result to regular DataFrame
   - Restores cluster_id and plot_id as columns

6. **Plot ID Creation**
   ```python
   plot_level_df["plotId"] = (
       plot_level_df["cluster_id"] + "-" + plot_level_df["plot_id"]
   )
   ```
   - Creates composite plot identifier
   - Format: `{cluster_id}-{plot_id}`
   - Example: `"184-35"` (plot 35 in cluster 184)
   - Used for matching with satellite data

7. **Column Cleanup**
   ```python
   plot_level_df.drop(columns=["cluster_id", "plot_id"], inplace=True)
   ```
   - Removes individual ID columns (now redundant)
   - Keeps only composite `plotId`

**Final DataFrame Structure:**
- 524 rows (one per plot)
- Columns: `plotId`, `plot_lon`, `plot_lat`, `plot_agb_mean_t_ha`, `plot_total_agb_t`, `subplot_count`

### Cell 6: Verify Output
```python
rows, _ = plot_level_df.shape
print('Number of plots : ', rows)
```

**Output:**
- Number of plots: 524

**Verification:**
- Confirms aggregation reduced 2,009 subplots → 524 plots
- Expected reduction (approximately 3.8 subplots per plot on average)

### Cell 7: Rationale Documentation
**Markdown:** "### Why this is the 'Golden' Dataset for Satellite Matching"

**Key Points:**

1. **Pixel Alignment**
   - Plot size (500-750 m²) covers multiple satellite pixels
   - Less sensitive to GPS coordinate shifts
   - Matches Sentinel-2 (10m) or Landsat (30m) pixel grids

2. **Statistical Robustness**
   - Averaging across subplots reduces outlier influence
   - Single large tree in one subplot doesn't dominate
   - More stable estimate of forest stand biomass

3. **Subplot Count Quality Indicator**
   - Plots with 4+ subplots have more reliable ground truth
   - Can apply sample weighting during model training
   - Helps identify higher-quality training examples

### Cell 8: Save Section
**Markdown:** "Save the final plot level data"

### Cell 9: Export Final Dataset
```python
plot_level_df.to_csv(FINAL_PLOT_DATA_PATH, index=False)
```

**Operations:**
- Exports aggregated plot-level data to CSV
- `index=False` excludes pandas index column
- Creates final ground-truth dataset for modeling

---

## Aggregation Mathematics

### Area-Weighted Average Calculation

**Formula:**
```
plot_agb_mean_t_ha = Σ(AGB_tha_i × Area_ha_i) / Σ(Area_ha_i)
```

**Example:**
If a plot has 3 subplots:
- Subplot 1: AGB = 100 t/ha, Area = 0.075 ha
- Subplot 2: AGB = 150 t/ha, Area = 0.075 ha
- Subplot 3: AGB = 200 t/ha, Area = 0.075 ha

**Simple Mean:** (100 + 150 + 200) / 3 = 150 t/ha

**Weighted Mean:** 
```
(100×0.075 + 150×0.075 + 200×0.075) / (0.075 + 0.075 + 0.075)
= (7.5 + 11.25 + 15) / 0.225
= 33.75 / 0.225
= 150 t/ha
```

*Note: When all subplot areas are equal, weighted average equals simple mean.*

### Total AGB Calculation

**Formula:**
```
plot_total_agb_t = Σ(AGB_tha_i × Area_ha_i)
```

**Example:**
```
= (100 × 0.075) + (150 × 0.075) + (200 × 0.075)
= 7.5 + 11.25 + 15
= 33.75 tonnes
```

---

## Data Transformation Summary

### Row Reduction
| Stage | Rows | Reduction |
|-------|------|-----------|
| Subplot-level (input) | 2,009 | - |
| Plot-level (output) | 524 | -1,485 rows (74% reduction) |

### Aggregation Ratio
- **Average subplots per plot:** 2,009 / 524 ≈ 3.83 subplots/plot
- **Range:** Typically 1-6 subplots per plot

### Coordinate Transformation
- **Input:** Subplot coordinates (lon, lat)
- **Output:** Plot centroid (mean lon, mean lat)
- **Method:** Simple arithmetic mean
- **Assumption:** Subplots are close together (valid for plot design)

---

## Quality Considerations

### Subplot Count Distribution
Plots with varying numbers of subplots:
- **1 subplot:** Less reliable (single measurement)
- **2-3 subplots:** Moderate reliability
- **4+ subplots:** High reliability (recommended for model training)

### Weighting Strategy for Modeling
Use `subplot_count` to weight training samples:
```python
sample_weights = plot_df['subplot_count'] / plot_df['subplot_count'].max()
```

This gives higher weight to plots with more subplot measurements.

### Coordinate Accuracy
- Plot centroid represents center of subplot locations
- GPS uncertainty affects individual subplot locations
- Plot-level centroid more stable than individual coordinates
- Suitable for satellite pixel extraction (±30m tolerance)

---

## Usage Notes

### When to Run
- **Final step** in Field Data processing pipeline
- Run after preprocessing and visualization
- Required before satellite data extraction

### Prerequisites
- Preprocessed data must exist: `data/field-data/preprocessed.csv`
- pandas and numpy libraries installed

### Output Requirements
- Output directory: `data/field-data/` (must exist)
- Sufficient disk space for CSV file (~50-100 KB)

### Next Steps
After running this notebook:
- Proceed to **Satellite-Data/1-geojson-extraction.ipynb** - Creates geometries for plots
- Use plot-level data for satellite image matching
- Ready for machine learning model training

---

## Technical Details

### Groupby Performance
- **Efficiency:** pandas groupby is highly optimized
- **Memory:** Creates intermediate grouped objects
- **Time complexity:** O(n log n) for grouping, O(n) for aggregation
- **Typical execution:** < 1 second for 2,009 rows

### Area Assumption
- **Assumed subplot area:** 0.075 hectares (750 m²)
- **If actual areas vary:** Would need actual area measurements in dataset
- **Current approach:** Assumes standard subplot size (common in forest inventories)

### Coordinate System
- **Input/Output CRS:** WGS84 (EPSG:4326)
- **Decimal degrees:** Standard GPS format
- **Precision:** Sufficient for plot-level satellite matching

---

## Error Handling

### Potential Issues

1. **Grouping Failures**
   - **Cause:** Inconsistent ID formats or whitespace
   - **Prevention:** String conversion and stripping (done in code)
   - **Validation:** Check for unexpected group counts

2. **Empty Groups**
   - **Cause:** Missing cluster_id or plot_id values
   - **Prevention:** `dropna=False` preserves groups, but values may be NaN
   - **Handling:** Verify all plots have valid IDs

3. **Area Calculation Errors**
   - **Cause:** If subplot areas vary significantly
   - **Current:** Assumes uniform areas
   - **Enhancement:** Could read actual areas from metadata if available

---

## Validation Checks

### Recommended Validations

1. **Row Count Verification**
   ```python
   assert plot_level_df.shape[0] <= df.shape[0]  # Should be fewer plots than subplots
   ```

2. **Subplot Count Sum**
   ```python
   assert plot_level_df['subplot_count'].sum() == df.shape[0]  # Should equal original count
   ```

3. **Coordinate Ranges**
   ```python
   assert plot_level_df['plot_lon'].between(80, 88).all()  # Nepal longitude
   assert plot_level_df['plot_lat'].between(26, 31).all()  # Nepal latitude
   ```

4. **AGB Value Validity**
   ```python
   assert (plot_level_df['plot_agb_mean_t_ha'] > 0).all()  # Positive values
   ```

---

## Related Notebooks

- **2-data-preprocessing.ipynb** - Provides input data (subplot-level)
- **3-data-visualization.ipynb** - Visualizes both subplot and plot-level data
- **Satellite-Data/1-geojson-extraction.ipynb** - Uses plot-level data to create geometries

---

## Performance Considerations

### Execution Time
- **Grouping and aggregation:** < 1 second for 2,009 rows
- **File I/O:** < 1 second for CSV operations
- **Total:** Typically < 2 seconds

### Scalability
- Handles thousands of subplots efficiently
- Memory usage proportional to input size
- Processing time scales linearly with number of unique plots

### Optimization Opportunities
- Could use `agg()` method for specific aggregations (may be faster)
- Vectorized operations already optimal for this use case
- No significant bottlenecks identified
