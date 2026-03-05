---
description: Build interactive maps with Mapbox GL JS and ODP geodata in Next.js — covers setup, dynamic imports, CSS loading, geometry conversion, layer management, and common pitfalls
---

# Mapbox GL + ODP Maps Skill

Use this skill when the user wants to build or debug interactive maps that display geodata from the Ocean Data Platform (ODP) using Mapbox GL JS in a Next.js application.

## Prerequisites

### npm Dependencies

```bash
npm install mapbox-gl wkx apache-arrow
npm install -D @types/geojson
```

| Package | Purpose |
|---------|---------|
| `mapbox-gl` | Map rendering engine |
| `wkx` | Convert WKT/WKB geometry (from ODP) to GeoJSON |
| `apache-arrow` | Parse Arrow IPC binary format from ODP tabular API |

### Environment Variables

```env
NEXT_PUBLIC_MAPBOX_TOKEN=pk.your_token_here
ODP_API_KEY=sk_your_key_here
```

`NEXT_PUBLIC_MAPBOX_TOKEN` is client-side (public). `ODP_API_KEY` is server-side only.

## Next.js Setup — Critical Steps

### 1. next.config.ts

Mapbox GL and Apache Arrow need special configuration:

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  transpilePackages: ["mapbox-gl"],
  serverExternalPackages: ["apache-arrow"],
};
```

- `transpilePackages: ["mapbox-gl"]` — required because mapbox-gl ships untranspiled ESM
- `serverExternalPackages: ["apache-arrow"]` — Apache Arrow has native bindings that break in bundling

### 2. Mapbox GL CSS — Import in the map component

Load CSS via a direct import inside the map component file:

```tsx
// components/map/map-view.tsx
"use client";
import mapboxgl from "mapbox-gl";
import "mapbox-gl/dist/mapbox-gl.css";
```

This works reliably with the manual client-only import pattern (see step 3). **Do NOT add a `<head>` tag to layout.tsx** — Next.js 16 with Turbopack can error with "Missing `<html>` and `<body>` tags" if you add `<head>` manually in the root layout.

### 3. Client-Only Import Pattern — Do NOT use next/dynamic

**IMPORTANT (Next.js 16 / Turbopack)**: Do NOT use `next/dynamic` with `ssr: false` for the map component. In Next.js 16 with Turbopack, `dynamic()` with `ssr: false` triggers a false-positive runtime error: "Missing `<html>` and `<body>` tags in the root layout." This is caused by the SSR bailout mechanism confusing the Turbopack dev overlay.

Instead, use a **manual `useEffect` + lazy `import()` pattern**:

```tsx
// components/map/index.tsx (wrapper — client-only without next/dynamic)
"use client";
import { useEffect, useState, type ComponentType } from "react";

export function MapView() {
  const [Component, setComponent] = useState<ComponentType | null>(null);

  useEffect(() => {
    import("./map-view").then((mod) => setComponent(() => mod.default));
  }, []);

  if (!Component) {
    return (
      <div className="flex h-full w-full items-center justify-center bg-slate-100">
        <div className="h-8 w-8 animate-spin rounded-full border-2 border-blue-600 border-t-transparent" />
      </div>
    );
  }

  return <Component />;
}
```

```tsx
// components/map/map-view.tsx (actual map)
"use client";
import { useEffect, useRef } from "react";
import mapboxgl from "mapbox-gl";
import "mapbox-gl/dist/mapbox-gl.css";

export default function MapView() {
  const mapContainer = useRef<HTMLDivElement>(null);
  const map = useRef<mapboxgl.Map | null>(null);
  // ...
}
```

Import the wrapper (not the implementation) in pages:
```tsx
import { MapView } from "@/components/map";
```

### 4. Root Layout — Keep it minimal

Do NOT add `<head>`, Google Fonts, or `<link>` tags to the root layout. Next.js 16 Turbopack is strict about the root layout structure. Keep it simple:

```tsx
// app/layout.tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "My Map App",
  description: "...",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en">
      <body className="antialiased">{children}</body>
    </html>
  );
}
```

### 5. Map Container Sizing

The map container MUST have explicit dimensions. Use `h-full w-full` with a parent that has a defined height:

```tsx
// In the map component
return (
  <div className="relative h-full w-full">
    <div ref={mapContainer} className="h-full w-full" />
    {/* Overlays go here */}
  </div>
);

// In the page — parent MUST have height
<div className="relative flex-1">
  <MapView />
