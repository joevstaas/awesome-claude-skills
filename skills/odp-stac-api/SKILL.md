---
description: Search for ocean data, marine datasets, and geospatial collections from Hub Ocean / Ocean Data Platform (ODP) using the STAC API
---

# ODP STAC API Skill

Use this skill when the user wants to search for ocean data, marine datasets, or geospatial collections from Hub Ocean / Ocean Data Platform (ODP).

## Overview

The ODP STAC API provides access to ocean and marine datasets following the SpatioTemporal Asset Catalog (STAC) specification. It allows searching for data collections and individual dataset items with spatial and temporal filters.

**Base URL:** `https://api.hubocean.earth/api/stac`

## Key Concepts: Collections vs Items

| Concept | Description | Endpoint |
|---------|-------------|----------|
| **Collection** | A dataset category containing multiple items. Has metadata like title, description, spatial/temporal extent, and keywords. Think of it as a "folder" of related data. | `GET /stac/collections` |
| **Item** | An individual geospatial record within a collection. Contains specific geometry, timestamps, and links to actual data assets. Think of it as a single "file" or observation. | `POST /stac/search` |

### Quick Reference
- **Want to browse available datasets?** → List Collections
- **Want to find specific data points/observations?** → Search Items within a Collection

## Endpoints

### 1. Root Catalog
```bash
curl -X GET "https://api.hubocean.earth/api/stac"
```

### 2. List All Collections
```bash
curl -X GET "https://api.hubocean.earth/api/stac/collections"
```

With pagination:
```bash
curl -X GET "https://api.hubocean.earth/api/stac/collections?offset=0&limit=10"
```

### 3. Get Single Collection
```bash
curl -X GET "https://api.hubocean.earth/api/stac/collections/{collection-id}"
```

### 4. Search Items
```bash
curl -X POST "https://api.hubocean.earth/api/stac/search" \
  -H "Content-Type: application/json" \
  -d '{
    "collections": ["collection-uuid"],
    "bbox": [minLon, minLat, maxLon, maxLat],
    "datetime": "start/end",
    "limit": 10
  }'
```

## Search Parameters

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `collections` | array | Collection UUIDs to search within | `["7c61c869-a7c1-4f1c-900e-34636ee3392a"]` |
| `ids` | array | Specific item UUIDs to retrieve | `["item-uuid-1", "item-uuid-2"]` |
| `bbox` | array | Bounding box [minLon, minLat, maxLon, maxLat] | `[5, 58, 11, 62]` (Oslo Fjord area) |
| `intersects` | object | GeoJSON geometry for spatial queries | `{"type": "Point", "coordinates": [10.7, 59.9]}` |
| `datetime` | string | ISO 8601 time range | `"2023-01-01T00:00:00Z/2024-01-01T00:00:00Z"` |
| `limit` | integer | Max results (default varies) | `100` |
| `offset` | integer | Skip N results for pagination | `20` |

## Example Queries

### List All Available Collections
```bash
curl -s "https://api.hubocean.earth/api/stac/collections" | jq '.collections[] | {id, title, description}'
```

### Search for Items in Norwegian Waters (2023)
```bash
curl -X POST "https://api.hubocean.earth/api/stac/search" \
  -H "Content-Type: application/json" \
  -d '{
    "bbox": [3, 57, 31, 71],
    "datetime": "2023-01-01T00:00:00Z/2023-12-31T23:59:59Z",
    "limit": 50
  }'
```

### Search Echosounder Data in Southern Ocean
```bash
curl -X POST "https://api.hubocean.earth/api/stac/search" \
  -H "Content-Type: application/json" \
  -d '{
    "collections": ["7c61c869-a7c1-4f1c-900e-34636ee3392a"],
    "bbox": [-180, -90, 180, -60],
    "limit": 100
  }'
```

### Get Items Near Oslo Fjord
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

## Known Collections

| Collection ID | Name | Description |
|--------------|------|-------------|
| `7c61c869-a7c1-4f1c-900e-34636ee3392a` | Aker BioMarine EK60/EK80 Echosounder | Echosounder data from Southern Ocean krill fishing missions (10+ years) |
| *(check API for current list)* | Aker BP Metocean Data | Wind, waves, currents from Norwegian Continental Shelf (2021+) |

> **Note:** Collections may be added or removed. Always query `/stac/collections` for the current list.

## Response Structure

### Collection Response
```json
{
  "id": "uuid",
  "title": "Dataset Name",
  "description": "What this dataset contains",
  "license": "ODC-BY-1.0",
  "extent": {
    "spatial": {"bbox": [[-180, -90, 180, 90]]},
    "temporal": {"interval": [["2021-01-01T00:00:00Z", null]]}
  },
  "keywords": ["oceanography", "marine", ...],
  "links": [...]
}
```

### Item Response (from search)
```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "id": "item-uuid",
      "geometry": {"type": "Point", "coordinates": [10.7, 59.9]},
      "properties": {
        "datetime": "2023-06-15T12:00:00Z",
        ...
      },
      "links": [...],
      "assets": {...}
    }
  ]
}
```

## Common Bounding Boxes

| Area | Bbox |
|------|------|
| Oslo Fjord | `[10.2, 59.0, 10.9, 59.9]` |
| Norwegian Coast | `[3, 57, 31, 71]` |
| North Sea | `[-4, 51, 9, 62]` |
| Southern Ocean | `[-180, -90, 180, -60]` |
| Global | `[-180, -90, 180, 90]` |

## Tips for Claude Desktop Usage

1. **First, list collections** to see what data is available
2. **Note collection IDs** - you need these UUIDs to search for items
3. **Use bbox for regional searches** - faster than global queries
4. **Add datetime filters** to narrow results to relevant time periods
5. **Check the `links` array** in responses for data download URLs

## Related Resources

- **STAC Specification:** https://stacspec.org/
- **Hub Ocean Data Catalog UI:** https://app.hubocean.earth/catalog
- **API Documentation:** https://docs.hubocean.earth/stac-api/
