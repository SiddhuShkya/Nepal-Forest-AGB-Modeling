# Code Documentation Index

This directory contains comprehensive code documentation for all notebooks in the Nepal Forest AGB Modeling project. Each documentation file provides detailed explanations of notebook functionality, code structure, dependencies, inputs/outputs, and usage guidelines.

---

## Field Data Processing Notebooks

### 1. [1-data-understanding.md](./1-data-understanding.md)
**Notebook:** `notebooks/Field-Data/1-data-understanding.ipynb`

**Purpose:** Initial exploration and understanding of raw field inventory data.

**Key Topics:**
- Dataset loading and inspection
- Hierarchical plot structure (cluster-plot-subplot)
- Variable descriptions and characteristics
- Plot ID decomposition demonstration

**When to Read:** Start here to understand the dataset structure before processing.

---

### 2. [2-data-preprocessing.md](./2-data-preprocessing.md)
**Notebook:** `notebooks/Field-Data/2-data-preprocessing.ipynb`

**Purpose:** Clean and preprocess raw field data.

**Key Topics:**
- Data cleaning operations (removing nulls, duplicates)
- Column removal (SOC column)
- Hierarchical ID decomposition
- Data transformation summary

**When to Read:** Understanding data cleaning steps and transformations.

---

### 3. [3-data-visualization.md](./3-data-visualization.md)
**Notebook:** `notebooks/Field-Data/3-data-visualization.ipynb`

**Purpose:** Create visualizations of field data characteristics.

**Key Topics:**
- Hierarchical structure visualization
- AGB distribution analysis
- Spatial mapping of plot locations
- Statistical insights and observations

**When to Read:** Reviewing data distributions and geographic coverage.

---

### 4. [4-data-preparation.md](./4-data-preparation.md)
**Notebook:** `notebooks/Field-Data/4-data-preparation.ipynb`

**Purpose:** Aggregate subplot data to plot level for satellite matching.

**Key Topics:**
- Plot-level aggregation methodology
- Area-weighted AGB averaging
- Coordinate calculation (plot centroids)
- Rationale for plot-level aggregation

**When to Read:** Critical for understanding final ground-truth dataset preparation.

---

## Satellite Data Processing Notebooks

### 5. [1-geojson-extraction.md](./1-geojson-extraction.md)
**Notebook:** `notebooks/Satellite-Data/1-geojson-extraction.ipynb`

**Purpose:** Create circular Area of Interest (AOI) polygons for each plot.

**Key Topics:**
- Coordinate system transformations (WGS84 ↔ UTM)
- Circular buffer creation
- GeoJSON file generation
- Geometry accuracy considerations

**When to Read:** Understanding spatial geometry preparation for satellite extraction.

---

### 6. [2-satellite-image-fetch.md](./2-satellite-image-fetch.md)
**Notebook:** `notebooks/Satellite-Data/2-satellite-image-fetch.ipynb`

**Purpose:** Download multi-spectral satellite imagery from Google Earth Engine.

**Key Topics:**
- Google Earth Engine authentication
- Image collection filtering (temporal, spatial, cloud cover)
- Spectral band extraction
- Download organization and checkpointing

**When to Read:** Understanding satellite data acquisition process and configuration.

---

## Documentation Structure

Each documentation file follows a consistent structure:

1. **Overview** - High-level purpose and context
2. **Purpose** - Specific objectives
3. **Dependencies** - Required libraries and packages
4. **Input Data** - Data sources and formats
5. **Output Data** - Generated files and formats
6. **Code Structure** - Detailed explanation of each code cell
7. **Technical Details** - Implementation specifics
8. **Usage Notes** - When to run and prerequisites
9. **Error Handling** - Common issues and solutions
10. **Related Notebooks** - Connections to other components

---

## Workflow Order

### Recommended Reading/Execution Order:

1. **Data Understanding** → [1-data-understanding.md](./1-data-understanding.md)
2. **Data Preprocessing** → [2-data-preprocessing.md](./2-data-preprocessing.md)
3. **Data Visualization** → [3-data-visualization.md](./3-data-visualization.md) *(can be parallel)*
4. **Data Preparation** → [4-data-preparation.md](./4-data-preparation.md)
5. **GeoJSON Extraction** → [1-geojson-extraction.md](./1-geojson-extraction.md)
6. **Satellite Image Fetch** → [2-satellite-image-fetch.md](./2-satellite-image-fetch.md)

---

## Quick Reference

### File Locations

| Documentation | Notebook Location |
|--------------|-------------------|
| [1-data-understanding.md](./1-data-understanding.md) | `notebooks/Field-Data/1-data-understanding.ipynb` |
| [2-data-preprocessing.md](./2-data-preprocessing.md) | `notebooks/Field-Data/2-data-preprocessing.ipynb` |
| [3-data-visualization.md](./3-data-visualization.md) | `notebooks/Field-Data/3-data-visualization.ipynb` |
| [4-data-preparation.md](./4-data-preparation.md) | `notebooks/Field-Data/4-data-preparation.ipynb` |
| [1-geojson-extraction.md](./1-geojson-extraction.md) | `notebooks/Satellite-Data/1-geojson-extraction.ipynb` |
| [2-satellite-image-fetch.md](./2-satellite-image-fetch.md) | `notebooks/Satellite-Data/2-satellite-image-fetch.ipynb` |

### Data Flow

```
Raw Data → Understanding → Preprocessing → Visualization
                                            ↓
                                    Preparation
                                            ↓
                          GeoJSON Extraction → Satellite Fetch
```

---

## Key Concepts

### Hierarchical Structure
- **Clusters:** 195 unique clusters
- **Plots:** 524 unique plots (after aggregation)
- **Subplots:** 2,009 unique subplots

### Data Transformations
- **Subplot-level** (2,009 records) → **Plot-level** (524 records)
- **Point coordinates** → **Circular polygons** (750 m²)
- **Plot-level data** → **Satellite imagery** (6 bands per plot)

### Coordinate Systems
- **WGS84 (EPSG:4326):** Standard GPS coordinates, GeoJSON format
- **UTM Zone 45N (EPSG:32645):** Metric calculations for Nepal region

---

## Additional Resources

- **Project Methodology:** See `project-methodology.md` in project root
- **README:** See main `README.md` for project overview
- **Requirements:** See `requirements.txt` for dependencies

---

## Contributing

When adding new notebooks:
1. Create corresponding markdown documentation file
2. Follow the standard documentation structure
3. Update this README with new entry
4. Ensure cross-references to related notebooks

---

**Last Updated:** 2024  
**Project:** Nepal Forest AGB Modeling
