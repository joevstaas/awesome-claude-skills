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
| `bcf26dd9-c9e4-445b-a9d3-4f9d28d46224` | AkerBP Metocean data sharing | Wind, waves, currents from Norwegian Continental Shelf (2021+) |
| `443e136f-32c8-4f3c-aeb5-1272ba452a8d` | Global Fishing Watch - Ocean Monitoring | AIS apparent fishing effort + offshore infrastructure (CC-BY-NC-4.0) |
| `a8228453-d089-433b-90e1-92fe9c2aa7e1` | ProtectedSeas Navigator | MPA/MMA boundaries and regulations (CC-BY-4.0) |
| `82a402f5-f5c0-4ec2-af55-ff65d41e122a` | Seagrass webinar | Includes OBIS-derived seagrass occurrences |
| `99c5472e-f8ec-426d-842c-7cd9d688cd4e` | Tides of Transparency | OBIS-derived private-sector classification layers |

> **Note:** Collections may be added or removed. Always query `/stac/collections` for the current list (49 collections as of Oct 2026).

## Known Datasets (Items)

Verified via STAC in Oct 2026. Web link for any item: `https://app.hubocean.earth/catalog/dataset/<item-id>`.

| Item ID | Title | Licence | Coverage / notes |
|---------|-------|---------|------------------|
| `6b84a740-926a-46e6-bc5e-01c5eac3c9b8` | GFW AIS-based Apparent Fishing Effort - 2024 - derived summary | CC-BY-NC-4.0 | Global, 2024, 0.1° cells; presence + apparent fishing hours, gear and flag breakdowns. Derived from GFW v3.0 (Mar 2025) |
| `48a20311-3564-4045-b472-de775ecee31c` | GFW Apparent Fishing Effort - monthly - derived summary | CC-BY-NC-4.0 | Global, 2012–2024, 0.1°, monthly hours per cell |
| `e10d9a76-1949-482d-9292-30b6ed4de9bb` | GFW Apparent Fishing Effort - monthly - source data v3.0 | CC-BY-NC-4.0 | Global, 2012–2024, 0.1°, by flag × gear × month. Large: server-side filters only |
| `5a873571-1f58-4257-b636-4571b669faf8` | GFW Apparent Fishing Effort - yearly summary 2012-2024 | CC-BY-NC-4.0 | Global, 0.1°, one wide row per cell; good for fast map layers |
| `2b69f875-cc3d-4dad-b816-8a64d41c6932` | Global Offshore Infrastructure Detection (2017-2021) | CC-BY-NC-4.0 | SAR-detected fixed structures |
| `a608f54b-75c7-4df9-a3a8-cedbfa391873` | ProtectedSeas Navigator V2 — Focused: Area-Based Protections | CC-BY-4.0 | Excludes jurisdictional/fishery-management areas and Global/High Seas entries |
| `6fdd5db8-4866-4634-9011-d7591ae7ddc4` | ProtectedSeas Navigator V2 — Complete Global Database | CC-BY-4.0 | All categories incl. EEZs and fishery zones; snapshot 2025-10-29 |
| `479d9b42-cb21-4beb-8a57-8c217ad77409` | OBIS Seagrass Observations - Global | CC-BY-4.0 | Ten seagrass genera only, extracted from the full OBIS dataset on ODP |
| `3975fa3a-52f2-4571-9ae5-9b4d8a39a65d` / `561710c3-ef82-481c-9460-b3eeaa710b66` / `c40229c8-a0de-459e-851f-303e1f929910` | Tides of Transparency - OBIS private sector classification (H3 res 2 / 4 / 6) | CC-BY-4.0 | OBIS snapshot Aug 2024; who contributed records, not species observations |
| **unknown** | OBIS occurrence records (full) | unknown | **Not in STAC**, but exists on ODP (the seagrass item says it was extracted "from OBIS on ODP"). Get the ID from the catalog UI and fill it in here |

## Limitations and Gotchas (verified)

- **No free-text search.** `q` is silently ignored — the response is just unfiltered results, which can look like a match. `filter`/CQL2, `sortby` and `fields` aren't documented either. To search by keyword: dump `/collections?limit=500` plus each relevant collection's `/collections/{id}/items?limit=100`, then match titles, descriptions and keywords locally.
- **STAC is the public catalog, not the whole catalog.** A dataset missing from STAC (e.g. full OBIS) may still exist on ODP as draft or unlisted. Never report "not available" from STAC alone; fall back to the authenticated SDK by ID (see `odp-data-consume`) or ask the user to look it up in https://app.hubocean.earth/catalog.
- **Global-extent items match every bbox.** A Galápagos bbox search returns ~120 items, mostly global. Filter on each item's own `bbox` to find genuinely regional data.
- **Item `assets` are empty.** Data is reached through the SDK by item ID, not via asset hrefs. Use the `alternate` link (or the URL pattern above) for the web page.
- **Dates are local midnight in UTC.** E.g. `2023-12-31T23:00:00Z` means 2024-01-01 at UTC+1. An end of `2024-11-30T23:00:00Z` could mean "through November" or "December month-start". Check the actual data before claiming a month is covered.
- **Pagination:** use `offset`/`limit`, or follow the `next`/`prev` links in responses.
- **No auth needed** for the public STAC endpoints.

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
| Galápagos (marine reserve + margin, approx.) | `[-94, -3, -87, 3]` |
| Global | `[-180, -90, 180, 90]` |

## Tips for Claude Desktop Usage

1. **First, list collections** to see what data is available
2. **Note collection IDs** - you need these UUIDs to search for items
3. **Use bbox for regional searches** - faster than global queries
4. **Add datetime filters** to narrow results to relevant time periods
5. **Check the `links` array** for the `alternate` web-catalog link; data itself is accessed via the SDK by item ID

## Related Resources

- **STAC Specification:** https://stacspec.org/
- **Hub Ocean Data Catalog UI:** https://app.hubocean.earth/catalog
- **API Documentation:** https://docs.hubocean.earth/reference/stac-api/
