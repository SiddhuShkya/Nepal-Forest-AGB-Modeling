# Code Documentation: 3-data-visualization.ipynb

## Overview

This notebook creates comprehensive visualizations of the preprocessed field inventory data, including hierarchical structure counts, target variable distribution, and spatial distribution maps. It provides visual insights into the data characteristics and geographic coverage.

**Location:** `notebooks/Field-Data/3-data-visualization.ipynb`

---

## Purpose

- Visualize the hierarchical data structure (clusters, plots, subplots)
- Analyze the distribution of the target variable (AGB)
- Create spatial maps showing plot locations across Nepal
- Generate publication-quality visualizations
- Identify data characteristics and potential issues through visual inspection

---

## Dependencies

### Libraries Used

```python
import os
import pandas as pd
import geopandas as gpd
import matplotlib.pyplot as plt
import seaborn as sns
from shapely.geometry import Point
import contextily as ctx
```

**Libraries:**
- **os** - File path operations
- **pandas** - Data manipulation
- **geopandas** - Spatial data handling
- **matplotlib** - Plotting
- **seaborn** - Statistical visualizations
- **shapely** - Geometric operations
- **contextily** - Basemap tiles for maps

---

## Input Data

### File Path
```python
PREPROCESSED_DATA_PATH = "../../data/field-data/preprocessed.csv"
```

### Input Dataset Characteristics
- **Rows:** 2,009
- **Columns:** 7
- **Format:** CSV

### Required Variables
| Variable | Description | Used For |
|----------|-------------|----------|
| `cluster_id` | Cluster identifier | Hierarchical counts |
| `plot_id` | Plot identifier | Hierarchical counts |
| `subplot_id` | Subplot identifier | Hierarchical counts |
| `lon` | Longitude | Spatial mapping |
| `lat` | Latitude | Spatial mapping |
| `AGB_tha` | Aboveground Biomass (t/ha) | Distribution analysis |

---

## Output Files

### Directory
```python
IMAGES_PATH = "../../images"
```

### Generated Visualizations

1. **clusters_plots_subplots.png**
   - Bar plot showing counts of hierarchical levels
   - Location: `images/clusters_plots_subplots.png`

2. **agb_distribution.png**
   - Histogram with KDE of AGB distribution
   - Location: `images/agb_distribution.png`

3. **nepal_agb_map.png**
   - Georeferenced map showing plot locations
   - Location: `images/nepal_agb_map.png`

---

## Code Structure

### Cell 0-2: Setup
**Markdown:** "Importing Necessary Dependencies"

Imports all required libraries and defines file paths.

### Cell 2: Define Paths and Create Directory
```python
PREPROCESSED_DATA_PATH = "../../data/field-data/preprocessed.csv"
IMAGES_PATH = "../../images"
os.makedirs(IMAGES_PATH, exist_ok=True)
```

- Defines input data path
- Defines output images directory
- Creates images directory if it doesn't exist (`exist_ok=True` prevents errors if exists)

### Cell 3: Load Preprocessed Data
```python
df = pd.read_csv(PREPROCESSED_DATA_PATH)
print(f"rows = {df.shape[0]}\ncolumns = {df.shape[1]}")
df.head()
```

**Operations:**
1. Loads the preprocessed CSV
2. Prints dataset dimensions
3. Displays first rows

**Output:**
- 2,009 rows × 7 columns

### Cell 4: Hierarchical Counts Section
**Markdown:** "Unique number of clusters, plots and subplots"

### Cell 5: Calculate Unique Counts
```python
unique_clusters = df["cluster_id"].nunique()
unique_plots = df[["cluster_id", "plot_id"]].drop_duplicates().shape[0]
unique_subplots = df[["cluster_id", "plot_id", "subplot_id"]].drop_duplicates().shape[0]
```

**Operations:**
1. **Clusters:** Count unique values in `cluster_id` column
2. **Plots:** Count unique combinations of `cluster_id` and `plot_id`
3. **Subplots:** Count unique combinations of all three IDs

**Results:**
- Unique clusters: 195
- Unique plots: 524
- Unique subplots: 2,009

### Cell 6: Prepare Count Data for Visualization
```python
data = pd.DataFrame(
    {
        "Category": ["Clusters", "Plots", "Subplots"],
        "Count": [unique_clusters, unique_plots, unique_subplots],
    }
)
```

Creates a DataFrame with hierarchical counts for plotting.

