# Milimetre Project Memory

## Purpose

This file is the working memory and guardrail document for the `Milimetre` project.
It is intended to preserve important product decisions, renderer invariants, and collaboration rules.

This document should be treated as authoritative unless the user explicitly changes direction.

## Memory Maintenance Rule

- Any significant product decision, UI system decision, renderer decision, workflow rule, or architectural change should be added to this file automatically as part of the work
- Codex should not wait for a separate reminder before updating this memory when a change is important enough to affect future work

## Product Identity

- Product name: `Milimetre`
- Core concept: infinite measured drawing canvas
- Base grid unit: each cell is `5 x 5 cm`
- UX direction: minimal, quiet, precise, compact controls, compact typography
- Avoid default helper/explanatory filler text in the UI unless the user explicitly asks for it
- Floating UI surfaces should use their own dedicated global tokens, separate from the shared `surface` / `bg-elevated` tokens used elsewhere

## Paint And Annotation Systems

- Brush dimensions are stored internally in cells, while `Brush Settings` exposes centimeters (`5 cm` per cell)
- `Brush Settings` includes a width/height link toggle:
  - linked = width and height stay equal
  - unlinked = width and height can be edited independently
- Brush supports `Shift` drafting:
  - axis lock during drag
  - sticky lock for that stroke
  - pressing `Shift` mid-stroke resets the lock anchor to the current pointer position
  - ghost preview follows the locked axis
  - `Shift + click` continues a straight line from the previous brush point for both paint and erase
- `Color` is a global floating color subsystem:
  - curated swatches in one row
  - one active swatch at all times
  - `HSB` sliders update the active swatch itself
  - the active swatch keeps stable live `HSV`
  - eyedropper samples only painted canvas cells and writes back into the active swatch
  - eyedropper hides the brush footprint preview and uses the browser pick cursor
- Left floating panels such as `Color`, `Brush Settings`, `Shape Settings`, and `Outline Settings` should:
  - reuse the same compact header language as sidebar sections
  - collapse with the same arrow behavior
  - behave like one compact vertical stack
- `Shape` is a first-class paint tool alongside `Brush`:
  - `Rectangular`
  - `Elliptical`
  - `Circular`
  - `Polygon`
  - `Rectangular`, `Elliptical`, and `Circular` support both drag and click-click workflows
  - `Rectangular + Shift` = true square
  - `Elliptical + Shift` = draw from center outward
  - `Circular` = true circle
  - `Circular + Shift` = center-to-radius
  - `Polygon` uses custom double-click detection, not native browser `dblclick`
  - `Polygon` closes by clicking the first point or by custom double-click
  - `Polygon` now shows a live preview segment while the pointer moves
  - `Polygon + Shift` applies the same axis lock behavior to the current segment as `Area`
  - `Escape` cancels the polygon draft
- `Outline Settings` is a separate global paint subsystem for `Brush` and `Shape`:
  - independent outline color palette
  - independent eyedropper
  - `Inside` / `Outside` / `No Fill`
  - width in cell-based integer steps
  - outline state is independent from the main fill color palette
- `Measurements` is a first-class annotation subsystem alongside `Scenes` and `Layers`
- New measurement annotations use the global `Color Palette` at creation time; later global color changes do not rewrite existing annotations
- Measurements:
  - support `Points`, `Length`, and `Area`
  - use width and opacity controls
  - snap on the shared half-cell grid (`2.5 cm`)
  - therefore `Points` can land on corners, side midpoints, and centers
  - `Length` supports both drag and click-click creation
  - `Length + Shift` axis-locks the active segment during both drag and click-click creation
  - `Area` is polygonal and closes by first-point click or custom double-click
  - `Area + Shift` axis-locks the current segment from the last vertex
  - right-click drag acts as a measurement eraser for the active measurement
  - `Escape` is hierarchical:
    - cancel the active draft first
    - then return to the measurement's `Select` state
    - only later fully deselect the measurement

## Core Interaction Model

- Default tool state is `Selection`
- Pressing `Escape` must always return the app to `Selection`
- Exception for brush eyedropper:
  - the first `Escape` exits only the eyedropper mode
  - a second `Escape` then follows the normal return-to-`Selection` behavior
