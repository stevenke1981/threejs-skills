---
name: threejs-math-noise
description: "Procedural noise for three.js: integer and sin-free hashes, value, gradient, simplex and Worley noise, fBm, ridged and turbulence variants, domain warping, curl noise, seamless tiling and analytic derivatives, on the CPU with the ImprovedNoise and SimplexNoise addons, in GLSL, and with the TSL mx_ noise nodes. Use when generating terrain, clouds, fire, marble, water or displacement, or when a shader needs a random or organic pattern."
---

# Three.js Math: Noise

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start

A subdivided plane displaced by fBm in the vertex shader, lit with a normal rebuilt from two neighbor samples. The helper functions (`hash21`, `gradientNoise`, `fbm`) are defined in the sections below and are pasted into the shader as `noiseGLSL`.

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x0b1220);
const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 3, 5);
camera.lookAt(0, 0, 0);

// noiseGLSL = hash21 + gradientNoise + fbm from the sections below, as one string
const noiseGLSL = /* glsl */ `
  uint pcg(uint v) {
    uint state = v * 747796405u + 2891336453u;
    uint word = ((state >> ((state >> 28u) + 4u)) ^ state) * 277803737u;
    return (word >> 22u) ^ word;
  }
  float hash21(vec2 p) {
    uvec2 q = uvec2(ivec2(floor(p)));
    return float(pcg(q.x + pcg(q.y)) >> 8u) * (1.0 / 16777216.0);
  }
  vec2 grad2(vec2 i) {
    float a = 6.2831853 * hash21(i);
    return vec2(cos(a), sin(a));
  }
  float gradientNoise(vec2 p) {
    vec2 i = floor(p);
    vec2 f = fract(p);
    vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);
    float a = dot(grad2(i), f);
    float b = dot(grad2(i + vec2(1.0, 0.0)), f - vec2(1.0, 0.0));
    float c = dot(grad2(i + vec2(0.0, 1.0)), f - vec2(0.0, 1.0));
    float d = dot(grad2(i + vec2(1.0, 1.0)), f - vec2(1.0, 1.0));
    return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
  }
  float fbm(vec2 p) {
    const mat2 m = mat2(1.6, 1.2, -1.2, 1.6);
    float sum = 0.0, amp = 0.5;
    for (int i = 0; i < 5; i++) {
      sum += amp * gradientNoise(p);
      p = m * p;
      amp *= 0.5;
    }
    return sum;
  }
`;

const material = new THREE.ShaderMaterial({
  uniforms: { time: { value: 0 } },
  vertexShader: /* glsl */ `
    ${noiseGLSL}
    uniform float time;
    varying float vHeight;
    varying vec3 vNormal;

    float height(vec2 p) { return 0.9 * fbm(p * 0.7 + vec2(0.0, time * 0.15)); }

    void main() {
      vec2 p = position.xy; // the plane is built in XY, z is its normal
      float h = height(p);
      float e = 0.02;
      float dhdx = (height(p + vec2(e, 0.0)) - h) / e;
      float dhdy = (height(p + vec2(0.0, e)) - h) / e;
      vNormal = normalMatrix * normalize(vec3(-dhdx, -dhdy, 1.0));
      vHeight = h;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(p, h, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    varying float vHeight;
    varying vec3 vNormal;

    void main() {
      vec3 low = vec3(0.02, 0.10, 0.20);
      vec3 high = vec3(0.85, 0.80, 0.60);
      vec3 albedo = mix(low, high, smoothstep(-0.4, 0.5, vHeight));
      float diffuse = max(dot(normalize(vNormal), normalize(vec3(0.4, 0.8, 0.5))), 0.0);
      gl_FragColor = vec4(albedo * (0.25 + 0.9 * diffuse), 1.0);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
    }
  `,
});

const mesh = new THREE.Mesh(new THREE.PlaneGeometry(4, 4, 256, 256), material);
mesh.rotation.x = -Math.PI / 2; // lay the plane flat; local +z becomes world up
scene.add(mesh);

const timer = new THREE.Timer();
timer.connect(document);

renderer.setAnimationLoop(animate);

