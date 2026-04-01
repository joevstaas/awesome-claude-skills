---
description: Consume, query, and download data from Ocean Data Platform (ODP) / HUB Ocean — covers the Python SDK, STAC API, tabular queries, file downloads, and spatial data reconstruction
---

# ODP Data Consume Skill

Use this skill when the user wants to read, query, download, or pull data from the Ocean Data Platform (ODP) by Hub Ocean.

## Prerequisites

### Python Dependencies

```bash
pip install odp-sdk pyarrow shapely pandas python-dotenv
```

| Package | Purpose |
|---------|---------|
| `odp-sdk` | ODP client library (authentication, catalog, dataset operations) |
| `pyarrow` | Tabular data deserialization |
| `shapely` | Convert WKT geometry back to GeoJSON |
| `pandas` | DataFrame handling for tabular query results |

### Authentication

```python
import os
from odp.client import Client

client = Client(api_key=os.environ["ODP_API_KEY"])
```

## Two Ways to Access Data

| Method | Best for | Authentication |
|--------|----------|----------------|
| **Python SDK** | Downloading files, querying tabular data, programmatic access | API key required |
| **STAC API** | Discovering collections, spatial/temporal search, browsing catalog | No auth for public data |

## Python SDK: Inspecting Table Schema

Read the schema to see column names, types, and metadata (descriptions, geometry markers):

```python
ds = client.dataset("dataset-uuid")
schema = ds.table.schema()

for field in schema:
    meta = field.metadata or {}
    desc = meta.get(b"description", b"").decode()
    # Note: use str(field.type), not field.type directly in f-strings
    print(f"  {field.name:25s} {str(field.type):10s} {desc}")
```

## Python SDK: Querying Tabular Data

### Select All Rows (No Filter)

The tabular API returns data in batches via a cursor. Iterate to collect all rows. **Only use this for small datasets** — for large datasets, always use server-side filters (see Filtering Rows below):

```python
ds = client.dataset("dataset-uuid")

cursor = ds.table.select()
df = None

for batch_df in cursor.dataframes():
    if df is None:
        df = batch_df
    else:
        import pandas as pd
        df = pd.concat([df, batch_df], ignore_index=True)

print(f"Loaded {len(df)} rows with columns: {list(df.columns)}")
```

### Filtering Rows (Preferred for Large Datasets)

**IMPORTANT:** Always use server-side filters when querying large datasets. Datasets can have millions of rows — scanning client-side is extremely slow and will likely time out. The `select()` method accepts a `filter` parameter with SQL/Arrow-style expressions, including geospatial operations. This pushes filtering to the server.

```python
# Basic comparison
cursor = ds.table.select(filter='depth_m > 10')

# Combined filters
cursor = ds.table.select(filter='count >= 5 AND method == "Undervannsvideo"')

# Null checks
cursor = ds.table.select(filter='notes is not null')
```

#### Parameterized Queries

Use `vars` to pass variables safely:

```python
# Named variables
cursor = ds.table.select(
    filter='depth_m >= $min_depth AND depth_m <= $max_depth',
    vars={"min_depth": 5.0, "max_depth": 15.0}
)

# Positional variables
cursor = ds.table.select(
    filter='year >= ? AND year < ?',
    vars=[2020, 2025]
)
```

### Geospatial Filtering (Server-Side)

The filter language supports spatial operators on geometry columns. Pass a WKT polygon as the filter value:

```python
bbox_wkt = 'POLYGON ((10.639 59.912, 10.639 59.904, 10.660 59.904, 10.660 59.912, 10.639 59.912))'

# Find observations within a bounding box
cursor = ds.table.select(filter=f'geometry within "{bbox_wkt}"')
```

| Operator | Syntax | Description |
|----------|--------|-------------|
| `within` | `geometry within "POLYGON (...)"` | Points/polygons inside the given polygon |
| `intersect` | `geometry intersect "POLYGON (...)"` | Geometries that overlap |
| `contains` | `geometry contains "POLYGON (...)"` | Geometries that enclose the given polygon |

