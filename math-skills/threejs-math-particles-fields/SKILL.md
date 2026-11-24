---
name: threejs-math-particles-fields
description: "Particle motion from math in three.js: vector fields and curl-noise flow, Euler, semi-implicit Euler, Verlet and RK4 integrators and their stability, strange attractors such as Lorenz, Aizawa and Thomas, flocking rules, GPU simulation with GPUComputationRenderer on WebGL and TSL compute on WebGPU for 100k to 1M particles, and trails. Use when simulating particles, flow fields, attractors, swarms or any point system too large for a CPU loop."
---

# Three.js Math: Particles and Fields

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start

5,000 particles, each tracing its own path on the Lorenz attractor. The state lives in JS arrays, RK4 advances it, and the result is copied into a `BufferAttribute`.

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 3);

const COUNT = 5000;
const STEP = 0.005; // simulation time per RK4 step, not frame time
const STEPS_PER_FRAME = 4;
const state = new Float64Array(COUNT * 3);
const positions = new Float32Array(COUNT * 3);
for (let i = 0; i < COUNT; i++) {
  state[i * 3] = (Math.random() - 0.5) * 30; // scattered starts; nearby starts take seconds to separate
  state[i * 3 + 1] = (Math.random() - 0.5) * 30;
  state[i * 3 + 2] = Math.random() * 40 + 5;
}

function lorenz(x, y, z, out) {
  out[0] = 10 * (y - x);        // sigma = 10
  out[1] = x * (28 - z) - y;    // rho = 28
  out[2] = x * y - (8 / 3) * z; // beta = 8/3
}

const k1 = [0, 0, 0], k2 = [0, 0, 0], k3 = [0, 0, 0], k4 = [0, 0, 0];
function rk4(i, dt) {
  const x = state[i], y = state[i + 1], z = state[i + 2];
  const h = dt / 2;
  lorenz(x, y, z, k1);
  lorenz(x + k1[0] * h, y + k1[1] * h, z + k1[2] * h, k2);
  lorenz(x + k2[0] * h, y + k2[1] * h, z + k2[2] * h, k3);
  lorenz(x + k3[0] * dt, y + k3[1] * dt, z + k3[2] * dt, k4);
  state[i] = x + (dt / 6) * (k1[0] + 2 * k2[0] + 2 * k3[0] + k4[0]);
  state[i + 1] = y + (dt / 6) * (k1[1] + 2 * k2[1] + 2 * k3[1] + k4[1]);
  state[i + 2] = z + (dt / 6) * (k1[2] + 2 * k2[2] + 2 * k3[2] + k4[2]);
}

for (let s = 0; s < 1500; s++) for (let i = 0; i < COUNT * 3; i += 3) rk4(i, STEP); // pre-warm: spread along the attractor
const geometry = new THREE.BufferGeometry();
geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));
const points = new THREE.Points(
  geometry,
  new THREE.PointsMaterial({ size: 0.01, color: 0x66aaff, transparent: true, opacity: 0.6, blending: THREE.AdditiveBlending, depthWrite: false }),
);
points.frustumCulled = false; // the bounding sphere was computed from the first positions
scene.add(points);

const timer = new THREE.Timer();
timer.connect(document);

renderer.setAnimationLoop(animate);

function animate() {
  timer.update();
  for (let s = 0; s < STEPS_PER_FRAME; s++) {
    for (let i = 0; i < COUNT * 3; i += 3) rk4(i, STEP);
  }
  for (let i = 0; i < COUNT * 3; i += 3) {
    positions[i] = state[i] * 0.05;
    positions[i + 1] = (state[i + 2] - 25) * 0.05; // Lorenz z (centered near 25) is up: wings face the camera
    positions[i + 2] = state[i + 1] * 0.05;
  }
  geometry.attributes.position.needsUpdate = true;
  points.rotation.y = timer.getElapsed() * 0.1;
  renderer.render(scene, camera);
}
```

Each frame runs a fixed number of steps, so the speed depends on the frame rate; use the accumulator under Stability when it must not.

## Fields

A velocity field `F(x)` gives each point a velocity: `x' = F(x)`. A tracer has no state beyond its position. A force field `a(x)` gives an acceleration: `v' = a(x)`, `x' = v`. Particles then carry velocity, inertia and drag, and overshoot the field's flow lines.