function animate() {
  timer.update();
  material.uniforms.time.value = timer.getElapsed();
  renderer.render(scene, camera);
}
```

Per vertex: 15 noise evaluations (3 fBm calls, 5 octaves). The normal is rebuilt because `normal` still describes the flat plane.

## Choosing a Noise

| Basis | Look | Cost | Use |
| --- | --- | --- | --- |
| Value | Blocky, blobby, grid-aligned | Lowest: 4 hashes (2D), 8 (3D) | Cheap backgrounds, dissolve masks, roughness variation |
| Gradient (Perlin) | Smooth, some axis-aligned bias | 4 hashes and 4 dots (2D) | Terrain, clouds, water, general fBm |
| Simplex | Fewer directional artifacts | 3 corners in 2D, 4 in 3D, against 4 and 8 for gradient | 3D and 4D noise, animated volumes |
| Worley (cellular) | Cells, cracks, scales | 9 cells in 2D, 27 in 3D | Caustics, cracked earth, cloud billows when inverted |

Where to compute it:

| Where | Use when |
| --- | --- |
| CPU addon (`ImprovedNoise`, `SimplexNoise`) | Geometry built once, height maps, bakes, gameplay queries |
| GLSL in a `ShaderMaterial` | WebGL materials; see `threejs-shaders` |
| TSL `mx_*` nodes | Node materials; one graph compiles for both backends |
| Baked texture (`DataTexture`, `Data3DTexture`) | Many lookups per pixel, such as volume marching: one fetch replaces a whole fBm |

There is no noise chunk in `THREE.ShaderChunk`; GLSL noise is written by hand.

## Hashes

An integer hash (PCG) returns the same bits on every GPU. The constants are the ones `src/nodes/math/Hash.js` uses.

```glsl
// Integer hash (PCG). Same result on every GPU. GLSL ES 3.00.
uint pcg(uint v) {
  uint state = v * 747796405u + 2891336453u;
  uint word = ((state >> ((state >> 28u) + 4u)) ^ state) * 277803737u;
  return (word >> 22u) ^ word;
}

float hash21(vec2 p) {
  uvec2 q = uvec2(ivec2(floor(p))); // through ivec2: float-to-uint of a negative number is undefined
  return float(pcg(q.x + pcg(q.y)) >> 8u) * (1.0 / 16777216.0); // 24 bits: exactly representable, result in [0, 1)
}

