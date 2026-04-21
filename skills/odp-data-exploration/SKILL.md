---
description: Triage unknown data files (CSV, Parquet, JSON, GeoJSON, Shapefile, GeoTIFF, NetCDF, JPEG, GPX, …). Runs inventory + syntactic + semantic data-quality checks with a geospatial focus, detects mid-dataset protocol changes (sensor swaps, calibration shifts, changed measurement conventions), and produces pre-ingestion advice for the Ocean Data Platform (ODP). Use when the user wants to understand, quality-check, or prepare a data file for visualisation or ingestion.
---

# ODP Data Exploration

## When to use this skill

Invoke when the user:
- Hands over a data file and asks *what is this / is it any good / can I ingest this / any outliers*
- Is preparing data for visualisation and wants to know what to filter first
- Suspects a dataset changed partway through (sensor/firmware/calibration change)
- Is about to ingest to ODP and wants a pre-flight check

Skip when the user is asking a pure coding question, or when they want the actual ODP upload mechanics — defer to the `odp-data-ingest` skill for ingestion, `odp-data-consume` for reading existing ODP datasets.

## Core principles

1. **Two passes always, in this order.** Syntactic first (is it well-*formed*?), semantic second (does it make sense in the *world*?). Mixing them hides real issues.
2. **Never modify raw data in place.** Produce a *cleaned derivative*; the raw file stays untouched. When recommending action, phrase as filter/transform rules, not mutations.
3. **Three outcomes per issue: drop the row / drop the field / keep + flag.** Most noisy data has *some* real information — dropping whole rows because one channel is broken is wasteful.
4. **Plain-language output.** The user may not speak data-science jargon. Translate every numeric finding into a takeaway: *"41 fixes at exactly (0, 0) — failed GPS acquisitions the tag logged as valid; drop before mapping."* Numbers alone are not a report.
5. **Geospatial means "where on Earth" is a first-class dimension.** Always report bbox, CRS, and — for outliers — the actual place name or a distance from something the user recognises. Include OpenStreetMap links (`https://www.openstreetmap.org/?mlat=<lat>&mlon=<lon>&zoom=11`) for individual suspect points.
6. **Honest about role.** The skill runs the code; the user decides what to do with findings. Surface decisions, don't make them silently.

## Workflow — four passes

Work through all four, in order. Skip a pass only if the file truly doesn't support it (e.g. protocol-change is meaningless on a single-image file).

### Pass 1 — Inventory

Goal: know what you're looking at before checking quality.

Detect format from **both extension and magic bytes** (a `.json` file may actually be NDJSON, a `.csv` may be TSV, a `.tif` may be plain TIFF not GeoTIFF). Use `file <path>` and library sniffing, not extension alone.

Report:
- **Format & encoding** (utf-8? latin-1? BOM?)
- **Volume** — rows × columns / features / pixels / bytes; file size on disk
- **Schema** — per column: dtype, null %, n_unique, 3 example values
- **Spatial extent** — CRS, bbox (min/max lat, lon), geometry types, projected vs geographic
- **Temporal extent** — min/max timestamp, time zone, cadence
- **Entity structure** — is there a natural key (`device_id`, `station_id`, `track_id`)? How many entities?
- **For tabular with mixed event types** — distribution of the "kind-of-row" column (like `datatype` in biologger CSVs); different kinds have different required fields

### Pass 2 — Syntactic data quality

Goal: detect anything structurally broken. Keep this pass narrow and mechanical.

Standard checks:
- Field-count consistency (every row has the declared number of columns)
- **Type coercion surprises** — columns declared numeric that contain strings. Common sign of schema overload: a column reused to hold a text label for a subset of rows.
- Date/timestamp parseability; internal consistency of redundant date fields
- Encoding issues (mojibake, mixed line endings, stray BOM mid-file)
- Duplicate rows (full) and duplicate natural keys (`entity + timestamp + event_type`)
- Required-field completeness *per kind of row* (not globally — a footer row legitimately lacks data)
- Unknown enum values in categorical columns
- **Geometry validity** (for vector data) — `.is_valid`, self-intersections, empty geometries, ring direction
- **Raster integrity** — readable bands, consistent georeferencing across tiles, declared vs actual nodata
- **Schema drift** across multi-file datasets — same column name, different dtype; or missing columns in some files

