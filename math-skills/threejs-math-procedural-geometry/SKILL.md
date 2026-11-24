---
name: threejs-math-procedural-geometry
description: "Building three.js geometry from math: parametric surfaces with the ParametricGeometry addon, custom indexed grids, curves with rotation-minimizing frames, tubes, lathes, extrusions and lofts from computed profiles, superformula shapes, L-systems and recursive branching, analytic normals and tangents, seams and poles, and isosurfaces with the MarchingCubes addon. Use when generating meshes from equations, sweeping a profile along a path, growing plants or meshing a scalar field."
---

# Three.js Math: Procedural Geometry

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

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
const material = new THREE.MeshStandardMaterial({ color: 0x66aaff, roughness: 0.4, side: THREE.DoubleSide });
const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);

const timer = new THREE.Timer();
timer.connect(document);

renderer.setAnimationLoop(animate);

function animate() {
  timer.update();
  mesh.rotation.y = timer.getElapsed() * 0.3;
  renderer.render(scene, camera);
}
```

`ParametricGeometry` lives in addons; `THREE.ParametricGeometry` does not exist. This skill builds geometry on the CPU once (or when parameters change). For `BufferGeometry` attributes, indices, merging and instancing see `threejs-geometry`.

## Parametric Surfaces

`new ParametricGeometry(func, slices = 8, stacks = 8)` samples `func(u, v, target)` on a grid. `u` runs over `slices`, `v` over `stacks`, both in [0, 1]. It creates `(slices + 1) * (stacks + 1)` vertices, an index, `uv = (u, v)` and normals.

- The normals are finite differences of `func` with step 1e-5 (backward, forward at the borders): `normalize(cross(dP/du, dP/dv))`. They are not analytic, and seams are not welded: a closed surface has a duplicate column at `u = 0` and `u = 1`.
- Remap the domain inside `func`: `u * Math.PI * 2` for an angle, `(v - 0.5) * height` for a range.
- Orientation: the front face (counterclockwise) is the side `cross(dP/du, dP/dv)` points to. Swapping the roles of `u` and `v` flips it; `side: THREE.DoubleSide` hides the problem but shades the back with the same normal direction, so lighting looks wrong on a surface that is really inside out.
- `ParametricFunctions` exports `klein`, `plane`, `mobius`, `mobius3d`, each itself a `(u, v, target)` function. `plane(u, v, target)` writes `(u, 0, v)` and takes no size; scale with `mesh.scale`.

```javascript
import * as ParametricFunctions from "three/addons/geometries/ParametricFunctions.js";

const klein = new ParametricGeometry(ParametricFunctions.klein, 64, 32);
const strip = new ParametricGeometry(ParametricFunctions.mobius, 64, 8); // both are non-orientable: use DoubleSide
```

## Building the Grid Yourself

Write the grid by hand when you have a closed form for the normal, need extra attributes, build a large mesh, or update positions in place. The recipe takes a position function `P(u, v, target)` and a normal function `N(u, v, target)`:

```javascript
function buildSurface(P, N, uSegments, vSegments) {
  const cols = uSegments + 1;
  const rows = vSegments + 1;
  const positions = new Float32Array(cols * rows * 3);
  const normals = new Float32Array(cols * rows * 3);
  const uvs = new Float32Array(cols * rows * 2);
  const p = new THREE.Vector3();
  const n = new THREE.Vector3();
  for (let j = 0; j < rows; j++) {
    for (let i = 0; i < cols; i++) {
      const u = i / uSegments;
      const v = j / vSegments;
      const k = j * cols + i;
      P(u, v, p);
      N(u, v, n);
      positions.set([p.x, p.y, p.z], k * 3);
      normals.set([n.x, n.y, n.z], k * 3);
      uvs.set([u, v], k * 2);
    }
  }
  const indices = [];
  for (let j = 0; j < vSegments; j++) {
    for (let i = 0; i < uSegments; i++) {
      const a = j * cols + i;
      const b = a + 1;
      const c = a + cols;
      const d = c + 1;
      indices.push(a, b, d, a, d, c); // counterclockwise when cross(dP/du, dP/dv) faces the viewer
    }
  }
  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));
  geometry.setAttribute("normal", new THREE.BufferAttribute(normals, 3));
  geometry.setAttribute("uv", new THREE.BufferAttribute(uvs, 2));
  geometry.setIndex(indices); // a plain array picks Uint16 or Uint32 by vertex count
  return geometry;
}

