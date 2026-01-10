# Code Documentation: 2-satellite-image-fetch.ipynb

## Overview

This notebook downloads multi-spectral satellite imagery for each plot from Google Earth Engine (GEE). It handles authentication, filters image collections by date and cloud cover, selects optimal images, and downloads specified spectral bands as GeoTIFF files for each plot location.

**Location:** `notebooks/Satellite-Data/2-satellite-image-fetch.ipynb`

---

## Purpose

- Authenticate with Google Earth Engine using service account credentials
- Filter satellite image collections by temporal, spatial, and quality criteria
- Select best available images (lowest cloud cover) for each plot
- Download multi-spectral bands as GeoTIFF files
- Organize downloaded imagery in a structured directory hierarchy
- Implement checkpointing to resume interrupted downloads

**Critical Functionality:**
This notebook bridges field data with satellite imagery, enabling the extraction of spectral features for machine learning model training.

---

## Dependencies

### Libraries Used

```python
import os
import json
import ee
import requests
from tqdm import tqdm
from dotenv import load_dotenv
```

**Libraries:**
- **os** - File path operations and directory management
- **json** - JSON file parsing (service account key)
- **ee** - Google Earth Engine Python API
- **requests** - HTTP requests for downloading GeoTIFF URLs
- **tqdm** - Progress bar for download tracking
- **dotenv** - Environment variable loading

---

## Configuration

### Environment Variables
Requires `.env` file with:
```python
PROJECT_ID='your-gcp-project-id'
SERVICE_ACCOUNT='your-service-account-email'
```

### Service Account Key
**File Path:** `service-account-key.json` (project root)

**Required Fields:**
- `project_id`
- `client_email`
- Private key credentials

### Satellite Configuration
```python
SATELLITE = "Landsat-8"  # Options: "Sentinel-2", "Landsat-8"
BANDS = [
    "SR_B2",  # Blue
    "SR_B3",  # Green
    "SR_B4",  # Red
    "SR_B5",  # Near-Infrared
    "SR_B6",  # Shortwave Infrared 1
    "SR_B7",  # Shortwave Infrared 2
]
YEAR = 2014
CLOUD_FILTER = 10  # Maximum cloud cover percentage
```

---

## Input Data

### Directory Path
```python
GEOJSON_DIR = "../../data/geojson-data"
```

### Input Format
- **File format:** GeoJSON files
- **Number of files:** 524 (one per plot)
- **File naming:** `plot-{plotId}.geojson`
- **Content:** Circular polygon geometries for each plot

### Required Files
Each GeoJSON file must contain:
- **Geometry:** Polygon (circular buffer around plot center)
- **CRS:** WGS84 (EPSG:4326)
- **Properties:** Plot metadata (plotId, coordinates, AGB, etc.)

---

## Output Data

### Directory Structure
```python
OUTPUT_DIR = "../../data/satellite-images"
```

### Output Hierarchy
```
data/satellite-images/
└── 2014/
    ├── plot-10-75/
    │   ├── plot-10-75_SR_B2.tif
    │   ├── plot-10-75_SR_B3.tif
    │   ├── plot-10-75_SR_B4.tif
    │   ├── plot-10-75_SR_B5.tif
    │   ├── plot-10-75_SR_B6.tif
    │   └── plot-10-75_SR_B7.tif
    ├── plot-10-84/
    │   └── ...
    └── ...
```

### Output Format
- **File format:** GeoTIFF (.tif)
- **Bands per plot:** 6 (for Landsat-8 configuration)
- **Spatial resolution:** 30m (Landsat-8) or 10m (Sentinel-2)
- **Total files:** 524 plots × 6 bands = 3,144 files

### File Naming Convention
```
{plotId}_{bandName}.tif
```
Example: `plot-10-75_SR_B2.tif`

---

## Code Structure

### Cell 0-1: Setup
**Markdown:** "Importing Necessary Dependencies"

Imports all required libraries.

### Cell 2: Load Environment Variables
**Markdown:** "Load the environment variables"

### Cell 3: Load Environment Configuration
```python
load_dotenv()
PROJECT_ID = os.getenv("PROJECT_ID")
SERVICE_ACCOUNT = os.getenv("SERVICE_ACCOUNT")
```

**Operations:**
- Loads variables from `.env` file
- Retrieves GCP project ID
- Retrieves service account email

**Note:** These variables are also read from service account key JSON (more reliable method used later).

