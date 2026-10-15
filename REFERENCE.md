# PineCanvas

> PineCanvas is an open-source graphics engine for TradingView, published as a Pine Script v6 library by Alien_Algorithms. It draws heatmaps, detailed images, rasterized 3D frames and large batches of filled shapes by tracing same-colour regions and packing them into shared polylines, so one indicator can show over a million cells instead of the few hundred that one box per cell allows.

- Product page: https://alienalgorithms.com/pinecanvas
- Documentation: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview
- Library and source code on TradingView: https://www.tradingview.com/script/KqYfTY0F-PineCanvas-A-Native-High-Resolution-Graphics-Engine/
- Source code, examples and this reference on GitHub: https://github.com/AlienAlgorithms/PineCanvas-A-Native-High-Resolution-Graphics-Engine-for-TradingView
- Import: `import Alien_Algorithms/PineCanvas/1 as pc` (use the latest version number shown on the TradingView publication)
- Language: Pine Script v6
- Author: Alien_Algorithms, https://www.tradingview.com/u/Alien_Algorithms/?aff_id=169033#published-scripts
- Licence: CC BY-NC-SA 4.0. Free for non-commercial use when Alien_Algorithms is credited. Commercial use needs a licence from Alien_Algorithms.

This file is the plain-text reference for the product page. It is written so that a person or an AI assistant can write correct PineCanvas code from it. Signatures, parameter names, defaults and ranges are taken from the library source.

## When to use PineCanvas

Use it when a TradingView indicator has to draw more than Pine Script's own drawing objects allow:

- Heatmaps placed on price: liquidation maps, volatility maps, order-flow estimates, probability surfaces, multi-length oscillator maps.
- Images and exact-colour graphics: fractals, scalograms, pixel scenes, rasterized 3D frames from Pine3D.
- Many filled shapes: session volume profiles, zones, and whole chart interfaces built from rectangles, triangles, ellipses and polygons.

Pine Script allows 500 boxes and 100 polylines per script. One box per cell runs out at 20 x 25 cells. PineCanvas traces each same-colour region, discards the internal edges and joins separate regions with zero-width links, so the regions of one colour share polylines of up to 10,000 points each. A solid block of thousands of cells costs four corners. Transparent cells cost nothing.

Around 1.1 million cells in one image is an attainable scale. The published Julia fractal demo is 1344 x 804 cells, which is 1,080,576. The practical limit depends on boundary complexity, colour distribution, memory and execution time, not on a fixed pixel count.

## The model in three steps

1. A source holds the data. A `ValueField` holds numbers, a `PaletteField` holds exact palette indices, and `ValueScroll` / `PaletteScroll` are rolling versions that take one new column per bar.
2. A `Canvas` holds the colours and the style, and owns the drawings it creates.
3. A draw call places the source on the chart between bar indices, timestamps or explicit edges. Shapes skip the source: they are queued on a Canvas and drawn with `drawShapes()`.

Rows and columns: row 0 is the bottom row unless `invertRows()` is set, and column 0 is the leftmost, oldest column. In a `ValueField`, `na` is transparent. In a `PaletteField`, the index `-1` is transparent and `0` selects the first palette colour.

## Minimal example

A complete indicator. It colours the last 30 candles green or red by direction.

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

Each candle contributes one coloured cell. The untouched row stays transparent.

## Rules that keep a script correct

- Declare the indicator with `max_polylines_count = 100`. PineCanvas draws with polylines, and every Canvas in a script shares that limit.
- Keep sources and canvases in `var` variables. A Canvas owns its polylines and replaces them each time it draws.
- Draw once per update, usually inside `if barstate.islast`. Use `barstate.islastconfirmedhistory` for a heavy picture that should render once.
- Stateful series functions such as `ta.rsi()` or `ta.sma()` must run on every bar, outside the `barstate.islast` block.
- For a rolling source, call `push()` once per bar inside `if barstate.isconfirmed`, with exactly `rows` values, bottom row first.
- `bottom` and `top` are the vertical bounds of the picture. Use real prices on an overlay, or any numbers in a separate pane. `top` must be greater than `bottom`.
- `drawLookback(field, 0, bottom, top)` uses one bar per column and ends at the current bar. A non-zero `lookback` must be at least the field's column count. The left edge is `bar_index + rightOffset - span` and must not be negative, so wait until enough bars exist.
- Bar-index placement reaches 10,000 bars back and 500 bars forward. Use `drawTime()` or `drawTimeGrid()` for anything farther.
- A drawn field needs at least 2 rows and 2 columns.
- A palette or gradient has 2 to 100 colours. Each visible colour needs at least one polyline, so fewer colours leave room for more detail.
- One Canvas is one layer. Every `draw...()` call and every `drawShapes()` call replaces what that Canvas drew before. Use a separate Canvas for each layer that has to stay visible.
- Shape corners must come from `chart.point.from_time()`, in time and price. Points made with `chart.point.from_index()` are rejected.
- For large images, write through `row()` or `band()` instead of calling `set()` for every cell, and spread heavy calculation across bars.
- Canvas setters return the Canvas, so they chain: `pc.heatmap().colorLevels(48).opacity(10, 90).gamma(0.7)`.
- Draw on the main price chart with `overlay = true` in `indicator()`, or with `forceOverlay()` on the Canvas.

