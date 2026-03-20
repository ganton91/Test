# AI Memories

Αυτό το αρχείο ενημερώνεται αυτόματα από εμένα (AI) καθώς δουλεύουμε στο project.

**Κανόνας περιεχομένου:** Ό,τι θεωρώ σημαντικό για την κατανόηση της λογικής του κώδικα — αρχιτεκτονικές αποφάσεις, κρίσιμα subsystems, μη προφανείς συνδέσεις μεταξύ τμημάτων — το προσθέτω εδώ χωρίς να χρειαστεί να μου το ζητηθεί.

**Κανόνας δομής:** Το αρχείο πρέπει να παραμένει καθαρό και δομημένο ανά πάσα στιγμή. Δεν επιτρέπονται: διπλές καταχωρίσεις, αντικρουόμενες πληροφορίες, ορφανές αναφορές, ή παλιά στοιχεία που έχουν αλλάξει. Όταν αλλάζει ένα σύστημα, ενημερώνω ή διαγράφω την αντίστοιχη ενότητα — δεν προσθέτω νέα δίπλα στην παλιά.

**Κανόνας ενημέρωσης (ΥΠΟΧΡΕΩΤΙΚΟ):** Όταν μια αλλαγή στον κώδικα εμπίπτει στους παραπάνω κανόνες περιεχομένου και δομής, ενημερώνω το AI Memories **αμέσως** ως μέρος της ίδιας εργασίας — όχι αφού μου υπενθυμίσει ο χρήστης. Τετριμμένες αλλαγές (π.χ. χρώματα, spacing, labels) δεν καταχωρίζονται.

**Κανόνας SETTINGS.md (ΥΠΟΧΡΕΩΤΙΚΟ):** Όταν εντοπίζω στοιχεία του κώδικα που αποτελούν ρυθμίσιμες παραμέτρους (τιμές, όρια, constants που ελέγχουν συμπεριφορά του προγράμματος), τα καταχωρίζω στο `SETTINGS.md` με αρχείο, μεταβλητή, τρέχουσα τιμή και τρόπο αναζήτησης — ώστε να βρίσκεται γρήγορα όταν χρειαστεί να αλλαχτεί.

**Κανόνας αναφοράς PROJECT_MEMORY.md:** Αν δυσκολεύομαι να βρω κάτι — συμπεριφορά subsystem, παλιά απόφαση, feature spec, invariant — τσεκάρω πρώτα το `PROJECT_MEMORY.md`. Είναι το ιστορικό αρχείο καταγραφής του project με λεπτομερή τεκμηρίωση όλων των subsystems.

**Αρχεία αναφοράς:** `TODO.md` = εκκρεμή features/migrations, `General Notes.md` = γενική φιλοσοφία και UX direction, `SETTINGS.md` = tunable parameters.

---

## Collaboration Rules

**Pre-Edit Confirmation Rule (ΥΨΗΛΗ ΠΡΟΤΕΡΑΙΟΤΗΤΑ):** Πριν από κάθε αλλαγή αρχείου:
1. Συνοπτική επανάληψη του τι κατάλαβα
2. Ο χρήστης επιβεβαιώνει
3. Μόνο τότε η αλλαγή

Εξαίρεση: αν ο χρήστης πει ρητά να προχωρήσω χωρίς επιβεβαίωση.

**Protected subsystems — προειδοποίηση πριν αλλαγή:**
- canvas event flow, renderer pipeline, tile/chunk logic
- dirty-flag scheduling, zoom math, pan math, ruler math
- brush-to-cell mapping, selection hit logic
- sharpness-related draw behavior, performance-related renderer architecture

Αν αγγίξω protected subsystem: πρώτα εξηγώ ποιο subsystem αφορά και ποιος είναι ο κίνδυνος (performance / sharpness / coordinate consistency / interaction regressions), μετά προχωράω.

**Safe changes (χωρίς προειδοποίηση):** naming, typography, labels, panel layout, spacing, iconography, cosmetic styling, static non-renderer UI structure. Αν ένα "UI" request αγγίζει renderer behavior, ανακατατάσσεται ως protected.

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
4. Section cut fills (`isCut` κελιά) — **πάντα ενεργό**, χωρίς On/Off toggle
5. **Ground overlay** (sides) ή **Underground overlay** (plan < 0)
6. Horizon line
7. **Global outline** — με visibility filter (`isCellVisibleAfterGround`): μόνο για ορατά κελιά
8. Section cut outline — **μόνο αυτό έχει On/Off toggle**
9. Per-layer outlines — **pass 2**: μόνο `excludedFromSectionCut` layers, με visibility filter