### Cell 7: Create Hierarchical Structure Bar Plot
```python
PLOT_FILENAME = os.path.join(IMAGES_PATH, "clusters_plots_subplots.png")
if os.path.exists(PLOT_FILENAME):
    print(f"Loading existing plot from: {PLOT_FILENAME}")
    from IPython.display import Image, display
    display(Image(filename=PLOT_FILENAME))
else:
    print("Generating new plot...")
    plt.figure(figsize=(6, 4))
    ax = sns.barplot(data=data, x="Category", y="Count", hue="Category", legend=True)
    
    ax.set_title("Unique Counts of Clusters, Plots, and Subplots")
    ax.set_xlabel("")
    ax.set_xticks([])
    ax.yaxis.grid(True, linestyle="--", alpha=0.7)
    ax.set_axisbelow(True)
    ax.legend(title="Type")
    
    plt.tight_layout()
    plt.savefig(PLOT_FILENAME)
    plt.show()

print("Unique clusters :", unique_clusters)
print("Unique plots    :", unique_plots)
print("Unique subplots :", unique_subplots)
```

**Key Features:**
- **Checkpointing:** Checks if plot exists before generating (saves time on re-runs)
- **Visualization:** Creates bar plot using seaborn
- **Styling:** Custom title, grid, legend
- **Saving:** Exports to PNG file
- **Output:** Prints counts to console

**Visualization Details:**
- Figure size: 6×4 inches
- Grid: Horizontal dashed lines for easier reading
- Legend: Shows category types
- X-axis: No labels (categories in legend)

### Cell 8: AGB Distribution Section
**Markdown:** "Target Distribution"

### Cell 9: Create AGB Distribution Histogram
```python
PLOT_FILENAME = os.path.join(IMAGES_PATH, "agb_distribution.png")

if os.path.exists(PLOT_FILENAME):
    print(f"Loading existing plot: {PLOT_FILENAME}")
    from IPython.display import Image, display
    display(Image(filename=PLOT_FILENAME))
else:
    print("Generating new AGB distribution plot...")
    
    plt.figure(figsize=(6, 4))
    sns.histplot(data=df, x="AGB_tha", bins=20, kde=True)
    
    plt.title("Distribution of AGB (t/ha)")
    plt.xlabel("AGB (t/ha)")
    plt.ylabel("Frequency")
    
    plt.grid(axis="y", linestyle="--", alpha=0.7)
    plt.tight_layout()
    plt.savefig(PLOT_FILENAME)
    plt.show()
```

**Key Features:**
- **Histogram:** 20 bins showing frequency distribution
- **KDE:** Kernel Density Estimate overlay (smooth curve)
- **Checkpointing:** Reuses existing plot if available
- **Styling:** Custom title, labels, grid

**Purpose:** 
- Reveals data distribution shape (normal, skewed, bimodal)
- Identifies outliers and extreme values
- Informs modeling decisions (log transformation, etc.)

### Cell 10: AGB Descriptive Statistics
```python
df['AGB_tha'].describe()
```

**Output:** Statistical summary including:
- Count
- Mean
- Std (Standard Deviation)
- Min, 25%, 50% (median), 75%, Max

**Use:** Quantifies distribution characteristics

### Cell 11: Distribution Observations
**Markdown:** Key observations about AGB distribution:
- **Right-skewed:** Mean > median
- **Low-moderate majority:** Most plots have lower AGB
- **High-value outliers:** Few very high values dominate mean
- **Modeling recommendations:**
  - Use median & IQR instead of mean
  - Log-transform AGB
  - Analyze by cluster/plot for variability

### Cell 12: Spatial Mapping Section
**Markdown:** "Geo-locations of the plots on the map"

### Cell 13: Create Spatial Distribution Map
```python
PLOT_FILENAME = os.path.join(IMAGES_PATH, "nepal_agb_map.png")
if os.path.exists(PLOT_FILENAME):
    print(f"Loading existing map from: {PLOT_FILENAME}")
    from IPython.display import Image, display
    display(Image(filename=PLOT_FILENAME))
else:
    print("Generating new map plot...")
    
    # 1. Convert DataFrame to GeoDataFrame
    geometry = [Point(xy) for xy in zip(df["lon"], df["lat"])]
    gdf = gpd.GeoDataFrame(df, geometry=geometry, crs="EPSG:4326")
    
    # 2. Plotting
    fig, ax = plt.subplots(figsize=(10, 10))
    gdf.plot(ax=ax, color="forestgreen", markersize=20, alpha=0.8)
    
    # Add basemap
    ctx.add_basemap(ax, crs=gdf.crs.to_string(), source=ctx.providers.OpenStreetMap.Mapnik)
    
    # Set extent to Nepal
    ax.set_xlim(80, 88)
    ax.set_ylim(26, 31)
    
    ax.set_title("Forest AGB Plots in Nepal", fontsize=14)
    ax.set_xlabel("Longitude")
    ax.set_ylabel("Latitude")
    
    # 3. Save and Show
    plt.tight_layout()
    plt.savefig(PLOT_FILENAME, bbox_inches='tight', pad_inches=0.1)
    plt.show()
```

**Step-by-Step Operations:**

1. **Create Point Geometries**
   ```python
   geometry = [Point(xy) for xy in zip(df["lon"], df["lat"])]
   ```
   - Creates Shapely Point objects from coordinate pairs
   - List comprehension for efficient creation