// Torus around the Z axis; for P = ((R + r cos b) cos a, (R + r cos b) sin a, r sin b) the normal is closed-form
const R = 1;
const r = 0.4;
const torusP = (u, v, t) => {
  const a = u * Math.PI * 2;
  const b = v * Math.PI * 2;
  return t.set((R + r * Math.cos(b)) * Math.cos(a), (R + r * Math.cos(b)) * Math.sin(a), r * Math.sin(b));
};
const torusN = (u, v, t) => {
  const a = u * Math.PI * 2;
  const b = v * Math.PI * 2;
  return t.set(Math.cos(b) * Math.cos(a), Math.cos(b) * Math.sin(a), Math.sin(b));
};
const torus = buildSurface(torusP, torusN, 64, 24);
```

- Vertex count is `(uSegments + 1) * (vSegments + 1)`, index count `uSegments * vSegments * 6`. A 256 by 256 segment grid has 66049 vertices, past what a `Uint16Array` index can address, so pass a plain array to `setIndex`.
- Winding: with this index order the front face is the side `cross(dP/du, dP/dv)` points to. If the surface renders inside out, swap two indices per triangle and negate the normals together, never one alone.
- After changing positions in place: set `needsUpdate = true` on `position` and `normal`, then call `geometry.computeBoundingSphere()`; frustum culling uses the old sphere otherwise.
- A height field is the common in-place case, a terrain grid whose `y` comes from a function (swap in noise from `threejs-math-noise`):

```javascript
const size = 20;
const segments = 128;
const terrain = new THREE.PlaneGeometry(size, size, segments, segments).rotateX(-Math.PI / 2); // faces +Y
const height = (x, z) => 0.8 * Math.sin(x * 0.5) * Math.cos(z * 0.4);

