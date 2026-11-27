# Performance under WebGPURenderer and TSL

Purpose: reading WebGPURenderer's own counters, the backend-specific costs (BatchedMesh draws, points), and moving per-particle CPU loops into a compute dispatch (TSL names and renderer behaviour checked against the r187 source). Full compute simulations are owned by `threejs-math-particles-fields`.

## Setup and Counters

```javascript
import * as THREE from "three/webgpu";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111111); // the WebGPU canvas is transparent by default
const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 4, 18);

function animate() {
  renderer.render(scene, camera);
  // autoReset clears the per-frame counters at the start of each loop tick
  console.log(`${renderer.info.render.drawCalls} draw calls, ${renderer.info.memory.total} bytes estimated`);
}
renderer.setAnimationLoop(animate);
```

- `info.render.drawCalls` is this frame's draw count. `info.render.calls` counts render calls (each `render()` plus internal ones such as the output pass) since the app started; `info.render.frameCalls` is the per-frame version. The full table is in measuring.md.
- `info.memory.total` is a byte estimate; compressed textures count as 1 byte.
- `renderer.compute()` before `init()` logs a warning; after `await renderer.init()` (or inside `setAnimationLoop`, which awaits it) it submits directly.

## Backend Differences That Cost Time

- `BatchedMesh`: no multi-draw; one draw per run of consecutive instances sharing a geometry (runs merged into instanced draws since r187).
- `Points` are one pixel on the WebGPU backend, so large point clouds are usually an instanced `Sprite` with `PointsNodeMaterial` or `SpriteNodeMaterial` (`sprite.count = n`): one draw for all of them (`threejs-geometry`).
- `ShaderMaterial` is not supported; a fallback NodeMaterial renders instead, so a "fast custom shader" ported from WebGL silently draws something else.

## Compute Instead of a CPU Particle Loop

Compute moves particle work off the CPU: no per-particle JavaScript loop, no buffer re-upload per frame, one dispatch updates every particle:

```javascript
import * as THREE from "three/webgpu";
import { Fn, cos, hash, instanceIndex, instancedArray, sin, time, TWO_PI, vec3 } from "three/tsl";

const count = 100000;
const positionBuffer = instancedArray(count, "vec3"); // a storage buffer living on the GPU
const simulate = Fn(() => {
  const rand = hash(instanceIndex);
  const angle = rand.mul(TWO_PI).add(time); // `time` is a uniform the renderer updates every frame
  positionBuffer.element(instanceIndex).assign(vec3(cos(angle).mul(8), sin(rand.mul(97).add(time.mul(2))).mul(2).add(2), sin(angle).mul(8)));
});
const simulateCompute = simulate().compute(count); // one compute invocation per particle (workgroup size 64 by default)

const material = new THREE.SpriteNodeMaterial();
material.positionNode = positionBuffer.toAttribute(); // the material reads the storage buffer directly
material.colorNode = vec3(1, 0.7, 0.3);
material.scaleNode = vec3(0.05); // sprite size in world units
const particles = new THREE.Sprite(material);
particles.count = count; // the sprite draws once per buffer element
particles.frustumCulled = false; // culling tests one small sphere at the sprite origin, not the particles
scene.add(particles);

function animate() {
  renderer.compute(simulateCompute); // GPU updates the buffer, then the draw reads it
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

- This loop replaces the `animate` of the setup block above.
- `time` and `deltaTime` from `three/tsl` are uniforms the renderer already updates on every render; do not feed a hand-made `uniform()` from a Timer for the same value.
- Reading GPU results back to the CPU (`renderer.getArrayBufferAsync(attribute)`) stalls and should not run every frame.
- When the storage buffer is no longer needed, `positionBuffer.value.dispose()` frees it (buffer-attribute dispose under WebGPURenderer, extended to all attributes in r187).