- Left click paints
- Right click erases
- Space + drag pans
- Mouse wheel zooms
- `Scenes`, `Layers`, `Measurements`, and `Views` use one shared active-state model:
  - only one active target family should own selection at a time
  - activating an existing `Scene`, `Layer`, `Measurement`, or `View` from selection should prefer `Select` as the stable top-level tool state
  - `Select` is conceptually `Select + Transform`, not a separate pre-transform mode
  - existing `Layers`, `Scenes`, `Views`, and `Measurements` can expose transform controls while `Select` stays active
  - creating a new `Layer` still drops into `Brush`
  - creating a new `Measurement` still drops into the measurement authoring flow
  - `Escape`, clicking the same active card again, or manually choosing `Selection` clears all active state
- Card interaction pattern for `Scenes` and `Layers` is shared:
  - click on an inactive card activates it
  - click on the name of an already active card enters rename mode
  - click elsewhere on an already active card deactivates it
- In `Selection` mode, clicking a visible `Scene` on the canvas should select the topmost matching scene before checking painted layers underneath
- Selection hit priority must follow visual stacking:
  - check `Measurements` from topmost to bottommost first
  - check painted `Layers` from topmost to bottommost first
  - only if no measurement or painted layer is hit should selection fall through to `Scenes`
- Once a `Measurement` is active in `Select`:
  - the whole measurement can transform as a group:
    - move
    - `90°` rotate
    - flip
  - item-level transform is a second layer on top of that model:
    - single click keeps whole-measurement selection
    - double click can enter an individual measurement item
    - `Point` item = move only
    - `Length` item:
      - body = move whole length
      - endpoints = move that endpoint
    - `Area` item:
      - body = move whole area
      - vertices = move that vertex
  - when an individual `Length` or `Area` is active, its vertices should be treated as subobjects of that active item and respond to normal click-drag, not require a second double-click

## Layout Rules

- Left sidebar is docked, not floating
- The canvas begins immediately to the right of the sidebar
- The app uses rulers on all four sides of the canvas
- Ruler corner blocks should exist visually on all four corners
- Panel section headers should follow one compact shared pattern by default:
  - title on the left
  - compact action button on the right
  - tight row height and tight vertical padding
  - action symbols/icons inside those buttons should use a dedicated inner icon wrapper, not raw text directly in the button
  - if a new section header is added later, it should inherit this same visual system unless the user explicitly asks for a different treatment
- Sidebar sections can be independently collapsed:
  - clicking the section title toggles open/closed state
  - a small arrow at the left of the title indicates state
  - arrow points down when open and sideways when collapsed
- Inactive `Scene`, `Layer`, and `Measurement` cards collapse to a compact state:
  - only drag handle and name remain visible
  - `Hide` and `Delete` remain visible as compact inline controls in the name row
  - full controls/details otherwise appear only when the card is active
  - drag-and-drop must still work from the compact card state

## Modal Rules

- Modal behavior is global, not per-modal by default
- All true modals should open centered in the viewport
- The first click outside a modal should close only the modal and should not pass through to the canvas or other UI
- `Escape` should close the active modal before any lower-priority app shortcut runs
- Modal backdrop blur is currently disabled and should remain off unless the user explicitly asks for it
- If the user asks for a change to one modal, Codex should first ask whether that change is meant to become the global rule for all modals

## Ruler Rules

- Major ruler marks are at `1m`
- Intermediate ruler marks are at `50cm`
- Fine ruler marks are at `5cm` when zoom level allows
- Meter labels must remain visually stronger than centimeter labels
- Horizontal rulers may increase positively left-to-right in the normal mathematical way
- Vertical rulers must follow this convention:
  - positive values above `0`
  - negative values below `0`
- Intermediate centimeter labels between meter labels must be local-within-meter:
  - `5cm` through `95cm`
  - never cumulative labels such as `105cm`, `155cm`, `205cm`

## Renderer Status

The renderer is now a protected subsystem.

It has already been improved significantly for:
- responsiveness
- clarity
- sharper edges
- lower redraw cost

Because of that, renderer-related work must be handled with caution.

## Renderer Architecture

Current renderer shape in [index.html](/workspaces/Test/index.html):