const position = terrain.attributes.position;
const normal = terrain.attributes.normal;
const e = 0.01; // central difference step in world units
const n = new THREE.Vector3();
for (let k = 0; k < position.count; k++) {
  const x = position.getX(k);
  const z = position.getZ(k);
  position.setY(k, height(x, z));
  n.set(-(height(x + e, z) - height(x - e, z)) / (2 * e), 1, -(height(x, z + e) - height(x, z - e)) / (2 * e)).normalize();
  normal.setXYZ(k, n.x, n.y, n.z);
}
position.needsUpdate = true;
normal.needsUpdate = true;
terrain.computeBoundingBox();
terrain.computeBoundingSphere();
```

## Normals and Tangents

- Analytic: `n = normalize(cross(dP/du, dP/dv))`. Exact, needs no neighbors, and both copies of a seam vertex evaluate the same formula, so the seam shades smoothly.
- Height field `y = h(x, z)`: `n = normalize(-dh/dx, 1, -dh/dz)`, as above.
- A sphere-like surface `P = r(theta, phi) * dir` has normal `dir` only when `r` is constant. Take the cross product of the partials.
- `geometry.computeVertexNormals()` on an indexed geometry sums the area-weighted face normals of the triangles that share each index, then normalizes. On a non-indexed geometry every vertex gets its triangle's flat normal. Duplicated vertices are not averaged together, so a seam keeps a lighting line. Fixes: analytic normals; copy the normal of column 0 onto column `uSegments` (averaged) after computing; or drop the duplicate column and wrap indices with `% uSegments` when no UVs are needed.
- `BufferGeometryUtils.mergeVertices(geometry, tolerance)` merges only vertices whose attributes all match, so a seam whose `uv` differs stays split. `BufferGeometryUtils.toCreasedNormals(geometry, creaseAngle)` smooths only below the crease angle.
- Tangents for a normal map: call `geometry.computeTangents()` once `index`, `position`, `normal` and `uv` exist (it logs an error otherwise and writes a 4-component `tangent`). For the grid above it yields `w = +1`. The analytic form is `t = normalize(dP/du)` with `w = sign(dot(cross(n, t), dP/dv))`.

## Seams and Poles

- Closed in `u` (cylinder, torus): keep the duplicate column so `uv.x` reaches 1.0. Positions at `u = 0` and `u = 1` must match; `Math.sin(2 * Math.PI)` is `-2.4e-16`, not 0, so evaluate the last column as `i % uSegments` to make them bit-identical instead of relying on `u = 1`.
- Poles (sphere, supershape at `phi = +-pi/2`): a whole row collapses to one point, `dP/du = 0`, and the cross product is zero. On a `ParametricGeometry` sphere the normals of the pole at `v = 0` are exactly `(0, 0, 0)` (an unlit dot); at the `v = 1` pole they are built from rounding residue (`sin(pi)` is not exactly 0) and come out along the axis only by accident. Set the pole normal explicitly to the axis direction, or keep `v` inside `[0.001, 0.999]`. Pole-row triangles have zero area and break `computeTangents`.

## Curves

A curve is a subclass of `THREE.Curve` implementing `getPoint(t, target)`; `getPointAt`, `getPoints`, `getSpacedPoints`, `getLength`, `getTangent` and `computeFrenetFrames` come with it.

```javascript
class Lissajous extends THREE.Curve {
  getPoint(t, target = new THREE.Vector3()) {
    const a = t * Math.PI * 2;
    return target.set(Math.sin(3 * a), Math.sin(2 * a), Math.cos(5 * a));
  }
}
const path = new Lissajous();
const uniformInT = path.getPoints(200); // uniform in the parameter t
const uniformInDistance = path.getSpacedPoints(200); // uniform in arc length: use this for rings and spacing
```

`three/addons/curves/CurveExtras.js` ships ready-made curves: `GrannyKnot`, `HeartCurve`, `VivianiCurve`, `KnotCurve`, `HelixCurve`, `TrefoilKnot`, `TorusKnot`, `CinquefoilKnot`, `TrefoilPolynomialKnot`, `FigureEightPolynomialKnot` and `DecoratedTorusKnot4a/4b/5a/5c`. NURBS curves live in `three/addons/curves/NURBSCurve.js`. Timing along a curve by arc length is in `threejs-math-motion`.

## Frames Along a Curve

The Frenet frame is `T = P'/|P'|`, `N = T'/|T'|`, `B = T x N`. `N` is undefined where the curve is straight and flips 180 degrees at inflection points, which twists or pinches a swept profile. Do not build a sweep from the second derivative.

`curve.computeFrenetFrames(segments, closed)` returns `{ tangents, normals, binormals }`, each `segments + 1` `Vector3`. Despite the name it does not compute Frenet frames. It picks an initial normal perpendicular to the first tangent (along the smallest tangent component), then rotates the previous normal by the rotation that takes `T[i-1]` to `T[i]` (axis `T[i-1] x T[i]`, angle `acos(clamp(dot))`): parallel transport, a rotation-minimizing frame. `B = T x N`, so `N x B = T`. Tangents come from `getTangentAt`, so samples are uniform in arc length.

- On a closed curve pass `closed = true`: the code then spreads the leftover twist evenly so the last frame meets the first. On a `TrefoilKnot` with 100 segments the last normal sits 3.07 rad from the first without the flag and 0.002 rad with it.
- A sharp corner still pinches the profile: raise `segments` or round the path. The initial normal can jump when the start direction changes, so an animated path may spin as a whole. Frames also depend on `segments`; re-sampling changes the twist slightly.
- A ring point is `P + (N cos a + B sin a) * r`. To orient an object along the path, use the frame as a basis:

```javascript
const frames = path.computeFrenetFrames(100, true);
const points = path.getSpacedPoints(100); // same arc-length samples as the frames
const marker = new THREE.Mesh(new THREE.ConeGeometry(0.05, 0.2, 8).rotateX(Math.PI / 2), new THREE.MeshNormalMaterial());
marker.position.copy(points[10]);
const basis = new THREE.Matrix4();
marker.quaternion.setFromRotationMatrix(basis.makeBasis(frames.normals[10], frames.binormals[10], frames.tangents[10])); // local +Z follows the tangent
scene.add(marker);
```

## Sweeps: Tube, Lathe, Extrude, Loft

`TubeGeometry(path, tubularSegments = 64, radius = 1, radialSegments = 8, closed = false)` uses those frames (exposed as `geometry.tangents`, `normals`, `binormals`). Its radius is constant. For a radius or a profile that changes along the path, build one ring of points per frame and hand the rings to `LoftGeometry`:

```javascript
import { LoftGeometry } from "three/addons/geometries/LoftGeometry.js";

