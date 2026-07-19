---
description: Ingest data into Ocean Data Platform (ODP) / HUB Ocean using the ODP Python SDK — covers datasets, file uploads, tabular data, and spatial data
---

# ODP Data Ingest Skill

Use this skill when the user wants to ingest, upload, or manage datasets in the Ocean Data Platform (ODP) by Hub Ocean.

## Prerequisites

### Python Dependencies

```bash
pip install odp-sdk pyarrow shapely python-dotenv h3
```

| Package | Purpose |
|---------|---------|
| `odp-sdk` | ODP client library (authentication, catalog, dataset operations) |
| `pyarrow` | Define table schemas and serialize tabular data |
| `shapely` | Convert geometries between GeoJSON and WKT (for spatial data) |
| `h3` | Convert H3 cell ids to boundary polygons when precomputing H3-aggregated datasets |

### Authentication

The ODP SDK authenticates via API key:

```python
from odp.client import Client

client = Client(api_key="your-api-key")
```

Store the key in an environment variable (`ODP_API_KEY`) and load via `python-dotenv` or similar.

## Core Concepts

### Data Collections and Datasets

ODP organizes data hierarchically:

- **Data Collection** — a logical grouping of related datasets (identified by UUID)
- **Dataset** — a single data entity within a collection, containing files and/or tabular data

### Data Storage Options

Each dataset can hold two types of data:

| Type | Use case | API |
|------|----------|-----|
| **Files** | Raw file storage (GeoJSON, CSV, images, etc.) | `ds.files.upload()` / `ds.files.download()` |
| **Tabular** | Structured rows with a PyArrow schema, supports spatial queries | `ds.table.create()` / `ds.insert()` |

You can use both in the same dataset (e.g., store the raw GeoJSON file *and* a queryable table).

## Ingest Workflow

### Step 1: Create a Dataset

Check if a dataset already exists by name, then create if needed:

```python
import requests
from odp.catalog_v2 import get_dataset_meta_by_name

# Check for existing dataset
# Returns a DatasetMeta dataclass (not a dict) — access fields via .id, .name, .description
existing = get_dataset_meta_by_name(client, "My Dataset Name")

if not existing:
    # Create new dataset
    res = client._request(
        requests.Request(
            method="POST",
            url=client.base_url + "/api/catalog/v2/datasets",
            json={
                "name": "My Dataset Name",
                "description": "Description of the dataset",
            },
        ),
        retry=False,
    )
    res.raise_for_status()
    dataset_id = res.json()["id"]

    # Add to a data collection
    res2 = client._request(
        requests.Request(
            method="POST",
            url=client.base_url + f"/api/catalog/v2/data-collections/{collection_uid}/datasets/{dataset_id}",
        ),
        retry=False,
    )
    res2.raise_for_status()
else:
    dataset_id = existing.id  # DatasetMeta is a dataclass, use attribute access
```

### Step 2: Upload Raw Files

Upload any file to the dataset's file storage:

```python
ds = client.dataset(dataset_id)

with open("data.geojson", "rb") as f:
    file_id = ds.files.upload("data.geojson", f)
```

Upload also accepts raw bytes: `ds.files.upload("hello.txt", b"Hello World!")`

**Note:** `ds.files.update_meta()` has limited field support. Supported fields: `name`, `format`. Setting `"description"` will raise an error.

### Step 3: Upload Tabular Data

Define a PyArrow schema and insert rows:

```python
import pyarrow as pa

schema = pa.schema([
    pa.field("id", pa.string(), nullable=False,
             metadata={"description": "Unique UUID primary key."}),
    pa.field("name", pa.string(), nullable=True,
             metadata={"description": "Human-readable station name."}),
    pa.field("value", pa.float64(), nullable=True,
             metadata={"description": "Measured value (unit: °C).", "aggr": "mean"}),
    pa.field("geometry", pa.string(), nullable=True,
             metadata={"isGeometry": "1", "index": "1", "class": "geometry",
                       "description": "Point location in WKT format (POINT (lon lat))."}),
])

# Create the table (idempotent if schema matches)
ds.table.create(schema)

# Insert rows within a transaction
rows = [
    {"id": "uuid-1", "name": "Station A", "value": 12.5, "geometry": "POINT (10.7 59.9)"},
    {"id": "uuid-2", "name": "Station B", "value": 8.3, "geometry": "POINT (10.8 59.8)"},
]

with ds as tx:
    tx.insert(rows)
```

