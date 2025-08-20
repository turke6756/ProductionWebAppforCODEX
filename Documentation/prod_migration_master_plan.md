# Cover Crop Explorer — Production Migration Plan (Agreed Scope)

**Author:** ChatGPT · **Last Updated:** 2025-08-19 19:09 · **Status:** Step 1 completed  
**Current Demo:** `deck_gl_polygon_dashboard_time_agnostic_fixed_v6.html`  
**Target:** Production web app using **vector tiles (PMTiles)** + **per‑season JSON attributes** + **manifest‑driven config**

---

## 0) Scope & Principles (Agreed)

This document captures only the migration ideas from the original *Web App Production Upgrade Analysis* that we **agree** to implement, plus the exact changes completed in **Step 1**. Guiding principles:

- **Separate geometry from attributes.** Geometry ships as **PMTiles**; attributes ship as **small JSONs**.
- **Manifest-driven behavior.** Thresholds, years, versions come from a single `manifest.json`.
- **Lazy-load + cache.** Load only the active season, prefetch neighbors, keep a tiny LRU cache.
- **Time-agnostic stats are precomputed.** Use a compact `streaks.json`.
- **Feature flags & rollback.** Keep the CSV path as a fallback until production is battle-tested.
- **Parity first.** No UX regressions; ship with tests and monitoring.
- **CDN-friendly.** Static assets, content hashing, long-lived cache headers.

---

## 1) Target Architecture (Final Workflow We’ll Build)

**Data plane**  
- **Geometry**: `orchards.pmtiles` (vector tiles), served via CDN, immutable cache.  
- **Attributes (time-varying)**: `attrs/<YEAR>.json` (e.g., `attrs/2019.json`) — object keyed by `orch_id`.  
- **Attributes (time-agnostic)**: `streaks.json` — bitmasks/flags for quick filters and chips.  
- **Manifest**: `manifest.json` — authoritative list of `years`, `ui_uncertain_threshold`, optional `version`/hash map.

**App plane**  
- **Hybrid loader**: configurable `dataMode: 'csv' | 'json'` feature-flag.  
- **Join key**: `orch_id` as **string** end-to-end (tiles & attrs).  
- **Map**: deck.gl `MVTLayer` (prod) or `PolygonLayer` (demo).  
- **Season switching**: lazy fetch of `YEAR.json`, prefetch ±1, LRU cache of 3–5 seasons.  
- **UI logic**: uncertainty threshold pulled from manifest; same threshold drives map & charts.

**CDN/ops**  
- **Immutable URLs** with content hashes (or `manifest.version` + `?v=`).  
- **S3/CloudFront (or equivalent)** with `Cache-Control: public, max-age=31536000, immutable` for hashed assets.  
- **Observability**: capture fetch errors, timing, and cache hit ratio.

---

## 2) Migration Timeline

### ✅ **Step 1 — Completed: Manifest-driven threshold + Season JSON ingestion (no UX change)**
> Implemented in `deck_gl_polygon_dashboard_time_agnostic_fixed_v7.html`

- **What changed**
  - Replaced hardcoded `0.15` with **`UNCERTAIN_THRESHOLD` read from `manifest.json`**.
  - Added **JSON data mode** that builds CSV-like rows from per‑season JSON files (keeps UI unchanged).
  - Legend now shows **dynamic threshold** (auto-updates from manifest).
  - Season label handling made robust for `YYYY` and `YYYY_YYYY` formats.
  - Boot logic selects **JSON or CSV** based on `CONFIG.dataMode`.

- **Why**
  - Make the **manifest the single source of truth** (prevents drift).  
  - De‑risk the move away from a monolithic CSV by **adopting season JSONs** with zero UI refactor.

- **How to use**
  1. Place the updated HTML next to:
     - `manifest.json` (must include `years` and `ui_uncertain_threshold`)
     - `attrs/<YEAR>.json` files for those years
     - `FresnoAlmondOrchards.geojson` (geometry for the demo; PMTiles comes later)
  2. In the HTML `CONFIG`, set:
     ```js
     dataMode: 'json',          // or 'csv' for fallback
     manifestUrl: './manifest.json',
     attrsBaseUrl: './',         // or './attrs/' if you move the files
     streaksUrl: './streaks.json'
     ```
  3. Open the HTML. The app loads the manifest, applies the threshold, loads all listed years, and renders like the CSV version.

