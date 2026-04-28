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

`q` is a Solr query — clauses joined with ` AND `. The skill's job is to map natural language to those clauses, then call `GET /metadata-api/search?q=...&offset=...&beginDate=...&endDate=...&dateSearchMode=...`.

There are four clause types: free text, provider, scientific keyword, and geographic. They AND together. If the user's input doesn't clearly match a provider or keyword facet, fall back to free text — don't guess.

### Free text

Split the user's free-text words, preserving anything inside double quotes as a single phrase. For each word: lowercase it, escape Lucene specials (`+ - && || ! ( ) { } [ ] ^ " ~ * ? : \ /`), wrap as `*word*`. Join with ` OR `, parenthesise if more than one term. Quoted phrases keep their quotes and skip the `*…*` wrap.

| User says | Free-text clause |
|---|---|
| `cod` | `*cod*` |
| `microplastic mareano` | `(*microplastic* OR *mareano*)` |
| `"one ocean expedition"` | `"one ocean expedition"` |
| `Argo` | `*argo*` |

Free text matches against `Entry_Title` and `Data_Summary` (and a few other fields the Solr schema indexes). Programme names like Argo, Mareano, MOSAiC, and "One Ocean Expedition" generally appear in titles, so free text catches them reliably.

### Provider

NMDC has ~45 providers. Get the live list from `GET /metadata-api/getFacets` and match the user's institution name case-insensitively, allowing substring matches. If multiple providers match, pick the most specific one and mention any others in the answer.

Common short forms (canonical → user shorthand):

| Canonical | Common shorthand |
|---|---|
| Institute of Marine Research | IMR, Havforskningsinstituttet, HI |
| Norwegian Meteorological Institute | MET Norway, Met.no, MET, Meteorologisk institutt |
| Norwegian Polar Institute | NPI, Norsk Polarinstitutt |
| Norwegian Institute for Water Research | NIVA |
| Geological Survey of Norway | NGU, Norges geologiske undersøkelse |
| University of Bergen | UiB |
| University of Oslo | UiO |
| UiT The Arctic University of Norway | UiT |
| Akvaplan-niva AS | Akvaplan-niva, APN |
| Nansen Environmental and Remote Sensing Center | NERSC |
| University Centre in Svalbard | UNIS |

Emit:

```
Provider:"<canonical name>"
```

For multiple providers, OR them and parenthesise:

```
(Provider:"University of Bergen" OR Provider:"University of Oslo")
```

If the user names something that doesn't appear in the live facet list, do **not** invent a clause — show the top 3 fuzzy candidates (similarity ≥ 0.7) and ask which one they meant, or fall back to free text and say so.

### Scientific keyword

The `Scientific_Keyword` facet is the CEOS GCMD hierarchy, like:

```
EARTH SCIENCE
  > OCEANS
    > OCEAN TEMPERATURE
      > WATER TEMPERATURE
      > SEA SURFACE TEMPERATURE
    > SALINITY/DENSITY
    > OCEAN OPTICS
  > BIOSPHERE
    > AQUATIC ECOSYSTEMS
```

Walk the live tree from `getFacets`. Match the user's term (case-insensitive substring) against `value` at every level. Pick the **deepest** path whose leaf substring-matches. If multiple paths tie, prefer the one with the highest `matches` count.

Emit:

```
Scientific_Keyword:"<full path with > separators>"
```

Examples:

| User term | Likely path |
|---|---|
| `temperature` | `EARTH SCIENCE>OCEANS>OCEAN TEMPERATURE` |
| `salinity` | `EARTH SCIENCE>OCEANS>SALINITY/DENSITY>SALINITY` |
| `chlorophyll` | `EARTH SCIENCE>OCEANS>OCEAN CHEMISTRY>CHLOROPHYLL` |

When the user's term doesn't match a single deepest path cleanly (e.g. `CTD` is an instrument, not a GCMD term), **don't force it** — fall back to free text. Many practical terms (CTD, Argo, microplastic, cod) live in titles and summaries, not in GCMD.

### Geographic — GeoJSON workflow

When a query mentions a place, never inject a hardcoded bbox into the query without showing it to the user first. The flow:

