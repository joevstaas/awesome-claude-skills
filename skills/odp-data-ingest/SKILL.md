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

**Note:** `ds.files.update_meta()` has limited field support. Setting `"description"` will raise an error (`Cannot update field 'description'`). Only use `update_meta` for fields the API accepts (e.g., `"display_name"`).

### Step 3: Upload Tabular Data

Define a PyArrow schema and insert rows:

```python
import pyarrow as pa

schema = pa.schema([
    pa.field("id", pa.string(), nullable=False),
    pa.field("name", pa.string(), nullable=True,
             metadata={"description": "Station name"}),
    pa.field("value", pa.float64(), nullable=True,
             metadata={"description": "Measured value"}),
    pa.field("geometry", pa.string(), nullable=True,
             metadata={"isGeometry": "1", "index": "1",
                        "description": "Point geometry in WKT format"}),
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

PyArrow field metadata can set column descriptions, classifications, and aggregation hints that appear in the ODP portal.

### Supported Metadata Keys

| Key | Values | Effect in ODP |
|-----|--------|---------------|
| `isGeometry` | `"1"` | Marks column as the geometry column |
| `index` | `"1"` | Creates a spatial index on the column |
| `description` | Any string | Sets the column's Description in the portal |
| `class` | `"geometry"`, `"latitude"`, `"longitude"` | Sets the column's Classification dropdown |
| `aggr` | `"sum"`, `"mean"`, `"min"`, `"max"`, `"count"` | Aggregation hint for the column |

### Example: Schema with Column Descriptions

```python
schema = pa.schema([
    pa.field("id", pa.string(), nullable=False),
    pa.field("water_location_id", pa.int64(), nullable=True,
             metadata={"description": "Vannlokasjon-ID fra Vannmiljø"}),
    pa.field("station_name", pa.string(), nullable=True,
             metadata={"description": "Navn på målestasjonen"}),
    pa.field("value", pa.float64(), nullable=True,
             metadata={"description": "Målt verdi", "aggr": "mean"}),
    pa.field("geometry", pa.string(), nullable=True,
             metadata={
                 "isGeometry": "1",
                 "index": "1",
                 "description": "Punkt-geometri i WKT-format",
             }),
])
```

All metadata values must be strings. The metadata dict is passed to `pa.field(..., metadata={...})` and propagated to ODP when `ds.table.create(schema)` is called.

## Working with Spatial Data (GeoJSON to ODP)

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

Mark the geometry column with special PyArrow field metadata so ODP recognizes it as spatial:

```python
pa.field(
    "geometry",
    pa.string(),
    nullable=True,
    metadata={"isGeometry": "1", "index": "1"}
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
- **`DatasetMeta` is a dataclass** — `get_dataset_meta_by_name()` returns a `DatasetMeta` object with `.id`, `.name`, `.description` attributes. Do not use dict-style access (`["id"]`).
- **`files.update_meta()` limitations** — the `"description"` field is not supported and will raise an error. Only use supported fields like `"display_name"`.