Divergence measures net outflow of a velocity field. A field with `div F = 0` neither compresses nor expands a cloud of tracers: density stays what it was at the start. The curl of any smooth vector field has zero divergence, so a curl-noise field gives swirling flow with no sinks where particles pile up and no sources where they thin out. A radial pull (`F = -x`) has negative divergence and collapses everything to a point. The curl definition and its GLSL and TSL code belong to `threejs-math-noise`; this skill advects with it.

Sample a field at the particle position, once per `a` evaluation. A grid or texture field needs trilinear interpolation (a 3D texture with `LinearFilter`).

## Integrators

State `x`, `v`, acceleration `a(x, v)`:

| Method | Step | Order | `a` evaluations | Notes |
| --- | --- | --- | --- | --- |
| Explicit Euler | `x += v*dt; v += a(x_old)*dt` | 1 | 1 | Gains energy on any oscillator; orbits spiral outward |
| Semi-implicit (symplectic) Euler | `v += a(x)*dt; x += v*dt` | 1 | 1 | Same cost, bounded energy error; the default for particles |
| Verlet | `x_new = 2x - x_prev + a(x)*dt²` | 2 | 1 | No stored velocity; needs a constant `dt`; easy constraints |
| RK4 | weighted average of 4 slopes | 4 | 4 | Smooth non-stiff ODEs where accuracy matters (attractors) |

The two Euler variants differ only in the order of two lines. Update velocity first and use the new velocity for the position.

```javascript
const gravity = new THREE.Vector3(0, -9.8, 0);
const drag = 0.5;
// Semi-implicit Euler: the default
function stepSemiImplicit(p, dt) {
  p.velocity.addScaledVector(gravity, dt);
  p.velocity.multiplyScalar(Math.exp(-drag * dt));
  p.position.addScaledVector(p.velocity, dt);
}

// Position Verlet with damping: p.previous holds the position one step ago
const next = new THREE.Vector3();
function stepVerlet(p, dt) {
  next.copy(p.position).sub(p.previous).multiplyScalar(1 - drag * dt); // implicit velocity, damped
  p.previous.copy(p.position);
  p.position.add(next).addScaledVector(gravity, dt * dt);
}
```

Verlet takes the velocity from `x - x_prev`, which assumes the previous step had the same `dt`. With a varying `dt`, scale that term by `dt / dt_prev` or use a fixed step; without a constant step it is not more accurate than Euler.

RK4 for a state `s` and `ds/dt = f(s)`: `k1 = f(s); k2 = f(s + k1*dt/2); k3 = f(s + k2*dt/2); k4 = f(s + k3*dt); s += dt/6 * (k1 + 2*k2 + 2*k3 + k4)`. The Quick Start has the scalar version. The extra three evaluations pay off for smooth fields and cost a lot on the GPU when the field is a noise function.

A massless tracer in `x' = F(x)` has no velocity. Euler works, but in rotating fields it drifts outward. The midpoint rule fixes most of it for one extra field sample:

```javascript
const mid = new THREE.Vector3();
function advectMidpoint(position, field, dt) {
  field(position, mid);                     // F(x)
  mid.multiplyScalar(dt / 2).add(position); // x + F(x) dt / 2
  field(mid, mid);                          // F at the midpoint
  position.addScaledVector(mid, dt);
}
```

## Stability and Time Steps

- Spring `a = -ω² x`, `ω = sqrt(k / m)`: semi-implicit Euler and Verlet are stable only while `ω * dt < 2`. Explicit Euler multiplies the amplitude by `sqrt(1 + ω² dt²)` every step, whatever `dt` is. In a test with `ω = 1`, `dt = 0.1`, 1,000 steps, the energy of the explicit variant grew from 0.5 to about 10,000, while the semi-implicit one stayed near 0.5. A stiff spring needs a smaller `dt` or substeps, not more damping.
- Drag: `v *= Math.exp(-drag * dt)`. `v *= 0.98` per frame is a different damping at every frame rate.
- Clamp the frame delta: `const dt = Math.min(timer.getDelta(), 1 / 30);`. A long frame (tab switch, shader compile) otherwise becomes one huge step.
- Fixed step with an accumulator, for results that do not depend on the frame rate:
```javascript
const STEP = 1 / 120;
let accumulator = 0;
function simulate(frameDelta) {
  accumulator += Math.min(frameDelta, 0.1);
  for (let n = 0; accumulator >= STEP && n < 8; n++) { // capped: a slow frame must not cause more work next frame
    step(STEP);
    accumulator -= STEP;
  }
}
```

