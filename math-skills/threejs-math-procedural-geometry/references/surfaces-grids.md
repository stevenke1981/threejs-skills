# Parametric Surfaces, Hand-Built Grids, Normals, Seams and Poles

Purpose: how the ParametricGeometry addon samples a surface, the hand-built indexed grid recipe with analytic normals, height fields, and fixing seams, poles and tangents (ParametricGeometry and BufferGeometry checked against the r187 source).

## ParametricGeometry

`new ParametricGeometry(func, slices = 8, stacks = 8)` samples `func(u, v, target)` on a grid. `u` runs over `slices`, `v` over `stacks`, both in [0, 1]. It creates `(slices + 1) * (stacks + 1)` vertices, an index, `uv = (u, v)` and normals.

- The normals are finite differences of `func` with step 1e-5 (backward, forward at the borders): `normalize(cross(dP/du, dP/dv))`. They are not analytic, and seams are not welded: a closed surface has a duplicate column at `u = 0` and `u = 1`.
- Remap the domain inside `func`: `u * Math.PI * 2` for an angle, `(v - 0.5) * height` for a range.
- Orientation: the front face (counterclockwise) is the side `cross(dP/du, dP/dv)` points to. Swapping the roles of `u` and `v` flips it; `side: THREE.DoubleSide` hides the problem but shades the back with the same normal direction, so lighting looks wrong on a surface that is really inside out.
- `ParametricFunctions` (renamed from `ParametricGeometries` in r175) exports `klein`, `plane`, `mobius`, `mobius3d`, each itself a `(u, v, target)` function. `plane(u, v, target)` writes `(u, 0, v)` and takes no size; scale with `mesh.scale`.

```javascript
import { ParametricGeometry } from "three/addons/geometries/ParametricGeometry.js";
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
  geometry.setIndex(indices); // a plain array picks Uint16 or Uint32 by the largest index
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

- Vertex count is `(uSegments + 1) * (vSegments + 1)`, index count `uSegments * vSegments * 6`. A 256 by 256 segment grid has 66049 vertices, past what a `Uint16Array` index can address (65535 is reserved for primitive restart), so pass a plain array to `setIndex`.
- Winding: with this index order the front face is the side `cross(dP/du, dP/dv)` points to. If the surface renders inside out, swap two indices per triangle and negate the normals together, never one alone.
- After changing positions in place: set `needsUpdate = true` on `position` and `normal`, then call `geometry.computeBoundingSphere()`; frustum culling uses the old sphere otherwise.

## Height Fields

A terrain grid whose `y` comes from a function (swap in noise from `threejs-math-noise`):

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
- `BufferGeometryUtils.mergeVertices(geometry, tolerance = 1e-4)` merges only vertices whose attributes all match, so a seam whose `uv` differs stays split. `BufferGeometryUtils.toCreasedNormals(geometry, creaseAngle = Math.PI / 3)` smooths only below the crease angle (it returns a non-indexed geometry).
- Tangents for a normal map: call `geometry.computeTangents()` once `index`, `position`, `normal` and `uv` exist (it logs an error otherwise and writes a 4-component `tangent`). For the grid above it yields `w = +1`. The analytic form is `t = normalize(dP/du)` with `w = sign(dot(cross(n, t), dP/dv))`.

## Seams and Poles

- Closed in `u` (cylinder, torus): keep the duplicate column so `uv.x` reaches 1.0. Positions at `u = 0` and `u = 1` must match; `Math.sin(2 * Math.PI)` is `-2.4e-16`, not 0, so evaluate the last column as `i % uSegments` to make them bit-identical instead of relying on `u = 1`.
- Poles (sphere, supershape at `phi = +-pi/2`): a whole row collapses to one point, `dP/du = 0`, and the cross product is zero. On a `ParametricGeometry` sphere the normals of the pole at `v = 0` come out as `(0, 0, 0)` (an unlit dot); at the `v = 1` pole they are built from rounding residue (`sin(pi)` is not exactly 0) and point along the axis only by accident. Set the pole normal explicitly to the axis direction, or keep `v` inside `[0.001, 0.999]`. Pole-row triangles have zero area and break `computeTangents`.
