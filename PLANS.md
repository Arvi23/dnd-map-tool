# D&D Map Editor — Agent Handoff Document

> **Read this first.** This document exists so that a future LLM agent can continue development without needing the full conversation history. It covers what the project is, how it is built, what exists, what is planned, and every gotcha we have hit so far.

---

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
editor.html     — The entire DM editor (~2400 lines, single file)
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
  buildings: Building[]
}
```

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
| `buildPinPalette()` | Populate the drag palette in the sidebar |
| `buildTypeFilter()` | Build the type-chip filter row in sidebar |
| `setTool(tool)` | Activate a drawing tool ('polygon','label',null) |
| `closePolygon()` | Finish a polygon being drawn |
| `closeRoad()` | Finish a road being drawn (handles continuation too) |

---

## Drawing Tools

### Polygon (`P` key)
- Click to place vertices. Near first vertex, snap ring appears (gold circle). Double-click or Enter to close. Right-click cancels.
- Uses `mapSvg.addEventListener('click')` — note: roads do NOT use click (see below).
- `snapClose` flag set in `updateDrawingCursor()`.

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
- Button 2 when something selected: deselects then pans

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

4. **Road click vs. polygon click** — Polygon uses `mapSvg.addEventListener('click')`. Road uses `document.addEventListener('mouseup')` with a drift check. Do not add road point placement back to the click handler — it won't work reliably.

5. **Labels have `tag: 'Label'`** which is not in the `TYPES` array and not in the `ep-tag` dropdown. The Type field is hidden for labels. Do not try to show it — setting `ep-tag.value = 'Label'` silently fails and shows the wrong value.

6. **`compressImage()` is async** — It returns a `Promise`. Any code calling it must be `async/await` or use `.then()`. The file input `onchange` handlers use `async function()`.

7. **`loadState()` is async** — The init section uses `loadState().then(...)`. Do not call functions that depend on loaded state synchronously after `loadState()`.

8. **Template literal in generatePlayerExport** — Any backtick character inside the template will break the outer template. The player export JS uses string concatenation (`'...' + variable + '...'`) instead of template literals for this reason.

9. **`migrateRegions()` must be updated** — Every time a new field is added to a region object, add a default for it in `migrateRegions()`. Otherwise old saves will have `undefined` for that field and cause subtle bugs.

10. **`innerHTML` on SVG elements** — The pin icon shapes use `iconG.innerHTML = PIN_ICONS[r.tag]`. This works in all modern browsers for SVG elements. SVG elements parse their innerHTML as SVG XML, so all SVG elements (`rect`, `polygon`, `path`, etc.) are created correctly.

---

## Planned Features (Player Experience Focus)

These are ordered by impact-to-effort ratio. The core question for each: *does this help players engage with the world?*

### High priority

**1. Multiple maps with linking**
The single biggest impact feature. A world map can contain city pins; clicking a city pin in the player export navigates to the city's own map (another embedded map page). DM creates child maps and links them to parent pins.

Architecture sketch:
- `state.maps[]` array, each map has its own image, regions, etc.
- Current structure becomes one entry in `state.maps`.
- A pin can have a `linkedMapId` field.
- Player export generates a multi-page HTML with JS navigation between embedded maps.
- Editor gets a "Maps" panel to switch between maps and a "Link to map" field in the pin editor.

**2. Status tags per location**
Quick visual flags: *Quest Active*, *Party Visited*, *Cleared*, *Rumoured*, *Hostile*.
- Stored as `r.statusTags: string[]` on each region.
- In the editor: small colored chips in the edit panel; small icon overlaid on the region in the SVG.
- In the player export: shown in the popup header. Could replace fog of war for "rumoured" locations (name visible, lore hidden).

**3. Session tracker**
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

**7. Relationship notes**
- Simple text field on an NPC: "Connected to [region/NPC name]".
- No visual graph — just a text reference that shows in the player popup.

**8. Mini-map**
- Small thumbnail of the full map visible when zoomed in.
- Fixed position, bottom-right of the map container.
- Click to teleport viewport.

**9. Hex/square grid overlay**
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
$scriptStart = $content.IndexOf('<script>', $content.IndexOf('generatePlayerExport') - 85000) + 8
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