- `staticCanvas`
  - renders static scene elements
  - currently includes grid and origin axes
- `sceneCanvas`
  - renders imported reference scenes/images
  - remains separate from painted layer content
- `contentCanvas`
  - renders painted drawing content
  - uses tile-based drawing
- `overlayCanvas`
  - renders transient interaction visuals
  - currently includes brush ghost and selection highlight
- `rulerTop`, `rulerLeft`, `rulerBottom`, `rulerRight`
  - render the rulers separately from the main canvas stack

## Renderer Data Model

- Painted content is not treated as one huge flat canvas
- Painted content is organized per layer
- Each layer stores content in `tiles`
- Each tile is a `TILE_SIZE x TILE_SIZE` block
- Each tile owns an offscreen canvas and a local pixel map
- Only visible tiles should be drawn to the content canvas
- Reference scenes/images are stored separately from painted layers
- Scene opacity, visibility, ordering, and future transforms should remain outside the tile paint model
- Scene transforms and transform handles should live in overlay/scene interaction logic, not in the paint tile model
- Active scene transform rules:
  - rotation remains free, but soft-snaps near `90°` increments
  - corner scaling keeps the original image aspect ratio locked
- Scenes can also be `locked`:
  - locked scenes can still be selected
  - locked scenes do not show transform controls
  - locked scenes do not allow move / scale / rotate
  - active locked scenes show a continuous red outline as the lock indicator
- Scene scale calibration is a separate workflow from transform:
  - user picks two points on the active scene
  - enters the real-world distance in meters
  - the scene rescales against the canvas world units without touching the paint tile model

## Renderer Scheduling Rules

- Rendering must remain scheduled through the existing `requestAnimationFrame` flow
- Avoid immediate synchronous full redraws in pointer handlers
- The render pipeline is currently split into distinct dirty regions:
  - `uiDirty`
  - `staticDirty`
  - `contentDirty`
  - `overlayDirty`
  - `rulersDirty`
- New work should preserve or improve this separation, not collapse it

## Sharpness Rules

- Canvas sharpness is an explicit requirement
- `imageSmoothingEnabled` should remain disabled where relevant
- Drawing positions for tiles and overlays should remain pixel-snapped where appropriate
- Any change that reintroduces blurred edges is considered a regression

## Renderer Invariants

The following must remain true unless the user explicitly approves a renderer redesign:

- Static scene rendering is separate from content rendering
- Content rendering is separate from overlay rendering
- Visible-area rendering is preferred over global full-scene iteration
- Tile-based painted content storage remains in place or is replaced only by something strictly better
- Pointer interactions should not trigger unnecessary UI rebuilds
- Zoom/pan behavior must remain stable and visually smooth
- Ruler math must stay aligned with canvas world coordinates

## View Outputs Decisions (March 2026)

- `View Outputs` now use a locked-aspect AutoFit policy:
  - a single uniform source scale is used for both axes (`uniformSourceScale`)
  - this replaced independent X/Y reduction and fixed vertical-only compression
  - fit still remains grid-quantized and pixel-snapped, so step-like size transitions are expected at thresholds

- `View Outputs` rendering path was upgraded toward vector workflows:
  - cell-by-cell fill was replaced by merged row runs (`buildViewRowRuns`) for faster drawing
  - contour extraction from grid geometry was added (`buildViewContoursFromGrid`)
  - optional contour debug overlay exists (`DEBUG_VIEW_VECTOR_CONTOURS`, default off)

- Shared geometry builder introduced for consistency between render and export:
  - `buildDirectionalOcclusionGrid(view, direction)` is now the canonical projected/occlusion grid source for side views

- DXF export was introduced for view panes (in addition to PNG):
  - each view pane now has `PNG` + `DXF` export actions
  - DXF uses meter units (`$INSUNITS = 6`) and writes classic `POLYLINE/VERTEX/SEQEND` entities for compatibility
  - exported geometry includes:
    - per-visible-layer projected contours (separate CAD layers)
    - section-cut contours (when enabled)
    - global outline/depth boundaries
    - per-layer outline boundaries
    - section-cut outline boundaries (when enabled)
    - horizon line (when enabled)
  - contour simplification removes collinear vertices to reduce 5 cm stair-stepping on straight segments
  - DXF now exports from the final visible pane composition (`VIEW_VISIBLE`) instead of per-source-layer decomposition
  - when `Section Cut` is off, DXF masks below-ground geometry (z < 0) to match what is visibly shown in the pane
  - boundary line export (global/layer/cut outlines) merges contiguous orthogonal 5 cm segments into longer lines for cleaner CAD output