**Visibility helper `isCellVisibleAfterGround(r, c)`:** κελί θεωρείται ορατό αν είναι πάνω από το ground (side views: `r < undergroundStartRow`) ή μέσα στο hole mask (`groundHoleMask` / `planHoleMask`). Αν το ground overlay είναι ανενεργό, όλα τα κελιά θεωρούνται ορατά. Χρησιμοποιείται από Global Outline και pass 2.

**`planeElevation` + `planHoleMask` στο outer scope:** Και τα δύο υπολογίζονται πριν το `// === Ground ===` block, ώστε να είναι διαθέσιμα σε όλα τα επόμενα render steps (Global Outline, pass 2). Για side views έχουν τιμή `null`.

**`isCellVisibleAfterGround` για plan views:** Ελέγχει `planHoleMask[r][c] || (renderGrid[r][c] && renderGrid[r][c].depth > 0)` — ώστε τα outlines αντικειμένων κάτω από το cut plane αλλά πάνω από z=0 να μην κόβονται.

**`buildViewContoursFromGrid(renderGrid, predicate)`:** Το predicate δέχεται `(cell, rowIndex, columnIndex)` — όχι μόνο `(cell)`. Το `r` και `c` είναι απαραίτητα για visibility checks που εξαρτώνται από θέση (π.χ. `groundHoleMask[r][c]`).

**Global Outline — depth-awareness:** Το Global Outline δεν είναι απλό silhouette. Ζωγραφίζει δύο πράγματα:
- **Mass boundary**: εξωτερικό περίγραμμα όλων των non-cut κελιών (σχήμα)
- **Depth transitions**: γραμμές εκεί όπου γειτονικά non-cut κελιά έχουν διαφορετικό `depth` value — δηλ. φανερώνει τα "βάθη" της προβολής σαν ανάγλυφο

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
- Τρύπες = `planHoleMask` cells (isCutGeometry + εγκλωβισμένα κενά) **ΚΑΙ** cells με `depth > 0` (geometry πάνω από z=0 αλλά κάτω από το cut plane). Αυτό εξασφαλίζει ότι αντικείμενα χαμηλότερα από το cut level αλλά ορατά πάνω από το έδαφος δεν καλύπτονται από το planGroundColor.
- Το φίλτρο `if (planeCells >= 0 && topCells <= 0) continue;` **έχει αφαιρεθεί** από το `buildPlanOcclusionGrid` → underground layers μπαίνουν στο grid, καλύπτονται από το overlay

**Elevation < 0 (Underground Plan Color):**
- Background = `planGroundColor`
- Painted content από πάνω
- Underground overlay: `buildGroundHoleMaskFromGrid(renderGrid, 0)` (ολόκληρο το grid = underground zone)
- Full-canvas `rect` με `evenodd` τρύπες → fills με `planUndergroundColor`
- Τρύπες = `planHoleMask` cells **ΚΑΙ** cells με `depth > 0` — ίδια λογική με ground plan

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

**Section Cut On/Off:** Το Section Cut fill είναι **πάντα On** (`sectionCutEnabled = true` hardcoded). Δεν υπάρχει UI toggle γι' αυτό. Μόνο το **Section Cut Outline** έχει On/Off. *(Σημείωση: αν χρειαστεί να επαναφερθεί το On/Off για το fill, η διόρθωση είναι να αντικαταστήσεις `if (!cell || cell.isCut) continue;` στο `buildViewVectorFillGroups` με `if (!cell || (sectionCutEnabled && cell.isCut)) continue;` — έτσι τα isCut cells ζωγραφίζονται με το layer χρώμα τους όταν το section cut είναι Off.)*

---

## View Properties

Ανοίγει με "View Properties" button σε κάθε View card → `openViewPropertiesModal(viewId)`.

**Πεδία:**
- `sectionAxes` — custom section planes μέσα στο view box (βλ. παρακάτω)

**Read-only dimensions (modal, για reference):**
- **H** — auto-computed plan elevation: `(max(baseCells + heightCells) + 1) / CELLS_PER_METER` από τα configured layers του view
- **L** — view box X dimension σε μέτρα: `(cellBounds.maxX - cellBounds.minX + 1) / CELLS_PER_METER`
- **W** — view box Y dimension σε μέτρα: `(cellBounds.maxY - cellBounds.minY + 1) / CELLS_PER_METER`

**`planElevation`** — αφαιρέθηκε από το UI και από το render pipeline. Το `view.planElevation` field παραμένει στο data model για backwards compat με παλιά projects αλλά **αγνοείται**. Το `buildPlanOcclusionGrid` υπολογίζει το `planeCells` αυτόματα (max layer top + 1 cell) εκτός αν δοθεί `planElevationOverride` (για Z section axes). Επιστρέφει `planeElevation` στο return object — χρησιμοποιείται από `renderDirectionalViewOutput` για το ground overlay.

Αλλαγή trigger: `render({ ui: true, content: true, overlay: true })`

---

## Multi-Section Axes

