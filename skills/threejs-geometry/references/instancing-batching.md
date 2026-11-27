# InstancedMesh, InstancedBufferGeometry and BatchedMesh

Purpose: the full API of the three ways to draw many objects in few calls, with update, culling and raycasting details (checked against the r187 source). Which one to pick for speed, and how to measure it, is in `threejs-performance`.

## InstancedMesh

One draw call for many copies of one geometry and material, each with its own transform (and optionally color or morph weights).

```javascript
const geometry = new THREE.BoxGeometry(0.2, 0.2, 0.2);
const material = new THREE.MeshBasicMaterial(); // multiplies by instance colors; MeshNormalMaterial ignores them
const count = 1000;
const instancedMesh = new THREE.InstancedMesh(geometry, material, count);
const dummy = new THREE.Object3D();
for (let i = 0; i < count; i++) {
  dummy.position.set((Math.random() - 0.5) * 20, (Math.random() - 0.5) * 20, (Math.random() - 0.5) * 20);
  dummy.rotation.set(Math.random() * Math.PI, Math.random() * Math.PI, 0);
  dummy.scale.setScalar(0.5 + Math.random());
  dummy.updateMatrix();
  instancedMesh.setMatrixAt(i, dummy.matrix);
}
instancedMesh.instanceMatrix.needsUpdate = true; // needed after changes once the mesh has rendered; harmless before

// setColorAt creates instanceColor (filled with white) on first use; flag it after the loop
const color = new THREE.Color();
for (let i = 0; i < count; i++) {
  instancedMesh.setColorAt(i, color.setRGB(Math.random(), Math.random(), Math.random()));
}
instancedMesh.instanceColor.needsUpdate = true;

scene.add(instancedMesh);
```

At runtime, update and raycast instances like this:

```javascript
// mesh here is an InstancedMesh
const matrix = new THREE.Matrix4();
const index = 7;
mesh.getMatrixAt(index, matrix);
matrix.setPosition(0, 2, 0); // move just this instance
mesh.setMatrixAt(index, matrix);
mesh.instanceMatrix.needsUpdate = true; // flag after any batch of setMatrixAt calls
mesh.computeBoundingSphere(); // instances moved: refresh the bounds used by frustum culling and raycasting

// Raycasting reports which instance was hit
const raycaster = new THREE.Raycaster();
const hits = raycaster.intersectObject(mesh);
if (hits.length > 0) {
  const instanceId = hits[0].instanceId; // undefined for a non-instanced object
}
```

- Methods: `setMatrixAt` / `getMatrixAt`, `setColorAt` / `getColorAt`, `setMorphAt` / `getMorphAt` (per-instance morph weights, stored in `morphTexture`), `computeBoundingBox`, `computeBoundingSphere`, `dispose`.
- `mesh.count` can be lowered at runtime to draw only the first N instances; it cannot exceed the count passed to the constructor.
- Culling and raycasting use `mesh.boundingSphere` (the union of all instances), not `geometry.boundingSphere`. It is computed lazily on the first cull test; after moving instances, call `mesh.computeBoundingSphere()` or instances outside the old sphere vanish.
- For data rewritten every frame, `mesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage)`.
- `dispose()` releases the instance matrix and color attributes and the morph texture as well as the object's renderer state; the shared geometry and material still need their own `dispose()`.

## InstancedBufferGeometry

Per-instance attributes beyond transform and color, read by your own shader code.

```javascript
const count = 1000;
const geometry = new THREE.InstancedBufferGeometry();
geometry.copy(new THREE.BoxGeometry(1, 1, 1)); // copies the index and all attributes
geometry.instanceCount = count; // required: copy() sets instanceCount to the source's (undefined)

const offsets = new Float32Array(count * 3).map(() => Math.random() * 10);
geometry.setAttribute("offset", new THREE.InstancedBufferAttribute(offsets, 3));

// With WebGLRenderer and a ShaderMaterial, declare `in vec3 offset;` in the vertex
// shader yourself and add it to position (see threejs-shaders). Under WebGPURenderer,
// per-instance data is a TSL node - see webgpu-tsl.md.
```

A fresh `InstancedBufferGeometry` has `instanceCount = Infinity` (draw as many as the instanced attributes hold). After `copy()` from a plain geometry it is `undefined`, which WebGLRenderer turns into a draw of zero instances.

## BatchedMesh

Many different geometries sharing one material, each instance with its own transform, color and visibility.

```javascript
const material = new THREE.MeshNormalMaterial();
const maxInstanceCount = 1000;
const maxVertexCount = 20000;
const maxIndexCount = 20000; // optional, defaults to maxVertexCount * 2
const batchedMesh = new THREE.BatchedMesh(maxInstanceCount, maxVertexCount, maxIndexCount, material);
const boxGeometryId = batchedMesh.addGeometry(new THREE.BoxGeometry(1, 1, 1));
const sphereGeometryId = batchedMesh.addGeometry(new THREE.SphereGeometry(0.5, 16, 16));

// A geometry renders only through its instances (addInstance, required since r166)
const boxInstanceId = batchedMesh.addInstance(boxGeometryId);
batchedMesh.setMatrixAt(boxInstanceId, new THREE.Matrix4().setPosition(-2, 0, 0));
const sphereInstanceId = batchedMesh.addInstance(sphereGeometryId);
batchedMesh.setMatrixAt(sphereInstanceId, new THREE.Matrix4().setPosition(2, 0, 0));

scene.add(batchedMesh);
```

- Compatibility: the first geometry added defines the layout. Every later geometry must match its index usage (all indexed or none) and provide each of its attributes with the same `itemSize` and `normalized`; extra attributes are ignored. A mismatch throws.
- Capacity: `addGeometry(geometry, reservedVertexCount = -1, reservedIndexCount = -1)` throws when the shared buffers or geometry slots are full. Grow with `setInstanceCount(n)` and `setGeometrySize(maxVertexCount, maxIndexCount)`; `optimize()` compacts after deletions. Reserve extra space up front if `setGeometryAt(geometryId, geometry)` will later swap in a larger geometry.
- Per instance: `setMatrixAt` / `getMatrixAt`, `setColorAt` / `getColorAt`, `setVisibleAt` / `getVisibleAt`, `setGeometryIdAt` / `getGeometryIdAt`, `deleteInstance`. `setMatrixAt` and `setColorAt` flag their data textures themselves; no `needsUpdate` is needed.
- Per geometry: `deleteGeometry`, `getBoundingBoxAt`, `getBoundingSphereAt`, `getGeometryRangeAt`.
- `perObjectFrustumCulled` and `sortObjects` (both `true` by default) cull and sort each instance on the CPU every frame; `setCustomSort(fn)` replaces the sort. Turn them off for static, always-visible batches to save CPU.
- Draws: WebGLRenderer issues one multi-draw call with `WEBGL_multi_draw` and falls back to one draw per instance without it. The WebGPU backend issues one draw per run of consecutive instances that share a geometry range; r187 merges each such run into one instanced draw, so distance sorting that interleaves geometries splits runs.
- Raycasting reports `batchId` on hits.

## Choosing

| Need | Use |
| --- | --- |
| Static meshes that never move individually, one material | `BufferGeometryUtils.mergeGeometries` (one bounding sphere for all) |
| Many copies of one geometry, per-copy transform or color | `InstancedMesh` |
| Many copies, custom per-instance data in a shader | `InstancedBufferGeometry` (WebGL) or `instancedBufferAttribute` nodes (WebGPU) |
| Many different geometries, one material, per-object transform and culling | `BatchedMesh` |
