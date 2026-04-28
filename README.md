# Awesome Claude Skills

A curated collection of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for data engineering, geospatial workflows, and ocean data.

## What Are Claude Code Skills?

Skills are reusable knowledge files (Markdown) that teach Claude Code domain-specific patterns, APIs, and best practices. When installed, Claude Code automatically activates the relevant skill based on your task — no manual prompting needed.

## Available Skills

| Skill | Description |
|-------|-------------|
| [odp-data-exploration](skills/odp-data-exploration/) | Data-quality check on an unknown file (CSV, Parquet, JSON, GeoJSON, Shapefile, GeoTIFF, NetCDF, JPEG, GPX) — inventory + syntactic checks, optional semantic and protocol-change checks (sensor swaps, calibration shifts), pre-ingest advice for [ODP](https://hubocean.earth). Runs only what was asked; writes reports in the user's language; never overwrites raw files |
| [odp-data-ingest](skills/odp-data-ingest/) | Ingest data into [Ocean Data Platform (ODP)](https://hubocean.earth) — datasets, file uploads, tabular data with PyArrow schemas, and spatial data with WKT geometries |
| [odp-data-consume](skills/odp-data-consume/) | Consume and query data from [Ocean Data Platform (ODP)](https://hubocean.earth) — tabular queries, geospatial filtering, aggregation, file downloads, and GeoJSON reconstruction |
| [odp-stac-api](skills/odp-stac-api/) | Search for ocean data, marine datasets, and geospatial collections from [ODP](https://hubocean.earth) using the STAC API — spatial/temporal queries, collections, and items |
| [mapbox-odp-maps](skills/mapbox-odp-maps/) | Build interactive maps with [Mapbox GL JS](https://docs.mapbox.com/mapbox-gl-js/) and ODP geodata in Next.js — setup, dynamic imports, CSS loading, geometry conversion, layer management, and common pitfalls |
| [vannmiljo](skills/vannmiljo/) | Vannmiljo data pipeline — scrape water monitoring data from [vannmiljo.miljodirektoratet.no](https://vannmiljo.miljodirektoratet.no), build GeoJSON, and ingest into ODP |
| [nmdc-explore](skills/nmdc-explore/) | Search and inspect datasets in the [Norwegian Marine Data Centre (NMDC)](http://metadata.nmdc.no/UserInterface/) — natural-language queries by keyword, provider, geography, and period; dataset detail views with file lists and on-demand link/size probing; and ODP ingestion feasibility advice |

## Installation

### Option 1: Copy into your project

```bash
# Clone this repo
git clone https://github.com/joevstaas/awesome-claude-skills.git

# Copy a skill into your project's .claude/skills/ directory
mkdir -p .claude/skills/odp-data-ingest
cp awesome-claude-skills/skills/odp-data-ingest/SKILL.md .claude/skills/odp-data-ingest/
```

### Option 2: Copy into your global Claude config

To make a skill available across all projects:

```bash
mkdir -p ~/.claude/skills/odp-data-ingest
cp awesome-claude-skills/skills/odp-data-ingest/SKILL.md ~/.claude/skills/odp-data-ingest/
```

## Contributing

Contributions are welcome! To add a new skill:

1. Create a directory under `skills/` with a descriptive name
2. Add a `SKILL.md` file with the YAML frontmatter (`description`) and comprehensive documentation
3. Include real code examples that have been tested against the actual APIs
4. Document known gotchas and limitations
5. Open a pull request

## License

MIT
