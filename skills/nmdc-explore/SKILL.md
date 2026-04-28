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
| `Data_URL_Type` | Stringified array of access type tags (`GET DATA`, `PARENT`, `PART`, `VIEW PROJECT HOME PAGE`, `Access to OPeNDAP service`, …) |
| `Data_URL_Subtype` | Stringified array of subtype tags (`OPENDAP DATA (DODS)`, …) |

**Hierarchical records.** `Data_URL` and `Data_URL_Type` are parallel arrays. NMDC catalog records form a tree (e.g. **Expedition Overview → Leg → Instrument**), encoded with three distinguished tags:

- `PARENT` — single entry pointing UP to the immediate parent's landing page. Always URL-decode and surface it.
- `PART` — multiple entries pointing DOWN to children's landing pages. Top-level Overview records expose all descendants this way.
- `VIEW PROJECT HOME PAGE` — points to an external project website (not an NMDC record). Surface as a project link.

Two consequences to plan for:

1. **Chains can be 2+ levels deep** and the `PARENT` pointer may skip a level (e.g. an instrument dataset whose `PARENT` points straight at the Expedition Overview, not the Leg). When summarising results, don't trust that the immediate parent is the umbrella — look for a record in the result set with `PART` entries, or with a title prefix like `Overview - …`.
2. **DOI and license commonly live on the topmost Overview record, not on sub-datasets.** Before reporting "no DOI" or "license unknown", walk the `PARENT` chain (or look for a sibling `PART`-bearing record in the results) and re-check there.

## Capabilities

| # | Capability | Trigger phrases | What it does |
|---|---|---|---|
| 1 | **Search** | "find …", "datasets about …", "data covering …" | Map natural language to a Solr `q`, call `GET /metadata-api/search`, render a ranked list with title, provider, dates, and area. If the result set contains an Overview / umbrella record (one whose `Data_URL_Type` includes `PART` entries, or whose title starts with `Overview - …`), promote it to the top — that's where the DOI and license live. Otherwise, when results share a single immediate parent, surface the parent URL once at the top; for mixed parents, include each parent inline. Do not let title-pattern aggregation (e.g. parsing `LegN - Instrument` out of titles) silently drop records that don't match the pattern. |
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

When the user drills into a single dataset, render in this order:

1. **Title** — `Entry_Title`
2. **Identifier** — `Entry_ID`
3. **DOI** — `Dataset_DOI` (linked as `https://doi.org/<doi>`). If empty on this record, walk up the `PARENT` chain and report the first DOI you find, labelled with which level it came from (e.g. *"DOI (inherited from Overview)"*).
4. **Parent collection** — if any `Data_URL_Type` entry equals `PARENT`, render the URL-decoded `Data_URL` at the same index as a labelled link (e.g. *"Parent: One Ocean Expedition 2025-2026 — http://metadata.nmdc.no/metadata-api/landingpage/&lt;hash&gt;"*). Always include this when present. Optionally fetch the parent landing page once to read its `<title>` for the label. If the parent itself has its own `PARENT`, walk one more level so the user can see the full chain.
5. **Provider(s)** — parsed from the stringified array
6. **Personnel** — first author from `Dataset_Creator`, plus `PersonFirstName` / `PersonLastName` if useful
7. **Summary** — `Data_Summary`, first 400 characters with ellipsis if longer
8. **Temporal coverage** — `Start_Date` → `Stop_Date`, normalised to `YYYY-MM-DD`
9. **Spatial coverage** — bbox of the `location_rpt` polygon (or the point coordinates). Also write `nmdc-detail-<entryid>.geojson` to cwd so the user can open it visually.
10. **License** — parsed from the landing page (`GET /metadata-api/landingpage/<hash>`). Look for "Use Constraints" or the `Usage:` block (rendered with a Creative Commons badge and a license name like *"Creative Commons Attribution 4.0 International License"*). If this dataset's page has no license, fetch the `PARENT`'s landing page (and its parent's, up the chain) — license is usually declared once at the Overview level and inherits to children. Only mark as "unknown" after the chain is exhausted; report which level the license came from when inherited.
11. **File list** — parsed from the same landing page. For each `Data_URL` (excluding `PARENT`, `PART`, and `VIEW PROJECT HOME PAGE` entries — those are navigation, not data files):
    - Filename (last path segment)
    - Type tag from `Data_URL_Type` / `Data_URL_Subtype` (`OPENDAP DATA (DODS)`, `GET DATA`, `Access to OPeNDAP service`, …)
    - Stated size if the metadata mentions one (e.g. "190 CTD profiles", "18.7 days of data", "~340 MB")
    - URL
12. **Landing page URL** — `landingpage` field

Skip fields that are empty rather than rendering "n/a" — *except* the parent collection link, which must always be shown when present.

### On-demand probe

Triggered when the user asks for **actual** sizes, link health, or freshness — phrases like "actual sizes", "check the URLs", "is this alive", "are these still up", "byte sizes".

For each `Data_URL`:

- For OPeNDAP endpoints: `GET <url>.dds` and `GET <url>.das` to confirm the dataset is reachable and to read variable metadata. Use these as a proxy for "alive".
- For plain HTTP: `HEAD <url>` to read `Content-Length`, `Last-Modified`, and the status code.

Render as a small table:

| File | Status | Bytes | Last modified |
|---|---|---|---|

For datasets with more than 20 files, do these requests with a **max-in-flight bound of 5** and a per-request timeout of **10 seconds**. Report partial results if the host is slow rather than blocking the conversation.

Don't run the on-demand probe automatically — only when the user asks. It costs N round-trips per dataset and the metadata catalog answers most questions on its own.

## ODP feasibility advisory

When the user asks whether an NMDC dataset can go into ODP, classify on three axes:

| Axis | Values | Verdict drivers |
|---|---|---|
| **Format** | NetCDF, HDF, CSV/TSV, GeoTIFF, JSON, ZIP, other | NetCDF/HDF → **file ingest** by default; **tabular** is also possible if the user wants to flatten with `xarray` (typical CTD profile files flatten to one row per `(cast, depth)`). CSV/TSV → **tabular**. GeoTIFF/JSON/ZIP/other → **file**. |
| **Access** | OPeNDAP, HTTP, FTP, auth-required, none | OPeNDAP / HTTP → straightforward. FTP / auth-required → **manual** (out of scope for automated ingest). No `Data_URL` → **skip** (catalog-only entry). |
| **License** | CC-BY, CC0, restricted, unknown | CC-* → fine. Restricted / unknown → **flag**, ask the user to confirm before ingest. |

Output for each dataset, two parts:

**1. One-paragraph verdict.** Plain language, key numbers, and the classification. Example:

> *File ingest, straightforward. 3 NetCDF files (~340 MB total stated) on OPeNDAP, CC-BY 4.0. Tabular alternative possible: CTD profiles flatten naturally to one row per (cast, depth).*

**2. Recipe sketch (5–10 lines, no executable code).** Tell the user which `odp-data-ingest` pattern to follow:

- For **file ingest**: which SDK call (file upload), what to put in the dataset metadata (title, license, spatial / temporal bounds — all available from the catalog), and the source URL.
- For **tabular ingest**: the columns the user will likely want in the PyArrow schema (typical: timestamp, lat/lon → WKT geometry, instrument id, measurement variables), plus a one-line note about the conversion step (e.g. "use `xarray.open_dataset(opendap_url)` then `.to_dataframe()`").
- For **manual / skip**: explain the blocker and stop.

End with: *"To actually run the ingest, switch to the `odp-data-ingest` skill."*

Don't generate executable Python here — recipe sketches only. The actual code lives in `odp-data-ingest`, and that skill knows the current SDK API.

## Examples coverage

| User prompt | Clauses & parameters |
|---|---|
| Find datasets for the One Ocean Expedition 2025-26 | `q=(*one* OR *ocean* OR *expedition*)`, `beginDate=2025-01-01T00:00:00Z`, `endDate=2026-12-31T23:59:59Z` |
| Find CTD data | `q=*ctd*` (free text — CTD is an instrument, not a GCMD path) |
| Find data covering the inner Oslofjord | propose bbox → write `nmdc-aoi-inner-oslofjord.geojson` → confirm → `q=location_rpt:"Intersects(POLYGON((...)))"` |
| Find cod data in Skagerak | `q=*cod* AND location_rpt:"Intersects(POLYGON((...)))"` (Skagerrak GeoJSON workflow) |
| Find Argo data | `q=*argo*` (free text — programme name appears in titles) |
| Find data from the University of Bergen collected in 2026 | `q=Provider:"University of Bergen"`, `beginDate=2026-01-01T00:00:00Z`, `endDate=2026-12-31T23:59:59Z`, `dateSearchMode=isWithin` |
| Give me an overview of all the data | `GET /metadata-api/getFacets` → render full taxonomy with counts |
| List all the data providers | `GET /metadata-api/getFacets` → render Provider facet only |
| Find microplastic data from Mareano | `q=(*microplastic* OR *mareano*)` (both as free text — Mareano is a programme name that appears as a `MAREANO -` prefix in `Entry_Title`, not a separate Provider) |
| Find datasets captured last month | `beginDate=<first day of previous month>`, `endDate=<last day of previous month>`, `dateSearchMode=isWithin` (resolved against today) |

Sort-style prompts ("the latest dataset in the Oslofjord", "most recent NIVA data") are not supported. Ask the user for a concrete period instead — the API has no `sort` parameter.

## Failure modes

- **HTTP-only API.** `metadata.nmdc.no` is HTTP, not HTTPS. Note this once if the user asks; otherwise proceed without comment.
- **5xx / timeouts.** Retry once with a short backoff (1 s). On second failure, surface the error verbatim — don't pretend it worked.
- **0 results.** Don't dump the full Solr query at the user. Suggest one specific relaxation: drop the geo filter, widen the period, drop a facet, or fall back to free text. Pick the most-restrictive clause and offer to drop it.
- **Provider / keyword name doesn't match a facet.** Fuzzy-match against the live `getFacets` list (similarity ≥ 0.7). Show the top 3 candidates and ask which one the user meant. If none look right, fall back to free text and say so.
- **Place not in your geographic knowledge.** Don't guess wildly. Ask the user to draw the area in [geojson.io](https://geojson.io) and save the file, then proceed from step 4 of the geographic workflow.
- **Sort-by-recency requested.** "Latest" / "most recent" / "newest" without a period: explain that the API doesn't sort, then offer concrete period options ("last month", "2025", "2024–2026").
- **Norwegian month name ambiguous with English.** "Mai" is May; "Mars" is March. If the surrounding sentence is mixed-language, ask the user to clarify rather than guessing.
- **Landing page parse fails.** If a landing page returns 404 or doesn't contain the expected fields, render the catalog fields you do have and note "landing page details unavailable — see <URL>".
- **Probe times out on >5 files.** Don't block. Report the rows you got, mark the rest as "timeout", and let the user decide whether to retry.
