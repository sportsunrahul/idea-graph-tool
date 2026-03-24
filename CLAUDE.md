# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the tool

The app is a single HTML file with no build step or dependencies:

```bash
cd idea-tool
python3 -m http.server 8080
# Open http://localhost:8080/idea-graph-tool.html
```

> Google Drive sync requires HTTP (not `file://`). For local testing without Drive, just open the HTML file directly in a browser.

## Architecture

The entire application lives in one file: `idea-tool/idea-graph-tool.html`. It contains inline CSS (~350 lines), inline HTML structure, and inline JavaScript (~1700 lines). There is no build system, bundler, framework, or external runtime dependency.

### Data model

**Node:**
```js
{ id, title, desc, tags[], color, status, mastery, resources[], year, x, y, createdAt }
```
- `color`: one of `purple | teal | amber | coral | green | gray` (maps to `COLOR_PALETTE`). Auto-assigned from `tagToColor(tags[0])` when not explicitly set.
- `status`: `want | in-progress | done`
- `mastery`: `0–5` (maps to `MASTERY_LABELS`)
- `resources`: array of URL strings (arXiv URLs are auto-fetched)
- `year`: publication year, integer or null

**Edge:**
```js
{ id, source, target, type, label }
```
- `type`: `related | prerequisite | improves`
- `label`: optional string annotation (shown on canvas for `improves` edges)
- For `prerequisite` and `improves` edges, `source` must be understood before `target`.

### Migration

Every load path (localStorage + Google Drive) runs `migrateData()` before assigning to `nodes`/`edges`. This normalises any missing fields to their defaults. **When adding a new field to a node or edge, always add its default to `migrateNode()` or `migrateEdge()`** — never rely solely on `|| default` inline fallbacks.

### State

All mutable state is module-level `let` variables. Key ones:
- `nodes[]`, `edges[]`, `nextId` — the graph data
- `selectedNode`, `linkMode`, `prereqLinkMode`, `improvesLinkMode`, `linkSourceNode` — interaction state
- `camX`, `camY`, `camZoom` — canvas camera
- `activeTag`, `activeStatus`, `searchQuery`, `sidebarView` — sidebar filters (`sidebarView`: `'list' | 'order'`)
- `selectedColor`, `selectedStatus`, `selectedMastery`, `colorExplicitlySet`, `editingNodeId` — modal form state
- `hoveredNode` — node currently under cursor (drives tooltip + hover ring)

### Rendering

The canvas uses **immediate-mode rendering** — `draw()` redraws everything from scratch each frame. Call `draw()` after any state change that affects the visual. The draw order inside the world-space transform is:

```
drawClusterHalos() → drawEdges() → drawNodes() → drawLinkGhost()
```

The canvas context is translated/scaled to world coordinates inside `draw()`:
```
canvas pixel space → translate(width/2, height/2) → scale(camZoom) → translate(camX, camY) → world space
```

Node positions (`n.x`, `n.y`) are in world space. Use `worldToScreen()` / `screenToWorld()` to convert. Labels and text sizes that should appear constant on screen must be divided by `camZoom` (e.g. `fontSize = 11 / camZoom`).

### Node visuals

- **Stroke colour** — driven by `status` (gray/amber/green), not the node's `color`.
- **Fill** — driven by `color` via `COLOR_PALETTE`.
- **Mastery arc** — partial ring outside the node circle, from top going clockwise, covering `mastery/5` of circumference.
- **Status badge** — small filled circle at bottom-right of node, colour matches status.
- **Label** — rendered below the node circle (not inside), zoom-adaptive (`camZoom > 0.28` to show), full title never truncated.
- **Cluster halos** — `drawClusterHalos()` draws soft radial gradients behind nodes sharing a tag, colour from `tagToColor()`.

### Edge visuals

| Type | Style | Arrow |
|---|---|---|
| `related` | Solid, white/purple | Only when highlighted |
| `prerequisite` | Dashed, teal | Always (single chevron) |
| `improves` | Solid, amber, thicker | Always (double chevron) |

### Persistence

Data auto-saves to `localStorage` key `ideagraph_v1` on every mutation via `saveData()`. Google Drive sync is debounced 3 seconds after `saveData()`. Load order on startup: `loadData()` (localStorage) → if empty, `seedData()`.

### Key functions

| Function | What it does |
|---|---|
| `draw()` | Full canvas redraw |
| `updateSidebar()` | Rebuilds node list / study-order view, tag pills, status filter, calls `updateStudyNextCard()` |
| `selectNode(n)` | Sets `selectedNode`, opens detail panel, redraws |
| `openDetailPanel(n)` | Populates all detail panel sections for node `n` |
| `openModal(prefillTitle?, editId?)` | Opens create/edit modal; `editId` triggers edit mode |
| `saveModal()` | Reads form state and creates or updates a node |
| `createNode(title, desc, tags, color, x, y, status, mastery, resources, year)` | Adds node to `nodes[]`, saves, redraws. Pass `null` for `color` to auto-assign from tags. |
| `migrateData(d)` | Normalises raw JSON to current schema — call on every load |
| `enterLinkMode(src)` | Activates link-drawing mode; checks `prereqLinkMode`/`improvesLinkMode` for edge type |
| `getFilteredNodes()` | Returns nodes matching current search + tag + status filters |
| `tagToColor(tag)` | Deterministically maps a tag string → color id (never purple) |
| `buildPrereqMap()` | Returns `{ nodeId → [prerequisite nodeIds] }` from `prerequisite` + `improves` edges |
| `getTopologicalStudyOrder()` | Kahn's algorithm over the prereq graph; used by Study Order sidebar view |
| `getNextToStudy()` | Returns nodes with `status !== 'done'` whose all prereqs are `done`, sorted by connection count |
| `updateStudyNextCard()` | Refreshes the "Study Next" card in the sidebar |
| `fetchArxivMetadata(arxivId)` | Fetches title/abstract/year/tags from `export.arxiv.org` API |
| `scheduleArxivFetch()` | Debounced (800ms) auto-fetch triggered when an arXiv URL is detected in the resources field |
| `searchArxivPapers(query)` | Full-text arXiv search, returns up to 10 papers |
| `openPaperSearch()` / `closePaperSearch()` | Opens/closes the paper search modal (⌘P) |
| `addPaperToGraph(p)` | Creates a node from a search result with all metadata pre-filled |
| `autoLayout()` | Runs 200-iteration force-directed layout |

### arXiv integration

Two entry points:
1. **Resources field in modal** — paste an `arxiv.org/abs/…` URL and `scheduleArxivFetch()` auto-triggers, filling title, abstract, year, and suggesting tags.
2. **Paper search modal** (`⌘P` or toolbar `⌕`) — full-text search against arXiv, results rendered with add button; already-in-graph papers shown as `✓ In graph`.

arXiv categories are mapped to tags via `ARXIV_CAT_TAGS` constant (e.g. `cs.CV → computer-vision`).

### Google Drive

Configured via `GD_CLIENT_ID` constant at the top of the script. Uses GIS (Google Identity Services) OAuth 2.0 with `drive.file` scope. The token client is initialized in `initGoogleDrive()`, called once the GIS script loads. Drive sync saves `{ nodes, edges, nextId }` as `ideagraph_v1.json`. Both `loadData()` and `loadFromGDrive()` pass raw JSON through `migrateData()` before use.