**Data model:** `view.sectionAxes = { z: [{id, elevation}], x: [{id, fromSide, distance, direction}], y: [{id, fromSide, distance, direction}] }`

- Z sections: horizontal cuts σε συγκεκριμένο ύψος → render ως plan view με `planElevationOverride`
- X sections: vertical cuts κατά τον X άξονα → direction "leftToRight" ή "rightToLeft", `frontBoundaryOverride` computed από `fromSide` + `distance`
- Y sections: vertical cuts κατά τον Y άξονα → direction "topToBottom" ή "bottomToTop", `frontBoundaryOverride` computed από `fromSide` + `distance`

**Helper functions:**
- `generateSectionAxisId()` → `"sax-" + random`
- `isSectionAxisId(direction)` → true αν δεν είναι στο `STANDARD_VIEW_DIRECTIONS`
- `resolveSectionAxisInfo(view, axisId)` → `{ type, axis }` ή `null`
- `computeSectionFrontBoundary(sectionInfo, bounds)` → absolute cell coordinate
- `cloneSectionAxes / restoreSectionAxes` → για snapshot/restore
- `sectionAxisShortLabel(type, axis)` → e.g. `"Z2.4"`, `"X3→"`, `"Y5↓"` (για selector buttons)
- `sectionAxisFullLabel(type, axis)` → e.g. `"Z 2.40m"`, `"X 3m →"` (για title/export)
- `formatSectionValue(v)` → trim trailing zeros

**Render pipeline extensions:**
- `collectLayerProjectedColumns`: `options.frontBoundaryOverride` αντικαθιστά computed `frontBoundaryValue`. **Κρίσιμο:** όταν υπάρχει override, κελιά που βρίσκονται μπροστά από το section plane (πιο κοντά στον θεατή) **εξαιρούνται ρητά** πριν το occlusion test, αλλιώς θα κέρδιζαν το test και θα απέκρυπταν το section:
  - `frontBoundary === "min"` (π.χ. L→R): `depthValue < frontBoundaryValue → continue`
  - `frontBoundary === "max"` (π.χ. R→L): `depthValue > frontBoundaryValue → continue`
  - Κελιά ακριβώς στο `frontBoundaryValue` περνούν και flagγάρονται `isCut`/`isCutGeometry` κανονικά
- `buildDirectionalOcclusionGrid(view, direction, options)`: passes `frontBoundaryOverride` κάτω
- `buildPlanOcclusionGrid(view, options)`: `options.planElevationOverride` αντικαθιστά `view.planElevation`
- `renderDirectionalViewOutput`: αν `isSectionAxisId(direction)` → lookup axis → dispatch σωστά
  - Z section: `buildPlanOcclusionGrid` με `planElevationOverride`
  - X/Y section: `buildDirectionalOcclusionGrid` με axis direction + `frontBoundaryOverride`
  - `isPlanLike = isPlanDirection || (Z section)` → ελέγχει sky/ground/horizon rendering
- `buildViewPaneDxfContent`: ίδια λογική, χρησιμοποιεί `isPlanLike` αντί `isPlanDirection`

**UI:**
- View Properties modal: 3 dynamic groups (Z/X/Y Sections) κάτω από `planElevation`
  - Rendered via `renderViewPropertiesModalSections()`, called on open + add/delete
  - Edits mutate `state.viewPropertiesDraft.sectionAxes` inline
- Pane selector: dynamic buttons μετά το PL, rendered στο `renderViewWorkspaceControls`
  - Containers: `#viewPaneSectionButtons{0-3}` (div.view-pane-section-buttons, display:contents)
  - Click listeners added per-button during render
- `viewDirectionLabel(direction)`: αν section axis ID → lookup → `sectionAxisFullLabel`

**Main View overlay lines (`drawOverlayScene`):** Για κάθε view που έχει X ή Y section axes, σχεδιάζεται μέσα στο view box rectangle:
- X axis → κατακόρυφη διακεκομμένη γραμμή (`setLineDash([6, 5])`) στο `cellX * CELL_SIZE` world X, από `view.bounds.minY` έως `view.bounds.maxY`
- Y axis → οριζόντια διακεκομμένη γραμμή στο `cellY * CELL_SIZE` world Y, από `view.bounds.minX` έως `view.bounds.maxX`
- Κεντρικό γεμιστό τριγωνικό βέλος που δείχνει προς την `axis.direction` (leftToRight → 0, rightToLeft → π, topToBottom → π/2, bottomToTop → -π/2)
- Χρώμα: `--crosshair` αν active view, `--selection` αν inactive, opacity 0.75

**Validation:** `viewPaneDirections` serialization/restore δέχεται οποιοδήποτε string (no strict whitelist), fallback σε standard direction αν `null/undefined`. Ορφανά section IDs → graceful empty state στο render.