### Cell 4: Define Service Account Key Path
```python
base_dir = os.path.abspath(os.path.join(os.getcwd(), "..", ".."))
key_path = os.path.join(base_dir, "service-account-key.json")
```

**Operations:**
- Constructs absolute path to project root
- Locates service account key file
- Platform-independent path handling

### Cell 5: Define Paths Section
**Markdown:** "Input and Output Path"

### Cell 6: Define Input/Output Paths
```python
GEOJSON_DIR = "../../data/geojson-data"
OUTPUT_DIR = "../../data/satellite-images"
```

### Cell 7: Configuration Section
**Markdown:** "📌 Important: Configurations"

### Cell 8: Satellite and Band Configuration
```python
SATELLITE = "Landsat-8"  # Options: "Sentinel-2", "Landsat-8"
BANDS = [
    "SR_B2",
    "SR_B3",
    "SR_B4",
    "SR_B5",
    "SR_B6",
    "SR_B7",
]
YEAR = 2014
CLOUD_FILTER = 10
```

**Configuration Details:**

**SATELLITE:**
- **Landsat-8:** Primary sensor (30m resolution)
- **Sentinel-2:** Alternative option (10m resolution)
- Determines image collection and band names

**BANDS (Landsat-8 Surface Reflectance):**
- **SR_B2:** Blue band (0.452-0.512 μm)
- **SR_B3:** Green band (0.533-0.590 μm)
- **SR_B4:** Red band (0.636-0.673 μm)
- **SR_B5:** Near-Infrared (0.851-0.879 μm)
- **SR_B6:** Shortwave Infrared 1 (1.566-1.651 μm)
- **SR_B7:** Shortwave Infrared 2 (2.107-2.294 μm)

**YEAR:**
- **2014:** Matches field data collection period
- Used for temporal filtering

**CLOUD_FILTER:**
- **10%:** Maximum acceptable cloud cover
- Ensures relatively clear imagery

### Cell 9: Authentication Section
**Markdown:** "Service Account Authentication for Google Earth Engine"

### Cell 10: Initialize Earth Engine
```python
def init_ee():
    try:
        # Read the file as a dictionary
        with open(key_path, "r") as f:
            key_data = json.load(f)

        # Get Project ID and Service Account from the JSON itself
        project_id = key_data["project_id"]
        service_account = key_data["client_email"]

        # Initialize
        credentials = ee.ServiceAccountCredentials(
            email=service_account, key_data=json.dumps(key_data)
        )
        ee.Initialize(credentials, project=project_id)

        print(f"✅ Success! Connected to project: {project_id}")
        print(f"🔑 Using Service Account: {service_account}")

    except FileNotFoundError:
        print(f"❌ Error: Could not find file at {key_path}")
    except Exception as e:
        print(f"❌ Authentication failed: {e}")
```

**Step-by-Step Authentication:**

1. **Load Service Account Key**
   ```python
   with open(key_path, "r") as f:
       key_data = json.load(f)
   ```
   - Reads JSON file containing credentials
   - Loads as Python dictionary

2. **Extract Credentials**
   ```python
   project_id = key_data["project_id"]
   service_account = key_data["client_email"]
   ```
   - Gets GCP project ID from key file
   - Gets service account email
   - **Safer approach:** Reads from key file directly (avoids .env inconsistencies)

3. **Create Credentials Object**
   ```python
   credentials = ee.ServiceAccountCredentials(
       email=service_account, 
       key_data=json.dumps(key_data)
   )
   ```
   - Creates Earth Engine credentials from service account
   - Requires email and full key data

4. **Initialize Earth Engine**
   ```python
   ee.Initialize(credentials, project=project_id)
   ```
   - Authenticates with Google Earth Engine
   - Associates with specific GCP project
   - Enables API access

**Error Handling:**
- **FileNotFoundError:** Service account key missing
- **General Exception:** Authentication failures (invalid key, permissions, etc.)

### Cell 11: Image Collection Filtering Section
**Markdown:** "Dynamic Satellite Image Collection Filtering by Sensor and Metadata"

### Cell 12: Get Filtered Image Collection
```python
def get_collection(satellite, geom, start_date, end_date):
    """Retrieve filtered ImageCollection based on satellite type."""
    if satellite == "Sentinel-2":
        coll_id = "COPERNICUS/S2_SR_HARMONIZED"
        cloud_prop = "CLOUDY_PIXEL_PERCENTAGE"
    else:
        coll_id = "LANDSAT/LC08/C02/T1_L2"
        cloud_prop = "CLOUD_COVER"

    return (
        ee.ImageCollection(coll_id)
        .filterBounds(geom)
        .filterDate(start_date, end_date)
        .filter(ee.Filter.lt(cloud_prop, CLOUD_FILTER))
        .sort(cloud_prop)
    )
```

