# Milimetre Project Memory

## Purpose

This file is the working memory and guardrail document for the `Milimetre` project.
It is intended to preserve important product decisions, renderer invariants, and collaboration rules.

This document should be treated as authoritative unless the user explicitly changes direction.

## Product Identity

- Product name: `Milimetre`
- Core concept: infinite measured drawing canvas
- Base grid unit: each cell is `5 x 5 cm`
- UX direction: minimal, quiet, precise, compact controls, compact typography

## Core Interaction Model

- Default tool state is `Selection`
- Pressing `Escape` must always return the app to `Selection`
- Left click paints
- Right click erases
- Space + drag pans
- Mouse wheel zooms

## Layout Rules

- Left sidebar is docked, not floating
- The canvas begins immediately to the right of the sidebar
- The app uses rulers on all four sides of the canvas
- Ruler corner blocks should exist visually on all four corners

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
