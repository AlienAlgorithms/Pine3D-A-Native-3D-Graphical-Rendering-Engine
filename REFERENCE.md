# Pine3D

> Pine3D is an open-source 3D rendering engine for TradingView, published as a Pine Script v6 library by Alien_Algorithms. It draws meshes, surfaces, 3D bar charts, streamed trails and scatter clouds on a chart, with a perspective camera, directional and point lights, shadows, depth sorting and occlusion culling, inside TradingView's drawing limits.

- Product page: https://alienalgorithms.com/pine3d
- Documentation: https://docs.alienalgorithms.com/Pine3D-Graphics-Engine-Tradingview
- Library and source code on TradingView: https://www.tradingview.com/script/I2K4hIdP-Pine3D-A-Native-3D-Graphical-Rendering-Engine/
- Source code, examples and this reference on GitHub: https://github.com/AlienAlgorithms/Pine3D-A-Native-3D-Graphical-Rendering-Engine
- Import: `import Alien_Algorithms/Pine3D/5 as p3d` (use the latest version number shown on the TradingView publication)
- Language: Pine Script v6
- Author: Alien_Algorithms, https://www.tradingview.com/u/Alien_Algorithms/?aff_id=169033#published-scripts
- Licence: CC BY-NC-SA 4.0. Free for non-commercial use when Alien_Algorithms is credited. Commercial use needs a licence from Alien_Algorithms.

This file is the plain-text reference for the product page. It is written so that a person or an AI assistant can write correct Pine3D code from it. Names, fields and defaults are taken from the library source, version 5.

The API is chainable and addressed by tags, so creating a mesh, adding it, aiming the camera at it and drawing it takes four lines:

```pine
p3d.Mesh box = p3d.cube(40.0, color.orange).setTag("hero").rotateBy(0.0, 45.0, 0.0)
scene.add(box)
scene.lookAt("hero")
p3d.render(scene)
```

## Contents

1. Setup requirements
2. Quick start: your first scene
3. The scene lifecycle
4. Meshes and primitives
5. Transforms and animation
6. The camera
7. Lighting and shadows
8. Contour surfaces
9. Bars3D: categorical 3D bar charts
10. Trail3D: streamed oscillator paths
11. Scatter clouds
12. Overlay primitives: labels, lines, polylines
13. Custom meshes and merging
14. The renderer
15. Rasterizing a scene for PineCanvas
16. Mesh drawing modes
17. Performance guide
18. Common pitfalls
19. API quick reference
20. Licence

## Setup requirements

Pine3D draws with polylines, lines and labels at the same time, so every script that uses it raises all three budgets in its `indicator()` declaration and imports the library under an alias:

```pine
//@version=6
indicator("My 3D Scene", overlay = false,
     max_polylines_count = 100,
     max_lines_count     = 500,
     max_labels_count    = 500)

import Alien_Algorithms/Pine3D/5 as p3d
```

These are TradingView's hard maximums. Pine3D's job is to fit as much geometry as possible inside them (see Contour surfaces for how far that goes).

## Quick start: your first scene

A complete, lit, animated 3D scene:

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

What happens:

1. The `Scene` and the mesh live in `var` variables, so they persist across bars.
2. Geometry is built once, in `barstate.isfirst`.
3. On the last bar the mesh is changed in place (`rotateBy`) and the frame is drawn with one `render()` call.

Build once, change later, render once per frame: that is the lifecycle for every Pine3D script.

## The scene lifecycle

The recommended order:

1. Create one persistent `Scene` with `p3d.newScene()` in a `var`. It comes with a default camera at (0, 0, 500) looking at the origin, and a default directional light.
2. Build meshes and overlays once, in `barstate.isfirst`.
3. On later bars, change things in place: transforms (`rotateBy`, `moveTo`), data updates (`updateSurface()`, `updateBars()`, `trail.pushSample()`), or direct field assignment on labels and lights.
4. Call `render(scene, ...)` once per frame. It clears the previous frame's drawings by itself.

Two calls that are easy to confuse:

- `scene.clear()` removes objects from the scene graph (meshes, labels, lines, polylines). The build-once pattern rarely needs it.
- `render()` clears only the previous frame's TradingView drawings, then draws again from the current scene graph.

## Meshes and primitives

### Ready-made geometry