- NaN is permanent. One particle that divides by zero (`normalize` of a zero vector, `1 / r²` at `r = 0`) stays NaN and, in flocking, infects its neighbors. Guard with `d / max(length(d), 1e-6)` and soften gravity with `1 / (r² + ε²)`.

## Strange Attractors

Each system is `(x', y', z')` with the constants below. All six were run in Node with RK4 at the listed step for 200,000 steps from `(0.1, 0, 0)` (Dadras from `(0.1, 0.1, 0.1)`) and stayed finite; the extents are from those runs and shift a little with the start point.

| Name | Equations | Constants | Step | Extent |
| --- | --- | --- | --- | --- |
| Lorenz | σ(y − x), x(ρ − z) − y, xy − βz | σ = 10, ρ = 28, β = 8/3 | 0.005 | x, y within about ±26, z from 0 to 49 |
| Rössler | −y − z, x + ay, b + z(x − c) | a = 0.2, b = 0.2, c = 5.7 | 0.01 | x, y within about ±11, z from 0 to 23 |
| Aizawa | (z − b)x − dy, dx + (z − b)y, c + az − z³/3 − (x² + y²)(1 + ez) + f·z·x³ | a = 0.95, b = 0.7, c = 0.6, d = 3.5, e = 0.25, f = 0.1 | 0.01 | x, y within ±1.5, z from −0.4 to 1.9 |
| Thomas | sin y − bx, sin z − by, sin x − bz | b = 0.208186 | 0.05 | each axis from −1.1 to 3.8 |
| Halvorsen | −ax − 4y − 4z − y², −ay − 4z − 4x − z², −az − 4x − 4y − x² | a = 1.89 | 0.005 | each axis from −12 to 6.5 |
| Dadras | y − ax + byz, cy − xz + z, dxy − ez | a = 3, b = 2.7, c = 1.7, d = 2, e = 9 | 0.005 | x within ±20, y, z within about ±14 |

- The steps are simulation time per step. Run a fixed number per frame; do not feed the frame delta. Explicit Euler on Lorenz stayed finite at a step of 0.02 and reached NaN at 0.03 in a test; RK4 at 0.005 is far inside its limit.
- Chaotic systems amplify error: different frame rates or float32 against float64 give different curves on the same attractor shape. RK4 does not make a trajectory reproducible across step sizes.
- Start off the fixed points. The origin is a fixed point of Lorenz, Thomas, Halvorsen and Dadras. Use `(0.1, 0, 0)` plus a small per-particle offset on all three axes; in Dadras the whole x axis decays to the origin, so a start with `y = z = 0` dies. Small offsets take several time units to spread over the attractor; scatter the starts over its extent and pre-warm, as the Quick Start does, to show the full shape on the first frame.
- Lorenz is off-center: subtract about 25 from z and scale by 0.05 to fit a unit-scale scene. Scale every other system by `1 / extent`.
- Two uses: one long curve (a `Line` filled progressively, with `geometry.setDrawRange(0, n)`) or many particles that each advance their own state.

## Curl-Noise Flow

Advect tracers through `F(x) = curl(noise(x * frequency))` with the midpoint rule above. The step is `speed * dt`; keep a particle's per-frame move below about half the noise feature size, or it skips over structure. For a field that evolves, add time as an offset inside the noise: `curl(noise(x * frequency + vec3(0, 0, t * 0.1)))`. A slow `t` makes particles flow along changing streamlines; a fast `t` makes them jitter.

A divergence-free field does not keep tracers uniform forever: integration error still lets them clump slowly. Respawn them (below), and in a bounded volume add a weak pull toward the origin.

The GPU forms of the same loop are in the two GPU sections: `curlNoise` from `three/addons/tsl/math/curlNoise.js` for TSL, and your own GLSL curl from `threejs-math-noise` for the WebGL compute shader. In TSL a tracer step is `p.addAssign(curlNoise(p.mul(0.5)).mul(dt.mul(speed)))` (graph not executed here; only the import was checked).

## Flocking

Boid `i` looks at neighbors `j` within radius `R`:

