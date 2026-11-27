# Noise on the CPU and in TSL

Purpose: the `ImprovedNoise` and `SimplexNoise` addons with seeding and texture baking, the TSL `mx_*` noise nodes with their exact r187 signatures and output ranges, the `curlNoise`/`snoise` and `voronoiNoise` addons, and a complete WebGPURenderer noise material. Open it for CPU noise, baked noise textures or node materials.

Signatures below were read from the r187 sources (`src/nodes/materialx/MaterialXNodes.js`, `MaterialXNoise.js`, `src/nodes/math/Hash.js`, `TriNoise3D.js`, `examples/jsm/math/*.js`, `examples/jsm/tsl/math/*.js`), and every import was resolved against the published `three@0.187.0` package. Ranges marked "measured" come from sampling the CPU addons, or a JS port of the TSL code, at a few hundred thousand random points.

## CPU addons

```javascript
import * as THREE from "three";
import { ImprovedNoise } from "three/addons/math/ImprovedNoise.js";
import { SimplexNoise } from "three/addons/math/SimplexNoise.js";

const perlin = new ImprovedNoise();
const h = perlin.noise(x * 0.1, y * 0.1, 0); // all three arguments required

// SimplexNoise fills its 256-entry table from r.random(); the default r is Math, so every instance differs.
THREE.MathUtils.seededRandom(42); // sets the seed (Mulberry32) and returns the first value
const simplex = new SimplexNoise({ random: () => THREE.MathUtils.seededRandom() }); // continues the sequence
const a = simplex.noise(x, y); // 2D
const b = simplex.noise3d(x, y, z);
const c = simplex.noise4d(x, y, z, w);
```

- `ImprovedNoise` is Ken Perlin's 2002 improved noise with his fixed permutation table: it cannot be seeded (offset the coordinates for a different pattern), repeats every 256 units on each axis (`floor(x) & 255`), and returns exactly 0 at every integer lattice point. Measured range about -0.89 to 0.98; treat it as roughly [-1, 1].
- `SimplexNoise` methods are `noise` (2D), `noise3d` and `noise4d` (lowercase `d`; `noise3D` does not exist). Measured range just inside [-1, 1] for all three.
- The table is filled with `Math.floor(r.random() * 256)`, so it is a random table with repeats, not a permutation. Passing `() => THREE.MathUtils.seededRandom(42)` reseeds on every call and fills all 256 entries with one value (measured: 1 distinct entry), which flattens the noise.
- The same seed gives the same table: two instances built after `seededRandom(42)` return identical values.

### Baking a tiling noise texture with the torus mapping

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

`DataTexture`, `Data3DTexture` and color spaces for data textures belong to `threejs-textures`. For volume marching, bake fBm into a `Data3DTexture` once instead of evaluating it per step.

## WebGPU and TSL material

`mx_*` noise compiles for both WebGPURenderer backends and replaces hand-written GLSL there. `glslFn`-wrapped noise only runs on the WebGL 2 backend (see `threejs-shaders`). The Quick Start terrain as a lit node material:

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

## The `mx_*` nodes (from `three/tsl`)

| Node | Signature (defaults) | Returns |
| --- | --- | --- |
| `mx_noise_float` | `(texcoord = uv(), amplitude = 1, pivot = 0)`, vec2 or vec3 | Perlin gradient noise, about [-1, 1], times `amplitude` plus `pivot` |
| `mx_noise_vec3`, `mx_noise_vec4` | same | three or four decorrelated Perlin channels |
| `mx_fractal_noise_float` | `(position = uv(), octaves = 3, lacunarity = 2, diminish = 0.5, amplitude = 1)`; the kernel is declared vec3, a vec2 is padded to `vec3(p, 0)` | fBm starting at weight 1, no rotation between octaves; 3 octaves are bounded by about ±1.75 |
| `mx_fractal_noise_float_2d` | same parameters, vec2 position | 2D fBm |
| `mx_fractal_noise_vec2`, `_vec3`, `_vec4` | same parameters, vec3 position | vector fBm |
| `mx_worley_noise_float` | `(texcoord = uv(), jitter = 1, style = 0)`, always the 3D kernel | F1, Euclidean distance; `style = 1` returns a random value per cell instead |
| `mx_worley_noise_float_2d` | `(texcoord = uv(), jitter = 1, style = 0)`, vec2 | the same in 2D, 9 cells |
| `mx_worley_noise_vec2` | `(texcoord = uv(), jitter = 1)` | `vec2(F1², F2²)`: the wrapper passes MaterialX metric 1, which is squared distance, so take `sqrt()` before `F2 - F1` |
| `mx_worley_noise_vec3` | `(texcoord = uv(), jitter = 1, metric = 1)` | F1, F2, F3; metric 0 Euclidean, 1 squared (the default), 2 Manhattan, 3 Chebyshev |
| `mx_cell_noise_float` | `(texcoord = uv())`, float to vec4 | one value per integer cell in [0, 1] (bits divided by 0xffffffff) |
| `mx_unifiednoise2d`, `mx_unifiednoise3d` | `(noiseType, texcoord, freq, offset, jitter, outmin, outmax, clampoutput, octaves, lacunarity, diminish, style)` | the MaterialX unified noise node |

