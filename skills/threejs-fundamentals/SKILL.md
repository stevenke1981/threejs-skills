---
name: threejs-fundamentals
description: "Three.js r187 scene setup: scene and fog, cameras, WebGLRenderer or WebGPURenderer creation, tone mapping, the render loop with Timer, resize, Object3D hierarchy and transforms, core math, import maps and cleanup. Use when starting a scene, a black or stretched canvas, resize, the Clock deprecation warning, or WebGL vs WebGPU. Not for models (use threejs-loaders), lights or shadows (use threejs-lighting), materials (use threejs-materials), speed (use threejs-performance), React (use threejs-react-three-fiber). 中文: 場景建立, 攝影機, 渲染器, 動畫迴圈, 視窗縮放, 座標系, 資源釋放"
---

# Three.js Fundamentals

Targets three.js r187 (`three@0.187.0`). Examples use WebGLRenderer; WebGPURenderer and TSL setup is in `references/webgpu-and-tsl.md`. This is the entry skill: the See Also list at the end points to every other three.js skill.

## Quick Start

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // cap the fill rate on high-DPI displays
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111318);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 1.5, 5);

const cube = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0x3a86ff }),
);
scene.add(cube);
const light = new THREE.DirectionalLight(0xffffff, 2);
light.position.set(3, 4, 5); // shines from its position toward the origin
scene.add(light);
scene.add(new THREE.AmbientLight(0xffffff, 0.4));

const timer = new THREE.Timer();
timer.connect(document); // freezes the delta while the tab is hidden

function animate() {
  timer.update(); // advance the timer first thing each frame
  cube.rotation.y += timer.getDelta(); // seconds since the previous frame
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);

