# WebGPURenderer and TSL Setup

Purpose: creating a scene on WebGPURenderer (r187), its WebGL 2 fallback, node materials, and the API differences from WebGLRenderer. Custom TSL code lives in `threejs-shaders`; node post-processing in `threejs-postprocessing`.

## Minimal scene

```javascript
import * as THREE from "three/webgpu";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // must complete before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111318); // the WebGPU canvas is transparent by default (alpha: true)

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 1.5, 5);

const cube = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardNodeMaterial({ color: 0x3a86ff }),
);
scene.add(cube);
const light = new THREE.DirectionalLight(0xffffff, 2);
light.position.set(3, 4, 5);
scene.add(light);
scene.add(new THREE.AmbientLight(0xffffff, 0.4));

const timer = new THREE.Timer();
timer.connect(document);

function animate() {
  timer.update();
  cube.rotation.y += timer.getDelta();
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

Use the four-key import map from SKILL.md when there is no bundler.

## Initialization and backends

- `renderer.render()` throws `.render() called before the backend is initialized` until `await renderer.init()` has resolved. `setAnimationLoop()` awaits `init()` itself, but await it right after construction anyway: on fallback, `init()` replaces `renderer.backend`, so setup code that ran earlier talked to the wrong backend.
- WebGPURenderer falls back to WebGL 2 by itself when WebGPU is unavailable. `new THREE.WebGPURenderer({ forceWebGL: true })` forces the fallback; after `init()`, `renderer.backend.isWebGPUBackend` tells which backend runs.
- `hasFeature()` throws before `init()`; `renderAsync()`, `hasFeatureAsync()`, `clearAsync()` and `initTextureAsync()` are deprecated since r181 in favour of `init()` plus the sync methods.
- r187: `render()` starts the internal animation loop on first use, and `setAnimationLoop(null)` no longer starts it.

## Materials on each renderer

- Node materials and TSL target WebGPURenderer (either backend). Built-in materials such as `MeshStandardMaterial` are converted to their node versions automatically; use the `*NodeMaterial` classes when you set node slots (`colorNode`, `positionNode`, ...).
- `ShaderMaterial`, `RawShaderMaterial` and `onBeforeCompile` are WebGLRenderer-only; port them to TSL (`threejs-shaders`).
- WebGLRenderer renders node materials only through the `WebGLNodesHandler` addon (`three/addons/tsl/WebGLNodesHandler.js`, passed to `renderer.setNodesHandler()`). It imports from `three/webgpu`, so the import map needs that key. Its source lists the limits: no VSM shadows, MRT, transmission, WebGPU post-processing stack or storage textures; fog and environment changes need a `dispose`; instanced geometry cannot be shared. Treat it as a migration aid.

## API differences from WebGLRenderer

- There is no `renderer.capabilities` object; anisotropy comes from `renderer.getMaxAnisotropy()`.
- GPU readback is asynchronous: `await renderer.getArrayBufferAsync(attribute)`.
- Cube render targets: `CubeRenderTarget` from `three/webgpu`, not `WebGLCubeRenderTarget` (r183).
- `PCFSoftShadowMap` was removed in r186 and falls back to `PCFShadowMap` with a warning.
- ArrayCamera viewports are CSS pixels (the renderer multiplies by the pixel ratio); see `cameras.md`.
- `renderer.dispose()` is asynchronous on WebGPURenderer; await it before reusing the canvas.