**Function Parameters:**
- `satellite`: Sensor type ("Landsat-8" or "Sentinel-2")
- `geom`: Earth Engine Geometry object (plot polygon)
- `start_date`: Start date string (YYYY-MM-DD)
- `end_date`: End date string (YYYY-MM-DD)

**Step-by-Step Filtering:**

1. **Select Image Collection**
   ```python
   if satellite == "Sentinel-2":
       coll_id = "COPERNICUS/S2_SR_HARMONIZED"
       cloud_prop = "CLOUDY_PIXEL_PERCENTAGE"
   else:
       coll_id = "LANDSAT/LC08/C02/T1_L2"
       cloud_prop = "CLOUD_COVER"
   ```
   - **Sentinel-2:** Copernicus Surface Reflectance Harmonized collection
   - **Landsat-8:** Landsat Collection 2 Level-2 Surface Reflectance
   - Different cloud property names for each sensor

2. **Spatial Filter**
   ```python
   .filterBounds(geom)
   ```
   - Filters images that intersect plot geometry
   - Only includes images covering plot area

3. **Temporal Filter**
   ```python
   .filterDate(start_date, end_date)
   ```
   - Filters images within date range
   - Used for seasonal window selection

4. **Cloud Cover Filter**
   ```python
   .filter(ee.Filter.lt(cloud_prop, CLOUD_FILTER))
   ```
   - Keeps only images with cloud cover < 10%
   - Ensures relatively clear imagery

5. **Sort by Cloud Cover**
   ```python
   .sort(cloud_prop)
   ```
   - Orders images by cloud cover (ascending)
   - Best image (lowest clouds) first

**Returns:** Filtered, sorted ImageCollection

### Cell 13: Band Download Section
**Markdown:** "Automated Multi-Spectral Band Extraction and GeoTIFF Export"

### Cell 14: Download Bands Function
```python
def download_bands(image, geom, plot_id, output_path):
    """Download specified bands as GeoTIFFs."""
    region = geom.bounds().getInfo()["coordinates"]

    for band in BANDS:
        # Scale: 10m for S2 visible/NIR, 30m for Landsat
        scale = 10 if SATELLITE == "Sentinel-2" else 30

        try:
            url = image.select(band).getDownloadURL(
                {"scale": scale, "region": region, "format": "GEO_TIFF"}
            )

            r = requests.get(url, timeout=30)
            if r.status_code == 200:
                filename = f"{plot_id}_{band}.tif"
                with open(os.path.join(output_path, filename), "wb") as f:
                    f.write(r.content)
        except Exception as e:
            print(f"  ⚠️ Error downloading {band} for {plot_id}: {e}")
```

**Function Parameters:**
- `image`: Earth Engine Image object
- `geom`: Earth Engine Geometry (plot polygon)
- `plot_id`: Plot identifier string (for file naming)
- `output_path`: Directory path for saving files

**Step-by-Step Band Download:**

1. **Get Bounding Box**
   ```python
   region = geom.bounds().getInfo()["coordinates"]
   ```
   - Extracts bounding box coordinates from geometry
   - `.getInfo()` executes Earth Engine computation (client-side)
   - Returns coordinate array for download region

2. **Iterate Through Bands**
   ```python
   for band in BANDS:
   ```
   - Loops through configured band list
   - Downloads each band separately

3. **Set Pixel Scale**
   ```python
   scale = 10 if SATELLITE == "Sentinel-2" else 30
   ```
   - **Sentinel-2:** 10 meters (native resolution for visible/NIR)
   - **Landsat-8:** 30 meters (native resolution)
   - Determines output pixel size

4. **Get Download URL**
   ```python
   url = image.select(band).getDownloadURL({
       "scale": scale,
       "region": region,
       "format": "GEO_TIFF"
   })
   ```
   - Selects single band from image
   - Requests download URL from Earth Engine
   - **Parameters:**
     - `scale`: Output pixel resolution
     - `region`: Bounding box coordinates
     - `format`: GeoTIFF format

