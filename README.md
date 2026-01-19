# Structric Raw Datasets for Academic Research

**Version**: 1.0.0  
**Created**: 2026-01-19  
**Contact**: Structric Data Team  

## Purpose

This repository provides **raw, provenance-explicit datasets** derived from Structric's California property data holdings. These datasets are designed for academic research requiring:

- Transparent data lineage
- Minimal preprocessing
- Independent verification capability
- Python/R-friendly formats

## Philosophy: Raw Over Polished

These datasets prioritize **transparency over convenience**:

1. **No silent inference** - Every derived field is explicitly flagged
2. **No cross-source merging** - Parcels and buildings remain unlinked
3. **No normalization** - Zoning codes remain as raw strings from source
4. **Full provenance** - Every record carries source system metadata

If you need linked parcel-building data, you must perform the spatial join yourself (and document your methodology).

## Datasets

### 1. `parcels_raw.parquet` (Primary)

California parcel boundaries from state-level parcel data.

| Field | Type | Description |
|-------|------|-------------|
| `apn` | string | Assessor's Parcel Number |
| `geometry_wkb` | binary | Parcel boundary (WKB, EPSG:4326) |
| `area_sqft` | float | Parcel area in square feet |
| `city` | string | City name (raw from source) |
| `county` | string | County name |
| `state` | string | Always "California" |
| `zoning_raw` | string | Raw zoning code (NULL - not available in source) |
| `zoning_source` | string | Source of zoning data |
| `land_use_raw` | string | Raw land use code (NULL - not available) |
| `source_system` | string | Origin system identifier |
| `source_table` | string | Origin table name |
| `source_id` | string | Record ID in source system |
| `spatial_resolution` | string | "parcel-level" |
| `effective_date` | date | Date data was effective (NULL if unknown) |
| `ingested_at` | timestamp | When this export was created |
| `inferred_flag` | boolean | Whether any fields are inferred |
| `inference_method` | string | Description of inference (NULL if none) |
| `license_note` | string | Data license information |

**Filtering Applied**:
- Geometry must be valid (`ST_IsValid`)
- Area >= 1,200 sqft (minimum residential lot)
- Area < 50,000,000 sqft (excludes massive parcels)

**Row Count**: ~2.63 million parcels  
**Coverage**: 33 California counties  
**Source**: State of California / Regrid parcel data

### 2. `buildings_raw.parquet` (Building Footprints)

Building footprints from Microsoft's ML-derived building dataset.

| Field | Type | Description |
|-------|------|-------------|
| `building_id` | int | Sequential ID for this export |
| `parcel_apn` | string | **Always NULL** - no linkage performed |
| `geometry_wkb` | binary | Building footprint (WKB, EPSG:4326) |
| `footprint_area_sqft` | float | Footprint area in square feet |
| `height_m` | float | Building height (NULL - not in source) |
| `stories` | int | Number of stories (NULL - not in source) |
| `building_source` | string | "microsoft_building_footprints" |
| `source_system` | string | Origin system identifier |
| `source_table` | string | Origin table pattern |
| `source_id` | string | Record ID in this export |
| `spatial_resolution` | string | "building-level" |
| `inferred_flag` | boolean | Always FALSE (no inference) |
| `inference_method` | string | Always NULL |
| `ms_confidence` | float | Microsoft's ML confidence score |
| `ms_release` | int | Microsoft dataset release number |
| `ms_capture_dates` | string | Imagery capture date info |
| `ingested_at` | timestamp | When this export was created |
| `license_note` | string | "Microsoft Open Buildings - ODbL" |

**Filtering Applied**:
- Geometry must be valid
- Footprint area >= 100 sqft (min detectable structure)
- Footprint area < 1,000,000 sqft (excludes anomalies)

**Row Count**: ~11.54 million buildings  
**Coverage**: California statewide  
**Source**: Microsoft Building Footprints (ML-derived from satellite imagery)

## What IS Inferred vs. Observed

### Observed (direct from source):
- Parcel boundaries and APNs
- City/County names
- Building footprint geometries
- Microsoft confidence scores

### Computed (documented transformation):
- `area_sqft` / `footprint_area_sqft`: Computed from geometry using WGS84→California Albers projection conversion factor (~1.087e11 sqft per square degree at 35°N)

### NOT Inferred (explicitly NULL):
- `parcel_apn` in buildings dataset - **We do NOT link buildings to parcels**
- `zoning_raw` - Not available in state parcel source
- `height_m`, `stories` - Not in Microsoft footprints

## Known Limitations

1. **No parcel-building linkage**: Buildings are not spatially joined to parcels. If you need this, perform your own centroid-in-polygon or intersection analysis.

2. **Area calculation approximation**: Area is computed using a fixed conversion factor for ~35°N latitude. For parcels at extreme north (42°N) or south (32.5°N) of California, error may be ±5%.

