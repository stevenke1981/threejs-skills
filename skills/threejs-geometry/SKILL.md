---
name: threejs-geometry
description: "three.js geometry API: built-in shapes, TextGeometry and FontLoader, BufferGeometry attributes and indices, mergeGeometries and BufferGeometryUtils, InstancedMesh and BatchedMesh, points and lines, with WebGPU/TSL notes. Use for custom vertex data, 3D text, mergeGeometries returning null, instancing many copies, thick lines, or points stuck at 1px under WebGPU. Not for meshes from equations or sweeps (use threejs-math-procedural-geometry), draw-call profiling or LOD (use threejs-performance), loading models (use threejs-loaders). 中文: 幾何體, 頂點緩衝, 自訂網格, 實例化, 合併幾何, 3D文字, 點雲, 線條"
---

# Three.js Geometry

Targets three.js r187 (npm `three@0.187.0`). Examples use WebGLRenderer; `references/webgpu-tsl.md` covers WebGPURenderer.

## Quick Start

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 5);

// A geometry is the shape; many meshes can share one geometry instance
const mesh = new THREE.Mesh(new THREE.BoxGeometry(1, 1, 1), new THREE.MeshNormalMaterial());
scene.add(mesh);

renderer.setAnimationLoop(() => {
  mesh.rotation.y += 0.01;
  renderer.render(scene, camera);
});
```

## Choosing an Approach

| You need | Use | Details |
| --- | --- | --- |
| A box, sphere, cylinder, capsule, torus, ring, polyhedron | Built-in `*Geometry` classes | `references/builtin-geometries.md` |
| A fixed 2D outline extruded, revolved or swept | `ExtrudeGeometry`, `LatheGeometry`, `TubeGeometry` | `references/builtin-geometries.md` |
| 3D text | `TextGeometry` + `FontLoader` (addons) | below, and `references/builtin-geometries.md` |
| Your own vertices | `BufferGeometry` + `BufferAttribute` | below, and `references/buffergeometry.md` |
| A surface, sweep, plant or isosurface from maths | | `threejs-math-procedural-geometry` |
| Static meshes combined into one | `BufferGeometryUtils.mergeGeometries` | `references/buffergeometry.md` |
| Many copies of one geometry | `InstancedMesh` | `references/instancing-batching.md` |
| Many different geometries, one material | `BatchedMesh` | `references/instancing-batching.md` |
| Point clouds, lines, thick lines | `Points`, `Line*`, `Line2` | `references/points-lines.md` |

Whether merging, instancing, batching or LOD actually helps is a measurement question: see `threejs-performance`.

## Built-in Shapes in Brief

```javascript
new THREE.BoxGeometry(1, 1, 1); // width, height, depth, widthSegments, heightSegments, depthSegments
new THREE.SphereGeometry(1, 32, 16); // radius, widthSegments=32, heightSegments=16, phi/theta start and length
new THREE.PlaneGeometry(10, 10, 1, 1); // width, height, widthSegments, heightSegments
new THREE.CylinderGeometry(1, 1, 2, 32); // radiusTop, radiusBottom, height, radialSegments, ...
new THREE.CapsuleGeometry(0.5, 1, 4, 8, 1); // radius, height (middle section), capSegments, radialSegments, heightSegments
new THREE.ExtrudeGeometry(new THREE.Shape([new THREE.Vector2(0, 0), new THREE.Vector2(1, 0), new THREE.Vector2(0, 1)]), { depth: 1, bevelEnabled: false }); // bevel is ON by default
```

Every built-in class is already a `BufferGeometry`; the old `*BufferGeometry` aliases are gone.

## 3D Text

```javascript
import { FontLoader } from "three/addons/loaders/FontLoader.js";
import { TextGeometry } from "three/addons/geometries/TextGeometry.js";

