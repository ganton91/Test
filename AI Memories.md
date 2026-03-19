# AI Memories

Αυτό το αρχείο ενημερώνεται αυτόματα από εμένα (AI) καθώς δουλεύουμε στο project.

**Κανόνας:** Ό,τι θεωρώ σημαντικό για την κατανόηση της λογικής του κώδικα — αρχιτεκτονικές αποφάσεις, κρίσιμα subsystems, μη προφανείς συνδέσεις μεταξύ τμημάτων — το προσθέτω εδώ χωρίς να χρειαστεί να μου το ζητηθεί. Αν αλλάξει κάποιο σύστημα, ενημερώνω ή διαγράφω αντίστοιχα.

---

## Main View Rendering

Το Main View χρησιμοποιεί το `render(flags)` → `flushRender()` pipeline με dirty flags και `requestAnimationFrame` scheduling.

**Dirty flags:**
- `uiDirty` → rebuilds UI (sidebar, panels, view switcher κ.λπ.) + καλεί `renderViewOutputs()`
- `staticDirty` → `drawStaticScene()` (grid, origin axes)
- `scenesDirty` → `drawSceneLayer()` (reference images/scenes)
- `contentDirty` → `drawContentScene()` (painted tiles) + καλεί `renderViewOutputs()`
- `overlayDirty` → `drawOverlayScene()` (brush ghost, selection highlight)
- `rulersDirty` → `drawRulers()`

**Canvas layers (από κάτω προς τα πάνω):**
- `staticCanvas` — grid και axes
- `sceneCanvas` — imported reference scenes
- `contentCanvas` — painted drawing content (tile-based)
- `overlayCanvas` — transient interaction visuals
- Ξεχωριστά: `rulerTop`, `rulerLeft`, `rulerBottom`, `rulerRight`

**Κανόνας:** Κανένας pointer handler δεν κάνει synchronous full redraw. Όλα περνούν από το dirty flag → rAF pipeline.

---

## Panels (View Outputs) Rendering

Τα View Panels χρησιμοποιούν τη `renderDirectionalViewOutput(canvas, emptyState, direction, placeholder, slotIndex, exportOverride)`, που καλείται από τη `renderViewOutputs()`.

**Κανονικό render (panel mode):**
- Το canvas αλλάζει μέγεθος με `resizeViewPaneCanvas()` στο CSS size
- Γίνεται render **μία φορά** σε `offscreenScale × CSS size` pixels
- `offscreenScale` = έως 8×, max canvas `MAX_OFFSCREEN_PX = 8192px`
- Zoom/pan στο panel = **pure CSS transform** — δεν ξανακάνει render
- AutoFit `padding = 64px` από κάθε πλευρά (tunable στο `SETTINGS.md`)

**Export mode (PNG/PDF):**
- Ξεχωριστό path όταν υπάρχει `exportOverride`
- `targetResolution = 4000px` (default)
- `exportPaddingPx = targetRes * 0.05` (5% margin)
- `ratio = 1` (no devicePixelRatio)

**Κανόνας:** Το panel render δεν συνδέεται με το Main View rAF pipeline. Triggered μόνο όταν αλλάζει το `uiDirty` ή `contentDirty`.

---

## Ground & Underground Rendering στα Panels

### Σειρά ζωγραφίσματος μέσα στη `renderDirectionalViewOutput`

1. Background fill (sky για sides, `planGroundColor` για plan)
2. Vector fills — painted non-cut κελιά (`buildViewVectorFillGroups`)
3. Per-layer outlines
4. Global outline
5. Section cut fills (isCut κελιά)
6. **Ground overlay** (sides) ή **Underground overlay** (plan < 0)

### Side Views — Ground

- Το ground ζωγραφίζεται **μετά** τα content fills, καλύπτοντας όλη την underground ζώνη (`undergroundStartRow` και κάτω)
- Χρησιμοποιεί `evenodd` fill: ένα μεγάλο `rect` + contour paths από το `groundHoleMask` για να "τρυπήσει" το χρώμα
- `groundHoleMask` (`buildGroundHoleMaskFromGrid`): `isCut` κελιά + κενά κελιά εγκλωβισμένα από (isCut + zero-cap row)
- Τα underground non-isCut κελιά ζωγραφίζονται στο step 2 αλλά **καλύπτονται** από το ground — δεν φιλτράρονται ρητά στο render path
- Ρητό φιλτράρισμα υπάρχει **μόνο στο DXF export** (`buildViewPaneDxfContent`, γρ. ~4857): nulls out `cellZ < 0` + `applyGroundMaskToVisibleGrid`

### Plan View — Ground & Underground

**Elevation >= 0 (Ground Plan Color):**
- ΔΕΝ υπάρχει πλέον flat background fill για plan direction
- Painted content ζωγραφίζεται κανονικά
- Ground Plan overlay: `buildGroundHoleMaskFromGrid(renderGrid, 0)` — ίδια λογική με underground
- Full-canvas `rect` με `evenodd` τρύπες → fills με `planGroundColor`
- Το φίλτρο `if (planeCells >= 0 && topCells <= 0) continue;` **έχει αφαιρεθεί** από το `buildPlanOcclusionGrid` → underground layers μπαίνουν στο grid, καλύπτονται από το overlay

**Elevation < 0 (Underground Plan Color):**
- Background = `planGroundColor`
- Painted content από πάνω
- Underground overlay: `buildGroundHoleMaskFromGrid(renderGrid, 0)` (ολόκληρο το grid = underground zone)
- Full-canvas `rect` με `evenodd` τρύπες → fills με `planUndergroundColor`
- Τρύπες = isCut κελιά + εγκλωβισμένα κενά (ίδια λογική με sides, χωρίς zero-cap row)

### Βασική διαφορά sides vs plan underground

| | Side Views | Plan (elevation < 0) |
|---|---|---|
| Flood fill boundary | isCut + zero-cap row (z=0 line) | isCut μόνο (no cap row) |
| `undergroundStartRow` | `contentMaxZ` (z=0 row) | `0` (ολόκληρο το grid) |
| Explicit pre-filter | Μόνο στο DXF, όχι στο render | Κανένας πλέον (φίλτρο αφαιρέθηκε) |