- Separation: `sum (p_i - p_j) / max(|p_i - p_j|², ε)` over neighbors closer than a smaller radius.
- Alignment: `mean(v_j) - v_i`.
- Cohesion: `mean(p_j) - p_i`.

`a = wSep * sep + wAli * ali + wCoh * coh`, integrated with semi-implicit Euler, then the speed is rescaled into `[vMin, vMax]`. A minimum speed keeps boids from stalling.

```javascript
function flockAccel(b, boids, R = 2, Rsep = 0.7) {
  const sep = new THREE.Vector3(), ali = new THREE.Vector3(), coh = new THREE.Vector3(), d = new THREE.Vector3();
  let n = 0;
  for (const o of boids) { // all pairs: O(N²)
    if (o === b) continue;
    d.subVectors(b.position, o.position);
    const r2 = d.lengthSq();
    if (r2 > R * R) continue;
    if (r2 < Rsep * Rsep) sep.addScaledVector(d, 1 / Math.max(r2, 1e-4));
    ali.add(o.velocity);
    coh.add(o.position);
    n++;
  }
  if (n > 0) { ali.divideScalar(n).sub(b.velocity); coh.divideScalar(n).sub(b.position); }
  return sep.multiplyScalar(1.5).addScaledVector(ali, 1.0).addScaledVector(coh, 0.5);
}
// per boid afterwards: velocity += accel * dt; rescale speed into [1, 4]; position += velocity * dt
```

Neighbor search is the cost. All pairs is N²: 4,096 boids make 16.8 million pair tests per frame, within reach of a GPU pass; 100,000 boids make 10^10, too many for an all-pairs pass every frame. Large flocks need a spatial grid (a hash of cell index to boid list); this skill does not implement one. The weights and radii above are starting points to tune by eye.

## Choosing Where to Simulate

| Where | Typical use | State lives in | Renderer |
| --- | --- | --- | --- |
| CPU arrays | Up to roughly tens of thousands of particles, complex per-particle logic, collisions with JS objects | `Float32Array` + `needsUpdate` | any |
| `GPUComputationRenderer` | Hundreds of thousands to about a million, field-driven motion | RGBA float textures, ping-pong | `WebGLRenderer` |
| TSL compute | Same range, arbitrary buffers, no texture packing | storage buffers (`instancedArray`) | `WebGPURenderer` |

These thresholds are rules of thumb; measure with `threejs-performance`. With 1M additive points the overdraw, not the simulation, is usually the limit. `GPUComputationRenderer` takes a `WebGLRenderer` and builds `ShaderMaterial`s, so it does not run under `WebGPURenderer`.

## GPGPU on WebGL: GPUComputationRenderer

`new GPUComputationRenderer(sizeX, sizeY, renderer)` simulates `sizeX * sizeY` particles, one per texel (512 x 512 = 262,144; 1024 x 1024 = 1,048,576). Each variable is a texture updated by a fragment shader. The addon prepends `uniform sampler2D <name>;` for every dependency and defines `resolution` as `vec2(sizeX, sizeY)`. Targets are `NearestFilter`, `ClampToEdgeWrapping`, RGBA `FloatType`.

In one `compute()` call every variable reads the previous frame's textures of all its dependencies, whatever the order of `addVariable`. The position pass therefore sees last frame's velocity. The code below repeats the velocity update inside the position shader so that it integrates with this frame's velocity (semi-implicit Euler); dropping those two lines gives explicit Euler, which damped systems tolerate.

```javascript
import * as THREE from "three";
import { GPUComputationRenderer } from "three/addons/misc/GPUComputationRenderer.js";

const SIZE = 512, count = SIZE * SIZE, MAX_LIFE = 6;

const common = /* glsl */ `
  uniform float dt;
  uniform float frame;

  uint pcg(uint v) {
    uint state = v * 747796405u + 2891336453u;
    uint word = ((state >> ((state >> 28u) + 4u)) ^ state) * 277803737u;
    return (word >> 22u) ^ word;
  }
  float hash31(vec3 p) {
    uvec3 q = uvec3(ivec3(floor(p)));
    return float(pcg(q.x + pcg(q.y + pcg(q.z))) >> 8u) * (1.0 / 16777216.0);
  }

  // Force field: pull toward the origin plus a swirl around the y axis
  vec3 acceleration(vec3 pos) {
    return -0.4 * pos + 0.8 * vec3(-pos.z, 0.0, pos.x);
  }

  vec3 spawnPosition() {
    vec3 seed = vec3(gl_FragCoord.xy, frame * 3.0);
    return (vec3(hash31(seed), hash31(seed + vec3(0.0, 0.0, 1.0)), hash31(seed + vec3(0.0, 0.0, 2.0))) - 0.5) * 4.0;
  }
