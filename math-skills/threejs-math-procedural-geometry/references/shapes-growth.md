# Superformula Shapes, L-Systems and Recursive Branching

Purpose: the superformula in 2D and as a 3D supershape, and growing plants with L-system rewriting, a quaternion turtle and one InstancedMesh (the maths is from the original skill; the three.js calls used are checked against r187).

## Superformula and Supershapes

`r(phi) = ( |cos(m phi / 4) / a|^n2 + |sin(m phi / 4) / b|^n3 )^(-1/n1)`

```javascript
import { ParametricGeometry } from "three/addons/geometries/ParametricGeometry.js";

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
const badge = new THREE.ShapeGeometry(new THREE.Shape(outline));

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

## L-Systems

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
plant.computeBoundingSphere(); // the instances, not the unit cylinder, define the culling bounds
scene.add(plant);
```

- The heading is a quaternion, not Euler angles: accumulated Euler turns hit gimbal lock when a branch points along the middle axis. `q.multiply(turn)` rotates about the turtle's local axis; `premultiply` would rotate about a world axis.
- `[` and `]` push and pop clones. Pushing the live objects stores references and `]` restores nothing.
- Branch radius: parent radius^e = sum of child radius^e, with e between 2 (da Vinci's area-preserving rule) and 3. A constant factor per generation (0.7 above) is the simple version.
- To bake the plant into one static mesh instead (one draw call, no per-instance data), apply each matrix to a cloned cylinder and merge with `BufferGeometryUtils.mergeGeometries` (see `threejs-geometry`).

## Phyllotaxis and String-Free Branching

- Phyllotaxis (Vogel) for leaves, seeds and petals: element `k` sits at angle `k * 137.508` degrees (the golden angle, `pi * (3 - sqrt(5))` rad) and radius `c * sqrt(k)`.
- Recursive branching without strings: `branch(position, quaternion, length, radius, depth)` emits a segment, then calls itself for each child with a rotated copy of `quaternion` until `depth` reaches 0. Same output, no parsing. Vary angles with `THREE.MathUtils.seededRandom(seed)` so a plant is reproducible.