- Important current limitation:
  - geometry is still derived from the 5 cm base grid model, so fully continuous (non-quantized) vector edges are not yet part of the system

## Protected Change List

Before changing any of the following, Codex must warn the user first and explain the risk:

- canvas event flow
- renderer pipeline
- tile or chunk logic
- dirty-flag scheduling
- zoom math
- pan math
- ruler math
- brush-to-cell mapping
- selection hit logic
- any sharpness-related draw behavior
- any performance-related renderer architecture

## Safe Change List

The following can usually be changed without warning first:

- naming
- typography
- labels
- panel layout
- spacing
- iconography
- cosmetic styling
- static non-renderer UI structure

If a “UI” request actually affects renderer behavior, it must be reclassified as protected work.

## Collaboration Rule

If the user asks for a change that touches protected renderer logic:

1. Warn first
2. Explain what subsystem is affected
3. Explain the likely risk:
   - performance
   - sharpness
   - coordinate consistency
   - interaction regressions
4. Only then proceed

## Pre-Edit Confirmation Rule

This rule is high priority.

Before making any file change, Codex must:

1. Briefly restate what it understood from the user's request
2. Let the user confirm that understanding
3. Only then make the edit

This applies generally, not only to renderer work.

Exception:
- If the user explicitly says to proceed without confirmation for a specific change, that explicit instruction can be followed

## Current Goal

The current goal is to keep building `Milimetre` as a strong, minimal infinite canvas editor without accidentally destabilizing the renderer foundation that is already working well.

## Recent UI Behavior

- In `Selection` mode, marquee selection now supports:
  - left drag = intersection
  - right drag = full containment
- Marquee hits follow the shared selection priority:
  - `Measurements`
  - `Layers`
  - `Scenes`
- If marquee selection finds multiple hits, a contextual picker appears near the cursor so the user can choose the target item.
- The shared canvas hint is now a global mode hint system:
  - `Drag to create view box`
  - `Pick two points to set scale`
  - measurement tool prompts
  - eyedropper prompt: `Click on a Painted Cell to pick a color`
  - future workflow hints should reuse this same shared component instead of adding ad hoc status pills

## Views Subsystem

- `Views` is now a first-class sidebar section below `Measurements`.
- New views are created from the `+` button in the left panel and immediately enter first-box draw mode on the main canvas.
- If the user cancels before completing the first rectangle, the empty pending view is removed completely.
- The canvas now includes a floating shared `Main View / View n` switcher:
  - `Main View` returns to the normal drawing canvas
  - each created view gets its own selectable tab in the same pill
  - the switcher is a floating canvas control, not a structural canvas header bar
- Entering a specific `View` opens the dedicated `View` workspace:
  - a dedicated orthographic workspace with layout presets:
    - `1 Side`
    - `2 Sides`
    - `4 Sides`
  - each visible pane can be assigned any direction independently:
    - `Top to Bottom`
    - `Bottom to Top`
    - `Left to Right`
    - `Right to Left`
  - duplicate directions are allowed across panes by design
  - `Main View` floating drawing tools and rulers are suppressed there where appropriate
- Active view boxes use their own transform mode on the main canvas:
  - move
  - free resize from corner handles
  - `Shift` while drawing or resizing a view box constrains it to a square
  - no rotation
- In `Selection` mode, views can be selected from their outline or label, not from the box interior.
- View cards show the view-box dimensions in meters directly below the card header.
- `Edit Layer Elevations` on a view card opens a modal that shows only the painted layers intersecting that view box.
- The modal stores per-view layer settings:
  - `Color`
  - `Opacity`
  - `Height`
  - `Start`
  - derived `End`
  - and view-specific layer reorder