### Pass 3 — Semantic data quality (geospatial-aware)

Goal: does the data make sense *in the world*? This is where domain knowledge comes in.

**Generic geo checks (always run):**
- `latitude ∈ [-90, 90]`, `longitude ∈ [-180, 180]`
- **Null-island detection**: exact `(0, 0)` is almost always a failed fix, not a real location in the Gulf of Guinea
- CRS declared vs coordinate values (values in the 100,000s with CRS claiming EPSG:4326 = projected data mislabelled as geographic)
- Points falling on land for marine data, or in ocean for terrestrial data — use a coarse land/ocean mask (Natural Earth or `geopandas.datasets`)
- Bounding box sanity: does the declared bbox match the actual data extent?
- For tracks: implied speed between consecutive fixes per entity; cap by physically plausible speed for the platform (ship ~40 km/h, seabird ~90 km/h, glider ~3 km/h, drifting buoy ~5 km/h)

**Domain-specific checks (ask the user):**
Before running these, ask for a **plausibility spec** — one line per column:

```
column_name: (min, max, unit, notes)
```

Example:
```
water_temperature_C: (-2, 32, °C, freezing point of seawater to tropical max)
salinity_PSU:       (0, 40, PSU)
dive_depth_m:       (0, 200, m, species-specific max)
```

Don't guess these. Ask. If the user doesn't know, ask for a reference dataset or literature value, or mark the check as skipped.

**Cross-field consistency:**
- `altitude` vs `depth` vs `speed` — impossible combinations (below-sea-level altitude while surface speed reported)
- Timestamp monotonicity per entity
- Quality flags vs data (row flagged valid but all sensors NaN?)
- `satcount` vs `hdop` on GPS — few satellites shouldn't coexist with good geometry

**Outlier localisation — for every suspect point, report where it is.** Not just `(lat, lon)`; include distance to a known anchor (e.g. "113 km south of Oslo") and a one-click OSM link. A user can't judge whether an outlier is plausible unless they can visualise it.

### Pass 4 — Protocol-change detection

Goal: detect places where the *way data is being produced* changed mid-dataset. Symptoms of sensor swap, calibration change, firmware update, or a human starting to measure something differently.

This pass is often more valuable than raw outlier detection, because a protocol change can poison an *entire downstream analysis* silently.

Run these checks **per entity** (per device, per station, per track), not globally — a shift averaged across 10 sensors is invisible; a shift in one sensor is obvious.

**Signals to look for:**

1. **Distributional shift in a numeric column.** Split the entity's time series into halves (or sliding windows); compare means, std, and quantiles. A shift of more than ~3σ of the within-half std is suspicious. Visualise with a rolling mean plot — eyeballing is genuinely effective here.

2. **Missingness pattern shift.** A column that was 5 % null suddenly becomes 80 % null (or vice versa). Often indicates a sensor was disabled or added.

3. **Categorical value-set change.** Column was using codes `{A, B, C}` then starts using `{A, B, D}` or `{1, 2, 3}` — new firmware, new sensor model, recoded labels.

4. **Sampling-cadence shift.** Distribution of inter-row `dt` changes (1-minute rows become 10-minute, or bursty instead of steady). Firmware change or duty-cycle edit.

5. **Numeric resolution change.** A column reporting integers (`0, 1, 2, …`) suddenly reports fractional values, or vice versa. Different ADC, different units, or quantisation change.

6. **Range shift.** Column's min/max changes abruptly between windows (not gradually), e.g. a temperature that lived in `[10, 14]` starts living in `[8, 12]`.

7. **Metadata column change.** If the file carries `firmware_version`, `sensor_id`, `calibration_date`, `station_version`, anomalies should be aligned to changes in those. Always join a suspected protocol change against metadata columns first — often it's explicitly documented.