### Step 4: Delete a Dataset (for re-ingestion)

```python
res = client._request(
    requests.Request(
        method="DELETE",
        url=client.base_url + f"/api/catalog/v2/datasets/{dataset_id}",
    ),
    retry=False,
)
res.raise_for_status()
```

## Column-Level Metadata

PyArrow field metadata sets descriptions, classifications, and aggregation hints visible in the ODP table explorer.

**Rule: every column must have a `description`.** Without it, the table explorer shows blank column headers with no context. Write descriptions in plain English; include units for numerics, possible values for categoricals, and what NULL means for nullable columns.

**Rule: every spatial column must have the correct `class`.** The portal uses `class` to power its geometry/map features. Missing it means the column is not recognised as spatial even if `isGeometry` is set.

### Supported Metadata Keys

| Key | Values | Effect in ODP |
|-----|--------|---------------|
| `description` | Any string | Column description shown in the table explorer — **required on every column** |
| `class` | `"geometry"`, `"latitude"`, `"longitude"`, `"h3_index"` | Spatial classification — **required on every spatial column** |
| `isGeometry` | `"1"` | Marks the WKT geometry column (use together with `class: geometry`) |
| `index` | `"1"` | Creates a spatial index (use together with `isGeometry`) |
| `aggr` | `"sum"`, `"mean"`, `"min"`, `"max"`, `"count"` | Aggregation hint for numeric columns |

### Spatial Column Quick Reference

| Column holds | `class` | Also set | Type |
|---|---|---|---|
| WKT geometry string (POINT / LINESTRING / …) | `"geometry"` | `"isGeometry": "1"`, `"index": "1"` | `pa.string()` |
| Latitude float | `"latitude"` | — | `pa.float64()` |
| Longitude float | `"longitude"` | — | `pa.float64()` |
| H3 cell index string | `"h3_index"` | — | `pa.string()` |

### Classifying an H3 Index Column

If a table has a column holding H3 cell IDs (e.g. you pre-binned rows to an H3 grid before ingesting), tag it `class: "h3_index"` — mirrors how `class: "geometry"` works, and is what powers the "H3_index" option in the portal's per-field Classification dropdown (confirmed by inspecting the dropdown directly: it offers None / Latitude / Longitude / Geometry / H3_index, and the raw value is the lowercase `.capitalize()`-inverse of the label, i.e. `"h3_index"`):

```python
pa.field("h3_cell", pa.string(), nullable=False,
         metadata={"class": "h3_index", "description": "H3 cell index at resolution 8."})
```

**Adding this to an already-ingested table doesn't require re-fetching your source data.** Despite this doc's older advice below about `alter()` triggering "a full re-ingestion," a schema-metadata-only change (same column names/types, just added/changed metadata) is a cheap server-side patch — confirmed live on a 7,628-row table: all rows survived untouched after altering just the `class` metadata on one column:

```python
import pyarrow as pa

old_schema = ds.table.schema()
new_fields = []
for f in old_schema:
    if f.name == "h3_cell":
        meta = dict(f.metadata or {})
        meta[b"class"] = b"h3_index"
        f = pa.field(f.name, f.type, nullable=f.nullable, metadata=meta)
    new_fields.append(f)

ds.table.alter(pa.schema(new_fields))  # metadata-only change — existing rows are preserved
```

Reserve delete-and-recreate for actual column type/name changes, not metadata tweaks.

### Example: Full Schema with Descriptions and Spatial Tags

