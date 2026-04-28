---
description: Search and inspect datasets in the Norwegian Marine Data Centre (NMDC, metadata.nmdc.no) — natural-language queries by keyword, provider, geography, and period; dataset detail views with file lists and on-demand link/size probing; and feasibility advice for ingesting NMDC data into the Ocean Data Platform (ODP). Trigger when the user mentions NMDC, Norwegian Marine Data Centre, or asks about Norwegian or Nordic marine data sources by topic, place, or time.
---

# NMDC Explore

## When to use this skill

Invoke when the user:

- Mentions NMDC or the Norwegian Marine Data Centre by name.
- Asks for marine, ocean, or fisheries data with a Norwegian or Nordic flavour ("data from IMR", "Barents Sea cruises", "Mareano survey", "Argo floats off Norway").
- Wants to discover datasets by topic, provider, geography, or time period.
- Drills into a single dataset for file lists, sizes, license, and access details.
- Asks whether an NMDC dataset can be ingested into ODP and how.

Skip when the user is asking about ODP itself (use `odp-data-consume`, `odp-data-ingest`, or `odp-stac-api`), about a different Norwegian source like Vannmiljø (use `vannmiljo`), or about local files (use `odp-data-exploration`).

## Out of scope

This skill does not:

- Perform the actual ODP ingest (hand off to `odp-data-ingest`).
- Download or subset NetCDF data from OPeNDAP (use `xarray` / `pydap` directly).
- Drive the `/Subsetter/` basket workflow on the NMDC site (browser-only feature).
- Sort search results by recency. The NMDC API has no `sort` parameter; ask the user for a concrete period instead.
- Search anything outside `metadata.nmdc.no`.

## Backend overview

`metadata.nmdc.no` exposes a small read-only API under `/metadata-api/`:

| Endpoint | Use |
|---|---|
| `GET /metadata-api/getFacets` | Full facet tree: `Scientific_Keyword` (CEOS GCMD hierarchy, `>`-delimited paths) and `Provider` (~45 institutions). Each node has a match count. |
| `GET /metadata-api/search?q=&offset=&beginDate=&endDate=&bbox=&dateSearchMode=` | Solr-backed catalog search. Returns `{matches, numFound, results[]}`. **Page size is hardcoded to 10** — only `offset` paginates. |
| `GET /metadata-api/landingpage/{hash}` | HTML landing page for one dataset, rendered from CEOS DIF 9.7.1 metadata. Carries the file list, license, and stated sizes. |

Caveats to keep in mind:

- HTTP only (not HTTPS). Don't error — note it and proceed.
- No `sort` parameter on `search`. If the user asks for "latest", request a concrete period.
- The `bbox` query param is appended as a Solr filter query, but in practice the live UI puts geographic constraints into `q` directly via `location_rpt` — this skill does the same.

Each result document carries these fields (subset relevant to the skill):

| Field | Meaning |
|---|---|
| `Entry_ID` | Stable identifier |
| `Entry_Title` | Human title |
| `Data_Summary` | Free-text description |
| `Dataset_DOI` | DOI string (without `https://doi.org/` prefix) |
| `Dataset_Creator` | Comma-separated author list |
| `Provider` | Stringified array of institution names, e.g. `"[University of Bergen]"` |
| `Scientific_Keyword` | Stringified array of GCMD paths, e.g. `"[EARTH SCIENCE>OCEANS>OCEAN TEMPERATURE]"` |
| `Start_Date` / `Stop_Date` | Date strings, often in long form like `"Mon Oct 31 01:00:00 CET 2022"` |
| `location_rpt` | WKT — either `"lon lat"` for a point or `"POLYGON((lon lat, ...))"` |
| `landingpage` | Full URL to the landing page |
| `Data_URL` | Stringified array of URL-encoded file URLs |
| `Data_URL_Type` | Stringified array of access type tags (`GET DATA`, `PARENT`, `Access to OPeNDAP service`, …) |
| `Data_URL_Subtype` | Stringified array of subtype tags (`OPENDAP DATA (DODS)`, …) |

## Capabilities

| # | Capability | Trigger phrases | What it does |
|---|---|---|---|
| 1 | **Search** | "find …", "datasets about …", "data covering …" | Map natural language to a Solr `q`, call `GET /metadata-api/search`, render a ranked list with title, provider, dates, and area. |
| 2 | **Browse taxonomy** | "overview of all data", "list providers", "what keywords exist" | `GET /metadata-api/getFacets` and render the facet tree with counts. |
| 3 | **Dataset detail** | "tell me about Entry_ID X", drill-down after a search | Combine catalog fields with the parsed landing page to show file list, types, stated sizes, license, and summary. |
| 4 | **ODP feasibility** | "can this go into ODP?", "how would I ingest this?" | Verdict (file / tabular / manual / skip) plus a recipe sketch that hands off to `odp-data-ingest`. |

## Query construction

### Free text

### Provider

### Scientific keyword

### Geographic — GeoJSON workflow

### Temporal

### Combining and pagination

## Detail view

### Default — catalog only

### On-demand probe

## ODP feasibility advisory

## Examples coverage

## Failure modes
