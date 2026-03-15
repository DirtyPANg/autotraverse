# AutoTraverse

**Single-file browser application for total-station survey data visualisation, traverse adjustment, and analysis.**

No backend, no installation — just open the HTML file in any modern browser.

---

## Features

- **Multi-format file import** — Leica GSI, `.tra` traverse, `.csv` fixpoint, `.tht` height traverse
- **Automatic traverse detection** — Identifies closed traverses from observation data and fixed points
- **Bowditch adjustment** — 7-step horizontal traverse adjustment with angular & coordinate closure checks
- **Height traverse** — Trigonometric height adjustment with misclosure distribution
- **Interactive 2D map** — Canvas-based point/line rendering with pan, zoom, search, and point selection
- **3D visualisation** — Three.js scene with orbit controls, labelled stations, and traverse lines
- **Leaflet basemap** — Optional tile layer (OSM / satellite) with coordinate projection (proj4)
- **Drawing tools** — Point, polyline, arc, section, extend, offset, mirror, delete, undo/redo
- **Station setup editor** — Manual coordinate, instrument height, backsight, and bearing entry
- **Alias management** — Merge duplicate point IDs across stations
- **Export** — Map snapshot (PNG), coordinate CSV, DXF, traverse reports (HTML / text / clipboard)
- **Offline-ready** — All state persisted to `localStorage`; works without network after first load of CDN libs
- **AI-agent friendly** — Public JS API on `window.SurveyApp` + embedded SKILL header for LLM tool use

---

## Quick Start

1. **Open the file**

   Double-click `autotraverse.html` or serve it via any static HTTP server:

   ```bash
   # Python
   python -m http.server 8080
   # Node
   npx serve .
   ```

2. **Import survey data**

   Click **Choose Files** in the controls panel and select one or more files:

   | File type | Extension | Content |
   |-----------|-----------|---------|
   | Leica GSI | `.gsi` | Raw total-station observations |
   | Traverse | `.tra` | Manual traverse definition (control points + observations) |
   | Fixpoints | `.csv` | Known coordinates (`Name, N, E, Z`) |
   | Height traverse | `.tht` | Height traverse definition (control RLs + observations) |

   Multiple files can be imported at once — the app automatically detects each type.

3. **View results**

   - The **2D map** renders all observations immediately after import.
   - Open the **Analysis panel** (📊 button) to see auto-detected traverses, adjustment reports, and statistics.
   - Toggle **3D view** from the bottom toolbar to explore the scene in three dimensions.

4. **Manual traverse**

   Click **+ Horizontal** or **+ Height** in the analysis panel to create a manual traverse. Fill in control coordinates, observation rows, then click **Calc**.

5. **Export**

   Use the **Export ▾** dropdown for map snapshots (PNG) or data exports (CSV / DXF). Each traverse report card also has Copy / Download / CSV buttons.

---

## Supported Instruments & Software

Currently the built-in parsers target two specific sources:

| Source | Format | Description |
|--------|--------|-------------|
| **Leica TS-02 / TS-06 series** | `.gsi` (GSI-16) | Raw observation export from the instrument |
| **GENLAY** (Hong Kong land computation software) | `.tra` / `.csv` | Traverse and fixpoint files exported by GENLAY |

> **Extensible by design** — All file parsing is routed through `ParserRegistry` (strategy pattern).
> You can register your own parser for any instrument or software by calling
> `ParserRegistry.register({ name, type, detect, parse })`.
> See the AI Skill reference below for details on the registry API.

---

## File Format Reference

### `.tra` — Traverse Definition

```
CONTROLS
CP1, 1000.000, 2000.000, 50.000
CP2, 1050.000, 2080.000, 52.000

OBSERVATIONS
CP1, T1, 45.3020, 25.000, START_RO
T1, T2, 90.0000, 30.500, TRAVERSE
T2, CP2, 180.0000, 20.000, CLOSE_RO
```

- **CONTROLS** — `Name, Northing, Easting, Elevation` (metres)
- **OBSERVATIONS** — `From, To, Angle(DDD.MMSS), Distance(m), Type`
- Types: `START_RO`, `TRAVERSE`, `CLOSE_RO`

