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
