# nmdc-explore Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `skills/nmdc-explore/SKILL.md`, a single-file Claude skill that searches and inspects datasets in the Norwegian Marine Data Centre (`metadata.nmdc.no`) from natural-language prompts and provides ODP ingestion feasibility advice.

**Architecture:** Pure-prose skill (no code shipped). Claude reads the skill, makes live HTTP calls to `metadata.nmdc.no/metadata-api/`, parses JSON/HTML responses, and writes GeoJSON files to the user's cwd for visual validation of geographic queries. Hands off to `odp-data-ingest` when the user wants to actually ingest a dataset.

**Tech Stack:** Markdown + YAML frontmatter. Live API calls happen at conversation time using whatever HTTP client the harness exposes (typically `curl`/`WebFetch`). No code is shipped with the skill.

**Reference spec:** `docs/superpowers/specs/2026-04-28-nmdc-explore-design.md`

**Branch:** `add-nmdc-explore-skill` (already created, spec already committed).

---

## File structure

| File | Purpose |
|---|---|
| `skills/nmdc-explore/SKILL.md` (new) | The skill itself — frontmatter + content for all sections |
| `README.md` (modify) | Add a row to the "Available Skills" table |

No other files. The skill is self-contained.

## Validation strategy

A skill is "tested" by:
1. **Frontmatter validity** — YAML parses, required fields present.
2. **Live API smoke tests** — every endpoint and query pattern documented in the skill is reproduced via `curl` against the real API and verified to behave as documented.
3. **Example prompt walk-through** — for each of the 10 example prompts in the spec, mentally compose the query the skill would emit, run it, confirm a non-empty (or appropriately empty) result.

These checks run in Tasks 2 and 6.

---

## Task 1: Scaffold skill directory and verify live API

**Files:**
- Create: `skills/nmdc-explore/SKILL.md` (skeleton only — frontmatter + section headings)