5. **Download File**
   ```python
   r = requests.get(url, timeout=30)
   if r.status_code == 200:
       filename = f"{plot_id}_{band}.tif"
       with open(os.path.join(output_path, filename), "wb") as f:
           f.write(r.content)
   ```
   - Downloads GeoTIFF from URL
   - **Timeout:** 30 seconds (prevents hanging)
   - **Success check:** HTTP 200 status code
   - **File writing:** Binary mode ('wb') for GeoTIFF
   - **File naming:** `{plotId}_{bandName}.tif`

6. **Error Handling**
   ```python
   except Exception as e:
       print(f"  ⚠️ Error downloading {band} for {plot_id}: {e}")
   ```
   - Catches download/network errors
   - Prints warning (continues with other bands)
   - Prevents single band failure from stopping entire process

### Cell 15: Initialization and Setup
```python
# Initialize Earth Engine with your secure key
init_ee()
os.makedirs(OUTPUT_DIR, exist_ok=True)
geojson_files = [f for f in os.listdir(GEOJSON_DIR) if f.endswith('.geojson')]
print(f"📂 Found {len(geojson_files)} plots to process.")

start_date = f"{YEAR}-01-01"
end_date = f"{YEAR}-12-31"
```

**Operations:**
1. Authenticates with Earth Engine
2. Creates output directory if needed
3. Lists all GeoJSON files in input directory
4. Prints number of plots to process
5. Defines full-year date range (used for initial filtering, refined later)

**Output:**
```
✅ Success! Connected to project: geopulse-477105
🔑 Using Service Account: geopulse-service@geopulse-477105.iam.gserviceaccount.com
📂 Found 524 plots to process.
```

### Cell 16: Main Processing Loop
```python
year_folder = os.path.join(OUTPUT_DIR, str(YEAR))
os.makedirs(year_folder, exist_ok=True)

for file_name in tqdm(geojson_files, desc=f"Processing Plots for {YEAR}"):
    plot_id = file_name.replace(".geojson", "").replace("plot_", "")
    
    # Nested directory structure: YEAR -> PLOT_ID
    plot_folder = os.path.join(year_folder, plot_id)
    
    # Skip if plot already processed (Checkpointing)
    if os.path.exists(plot_folder) and len(os.listdir(plot_folder)) >= len(BANDS):
        continue
    
    os.makedirs(plot_folder, exist_ok=True)
    
    # Load GeoJSON geometry
    with open(os.path.join(GEOJSON_DIR, file_name), "r") as f:
        data = json.load(f)
        geom_dict = (
            data["features"][0]["geometry"] if "features" in data else data["geometry"]
        )
        
        # Remove Z-coords
        if len(geom_dict["coordinates"][0][0]) > 2:
            geom_dict["coordinates"] = [
                [[coord[0], coord[1]] for coord in ring]
                for ring in geom_dict["coordinates"]
            ]
        
        geom = ee.Geometry(geom_dict)
    
    # Define seasonal window (Post-Monsoon Dry Season)
    seasonal_start = f"{YEAR}-11-01"
    seasonal_end = f"{YEAR + 1}-03-31"
    
    # Get best image
    collection = get_collection(SATELLITE, geom, seasonal_start, seasonal_end)
    
    if collection.size().getInfo() > 0:
        best_img = ee.Image(collection.first()).clip(geom)
        download_bands(best_img, geom, plot_id, plot_folder)
    else:
        # Clean up empty folder if no imagery found
        if not os.listdir(plot_folder):
            os.rmdir(plot_folder)
        print(f"  ❌ No clear imagery found for {plot_id} in {YEAR}")
```

**Step-by-Step Processing:**

1. **Create Year Directory**
   ```python
   year_folder = os.path.join(OUTPUT_DIR, str(YEAR))
   os.makedirs(year_folder, exist_ok=True)
   ```
   - Creates directory for year (e.g., `2014/`)
   - Organizes files by year

2. **Iterate Through Plots**
   ```python
   for file_name in tqdm(geojson_files, desc=f"Processing Plots for {YEAR}"):
   ```
   - Loops through all GeoJSON files
   - `tqdm` shows progress bar
   - Processes one plot at a time

3. **Extract Plot ID**
   ```python
   plot_id = file_name.replace(".geojson", "").replace("plot_", "")
   ```
   - Removes file extension
   - Removes "plot_" prefix (if present)
   - Example: `plot-10-75.geojson` → `plot-10-75`

4. **Create Plot Directory**
   ```python
   plot_folder = os.path.join(year_folder, plot_id)
   ```
   - Creates plot-specific subdirectory
   - Structure: `2014/plot-10-75/`

