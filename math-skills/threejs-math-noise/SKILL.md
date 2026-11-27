---
name: threejs-math-noise
description: "Procedural noise for three.js r187: PCG and sin-free hashes, value, Perlin, simplex and Worley noise, fBm, ridged, turbulence, domain warping, curl noise, seamless tiling and analytic derivatives in GLSL, the ImprovedNoise and SimplexNoise addons and the TSL mx_ noise nodes. Use for terrain, clouds, fire, marble, displacement, grid-aligned or banding noise, mx_fractal_noise_float or snoise ranges. Not for volumetric clouds (use threejs-math-atmosphere), advecting particles (use threejs-math-particles-fields) or terrain meshes (use threejs-math-procedural-geometry). 中文: 噪聲, 雜訊, 柏林噪聲, 分形噪聲, 程序化地形, 域扭曲, 旋度噪聲"
---

# Three.js Math: Noise

Targets three.js r187 (`three@0.187.0`). Examples use WebGLRenderer; the TSL material in `references/tsl-and-cpu.md` uses WebGPURenderer. For this topic r186 to r187 changed only the internal layout of the MaterialX noise nodes and a small `mx_noise_vec4` fix; every name and signature used here is the same.

## Quick Start

A subdivided plane displaced by fBm in the vertex shader, lit with a normal rebuilt from two neighbor samples. `noiseGLSL` holds the three helpers every other snippet in this skill builds on: the PCG hash `hash21`, `gradientNoise` and `fbm`.

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