**Goal of this task:** establish the file with valid frontmatter and confirm the three live endpoints behave as the spec describes (so subsequent tasks document what's real, not what we *think* is real).

- [ ] **Step 1.1: Create the skill file with frontmatter and section skeleton**

Write `skills/nmdc-explore/SKILL.md` with exactly this content:

```markdown
---
description: Search and inspect datasets in the Norwegian Marine Data Centre (NMDC, metadata.nmdc.no) — natural-language queries by keyword, provider, geography, and period; dataset detail views with file lists and on-demand link/size probing; and feasibility advice for ingesting NMDC data into the Ocean Data Platform (ODP). Trigger when the user mentions NMDC, Norwegian Marine Data Centre, or asks about Norwegian or Nordic marine data sources by topic, place, or time.
---

# NMDC Explore

## When to use this skill

## Out of scope

## Backend overview

## Capabilities

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
```

- [ ] **Step 1.2: Verify the frontmatter parses as YAML**

Run:

```bash
python3 -c "
import yaml, sys
text = open('skills/nmdc-explore/SKILL.md').read()
assert text.startswith('---\n'), 'must start with frontmatter delimiter'
end = text.index('\n---\n', 4)
fm = yaml.safe_load(text[4:end])
assert 'description' in fm, 'description field missing'
assert len(fm['description']) > 80, 'description should be substantial'
print('OK:', list(fm.keys()))
"
```

Expected output: `OK: ['description']`

- [ ] **Step 1.3: Smoke-test live API — getFacets**

Run:

```bash
curl -sf 'http://metadata.nmdc.no/metadata-api/getFacets' -o /tmp/nmdc-facets.json && \
python3 -c "
import json
d = json.load(open('/tmp/nmdc-facets.json'))
facets = d['facets']
names = [f['name'] for f in facets]
assert 'Scientific_Keyword' in names, 'missing Scientific_Keyword'
assert 'Provider' in names, 'missing Provider'
provider = next(f for f in facets if f['name'] == 'Provider')
print(f'OK: {len(facets)} top facets, {len(provider[\"children\"])} providers')
"
```

Expected output: `OK: 2 top facets, 45 providers` (count may drift slightly).

If this fails, **stop** and investigate — the skill cannot proceed if the API has changed shape.

- [ ] **Step 1.4: Smoke-test live API — search**

Run:

```bash
curl -sf 'http://metadata.nmdc.no/metadata-api/search?q=*temperature*&offset=0' -o /tmp/nmdc-search.json && \
python3 -c "
import json
d = json.load(open('/tmp/nmdc-search.json'))
assert 'results' in d, 'missing results'
assert 'numFound' in d, 'missing numFound'
required = {'Entry_ID', 'Entry_Title', 'Provider', 'Start_Date', 'Stop_Date', 'location_rpt', 'landingpage', 'Data_URL'}
missing = required - set(d['results'][0].keys())
assert not missing, f'result fields missing: {missing}'
print(f'OK: numFound={d[\"numFound\"]}, fields look right')
"
```

Expected: `OK: numFound=...` (a positive integer).

- [ ] **Step 1.5: Smoke-test live API — landing page**

Run:

```bash
curl -sf 'http://metadata.nmdc.no/metadata-api/landingpage/fde3cb4955662933c39ea781b7607f10' -o /tmp/nmdc-landing.html && \
python3 -c "
text = open('/tmp/nmdc-landing.html').read()
assert 'CEOS IDN DIF' in text or 'DIF' in text, 'expected DIF metadata reference'
assert 'OPeNDAP' in text or 'Data_URL' in text or 'opendap' in text.lower(), 'expected data url reference'
print('OK: landing page parsable, len=', len(text))
"
```

Expected: `OK: landing page parsable, len=...`

- [ ] **Step 1.6: Commit the scaffold**

```bash
git add skills/nmdc-explore/SKILL.md
git commit -m "$(cat <<'EOF'
Scaffold nmdc-explore skill

Empty SKILL.md with frontmatter description that triggers on NMDC
or Norwegian marine data prompts. Section skeleton mirrors the
design spec. Live API endpoints verified to still match expected
shape before authoring.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Fill "When to use", "Out of scope", "Backend overview", "Capabilities"

**Files:**
- Modify: `skills/nmdc-explore/SKILL.md`

- [ ] **Step 2.1: Replace empty `## When to use this skill` section with**

```markdown
## When to use this skill

Invoke when the user:

- Mentions NMDC or the Norwegian Marine Data Centre by name.
- Asks for marine, ocean, or fisheries data with a Norwegian or Nordic flavour ("data from IMR", "Barents Sea cruises", "Mareano survey", "Argo floats off Norway").
- Wants to discover datasets by topic, provider, geography, or time period.
- Drills into a single dataset for file lists, sizes, license, and access details.
- Asks whether an NMDC dataset can be ingested into ODP and how.

Skip when the user is asking about ODP itself (use `odp-data-consume`, `odp-data-ingest`, or `odp-stac-api`), about a different Norwegian source like Vannmiljø (use `vannmiljo`), or about local files (use `odp-data-exploration`).
```

- [ ] **Step 2.2: Replace empty `## Out of scope` section with**

```markdown
## Out of scope

This skill does not:

- Perform the actual ODP ingest (hand off to `odp-data-ingest`).
- Download or subset NetCDF data from OPeNDAP (use `xarray` / `pydap` directly).
- Drive the `/Subsetter/` basket workflow on the NMDC site (browser-only feature).
- Sort search results by recency. The NMDC API has no `sort` parameter; ask the user for a concrete period instead.
- Search anything outside `metadata.nmdc.no`.
```

- [ ] **Step 2.3: Replace empty `## Backend overview` section with**

```markdown
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
```

- [ ] **Step 2.4: Replace empty `## Capabilities` section with**

```markdown
## Capabilities

| # | Capability | Trigger phrases | What it does |
|---|---|---|---|
| 1 | **Search** | "find …", "datasets about …", "data covering …" | Map natural language to a Solr `q`, call `GET /metadata-api/search`, render a ranked list with title, provider, dates, and area. |
| 2 | **Browse taxonomy** | "overview of all data", "list providers", "what keywords exist" | `GET /metadata-api/getFacets` and render the facet tree with counts. |
| 3 | **Dataset detail** | "tell me about Entry_ID X", drill-down after a search | Combine catalog fields with the parsed landing page to show file list, types, stated sizes, license, and summary. |
| 4 | **ODP feasibility** | "can this go into ODP?", "how would I ingest this?" | Verdict (file / tabular / manual / skip) plus a recipe sketch that hands off to `odp-data-ingest`. |
```

- [ ] **Step 2.5: Verify "When to use" triggers cleanly**

Read the file and check that the trigger description in the frontmatter and the "When to use" section are consistent. Specifically: the frontmatter mentions "natural-language queries by keyword, provider, geography, and period" — the "When to use" bullets should mention the same axes. They do.

Run:

```bash
grep -E "topic|provider|geography|time period" skills/nmdc-explore/SKILL.md
```

Expected: at least 2 matches in "When to use".

- [ ] **Step 2.6: Commit**

```bash
git add skills/nmdc-explore/SKILL.md
git commit -m "$(cat <<'EOF'
Document NMDC backend and skill capabilities

Adds the When-to-use, Out-of-scope, Backend-overview, and
Capabilities sections. Backend table lists the three live
endpoints and the result fields the skill depends on.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Fill query construction — free text, provider, scientific keyword

**Files:**
- Modify: `skills/nmdc-explore/SKILL.md`

- [ ] **Step 3.1: Add introductory paragraph under `## Query construction` (above subsections)**

Insert directly under the `## Query construction` heading and before `### Free text`:

```markdown
`q` is a Solr query — clauses joined with ` AND `. The skill's job is to map natural language to those clauses, then call `GET /metadata-api/search?q=...&offset=...&beginDate=...&endDate=...&dateSearchMode=...`.

There are four clause types: free text, provider, scientific keyword, and geographic. They AND together. If the user's input doesn't clearly match a provider or keyword facet, fall back to free text — don't guess.
```

- [ ] **Step 3.2: Replace empty `### Free text` with**

```markdown
### Free text

Split the user's free-text words, preserving anything inside double quotes as a single phrase. For each word: lowercase it, escape Lucene specials (`+ - && || ! ( ) { } [ ] ^ " ~ * ? : \ /`), wrap as `*word*`. Join with ` OR `, parenthesise if more than one term. Quoted phrases keep their quotes and skip the `*…*` wrap.

| User says | Free-text clause |
|---|---|
| `cod` | `*cod*` |
| `microplastic mareano` | `(*microplastic* OR *mareano*)` |
| `"one ocean expedition"` | `"one ocean expedition"` |
| `Argo` | `*argo*` |

Free text matches against `Entry_Title` and `Data_Summary` (and a few other fields the Solr schema indexes). Programme names like Argo, Mareano, MOSAiC, and "One Ocean Expedition" generally appear in titles, so free text catches them reliably.
```

- [ ] **Step 3.3: Replace empty `### Provider` with**

```markdown
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
```

- [ ] **Step 3.4: Replace empty `### Scientific keyword` with**

```markdown
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
```

- [ ] **Step 3.5: Verify the documented patterns work live**

Test the provider example:

```bash
curl -sf 'http://metadata.nmdc.no/metadata-api/search?q=Provider:%22University%20of%20Bergen%22&offset=0' | \
python3 -c "import sys,json; d=json.load(sys.stdin); print(f'numFound={d[\"numFound\"]}'); assert d['numFound']>0, 'expected hits'"
```

Expected: `numFound=` followed by a positive integer.

Test a free-text query:

```bash
curl -sf 'http://metadata.nmdc.no/metadata-api/search?q=*cod*&offset=0' | \
python3 -c "import sys,json; d=json.load(sys.stdin); print(f'numFound={d[\"numFound\"]}')"
```

Expected: positive integer.

Test a scientific keyword query (URL-encoded path):

```bash
curl -sf 'http://metadata.nmdc.no/metadata-api/search?q=Scientific_Keyword%3A%22EARTH%20SCIENCE%3EOCEANS%3EOCEAN%20TEMPERATURE%22&offset=0' | \
python3 -c "import sys,json; d=json.load(sys.stdin); print(f'numFound={d[\"numFound\"]}')"
```

Expected: positive integer.

If any of these return zero or fail, **fix the documented pattern in the SKILL.md before continuing** — don't ship an example that doesn't work.

- [ ] **Step 3.6: Commit**

```bash
git add skills/nmdc-explore/SKILL.md
git commit -m "$(cat <<'EOF'
Document free-text, provider, and keyword query patterns

Adds the introductory paragraph and three of the four clause-type
subsections under Query construction. Each subsection shows the
exact Solr clause emitted plus user-facing input examples.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Fill query construction — geographic GeoJSON workflow, temporal, combining

**Files:**
- Modify: `skills/nmdc-explore/SKILL.md`

- [ ] **Step 4.1: Replace empty `### Geographic — GeoJSON workflow` with**

````markdown
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
````

- [ ] **Step 4.2: Replace empty `### Temporal` with**

```markdown
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
```

- [ ] **Step 4.3: Replace empty `### Combining and pagination` with**

```markdown
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
```

- [ ] **Step 4.4: Verify the geographic and temporal patterns work live**

Test a polygon-with-Intersects query (small bbox around Oslofjord):

```bash
Q='location_rpt:"Intersects(POLYGON((10.40 59.65, 10.95 59.65, 10.95 59.92, 10.40 59.92, 10.40 59.65)))"'
curl -sf "http://metadata.nmdc.no/metadata-api/search?q=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "$Q")&offset=0" | \
python3 -c "import sys,json; d=json.load(sys.stdin); print(f'numFound={d[\"numFound\"]}')"
```

Expected: a non-negative integer (Oslofjord might have few records — zero is acceptable, but the query must not error).

Test a temporal+provider combination:

```bash
curl -sf "http://metadata.nmdc.no/metadata-api/search?q=Provider:%22University%20of%20Bergen%22&beginDate=2020-01-01T00:00:00Z&endDate=2020-12-31T23:59:59Z&dateSearchMode=isWithin&offset=0" | \
python3 -c "import sys,json; d=json.load(sys.stdin); print(f'numFound={d[\"numFound\"]}')"
```

Expected: positive integer.

If either query errors or zero-out unexpectedly, fix the documented format in the SKILL.md before committing.

- [ ] **Step 4.5: Commit**

```bash
git add skills/nmdc-explore/SKILL.md
git commit -m "$(cat <<'EOF'
Document GeoJSON geo workflow and temporal parsing

Adds the geographic, temporal, and combining-and-pagination
subsections. Geo subsection details the 5-step file-based
human-in-the-loop bbox validation. Temporal subsection includes
both English and Norwegian month names plus dateSearchMode rules.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Fill detail view, ODP feasibility, examples coverage, failure modes

**Files:**
- Modify: `skills/nmdc-explore/SKILL.md`

- [ ] **Step 5.1: Replace empty `### Default — catalog only` with**

```markdown
### Default — catalog only

When the user drills into a single dataset, render in this order:

1. **Title** — `Entry_Title`
2. **Identifier** — `Entry_ID`
3. **DOI** — `Dataset_DOI` (linked as `https://doi.org/<doi>`)
4. **Provider(s)** — parsed from the stringified array
5. **Personnel** — first author from `Dataset_Creator`, plus `PersonFirstName` / `PersonLastName` if useful
6. **Summary** — `Data_Summary`, first 400 characters with ellipsis if longer
7. **Temporal coverage** — `Start_Date` → `Stop_Date`, normalised to `YYYY-MM-DD`
8. **Spatial coverage** — bbox of the `location_rpt` polygon (or the point coordinates). Also write `nmdc-detail-<entryid>.geojson` to cwd so the user can open it visually.
9. **License** — parsed from the landing page (`GET /metadata-api/landingpage/<hash>`). Look for "Use Constraints" or "License" anchor text. If not found, mark as "unknown".
10. **File list** — parsed from the same landing page. For each `Data_URL`:
    - Filename (last path segment)
    - Type tag from `Data_URL_Type` / `Data_URL_Subtype` (`OPENDAP DATA (DODS)`, `GET DATA`, `Access to OPeNDAP service`, `PARENT`, …)
    - Stated size if the metadata mentions one (e.g. "190 CTD profiles", "18.7 days of data", "~340 MB")
    - URL
11. **Landing page URL** — `landingpage` field

Skip fields that are empty rather than rendering "n/a".
```

- [ ] **Step 5.2: Replace empty `### On-demand probe` with**

```markdown
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
```

- [ ] **Step 5.3: Replace empty `## ODP feasibility advisory` with**

````markdown
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
````

- [ ] **Step 5.4: Replace empty `## Examples coverage` with**

```markdown
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
| Find microplastic data from Mareano | `q=(*microplastic* OR *mareano*)` (both as free text — Mareano matches in titles and provider) |
| Find datasets captured last month | `beginDate=<first day of previous month>`, `endDate=<last day of previous month>`, `dateSearchMode=isWithin` (resolved against today) |

Sort-style prompts ("the latest dataset in the Oslofjord", "most recent NIVA data") are not supported. Ask the user for a concrete period instead — the API has no `sort` parameter.
```

- [ ] **Step 5.5: Replace empty `## Failure modes` with**

```markdown
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
```

- [ ] **Step 5.6: Verify final SKILL.md is well-formed**

Run:

```bash
python3 -c "
import yaml, re
text = open('skills/nmdc-explore/SKILL.md').read()
end = text.index('\n---\n', 4)
fm = yaml.safe_load(text[4:end])
print('frontmatter keys:', list(fm.keys()))

body = text[end+5:]
required_sections = [
    '## When to use this skill',
    '## Out of scope',
    '## Backend overview',
    '## Capabilities',
    '## Query construction',
    '### Free text',
    '### Provider',
    '### Scientific keyword',
    '### Geographic — GeoJSON workflow',
    '### Temporal',
    '### Combining and pagination',
    '## Detail view',
    '### Default — catalog only',
    '### On-demand probe',
    '## ODP feasibility advisory',
    '## Examples coverage',
    '## Failure modes',
]
missing = [s for s in required_sections if s not in body]
assert not missing, f'missing sections: {missing}'

# No placeholders
for bad in ['TODO', 'TBD', 'FIXME', 'XXX']:
    assert bad not in body, f'placeholder found: {bad}'
print(f'OK: all {len(required_sections)} sections present, no placeholders')
"
```

Expected: `OK: all 17 sections present, no placeholders`.

- [ ] **Step 5.7: Commit**

```bash
git add skills/nmdc-explore/SKILL.md
git commit -m "$(cat <<'EOF'
Document detail view, ODP feasibility, examples, failure modes

Completes the body of the nmdc-explore skill: catalog-default and
on-demand-probe rendering for single datasets, three-axis ODP
feasibility classifier with verdict-plus-recipe-sketch shape, the
ten example prompts mapped to concrete clauses, and the red-flag
catalogue.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: End-to-end live integration test against real prompts

**Files:**
- Modify: `skills/nmdc-explore/SKILL.md` (only if a documented pattern is wrong)

This task does **no new authoring** — it walks through a sample of the 10 example prompts, composes the queries the skill prescribes, runs them, and verifies the results are sane. If anything diverges, fix the SKILL.md.

- [ ] **Step 6.1: Run "University of Bergen 2026" end-to-end**

```bash
curl -sf "http://metadata.nmdc.no/metadata-api/search?q=Provider:%22University%20of%20Bergen%22&beginDate=2026-01-01T00:00:00Z&endDate=2026-12-31T23:59:59Z&dateSearchMode=isWithin&offset=0" | \
python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'numFound={d[\"numFound\"]}, page1={len(d[\"results\"])}')
for r in d['results'][:3]:
    print(' -', r['Entry_ID'], '|', r['Provider'], '|', r.get('Start_Date',''), '→', r.get('Stop_Date',''))