function sweepTube(curve, segments, radialSegments, radiusAt, closed = false) {
  const frames = curve.computeFrenetFrames(segments, closed);
  const points = curve.getSpacedPoints(segments);
  const sections = [];
  for (let i = 0; i <= segments; i++) {
    const radius = radiusAt(i / segments);
    const ring = [];
    for (let j = 0; j < radialSegments; j++) {
      const a = (j / radialSegments) * Math.PI * 2;
      ring.push(
        points[i].clone()
          .addScaledVector(frames.normals[i], Math.cos(a) * radius)
          .addScaledVector(frames.binormals[i], Math.sin(a) * radius),
      );
    }
    sections.push(ring);
  }
  return new LoftGeometry(sections, { closed: true, capStart: !closed, capEnd: !closed });
}

const tube = sweepTube(path, 200, 16, (s) => 0.1 + 0.06 * Math.sin(s * Math.PI * 8), true);
```

`LoftGeometry(sections, { closed = true, capStart = false, capEnd = false })` takes at least two sections of equal point count. `closed` repeats the first point of every section and averages the seam normals. Normals point outward when each section runs counterclockwise seen from the end of the loft looking back to the start; with `a` increasing from `N` toward `B` and `N x B = T` the rings above satisfy this. If a loft shows inside out, reverse each section. For a closed path the last section duplicates the first, which is intended: it closes the tube. The `uv` follows arc length.

Lathe, extrude and shape from a computed profile:

```javascript
// Lathe: x is the radius (>= 0), y the height; the profile revolves around Y
const n = 40;
const profile = Array.from({ length: n + 1 }, (_, i) => {
  const s = i / n;
  return new THREE.Vector2(0.15 + 0.35 * Math.sin(s * Math.PI) + 0.05 * Math.sin(s * 30), s * 2);
});
const vase = new THREE.LatheGeometry(profile, 48); // points listed bottom to top face outward

// Extrude a computed outline; Shape accepts an array of Vector2
const star = Array.from({ length: 10 }, (_, i) => {
  const a = (i / 10) * Math.PI * 2;
  const rr = i % 2 === 0 ? 1 : 0.45;
  return new THREE.Vector2(Math.cos(a) * rr, Math.sin(a) * rr);
});
const prism = new THREE.ExtrudeGeometry(new THREE.Shape(star), { depth: 0.3, bevelEnabled: false });
```

- Lathe normals come from the profile's segment directions, averaged at interior points. A profile that crosses the axis (negative `x`) turns inside out, and a point on the axis creates a pole.
- `ExtrudeGeometry` has `bevelEnabled: true` by default (`bevelThickness` 0.2, `bevelSize` 0.1): set `bevelEnabled: false` for a plain prism. Other options: `steps`, `bevelOffset`, `bevelSegments`, `curveSegments`, `UVGenerator`.
- With `extrudePath` (a 3D curve) the shape follows `getSpacedPoints(steps)` and `computeFrenetFrames(steps, closed)`. `depth` and bevels are ignored, `steps` defaults to 1 so raise it, and `closed` is read only from a `CatmullRomCurve3`.

## Superformula and Supershapes

`r(phi) = ( |cos(m phi / 4) / a|^n2 + |sin(m phi / 4) / b|^n3 )^(-1/n1)`

```javascript
function superformula(phi, m, n1, n2, n3, a = 1, b = 1) {
  const t1 = Math.pow(Math.abs(Math.cos((m * phi) / 4) / a), n2);
  const t2 = Math.pow(Math.abs(Math.sin((m * phi) / 4) / b), n3);
  return Math.min(Math.pow(t1 + t2, -1 / n1), 20); // clamp: Infinity breaks the bounding sphere
}

// 2D outline for ExtrudeGeometry or ShapeGeometry
const outline = Array.from({ length: 256 }, (_, i) => {
  const phi = (i / 256) * Math.PI * 2;
  const r = superformula(phi, 6, 1, 1, 1);
  return new THREE.Vector2(r * Math.cos(phi), r * Math.sin(phi));
});

