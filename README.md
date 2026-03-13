# Sentinel-2 Change Detection Pipeline

A geospatial analytics pipeline for detecting land-surface changes at an open-pit mining site in Zambia's Copperbelt, using Sentinel-2 satellite imagery.

## Overview

This project implements a complete change detection workflow:

1. **Data Preparation** - Load, validate, and stack Sentinel-2 bands (Blue, Green, Red)
2. **Change Detection** - Spectral Euclidean Distance with Otsu's automatic thresholding
3. **Feature Extraction** - Convert raster changes to vector polygons with attributes
4. **Database Storage** - Store results in a queryable SQLite database
5. **Visualization** - Interactive HTML map and static analysis plots
6. **Interpretation** - Analytical report with findings and limitations

## Repository Structure

```
project/
│
├── inputs/                          # Raw input data
│   ├── aoi.geojson                  # Area of Interest boundary
│   └── data/
│       ├── sentinel2_20230812/      # Date 1: Aug 12, 2023
│       │   ├── B02.tif              #   Blue band
│       │   ├── B03.tif              #   Green band
│       │   ├── B04.tif              #   Red band
│       │   └── B08.tif              #   NIR band (Custom Fetch)
│       └── sentinel2_20230902/      # Date 2: Sep 2, 2023
│           ├── B02.tif
│           ├── B03.tif
│           ├── B04.tif
│           └── B08.tif
│
├── data/
│   └── processed/                   # Generated outputs
│       ├── sentinel2_20230812_stack.tif   # Stacked 3-band raster (Date 1)
│       ├── sentinel2_20230902_stack.tif   # Stacked 3-band raster (Date 2)
│       ├── change_map.tif                 # Change intensity (continuous)
│       ├── change_binary.tif              # Binary change map (0/1)
│       ├── change_per_band.tif            # Per-band difference (3 bands)
│       ├── confidence_map.tif             # Confidence scores (0-1)
│       └── change_polygons.geojson        # Vectorized change areas
│
├── scripts/
│   ├── data_preparation.py          # Step 1: Load, validate, stack bands
│   ├── change_detection.py          # Step 2: Change detection + thresholding
│   ├── vector_extraction.py         # Step 3: Raster → vector + SQLite storage
│   └── visualization.py             # Step 4: Interactive map + static plots
│
├── database/
│   └── change_detection.sqlite      # SQLite database with change features
│
├── visualizations/
│   ├── change_detection_map.html    # Interactive folium map
│   ├── rgb_comparison.png           # Side-by-side RGB composites
│   └── change_analysis.png          # Change intensity + binary panels
│
├── report.md                        # Analysis and interpretation
├── README.md                        # This file
└── requirements.txt                 # Python dependencies
```

## Installation

### Prerequisites

- Python 3.9 or higher
- pip (Python package manager)

### Setup

```bash
# Clone the repository
git clone <repo-url>
cd project

# Install dependencies
pip install -r requirements.txt
```

## Required Libraries

| Library | Purpose |
|---------|---------|
| rasterio | Reading/writing GeoTIFF rasters |
| numpy | Numerical array operations |
| geopandas | Vector geospatial data handling |
| shapely | Geometric operations on polygons |
| scipy | Median filter for noise removal |
| matplotlib | Static plot generation |
| folium | Interactive web map creation |

## How to Run

Execute the scripts **in order** from the project root directory:

```bash
# Step 1: Load and stack Sentinel-2 bands
python scripts/data_preparation.py

# Step 2: Run change detection
python scripts/change_detection.py

# Step 3: Extract vectors and store in database
python scripts/vector_extraction.py

# Step 4: Generate visualizations
python scripts/visualization.py
```

Each script prints progress and validation information to the terminal. The interactive map can be opened in any web browser:

```bash
# Open interactive map (Linux)
xdg-open visualizations/change_detection_map.html

# Open interactive map (macOS)
open visualizations/change_detection_map.html

# Open interactive map (Windows)
start visualizations/change_detection_map.html
```

## Approach