2. **Convert to GeoDataFrame**
   ```python
   gdf = gpd.GeoDataFrame(df, geometry=geometry, crs="EPSG:4326")
   ```
   - Combines DataFrame with geometry
   - Sets coordinate reference system (WGS84)

3. **Create Map Figure**
   ```python
   fig, ax = plt.subplots(figsize=(10, 10))
   gdf.plot(ax=ax, color="forestgreen", markersize=20, alpha=0.8)
   ```
   - 10×10 inch figure (square for map)
   - Green markers, size 20, 80% opacity

4. **Add Basemap**
   ```python
   ctx.add_basemap(ax, crs=gdf.crs.to_string(), source=ctx.providers.OpenStreetMap.Mapnik)
   ```
   - Adds OpenStreetMap tiles as background
   - Provides geographic context

5. **Set Map Extent**
   ```python
   ax.set_xlim(80, 88)  # Longitude range
   ax.set_ylim(26, 31)  # Latitude range
   ```
   - Focuses on Nepal's geographic bounds
   - Excludes areas outside study region

6. **Styling and Export**
   - Custom title and axis labels
   - Tight layout with minimal padding
   - High-quality PNG export

---

## Visualization Insights

### 1. Hierarchical Structure
- **195 clusters** → **524 plots** → **2,009 subplots**
- Average: ~2.7 plots per cluster, ~3.8 subplots per plot
- Shows data organization complexity

### 2. AGB Distribution Characteristics
- **Strongly right-skewed distribution**
- Most plots have low-moderate AGB values
- Few very high values (outliers) significantly affect mean
- **Implications:**
  - Log transformation recommended for modeling
  - Robust statistics (median) preferred over mean
  - Consider outlier handling strategies

### 3. Spatial Coverage
- **Geographic extent:** 80-88°E, 26-31°N (Nepal)
- Plots distributed across the country
- Visual verification of spatial coverage
- Can identify geographic clusters or gaps

---

## Technical Details

### Coordinate Reference System
- **Input:** WGS84 (EPSG:4326) - Geographic coordinates
- **Map display:** WGS84 with OpenStreetMap basemap
- **Extent:** Nepal boundaries (80-88°E, 26-31°N)

### Image Format
- **Format:** PNG (Portable Network Graphics)
- **Resolution:** Default matplotlib DPI (typically 100-150 DPI)
- **Size:** Variable (6×4 for plots, 10×10 for map)

### Checkpointing Strategy
- Checks for existing files before generating
- Saves computation time on notebook re-runs
- Useful for long-running visualization operations

---

## Usage Notes

### When to Run
- **Third step** in Field Data pipeline (after preprocessing)
- Can run in parallel with other notebooks
- Useful for data exploration and presentation

### Prerequisites
- Preprocessed data must exist: `data/field-data/preprocessed.csv`
- All visualization libraries installed
- Internet connection for basemap tiles (first run)

### Output Requirements
- Output directory: `images/` (created automatically)
- Internet connection for OpenStreetMap tiles

### Next Steps
After running this notebook:
- Use insights for data preparation decisions
- Share visualizations in reports/presentations
- Proceed to **4-data-preparation.ipynb** for final data preparation

---

## Error Handling

### Potential Issues

1. **Missing Basemap Tiles**
   - **Symptom:** Map shows points without background
   - **Cause:** No internet connection or tile server unavailable
   - **Solution:** Can still generate map without basemap (comment out ctx.add_basemap)

2. **Coordinate Outliers**
   - **Symptom:** Points outside Nepal boundaries
   - **Cause:** Data quality issues
   - **Solution:** Add coordinate filtering before visualization

3. **Memory Issues (Large Datasets)**
   - **Symptom:** Slow performance or crashes
   - **Cause:** Too many points to render
   - **Solution:** Sample data or use spatial aggregation

---

## Performance Considerations

### Execution Time
- **Count calculations:** < 1 second
- **Bar plot:** < 1 second
- **Histogram:** < 1 second
- **Map generation:** 5-30 seconds (depends on basemap download)

### Optimization Tips
- Checkpointing significantly speeds up re-runs
- Basemap caching reduces download time
- Consider downsampling for very large datasets

---

## Related Notebooks

- **2-data-preprocessing.ipynb** - Provides input data
- **4-data-preparation.ipynb** - Uses visualizations to inform aggregation strategy
- **1-data-understanding.ipynb** - Provides context for visualizations

---

## Visualization Best Practices Applied

1. **Clear Titles and Labels:** All plots have descriptive titles
2. **Appropriate Colors:** Green for forest data, intuitive color schemes
3. **Grid Lines:** Improve readability of numeric values
4. **Legend Usage:** Clarifies categories and values
5. **Appropriate Figure Sizes:** Large maps, smaller statistical plots
6. **Export Quality:** High-resolution PNG for publications