const font = await new FontLoader().loadAsync("fonts/helvetiker_regular.typeface.json");
const text = new TextGeometry("Hello", { font, size: 1, depth: 0.2, curveSegments: 12 });
text.center();
scene.add(new THREE.Mesh(text, new THREE.MeshNormalMaterial()));
```

Pass `size` and `depth` explicitly: the defaults are 100 and 50 units.

## Custom BufferGeometry

```javascript
const geometry = new THREE.BufferGeometry();
// Four vertices, three floats each (x, y, z)
const vertices = new Float32Array([-1, -1, 0, 1, -1, 0, 1, 1, 0, -1, 1, 0]);
geometry.setAttribute("position", new THREE.BufferAttribute(vertices, 3));
geometry.setIndex([0, 1, 2, 0, 2, 3]); // a plain array picks Uint16 or Uint32 itself
geometry.setAttribute("normal", new THREE.BufferAttribute(new Float32Array([0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1]), 3));
geometry.setAttribute("uv", new THREE.BufferAttribute(new Float32Array([0, 0, 1, 0, 1, 1, 0, 1]), 2));
// Optional per-vertex colors, rendered with material.vertexColors = true
geometry.setAttribute("color", new THREE.BufferAttribute(new Float32Array([1, 0, 0, 0, 1, 0, 0, 0, 1, 1, 1, 0]), 3));
scene.add(new THREE.Mesh(geometry, new THREE.MeshBasicMaterial({ vertexColors: true })));
```

After editing an attribute in place: set `attribute.needsUpdate = true`, recompute normals if the shape changed, and call `geometry.computeBoundingSphere()` (culling and raycasting use it). Arrays cannot grow after the first render: allocate the maximum and use `geometry.setDrawRange(0, n)`.

## Instancing and Batching in Brief

```javascript
// InstancedMesh: one geometry, one material, N transforms, one draw call
const trees = new THREE.InstancedMesh(new THREE.ConeGeometry(0.5, 2, 6), new THREE.MeshStandardMaterial(), 500);
const m = new THREE.Matrix4();
for (let i = 0; i < trees.count; i++) trees.setMatrixAt(i, m.makeTranslation(Math.random() * 40 - 20, 1, Math.random() * 40 - 20));
trees.instanceMatrix.needsUpdate = true; // after every batch of changes once rendered
trees.computeBoundingSphere(); // culling uses the mesh's own sphere around all instances
scene.add(trees);

