# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

CHRONOS · Humanité — an interactive 3D Earth globe visualizing human history from -300,000 to today, built with Three.js. Zero npm dependencies: a single self-contained `index.html` plus external JSON data files.

- Live: https://chronos-humanite.vercel.app
- Repo: https://github.com/kevin-ets-code/chronos-humanite

## Commands

```bash
# Run locally (textures fail to load via file:// due to CORS — use a server)
npx serve .
# or
python -m http.server 8080

# Deploy: just push to main, Vercel auto-deploys
git push origin main
```

There is no build step, bundler, linter, or test suite — everything runs directly in the browser from `index.html`.

## Structure

- `index.html` — the entire app: HTML, CSS (`<style>`), and all JS (`<script>` blocks). ~1800 lines.
- `data/*.json` — event data, split by historical era (`01-prehistoire.json` … `13-xxie.json`), plus `categories.json` and `eras.json`.
- `textures/` — Earth day/night/clouds JPGs (loaded by Three.js).

## Architecture: the five script blocks in index.html

The page wires together five separate IIFEs (in source order), each owning one concern and communicating only through `window._*` globals (see below). When changing one, check whether it reads/writes a global another block depends on.

1. **Globe renderer** (~`L702-974`): sets up the Three.js renderer/scene/camera, loads the day/night/clouds textures, builds the Earth mesh with a custom GLSL day/night shader (`earthMat`), the cloud layer, atmospheric halo (two shader spheres), procedural starfield, and the render/rotation loop (`(function loop() {...})`).
2. **Sun control** (~`L1019-1041`): drag handle UI (`#sun-control` / `#sun-handle`) that recomputes `SUN_DIR` and writes it to `earthMat.uniforms.uSun`; also tracks `window._userPausedRotation`.
3. **Timeline slider** (~`L1052-1176`): drives `#tl-slider`, converts slider position to a year via breakpoints (`BP` / `sliderToYear`), updates the displayed year/era label, and pushes `uNightLights` into `earthMat`.
4. **Sprites + hover + filters** (~`L1246-1708`): the largest block. Loads all JSON data (`loadData`/`ERA_FILES`), builds one `THREE.Sprite` per event placed via `latLngToVec3`, fades sprite opacity in/out based on the current year (`getOpacity`/`smoothstep`), handles hover tooltips and HTML hit-zones overlaid on the canvas (`#sprite-overlay`), spreads overlapping events with `applyClusterOffsets`, and builds the category filter chips (`#filters-list`).
5. **Side panel** (~`L1735-1804`): opens `#side-panel` with an event's details (`openPanel`/`closePanel`) when a sprite is selected.

### Cross-block globals (`window._*`)

Because each block is an isolated IIFE, they share state exclusively via globals on `window`:

| Global | Set by | Meaning |
|---|---|---|
| `_scene`, `_camera` | renderer | Three.js scene/camera, used by other blocks for projections |
| `_earth`, `_earthMat` | renderer | Earth mesh and its `ShaderMaterial` (sprites are added as children of `_earth`; slider/sun write to `_earthMat.uniforms`) |
| `_hoverPause` | sprites block | true while hovering a sprite — pauses auto-rotation |
| `_userPausedRotation` | sun control | true once the user manually drags the globe — stops auto-rotation permanently |
| `_categories`, `_eras` | sprites block (`loadData`) | parsed `categories.json` / `eras.json`, read by timeline & panel for labels/colors |
| `selectedEvent` | sprites block (`selectEvent`) | currently selected event object, read by the filter logic to keep it visible |
| `getCatColor`, `getCatLabel`, `_closePanel` | sprites/panel blocks | helper functions exposed for cross-block use |

## GLSL shaders

The Earth uses a custom `ShaderMaterial` (`earthMat`, ~`L753-789`) that blends day and night textures based on the dot product of the surface normal and `uSun`, smoothstepped across the terminator, and modulated by `uNightLights` (driven by the timeline year — city lights only show in eras where they'd exist). The atmospheric halo is two more `ShaderMaterial` spheres (BackSide outer rim + FrontSide inner rim). Don't touch these uniforms (`uDay`, `uNight`, `uSun`, `uNightLights`) without understanding how the slider and sun control depend on them.

## Data — adding an event

Add an entry to the appropriate era file in `data/` (`01-prehistoire.json` … `13-xxie.json`, chosen by date — see `eras.json` for date ranges). Never hardcode event data in `index.html`; `loadData()` fetches and merges all era files plus `categories.json`/`eras.json` at startup.

Event fields:

| Field | Type | Meaning |
|---|---|---|
| `id` | string | Unique slug |
| `title` | string | Event name shown in the panel/tooltip |
| `year` | number | Start year (negative = BCE) |
| `yearEnd` | number | End year — used for fade-out timing (`getOpacity`); set equal to `year` for instantaneous events |
| `lat`, `lng` | number | Coordinates, converted to a 3D position via `latLngToVec3` |
| `cat` | string | Category id, must match an entry in `categories.json` |
| `region` | string | Display region name |
| `desc` | string | Description shown in the side panel |

## Categories (`data/categories.json`)

| id | label | color |
|---|---|---|
| `sc` | Science | `#5b9bd5` |
| `te` | Technologie | `#a8d46a` |
| `em` | Empires | `#c97a5b` |
| `po` | Politique | `#e08a5a` |
| `ph` | Philosophie | `#d4c06a` |
| `re` | Religion | `#9b7bc4` |
| `ar` | Art & Culture | `#7bc49b` |
| `gu` | Guerre | `#d46a6a` |
| `ex` | Exploration | `#6abed4` |

## Eras (`data/eras.json`)

`aube-humanite` (-300000 → -10000), `neolithique` (-10000 → -3500), `antiquite` (-3500 → -500), `periode-classique` (-500 → 500), `haut-moyen-age` (500 → 1000), `moyen-age` (1000 → 1400), `renaissance` (1400 → 1600), `ere-moderne` (1600 → 1800), `ere-contemporaine` (1800 → 1945), `monde-actuel` (1945 → 2100).

## Design system

```css
--ink:       #0d0c0a   /* background */
--parchment: #f5f0e8   /* light text */
--gold:      #c9a84c   /* accents */
```

Fonts: Playfair Display (titles), DM Mono, Cormorant Garamond (body/labels).

## Notes

- `index.html` is loaded via Three.js r128 from cdnjs — there is no local copy to update.
- The slider's year mapping is non-linear (`BP` breakpoints in the timeline block) to give more resolution to recent history; `sliderToYear` is duplicated in both the timeline and sprites blocks — keep them in sync if you change the breakpoints.
