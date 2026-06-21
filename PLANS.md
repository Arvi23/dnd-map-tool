# D&D Map Editor — Agent Handoff Document

> **Read this first.** This document exists so that a future LLM agent can continue development without needing the full conversation history. It covers what the project is, how it is built, what exists, what is planned, and every gotcha we have hit so far.

---

## ✅ Pin/Polygon refinements (DONE — latest session)
Four features, all editor + export, data fields added to `newRegion`/`migrateRegions`/export passthrough:
- **labelScale** (50–400%): "Label Text Size" slider — pins & polygons. Multiplies name-label font in `appendLabel`/`appendPlayerLabel`.
- **showIcon** (bool): "Show type icon inside" toggle — polygons only. Renders the tag's PIN_ICON above the centroid label in editor + export.
- **hideBorder** (bool): "Hide pin border" toggle — pins only. Ring becomes invisible (transparent fill, no stroke) but keeps `pointer-events:all` hit area + hover feedback.
- **labelDx/labelDy** (normalized): drag a selected pin/polygon's name label to reposition; "↺ Reset label position" button appears when offset ≠ 0. Drag handled via global `labelDrag` state in mousemove/mouseup/Esc.
Note: file is CRLF — multi-line str_replace must use EOL-aware run_script (E() helper). Export JSON stringifies fields as `"labelScale":` etc.

## ✅ Rumor system overhaul (DONE — latest session)
Rumor model is now `{id, text (headline), details (expandable, player-facing), isTrue (DM), dmNote (DM)}`. New `state.worldRumours` array for rumours not tied to any location.
- **A (data):** `migrateRegions` adds `state.worldRumours`, normalizes every rumour (id + details). Export passthrough now includes `id`+`details` per region rumour and a top-level `worldRumours` array.
- **B (editor 💬 Rumor Tracker):** toolbar button (next to 🌍) → `#rumor-modal`. Grouped by location (World group first), "Add to" location dropdown (incl. World) + Add button, "Show" location filter. Each row: truth toggle + headline input + expand chevron → Details + DM-note textareas. `renderRumorTracker()`, `buildRumorRow()`, `rumorBucket()`, `rumorLocations()`, `fillLocOptions()`. Closing re-renders the side-panel rumour list.
- **C (player export 💬 Rumors):** `#pr-overlay` toolbar button. Grouped by location (only EXPLORED locations + World). Each row: headline + Following/Confirmed/Debunked buttons (saved to `localStorage` key `dndmap_rumor_status:<world>`, per-player) + expandable details. Location filter dropdown. Truth + dmNote never exported. `initRumorTracker()` IIFE.
Note: the per-location edit-panel rumour card still only edits text+dmNote (not details) — details live in the tracker. Could add details there later for parity.