- The internal kernels (`mx_perlin_noise_float` and friends) are not exported from `three/tsl`; import the `mx_noise_*` wrappers. (Checked: `mx_perlin_noise_float` is `undefined` in the r187 `three/tsl` module.)
- The fractal kernels other than `_2d` are declared with a vec3 parameter; a vec2 argument is converted to `vec3(p, 0)` by the node builder's type conversion (`NodeBuilder.format` in r187), so it works but samples the z = 0 slice of 3D noise. Use `mx_fractal_noise_float_2d` for true 2D fBm.
- `mx_worley_noise_float` always calls the 3D kernel (27 cells), so a vec2 becomes `vec3(p, 0)`; `mx_worley_noise_float_2d(texcoord, jitter, style)` is the 9-cell 2D version (both exported from `three/tsl`). `mx_worley_noise_vec2` and `mx_worley_noise_vec3` pick the 2D or 3D overload from the argument type.
- None of the `mx_*` functions takes a period, so they do not tile.
- r186 to r187 (diffed): names, parameters and defaults are unchanged. The kernels moved to the compact `Fn(..., { p: 'vec3', return: 'float' })` layout form, and `mx_noise_vec4` now builds its fourth channel for a vec3 input with a `vec3(19, 73, 29)` offset through a proper overload (r186 added a `vec2(19, 73)` offset to either input type).

Other noise nodes:

- `hash(seed)` from `three/tsl`: the PCG hash on a `uint` seed (for example `instanceIndex`), returning a float in [0, 1]; see `glsl-bases.md` for why 1.0 is possible.
- `triNoise3D(position, speed, time)` from `three/tsl`: cheap positive triangle-wave noise (port of cabbibo's glsl-tri-noise-3d), for fog wisps and flowing smoke.

```javascript
import { mx_noise_float, mx_noise_vec3, mx_worley_noise_float, mx_worley_noise_vec2, positionLocal, uv, vec3 } from "three/tsl";
import { curlNoise, snoise } from "three/addons/tsl/math/curlNoise.js";

const p = positionLocal.mul(2);
const cells = mx_worley_noise_float(vec3(uv().mul(8), 0)); // F1: distance to the nearest feature point
const f12 = mx_worley_noise_vec2(vec3(uv().mul(8), 0)).sqrt(); // squared F1, F2 -> distances
const walls = f12.y.sub(f12.x);                               // 0 on cell borders
const ridge = mx_noise_float(p).abs().oneMinus(); // ridged: fold the signed basis
const warped = mx_noise_float(p.add(mx_noise_vec3(p).mul(0.5))); // domain warp: offset the lookup by a vec3 noise
const flow = curlNoise(positionLocal.mul(0.5)); // divergence-free vec3; central differences with a fixed step of 0.1
const simplex = snoise(p); // centred on 0.5, not clamped to [0, 1]
```

## Addons in `examples/jsm/tsl/math/`

`curlNoise.js` exports `snoise`, `snoiseVec3`, `curlNoise` and `permute`:

- `snoise(v: vec3)` is a 3D simplex in the Ashima structure, but with a cubic falloff `m^3` and scale 12 plus an offset of 0.5 (the common GLSL version uses `m^4` and 42 for a signed ±1 result). Measured with a JS port: centred on 0.5, about -0.2 to 1.2. Subtract 0.5 for a signed basis; do not assume [0, 1].
- `snoiseVec3(x)` returns three decorrelated samples. In r187 the first component applies `mul(2).sub(1)` to the input coordinate instead of to the result, so that channel has twice the frequency and stays centred on 0.5; the curl is still divergence-free because any smooth potential works.
- `curlNoise(p: vec3)` takes central differences of `snoiseVec3` with a fixed step of 0.1: 6 `snoiseVec3`, 18 `snoise` calls.

`voronoiNoise.js` exports `voronoi2d(p, time)`, `voronoi3d(p, time)`, `hash2d`, `hash3d`: animated cellular noise whose feature points orbit inside their cells, returning the squared F1 distance, roughly [0, 1]. Its hashes are `fract(sin(...))`, so the pattern can differ between GPUs.
