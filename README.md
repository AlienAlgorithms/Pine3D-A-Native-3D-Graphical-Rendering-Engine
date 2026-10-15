# Pine3D

A 3D rendering engine for TradingView, written in Pine Script v6: meshes, a perspective camera, directional and point lights with shadows, contour surfaces, 3D bar charts, streamed trails and scatter clouds, all drawn natively on the chart.

[![Pine3D on TradingView](https://s3.tradingview.com/i/I2K4hIdP.png?v=1791548529)](https://www.tradingview.com/script/I2K4hIdP-Pine3D-A-Native-3D-Graphical-Rendering-Engine/)

- **TradingView publication:** [Pine3D: A Native 3D Graphical Rendering Engine](https://www.tradingview.com/script/I2K4hIdP-Pine3D-A-Native-3D-Graphical-Rendering-Engine/)
- **Documentation:** [docs.alienalgorithms.com](https://docs.alienalgorithms.com/Pine3D-Graphics-Engine-Tradingview)
- **Product page:** [alienalgorithms.com/pine3d](https://alienalgorithms.com/pine3d)
- **Version:** 5, published 2026-10-09. See [CHANGELOG.md](CHANGELOG.md).
- **Licence:** CC BY-NC-SA 4.0, free for non-commercial use with credit. See [Licence](#licence).

## Use it in a script

```pine
//@version=6
indicator("My 3D Scene", overlay = false, max_polylines_count = 100, max_lines_count = 500, max_labels_count = 500)
import Alien_Algorithms/Pine3D/5 as p3d
```

Pine Script loads libraries from TradingView, not from GitHub. The import line above uses the published library; the source in this repository is the same code, here to read, search and reference. When Pine3D is updated on TradingView, the version number in the import line goes up.

## Coding with an AI assistant

Pine Script is rare in AI training data, so assistants tend to guess library calls. Give the assistant [REFERENCE.md](REFERENCE.md) before it writes Pine3D code. It documents the calls with their parameters, the rules for correct scripts and complete examples. The same file is online at https://alienalgorithms.com/pine3d.md.

## In this repository

| File | What it is |
|---|---|
| [Pine3D.pine](Pine3D.pine) | The library source, version 5, as published on TradingView |
| [REFERENCE.md](REFERENCE.md) | The whole API in one file, for people and AI assistants |
| [CHANGELOG.md](CHANGELOG.md) | What each version changed |
| [LICENSE](LICENSE) | CC BY-NC-SA 4.0 legal text |
| [examples/my-first-3d-scene.pine](examples/my-first-3d-scene.pine) | My First 3D Scene |
| [examples/contour-surface.pine](examples/contour-surface.pine) | Pine3D - Contour Surface |
| [examples/trail3d.pine](examples/trail3d.pine) | Pine3D - Trail3D |
| [examples/bars3d.pine](examples/bars3d.pine) | Pine3D - Bars3D |
| [examples/scatter-cloud.pine](examples/scatter-cloud.pine) | Pine3D - Scatter Cloud |
| [examples/pine3d-with-pinecanvas.pine](examples/pine3d-with-pinecanvas.pine) | Pine3D with PineCanvas |

---

**Pine3D** is a full 3D rendering engine for TradingView, powered by Pine Script™ v6.

Pine3D pushes forward the frontier of TradingView 3D rendering capabilities, providing a fully fledged graphical engine under an intuitive, chainable, object oriented API. Build meshes, transform them in world space, light them, cast shadows, project them through a perspective camera, and render the result directly on your chart, all without ever bothering about trigonometry synchronization or optimization.

The library brings forth a streamlined process for anyone that wishes to visualize data in 3D, without needing to know anything about the complex math that has previously gatekept such indicators. Pine3D does all the heavy lifting, including extreme optimization techniques designed for production ready indicators.

[![Pine3D on a TradingView chart](https://s3.tradingview.com/snapshots/v/vL3vg2q4.png)](https://www.tradingview.com/x/vL3vg2q4/)

The entire API is chainable and tag addressable, so spawning a mesh, registering it, pointing the camera at it, and rendering the frame is a four line affair:

```pine
p3d.Mesh mybox = p3d.cube(40.0, color.orange).setTag("hero").rotateBy(0.0, 45.0, 0.0)
scene.add(mybox)
scene.lookAt("hero")
p3d.render(scene)
```

## Surfaces: contour band rendering

Pine Script imposes a hard ceiling of **100 polylines and 500 lines per indicator**. On the surface this looks fatal for dense 3D meshes: every triangle drawn naively burns one of those 100 slots, or two of the 500, and the budget evaporates within a few hundred faces.

The conventional escape hatch is **strip stitching**, tracing a polyline forward along one row of a grid and back along the next, packing a ribbon of quads into a single drawing slot. It buys a meaningful multiplier, but it pays for that multiplier with two structural constraints baked into the geometry itself:

- **One color per strip.** A polyline carries a single stroke and fill color, so every cell along the ribbon must share the same shade. The moment you want per cell lighting, contour banding, or value driven gradients, every color change forces a new polyline and the budget collapses.
- **One contiguous ribbon per slot.** Strips can only describe topologically connected runs of cells. Disjoint regions, holes, islands, and value clustered fragments scattered across the surface each demand their own polyline.

**Pine3D breaks both constraints at once.**

At the core of the engine sits an innovation that redefines the limits for visual fidelity: **contour band rendering using degenerate bridge stitching**. The technique quantizes a surface's elevation into colored bands, then collapses every cell that falls inside the same band, *no matter where it sits on the screen*, into one continuous, hole aware polyline path per band, threading invisible zero width bridges between disjoint islands so that a single polyline can carry thousands of polygon equivalent fragments scattered across the geometry.

The result:

- A single polyline can render up to **2,000 disconnected triangle equivalents**, spread across arbitrarily separated regions of the surface.
- Theoretical ceiling of around **200,000 disconnected faces** inside the 100 polyline budget, a regime that strip based stitching cannot enter at any color count above one.
- A **40 x 40 heightmap** (around 3,000 triangles) renders inside the budget with full per band contour coloring and room to spare. Stress harnesses have run **40 x 80 grids**.
- Each band's path is depth sorted and near plane culled, and **cached between bars**, so once geometry is built only the screen space projection runs per frame.

[![Pine3D: Surfaces: contour band rendering, on a TradingView chart](https://s3.tradingview.com/snapshots/l/luknFNTj.png)](https://www.tradingview.com/x/luknFNTj/)

This algorithm enables scenes with extreme detail relative to the 100 polyline limit, and shifts the optimization focus from "drawing limits" to "CPU limits", which Pine3D natively handles with aggressive caching at every layer of the pipeline. The contour technique is currently integrated into the `surface()` function, with the same compression strategy generalizable to any mesh class and ultimately full scene rendering in future versions.

**Non-uniform grids out of the box.** `surface()` accepts optional `axisX` and `axisZ` arrays that override the default uniform spacing with custom column and row positions. This means logarithmic strike spacing on an option volatility surface, irregular timestamp spacing on a market depth heatmap, or any other non-evenly-sampled grid renders correctly without resampling the data first. The contour band engine, axis ticks, and gridBox cage all snap to the custom positions automatically.

A full contour surface is just a handful of lines; the damped ripple below builds once and never needs updating:

```pine
//@version=6
indicator("Pine3D - Contour Surface", overlay = false, max_polylines_count = 100, max_lines_count = 500, max_labels_count = 500)
import Alien_Algorithms/Pine3D/5 as p3d

var p3d.Scene scene   = p3d.newScene()
var p3d.Mesh  heatmap = na

if barstate.isfirst
    // Damped cosine ripple
    int N = 20
    matrix<float> data = matrix.new<float>(N, N, 0.0)
    for r = 0 to N - 1
        for c = 0 to N - 1
            float dx = c - (N - 1) / 2.0
            float dz = r - (N - 1) / 2.0
            float d  = math.sqrt(dx * dx + dz * dz) * 0.7
            data.set(r, c, math.cos(d) * math.exp(-d * 0.12) * 50.0)

    heatmap := p3d.surface(data, 200.0, color.blue, color.red, 24)
      .gridBox()
      .gridLabels(color.white, "X", "Amplitude", "Z")
    scene.add(heatmap)
    scene.camera.orbit(35.0, 25.0, 380.0)

if barstate.islast
    p3d.render(scene, lighting = true)
```

## Trail3D: streamed oscillator paths

**Trail3D** is a first class streaming primitive built for visualizing two correlated time series as a 3D ribbon evolving through time. You give it a rolling buffer capacity and push (u, v) samples bar by bar; the primitive maintains the buffer, builds the ribbon geometry, and renders it inside a normalized bounding cube so the path always fits cleanly in view regardless of the underlying data range.

[![Pine3D: Trail3D: streamed oscillator paths, on a TradingView chart](https://s3.tradingview.com/snapshots/n/NmPEybjk.png)](https://www.tradingview.com/x/NmPEybjk/)

Under the hood, **Trail3D** is a coordinated bundle of polylines: one for the main ribbon, two for optional shadow projections onto the back wall and floor, and one for the wireframe cage. All four are depth sorted and occlusion clipped against the rest of the scene, and the primitive auto normalizes incoming samples against the rolling window's min/max so streaming data always fills the cube without manual scaling.

This enables a class of visualizations that would otherwise require dozens of polylines and manual buffer management: phase space portraits, Lissajous figures, oscillator pair correlations, attractor trajectories, and any "two indicators evolving together over time" study. The demo above shows a sine and cosine pair pushing samples each bar to trace a clean spiral inside the cage, the same pattern you would use to plot RSI vs MFI, momentum vs volatility, or any custom (u, v) signal pair.

A full streamed scene is a handful of lines:

```pine
//@version=6
indicator("Pine3D - Trail3D", overlay = false, max_polylines_count = 100, max_lines_count = 500, max_labels_count = 500)
import Alien_Algorithms/Pine3D/5 as p3d

var p3d.Scene   scene = p3d.newScene()
var p3d.Trail3D trail = na

if barstate.isfirst
    trail := p3d.trail3D(220.0, 200, color.yellow)
      .cage(true)
      .axisLabels("sin", "cos", color.white)
    trail._uProj.col := #00ffff69
    trail._vProj.col := #ff00ff71
    scene.add(trail)
    scene.camera.orbit(215.0, 20.0, 360.0)

float phase = bar_index * 0.15
float sinX  = math.sin(phase) * 100.0
float cosY  = math.cos(phase) * 100.0
if barstate.isconfirmed
    trail.pushSample(sinX, cosY)
    p3d.render(scene)
```

## Bars3D: categorical 3D bar charts

`bars3D()` turns any series of values into a fully lit, depth sorted 3D bar chart in a single call. Each bar is height mapped to its value, color graded between a **low** and **high** color, and packed into one combined mesh with per bar depth grouping so individual bars sort correctly even inside the merged geometry. The companion `updateBars()` mutator refreshes heights, colors, and labels in place every bar without rebuilding geometry, making it suitable for live rankings, rolling windows, and animated comparisons.

[![Pine3D: Bars3D: categorical 3D bar charts, on a TradingView chart](https://s3.tradingview.com/snapshots/4/4ZUmmmsf.png)](https://www.tradingview.com/x/4ZUmmmsf/)

The chainable `barLabels(catNames, valNames)` helper attaches category labels at the base of each bar and value labels at the top, both depth sorted with the rest of the scene. Category labels are set once at build time, while value labels can be passed to `updateBars(values, valLabels = ...)` each frame to reflect live data. Combined with `wireGrid()` for the floor and a contour `surface()` in the background, `bars3D()` becomes the centerpiece of dashboards comparing assets, sectors, timeframes, or any categorical metric.

Negative values are handled automatically: bars below zero extrude *downward* from the base plane with reversed face winding, so signed series like PnL, delta, or momentum histograms render correctly without any extra setup.

A complete labeled bar chart is just a few lines:

```pine
//@version=6
indicator("Pine3D - Bars3D", overlay = false, max_polylines_count = 100, max_lines_count = 500, max_labels_count = 500)
import Alien_Algorithms/Pine3D/5 as p3d

var p3d.Scene scene = p3d.newScene()
var p3d.Mesh  bars  = na
array<float>  values = array.from(volume - volume[1], volume[1] - volume[2], volume[2] - volume[3], volume[3] - volume[4], volume[4] - volume[5], volume[5] - volume[6])
array<string> names  = array.from("ΔV0", "ΔV-1", "ΔV-2", "ΔV-3", "ΔV-4", "ΔV-5")

if barstate.isfirst
    bars := p3d.bars3D(values, 30.0, 30.0, 10.0, color.blue, color.red, 200.0)
      .barLabels(names)
    scene.add(bars)
    p3d.wireGrid(scene, 300.0, 300.0, 6, 6, color.new(color.gray, 80))
    scene.camera.orbit(215.0, 25.0, 360.0)

if barstate.islast
    bars.updateBars(values)
    p3d.render(scene, lighting = true)
```

*Omitting `valLabels` in `updateBars()` tells the engine to auto format each numeric value via `str.tostring()`. Pass `valLabels` only when you need custom strings.*

## Scatter clouds: points in 3D space

Pine3D treats scatter clouds as a first class use case without needing a dedicated scatter API. Because **Label3D** is the primitive and `scene.add(array<Label3D>)` is a single batch operation, you can scatter **up to 500 points** anywhere in 3D space, each with **independent color, symbol, size, and tooltip**, and have them depth sorted and occlusion clipped against the rest of the scene automatically.

[![Pine3D: Scatter clouds: points in 3D space, on a TradingView chart](https://s3.tradingview.com/snapshots/e/e0jWjItw.png)](https://www.tradingview.com/x/e0jWjItw/)

Each point is a fully addressable **Label3D** with mutable fields. You can change **position**, `textColor`, `bgColor`, `labelStyle` (any **label.style_\*** glyph including circles, squares, diamonds, triangles, crosses, arrows, flags), `labelSize` (any **size.\*** preset), and **txt** per point per bar. The renderer reads these mutations every frame, so animation is just direct field assignment.

This unlocks a wide class of visualizations: clustered data scatter, K means visualizations, particle systems, parametric surfaces sampled as point clouds, gradient colored attractors, multi class classification overlays, and structured curves like the demo above. The double helix demo plots two intertwined parametric strands as ~500 points with alternating colors and per point sizing, all inside the standard `scene.add(array<Label3D>)` pipeline.

The pattern is straightforward: build the array once in `barstate.isfirst`, add it to the scene, then mutate point fields per bar to animate.

```pine
//@version=6
indicator("Pine3D - Scatter Cloud", overlay = false, max_polylines_count = 100, max_lines_count = 500, max_labels_count = 500)
import Alien_Algorithms/Pine3D/5 as p3d

var p3d.Scene scene = p3d.newScene()
var array<p3d.Label3D> points = array.new<p3d.Label3D>()

if barstate.isfirst
    for i = 0 to 499
        p3d.Vec3 pos = p3d.vec3(0.0, 0.0, 0.0)
        points.push(p3d.Label3D.new(position = pos, txt = "•"))
    scene.add(points)
    scene.camera.orbit(35.0, 20.0, 400.0)

if barstate.islast
    for i = 0 to points.size() - 1
        float t = i * 0.05 + bar_index * 0.01
        p3d.Label3D pt = points.get(i)
        pt.position  := p3d.vec3(80.0 * math.cos(t), i * 0.4 - 100.0, 80.0 * math.sin(t))
        pt.textColor := i % 2 == 0 ? color.aqua : color.fuchsia
    p3d.render(scene)
```

---

## Two layer architecture

Pine3D ships as a clean, two layer library:

**Layer 1 - DIY API.** First principle building blocks (**Vec3**, **Mesh**, **Camera**, **Light**, **Scene**, plus world space overlay primitives) for total creative control. Author your own geometry, camera behavior, lighting setup, and scene graph from scratch.

**Layer 2 - High Level Helpers.** Production ready wrappers like `surface()`, `bars3D()`, `trail3D()`, `updateBars()`, `updateSurface()`, `sphere()`, `torus()`, `cylinder()`, and `wireGrid()`, plus chainable contour helpers `gridBox()` and `gridLabels()` that wrap the primitives into a few lines of code. Scatter clouds use the standard **Label3D** primitive directly.

The object model is chainable and scene oriented, so complex setups still read cleanly.

## Feature list

- **Contour Surface Rendering** - The most powerful 3D surface engine ever released for Pine Script. Render tens of thousands of polygon equivalent faces using a single polyline per contour band, delivering smooth, continuous terrain with natural ridges and valleys.
- **Adaptive Rail Sharing** - Solid meshes drawn with the default **linefill** backend reuse one edge line between adjacent coplanar faces, averaging roughly **1.6 lines per face** instead of the naive two, lifting practical mesh capacity from 250 faces to around 360, depending on topology.
- **Interior Face Culling on Merge** - `mergeMeshes(meshes, removeInterior = true)` detects coincident faces with opposing normals and strips them, so voxel style scenes (stacked cubes, block walls, lattice geometry) ship only their exterior shell and spend no budget on hidden interior faces.
- **True Perspective Camera System** - Full 3D camera with position, target, fov, and `orbit()` controls. Supports cinematic camera movement, lookAt by mesh tag, and realistic depth.
- **Real Time Lighting and Shadows** - Directional and point lights with configurable ambient, shadow strength, self shadowing, and a spatial grid acceleration structure for fast shadow queries.
- **High Performance Update System** - `updateSurface()` and `updateBars()` let you animate massive datasets bar by bar without rebuilding geometry, keeping CPU usage minimal.
- **Rich Primitive Library** - Cubes, cuboids, spheres, cylinders, tori, pyramids, planes, discs, circles, custom meshes, and the groundbreaking `bars3D()` with automatic labels.
- **Streamed Trail Primitive** - `trail3D()` maintains a rolling buffer of (u, v) samples and renders them as a 3D ribbon inside a bounding cube, with optional projections onto the back wall and floor and a wireframe cage.
- **Depth Sorted Overlays** - 3D labels, lines, polylines, wire grids, and trails, all correctly occluded and painter sorted against the rest of the scene.
- **Professional Contour Helpers** - `gridBox()` and `gridLabels()` automatically add clean bounding boxes and axis titles, ticks, and series names that refresh on every `updateSurface()` call.
- **Tag Based Scene Graph** - Every **Mesh**, **Label3D**, **Line3D**, and **Polyline3D** can carry a string tag. **Scene** exposes `getMesh()`, `getLabel()`, `getLine()`, `getPolyline()`, `lookAt()`, and `remove()` by tag, turning your scene into a lookup by name graph instead of an index juggling exercise.
- **Chainable, Intuitive API** - Everything is designed for maximum readability and speed of development. Build complex scenes in just a few lines.
- **Production Ready Optimizations** - World vertex caching, view projection caching, face preprocessing cache, shadow grid cache, and contour geometry cache, all managed automatically.

---

## The renderer

Every frame is produced by a single call to `render(scene, ...)`. The renderer runs the full pipeline: world transform, camera transform, back face culling, occlusion culling, depth sort, directional or point lighting with shadows, and perspective projection.

**⚠ render() clears the entire chart drawing pool at the start of every call** - every **polyline**, **line**, **label**, and **linefill** on the chart is deleted before Pine3D redraws, not just the ones it created. If you mix Pine3D with manual `label.new()`, `line.new()`, or similar calls, those drawings must be emitted *after* `render()` or they will be wiped every frame.

**Setup Requirements.** Pine3D consumes polylines, lines, and labels simultaneously, so your `indicator()` declaration must raise all three budgets, and the library must be imported under an alias:

```pine
indicator("My 3D Scene", overlay = false,
     max_polylines_count = 100,
     max_lines_count     = 500,
     max_labels_count    = 500)

import Alien_Algorithms/Pine3D/5 as p3d
```

### render() parameters

- `maxFaces` (int, default 100). Hard cap on **solid faces** drawn per frame. Contour bands, wireframe edges, labels, lines, and overlay polylines are not counted against this cap, and are bounded only by TradingView's global 100 polyline / 500 line / 500 label budgets.
- **culling** (bool, default true). Enable back face culling.
- **lighting** (bool, default false). Enable diffuse shading. Uses `scene.light` unless `lightDir` is passed.
- `lightDir` (Vec3). A light direction for this call only. It points the way the light travels, away from the light, and replaces the scene light for the call.
- **ambient** (float, default 0.3). Minimum brightness for shadowed faces (0.0-1.0). Applies with `lightDir`, or when the scene light is disabled; otherwise the scene light keeps its own ambient.
- **wireframe** (bool, default false). Force outline only output for the entire scene.
- **occlusion** (bool, default true). Sparse raster pass that drops hidden faces before drawing. Major perf win on dense scenes.
- `occlusionRaster` (int, default 768). Raster resolution of the occlusion buffer. Lower = faster but coarser; higher = stricter hidden face rejection.

A `lightDir` passed to `render()` wins over `scene.light`, which makes render() the right place for ad hoc, per frame lighting tweaks.

---

## Rasterizing a scene for PineCanvas

Version 4 added `rasterize()`, and version 5 improved its shadows. It renders the scene's solid meshes per pixel instead of as chart drawings: a depth buffer, smooth shading across curved surfaces with sharp creases kept, Blinn-Phong highlights, and shadow mapping when the scene light's shadow strength is above zero. It uses the same camera and lights as `render()`.

`scene.rasterize(pixels, width, height)` writes an array of palette indices, one per pixel (row-major, row 0 at the bottom, -1 where nothing is drawn), and returns the matching palette. Optional parameters: `rowStart` (draw frames larger than one array in row bands), `shades` per colour, `zoom` and `culling`. [PineCanvas](https://www.tradingview.com/script/KqYfTY0F-PineCanvas-A-Native-High-Resolution-Graphics-Engine/) then draws the result on the chart:

```pine
//@version=6
indicator("Pine3D with PineCanvas", overlay = false, max_polylines_count = 100)
import Alien_Algorithms/Pine3D/5 as p3d
import Alien_Algorithms/PineCanvas/1 as pc

var p3d.Scene scene = p3d.newScene()
var pc.PaletteField frame = pc.paletteField(240, 384)
var pc.Canvas canvas = pc.newCanvas()

if barstate.isfirst
    scene.add(p3d.cube(60.0, color.orange))
    // The floor: grid() faces down, so it is turned over
    scene.add(p3d.grid(600.0, 600.0, 8, 8, #3a4258).rotateTo(180.0, 0.0, 0.0).moveTo(0.0, -30.0, 0.0))
    scene.camera.orbit(35.0, 25.0, 180.0)
    scene.setLightDir(-0.7, -1.0, -0.3).setAmbient(0.25).setShadowStrength(0.6)

if barstate.islastconfirmedhistory and bar_index >= 384
    array<color> shades = scene.rasterize(frame.band(0), 384, 240, shades = 16, zoom = 2.4)
    canvas.palette(shades)
    canvas.drawLookback(frame, 0, 0.0, 240.0)
```

## Mesh drawing modes

Two independent axes control how a mesh appears on the chart:

**Style** (via `mesh.setStyle(...)`) - what gets drawn:

- `"solid"`. Filled faces. Default.
- `"wireframe"`. All edges, no fill. Shows interior geometry.
- `"wireframe_front"`. Only front facing edges. Cleaner silhouette for convex meshes.

**Draw Mode** (via `mesh.drawMode`) - which TradingView primitive carries the solid faces:

- `"linefill"` (default). Uses the line and linefill budgets. An adaptive **rail sharing** optimization reuses one edge line between adjacent coplanar faces, pushing practical capacity up to **~360 faces** per mesh depending on topology. Supports in place updates via `updateSurface()` and `updateBars()`. Rails are drawn transparent, so solid faces in this mode have no visible outline - use a wireframe style or `"poly"` drawMode if you need stroked edges. Recommended for all new code.
- `"poly"`. Legacy polyline backend. Capacity ~100 faces, no in place updates, but renders the face outline using `mesh.lineStyle` and `mesh.lineWidth`. Use only when you need styled solid face outlines.

Wireframe styles always render with line primitives regardless of drawMode. Stroke width and style on edges (and on poly mode face outlines) come from `mesh.lineWidth` and `mesh.lineStyle`, which you mutate by direct field assignment.

---

## Quick start

The best practice lifecycle is simple:

1. Create one persistent **Scene** with `newScene()`.
2. Build meshes and helper overlays once in `barstate.isfirst`.
3. On later bars, mutate objects in place with transforms or helper mutators like `updateBars()` and `updateSurface()`.
4. Call `render(scene, ...)` once per frame. It automatically clears the previous chart drawings.

A complete, lit, animated 3D scene is still a handful of lines:

```pine
//@version=6
indicator("My First 3D Scene", overlay = false, max_polylines_count = 100, max_lines_count = 500, max_labels_count = 500)
import Alien_Algorithms/Pine3D/5 as p3d

var p3d.Scene scene = p3d.newScene()
var p3d.Mesh  sun   = na

if barstate.isfirst
    scene.setLightDir(1.0, -1.0, 0.5).setAmbient(0.3)
    sun := p3d.sphere(50.0, 16, 12, color.orange).setTag("sun")
    scene.add(sun)
    p3d.wireGrid(scene, 300.0, 300.0, 6, 6, color.new(color.gray, 80))
    scene.camera.orbit(35.0, 25.0, 220.0)

if barstate.islast
    sun.rotateBy(0.0, 1.5, 0.0)
    p3d.render(scene, lighting = true)
```

---

## Recommended usage pattern

- Use your **Scene** and major meshes in **var**.
- Build geometry once in `barstate.isfirst`.
- Use `updateSurface()` and `updateBars()` on later bars instead of rebuilding meshes.
- Use scene level helpers like `wireGrid()` when you want overlays added immediately.
- Use `trail3D()` when you want a streamed oscillator style path with built in wall projections and cage geometry.
- For scatter clouds, build an `array<Label3D>` once, hand it to `scene.add(pts)`, then mutate `pt.position`, `pt.textColor`, etc. each bar to animate.
- Use mesh level `gridBox()` and `gridLabels()` (contour) and `barLabels()` (bars) to attach overlays to the mesh setup chain. They are drained into the scene by `scene.add(mesh)`.

## Considerations

**scene.clear() vs render().** `scene.clear()` removes objects from the scene graph (meshes, labels, lines, polylines). `render()` only clears the previous frame's TradingView drawings and redraws from the current scene graph. You almost never need `scene.clear()` in the build once and update pattern.

**Global scope series for updateSurface() / updateBars().** If your data uses Pine's history operator (**[]**) or calls functions like `ta.rsi()`, `ta.atr()`, `request.security()`, those must be declared at global scope so Pine tracks their bar by bar history. Calling them inside `barstate.islast` produces inconsistent results or compiler errors.

**gridLabels() tick values auto refresh.** When you call `updateSurface()`, any tick value labels created by `gridLabels()` are automatically updated to reflect the new data range. Axis titles and positions stay constant. You don't need to rebuild them.

**barLabels() value labels via updateBars().** Create category labels once with `mesh.barLabels(catNames)` at build time, then pass `valLabels` to `updateBars()` on each frame. Value labels are refreshed automatically. Don't call `barLabels()` again.

**Lighting convenience methods are chainable.** `scene.setLightDir()`, `setLightPos()`, `setLightMode()`, `setAmbient()`, `setShadowStrength()`, and `showLightSource()` all return Scene and can be chained: `scene.setLightMode("point").setLightPos(0, 200, 150).setAmbient(0.25)`.

**Mesh transforms return Mesh.** `moveTo()`, `moveBy()`, `rotateTo()`, `rotateBy()`, `scaleTo()`, `scaleUniform()`, `setTag()`, `setStyle()`, `setColor()`, `show()`, `hide()` all return Mesh for chaining: `mesh.moveTo(0, -20, 0).rotateTo(0, 45, 0).setStyle("solid")`.

**Degrees vs radians.** `rotateTo()` and `rotateBy()` on Mesh expect degrees. The low level **Vec3.rotateX/Y/Z()** methods expect radians.

**scene.lookAt() is tag only.** `scene.lookAt(t)` accepts a string tag and points the camera at that mesh. To aim the camera at an arbitrary **Vec3**, call `scene.camera.lookAt(vec)` directly.

**remove(tag) removes one object.** The search order is meshes, then labels, then lines, then polylines, and the first hit wins. Avoid reusing tags across primitive types if you intend to delete by tag.

**Shadow grid acceleration is directional light only.** The spatial shadow grid is only built when **lightMode == "directional"**. Point lights fall back to a linear O(M) scan, so heavy shadow scenes are fastest in directional mode.

**guiShift and yOffset.** `scene.guiShift` and `scene.yOffset` position the 3D viewport on the chart without consuming historical bar slots. Increase `guiShift` to push the scene rightward into future bar space; adjust `yOffset` to slide it vertically in price units. `scene.yScale` (added in version 3) scales the projected scene height on price charts.

**bar_time projection.** All chart drawings are emitted with `xloc.bar_time`, so the scene can sit arbitrarily far left or right of **bar_index** without forcing Pine to extend its history buffer. This is what keeps the engine stable on long charts and future projected scenes.

**barLabels() without values.** When you call `mesh.barLabels(catNames)` and omit value labels, every later `updateBars(values)` auto formats the numeric values via `str.tostring()`. Pass `valLabels` only when you need custom strings.

**Direct mesh.vertices mutation requires invalidateCache().** Transform mutators (`moveTo`, `rotateBy`, `scaleTo`, etc.) invalidate the world vertex cache on their own. Only raw index writes like `mesh.vertices.set(i, newVec)` need a manual `mesh.invalidateCache()` call to force re-projection. Skipping it will make the renderer draw stale geometry.

**Drawing budgets fail silently.** If a scene emits more than 100 polylines, 500 lines, or 500 labels in a single frame, TradingView silently drops the overflow without raising a runtime error. Missing geometry almost always means a budget overrun - lower `maxFaces`, drop a contour level, or simplify overlay primitives to bring the frame back inside the caps.

**render() deletes non Pine3D drawings too.** Every `render()` call clears `polyline.all`, `line.all`, `label.all`, and `linefill.all` before redrawing. Any manual `label.new()`, `line.new()`, etc. issued *before* `render()` in the same frame will be wiped. Issue custom drawings *after* the render call if you need them to persist.

**mergeMeshes() preserves depth grouping.** When every source mesh passed into `mergeMeshes()` has the same vertex and face count (e.g. identical primitives in a voxel grid), the merged mesh auto derives depth group boundaries so the combined geometry still sorts correctly per original instance. Mixing primitives with different topologies disables the grouping.

**CPU timeouts: knobs to turn.** Pine Script enforces a per bar execution budget, and dense scenes can trip it before the drawing budget ever does. If a scene compiles but times out at runtime, reach for these levers in order: lower `occlusionRaster` (e.g. 768 -&gt; 384) for the biggest single perf win, reduce `maxFaces` to cap the solid face pool, drop **levels** on contour surfaces, simplify sphere/torus segment counts, and gate heavy work behind `barstate.islast` so history bars only build geometry rather than render it.

---

## More examples

The following scenes were all built entirely in Pine Script™ v6 using Pine3D as the rendering layer. They exist to demonstrate that the library is a real engine capable of complex, production grade visualizations.

**4D Hypercube (Tesseract).** A rotating tesseract, projected from 4D to 3D to 2D in real time using a custom 4D rotation matrix layered on top of Pine3D's standard projection pipeline.

[![Pine3D: a rotating 4D hypercube (tesseract), on a TradingView chart](https://s3.tradingview.com/snapshots/0/09nbQrgB.png)](https://www.tradingview.com/x/09nbQrgB/)

**Solar System.** Following the publication of my **3D Solar System** back in 2024, which introduced new graphical rendering concepts into Pine Script, we have seen a wave of various interpretations of the underlying vector classes, ranging from tutorials to niche specific integrations using hardcoded math. It became clear that a unified architecture was needed, one that would lower the barrier to entry while simultaneously handling the optimization process, which is both complex and error prone to do manually.

That architecture is what Pine3D delivers. Below is a re-creation of the classic 3D Solar System rebuilt entirely on top of the library. It uses a **fraction of the original code**, renders **roughly 5x faster**, and adds **real lighting cast directly from the Sun**, all while consuming only **a third of the available drawing budget** thanks to the occlusion and culling mechanisms Pine3D handles out of the box.

[![Pine3D: the 3D Solar System rebuilt on Pine3D, lit from the Sun, on a TradingView chart](https://s3.tradingview.com/snapshots/d/DP2tSYed.png)](https://www.tradingview.com/x/DP2tSYed/)

---

## API reference

**Top Level Entry Points.** `newScene()` creates a ready to use Scene with a default camera and light. `render(scene, ...)` draws the current frame and auto clears the previous frame's chart drawings; see the **Renderer** section above for the full parameter list. `vec3(x, y, z)` creates a Vec3. `colorBrightness()` is an exported color utility helper.

### Mesh Factories

- **Primitives** - `cube()`, `cuboid()`, `pyramid()`, `plane()`, `sphere()`, `cylinder()`, `torus()`, `grid()`, `disc()`, `circle()` for ready made geometry.
- `customMesh(verts, faces)` - Low level escape hatch for authoring your own topology.
- `mergeMeshes(meshes, tag, removeInterior)` - Bakes transforms and combines many meshes into one. With **removeInterior = true**, coincident faces with opposing normals (e.g. shared walls between adjacent cubes in a grid) are culled so only the exterior shell survives, a major optimization for dense voxel style scenes.
- `surface(heights, size, lowCol, highCol, levels, axisX, axisZ)` - Creates a contour surface mesh.
- `bars3D(values, barWidth, barDepth, spacing, lowCol, highCol, maxHeight)` - Creates a combined 3D bar chart mesh; add labels with the chainable `barLabels(names, values)` method.

**UDT Constructors.** Overlay primitives and face descriptors are plain UDTs. Because these types have many fields, **always instantiate them with named arguments** rather than positional, e.g. `Label3D.new(position = pos, txt = "•")`:

- **Face** - fields: **vi** (array&lt;int&gt; of vertex indices into the parent mesh), **col**. Used when authoring `customMesh()` topology; every face must have at least 3 indices and should be planar.
- **Label3D** - fields: **position**, **txt**, `textColor`, `bgColor`, `labelStyle`, `labelSize`, `fontFamily`, **tooltip**, **visible**, **tag**. Only **position** is required.
- **Line3D** - fields: **start**, **end**, **col**, **width**, **visible**, **tag**, `lineStyle`.
- **Polyline3D** - fields: **points**, **col**, `fillColor`, **width**, **closed**, **visible**, **tag**, `lineStyle`.
- `Vec3.new(x, y, z)` or the `vec3(x, y, z)` shorthand.

**Trail Primitive.** `trail3D(size, capacity, trailCol, minSamples, rollingBounds)` creates a streamed Trail3D primitive with a main trail, two projection polylines, and a cage polyline. **capacity** is internally clamped to **300** samples to keep the rolling buffer inside Pine's execution budget; passing a larger value silently resolves to 300. `minSamples` (default 60) is the sample count at which the cage reaches its full cube width: below that the cage stays cube shaped and samples stretch across it; above that the cage grows leftward at a fixed step until **capacity** is hit. `rollingBounds` (default true, added in version 2) rescales both streams to the current window on every push; with `false`, the bounds only ever expand. `scene.add(trail)` registers the sub primitives into the scene. Trail3D methods: `pushSample()`, `axisLabels()`, `cage()`, `moveTo()`, `show()`, `hide()`.

### Mesh Methods

- **Transform** - `moveTo()`, `moveBy()`, `rotateTo()`, `rotateBy()`, `scaleTo()`, `scaleUniform()`.
- **Appearance** - `setColor()`, `setFaceColor()`, `setStyle()`, `show()`, `hide()`, `setTag()`.
- **Stroke styling (direct)** - **mesh.lineWidth := 3** and **mesh.lineStyle := line.style_dashed** control width and style of every visible mesh edge in wireframe modes and the outline of solid faces in **drawMode = "poly"**.
- **Shadow opt out (direct)** - **mesh.castShadow := false** excludes the mesh from shadow casting while still receiving light. Useful for ghost overlays, debug geometry, or semi transparent meshes you do not want occluding the scene.
- **Lifecycle** - `clone()`, `faceCount()`, `invalidateCache()`.
- **Data mutation** - `updateSurface()` and `updateBars()` refresh persistent meshes in place. `updateBars()` refreshes any bar label positions automatically; pass **catLabels / valLabels** to also update the text.
- **Contour helpers** - `gridBox()` and `gridLabels()` queue overlays on the mesh and hand them to the scene when you call `scene.add(mesh)`.
- **Bar helpers** - `barLabels()` is chainable on a `bars3D()` mesh and queues its category and value labels for the next `scene.add(mesh)`.

***Note:** `rotateTo()` and `rotateBy()` expect degrees. The low level **Vec3.rotateX/Y/Z()** methods work in radians.*

### Scene Methods

- **Lighting** - `setLightDir()`, `setLightPos()`, `setLightMode()`, `setAmbient()`, `setShadowStrength()`, `showLightSource()`.
- **Scene graph** - `add(mesh)`, `add(label)`, `add(array<Label3D>)`, `add(line)`, `add(polyline)`, `add(trail)`, `remove(index)`, `remove(tag)`, `clear()`.
- **Lookup and navigation** - `getMesh()`, `getLabel()`, `getLine()`, `getPolyline()`, `lookAt()`, `totalFaces()`.
- **Rendering to pixels** - `rasterize(pixels, width, height, rowStart, shades, zoom, culling)`, added in version 4. See the section above.
- **Cache control** - `invalidateLightCache()` after mutating light direction or scene bounds externally; `invalidateAllCaches()` to also invalidate every mesh's world vertex cache (use after directly mutating `mesh.vertices`).

***Note:** `scene.clear()` clears the scene graph itself. `render()` only clears the previous frame's TradingView drawings.*

**Camera Methods.** `setPosition(x, y, z)` moves the camera. `lookAt(x, y, z)` / `lookAt(vec3)` points at a world space target. `orbit(angleX, angleY, distance)` does a spherical orbit around the current target. `setFov(val)` sets the perspective scale factor. Camera fields (**position**, **target**, **fov**) are also directly mutable via assignment when you need to tune them outside the provided setters, e.g. **scene.camera.fov := 1200.0**.

**Light Field Mutation.** In addition to the scene level convenience setters, every field on `scene.light` is directly mutable for fine grained tuning: **scene.light.selfShadow := true** enables self shadowing, **scene.light.shadowBias := 0.2** adjusts the shadow acne offset, `scene.light.shadowStr` and `scene.light.ambient` are also exposed. Mutate them after `newScene()` or between frames; the renderer reads them every call.

**Vec3 Methods.** Core math: `add()`, `sub()`, `scale()`, `negate()`, `dot()`, `cross()`, `length()`, `normalize()`, `distanceTo()`, `lerp()`. Rotation and helpers: `rotateX()`, `rotateY()`, `rotateZ()`, `copy()`, `toString()`.

### Overlay Primitive Methods

- **Label3D** - `moveTo()`, `moveBy()`, `setText()`, `setTextColor()`, `setTooltip()`, `show()`, `hide()`, `setTag()`.
- **Line3D** - `setStart()`, `setEnd()`, `setPoints()`, `setColor()`, `show()`, `hide()`, `setTag()`.
- **Polyline3D** - `setColor()`, `show()`, `hide()`, `setTag()`.

Every UDT field is mutable via direct assignment for properties without a chainable setter:

- **Label3D** - `bgColor`, `labelStyle` (label.style_\*), `labelSize` (size.\*), `fontFamily` (font.family_\*), **visible**.
- **Line3D** - **width**, `lineStyle` (line.style_solid / _dashed / _dotted / _arrow_left / _arrow_right / _arrow_both), **visible**.
- **Polyline3D** - **width**, `lineStyle` (line.style_solid / _dashed / _dotted only; arrow styles are **not** supported by TradingView's polyline primitive), `fillColor`, **closed**, **visible**.

Mutations are read per frame by the renderer, so they animate freely.

**High Level Scene Helpers.** `wireGrid(scene, w, d, divX, divZ, col)` adds a depth sorted ground grid. `scene.add(array<Label3D>)` adds a batch of labels in one call - the idiomatic way to push a scatter cloud into the scene.

**Mesh Level Chainable Overlays.** `mesh.barLabels(names, values, ...)` adds category and value labels on a `bars3D()` mesh. `mesh.gridBox(col, divs)` adds a wireframe bounding box cage on a `surface()` mesh. `mesh.gridLabels(col, xName, yName, zName, ticks, fmt)` adds axis titles and tick value labels on a `surface()` mesh; tick values auto refresh on `updateSurface()`. All three are queued on the mesh and drained into the scene by `scene.add(mesh)`.

---

## Related libraries

- [PineCanvas](https://github.com/AlienAlgorithms/PineCanvas-A-Native-High-Resolution-Graphics-Engine-for-TradingView): draws the frames from Pine3D's `rasterize()` as high-resolution images on the chart.
- [NeuraLib](https://github.com/AlienAlgorithms/NeuraLib-A-Native-AI-and-Deep-Learning-Runtime-for-TradingView): machine learning in Pine Script. The diagrams in its publication are drawn with Pine3D.

## Licence

Pine3D is licensed under [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/). The full legal text is in [LICENSE](LICENSE).

- **Non-commercial use is free** when [Alien_Algorithms](https://www.tradingview.com/u/Alien_Algorithms/?aff_id=169033#published-scripts) is credited in the description of any script that uses it.
- **Share alike:** a published adaptation must carry the same licence.
- **Commercial use** needs a separate licence. Contact [Alien_Algorithms on TradingView](https://www.tradingview.com/u/Alien_Algorithms/?aff_id=169033#published-scripts).

## Author

Built by [Alien_Algorithms on TradingView](https://www.tradingview.com/u/Alien_Algorithms/?aff_id=169033#published-scripts) and [AlienAlgorithms on GitHub](https://github.com/AlienAlgorithms). More indicators and libraries at [alienalgorithms.com](https://alienalgorithms.com).