### Change Detection Method

**Spectral Euclidean Distance** was chosen as the primary change detection method. For each pixel, the 3D Euclidean distance between the Date 1 and Date 2 spectral values (Blue, Green, Red) is computed:

```
distance = sqrt( (B2_d1 - B2_d2)² + (B3_d1 - B3_d2)² + (B4_d1 - B4_d2)² )
```

This method uses all available bands simultaneously and captures any type of spectral change without requiring specific non-visible indices (like NDVI, which needs NIR).

### Secondary Method: NDVI and GRVI
Because Band 8 (NIR) was not originally provided, the source data was augmented with manually fetched, resampled, and re-projected `B08.tif` data to unlock the **Normalized Difference Vegetation Index (NDVI)**. By segmenting NDVI classes, we can explicitly compute transitions (e.g., Vegetation to Bare Ground). As an optical backup, the **Visible Atmospherically Resistant Index (VARI) / Green-Red Vegetation Index (GRVI)** was also computed. The final change map represents the intersection of these indices with the Spectral Distance.

### Threshold Selection

**Otsu's method** automatically determines the optimal threshold for binary classification by maximizing the between-class variance of the change intensity histogram. This is data-driven and avoids arbitrary threshold selection.

### Confidence Scoring

Each change polygon receives a confidence score (0–1) based on how far its pixel intensities exceed the threshold, normalized by the maximum observed intensity. Higher confidence indicates stronger, more certain change.

## Assumptions

1. **CRS consistency** - All input bands share the same Coordinate Reference System (EPSG:32735, UTM Zone 35S). This is verified programmatically.

2. **NoData handling** - Pixels with value 0 in any band are treated as NoData and excluded from analysis. Both dates have the same NoData footprint (26,069 pixels at the image edges).

3. **Atmospheric conditions** - The analysis assumes both dates have comparable atmospheric conditions. As noted in the report, Date 1 (August 12) shows apparent atmospheric haze, which inflates the detected change proportion. Dark Channel Prior (DCP) dehazing was applied to combat this.

4. **Input Constraints (Merging scan gap)** - The September (Date 2) imagery features horizontal spacing constraints / mosaic scan gaps. While algorithms were applied to mask these rows during polygonization, residual data near the seams still mathematically triggers as physical change. These are documented data artifacts, not true geographical movement.

5. **Minimum polygon size** - Polygons smaller than 500 m² (approximately 50 pixels) are filtered out to reduce noise in the vector output.

6. **Temporal context** - A 21-day interval between dates is short enough that most detected spectral changes represent either real surface modification (e.g. mining) or the atmospheric/scan effects noted above, rather than long-term seasonal transitions.

## Querying the Database

The SQLite database can be queried directly:

```python
import sqlite3

conn = sqlite3.connect("database/change_detection.sqlite")

# Find large change areas
cursor = conn.execute(
    "SELECT id, area_m2, confidence FROM change_features WHERE area_m2 > 10000 ORDER BY area_m2 DESC"
)
for row in cursor:
    print(f"ID: {row[0]}, Area: {row[1]:,.0f} m², Confidence: {row[2]:.4f}")

# Summary statistics
cursor = conn.execute(
    "SELECT COUNT(*), AVG(area_m2), AVG(confidence) FROM change_features"
)
count, avg_area, avg_conf = cursor.fetchone()
print(f"Polygons: {count}, Avg area: {avg_area:,.0f} m², Avg confidence: {avg_conf:.4f}")

conn.close()
```

## Future Extensions

- **Machine learning classification** - Supervised classification of change types (mining expansion, vegetation loss, water change)
- **Atmospheric correction** - Pre-processing using Scene Classification Layer (SCL) to mask hazy/cloudy pixels
- **Multi-temporal analysis** - Using 5+ dates for more robust time-series change detection
- **QGIS plugin** - Wrapping the pipeline into an interactive QGIS plugin with a graphical interface
- **Machine learning classification** - Supervised classification of change types (mining expansion, vegetation loss, water change)
