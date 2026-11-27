# Draw Calls: What Multiplies Them and How to Cut Them

Purpose: the cost model behind `info.render.calls`, worked examples of merging, instancing, batching and LOD as performance levers, and keeping frustum culling working (renderer behaviour checked against the r187 source). The full InstancedMesh and BatchedMesh APIs are in `threejs-geometry`.

## What Counts as a Draw Call

Each visible mesh queues one draw call, or one per geometry group when the material is an array. Things that multiply it:

- A `transparent` material with `side: THREE.DoubleSide` draws each mesh twice on WebGLRenderer, back faces then front faces; `material.forceSinglePass = true` makes it one pass.
- While any MeshPhysicalMaterial with `transmission > 0` is in view, WebGLRenderer draws the opaque objects a second time into the transmission render target.
- Every shadow-casting light re-draws its casters into its shadow map (a PointLight once per cube face the object falls in).
- Every extra `render()` call (reflections, picking, post-processing scene passes).

Without those, a thousand single-material meshes is a thousand draw calls.

## Merge Static Geometry

Many meshes that never move and share one material become one mesh and one call:

```javascript
import * as BufferGeometryUtils from "three/addons/utils/BufferGeometryUtils.js";

const rockGeometry = new THREE.DodecahedronGeometry(1, 0);
const transform = new THREE.Matrix4();
const parts = [];
for (let i = 0; i < 200; i++) {
  const baked = rockGeometry.clone();
  transform.makeTranslation((Math.random() - 0.5) * 40, 0, (Math.random() - 0.5) * 40);
  baked.applyMatrix4(transform); // bake the placement into the vertices
  parts.push(baked);
}
const merged = BufferGeometryUtils.mergeGeometries(parts); // null if attributes or index state differ
scene.add(new THREE.Mesh(merged, new THREE.MeshNormalMaterial())); // 200 rocks, one call, one bounding sphere
```

Merging trades culling granularity for the call count: the merged mesh has a single bounding sphere, so frustum culling is all-or-nothing, and memory grows with every copy. Merge per spatial cluster, not one merge for a whole level.

## Instance Repeats

One geometry drawn many times with per-instance transforms stays one call:

```javascript
const tree = new THREE.ConeGeometry(0.5, 2, 6);
const trees = new THREE.InstancedMesh(tree, new THREE.MeshNormalMaterial(), 500);
const placement = new THREE.Object3D();
for (let i = 0; i < 500; i++) {
  placement.position.set((Math.random() - 0.5) * 40, 1, (Math.random() - 0.5) * 40);
  placement.rotation.y = Math.random() * 6.28;
  placement.updateMatrix();
  trees.setMatrixAt(i, placement.matrix);
}
// culling tests one sphere around all instances, so all 500 draw whenever it touches the view;
// after moving instances later, set trees.instanceMatrix.needsUpdate = true and call trees.computeBoundingSphere()
scene.add(trees);
```

## Batch Different Geometries

Many geometries sharing one material pack into shared buffers:

```javascript
const material = new THREE.MeshNormalMaterial();
const batchedMesh = new THREE.BatchedMesh(500, 20000, 40000, material);
const coneId = batchedMesh.addGeometry(new THREE.ConeGeometry(0.5, 2, 6));
const boxId = batchedMesh.addGeometry(new THREE.BoxGeometry(1, 1, 1));
const placement = new THREE.Matrix4();
for (let i = 0; i < 500; i++) {
  const instanceId = batchedMesh.addInstance(i % 2 ? coneId : boxId); // a geometry renders only through an instance
  placement.makeTranslation((Math.random() - 0.5) * 40, 1, (Math.random() - 0.5) * 40);
  batchedMesh.setMatrixAt(instanceId, placement);
}
scene.add(batchedMesh);
```

- Unlike InstancedMesh it culls and sorts each instance on the CPU every frame (`perObjectFrustumCulled`, `sortObjects`, both on by default). That CPU cost grows with the instance count; switch both off for static batches that are always in view.
- WebGLRenderer submits one multi-draw where `WEBGL_multi_draw` exists (counted as one call), one draw per instance otherwise.
- The WebGPU backend issues one draw per run of consecutive instances sharing a geometry; r187 merges each run into one instanced draw. Sorting by distance interleaves geometries and splits runs, so `info.render.drawCalls` can be close to the instance count.

## Drop Detail with Distance (LOD)

`LOD` keeps every level as a child and, inside every `render()` (`autoUpdate` is on by default), shows only the level for the current camera distance, so far objects submit fewer triangles. The call count stays the same:

```javascript
const detailMaterial = new THREE.MeshNormalMaterial();
const lod = new THREE.LOD();
lod.addLevel(new THREE.Mesh(new THREE.SphereGeometry(1, 64, 32), detailMaterial), 0); // distance from camera to object
lod.addLevel(new THREE.Mesh(new THREE.SphereGeometry(1, 16, 8), detailMaterial), 12);
lod.addLevel(new THREE.Mesh(new THREE.SphereGeometry(1, 8, 4), detailMaterial), 30, 0.1); // third argument: hysteresis
scene.add(lod);
```

`addLevel(object, distance = 0, hysteresis = 0)`: hysteresis is a fraction of the distance that stops a level flickering when the camera hovers at the threshold. Every level's geometry stays in GPU memory. For thousands of LOD objects, a per-object `LOD` costs CPU per frame; batching per level (or a BVH-driven BatchedMesh LOD scheme, as in the `webgl_batch_lod_bvh` example) scales better.

## Keep Frustum Culling Working

Frustum culling is on by default: before a mesh is queued, its bounding sphere is tested against the camera frustum, and a failed test removes the draw entirely (InstancedMesh and BatchedMesh use their own `boundingSphere`; other meshes use `geometry.boundingSphere`). Ways to lose it silently:

- Editing vertices on the CPU without `geometry.computeBoundingSphere()` leaves a stale sphere; one left smaller than the moved vertices hides the mesh while it is still in view.
- Moving vertices in a shader leaves the CPU sphere untouched. Set `mesh.frustumCulled = false` on that one mesh, or set a generous `geometry.boundingSphere` by hand; never disable culling scene-wide.
- One merged mesh spanning the level is always "in view".
