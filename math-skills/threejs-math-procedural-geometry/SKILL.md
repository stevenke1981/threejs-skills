---
name: threejs-math-procedural-geometry
description: "three.js meshes generated from maths: ParametricGeometry and indexed grids, height fields, analytic normals, seams and poles, curve frames, varying-radius tubes with LoftGeometry, lathe and extrude from computed profiles, superformula, L-systems, MarchingCubes, vertex-stage surfaces in TSL. Use for meshing an equation or SDF, twisted tubes, seam lighting lines, metaballs, plants. Not for built-in shape parameters or instancing (use threejs-geometry), noise (use threejs-math-noise), raymarched SDFs (use threejs-math-sdf-raymarching). 中文: 程序化幾何, 參數曲面, 掃掠, 放樣, 地形網格, L系統, 行進立方體, 超級公式"
---

# Three.js Math: Procedural Geometry

Targets three.js r187 (npm `three@0.187.0`). Examples use WebGLRenderer; `references/webgpu-tsl.md` covers WebGPURenderer. Geometry here is built on the CPU once, or when parameters change. For `BufferGeometry` attributes, merging and the instancing API see `threejs-geometry`.

## Quick Start

```javascript
import * as THREE from "three";
import { ParametricGeometry } from "three/addons/geometries/ParametricGeometry.js";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 2, 5);
camera.lookAt(0, 0, 0);

const sun = new THREE.DirectionalLight(0xffffff, 3);
sun.position.set(2, 3, 4);
scene.add(sun, new THREE.HemisphereLight(0xffffff, 0x223344, 1));

// u and v run over [0, 1]; write the point into target, the return value is ignored
function wavyTorus(u, v, target) {
  const a = u * Math.PI * 2; // around the ring
  const b = v * Math.PI * 2; // around the tube
  const r = 0.35 + 0.08 * Math.sin(6 * a); // tube radius varies along the ring
  const ring = 1 + r * Math.cos(b);
  return target.set(ring * Math.cos(a), r * Math.sin(b), -ring * Math.sin(a));
}

// slices = segments along u, stacks = segments along v
const geometry = new ParametricGeometry(wavyTorus, 128, 32);
const material = new THREE.MeshStandardMaterial({ color: 0x66aaff, roughness: 0.4 });
const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);

const timer = new THREE.Timer();
timer.connect(document);

renderer.setAnimationLoop(() => {
  timer.update();
  mesh.rotation.y = timer.getElapsed() * 0.3;
  renderer.render(scene, camera);
});
```

`ParametricGeometry` lives in addons; `THREE.ParametricGeometry` does not exist.

## Choosing a Technique

| Shape | Technique | Reference |
| --- | --- | --- |
| Closed-form surface `P(u, v)`, quick to try | `ParametricGeometry` (finite-difference normals) | `references/surfaces-grids.md` |
| Same, with exact normals, extra attributes, large or updated in place | Hand-built indexed grid | `references/surfaces-grids.md` |
| Terrain `y = h(x, z)` | Displace a rotated `PlaneGeometry` | `references/surfaces-grids.md` |
| Constant-radius tube along a curve | `TubeGeometry` | `references/curves-sweeps.md` |
| Radius or profile changing along a path | Rings from `computeFrenetFrames` + `LoftGeometry` | `references/curves-sweeps.md` |
| Solid of revolution, extruded outline | `LatheGeometry`, `ExtrudeGeometry` from computed points | `references/curves-sweeps.md` |
| Starfish, flowers, organic blobs | Superformula outline or supershape | `references/shapes-growth.md` |
| Plants, trees, recursive structures | L-system or recursive turtle into one `InstancedMesh` | `references/shapes-growth.md` |
| Metaballs, an SDF or scalar field as a mesh | `MarchingCubes` addon | `references/marching-cubes.md` |
| A surface animated every frame | Formula in `positionNode` / `normalNode` (TSL) | `references/webgpu-tsl.md` |

## The Rules That Break Meshes

**Grid sizes.** A grid of N segments has N + 1 vertices per side: `(uSegments + 1) * (vSegments + 1)` vertices, `uSegments * vSegments * 6` indices. Above 65535 vertices a `Uint16Array` index wraps; pass a plain array to `geometry.setIndex()` and three picks `Uint32`.

**Winding and normals.** The front face is the side `cross(dP/du, dP/dv)` points to. If a surface renders inside out, flip the triangle order and the normals together. `DoubleSide` only hides the problem.

**Analytic normals beat `computeVertexNormals()`** on closed surfaces: the duplicate seam column is not averaged by `computeVertexNormals()`, so it shows a lighting line. Evaluate the last column as `i % uSegments` so seam positions are bit-identical.

**Poles.** A row that collapses to a point has a zero cross product: the normal is `(0, 0, 0)` or noise. Set it to the axis, or keep `v` inside `[0.001, 0.999]`.

**Curve frames.** `computeFrenetFrames(segments, closed)` is parallel transport, not Frenet. Pass `closed = true` on a closed path or the tube twists at the join. Sample with `getSpacedPoints` (arc length), not `getPoints` (parameter).

**After moving vertices** set `needsUpdate = true` on `position` and `normal` and call `geometry.computeBoundingSphere()`, or culling uses the old bounds.

**Infinity kills the mesh.** One infinite vertex (superformula with tiny `n1`) makes the bounding sphere NaN and the mesh disappears: clamp radii.

