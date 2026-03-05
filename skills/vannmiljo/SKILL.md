---
description: Vannmiljo data pipeline — scrape water monitoring data from vannmiljo.miljodirektoratet.no, build GeoJSON, and ingest into ODP
---

# Vannmiljo Data Pipeline Skill

Use this skill when working with Vannmiljo (Norwegian water monitoring) data — scraping, GeoJSON building, or ingesting into ODP.

## Project Overview

The Vannmiljo project extracts water quality monitoring data from Miljodirektoratet's Vannmiljo database and ingests it into the Ocean Data Platform (ODP).

**Pipeline:** Vannmiljo web scraper → GeoJSON files → ODP datasets

**Source:** vannmiljo.miljodirektoratet.no
**Field documentation:** vannmiljokoder.miljodirektoratet.no/files/Om_VMS_Registreringsfelt.htm

## Project Structure

```
Vannmiljo/
├── geojson_builder.py          # Builds GeoJSON from scraped Vannmiljo data
├── ingest_holmestrandsfjorden.py  # Ingest script for Holmestrandsfjorden
├── test_column_metadata.py     # Test script for ODP column metadata
├── input_polygons/             # GeoJSON polygons defining areas of interest
│   ├── holmestrandsfjorden.geojson
│   ├── fordefjorden.geojson
│   └── langoya.geojson
└── output/                     # Generated GeoJSON files with measurements
    └── holmestrandsfjorden_alle_stasjoner.geojson
```

## PyArrow Schema for Vannmiljo Data

All Vannmiljo datasets share the same schema. All columns are `pa.string()` except `water_location_id` (int64). The `id` column is a generated UUID primary key.

```python
import pyarrow as pa

VANNMILJO_SCHEMA = pa.schema([
    pa.field("id", pa.string(), nullable=False),
    pa.field("water_location_id", pa.int64(), nullable=True,
             metadata={"description": "Unik ID for vannlokaliteten i Vannmiljo"}),
    pa.field("water_location_code", pa.string(), nullable=True,
             metadata={"description": "Kode som knytter registreringen til en bestemt vannlokalitet"}),
    pa.field("station_name", pa.string(), nullable=True,
             metadata={"description": "Navn på målestasjonen"}),
    pa.field("station_description", pa.string(), nullable=True,
             metadata={"description": "Beskrivelse av vannlokaliteten"}),
    pa.field("act_info", pa.string(), nullable=True,
             metadata={"description": "Aktivitetsinformasjon for stasjonen"}),
    pa.field("faktaark_url", pa.string(), nullable=True,
             metadata={"description": "URL til stasjonens faktaark i Vannmiljo"}),
    pa.field("betegnelse", pa.string(), nullable=True,
             metadata={"description": "Betegnelse for provetakingsaktiviteten"}),
    pa.field("aktivitet", pa.string(), nullable=True,
             metadata={"description": "Overvåkings- eller kartleggingsaktivitet registreringen tilhorer"}),
    pa.field("oppdragsgiver", pa.string(), nullable=True,
             metadata={"description": "Ansvarlig for anskaffelsen av overvåkingsoppdraget"}),
    pa.field("oppdragstaker", pa.string(), nullable=True,
             metadata={"description": "Utforer av overvåkingen"}),
    pa.field("parameter_id", pa.string(), nullable=True,
             metadata={"description": "ID for parameteren som er målt eller observert"}),
    pa.field("parameter", pa.string(), nullable=True,
             metadata={"description": "Navn på parameteren som er målt eller observert"}),
    pa.field("cas_nr", pa.string(), nullable=True,
             metadata={"description": "CAS-nummer for kjemisk komponent"}),
    pa.field("medium", pa.string(), nullable=True,
             metadata={"description": "Medium parameteren er målt i (vann, sediment, biota, partikler)"}),
    pa.field("vitenskapelig_navn", pa.string(), nullable=True,
             metadata={"description": "Vitenskapelig (latinsk) navn på art/takson"}),
    pa.field("provetakingsmetode", pa.string(), nullable=True,
             metadata={"description": "Standard provetakingsmetode benyttet"}),
    pa.field("analysemetode", pa.string(), nullable=True,
             metadata={"description": "Standard analysemetode benyttet"}),
    pa.field("provenr", pa.string(), nullable=True,
             metadata={"description": "Provenummer for replikate prover"}),
    pa.field("filtrert_prove", pa.string(), nullable=True,
             metadata={"description": "Om vannproven er filtrert gjennom 0,45 um filter"}),
    pa.field("opprinnelse", pa.string(), nullable=True,
             metadata={"description": "Opprinnelse for registreringen"}),
    pa.field("provedato", pa.string(), nullable=True,
             metadata={"description": "Tidspunkt for provetaking"}),
    pa.field("operator", pa.string(), nullable=True,
             metadata={"description": "Verdioperator: = (påvist), < (under grense), > (over område), ND (ikke påvist)"}),
    pa.field("verdi", pa.string(), nullable=True,
             metadata={"description": "Målt eller observert verdi"}),
    pa.field("enhet", pa.string(), nullable=True,
             metadata={"description": "Enhet for registrert verdi"}),
    pa.field("listenavn", pa.string(), nullable=True,
             metadata={"description": "Navn på verdilisten parameteren tilhorer"}),
    pa.field("ovre_dyp", pa.string(), nullable=True,
             metadata={"description": "Ovre dybdegrense i meter (vann) eller centimeter (sediment)"}),
    pa.field("nedre_dyp", pa.string(), nullable=True,
             metadata={"description": "Nedre dybdegrense i meter (vann) eller centimeter (sediment)"}),
    pa.field("deteksjonsgrense", pa.string(), nullable=True,
             metadata={"description": "Deteksjonsgrensen til den aktuelle måleverdi"}),
    pa.field("kvantifiseringsgrense", pa.string(), nullable=True,
             metadata={"description": "Kvantifiseringsgrensen for den aktuelle måleverdi"}),
    pa.field("kommentarer", pa.string(), nullable=True,
             metadata={"description": "Kommentarer knyttet til registreringen (maks 2000 tegn)"}),
    pa.field("sist_endret", pa.string(), nullable=True,
             metadata={"description": "Dato for siste endring av registreringen"}),
    pa.field("geometry", pa.string(), nullable=True,
             metadata={
                 "isGeometry": "1",
                 "index": "1",
                 "description": "Punkt-geometri for målestasjonen (WKT)",
             }),
])
```

