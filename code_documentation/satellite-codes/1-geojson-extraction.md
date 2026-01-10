# Code Documentation: 1-geojson-extraction.ipynb

## Overview

This notebook creates circular Area of Interest (AOI) polygons for each plot in GeoJSON format. These geometries are used for extracting satellite imagery from Google Earth Engine. The notebook handles coordinate system transformations to ensure accurate metric-based buffering.

**Location:** `notebooks/Satellite-Data/1-geojson-extraction.ipynb`

---

## Purpose

- Convert plot coordinates to spatial geometries
- Create circular buffers representing plot areas (750 m²)
- Generate individual GeoJSON files for each plot
- Ensure accurate spatial representation using proper coordinate systems
- Prepare geometries for satellite data extraction

**Critical Rationale:**
Google Earth Engine requires geometry objects (polygons) to extract satellite imagery for specific areas. Circular buffers represent the plot footprint, matching the field inventory plot design.

---

## Dependencies

### Libraries Used

```python
import os
import pandas as pd
import geopandas as gpd
import numpy as np
```

**Libraries:**
- **os** - Directory creation and file path operations
- **pandas** - Data manipulation
- **geopandas** - Spatial data handling and geometry operations
- **numpy** - Mathematical calculations (square root for radius)

---

## Input Data

### File Path
```python
FINAL_PLOT_DATA = "../../data/field-data/final.csv"
```

### Input Dataset Characteristics
- **Rows:** 524 (plot-level records)
- **Columns:** 5
- **Format:** CSV

### Required Variables
| Variable | Description | Usage |
|----------|-------------|-------|
| `plotId` | Composite plot identifier | File naming |
| `plot_lon` | Plot longitude (decimal degrees) | Point geometry creation |
| `plot_lat` | Plot latitude (decimal degrees) | Point geometry creation |
| `plot_agb_mean_tha` | Plot AGB (tonnes/ha) | Metadata (preserved but not used) |
| `subplot_count` | Number of subplots | Metadata (preserved but not used) |

---

## Output Data

### Directory Path
```python
GEOJSON_DIR_PATH = "../../data/geojson-data"
```

### Output Format
- **File format:** GeoJSON (GeoJSON specification)
- **Number of files:** 524 (one per plot)
- **File naming convention:** `plot-{plotId}.geojson`
  - Example: `plot-10-75.geojson`

### Output Structure
Each GeoJSON file contains:
- **Feature:** Single polygon feature
- **Geometry:** Circular polygon (buffered point)
- **Properties:** All original DataFrame columns preserved
- **CRS:** WGS84 (EPSG:4326)

---

## Code Structure

### Cell 0-2: Setup
**Markdown:** "Importing Dependencies"

Imports required libraries and defines file paths.

### Cell 2: Define Paths
```python
GEOJSON_DIR_PATH = "../../data/geojson-data"
FINAL_PLOT_DATA = "../../data/field-data/final.csv"
```

### Cell 3: Create Output Directory
```python
if not os.path.exists(GEOJSON_DIR_PATH):
    os.makedirs(GEOJSON_DIR_PATH)
```

**Operations:**
- Checks if output directory exists
- Creates directory if missing
- Prevents errors during file writing

### Cell 4: Load Data Section
**Markdown:** "Loading our data"

### Cell 5: Load Plot-Level Data
```python
df = pd.read_csv(FINAL_PLOT_DATA)
rows, _ = df.shape
print(df.head())
print('\nNumber of plots = ', rows)
```

**Operations:**
- Loads final plot-level dataset
- Displays first rows for verification
- Prints total number of plots

**Output:**
- 524 plots loaded
- Displays sample data structure

### Cell 6: Methodology Explanation
**Markdown:** Explains the approach:
- Create circular AOI for each plot
- Use UTM Zone 45N (EPSG:32645) for metric calculations
- Convert back to WGS84 (EPSG:4326) for GeoJSON standard
- Ensures accurate 750 m² area representation