// 3D: spherical product of two superformulas, u -> theta in [-pi, pi], v -> phi in [-pi/2, pi/2]
function supershape(u, v, target) {
  const theta = (u - 0.5) * Math.PI * 2;
  const phi = (0.001 + 0.998 * v - 0.5) * Math.PI; // stay off the poles
  const r1 = superformula(theta, 7, 0.3, 0.3, 0.3);
  const r2 = superformula(phi, 3, 1, 1, 1);
  return target.set(r1 * Math.cos(theta) * r2 * Math.cos(phi), r2 * Math.sin(phi), r1 * Math.sin(theta) * r2 * Math.cos(phi));
}
const blob = new ParametricGeometry(supershape, 128, 64);
```

- `m` is the lobe count; `m = 0` gives a circle, and so does `m = 4` with `n1 = n2 = n3 = 2` (r = 1).
- `r` is 2 pi periodic when `m` is an even integer. For odd `m` it is periodic only if `n2 = n3` and `a = b` (the shift swaps cos and sin); otherwise it closes with a kink at the seam, and `phi` must run over `[0, 4 pi]`.
- `n1 = 0` divides by zero and a tiny `n1` or `t1 + t2 = 0` gives Infinity. One infinite vertex makes the bounding sphere NaN and the mesh disappears, hence the clamp.

## L-Systems and Recursive Branching

Rewriting applies the rule map to every symbol in parallel. The rule `F -> F[&+F]F[^-F]F` multiplies the `F` count by 5 per iteration: 4 iterations give 625 segments, 5 give 3125, 7 give 78125. Cap iterations and the string length before building.

```javascript
function expand(axiom, rules, iterations, maxLength = 200000) {
  let s = axiom;
  for (let i = 0; i < iterations; i++) {
    let next = "";
    for (const c of s) next += rules[c] ?? c;
    if (next.length > maxLength) throw new Error("L-system too long");
    s = next;
  }
  return s;
}

function turtle(commands, angle, length) {
  const X = new THREE.Vector3(1, 0, 0);
  const Y = new THREE.Vector3(0, 1, 0);
  const Z = new THREE.Vector3(0, 0, 1);
  const turns = { "+": [Z, angle], "-": [Z, -angle], "&": [X, angle], "^": [X, -angle], "\\": [Y, angle], "/": [Y, -angle] };
  const segments = [];
  const stack = [];
  let position = new THREE.Vector3();
  let quaternion = new THREE.Quaternion();
  let depth = 0;
  const turn = new THREE.Quaternion();
  const heading = new THREE.Vector3();
  for (const c of commands) {
    if (c === "F") {
      const len = length * Math.pow(0.85, depth); // shorter segments per branch level
      segments.push({ start: position.clone(), quaternion: quaternion.clone(), length: len, depth });
      position.addScaledVector(heading.set(0, 1, 0).applyQuaternion(quaternion), len);
    } else if (turns[c]) {
      quaternion.multiply(turn.setFromAxisAngle(turns[c][0], turns[c][1])); // right-multiply: rotate about the turtle's own axis
    } else if (c === "[") {
      stack.push({ position: position.clone(), quaternion: quaternion.clone(), depth });
      depth++;
    } else if (c === "]") {
      ({ position, quaternion, depth } = stack.pop());
    }
  }
  return segments;
}

const commands = expand("F", { F: "F[&+F]F[^-F]F" }, 4);
const segments = turtle(commands, THREE.MathUtils.degToRad(25), 0.5);

