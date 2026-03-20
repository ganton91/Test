# AI Memories

Αυτό το αρχείο ενημερώνεται αυτόματα από εμένα (AI) καθώς δουλεύουμε στο project.

**Κανόνας περιεχομένου:** Ό,τι θεωρώ σημαντικό για την κατανόηση της λογικής του κώδικα — αρχιτεκτονικές αποφάσεις, κρίσιμα subsystems, μη προφανείς συνδέσεις μεταξύ τμημάτων — το προσθέτω εδώ χωρίς να χρειαστεί να μου το ζητηθεί.

**Κανόνας δομής:** Το αρχείο πρέπει να παραμένει καθαρό και δομημένο ανά πάσα στιγμή. Δεν επιτρέπονται: διπλές καταχωρίσεις, αντικρουόμενες πληροφορίες, ορφανές αναφορές, ή παλιά στοιχεία που έχουν αλλάξει. Όταν αλλάζει ένα σύστημα, ενημερώνω ή διαγράφω την αντίστοιχη ενότητα — δεν προσθέτω νέα δίπλα στην παλιά.

**Κανόνας ενημέρωσης (ΥΠΟΧΡΕΩΤΙΚΟ):** Όταν μια αλλαγή στον κώδικα εμπίπτει στους παραπάνω κανόνες περιεχομένου και δομής, ενημερώνω το AI Memories **αμέσως** ως μέρος της ίδιας εργασίας — όχι αφού μου υπενθυμίσει ο χρήστης. Τετριμμένες αλλαγές (π.χ. χρώματα, spacing, labels) δεν καταχωρίζονται.

**Κανόνας SETTINGS.md (ΥΠΟΧΡΕΩΤΙΚΟ):** Όταν εντοπίζω στοιχεία του κώδικα που αποτελούν ρυθμίσιμες παραμέτρους (τιμές, όρια, constants που ελέγχουν συμπεριφορά του προγράμματος), τα καταχωρίζω στο `SETTINGS.md` με αρχείο, μεταβλητή, τρέχουσα τιμή και τρόπο αναζήτησης — ώστε να βρίσκεται γρήγορα όταν χρειαστεί να αλλαχτεί.

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
3. Per-layer outlines — **pass 1**: μόνο layers που ΔΕΝ είναι `excludedFromSectionCut`
4. Section cut fills (`isCut` κελιά)
5. **Ground overlay** (sides) ή **Underground overlay** (plan < 0)
6. Horizon line
7. **Global outline** — με visibility filter (`isCellVisibleAfterGround`): μόνο για ορατά κελιά
8. Section cut outline
9. Per-layer outlines — **pass 2**: μόνο `excludedFromSectionCut` layers, με visibility filter

**Visibility helper `isCellVisibleAfterGround(r, c)`:** κελί θεωρείται ορατό αν είναι πάνω από το ground (side views: `r < undergroundStartRow`) ή μέσα στο hole mask (`groundHoleMask` / `planHoleMask`). Αν το ground overlay είναι ανενεργό, όλα τα κελιά θεωρούνται ορατά. Χρησιμοποιείται από Global Outline και pass 2.

### Side Views — Ground

- Το ground ζωγραφίζεται **μετά** τα content fills, καλύπτοντας όλη την underground ζώνη (`undergroundStartRow` και κάτω)
- Χρησιμοποιεί `evenodd` fill: ένα μεγάλο `rect` + contour paths από το `groundHoleMask` για να "τρυπήσει" το χρώμα
- `groundHoleMask` (`buildGroundHoleMaskFromGrid`): `isCutGeometry` κελιά + κενά κελιά εγκλωβισμένα από (isCutGeometry + zero-cap row)
- Τα underground non-isCut κελιά ζωγραφίζονται στο step 2 αλλά **καλύπτονται** από το ground — δεν φιλτράρονται ρητά στο render path
- Ρητό φιλτράρισμα υπάρχει **μόνο στο DXF export** (`buildViewPaneDxfContent`, γρ. ~4857): nulls out `cellZ < 0` + `applyGroundMaskToVisibleGrid`

### Plan View — Ground & Underground

**Elevation >= 0 (Ground Plan Color):**
- ΔΕΝ υπάρχει flat background fill για plan direction
- Painted content ζωγραφίζεται κανονικά
- Ground Plan overlay: `buildGroundHoleMaskFromGrid(renderGrid, 0)` — ίδια λογική με underground
- Full-canvas `rect` με `evenodd` τρύπες → fills με `planGroundColor`
- Το φίλτρο `if (planeCells >= 0 && topCells <= 0) continue;` **έχει αφαιρεθεί** από το `buildPlanOcclusionGrid` → underground layers μπαίνουν στο grid, καλύπτονται από το overlay

