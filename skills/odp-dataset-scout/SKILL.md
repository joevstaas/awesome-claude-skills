---
description: Scout Ocean Data Platform (ODP) datasets for a study — find candidate datasets for a place, period, and set of themes (e.g. species observations, protected areas, fishing effort), verify their metadata, and return a compact comparison table with IDs, links, licences, coverage, fields, provenance, and limitations, plus a proposed common study area and period. Discovery and metadata only — never downloads in bulk, modifies, or ingests. Trigger when the user asks which ODP datasets exist for a region or topic, wants a data inventory or feasibility check before an analysis or map, or asks about provenance, lineage, or coverage of ODP datasets.
---

# ODP Dataset Scout

A repeatable workflow for "find me datasets for X in region Y and period Z" on the Ocean Data Platform. It combines the public STAC API (`odp-stac-api`) with authenticated SDK lookups (`odp-data-consume`) and ends in a verified, clearly-labelled inventory.

**Hard rules**
- Read-only. No bulk downloads, no writes, no ingest. Use server-side filters and aggregates for any data check.
- Every fact in the output is either *verified* (from STAC, catalog record, schema, or a server-side query) or explicitly labelled **unknown** / **assumption**.
- A dataset missing from STAC is **not** evidence that it's absent from ODP.

## Workflow

### 1. Frame the request
Pin down the themes, the region (as a bbox, and a named polygon if one exists, e.g. an MPA), and the candidate period. Name each source the user expects (e.g. OBIS, ProtectedSeas, Global Fishing Watch).

### 2. STAC discovery (public, no auth)
STAC has no free-text search (`q` is silently ignored), so:

```bash
curl -s "https://api.hubocean.earth/api/stac/collections?limit=500" -o cols.json
# match titles/descriptions/keywords locally, then for each hit:
curl -s "https://api.hubocean.earth/api/stac/collections/<collection-id>/items?limit=100"
```

- Check the "Known Datasets" table in `odp-stac-api` first. It may already have the IDs.
- Also grep collection and item JSON for the source name (e.g. `OBIS`). Derived products often mention their parent dataset ("extracted from OBIS on ODP"), which proves the parent exists even if it isn't listed.
- A bbox search returns every global-extent item. Judge regional relevance from each item's own `bbox`, not from the fact that it matched.

### 3. Catalog fallback for anything STAC doesn't list
There is no documented keyword search in the catalog API or SDK. Lookup is by ID (and `get_dataset_meta_by_name` only matches **exact** names). So:
1. Try `get_dataset_meta_by_name` with the most likely exact title.
2. Otherwise ask the user to find the dataset in https://app.hubocean.earth/catalog and paste its ID (the UUID in `/catalog/dataset/<id>`).
3. Don't guess at undocumented endpoints. `/api/catalog/v2/datasets` (list) returned 401 when tried.

Authentication follows `odp-data-consume`: `ODP_API_KEY` env var or `~/.config/odp/.env` only, never a search for keys. If no key is available, finish everything you can from STAC and list what's blocked.

### 4. Verify each candidate (authenticated, metadata only)
For each dataset ID:
- **Catalog record** `GET /api/catalog/v2/datasets/{id}`: `audit_fields.created_at`/`updated_at`, `tabular_metadata_version`, `publish_status`/`published_at`, provider, licence.
- **Schema** `ds.table.schema()`: column names, types, field descriptions. Read the units from the descriptions; don't infer them from column names.
- **Area/period check** via `ds.table.aggregate(filter=<geometry within AOI AND time range>, aggr={...})`: row count, min/max date. That confirms the dataset really covers the AOI and period. Use `aggregate`, never `select` of all rows.

### 5. Propose a common study area and period
- Prefer an authoritative polygon (e.g. the MPA boundary from ProtectedSeas) plus a context bbox for activity just outside it.
- Choose the period as the overlap that every dataset actually supports, confirmed in step 4. STAC dates are local midnight in UTC (`2024-11-30T23:00:00Z` ≈ 1 Dec UTC+1), so confirm edge months from the data.
- Point out mismatched time semantics, e.g. a boundary snapshot dated after the analysis year, or slow publication of recent biodiversity records.

## Output template

One compact table, one row per dataset:

| Dataset · ID · link | Provider / original source / licence | Geographic + temporal coverage | Fields, units, resolution | Access + limitations |
|---|---|---|---|---|

- Link format: `https://app.hubocean.earth/catalog/dataset/<id>`.
- Mark the licence prominently when it's restrictive (e.g. CC-BY-NC).
- A row for a dataset that exists but can't yet be verified is fine: put **unknown** in its cells.

Then short sections:
1. **Provenance & lineage.** Available: source version, method, snapshot dates, parent dataset, catalog timestamps. Missing: no processing code or checksums, STAC `assets` empty, unclear interval semantics, etc.
2. **Proposed study area & period**, with the reasoning, and labelled as an assumption where unverified.
3. **Interpretation guardrails** (below), only those that apply.
4. **Next steps / blockers**: what needs a key, an ID from the user, or a permission rule.

## Interpretation guardrails

| Data type | Is | Is not |
|---|---|---|
| OBIS / GBIF occurrences | A record of a taxon at a place and time, with uneven sampling effort | Animal tracks, abundance, absence, or distribution without effort correction |
| Telemetry / animal tracking | Movement paths of tagged individuals | Population-level distribution |
| GFW apparent fishing effort | A model's estimate of AIS behaviour per grid cell (hours) | Confirmed fishing, catch, or illegal activity. It misses vessels without AIS or with AIS off, and under-covers small-scale fleets |
| Fishing effort inside an MPA polygon | Spatial overlap | A violation, until the regulations, zoning, dates, and vessel permits are checked |
| ProtectedSeas protection ratings | A standardised regulatory rating | Enforcement or compliance on the water |

## Related skills
- `odp-stac-api`: STAC endpoints, known collection/item IDs, gotchas
- `odp-data-consume`: SDK auth, schema, filtered queries, aggregation, catalog record fields
- `odp-data-exploration`: quality check once a file is actually in hand
