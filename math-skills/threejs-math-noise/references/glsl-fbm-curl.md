# fBm, warping, curl, derivatives and tiling in GLSL

Purpose: the layered constructions built on a noise basis (fBm, ridged, turbulence, domain warping, curl noise, analytic derivatives, seamless and looping noise), normals for displaced surfaces, and short recipes. Open it once a basis from `glsl-bases.md` or the Quick Start is in place.

## fBm

fBm(p) = sum over i of `g^i * n(l^i * p)`, with lacunarity `l` (2.0) and gain `g` (0.5). The `fbm` in the SKILL.md Quick Start is this with 5 octaves and a rotation between octaves.

- The maximum amplitude is `(1 - g^N) / (1 - g)` when the first weight is 1: 1.9375 for g = 0.5 and N = 5. Divide by it to keep the range of the basis. The Quick Start starts at `amp = 0.5`, so its weights sum to 0.96875 and, with the ±0.707 gradient basis, it stays within about ±0.685.
- Octaves above the Nyquist limit of the pixel grid are pure aliasing. The count is bounded by `log2(pixels per noise unit)`: 4 to 6 for surfaces, fewer in volume marching.
- Lacunarity exactly 2.0 with no rotation stacks the lattice lines of every octave. Rotate the domain between octaves and optionally use 2.01 to 2.03. `mat2(1.6, 1.2, -1.2, 1.6)` is a 36.87 degree rotation scaled by 2 (determinant 4). Seamless tiling needs the opposite: integer lacunarity and no rotation.
- `mx_fractal_noise_float` (TSL) has no rotation between octaves and starts at weight 1.

## Ridged and turbulence

- Ridged: per octave `r = 1.0 - abs(n)`, then `r *= r`. Sharp crests for mountains and lightning.
- Turbulence: sum of `g^i * abs(n)`. Creases at the zero crossings for fire, marble veins and smoke.
- Both need a signed basis. `abs()` of a [0, 1] value noise changes nothing.

```glsl
float ridged(vec2 p) {
  const mat2 m = mat2(1.6, 1.2, -1.2, 1.6);
  float sum = 0.0, amp = 0.5;
  for (int i = 0; i < 5; i++) {
    float r = 1.0 - abs(gradientNoise(p) * 1.4); // signed basis, rescaled to about [-1, 1]
    sum += amp * r * r;
    p = m * p;
    amp *= 0.5;
  }
  return sum;
}

float turbulence(vec2 p) {
  const mat2 m = mat2(1.6, 1.2, -1.2, 1.6);
  float sum = 0.0, amp = 0.5;
  for (int i = 0; i < 5; i++) {
    sum += amp * abs(gradientNoise(p));
    p = m * p;
    amp *= 0.5;
  }
  return sum;
}
```

## Domain warping

Offset the lookup coordinates by another noise, twice. This is the two-level form and the offsets from Inigo Quilez, "Domain Warping" (iquilezles.org/articles/warp):

```glsl
float warped(vec2 p) {
  vec2 q = vec2(fbm(p), fbm(p + vec2(5.2, 1.3)));
  vec2 r = vec2(fbm(p + 4.0 * q + vec2(1.7, 9.2)), fbm(p + 4.0 * q + vec2(8.3, 2.8)));
  return fbm(p + 4.0 * r);
}
```

The constant offsets decorrelate the components; any pair of different offsets works. That is 5 fBm calls, so 25 noise evaluations per pixel at 5 octaves. One level (`fbm(p + 4.0 * q)`) costs 3 calls and already gives most of the swirl.

## Curl noise

In 2D, for a scalar potential psi(x, y): `v = (dpsi/dy, -dpsi/dx)`. In 3D, for a vector potential `Psi = (Psi_x, Psi_y, Psi_z)` of three decorrelated noises: `v = curl Psi = (dPsi_z/dy - dPsi_y/dz, dPsi_x/dz - dPsi_z/dx, dPsi_y/dx - dPsi_x/dy)`.