#### Combining Column + Geo Filters

Column and geo filters can be combined in a single query for maximum efficiency:

```python
# Filter by taxonomy AND geography in one server-side query
cursor = ds.table.select(
    filter='family = "Acipenseridae" AND geometry within "POLYGON ((-74.5 39.5, -72.0 39.5, -72.0 41.0, -74.5 41.0, -74.5 39.5))"'
)
```

#### Converting GeoJSON to WKT for Filters

```python
from shapely.geometry import shape
from shapely import wkt

geojson_geom = {"type": "Polygon", "coordinates": [[[10.639, 59.912], ...]]}
bbox_wkt = wkt.dumps(shape(geojson_geom))
cursor = ds.table.select(filter=f'geometry within "{bbox_wkt}"')
```

## Python SDK: Aggregation

Server-side aggregation with optional grouping. Returns a pandas DataFrame.

```python
ds.table.aggregate(
    filter='...',           # optional, same filter syntax as select()
    group_by='field_name',  # optional, column to group by
    aggr={'column': 'func'} # aggregation functions to apply
)
```

### Aggregation Functions

| Function | Description |
|----------|-------------|
| `sum` | Sum of values |
| `avg` | Average of values |
| `min` | Minimum value |
| `max` | Maximum value |
| `count` | Count of non-null values |

### Examples

```python
# Total across all rows (no group by)
result = ds.table.aggregate(aggr={'count': 'sum', 'depth_m': 'avg'})

# Group by a column
result = ds.table.aggregate(group_by='observer', aggr={'count': 'avg'})

# Combine filter with aggregation
result = ds.table.aggregate(
    filter='method == "Undervannsvideo"',
    group_by='observer',
    aggr={'depth_m': 'avg'}
)
```

### H3 Spatial Aggregation

Group by hexagonal grid cells using H3. Resolution ranges from 0 (coarsest) to 15 (finest):

```python
result = ds.table.aggregate(
    group_by='h3(geometry, 5)',
    aggr={'count': 'sum'}
)
# Returns H3 hex IDs as the index (e.g., "8509990ffffffff")
```

### Result Format

The returned DataFrame has:
- **Index**: unique values of the grouped field (or `"TOTAL"` when no group_by)
- **`*` column**: row count per group
- **Aggregated columns**: one column per entry in `aggr`

### Working with the Results

The returned DataFrames have standard pandas types. Watch out for:

```python
for _, row in df.iterrows():
    value = row["column_name"]

    # Handle NaN values (common in nullable columns)
    if isinstance(value, float) and (value != value):  # NaN check
        value = None

    # Handle numpy types if needed
    if hasattr(value, 'item'):
        value = value.item()  # Convert numpy scalar to Python type
```

## Python SDK: Downloading Files

### List Files in a Dataset

**Note:** `ds.files.list()` returns dicts, not objects. Use `f["name"]`, not `f.name`.

```python
ds = client.dataset("dataset-uuid")
files = list(ds.files.list())

for f in files:
    print(f"  {f['name']} (id: {f['id']}, size: {f['size']} bytes)")
```

### Download a Specific File

**Note:** `download()` returns an `urllib3.HTTPResponse`, not bytes. Call `.read()` to get the content.

```python
# Download by file ID
response = ds.files.download(file_id)
content = response.read()  # returns bytes

# Parse as JSON
import json
data = json.loads(content.decode("utf-8"))
```

### Download by File Extension

```python
files = list(ds.files.list())
geojson_files = [f for f in files if f["name"].endswith(".geojson")]

if geojson_files:
    response = ds.files.download(geojson_files[0]["id"])
    geojson = json.loads(response.read().decode("utf-8"))
```

### Delete a File

```python
ds.files.delete(file_id)  # permanent deletion
```

### Update File Metadata

Supported fields: `name`, `format`. Setting `description` will raise an error.