float hash31(vec3 p) {
  uvec3 q = uvec3(ivec3(floor(p)));
  return float(pcg(q.x + pcg(q.y + pcg(q.z))) >> 8u) * (1.0 / 16777216.0);
}
```

Dividing the full 32-bit result by 2^32 in float can round up to exactly 1.0, which is why the code keeps 24 bits. Hash lattice coordinates (integers) with these. For arbitrary float inputs a sin-free float hash works:

```glsl
// Sin-free float hash (Dave Hoskins, "Hash without Sine"). For non-integer inputs.
float hash12(vec2 p) {
  vec3 p3 = fract(vec3(p.xyx) * 0.1031);
  p3 += dot(p3, p3.yzx + 33.33);
  return fract((p3.x + p3.y) * p3.z);
}
```

The float hash loses quality beyond a few thousand. `fract(sin(dot(p, k)) * 43758.5453)` is not used here: its result depends on the precision of the GPU's `sin`, so the pattern differs between devices, bands for large inputs and degrades on mediump mobile GPUs.

## Value and Gradient Noise in GLSL

Value noise interpolates random values at lattice corners. Gradient noise interpolates the dot product of a random unit gradient with the offset to each corner. Both use the quintic fade `6t^5 - 15t^4 + 10t^3`, which has continuous first and second derivatives.

```glsl
float valueNoise(vec2 p) { // [0, 1)
  vec2 i = floor(p);
  vec2 f = fract(p);
  vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);
  return mix(mix(hash21(i), hash21(i + vec2(1.0, 0.0)), u.x),
             mix(hash21(i + vec2(0.0, 1.0)), hash21(i + vec2(1.0, 1.0)), u.x), u.y);
}
```

`gradientNoise` is in the Quick Start: signed, about [-0.7, 0.7] in 2D, and exactly 0 at every integer lattice point. Gradients must be unit length (`grad2` is): random vectors of uneven length make some cells visibly stronger. To get roughly [-1, 1] multiply by 1.4. Value noise is not signed, so `abs()` on it folds nothing (see ridged below).

## Simplex and Worley

For simplex use a tested implementation instead of a hand port. On the CPU, `SimplexNoise` (below). In TSL, `snoise` from `three/addons/tsl/math/curlNoise.js` returns about [0, 1] (it adds 0.5, so it is not signed). A GLSL 3D simplex is about 40 lines; `examples/jsm/tsl/math/curlNoise.js` shows its structure.

Worley noise returns distances to jittered feature points, one per cell:

```glsl
// returns vec2(F1, F2): distances to the nearest and second-nearest feature point
vec2 worley(vec2 p) {
  vec2 i = floor(p);
  vec2 f = fract(p);
  float f1 = 8.0, f2 = 8.0;
  for (int y = -1; y <= 1; y++) {
    for (int x = -1; x <= 1; x++) {
      vec2 g = vec2(float(x), float(y));
      vec2 o = vec2(hash21(i + g), hash21(i + g + vec2(37.0, 17.0))); // feature point inside the cell
      vec2 d = g + o - f;
      float dist = dot(d, d);
      if (dist < f1) { f2 = f1; f1 = dist; } else if (dist < f2) { f2 = dist; }
    }
  }
  return sqrt(vec2(f1, f2));
}
```

- Compare squared distances and take one `sqrt` at the end.
- The 3x3 neighborhood (3x3x3 = 27 cells in 3D) is the standard search, not an exact one: a feature point two cells away can occasionally be nearer. The error is rare for F1 and more visible for F2. A 2x2 search is visibly wrong.
- `F2 - F1` is zero on cell borders: cracks, scales, cell walls. `1.0 - F1` gives billowy blobs for clouds.

## fBm, Ridged, Turbulence

fBm(p) = sum over i of `g^i * n(l^i * p)`, with lacunarity `l` (2.0) and gain `g` (0.5). The `fbm` in the Quick Start is this with 5 octaves.

- The maximum amplitude is `(1 - g^N) / (1 - g)`: 1.9375 for g = 0.5 and N = 5. Divide by it to keep the range of the basis. The Quick Start starts at `amp = 0.5`, so its weights sum to 0.96875.
- Octaves above the Nyquist limit of the pixel grid are pure aliasing. The count is bounded by `log2(pixels per noise unit)`: 4 to 6 for surfaces, fewer in volume marching.
- Lacunarity exactly 2.0 with no rotation stacks the lattice lines of every octave. Rotate the domain between octaves and optionally use 2.01 to 2.03. `mat2(1.6, 1.2, -1.2, 1.6)` is a 36.87 degree rotation scaled by 2. Seamless tiling needs the opposite: integer lacunarity and no rotation.
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

## Domain Warping

Offset the lookup coordinates by another noise, twice (Inigo Quilez's two-level form):

```glsl
float warped(vec2 p) {
  vec2 q = vec2(fbm(p), fbm(p + vec2(5.2, 1.3)));
  vec2 r = vec2(fbm(p + 4.0 * q + vec2(1.7, 9.2)), fbm(p + 4.0 * q + vec2(8.3, 2.8)));
  return fbm(p + 4.0 * r);
}
```

The constant offsets decorrelate the components; any pair of different offsets works. That is 5 fBm calls, so 25 noise evaluations per pixel at 5 octaves. One level (`fbm(p + 4.0 * q)`) costs 3 calls and already gives most of the swirl.

## Curl Noise

In 2D, for a scalar potential psi(x, y): `v = (dpsi/dy, -dpsi/dx)`. In 3D, for a vector potential `Psi = (Psi_x, Psi_y, Psi_z)` of three decorrelated noises: `v = curl Psi = (dPsi_z/dy - dPsi_y/dz, dPsi_x/dz - dPsi_z/dx, dPsi_y/dx - dPsi_x/dy)`.

The curl of a smooth field has zero divergence, so particles advected by it neither bunch up nor leave holes. That is the reason to prefer it to raw noise as a velocity field. Advection and integration belong to `threejs-math-particles-fields`.

```glsl
vec2 curl2(vec2 p) { // central differences on a scalar potential
  const float e = 0.01;
  float dx = gradientNoise(p + vec2(e, 0.0)) - gradientNoise(p - vec2(e, 0.0));
  float dy = gradientNoise(p + vec2(0.0, e)) - gradientNoise(p - vec2(0.0, e));
  return vec2(dy, -dx) / (2.0 * e);
}

vec3 potential(vec3 p) { // three decorrelated signed value noises; large different offsets
  return vec3(noise3(p), noise3(p + vec3(31.4, 17.1, 5.9)), noise3(p + vec3(7.7, 63.2, 28.4)));
}