```pine
p3d.Mesh box      = p3d.cube(40.0, color.orange)               // uniform cube
p3d.Mesh brick    = p3d.cuboid(18.0, 14.0, 10.0, color.red)    // width, height, depth
p3d.Mesh ball     = p3d.sphere(50.0, 16, 12, color.aqua)       // radius, segments, rings
p3d.Mesh tube     = p3d.cylinder(20.0, 60.0, 12, color.lime)
p3d.Mesh donut    = p3d.torus(40.0, 12.0, 24, 12, color.fuchsia)
p3d.Mesh tip      = p3d.pyramid(30.0, 50.0, color.yellow)
p3d.Mesh floor    = p3d.plane(200.0, 200.0, color.gray)
p3d.Mesh ringFlat = p3d.circle(30.0, 48, color.blue)
```

Also available: `grid(w, d, divX, divZ, col)`, `disc(radiusX, radiusZ, seg, col)`, and the low-level `customMesh(verts, faces)` (see Custom meshes and merging).

### Appearance

Every appearance method returns the `Mesh`, so they chain:

```pine
box := box
  .setColor(color.new(color.orange, 20))
  .setStyle("wireframe")     // "solid" (default), "wireframe", "wireframe_front"
  .setTag("hero")
```

`setFaceColor(faceIndex, col)` colours one face, and `show()` / `hide()` toggle visibility.

Stroke styling for wireframe edges, and for face outlines in `"poly"` draw mode, is set by direct field assignment:

```pine
box.lineWidth := 3
box.lineStyle := line.style_dashed
```

### The tag-based scene graph

Every `Mesh`, `Label3D`, `Line3D` and `Polyline3D` can carry a string tag, so the scene can be searched by name:

```pine
scene.add(p3d.sphere(15, 10, 8, color.blue).setTag("planet"))

// later, on any bar:
p3d.Mesh planet = scene.getMesh("planet")
if not na(planet)
    planet.rotateBy(0, 2, 0)

scene.lookAt("planet")     // aim the camera at a tagged mesh
scene.remove("planet")     // remove by tag
```

`remove(tag)` searches meshes, then labels, then lines, then polylines, and removes the first match only. Do not reuse one tag across object types if you delete by tag.

## Transforms and animation

Transform methods change the mesh in place and return it for chaining:

```pine
mesh.moveTo(0, -20, 0)        // absolute position
mesh.moveBy(5, 0, 0)          // relative move
mesh.rotateTo(0, 45, 0)       // absolute rotation, in degrees
mesh.rotateBy(0, 1.5, 0)      // relative rotation, in degrees
mesh.scaleTo(2.0, 1.0, 1.0)   // per-axis scale
mesh.scaleUniform(1.5)        // uniform scale
```

Chained: `mesh.moveTo(0, -20, 0).rotateTo(0, 45, 0).setStyle("solid")`.

Animation is calling these on later bars, usually inside `barstate.islast`, before `render()`. The transform methods invalidate the mesh's cached world vertices automatically.

Degrees and radians: `rotateTo()` and `rotateBy()` on a Mesh take degrees. The low-level `Vec3` methods `rotateX()`, `rotateY()` and `rotateZ()` take radians.

If you write to `mesh.vertices` directly (for example `mesh.vertices.set(i, newVec)`), call `mesh.invalidateCache()` yourself, or the renderer draws stale geometry.

## The camera

Every scene has a perspective camera at `scene.camera`:

```pine
scene.camera.setPosition(0, 100, 300)   // place the camera
scene.camera.lookAt(0, 20, 0)           // aim at a world-space point (also accepts a Vec3)
scene.camera.orbit(35.0, 25.0, 380.0)   // spherical orbit: yaw, pitch, distance from target
scene.camera.setFov(700.0)              // perspective scale factor
```

`orbit()` is the everyday call: set a target with `lookAt()` (or keep the origin), then orbit around it. For a slow rotation, call `orbit()` each frame with an angle driven by `bar_index`.

The camera's fields `position`, `target` and `fov` can also be assigned directly. A higher `fov` gives less distortion, a lower one a more dramatic perspective. The default is 800.

```pine
scene.camera.fov := 1200.0
```

`scene.lookAt(tag)` on the Scene takes a mesh tag only. To aim at a point, use `scene.camera.lookAt(x, y, z)` or `scene.camera.lookAt(vec)`.

## Lighting and shadows

Shading is off by default. Turn it on per frame with `render(scene, lighting = true)`.

### Configuring the scene light

The scene-level lighting setters chain:

```pine
// Directional light (sun-style, parallel rays)
scene.setLightDir(1.0, -1.0, 0.5).setAmbient(0.3).setShadowStrength(0.6)

// Point light (bulb-style, radiates from a position)
scene.setLightMode("point")
  .setLightPos(0.0, 60.0, 150.0)
  .setAmbient(0.2)
  .setShadowStrength(0.7)
  .showLightSource(true)      // draws a marker where the light sits
```

- A light direction is the way the light travels, so it points away from the light. `setLightDir(1.0, -1.0, 0.5)` lights the scene from above.
- `setAmbient()` (0.0 to 1.0) sets the minimum brightness of faces turned away from the light.
- `setShadowStrength()` (0.0 to 1.0) sets how dark cast shadows are. The default is 0, no shadows.
- `showLightSource(true)` helps while composing a scene.

### Light fields

Every field of `scene.light` can be assigned directly, which is also how the light is animated:

| Field | Meaning | Default |
|---|---|---|
| `mode` | `"directional"` or `"point"` | `"directional"` |
| `direction` | Directional light: the way the light travels | (0.4, -0.7, -0.5) on a new scene |
| `position` | Point light: world position of the source | na |
| `ambient` | Minimum brightness, 0.0 to 1.0 | 0.3 |
| `shadowStr` | Shadow strength, 0.0 (none) to 1.0 (full) | 0.0 |
| `shadowBias` | Offset that removes shadow acne | 0.5 |
| `selfShadow` | Lets a mesh shadow its own faces | false |
| `showSource` | Draws a marker at a point light | false |
| `enabled` | Master switch for the scene light | true |

```pine
scene.light.selfShadow := true      // meshes shadow themselves
scene.light.shadowBias := 0.2       // adjust the shadow acne offset

// A point light circling the scene:
float a = bar_index * 0.03
scene.light.position := p3d.Vec3.new(80.0 * math.cos(a), 60.0 + 50.0 * math.sin(a), 200.0)
```

Per mesh: `mesh.castShadow := false` stops a mesh casting shadows while it still receives light. Useful for ghost overlays and debug geometry.

### A complete lit scene

Adapted from the official Lights Demo: a point-lit sphere in front of a brick wall built from merged cuboids.

```pine
//@version=6
indicator("Pine3D - Lights Demo", overlay = false, max_polylines_count = 100, max_lines_count = 500, max_labels_count = 500)
import Alien_Algorithms/Pine3D/5 as p3d

var p3d.Scene scene = p3d.newScene()

if barstate.isfirst
    scene.camera.lookAt(0, 20, 0)
    scene.camera.orbit(70, 15, 500)

    p3d.Mesh planet = p3d.sphere(15, 10, 8, color.new(#4488ff, 10))
    planet.moveTo(0, 40, 60).setTag("planet")
    scene.add(planet)

    // Brick wall: many cuboids merged into one mesh, interior faces stripped
    array<p3d.Mesh> bricks = array.new<p3d.Mesh>()
    for r = 0 to 9
        for c = 0 to 19
            p3d.Mesh brick = p3d.cuboid(18.0, 14.0, 10.0, color.new(#cc6633, 5))
            brick.moveTo(-171.0 + c * 18.0, r * 14.0, -120.0)
            bricks.push(brick)
    scene.add(p3d.mergeMeshes(bricks, "wall", removeInterior = true))

    scene.setLightMode("point")
      .setLightPos(0.0, 60.0, 150.0)
      .setAmbient(0.2)
      .setShadowStrength(0.7)
      .showLightSource(true)
    p3d.wireGrid(scene, 400, 400, 8, 8, color.new(color.gray, 85))

if barstate.islast
    float a = bar_index * 0.03
    scene.light.position := p3d.Vec3.new(80.0 * math.cos(a), 60.0 + 50.0 * math.sin(a), 200.0)
    p3d.render(scene, maxFaces = 400, culling = true, lighting = true, occlusion = true)
```

Performance note: the shadow grid that speeds up shadow tests is built for directional lights only. Point lights fall back to a linear scan, so shadow-heavy scenes run fastest with a directional light.

## Contour surfaces

Pine Script caps every indicator at 100 polylines and 500 lines. Drawn one triangle at a time, that budget runs out after a few hundred triangles. Pine3D's `surface()` uses contour band rendering instead: the surface's height is split into coloured bands, and every cell in one band, connected or not, is joined into a single polyline, with invisible zero-width bridges between separate islands.

The practical result:

- One polyline can carry up to about 2,000 separate triangle-equivalents.
- A 40 x 40 heightmap (about 3,000 triangles) renders inside the budget with full per-band colouring; stress tests have run 40 x 80 grids.
- Each band's path is depth-sorted, near-plane culled and cached between bars. After the build, only the screen projection runs per frame.

### Creating a surface

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
      .gridBox()                                      // wireframe bounding cage
      .gridLabels(color.white, "X", "Amplitude", "Z") // axis titles and tick values
    scene.add(heatmap)
    scene.camera.orbit(35.0, 25.0, 380.0)

if barstate.islast
    p3d.render(scene, lighting = true)
```

Signature: `surface(matrix<float> heights, float size, color lowCol, color highCol, int levels = 20, array<float> axisX = na, array<float> axisZ = na)`. `levels` is the number of contour bands, and colours grade from `lowCol` to `highCol`.

### Non-uniform grids

The optional `axisX` and `axisZ` arrays replace uniform spacing with your own column and row positions, for example logarithmic strike spacing for a volatility surface or irregular timestamps for a depth heatmap, with no resampling. Contour bands, axis ticks and the `gridBox()` cage all follow the custom positions.

### Animating: updateSurface()

Never rebuild the mesh per bar. Refresh the data in place:

```pine
if barstate.islast
    heatmap.updateSurface(newDataMatrix)
    p3d.render(scene, lighting = true)
```

`updateSurface(heights, size, lowCol, highCol, levels, axisX, axisZ)` takes the same optional arguments as `surface()`. Tick labels from `gridLabels()` follow the new data range; axis titles stay.

Global-scope rule: any data feeding `updateSurface()` that uses the history operator (`[]`) or calls such as `ta.rsi()`, `ta.atr()` or `request.security()` must be computed at global scope, not inside `barstate.islast`, so Pine keeps its bar-by-bar history correctly.

## Bars3D: categorical 3D bar charts

`bars3D()` turns an array of values into a lit, depth-sorted 3D bar chart in one call. Bar heights follow the values, colours grade between a low and a high colour, and all bars are packed into one merged mesh that still sorts each bar correctly.

```pine
//@version=6
indicator("Pine3D - Bars3D", overlay = false, max_polylines_count = 100, max_lines_count = 500, max_labels_count = 500)
import Alien_Algorithms/Pine3D/5 as p3d

var p3d.Scene scene = p3d.newScene()
var p3d.Mesh  bars  = na
array<float>  values = array.from(volume - volume[1], volume[1] - volume[2], volume[2] - volume[3],
                                  volume[3] - volume[4], volume[4] - volume[5], volume[5] - volume[6])
array<string> names  = array.from("dV0", "dV-1", "dV-2", "dV-3", "dV-4", "dV-5")

if barstate.isfirst
    bars := p3d.bars3D(values, 30.0, 30.0, 10.0, color.blue, color.red, 200.0)
      .barLabels(names)      // category labels at the base of each bar
    scene.add(bars)
    p3d.wireGrid(scene, 300.0, 300.0, 6, 6, color.new(color.gray, 80))
    scene.camera.orbit(215.0, 25.0, 360.0)

if barstate.islast
    bars.updateBars(values)  // refresh heights, colours and value labels in place
    p3d.render(scene, lighting = true)
