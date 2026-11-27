# Measuring: renderer.info, Frame Timing, Profilers

Purpose: every `renderer.info` field on WebGLRenderer and WebGPURenderer with when it resets, accumulating over multi-render frames, and timing frames correctly (fields checked against the r187 source: `src/renderers/webgl/WebGLInfo.js` and `src/renderers/common/Info.js`).

## WebGLRenderer `renderer.info`

| Field | Meaning |
| --- | --- |
| `render.calls` | Draw calls in the last `render()` (a multi-draw `BatchedMesh` counts once) |
| `render.triangles`, `render.lines`, `render.points` | Primitives in the last `render()`, instances multiplied in |
| `render.frame` | Frame counter |
| `memory.geometries`, `memory.textures` | Live GPU uploads (counts, not bytes); drop when `dispose()` runs |
| `programs` | Array of compiled shader programs in the cache (`programs.length`) |
| `autoReset` (default `true`), `reset()` | Whether `render()` clears the per-render counters |

```javascript
function animate() {
  renderer.render(scene, camera);
  console.table({
    calls: renderer.info.render.calls, // draw calls submitted by this render
    triangles: renderer.info.render.triangles,
    lines: renderer.info.render.lines,
    points: renderer.info.render.points,
    geometries: renderer.info.memory.geometries,
    textures: renderer.info.memory.textures,
    programs: renderer.info.programs.length,
  });
}
renderer.setAnimationLoop(animate);
```

- The counters reset inside every `render()` (after the render list is built, before the shadow pass) while `autoReset` is true. Reading them before `render()` reports the previous frame, and a frame with several `render()` calls (picking, reflections, post passes) shows only the last one.
- Shadow-map drawing runs through the same counters: each shadow-casting light re-draws its casters inside `render()`, and those draws appear in `calls` and `triangles`.
- Geometries and textures upload once and are shared by every mesh that references them.

When a frame has several `render()` calls, accumulate over the whole frame:

```javascript
renderer.info.autoReset = false; // render() stops clearing the counters

function animate() {
  renderer.render(scene, camera);
  // ... further render() calls of this frame
  console.log(renderer.info.render.calls); // every render of this frame, summed
  renderer.info.reset(); // once per frame, after the last render
}
renderer.setAnimationLoop(animate);
```

## WebGPURenderer `renderer.info`

| Field | Resets per frame | Meaning |
| --- | --- | --- |
| `render.drawCalls` | yes | Draw calls this frame (the number to compare with WebGL's `render.calls`) |
| `render.frameCalls` | yes | Render calls (each `render()` plus internal ones such as the output pass) this frame |
| `render.triangles`, `render.lines`, `render.points` | yes | Primitives this frame |
| `render.calls` | no | Render calls since the app started |
| `compute.frameCalls` | yes | `compute()` calls this frame |
| `compute.calls` | no | Compute calls since start |
| `calls` | no | Render plus compute calls since start |
| `frame` | — | Frame id from the node frame |
| `memory.total` | live | Byte estimate over attributes, index, storage and indirect buffers, readback buffers, textures, uniform buffers and programs |
| `memory.*Size` | live | The parts: `attributesSize`, `indexAttributesSize`, `storageAttributesSize`, `indirectStorageAttributesSize`, `readbackBuffersSize`, `texturesSize`, `uniformBuffersSize`, `programsSize` |
| `memory.geometries`, `memory.textures`, `memory.renderTargets`, ... | live | Counts |

- With `autoReset` true the per-frame fields are cleared at the start of each animation-loop tick (`setAnimationLoop`), not inside `render()`. Apps that drive their own loop set `autoReset = false` and call `renderer.info.reset()` once per frame.
- Texture bytes are estimates: width × height × depth × bytes per pixel, ×1.333 for a generated mip chain; a compressed texture counts as 1 byte (a placeholder), so KTX2 memory is under-reported. Program size is the shader source length.

```javascript
import * as THREE from "three/webgpu";

function animate() {
  renderer.render(scene, camera);
  const { render, memory } = renderer.info;
  console.log(`${render.drawCalls} draw calls, ${render.triangles} triangles, ${(memory.total / 1048576).toFixed(1)} MB estimated`);
}
renderer.setAnimationLoop(animate);
```

## Frame Timing

`Timer` gives the frame interval; average it over a second of frames, single frames jitter too much to compare. The interval never drops below one display refresh (16.7 ms at 60 Hz), so it shows dropped frames but not headroom. Time the CPU side with `performance.now()` around the work:

```javascript
const timer = new THREE.Timer();
timer.connect(document); // pauses while the tab is hidden

let interval = 0;
let cpu = 0;
let frames = 0;

function animate() {
  timer.update();
  interval += timer.getDelta(); // seconds since the previous frame
  const start = performance.now();
  renderer.render(scene, camera);
  cpu += performance.now() - start; // CPU time spent in render(); the GPU executes the commands later
  frames++;
  if (frames === 60) {
    console.log(`${((interval / frames) * 1000).toFixed(2)} ms per frame, ${(cpu / frames).toFixed(2)} ms CPU in render()`);
    interval = 0;
    cpu = 0;
    frames = 0;
  }
}
renderer.setAnimationLoop(animate);
```

To tell CPU-bound from GPU-bound: if CPU time in `render()` plus your update code is well under the frame interval while frames still drop, the GPU is the limit (fill rate, shadows, post-processing). Halving the pixel ratio is the quickest GPU test: if frame time improves, it is fragment-bound.

## Profilers

- Chrome DevTools Performance panel: long main-thread tasks, scripting versus rendering time, GC pauses, and GPU work on its GPU track.
- The `stats-gl` package shows a live per-frame statistics overlay when you want the numbers always visible (it replaced the removed `GPUStatsPanel` addon in r169).
- A WebGL frame-capture extension such as Spector.js lists one frame's draw calls one by one, useful to see what each call draws.