vec3 curl3(vec3 p) {
  const float e = 0.01;
  vec3 dx = (potential(p + vec3(e, 0.0, 0.0)) - potential(p - vec3(e, 0.0, 0.0))) / (2.0 * e);
  vec3 dy = (potential(p + vec3(0.0, e, 0.0)) - potential(p - vec3(0.0, e, 0.0))) / (2.0 * e);
  vec3 dz = (potential(p + vec3(0.0, 0.0, e)) - potential(p - vec3(0.0, 0.0, e))) / (2.0 * e);
  return vec3(dy.z - dz.y, dz.x - dx.z, dx.y - dy.x);
}
```

`noise3` is any signed 3D noise, for example a trilinear quintic value noise built on `hash31` and mapped to [-1, 1]. 3D curl costs 6 potential evaluations, 18 scalar noises. The signs above match `examples/jsm/tsl/math/curlNoise.js`; one wrong sign still looks swirly but is no longer divergence-free.

## Analytic Derivatives

Value noise can return its own gradient, which gives normals and slope-aware fBm without extra samples:

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

Chain rule: the derivative of `noise(p * freq)` with respect to `p` is `freq * dNoise`. For `amp * noise(M * p)` it is `amp * transpose(M) * dNoise`. Forgetting the frequency factor is the usual bug. The cubic fade `f*f*(3-2f)` has a discontinuous second derivative, which shows as grid creases once the noise feeds normals.

```glsl
vec3 terrainD(vec2 p) { // x: height, yz: gradient of the undamped sum (approximate slope)
  float sum = 0.0, amp = 0.5, freq = 1.0;
  vec2 d = vec2(0.0);
  for (int i = 0; i < 6; i++) {
    vec3 n = valueNoiseD(p * freq);
    d += amp * freq * n.yz;             // chain rule: frequency and amplitude
    sum += amp * n.x / (1.0 + dot(d, d)); // steep areas get less detail, as if eroded
    freq *= 2.0;
    amp *= 0.5;
  }
  return vec3(sum, d);
}
```

This form has no per-octave rotation, so the lattices align; with a rotation `M` the gradient must be multiplied by `transpose(M)` per octave.

## Seamless and Looping Noise

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
Call it as `tiledGradient(uv * 8.0, 8.0)`; per octave `period *= 2.0` and `p *= 2.0`.

## CPU Noise

```javascript
import * as THREE from "three";
import { ImprovedNoise } from "three/addons/math/ImprovedNoise.js";
import { SimplexNoise } from "three/addons/math/SimplexNoise.js";

const perlin = new ImprovedNoise();
const h = perlin.noise(x * 0.1, y * 0.1, 0); // all three arguments required; about [-1, 1]

// SimplexNoise builds its permutation from r.random(); the default is Math, so every instance differs.
THREE.MathUtils.seededRandom(42); // sets the seed (Mulberry32)
const simplex = new SimplexNoise({ random: () => THREE.MathUtils.seededRandom() }); // continues the sequence
const a = simplex.noise(x, y); // 2D
const b = simplex.noise3d(x, y, z);
const c = simplex.noise4d(x, y, z, w);
```

`ImprovedNoise` uses Ken Perlin's fixed permutation table: it cannot be seeded (offset the coordinates for a different pattern), repeats every 256 units on each axis, and returns exactly 0 at every integer lattice point. `SimplexNoise` methods are `noise` (2D), `noise3d` and `noise4d`; output is about [-1, 1].

Baking a tiling noise texture with the torus mapping:

```javascript
const size = 256;
const R = 1.5;
const data = new Uint8Array(size * size);
for (let y = 0; y < size; y++) {
  for (let x = 0; x < size; x++) {
    const u = (x / size) * Math.PI * 2;
    const v = (y / size) * Math.PI * 2;
    const n = simplex.noise4d(R * Math.cos(u), R * Math.sin(u), R * Math.cos(v), R * Math.sin(v));
    data[y * size + x] = Math.round((n * 0.5 + 0.5) * 255);
  }
}