`;

const velocityShader = /* glsl */ `
  ${common}
  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;
    vec4 pos = texture2D(texturePosition, uv);
    vec3 vel = texture2D(textureVelocity, uv).xyz;
    if (pos.w - dt < 0.0) { // the position pass respawns this texel: start at rest
      gl_FragColor = vec4(0.0, 0.0, 0.0, 1.0);
      return;
    }
    vel += acceleration(pos.xyz) * dt;
    vel *= exp(-0.5 * dt);
    gl_FragColor = vec4(vel, 1.0);
  }
`;

const positionShader = /* glsl */ `
  ${common}
  void main() {
    vec2 uv = gl_FragCoord.xy / resolution.xy;
    vec4 pos = texture2D(texturePosition, uv);
    vec3 vel = texture2D(textureVelocity, uv).xyz; // last frame's velocity
    float life = pos.w - dt;
    if (life < 0.0) {
      gl_FragColor = vec4(spawnPosition(), ${MAX_LIFE.toFixed(1)});
      return;
    }
    vel += acceleration(pos.xyz) * dt; // redo the velocity update to get this frame's velocity
    vel *= exp(-0.5 * dt);
    gl_FragColor = vec4(pos.xyz + vel * dt, life);
  }
`;

const gpu = new GPUComputationRenderer(SIZE, SIZE, renderer);
if (!renderer.extensions.has("EXT_color_buffer_float")) gpu.setDataType(THREE.HalfFloatType); // before init()

const pos0 = gpu.createTexture(), vel0 = gpu.createTexture();
const posData = pos0.image.data, velData = vel0.image.data; // Float32Array, 4 floats per texel
for (let i = 0; i < count; i++) {
  for (let c = 0; c < 3; c++) posData[i * 4 + c] = (Math.random() - 0.5) * 4;
  posData[i * 4 + 3] = Math.random() * MAX_LIFE; // staggered lives
  velData[i * 4 + 3] = 1;
}

const velVar = gpu.addVariable("textureVelocity", velocityShader, vel0);
const posVar = gpu.addVariable("texturePosition", positionShader, pos0);
gpu.setVariableDependencies(velVar, [posVar, velVar]);
gpu.setVariableDependencies(posVar, [posVar, velVar]);
for (const v of [velVar, posVar]) {
  v.material.uniforms.dt = { value: 0 };
  v.material.uniforms.frame = { value: 0 };
}
const error = gpu.init();
if (error !== null) console.error(error); // init returns a message, it does not throw
```

Draw with `THREE.Points`; each vertex reads its own texel through a `reference` attribute holding the texel center.

