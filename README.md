# California Parcel Dataset

Raw California parcel boundaries for academic research.

**2,626,755 parcels** across **33 counties** with full provenance metadata.

## Quick Start

```bash
# Download and explore (copy-paste this entire block)
mkdir -p ~/structric-data && cd ~/structric-data && \
curl -L -o parcels_raw.parquet "https://github.com/thefazzer/structric-raw-datasets/releases/download/v1.0.0/parcels_raw.parquet" && \
pip install duckdb -q && \
python3 << 'EOF'
import duckdb

conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial; LOAD spatial;")

print("=" * 70)
print("CALIFORNIA PARCEL DATASET SUMMARY")
print("=" * 70)

# Basic stats
stats = conn.execute("""
    SELECT 
        COUNT(*) as total_parcels,
        COUNT(DISTINCT county) as counties,
        COUNT(DISTINCT city) as cities,
        MIN(area_sqft)::INTEGER as min_sqft,
        MAX(area_sqft)::INTEGER as max_sqft,
        AVG(area_sqft)::INTEGER as avg_sqft,
        APPROX_QUANTILE(area_sqft, 0.5)::INTEGER as median_sqft
    FROM read_parquet('parcels_raw.parquet')
""").fetchone()

print(f"\nTotal Parcels:  {stats[0]:>12,}")
print(f"Counties:       {stats[1]:>12}")
print(f"Cities:         {stats[2]:>12}")
print(f"\nArea (sqft):")
print(f"  Min:          {stats[3]:>12,}")
print(f"  Max:          {stats[4]:>12,}")
print(f"  Mean:         {stats[5]:>12,}")
print(f"  Median:       {stats[6]:>12,}")

# Geospatial bounds (geometry_wkb is already GEOMETRY type)
bounds = conn.execute("""
    SELECT 
        MIN(ST_XMin(geometry_wkb))::DECIMAL(10,4) as min_lon,
        MAX(ST_XMax(geometry_wkb))::DECIMAL(10,4) as max_lon,
        MIN(ST_YMin(geometry_wkb))::DECIMAL(10,4) as min_lat,
        MAX(ST_YMax(geometry_wkb))::DECIMAL(10,4) as max_lat
    FROM read_parquet('parcels_raw.parquet')
""").fetchone()

print(f"\nGeospatial Bounds (EPSG:4326):")
print(f"  Longitude:    {bounds[0]:>12}° to {bounds[1]}°")
print(f"  Latitude:     {bounds[2]:>12}° to {bounds[3]}°")

# County breakdown
print(f"\nTop 10 Counties by Parcel Count:")
print("-" * 45)
counties = conn.execute("""
    SELECT county, COUNT(*) as cnt, 
           (COUNT(*) * 100.0 / SUM(COUNT(*)) OVER())::DECIMAL(5,2) as pct
    FROM read_parquet('parcels_raw.parquet')
    GROUP BY county ORDER BY cnt DESC LIMIT 10
""").fetchall()
for c in counties:
    print(f"  {c[0]:<20} {c[1]:>10,}  ({c[2]:>5}%)")

# Schema
print(f"\nDataset Schema:")
print("-" * 45)
schema = conn.execute("""
    DESCRIBE SELECT apn, geometry_wkb, area_sqft, city, county, 
                    source_system, inferred_flag, license_note
    FROM read_parquet('parcels_raw.parquet')
""").fetchall()
for col in schema:
    print(f"  {col[0]:<20} {col[1]}")

print("\n" + "=" * 70)
print("Data location: ~/structric-data/parcels_raw.parquet (196 MB)")
print("=" * 70)
EOF
```

## Schema

| Field | Type | Description |
|-------|------|-------------|
| `apn` | string | Assessor's Parcel Number |
| `geometry_wkb` | geometry | Parcel boundary (EPSG:4326) |
| `area_sqft` | float | Parcel area in square feet |
| `city` | string | City name |
| `county` | string | County name |
| `state` | string | "California" |
| `source_system` | string | Origin system |
| `source_table` | string | Origin table |
| `source_id` | string | Record ID in source |
| `inferred_flag` | boolean | Always FALSE |
| `license_note` | string | License information |
| `ingested_at` | timestamp | Export timestamp |

## Provenance

- **Source**: State of California / Regrid parcel data
- **License**: Open Database License (ODbL)
- **Filtering**: Valid geometry, 1,200 ≤ area < 50M sqft
- **No inference**: All fields directly from source

## Example Queries

```python
import duckdb

conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial; LOAD spatial;")

# LA County parcels > 10,000 sqft
large_la = conn.execute("""
    SELECT apn, city, area_sqft
    FROM read_parquet('~/structric-data/parcels_raw.parquet')
    WHERE county = 'LOS ANGELES' AND area_sqft > 10000
    LIMIT 100
""").fetchdf()

# Parcels in a bounding box (downtown LA)
downtown = conn.execute("""
    SELECT apn, city, area_sqft, ST_AsText(geometry_wkb) as wkt
    FROM read_parquet('~/structric-data/parcels_raw.parquet')
    WHERE ST_Intersects(geometry_wkb, 
          ST_MakeEnvelope(-118.26, 34.04, -118.24, 34.06))
""").fetchdf()
```

## Citation

```
California Parcel Dataset, v1.0.0 (2026).
Source: California State Parcel Data / Regrid.
https://github.com/thefazzer/structric-raw-datasets
```
