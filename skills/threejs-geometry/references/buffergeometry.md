# BufferGeometry: Attributes, Updates, Utilities, Morphs

Purpose: attribute types, interleaved buffers, in-place updates, BufferGeometryUtils (merge, weld, tangents), geometry transforms and pivots, and morph targets (checked against the r187 source).

## Attribute Types

```javascript
const count = 4;
const geometry = new THREE.BufferGeometry();
// Float32 for positions, normals, UVs and floating-point colors
geometry.setAttribute("position", new THREE.BufferAttribute(new Float32Array(count * 3), 3));
// Uint16 indices address vertices 0 to 65534 (65535 is the primitive-restart value);
// setIndex([...]) with a plain array picks Uint16 or Uint32 itself (Uint32 once any index >= 65535)
geometry.setIndex(new THREE.BufferAttribute(new Uint16Array(count), 1));
// Uint8 colors need normalized = true (third argument) so bytes reach the shader as 0.0-1.0
geometry.setAttribute("color", new THREE.BufferAttribute(new Uint8Array(count * 3), 3, true));
```

`position`, `normal`, `uv` (channel 0), `uv1`-`uv3` (channels 1-3), `color` and `tangent` are the names built-in materials read. A material's texture reads `uv` unless that texture's `channel` is set (for example `aoTexture.channel = 1` reads `uv1`).

## Updating a BufferGeometry

```javascript
const geometry = new THREE.PlaneGeometry(2, 2);
const positions = geometry.attributes.position;
positions.setXYZ(0, -2, -2, 0); // overwrite vertex 0
positions.getX(0); // read single components with getX / getY / getZ
positions.needsUpdate = true; // upload the change
geometry.computeVertexNormals(); // normals must match the new positions
geometry.computeBoundingBox(); // bounding volumes drive frustum culling and raycasting
geometry.computeBoundingSphere();
```

- An attribute's array cannot grow after its first render. Allocate the maximum up front and draw a prefix with `geometry.setDrawRange(0, n)`, or replace the geometry (dispose the old one) when it outgrows its buffers.
- For data rewritten every frame, `attribute.setUsage(THREE.DynamicDrawUsage)` hints the driver. `attribute.addUpdateRange(start, count)` uploads a sub-range instead of the whole array.

## Interleaved Buffers

```javascript
// One buffer holding position (3 floats) and uv (2 floats) per vertex
const interleavedBuffer = new THREE.InterleavedBuffer(
  new Float32Array([
    -1, -1, 0, 0, 0, 1, -1, 0, 1, 0, 1, 1, 0, 1, 1, -1, 1, 0, 0, 1, // four vertices: xyz + uv
  ]),
  5, // stride: floats per vertex
);
const geometry = new THREE.BufferGeometry();
geometry.setAttribute("position", new THREE.InterleavedBufferAttribute(interleavedBuffer, 3, 0));
geometry.setAttribute("uv", new THREE.InterleavedBufferAttribute(interleavedBuffer, 2, 3));
geometry.setIndex([0, 1, 2, 0, 2, 3]); // two triangles over the four vertices
// BufferGeometryUtils.interleaveAttributes([position, normal, uv]) packs existing attributes this way
```

## BufferGeometryUtils

```javascript
import * as BufferGeometryUtils from "three/addons/utils/BufferGeometryUtils.js";
import * as MikkTSpace from "three/addons/libs/mikktspace.module.js";

const geo1 = new THREE.BoxGeometry(1, 1, 1);
const geo2 = new THREE.SphereGeometry(0.5, 16, 16);
// Merge static geometries. Inputs need the same attribute set, all indexed or all non-indexed,
// and matching morph attributes and morphTargetsRelative; otherwise it logs an error and returns null.
// useGroups = true adds one group per input, for a material array such as [boxMaterial, sphereMaterial].
const merged = BufferGeometryUtils.mergeGeometries([geo1, geo2], true);

// Weld duplicate vertices (all attributes must match within tolerance=1e-4);
// also generates an index for a non-indexed geometry. Interleaved data is supported since r187.
const welded = BufferGeometryUtils.mergeVertices(geo2);

// Tangents for normal mapping - the core method needs an index plus position, normal and uv
welded.computeTangents();

// MikkTSpace tangents (the convention glTF normal maps use); this de-indexes `merged` in place.
// Since r187 the bundled mikktspace.module.js is plain JavaScript and ready at once;
// `await MikkTSpace.ready` is only needed for the WebAssembly `mikktspace` npm package, and is harmless here.
await MikkTSpace.ready;
BufferGeometryUtils.computeMikkTSpaceTangents(merged, MikkTSpace);
```

Exports in r187: `computeMikkTSpaceTangents`, `mergeGeometries`, `mergeAttributes`, `deepCloneAttribute`, `deinterleaveAttribute`, `deinterleaveGeometry`, `interleaveAttributes`, `estimateBytesUsed`, `mergeVertices`, `remapAttribute` (added in r187), `toTrianglesDrawMode` (modifies the index in place since r186; clone first to keep the original), `computeMorphedAttributes`, `mergeGroups`, `toCreasedNormals`.

## Transforming Geometry

```javascript
const geometry = new THREE.TorusKnotGeometry(0.5, 0.15, 100, 16);

geometry.center(); // move the vertices so the bounding-box center lands on the origin
geometry.computeBoundingBox(); // refresh after vertex changes
const size = new THREE.Vector3();
geometry.boundingBox.getSize(size);
const maxDim = Math.max(size.x, size.y, size.z);
geometry.scale(1 / maxDim, 1 / maxDim, 1 / maxDim); // fit inside a unit cube

const clone = geometry.clone(); // clone first: these methods mutate the geometry
clone.rotateX(Math.PI / 2);
clone.translate(0, 1, 0);
```

To rotate or scale an object around a point without touching vertex data, set a pivot instead of re-centering the geometry:

```javascript
// Rotation and scale normally use the object origin; pivot is a point in the object's own coordinates
mesh.pivot = new THREE.Vector3(1, 0, 0); // default null
mesh.rotation.y += 0.05; // now swings around the local point (1, 0, 0)
```

`updateMatrix()` folds the pivot into `mesh.matrix`, and since r187 `applyMatrix4()` honours the pivot when it decomposes the matrix back into position, rotation and scale.

## Morph Targets

```javascript
const geometry = new THREE.BoxGeometry(1, 1, 1, 4, 4, 4);
const morphPositions = geometry.attributes.position.array.slice();
for (let i = 0; i < morphPositions.length; i += 3) {
  morphPositions[i] *= 2; // stretch X
  morphPositions[i + 1] *= 0.5; // squash Y
}
geometry.morphAttributes.position = [new THREE.BufferAttribute(morphPositions, 3)];

// Build the mesh after assigning morphAttributes so it picks up the targets
const mesh = new THREE.Mesh(geometry, new THREE.MeshNormalMaterial());
mesh.morphTargetInfluences[0] = 0.5; // 50% blend between base and target
scene.add(mesh);
```

The `Mesh` constructor calls `updateMorphTargets()`; when targets are added to a geometry that already has a mesh, call `mesh.updateMorphTargets()` before touching `morphTargetInfluences`. Animating influences over time belongs to `threejs-animation`.