3. **Zoning data gap**: State-level parcel data does not include zoning. Zoning would require county-by-county integration.

4. **Building heights unavailable**: Microsoft footprints are 2D only. Height/stories would require LIDAR or permit data integration.

5. **Temporal coverage**: Parcel data represents a snapshot (effective date unknown). Building footprints were captured at various dates (see `ms_capture_dates`).

6. **No residential filtering**: Parcels include all land uses. The source does not provide reliable land use classification.

## Loading Examples

### Python (pandas + geopandas)

```python
import pandas as pd
import geopandas as gpd
from shapely import wkb

# Load parcels
parcels = pd.read_parquet('data/parcels_raw.parquet')
parcels['geometry'] = parcels['geometry_wkb'].apply(wkb.loads)
parcels_gdf = gpd.GeoDataFrame(parcels, geometry='geometry', crs='EPSG:4326')

# Load buildings
buildings = pd.read_parquet('data/buildings_raw.parquet')
buildings['geometry'] = buildings['geometry_wkb'].apply(wkb.loads)
buildings_gdf = gpd.GeoDataFrame(buildings, geometry='geometry', crs='EPSG:4326')

# Example: filter to Los Angeles County
la_parcels = parcels_gdf[parcels_gdf['county'] == 'LOS ANGELES']
print(f"LA County parcels: {len(la_parcels):,}")
```

### Python (DuckDB - faster for large datasets)

```python
import duckdb

conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial; LOAD spatial;")

# Query parcels
result = conn.execute("""
    SELECT county, COUNT(*) as parcel_count, AVG(area_sqft) as avg_area
    FROM read_parquet('data/parcels_raw.parquet')
    GROUP BY county
    ORDER BY parcel_count DESC
""").fetchdf()
print(result)

# Spatial query on buildings
buildings_in_bbox = conn.execute("""
    SELECT building_id, footprint_area_sqft, ms_confidence
    FROM read_parquet('data/buildings_raw.parquet')
    WHERE ST_Within(
        ST_GeomFromWKB(geometry_wkb),
        ST_MakeEnvelope(-118.5, 33.9, -118.1, 34.2)  -- LA area
    )
""").fetchdf()
```

### R (arrow + sf)

```r
library(arrow)
library(sf)

# Load parcels
parcels <- read_parquet("data/parcels_raw.parquet")
parcels_sf <- st_as_sf(parcels, wkb = "geometry_wkb", crs = 4326)

# Load buildings  
buildings <- read_parquet("data/buildings_raw.parquet")
buildings_sf <- st_as_sf(buildings, wkb = "geometry_wkb", crs = 4326)
```

## Provenance Schema

Every record in every dataset contains these provenance fields:

| Field | Purpose |
|-------|---------|
| `source_system` | Which system/database the data came from |
| `source_table` | Specific table within that system |
| `source_id` | Primary key in the source system |
| `spatial_resolution` | Granularity of the data |
| `inferred_flag` | TRUE if any fields were inferred/derived |
| `inference_method` | Description of inference methodology |
| `license_note` | Data licensing information |
| `ingested_at` | Timestamp of this export |

## File Sizes

| File | Size | Rows |
|------|------|------|
| `parcels_raw.parquet` | 196 MB | 2,626,755 |
| `buildings_raw.parquet` | 893 MB | 11,542,719 |

**Total**: ~1.1 GB

### Data Download

Data files exceed GitHub's file size limits and are hosted via **GitHub Releases**.

**Download the data**:
1. Go to the [Releases page](../../releases)
2. Download `parcels_raw.parquet` and `buildings_raw.parquet`
3. Place them in the `data/` directory

Or use the command line:
```bash
# After cloning the repo
cd structric-raw-datasets/data
gh release download v1.0.0
```

```python
# Example: Query from cloud storage (if hosted on S3/GCS)
import duckdb
conn = duckdb.connect()
conn.execute("INSTALL httpfs; LOAD httpfs;")
df = conn.execute("""
    SELECT * FROM read_parquet('https://storage.example.com/parcels_raw.parquet')
    WHERE county = 'LOS ANGELES'
    LIMIT 1000
""").fetchdf()
```

## License

- **Parcel data**: State of California / Regrid - Open Database License (ODbL)
- **Building footprints**: Microsoft Open Buildings - Open Database License (ODbL)

Both sources permit academic and commercial use with attribution.

## Citation

If using this data in academic work, please cite:

```
Structric Raw California Property Datasets, v1.0.0 (2026).
Derived from California State Parcel Data and Microsoft Building Footprints.
https://github.com/structric/structric-raw-datasets
```

## Changelog

### v1.0.0 (2026-01-19)
- Initial release
- California parcels from state-level data (2.63M records)
- Microsoft building footprints for California (11.54M records)
- No parcel-building linkage (by design)