### `.tht` — Height Traverse Definition

```
CONTROLS
BM1, 100.000
BM2, 102.350

OBSERVATIONS
BM1, T1, 1.500, 88.3025, 1.600, 45.000
T1, T2, 1.500, 87.1540, 1.600, 60.200
T2, BM2, 1.500, 89.4510, 1.600, 38.750
```

- **CONTROLS** — `Name, RL` (reduced level, metres)
- **OBSERVATIONS** — `From, To, IH, VA(DDD.MMSS), TH, SD` (instrument height, vertical angle, target height, slope distance)

### `.csv` — Fixpoint Coordinates

```
Name, Northing, Easting, Elevation
CP1, 1000.000, 2000.000, 50.000
CP2, 1050.000, 2080.000, 52.000
```

---

## AI Skill & Public API Reference

> The following reference is also embedded inside the HTML file as `AI-SKILL-HEADER`
> and can be downloaded via the UI (📖 button) or `downloadSkillDoc()` in the console.

### Public JavaScript API

All APIs are exposed on `window` and can be called from DevTools, Puppeteer, Playwright, or any JS automation framework.

#### `window.SurveyApp.calculateManualTraverse(jsonData)`

**External-only computation endpoint.** Calculate a traverse from JSON input.
> **Note:** Internal code no longer calls this function — it uses `adjustTraverseFromData(data)` directly
> to avoid side-effects (overwriting fixpointInput and replacing all observations).
> This API is kept for external / AI automation use only.

| Parameter | Type | Description |
|-----------|------|-------------|
| `jsonData.controls` | `Object` | Known fixed-point coordinates keyed by point name. `{ "PT1": {n, e, z}, ... }` — n/e in **metres**, z optional. |
| `jsonData.observations` | `Array` | Array of observation objects (see below). |

**Observation object fields:**

| Field | Type | Description |
|-------|------|-------------|
| `from` | `string` | Station identifier (from). |
| `to` | `string` | Station identifier (to). |
| `angle` | `string` | Horizontal angle in **DDDMMSS** format, e.g. `"2984654"` = 298°46′54″. |
| `dist` | `string \| number` | Slope / horizontal distance in **metres**. |
| `type` | `string` | Optional: `"TRAVERSE"` (default), `"START_RO"`, or `"CLOSE_RO"`. |

**Returns:** `string` — e.g. `"Calculated 5 observations."` or `"No valid observations."`

**Side-effects:** Overwrites `fixpointInput` with controls from jsonData, then triggers full parse → compute → render pipeline. Use only for standalone/external calls.

#### `window.SurveyApp.updateObservation(uniqueId, changes)`

Patch fields of a single observation and re-render.

| Parameter | Type | Description |
|-----------|------|-------------|
| `uniqueId` | `string` | The observation's unique identifier. |
| `changes` | `Object` | Key-value pairs, e.g. `{ slopeDistance: "12.345", prismConstant: "0" }`. |

**Returns:** `string` — confirmation or `"No changes applied."`

#### `window.SurveyApp.addMapSection(p1, p2, offset)`

Draw a rectangular cross-section strip on the 2D map.

| Parameter | Type | Description |
|-----------|------|-------------|
| `p1`, `p2` | `{x, y, z}` | Point objects in **map-millimetre** coordinates. |
| `offset` | `number` | Half-width of the section strip in **metres**. |

#### `window.SurveyApp.getSkillDoc()`

Returns the full Markdown documentation string (this document).

#### `window.analysisOptions`

Global thresholds for FS/BS pair analysis.

```js
window.analysisOptions = {
  hdThreshold: 0.015,      // metres
  bearingThreshold: 300     // DDDMMSS units
};
```

Set before rendering to customise tolerance, e.g.: `window.analysisOptions.hdThreshold = 0.020;`

#### `window.set3DView(type)`

Snap the 3D camera to a preset viewpoint.

| `type` | View |
|--------|------|
| `"top"` | Bird's-eye (North up) |
| `"front"` | South → North |
| `"back"` | North → South |
| `"left"` | West → East |
| `"right"` | East → West |

#### `window.toggleStationVisibility(stationId, visible)`

