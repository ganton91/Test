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
- Brush dimensions are stored internally in cells, but the Brush Settings UI exposes them in centimeters (`5 cm` per cell)
- Brush Settings includes a width/height link toggle:
  - when linked, width and height stay equal and update together
  - when unlinked, they can be edited independently
- Brush Settings is now a standalone floating panel below the global `Color` panel:
  - it is no longer attached to the `Brush` button
  - it uses the same collapsible header/arrow behavior as the `Color` panel
  - left-side floating panels should behave as one compact vertical stack, so lower panels move up automatically when upper panels collapse
- Left floating panels such as `Color` and `Brush Settings` should reuse the same header format as sidebar sections:
  - same arrow
  - same title typography
  - same compact header row treatment
- Brush color is managed through a custom color UI:
  - the color UI is now a global floating `Color` panel, not part of `Brush Settings`
  - curated palette swatches in a single row and stretched across the full palette row width
  - one swatch is always the active palette slot
  - synchronized `HSB` sliders update the active swatch itself, not just a temporary brush color
  - an eyedropper mode that samples only from painted cells on the canvas
  - the eyedropper also writes back into the active swatch
  - when eyedropper mode is active, the brush footprint preview is hidden and the browser pick cursor is used without any extra overlay marker
  - the `Color` panel is collapsible with the same arrow-toggle behavior used by the sidebar sections
- `Measurements` is now a third annotation subsystem alongside `Scenes` and `Layers`
- Measurements use the global `Color Palette` for new annotations:
  - each new point / length / area stores its own color at creation time
  - changing the global color affects the next measurement annotation, not existing ones
- Measurement cards currently support the first safe phase:
  - section + cards + shared active-state integration
  - visible point annotations
  - width and opacity controls
  - visible `Length` and `Area` annotations are now enabled
  - `Points` snap to the center of each `5 x 5 cm` cell
  - `Length` and `Area` snap on a half-cell grid (`2.5 cm`)
  - `Length` supports both click-drag-release and click-first / click-second workflows
  - `Area` is polygon-based and closes when the first point is clicked again
  - `Escape` should cancel an unfinished measurement draft before it exits measurement mode entirely
  - while a measurement is active, right-click drag acts as a measurement eraser and removes whole point/length/area annotations from the active measurement

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
- `Scenes`, `Layers`, and `Measurements` use one shared active-state model:
  - only one `Scene`, one `Layer`, or one `Measurement` can be active at a time
  - active `Layer` switches the app into `Brush`
  - active `Scene` switches the app into scene transform mode
  - active `Measurement` switches the app into measurement mode
  - `Escape`, clicking the same active card again, or manually choosing `Selection` clears all active state
- Card interaction pattern for `Scenes` and `Layers` is shared:
  - click on an inactive card activates it
  - click on the name of an already active card enters rename mode
  - click elsewhere on an already active card deactivates it
- In `Selection` mode, clicking a visible `Scene` on the canvas should select the topmost matching scene before checking painted layers underneath
- Selection hit priority must follow visual stacking:
  - check `Measurements` from topmost to bottommost first
  - check painted `Layers` from topmost to bottommost first
  - then check painted `Layers` from topmost to bottommost
  - only if no measurement or painted layer is hit should selection fall through to `Scenes`

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
- Inactive `Scene` and `Layer` cards collapse to a compact state:
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