The curl of a smooth field has zero divergence (the mixed partial derivatives cancel), so particles advected by it neither bunch up nor leave holes. That is the reason to prefer it to raw noise as a velocity field. Advection and integration belong to `threejs-math-particles-fields`.

```glsl
vec2 curl2(vec2 p) { // central differences on a scalar potential
  const float e = 0.01;
  float dx = gradientNoise(p + vec2(e, 0.0)) - gradientNoise(p - vec2(e, 0.0));
  float dy = gradientNoise(p + vec2(0.0, e)) - gradientNoise(p - vec2(0.0, e));
  return vec2(dy, -dx) / (2.0 * e);
}

vec3 potential(vec3 p) { // three decorrelated signed noises; large, different offsets
  return vec3(valueNoise3(p), valueNoise3(p + vec3(31.4, 17.1, 5.9)), valueNoise3(p + vec3(7.7, 63.2, 28.4)));
}

vec3 curl3(vec3 p) {
  const float e = 0.01;
  vec3 dx = (potential(p + vec3(e, 0.0, 0.0)) - potential(p - vec3(e, 0.0, 0.0))) / (2.0 * e);
  vec3 dy = (potential(p + vec3(0.0, e, 0.0)) - potential(p - vec3(0.0, e, 0.0))) / (2.0 * e);
  vec3 dz = (potential(p + vec3(0.0, 0.0, e)) - potential(p - vec3(0.0, 0.0, e))) / (2.0 * e);
  return vec3(dy.z - dz.y, dz.x - dx.z, dx.y - dy.x);
}
```

- `valueNoise3` is in `glsl-bases.md`; any signed 3D noise works as the potential.
- 3D curl costs 6 potential evaluations, 18 scalar noises.
- The signs match `curlNoise` in `examples/jsm/tsl/math/curlNoise.js` (checked against the r187 source). One wrong sign still looks swirly but is no longer divergence-free.
- The step `e` must be small against the noise feature size but large enough for float precision; the TSL `curlNoise` uses a fixed 0.1.

## Analytic derivatives

Value noise can return its own gradient, which gives normals and slope-aware fBm without extra samples. The formulas follow Inigo Quilez, "Value Noise Derivatives" (iquilezles.org/articles/morenoise); his version maps the value to [-1, 1] and doubles the derivative, this one stays in [0, 1].

```glsl
// returns vec3(value in [0,1], d/dx, d/dy)
vec3 valueNoiseD(vec2 p) {
  vec2 i = floor(p);
  vec2 f = fract(p);
  vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);   // quintic fade: C2 continuous
  vec2 du = 30.0 * f * f * (f * (f - 2.0) + 1.0);       // its derivative
  float a = hash21(i);
  float b = hash21(i + vec2(1.0, 0.0));
  float c = hash21(i + vec2(0.0, 1.0));
  float d = hash21(i + vec2(1.0, 1.0));
  float k1 = b - a, k2 = c - a, k4 = a - b - c + d;
  return vec3(a + k1 * u.x + k2 * u.y + k4 * u.x * u.y,
              du * vec2(k1 + k4 * u.y, k2 + k4 * u.x));
}
```

Chain rule: the derivative of `noise(p * freq)` with respect to `p` is `freq * dNoise`. For `amp * noise(M * p)` it is `amp * transpose(M) * dNoise`. Forgetting the frequency factor is the usual bug.

```glsl
vec3 terrainD(vec2 p) { // x: height, yz: gradient of the undamped sum (approximate slope)
  float sum = 0.0, amp = 0.5, freq = 1.0;
  vec2 d = vec2(0.0);
  for (int i = 0; i < 6; i++) {
    vec3 n = valueNoiseD(p * freq);
    d += amp * freq * n.yz;               // chain rule: frequency and amplitude
    sum += amp * n.x / (1.0 + dot(d, d)); // steep areas get less detail, as if eroded
    freq *= 2.0;
    amp *= 0.5;
  }
  return vec3(sum, d);
}
```