</div>
```

**Common pitfall**: Using `absolute inset-0` on the map container. This can cause the map canvas to render at the wrong size. Use `h-full w-full` instead.

**Height chain**: Ensure every ancestor up to the page root has a defined height. Typical pattern:
```
div.h-screen.flex.flex-col
  header.flex-shrink-0
  main.flex-1.overflow-hidden    ← use relative here
    MapView                      ← h-full w-full
```

## ODP → GeoJSON Pipeline

### Architecture Overview

```
ODP Tabular API (Arrow IPC binary)
  → apache-arrow (parse to rows)
  → wkx (WKT/WKB geometry → GeoJSON)
  → GeoJSON FeatureCollection
  → Next.js API route (serves JSON)
  → Mapbox GL (renders on map)
```

### Server-Side: ODP Client

```typescript
// lib/odp-client.ts
const ODP_BASE_URL = "https://api.hubocean.earth";

class ODPClient {
  private apiKey: string;

  constructor(apiKey: string) {
    this.apiKey = apiKey;
  }

  async queryTabularData(datasetId: string, options: {
    query?: string;
    sample?: number;
    columns?: string[];
  } = {}): Promise<ArrayBuffer> {
    const body: Record<string, unknown> = {};
    if (options.query) body.query = options.query;
    if (options.sample) body.sample = options.sample;
    if (options.columns) body.columns = options.columns;

    const response = await fetch(
      `${ODP_BASE_URL}/api/table/v2/sdk/select?table_id=${datasetId}`,
      {
        method: "POST",
        headers: {
          Authorization: `ApiKey ${this.apiKey}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify(body),
      }
    );

    if (!response.ok) {
      const text = await response.text().catch(() => "Unknown error");
      throw new Error(`ODP API error: ${response.status} - ${text}`);
    }

    return response.arrayBuffer();
  }
}
```

**Key details**:
- Auth header: `ApiKey {key}` (NOT `Bearer`)
- Tabular endpoint: `POST /api/table/v2/sdk/select?table_id={uuid}`
- Request body uses `query`, `sample`, `columns` (NOT `filter`, `limit`)
- Response is Apache Arrow IPC binary (NOT JSON)

### Server-Side: Arrow Parser

```typescript
// lib/arrow-parser.ts
import * as Arrow from "apache-arrow";

export function parseArrowIPC(buffer: ArrayBuffer): Record<string, unknown>[] {
  const table = Arrow.tableFromIPC(buffer);

  const data: Record<string, unknown>[] = [];
  for (let i = 0; i < table.numRows; i++) {
    const row: Record<string, unknown> = {};
    for (const field of table.schema.fields) {
      const col = table.getChild(field.name);
      let val = col ? col.get(i) ?? null : null;
      // Convert BigInt to Number for JSON serialization
      if (typeof val === "bigint") val = Number(val);
      // Convert NaN to null
      if (typeof val === "number" && isNaN(val)) val = null;
      row[field.name] = val;
    }
    data.push(row);
  }

  return data;
}
```

**Important**: Arrow may return `BigInt` for integer columns (breaks `JSON.stringify`) and `NaN` for missing values. Always convert both.

### Server-Side: Geometry Conversion

ODP stores geometry as WKT or WKB strings. Convert to GeoJSON:

```typescript
// lib/geometry-utils.ts
import wkx from "wkx";

export function toGeoJsonGeometry(value: unknown): GeoJSON.Geometry | null {
  if (!value) return null;

  try {
    if (value instanceof Uint8Array || value instanceof Buffer) {
      const geom = wkx.Geometry.parse(Buffer.from(value));
      return geom.toGeoJSON() as GeoJSON.Geometry;
    }

    if (typeof value === "string") {
      // Try WKT first
      if (value.startsWith("POINT") || value.startsWith("POLYGON") ||
          value.startsWith("LINE") || value.startsWith("MULTI") ||
          value.startsWith("GEOMETRY")) {
        const geom = wkx.Geometry.parse(value);
        return geom.toGeoJSON() as GeoJSON.Geometry;
      }
      // Try hex-encoded WKB
      const geom = wkx.Geometry.parse(Buffer.from(value, "hex"));
      return geom.toGeoJSON() as GeoJSON.Geometry;
    }
  } catch {
    return null;
  }

  return null;
}
```

### Server-Side: API Route

```typescript
// app/api/datasets/[id]/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;

  const odpUuid = DATASET_UUIDS[id];
  if (!odpUuid) return NextResponse.json({ error: "Unknown dataset" }, { status: 404 });

  const client = getODPClient();
  const buffer = await client.queryTabularData(odpUuid, { sample: 50000 });
  const parsed = parseArrowIPC(buffer);

  const features = parsed.data.map((row) => ({
    type: "Feature" as const,
    geometry: toGeoJsonGeometry(row.geometry),
    properties: Object.fromEntries(
      Object.entries(row).filter(([k]) => k !== "id" && k !== "geometry")
    ),
  }));

  return NextResponse.json({ type: "FeatureCollection", features });
}
```

### Client-Side: Loading Data into Mapbox

```typescript
// In the map component, inside map.on("load", async () => { ... })
const res = await fetch(`/api/datasets/${datasetId}`);
const geojson = await res.json();

map.current.addSource("observations", {
  type: "geojson",
  data: geojson,
});

const geomType = geojson.features[0]?.geometry?.type;

if (geomType === "Point" || geomType === "MultiPoint") {
  map.current.addLayer({
    id: "points",
    type: "circle",
    source: "observations",
    paint: {
      "circle-color": color,
      "circle-radius": 7,
      "circle-stroke-width": 2,
      "circle-stroke-color": "#fff",
    },
  });
} else {
  // Polygons
  map.current.addLayer({
    id: "polygons",
    type: "fill",
    source: "observations",
    paint: {
      "fill-color": color,
      "fill-opacity": 0.5,
    },
  });
  map.current.addLayer({
    id: "polygons-outline",
    type: "line",
    source: "observations",
    paint: {
      "line-color": color,
      "line-width": 1.5,
    },
  });
}
```

## Popup Styling

When using `mapboxgl.Popup` with `.setHTML()`, Mapbox applies its own default styles which can result in very low contrast text (light gray on white). **Always set explicit text colors on popup content**:

```typescript
const html = `
  <div style="font-family: system-ui, sans-serif; max-width: 260px;">
    <h3 style="margin: 0 0 4px; font-size: 15px; color: #1a1a2e;">
      ${title}
    </h3>
    <p style="margin: 0 0 8px; font-size: 12px; color: #666; font-style: italic;">
      ${subtitle}
    </p>
    <table style="font-size: 12px; border-collapse: collapse; width: 100%;">
      <tr>
        <td style="padding: 2px 8px 2px 0; color: #888;">Label</td>
        <td style="color: #333;">Value</td>
      </tr>
    </table>
  </div>
`;
```

Key rules:
- Labels (left column): `color: #888` for muted appearance
- Values (right column): `color: #333` for readable dark text
- Title: `color: #1a1a2e` for strong heading contrast
- Never rely on inherited/default text color in popups

## Caching Strategy

ODP data changes infrequently. Cache GeoJSON responses server-side:

```typescript
const cache = new Map<string, { data: unknown; expires: number }>();
const TTL = 60 * 60 * 1000; // 1 hour

function getCached<T>(key: string): T | null {
  const entry = cache.get(key);
  if (entry && Date.now() < entry.expires) return entry.data as T;
  cache.delete(key);
  return null;
}

function setCache(key: string, data: unknown) {
  cache.set(key, { data, expires: Date.now() + TTL });
}
```

**Note**: In-memory cache works for dev and serverless (within a single invocation). For Vercel serverless, consider that each cold start resets the cache. For persistent caching, use Vercel KV or write to `/tmp`.

## Stale Closures in Map Event Handlers

**Critical bug pattern**: Mapbox event handlers are registered once during map initialization and capture the closure at that point. If a callback prop changes (e.g., due to React state updates), the map handler still calls the stale version.

**Problem:**
```tsx
// BAD — click handler captures initial onStationSelect and never updates
map.current.on("click", layerId, (e) => {
  onStationSelect(e.features[0]); // stale closure!
});
```

**Solution — use a ref:**
```tsx
// Keep a stable ref that always points to the latest callback
const onStationSelectRef = useRef(onStationSelect);
onStationSelectRef.current = onStationSelect;

// In the map init effect, read from the ref
map.current.on("click", layerId, (e) => {
  onStationSelectRef.current(e.features[0]); // always fresh
});
```

This is especially important when the callback depends on changing state (e.g., a "compare mode" toggle).

## Null Safety After Async Operations

When loading data inside `useEffect` with `async/await`, always check `map.current` after the await — the component may have unmounted and the map destroyed while waiting.

```tsx
const loadData = async () => {
  const res = await fetch("/api/data");  // async gap
  const geojson = await res.json();

  if (!map.current) return;  // map may be gone!

  map.current.addSource("data", { type: "geojson", data: geojson });
};
```

**Avoid `map.current!`** — non-null assertions hide this bug. Use null checks instead.

## Security Headers on Vercel

`next.config.ts headers()` and Next.js middleware may NOT reliably set response headers on Vercel for cached/statically prerendered pages. Use `vercel.json` instead:

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
        { "key": "Permissions-Policy", "value": "camera=(), microphone=(), geolocation=()" },
        {
          "key": "Content-Security-Policy",
          "value": "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline' blob:; style-src 'self' 'unsafe-inline' https://api.mapbox.com; img-src 'self' data: blob: https://*.mapbox.com; connect-src 'self' https://*.mapbox.com https://events.mapbox.com; worker-src 'self' blob:; child-src blob:; frame-src 'none'; object-src 'none'"
        }
      ]
    }
  ]
}
```

The CSP must whitelist Mapbox domains for tiles, styles, workers, and telemetry.

## Common Pitfalls

### Next.js 16 / Turbopack issues
1. **"Missing html/body tags" false positive**: Caused by using `next/dynamic` with `ssr: false`. Use the manual `useEffect` + `import()` pattern instead (see step 3 above).
2. **`<head>` in root layout**: Do NOT add an explicit `<head>` tag in the root layout — Turbopack can misparse it and throw the "Missing html/body" error. Use `metadata` export or load CSS via component imports.
3. **Stale Turbopack cache**: When changing layout.tsx or env vars, always `rm -rf .next` and restart the dev server. Turbopack caches aggressively.

### Map is invisible / not rendering
4. **CSS not loaded**: With the manual import pattern, `import "mapbox-gl/dist/mapbox-gl.css"` in map-view.tsx works reliably. Verify with browser DevTools that mapbox styles are present.
5. **Container has no height**: Every ancestor must have explicit height. Check with browser DevTools → computed styles.
6. **Version mismatch**: If using CDN CSS, version must match `npm ls mapbox-gl` version.

### WebGL / canvas issues
7. **Canvas renders at wrong size**: Use `h-full w-full` on the map container div, NOT `absolute inset-0`. Mapbox calculates canvas size from container dimensions.
8. **Map.resize() needed**: If the container resizes after map init (e.g., panel toggle), call `map.resize()`.

### ODP data issues
9. **Auth header format**: Use `ApiKey {key}`, not `Bearer {key}`.
10. **Tabular API endpoint**: `POST /api/table/v2/sdk/select?table_id={uuid}`, NOT `/data/{uuid}`.
11. **Binary response**: ODP returns Apache Arrow IPC, not JSON. Must parse with `apache-arrow` library.
12. **Geometry column**: Contains WKT or WKB — must convert to GeoJSON before Mapbox can render it.
13. **NaN/null values**: Arrow data often contains NaN for missing values. Check with `typeof val === "number" && isNaN(val)`.
14. **BigInt values**: Arrow may return BigInt for integer columns. Convert with `Number(val)` before JSON serialization.

### Popup issues
15. **Low contrast text**: Mapbox popup default styles can make text nearly invisible. Always set explicit `color` on all text elements inside popup HTML (see Popup Styling section).

### Event handler issues
16. **Stale closure in click/hover handlers**: Map event handlers capture the closure at registration time. If callback props change later, handlers use stale values. Use a `useRef` to always read the latest callback (see "Stale Closures" section above).
17. **Null map after async**: After any `await` inside a map `useEffect`, check `if (!map.current) return` — the map may have been destroyed. Never use `map.current!`.

### Vercel deployment
18. **Serverless timeout**: Hobby plan has 10s timeout. Large ODP datasets may exceed this. Use `sample` parameter to limit rows.
19. **Bundle size**: `apache-arrow` is large. Keep it server-side only with `serverExternalPackages`.
20. **Security headers not applied**: `next.config.ts headers()` and middleware may not set headers on cached pages. Use `vercel.json` headers instead (see "Security Headers on Vercel" section).

## Debugging Checklist

When a map doesn't render, check in this order:

```typescript
// Add to map init useEffect:
useEffect(() => {
  if (!mapContainer.current) {
    console.error("[Map] Container ref is null");
    return;
  }
  const rect = mapContainer.current.getBoundingClientRect();
  console.log("[Map] Container:", rect.width, "x", rect.height);
  // Both must be > 0

  map.current.on("load", () => {
    console.log("[Map] Loaded OK");
    const canvas = mapContainer.current?.querySelector("canvas");
    if (canvas) {
      console.log("[Map] Canvas:", canvas.width, "x", canvas.height);
    }
  });

  map.current.on("error", (e) => {
    console.error("[Map] Error:", e.error?.message || e);
  });
}, []);
```

1. Container dimensions > 0? If not → fix CSS height chain
2. Map "load" event fires? If not → check token, check network for tile 401/403
3. Canvas dimensions match container (2x on retina)? If not → CSS issue
4. No errors in console? Check for WebGL context lost, token errors