"
```

Expected: `numFound` is a non-negative integer; if positive, the printed records have Provider containing "University of Bergen". If you get hits with the wrong provider, the Solr field name or quoting in the SKILL.md is wrong — fix and re-test.

- [ ] **Step 6.2: Run "microplastic + mareano" end-to-end**

```bash
curl -sf "http://metadata.nmdc.no/metadata-api/search?q=%28%2Amicroplastic%2A%20OR%20%2Amareano%2A%29&offset=0" | \
python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'numFound={d[\"numFound\"]}, page1={len(d[\"results\"])}')
for r in d['results'][:5]:
    title = r.get('Entry_Title','')[:80]
    print(' -', r['Entry_ID'], '|', title)
"
```

Expected: positive `numFound`. Eyeball the titles — they should mention either microplastic or Mareano (or both).

- [ ] **Step 6.3: Run a Skagerrak geographic query end-to-end**

Skagerrak rough bbox: `[8.5, 57.5, 11.5, 59.0]` (lon-lat). Polygon ring is closed (last vertex == first).

```bash
Q='*cod* AND location_rpt:"Intersects(POLYGON((8.5 57.5, 11.5 57.5, 11.5 59.0, 8.5 59.0, 8.5 57.5)))"'
ENC=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "$Q")
curl -sf "http://metadata.nmdc.no/metadata-api/search?q=${ENC}&offset=0" | \
python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'numFound={d[\"numFound\"]}, page1={len(d[\"results\"])}')
for r in d['results'][:3]:
    print(' -', r['Entry_ID'], '|', r.get('Entry_Title','')[:60], '|', r.get('location_rpt',''))