This form has no per-octave rotation, so the lattices align; with a rotation `M` the gradient must be multiplied by `transpose(M)` per octave. Quilez's original accumulates the raw per-octave derivatives (`d += n.yz`) and rotates with `p = m * p * 2.0`; the damping idea is the same.

## Seamless and looping noise

- Lattice wrap: wrap the integer lattice coordinate before hashing, `hash21(mod(i + g, period))`, with an integer `period`. Every octave needs an integer multiple: lacunarity exactly 2, `period *= 2.0` per octave, no rotation.
- Torus mapping for a seamless 2D tile from 4D noise: `nx = R cos(2 pi u)`, `ny = R sin(2 pi u)`, `nz = R cos(2 pi v)`, `nw = R sin(2 pi v)`, value `noise4d(nx, ny, nz, nw)`. `R` sets the feature count per tile (the circle has circumference `2 pi R` noise units).
- Time loop: sample time on a circle, `noise4d(x, y, R cos(2 pi t / T), R sin(2 pi t / T))`. Scrolling `z = t` never returns to its start.
- `RepeatWrapping` on ordinary noise shows a seam: the wrap mode repeats the texture, it does not make the function periodic.

```glsl
float tiledGradient(vec2 p, float period) { // period: integer-valued float
  vec2 i = floor(p);
  vec2 f = fract(p);
  vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);
  float a = dot(grad2(mod(i, period)), f);
  float b = dot(grad2(mod(i + vec2(1.0, 0.0), period)), f - vec2(1.0, 0.0));
  float c = dot(grad2(mod(i + vec2(0.0, 1.0), period)), f - vec2(0.0, 1.0));
  float d = dot(grad2(mod(i + vec2(1.0, 1.0), period)), f - vec2(1.0, 1.0));
  return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}
```

Call it as `tiledGradient(uv * 8.0, 8.0)`; per octave `period *= 2.0` and `p *= 2.0`. GLSL `mod` is `x - y * floor(x / y)`, so negative lattice indices wrap correctly.

## Displacement and normals

`geometry.computeVertexNormals()` reads the CPU positions. A vertex-shader or `positionNode` displacement is invisible to it, and the `normal` attribute keeps describing the undisplaced surface, so lighting stays flat. Rebuild the normal in the shader:

- Height field `y = h(x, z)`: `normal = normalize(vec3(-dh/dx, 1, -dh/dz))`. With `valueNoiseD` the partials come with the value (apply the frequency and amplitude chain rule). The Quick Start uses forward differences instead, in the plane's local XY with z up.
- A surface displaced along its own normal: evaluate the displaced position at `p`, `p + t * eps` and `p + b * eps` (tangent and bitangent), then `normalize(cross(pT - p0, pB - p0))`. That is 3 displacement evaluations per vertex.
- Fragment fallback: `normalize(cross(dFdx(vViewPosition), dFdy(vViewPosition)))` gives faceted normals, where `vViewPosition` is a view-space position varying you pass from the vertex shader. With a node material, `flatShading: true` does the same.

## Recipes

- Terrain: `terrainD(p * 0.5).x` or `ridged(p)`; the damped form flattens valleys. Color by height and slope (`1.0 - normal.z` in the Quick Start's local frame).
- Clouds (2D layer): `float c = fbm(p) / 1.4 + 0.5; c = smoothstep(coverage, coverage + 0.25, c);` (the Quick Start `fbm` stays within about ±0.685, so the first `c` is in [0, 1]). Raising `coverage` removes cloud. Volumetric clouds and light marching belong to `threejs-math-atmosphere`.
- Fire: scroll fBm up and warp it, then shape it with a vertical gradient: `vec2 q = vec2(p.x, p.y - time * 1.5); float n = turbulence(q + 0.6 * vec2(fbm(q * 2.0), 0.0)); float flame = clamp(n * 1.6 - p.y, 0.0, 1.0);` where `p.y` runs 0 to 1 up the quad.
- Marble: `float m = sin(p.x * 6.0 + 4.0 * turbulence(p));` then `0.5 + 0.5 * m` as the blend factor between two colors.
