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

### Temporal

### Combining and pagination

## Detail view

### Default — catalog only

### On-demand probe

## ODP feasibility advisory

## Examples coverage

## Failure modes