**Detection technique, simplest first:**
```python
# Rolling-window mean/std comparison, per entity
for entity_id, g in df.sort_values("t").groupby("entity_id"):
    s = g["value"].rolling(window=N, min_periods=N//2)
    rm = s.mean(); rs = s.std()
    # Flag where rm changes by > 3*rs from the overall median of rs
    ...
```

PELT / CUSUM / Bayesian change-point are overkill for first pass — a rolling-window plot almost always reveals the break if it's there. Escalate to formal change-point detection only if the user needs precise boundaries.

Always **report the change in plain language**: *"sensor 374294 reports conductivity around 35 PSU from Nov 5–Nov 11, then jumps to ~42 PSU from Nov 12 onwards — pattern consistent with a recalibration event. Confirm with deployment log before treating either half as truth."*

## Output: the triage report

Produce a single report with these sections, in this order:

```
# Triage report — <filename>

## 1. Inventory
  - format, size, rows/features, schema table, spatial extent, temporal extent, entities

## 2. Syntactic quality — N issues
  - one bullet per finding, severity, "what it means"

## 3. Semantic quality — N issues
  - one bullet per finding, with location (for geo) and plain-language interpretation
  - OSM links for individual suspect points

## 4. Protocol changes — N suspected
  - per-entity narrative; what column, what window, what kind of shift

## 5. ODP ingestion verdict
  - target dataset type (tabular / vector / raster / file)
  - volume class (trivial / normal / partition / stream)
  - blocking issues (CRS, invalid geometries, schema drift)
  - recommended pre-ingest transformations

## 6. Action list — ranked
  | Priority | Rule                                        | Outcome                  |
  | H        | drop rows where lat == 0 and lon == 0       | removes 41 failed fixes  |
  | H        | drop field altitude where depth_m > 0       | keeps position, removes  |
  |          |                                             | nonsensical altitude     |
  | M        | flag rows in conductivity-shift window      | don't modify, annotate   |
  | L        | dedupe (entity, timestamp, datatype)        | removes 9 duplicates     |
```

Rank actions High / Medium / Low based on: impact on downstream analysis, proportion of data affected, and reversibility.