"
```

Expected: returns without error. `numFound` may be small or zero — that's fine; what matters is the polygon syntax doesn't make Solr error out.

- [ ] **Step 6.4: Run getFacets and confirm provider list**

```bash
curl -sf 'http://metadata.nmdc.no/metadata-api/getFacets' | \
python3 -c "
import sys, json
d = json.load(sys.stdin)
prov = next(f for f in d['facets'] if f['name'] == 'Provider')
names = [c['value'] for c in prov['children']]
expected = ['Institute of Marine Research', 'University of Bergen', 'Norwegian Polar Institute']
for e in expected:
    matches = [n for n in names if e.lower() in n.lower()]
    print(f'{e}: {matches[:2]}')
"
```

Expected: each "expected" name has at least one match in the live list. If any don't, your alias-map canonical names in SKILL.md are wrong — update them.

- [ ] **Step 6.5: Run a landing-page parse for the file list**

```bash
curl -sf 'http://metadata.nmdc.no/metadata-api/landingpage/fde3cb4955662933c39ea781b7607f10' -o /tmp/lp.html && \
python3 -c "
text = open('/tmp/lp.html').read()
# the skill says we look for OPeNDAP type tags and Data_URLs in the page
markers = ['OPENDAP', 'Data_URL', 'opendap', 'CC BY', 'CEOS']
print('present:', [m for m in markers if m in text])
"
```

Expected: at least 3 of the markers appear. If fewer, the landing-page parse instructions in SKILL.md need to be loosened or made more tolerant — update the "License" and "File list" steps in detail view.

- [ ] **Step 6.6: If any of the previous steps surfaced a fix, commit it**

```bash
# Only run if SKILL.md was modified during the integration test.
git diff --quiet skills/nmdc-explore/SKILL.md || (
  git add skills/nmdc-explore/SKILL.md
  git commit -m "$(cat <<'EOF'
Fix documented patterns to match live API

Integration test against metadata.nmdc.no surfaced one or more
discrepancies between the skill's documented queries/parsing and
the live response shape. Patterns updated to match.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
)
```

If the diff is empty (everything matched), this step is a no-op — skip the commit.

---

## Task 7: Add nmdc-explore to README

**Files:**
- Modify: `README.md`

- [ ] **Step 7.1: Read the current skills table to find the right insertion point**

```bash
grep -n '|' /Users/joovstaas/Projects/awesome-claude-skills/README.md | head -20
```

This locates the "Available Skills" markdown table. The table currently lists `odp-data-exploration`, `odp-data-ingest`, `odp-data-consume`, `odp-stac-api`, `mapbox-odp-maps`, `vannmiljo`. Insert the new row at the end of the data rows (before the next blank line / next section).

- [ ] **Step 7.2: Insert the new skill row**

Use the Edit tool to add this row after the `vannmiljo` row in the table:

```markdown
| [nmdc-explore](skills/nmdc-explore/) | Search and inspect datasets in the [Norwegian Marine Data Centre (NMDC)](http://metadata.nmdc.no/UserInterface/) — natural-language queries by keyword, provider, geography, and period; dataset detail views with file lists and on-demand link/size probing; and ODP ingestion feasibility advice |
```

- [ ] **Step 7.3: Verify the table still renders cleanly**

```bash
python3 -c "
import re
text = open('/Users/joovstaas/Projects/awesome-claude-skills/README.md').read()
# Find the skills table
m = re.search(r'## Available Skills.*?(?=\n## )', text, re.DOTALL)
assert m, 'skills table section not found'
section = m.group(0)
rows = [r for r in section.split('\n') if r.startswith('|') and '---' not in r]
print(f'found {len(rows)} table rows (header + data)')
assert any('nmdc-explore' in r for r in rows), 'new row missing'
# every data row has same column count
counts = {r.count('|') for r in rows}
assert len(counts) == 1, f'inconsistent column counts: {counts}'
print('table well-formed')
"
```

Expected: positive count, "table well-formed".

- [ ] **Step 7.4: Commit**

```bash
git add README.md
git commit -m "$(cat <<'EOF'
Add nmdc-explore to README

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Final review and PR readiness

**Files:** none (review only)

- [ ] **Step 8.1: Re-read SKILL.md from top to bottom in one pass**

Read the whole file and check, in your head:

- Does the frontmatter `description` actually trigger on every example prompt the user listed? (Mention: NMDC, Norwegian Marine Data Centre, Norwegian/Nordic marine data, by topic / place / period.)
- Is every section needed? (No filler.)
- Any contradiction between sections? (e.g., "page size is 10" said once but treated as variable elsewhere.)
- Does the file flow logically: trigger → boundary → backend reference → capabilities → how-to → details → failure?

If anything is off, fix inline and amend the most recent commit (do not amend a published commit if the branch has already been pushed; prefer a new commit).

- [ ] **Step 8.2: Final sanity sweep**

```bash
# No emoji slipped in
python3 -c "
import re
text = open('skills/nmdc-explore/SKILL.md').read()
emoji = re.findall(r'[\U0001F300-\U0001FAFF\U0001F600-\U0001F64F☀-➿]', text)
assert not emoji, f'emoji found: {emoji}'
print('OK: no emoji')
"

# Word count sanity check (skill should be neither tiny nor monstrous)
wc -w skills/nmdc-explore/SKILL.md
```

Expected: `OK: no emoji`. Word count: roughly 1500–3500 words (similar order of magnitude as the existing skills in the repo).

- [ ] **Step 8.3: Branch state check**

```bash
git log --oneline main..HEAD
git status
```

Expected: clean working tree, ~6 commits on the branch, all on top of `main`.

- [ ] **Step 8.4: Hand back to the user**

Tell the user:

> "Skill complete on branch `add-nmdc-explore-skill`. Files: `skills/nmdc-explore/SKILL.md` and the updated README. To use it locally, copy or symlink `skills/nmdc-explore/SKILL.md` into `.claude/skills/nmdc-explore/`. Want me to open a PR?"

Wait for the user's call on PR-or-not. Don't open the PR autonomously.

---

## Self-review checklist

(Done by the plan author before handing off.)

- ✅ **Spec coverage:** Every section of the design spec has a task. Identity & scope → Task 2.1–2.4. Query construction → Tasks 3 & 4. Detail view → Task 5.1–5.2. ODP feasibility → Task 5.3. Examples coverage → Task 5.4. Failure modes → Task 5.5. Live validation → Task 6. README integration → Task 7.
- ✅ **No placeholders:** No "TODO", "TBD", "fill in later". Every step shows the exact prose / code / command to run.
- ✅ **Type consistency:** Field names (`Entry_ID`, `numFound`, `dateSearchMode`, `location_rpt`) are used identically across tasks. Operator names (`Intersects` vs `IsWithin`) match.
- ✅ **Frequent commits:** ~6 commits across the plan, one per cohesive section + one for README.