```javascript
const references = new Float32Array(count * 2);
for (let i = 0; i < count; i++) {
  references[i * 2] = ((i % SIZE) + 0.5) / SIZE; // + 0.5: texel centers
  references[i * 2 + 1] = (Math.floor(i / SIZE) + 0.5) / SIZE;
}
const geometry = new THREE.BufferGeometry();
geometry.setAttribute("position", new THREE.BufferAttribute(new Float32Array(count * 3), 3)); // vertex count only
geometry.setAttribute("reference", new THREE.BufferAttribute(references, 2));

const material = new THREE.ShaderMaterial({
  uniforms: {
    texturePosition: { value: null },
    size: { value: 0.06 },
    scale: { value: window.innerHeight * 0.5 },
  },
  vertexShader: /* glsl */ `
    attribute vec2 reference;
    uniform sampler2D texturePosition;
    uniform float size;
    uniform float scale;
    varying float vLife;
    void main() {
      vec4 data = texture2D(texturePosition, reference);
      vLife = data.w;
      vec4 mvPosition = modelViewMatrix * vec4(data.xyz, 1.0);
      gl_PointSize = size * (scale / -mvPosition.z);
      gl_Position = projectionMatrix * mvPosition;
    }
  `,
  fragmentShader: /* glsl */ `
    varying float vLife;
    void main() {
      float r = length(gl_PointCoord - 0.5);
      if (r > 0.5) discard;
      float fade = smoothstep(0.0, 0.5, vLife) * (1.0 - r * 2.0);
      gl_FragColor = vec4(vec3(1.0, 0.55, 0.2) * 0.25, fade);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
    }
  `,
  transparent: true,
  depthWrite: false,
  blending: THREE.AdditiveBlending,
});
const points = new THREE.Points(geometry, material);
points.frustumCulled = false; // the placeholder positions are all zero, so the bounding sphere is a point
scene.add(points);

function animate() {
  timer.update();
  const dt = Math.min(timer.getDelta(), 1 / 30);
  for (const v of [velVar, posVar]) {
    v.material.uniforms.dt.value = dt;
    v.material.uniforms.frame.value += 1;
  }
  gpu.compute();
  material.uniforms.texturePosition.value = gpu.getCurrentRenderTarget(posVar).texture; // alternates every frame
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

- Point sprites are clamped by the GPU's maximum point size and cost fill rate; keep them a few pixels wide.
- Half-float state has 11 significant bits, a relative precision of about 1/1024. Positions near 1000 units resolve to roughly one unit. Keep the simulation near the origin and in a small range, or store offsets.
- Compute shaders write data. They end with `gl_FragColor = ...;` and no `#include <colorspace_fragment>`.
- `gpu.dispose()` frees the targets and materials. The `w` channel is free for life, mass or a seed.

## Lifetime and Respawn

Life lives in `pos.w` and decreases by `dt`. At `life < 0` the shader writes a new position from a hash of the texel and the frame counter, as above, and the velocity pass resets that texel's velocity by testing the same condition on the same previous-frame texture. Two rules:

- Stagger the initial lives (`life = random * maxLife`). With equal lives every particle respawns in the same frame and the cloud pulses.
- Use an integer hash (`pcg`, copied from `threejs-math-noise`). A sin-based hash bands on mobile GPUs and for large inputs. The hash input floors its argument, so the frame counter must change by at least 1 per frame; a continuous `time` value would repeat the same spawn for many frames.

## Trails

1. History in geometry. For one curve or a few hundred heads, keep a ring of the last N positions and draw it as `Line` or `LineSegments`. Shift with `array.copyWithin(3, 0, (N - 1) * 3)`, write the new head at index 0, set `needsUpdate`.
2. Screen feedback. `AfterimagePass` (WebGL) or `afterImage()` (WebGPU) multiplies the previous frame by `damp` and keeps the per-channel maximum with the new frame, so the trail length depends on the frame rate. For frame-rate independence, set `damp = Math.exp(-rate * dt)` each frame.
3. Stretched sprites. Orient and scale a quad along the velocity with `length` proportional to `|v|`. No history is stored, and the trail is exact for straight motion only.

```javascript
import { AfterimagePass } from "three/addons/postprocessing/AfterimagePass.js";

const afterimage = new AfterimagePass(0.9); // add after RenderPass in the EffectComposer chain
// per frame: afterimage.damp = Math.exp(-6 * dt);
```

Feedback trails smear the whole frame; render over a dark background. Composer rules are in `threejs-postprocessing`.

## WebGPU and TSL

Same simulation with storage buffers. `instancedArray(count, "vec3")` creates a GPU buffer, `buffer.element(instanceIndex)` reads and writes one entry, `Fn(...)().compute(count)` builds a dispatch, and `renderer.compute(node)` runs it. A write to a storage buffer is in place: there is no previous-frame copy as in the WebGL ping-pong, so `p` and `v` below read the values just written, and one pass gives semi-implicit Euler.