Show or hide all observations belonging to a station on the map.

| Parameter | Type |
|-----------|------|
| `stationId` | `string` |
| `visible` | `boolean` |

#### `window.exportStationData(stationId)`

Download a JSON file (`{stationId}_points.json`) containing all observations for one station.

#### `window.openStationSetupModal(stationId)`

Open the station-setup editor modal. Allows manual entry of coordinates, IH, backsight target, and bearing.

#### `window.applyUserStationSetups()`

Apply all manually entered station setups to recalculate orientations and coordinates.
Modifies `allObservations` in-place, then calls `recalculateAndRender()`.

#### `window.stationCoordsCache` _(read-only)_

`Map<stationId, {x, y}>` of the most recent station coordinate positions (in mm).
Updated automatically after each render cycle.

#### `downloadSkillDoc()`

Download this API documentation as a Markdown file (`SurveyApp_SKILL.md`).
Callable via the 📖 button in the controls panel or via `downloadSkillDoc()` in the console.

---

### Automation Example (Puppeteer)

```js
const puppeteer = require("puppeteer");

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.goto("http://localhost:8080/reconstruction.html");

  const result = await page.evaluate(() => {
    return window.SurveyApp.calculateManualTraverse({
      controls: {
        "CP1": { n: 100, e: 200, z: 50 },
        "CP2": { n: 150, e: 250, z: 52 }
      },
      observations: [
        { from: "CP1", to: "T1", angle: "0000000", dist: "25.000", type: "START_RO" },
        { from: "T1",  to: "T2", angle: "900000",  dist: "30.500", type: "TRAVERSE" },
        { from: "T2",  to: "CP2", angle: "1800000", dist: "20.000", type: "CLOSE_RO" }
      ]
    });
  });

  console.log(result); // "Calculated 3 observations."
  await browser.close();
})();
```

---

### Conventions

| Rule | Detail |
|------|--------|
| **Units** | UI inputs are in metres; internal map math uses millimetres. |
| **Angles** | Raw survey angles use DDDMMSS digits — e.g. `2984654` = 298°46′54″. |
| **State** | Most runtime state lives inside the `DOMContentLoaded` closure. |
| **Removed systems** | Firebase, chat AI, cloud sync, and Gemini backend hooks were intentionally removed. |

---

### Key Internal Modules (for AI agents)

| Anchor | Purpose |
|--------|---------|
| `const SurveyMath` | Angle conversion & formatting (parseAngleParts, DDDMMSS). |
| `function parseGsiData` | Leica GSI parser → observation objects. |
| `const ParserRegistry` | Strategy-pattern file type detection & parser dispatch (leica-gsi, traverse-tra, fixpoint-csv, height-tht). |
| `const CalcEngineRegistry` | Strategy-pattern calculation engine dispatch — registered engines: `horizontal` (Bowditch), `height` (Trig Height). |
| `const TraverseManager` | Unified traverse management — auto/manual entries, expandable card UI, localStorage persistence. |
| `const DataFusion` | Station alias resolution, FS/BS pair fusion, coordinate inference, foresight identification. |
| `const TraverseEngine` | Traverse path discovery, bearing propagation, closure & Bowditch adjustment. |
| `const MapEngine` | Coordinate normalisation, viewport calc, base-map sync. |
| `function adjustTraverseFromData` | 🔥 Unified Bowditch 7-step engine (manual & auto traverses share this). |
| `function parseThtFile` | .tht height traverse file parser → height traverse data object. |
| `function adjustHeightTraverseFromData` | Height traverse calculation engine (trigonometric height adjustment). |
| `function generateHeightTraverseReportHTML` | HTML report generator for height traverse adjustment cards. |
| `function handleRenderRequest` | Top-level parse → compute → render entry point. |
| `function recalculateAndRender` | Re-derive results after edits / alias / setup changes; auto-identifies traverses. |
| `function updateDataViews` | Redraw tables, map, 3D, and statistics. |
| `function generateTraverseReportHTML` | HTML report generator for traverse adjustment cards. |
| `function init3D / update3DScene / animate3D` | Three.js scene lifecycle. |

#### Computation Pipeline (File Import → Traverse Identification)

