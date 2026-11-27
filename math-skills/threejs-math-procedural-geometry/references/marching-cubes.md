# Isosurfaces with the MarchingCubes Addon

Purpose: how `three/addons/objects/MarchingCubes.js` stores its field, where the surface lies, metaballs, and meshing an arbitrary field or SDF (API checked against the r187 source; the metaball radius figures are derived from the addon's falloff formula, not measured).

`new MarchingCubes(resolution, material, enableUvs = false, enableColors = false, maxPolyCount = 10000)` is a `Mesh` that polygonizes a scalar field on the CPU. The geometry is non-indexed with its own normals, and its attributes use `DynamicDrawUsage`. Nothing is GPU-based, and nothing rebuilds on render: call `update()`.

- The mesh occupies the local cube [-1, 1]^3; place it with `scale` and `position`.
- `effect.field` is a `Float32Array(resolution^3)` indexed `x + y * resolution + z * resolution^2` (`setCell(x, y, z, value)`, `getCell(x, y, z)`). Cell `x` sits at local `(x - resolution / 2) / (resolution / 2)`.
- The surface is where the field equals `effect.isolation` (default 80). Values above it are inside; normals point toward lower values.
- `update()` polygonizes cells 1 to `resolution - 3` on each axis, so the outer cells are never meshed: a shape that reaches the cube boundary is left open there.
- `maxPolyCount` bounds the triangle buffer; past it the console warns "Geometry buffers too small for rendering" and the mesh is cut off. Raise it with the resolution.
- The cost is a loop over `resolution^3` cells per `update()`. Resolution 28 to 64 is a starting range for per-frame updates, not a measurement; update a static field once.
- `blur(intensity = 1)` smooths the field in place before `update()`.

## Metaballs

Per frame call `reset()`, `addBall(x, y, z, strength, subtract, color)` for each ball, then `update()`. Ball coordinates are in [0, 1] of the field cube, not world units. A ball's surface radius is about `sqrt(strength / (isolation + subtract))` as a fraction of the cube (0.074 for strength 0.5 and subtract 12 at the default isolation); `sqrt(strength / subtract)` is where its field reaches zero. A negative `strength` subtracts the same falloff, carving a hole. `addPlaneX/Y/Z(strength, subtract)` add wall and floor fields.

```javascript
import { MarchingCubes } from "three/addons/objects/MarchingCubes.js";

const blobs = new MarchingCubes(48, new THREE.MeshStandardMaterial({ color: 0xff7755 }), false, false, 40000);
blobs.scale.setScalar(2);
scene.add(blobs);

function updateBlobs(elapsed) {
  blobs.reset();
  for (let i = 0; i < 3; i++) {
    const x = 0.5 + 0.25 * Math.sin(elapsed + i * 2.1);
    const y = 0.5 + 0.25 * Math.cos(elapsed * 0.7 + i);
    blobs.addBall(x, y, 0.5, 0.5, 12);
  }
  blobs.update();
}
```

## Meshing an Arbitrary Field or SDF

The SDF functions themselves are in `threejs-math-sdf-raymarching`. Inside must be the higher value, so write the negated distance and set the isolation to 0:

```javascript
const resolution = 48;
const effect = new MarchingCubes(resolution, new THREE.MeshStandardMaterial({ color: 0x66aaff }), false, false, 60000);
effect.isolation = 0;
effect.scale.setScalar(2);
scene.add(effect);

const sdf = (x, y, z) => Math.hypot(x, y, z) - 0.6 + 0.1 * Math.sin(8 * x) * Math.sin(8 * y) * Math.sin(8 * z);

effect.reset(); // zeroes the field, the color palette and the normal cache; stale normals are reused otherwise
const half = resolution / 2;
for (let z = 0; z < resolution; z++) {
  for (let y = 0; y < resolution; y++) {
    for (let x = 0; x < resolution; x++) {
      effect.setCell(x, y, z, -sdf((x - half) / half, (y - half) / half, (z - half) / half));
    }
  }
}
effect.update();
```

`enableUvs` and `enableColors` add those attributes; set `material.vertexColors = true` to see colors. Under WebGPURenderer the addon works unchanged (it is a plain Mesh with dynamic attributes); no GPU marching-cubes implementation ships in the package.