## API reference

All names below are exported by the library. With `import Alien_Algorithms/PineCanvas/1 as pc`, functions are called as `pc.name(...)` and methods on the object they belong to.

### Types

| Type | What it is |
| --- | --- |
| `pc.ValueField` | A fixed grid of numbers, coloured through the Canvas gradient. Fields: `rows`, `columns`. |
| `pc.PaletteField` | A fixed grid of exact palette indices. `-1` is transparent. Fields: `rows`, `columns`. |
| `pc.ValueScroll` | A rolling grid of number columns. Fields: `rows`, `capacity`, `written`. |
| `pc.PaletteScroll` | A rolling grid of palette-index columns. Fields: `rows`, `capacity`, `written`. |
| `pc.Canvas` | Holds colours and style, owns its polylines, and draws sources and shapes. |
| `pc.Style` | A complete rendering configuration, built with `pc.newStyle()`. |
| `pc.ScaleMode` | Enum for automatic scaling: `wholeGrid`, `eachColumn`, `eachRow`. |

### Sources

| Call | Description |
| --- | --- |
| `pc.valueField(int rows, int columns, float initial = na)` | A grid of numbers. Rows at least 1, columns 1 to 100,000. `na` cells are transparent. |
| `pc.paletteField(int rows, int columns, int initial = -1)` | A grid of exact palette indices. `-1` is transparent. |
| `pc.valueScroll(int rows, int capacity)` | A rolling source of number columns. Rows and capacity from 2 to 99,999. |
| `pc.paletteScroll(int rows, int capacity)` | A rolling source of palette-index columns. |
| `scroll.push(array<float> column)` / `scroll.push(array<int> column)` | Appends one column of exactly `rows` values, bottom row first. Overwrites the oldest column once full. |
| `field.get(int row, int column)` | Reads one cell. |
| `field.set(int row, int column, value)` | Writes one cell. Returns the field. |
| `field.fill(value)` | Writes one value into every cell. Returns the field. |
| `field.row(int row)` | A writable array view of one row. Writes update the field. Do not resize it. |
| `field.bandCount()`, `field.band(int index)`, `field.bandStart(int index)`, `field.bandRows(int index)` | Direct access to the field's storage bands. A band array is row-major: the offset of a cell is `(row - bandStart(index)) * columns + column`. |

### Field stamps

They write into a field, clip to its bounds and return the field. Coordinates are rows and columns.

| Call | Description |
| --- | --- |
| `field.rect(int row0, int column0, int row1, int column1, value, bool filled = true)` | A filled or outlined rectangle. |
| `field.segment(int row0, int column0, int row1, int column1, value)` | A straight line of cells. |
| `field.circle(int centerRow, int centerColumn, int radius, value, bool filled = false)` | A ring of cells, or a disc when filled. |

### Canvas