// BatchedMesh: different geometries, one material; geometries render only through instances
const batch = new THREE.BatchedMesh(100, 10000, 20000, new THREE.MeshStandardMaterial());
const boxId = batch.addGeometry(new THREE.BoxGeometry(1, 1, 1));
batch.setMatrixAt(batch.addInstance(boxId), m.makeTranslation(2, 0, 0)); // flags its own data texture
scene.add(batch);
```

All geometries in one BatchedMesh must match the first one's index usage and attribute `itemSize`/`normalized`, or `addGeometry` throws.

## Disposal

`geometry.dispose()` frees the GPU buffers immediately, even while another mesh still uses them (that mesh re-uploads them). `mesh.dispose()` frees only that object's renderer state. Traversing a whole loaded scene and releasing textures is covered in `threejs-performance` (memory) and `threejs-fundamentals` (cleanup).

## Common Mistakes

- `new THREE.BoxBufferGeometry(1, 1, 1)` → `new THREE.BoxGeometry(1, 1, 1)`: the `*BufferGeometry` aliases no longer exist.
- `BufferGeometryUtils.mergeBufferGeometries([a, b])` → `BufferGeometryUtils.mergeGeometries([a, b])`: the old name is not exported, so the call throws a TypeError.
- `mergeGeometries([indexedBox, nonIndexedShape])` returns `null` → make inputs all indexed (`mergeVertices`) or all non-indexed (`toNonIndexed()`), with the same attribute set and morph attributes.
- `BufferGeometryUtils.computeTangents(geometry)` → `geometry.computeTangents()`: the core method needs an index plus `position`, `normal` and `uv`.
- `new THREE.BufferAttribute(new Uint8Array(rgbBytes), 3)` → add `true` as the third argument: unnormalized bytes reach the shader as 0 to 255 and the colors blow out.
- `geometry.copy(new THREE.BoxGeometry(1, 1, 1))` on an InstancedBufferGeometry → follow it with `geometry.instanceCount = count`: `copy()` overwrites it with `undefined` and nothing draws.
- `batchedMesh.addGeometry(box)` alone → `batchedMesh.addInstance(batchedMesh.addGeometry(box))`: since r166 a geometry renders only through an instance.
- `import { TextGeometry } from "three/examples/jsm/geometries/TextGeometry.js"` → `"three/addons/geometries/TextGeometry.js"`: the documented import maps map `three/addons/`, not `three/examples/`.
- `{ font, size: 1, height: 0.2 }` → `{ font, size: 1, depth: 0.2 }`: renamed in r163; `height` is ignored and the text extrudes the default 50 units.
- `capsule.parameters.length` → `capsule.parameters.height`: renamed in r176, which also added `heightSegments`.
- `geometry.setAttribute("uv2", geometry.attributes.uv)` → leave the aoMap on `uv` (channel 0), or put a separate unwrap in `uv1` and set `aoTexture.channel = 1`: `uv2` is now the third UV channel.
- `mesh.morphTargetInfluences[0] = 0.5` after adding targets to an existing mesh → call `mesh.updateMorphTargets()` first.
- `new THREE.LineBasicMaterial({ linewidth: 4 })` → `Line2` + `LineGeometry` + `LineMaterial` from `three/addons/lines/` (WebGPU: `lines/webgpu/Line2.js` + `Line2NodeMaterial`): line primitives are always one pixel wide.
- `new THREE.LineLoop(geometry, material)` under WebGPURenderer → a `Line` whose points repeat the first point: LineLoop is skipped with an error.
- `new THREE.Points(cloud, new THREE.PointsMaterial({ size: 10 }))` under the WebGPU backend → instanced `Sprite` with `PointsNodeMaterial` (`references/webgpu-tsl.md`): points are one pixel there.
- `new THREE.ShaderMaterial({ vertexShader })` under WebGPURenderer → `material.positionNode = positionLocal.add(instancedBufferAttribute(offsets))`: ShaderMaterial falls back to a plain NodeMaterial with an error.
- `material.positionNode = positionGeometry.add(offset)` → `positionLocal.add(offset)`: `positionLocal` already carries skinning, morphs, batching and instancing; `positionGeometry` is the raw attribute.

## Reference Files

- `references/builtin-geometries.md` — every core constructor with defaults, Lathe/Shape/Extrude/Tube, full TextGeometry options, Edges and Wireframe, geometry addons. Open when you need a parameter order or default.
- `references/buffergeometry.md` — attribute types, interleaved buffers, in-place updates, BufferGeometryUtils exports, MikkTSpace tangents, center/scale/pivot, morph targets.
- `references/instancing-batching.md` — InstancedMesh update, culling and raycasting; InstancedBufferGeometry; full BatchedMesh API, capacity and draw behaviour; a choosing table.
- `references/points-lines.md` — Points, Line/LineSegments/LineLoop, dashed lines, Line2 on both renderers.
- `references/webgpu-tsl.md` — WebGPURenderer differences, sized points via instanced Sprite, per-instance TSL attributes, positionLocal vs positionGeometry.

## See Also

- `threejs-math-procedural-geometry` — parametric surfaces, curve frames, varying-radius tubes, lofts, L-systems, marching cubes.
- `threejs-performance` — choosing between merging, instancing, batching and LOD by measured draw calls; memory and disposal.
- `threejs-fundamentals` — scene setup, render loop, Object3D transforms, cleanup.
- `threejs-materials` — materials for these meshes, `vertexColors`, `side`, line and points materials.
- `threejs-shaders` — GLSL for InstancedBufferGeometry attributes; custom TSL logic.
- `threejs-animation` — animating morph target influences.
- `threejs-loaders` — glTF and other formats that bring their own geometry.
- `threejs-interaction` — raycasting instances and points.