const noiseTexture = new THREE.DataTexture(data, size, size, THREE.RedFormat, THREE.UnsignedByteType);
noiseTexture.wrapS = noiseTexture.wrapT = THREE.RepeatWrapping;
noiseTexture.minFilter = noiseTexture.magFilter = THREE.LinearFilter;
noiseTexture.needsUpdate = true; // colorSpace stays NoColorSpace: this is data, not a color
```

## Displacement and Normals

`geometry.computeVertexNormals()` reads the CPU positions. A vertex-shader or `positionNode` displacement is invisible to it, and the `normal` attribute keeps describing the undisplaced surface, so lighting stays flat. Rebuild the normal in the shader:

- Height field `y = h(x, z)`: `normal = normalize(vec3(-dh/dx, 1, -dh/dz))`. With `valueNoiseD` the partials come with the value (apply the frequency and amplitude chain rule). The Quick Start uses forward differences instead, in the plane's local XY with z up.
- A surface displaced along its own normal: evaluate the displaced position at `p`, `p + t * eps` and `p + b * eps` (tangent and bitangent), then `normalize(cross(pT - p0, pB - p0))`. That is 3 displacement evaluations per vertex.
- Fragment fallback: `normalize(cross(dFdx(vViewPosition), dFdy(vViewPosition)))` gives faceted normals, where `vViewPosition` is a view-space position varying you pass from the vertex shader. In TSL, `new THREE.MeshStandardNodeMaterial({ flatShading: true })` does the same.

## Recipes

- Terrain: `terrainD(p * 0.5).x` or `ridged(p)`; the damped form flattens valleys. Color by height and slope (`1.0 - normal.z`).
- Clouds: `float c = fbm(p) / 1.4 + 0.5; c = smoothstep(coverage, coverage + 0.25, c);` (this `fbm` stays within about +-0.68, so the first `c` is in [0, 1]). Raising `coverage` removes cloud. Volumetric lighting belongs to `threejs-math-atmosphere`.
- Fire: scroll fBm up and warp it, then shape it with a vertical gradient: `vec2 q = vec2(p.x, p.y - time * 1.5); float n = turbulence(q + 0.6 * vec2(fbm(q * 2.0), 0.0)); float flame = clamp(n * 1.6 - p.y, 0.0, 1.0);` where `p.y` runs 0 to 1 up the quad.
- Marble: `float m = sin(p.x * 6.0 + 4.0 * turbulence(p));` then `0.5 + 0.5 * m` as the blend factor between two colors.

## WebGPU and TSL

`mx_*` noise compiles for both backends and replaces hand-written GLSL under WebGPURenderer. `glslFn`-wrapped noise runs only on the WebGL 2 backend (see `threejs-shaders`). The Quick Start as a lit node material:

```javascript
import * as THREE from "three/webgpu";
import { color, mix, mx_fractal_noise_float, normalLocal, positionLocal, smoothstep, time, varying, vec3 } from "three/tsl";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x0b1220); // the WebGPU canvas is transparent by default
const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 3, 5);
camera.lookAt(0, 0, 0);

const sun = new THREE.DirectionalLight(0xffffff, 2);
sun.position.set(1, 2, 1.5);
scene.add(sun, new THREE.HemisphereLight(0xffffff, 0x223344, 0.6)); // lit material: black without lights

// vec3 position, 5 octaves, lacunarity 2, diminish 0.5; the sum is not normalized
const h = mx_fractal_noise_float(vec3(positionLocal.xy.mul(1.5), time.mul(0.1)), 5, 2, 0.5).mul(0.3);
const vHeight = varying(h, "vHeight"); // computed per vertex, interpolated to the fragment stage

const material = new THREE.MeshStandardNodeMaterial({ flatShading: true }); // vertex normals ignore positionNode
material.positionNode = positionLocal.add(normalLocal.mul(h));
material.colorNode = mix(color(0x061a33), color(0xd9cc99), smoothstep(-0.3, 0.4, vHeight));

const mesh = new THREE.Mesh(new THREE.PlaneGeometry(4, 4, 256, 256), material);
mesh.rotation.x = -Math.PI / 2;
scene.add(mesh);

renderer.setAnimationLoop(animate);

function animate() {
  renderer.render(scene, camera);
}
```

Other nodes, all from `three/tsl` unless noted. Pass a vec3 coordinate to the Worley and fractal nodes.

```javascript
import { mx_noise_float, mx_noise_vec3, mx_worley_noise_float, positionLocal, uv, vec3 } from "three/tsl";
import { curlNoise, snoise } from "three/addons/tsl/math/curlNoise.js";

