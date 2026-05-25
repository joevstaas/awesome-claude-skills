---
description: Ingest data into Ocean Data Platform (ODP) / HUB Ocean using the ODP Python SDK — covers datasets, file uploads, tabular data, and spatial data
---

# ODP Data Ingest Skill

Use this skill when the user wants to ingest, upload, or manage datasets in the Ocean Data Platform (ODP) by Hub Ocean.

## Prerequisites

### Python Dependencies

```bash
pip install odp-sdk pyarrow shapely python-dotenv
```

| Package | Purpose |
|---------|---------|
| `odp-sdk` | ODP client library (authentication, catalog, dataset operations) |
| `pyarrow` | Define table schemas and serialize tabular data |
| `shapely` | Convert geometries between GeoJSON and WKT (for spatial data) |

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
| `class` | `"geometry"`, `"latitude"`, `"longitude"` | Spatial classification — **required on every spatial column** |
| `isGeometry` | `"1"` | Marks the WKT geometry column (use together with `class: geometry`) |
| `index` | `"1"` | Creates a spatial index (use together with `isGeometry`) |
| `aggr` | `"sum"`, `"mean"`, `"min"`, `"max"`, `"count"` | Aggregation hint for numeric columns |

### Spatial Column Quick Reference

| Column holds | `class` | Also set | Type |
|---|---|---|---|
| WKT geometry string (POINT / LINESTRING / …) | `"geometry"` | `"isGeometry": "1"`, `"index": "1"` | `pa.string()` |
| Latitude float | `"latitude"` | — | `pa.float64()` |
| Longitude float | `"longitude"` | — | `pa.float64()` |

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

## Reading Data Back from ODP

### Download a raw file
```python
ds = client.dataset(dataset_id)
content = ds.files.download(file_id)
```

### Query tabular data
Use the STAC API for spatial/temporal queries — see the `odp-stac-api` skill.

## Tips and Gotchas

- **Table schema is immutable** — once created, you cannot change column types. You can use `ds.table.alter(new_schema)` but this triggers a full data re-ingestion. For simple metadata changes, it's often easier to delete and recreate the dataset.
- **Transactions are required** — always use `with ds as tx: tx.insert(rows)` for tabular inserts.
- **Large datasets** — for datasets with many features (>10k rows), consider batching inserts.
- **Mixed types** — if a GeoJSON property has mixed types across features (e.g., sometimes `int`, sometimes `string`), fall back to `pa.string()` for that column.
- **Complex values** — lists and dicts in GeoJSON properties should be serialized to JSON strings before insertion.
- **UUID primary keys** — always generate UUIDs for the `id` column; do not reuse source IDs as the primary key.