1. Propose a bbox from your own knowledge of the place (e.g. inner Oslofjord ≈ `[10.4, 59.65, 10.95, 59.92]` in `[minLon, minLat, maxLon, maxLat]` order).
2. Write `nmdc-aoi-<slug>.geojson` to the user's cwd. Slug is a kebab-case version of the place. The file is a `FeatureCollection` with one `Polygon` feature plus a `description` property.
3. Tell the user: "Open this in [geojson.io](https://geojson.io), kepler.gl, or Mapbox to validate the area. Edit or replace the file if you want a different shape, then say 'go'."
4. On confirmation, **re-read the file** (so any user edits are picked up). Take the first `Polygon` feature; if the user provided a `MultiPolygon` use the first polygon; if they provided a non-polygon (point, line) take its bbox and convert.
5. Format as a `location_rpt` clause:

   ```
   location_rpt:"<op>(POLYGON((lon lat, lon lat, ..., lon lat)))"
   ```

   where the ring is closed (first vertex == last vertex) and coordinates are in **lon-lat order** (GeoJSON's native convention; matches Solr SpatialRPT). The valid operators are `Intersects` (default for "covering X" / "in X") and `IsWithin` (for "within X" / "fully inside X"). State which operator you picked in the answer.

Bbox-as-polygon template (for step 5 when starting from a rectangle):

```
location_rpt:"Intersects(POLYGON((minLon minLat, maxLon minLat, maxLon maxLat, minLon maxLat, minLon minLat)))"
```

Within-conversation cache: remember slug → file path so re-asking with the same place reuses the file (no re-prompting).

If you don't know a bbox for the named place, **don't guess wildly**. Ask the user to draw the area in geojson.io and save the file, then proceed from step 4.

Example GeoJSON file content for "inner Oslofjord":

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": {
        "description": "Inner Oslofjord, north of Drøbak narrows. Edit or replace this polygon to refine the AOI."
      },
      "geometry": {
        "type": "Polygon",
        "coordinates": [[
          [10.40, 59.65],
          [10.95, 59.65],
          [10.95, 59.92],
          [10.40, 59.92],
          [10.40, 59.65]
        ]]
      }
    }
  ]
}
```

### Temporal

Resolve natural-language periods to ISO 8601 (`YYYY-MM-DDTHH:MM:SSZ`) and pass them as `beginDate` / `endDate` query parameters. Use the conversation's current date for relative phrases.

Month names — accept English **and Norwegian**:

| Month | English | Norwegian |
|---|---|---|
| 1 | January, Jan | januar |
| 2 | February, Feb | februar |
| 3 | March, Mar | mars |
| 4 | April, Apr | april |
| 5 | May | mai |
| 6 | June, Jun | juni |
| 7 | July, Jul | juli |
| 8 | August, Aug | august |
| 9 | September, Sep | september |
| 10 | October, Oct | oktober |
| 11 | November, Nov | november |
| 12 | December, Dec | desember |

Forms to handle:

| User input | beginDate | endDate |
|---|---|---|
| `2026` | `2026-01-01T00:00:00Z` | `2026-12-31T23:59:59Z` |
| `Jan 2025` / `januar 2025` | `2025-01-01T00:00:00Z` | `2025-01-31T23:59:59Z` |
| `Mar` (year inferred from today) | `<currentYear>-03-01T00:00:00Z` | `<currentYear>-03-31T23:59:59Z` |
| `Jan–Mar 2025` | `2025-01-01T00:00:00Z` | `2025-03-31T23:59:59Z` |
| `2018–2020` | `2018-01-01T00:00:00Z` | `2020-12-31T23:59:59Z` |
| `2025–26` | `2025-01-01T00:00:00Z` | `2026-12-31T23:59:59Z` (mention: interpreted as full calendar span — narrow if you meant a season) |
| `last month` | first day of previous month | last day of previous month |
| `last year` | `<currentYear-1>-01-01T00:00:00Z` | `<currentYear-1>-12-31T23:59:59Z` |

`dateSearchMode`:

| User intent | Mode | Meaning |
|---|---|---|
| "data **collected/captured during** P" | `dateSearchMode=isWithin` | Dataset's coverage range is contained inside P. |
| "data **covering/spanning/about** P" | omit (default = intersects) | Dataset's coverage range overlaps P. |

State which mode you picked in the answer.

If month-name language is mixed and ambiguous (e.g. just "mai" by itself with no other Norwegian context), ask the user.

### Combining and pagination

All four clause types AND together to build `q`:

```
(*free* OR *text*) AND Provider:"…" AND Scientific_Keyword:"…>…" AND location_rpt:"Intersects(POLYGON((…)))"
```

Order doesn't matter to Solr but keep this order for readability. Temporal goes via the `beginDate` / `endDate` parameters, not in `q`.

Pagination: page size is 10 (hardcoded). Pass `offset=0`, then `offset=10`, `offset=20`, … Stop when:

- `offset >= numFound`, or
- you've collected the user's requested cap (default 50 results — bump on request).

Show the user the total `numFound` and how many you fetched, so they know whether to keep paging.

## Detail view

### Default — catalog only

### On-demand probe

## ODP feasibility advisory

## Examples coverage

## Failure modes
