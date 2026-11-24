---
name: threejs-math-motion
description: "Cinematic motion math for three.js: easing functions, frame-rate independent damping with exponential decay and MathUtils.damp, critically damped springs, quaternion slerp and smooth look-at without gimbal lock, camera rails on CatmullRomCurve3 with arc-length parametrization, dolly and orbit moves, noise-driven camera shake, and timing with Timer. Use when animating cameras or objects procedurally, smoothing a follow or look-at, moving along a path at constant speed, or adding shake."
---

# Three.js Math: Motion

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

Procedural motion on the CPU: time, easing, smoothing, springs, rotation interpolation, path following and shake. Keyframe clips belong to `threejs-animation`, controls to `threejs-interaction`.

## Quick Start

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111318);

const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);

// Scenery: a ring of boxes so the motion reads against something
const ringMaterial = new THREE.MeshNormalMaterial();
for (let i = 0; i < 12; i++) {
  const angle = (i / 12) * Math.PI * 2;
  const box = new THREE.Mesh(new THREE.BoxGeometry(0.6, 0.6, 0.6), ringMaterial);
  box.position.set(Math.cos(angle) * 10, 0.3, Math.sin(angle) * 10);
  scene.add(box);
}

// The subject the camera looks at
const subject = new THREE.Mesh(new THREE.SphereGeometry(0.4, 32, 16), new THREE.MeshNormalMaterial());
scene.add(subject);

// Closed rail around the origin; the default centripetal type avoids loops on uneven points
const railPoints = [
  new THREE.Vector3(-5, 2.0, -5),
  new THREE.Vector3(5, 2.5, -5),
  new THREE.Vector3(6, 3.0, 0),
  new THREE.Vector3(5, 2.2, 5),
  new THREE.Vector3(-5, 2.8, 5),
  new THREE.Vector3(-6, 2.0, 0),
];
const rail = new THREE.CatmullRomCurve3(railPoints, true);
rail.arcLengthDivisions = 1000; // set before the first length query
const railLength = rail.getLength();
const railSpeed = 2; // world units per second
let railU = 0; // fraction of the total length, 0..1

const lookMatrix = new THREE.Matrix4(); // scratch objects, allocated once
const desiredQuaternion = new THREE.Quaternion();

const timer = new THREE.Timer();
timer.connect(document); // delta is 0 while the tab is hidden

function animate() {
  timer.update();
  const dt = Math.min(timer.getDelta(), 1 / 30); // one long frame becomes one small step
  const elapsed = timer.getElapsed();

  // Subject: a figure-eight around the origin
  subject.position.set(
    Math.sin(elapsed * 0.7) * 1.5,
    0.8 + Math.sin(elapsed * 1.3) * 0.25,
    Math.sin(elapsed * 1.4) * 0.8,
  );

  // Camera position: constant speed along the rail, measured in arc length
  railU = (railU + (railSpeed * dt) / railLength) % 1; // closed rail wraps
  rail.getPointAt(railU, camera.position);

  // Camera orientation: target quaternion from look-at, damped toward it
  lookMatrix.lookAt(camera.position, subject.position, camera.up); // eye, target, up
  desiredQuaternion.setFromRotationMatrix(lookMatrix);
  camera.quaternion.slerp(desiredQuaternion, 1 - Math.exp(-6 * dt));

  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);

