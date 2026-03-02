# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
pnpm dev          # Start dev server
pnpm build        # Type-check with vue-tsc then build with vite
pnpm preview      # Preview production build
```

No test runner is configured in this project.

## Architecture

**sp-edge** is a parking lot designer built with Vue 3 + TypeScript + Vite, using [Konva.js](https://konvajs.org/) (via `vue-konva`) for canvas rendering.

### Key Concepts

- **PIXELS_PER_METER = 50** — all user-facing dimensions are in meters; internal canvas coordinates are meters × 50.
- State is persisted to `localStorage` under key `sp-edge-state-v1`.
- The app has two modes (controlled by `mode` ref in `App.vue`): `PARKING` to define the parking lot boundary, and `SLOT_AREA` to place parking slot groups inside it.

### Component Tree

```
App.vue                     — root; holds all state, localStorage I/O, canvas event handling
  └─ ParkingArea.vue        — renders the parking lot boundary rect and its children
       ├─ ParkingSlot.vue   — single slot shape (v-line for angled polygon, v-rect otherwise)
       ├─ ParkingSlotGroup.vue — a labeled group of ParkingSlot shapes
       └─ SlotArea.vue      — a draggable group of generated slots with rotate/edit/delete controls
  └─ SlotEdit.vue           — fixed-position HTML overlay panel for editing a SlotArea's params
```

### Canvas Structure (`App.vue` template)

- `v-stage` wraps everything; it is made draggable for panning. Scale and position are tracked in `configKonva`.
- `v-layer > v-rect (background)` — a large rect that captures mouse events for stage panning.
- `ParkingArea` renders inside the layer and receives the parking rect config plus all slot area data.
- `SlotEdit` is rendered **outside** the canvas as a regular DOM element, positioned with `fixed` CSS.

### Panning & Zoom

- **Scroll wheel** → zoom (updates `configKonva.scaleX/Y` and `textScale = 1/scale`).
- **Space + drag**, **Ctrl + drag**, or **middle-mouse drag** → pan the stage.
- `textScale` is propagated to all text/label elements so they stay a constant screen size when zoomed.

### SlotArea Geometry

`SlotArea.vue` generates parallelogram-shaped slots using `v-line` (polygon points). The `slotAngle` prop creates a horizontal offset (`slantOffset = slotHeight * tan(angle)`). `dragBoundFunc` constrains dragging within the parking boundary using rotated bounding box math.

### Data Flow

- All `parkingSlotAreas` state lives in `App.vue` as a `ref<any[]>`.
- Child components emit events (`delete`, `rotate`, `edit`, `update-position`) that bubble up through `ParkingArea` to `App.vue` handlers.
- `SlotEdit` emits `update-params` which `App.vue` handles in `handleUpdateSlotAreaParams`, recomputing `config.width` and `config.height` from slot count × pixel dimensions.
