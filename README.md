# PineCanvas

A high-resolution graphics engine for TradingView, written in Pine Script v6: heatmaps on price, images of over a million cells, rasterized Pine3D frames and thousands of filled shapes, packed into Pine's 100-polyline budget.

[![PineCanvas on TradingView](https://s3.tradingview.com/k/KqYfTY0F.png?v=1791628508)](https://www.tradingview.com/script/KqYfTY0F-PineCanvas-A-Native-High-Resolution-Graphics-Engine/)

- **TradingView publication:** [PineCanvas: A Native High-Resolution Graphics Engine](https://www.tradingview.com/script/KqYfTY0F-PineCanvas-A-Native-High-Resolution-Graphics-Engine/)
- **Documentation:** [docs.alienalgorithms.com](https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview)
- **Product page:** [alienalgorithms.com/pinecanvas](https://alienalgorithms.com/pinecanvas)
- **Version:** 1. See [CHANGELOG.md](CHANGELOG.md).
- **Licence:** CC BY-NC-SA 4.0, free for non-commercial use with credit. See [Licence](#licence).

## Use it in a script

```pine
//@version=6
indicator("My PineCanvas Script", overlay = false, max_polylines_count = 100)
import Alien_Algorithms/PineCanvas/1 as pc
```

Pine Script loads libraries from TradingView, not from GitHub. The import line above uses the published library; the source in this repository is the same code, here to read, search and reference. When PineCanvas is updated on TradingView, the version number in the import line goes up.

## Coding with an AI assistant

Pine Script is rare in AI training data, so assistants tend to guess library calls. Give the assistant [REFERENCE.md](REFERENCE.md) before it writes PineCanvas code. It documents the calls with their parameters, the rules for correct scripts and complete examples. The same file is online at https://alienalgorithms.com/pinecanvas.md.

## In this repository

| File | What it is |
|---|---|
| [PineCanvas.pine](PineCanvas.pine) | The library source, version 1, as published on TradingView |
| [REFERENCE.md](REFERENCE.md) | The whole API in one file, for people and AI assistants |
| [CHANGELOG.md](CHANGELOG.md) | What each version changed |
| [LICENSE](LICENSE) | CC BY-NC-SA 4.0 legal text |
| [examples/candle-colours.pine](examples/candle-colours.pine) | PineCanvas - Candle Colours |
| [examples/rolling-rsi-heatmap.pine](examples/rolling-rsi-heatmap.pine) | PineCanvas - Rolling RSI Heatmap |
| [examples/pine3d-with-pinecanvas.pine](examples/pine3d-with-pinecanvas.pine) | Pine3D with PineCanvas |
| [examples/high-volume-zones.pine](examples/high-volume-zones.pine) | PineCanvas - High Volume Zones |

---

**PineCanvas** is a high-definition contour rendering engine for TradingView, powered natively by Pine Script™.

Build detailed heatmaps, complex animated images, 2D/3D graphics, and tens of thousands of filled shapes through a chainable API. PineCanvas packs matching-colour regions into shared drawings instead of spending one drawing object per cell.

Each **cell** acts like a pixel in the source image. Around **1.1 million cells** is an attainable image scale, not a theoretical maximum: transparent cells add no filled drawing geometry, and same-colour regions share drawings. Simpler or sparser images can go considerably higher, within Pine's memory and execution limits.

---

## Quick start

A **PaletteField** holds colour indices in a grid. Give it a palette, assign values to its cells, and draw. This complete indicator colours the last 30 candles green or red by direction:

```pine
//@version=6
indicator("PineCanvas - Candle Colours", overlay = false, max_polylines_count = 100)
import Alien_Algorithms/PineCanvas/1 as pc
// Make sure to update to the latest library version

var pc.PaletteField pixels = pc.paletteField(2, 30)
var pc.Canvas canvas = pc.newCanvas(array.from(color.red, color.lime))

if barstate.islast and bar_index >= 29
    for col = 0 to 29
        pixels.set(0, col, close[29 - col] >= open[29 - col] ? 1 : 0)
    canvas.drawLookback(pixels, 0, 0.0, 2.0)
```

Each candle contributes one coloured cell; the untouched row remains transparent. The Canvas draws the strip without requiring a separate box for every candle.

---

## High resolution: over one million cells

[![PineCanvas: High resolution: over one million cells, on a TradingView chart](https://s3.tradingview.com/snapshots/m/MROlIUzg.png)](https://www.tradingview.com/x/MROlIUzg/)

*Julia fractal: **1344 × 804 cells**, or **1,080,576 individually addressable cells**, rendered natively on TradingView.*

Pine Script allows a maximum of **500 boxes** and **100 polylines** per indicator. Drawing one box per cell would exhaust the box budget at just 20 × 25 cells. PineCanvas breaks that one-cell, one-drawing relationship.

**The breakthrough is drawing boundaries instead of individual cells.** PineCanvas traces each same-colour region, discards internal edges, and connects separate islands through invisible, zero-width links. A solid block containing thousands of cells can be represented by just its four outer corners. Holes and empty space remain unfilled.

This makes detailed images possible without requiring a separate drawing for every visible cell. The fractal pushes that principle through thin branches, isolated islands, and multiple colours. Its calculations are distributed across historical bars, and its final image is rendered in sections to manage heavier workloads.

The practical limit depends on **boundary complexity, colour distribution, memory, and execution time**, not a fixed pixel count. Smooth or sparse images can exceed the fractal's resolution; highly fragmented images can be more demanding at much smaller sizes.

---

## Heatmaps: putting data directly on price

High resolution is useful for much more than artwork. A heatmap places numerical data directly across *time* and *price*, turning levels and changes into visible bands over the candles.

[![PineCanvas: Heatmaps: putting data directly on price, on a TradingView chart](https://s3.tradingview.com/snapshots/s/sG0BGuFk.png)](https://www.tradingview.com/x/sG0BGuFk/)

The liquidation heatmap above models leveraged exposure from price and volume, distributes it across leverage levels, fades older positions, and removes modelled levels when price crosses them. Brighter bands show where the **estimated** exposure remains concentrated.

The liquidation model belongs to the indicator, not the library. Its calculated values go into a **ValueField**; PineCanvas maps their intensity to colour and positions the result at real price levels.

```pine
// Integration pseudocode. The calling script computes estimatedExposure,
// chooses row/column indices, and supplies its own price bounds.
var pc.ValueField levels = pc.valueField(240, 400)
var pc.Canvas heat = pc.heatmap()
  .colorLevels(48)
  .opacity(10, 90)
  .gamma(0.7)

// For each calculated liquidation level:
levels.set(row, column, estimatedExposure)

// After the model has populated the field:
if barstate.islast
    heat.drawLookback(levels, 0, bottomPrice, topPrice)
```

**Colour control.** `colorLevels()` selects 2 to 100 gradient levels, `opacity()` sets their visible transparency, and `gamma()` controls how strongly weaker values appear. Use `autoScale()` to adapt to current data or `valueRange()` to keep the same colours meaningful between updates.

**Rolling sources.** **ValueScroll** and **PaletteScroll** retain a moving window when one new column arrives each bar. They overwrite the oldest stored column rather than shifting the entire history, and can normalize the whole image, each row, or each column.

Here is a complete, copy-paste example of that second workflow. It produces a five-row RSI heatmap, with one row for each lookback period and one column per completed bar:

```pine
//@version=6
indicator("PineCanvas - Rolling RSI Heatmap", overlay = false,
  max_polylines_count = 100)
import Alien_Algorithms/PineCanvas/1 as pc
// Make sure to update to the latest library version

// Stateful series calculations must run on every bar.
float rsi5 = ta.rsi(close, 5)
float rsi8 = ta.rsi(close, 8)
float rsi13 = ta.rsi(close, 13)
float rsi21 = ta.rsi(close, 21)
float rsi34 = ta.rsi(close, 34)

var pc.ValueScroll history = pc.valueScroll(5, 150)
var pc.Canvas heat = pc.heatmap(
  pc.gradient(#602060, #305bb4, #31b6ad, #f4d24c, #f07735))
  .colorLevels(32)
  .valueRange(0.0, 100.0)
  .opacity(25, 100)

if barstate.isconfirmed
    history.push(array.from(rsi5, rsi8, rsi13, rsi21, rsi34))

if barstate.islast
    heat.drawLookback(history, 0.0, 5.0)
```

The same approach suits volatility maps, order-flow estimates, probability surfaces, and other studies that build up over time.

---

## Pine3D: from 3D scenes to high-definition images

[Pine3D](https://www.tradingview.com/script/I2K4hIdP-Pine3D-A-Native-3D-Graphical-Rendering-Engine/) creates the meshes, camera, and lighting. PineCanvas turns the resulting image into chart drawings. The two libraries connect through `rasterize()`, which writes the scene directly into a palette field.

[![PineCanvas: Pine3D: from 3D scenes to high-definition images, on a TradingView chart](https://s3.tradingview.com/snapshots/h/HOvv23QA.png)](https://www.tradingview.com/x/HOvv23QA/)

The scene above combines a torus, sphere, and cube at **1536 × 960 cells**, with depth-tested visibility, smooth shading, and cast shadows. Creating a simpler scene, a lit cube casting its shadow on a floor, follows the same pipeline. The example below includes the complete scene setup and renders it through PineCanvas:

```pine
//@version=6
indicator("Pine3D with PineCanvas", overlay = false, max_polylines_count = 100)
import Alien_Algorithms/Pine3D/5 as p3d
import Alien_Algorithms/PineCanvas/1 as pc
// Make sure to update to the latest library version

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

The 384 × 240 example fits in one field storage band. Larger frames can be split across bands, with each section written directly through `band()` and `bandStart()`.

---

## Filled shapes: thousands of session profile bars

Not everything belongs in a grid. The Shape API queues **rectangles, triangles, circles or ellipses, and custom filled polygons** directly in chart time and price coordinates, using `rect()`, `triangle()`, `circle()`, and `poly()`.

TradingView allows **100 polylines with up to 10,000 points each**, or a theoretical **one million point slots** across a script. Instead of spending one drawing on each filled shape, PineCanvas packs many disconnected shapes of the same colour into shared polylines. The native limits still apply, but **fewer distinct colours generally leave more room for shapes**, since each colour requires its own drawing allocation.

[![PineCanvas: Filled shapes: thousands of session profile bars, on a TradingView chart](https://s3.tradingview.com/snapshots/p/pVUF6Ege.png)](https://www.tradingview.com/x/pVUF6Ege/)

The session-profile demo uses PineCanvas's **Shape API** to retain **10,000 to 24,000 filled rectangles** across completed sessions, keeping the newest 24,000 as history advances. Colours distinguish the value area, point of control, and remaining profile rows.

Each profile row is queued as an ordinary rectangle. The full profile calculation belongs to the demo, but the same API works for simpler tasks too. This complete indicator shades every candle's range by its volume: the heavier the candle against its 20-bar average, the more opaque its rectangle.

```pine
//@version=6
indicator("PineCanvas - High Volume Zones", overlay = true,
  max_polylines_count = 100)
import Alien_Algorithms/PineCanvas/1 as pc
// Make sure to update to the latest library version

int lookback = input.int(100, "Lookback bars", minval = 20, maxval = 300)
float averageVolume = ta.sma(volume, 20)
var pc.Canvas zones = pc.newCanvas()

if barstate.islast and bar_index >= lookback + 19
    for column = 0 to lookback - 1
        int offset = lookback - 1 - column
        int leftTime = time[offset]
        int rightTime = offset > 0 ? time[offset - 1] : time + int(time - time[1])
        // Heavier volume draws more opaque, in four steps so the colours stay few
        int tier = int(math.min(3, nz(volume[offset] / averageVolume[offset]) * 2))
        color zoneColor = color.new(
          close[offset] >= open[offset] ? color.teal : color.red, 85 - 20 * tier)
        zones.rect(
          chart.point.from_time(leftTime, high[offset]),
          chart.point.from_time(rightTime, low[offset]),
          zoneColor)
    zones.drawShapes()
```

This simplified example shades whole candle ranges, not their true internal volume distribution. The current candle's right edge is estimated from the previous bar spacing. It also illustrates the queue lifecycle: enqueue rectangles, then call `drawShapes()` once to replace the previous frame.

One polyline can hold approximately **1,600 four-corner rectangles of one colour**. A queued batch accepts up to **100,000 source vertices**, equivalent to 25,000 rectangles. The actual drawing count depends on their colours and geometry.

The same approach applies to session profiles, fair value gaps, order blocks, and other studies built from large groups of filled regions. Shapes use chart times and prices, so they stay anchored to their underlying bars.

---

## Chart interfaces: an Astro HUD

The same Shape API can build a complete interface around the candles. The **Astro HUD** brings a zodiac wheel, a band aligned with chart time, and an ephemeris panel together in one continuous translucent overlay.

[![PineCanvas: Chart interfaces: an Astro HUD, on a TradingView chart](https://s3.tradingview.com/snapshots/e/EMxUTW1a.png)](https://www.tradingview.com/x/EMxUTW1a/)

The wheel places the Sun, Moon, and planets at their calculated zodiac positions, connects their aspects, and displays the lunar phase at its centre. Across the top, coloured ribbons track sign changes, a separate lane marks Mercury's retrograde periods, and an aspect-tide curve follows the changing planetary relationships. New and full moons mark the band's curved lower edge, with optional guides extending down through the chart.

The ephemeris panel presents the same sky numerically: positions, daily motion, the closest aspects, and lunar readings. Harmonic patterns sit directly on the price swings, with their measured ratios and projected levels.

**The interface is built from filled geometry.** Rings, coloured sectors, curved panels, glow layers, and pattern fills are assembled through `rect()` and `poly()`. PineCanvas packs their matching colours into shared polylines, allowing thousands of shapes to form one detailed interface. Ordinary Pine labels supply the text and glyphs.

The layout is calculated from the visible bars and price range, so it refits the view as the chart is panned or zoomed. The astronomical calculations and harmonic-pattern logic belong to the demo; PineCanvas handles their visual presentation through the same Shape API used for the session profiles above.

---

## Exact colour graphics: full images inside indicators

A **PaletteField** is not limited to a few market-state colours. Every cell can select an exact palette entry, so a script can produce detailed graphics with precise edges, fine shading, and transparent areas.

The two examples below demonstrate intricate, multi-colour scenes written in Pine Script and displayed through PineCanvas.

[![PineCanvas: Exact colour graphics: full images inside indicators, on a TradingView chart](https://s3.tradingview.com/snapshots/j/j16vNQ2p.png)](https://www.tradingview.com/x/j16vNQ2p/)

The calling script is responsible for generating the picture. PineCanvas handles its palette, filled contours, and placement on the chart. For large images, direct `row()` or `band()` access lets the producer write into the source arrays without calling `set()` for every cell. The palette index **-1** leaves a cell transparent.

---

## Rendering, caching, and real-world limits

PineCanvas reuses cached contour work and can skip drawing replacement when the source, style, and placement are unchanged. Large sources use tiled processing and storage bands to stay within Pine's collection limits. Practical constraints still apply:

- **Drawing budgets are shared.** All Canvas instances in one script share the 100-polyline limit with other polylines. `drawingLimit()` caps an individual Canvas; it does not allocate or coordinate a global budget automatically.
- **Complexity determines practical resolution.** More colours and more fragmented boundaries generally require more geometry. Memory and execution time remain limits, even when the source fits in the field's storage bands.
- **Shapes are filled areas, not independent outlined objects.** A packed shape batch is grouped by colour. Same-colour overlaps form a union, and separate shape borders or independent per-shape layering are not supported by the packed fill itself.
- **Use appropriate chart coordinates.** Bar-index drawings are subject to the platform's 10,000-bars-back and 500-bars-forward bounds. Use time-based placement for farther history, and real chart timestamps when exact candle alignment matters.
- **Canvas ownership is local.** Each Canvas manages its own polylines. Field drawing and shape drawing replace that Canvas's previous frame, so separate canvases are useful for independent layers.
- **Chart rendering has its own behaviour.** Animation follows Pine executions, not a separate timer, and chart anti-aliasing can sometimes expose thin seams between adjacent fills at certain zoom levels.

[TradingView's Pine Script limits](https://www.tradingview.com/pine-script-docs/writing/limitations/) remain the underlying constraints.

---

## API reference

### Sources

- `valueField(rows, columns, initial)` - A numerical grid mapped through the Canvas gradient. **na** cells are transparent.
- `paletteField(rows, columns, initial)` - A grid of exact palette indices. **-1** is transparent; **0** selects the first palette entry.
- `valueScroll(rows, capacity)` and `paletteScroll(rows, capacity)` - Rolling sources retaining up to **capacity** columns.
- `push(column)` - Appends a column to a scroll, bottom row first, overwriting the oldest once full.
- `get(row, column)` and `set(row, column, value)` - Read or write a cell in a field.
- `row(rowIndex)` - A writable array sharing one field row's storage. Do not resize it.
- `fill(value)` - Assigns one value to every cell in a field.
- `bandCount()`, `band(index)`, `bandStart(index)`, and `bandRows(index)` - Direct access to a field's storage blocks.

### Field Stamps

These helpers write directly into a field, clip to its bounds, and return the field for chaining:

- `rect(row0, column0, row1, column1, value, filled)` - A filled or outlined rectangle.
- `segment(row0, column0, row1, column1, value)` - A straight line.
- `circle(centerRow, centerColumn, radius, value, filled)` - A circle or disc.

### Canvas Configuration

- `newCanvas(colors)` - Neutral Canvas defaults: exact colours for palette sources, gradient stops for value sources, full opacity, and a fixed 0 to 1 value range.
- `heatmap(colors)` - A heatmap preset with 64 colour levels, automatic scaling, and a 15 to 90 opacity ramp. Omitting colours selects the thermal gradient.
- `palette(colors)` - Replaces the palette or gradient stops, with 2 to 100 entries.
- `colorLevels(count)` - Samples a value-source gradient into 2 to 100 levels. Palette sources use their palette entries directly.
- `opacity(low, high)` - Sets visible opacity across the levels. One argument gives every level the same opacity.
- `autoScale(mode)` - Enables normalization over `ScaleMode.wholeGrid`, `eachColumn`, or `eachRow`.
- `valueRange(minimum, maximum)` - Sets a fixed value range.
- `gamma(value)` - Adjusts the value response curve. Below 1 lifts weak values; above 1 suppresses them.
- `logarithmicY(enabled)` - Spaces regular rows logarithmically between positive price bounds.
- `invertRows(enabled)` - Places row 0 at the top.
- `forceOverlay(enabled)` - Places drawings on the main price chart.
- `drawingLimit(count)` - Caps this Canvas's polyline allocation, from 1 to 100.
- `newStyle(...)` and `configure(style)` - Build and apply a complete Style.
- `clear()` - Deletes the Canvas's owned drawings.

### Drawing Fields

- `drawLookback(source, lookback, bottom, top, rightOffset)` - Places a field relative to the current bar. A lookback of 0 uses one bar per column.
- `draw(source, leftIndex, rightIndex, bottom, top)` - Places a field between two bar indices.
- `drawTime(source, leftTime, rightTime, bottom, top)` - Places a field between two timestamps.
- `drawGrid(source, columnEdges, rowEdges)` - Uses explicit bar-index and price edges for an uneven grid.
- `drawTimeGrid(source, timeEdges, rowEdges)` - Uses explicit timestamp and price edges.
- `drawLookback(scroll, bottom, top, rightOffset)` - Draws a rolling source using its current column count.

### Shapes

- `rect(topLeft, bottomRight, color)`, `triangle(a, b, c, color)`, and `poly(points, color)` - Queue filled shapes using points from `chart.point.from_time()`.
- `circle(centre, radiusTime, radiusPrice, segments, color)` - Queues an ellipse with separate time and price radii.
- `drawShapes()` - Draws the queued shapes, replaces the previous frame, empties the queue, and returns the number of polylines used.
- `clearShapes()` and `shapeCount()` - Discard or count queued shapes.
- `barsToTime(offsetBars)` - Converts a bar offset into a timestamp under an evenly spaced bar assumption.

### Inspection and Helpers

- `drawingCount()` - Number of polylines currently owned by the Canvas.
- `componentCount()` - Connected same-colour regions found by the last field render.
- `dimensions()` - The last rendered field's [rows, columns].
- `scaledRange()` - The value range reported by the last field render.
- `gradient(low, high)` and `gradient(c0, c25, c50, c75, c100)` - Two-stop and five-stop gradients.

**PineCanvas makes detailed visualizations practical within Pine Script's drawing limits, giving indicator authors more freedom to decide what belongs on the chart.**

The publication cover artwork was rendered natively in Pine Script using PineCanvas.

---

## Related libraries

- [Pine3D](https://github.com/AlienAlgorithms/Pine3D-A-Native-3D-Graphical-Rendering-Engine): builds the 3D scenes that PineCanvas draws through `rasterize()`.
- [NeuraLib](https://github.com/AlienAlgorithms/NeuraLib-A-Native-AI-and-Deep-Learning-Runtime-for-TradingView): machine learning in Pine Script, for models whose output PineCanvas can map onto price.

## Licence

PineCanvas is licensed under [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/). The full legal text is in [LICENSE](LICENSE).

- **Non-commercial use is free** when [Alien_Algorithms](https://www.tradingview.com/u/Alien_Algorithms/?aff_id=169033#published-scripts) is credited in the description of any script that uses it.
- **Share alike:** a published adaptation must carry the same licence.
- **Commercial use** needs a separate licence. Contact [Alien_Algorithms on TradingView](https://www.tradingview.com/u/Alien_Algorithms/?aff_id=169033#published-scripts).

## Author

Built by [Alien_Algorithms on TradingView](https://www.tradingview.com/u/Alien_Algorithms/?aff_id=169033#published-scripts) and [AlienAlgorithms on GitHub](https://github.com/AlienAlgorithms). More indicators and libraries at [alienalgorithms.com](https://alienalgorithms.com).
