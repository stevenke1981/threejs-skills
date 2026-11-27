# Animation under WebGPURenderer and TSL

Purpose: what the mixer does under WebGPURenderer, and GPU-side animation with TSL time nodes, including displacement on skinned meshes. Verified against three.js r187 `src/nodes/utils/Timer.js`, `src/nodes/utils/Oscillators.js` and `src/materials/nodes/NodeMaterial.js`.

Everything in the main skill — mixer, actions, bone transforms, morph weights — runs on the CPU and drives a model under WebGPURenderer unchanged. TSL adds animation evaluated on the GPU, with no per-frame JavaScript.

## Time Nodes and Oscillators

- `time` — elapsed seconds, a float uniform the renderer updates every frame. `deltaTime` — seconds since the previous frame. `frameId` — frame counter (uint).
- `oscSine(t = time)`, `oscSquare`, `oscTriangle`, `oscSawtooth` — 0-to-1 waves with a period of 1 unit of `t`: `oscSine(time.mul(0.5))` takes two seconds per cycle. `oscSine` starts at its minimum at t = 0.

```javascript
import * as THREE from "three/webgpu";
import { positionLocal, time, vec3 } from "three/tsl";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init();
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111418); // the WebGPU canvas is transparent by default

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 2, 3);
camera.lookAt(0, 0, 0);

// Ripple a subdivided plane along its normal (local z) on the GPU
const material = new THREE.MeshBasicNodeMaterial({ wireframe: true });
material.positionNode = positionLocal.add(
  vec3(0, 0, positionLocal.x.mul(2).sin().mul(time.mul(2).cos()).mul(0.2)),
);
const mesh = new THREE.Mesh(new THREE.PlaneGeometry(3, 3, 32, 32), material); // per-vertex motion needs vertices
mesh.rotation.x = -Math.PI / 2;
scene.add(mesh);

renderer.setAnimationLoop(() => renderer.render(scene, camera));
```

## Displacing a Skinned or Morphed Mesh

`NodeMaterial.setupPosition` applies morph targets, skinning, the displacement map, batching and instancing to `positionLocal` first, and only then assigns `positionNode`. Building on `positionLocal` keeps the mixer-driven pose; `positionGeometry` is the raw rest-pose attribute and drops all of it.

```javascript
import * as THREE from "three/webgpu";
import { color, mix, oscSine, positionLocal, vec3 } from "three/tsl";

const material = new THREE.MeshStandardNodeMaterial();
material.positionNode = positionLocal.add(vec3(0, oscSine().mul(0.05), 0)); // rides on top of the skinned pose
material.colorNode = mix(color(0x202030), color(0xffd24a), oscSine()); // oscillators drive any node slot

const skinnedMesh = scene.getObjectByProperty("isSkinnedMesh", true); // a loaded, mixer-driven model
if (skinnedMesh) skinnedMesh.material = material;
```

## Mistakes

- `material.positionNode = positionGeometry.add(offset)` → `positionLocal.add(offset)`: positionGeometry drops skinning, morph targets, batching and instancing.
- `const uTime = uniform(0); uTime.value = timer.getElapsed();` → use `time`: the renderer already advances it.
- Shader-side motion has no memory between frames. Anything with state (springs, damping, accumulated phase under a changing speed) stays on the CPU and reaches the shader through a `uniform` — see `threejs-math-motion`.
- Custom GLSL or larger TSL graphs belong to `threejs-shaders`.
