# nmdc-explore — design

**Status:** approved (2026-04-28)
**Target:** new skill at `skills/nmdc-explore/SKILL.md`

## Purpose

Help a user search and inspect datasets in the Norwegian Marine Data Centre (NMDC, `metadata.nmdc.no`) using natural-language prompts, drill into individual datasets for detail, and get a feasibility verdict for ingesting a dataset into the Ocean Data Platform (ODP).

The skill is a single-file `SKILL.md` guide for Claude — no auxiliary scripts, no cached data files. All API calls are made live by Claude per user prompt.

## Out of scope

- Performing the actual ODP ingest (lives in the `odp-data-ingest` skill)
- Downloading or subsetting NetCDF data (use OPeNDAP clients separately)
- The `/Subsetter/` basket workflow (browser-only feature)
- Sort-by-recency queries — the NMDC API does not support sort, so the skill asks for a concrete period instead of trying to fake it
- Any source other than `metadata.nmdc.no`

## Backend overview

`metadata.nmdc.no` exposes a small read-only API under `/metadata-api/`:

| Endpoint | Use |
|---|---|
| `GET /metadata-api/getFacets` | Full facet tree: `Scientific_Keyword` (CEOS GCMD hierarchy, `>`-delimited paths) and `Provider` (~45 institutions). Each node has a match count. |
| `GET /metadata-api/search?q=&offset=&beginDate=&endDate=&bbox=&dateSearchMode=` | Solr-backed catalog search. Returns `{matches, numFound, results[]}`. **Page size is hardcoded to 10.** |
| `GET /metadata-api/landingpage/{hash}` | HTML landing page for one dataset (rendered from CEOS DIF 9.7.1 metadata). Carries the file list, license, and stated sizes. |

Caveats:
- HTTP only (not HTTPS).
- No `sort` parameter on `search`.
- `bbox` query param is appended as a Solr filter query — but the official UI puts geo into `q` directly via `location_rpt`, which is what this skill does too.
- Result records expose: `Entry_ID`, `Entry_Title`, `Data_Summary`, `Dataset_DOI`, `Dataset_Creator`, `Provider`, `Scientific_Keyword`, `Start_Date`/`Stop_Date`, `location_rpt` (WKT), `landingpage`, `Data_URL`, `Data_URL_Type`, `Data_URL_Subtype`.

## Capabilities

| # | Capability | Trigger phrases | Behaviour |
|---|---|---|---|
| 1 | Search | "find …", "datasets about …", "data covering …" | NL → Solr query → `GET search` → ranked list (title, provider, dates, area) |
| 2 | Browse taxonomy | "overview of all data", "list providers", "what keywords exist" | `GET getFacets` → render the facet tree with counts |
| 3 | Dataset detail | "tell me about Entry_ID X", drill-down after a search | Catalog fields + landing page parse → file list, types, stated sizes, license, summary |
| 4 | ODP feasibility | "can this go into ODP?", "how would I ingest this?" | Verdict (file / tabular / manual / skip) + recipe sketch → hand off to `odp-data-ingest` |

## Query construction

`q` is a Solr query — clauses joined with ` AND `. The skill maps natural language to the four clause types below.

### 1. Free text

Split into words (preserving double-quoted phrases as a unit). For each word, lowercase and Lucene-escape special characters, then emit `*word*`. Join with ` OR `, wrap in parens if more than one term. Quoted phrases pass through with surrounding `*…*` removed.

Example: `One Ocean Expedition` → `(*one* OR *ocean* OR *expedition*)`.

### 2. Provider

Match the user's institution name (e.g. "University of Bergen", "IMR", "MET Norway") against the live `getFacets` Provider list, case-insensitive. Use a small alias map in the skill body for common short forms (IMR → Institute of Marine Research, MET → Norwegian Meteorological Institute, NPI → Norwegian Polar Institute, NIVA, NGU, UiB, UiO, UiT, UNIS, etc.).

Emit `Provider:"<canonical>"`. Multi-select joins with ` OR `.

### 3. Scientific_Keyword

Walk the GCMD hierarchy and pick the deepest matching path for the user's term. Emit `Scientific_Keyword:"<full path>"`, e.g. `"EARTH SCIENCE>OCEANS>OCEAN TEMPERATURE>WATER TEMPERATURE"`.

If the term doesn't clearly match a single deepest path, fall back to free text instead of guessing.

### 4. Geographic — GeoJSON workflow

When a query mentions a place:

1. Skill proposes a bbox from its own knowledge (e.g. inner Oslofjord ≈ `[10.4, 59.65, 10.95, 59.92]`).
2. Writes `nmdc-aoi-<slug>.geojson` to the cwd — a `FeatureCollection` with one rectangular `Polygon` feature plus `properties.description`.
3. Tells the user: open in [geojson.io](https://geojson.io) / kepler / Mapbox to validate. Edit or replace the file if needed, then say "go".
4. On confirmation, re-reads the file (so any user edits land), takes the first polygon, formats as `location_rpt:"<op>(POLYGON((lon lat, lon lat, ...)))"` with a closed ring in lon-lat order.
5. Operator selection:
   - "covering X" / "in X" → `Intersects` (datasets that touch the AOI)
   - "within X" / "inside X" → `IsWithin` (datasets fully contained)
   The skill states which it picked.
6. Caches the slug → file mapping in conversation; re-asking with the same place reuses the file.

If the skill cannot propose a bbox for a named place, it asks the user to draw one in geojson.io and save the file, then proceeds from step 4.

### Temporal

Parse to ISO 8601 (`YYYY-MM-DDTHH:MM:SSZ`):

- Concrete months in **English and Norwegian** (mars/March, januar/January, mai/May, …)
- Years (`2026` → `2026-01-01`..`2026-12-31`)
- Ranges (`2018–2020`, `Jan–Mar 2025`)
- Relative phrases against today (`last month`, `last year`)
- Cruise-style spans (`2025–26` → full calendar span, with a one-line caveat)

Emit `beginDate` / `endDate` and `dateSearchMode=isWithin` when the user says "captured / collected during …" (default mode "intersects record range" applies otherwise — i.e., dataset coverage overlaps the period).

### Combining and pagination

All clause types AND together. Page size is 10. Loop with `offset` until `numFound` is reached or a default cap of 50 results is hit. The cap can be raised on user request.

## Detail view

### Default — catalog only

Render in this order:

1. Title + Entry_ID + DOI (linked)
2. Provider(s) + key personnel (first author, principal investigator)
3. Summary (first 400 chars + ellipsis if longer)
4. Temporal coverage — `Start_Date` → `Stop_Date`
5. Spatial coverage — render the `location_rpt` polygon as a bbox, and write `nmdc-detail-<entryid>.geojson` so the user can open it visually
6. License (parsed from landing page; "unknown" if not found)
7. File list (parsed from landing page) — for each file: name, type tag (`OPENDAP DATA (DODS)`, `GET DATA`, …), stated size if metadata gives one, access URL
8. Landing page URL

### On-demand probe

Triggered by phrases like "actual sizes", "check the URLs", "is this alive". For each `Data_URL`:

- `HEAD` request, or OPeNDAP `.dds`/`.das` for OPeNDAP endpoints → real Content-Length, status code, last-modified
- Render as a small table: file, status (alive/404/timeout), bytes, last modified
- For >20 files: concurrent with a max-in-flight bound of 5 and a per-request timeout (default 10 s); report partial results if the host is slow

## ODP feasibility advisory

Three-axis classifier:

| Axis | Values | Verdict drivers |
|---|---|---|
| Format | NetCDF, HDF, CSV/TSV, GeoTIFF, JSON, ZIP, other | NetCDF/HDF → **file** by default, **tabular** candidate if user wants xarray-flatten. CSV → **tabular**. Other → **file**. |
| Access | OPeNDAP, HTTP, FTP, auth-required, none | OPeNDAP/HTTP → straightforward. FTP/auth → **manual**. None → **skip**. |
| License | CC-BY, CC0, restricted, unknown | CC-* → fine. Restricted/unknown → flag, ask user to confirm before ingest. |

Output per dataset:

- **One-paragraph verdict** — classification plus key numbers, e.g. *"File ingest, straightforward. 3 NetCDF files (~340 MB total stated) on OPeNDAP, CC-BY 4.0. Tabular alternative possible: CTD profiles flatten naturally to one row per (cast, depth)."*
- **Recipe sketch** (5–10 lines, no executable code) — which `odp-data-ingest` SDK call to use, which schema fields likely needed (geometry from `Lon`/`Lat` → WKT, time → timestamp, identifier → cast id), any conversion step. Ends with: *"switch to the `odp-data-ingest` skill to actually do this."*

## Examples coverage

| Prompt | Clauses |
|---|---|
| "Find datasets for the One Ocean Expedition 2025-26" | free text `(*one* OR *ocean* OR *expedition*)` + `beginDate=2025-01-01`, `endDate=2026-12-31` |
| "Find CTD data" | free text `*ctd*`; `Scientific_Keyword` only if a single deepest GCMD path matches cleanly |
| "Find data covering the inner Oslofjord" | propose Inner Oslofjord bbox → GeoJSON → confirm → `location_rpt:"Intersects(POLYGON(...))"` |
| "Find cod data in Skagerak" | free text `*cod*` + Skagerrak GeoJSON workflow |
| "Find Argo data" | free text `*argo*` |
| "Find data from the University of Bergen collected in 2026" | `Provider:"University of Bergen"` + `beginDate=2026-01-01`, `endDate=2026-12-31`, `dateSearchMode=isWithin` |
| "Give me an overview of all the data" | `getFacets` → render full taxonomy with counts |
| "List all the data providers" | `getFacets` → Provider facet only |
| "Find microplastic data from Mareano" | free text `*microplastic* AND *mareano*` |
| "Find datasets captured last month" | `beginDate=2026-03-01`, `endDate=2026-03-31`, `dateSearchMode=isWithin` (resolved against today's date) |

## Failure modes / red flags

- **HTTP-only API:** don't error; note "metadata.nmdc.no is HTTP-only — fine for read-only metadata calls".
- **5xx / timeouts:** retry once with backoff, then surface a plain error.
- **0 results:** suggest one specific relaxation ("drop the geo filter", "widen to 2024-2026") rather than dumping the full query.
- **Provider/keyword name doesn't match a facet:** fuzzy-match (similarity ≥ 0.7), show top-3 candidates, or fall back to free text and say so.
- **Place not in skill's geographic knowledge:** ask the user to draw a polygon in geojson.io and save the file.
- **"Latest" / sort-by-recency requests:** answer "the API doesn't sort; specify a period instead" and offer concrete options.
- **Norwegian month names ambiguous with English** (e.g. "mai" = May): trust the language of the surrounding sentence; if mixed, ask.

## Relationship to other skills in the repo

- `odp-data-exploration` — pre-ingest data quality on local files. Independent.
- `odp-data-ingest` — performs ODP ingest. The recipe sketch in capability 4 hands off here.
- `odp-data-consume` / `odp-stac-api` — query ODP after ingestion. Independent.
- `vannmiljo` — different Norwegian source, similar single-source-skill shape.

`nmdc-explore` slots in as a discovery layer that feeds `odp-data-ingest` when an NMDC dataset is a good fit for ODP.
