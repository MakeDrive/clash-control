# ClashControl
> Version: **v3.3.11** (2026-03-27)

**Free, open-source IFC clash detection — right in your browser.**

No installs. No licenses. No subscriptions. Just open the file and start checking your models.

## What is it?

ClashControl is a lightweight web app for BIM coordination. Load your IFC models, detect geometric clashes between building elements, create and manage issues, and export everything to BCF — all without leaving your browser.

Built for architects, engineers, and BIM coordinators who are tired of paying thousands for clash detection software that does the same thing.

## Features

- **Load multiple IFC models** — drag & drop or browse, supports any IFC 2x3/4 file
- **Geometric clash detection** — hard clashes (intersections) and soft clashes (clearance violations) using a sweep-and-prune + BVH triangle test pipeline
- **3D viewer** — orbit, pan, zoom, section planes, section boxes, floor plan cuts, measurement tools
- **Model explorer** — browse elements by storey, IFC type, discipline, or material with visibility toggles and color-by-classification
- **Issue management** — create issues linked to elements, set priority/status/category, assign to team members
- **BCF import/export** — standard BCF 2.1 format for interoperability with other BIM tools
- **Viewpoints** — save and restore camera positions with snapshots
- **Dark & light mode** — full theme support
- **Works offline** — PWA with service worker caching, no server required
- **Zero dependencies** — single HTML file, no build step, no node_modules

## What's new

- **Discipline-colored outlines** — element outlines now match the model category color (structural = blue, MEP = red, architectural = purple, civil = green) when selecting or inspecting clashes
- **Smarter soft clash markers** — markers are placed at the actual closest point between elements, weighted toward the smaller element so they no longer appear in the middle of a long beam
- **Detection status in chat** — when clash detection is running with the chat panel open, the input area shows a live status bar with a glowing animated border
- **Clearance & Tolerance tooltips** — hover over the labels for 1 second to see what each setting does
- **Cleaner clash cards** — removed penetration depth display from hard clash cards for a less cluttered list

## How to use

1. Open `index.html` in a modern browser (Chrome, Edge, Firefox)
2. Load two or more IFC models via the sidebar
3. Configure clash rules (model A vs B, hard/soft, clearance distance)
4. Hit **Run** — clashes appear in the right panel
5. Click a clash to fly to it, inspect element properties, change status
6. Export to BCF when done

## Why free?

BIM coordination shouldn't be locked behind expensive licenses. ClashControl gives every project team access to clash detection, regardless of budget. No more paying a lot of money for a license to just do your job.

Of course donations to keep the project running and expand more are welcome.

## Important

ClashControl is not complete nor perfect and you should always verify results yourselves. That said, it will save you lots of money and frustration.

## Tech

Single-file app built with Preact, Three.js, and web-ifc. No build tools, no bundler — just open and go. See [CLAUDE.md](CLAUDE.md) for architecture details.

## How clash detection works

The clash engine lives in `index.html` and is centered around these functions: `_sweepAndPrune()`, `_buildBVHNode()`, `_bvhTraverseAll()`, `_triTriTest()`, `_meshMinDist()`, `_detectClashesCore()`, `detectClashesAsync()`, and `detectClashes()`.

### Detection pipeline

1. **Broad phase (`_sweepAndPrune`)**  
   It gathers the candidate elements from the selected model groups, chooses the axis with the highest AABB-center variance, sorts by min bound on that axis, and sweeps an active set. This replaces an `O(n²)` pair loop with an `O(n log n + k)` candidate pass.
2. **Hard clash path (`_getBVH` + `_bvhTraverseAll` + `_triTriTest`)**  
   Each candidate element lazily gets a triangle BVH built from world-space triangles. Detection then traverses the two BVHs together, pruning non-overlapping node AABBs until it reaches leaf triangles. Exact hits are confirmed with the Möller triangle-triangle intersection test.
3. **Soft clash path (`_meshMinDist`)**  
   If there is no hard clash and a clearance is configured, the engine uses a spatial hash grid over world-space vertices to estimate the minimum vertex-to-vertex distance and flags a soft clash when that distance is within the configured gap.
4. **Post-processing (`_detectClashesCore`)**  
   The core loop applies rule filters such as self-clash selection, excluded IFC type pairs, optional semantic suppression, duplicate detection, per-type-pair tolerances, merge of nearby segment clashes, and async chunking via `detectClashesAsync()` to keep the UI responsive.

### Why it was implemented this way

- It fits the single-file, zero-build architecture: no extra runtime dependencies are needed beyond the existing Three.js data structures and browser typed arrays.
- Sweep-and-prune removes most impossible pairs early, so the exact triangle test only runs on a much smaller subset.
- The BVH keeps hard-clash checks precise without testing every triangle against every triangle.
- The soft-clash path is intentionally cheaper than full mesh-to-mesh distance, which keeps clearance checks responsive on large IFC models.

### How to separate it into a clash detection service

The cleanest extraction is to move the geometry-specific helpers plus `_detectClashesCore()` into a standalone module or service object that accepts a neutral geometry payload and returns plain clash records.

Suggested service boundary:

```js
detectClashesService({
  models,   // [{ id, discipline, elements: [{ expressId, box, props, meshes? }] }]
  rules,    // same rule object used today
  onProgress
}) => clashes
```

Recommended split:

1. **Adapter layer**: keep IFC parsing and scene/Three.js setup in the app, but normalize each element into the shape already expected by the detector (`expressId`, `box`, `props`, world-space verts/tris or a way to build them).
2. **Pure detection layer**: move the clash helpers and `_detectClashesCore()` into a reusable service file. This layer should not know about UI state, reducers, panels, or Preact components.
3. **Integration layer**: keep `detectClashesAsync()` as the browser-facing wrapper that reports progress and cancellation, or replace it with a worker-based wrapper if the service is ever moved off the main thread.

In practice, the extraction risk is low because the detector is already mostly written as pure functions over element geometry plus rule settings.

### Would it tend to work with That Open Company's `.frag` format?

- **As the repository exists today: no direct support.** ClashControl currently imports IFC and the codebase itself already notes a TODO to consider Fragments if the project moves away from the current single-file architecture.
- **As a detection algorithm: yes, with an adapter.** The clash engine is mostly geometry-format agnostic. If a `.frag` loader can expose each fragment/element as world-space triangles or vertices plus bounding boxes and metadata, the existing broad phase, BVH traversal, and soft-clash logic can run on that data.
- **Main integration caveats:** That Open Fragments is designed around an ESM/bundled setup, has its own loading/runtime expectations, and would likely require version alignment work with the app's current Three.js setup before becoming a first-class input format here.

## License

See [LICENSE](LICENSE) for details.
