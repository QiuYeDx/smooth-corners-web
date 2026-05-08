# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`@lixiaolin94/smooth-corners` — Figma/iOS-style corner smoothing for CSS. Converts `radius + smoothing` into `border-radius` (compensated) + `corner-shape: superellipse(K)`.

## Tech Stack

Pure ESM, zero dependencies, zero build step. Type declarations are hand-written in `types/`. Published to npm as `@lixiaolin94/smooth-corners`.

## Development

```bash
npx serve .          # Preview demo (index.html)
```

No lint, no tests, no build. npm publish is handled by GitHub Actions on release.

## Architecture

Three API layers, all sharing `core.js` for the superellipse K calculation:

**css.js (primary)**: `smoothCorners(radius, smoothing?)` returns CSS custom properties (`--sc-r`, `--sc-i`, `--sc-s`) to pair with `.smooth-corners` class. Uses `@supports` for progressive enhancement. No element size needed.

**compute.js**: `computeSmoothCorners(width, height, radius, smoothing)` — dimension-aware version that clamps radius to `min(w,h)/2` and reduces smoothing when space is insufficient. Returns `{ radius, compensatedRadius, k, smoothing }`.

**observer.js**: DOM runtime using ResizeObserver + MutationObserver. Writes `--sc-r/i/s` custom properties + adds `.smooth-corners` class (never directly writes `border-radius`). Auto-injects `smoothCornersCSS` on first use. `declarative.js` is a side-effect import that calls `startAutoObserve()`.

## Core Algorithm

The problem: superellipse curves "inset" at the 45° apex compared to circular arcs. A superellipse with the same radius as a circle appears visually smaller. The compensated radius corrects this.

1. **Compensated radius**: `compensated = r × (1 + s)` (css.js) or `r + min(r·s, limit-r)` (compute.js)
2. **Superellipse exponent n**: `d = 1 - 1/√2`, `u = 1 - (r·d)/compensated`, `n = ln2/ln(1/u)`
3. **CSS K value**: `K = log₂(max(2, n))`

When K ≤ 1 (equivalent to circular arc), css.js falls back to original radius without superellipse output.

## npm Exports

```
"."            → src/index.js   (smoothCorners, applySmooth, smoothCornersCSS, computeSmoothCorners)
"./css"        → src/css.js
"./compute"    → src/compute.js
"./observer"   → src/observer.js (import has no side effects; auto-injects base styles on first use)
"./declarative"→ src/declarative.js (side-effect: starts DOM auto-observe)
```

## Key Design Decisions

- Observer writes CSS custom properties + class, never directly writes `border-radius`/`corner-shape` — avoids destroying host styles
- Observer tracks `addedClass` flag to only remove `.smooth-corners` if the library added it
- All public functions guard against NaN/Infinity inputs via `Number.isFinite`
- Non-ResizeObserver code paths use `requestAnimationFrame` batching (with `setTimeout` fallback)
- `smoothCornersCSS` base styles are auto-injected by observer on first use; CSS variable API users must inject manually