const p = positionLocal.mul(2);
const cells = mx_worley_noise_float(vec3(uv().mul(8), 0)); // F1: distance to the nearest feature point
const ridge = mx_noise_float(p).abs().oneMinus(); // ridged: fold the signed basis
const warped = mx_noise_float(p.add(mx_noise_vec3(p).mul(0.5))); // domain warp: offset the lookup by a vec3 noise
const flow = curlNoise(positionLocal.mul(0.5)); // divergence-free vec3; central differences with a fixed step of 0.1
const simplex = snoise(p); // about [0, 1]
```

`mx_worley_noise_float(p, jitter, style)` with `style = 1` returns a random value per cell; `mx_worley_noise_vec2` returns F1 and F2 in `.x` and `.y`. `mx_noise_float` is Perlin gradient noise, about [-1, 1], for a vec2 or vec3 input; `mx_cell_noise_float(p)` gives one random value in [0, 1) per integer cell; `mx_fractal_noise_vec3` is the vector fBm. None of the `mx_*` functions takes a period, so they do not tile. For cheap fog wisps `triNoise3D(position, speed, time)` returns positive triangle-wave noise.

## Common Mistakes

- `fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453)` → `hash21(p)` (integer PCG): the sin hash depends on GPU `sin` precision and bands or repeats on mobile and for large coordinates.
- `float(pcg(n)) / 4294967296.0` → `float(pcg(n) >> 8u) * (1.0 / 16777216.0)`: a 32-bit value does not fit a float mantissa and can round to exactly 1.0, which breaks `floor(r * count)` indexing.
- `uvec2(floor(p))` → `uvec2(ivec2(floor(p)))`: converting a negative float to uint is undefined in GLSL, so the pattern breaks left of the origin.
- `vec2 u = f * f * (3.0 - 2.0 * f);` → `vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);`: the cubic fade has a discontinuous second derivative, visible as grid creases in normals and displacement.
- `new ImprovedNoise().noise(x, y, 0)` at integer `x, y` → `noise(x * 0.1, y * 0.1, 0)`: Perlin noise is exactly 0 at every lattice point, so integer coordinates give a flat plane.
- `new SimplexNoise().noise3D(x, y, z)` → `new SimplexNoise().noise3d(x, y, z)`: the methods are `noise` (2D), `noise3d` and `noise4d`.
- `new SimplexNoise({ random: () => THREE.MathUtils.seededRandom(42) })` → `THREE.MathUtils.seededRandom(42); new SimplexNoise({ random: () => THREE.MathUtils.seededRandom() })`: passing the seed on every call restarts the generator and fills the table with one value.
- `sum += amp * abs(valueNoise(p))` → `sum += amp * abs(gradientNoise(p))`: turbulence and ridged noise fold a signed basis around zero; a [0, 1] basis has nothing to fold.
- `fbm(p) * 0.5 + 0.5` assumed in [0, 1] → divide by the amplitude sum `(1.0 - pow(gain, N)) / (1.0 - gain)` first: fBm is not normalized, and `mx_fractal_noise_float` with 3 octaves reaches about +-1.75.
- `p *= 2.0;` between octaves → `p = mat2(1.6, 1.2, -1.2, 1.6) * p;`: without a rotation the lattices of all octaves line up and axis-aligned streaks show (keep plain `2.0` only for tiling noise).
- `float dhdx = valueNoiseD(p * freq).y;` → `float dhdx = freq * valueNoiseD(p * freq).y;`: the chain rule multiplies the derivative by the frequency (and by the amplitude).
- `noise3d(x, y, time)` for a looping clip → `noise4d(x, y, r * Math.cos(a), r * Math.sin(a))` with `a = 2 * Math.PI * t / T`: a straight line through noise never returns to its start; a circle does.
- `import { mx_perlin_noise_float } from "three/tsl"` → `import { mx_noise_float } from "three/tsl"`: the Perlin kernel is not exported; the entry points are `mx_noise_float`, `mx_fractal_noise_float`, `mx_worley_noise_float` and `mx_cell_noise_float`.

## See Also

- `threejs-shaders` — ShaderMaterial rules, the two output includes, TSL basics, porting GLSL to TSL
- `threejs-textures` — `DataTexture` and `Data3DTexture`, color spaces for baked noise
- `threejs-math-sdf-raymarching` — noise displacing SDFs
- `threejs-math-atmosphere` — fBm as cloud density and light marching
- `threejs-math-particles-fields` — advecting particles through curl noise
- `threejs-math-procedural-geometry` — CPU terrain meshes from noise
