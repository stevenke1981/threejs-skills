# SDF primitives, transforms and operators

Purpose: GLSL for the common exact primitives, domain transforms (rigid, scale, mirror, repetition, twist), booleans and the smooth minimum, with which operations keep an exact distance and which only give a bound. Open it when building or editing `map()`.

Formulas are checked against Inigo Quilez's articles: "Distance functions" (iquilezles.org/articles/distfunctions), "Smooth minimum" (/articles/smin), "Interior SDFs" (/articles/interiordistance) and "Domain repetition" (/articles/sdfrepetition).

## Primitives

`p` is already in the primitive's local space. All of these are exact distances, inside and outside.

```glsl
float sdSphere(vec3 p, float r) { return length(p) - r; }

float sdBox(vec3 p, vec3 b) { // b = half extents
  vec3 q = abs(p) - b;
  return length(max(q, 0.0)) + min(max(q.x, max(q.y, q.z)), 0.0);
}

float sdRoundBox(vec3 p, vec3 b, float r) { return sdBox(p, b - r) - r; } // outer size stays b; same as IQ's sdRoundBox

float sdTorus(vec3 p, float R, float r) { // ring in the xz plane
  return length(vec2(length(p.xz) - R, p.y)) - r;
}

float sdCapsule(vec3 p, vec3 a, vec3 b, float r) {
  vec3 pa = p - a, ba = b - a;
  float h = clamp(dot(pa, ba) / dot(ba, ba), 0.0, 1.0);
  return length(pa - ba * h) - r;
}

float sdCylinder(vec3 p, float r, float h) { // capped, axis y, half height h (IQ's sdCappedCylinder)
  vec2 d = abs(vec2(length(p.xz), p.y)) - vec2(r, h);
  return min(max(d.x, d.y), 0.0) + length(max(d, 0.0));
}

float sdPlane(vec3 p, vec3 n, float h) { return dot(p, n) + h; } // n must be unit length
```

- The second term of `sdBox` is the interior distance (negative inside). Without it the field is 0 everywhere inside: fine for tracing from outside, wrong for subtraction, AO and shells.
- `sd - r` grows any shape by `r` and rounds its convex edges. `abs(sd) - thickness` turns a solid into a shell.
- More primitives (cone, ellipsoid, triangle, Bezier) are in Quilez's distance-functions article; note which ones he marks as bound rather than exact (the ellipsoid is listed as "lower bound", the octahedron as "bound (not exact)").

## Transforms

A shape is moved by evaluating it at the inverse-transformed point: `sdBox(transpose(R) * (p - center), b)` for a rotation `R`. To drive an SDF shape from an `Object3D`, upload the inverse of its world matrix:

```javascript
import * as THREE from "three";

const proxy = new THREE.Object3D(); // position and rotate it like any object
const uInvModel = { value: new THREE.Matrix4() }; // add to the material's uniforms
proxy.updateMatrixWorld(); // each frame, after moving proxy
uInvModel.value.copy(proxy.matrixWorld).invert();
// GLSL: float d = sdBox((uInvModel * vec4(p, 1.0)).xyz, vec3(0.5)); exact for translation and rotation only
```

- Uniform scale: `sdShape(p / s) * s`. Without the trailing `* s` the distance is off by the factor `s`; the ray crawls (s > 1) or overshoots (s < 1). If the proxy carries a uniform scale, multiply the result by it.
- Mirror: `p.x = abs(p.x);` evaluates one half and gets the other for free (exact when the shape is symmetric about the plane).
- Infinite repetition with cell size `c`: `vec3 q = p - c * round(p / c);`, one copy centered on the origin. Limited repetition: `vec3 q = p - c * clamp(round(p / c), -l, l);` with `l` copies per side. Both are correct only when the shape fits inside its cell and is symmetric about the cell borders; otherwise the nearest surface can be in a neighbor cell, the distance is overestimated and holes appear. Quilez's repetition article evaluates the neighbor cell in the direction `sign(p - c * id)` as the fix.

### Operations that break the distance