5. **Checkpointing**
   ```python
   if os.path.exists(plot_folder) and len(os.listdir(plot_folder)) >= len(BANDS):
       continue
   ```
   - **Skip if complete:** Checks if plot folder exists and has all bands
   - **Resume capability:** Allows interrupted runs to continue
   - **Efficiency:** Avoids re-downloading existing files

6. **Create Plot Folder**
   ```python
   os.makedirs(plot_folder, exist_ok=True)
   ```
   - Creates directory for this plot's bands

7. **Load GeoJSON Geometry**
   ```python
   with open(os.path.join(GEOJSON_DIR, file_name), "r") as f:
       data = json.load(f)
       geom_dict = data["features"][0]["geometry"] if "features" in data else data["geometry"]
   ```
   - Reads GeoJSON file
   - Extracts geometry (handles both FeatureCollection and Feature formats)

8. **Remove Z-Coordinates**
   ```python
   if len(geom_dict["coordinates"][0][0]) > 2:
       geom_dict["coordinates"] = [
           [[coord[0], coord[1]] for coord in ring]
           for ring in geom_dict["coordinates"]
       ]
   ```
   - **Issue:** Some geometries include elevation (Z coordinate)
   - **Earth Engine:** Requires 2D coordinates (lon, lat)
   - **Solution:** Strips Z coordinates, keeps only X, Y

9. **Create Earth Engine Geometry**
   ```python
   geom = ee.Geometry(geom_dict)
   ```
   - Converts GeoJSON dict to Earth Engine Geometry
   - Used for spatial filtering

10. **Define Seasonal Window**
    ```python
    seasonal_start = f"{YEAR}-11-01"  # November 2014
    seasonal_end = f"{YEAR + 1}-03-31"  # March 2015
    ```
    - **Post-Monsoon Dry Season:** November to March
    - **Rationale:**
      - Lower cloud cover
      - Better atmospheric conditions
      - Optimal leaf conditions for vegetation analysis

11. **Get Filtered Image Collection**
    ```python
    collection = get_collection(SATELLITE, geom, seasonal_start, seasonal_end)
    ```
    - Filters images for this plot's location and season
    - Returns sorted collection (best image first)

12. **Select Best Image**
    ```python
    if collection.size().getInfo() > 0:
        best_img = ee.Image(collection.first()).clip(geom)
        download_bands(best_img, geom, plot_id, plot_folder)
    ```
    - **Check availability:** Ensures images exist
    - **Select best:** First image has lowest cloud cover
    - **Clip to geometry:** Restricts to plot area (reduces file size)
    - **Download bands:** Extracts all configured bands

13. **Handle Missing Imagery**
    ```python
    else:
        if not os.listdir(plot_folder):
            os.rmdir(plot_folder)
        print(f"  ❌ No clear imagery found for {plot_id} in {YEAR}")
    ```
    - **Cleanup:** Removes empty folder if no imagery found
    - **Notification:** Prints warning message
    - **Continuation:** Process continues with next plot

---

## Temporal Filtering Strategy

### Full Year vs. Seasonal Window

**Initial Date Range:**
- `start_date`: `2014-01-01`
- `end_date`: `2014-12-31`
- **Purpose:** Used for initial filtering (not actually used in final implementation)

**Seasonal Window (Actually Used):**
- `seasonal_start`: `2014-11-01` (November)
- `seasonal_end`: `2015-03-31` (March of following year)
- **Rationale:**
  - Post-monsoon period (October-December): Clear skies
  - Dry winter (January-March): Minimal cloud cover
  - Optimal for vegetation analysis

**Benefits:**
- Higher probability of clear images
- Better spectral quality
- Consistent temporal window across plots

---

## Checkpointing Mechanism

### How It Works

**Before Processing Each Plot:**
```python
if os.path.exists(plot_folder) and len(os.listdir(plot_folder)) >= len(BANDS):
    continue
```

**Checks:**
1. Plot directory exists
2. Directory contains at least `len(BANDS)` files (e.g., 6 for Landsat-8)

**If Complete:**
- Skips to next plot
- Saves time and API quota

**If Incomplete:**
- Processes plot (may re-download some files)
- Ensures all bands are present

### Benefits
- **Resume interrupted downloads:** Can restart notebook without losing progress
- **Efficiency:** Avoids redundant downloads
- **Robustness:** Handles network interruptions gracefully

---

## Error Handling

### Potential Issues and Solutions

1. **Authentication Failures**
   - **Symptom:** `❌ Authentication failed`
   - **Causes:** Invalid key, missing permissions, expired credentials
   - **Solution:** Verify service account key and GEE access