```javascript
import * as THREE from "three/webgpu";
import { Fn, float, hash, instanceIndex, instancedArray, uniform, vec3 } from "three/tsl";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before compute() or render()
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x050508); // the WebGPU canvas is transparent by default
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 6);
const timer = new THREE.Timer();
timer.connect(document);

const count = 200000;
const positions = instancedArray(count, "vec3");
const velocities = instancedArray(count, "vec3");
const dt = uniform(0);

const init = Fn(() => {
  const p = positions.element(instanceIndex);
  p.assign(vec3(hash(instanceIndex), hash(instanceIndex.add(count)), hash(instanceIndex.add(count * 2))).sub(0.5).mul(4));
  velocities.element(instanceIndex).assign(vec3(0));
})().compute(count);

const update = Fn(() => {
  const p = positions.element(instanceIndex);
  const v = velocities.element(instanceIndex);
  v.addAssign(p.negate().mul(dt)); // acceleration toward the origin
  v.mulAssign(dt.mul(-0.5).exp()); // exponential drag
  p.addAssign(v.mul(dt));          // uses the updated velocity: semi-implicit Euler
})().compute(count);

renderer.compute(init); // once, after await renderer.init()

const material = new THREE.SpriteNodeMaterial({ transparent: true, depthWrite: false, blending: THREE.AdditiveBlending });
material.positionNode = positions.toAttribute();
material.colorNode = vec3(1, 0.6, 0.2);
material.scaleNode = float(0.02); // sprite size in world units
const particles = new THREE.Sprite(material);
particles.count = count;
particles.frustumCulled = false;
scene.add(particles);

function animate() {
  timer.update();
  dt.value = Math.min(timer.getDelta(), 1 / 30);
  renderer.compute(update);
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

- Use your own `uniform` for `dt`; whether the built-in `deltaTime` node updates inside a compute dispatch was not verified.
- `SpriteNodeMaterial` multiplies `positionNode` by the full model-view matrix, so the sprite's `position`, `rotation` and `scale` transform the whole cloud. `particles.scale.setScalar(0.02)` shrinks every particle position as well as the sprites; set the size in `scaleNode`.
- `THREE.Points` draws one-pixel points on the WebGPU backend; use the instanced `Sprite` (see `threejs-geometry`).
- Trails: `afterImage(scenePassColor, damp)` from `three/addons/tsl/display/AfterImageNode.js`, assigned to a `RenderPipeline`'s `outputNode` (see `threejs-postprocessing`).

## Common Mistakes

- `pos += vel * dt; vel += acc * dt;` → `vel += acc * dt; pos += vel * dt;`: updating position with the old velocity is explicit Euler, which adds energy to every oscillation; the swapped order costs nothing and is stable for `ω·dt < 2`.
- `vel *= 0.98;` → `vel *= Math.exp(-drag * dt);`: a per-frame factor damps twice as fast at 120 Hz as at 60 Hz.
- `step(timer.getDelta())` → `step(Math.min(timer.getDelta(), 1 / 30))`: one long frame becomes one huge step and the simulation blows up.
- `"uniform sampler2D texturePosition;"` written in a compute shader → remove it: `setVariableDependencies` prepends that declaration and a second one fails to compile.
- `material.uniforms.texturePosition.value = gpu.getCurrentRenderTarget(posVar).texture;` once at setup → assign it every frame after `gpu.compute()`: the current target alternates, so a cached reference shows every other frame.
- `references[i * 2] = (i % SIZE) / SIZE;` → `((i % SIZE) + 0.5) / SIZE`: texel centers sit at half-integer offsets; edge coordinates sample between two particles' texels.
- `gpu.init();` → `const error = gpu.init(); if (error !== null) console.error(error);`: init reports failure by returning a string, not by throwing.
- `gpu.init(); gpu.setDataType(THREE.HalfFloatType);` → `gpu.setDataType(THREE.HalfFloatType); gpu.init();`: the targets are created in `init()`, so a later type change has no effect.
- A compute fragment shader ending with `#include <colorspace_fragment>` → write `gl_FragColor` only: the state texture holds data, and an sRGB encode corrupts it.
- `vec3 dir = normalize(target - pos);` → `vec3 d = target - pos; vec3 dir = d / max(length(d), 1e-6);`: normalizing a zero vector is undefined in GLSL and usually gives NaN, and a NaN particle never recovers.

## See Also

- `threejs-math-noise` — hash functions and the curl-noise definition this skill advects with.
- `threejs-math-motion` — closed-form springs and damping for camera and UI motion, in place of integrators.
- `threejs-performance` — measuring cost; the shorter TSL compute particle example.
- `threejs-geometry` — `Points`, `BufferAttribute` updates, instanced `Sprite` under WebGPU.
- `threejs-shaders` — ShaderMaterial rules and the two output includes for the draw shader.
- `threejs-postprocessing` — EffectComposer, `RenderPipeline`, and the pass chain that hosts the afterimage trails.