If the user is going to act on the report, offer (don't assume) to write it to a file: `<filename>.triage.md` alongside the source. Never write the cleaned data automatically — the user decides which rules to apply.

## Format handlers — quick reference

### CSV / TSV
```python
import pandas as pd
df = pd.read_csv(path, low_memory=False)
# For dialect detection: csv.Sniffer on the first 16 KB
```

### Parquet
```python
import pyarrow.parquet as pq
t = pq.read_table(path)             # schema without full materialisation
print(t.schema, t.num_rows)
df = t.to_pandas()
```

### JSON / NDJSON
```python
# JSON: single object or list. NDJSON: one object per line.
# Detect: peek at first non-whitespace char; NDJSON if multiple top-level values
import json
with open(path) as f:
    first = f.read(1024).lstrip()
if first.startswith("["):
    data = json.load(open(path))                   # array of records
else:
    df = pd.read_json(path, lines=True)            # NDJSON
```

### GeoJSON / Shapefile / GeoPackage
```python
import geopandas as gpd
gdf = gpd.read_file(path)
print(gdf.crs, gdf.total_bounds, gdf.geometry.type.value_counts())
invalid = gdf[~gdf.geometry.is_valid]
```
If `crs is None`, that is a blocking issue for ODP ingest — ask the user.

### GeoTIFF / raster
```python
import rasterio
with rasterio.open(path) as src:
    print(src.crs, src.transform, src.shape, src.count, src.dtypes, src.nodata)
    # Read a small overview for sanity visualisation, not the full array
    arr = src.read(out_shape=(src.count, 512, 512))
```
Check: CRS set, transform not identity, nodata declared, no all-zero bands.

### NetCDF (CF conventions)
```python
import xarray as xr
ds = xr.open_dataset(path)
print(ds)                                # dims, coords, data_vars, attrs
# CF compliance — check attrs: units, standard_name, _FillValue
```
Common issues: time encoded as float days-since-epoch; check `ds.time.encoding`.

### JPEG / images
```python
from PIL import Image
from PIL.ExifTags import TAGS, GPSTAGS
img = Image.open(path)
exif = img._getexif() or {}
# Extract GPS EXIF if present — for geospatial image collections
```
For image sets: extract EXIF GPS into a table and triage that table as tabular.

### GPX
```python
import gpxpy
with open(path) as f:
    gpx = gpxpy.parse(f)
# Flatten to a DataFrame of (track, segment, point, lat, lon, ele, time) then triage tabular
```

## ODP ingestion advice

Route by data shape:

| Data shape                              | ODP dataset type | Notes                                                     |
|----------------------------------------|------------------|-----------------------------------------------------------|
| Rows with lat/lon columns              | tabular + geometry | Build WKT POINT geometry column; CRS → EPSG:4326         |
| Rows with polygons/tracks              | tabular + geometry | WKT LINESTRING / POLYGON                                  |
| Standalone vector features              | vector (GeoJSON) | Keep feature attributes; validate geometry                |
| Single-layer raster (continuous)        | raster           | Cloud-Optimized GeoTIFF; reproject to EPSG:4326 or 3857   |
| Multi-dimensional gridded               | file + raster    | NetCDF as file; slice per-time to raster if needed        |
| Arbitrary binary / images               | file             | Document each file in a sidecar manifest                  |

**Volume guidance** (rules of thumb; confirm with the user's ODP quota):
- `< 10 k` rows: trivial, upload as one chunk
- `10 k – 1 M`: normal, one dataset fine
- `1 M – 100 M`: consider partitioning by time or space; write as Parquet first
- `> 100 M`: stream in batches; use server-side Arrow filters when reading back

**Blocking issues — fix before ingest:**
- No CRS declared on geo data
- `lat`/`lon` swapped (detectable: latitude outside `[-90, 90]`)
- Mixed timezones or naive timestamps (normalise to UTC)
- Schema drift across files (reconcile to a single schema)
- Invalid geometries (apply `buffer(0)` or reject)
- Null-island and impossible-speed fixes (drop)

**Nice-to-have — fix if possible:**
- Inconsistent column naming (`lat` vs `latitude` vs `Latitude`)
- Mixed units within a column
- Undocumented enum values

For ingestion mechanics (SDK calls, manifest structure, schema registration), hand off to the `odp-data-ingest` skill.

## Working style — carry over from the user's established preferences

- Lead with plain-language interpretation; put numbers after, not before
- Use analogies when introducing new terminology — the user may be building vocabulary
- For every suspect point in geo data, tell the user **where it is**, not just its coordinates
- When recommending rules, write them as code the user can paste, not prose
- Be honest about analyst vs tooling: the skill runs checks; the user owns decisions
- Ask for plausibility ranges per column rather than guessing — domain knowledge lives with the user

## Pitfalls

- **Trusting file extension.** `.json` may be NDJSON. `.tif` may lack georef. `.csv` may be semicolon-separated (Norwegian/European locale). Always sniff.
- **Assuming WGS84.** Many datasets ship in UTM, national grids, or web mercator. Check CRS before plotting.
- **Averaging over entities.** A protocol change in one of 50 devices disappears in the grand mean. Always group by entity first.
- **Reporting "N outliers" without showing where.** Means nothing to a human. Always localise.
- **Silent mutation.** Never write to the raw file. Always derive.
- **Dropping rows without recording why.** Produce a reject log (`<filename>.rejected.csv` with a `reason` column) whenever rules are applied.
- **Treating protocol change as outlier noise.** If a shift is coherent (many columns change at the same timestamp), it's a protocol event, not noise — investigate before filtering.
- **Over-cleaning.** If 30 % of your rows get dropped, you are probably applying rules with too narrow a plausibility spec. Re-confirm with the user.