## ✅ Icon Size decoupled from Label Text Size (DONE — latest session)
New field `iconScale` (default 1; in newRegion/migrate/export passthrough).
- **Polygons with "Show type icon inside":** the embedded icon previously scaled with `labelScale` (same slider as the kingdom's name text). Now it uses `iconScale` via a new **"Icon Size"** slider (`ep-iconscale`, 30–500%), shown only when showIcon is on. Label Text Size now affects only the text.
- **Pins:** already independent (pinSize=icon+ring marker, labelScale=text). Renamed the pin's "Pin Size" slider label → **"Icon Size"** for vocabulary consistency. No functional change for pins.
Both editor render (~line 1755) and export render (~line 3873) switched from `r.labelScale` → `r.iconScale` for the polygon icon.

## ✅ Polygon hide-border + Kingdom Name Style (DONE — latest session)
- **Hide border for polygons:** the pin `hideBorder` toggle now also applies to polygons (label "Hide pin border" → "Hide border"). Polygon `<polygon>` stroke set to 'none' when `hideBorder && dispMode!=='border'` (and, in editor, not selected/hovered so it stays editable). Both render paths.
- **Name Style (`labelStyle`: 'normal' | 'caps' | 'spread'):** new select shown for polygons. `caps` = UPPERCASE + ~0.22em tracking (engraved look). `spread` = UPPERCASE with letter-spacing auto-computed to span ~60% of the polygon's bbox width (classic kingdom-name-across-territory look). Applied in both `appendLabel` (editor) and `appendPlayerLabel` (export); x shifted by −lspacing/2 to keep centered. Field added to newRegion/migrate/export passthrough.

## ✅ Player-export interactions: party drag, kingdom display toggle, rename (DONE — latest session)
- **Movable Party pin (export only):** the Party Location marker's hit circle now supports pointer drag (the ONLY movable pin). `pointerdown/move/up` on `phit`; updates `MAP_DATA.partyMarker.px/py`, repositions ring+glyph+hit live; a <5px move counts as a click → opens its popup. Dragged position saved to `localStorage` key `dndmap_party:<world>` and restored on load (one-time block near top of export script). Cursor grab/grabbing.
- **Player kingdom display toggle (export only):** new toolbar button `#terr-toggle` ("⬡ Kingdoms: Default/Filled/Outline/Hidden"). Cycles a global `terrOverride` that overrides EVERY territory/faction polygon's `dispMode` in `renderMap()` (`if((isTerritory||isFaction)&&terrOverride) dispMode=terrOverride`). Only affects kingdoms/territories+factions, never other regions. Saved to `localStorage` `dndmap_terr_disp:<world>`. Button hidden if the map has no territories/factions. `initTerrToggle()` IIFE.
- **Rename:** "Settlements within these Borders" → **"Locations within these Borders"** (5 places: 2 editor panel labels, 1 export popup, 2 export Compendium) since ruins/wilderness/etc. also appear there.

## ✅ Road Rework (DONE — latest session)
Roads went from a single dotted style to a full typed/curved/labelled system. All editor + export, with new data fields in `newRegion`/`migrateRegions`/export passthrough. Shared helpers (`ROAD_STYLES`, `roadStyleOf`, `rcatmull`, `roadPathD`, `wallTicks`, `paintRoad`) are inserted into BOTH the editor scope (before `closeRoad`) and the export scope (before `renderMap`) as identical text.

- **Road types (`roadStyle`):** six presets in `ROAD_STYLES` — `trail` (thin faint dotted), `road` (default dotted), `kings` (cased double-stroke warm stone), `river` (smooth wavy blue, renders under terrain), `sea` (dashed cool), `wall` (solid dark + crenellation ticks via `wallTicks`). Each has `color`/`wf` (width factor)/`kind`. Picking a style seeds `color` (still per-road overridable) and forces `smooth` for water styles. Edit panel: "Road Type" `<select>` (`#ep-roadstyle`).
- **Smooth/curved (`smooth`):** `roadPathD(points, smooth)` returns a Catmull-Rom cubic-bezier path (`rcatmull`) when smooth, else straight `M…L`. Per-road toggle `#road-smooth-toggle`; auto-on for river/sea.
- **Path-following name label (`showRouteLabel`, off by default):** `<textPath>` along a `<path id="rdp_ID">`, italic Georgia with stroke halo, color = road color. Toggle `#road-label-toggle`.
  - **Label adjustment (added same session):** `labelOffset` (startOffset %), `labelDy` (perpendicular offset in font-size units), `labelStraight` (0–100% lerp of label path toward the straight endpoint-to-endpoint line, to tame jagged curvature), `labelFlip` (reverse path direction for upside-down text). Editor-only: the label text is draggable on the map (`startRoadLabelDrag` projects cursor onto the path via `getPointAtLength` to set offset + perpendicular dy). Sliders/toggle live in `#road-label-adjust` (shown only when label is on).
- **Endpoint snapping (`fromRef`/`toRef`):** `snapRoadEnds(r)` runs after drawing/continuing — snaps each end to a nearby pin (within ~0.025 norm) or detects an enclosing polygon, storing the location id. Live gold snap-ring preview while drawing. Edit panel "Connects" two `<select>`s let you set/override endpoints manually.
- **Player route info (export):** road popups show a "Type" + "Connects A ↔ B / Leads to" `kingdom-info` block. Legend now emits **one row per road style present** (style-specific SVG glyph). Compendium has a **Routes** section: `routes()` lists named roads; `routeReadHtml(r)` renders type, connects, lore, rumours. `show(id)` short-circuits to `routeReadHtml` for road kinds.
- **Z-order:** both editor (`renderSvg`) and export (`renderMap`) sort the "others" array by a `_zr` rank — rivers (-1) under everything, other roads (1) above fills, pins (2), labels (3) on top — with original index as tiebreaker for stable order.

Vertex-edit parity (chunk 4 of the original plan) was deprioritized — roads still use Continue-from-End + whole-road move. Measured distances were intentionally skipped (no map-scale setting; maintainer chose to defer).

## ✅ Player party-marker fixes (DONE — latest session)
- **Export honored `partyMarker.size`:** the export render had hardcoded `pms=iconSize*1.15`, ignoring the DM's size — so a marker set small in the editor exported at full size. Now `pms=iconSize*1.15*partyScale`.
- **Player-resizable party pin (export only, that one pin):** `partyScale` starts from `partyMarker.size`, then any player override in `localStorage['dndmap_partysize:'+worldKey]`. Resize via scroll-wheel over the marker OR the −/+ "Marker size" buttons injected into the party popup (`window.__partyResize`). The marker block was refactored to a `drawParty()` that recomputes ring/icon/hit + the pulse keyframes; persists on change. Clamped 0.3–4×.
- **One popup per unique entity (export):** `showPopup` now stamps `dataset.popupKey` (`r.id`, or `__party__`, or name|tag fallback) and, if a popup with that key is already open, just brings it to front instead of cloning — fixes infinite duplicate popups on repeated clicks. Different entities still open side-by-side.

## Mission Statement

This is a **lore and immersion tool** for tabletop RPG dungeon masters. The primary audience is the **players** — the DM uses `editor.html` to annotate a world map with regions, lore, NPCs, and buildings, then exports a polished self-contained `world-map.html` that players can open in any browser to read about the world.

**What this is NOT:** a virtual tabletop, a combat tracker, a dice roller, or a Foundry VTT replacement. Every feature decision should be filtered through the question: *does this help players get immersed in the world?*

---

## Technical Constraints (non-negotiable)

- **Single HTML file** — `editor.html` opens directly in a browser. No build step, no npm, no server, no dependencies, no CDN.
- **Player export** (`world-map.html`) is also a single self-contained file — image, data, and all CSS/JS are embedded. Players double-click it and it works offline.
- **No backend** — all storage is browser-side (IndexedDB). Collaboration is out of scope.

---

## Repository Layout

```
editor.html     — The entire DM editor (~3750 lines, single file)
README.md       — User-facing documentation
PLANS.md        — This file (agent handoff)
```

The player export is not a file on disk — it is generated at runtime by `generatePlayerExport()` inside `editor.html` and downloaded as `world-map.html`.

---

## Architecture

### Map rendering
- A `<div id="map-inner">` holds an `<img>` and an `<svg>` stacked on top of each other.
- CSS `transform: translate(x,y) scale(s)` on `map-inner` handles all zoom/pan — the browser does the work, no canvas redraws.
- The SVG has `viewBox="0 0 imgWidth imgHeight"` so SVG coordinates = image pixel coordinates.
- **All coordinates stored as fractions (0–1)** of image dimensions, so they are resolution-independent.
- `vector-effect="non-scaling-stroke"` keeps stroke widths constant in screen pixels at all zoom levels.

### Storage
- **IndexedDB** database `dnd_map_db`, object store `state`, key `current`.
- Stores the entire `state` object serialized as a JS object (not JSON string).
- `saveState()` is debounced 400ms — call it after every mutation.
- `loadState()` is `async` — it returns a Promise. The init section uses `.then()`.
- **Migration:** when loading old saves, `migrateRegions()` adds missing fields. Every time a new field is added to a region, add a default there.
- There is a one-time migration from the old `localStorage` key `dnd_map_state` — this ran for existing users upgrading from the localStorage version.

### Undo/redo
- `undoStack` and `redoStack` store **JSON snapshots of `state.regions`** (not the full state — scratchpad and partyMarker are not undoable).
- Call `snapshot()` before every mutation that should be undoable.
- `saveState()` persists to IndexedDB; `snapshot()` just pushes to the in-memory stack.
- 100-step limit.

### SVG rendering
- `renderSvg()` completely clears and redraws the SVG on every call. This is intentional — it is fast enough for typical maps with <200 regions.
- Do not try to do incremental updates to the SVG. Always call `renderSvg()` after state changes.
- Glow filter is defined in `<defs>` at the top of every render.
- **Render order = z-order = click priority** (later elements are on top and capture clicks first via `stopPropagation` in `onShapeClick`). `renderSvg()` splits `state.regions` into `territories` (`kind==='polygon' && isTerritory`, rendered FIRST/bottom) and `others` (rendered after). This means clicking empty space inside a kingdom's border falls through to the territory polygon, while clicking a pin/region inside it selects that smaller shape — no custom hit-testing needed, it's a natural consequence of paint order. Same split exists in the player export's `renderMap()`.

---

## State Object Reference

```javascript
state = {
  imageB64: string|null,          // base64 data URL of the map image
  imgW: number,                   // natural image width in pixels
  imgH: number,                   // natural image height in pixels
  nextId: number,                 // auto-increment for region IDs
  scratchpad: string,             // DM scratchpad text
  partyMarker: {
    active: boolean,
    px: number,  // normalized 0-1
    py: number,
    size: number // size multiplier, default 1
  },
  regions: Region[]
}
```

### Region object

All region kinds share this base shape. Fields that don't apply to a kind are still present (set to defaults) to avoid null checks everywhere.

```javascript
{
  id: string,           // 'r1', 'r2', etc.
  kind: 'polygon'|'pin'|'road'|'label',
  name: string,
  tag: string,          // one of TYPES[] — 'City','Town','Village','Dungeon','Ruin','Wilderness','POI','Road','Water','Other'
                        // labels have tag:'Label' but the Type field is hidden for them
  color: string,        // hex color e.g. '#c0392b'
  publicLore: string,   // exported to players
  dmNotes: string,      // NEVER exported
  explored: boolean,    // false = fog of war
  coverImage: string|null,  // base64 JPEG, compressed to 800px max
  fillOpacity: number,  // 0-1, controls fill transparency (stroke always full)
  pinSize: number,      // size multiplier for pins, default 1
  roadSize: number,     // size multiplier for road dot size, default 1
  fontSize: number,     // size multiplier for label text, default 1
  fontStyle: string,    // 'normal'|'italic'|'bold'|'bold italic'

  // polygon and road only:
  points: [number, number][],  // array of [nx, ny] normalized coords

  // pin and label only:
  px: number,  // normalized x
  py: number,  // normalized y

  // NPC and building arrays (NOT on labels):
  npcs: NPC[],
  buildings: Building[],

  // status flags (NOT on labels):
  statusTags: string[]  // subset of STATUS_TAG_NAMES: 'Quest Active','Visited','Cleared','Hostile','Rumoured'

  // Territory (polygons only — see "Territory system" below):
  isTerritory: boolean,       // flag, NOT a separate `kind` — reuses all polygon machinery
  territoryDisplay: string,   // 'filled' | 'border' | 'hidden'
  ruler: string,              // free-text display name (kept in sync when rulerRef is set)
  government: string,         // free-text, no linking support
  capital: string,            // free-text display name (kept in sync when capitalRef is set)
  rulerRef: {regionId, npcId}|null,  // optional link to an existing NPC; cleared if `ruler` is hand-edited
  capitalRef: string|null,           // optional link to an existing region's id; cleared if `capital` is hand-edited
  _prevTag: string|undefined  // transient — remembers the tag a region had before being marked a territory, restored on un-marking
}
```

> **The `Territory` tag is special.** When `isTerritory` is set, `tag` is forced to `'Territory'` (a dedicated entry in `TYPE_COLORS` that deliberately is NOT in the `TYPES` list, so it never appears as a pickable Type for ordinary pins/regions). The Type field is hidden in the edit panel whenever `r.isTerritory` is true. Old saves used `tag:'Kingdom'` — `migrateRegions()` converts them silently. The `Kingdom` key is kept in `TYPE_COLORS` as an alias so legacy color lookups don't fail.

### NPC / Building object

```javascript
{
  id: string,
  name: string,
  description: string,  // public — exported
  dmNotes: string,      // private — NEVER exported
  image: string|null    // base64 JPEG, compressed to 400px max
}
```

### partyMarker

`state.partyMarker` is **not** a region. It is a separate singleton object. It is rendered separately at the end of `renderSvg()` and does not appear in the sidebar or region list.

---

## Key Functions

| Function | Purpose |
|----------|---------|
| `renderSvg()` | Clear and redraw the entire SVG overlay |
| `renderSidebar()` | Rebuild the region list in the sidebar |
| `repopulateEditPanel()` | Update all edit panel fields for `selectedId` |
| `selectRegion(id)` | Set `selectedId`, call renderSvg/sidebar/editPanel |
| `snapshot()` | Push current regions to undoStack (before mutations) |
| `saveState()` | Debounced write to IndexedDB |
| `loadState()` | async — opens IndexedDB, reads state, calls loadImageFromB64 |
| `migrateRegions()` | Add missing fields to old region objects after load |
| `newRegion(kind, tag)` | Create a new region with all defaults populated |
| `compressImage(b64, maxDim)` | Canvas-resize image to maxDim, returns Promise<b64> |
| `generatePlayerExport()` | Returns the complete world-map.html as a string |
| `appendLabel(cx,cy,r,W)` | Add an SVG text label above a shape (polygons and pins only — NOT roads) |
| `renderStatusDots(cx,cy,r,W)` | Render colored dot indicators below a region label on the SVG |
| `buildPinPalette()` | Populate the drag palette in the sidebar |
| `buildTypeFilter()` | Build the type-chip filter row in sidebar |
| `setTool(tool, asTerritory)` | Activate a drawing tool ('polygon','label',null). `asTerritory` (polygon only) sets the module-level `drawingTerritory` flag so the Kingdom button can reuse all polygon-drawing code paths — see Drawing Tools |
| `closePolygon()` | Finish a polygon being drawn; if `drawingTerritory` is set, also marks the new region as a territory (`isTerritory=true, tag='Kingdom'`) |
| `pointInPolygon(px,py,points)` | Ray-casting point-in-polygon test (normalized coords) |
| `settlementsInTerritory(territory)` | Returns regions (pins/small polygons, non-territory, non-label/road) whose centroid falls inside the territory's border |
| `entitiesInTerritory(territory)` | Aggregates `{npcs, buildings}` from every settlement `settlementsInTerritory` finds, each tagged with its parent region for display/navigation |
| `renderTerritorySettlements(territory)` | Populates the edit panel's settlements/NPCs/buildings lists for the selected territory |
| `renderTerritoryEntityList(containerId,fieldId,entries,entityType)` | Generic renderer for the territory's aggregated NPC/building chip lists; wires click-to-navigate |
| `navigateToEntity(regionId,entityType,entityId)` | Selects a region, switches the edit panel to its NPCs/Buildings tab, opens and scrolls to the given entity's accordion card |
| `allNpcsWithLocation()` | Flat list of every NPC across all regions with parent-location context, used to populate the Ruler picker |
| `candidateCapitals(excludeId)` | Non-territory pin/polygon regions eligible to be linked as a Capital |
| `renderRefChip(kind)` | Renders the clickable "🔗 linked to X" chip (with unlink ✕) below the Ruler/Capital fields when `rulerRef`/`capitalRef` is set |
| `unlinkRef(kind)` | Clears `rulerRef`/`capitalRef` without touching the free-text name |
| `closeRoad()` | Finish a road being drawn (handles continuation too) |
| `openLoreBrowser()` | Open the full-screen lore browser overlay (sets lbSelectedId if needed) |
| `closeLoreBrowser()` | Close the lore browser overlay |
| `renderLbList()` | Rebuild the sidebar list in the lore browser |
| `renderLbMain()` | Rebuild the main content area of the lore browser for the selected region; wires all inline-edit events |
| `lbMetaHTML(r,isLabel)` | Returns header HTML for lore browser (name input, tag badge, chips, status chips) |
| `lbEntityHTML(entity,entityType)` | Returns collapsible entity card HTML with editable name/description fields |
| `autoResize(el)` | Set a textarea's height to fit its content (call on load and oninput) |
| `debounceLb(regionId,field,value)` | 350ms debounced write of a region field back to state |
| `lbEntityChanged(...)` | Commit an entity field edit to state and sync visible DOM |
| `debounceLbEntity(...)` | 350ms debounced version of lbEntityChanged |
| `renderMarkdown(src)` | Convert a markdown string (headers, bold/italic, lists, wikilinks) to sanitized HTML for lore/notes display |
| `inlineMd(text)` | Inline-only markdown pass (bold/italic/wikilinks) used inside list items and headers by `renderMarkdown` |
| `resolveWikilink(name)` | Look up a `[[name]]` against all regions/NPCs/buildings; returns the matching entity or null |
| `parseMarkdownNote(text)` | Split an Obsidian-style note into `{name, description, dmNotes}` by reading frontmatter/headers/sections |
| `wireMdField(el, getter, setter)` | Wire a rendered markdown field for click-to-edit: shows rendered HTML, swaps to a raw-text textarea on click, saves on blur |
| `navigateLbWikilink(name)` | Resolve and jump to a `[[wikilink]]` target from inside the lore browser (selects region/entity, scrolls to it) |
| `importEntityFromMarkdown(type, r)` | Reads a `.md` file and creates a new NPC/Building in region `r` from its parsed name/description/DM notes |
| `importRegionLoreFromMarkdown(r)` | Reads a `.md` file and overwrites region `r`'s public lore / DM notes from its parsed sections |
| `openStubModal(names, regionName)` | Async modal — lets the DM classify each unresolved `[[wikilink]]` name as Person / Place / Skip; resolves `{npcs, places}` |
| `createStubReferences(r, parsed)` | Scans imported text for unresolved `[[wikilinks]]`, opens `openStubModal`, stubs chosen names as NPCs in `r` or queues them as pending locations |
| `describeStubResults(picks)` | Formats a `{npcs, places}` result into a one-line status message |
| `addPendingLocation(name)` / `renderPendingLocations()` | Queue a place name in the sidebar's "Places to add" list and (re)render it as draggable chips |

---

## Drawing Tools

### Polygon (`P` key)
- Click to place vertices. Near first vertex, snap ring appears (gold circle). Double-click or Enter to close. Right-click cancels.
- **Point placement uses `mouseup` not `click`** — same reason as roads (see below). The `click` handler on `mapSvg` explicitly gates out `activeTool==='polygon'`. The `mouseup` handler uses the same 10px drift threshold (`dx²+dy² < 100`).
- `snapClose` flag set in `updateDrawingCursor()`.

### Territory border (`K` key)
- Its own toolbar button (👑) and shortcut, separate from the plain Polygon tool.
- **Implementation reuses the polygon tool wholesale**: `setTool('polygon', true, false)` sets `activeTool='polygon'` plus `drawingTerritory=true`. Every event path that handles `activeTool==='polygon'` just works for territories too — only `closePolygon()`, `setTool()`, and the live-drawing preview check `drawingTerritory` to add territory-specific behavior (auto-set `isTerritory=true, tag='Territory'`, distinct purple dashed preview stroke, button highlight).
- `drawingTerritory` is reset to `false` on: finishing the polygon, Escape-cancel, right-click-cancel, and switching to a different tool.
- Existing polygons can also be converted via the "Mark as Territory" toggle in the edit panel.

### Faction territory (`F` key)
- Toolbar button (⚔), parallel to the Territory tool but for political factions.
- `setTool('polygon', false, true)` sets `drawingFaction=true`. `closePolygon()` auto-sets `isFaction=true, tag='Faction', fillOpacity=0.15`.
- Factions render at the very bottom of the SVG z-stack (below territories, below everything else) so locations inside them remain clickable. Stroke pattern: dotted `4 4` (vs territory's long dashes `13 6`).
- Territory and Faction flags are mutually exclusive on a single polygon — toggling one on turns the other off.
- Faction edit fields: `factionLeader`, `factionGoal`, `factionRelations[]` (`{targetId, type:'ally'|'neutral'|'rival'|'enemy', tension:1-5}`).
- `settlementsInFaction()` mirrors `settlementsInTerritory()` for auto-detected contents.
- Player export: faction overlays on map; location popups show "Faction Influence" chips for any factions covering that spot.

### Road (drag from palette)
- Dragging Road tile places first point and enters `activeTool='road'`.
- **Point placement uses `mouseup` not `click`** — this is intentional. The browser suppresses `click` events when the mouse moves even slightly between down and up, because `renderSvg()` is called on every `mousemove` during drawing. Using `mouseup` with a 10px drift threshold (`dx²+dy² < 100`) is reliable.
- Double-click detection uses a 280ms timer (`lastRoadClick`) between consecutive mouseups, not the browser's `dblclick` event.
- `continueRoadId` — when set, `closeRoad()` appends new points to an existing road instead of creating a new region.

### Label (`L` key)
- Click places a label, auto-exits the tool.
- Edit panel for labels: hides Type field, Cover Image section, NPC and Building tabs. Forces switch to Info tab on selection (bug prevention — activeTab can be stuck on 'npcs' from previous selection).
- Scroll wheel resizes font (requires selected + hovered, same as pins).

### Pin (drag from palette)
- Dragging any type tile (except Road, which is special) places a pin.
- `defaultPinSizes[tag]` remembers last scroll-resized size per type.

---

## Mouse Event Architecture

All `mousemove` and `mouseup` are on **document** (not mapContainer), so fast mouse movement outside the container doesn't drop events.

`mapContainer.mousedown` handles:
- Button 0: stores `toolMousedown` position for tool click detection
- Button 1 or 2: starts pan (`isDragging`)
- Button 2 during road drawing: finishes road
- Button 2 during polygon drawing: cancels
- Button 2: pans (no longer deselects — use Esc to deselect)

Special drag states (checked in order in `mousemove`):
1. `partyDrag` — moving party marker (no Ctrl needed)
2. `moveState` — Ctrl+dragging a region/road
3. `vertexDrag` — dragging a vertex handle

### Scroll wheel handler (order matters)

```
1. partyHovered → resize party marker
2. activeTool==='road' → resize road dot size during drawing
3. hoveredId===selectedId → resize pin/road/label
4. (fallthrough) → zoom map
```

---

## Player Export

`generatePlayerExport()` returns a complete HTML string. Key points:

- The embedded `<script>` tag is closed with `<\/script>` (backslash-escaped) to prevent the browser from closing the outer editor script.
- `PIN_ICONS` is passed in as `JSON.stringify(PIN_ICONS)` — the SVG paths are embedded as a JSON object in the generated file.
- DM notes are stripped from NPCs and buildings during export.
- Labels render as plain SVG text (no popup). If `r.explored === false`, the label is hidden.
- Roads render as dotted SVG polylines, scaled by `r.roadSize`.
- Party marker renders with a CSS pulse animation (`@keyframes pulse`). Clicking it shows a popup saying "Your party is currently here."
- Fog of war: polygons and pins get a dark overlay if `!r.explored`; labels are simply not rendered.

---

## Edit Panel Behaviour

The edit panel has three tabs: **Info**, **NPCs**, **Buildings**.

For **labels**: Info tab is forced active on selection, NPCs/Buildings tabs are hidden, Type field is hidden, Cover Image is hidden. Font style buttons and label presets appear instead.

For **pins**: Size slider (ep-size) shows as "Pin Size". No Continue Road button.

For **roads**: Size slider shows as "Road Size". Continue Road button appears.

The `ep-size` slider is reused for pin size, road size, and font size. The `oninput` handler checks `r.kind` to know which field to write to.

**Entity image upload** (NPC/building): updates DOM in-place using `updateEntityField`. Never calls `renderEntityList()` for image changes — that would collapse the accordion. Instead, it finds the specific DOM nodes by `[data-id]` and mutates them directly.

---

## CSS Notes

All styles are inline in `<style>` at the top of `editor.html`. No external stylesheets.

Key CSS variables: `--bg`, `--surface`, `--panel`, `--border`, `--accent` (#c9a84c gold), `--danger`, `--text`, `--text-dim`, `--dm-red`.

Modals use `.modal-backdrop` (fixed overlay) + `.modal-box` (centered content). Open state is toggled with `.open` class.

Player export has its own completely separate CSS inside the template literal.

---

## Gotchas and Known Issues

1. **`activeTab` persistence bug** — When switching from a region with NPCs to a label, the `activeTab` variable stays 'npcs', making the NPC pane visible. Fixed by checking `isLabel && activeTab !== 'info'` in `repopulateEditPanel()` and forcing the switch. Any future region kind that has a restricted tab set needs the same treatment.

2. **`r.kind === 'label'` missing from wheel condition** — When adding new kinds that support scroll resize, they must be added to the condition in the wheel handler: `(r.kind==='pin'||r.kind==='road'||r.kind==='label')`. Easy to miss.

3. **Party marker is not a region** — It lives in `state.partyMarker`, not `state.regions`. It does not appear in the sidebar, undo stack, type filter, or JSON export regions array. It IS included in the player export JSON as a separate top-level key. Don't try to select it with `selectRegion()`.

4. **Road and polygon both use `mouseup`** — Both use `document.addEventListener('mouseup')` with a 10px drift check (`dx²+dy² < 100`). The `mapSvg` click handler explicitly gates out both `'road'` and `'polygon'`. Do not move either back to the click handler — it suppresses events when `renderSvg()` runs during `mousemove`.

5. **Labels have `tag: 'Label'`** which is not in the `TYPES` array and not in the `ep-tag` dropdown. The Type field is hidden for labels. Do not try to show it — setting `ep-tag.value = 'Label'` silently fails and shows the wrong value.

6. **`compressImage()` is async** — It returns a `Promise`. Any code calling it must be `async/await` or use `.then()`. The file input `onchange` handlers use `async function()`.

7. **`loadState()` is async** — The init section uses `loadState().then(...)`. Do not call functions that depend on loaded state synchronously after `loadState()`.

8. **Template literal in generatePlayerExport** — Any backtick character inside the template will break the outer template. The player export JS uses string concatenation (`'...' + variable + '...'`) instead of template literals for this reason.

9. **`migrateRegions()` must be updated** — Every time a new field is added to a region object, add a default for it in `migrateRegions()`. Otherwise old saves will have `undefined` for that field and cause subtle bugs.

10. **`innerHTML` on SVG elements** — The pin icon shapes use `iconG.innerHTML = PIN_ICONS[r.tag]`. This works in all modern browsers for SVG elements. SVG elements parse their innerHTML as SVG XML, so all SVG elements (`rect`, `polygon`, `path`, etc.) are created correctly.

---

## Completed Features

### Resizable edit panel
The right-hand edit panel has a drag handle on its left edge. Dragging it resizes the panel between 280px and 700px. Implemented via a `#resize-handle` div (4px wide, `cursor:col-resize`), `mousedown` on it sets `resizing=true` and disables the width CSS transition so dragging is not laggy. The panel width is updated via `document.documentElement.style.setProperty('--edit-w', newW+'px')`.

### Full-screen Lore Browser (`←` / `B` key)
Toggled by the `←` button in the edit panel tab bar, the `📖` button in the toolbar, or the `B` key. Opens a fixed overlay covering the full viewport. Left sidebar shows a searchable, type-filterable list of all regions. Main area shows the selected region's cover image, metadata, status chips, public lore, DM notes, and collapsible NPC/building cards.

**All fields are inline-editable directly in the lore browser** — name, lore, DM notes, entity names and descriptions auto-save with a 350ms debounce back to `state`, the sidebar, and the SVG. Textareas auto-resize to fit content. DM notes are always shown (even when empty) so DMs can type directly without going back to the edit panel.

### Territory system (`r.isTerritory`)
A specialized layer on top of the polygon tool for marking regions/kingdoms, drawn with the dedicated 👑 Kingdom tool (`K` key — see Drawing Tools) or retro-fitted onto an existing polygon via the "Mark as Territory / Kingdom" toggle in its edit panel.

- **Click-through by design, no custom hit-testing**: territories are rendered first (bottom of the SVG z-order — see "SVG rendering" notes above), so clicking a pin or small region inside a kingdom's border selects *that* shape, while clicking empty space inside the border falls through to select the kingdom itself. This was the user's "crucial" requirement and falls out naturally from paint order + existing `stopPropagation` click handling.
- **Border display modes** (`territoryDisplay`): `'filled'` (normal polygon fill+stroke), `'border'` (transparent fill, `pointer-events='visibleStroke'` so only the outline is clickable and the interior is fully click-through), `'hidden'` (shape isn't rendered at all — except a faint preview while selected/hovered for editing — but the name label always still renders, an intentional "name without border" cartography effect). All three are respected in both the editor and the player export.
- **Dedicated `Territory` tag** fixes a real bug where territories could retain a mundane type like `'City'`; see the `_prevTag` note in the Region object reference above. Old saves that stored `tag:'Kingdom'` are silently migrated to `'Territory'` in `migrateRegions()`; the `Kingdom` key in `TYPE_COLORS` is kept as an alias so any pre-migration color lookups don't break.
- **Kingdom Details fields**: Ruler/Leader, Government, Capital — free-text, but Ruler and Capital can additionally be *linked* to an existing NPC or location via `rulerRef`/`capitalRef`. Linked refs render as clickable "🔗 Name" chips (editor edit panel, lore browser, and player popup) that jump straight to the target — for an NPC ruler, this means selecting their home region, switching to its NPCs tab, and auto-expanding+scrolling to their card (`navigateToEntity` / `navigateToPopupEntity`). Hand-editing the free-text field clears the ref (keeps things from silently going stale).
- **Auto-detected contents**: `settlementsInTerritory()` ray-casts every pin/small-polygon centroid against the border to build a live "Settlements within these Borders" list (click to select), and `entitiesInTerritory()` goes one level deeper, aggregating those settlements' NPCs and Buildings into "Notable NPCs/Buildings in these Lands" chip lists — all clickable, all kept in sync automatically as the map changes (no manual bookkeeping). The player-export equivalents additionally filter out unexplored locations (`!r.explored`) so fog-of-war secrets aren't leaked.

### Markdown lore + Obsidian-style import (`renderMarkdown`, `importEntityFromMarkdown`, `importRegionLoreFromMarkdown`)
Public lore, DM notes, and entity descriptions are written in a small Markdown subset (headers, **bold**/*italic*, lists, and `[[wikilinks]]`) and rendered to HTML wherever they're displayed — edit panel, lore browser, and player popups (`renderMarkdown`/`inlineMd`). Click any rendered field to swap it back to a raw-text textarea for editing (`wireMdField`); it re-renders on blur. `[[Name]]` and `[[Name|Display Text]]` links resolve against every region/NPC/building (`resolveWikilink`) and become clickable jumps (`navigateLbWikilink` in the lore browser; player popups get the read-only equivalent).

DMs can also **import a whole Obsidian note** instead of typing: `#btn-import-npc-md` / `#btn-import-building-md` create a new NPC/Building from a `.md` file's parsed name + body (`importEntityFromMarkdown`), and `#btn-import-lore-md` overwrites a region's public lore/DM notes the same way (`importRegionLoreFromMarkdown`). `parseMarkdownNote` splits the file into `{name, description, dmNotes}` by reading the title/frontmatter and a "DM Notes"-style heading, so a single export from Obsidian (or any markdown notebook) drops in with no reformatting.

### Stub-reference creation from imports (`createStubReferences`, `openStubModal`, pending locations)
Imported notes routinely mention `[[entities]]` that don't exist in the world yet — other NPCs, other settlements. After every markdown import, `createStubReferences` scans the new text for `[[wikilinks]]` that don't `resolveWikilink`, and — if any are found — opens `openStubModal`, a per-name classifier modal:

- **🧑 Person** → stubbed immediately as an empty NPC inside the region just imported into (a person has to be *somewhere*, and "wherever the DM put this note" is the best guess available).
- **📍 Place** → too risky to auto-place (a region needs real map geometry — a polygon or a pin position — that can't be guessed from a name). Instead it's queued in the sidebar's "📍 Places to add" list as a draggable chip; dragging it onto the map drops a pre-named pin there via the existing drag-and-drop pipeline (`__pending__:` payload prefix in the `mapContainer` drop handler), ready for the DM to reshape/retag/annotate.
- **— Skip** → ignored; the `[[link]]` simply stays unresolved until something with that name is created normally.

Either way, the moment a matching region/NPC/building is created — by this flow or by hand — the original `[[link]]` resolves and becomes clickable retroactively. The pending-locations queue is intentionally session-only (not persisted to `state`/IndexedDB): the `[[link]]` in the source note is the durable record, so there's nothing to lose by not saving the queue itself.

### Status tags (`r.statusTags`)
Five colored flags per region: *Quest Active* (gold), *Visited* (green), *Cleared* (grey), *Hostile* (red), *Rumoured* (purple).
- Toggle chips in the edit panel (Info tab, below fog-of-war toggle). Hidden for labels.
- Colored dot indicators rendered below region name labels in the SVG (`renderStatusDots()`).
- Colored chips shown in the lore browser header.
- Exported to `world-map.html` and shown as chips in the player popup header.

---

## Planned Features (Player Experience Focus)

These are ordered by impact-to-effort ratio. The core question for each: *does this help players engage with the world?*

### ✅ APPROVED ROADMAP (build in this order — agreed with maintainer, June 2026)

> These are confirmed and sequenced. Everything below this block under High/Medium/Lower priority is still-unscheduled backlog. **Multi-map linking is the FINAL feature** — do it last, on its own, after all of the below ship and feel solid.

**A. New pin types (✅ DONE — Temple, Shrine, Port, Harbor, Camp, Cave, Mine added to TYPES/TYPE_COLORS/PIN_ICONS)**
Add seven types to the existing palette. Same mechanism as the Bastion pin already added (`TYPES`, `TYPE_COLORS`, `PIN_ICONS` — auto-flows to palette, type-filter chips, `ep-tag` dropdown, and the player export which `JSON.stringify`s `PIN_ICONS`).
- New tags: **Temple**, **Shrine**, **Port**, **Harbor**, **Camp**, **Cave**, **Mine**.
- For each: add to `TYPES` array (sensible grouping), add a distinct color to `TYPE_COLORS`, add a 20×20 `PIN_ICONS` entry (simple geometric SVG, fill inherits type color — same convention as existing icons; keep them readable at small size, no over-detailed paths).
- Remember to keep the palette legible — these seven are the agreed ceiling for hardcoded types; anything beyond waits for **custom pin icon upload** (see Medium priority #5).

**B. Map Legend / Key (player export) — ✅ DONE** *(highest "help players read the world" win)*
Shipped: `#legend-toggle` button in the export toolbar opens `#legend-panel`; `buildLegend()` scans EXPLORED regions only, lists each used pin type (icon in its color) + Road / Kingdom border / Faction swatches. Unused types omitted; button hides itself if nothing to show.
A collapsible legend in `world-map.html` mapping each color/icon → its type label, plus territory/faction swatches.
- Only list types that actually appear on THIS map (scan exported regions; don't show unused types).
- Lives in the export only (DM doesn't need it). Suggested placement: a toggle button in the export `#toolbar` (next to "Locations") opening a small panel, OR a corner card. Parchment styling, same palette as the rest of the export.
- Pure presentation — reads existing data, stores nothing.

**C. World Overview intro card (player export) — ✅ DONE** — sets the stage on first open
A welcome/gazetteer card shown when a player first opens the export (before they touch the map).
- New DM-authored fields on the top-level state: `state.worldName: string`, `state.worldIntro: string` (markdown, public). Author them in the editor (small "World" section — toolbar button or a panel; NOT per-region).
- Export: on load, if `worldIntro` is set, show a centered parchment card with the world name (small-caps gold) + rendered intro markdown + a light auto-stat line (e.g. "4 kingdoms · 23 explored locations"). Dismiss to reveal the map; remember dismissal in the player's `localStorage` so it doesn't nag every visit (key namespaced per export, e.g. by world name).
- Add `worldName`/`worldIntro` defaults in `migrateRegions()`-equivalent state migration so old saves don't break.

**D. "First Glimpse" read-aloud text (DM authoring → export) — ✅ DONE** — boxed read-aloud immersion
A dedicated per-region field for the classic boxed read-aloud a DM reads when players arrive.
- New field: `r.firstGlimpse: string` (markdown, public — exported only when `r.explored`). Add to `newRegion()` and the state migration with default `''`.
- Editor: a field in the Info tab (above or below Public Lore), clearly labelled "First Glimpse / Read-Aloud". Markdown-enabled like other lore fields (`wireMdField`).
- Export: render at the TOP of the location popup's Lore pane, in a visually distinct boxed style (e.g. left gold rule + slightly inset parchment panel, italic serif) so it reads as boxed text, separate from regular lore.
- Add to the `exportRegions` map inside `generatePlayerExport()`.

**E. Traveler's Journal (player export) — ✅ DONE** — player agency, no DM involvement

**F. Player-export Compendium (Lore Browser) — ✅ DONE** — a 📖 Compendium button in the export opens a full-screen two-pane reader (searchable location list + read-only reading pane: cover, kingdom/faction info, First Glimpse box, lore, rumours, NPC/building cards). Mirrors the editor's lore browser; reuses `renderMarkdown`/`resolveWikilink`. Wikilinks navigate within the browser.

**G. Multiple simultaneous popups — ✅ DONE** — refactored the export's single reused `#popup` into a clone-from-`<template id="popup-tpl">` model. `showPopup()` clones a `.popup` node, `wirePopup()` gives each its own drag/expand/close + bring-to-front (z-index counter); map-click no longer closes popups; Esc closes the most recent. CSS popup IDs → classes.
A personal notes box inside `world-map.html` that the PLAYER writes in, saved to THEIR browser only.
- Export-only feature; stores to `localStorage` (namespaced per export, e.g. `journal:<worldName>`), never part of `state`/the DM's data, never re-imported.
- A toggle in the export `#toolbar` opens a draggable/parchment panel (mirror the editor scratchpad pattern: header + textarea, persist value on input). Optional niceties later: timestamped entries, but a single free-text area is enough for v1.
- Zero impact on the editor or exported data integrity.

**Z. Multiple maps with linking — FINAL FEATURE (see "Multiple maps with linking" under High priority below for the architecture sketch).** Do not start until A–E are shipped and stable; it's the biggest architectural change (`state.maps[]`, per-pin `linkedMapId`, multi-page export navigation) and should be its own dedicated effort.

### High priority

**1. Rumours system**
Each region can hold a list of rumours — short snippets of information that players might hear about a place before visiting it. Unlike public lore (which is "what we know for certain"), rumours are individually flagged as *true*, *false*, or *misleading*, visible only to the DM.

Data shape (new array on every region):
```javascript
r.rumours = [
  { id: string, text: string, isTrue: boolean|null, dmNote: string }
]
```
- `isTrue: true` = accurate rumour. `false` = false/planted rumour. `null` = unknown/ambiguous.
- Player export: shows rumours as a "Rumours heard about this place" section in the popup, without the true/false flag — players see the text only.
- Editor: a "Rumours" sub-section in the Info tab (below public lore), with a small truth-indicator toggle (🟢 true / 🔴 false / ⚪ unknown) and a private DM note per rumour.
- Rumours are exportable to players only when `r.explored` is true (same rule as lore).
- A possible future enhancement: a DM-side "Rumour Board" overlay showing all rumours across all locations, filterable by true/false/unknown — useful for campaign prep.

**2. Multiple maps with linking**
The single biggest impact feature. A world map can contain city pins; clicking a city pin in the player export navigates to the city's own map (another embedded map page). DM creates child maps and links them to parent pins.

Architecture sketch:
- `state.maps[]` array, each map has its own image, regions, etc.
- Current structure becomes one entry in `state.maps`.
- A pin can have a `linkedMapId` field.
- Player export generates a multi-page HTML with JS navigation between embedded maps.
- Editor gets a "Maps" panel to switch between maps and a "Link to map" field in the pin editor.

**2. Session tracker**
Each region can record the session number when it was revealed.
- `r.revealedSession: number|null`
- Filter in the editor: "Show only regions revealed by session N" → temporarily sets fog of war for everything revealed after N.
- In the player export: optionally show a "Discovered in Session X" chip in the popup.

### Medium priority

**4. Scale ruler + distance measurement**
- DM sets scale in the editor (e.g. "1% of map width = 50 miles").
- A "measure" tool: click two points, see the distance.
- In the player export: a small scale bar in the corner.
- Stored in `state.mapScale: { value: number, unit: string }`.

**5. Custom pin icon upload**
- Allow uploading a 32×32 PNG to replace a type's icon or a specific pin's icon.
- Stored as base64 in the pin's `customIcon` field.
- Fallback to type icon if null.

**6. Export options**
- Export NPC/location list as a formatted HTML document (no map, just text).
- Useful for session recaps or player handouts.
- "Export Lore Book" button — generates a document with all explored locations, their lore, NPCs, and buildings in a readable format.

### Lower priority

**7. Left panel rework**
The current left panel is a fixed-layout mix of the drag palette (pin types + party marker) and the region list sidebar. As the tool grows, this creates crowding. Planned redesign:
- Collapsible panel sections with headers (drag handles to rearrange order).
- "Palette" section: pin types, road, party marker — can be collapsed when not needed.
- "Regions" section: search + filter chips + region list — current behavior, promoted to full panel height when palette is collapsed.
- "Pending Places" section (currently appears/disappears inline): persistent collapsible section that shows `[]` count badge when empty, expands to list draggable chips when non-empty.
- Optional: "Factions" quick-view section listing all faction regions with their color dots and relation summaries — a political overview without having to click around the map.
- The panel itself should be resizable (drag handle on the right edge, mirroring the existing right-panel resize handle).

**9. Relationship notes**
- Simple text field on an NPC: "Connected to [region/NPC name]".
- No visual graph — just a text reference that shows in the player popup.

**10. Mini-map**
- Small thumbnail of the full map visible when zoomed in.
- Fixed position, bottom-right of the map container.
- Click to teleport viewport.

**11. Hex/square grid overlay**
- Toggle a grid over the map.
- Store grid settings in state: `state.grid: { type: 'hex'|'square'|null, size: number, color: string, opacity: number }`.
- Not exported to players (DM tool only).

---

## How to Add a New Feature Safely

1. **Add new fields to `newRegion()`** with sensible defaults.
2. **Add those same fields to `migrateRegions()`** so old saves don't break.
3. **If it's exported**: add to the `exportRegions` map inside `generatePlayerExport()`, and add rendering in the player export's `renderMap()` function.
4. **If it adds a new kind**: add it to `renderSvg()` with its own rendering block. Consider whether it needs vertex handles, Ctrl+drag (check `onShapeMousedown` — polygon/road check), scroll resize (check wheel handler condition), and sidebar display.
5. **If it's a new UI element**: add CSS inside the `<style>` block at the top of editor.html.
6. **Test the undo chain**: call `snapshot()` before the mutation, `saveState()` after. Verify Ctrl+Z restores the previous state.
7. **Validate**: check `Braces: X { X }` in the PowerShell validation script. Check that `</script>` inside the template literal is written as `<\/script>`.

## Validation Script (PowerShell)

Run this after any changes to check brace balance:

```powershell
$content = Get-Content "E:\DND_APPS\editor.html" -Raw
$scriptStart = $content.IndexOf('<script>') + 8
$scriptEnd = $content.IndexOf('</script>', $scriptStart)
$js = $content.Substring($scriptStart, $scriptEnd - $scriptStart)
$tplS = $js.IndexOf('return `<!DOCTYPE'); $tplE = $js.LastIndexOf('`;')
$outer = $js.Substring(0,$tplS) + $js.Substring($tplE+2)
$o = ($outer.ToCharArray() | Where-Object { $_ -eq '{' }).Count
$c = ($outer.ToCharArray() | Where-Object { $_ -eq '}' }).Count
Write-Host "Braces: $o { $c } -> $(if($o-eq$c){'BALANCED'}else{"OFF $($o-$c)"})"
```

---

## Git / GitHub

- Repo: https://github.com/Arvi23/dnd-map-tool
- Branch: `master`
- Author: `Arvi23` / `carpatirtg@yahoo.ro`
- All commits co-authored by the assisting Claude model.
- Use `gh release create vX.Y.Z editor.html` to publish releases with the file as a direct download asset.

## ✅ Rumour Rework v2 (DONE — latest session)

> Built and verified in an isolated harness: reveal gate omits hidden rumours from export; Known/Tracking/Finished render; player notes persist + Save/Load JSON round-trips. Implementation summary below; the original plan follows for reference.

- **Part 1 — DM single export toggle:** removed the truth cycle (🟢/🔴/⚪) from both the edit-panel rumour cards and the Rumor Tracker rows. The 👁/🚫 `revealed` toggle is now the only per-rumour control (titles say "Exported to players" / "DM-only"). `isTrue` stays in the data model, just unsurfaced.
- **Part 3 — player tracker:** statuses are now **Known / Tracking / Finished** (`.s-known/.s-tracking/.s-finished`; finished = strikethrough+dim). Every rumour row is expandable (chevron always shown) and holds a per-rumour **Notes** textarea (✎ flag when filled), auto-saved to `localStorage['dndmap_rumor_notes:+world']`, keyed by `rum.id`. DM `details` still show above the notes when present.
- **Part 2 — portable notes:** **⤓ Save** downloads `{type:"dndmap-notes", worldName, statusMap, notesMap}` as `mynotes-<world>.json`; **⤒ Load** file-picks a backup and merges it in (imported entries win), then re-renders — so players keep their tracking across DM re-sends and across devices.

---

### Original plan (for reference)

## 🔜 Rumour Rework v2 (PLANNED)

Builds on the reveal-gate work already shipped (👁 per-rumour toggle; export filters to revealed only; statuses partly renamed). Three remaining pieces:

### 1. Simplify the DM rumour control — drop the truth cycle, keep ONE export toggle
- Today each rumour card/row has TWO controls: the truth cycle (🟢 true → 🔴 false → ⚪ unknown, `isTrue`) AND the new reveal toggle (👁/🚫, `revealed`).
- **Change:** remove the truth toggle entirely. Keep a single toggle that just answers "will this rumour be exported to players?" — i.e. the existing `revealed` gate becomes the ONLY per-rumour switch.
- Touch points: `renderRumoursList` (edit-panel cards, ~line 2820) and DM tracker `rowEl` (~line 3625) — delete the `tb` truth button + its handler; keep the reveal button. Consider relabeling it plainly (e.g. a checkbox "Export to players" or "Visible" pill) instead of an emoji eye for clarity.
- `isTrue` can stay in the data model (harmless) but is no longer surfaced. `dmNote` stays (DM-only).
- Migration already defaults `revealed=true` for existing rumours; leave that.

### 2. Player notes persistence across map re-sends (the core pain)
- **Problem:** every time the DM edits the map they re-export and re-send a fresh `world-map.html`. Player notes/status live in that file's `localStorage` keyed by `worldName` — a NEW file on the same origin keeps the same key, so notes *usually* survive… BUT if the player opens the new file from a different path/origin (downloads folder, email attachment, different device) the `localStorage` is separate and their notes are lost.
- **Fix:** give players a portable notes store, independent of the HTML file:
  - **Export my notes**: button in the player Rumor panel → downloads a small JSON ( `{worldName, statusMap, notesMap}` ) as e.g. `mynotes-<world>.json`.
  - **Import my notes**: button → file picker → merges that JSON back into `localStorage`. Merge strategy: imported entries win; keep any local-only entries.
  - Keep auto-save to `localStorage` as the default fast path; export/import is the durable backup for when the DM sends a new file or they switch device.
  - Optional nicety: on load, if a matching `worldName` JSON was previously imported, offer "restore notes".

### 3. Rework player note-taking (finish the half-built tracker)
- Statuses → **Known / Tracking / Finished** (replace following/confirmed/debunked). s-tracking = accent left border; s-finished = strikethrough + dim; s-known = neutral/baseline.
  - CSS: `.pr-row.s-following/.s-confirmed/.s-debunked` + the `.pr-row.s-debunked .pr-text` strikethrough rule → rename to `.s-known/.s-tracking/.s-finished`.
  - rowEl status array (~line 4829): `[['known','Known'],['tracking','Tracking'],['finished','Finished']]`.
- **Per-rumour Notes**: a textarea under each rumour (in the expandable body, alongside DM `details`). Make the chevron always expandable so notes are reachable even when a rumour has no `details`.
  - Persist to `localStorage` key `dndmap_rumor_notes:<world>`, keyed by `rum.id`. Never touches the DM file.
  - This `notesMap` is exactly what part 2's export/import should carry (together with `statusMap` from `dndmap_rumor_status:<world>`).

### Build order when greenlit
1, then 3 (statuses + notes), then 2 (export/import wrapping statusMap+notesMap). Verify in an isolated iframe only — NEVER write test data into the user's live preview/localStorage. Then update PLANS.md + README.md.

## ✅ Click-reliability fix (DONE — latest session)

> Root cause found: hovering any shape ran `hoveredId=id; renderSvg()`, rebuilding the WHOLE SVG — so the node under the cursor was destroyed mid-press. Browsers then re-fired enter/leave on the replacement (render thrash), and because polygons/roads/labels selected on the `click` event (which needs the same target for mousedown+mouseup), the swallowed click meant selection silently failed. Felt worst on big polygons; pins were OK because they select on mousedown.
>
> **Fixes:** (1) hover now applies glow/fill **directly** to the element — zero re-renders on hover (also a big perf win); (2) `onShapeMousedown` selects on press for ALL shapes (not just pins); (3) all shapes use the deferred-threshold drag, so a plain click never nudges a shape or spams undo, and only pins drag without Ctrl; (4) outline/border polygons got a 14px transparent `pointer-events:stroke` hit-band so the thin dashed line is easy to click. Render-time styling switched from `isSel||isHov` to `isSel` so a real render no longer depends on stale hover state. Verified: 0 renders on hover, polygon/border click-select reliable, click doesn't move the shape.

---

### Original investigation notes (kept for reference)

## 🐞 Cross-browser interaction bug (notes)

**Symptom:** the editor works great inside the in-app design preview, but in **Floorp** and **Edge** (standalone browsers) it's clunky — many elements can't be clicked reliably. User currently prefers the in-app left-sidebar preview because of this.

**Likely causes to investigate (editor.html):**
- SVG hit-targets: many shapes use thin strokes / transparent fills for hit detection. `pointer-events` defaults differ — verify every clickable SVG node sets explicit `pointer-events` (e.g. transparent fills need `pointer-events:all`; `fill:none` paths only catch on the stroke). Roads already use a fat transparent hit-line; check pins/polygon vertices/labels behave the same in Firefox-family (Floorp) which is stricter about `pointer-events` on zero-opacity nodes.
- Event model: code may rely on `mousedown`/`mousemove`/`mouseup` + `e.button`/`e.ctrlKey`. Mixing with the newer pointer events used in the party-marker drag can behave differently across engines. Consider unifying on **pointer events** (`pointerdown/move/up` + `setPointerCapture`) for all drags so capture works consistently in Floorp/Edge.
- Overlapping transparent layers stealing clicks: the SVG overlay, drag-preview layer, and label nodes may sit above intended targets. Audit z-order / `pointer-events:none` on non-interactive overlays.
- Drag threshold + `getScreenCTM()`/coordinate math: confirm coordinate transforms are correct when the page is scrolled/zoomed in a real browser window (the in-app preview may differ in sizing).
- Touch/zoom handlers (`pdist`, pinch) attaching passive/non-passive listeners differently.

**Approach when greenlit:** reproduce in Firefox + Chromium engines, add a small interaction test checklist (select pin, drag pin, drag label, click road, vertex insert/drag, open popups), then fix `pointer-events` + unify on pointer events. Verify in an isolated iframe AND ask the user to sanity-check in Floorp/Edge.


## ✅ Player location-notes + unified Save/Load (DONE — latest session)

- **Per-location player notes:** every export popup with an `r.id` (pins, polygons, roads — not the party marker) now has a **✎ Your notes** textarea, auto-saved to `localStorage['dndmap_loc_notes:'+world]` keyed by region id (re-reads on each input to avoid cross-popup clobber).
- **Unified player export:** `collectPlayerData()` / `downloadPlayerData()` / `importPlayerData()` bundle ALL player-authored state — journal (`dndmap_journal`), rumour statuses (`dndmap_rumor_status`), rumour notes (`dndmap_rumor_notes`), and location notes (`dndmap_loc_notes`) — into one `{type:'dndmap-notes', worldName, journal, statusMap, notesMap, locNotes}` file. Toolbar **⤓ Save Notes / ⤒ Load Notes** buttons drive it; the Rumor panel’s Save/Load were repointed to the same functions. Import merges (imported entries win; journal is appended with a `---` divider) then `location.reload()`s to refresh every view.
- **⚠️ Bug caught in verification:** the journal separator was written as `'\n\n---\n\n'` *inside the export template literal*, so `\n` collapsed to real newlines in the generated player script → unterminated string literal → the **entire** player export silently failed to run. Fixed by using `String.fromCharCode(10,10)`. Lesson: never put bare `\n`/`\t` escapes in string literals that live inside the export template literal — use `String.fromCharCode` or double-escape.