const noiseGLSL = /* glsl */ `
  uint pcg(uint v) { // PCG hash, same constants as three's src/nodes/math/Hash.js
    uint state = v * 747796405u + 2891336453u;
    uint word = ((state >> ((state >> 28u) + 4u)) ^ state) * 277803737u;
    return (word >> 22u) ^ word;
  }
  float hash21(vec2 p) {
    uvec2 q = uvec2(ivec2(floor(p))); // through ivec2: negative float to uint is undefined
    return float(pcg(q.x + pcg(q.y)) >> 8u) * (1.0 / 16777216.0); // 24 bits: [0, 1)
  }
  vec2 grad2(vec2 i) {
    float a = 6.2831853 * hash21(i);
    return vec2(cos(a), sin(a)); // unit gradients
  }
  float gradientNoise(vec2 p) { // about [-0.707, 0.707], 0 at lattice points
    vec2 i = floor(p);
    vec2 f = fract(p);
    vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0); // quintic fade
    float a = dot(grad2(i), f);
    float b = dot(grad2(i + vec2(1.0, 0.0)), f - vec2(1.0, 0.0));
    float c = dot(grad2(i + vec2(0.0, 1.0)), f - vec2(0.0, 1.0));
    float d = dot(grad2(i + vec2(1.0, 1.0)), f - vec2(1.0, 1.0));
    return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
  }
  float fbm(vec2 p) {
    const mat2 m = mat2(1.6, 1.2, -1.2, 1.6); // rotate 36.87 degrees and scale by 2 per octave
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
      float diffuse = max(dot(normalize(vNormal), normalize(vec3(0.4, 0.8, 0.5))), 0.0); // view-space light
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

Per vertex: 15 noise evaluations (3 fBm calls, 5 octaves). The normal is rebuilt because the `normal` attribute still describes the flat plane. There is no noise chunk in `THREE.ShaderChunk`; GLSL noise is written by hand (ShaderMaterial rules: `threejs-shaders`).

## Choosing a Noise

| Basis | Look | Cost | Use |
| --- | --- | --- | --- |
| Value | Blocky, blobby, grid-aligned | Lowest: 4 hashes (2D), 8 (3D) | Cheap backgrounds, dissolve masks, roughness variation |
| Gradient (Perlin) | Smooth, some axis-aligned bias | 4 hashes and 4 dots (2D) | Terrain, clouds, water, general fBm |
| Simplex | Fewer directional artifacts | 3 corners in 2D, 4 in 3D, against 4 and 8 for gradient | 3D and 4D noise, animated volumes |
| Worley (cellular) | Cells, cracks, scales | 9 cells in 2D, 27 in 3D | Caustics, cracked earth, cloud billows when inverted |

| Where | Use when |
| --- | --- |
| CPU addon (`ImprovedNoise`, `SimplexNoise`) | Geometry built once, height maps, bakes, gameplay queries |
| GLSL in a `ShaderMaterial` | WebGLRenderer materials |
| TSL `mx_*` nodes | Node materials; one graph compiles for both WebGPURenderer backends |
| Baked texture (`DataTexture`, `Data3DTexture`) | Many lookups per pixel, such as volume marching: one fetch replaces a whole fBm |

## Output Ranges

Most "noise looks washed out / clipped" bugs are a wrong range assumption. Checked against the r187 sources; "measured" means sampled at 300k+ random points.

| Function | Range |
| --- | --- |
| `gradientNoise` (2D, unit gradients) | about ±0.707 (`sqrt(N)/2`); ×1.4 gives about ±1 |
| Quick Start `fbm` (weights sum to 0.96875) | about ±0.685 |
| `valueNoise`, `hash21`, `hash31` | [0, 1) |
| `ImprovedNoise.noise` | measured -0.89 to 0.98; exactly 0 at integer points |
| `SimplexNoise.noise` / `noise3d` / `noise4d` | measured just inside [-1, 1] |
| `mx_noise_float` | about [-1, 1] |
| `mx_fractal_noise_float`, 3 octaves | bounded by about ±1.75 (weights 1 + 0.5 + 0.25), not normalized |
| `mx_worley_noise_float` | F1 distance, Euclidean |
| `mx_worley_noise_vec2` | **squared** F1 and F2 (MaterialX metric 1) |
| `snoise` from `three/addons/tsl/math/curlNoise.js` | centred on 0.5, measured about -0.2 to 1.2 (JS port) |
| TSL `hash(seed)` | [0, 1], can be exactly 1.0 |

## Rules That Are Easy to Get Wrong

- Hash lattice coordinates with an integer hash (PCG). Convert `floor(p)` to `ivec` before `uvec`, keep 24 bits for the float, and never use `fract(sin(...))`: it depends on the GPU's `sin` precision and bands on mobile. Full hash set and a sin-free float hash: `references/glsl-bases.md`.
- Use the quintic fade. The cubic `f*f*(3-2f)` has a discontinuous second derivative that shows as grid creases in normals.
- fBm is not normalized. Divide by `(1 - g^N) / (1 - g)` (first weight 1) before mapping to [0, 1].
- Rotate the domain between octaves, or the lattices of all octaves line up as axis-aligned streaks. Tiling noise is the exception: integer lacunarity, no rotation.
- Octaves above the pixel Nyquist limit only alias: about `log2(pixels per noise unit)` octaves, 4 to 6 on surfaces.
- Ridged and turbulence fold a signed basis around 0. `abs()` of a [0, 1] noise does nothing.
- Curl noise: `v = (dpsi/dy, -dpsi/dx)` in 2D; in 3D the curl of three decorrelated noises. It is divergence-free only with exactly these signs. The advection belongs to `threejs-math-particles-fields`.
- Derivatives need the chain rule: `d/dp noise(freq * p) = freq * noise'`, and `transpose(M)` for a rotated octave.
- Seamless tiles: wrap the lattice index with `mod(i, period)` (integer period, doubled per octave), or sample 4D noise on a torus. A looping clip samples time on a circle. `RepeatWrapping` does not make noise periodic.
- Displacing in the vertex shader or `positionNode` leaves the `normal` attribute flat. Rebuild the normal from neighbor samples or analytic derivatives, or use `flatShading` on a node material.

## Common Mistakes