// One InstancedMesh of a unit cylinder whose base sits at the origin, one instance per segment
const trunk = new THREE.CylinderGeometry(1, 1, 1, 6).translate(0, 0.5, 0);
const plant = new THREE.InstancedMesh(trunk, new THREE.MeshStandardMaterial({ color: 0x6b8e4e }), segments.length);
const matrix = new THREE.Matrix4();
const scale = new THREE.Vector3();
segments.forEach((seg, i) => {
  const radius = 0.03 * Math.pow(0.7, seg.depth);
  plant.setMatrixAt(i, matrix.compose(seg.start, seg.quaternion, scale.set(radius, seg.length, radius)));
});
plant.instanceMatrix.needsUpdate = true;
scene.add(plant);
```

- The heading is a quaternion, not Euler angles: accumulated Euler turns hit gimbal lock when a branch points along the middle axis. `q.multiply(turn)` rotates about the turtle's local axis; `premultiply` would rotate about a world axis.
- `[` and `]` push and pop clones. Pushing the live objects stores references and `]` restores nothing.
- Branch radius: parent radius^e = sum of child radius^e, with e between 2 (da Vinci's area-preserving rule) and 3. A constant factor per generation (0.7 above) is the simple version.
- Phyllotaxis (Vogel) for leaves, seeds and petals: element `k` sits at angle `k * 137.508` degrees (the golden angle, `pi * (3 - sqrt(5))` rad) and radius `c * sqrt(k)`.
- Recursive branching without strings: `branch(position, quaternion, length, radius, depth)` emits a segment, then calls itself for each child with a rotated copy of `quaternion` until `depth` reaches 0. Same output, no parsing. Vary angles with `THREE.MathUtils.seededRandom(seed)` so a plant is reproducible.

## Isosurfaces with MarchingCubes

`MarchingCubes(resolution, material, enableUvs = false, enableColors = false, maxPolyCount = 10000)` is a `Mesh` that polygonizes a scalar field on the CPU. The geometry is non-indexed with its own normals. Nothing is GPU-based, and nothing rebuilds on render: call `update()`.

- The mesh occupies the local cube [-1, 1]^3; place it with `scale` and `position`.
- `effect.field` is a `Float32Array(resolution^3)` indexed `x + y * resolution + z * resolution^2` (`setCell(x, y, z, value)`, `getCell`). Cell `x` sits at local `(x - resolution / 2) / (resolution / 2)`.
- The surface is where the field equals `effect.isolation` (default 80). Values above it are inside; normals point toward lower values.
- The outer layer of cells is not polygonized, so a shape that reaches the cube boundary is left open there.
- `maxPolyCount` bounds the triangle buffer; past it the console warns "Geometry buffers too small" and the mesh is cut off. Raise it with the resolution.
- The cost is a loop over `resolution^3` cells per `update()`. Resolution 28 to 64 is a starting range for per-frame updates, not a measurement; update a static field once.

Metaballs: per frame call `reset()`, `addBall(x, y, z, strength, subtract, color)` for each ball, then `update()`. Ball coordinates are in [0, 1] of the field cube, not world units. A ball's surface radius is about `sqrt(strength / (isolation + subtract))` as a fraction of the cube (0.074 for 0.5 and 12 at the default isolation); `sqrt(strength / subtract)` is where its field reaches zero. `addPlaneX/Y/Z(strength, subtract)` add wall and floor fields.

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

Meshing an arbitrary field or an SDF (the SDF functions are in `threejs-math-sdf-raymarching`). Inside must be the higher value, so write the negated distance and set the isolation to 0:

```javascript
const resolution = 48;
const effect = new MarchingCubes(resolution, new THREE.MeshStandardMaterial({ color: 0x66aaff }), false, false, 60000);
effect.isolation = 0;
effect.scale.setScalar(2);
scene.add(effect);

const sdf = (x, y, z) => Math.hypot(x, y, z) - 0.6 + 0.1 * Math.sin(8 * x) * Math.sin(8 * y) * Math.sin(8 * z);

effect.reset(); // zeroes the field and the cached normals; stale normals are reused otherwise
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

`enableUvs` and `enableColors` add those attributes; set `material.vertexColors = true` to see colors.

## WebGPU and TSL

Everything above builds a plain `BufferGeometry` on the CPU and renders unchanged under WebGPURenderer: import `THREE` from `three/webgpu` and use a node material or a classic material. `MarchingCubes` is a `Mesh` with dynamic attributes and works the same way. No GPU marching-cubes path ships in the package.

When the surface changes every frame, move the formula into the vertex stage instead of rebuilding buffers. A flat grid carries `uv()`; `positionNode` evaluates `P(u, v)` from it and `normalNode` the analytic normal. A pulsing torus (the tube radius is uniform over the surface, so the radial normal stays exact):

```javascript
import * as THREE from "three/webgpu";
import { TWO_PI, cos, sin, time, transformNormalToView, uv, vec3 } from "three/tsl";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111827); // the WebGPU canvas is transparent by default
const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 2, 5);
camera.lookAt(0, 0, 0);
scene.add(new THREE.DirectionalLight(0xffffff, 3), new THREE.HemisphereLight(0xffffff, 0x223344, 1));

const a = uv().x.mul(TWO_PI); // around the ring
const b = uv().y.mul(TWO_PI); // around the tube
const radius = sin(time).mul(0.1).add(0.35);
const ring = cos(b).mul(radius).add(1);

const material = new THREE.MeshStandardNodeMaterial({ color: 0x66aaff, roughness: 0.4 });
material.positionNode = vec3(ring.mul(cos(a)), ring.mul(sin(a)), sin(b).mul(radius)); // replaces the position outright
const normalObject = vec3(cos(b).mul(cos(a)), cos(b).mul(sin(a)), sin(b));
material.normalNode = transformNormalToView(normalObject); // normalNode expects a view-space normal

const mesh = new THREE.Mesh(new THREE.PlaneGeometry(1, 1, 128, 32), material); // the flat grid supplies only uv
mesh.frustumCulled = false; // the bounding sphere describes the flat 1 x 1 plane, not the torus
mesh.rotation.x = -Math.PI / 3;
scene.add(mesh);

renderer.setAnimationLoop(animate);

function animate() {
  renderer.render(scene, camera);
}
```