### Cell 7: Create Point Geometries
```python
gdf_points = gpd.GeoDataFrame(
    df,
    geometry=gpd.points_from_xy(df.plot_lon, df.plot_lat),
    crs="EPSG:4326",
)
```

**Step-by-Step:**

1. **Create Point Geometries**
   ```python
   gpd.points_from_xy(df.plot_lon, df.plot_lat)
   ```
   - Converts coordinate pairs to Shapely Point objects
   - Creates geometry column from longitude/latitude

2. **Create GeoDataFrame**
   ```python
   gdf_points = gpd.GeoDataFrame(df, geometry=..., crs="EPSG:4326")
   ```
   - Combines DataFrame with geometry column
   - Sets coordinate reference system to WGS84 (geographic)
   - Preserves all original DataFrame columns

**Result:**
- GeoDataFrame with 524 point geometries
- CRS: EPSG:4326 (WGS84)
- All plot attributes preserved

### Cell 8: Project to Metric CRS and Calculate Radius
```python
gdf_metric = gdf_points.to_crs(epsg=32645)
radius_val = np.sqrt(750 / np.pi)
```

**Step-by-Step:**

1. **Coordinate System Transformation**
   ```python
   gdf_metric = gdf_points.to_crs(epsg=32645)
   ```
   - Transforms from WGS84 (EPSG:4326) to UTM Zone 45N (EPSG:32645)
   - **UTM Zone 45N:** Universal Transverse Mercator, Zone 45 North
   - **Coverage:** Nepal falls within UTM Zone 45N
   - **Units:** Meters (enables accurate metric buffering)

2. **Calculate Buffer Radius**
   ```python
   radius_val = np.sqrt(750 / np.pi)
   ```
   - **Formula:** r = √(A / π)
   - **Area:** 750 m² (standard plot size)
   - **Calculation:** √(750 / π) ≈ 15.45 meters
   - **Result:** Radius needed for 750 m² circular area

**Why UTM Zone 45N?**
- **Accurate distances:** Metric CRS ensures buffer distances are true meters
- **Minimal distortion:** UTM minimizes projection distortion in Nepal region
- **Standard practice:** Common choice for Nepal spatial analysis

### Cell 9: Extraction Section
**Markdown:** "Extracting the geojson data of our plots"

### Cell 10: Generate Individual GeoJSON Files
```python
print(f"Exporting {len(gdf_metric)} files to {GEOJSON_DIR_PATH}...")
for index, row in gdf_metric.iterrows():
    # Create the buffer (Polygon) for this specific row
    aoi_polygon = row.geometry.buffer(radius_val)

    # Create a single-row GeoDataFrame for export
    single_aoi = gpd.GeoDataFrame([row], geometry=[aoi_polygon], crs=gdf_metric.crs)

    # Project back to WGS84 for the GeoJSON standard
    single_aoi_wgs84 = single_aoi.to_crs(epsg=4326)

    # Clean file name using the unique_plot_id we created earlier
    file_name = f"plot-{row['plotId']}.geojson"
    file_path = os.path.join(GEOJSON_DIR_PATH, file_name)

    # Save to file
    single_aoi_wgs84.to_file(file_path, driver="GeoJSON")

print("Done! All records are saved in the 'geojson-data' directory.")
```

**Step-by-Step Iteration:**

For each plot (524 iterations):

1. **Create Circular Buffer**
   ```python
   aoi_polygon = row.geometry.buffer(radius_val)
   ```
   - Creates circular polygon around plot center point
   - Buffer distance: ~15.45 meters (from radius calculation)
   - **Geometry:** Shapely Polygon object (circle approximation)
   - **Resolution:** Default polygon segments (approximates circle)

2. **Create Single-Plot GeoDataFrame**
   ```python
   single_aoi = gpd.GeoDataFrame([row], geometry=[aoi_polygon], crs=gdf_metric.crs)
   ```
   - Creates GeoDataFrame with single row
   - Replaces point geometry with buffer polygon
   - Preserves all original attributes (plotId, AGB, etc.)
   - CRS: Still UTM Zone 45N (metric)