window.addEventListener("resize", () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

## Import Maps (no bundler)

The browser loads three.js as ES modules; `build/three.js` and `three.min.js` were removed in r161. Map for WebGLRenderer code:

```html
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.187.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.187.0/examples/jsm/"
  }
}
</script>
```

Map for WebGPURenderer code, the same four keys the official WebGPU examples use:

```html
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.187.0/build/three.webgpu.js",
    "three/webgpu": "https://cdn.jsdelivr.net/npm/three@0.187.0/build/three.webgpu.js",
    "three/tsl": "https://cdn.jsdelivr.net/npm/three@0.187.0/build/three.tsl.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.187.0/examples/jsm/"
  }
}
</script>
```

- In r187 `three.tsl.js` imports `./three.core.js` directly (up to r186 it imported the bare `three/webgpu` specifier). Keep the `three/webgpu` key anyway: app code imports `three/webgpu`, and so do WebGPU-era addons such as `three/addons/tsl/WebGLNodesHandler.js`, which also need that key under a WebGLRenderer map.
- Pin every URL to the same version; a second copy of the core logs `WARNING: Multiple instances of Three.js being imported.`
- Write `three/addons/...`: the package `exports` map it for bundlers, and the import maps above define it. `three/examples/jsm/...` also resolves under a bundler but not under these import maps.

## Choosing a Renderer

- **WebGLRenderer** (`import * as THREE from "three"`): the default. `ShaderMaterial`, `onBeforeCompile`, `EffectComposer` and the classic (non-TSL) addons work here.
- **WebGPURenderer** (`import * as THREE from "three/webgpu"`): needed for TSL node materials, compute, and the node post-processing pipeline. It falls back to WebGL 2 by itself, so it is not "WebGPU-only". Call `await renderer.init()` before anything touches the renderer. Details: `references/webgpu-and-tsl.md`.
- Do not mix the two stacks in one scene: GLSL `ShaderMaterial` does not run on WebGPURenderer, and node materials on WebGLRenderer need the `WebGLNodesHandler` migration addon, which has a reduced feature set.

## Scene and Renderer Essentials

```javascript
scene.fog = new THREE.Fog(0x111318, 1, 100); // linear: near, far (or FogExp2(color, density))
scene.environmentIntensity = 1; // scales scene.environment; material.envMapIntensity scales only material.envMap
renderer.toneMapping = THREE.ACESFilmicToneMapping; // or AgXToneMapping, NeutralToneMapping; NoToneMapping is the default
renderer.toneMappingExposure = 1.0;
// renderer.outputColorSpace is SRGBColorSpace by default; leave it alone
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap; // the default, already soft
```

- Plain three.js has tone mapping off; React Three Fiber turns ACES on by default (see `threejs-react-three-fiber`). The exposure and banding maths belong to `threejs-math-atmosphere`.
- `antialias`, `alpha`, `stencil` (off by default), `powerPreference` and `preserveDrawingBuffer` are constructor-only options. Full option list, fog, backgrounds, clear colour and `renderer.info`: `references/webgl-renderer.md`.
- r187: `setViewport()` and `setScissor()` no longer multiply by the pixel ratio while a render target is bound; they still do for the canvas.

## Cameras in Brief

- After changing `fov`, `aspect`, `near`, `far`, `zoom` or an orthographic frustum, call `camera.updateProjectionMatrix()`. Resize handlers that skip it produce a stretched image.
- Keep `near` as large as the scene allows; a tiny near plane with a huge far plane causes z-fighting.
- Orthographic: size the frustum in world units and recompute left/right from the aspect on resize.
- ArrayCamera (split screen, multi-view) and CubeCamera (dynamic reflections) have pixel-unit and self-reflection traps: `references/cameras.md`.

## Object3D in Brief

- `add()` keeps the child's local transform; `attach()` keeps its world transform.
- World-space getters (`getWorldPosition`, `getWorldQuaternion`, `getWorldDirection`) write into a target you pass.
- `layers.enable(n)` adds a layer; `layers.set(n)` replaces the mask and drops layer 0, where the default camera looks.
- Frustum culling tests bounding spheres that are computed once; after editing vertex positions call `geometry.computeBoundingSphere()`.
- Right-handed, +Y up, a default camera looks down −Z. `new THREE.AxesHelper(5)` shows red X, green Y, blue Z.
- API tables for Object3D, Vector3, Matrix4, Quaternion, Euler, Color, MathUtils and `pivot`: `references/object3d-and-math.md`.

## Cleanup

```javascript
scene.remove(mesh);
// dispose geometry, materials and textures only when nothing else still uses them
mesh.geometry.dispose();
const materials = Array.isArray(mesh.material) ? mesh.material : [mesh.material];
for (const material of materials) {
  for (const value of Object.values(material)) {
    if (value?.isTexture) value.dispose(); // material.dispose() leaves its textures alone
  }
  material.dispose();
}
mesh.dispose(); // frees renderer state kept for this object only (Object3D.dispose, r186+)

timer.dispose(); // disconnects the visibility listener
renderer.setAnimationLoop(null); // stop the loop
renderer.dispose(); // frees the renderer's GPU resources; the WebGL context stays alive
renderer.forceContextLoss(); // also release the context when the canvas is discarded
```

`Object3D.dispose()` does not visit children or free geometry, materials or textures; `traverse` a subtree and dispose each mesh as above. A custom Object3D subclass that overrides `dispose()` must call `super.dispose()`.

## Common Mistakes

- `const clock = new THREE.Clock()` → `const timer = new THREE.Timer(); timer.connect(document);`: Clock is deprecated since r183 and warns on construction; call `timer.update()` once per frame before `getDelta()` or `getElapsed()`.
- `requestAnimationFrame(animate)` → `renderer.setAnimationLoop(animate)`: both renderers' docs advise it, WebXR frames run only through it, and on WebGPURenderer it performs the async init that `render()` requires.
- `renderer.shadowMap.type = THREE.PCFSoftShadowMap` → `THREE.PCFShadowMap`: deprecated for WebGLRenderer in r182 and removed for WebGPURenderer in r186; both now warn and fall back to PCFShadowMap, which is already soft.
- `await renderer.renderAsync(scene, camera)` → `await renderer.init(); renderer.render(scene, camera)`: the async render methods are deprecated since r181.
- Resize without `camera.updateProjectionMatrix()` → always call it after changing `aspect`: the old projection stays and the image stretches.
- `mesh.layers.set(1)` → `mesh.layers.enable(1)`: `set` takes the mesh off layer 0, where the default camera and lights live.
- `scene.background = texture` → `texture.mapping = THREE.EquirectangularReflectionMapping; scene.background = texture`: without an equirect or cube mapping the texture is drawn as a flat screen-filling quad.
- `material.envMapIntensity = 0.5` → `scene.environmentIntensity = 0.5`: `envMapIntensity` scales only the material's own `envMap`, not `scene.environment`.
- `renderer.outputEncoding = THREE.sRGBEncoding` → delete the line, or `renderer.outputColorSpace = THREE.SRGBColorSpace`: the encoding API no longer exists and sRGB output is already the default.
- `renderer.physicallyCorrectLights = true` → delete the line: physical light units are always on; give point and spot lights candela intensities (`threejs-lighting`).
- `mesh.dispose()` alone → also dispose geometry, material and textures (Cleanup above): the object-level dispose frees only per-object renderer state.
- `subCamera.viewport = new THREE.Vector4(0, 0, 0.5, 0.5)` → pixel values: ArrayCamera viewports are pixels, not fractions (`references/cameras.md`).
- `cubeCamera.update(renderer, scene)` with the reflective mesh visible → hide it around the update: otherwise it is rendered into the very cube map its material samples.
- `import { WebGLCubeRenderTarget } from "three/webgpu"` → `import { CubeRenderTarget } from "three/webgpu"`: WebGLCubeRenderTarget cannot be used with WebGPURenderer since r183.
- `import ... from "three/examples/jsm/..."` → `"three/addons/..."`: the documented import maps define only `three/addons/`.
- `<script src="three.min.js">` → `<script type="module">` with an import map: the UMD builds were removed in r161.

## Reference Files

- `references/webgl-renderer.md` — WebGLRenderer constructor options, viewport and pixel ratio, background, fog and `scene.environment`, clear colour, `renderer.info` counters. Open when configuring a renderer or scene beyond the Quick Start.
- `references/webgpu-and-tsl.md` — WebGPURenderer setup, `init()`, the WebGL 2 fallback, node materials, `WebGLNodesHandler`, API differences from WebGLRenderer. Open for any `three/webgpu` or TSL scene.
- `references/cameras.md` — PerspectiveCamera, OrthographicCamera, ArrayCamera (r187 culling, viewport units) and CubeCamera code. Open for split screen, orthographic views or dynamic reflections.
- `references/object3d-and-math.md` — Object3D, Group and multi-material Mesh, matrix updates, `pivot`, bounding-sphere culling, and the Vector3, Matrix4, Quaternion, Euler, Color and MathUtils cheat sheets. Open when transforming objects or doing vector maths.

## See Also

- `threejs-geometry` — built-in geometries, BufferGeometry, merging, InstancedMesh and BatchedMesh, points and lines.
- `threejs-math-procedural-geometry` — meshes generated from equations: parametric surfaces, sweeps, lofts, L-systems, marching cubes.
- `threejs-materials` — choosing and configuring built-in and node materials.
- `threejs-shaders` — custom GLSL (ShaderMaterial, onBeforeCompile) and custom TSL node logic.
- `threejs-textures` — texture objects, colour space, wrap and filter, render targets, texture memory.
- `threejs-loaders` — GLTFLoader and other loaders, LoadingManager and progress, HDR environments.
- `threejs-asset-pipeline` — offline glTF optimisation with gltf-transform and gltfjsx.
- `threejs-lighting` — light types and units, shadows, environment lighting and light probes.
- `threejs-postprocessing` — EffectComposer passes and the WebGPU post-processing pipeline.
- `threejs-math-atmosphere` — sky, scattering, fog volumes, god rays, tone-mapping and banding maths.
- `threejs-animation` — AnimationMixer, clips, actions, skeletal and morph animation.
- `threejs-math-motion` — easing, damping, springs, slerp, camera rails and shake.
- `threejs-interaction` — raycasting, camera controls, TransformControls, drag and selection.
- `threejs-performance` — measuring and cutting draw calls, memory, fill rate and compile stalls; LOD.
- `threejs-react-three-fiber` — the same scene graph written as React components.
- `threejs-math-noise` — hashes, value and gradient noise, fBm, domain warping, curl noise.
- `threejs-math-sdf-raymarching` — signed distance functions and sphere tracing.
- `threejs-math-particles-fields` — particle simulation, force fields, integrators, flocking, GPU compute.