## Ingest Script Template

All Vannmiljo area ingest scripts follow the same pattern. Use `ingest_holmestrandsfjorden.py` as the reference implementation.

### Steps

1. **Check for existing dataset** via `get_dataset_meta_by_name()`, support `--clean` flag
2. **Load GeoJSON** from `output/<area>_alle_stasjoner.geojson`
3. **Create dataset** with descriptive name and Norwegian description
4. **Add to collection** `6a107394-4dfb-4095-96ea-6cd0607cb213`
5. **Upload raw GeoJSON** as file via `ds.files.upload()`
6. **Create table** with the shared `VANNMILJO_SCHEMA`
7. **Insert rows** in batches of 5000, converting geometry to WKT

### Geometry Conversion

GeoJSON Point `[lon, lat]` → WKT `POINT (lon lat)`:

```python
def point_to_wkt(geometry: dict | None) -> str | None:
    if not geometry or geometry.get("type") != "Point":
        return None
    coords = geometry.get("coordinates")
    if not coords or len(coords) < 2:
        return None
    return f"POINT ({coords[0]} {coords[1]})"
```

### Row Conversion

Each GeoJSON feature becomes a table row:
- Generate UUID for `id` (primary key)
- `water_location_id` → cast to `int`
- All other properties → cast to `str` or `None`
- Geometry → WKT via `point_to_wkt()`

## Column Descriptions Reference

Descriptions are sourced from Vannmiljo's official field documentation:
`vannmiljokoder.miljodirektoratet.no/files/Om_VMS_Registreringsfelt.htm`

| Column | Description |
|--------|-------------|
| water_location_id | Unik ID for vannlokaliteten i Vannmiljo |
| water_location_code | Kode som knytter registreringen til en bestemt vannlokalitet |
| station_name | Navn på målestasjonen |
| station_description | Beskrivelse av vannlokaliteten |
| act_info | Aktivitetsinformasjon for stasjonen |
| faktaark_url | URL til stasjonens faktaark i Vannmiljo |
| betegnelse | Betegnelse for provetakingsaktiviteten |
| aktivitet | Overvåkings- eller kartleggingsaktivitet |
| oppdragsgiver | Ansvarlig for anskaffelsen av overvåkingsoppdraget |
| oppdragstaker | Utforer av overvåkingen |
| parameter_id | ID for parameteren som er målt eller observert |
| parameter | Navn på parameteren |
| cas_nr | CAS-nummer for kjemisk komponent |
| medium | Medium parameteren er målt i |
| vitenskapelig_navn | Vitenskapelig (latinsk) navn på art/takson |
| provetakingsmetode | Standard provetakingsmetode |
| analysemetode | Standard analysemetode |
| provenr | Provenummer for replikate prover |
| filtrert_prove | Om vannproven er filtrert gjennom 0,45 um filter |
| opprinnelse | Opprinnelse for registreringen |
| provedato | Tidspunkt for provetaking |
| operator | Verdioperator: =, <, >, ND |
| verdi | Målt eller observert verdi |
| enhet | Enhet for registrert verdi |
| listenavn | Navn på verdilisten |
| ovre_dyp | Ovre dybdegrense i meter/centimeter |
| nedre_dyp | Nedre dybdegrense i meter/centimeter |
| deteksjonsgrense | Deteksjonsgrensen til måleverdi |
| kvantifiseringsgrense | Kvantifiseringsgrensen for måleverdi |
| kommentarer | Kommentarer (maks 2000 tegn) |
| sist_endret | Dato for siste endring |
| geometry | Punkt-geometri (WKT) |

## ODP Collection

All Vannmiljo datasets belong to collection: `6a107394-4dfb-4095-96ea-6cd0607cb213`

## Areas

| Area | Input Polygon | GeoJSON Output | ODP Dataset |
|------|--------------|----------------|-------------|
| Holmestrandsfjorden | `input_polygons/holmestrandsfjorden.geojson` | `output/holmestrandsfjorden_alle_stasjoner.geojson` | Vannmiljo Holmestrandsfjorden |
| Fordefjorden | `input_polygons/fordefjorden.geojson` | `output/fordefjorden_alle_stasjoner.geojson` | (pending) |
| Langoya | `input_polygons/langoya.geojson` | `output/langoya_alle_stasjoner.geojson` | (pending) |