3. **Transform to WGS84**
   ```python
   single_aoi_wgs84 = single_aoi.to_crs(epsg=4326)
   ```
   - Converts from UTM Zone 45N back to WGS84
   - **Why?** GeoJSON standard uses WGS84 (EPSG:4326)
   - Coordinates now in decimal degrees
   - Polygon shape preserved (transformed coordinates)

4. **Generate File Name**
   ```python
   file_name = f"plot-{row['plotId']}.geojson"
   ```
   - Format: `plot-{cluster_id}-{plot_id}.geojson`
   - Example: `plot-10-75.geojson` (plot 75 in cluster 10)
   - Consistent naming for easy identification

5. **Construct File Path**
   ```python
   file_path = os.path.join(GEOJSON_DIR_PATH, file_name)
   ```
   - Combines directory path with filename
   - Platform-independent path construction

6. **Export GeoJSON**
   ```python
   single_aoi_wgs84.to_file(file_path, driver="GeoJSON")
   ```
   - Writes GeoJSON file to disk
   - Uses GeoPandas GeoJSON driver
   - Includes geometry and all attributes

**Output:**
- 524 individual GeoJSON files
- Each file represents one plot's AOI
- Files ready for Google Earth Engine ingestion

---

## Geometry Details

### Buffer Creation

**Method:** Shapely `buffer()` method
- **Input:** Point geometry
- **Distance:** 15.45 meters (in UTM coordinates)
- **Resolution:** Default (approximates circle with line segments)
- **Output:** Polygon geometry (circular area)

**Note:** The buffer creates a polygon with multiple vertices approximating a circle. The number of vertices depends on buffer resolution.

### Coordinate System Transformations

**Transformation Chain:**
1. **Input:** WGS84 (EPSG:4326) - Geographic coordinates
2. **Buffer:** UTM Zone 45N (EPSG:32645) - Metric coordinates
3. **Output:** WGS84 (EPSG:4326) - Geographic coordinates

**Why Transform?**
- **WGS84:** Cannot do accurate metric buffering (degrees vary by latitude)
- **UTM:** Enables precise meter-based calculations
- **Final WGS84:** Standard for GeoJSON, compatible with Google Earth Engine

---

## Area Calculation Mathematics

### Circular Area Formula

**Given Area (A):** 750 m²  
**Find Radius (r):**

```
A = π × r²
750 = π × r²
r² = 750 / π
r = √(750 / π)
r ≈ 15.45 meters
```

### Verification

**Area from Radius:**
```
A = π × (15.45)²
A = π × 238.7
A ≈ 750 m² ✓
```

**Note:** Actual area may vary slightly due to:
- Polygon approximation of circle
- Coordinate transformation precision
- But should be very close to 750 m²

---

## Output File Structure