**Elevation < 0 (Underground Plan Color):**
- Background = `planGroundColor`
- Painted content από πάνω
- Underground overlay: `buildGroundHoleMaskFromGrid(renderGrid, 0)` (ολόκληρο το grid = underground zone)
- Full-canvas `rect` με `evenodd` τρύπες → fills με `planUndergroundColor`
- Τρύπες = `isCutGeometry` κελιά + εγκλωβισμένα κενά (ίδια λογική με sides, χωρίς zero-cap row)

### Βασική διαφορά sides vs plan underground

| | Side Views | Plan (elevation < 0) |
|---|---|---|
| Flood fill boundary | `isCutGeometry` + zero-cap row (z=0 line) | `isCutGeometry` μόνο (no cap row) |
| `undergroundStartRow` | `contentMaxZ` (z=0 row) | `0` (ολόκληρο το grid) |
| Explicit pre-filter | Μόνο στο DXF, όχι στο render | Κανένας (φίλτρο αφαιρέθηκε) |

---

## Layer Properties (per-View Layer Config)

Ανοίγει με "Layer Properties" button σε κάθε View card → `openViewEditModal(viewId)`.
Δείχνει μόνο τα layers που τέμνουν το view box (`intersectingLayersForView`).
Η σειρά layers αποθηκεύεται στο `view.layerOrder[]` — ανεξάρτητη per-view σειρά.

**Πεδία per-layer** (αποθηκεύονται στο `view.layerConfigs[layerId]`):

| Πεδίο | Τύπος | Default | Περιγραφή |
|---|---|---|---|
| `baseElevation` | meters, snapped | `0` | έναρξη layer |
| `height` | meters, non-negative, snapped | `0` | ύψος layer |
| `color` | string \| null | `null` → representative color | override χρώμα για αυτό το view |
| `opacity` | 0–1 | `1` | opacity override |
| `outline` | bool | `false` | per-layer outline on/off |
| `outlineColor` | string | global outline color | custom outline color |
| `outlineWidth` | int 1–12 | `1` | πλάτος outline |
| `excludeFromSectionCut` | bool | `false` | βλ. παρακάτω |

### `excludeFromSectionCut`

Στο UI: checkbox **"include in section cut"** με αντεστραμμένη λογική (`checked = !excludeFromSectionCut`).

Κάθε grid cell φέρει **δύο ξεχωριστά flags**:

| Flag | Τι σημαίνει | Ποιος το διαβάζει |
|---|---|---|
| `isCut` | γεωμετρικά κομμένο **και** δεν έχει εξαιρεθεί | Section fill coloring |
| `isCutGeometry` | γεωμετρικά κομμένο, **αγνοεί** το `excludeFromSectionCut` | Ground / Underground hole mask |
| `excludedFromSectionCut` | το layer είναι excluded — σε **όλα** τα κελιά του, όχι μόνο τα cut-boundary | Per-layer outline pass 2 (rendering μετά το Ground) |

**Side Views** (`collectLayerProjectedColumns`):
```
isCut:         !excludeFromSectionCut && depthValue === frontBoundaryValue
isCutGeometry: depthValue === frontBoundaryValue
```

**Plan View** (`buildPlanOcclusionGrid`):
```
isCut:         !excludeFromSectionCut && planeCells >= baseCells && planeCells < topCells
isCutGeometry: planeCells >= baseCells && planeCells < topCells
```

**`buildGroundHoleMaskFromGrid`** διαβάζει **`isCutGeometry`** — όχι `isCut` — ώστε το ground overlay να ανοίγει τρύπες ακόμα και σε layers που έχουν εξαιρεθεί από το section fill (π.χ. παράθυρα υπογείου).

**Κανόνας:** `excludeFromSectionCut` επηρεάζει μόνο το section cut fill color — δεν επηρεάζει την ορατότητα, τα outlines, ή τους υπολογισμούς ground/underground.

---

## View Properties

Ανοίγει με "View Properties" button σε κάθε View card → `openViewPropertiesModal(viewId)`.

**Μοναδικό πεδίο:** `planElevation` (meters, snapped) — το ύψος στο οποίο ο `Plan` pane κόβει οριζόντια το μοντέλο.
- Αποθηκεύεται στο `view.planElevation`
- Χρησιμοποιείται από `buildPlanOcclusionGrid`: `planeCells = Math.round(planElevation * CELLS_PER_METER)`
- Αλλαγή trigger: `render({ ui: true, content: true, overlay: true })`