- **Files the app looks for (Step 1)**
  - **Required (json mode):**
    - `manifest.json` — e.g.
      ```json
      {
        "schema_version": 1,
        "years": [2018,2019,2020,2021,2022,2023],
        "ui_uncertain_threshold": 0.15,
        "bit_order": "earliest=bit0"
      }
      ```
    - `<YEAR>.json` — an object keyed by orchard ID strings. Example shape:
      ```json
      {
        "00560000000000002bd1": { "cluster_role_mapped": "Cover", "best_soft_norm": 0.52, "stratum": "Young", ... },
        "00560000000000002c10": { "cluster_role_mapped": "Bare",  "best_soft_norm": 0.11, "stratum": "Old",   ... }
      }
      ```
  - **Optional (used later):** `streaks.json` — precomputed time-agnostic stats
  - **Geometry (demo):** `FresnoAlmondOrchards.geojson`
  - **Fallback (csv mode):** `viz_table.csv`

- **Notes**
  - Current per‑season JSON gzip sizes are ~**100 KB** each at your current density. This becomes the **realistic budget** (update any “<50 KB” target).
  - Keep **IDs as strings** across the pipeline. Leading zeros matter.

---

### Step 2 — Vector Tiles (PMTiles) & ID integrity
**Goal:** Decouple geometry, enable level-of-detail and large AOIs.

- Export orchard polygons → **PMTiles** (`orchards.pmtiles`).  
- Add PMTiles protocol and replace `PolygonLayer` with **`MVTLayer`**.  
- Validate that **tile features carry `orch_id` as a string** (tippecanoe: preserve as string).  
- **Join** attrs ↔ tiles by `String(f.properties.orch_id)` and the season’s JSON map.

**Success criteria**
- Visual parity with demo (colors, opacity/uncertain logic).  
- Hover/selection works by `orch_id` from tiles.  
- First render stays <3s on broadband, <5s on mobile.

---

### Step 3 — Lazy Loading, Prefetch, LRU Cache (attrs only)
**Goal:** Make season switching instant without loading everything upfront.

- Load only **active season**; **prefetch ±1** seasons after render.  
- **LRU cache** for 3–5 seasons (evict oldest).  
- Simple in-memory metrics: cache hits, season switch latency.

**Success criteria**
- Season switch p50 < 100 ms (from cache); p95 < 250 ms (from network).  
- Memory < 50% of the CSV demo’s footprint.

---

### Step 4 — Time-Agnostic Mode from `streaks.json`
**Goal:** Replace client-side recalculation with precomputed analytics.

- Plug in `streaks.json` (bitmask order from `manifest.bit_order`).  
- Use flags for “ever cover”, “longest streak”, “2+/3+ consecutive”, etc.  
- Keep the **manifest threshold** governing gray zone behavior everywhere.

**Success criteria**
- Identical results vs. demo’s client calculations.  
- No statistical drift after switching to precomputed values.

---

### Step 5 — Packaging, CDN, and Cache Busting
**Goal:** Make the app fast globally and safe to update.

- **Content hashing** on `pmtiles`, season JSONs, and `streaks.json` (or bump `manifest.version`).  
- Serve with **immutable** cache headers; reference via **`?v=manifest.version`**.  
- Push static bundle to CDN.

**Success criteria**
- Cache hit ratio > 85% for repeat sessions.  
- No stale asset issues after deploys.

---

### Step 6 — Telemetry, QA, and Parity Gates
**Goal:** Ship safely and prove improvements.

- **Telemetry:** log fetch errors, durations, cache hits, and tile load stats.  
- **Parity tests:**  
  - **ID type parity** (all joins happen on string IDs).  
  - **Threshold parity** (manifest threshold used in every code path).  
  - **Visual parity** screenshots across representative views.  