## Common Mistakes

- `new THREE.ParametricGeometry(fn, 32, 32)` → `import { ParametricGeometry } from "three/addons/geometries/ParametricGeometry.js"; new ParametricGeometry(fn, 32, 32)`: the class lives in addons and `THREE.ParametricGeometry` is undefined.
- `import { ParametricGeometries } from ".../ParametricGeometries.js"` → `import * as ParametricFunctions from "three/addons/geometries/ParametricFunctions.js"`: renamed in r175, and it now holds only functions.
- `(u, v) => new THREE.Vector3(x, y, z)` → `(u, v, target) => target.set(x, y, z)`: the addon reads `target` and ignores the return value, so returned vectors leave every vertex at the origin.
- `ParametricFunctions.plane(10, 10)` → `(u, v, target) => target.set(u * 10, 0, v * 10)`: the exported `plane` is itself the `(u, v, target)` function, not a factory.
- `new Float32Array(uSegments * vSegments * 3)` → `new Float32Array((uSegments + 1) * (vSegments + 1) * 3)`: N segments have N + 1 vertices per side.
- `new THREE.BufferAttribute(new Uint16Array(indices), 1)` on a large grid → `geometry.setIndex(indices)`: Uint16 indices wrap above 65535 vertices and connect the wrong ones.
- `geometry.computeVertexNormals()` on a surface with a duplicated seam column → analytic normals, or copy and average the seam normals.
- `curve.computeFrenetFrames(200)` on a closed curve → `curve.computeFrenetFrames(200, true)`: without `closed` the last frame is twisted against the first and the tube has a visible step.
- `curve.getPoints(n)` for evenly spaced rings or objects → `curve.getSpacedPoints(n)`: `getPoints` is uniform in the parameter, not in distance.
- Editing a curve's points and sampling again → set `curve.needsUpdate = true` (or call `curve.updateArcLengths()`) first: arc lengths are cached.
- `new THREE.ExtrudeGeometry(shape, { depth: 1 })` expecting a plain prism → add `bevelEnabled: false`: the bevel is on by default and enlarges the shape.
- `new THREE.ExtrudeGeometry(shape, { extrudePath: curve })` → add `steps: 100`: `steps` defaults to 1, which samples the path at its two ends only.
- `new LoftGeometry(sections)` with rings of different lengths → resample every section to the same point count: the addon logs an error and builds nothing.
- `stack.push(turtle)` → `stack.push({ position: position.clone(), quaternion: quaternion.clone(), depth })`: pushing the live object stores a reference, so `]` restores nothing.
- `effect.addBall(worldX, worldY, worldZ, 0.5, 12)` → pass coordinates in [0, 1] of the field cube: `addBall` works in normalized field space, while the mesh spans local [-1, 1] and is placed with `effect.scale` and `effect.position`.
- `effect.setCell(x, y, z, sdf(p))` with the default isolation → `effect.reset(); effect.isolation = 0; effect.setCell(x, y, z, -sdf(p)); effect.update();`: the addon treats higher values as inside, compares against `isolation` (80 by default) and reuses cached normals until `reset()`.
- `superformula(phi, 5, 1, 2, 6)` over `[0, 2 * Math.PI]` → run `phi` over `[0, 4 * Math.PI]` or use an even `m`: with `n2 != n3` an odd `m` is not 2 pi periodic and the outline closes with a kink.
- `material.positionNode = vec3(...)` on an instanced or skinned mesh → build on `positionLocal`: a bare `vec3` replaces the transformed position, instance matrix and skinning included.

## Reference Files

- `references/surfaces-grids.md` — ParametricGeometry sampling and normals, ParametricFunctions, the `buildSurface` grid recipe with an analytic torus, height fields, normals and tangents, seams and poles. Open for any `P(u, v)` surface or terrain grid.
- `references/curves-sweeps.md` — custom `Curve` subclasses, CurveExtras and NURBS, how `computeFrenetFrames` works, orienting objects on a path, TubeGeometry, varying-radius sweeps with LoftGeometry, lathe and extrude from computed profiles, `extrudePath`.
- `references/shapes-growth.md` — superformula outlines and supershapes, L-system expansion, the quaternion turtle into an InstancedMesh, branch radii, phyllotaxis.
- `references/marching-cubes.md` — MarchingCubes field layout, isolation, limits, metaballs, meshing an SDF.
- `references/webgpu-tsl.md` — running all of the above under WebGPURenderer; a torus evaluated in `positionNode` with an analytic `normalNode`.

## See Also

- `threejs-geometry` — `BufferGeometry` attributes and indices, BufferGeometryUtils, built-in shape parameters, merging and instancing the parts built here.
- `threejs-materials` — `side`, flat shading, vertex colors and normal maps for generated meshes.
- `threejs-math-noise` — noise and fBm to displace terrain grids and supershapes.
- `threejs-math-motion` — arc-length parametrization and timing along curves, camera rails.
- `threejs-math-sdf-raymarching` — SDF primitives and booleans that MarchingCubes can mesh.
- `threejs-shaders` — custom TSL beyond `positionNode` / `normalNode`.
- `threejs-performance` — when a generated mesh is too heavy: triangle budgets, merging, LOD.
- `procedural-gen` — engine-neutral generation of levels, dungeons and layouts.