```python
ds.files.update_meta(file_id, {"name": "renamed_file.geojson", "format": "geojson"})
```

## Python SDK: Reconstructing GeoJSON from Tabular Data

When spatial data was ingested as tabular (WKT geometry), reconstruct GeoJSON. Use server-side filters to limit the data before reconstruction:

```python
import json
from shapely import wkt
from shapely.geometry.base import BaseGeometry

ds = client.dataset("dataset-uuid")
# Use filters to avoid downloading the entire dataset
cursor = ds.table.select(
    filter='family = "Acipenseridae" AND geometry within "POLYGON ((-74.5 39.5, -72.0 39.5, -72.0 41.0, -74.5 41.0, -74.5 39.5))"'
)

features = []
for batch_df in cursor.dataframes():
    for _, row in batch_df.iterrows():
        # Convert WKT geometry back to GeoJSON
        geometry = None
        geom_value = row.get("geometry")
        if geom_value is not None:
            try:
                if isinstance(geom_value, str):
                    geom = wkt.loads(geom_value)
                elif isinstance(geom_value, BaseGeometry):
                    geom = geom_value
                else:
                    geom = geom_value
                geometry = json.loads(json.dumps(geom.__geo_interface__))
            except Exception as e:
                print(f"Warning: Could not parse geometry: {e}")

        # Build properties from remaining columns
        skip_cols = {"id", "source_dataset", "source_name", "geometry"}
        properties = {}
        for col in batch_df.columns:
            if col not in skip_cols:
                val = row[col]
                if hasattr(val, 'item'):
                    val = val.item()
                if isinstance(val, float) and (val != val):
                    val = None
                properties[col] = val

        features.append({
            "type": "Feature",
            "geometry": geometry,
            "properties": properties,
        })

geojson = {
    "type": "FeatureCollection",
    "features": features,
}
```

## Python SDK: Looking Up Datasets

### Find a Dataset by Name

```python
from odp.catalog_v2 import get_dataset_meta_by_name

meta = get_dataset_meta_by_name(client, "My Dataset Name")
if meta:
    print(f"Found: {meta.id}")
    ds = client.dataset(meta.id)
```

### Access a Dataset by UUID

```python
ds = client.dataset("dataset-uuid-here")
```

## STAC API: Discovering and Searching Data

The STAC API is a REST API for browsing the ODP catalog without authentication (for public data).

**Base URL:** `https://api.hubocean.earth/api/stac`

### List All Collections

```bash
curl -s "https://api.hubocean.earth/api/stac/collections" \
  | jq '.collections[] | {id, title, description}'
```

### Get a Single Collection

```bash
curl -s "https://api.hubocean.earth/api/stac/collections/{collection-id}"
```

### Search Items with Spatial Filter

```bash
curl -X POST "https://api.hubocean.earth/api/stac/search" \
  -H "Content-Type: application/json" \
  -d '{
    "collections": ["collection-uuid"],
    "bbox": [minLon, minLat, maxLon, maxLat],
    "datetime": "2023-01-01T00:00:00Z/2024-01-01T00:00:00Z",
    "limit": 100
  }'
```

### Search with GeoJSON Geometry

```bash
curl -X POST "https://api.hubocean.earth/api/stac/search" \
  -H "Content-Type: application/json" \
  -d '{
    "intersects": {
      "type": "Polygon",
      "coordinates": [[[10.2, 59.0], [10.9, 59.0], [10.9, 59.5], [10.2, 59.5], [10.2, 59.0]]]
    },
    "limit": 50
  }'
```

### STAC Search Parameters

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `collections` | array | Collection UUIDs to search | `["uuid-1"]` |
| `ids` | array | Specific item UUIDs | `["item-uuid"]` |
| `bbox` | array | `[minLon, minLat, maxLon, maxLat]` | `[10.2, 59.0, 10.9, 59.9]` |
| `intersects` | object | GeoJSON geometry | `{"type": "Point", ...}` |
| `datetime` | string | ISO 8601 range | `"2023-01-01T00:00:00Z/2024-01-01T00:00:00Z"` |
| `limit` | integer | Max results | `100` |
| `offset` | integer | Pagination offset | `20` |