```
File Import (fileInput change handler)
  ↓
Detect & accumulate: observations → dataInput.value, fixpoints → fixpointInput.value,
                     .tra → TraverseManager.add() (manual entries)
  ↓
┌─ If observations or fixpoints imported:
│    handleRenderRequest()
│      → parseFixpoints(fixpointInput.value)          — fixpointData (global Map)
│      → parse observations via ParserRegistry
│      → recalculateAndRender(true)
│          → parseFixpoints() again (re-parse, safe)
│          → identifyForesights(allObservations, fixpointData)
│          → analyzeClosedTraverses(foresightDataset, fixedStations, stationCoords)
│          → for each auto traverse: performTraverseAdjustment(traverse, fixpointData, stationCoords)
│          → TraverseManager.addAutoTraverses(adjustments)
│          → updateDataViews()
├─ If only .tra files imported (existing data present):
│    recalculateAndRender()  — re-runs full pipeline above with existing data
├─ .tra manual entries calculated via adjustTraverseFromData() directly
│    (NOT via calculateManualTraverse — that would overwrite fixpointInput)
└─ .tht height traverse entries:
     parseThtFile() → TraverseManager.add({calcType:'height'})
     → CalcEngineRegistry.get('height').calculate(data) → adjustHeightTraverseFromData()
     → generateHeightTraverseReportHTML() for report card
```

#### localStorage Keys

| Key | Content |
|-----|---------|
| `survey_app_full_state_v1` | Observation data, fixpoints, aliases, offsets, edits, geometries |
| `survey_tool_aliases` | User-defined point aliases |
| `survey_traverse_manager` | Manual traverse entries (saved on render, restored on load) |

---


### File Navigation Index (Tag-Pair Tree)

### How to read this index
- Each entry = one matched HTML tag pair, written as `start: <opening...>  end: </closing>`.
- Tag balance rule: when the same close-tag (e.g. `</div>`) appears many times,
  count every `<div` open and `</div>` close from the start tag forward; the first
  close that brings the counter back to zero is the matching end.
- Indentation = parent → child nesting in the DOM / code.
- For JS items inside `<script>`: use `anchor:` with function / const / class name.
- Location path format: `<parent> → <child>` indicates nesting hierarchy; no line numbers used (they change with edits).
  Use anchor names (e.g. `const DataFusion`) to locate items in the file.
- Descriptions follow `—`.
- Maintenance rule: when structure changes, update this tree in the same edit.
- State rule: most runtime state lives inside the DOMContentLoaded closure.
- Units rule: UI inputs in metres; internal map math uses millimetres.
- Angle rule: raw survey angles = DDDMMSS digits, e.g. 2984654 → 298°46′54″.

### Tag-pair tree

