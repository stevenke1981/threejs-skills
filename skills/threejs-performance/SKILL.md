---
name: threejs-performance
description: "Measuring and speeding up three.js: renderer.info on WebGL and WebGPU, frame timing, cutting draw calls (merge, instance, batch, LOD), GPU memory and disposal, pixel ratio, overdraw, shadow cost, CPU work per frame, shader compile stalls. Use for low FPS, stutter on first use, memory that keeps growing, too many draw calls, or render.calls looking wrong under WebGPU. Not for the instancing or merge API (use threejs-geometry), shadow setup (use threejs-lighting), glTF compression (use threejs-asset-pipeline), engine-neutral profiling (use performance-optimization). 中文: 效能優化, 掉幀, 卡頓, 繪製呼叫, 記憶體洩漏, 著色器編譯"
---

# Three.js Performance

Targets three.js r187 (npm `three@0.187.0`). Examples use WebGLRenderer; `references/webgpu-tsl.md` covers WebGPURenderer. This skill is strategy and measurement; each lever points to the skill that owns its API.

## Quick Start: Measure One Frame

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // the cheapest fill-rate fix
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.add(new THREE.HemisphereLight(0xffffff, 0x445566, 1));
const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 2, 8);
const mesh = new THREE.Mesh(new THREE.SphereGeometry(1, 32, 16), new THREE.MeshStandardMaterial({ color: 0x4488ff }));
scene.add(mesh);