### GeoJSON Format Example

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Polygon",
        "coordinates": [[
          [80.414..., 28.870...],
          [80.414..., 28.870...],
          ...
        ]]
      },
      "properties": {
        "plotId": "10-75",
        "plot_lon": 80.414795,
        "plot_lat": 28.870564,
        "plot_agb_mean_tha": 477.230220,
        "subplot_count": 2
      }
    }
  ]
}
```

### File Contents
- **Type:** FeatureCollection (GeoJSON standard)
- **Features:** Single feature per file (one plot)
- **Geometry Type:** Polygon (circular buffer)
- **Properties:** All original DataFrame columns
- **CRS:** WGS84 (EPSG:4326)

---

## Usage Notes

### When to Run
- **First step** in Satellite Data processing pipeline
- Run after **Field-Data/4-data-preparation.ipynb**
- Required before satellite image fetching

### Prerequisites
- Final plot data must exist: `data/field-data/final.csv`
- geopandas, pandas, numpy libraries installed
- Output directory will be created automatically

### Output Requirements
- Output directory: `data/geojson-data/` (created automatically)
- Disk space: ~1-2 MB for 524 GeoJSON files (~2-4 KB per file)

### Next Steps
After running this notebook:
- Proceed to **2-satellite-image-fetch.ipynb** - Uses GeoJSON files to extract imagery
- GeoJSON files ready for Google Earth Engine

---

## Technical Details

### Coordinate Reference Systems

**EPSG:4326 (WGS84)**
- **Type:** Geographic coordinate system
- **Units:** Decimal degrees
- **Use:** Standard GPS coordinates, GeoJSON format

**EPSG:32645 (UTM Zone 45N)**
- **Type:** Projected coordinate system
- **Units:** Meters
- **Use:** Accurate metric calculations in Nepal region
- **Coverage:** 78°E to 84°E (includes all of Nepal)

### Buffer Resolution

**Default Shapely Buffer:**
- Creates polygon with multiple segments
- Segment count depends on distance and resolution parameter
- For 15.45m radius: Typically 16-32 vertices (smooth circle)

**Alternative (Higher Resolution):**
```python
aoi_polygon = row.geometry.buffer(radius_val, resolution=16)
```
- Higher resolution = smoother circle
- More vertices = larger file size
- Default resolution sufficient for most uses

### File Size Considerations

**Typical File Sizes:**
- Single GeoJSON: 2-4 KB
- 524 files total: ~1-2 MB
- Polygon complexity affects size

**Optimization:**
- Could reduce coordinate precision (if needed)
- Current precision appropriate for analysis

---

## Error Handling

### Potential Issues

1. **Coordinate Outliers**
   - **Symptom:** Buffers created in wrong location
   - **Cause:** Invalid coordinates (outside Nepal)
   - **Solution:** Validate coordinates before processing

2. **CRS Transformation Errors**
   - **Symptom:** Buffer not circular or wrong size
   - **Cause:** Incorrect CRS specification
   - **Solution:** Verify UTM zone for Nepal region

3. **File Writing Errors**
   - **Symptom:** Missing or corrupted GeoJSON files
   - **Cause:** Permissions or disk space
   - **Solution:** Check directory permissions and available space

### Validation Recommendations

1. **Verify File Count**
   ```python
   import os
   geojson_files = [f for f in os.listdir(GEOJSON_DIR_PATH) if f.endswith('.geojson')]
   assert len(geojson_files) == 524
   ```

2. **Check Sample Geometry**
   ```python
   import geopandas as gpd
   sample = gpd.read_file("data/geojson-data/plot-10-75.geojson")
   print(sample.geometry.area)  # Should be close to 750 m² (when projected to metric)
   ```

3. **Validate Coordinates**
   ```python
   # Check all coordinates are in Nepal bounds
   sample_bounds = sample.total_bounds
   assert 80 <= sample_bounds[0] <= 88  # Longitude
   assert 26 <= sample_bounds[1] <= 31  # Latitude
   ```

---

## Performance Considerations

### Execution Time
- **Coordinate transformation:** < 1 second
- **Per-file generation:** ~0.01-0.05 seconds per plot
- **Total time:** 5-30 seconds for 524 plots
- **Bottleneck:** File I/O operations

### Optimization Opportunities
- Could use vectorized operations (if GeoPandas supports it)
- Parallel processing for file writing (complex)
- Batch writing (less flexible for individual files)

### Scalability
- Handles hundreds of plots efficiently
- Processing time scales linearly with plot count
- Memory usage minimal (processes one plot at a time)

---

## Related Notebooks

- **Field-Data/4-data-preparation.ipynb** - Provides input plot-level data
- **Satellite-Data/2-satellite-image-fetch.ipynb** - Uses GeoJSON files for imagery extraction

---

## Best Practices Applied

1. **Coordinate System Handling:** Proper transformation ensures accurate geometries
2. **Directory Management:** Automatic directory creation prevents errors
3. **File Naming:** Consistent, descriptive naming convention
4. **Metadata Preservation:** All plot attributes included in GeoJSON properties
5. **Standard Format:** WGS84 GeoJSON compatible with most GIS tools