- **Performance budgets** enforced in CI (simple Lighthouse + custom probes).

**Success criteria**
- Zero critical regressions, <1% error rate in production.  
- Performance improvement is measurable and reported.

---

### Step 7 — Decommission CSV Path
**Goal:** Remove the demo-only code once production path is proven and monitored.

- Remove CSV code paths and fallback assets.  
- Lock the manifest as the only config surface for thresholds/years.

---

## 3) Step 1 — Implementation Details (Recorded)

**Files updated**
- `deck_gl_polygon_dashboard_time_agnostic_fixed_v7.html` (new) — replaces v6 for testing.

**Key code changes**
- **Threshold**
  - `const UNCERTAIN_THRESHOLD = 0.15;` → `let UNCERTAIN_THRESHOLD = 0.15;` (fallback)
  - New `loadManifest()` sets `UNCERTAIN_THRESHOLD = manifest.ui_uncertain_threshold`.
  - All comparisons `<= 0.15` / `> 0.15` → `<= UNCERTAIN_THRESHOLD` / `> UNCERTAIN_THRESHOLD`.
  - Legend string → dynamic `<span id="uncertainLegendValue">…</span>` updated after manifest loads.

- **JSON ingestion (no UI refactor)**
  - `buildRowsFromSeasons(manifest, attrsBaseUrl)` constructs **CSV-like rows** from `<YEAR>.json` objects to feed the existing `preprocessCSV` and UI.
  - Join key normalized to `String(orch_id)` everywhere.

- **Labels**
  - `updateSeasonLabels()` now handles both `YYYY` and `YYYY_YYYY` season labels safely.

- **Boot**
  - If `CONFIG.dataMode === 'json'`, load manifest → build rows from season JSONs. Otherwise, fall back to CSV.

**How to validate**
- Load the v7 HTML with your current `manifest.json` + `2018.json..2023.json`.  
- Visually compare to the CSV demo on the same scenes; expect identical results.  
- Toggle `dataMode: 'csv'` to confirm fallback behavior still works.

---

## 4) File & Directory Layout (Proposed)

```
/app
  /assets
    /tiles/orchards.pmtiles
    /attrs/2018.json
    /attrs/2019.json
    ...
    /attrs/2023.json
    /streaks.json
    /manifest.json
  index.html (v7+)
```

> In Step 1 we accept flat paths (e.g., `./2019.json`) to match your current sample files. We’ll migrate to `/assets/attrs/` as soon as we wire the CDN paths.

---

## 5) Performance & Budgets (Updated to reality)

- **Per-season JSON (gz):** target **≤ 100 KB** (current ~94 KB for 2019).  
- **Initial load:** ≤ 3 s broadband, ≤ 5 s mobile.  
- **Season switch:** p50 ≤ 100 ms (cache), p95 ≤ 250 ms.  
- **Memory:** ≤ 50% of CSV demo baseline.  
- **Bundle size:** ≤ 60% of current demo.

---

## 6) Risk Controls & Rollback

- **Feature flag** (`dataMode`) to flip between CSV and JSON.  
- **Strict ID handling:** treat all `orch_id` values as strings (tiles, JSON, UI).  
- **Graceful errors:** if a season JSON fails, show toast + keep UI responsive; allow fallback to CSV.  
- **Rollback plan:** switch `dataMode` to `'csv'` and redeploy; no schema changes needed.

---

## 7) Checklist

- [x] Step 1: Manifest-driven threshold + JSON ingestion (this commit)
- [ ] Step 2: PMTiles + MVT join by `orch_id` (string)
- [ ] Step 3: Lazy load + prefetch ±1 + LRU cache
- [ ] Step 4: Wire `streaks.json` & replace client-side time-agnostic calculations
- [ ] Step 5: CDN packaging, hashing, cache-busting
- [ ] Step 6: Telemetry + QA gates (ID parity, threshold parity, visual parity)
- [ ] Step 7: Remove CSV path

---

**Changelog**  
- **2025-08-19:** Step 1 implemented; v7 HTML created; manifest threshold + JSON loader live.