Non-uniform scale, twist, bend and noise displacement stretch space, so the result is no longer a distance. Divide it by the largest stretch (the Lipschitz constant of the mapping), that is, multiply the step by a factor below 1, or the ray tunnels through thin parts.

A twist of `k` radians per unit of height is locally a shear of `k * r` at distance `r` from the axis. Its largest stretch is the largest singular value of `[[1, k r], [0, 1]]`, which is `(k * r + sqrt(k * k * r * r + 4.0)) / 2.0`; `sqrt(1.0 + k * k * r * r)` (the length of one column) is too small. Use the shape's largest `r`:

```glsl
vec3 opTwist(vec3 p, float k) { // rotates the xz plane by k radians per unit of y
  float c = cos(k * p.y), s = sin(k * p.y);
  return vec3(mat2(c, -s, s, c) * p.xz, p.y).xzy; // .xzy puts y back in the middle
}
// map: return sdBox(opTwist(p, 2.0), vec3(0.3, 1.0, 0.3)); trace: t += d * 0.6; (stretch 1.5 at r = 0.42)
```

Quilez's listing writes `vec3(m * p.xz, p.y)` and passes that to the primitive, which swaps the y and z axes of the twisted shape; the swizzle above keeps the axes.

For displacement `map(p) + amp * noise(freq * p)`, the extra stretch is bounded by `amp * freq * max|noise'|`; in practice start at `t += d * 0.5` to `0.7` and raise it until holes appear.

## Booleans

```glsl
float opUnion(float a, float b) { return min(a, b); }
float opIntersect(float a, float b) { return max(a, b); }
float opSubtract(float a, float b) { return max(a, -b); } // a with b carved out
```

- `min` is exact outside the union but only a bound inside it (Quilez, "Interior SDFs": the interior of a union is "totally broken" for shells, AO and negative-space modeling).
- `max`-based intersection and subtraction return bounds outside as well. All of them are safe to trace (they never overestimate); do not read the value as a physical distance away from the surface.

## Smooth minimum

```glsl
// k = blend width in distance units: smin == min wherever |a - b| >= k.
// The field drops by at most k / 4 (where a == b), so the blend adds material.
float smin(float a, float b, float k) {
  float h = max(k - abs(a - b), 0.0) / k;
  return min(a, b) - h * h * k * 0.25;
}

float smax(float a, float b, float k) { return -smin(-a, -b, k); }
```

- This is the quadratic polynomial smooth minimum. It is algebraically identical to Quilez's original `h = clamp(0.5 + 0.5 * (b - a) / k, 0.0, 1.0); mix(b, a, h) - k * h * (1.0 - h)`.
- Parameter convention: Quilez's current, normalized version starts with `k *= 4.0`, so his `k` is the maximum thickening and the blend region is `4k` wide. `smin(a, b, k)` here equals his with `k / 4`. Check which convention a copied snippet uses before tuning `k`.
- `k` must be above 0: `k = 0` divides by zero and returns NaN.
- Smooth subtraction is `smax(a, -b, k)`, smooth intersection `smax(a, b, k)`.
- The quadratic smin never overestimates: its gradient is a linear interpolation of the two input gradients, so its length is at most 1 (Quilez, "Smooth minimum", Gradients section). It is exact outside the blend region and a bound inside it.
- The exponential form `-log2(exp2(-k * a) + exp2(-k * b)) / k` is not locally supported (it changes the shapes everywhere) and in float32 it returns `+inf` once both `exp2` terms underflow to 0, which happens when `k * a` and `k * b` both exceed about 126 to 149 (far from both shapes; the exact point depends on denormal flushing), and overflows deep inside them. This is float32 arithmetic, not a measured GPU result. The polynomial form has neither problem.
- Blend a color or material across the seam with a matching weight, 1 where `a` is closer: `float h = clamp(0.5 + 0.5 * (b - a) / k, 0.0, 1.0); vec3 col = mix(colB, colA, h);`.

## Shapes from noise

Noise displacement turns a primitive into terrain or a rock: `sdSphere(p, 1.0) + 0.1 * fbm(p * 4.0)`. The noise functions belong to `threejs-math-noise`; the step relaxation above applies.