- `positionNode = vec3(...)` ignores the grid's own positions, and with them skinning, morphs and instancing. Add `positionLocal` instead when those must survive.
- A radius that varies over the surface needs the normal from the cross product of the partials, written out in TSL. When that is not worth it, use `flatShading: true` on the material.
- The uv grid is closed, so the duplicate seam column evaluates the same formula and shades smoothly.

## Common Mistakes

- `new THREE.ParametricGeometry(fn, 32, 32)` → `import { ParametricGeometry } from "three/addons/geometries/ParametricGeometry.js"; new ParametricGeometry(fn, 32, 32)`: the class lives in addons and `THREE.ParametricGeometry` is undefined.
- `(u, v) => new THREE.Vector3(x, y, z)` → `(u, v, target) => target.set(x, y, z)`: the addon reads `target` and ignores the return value, so returned vectors leave every vertex at the origin.
- `ParametricFunctions.plane(10, 10)` → `(u, v, target) => target.set(u * 10, 0, v * 10)`: the exported `plane` is itself the `(u, v, target)` function, not a factory.
- `new Float32Array(uSegments * vSegments * 3)` → `new Float32Array((uSegments + 1) * (vSegments + 1) * 3)`: a grid of N segments has N + 1 vertices per side.
- `new THREE.BufferAttribute(new Uint16Array(indices), 1)` on a large grid → `geometry.setIndex(indices)`: Uint16 indices wrap above 65535 vertices and connect the wrong ones; a plain array lets three pick Uint32.
- `geometry.computeVertexNormals()` on a surface with a duplicated seam column → analytic normals, or copy and average the seam normals: only triangles that share an index are averaged, so the seam shows a lighting line.
- `curve.computeFrenetFrames(200)` on a closed curve → `curve.computeFrenetFrames(200, true)`: without `closed` the last frame is twisted against the first (3.07 rad on a trefoil knot) and the tube has a visible step.
- `curve.getPoints(n)` for evenly spaced rings or objects → `curve.getSpacedPoints(n)`: `getPoints` is uniform in the parameter, not in distance.
- `new THREE.ExtrudeGeometry(shape, { depth: 1 })` expecting a plain prism → add `bevelEnabled: false`: the bevel is on by default and enlarges the shape.
- `new THREE.ExtrudeGeometry(shape, { extrudePath: curve })` → add `steps: 100`: `steps` defaults to 1, which samples the path at its two ends only.
- `stack.push(turtle)` → `stack.push({ position: position.clone(), quaternion: quaternion.clone(), depth })`: pushing the live object stores a reference, so `]` restores nothing.
- `effect.addBall(worldX, worldY, worldZ, 0.5, 12)` → pass coordinates in [0, 1] of the field cube: `addBall` works in normalized field space, while the mesh spans local [-1, 1] and is placed with `effect.scale` and `effect.position`.
- `effect.setCell(x, y, z, sdf(p))` with the default isolation → `effect.reset(); effect.isolation = 0; effect.setCell(x, y, z, -sdf(p)); effect.update();`: the addon treats higher values as inside, compares against `isolation` (80 by default) and reuses cached normals until `reset()`.
- `superformula(phi, 5, 1, 2, 6)` over `[0, 2 * Math.PI]` → run `phi` over `[0, 4 * Math.PI]` or use an even `m`: with `n2 != n3` an odd `m` is not 2 pi periodic and the outline closes with a kink.

## See Also

- `threejs-geometry` — `BufferGeometry` attributes and indices, `BufferGeometryUtils`, merging and instancing the segments built here.
- `threejs-materials` — `side`, flat shading, vertex colors and normal maps for generated meshes.
- `threejs-math-noise` — noise and fBm to displace terrain grids and supershapes.
- `threejs-math-motion` — arc-length parametrization and timing along curves, camera paths.
- `threejs-math-sdf-raymarching` — SDF primitives and booleans that MarchingCubes can mesh.
- `threejs-lighting` — lights and shadows for the generated meshes.