2. **Missing GeoJSON Files**
   - **Symptom:** No files found
   - **Causes:** Wrong directory path, files not generated
   - **Solution:** Run `1-geojson-extraction.ipynb` first

3. **No Imagery Available**
   - **Symptom:** `❌ No clear imagery found`
   - **Causes:** High cloud cover, temporal gaps, location issues
   - **Solution:** Adjust cloud filter, expand date range, verify coordinates

4. **Download Failures**
   - **Symptom:** `⚠️ Error downloading`
   - **Causes:** Network issues, URL expiration, Earth Engine quota
   - **Solution:** Retry download, check network, verify quota limits

5. **Memory Issues**
   - **Symptom:** Slow performance, crashes
   - **Causes:** Large image regions, too many concurrent requests
   - **Solution:** Process smaller batches, increase timeout

---

## Performance Considerations

### Execution Time
- **Per plot:** 5-30 seconds (depends on image availability, network speed)
- **Total (524 plots):** ~45 minutes - 4 hours
- **Bottlenecks:**
  - Earth Engine API calls (rate limited)
  - GeoTIFF download time
  - Image collection filtering

### Earth Engine Quota Limits
- **Compute Units:** Limited per day
- **Export Quota:** Limits on download requests
- **Best Practice:** Process in batches, use checkpointing

### Optimization Tips
1. **Use checkpointing:** Avoids re-downloading
2. **Batch processing:** Process subsets of plots
3. **Optimize date range:** Narrower ranges = faster filtering
4. **Parallel processing:** Not implemented (complex with GEE)

---

## Usage Notes

### When to Run
- **Second step** in Satellite Data pipeline
- Run after `1-geojson-extraction.ipynb`
- Required before feature extraction and modeling

### Prerequisites
- GeoJSON files must exist: `data/geojson-data/`
- Google Earth Engine access enabled
- Service account key configured
- Internet connection (for Earth Engine API and downloads)
- Sufficient disk space (~100-500 MB for all imagery)

### Configuration Before Running
1. Set `SATELLITE` variable (Landsat-8 or Sentinel-2)
2. Configure `BANDS` list (desired spectral bands)
3. Set `YEAR` to match field data period
4. Adjust `CLOUD_FILTER` if needed (default: 10%)

### Next Steps
After running this notebook:
- Verify downloaded imagery (check file counts)
- Proceed to feature extraction (spectral indices, statistics)
- Prepare data for machine learning model training

---

## Technical Details

### Earth Engine Image Collections

**Landsat-8 Collection:**
- **ID:** `LANDSAT/LC08/C02/T1_L2`
- **Product:** Collection 2 Level-2 Surface Reflectance
- **Processing:** Atmospherically corrected
- **Resolution:** 30m (all bands)

**Sentinel-2 Collection:**
- **ID:** `COPERNICUS/S2_SR_HARMONIZED`
- **Product:** Surface Reflectance Harmonized
- **Processing:** Atmospherically corrected, harmonized with Landsat
- **Resolution:** 10m (visible/NIR), 20m (SWIR)

### Spectral Bands

**Landsat-8 Bands Used:**
- **SR_B2 (Blue):** Water body mapping, atmospheric correction
- **SR_B3 (Green):** Vegetation health assessment
- **SR_B4 (Red):** Chlorophyll absorption
- **SR_B5 (NIR):** Vegetation biomass, structure
- **SR_B6 (SWIR1):** Moisture content, vegetation stress
- **SR_B7 (SWIR2):** Soil moisture, mineral mapping

### Coordinate Systems
- **Input Geometry:** WGS84 (EPSG:4326) - GeoJSON standard
- **Earth Engine:** Internal projection handling
- **Output GeoTIFF:** WGS84 or UTM (depends on location)

---

## Related Notebooks

- **Satellite-Data/1-geojson-extraction.ipynb** - Provides input GeoJSON files
- **Field-Data/4-data-preparation.ipynb** - Generates plot-level data used in GeoJSON creation

---

## Best Practices Applied

1. **Robust Authentication:** Reads credentials from key file (more reliable)
2. **Checkpointing:** Enables resume of interrupted processes
3. **Error Handling:** Continues processing despite individual failures
4. **Seasonal Filtering:** Optimizes for best image quality
5. **Organized Output:** Hierarchical directory structure for easy navigation
6. **Progress Tracking:** Visual progress bar for long-running operations
7. **Cloud Filtering:** Ensures high-quality imagery selection