```python
schema = pa.schema([
    pa.field("id", pa.string(), nullable=False,
             metadata={"description": "Unique UUID primary key."}),
    pa.field("water_location_id", pa.int64(), nullable=True,
             metadata={"description": "Vannlokasjon-ID fra Vannmiljø (kilde: Vannmiljø-databasen)."}),
    pa.field("station_name", pa.string(), nullable=True,
             metadata={"description": "Navn på målestasjonen."}),
    pa.field("latitude", pa.float64(), nullable=True,
             metadata={"description": "Breddegrad (WGS84, desimalgrader).",
                       "class": "latitude"}),
    pa.field("longitude", pa.float64(), nullable=True,
             metadata={"description": "Lengdegrad (WGS84, desimalgrader).",
                       "class": "longitude"}),
    pa.field("value", pa.float64(), nullable=True,
             metadata={"description": "Målt verdi (enhet: µg/l).", "aggr": "mean"}),
    pa.field("geometry", pa.string(), nullable=True,
             metadata={
                 "isGeometry": "1",
                 "index": "1",
                 "class": "geometry",
                 "description": "Punkt-geometri i WKT-format (POINT (lengdegrad breddegrad)).",
             }),
])
```

All metadata values must be strings. The metadata dict is passed to `pa.field(..., metadata={...})` and propagated to ODP when `ds.table.create(schema)` is called.

### Writing Good Column Descriptions

| Column type | What to include | Example |
|---|---|---|
| Numeric | Value, unit, what NULL means | `"Sea surface temperature (°C). NULL where sensor failed."` |
| Categorical | Possible values | `"Observation type. One of: ROV, AUV, HOV, Lander, Camera Tow."` |
| Identifier | Source system and what it identifies | `"Station ID from the Vannmiljø database."` |
| Timestamp / year | Format and timezone if relevant | `"Year the dive was conducted."` |
| Geometry / coords | CRS, format, what NULL means | `"Point location in WKT (POINT (lon lat), WGS84). NULL where coordinates are withheld."` |
| Primary key | Just say it's the primary key | `"Generated UUID primary key."` |

## Working with Spatial Data (GeoJSON → ODP)

### Geometry Conversion

ODP tabular storage expects geometry in **WKT (Well-Known Text)** format. Convert from GeoJSON using Shapely:

```python
from shapely.geometry import shape
from shapely import wkt

def geojson_geometry_to_wkt(geometry: dict) -> str | None:
    if not geometry:
        return None
    geom = shape(geometry)
    return wkt.dumps(geom)
```

### Geometry Column Metadata

Mark the geometry column with special PyArrow field metadata so ODP recognises it as spatial.
**All three keys are required** — `isGeometry`, `index`, and `class`:

```python
pa.field(
    "geometry",
    pa.string(),
    nullable=True,
    metadata={
        "isGeometry": "1",
        "index": "1",
        "class": "geometry",
        "description": "Point location in WKT format (POINT (lon lat), WGS84).",
    }
)
```

### Schema Inference from GeoJSON Features

When ingesting GeoJSON with varying properties, infer the schema dynamically:

```python
def infer_pyarrow_type(value):
    if value is None:
        return pa.string()
    elif isinstance(value, bool):
        return pa.bool_()
    elif isinstance(value, int):
        return pa.int64()
    elif isinstance(value, float):
        return pa.float64()
    elif isinstance(value, (list, dict)):
        return pa.string()  # Serialize complex types as JSON strings
    else:
        return pa.string()
```

Scan all features, collect types per property, and fall back to `pa.string()` when mixed types are detected.

### Field Name Sanitization

ODP has restrictions on column names. Sanitize before creating the schema:

| Pattern | Replacement | Reason |
|---------|-------------|--------|
| `id` | `source_id` | Avoids collision with the primary key column |
| `_prefix` | `meta_prefix` | Underscore-prefixed names not allowed |
| `.` and `-` | `_` | Special characters not allowed in column names |

### Recommended Standard Columns

Add provenance columns to every row for traceability:

```python
row = {
    "id": str(uuid.uuid4()),        # Primary key
    "source_dataset": "dataset_key", # Which dataset this came from
    "source_name": "Display Name",   # Human-readable source
    "geometry": wkt_string,          # WKT geometry
    # ... remaining properties from the source data
}
```

## Working with Non-Spatial Data

For structured data without geometry (e.g., action plans, reports, measurements):

1. Define a fixed PyArrow schema manually (no inference needed)
2. Omit the geometry column
3. Same upload pattern: `ds.table.create(schema)` then `ds.insert(rows)` in a transaction

## Idempotent Ingestion Pattern

A robust ingest script should support:

```
--file <key>     # Ingest a single dataset
--all            # Ingest all datasets
--clean          # Delete existing and re-ingest
--list           # List available datasets
```

The clean/re-ingest pattern:
1. Look up existing dataset by name via `get_dataset_meta_by_name()`
2. If found and `--clean`: delete it, then create fresh
3. If found and not `--clean`: skip (already ingested)
4. If not found: create new

## Dataset Metadata Management

The SDK does not have built-in methods for updating dataset metadata beyond name and description at creation time. Use the REST API directly via `client._request()` to update metadata after creation.

### Metadata PATCH Endpoints

Each metadata facet has its own endpoint at `/api/catalog/v2/datasets/{datasetId}/metadata/...`:

| Endpoint | Method | Payload |
|----------|--------|---------|
| `.../general` | PATCH | `{name: str, description: str, tags: str[]}` |
| `.../provider` | PATCH | `{provider_id: str}` — use an existing provider UUID |
| `.../license` | PATCH | `{license_enum: str}` — e.g. `"CC-BY-4.0"` |
| `.../citation` | PATCH | `{text: str, link: str}` |
| `.../additional-info` | PATCH | Free-form JSON object (any keys) |
| `.../constraints` | PATCH | `{constraints: [{text: str}]}` |
| `.../documentation` | PATCH | `{documentation: str[]}` |

### Example: Update All Metadata

```python
import requests

base = f"{client.base_url}/api/catalog/v2/datasets/{dataset_id}/metadata"

# General: name, description, tags
client._request(requests.Request(
    method="PATCH",
    url=f"{base}/general",
    json={
        "name": "My Dataset",
        "description": "A detailed description of the dataset.",
        "tags": ["ocean", "marine", "monitoring"],
    },
), retry=False).raise_for_status()

# Provider (use an existing provider UUID — find via GET /api/catalog/v2/providers or the portal)
client._request(requests.Request(
    method="PATCH",
    url=f"{base}/provider",
    json={"provider_id": "ec54f2cd-e56c-4ac0-8d29-82654090e658"},  # HUB Ocean
), retry=False).raise_for_status()

# License
client._request(requests.Request(
    method="PATCH",
    url=f"{base}/license",
    json={"license_enum": "CC-BY-4.0"},
), retry=False).raise_for_status()

# Citation
client._request(requests.Request(
    method="PATCH",
    url=f"{base}/citation",
    json={
        "text": "My Project (2026). Dataset Name. Ocean Data Platform.",
        "link": "https://github.com/org/repo",
    },
), retry=False).raise_for_status()

# Additional info (free-form — use for geographic coverage, update frequency, etc.)
client._request(requests.Request(
    method="PATCH",
    url=f"{base}/additional-info",
    json={
        "geographic_coverage": "Global — Atlantic, Pacific, Indian oceans",
        "temporal_coverage": "2025-01-01 to present",
        "update_frequency": "Daily (automated)",
    },
), retry=False).raise_for_status()
```

### Important Notes on Metadata

- **Provider:** The `custom_provider` field with `{name, description, website, kind}` exists but the `kind` enum validation is strict and undocumented. Using an existing `provider_id` is more reliable.
- **Geographic coverage:** There is no dedicated spatial extent endpoint for datasets. The spatial bounds shown in the portal are auto-computed from the geometry column in the tabular data. Use `additional-info` for descriptive geographic coverage text.
- **Full PUT:** `PUT /api/catalog/v2/datasets/{datasetId}` replaces all metadata at once but requires every field — prefer the granular PATCH endpoints.