| Call | Description |
| --- | --- |
| `pc.newCanvas(array<color> colors = na)` | Full opacity and a fixed value range of 0 to 1. The colours are exact palette entries for palette sources and gradient stops for value sources. `na` selects black and white. |
| `pc.heatmap(array<color> colors = na)` | A heatmap preset: automatic scaling, 64 colour levels, 15 to 90 opacity. `na` selects a thermal gradient. |
| `pc.gradient(color lowColor, color highColor)` | A two-stop colour array. |
| `pc.gradient(color color0, color color25, color color50, color color75, color color100)` | A five-stop colour array. |
| `canvas.palette(array<color> colors)` | Replaces the palette or gradient stops. 2 to 100 colours. |
| `canvas.colorLevels(int count)` | Samples a value source's gradient into 2 to 100 levels. Palette sources use their palette entries directly. |
| `canvas.opacity(float lowOpacity, float highOpacity = na)` | Opacity from the lowest level to the highest, 0 to 100. One argument gives every level the same opacity. |
| `canvas.autoScale(ScaleMode mode = ScaleMode.wholeGrid)` | Automatic normalization over the whole grid, each column or each row. |
| `canvas.valueRange(float minimum, float maximum)` | A fixed value range instead of automatic scaling, so colours stay comparable between updates. |
| `canvas.gamma(float value)` | The response curve, greater than 0. Below 1 lifts weak values, above 1 suppresses them. |
| `canvas.logarithmicY(bool enabled = true)` | Logarithmic row spacing. Needs positive vertical bounds. |
| `canvas.invertRows(bool enabled = true)` | Puts row 0 at the top. |
| `canvas.forceOverlay(bool enabled = true)` | Places the drawings on the main price chart. |
| `canvas.drawingLimit(int count)` | Caps the polylines this Canvas may own, 1 to 100. |
| `pc.newStyle(int colorLevels = 64, float lowOpacity = 15.0, float highOpacity = 90.0, bool autoScale = false, float valueMin = 0.0, float valueMax = 1.0, float gamma = 1.0, bool logarithmicY = false, bool invertRows = false, bool forceOverlay = false, ScaleMode scaleMode = ScaleMode.wholeGrid, int drawingLimit = 100)` | Builds a complete Style. |
| `canvas.configure(Style style)` | Applies a Style. |
| `canvas.clear()` | Deletes every polyline the Canvas owns. |

### Drawing fields

Every call returns the Canvas and replaces its previous drawings.

| Call | Description |
| --- | --- |
| `canvas.drawLookback(field, int lookback, float bottom, float top, int rightOffset = 0)` | Places a field ending at the current bar. `lookback` is the width in bars: 0 uses one bar per column, otherwise it must be at least the column count. |
| `canvas.drawLookback(scroll, float bottom, float top, int rightOffset = 0)` | Draws a rolling source at its current column count, newest column on the right. Needs at least two pushed columns. |
| `canvas.draw(field, int leftIndex, int rightIndex, float bottom, float top)` | Places a field between two bar indices. |
| `canvas.drawTime(field, int leftTime, int rightTime, float bottom, float top)` | Places a field between two UNIX timestamps in milliseconds. |
| `canvas.drawGrid(field, array<int> columnEdges, array<float> rowEdges)` | An uneven grid. Strictly increasing bar indices, `columns + 1` of them, and strictly increasing prices, `rows + 1` of them. |
| `canvas.drawTimeGrid(field, array<int> timeEdges, array<float> rowEdges)` | The same with timestamps in milliseconds. |

### Shapes

Shapes are queued in time and price, grouped by fill colour and packed into polylines when drawn.

| Call | Description |
| --- | --- |
| `canvas.rect(chart.point topLeft, chart.point bottomRight, color fillColor)` | Queues a rectangle from two opposite corners. |
| `canvas.triangle(chart.point cornerA, chart.point cornerB, chart.point cornerC, color fillColor)` | Queues a triangle. |
| `canvas.circle(chart.point centre, int radiusTime, float radiusPrice, int segments, color fillColor)` | Queues an ellipse. `radiusTime` is in milliseconds, `radiusPrice` in price units, `segments` from 3 to 9,999. |
| `canvas.poly(array<chart.point> points, color fillColor)` | Queues any closed polygon of 3 to 9,999 points, convex or concave. |
| `canvas.drawShapes()` | Draws the queue, replaces the Canvas's previous drawings, empties the queue and returns the number of polylines created. |
| `canvas.clearShapes()` | Discards the queue without drawing. |
| `canvas.shapeCount()` | Number of queued shapes. |
| `pc.barsToTime(int offsetBars)` | Estimates a timestamp from a bar offset on the current timeframe. It does not account for session gaps. |

One polyline holds about 1,600 four-corner rectangles of one colour. A batch accepts up to 100,000 source vertices, which is 25,000 rectangles. Same-colour overlaps form a union. Borders and per-shape layering are not part of the fill.

### Inspection

| Call | Description |
| --- | --- |
| `canvas.drawingCount()` | Polylines the Canvas currently owns. |
| `canvas.componentCount()` | Connected same-colour regions found by the last field render. |
| `canvas.dimensions()` | A `[rows, columns]` tuple from the last field render. |
| `canvas.scaledRange()` | A `[minimum, maximum]` tuple, the value range the last field render used. |

## Complete examples

Each one is a whole indicator.

### Rolling heatmap

A `ValueScroll` keeps a moving window when one new column arrives each bar. Five RSI lengths become five rows, one column per confirmed bar.

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

### Heatmap placed on price

The pattern for a heatmap anchored to real price levels. The model belongs to the calling script; this fragment is not a complete indicator.

