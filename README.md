# California Parcel Dataset

Raw California parcel boundaries for academic research.

**2,626,755 parcels** across **33 counties** with full provenance metadata.

## Quick Start

```bash
# Download and explore (copy-paste this entire block)
mkdir -p ~/structric-data && cd ~/structric-data && \
curl -L -o parcels_raw.parquet "https://github.com/thefazzer/structric-raw-datasets/releases/download/v1.0.0/parcels_raw.parquet" && \
pip install duckdb pyarrow pandas -q && \
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
        MIN(area_sqft) as min_sqft,
        MAX(area_sqft) as max_sqft,
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

# Geospatial bounds
bounds = conn.execute("""
    SELECT 
        MIN(ST_XMin(ST_GeomFromWKB(geometry_wkb))) as min_lon,
        MAX(ST_XMax(ST_GeomFromWKB(geometry_wkb))) as max_lon,
        MIN(ST_YMin(ST_GeomFromWKB(geometry_wkb))) as min_lat,
        MAX(ST_YMax(ST_GeomFromWKB(geometry_wkb))) as max_lat
    FROM read_parquet('parcels_raw.parquet')
""").fetchone()

print(f"\nGeospatial Bounds (EPSG:4326):")
print(f"  Longitude:    {bounds[0]:>10.4f}° to {bounds[1]:.4f}°")
print(f"  Latitude:     {bounds[2]:>10.4f}° to {bounds[3]:.4f}°")

# County breakdown
print(f"\nTop 10 Counties by Parcel Count:")
print("-" * 40)
counties = conn.execute("""
    SELECT county, COUNT(*) as cnt, 
           (COUNT(*) * 100.0 / SUM(COUNT(*)) OVER())::DECIMAL(5,2) as pct
    FROM read_parquet('parcels_raw.parquet')
    GROUP BY county ORDER BY cnt DESC LIMIT 10
""").fetchall()
for c in counties:
    print(f"  {c[0]:<20} {c[1]:>10,}  ({c[2]}%)")

# Schema
print(f"\nDataset Schema:")
print("-" * 40)
schema = conn.execute("DESCRIBE SELECT * FROM read_parquet('parcels_raw.parquet')").fetchall()
for col in schema:
    print(f"  {col[0]:<25} {col[1]}")

print("\n" + "=" * 70)
print(f"Data location: ~/structric-data/parcels_raw.parquet")
print("=" * 70)
EOF
```

## Schema

| Field | Type | Description |
|-------|------|-------------|
| `apn` | string | Assessor's Parcel Number |
| `geometry_wkb` | binary | Parcel boundary (WKB, EPSG:4326) |
| `area_sqft` | float | Parcel area in square feet |
| `city` | string | City name |
| `county` | string | County name |
| `state` | string | "California" |
| `zoning_raw` | string | Raw zoning (NULL - not in source) |
| `source_system` | string | Origin system |
| `source_table` | string | Origin table |
| `source_id` | string | Record ID in source |
| `inferred_flag` | boolean | Always FALSE |
| `license_note` | string | License information |
| `ingested_at` | timestamp | Export timestamp |

## Provenance

- **Source**: State of California / Regrid parcel data
- **License**: Open Database License (ODbL)
- **Filtering**: Valid geometry, 1,200 sqft ≤ area < 50M sqft
- **No inference**: All fields are directly from source (inferred_flag = FALSE)

## Loading in Python

```python
import duckdb

conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial; LOAD spatial;")

# Query LA County parcels
la_parcels = conn.execute("""
    SELECT apn, city, area_sqft, ST_AsText(ST_GeomFromWKB(geometry_wkb)) as wkt
    FROM read_parquet('~/structric-data/parcels_raw.parquet')
    WHERE county = 'LOS ANGELES'
    LIMIT 10
""").fetchdf()
```

## Citation

```
California Parcel Dataset, v1.0.0 (2026).
Derived from California State Parcel Data / Regrid.
https://github.com/thefazzer/structric-raw-datasets
```