## Precomputing H3-Aggregated Datasets

If you're storing raw point-level data (e.g. individual sensor pings, vessel positions) and want a fast, map-friendly H3-grid summary of it, **precompute the aggregation once and ingest the small result as its own dataset** rather than aggregating live on every read. Two independent reasons for this, both confirmed in practice:

1. **Live H3 aggregation over large tables is slow.** A `ds.table.aggregate(group_by="h3(geometry, N)", ...)` over ~1.5M rows took 30+ seconds — fine for a one-off script, unusable for an interactive app hitting it per request.
2. **Correctly consuming an aggregate response outside the SDK is non-trivial** — see the `odp-data-consume` skill's aggregate-pagination gotcha. Doing the aggregation once in Python via the SDK (which already gets this right) sidesteps that entirely.

Pattern (see `odp-data-consume` for the `aggregate()` call details):

```python
import h3
import pyarrow as pa
from shapely.geometry import Polygon

# 1. Aggregate the raw table server-side (SDK handles pagination/merging correctly)
agg_df = source_ds.table.aggregate(group_by="h3(geometry, 8)", aggr={"value": "sum"})
# agg_df is indexed by H3 cell id, e.g. "88099e4f6bfffff" -> {"*": 12, "value": 293.1}

# 2. The aggregate response only gives back the cell id string, not geometry —
#    reconstruct the hex boundary yourself
def cell_to_wkt_polygon(cell: str) -> str:
    boundary = h3.cell_to_boundary(cell)          # [(lat, lng), ...] — note lat/lng order
    ring = [(lng, lat) for lat, lng in boundary]   # WKT/GeoJSON want (lng, lat)
    return Polygon(ring).wkt

rows = [
    {"id": new_row_id(), "h3_cell": cell, "value": row["value"], "geometry": cell_to_wkt_polygon(cell)}
    for cell, row in agg_df.iterrows()
]

# 3. Ingest as a small, fast-to-query derived dataset — tag both spatial columns:
schema = pa.schema([
    pa.field("h3_cell", pa.string(), nullable=False, metadata={"class": "h3_index", "description": "..."}),
    pa.field("value", pa.float64(), nullable=False, metadata={"description": "...", "aggr": "sum"}),
    pa.field("geometry", pa.string(), nullable=False,
             metadata={"isGeometry": "1", "index": "1", "class": "geometry", "description": "H3 cell boundary, WKT POLYGON."}),
])
```

Re-run this precompute step whenever the source table refreshes; consumers then just do a normal bbox `select`, no aggregation needed at read time.

## Reading Data Back from ODP

### Download a raw file
```python
ds = client.dataset(dataset_id)
content = ds.files.download(file_id)
```

### Query tabular data
Use the STAC API for spatial/temporal queries — see the `odp-stac-api` skill.

## Tips and Gotchas

- **Table schema is immutable for column types** — once created, you cannot change a column's data type. `ds.table.alter(new_schema)` handles this by re-ingesting existing data into the new schema server-side (you don't need to re-supply rows yourself — pass `from_names` to rename/duplicate fields). But **metadata-only changes** (e.g. adding a `class` tag without touching types) are a cheap in-place patch, not a re-ingestion — confirmed by altering just one column's metadata on a 7,628-row table and seeing every row survive untouched. Don't default to delete-and-recreate for metadata tweaks.
- **Transactions are required** — always use `with ds as tx: tx.insert(rows)` for tabular inserts.
- **Large datasets** — for datasets with many features (>10k rows), consider batching inserts.
- **Mixed types** — if a GeoJSON property has mixed types across features (e.g., sometimes `int`, sometimes `string`), fall back to `pa.string()` for that column.
- **Complex values** — lists and dicts in GeoJSON properties should be serialized to JSON strings before insertion.
- **UUID primary keys** — always generate UUIDs for the `id` column; do not reuse source IDs as the primary key.
