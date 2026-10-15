# Changelog

Each library is published on TradingView. Every update there raises the version number in the import line, so use the newest one shown on the publication.

## Version 5 (2026-10-09)

Pine3D v5 greatly improves shadow quality for rasterize()

## Version 4 (2026-10-02)

Pine3D v4 adds Bug Fixes, Performance Gains and a new rasterize() function which renders a scene per pixel into a grid of palette indices, with a depth buffer, smooth shading, specular highlights and shadows.

NEW: rasterize()
- scene.rasterize(pixels, width, height) draws the scene's solid meshes into an array of palette indices, one per pixel (row-major, row 0 at the bottom, -1 where nothing is drawn), and returns the matching palette.
- Per-pixel depth, smooth shading across curved surfaces with sharp creases kept, Blinn-Phong highlights, and shadow mapping when the scene light's shadow strength is above zero.
- Uses the same camera and lights as render(). Optional parameters: rowStart (draw frames larger than one array in row bands), shades per color, zoom and culling.

PERFORMANCE
- Occlusion culling now visits only the samples each face covers instead of its whole bounding box. Cylinder caps and solid discs are much cheaper.
- When the camera and meshes are still, faces are reused instead of rebuilt every frame.
- Shadow queries look up a direct-indexed light grid instead of maps, and the grid is built in one pass.
- Leaner projection and per-face setup on every frame where something moves.
- Contour surfaces: faster band merging and faster updateSurface() rebuilds.

## Version 3 (2026-05-01)

- Implements PineScript's new UDT Sorting functionality for a 2x speedup on certain operations

- Optimizations for Surfaces, faster default-axis geometry setup

- Added Scene.yScale to scale projected scene height on price charts:

```pine
testScene.guiShift := bars_offset
testScene.yOffset := close_price - testScene.camera.fov
testScene.yScale := 100
```

## Version 2 (2026-04-22)

v2 FEATURE:

Added optional rollingBounds parameter when initializing trail3D.

Enabled: Dynamically adapts the two data stream bounds to fit along the min-max of the cage for the visible window.

Disabled: Expand-only behavior, which keeps the historical steps fixed but can cause large ranges to compress active window scaling.

Defaults to true / enabled.

## Version 1 (2026-04-21)

First publication.