```

Signature: `bars3D(array<float> values, float barWidth, float barDepth, float spacing = 4.0, color lowCol = #2196F3, color highCol = #FF5722, float maxHeight = 100.0)`.

How it behaves:

- `updateBars(values)` refreshes heights, colours and label positions every frame without rebuilding the geometry, for live rankings and rolling windows.
- Labels: call `barLabels(names)` once at build time for category labels. Value labels on top of each bar are handled by `updateBars()`: leave out `valLabels` and each value is formatted with `str.tostring()`, or pass `valLabels` for your own strings. Do not call `barLabels()` again after the build.
- Negative values extrude downward from the base plane, with the face winding reversed automatically, so signed series such as PnL or momentum histograms work as they are.

## Trail3D: streamed oscillator paths

`trail3D()` draws two related series as a 3D ribbon moving through time: phase portraits, Lissajous figures, RSI against MFI, momentum against volatility, any (u, v) pair.

You set a rolling capacity and push one sample per bar. The trail keeps the buffer, builds the ribbon and normalises samples against the rolling window's minimum and maximum, so the path always fills its bounding cube whatever the data's scale.

```pine
//@version=6
indicator("Pine3D - Trail3D", overlay = false, max_polylines_count = 100, max_lines_count = 500, max_labels_count = 500)
import Alien_Algorithms/Pine3D/5 as p3d

var p3d.Scene   scene = p3d.newScene()
var p3d.Trail3D trail = na

if barstate.isfirst
    trail := p3d.trail3D(220.0, 200, color.yellow)   // size, capacity, colour
      .cage(true)                                    // wireframe bounding cage
      .axisLabels("sin", "cos", color.white)
    trail._uProj.col := #00ffff69                    // wall projection colours
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

A Trail3D is four polylines drawn together: the main ribbon, two shadow projections onto the back wall and the floor, and the cage. All four are depth-sorted and occlusion-clipped against the rest of the scene. To hide one, set its `visible` field, for example `trail._cage.visible := false`.

Details:

- Signature: `trail3D(float size, int capacity = 300, color trailCol = #FFEB3B, int minSamples = 60, bool rollingBounds = true)`.
- `capacity` is capped at 300 samples. Larger values become 300.
- `minSamples` (default 60) is the sample count at which the cage reaches its cube width. Below it, samples stretch across a cube-shaped cage. Above it, the cage grows leftward at a fixed width per sample until it reaches `capacity`.
- `rollingBounds = true` recomputes the minimum and maximum from the current window on every push, so samples that dropped out no longer affect the scaling. With `false`, the bounds only ever expand.
- Methods: `pushSample()`, `axisLabels()`, `cage()`, `moveTo()`, `show()`, `hide()`.

## Scatter clouds

There is no separate scatter API: `Label3D` is the point primitive, and `scene.add(array<Label3D>)` registers a whole cloud in one call. Up to 500 points, each with its own colour, symbol, size and tooltip, all depth-sorted and occlusion-clipped.

The pattern: build the array once, add it, then change point fields on later bars to animate.

```pine
//@version=6
indicator("Pine3D - Scatter Cloud", overlay = false, max_polylines_count = 100, max_lines_count = 500, max_labels_count = 500)
import Alien_Algorithms/Pine3D/5 as p3d

var p3d.Scene scene = p3d.newScene()
var array<p3d.Label3D> points = array.new<p3d.Label3D>()

if barstate.isfirst
    for i = 0 to 499
        p3d.Vec3 pos = p3d.vec3(0.0, 0.0, 0.0)
        points.push(p3d.Label3D.new(position = pos, txt = "o"))
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

A point's fields include `position`, `txt`, `textColor`, `bgColor`, `labelStyle` (any `label.style_*` constant: circles, squares, diamonds, triangles, crosses, arrows, flags), `labelSize` (any `size.*` constant), `fontFamily`, `tooltip` and `visible`. The renderer reads them every frame, so animation is plain field assignment. For a dot instead of a character, set `txt = ""` and `labelStyle = label.style_circle`.

Uses: k-means views, particle systems, point-sampled parametric surfaces, classification overlays and colour-graded attractors.

## Overlay primitives: labels, lines, polylines

Besides meshes, three world-space overlay types can be added to a scene. They are depth-sorted and occluded against everything else. They are plain user-defined types with many fields, so create them with named arguments:

```pine
// A floating annotation
p3d.Label3D tagLabel = p3d.Label3D.new(position = p3d.vec3(0, 80, 0), txt = "Peak", textColor = color.white)
scene.add(tagLabel)

// A 3D line segment
p3d.Line3D axis = p3d.Line3D.new(start = p3d.vec3(0, 0, 0), end = p3d.vec3(0, 100, 0), col = color.yellow, width = 2)
scene.add(axis)

// A 3D polyline path
array<p3d.Vec3> pts = array.from(p3d.vec3(0,0,0), p3d.vec3(50,30,0), p3d.vec3(100,0,50))
p3d.Polyline3D path = p3d.Polyline3D.new(points = pts, col = color.aqua, width = 2)
scene.add(path)
```

Methods:

- `Label3D`: `moveTo()`, `moveBy()`, `setText()`, `setTextColor()`, `setTooltip()`, `show()`, `hide()`, `setTag()`.
- `Line3D`: `setStart()`, `setEnd()`, `setPoints()`, `setColor()`, `show()`, `hide()`, `setTag()`.
- `Polyline3D`: `setColor()`, `show()`, `hide()`, `setTag()`.

Fields without a setter are assigned directly and read every frame:

- `Label3D`: `bgColor`, `labelStyle`, `labelSize`, `fontFamily`, `visible`.
- `Line3D`: `width`, `lineStyle` (solid, dashed, dotted and arrow styles), `visible`.
- `Polyline3D`: `width`, `lineStyle` (solid, dashed and dotted only; TradingView polylines have no arrow styles), `fillColor`, `closed`, `visible`.

`wireGrid(scene, w, d, divX, divZ, col)` adds a depth-sorted ground grid to a scene.

## Custom meshes and merging

### customMesh()

When the primitives are not enough, build your own topology from vertices and `Face` descriptors:

```pine
// A single triangle
array<p3d.Vec3> verts = array.from(p3d.vec3(-30, 0, 0), p3d.vec3(30, 0, 0), p3d.vec3(0, 50, 0))
array<p3d.Face> faces = array.new<p3d.Face>()
faces.push(p3d.Face.new(vi = array.from(0, 1, 2), col = color.orange))
p3d.Mesh tri = p3d.customMesh(verts, faces)
scene.add(tri)
```

`Face.vi` holds vertex indices into the mesh. Every face needs at least 3 indices and should be planar.

### mergeMeshes()

`mergeMeshes(array<Mesh> meshes, string tag = "", bool removeInterior = false)` applies each mesh's transform and combines them into one mesh. Two behaviours matter:

- `removeInterior = true` finds coincident faces with opposite normals (such as the shared wall between two adjacent cubes) and removes them, so voxel-style scenes keep only their outer shell and spend nothing on hidden faces. The brick wall in the lighting example does this.
- Depth grouping is kept when every source mesh has the same vertex and face count (identical primitives in a grid), so the merged mesh still sorts per original piece. Mixing different shapes turns the grouping off.

## The renderer

Every frame is one call:

```pine
p3d.render(scene, maxFaces = 100, culling = true, lighting = false,
     ambient = 0.3, wireframe = false, occlusion = true, occlusionRaster = 768)
```

The steps of each call: world transform, camera transform, back-face culling, occlusion culling, depth sort, lighting and shadows, perspective projection, drawing.

| Parameter | Default | What it does |
|---|---|---|
| `maxFaces` | 100 | Cap on solid faces per frame. Contour bands, wireframe edges, labels, lines and overlay polylines do not count against it; only TradingView's own budgets bound them. |
| `culling` | true | Back-face culling. |
| `lighting` | false | Diffuse shading. |
| `lightDir` | na | A light direction for this call only, the way the light travels. It replaces the scene light for this call. |
| `ambient` | 0.3 | Minimum brightness of faces in shadow. Used with `lightDir`, or when the scene light is disabled. |
| `wireframe` | false | Draws the whole scene as outlines only. |
| `occlusion` | true | A sparse raster pass that drops hidden faces before drawing. A large speed gain on dense scenes. |
| `occlusionRaster` | 768 | Resolution of that raster. Lower is faster but coarser. |

Which light is used: a `lightDir` passed to `render()` wins, together with `render()`'s `ambient`. Without it, the scene's light is used with its own settings, as long as `scene.light.enabled` is true. Otherwise the defaults apply.

`render()` clears the whole chart drawing pool at the start of every call: every polyline, line, label and linefill on the chart, not only Pine3D's. If the script also calls `label.new()` or `line.new()`, make those calls after `render()`, or they are wiped every frame. Tables are not affected, which the budget monitor below relies on.

## Rasterizing a scene for PineCanvas

`rasterize()` renders the scene's solid meshes per pixel instead of as chart drawings, with per-pixel depth, smooth shading with sharp creases, a specular highlight, and shadows when the scene light's shadow strength is above zero. It writes palette indices into an integer array, which a PineCanvas field then draws on the chart.

```pine
array<color> palette = scene.rasterize(pixels, width, height, rowStart = 0, shades = 16, zoom = 1.0, culling = true)
```

- `pixels` receives rows `rowStart` onward of a `width` x `height` frame, row-major, with row 0 at the bottom. Each cell is an index into the returned palette, or -1 where no mesh covers it.
- The returned palette holds `shades` colours per material (each distinct face colour), from the ambient floor up to white highlights.
- One pixel is one unit of `render()`'s screen space times `zoom`.
- A Pine array holds at most 100,000 elements, so a larger frame is rasterized in row bands: one call per band, each with its own array and `rowStart`.
- Wireframe and contour meshes are skipped. A mesh with `castShadow` off casts no shadow and still receives them.

The PineCanvas reference, https://alienalgorithms.com/pinecanvas.md, has a complete example that draws a rasterized Pine3D frame.

## Mesh drawing modes

Two separate settings control how a mesh appears.

Style, set with `mesh.setStyle(...)`, decides what is drawn:

| Style | Result |
|---|---|
| `"solid"` | Filled faces (default). |
| `"wireframe"` | Every edge, no fill. Shows interior geometry. |
| `"wireframe_front"` | Front-facing edges only. A cleaner outline on convex meshes. |

Draw mode, the `mesh.drawMode` field, decides which TradingView drawing carries solid faces:

| Mode | Notes |
|---|---|
| `"linefill"` (default) | Uses lines and linefills, and shares one edge line between adjacent faces, about 1.6 lines per face instead of two. That lifts a mesh from 250 faces to around 360, depending on topology. Supports in-place `updateSurface()` and `updateBars()`. Edges are transparent, so solid faces have no visible outline. Recommended for new code. |
| `"poly"` | The older polyline backend. About 100 faces, no in-place updates, but it draws face outlines with `mesh.lineStyle` and `mesh.lineWidth`. Use it only when solid faces need styled outlines. |

Wireframe styles always draw with lines, whatever the draw mode.

## Performance guide

Pine limits the CPU time per bar, and a dense scene can hit that limit before the drawing budget. If a scene compiles but times out, try these in order:

1. Lower `occlusionRaster` (for example from 768 to 384). The largest single gain.
2. Reduce `maxFaces` to cap the solid faces.
3. Use fewer contour `levels` on surfaces.
4. Use fewer segments on spheres and tori.
5. Keep heavy work behind `barstate.islast`, so historical bars build geometry without rendering it.

Structural choices that cost nothing:

- Build once and update with `updateSurface()` and `updateBars()`. Never rebuild geometry per bar.
- Use `mergeMeshes(..., removeInterior = true)` for voxel-style geometry.
- Prefer directional lights when shadows are heavy.
- Pine3D caches world vertices, view projections, face preprocessing, shadow grids and contour geometry by itself.

### Monitoring the budgets

Drawing budget overruns fail silently: TradingView drops the extra drawings without an error, so missing geometry almost always means a budget overrun. A usage table makes it visible, since `render()` does not clear tables:

```pine
var table stats = table.new(position.bottom_right, 3, 2, bgcolor = color.new(#000000, 30))
table.cell(stats, 0, 0, "Polyline", text_color = color.white, text_size = size.small)
table.cell(stats, 1, 0, "Line", text_color = color.white, text_size = size.small)
table.cell(stats, 2, 0, "Label", text_color = color.white, text_size = size.small)
table.cell(stats, 0, 1, str.tostring(polyline.all.size()) + " / 100", text_color = color.white, text_size = size.small)
table.cell(stats, 1, 1, str.tostring(line.all.size()) + " / 500", text_color = color.white, text_size = size.small)
table.cell(stats, 2, 1, str.tostring(label.all.size()) + " / 500", text_color = color.white, text_size = size.small)
```

### Positioning the viewport

`scene.guiShift` moves the scene right, into future-bar space, and `scene.yOffset` moves it vertically in price units. All drawings use `xloc.bar_time`, so the scene can sit far to the left or right of `bar_index` without making Pine extend its history buffer. This keeps the engine stable on long charts and on scenes drawn into the future.

## Common pitfalls

- Your own drawings are wiped every frame: they were drawn before `render()`. Call `label.new()` and `line.new()` after it.
- Geometry is missing: a drawing budget overrun. Lower `maxFaces`, drop a contour level or simplify overlays, and use the monitor table above.
- `updateSurface()` or `updateBars()` gives inconsistent data or compiler errors: the source series (`[]` history, `ta.*`, `request.security()`) was computed inside `barstate.islast`. Compute it at global scope.
- Rotation looks wrong: mesh rotations are in degrees, `Vec3` rotations in radians.
- Stale geometry after editing `mesh.vertices`: call `mesh.invalidateCache()` or `scene.invalidateAllCaches()`.
- `scene.lookAt()` does not aim at a point: it takes a tag only. Use `scene.camera.lookAt()` for coordinates.
- `remove(tag)` removed the wrong object: tags are searched in meshes, labels, lines, then polylines, and the first match is removed. Do not share tags across types.
- The trail ignores your capacity: it is capped at 300 samples.
- `barLabels()` was called every frame: call it once at build time and pass `valLabels` to `updateBars()` instead.
- A point-light scene is slow: point lights skip the shadow grid. Use a directional light if you can.
- No shadows appear: shadow strength defaults to 0. Call `setShadowStrength()` and render with `lighting = true`.

## API quick reference

### Entry points
`newScene()`, `render(scene, ...)`, `vec3(x, y, z)`, `colorBrightness(c, factor)`

### Mesh factories
- Primitives: `cube()`, `cuboid()`, `pyramid()`, `plane()`, `sphere()`, `cylinder()`, `torus()`, `grid()`, `disc()`, `circle()`
- Advanced: `customMesh(verts, faces)`, `mergeMeshes(meshes, tag, removeInterior)`, `surface(heights, size, lowCol, highCol, levels, axisX, axisZ)`, `bars3D(values, barWidth, barDepth, spacing, lowCol, highCol, maxHeight)`, `trail3D(size, capacity, trailCol, minSamples, rollingBounds)`

### Mesh methods
- Transform: `moveTo()`, `moveBy()`, `rotateTo()`, `rotateBy()`, `scaleTo()`, `scaleUniform()`
- Appearance: `setColor()`, `setFaceColor()`, `setStyle()`, `show()`, `hide()`, `setTag()`
- Direct fields: `lineWidth`, `lineStyle`, `drawMode`, `castShadow`, `visible`
- Lifecycle: `clone()`, `faceCount()`, `invalidateCache()`
- Data updates: `updateSurface()`, `updateBars()`
- Chainable overlays: `gridBox()` and `gridLabels()` (surfaces), `barLabels()` (bars3D)

### Scene methods
- Lighting: `setLightDir()`, `setLightPos()`, `setLightMode()`, `setAmbient()`, `setShadowStrength()`, `showLightSource()`
- Scene graph: `add(mesh | label | array<Label3D> | line | polyline | trail)`, `remove(index | tag)`, `clear()`
- Lookup: `getMesh()`, `getLabel()`, `getLine()`, `getPolyline()`, `lookAt(tag)`, `totalFaces()`
- Rendering to pixels: `rasterize(pixels, width, height, rowStart, shades, zoom, culling)`
- Cache control: `invalidateLightCache()`, `invalidateAllCaches()`
- Direct fields: `guiShift`, `yOffset`, `camera`, `light` (`mode`, `direction`, `position`, `ambient`, `shadowStr`, `shadowBias`, `selfShadow`, `showSource`, `enabled`)

### Camera
`setPosition()`, `lookAt(x, y, z | vec3)`, `orbit(angleX, angleY, distance)`, `setFov()`, direct fields `position`, `target`, `fov`

### Vec3
- Math: `add()`, `sub()`, `scale()`, `negate()`, `dot()`, `cross()`, `length()`, `normalize()`, `distanceTo()`, `lerp()`
- Rotation and helpers: `rotateX()`, `rotateY()`, `rotateZ()` (radians), `copy()`, `toString()`

### Overlay types (create with named arguments)
- `Label3D` (position, txt, textColor, bgColor, labelStyle, labelSize, fontFamily, tooltip, visible, tag): `moveTo()`, `moveBy()`, `setText()`, `setTextColor()`, `setTooltip()`, `show()`, `hide()`, `setTag()`
- `Line3D` (start, end, col, width, visible, tag, lineStyle): `setStart()`, `setEnd()`, `setPoints()`, `setColor()`, `show()`, `hide()`, `setTag()`
- `Polyline3D` (points, col, fillColor, width, closed, visible, tag, lineStyle): `setColor()`, `show()`, `hide()`, `setTag()`
- `Face` (vi, col): topology for `customMesh()`

### Trail3D
`pushSample()`, `axisLabels()`, `cage()`, `moveTo()`, `show()`, `hide()`

### Scene helpers
`wireGrid(scene, w, d, divX, divZ, col)`, `scene.add(array<Label3D>)` for scatter clouds

## Licence

Pine3D is licensed under CC BY-NC-SA 4.0 (https://creativecommons.org/licenses/by-nc-sa/4.0/). Use is free for non-commercial purposes when Alien_Algorithms (https://www.tradingview.com/u/Alien_Algorithms/?aff_id=169033#published-scripts) is credited in the description of any script that uses it. For commercial licensing, contact Alien_Algorithms on TradingView.