start: <html lang="en">                              end: </html>              — Root document
  start: <head>                                          end: </head>              — Document head
    (AI-SKILL-HEADER comment block)                                                   — This document (API reference + navigation index)
    start: <script src="...three.min.js">                end: </script>            — Three.js core
    start: <script src="...OrbitControls.js">            end: </script>            — 3D orbit controls
    start: <script src="...CSS2DRenderer.js">            end: </script>            — 3D label renderer
    start: <link rel="stylesheet" href="...leaflet.css"> (self-closing)           — Leaflet CSS
    start: <script src="...leaflet.js">                  end: </script>            — Leaflet basemap engine
    start: <script src="...proj4.js">                    end: </script>            — Coordinate projection library
    start: <style> :root {                               end: </style>             — Global CSS (theme vars, layout, modals, toolbar, tables, canvas overlay)
  start: <body>                                          end: </body>              — Document body
    start: <div id="app-container">                      end: </div>               — App root container
      start: <div id="top-header">                       end: </div>               — Title bar + controls collapse button
      start: <div id="controls">                         end: </div>               — Controls panel (file import, render, export, log, cache reset, text input)
        start: <div id="export-dropdown">                end: </div>               —   Export dropdown (map snapshot / data CAD)
      start: <div id="main-content-area">                end: </div>               — Main content area layout container
        start: <div id="data-table-container">           end: </div>               —   Analysis panel (stats, traverse reports, measurement tables)
          start: <div id="survey-data-output">           end: </div>               —     Dynamic HTML output target
        start: <div id="bottom-toolbar">                 end: </div>               —   Bottom toolbar (basemap, FS, 3D toggles)
        start: <div id="bottom-more-menu">               end: </div>               —   Overflow settings menu (search, exclude, elevation filter, label sliders)
        start: <div id="canvas-container">               end: </div>               —   Canvas container
          start: <div id="basemap-container">            end: </div>               —     Leaflet basemap host (hidden by default)
          start: <canvas id="main-canvas">               end: </canvas>            —     2D main canvas
          start: <div id="three-canvas-container">       end: </div>               —     3D viewport host (hidden by default)
          start: <div id="edit-toolbar">                 end: </div>               —     Drawing edit toolbar (select/point/line/arc/section/extend/offset/mirror/delete/undo/redo)
      start: <div id="modal-container">                  end: </div>               — Generic modal (point detail & measurement popup)
      start: <div id="log-viewer-modal">                 end: </div>               — Log viewer modal
      start: <div id="alias-manager-modal">              end: </div>               — Alias manager modal
      start: <div id="setup-station-modal">              end: </div>               — Station setup editor modal
      start: <div id="export-map-modal">                 end: </div>               — Map snapshot export modal
      start: <div id="export-data-modal">                end: </div>               — CSV/DXF data export modal
      start: <div id="manual-input-modal">               end: </div>               — Manual traverse input modal (type selector + horizontal/height tables + observations)
      start: <div id="elevation-modal">                  end: </div>               — Elevation view modal

    start: <script>                                      end: </script>            — Main application script
      ├─ class Rectangle                                                           — Quadtree helper geometry class
      ├─ class Quadtree                                                            — Spatial index (proximity queries)
      └─ DOMContentLoaded closure                                                  — App initialization closure (nearly all runtime logic)
          ├─ function init3D                                                       —   Three.js scene / camera / renderer / controls creation
          ├─ function update3DScene                                                —   3D points, labels, traverse lines, FS lines update
          ├─ function animate3D                                                    —   requestAnimationFrame render loop
          ├─ function addTraEntry                                                  —   .tra textarea creation & compute/delete events
          ├─ saveAppState / loadAppState                                           —   LocalStorage persistence (incl. survey_traverse_manager)
          ├─ logMessages[]                                                         —   Log capture system (console wrapper)
          ├─ const TraverseManager { ... }                                         —   Traverse manager (manual/auto entries, expandable card UI, localStorage)
          │    ├─ .add / .addAutoTraverses / .remove / .get / .getAll             —     CRUD operations
          │    ├─ .confirmAutoToManual                                             —     Auto-to-manual conversion (removes original auto entry)
          │    ├─ ._saveState / ._loadState                                       —     localStorage read/write
          │    ├─ .render → renderTraverseList()                                  —     Render expandable card list
          │    ├─ ._toggle / ._calcManual / ._confirmAuto                         —     UI action methods
          │    └─ ._openEdit → populateManualModal()                              —     Open manual edit modal
          ├─ function populateManualModal                                          —   Populate manual traverse modal form
          ├─ function generateTraverseReportHTML                                   —   HTML report generator (traverse adjustment card content)
          ├─ function bindTraverseManagerButtons                                   —   Report card button event binding (copy/download/CSV/coord export)
          ├─ function renderTraverseList                                           —   Render traverse manager list UI (expandable cards)
          ├─ const ParserRegistry { ... }                                          —   Parser registry (strategy pattern)
          │    ├─ .register / .detect / .getAll / .getByType / .getByName         —     Register & query
          │    └─ Registered: leica-gsi (observation), traverse-tra, fixpoint-csv, height-tht     —     Four file type parsers
          ├─ function parseThtFile                                                 —   .tht height traverse file parser
          ├─ function parseFixpoints                                               —   Fixpoint CSV parser -> Map
          ├─ function parseTraFile                                                 —   .tra traverse file parser (incl. bearingToAngleStr DDD.MMSS)
          ├─ function parseCSVTemplate / generateCSVTemplate                       —   Manual traverse CSV template parse & export
          ├─ const SurveyMath { ... }                                              —   Angle conversion & formatting toolkit
          ├─ function parseGsiData                                                 —   Leica GSI data parser
          ├─ const DataFusion { ... }                                              —   Station FS data fusion & alias resolution & coordinate inference
          │    ├─ .buildPointAliases                                               —     Point alias building
          │    └─ .identifyForesights                                              —     Foresight identification (known stations via fixpoints)
          ├─ const TraverseEngine { ... }                                          —   Traverse path finding & bearing propagation & closure
          │    ├─ .analyzeClosedTraverses                                          —     Auto closed-traverse search
          │    └─ .performTraverseAdjustment (thin wrapper)                              —     Delegates to adjustTraverseFromData
          ├─ function generateSingleTraverseFullReport                             —   Single traverse full text report
          ├─ function buildTraverseObsData                                         —   Auto traverse format -> unified data (fills controls from fixpoints)
          ├─ function adjustTraverseFromData                                       —   🔥 Unified adjustment engine (manual/auto Bowditch steps 1-7)
          ├─ const CalcEngineRegistry { ... }                                      —   Calculation engine registry (horizontal Bowditch, height trig)
          │    ├─ .register / .get / .getAll                                       —     Register & query engines
          │    └─ Registered: horizontal (adjustTraverseFromData), height (adjustHeightTraverseFromData)
          ├─ function adjustHeightTraverseFromData                                 —   Height traverse trig height adjustment engine
          ├─ function generateHeightTraverseReportHTML                             —   HTML report for height traverse cards
          ├─ function handleRenderRequest                                          —   Top-level entry: parse -> compute -> render
          ├─ function recalculateAndRender                                         —   Recalculate after edit/alias/setup changes + auto traverse identification
          ├─ function updateDataViews                                              —   Update tables, map, 3D scene, statistics
          ├─ const MapEngine { ... }                                               —   Map coordinate normalization & viewport calc & basemap sync
          ├─ const BaseMapManager { ... }                                          —   Leaflet tile layer management
          ├─ // Drawing functions (refactored)                                                  —   2D Canvas render pipeline (points, labels, selection, user geometry, overlays)
          ├─ // Manual input logic                                                       —   Manual traverse form row management & CSV/.tra import & TraverseManager integration
          ├─ window.SurveyApp = { ... }                                            —   Public API object (calculateManualTraverse - external use only)
          ├─ // Station setup logic                                                       —   Station setup & bearing override
          ├─ // Drawing edit tool logic                                                   —   Edit tool state machine (handleEditToolClick etc.)
          ├─ // Event listeners                                                         —   UI event binding (touch, keyboard, buttons etc.)
          ├─ // Mouse interaction support                                                       —   2D Map mouse events
          └─ // Data export logic                                                       —   CSV / DXF export implementation

### How to use this index
- To edit CSS: search `<style>` in `<head>`.
- To edit a modal: search `<div id="XXX-modal">` in the tree, then search that exact text in the file.
- To edit JS logic: find the anchor name in the tree (e.g. `const DataFusion`), then search that exact text in the file.
- Parent→child: if you need to find which DOM container holds `#survey-data-output`, read up the tree → it's inside `#data-table-container` → `#main-content-area` → `#app-container`.
- Tag balance example: `<div id="controls">` has many nested `</div>` inside; to find its closing `</div>`, count `<div` opens vs `</div>` closes forward until balance = 0.
- Sync requirement: if you edit any tag/anchor named above, update this tree in the same edit.

*Generated from autotraverse.html AI-SKILL-HEADER.*

---

## Browser Compatibility

| Browser | Version |
|---------|---------|
| Chrome / Edge | 90+ |
| Firefox | 90+ |
| Safari | 15+ |

Requires ES2020+ features (optional chaining, nullish coalescing). No transpilation needed for modern browsers.

---

## Tech Stack

- **Three.js** — 3D visualisation
- **Leaflet** — Tile basemap
- **proj4js** — Coordinate projection (WGS84 ↔ local grid)
- **Canvas 2D** — Main drawing surface
- **localStorage** — State persistence

All dependencies are loaded from CDN. No build step required.

---

## License

MIT License

Copyright (c) 2025 ggen652

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.