### Common Bounding Boxes

| Area | Bbox |
|------|------|
| Oslo Fjord | `[10.2, 59.0, 10.9, 59.9]` |
| Norwegian Coast | `[3, 57, 31, 71]` |
| North Sea | `[-4, 51, 9, 62]` |
| Southern Ocean | `[-180, -90, 180, -60]` |
| Global | `[-180, -90, 180, 90]` |

### STAC Response Structure

**Collection:**
```json
{
  "id": "uuid",
  "title": "Dataset Name",
  "description": "...",
  "license": "ODC-BY-1.0",
  "extent": {
    "spatial": {"bbox": [[-180, -90, 180, 90]]},
    "temporal": {"interval": [["2021-01-01T00:00:00Z", null]]}
  },
  "keywords": ["oceanography", "marine"]
}
```

**Search results (FeatureCollection):**
```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "id": "item-uuid",
      "geometry": {"type": "Point", "coordinates": [10.7, 59.9]},
      "properties": {"datetime": "2023-06-15T12:00:00Z"},
      "links": [...],
      "assets": {...}
    }
  ]
}
```

## Caching Pattern

When serving ODP data to a frontend or repeatedly accessing the same datasets, cache results in memory:

```python
from datetime import datetime

_cache: dict = {}
_cache_timestamps: dict = {}
CACHE_TTL_SECONDS = 3600  # 1 hour

def get_data_cached(dataset_id: str):
    # Check cache
    if dataset_id in _cache_timestamps:
        age = (datetime.now() - _cache_timestamps[dataset_id]).total_seconds()
        if age < CACHE_TTL_SECONDS and dataset_id in _cache:
            return _cache[dataset_id]

    # Fetch from ODP
    data = fetch_from_odp(dataset_id)

    # Update cache
    _cache[dataset_id] = data
    _cache_timestamps[dataset_id] = datetime.now()
    return data
```

## Resilience: ODP with Local Fallback

For production use, try ODP first and fall back to local files:

```python
def load_data(dataset_id: str):
    try:
        return load_from_odp(dataset_id)
    except Exception as e:
        print(f"ODP unavailable ({e}), falling back to local file...")
        return load_from_local_file(dataset_id)
```

## Tips and Gotchas

- **Always use server-side filters on large datasets** — datasets can have millions of rows. Scanning client-side is extremely slow and will likely time out. Use the `filter` parameter on `ds.table.select()` with column filters and/or geo filters. Column and geo filters can be combined in a single expression with `AND`.
- **Tabular data comes in batches** — always iterate `cursor.dataframes()` and concatenate. A single batch may not contain all rows.
- **Geometry may be WKT or Shapely objects** — the SDK sometimes returns parsed `BaseGeometry` objects instead of WKT strings. Handle both cases.
- **NaN values are common** — nullable columns return `float('nan')` for missing values. Always check with `val != val` or `pd.isna(val)`.
- **numpy scalars** — pandas DataFrames may contain numpy types. Use `.item()` to convert to native Python types before JSON serialization.
- **STAC vs SDK** — use STAC for discovery (what data exists, spatial search), use the SDK for actual data download and tabular queries.
- **File listing returns dicts** — `ds.files.list()` returns a generator of dicts (not objects). Use `f["name"]` and `f["id"]`, not `f.name` or `f.id`. Wrap in `list()` to materialize.
- **PyArrow field.type in f-strings** — `field.type` does not support format specifiers. Use `str(field.type)` when formatting (e.g., `f"{str(field.type):10s}"`).
- **Rate limiting** — cache aggressively. ODP data typically changes infrequently (daily or less).

## Related Skills

- **odp-data-ingest** — uploading and ingesting data into ODP
- **odp-stac-api** — detailed STAC API reference for spatial/temporal search
