# Loaders under WebGPURenderer

Purpose: what changes when the same loaders run with `WebGPURenderer` from `three/webgpu`. Verified against three.js r187 `src/renderers/common/` and `examples/jsm/loaders/KTX2Loader.js`.

The loaders themselves are renderer-agnostic. GLTFLoader still creates `MeshStandardMaterial` / `MeshPhysicalMaterial`; WebGPURenderer renders them through their node-material equivalents on its own, so loaded materials need no conversion.

```javascript
import * as THREE from "three/webgpu";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { KTX2Loader } from "three/addons/loaders/KTX2Loader.js";
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before detectSupport, PMREMGenerator or any other renderer query
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111827); // the WebGPU canvas is transparent by default (alpha: true)
scene.environment = new THREE.PMREMGenerator(renderer).fromScene(new RoomEnvironment(), 0.04).texture;

const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(2, 1.5, 3);
camera.lookAt(0, 0, 0);

const ktx2Loader = new KTX2Loader().detectSupport(renderer); // uses renderer.hasFeature(), which throws before init
const gltf = await new GLTFLoader().setKTX2Loader(ktx2Loader).loadAsync("model.glb");
scene.add(gltf.scene);

renderer.setAnimationLoop(() => renderer.render(scene, camera));
```

## Ordering and API Differences

- `KTX2Loader.detectSupport(renderer)` calls `renderer.hasFeature()`, which throws `.hasFeature() called before the backend is initialized` until `await renderer.init()` resolves. `detectSupportAsync()` is deprecated (r181).
- `PMREMGenerator.fromScene`, `fromEquirectangular` and `fromCubemap` throw before `init()` as well; their `…Async` variants are deprecated (r181).
- Max anisotropy is `renderer.getMaxAnisotropy()`. `renderer.capabilities` exists on WebGLRenderer only.
- `renderer.setAnimationLoop` initialises the renderer itself if needed, but anything that queries the backend before the first frame still needs the explicit `await renderer.init()`.

## Gaussian Splats

`GaussianSplat` (`three/addons/objects/GaussianSplat.js`) imports from `three/webgpu` and uses compute-based sorting, so splat data from `SPLATLoader`, `SPZLoader`, `KSPLATLoader`, `GaussianSplatPLYLoader` or the `GLTFGaussianSplatLoaderExtension` plugin renders only under WebGPURenderer. Code is in `formats.md`.
