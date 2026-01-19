# California Parcel Dataset

Raw California parcel boundaries for academic research.

**2,626,755 parcels** across **33 counties** with full provenance metadata.

## Quick Start

```bash
# Download and explore (copy-paste this entire block)
mkdir -p ~/structric-data && cd ~/structric-data && \
curl -L --progress-bar -o parcels_raw.parquet "https://media.githubusercontent.com/media/thefazzer/structric-raw-datasets/main/data/parcels_raw.parquet" && \
echo "Installing duckdb..." && pip install duckdb -q && \
echo "Analyzing parcels..." && \
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

# Detect geometry column type
col_type = conn.execute("""
    SELECT typeof(geometry_wkb) FROM read_parquet('parcels_raw.parquet') LIMIT 1
""").fetchone()[0]
geom_expr = "geometry_wkb" if 'GEOMETRY' in col_type.upper() else "ST_GeomFromWKB(geometry_wkb)"

# Get bounds
bounds = conn.execute(f"""
    SELECT 
        MIN(ST_XMin({geom_expr})), MAX(ST_XMax({geom_expr})),
        MIN(ST_YMin({geom_expr})), MAX(ST_YMax({geom_expr}))
    FROM read_parquet('parcels_raw.parquet')
""").fetchone()

print(f"\nGeospatial Bounds (EPSG:4326):")
print(f"  Longitude:  {bounds[0]:>10.4f}° to {bounds[1]:.4f}°")
print(f"  Latitude:   {bounds[2]:>10.4f}° to {bounds[3]:.4f}°")

# County breakdown
print(f"\nTop 10 Counties:")
print("-" * 45)
counties = conn.execute("""
    SELECT county, COUNT(*) as cnt, 
           (COUNT(*) * 100.0 / SUM(COUNT(*)) OVER())::DECIMAL(5,2) as pct
    FROM read_parquet('parcels_raw.parquet')
    GROUP BY county ORDER BY cnt DESC LIMIT 10
""").fetchall()
for c in counties:
    print(f"  {c[0]:<20} {c[1]:>10,}  ({c[2]:>5}%)")

print("\n" + "=" * 70)
print("Data: ~/structric-data/parcels_raw.parquet (196 MB)")
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

## Provenance

- **Source**: State of California / Regrid parcel data
- **License**: Open Database License (ODbL)
- **Filtering**: Valid geometry, 1,200 ≤ area < 50M sqft
- **No inference**: All fields directly from source

## Example Queries

### Basic Queries

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

### Export to CSV

```python
import duckdb, time

conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial; LOAD spatial;")

def export_csv(query, filename, description):
    """Export query results to CSV with progress tracking."""
    # Count rows first
    count_query = f"SELECT COUNT(*) FROM ({query})"
    total = conn.execute(count_query).fetchone()[0]
    print(f"{description}: {total:,} rows")
    
    # Export with timing
    start = time.time()
    conn.execute(f"COPY ({query}) TO '{filename}' (HEADER, DELIMITER ',')")
    elapsed = time.time() - start
    rate = total / elapsed if elapsed > 0 else 0
    print(f"  → {filename} ({elapsed:.1f}s, {rate:,.0f} rows/sec)")

# Export LA County parcels to CSV (without geometry)
export_csv(
    """SELECT apn, city, county, area_sqft, source_system
       FROM read_parquet('parcels_raw.parquet')
       WHERE county = 'LOS ANGELES'""",
    "la_parcels.csv",
    "Exporting LA County"
)

# Export with geometry as WKT (slower due to geometry conversion)
export_csv(
    """SELECT apn, city, county, area_sqft, ST_AsText(geometry_wkb) as geometry_wkt
       FROM read_parquet('parcels_raw.parquet')
       WHERE county = 'ORANGE'""",
    "orange_parcels_with_geom.csv",
    "Exporting Orange County with geometry"
)
```

### ASCII Density Map

Generate a text-based visualization of parcel density across California:

```python
import duckdb, math

conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial; LOAD spatial;")

# Detect geometry type
col_type = conn.execute("""
    SELECT typeof(geometry_wkb) FROM read_parquet('parcels_raw.parquet') LIMIT 1
""").fetchone()[0]
geom_expr = "geometry_wkb" if 'GEOMETRY' in col_type.upper() else "ST_GeomFromWKB(geometry_wkb)"

# Get bounds
bounds = conn.execute(f"""
    SELECT 
        MIN(ST_XMin({geom_expr})), MAX(ST_XMax({geom_expr})),
        MIN(ST_YMin({geom_expr})), MAX(ST_YMax({geom_expr}))
    FROM read_parquet('parcels_raw.parquet')
""").fetchone()

# Build density grid
W, H = 48, 18
grid_data = conn.execute(f"""
    SELECT 
        FLOOR((ST_X(ST_Centroid({geom_expr})) - ({bounds[0]})) / ({bounds[1]} - {bounds[0]}) * {W})::INTEGER as gx,
        FLOOR(({bounds[3]} - ST_Y(ST_Centroid({geom_expr}))) / ({bounds[3]} - {bounds[2]}) * {H})::INTEGER as gy,
        COUNT(*) as cnt
    FROM read_parquet('parcels_raw.parquet')
    GROUP BY 1, 2
    HAVING gx >= 0 AND gx < {W} AND gy >= 0 AND gy < {H}
""").fetchall()

grid = [[0]*W for _ in range(H)]
max_cnt = 1
for gx, gy, cnt in grid_data:
    if 0 <= gx < W and 0 <= gy < H:
        grid[gy][gx] = cnt
        max_cnt = max(max_cnt, cnt)

# Render ASCII map
chars = ' .:-=+*#%@'
print(f"Parcel Density Map (log scale):")
print("┌" + "─"*W + "┐ N")
for y in range(H):
    row = "│"
    for x in range(W):
        if grid[y][x] == 0:
            row += ' '
        else:
            lvl = int(math.log10(grid[y][x]+1) / math.log10(max_cnt+1) * (len(chars)-1))
            row += chars[min(lvl, len(chars)-1)]
    if y == 0:
        row += f"│ {bounds[3]:.1f}°"
    elif y == H-1:
        row += f"│ {bounds[2]:.1f}°"
    else:
        row += "│"
    print(row)
print("└" + "─"*W + "┘ S")
print(f" {bounds[0]:.1f}°" + " "*(W-14) + f"{bounds[1]:.1f}°")
print(" W" + " "*(W-2) + "E")
print(f" Legend: ' '=none .=sparse @=dense ({max_cnt:,} max)")
```

Output:
```
Parcel Density Map (log scale):
┌────────────────────────────────────────────────┐ N
│=*==:         :--.+=-:                          │ 42.0°
│ ==+=         -+=*+===                          │
│=*=+-          ===++=:                          │
│=+=+=--:      -+++++==                          │
│  =+++====+=***=-=++==                          │
│  =+=*+**+=+=++*#+=+*#                          │  ← Bay Area
│    +:.-+*+-  *#***+=*===                       │
│       +*#####  =***+  -.==-                    │
│          #%#+  -+*+=+++==+:=+--.               │
│             + -+***+*#*+-: .==:::              │
│            ##++:==+=+##*+=  :=-  ..:-::        │
│             -=+==++++++-+  .-.-::--:::-=-      │
│                    -+=++##++*+=*+              │
│                       =+++++****+              │
│                           =####*=              │
│                          -*#%@%%#*             │  ← LA/OC
│                            := #%#       *= :..-│
│                            ::          :==#+===│ 32.7°
└────────────────────────────────────────────────┘ S
 -124.4°                                  -114.6°
 W                                              E
 Legend: ' '=none .=sparse @=dense (278,757 max)
```

## Citation

```
California Parcel Dataset, v1.0.0 (2026).
Source: California State Parcel Data / Regrid.
https://github.com/thefazzer/structric-raw-datasets
```