renderer.setAnimationLoop(() => {
  mesh.rotation.y += 0.01;
  renderer.render(scene, camera);
  // read AFTER render(): the counters reset inside every render() call
  const { render, memory, programs } = renderer.info;
  console.log(`${render.calls} calls, ${render.triangles} tris, ${memory.geometries} geometries, ${memory.textures} textures, ${programs.length} programs`);
});
```

## Triage: Symptom → Cause → Lever

| Symptom | Check | Likely lever | API owner |
| --- | --- | --- | --- |
| Low FPS, `render.calls` in the thousands | `info.render.calls` | Merge static meshes, instance repeats, batch mixed geometry | `threejs-geometry` |
| Low FPS, few calls, millions of triangles | `info.render.triangles` | LOD, lower segment counts, simplify offline | `threejs-asset-pipeline` |
| FPS rises a lot at pixel ratio 1 | Halve `setPixelRatio` | Cap pixel ratio, cut overdraw, cheaper post-processing | `threejs-postprocessing` |
| Calls jump when shadows are on | Toggle `castShadow` | Fewer casters, tighter shadow camera, freeze static shadows | `threejs-lighting` |
| Hitch the first time something appears | Profiler shows a long task in `render()` | `await renderer.compileAsync(scene, camera)` during loading | this skill |
| Periodic spikes every few seconds | Profiler shows GC | Stop allocating vectors and matrices per frame | this skill |
| `memory.textures` / `geometries` only grows | Count after removing objects | Dispose geometry, material and textures | `threejs-fundamentals` |
| Slow pointer hover | Profiler in `pointermove` | Raycast an explicit list with layers; BVH | `threejs-interaction` |
| CPU-bound particle loop | Profiler in your update | TSL compute on WebGPURenderer | `threejs-math-particles-fields` |

Optimize what the numbers name. CPU time in `render()` well below the frame interval while frames still drop means the GPU is the limit.

## Reading the Counters Correctly

- WebGLRenderer: `render.calls`, `triangles`, `lines`, `points` describe the last `render()` only. Several renders per frame (reflections, picking, post) → set `renderer.info.autoReset = false` and call `renderer.info.reset()` once per frame after the last render.
- WebGPURenderer uses different names: `info.render.drawCalls` is this frame's draw count; `info.render.calls` is render calls since the app started. Per-frame fields reset at each `setAnimationLoop` tick, not in `render()`.
- `memory.geometries` and `memory.textures` are counts, not bytes. WebGPURenderer adds byte estimates in `memory.total` (compressed textures count as 1 byte).
- Shadow passes are inside `render()` and count toward `calls` and `triangles`.
- Average frame time over many frames; a single frame interval never drops below the display refresh.

Details and full field tables: `references/measuring.md`.

## Draw Calls in One Screen

| Lever | Calls | Culling | Moves per object | Cost |
| --- | --- | --- | --- | --- |
| `mergeGeometries` | 1 per material | All-or-nothing (one sphere) | No | Memory per copy; rebuild to change |
| `InstancedMesh` | 1 | One sphere around all instances | Yes (`setMatrixAt`) | Same geometry and material only |
| `BatchedMesh` | 1 multi-draw on WebGL with `WEBGL_multi_draw` | Per instance on the CPU | Yes | CPU cull and sort each frame; WebGPU draws per geometry run |
| `LOD` | Unchanged | Per object | Yes | Fewer triangles far away; all levels in memory |

Also multiplying calls: transparent + `DoubleSide` (two passes unless `forceSinglePass`), any visible `transmission` material (opaque scene drawn twice), every shadow-casting light, every extra `render()`. Worked examples: `references/draw-calls.md`.

## Memory in One Screen

- RGBA8 is 4 bytes per pixel, plus about a third for mips: a 2048² color map is about 22 MB. KTX2 (Basis) textures stay block-compressed in VRAM.
- `geometry.dispose()`, `texture.dispose()` and `material.dispose()` free GPU resources; `material.dispose()` never frees its textures; `object.dispose()` frees only that object's renderer state, not children. Nothing is reference-counted: a mesh still using a disposed resource re-uploads it.

Byte math, KTX2 and a subtree disposal function: `references/memory-gpu.md`.

## Fill Rate, CPU and Compile Stalls in One Screen

- Cap `setPixelRatio` at 2 (cost scales with its square). Transparent layers cannot reject each other; prefer opaque or `alphaTest` cut-outs.
- Shadow maps: default 512²; set `renderer.shadowMap.autoUpdate = false` and `needsUpdate = true` once for static scenes.
- Static scenery: `matrixAutoUpdate = false` plus one `updateMatrix()`.
- Raycast an explicit list with `raycaster.layers`, not `scene.children` recursively.
- Reuse vectors and matrices; per-frame `new THREE.Vector3()` becomes GC spikes.
- `await renderer.compileAsync(scene, camera)` during loading; avoid toggling `transparent`, `side`, `vertexColors`, maps or light counts at runtime, since each builds a new program.

Code and details: `references/fill-rate-cpu.md`.

## Common Mistakes

- `console.log(renderer.info.render.calls); renderer.render(scene, camera);` → read after `render()`: the counters reset inside it, so reading before reports the previous frame.
- `renderer.info.autoReset = false` alone → also call `renderer.info.reset()` once per frame after the last `render()`, or the counters grow forever.
- `webgpuRenderer.info.render.calls` as a draw count → `webgpuRenderer.info.render.drawCalls`: under WebGPURenderer `render.calls` counts render calls since start; WebGLRenderer has no `drawCalls`.
- `renderer.compile(scene, camera)` → `await renderer.compileAsync(scene, camera)`: `compile()` queues builds and returns, so the next render still blocks on unfinished programs.
- `mesh.material.dispose()` expecting textures to free → also `dispose()` every texture on the material.
- `gltf.scene.dispose()` → traverse and dispose each geometry, material and texture, then each object (`references/memory-gpu.md`): `Object3D.dispose()` covers one object only.
- `raycaster.intersectObjects(scene.children, true)` on every pointermove → `raycaster.layers.set(1); raycaster.intersectObjects(pickables, false)`.
- `scene.traverse((child) => (child.frustumCulled = false))` → `geometry.computeBoundingSphere()` after CPU vertex edits, and `frustumCulled = false` only on meshes moved in a shader.
- `function animate() { mesh.position.add(new THREE.Vector3(0, 0.01, 0)); }` → allocate the vector once outside the loop.
- One `mergeGeometries` call for the whole level → merge per area: a single bounding sphere disables culling for everything.
- `new THREE.BatchedMesh(...)` for 50,000 static, always-visible objects with defaults → set `perObjectFrustumCulled = false` and `sortObjects = false`: per-instance culling and sorting run on the CPU every frame.
- `const dt = uniform(0)` fed from a Timer each frame (TSL) → `deltaTime` from `three/tsl`: the built-in `time` and `deltaTime` are already updated by the renderer.

Render-loop, Timer/Clock and shadow-type migration mistakes are covered by `threejs-fundamentals` and `threejs-lighting`.

## Reference Files

- `references/measuring.md` — every `renderer.info` field on both renderers, reset rules, multi-render accumulation, frame and CPU timing, profilers. Open first when numbers look wrong.
- `references/draw-calls.md` — what multiplies draw calls, merge / instance / batch / LOD examples with their trade-offs, keeping frustum culling working.
- `references/memory-gpu.md` — VRAM byte math, KTX2 transcoding summary, disposal rules and a subtree disposal function, WebGPU memory counters.
- `references/fill-rate-cpu.md` — pixel ratio, overdraw and sort order, shadow-map cost and freezing, static transforms, raycasting cost, allocation-free loops, shader compile warm-up.
- `references/webgpu-tsl.md` — WebGPURenderer counters and backend costs, a compute dispatch replacing a CPU particle loop.

## See Also

- `threejs-geometry` — the InstancedMesh, BatchedMesh, mergeGeometries and attribute APIs behind the draw-call cuts.
- `threejs-fundamentals` — renderer creation, render loop, resize, basic cleanup.
- `threejs-materials` — which material settings compile into the shader and force a rebuild.
- `threejs-textures` — texture formats, mipmaps, KTX2 textures and render-target memory.
- `threejs-loaders` — KTX2, Draco and meshopt loader wiring.
- `threejs-asset-pipeline` — gltf-transform commands that produce KTX2 textures, simplified, instanced or joined glTF files.
- `threejs-lighting` — light types and shadow setup whose cost is measured here.
- `threejs-postprocessing` — composer buffers, MSAA and resolution scale.
- `threejs-interaction` — raycasting and picking APIs.
- `threejs-math-particles-fields` — full GPU particle simulations with TSL compute.
- `performance-optimization` — engine-neutral frame budgets, CPU vs GPU diagnosis, pooling.