```pine
// The calling script computes estimatedExposure, chooses row and column
// indices, and supplies its own price bounds.
var pc.ValueField levels = pc.valueField(240, 400)
var pc.Canvas heat = pc.heatmap()
  .colorLevels(48)
  .opacity(10, 90)
  .gamma(0.7)

// For each calculated level:
levels.set(row, column, estimatedExposure)

// After the model has populated the field:
if barstate.islast
    heat.drawLookback(levels, 0, bottomPrice, topPrice)
```

### A Pine3D scene as an image

Pine3D builds the scene, its `rasterize()` call writes the frame into a palette field, and PineCanvas draws it.

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

The 384 x 240 frame fits in one storage band. Larger frames are split across bands, each written through `band()` and `bandStart()`.

### Filled shapes

One rectangle per candle, more opaque the heavier its volume, then one `drawShapes()` call.

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

## Limits

- Drawing budgets are shared. Every Canvas in a script shares the 100 polyline limit with any other polylines. `drawingLimit()` caps one Canvas; it does not coordinate a global budget.
- Complexity sets the practical resolution. More colours and more fragmented boundaries need more geometry. Memory and execution time remain limits even when the source fits in storage.
- Shapes are fills, not outlined objects. A packed batch is grouped by colour, and same-colour overlaps form a union.
- Bar-index drawings are bound to 10,000 bars back and 500 bars forward. Use time placement for farther history.
- Each Canvas owns its drawings. Field drawing and shape drawing both replace that Canvas's previous frame.
- Animation follows Pine executions, not a timer. Chart anti-aliasing can show thin seams between adjacent fills at some zoom levels.
- PineCanvas reuses cached contour work and skips the redraw when the source, style and placement are unchanged.

TradingView's own Pine Script limits still apply: https://www.tradingview.com/pine-script-docs/writing/limitations/

## Runtime errors and what they mean

| Message | Cause |
| --- | --- |
| `lookback must be 0 or >=columns` | `drawLookback()` was given a lookback smaller than the field's column count. |
| `bar index is negative` | The picture's left edge is before the first bar. Wait for more bars or draw fewer columns. |
| `X is >10000 bars back; use time` | The left edge is more than 10,000 bars back. Use `drawTime()`. |
| `X is >500 future bars` | The right edge is more than 500 bars ahead of the current bar. |
| `yMax must exceed yMin` | `top` is not greater than `bottom`. |
| `log Y requires yMin>0` | `logarithmicY()` is on and `bottom` is zero or negative. |
| `need N drawings, above this Canvas's drawingLimit.` | The picture needs more polylines than its Canvas may use. Use fewer colours or less fragmented detail, or raise `drawingLimit()` up to 100. |
| `palettes needs 2..100 colors` | A palette or gradient has fewer than 2 or more than 100 colours. |
| `colorLevels must be 2..100`, `opacity must be 0..100`, `gamma must be >0` | A style value is out of range. |
| `valueRange requires max>min` | `valueRange()` was given a maximum that is not above the minimum. |
| `push size must equal scroll rows` | The array given to `push()` does not have exactly `rows` values. |
| `field cell out of bounds`, `field row out of bounds` | A row or column index is outside the field. |
| `shape corners need from_time()` | A shape point was not created with `chart.point.from_time()`. |
| `shape queue >100000 vertices` | More than 100,000 vertices were queued before `drawShapes()`. |
| `shape colors exceed 100` | The queued shapes use more than 100 distinct fill colours. |
| `shapes exceed drawingLimit` | The queued shapes need more polylines than the Canvas may use. |
| `Canvas is na; use newCanvas()` | A method was called on a Canvas that was never created. |

## Documentation

The full documentation goes into each topic in more depth:

- Introduction: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview
- Quickstart and lifecycle: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/quickstart
- Core concepts: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/core-concepts
- Value and palette fields: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/fields
- Rolling sources: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/rolling-sources
- Canvas and styling: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/canvas
- Placement and coordinates: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/placement
- Filled shapes: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/shapes
- Layers and drawing budgets: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/layers
- Heatmaps on price: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/heatmaps
- Images and Pine3D frames: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/images-pine3d
- Performance and large images: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/performance
- Common pitfalls: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/troubleshooting
- API reference: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/api-reference
- License: https://docs.alienalgorithms.com/PineCanvas-Graphics-Engine-Tradingview/license

## Related

- Pine3D, a 3D rendering engine for Pine Script v6 that rasterizes scenes into PineCanvas fields: https://alienalgorithms.com/pine3d
- Liquidation Heatmap, a free TradingView indicator drawn with PineCanvas: https://alienalgorithms.com/liquidation-heatmap
- NeuraLib, a machine learning runtime for Pine Script v6: https://alienalgorithms.com/neurallib