window.addEventListener("resize", () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

## Time

```javascript
const timer = new THREE.Timer();
timer.connect(document);

function animate() {
  timer.update(); // reads performance.now(); see the note on timestamps below
  const dt = Math.min(timer.getDelta(), 1 / 30);
  const now = timer.getElapsed();
  // ...
}
renderer.setAnimationLoop(animate);
```

- `getDelta()` and `getElapsed()` return seconds. Both are fixed for the frame by `update()`, so calling them several times per frame gives the same value.
- `update(timestamp)` measures from `performance.now()` at `new THREE.Timer()`. The `setAnimationLoop` timestamp is the frame's start time, which can be earlier than the construction when slow setup code (building geometry, baking textures) runs before the first frame: the first delta and `getElapsed()` are then negative. Checked in Chrome: a 0.8 s terrain build gave a first delta of -0.2 s, and a rail fraction driven by it went below 0 and threw inside `getPointAt`. Call `update()` without an argument.
- Without `connect(document)` the first delta after a tab switch can be many seconds. With it, the delta is 0 while the page is hidden and the timer resets when it returns.
- Clamp `dt` in anything that integrates: a 2-second stall fed to a spring or a rail speed teleports the object.
- `timer.setTimescale(0.25)` scales both delta and elapsed (slow motion). Pause by setting it to 0. `getTimescale()` reads it back.
- `THREE.Clock` is deprecated since r183. Use `Timer`.
- Normalized time for a fixed-length move: `const t = THREE.MathUtils.clamp((now - start) / duration, 0, 1);`. Store `start = timer.getElapsed()` when the move begins, not at page load.
- For timeline tweens with several properties, `three/addons/libs/tween.module.js` ships in the package (tween.js). This skill does not document its API.

## Easing

```javascript
const easeInQuad = (t) => t * t;
const easeOutQuad = (t) => 1 - (1 - t) * (1 - t);
const easeInOutCubic = (t) => (t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2);
const easeOutExpo = (t) => (t === 1 ? 1 : 1 - Math.pow(2, -10 * t));
const easeOutBack = (t) => 1 + 2.70158 * Math.pow(t - 1, 3) + 1.70158 * Math.pow(t - 1, 2);
const easeOutElastic = (t) =>
  t === 0 ? 0 : t === 1 ? 1 : Math.pow(2, -10 * t) * Math.sin((t * 10 - 0.75) * ((2 * Math.PI) / 3)) + 1;
const smoothstep = (t) => t * t * (3 - 2 * t);
const smootherstep = (t) => t * t * t * (t * (t * 6 - 15) + 10);
```

- Clamp `t` to [0, 1] before easing. `easeInOutCubic(1.2)` extrapolates past the end.
- `smoothstep` has zero velocity at both ends. `smootherstep` also has zero acceleration, so there is no visible jerk at start and stop. Camera moves usually want ease-in-out; UI pops want ease-out.
- `easeOutExpo` reaches only 0.999 at t = 1 by formula, hence the special case. Without it the end pose snaps.
- Back and elastic leave [0, 1]. Fine for position and scale; clamp them before using them for opacity or a color mix.
- Ease the interpolation parameter, then interpolate: `position.lerpVectors(a, b, easeInOutCubic(t))`. For rotations, `quaternion.slerpQuaternions(qa, qb, easeInOutCubic(t))`.
- `THREE.MathUtils.smoothstep(x, min, max)` takes the value first, unlike GLSL.

## Frame-Rate Independent Smoothing

Checked in Node: `MathUtils.damp` after 1 s of simulated time gives the same value at 30 and 144 steps per second (0.993262 for λ = 5, the exact value of `1 − e^(−5)`).

```javascript
// Exact per-step blend: remaining distance decays as e^(−λ·t)
position.lerp(target, 1 - Math.exp(-lambda * dt));
THREE.MathUtils.damp(x, target, lambda, dt); // the same for a scalar: x = lerp(x, target, 1 − e^(−λ·dt))
quaternion.slerp(targetQuaternion, 1 - Math.exp(-lambda * dt));
color.lerp(targetColor, 1 - Math.exp(-lambda * dt));
```

- A per-frame `x += (target − x) * 0.1` is the same formula with `λ = −ln(0.9) · fps`. The smoothing depends on the frame rate: after one second it covers 99.8% of the distance at 60 fps and 95.8% at 30 fps.
- Converting an old per-frame factor `f` tuned at 60 fps: `lambda = -Math.log(1 - f) * 60`.
- `lambda` is a rate per second, not a blend factor. Half-life is `ln 2 / λ`: λ = 5 gives 0.139 s. The remaining distance is under 1% after `4.6 / λ` seconds.
- Against a target that moves at constant speed `v`, the steady-state lag is about `v / λ`. Raise λ to tighten the follow, and accept the jitter that comes with it.
- Exponential damping starts at full speed, so a jump in the target produces a velocity discontinuity. For a soft start, use the spring below.
- `MathUtils.damp` has no velocity state. It is a first-order decay, not a spring.

## Springs

Critically damped spring, closed-form step. Checked against the analytic solution, and the same at 10, 30, 60 and 144 steps per second.

```javascript
// state.x, state.v: current value and velocity. omega: angular frequency in rad/s (higher = faster).
function springStep(state, target, omega, dt) {
  const d = state.x - target;
  const e = Math.exp(-omega * dt);
  const t = (state.v + omega * d) * dt;
  state.x = target + (d + t) * e;
  state.v = (state.v - omega * t) * e;
}

const spring = { x: 0, v: 0 };
function animate() {
  timer.update();
  const dt = Math.min(timer.getDelta(), 1 / 30);
  springStep(spring, targetValue, 8, dt); // targetValue: set anywhere before this call
  mesh.position.x = spring.x;
}
```

- It is the exact solution of `x″ = −ω²(x − target) − 2ω·x′` over one step, so it is stable for any `dt` and gives the same trajectory at any frame rate while the target is constant.
- No overshoot from rest, zero velocity at the start, and it keeps its momentum when the target changes.
- Settles to within 1% in about `6.6 / ω` seconds (measured 0.667 s for ω = 10). With `ω = 2π · frequency` for a frequency in hertz.
- For a `Vector3`, keep a velocity `Vector3` and run `springStep` on each component, or write the same three lines with `Vector3` operations.

Under-damped bounce (damping ratio ζ < 1) has a different closed form (a decaying sine), not covered here. A shorter route is to integrate with semi-implicit Euler at a fixed step, using an accumulator so the step does not depend on the frame:

```javascript
const FIXED_STEP = 1 / 240; // seconds; omega * FIXED_STEP stays well below 1

function bounceUpdate(state, target, omega, zeta, dt) {
  state.acc += dt;
  while (state.acc >= FIXED_STEP) {
    state.acc -= FIXED_STEP;
    const a = -omega * omega * (state.x - target) - 2 * zeta * omega * state.v;
    state.v += a * FIXED_STEP;
    state.x += state.v * FIXED_STEP;
  }
}
// state = { x: 0, v: 0, acc: 0 }; call once per frame with the clamped dt
```

- Checked in Node: positions after 0.2 s agree to 1e-5 at 30, 60, 144 and 1000 steps per second. Recomputing the substep as `dt / n` per frame does not: the result drifts with the frame rate because Euler's error depends on the step size.
- The leftover `state.acc` means the rendered state trails the simulated time by less than 1/240 s.
- Stiffness: keep `omega * FIXED_STEP` well below 1. Semi-implicit Euler stays stable far longer than explicit Euler, but accuracy drops as the step grows.
- Integrators for particle systems and fields (Verlet, RK4, stability limits) belong to `threejs-math-particles-fields`.

## Rotations

- Interpolating Euler angles is not rotation interpolation. The three axes apply in order, so the path wobbles, and when the middle angle reaches ±90° two axes align (gimbal lock). Interpolate `quaternion` for any arbitrary orientation.
- `object.rotation` and `object.quaternion` stay in sync: writing one updates the other.
- `q.slerp(target, t)` takes the shorter arc. It negates the target internally when the dot product is negative, so do not negate quaternions yourself. Checked: a slerp from +3 rad to −3 rad about Y at t = 0.5 passes through π, the short way round.
- Constant angular speed: `q.rotateTowards(target, radiansPerSecond * dt)`. It never overshoots: a step larger than the remaining angle lands on the target (checked).
- A single angle (a turret's yaw) must wrap: take the shortest signed difference and damp toward it.

```javascript
// shortest signed difference from angle a to angle b, in [-π, π)
const angleDelta = (a, b) => THREE.MathUtils.euclideanModulo(b - a + Math.PI, Math.PI * 2) - Math.PI;

yaw += angleDelta(yaw, desiredYaw) * (1 - Math.exp(-lambda * dt)); // yaw stays unwrapped and continuous
```

- `THREE.MathUtils.damp(a, b, …)` on angles spins the long way across the ±π seam.
- `q.multiply(r)` applies `r` in the object's local frame. `q.premultiply(r)` applies it in the parent (world) frame.
- Accumulating many incremental multiplies drifts from unit length. Call `q.normalize()` once per frame when you do.

## Smooth Look-At

```javascript
const lookMatrix = new THREE.Matrix4();
const targetQuaternion = new THREE.Quaternion();

function smoothLookAt(camera, target, lambda, dt) {
  lookMatrix.lookAt(camera.position, target, camera.up); // eye, target, up: a camera looks down its -Z
  targetQuaternion.setFromRotationMatrix(lookMatrix);
  camera.quaternion.slerp(targetQuaternion, 1 - Math.exp(-lambda * dt));
}
```

- Checked: after the call the camera's −Z axis points at the target (dot product 1.000000 with the direction to the target).
- A non-camera object swaps the first two arguments: `lookMatrix.lookAt(target, object.position, object.up)`. Meshes face +Z, so this is the order `Object3D.lookAt` uses internally for non-cameras.
- The result is in local space. It is valid when the object has no rotated parent, because `position` here is world space. With a parent, convert the result as `Object3D.lookAt` does, or keep the camera at the scene root.
- `Object3D.lookAt(target)` sets the quaternion instantly. It has no smoothing and does not support parents with non-uniform scale.
- Singularity: when the view direction approaches `camera.up`, the roll is undefined and the camera can spin as the target passes overhead. Limit the pitch, pick another `camera.up` for top-down shots (for example `camera.up.set(0, 0, -1)`), or carry an up vector along a transported frame.
- The look-at target quaternion is orientation, not Euler. This does not avoid the up-vector singularity, which is a separate issue from gimbal lock in interpolation.

## Camera Rails

```javascript
const rail = new THREE.CatmullRomCurve3(points, true); // closed loop, centripetal
rail.arcLengthDivisions = 1000; // default 200: finer arc-length table for long or winding paths
const railLength = rail.getLength();
const speed = 2; // world units per second
let u = 0;

function moveAlongRail(dt) {
  u = (u + (speed * dt) / railLength) % 1; // closed path wraps; an open path clamps to [0, 1]
  rail.getPointAt(u, camera.position);
}
```

- `getPoint(t)` moves by curve parameter: the camera speeds up on long segments and crawls on short ones. Checked on an uneven rail: consecutive `getPoint` steps varied by about 60×, consecutive `getPointAt` steps by 1.01×.
- `getPointAt(u)` takes a fraction of arc length, and its accuracy is that of the arc-length table: piecewise linear over `arcLengthDivisions` samples. Set the divisions before the first length query, or call `updateArcLengths()` after changing them.
- The table is cached. After moving control points, call `rail.updateArcLengths()`. Checked: `getLength()` kept the old value after a point moved, and changed on the next `updateArcLengths()`.
- Position at a distance in world units: `rail.getPoint(rail.getUtoTmapping(0, distance))`. `getUtoTmapping` returns a curve parameter, not a fraction. A `distance` of 0 is falsy, so the method falls back to `u`; with `u = 0` that gives the start.
- Ease the whole move by easing the fraction: `rail.getPointAt(easeInOutCubic(t))`.
- Look direction: `rail.getTangentAt(u)`, or a look-ahead point `rail.getPointAt((u + 0.01) % 1)` fed to `smoothLookAt`. The exact tangent passes every wiggle of the path to the camera. A look-ahead point with damping is steadier.
- The default centripetal type avoids the loops and cusps that `"catmullrom"` produces on unevenly spaced points. Keep the default for camera paths.
- A Catmull-Rom spline is C1: velocity is continuous and acceleration is not. A sensitive viewer sees a change in turn rate at control points. Fewer, well-spaced control points look smoother than many.
- Loops and rolls (roller coasters): a fixed `up` flips when the tangent passes vertical. Use `rail.computeFrenetFrames(segments, closed)` and build the orientation from the tangent, normal and binormal. Details are in `threejs-math-procedural-geometry`.
- A moving path (a train, a boat) is a new curve each time its control points change: call `updateArcLengths()` after every move, or the speed drifts.

## Orbit, Dolly and Dolly Zoom

Orbit around a target with spherical coordinates. `phi` is the polar angle from +Y, `theta` the azimuth around Y from +Z.

```javascript
const orbit = new THREE.Spherical(8, Math.PI / 3, 0).makeSafe(); // makeSafe clamps phi to [1e-6, π − 1e-6]
const orbitTarget = new THREE.Vector3(0, 1, 0);

function updateOrbit(dt) {
  orbit.theta += 0.4 * dt; // radians per second
  camera.position.setFromSphericalCoords(orbit.radius, orbit.phi, orbit.theta).add(orbitTarget);
  camera.lookAt(orbitTarget);
}
```

- Keep `phi` away from 0 and π, or the azimuth is undefined at the poles. `makeSafe()` clamps only to 1e-6 from each pole; for a camera that must never flip over the top, clamp to a wider range such as [0.05, π − 0.05].
- `Vector3.setFromSphericalCoords(radius, phi, theta)` and `setFromSpherical(s)` write the position; `add(target)` places the orbit center.
- Dolly zoom (Hitchcock): keep the framed size at the subject constant while the field of view changes. Visible height at distance `d` is `2 · d · tan(fov / 2)`, so for a framed height `h`:

```javascript
// camera.fov is the vertical field of view in degrees
const fovRad = THREE.MathUtils.degToRad(camera.fov);
const distance = frameHeight / (2 * Math.tan(fovRad / 2));
camera.position.copy(subject.position).add(new THREE.Vector3(0, 0, distance)); // along +Z from the subject
camera.updateProjectionMatrix(); // after every change of fov
```

- Check: `frameHeight = 2`, `fov = 50` gives a visible height of 2.0000 at that distance.
- For a large zoom, interpolate in focal-length space and convert back: `camera.setFocalLength(mm)` sets fov from the film gauge (35 mm by default) and calls `updateProjectionMatrix()` itself. The film height is `filmGauge / Math.max(aspect, 1)`, so the result depends on the aspect. Checked: `setFocalLength(50)` gives fov 38.580 at aspect 1 and 22.275 at 16:9.

## Camera Shake

Trauma model with time-sampled noise. Rotation is the default: it reads as shake at any scene scale.

```javascript
import { SimplexNoise } from "three/addons/math/SimplexNoise.js";

const noise = new SimplexNoise();
const maxAngle = THREE.MathUtils.degToRad(2);
const shakeEuler = new THREE.Euler();
const shakeQuaternion = new THREE.Quaternion();
let trauma = 0; // add up to 1 on impacts: trauma = Math.min(trauma + 0.5, 1)

function applyShake(camera, elapsed, dt) {
  trauma = Math.max(trauma - 0.8 * dt, 0); // linear decay per second
  const shake = trauma * trauma; // quadratic response: small trauma is subtle
  const f = elapsed * 12; // shake frequency in noise units
  shakeEuler.set(
    maxAngle * shake * noise.noise(f, 0),
    maxAngle * shake * noise.noise(f, 10),
    maxAngle * shake * noise.noise(f, 20),
  );
  camera.quaternion.multiply(shakeQuaternion.setFromEuler(shakeEuler)); // on top of this frame's base orientation
}
```

- Call it after the base pose is written this frame (rail position, look-at). The base must be recomputed every frame. If nothing else sets `camera.quaternion`, the shake multiplies into a random walk.
- An alternative is a rig: make the camera a child of an empty object, move and aim the rig from the base pose, and shake only the camera's local transform. The rig never accumulates.
- Sample noise by time, not `Math.random()` per frame. Random values change at the frame rate (different feel at 30, 60 and 144 Hz, and visible strobing), and they have no low-frequency content.
- The offsets 0, 10 and 20 keep the three axes uncorrelated. `SimplexNoise` here is 2D: `noise(x, y)` returns values in [-1, 1].
- Handheld drift: the same code with a small constant `shake` (for example 0.3 of full), a frequency of 0.3–1 Hz, and two octaves of noise added.
- Translational shake needs tuning to the scene's units and can push the camera through walls. Use it only after checking the path.
- The Euler here is safe: the angles are a few degrees, far from gimbal lock.

## Working with Controls

- OrbitControls owns the camera transform. `controls.update()` overwrites position and orientation on every call, so a scripted move and active controls fight, visible as jitter or snapping. See `threejs-interaction` for the controls themselves.
- During a scripted shot set `controls.enabled = false` and skip `controls.update()`. When handing back, copy the look point into `controls.target`, then call `controls.update()` once.
- To animate with controls active, animate `controls.target` and let the controls place the camera.
- OrbitControls' own damping is outside this skill; do not assume it is frame-rate independent.

## WebGPU and TSL

- Everything above is JavaScript on `Object3D` and the curve classes, and runs the same under `WebGPURenderer` (`import * as THREE from "three/webgpu"`; `Timer`, `MathUtils`, `Quaternion`, `Matrix4`, curves are all re-exported there).
- Motion evaluated in the shader (per vertex or per instance) uses the `time` node, which the renderer advances itself; it needs no Timer uniform:

```javascript
import * as THREE from "three/webgpu";
import { positionLocal, oscSine, time, vec3 } from "three/tsl";

const material = new THREE.MeshNormalNodeMaterial();
// bob every vertex up and down by 0.2 units, once every two seconds
material.positionNode = positionLocal.add(vec3(0, oscSine(time.mul(0.5)).mul(0.2), 0));
```

- `oscSine`, `oscTriangle`, `oscSquare` and `oscSawtooth` return 0..1 and repeat every 1 unit of their argument (`oscSine(t = time)`).
- An easing as a TSL function, then a blend with `mix`:

```javascript
import * as THREE from "three/webgpu";
import { Fn, mix, select, time, color, oscSine } from "three/tsl";

const easeInOutCubic = Fn(([t]) =>
  select(t.lessThan(0.5), t.pow(3).mul(4), t.mul(-2).add(2).pow(3).div(2).oneMinus()),
);

const material = new THREE.MeshBasicNodeMaterial();
material.colorNode = mix(color(0x202030), color(0xffd24a), easeInOutCubic(oscSine(time.mul(0.5))));
```

- Shader-side motion is stateless. Springs and damping keep state, and a material graph has no memory between frames. Keep the state on the CPU and pass the result through a `uniform`, or in a compute buffer (`threejs-math-particles-fields`).
- The camera's own transform is CPU state even under WebGPU: scripted camera moves and controls work the same.

## Common Mistakes

- `camera.position.lerp(target, 0.1)` every frame → `camera.position.lerp(target, 1 - Math.exp(-lambda * dt))`: a constant factor per frame converges faster at higher frame rates.
- `THREE.MathUtils.damp(x, target, 0.1, dt)` → `THREE.MathUtils.damp(x, target, 6, dt)`: lambda is a rate per second, not a 0–1 blend; 0.1 takes about 7 s to cover half the distance.
- `const clock = new THREE.Clock(); const dt = clock.getDelta();` → `timer.update(); const dt = timer.getDelta();`: Clock is deprecated since r183, and Timer's delta is stable within a frame and zero while a connected page is hidden.
- `const dt = timer.getDelta();` fed to springs or rails → `const dt = Math.min(timer.getDelta(), 1 / 30);`: one long frame becomes one long step.
- `THREE.MathUtils.smoothstep(0, 1, t)` → `THREE.MathUtils.smoothstep(t, 0, 1)`: the value is the first argument, unlike GLSL.
- `mesh.rotation.y = THREE.MathUtils.lerp(a, b, t)` across the ±π seam → interpolate the shortest difference, or slerp quaternions: a plain lerp turns the long way round.
- `object.rotation.set(lerp(ax, bx, t), lerp(ay, by, t), lerp(az, bz, t))` → `object.quaternion.slerpQuaternions(qa, qb, t)`: interpolated Euler angles wobble and lock when the middle axis reaches 90°.
- `camera.lookAt(target)` each frame for a smooth follow → `camera.quaternion.slerp(targetQuaternion, 1 - Math.exp(-lambda * dt))` with the target built by `Matrix4.lookAt`: `lookAt` snaps instantly.
- `camera.position.copy(curve.getPoint(t))` → `curve.getPointAt(u, camera.position)`: `getPoint` is uniform in the curve parameter, so speed follows the control-point spacing.
- Moving `curve.points[i]` and then calling `getPointAt` → call `curve.updateArcLengths()` first: the arc-length table is cached.
- `rail.getPoint(rail.getUtoTmapping(0.5, 0))` meaning the start → `rail.getPoint(rail.getUtoTmapping(0, 0))`: a distance of 0 is falsy, so the method falls back to `u`, and 0.5 is the midpoint.
- `camera.fov = 30;` → `camera.fov = 30; camera.updateProjectionMatrix();`: the projection matrix is rebuilt only on request.
- `camera.position.x += (Math.random() - 0.5) * strength` for shake → time-sampled noise applied on top of a base pose recomputed every frame: per-frame random offsets depend on the frame rate and accumulate.
- Animating the camera while `controls.update()` runs → `controls.enabled = false` and skip `controls.update()` during the shot: OrbitControls rewrites the camera transform on each update.
- `timer.update(timestamp)` after slow setup code → `timer.update()`: the first frame's rAF timestamp can predate the timer's construction, giving a negative delta that runs every integrator and path fraction backwards.

## See Also

- `threejs-fundamentals` — `Timer`, the animation loop, core math classes, camera setup.
- `threejs-animation` — keyframe clips, the mixer and crossfades; use this skill for procedural motion, not clips.
- `threejs-interaction` — OrbitControls and other controls that compete with scripted cameras.
- `threejs-math-noise` — the noise functions behind shake and drift; curl flow for moving fields.
- `threejs-math-procedural-geometry` — curve frames for tubes and sweeps, and curves as geometry.
- `threejs-math-particles-fields` — integrators (Verlet, RK4) and GPU state for many moving objects.