- `fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453)` → `hash21(p)` (integer PCG): the sin hash depends on GPU `sin` precision and bands or repeats on mobile and for large coordinates.
- `float(pcg(n)) / 4294967296.0` → `float(pcg(n) >> 8u) * (1.0 / 16777216.0)`: a 32-bit value does not fit a float significand and can round to exactly 1.0, which breaks `floor(r * count)` indexing.
- `uvec2(floor(p))` → `uvec2(ivec2(floor(p)))`: converting a negative float to uint is undefined in GLSL, so the pattern breaks left of the origin.
- `vec2 u = f * f * (3.0 - 2.0 * f);` → `vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);`: the cubic fade has a discontinuous second derivative, visible as grid creases in normals and displacement.
- `new ImprovedNoise().noise(x, y, 0)` at integer `x, y` → `noise(x * 0.1, y * 0.1, 0)`: Perlin noise is exactly 0 at every lattice point, so integer coordinates give a flat plane.
- `new SimplexNoise().noise3D(x, y, z)` → `noise3d(x, y, z)`: the methods are `noise` (2D), `noise3d` and `noise4d`.
- `new SimplexNoise({ random: () => THREE.MathUtils.seededRandom(42) })` → `THREE.MathUtils.seededRandom(42); new SimplexNoise({ random: () => THREE.MathUtils.seededRandom() })`: passing the seed on every call restarts the generator and fills the table with one value.
- `sum += amp * abs(valueNoise(p))` → `sum += amp * abs(gradientNoise(p))`: turbulence and ridged noise fold a signed basis; a [0, 1] basis has nothing to fold.
- `fbm(p) * 0.5 + 0.5` assumed in [0, 1] → divide by the weight sum first: fBm is not normalized, and `mx_fractal_noise_float` with 3 octaves reaches past ±1.
- `p *= 2.0;` between octaves → `p = mat2(1.6, 1.2, -1.2, 1.6) * p;`: without a rotation the lattices of all octaves line up and axis-aligned streaks show (keep plain `2.0` only for tiling noise).
- `float dhdx = valueNoiseD(p * freq).y;` → `float dhdx = freq * valueNoiseD(p * freq).y;`: the chain rule multiplies the derivative by the frequency (and by the amplitude).
- `noise3d(x, y, time)` for a looping clip → `noise4d(x, y, r * Math.cos(a), r * Math.sin(a))` with `a = 2 * Math.PI * t / T`: a straight line through noise never returns to its start; a circle does.
- `import { mx_perlin_noise_float } from "three/tsl"` → `import { mx_noise_float } from "three/tsl"`: the Perlin kernel is not exported; use the `mx_noise_*`, `mx_fractal_noise_*`, `mx_worley_noise_*` and `mx_cell_noise_float` wrappers.
- `const f = mx_worley_noise_vec2(p); f.y.sub(f.x)` → `const f = mx_worley_noise_vec2(p).sqrt();` first: the vec2 Worley wrapper returns squared distances, so borders come out thin and uneven.
- `snoise(p)` used as a [0, 1] mask or a signed basis → `snoise(p).sub(0.5)` for signed, `clamp()` for a mask: the addon's simplex is centred on 0.5 and overshoots both ends.

## Reference Files

- `references/glsl-bases.md`: PCG `hash21`/`hash31`, the sin-free float hash, value noise (2D and signed 3D), gradient noise notes, simplex pointers, Worley F1/F2. Open when writing a basis by hand.
- `references/glsl-fbm-curl.md`: fBm, ridged and turbulence code, domain warping, 2D and 3D curl in GLSL, analytic value-noise derivatives and eroded terrain, tiled gradient noise and looping, normals for displaced surfaces, terrain/cloud/fire/marble recipes.
- `references/tsl-and-cpu.md`: `ImprovedNoise`/`SimplexNoise` with seeding and torus baking into a `DataTexture`, the full WebGPURenderer node-material version of the Quick Start, the `mx_*` signature table, `hash`, `triNoise3D`, and the `curlNoise`/`snoise`/`voronoiNoise` addons.

## See Also

- `threejs-shaders`: ShaderMaterial rules, the two output includes, TSL basics, porting GLSL to TSL.
- `threejs-textures`: `DataTexture` and `Data3DTexture`, color spaces for baked noise.
- `threejs-math-sdf-raymarching`: noise displacing SDFs (and the step relaxation it needs).
- `threejs-math-atmosphere`: fBm as cloud density and light marching.
- `threejs-math-particles-fields`: advecting particles through curl noise.
- `threejs-math-procedural-geometry`: CPU terrain meshes from noise.