- View rendering now supports a dedicated settings group:
  - `Depth Effect` on/off
  - `Depth Mode` = `Shadow` or `Fog`
  - `Depth Strength` as explicit `% overlay per depth cell`
  - `Outline` on/off with custom color
  - `Sky` on/off with custom color
  - `Ground` on/off with custom color
  - `Override View Colors` on/off
- Depth effect no longer changes layer colors through HSV manipulation:
  - it keeps the base visible color
  - then applies a per-depth-cell overlay
  - `Shadow` overlays toward black
  - `Fog` overlays toward the active sky color
- View outlines are depth-aware:
  - they react not only to empty space, but also to changes in the visible surface signature
  - the baseline `0` line is treated as part of the same outline visibility toggle
- When `Override View Colors` is off, views render the original painted cell colors from the main canvas.
- When `Override View Colors` is on, views use the per-view layer `Color` and `Opacity` values from `Edit Layer Elevations`.
- The current per-view opacity override is only opacity of the final visible layer winner in that projection column:
  - it does not reveal hidden geometry behind that layer
  - it blends against the already rendered view background instead

## Export And Import

- `Export / Import` is now a first-class project system.
- Exported project data includes the working project state:
  - `Scenes` including embedded scene image data
  - `Layers`
  - `Measurements`
  - `Views`
  - active ids / active tool
  - viewport
  - panel collapsed states
  - paint / shape / outline / view settings
  - remembered last-used layer authoring tool and measurement mode
- Export / Import is intended to restore the same working project state, not transient in-progress interaction:
  - no active drag gesture
  - no temporary draft stroke / polygon / measurement draft
  - no live eyedropper session
  - no imported undo/redo history stack
- `Views` also support direct per-pane `PNG` export from the pane header:
  - export is raster and matches the currently rendered pane
  - filenames include the active view name and pane direction
  - the pane export control is disabled when that pane has no rendered output
- When `Section Cut` is enabled in `Views`, `Ground` and `Horizon Line` render behind the view drawing instead of over it:
  - this preserves underground / below-grade geometry visibility in section-style views
  - when `Section Cut` is off, ground and horizon keep their normal foreground behavior
- `Section Cut Outline` depends on `Section Cut`:
  - if `Section Cut` is turned off, `Section Cut Outline` should automatically turn off too
  - the UI should not allow section outline to stay enabled by itself
- `Per Layer Outline` in `Views` now keeps its `outside` stroke when the neighboring visible cell is actually behind it in depth:
  - neighbor-based suppression should only block `outside` for true foreground occluders
  - this is intended to preserve roof/window/edge strokes in projections where deeper masses sit behind the outlined layer
- `Global Outline` and `Per Layer Outline` are intentionally both required because they solve different drawing problems:
  - `Global Outline` is mass-based / surface-based
  - it reads the merged visible masses of the view
  - it ignores layer identity when neighboring visible surfaces sit on the same level
  - therefore it gives the clean overall silhouette and major surface transitions without showing every internal layer seam
  - `Per Layer Outline` is object-based / layer-based
  - it is used selectively to emphasize specific layers as separate objects
  - if enabled on many layers, it will reveal more internal separations than the global outline
  - therefore the global outline should remain the clean whole-view language, while per-layer outline remains the targeted emphasis system

## Renderer Note

- Painted layer tiles on `contentCanvas` should be drawn from shared snapped tile boundaries:
  - `left/top` from the current snapped tile boundary
  - `right/bottom` from the next snapped tile boundary
  - draw size = next boundary minus current boundary
- This is preferred over using a separately rounded shared `tileSpan`, because the boundary-based approach prevents visible tile seam artifacts during zooming and at reduced opacity.
- Original state of this fix:
  - `snapPixel(value)` used plain CSS-pixel snapping with `Math.round(value)`
  - the seam fix depended on boundary-based tile drawing only
- Current tuning layered on top of that:
  - `snapPixel(value)` is now device-pixel aware:
    - `const ratio = window.devicePixelRatio || 1`
    - `return Math.round(value * ratio) / ratio`
  - the shared-boundary tile drawing model stays the same
  - only the snap precision changed, so future seam regressions on other browsers / browser zoom modes can be compared first against this exact `snapPixel()` change
