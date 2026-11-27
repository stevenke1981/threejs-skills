---
name: threejs-loaders
description: "Runtime asset loading in three.js r187: GLTFLoader with Draco, meshopt and KTX2 decoders, OBJ, FBX, STL, PLY, USD and splat loaders, texture and HDR/EXR/UltraHDR loading calls, LoadingManager progress and abort, caching, SkeletonUtils cloning, disposal. Use for loading a .glb, \"No DRACOLoader instance provided\", setMeshoptDecoder or detectSupport errors, RGBELoader warnings, progress bars, clones that ignore their animation. Not for compressing files (use threejs-asset-pipeline), texture settings (use threejs-textures), IBL setup (use threejs-lighting), playing clips (use threejs-animation), useGLTF (use threejs-react-three-fiber). 中文: 載入模型, 載入器, 載入進度, 解碼器, 複製模型, 釋放資源"
---

# Three.js Loaders

Targets three.js r187 (npm `three@0.187.0`). Examples use WebGLRenderer; `references/webgpu.md` covers WebGPURenderer. This skill owns the loading call; texture configuration is `threejs-textures`, environment lighting and PMREM are `threejs-lighting`, compression on disk is `threejs-asset-pipeline`.

## Quick Start

```javascript
import * as THREE from "three";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.environment = new THREE.PMREMGenerator(renderer).fromScene(new RoomEnvironment(), 0.04).texture; // glTF PBR materials render black without lights or an environment

const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(2, 1.5, 3);
camera.lookAt(0, 0, 0);

const gltf = await new GLTFLoader().loadAsync("model.glb"); // { scene, scenes, animations, cameras, asset, parser, userData }
scene.add(gltf.scene);

renderer.setAnimationLoop(() => renderer.render(scene, camera));
```

Every loader has `loadAsync(url, onProgress)`, which rejects on failure; do not wrap `load()` in a hand-made Promise. Addons import from `three/addons/...`, the specifier the three.js import map and package exports define.

## Choosing a Loader

| File | Loader (`three/addons/loaders/…`) | Result | Note |
|---|---|---|---|
| `.glb` / `.gltf` | `GLTFLoader` | `{ scene, animations, … }` | The web delivery format; prefer converting others to it |
| `.obj` + `.mtl` | `OBJLoader`, `MTLLoader` | `Group` | No animation; materials are Phong |
| `.fbx` | `FBXLoader` | `Group` with `.animations` | Often centimetre units: scale 0.01 |
| `.stl` | `STLLoader` | `BufferGeometry` | You supply material and lights |
| `.ply` | `PLYLoader` | `BufferGeometry` | `double` properties arrive as Float64 attributes |
| `.usd/.usda/.usdc/.usdz` | `USDLoader` | `Group` | `USDZLoader` is a deprecated alias (r179) |
| `.splat/.spz/.ksplat`, splat `.ply` | `SPLATLoader`, `SPZLoader`, `KSPLATLoader`, `GaussianSplatPLYLoader` | splat data | Needs WebGPURenderer, see `references/webgpu.md` |
| `.jpg/.png/.webp/.avif` | `THREE.TextureLoader` | `Texture` | Set `colorSpace` on color maps |
| `.hdr` | `HDRLoader` | `DataTexture` (half float) | Was `RGBELoader` before r180 |
| `.exr` | `EXRLoader` | `DataTexture` | |
| gain-map `.jpg` | `UltraHDRLoader` | `DataTexture` | HDR range at JPEG size |
| `.ktx2` | `KTX2Loader` | `CompressedTexture` | Needs `detectSupport(renderer)` |

Code for every non-glTF format and the less common loaders is in `references/formats.md`.

## glTF Compression Decoders

This is the runtime half of the codecs `threejs-asset-pipeline` writes. A file whose `extensionsRequired` lists a codec throws at load time unless that decoder is attached:

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { DRACOLoader } from "three/addons/loaders/DRACOLoader.js";
import { KTX2Loader } from "three/addons/loaders/KTX2Loader.js";
import { MeshoptDecoder } from "three/addons/libs/meshopt_decoder.module.js";

// One of each for the whole app: every active KTX2Loader starts its own transcoder and workers,
// and three.js warns "Multiple active KTX2 loaders may cause performance issues."
const dracoLoader = new DRACOLoader();
const ktx2Loader = new KTX2Loader().detectSupport(renderer); // after await renderer.init() on WebGPU

const gltfLoader = new GLTFLoader()
  .setDRACOLoader(dracoLoader) // KHR_draco_mesh_compression
  .setMeshoptDecoder(MeshoptDecoder) // EXT_meshopt_compression and KHR_meshopt_compression
  .setKTX2Loader(ktx2Loader); // KHR_texture_basisu

const gltf = await gltfLoader.loadAsync("model.opt.glb");
```

| Missing piece | Error thrown |
|---|---|
| DRACOLoader on a Draco file | `THREE.GLTFLoader: No DRACOLoader instance provided.` |
| MeshoptDecoder on a meshopt file | `THREE.GLTFLoader: setMeshoptDecoder must be called before loading compressed files` |
| KTX2Loader on a KTX2 file | `THREE.GLTFLoader: setKTX2Loader must be called before loading KTX2 textures` |
| `detectSupport` never called | `THREE.KTX2Loader: Missing initialization with \`.detectSupport( renderer )\`.` |

WebP and AVIF textures, `KHR_mesh_quantization` and `EXT_mesh_gpu_instancing` need no setup.

Decoder files: since r185 DRACOLoader and KTX2Loader resolve their WASM from the three package itself (`examples/jsm/libs/draco/` and `libs/basis/`, through `new URL(…, import.meta.url)`), so the default needs no path. Call `setDecoderPath` / `setTranscoderPath` only when you serve the files yourself, and copy them from the same three release you ship. `dracoLoader.setDecoderPath(DRACO_GLTF_CONFIG)` (also exported by `DRACOLoader.js`) selects the smaller glTF-only Draco build.

## LoadingManager and Progress

```javascript
const manager = new THREE.LoadingManager();
manager.onProgress = (url, loaded, total) => updateBar(loaded / total); // counts items, not bytes
manager.onLoad = () => hideBar(); // every tracked item finished
manager.onError = (url) => console.error("Failed:", url);

const gltfLoader = new GLTFLoader(manager);
const textureLoader = new THREE.TextureLoader(manager);
await Promise.all([gltfLoader.loadAsync("model.glb"), textureLoader.loadAsync("color.jpg")]);
```

- `manager.abort()` cancels in-flight requests of loaders that fetch (FileLoader, ImageBitmapLoader, and the loaders built on them: glTF, HDR, EXR, KTX2). TextureLoader loads through an `<img>` element and ignores it.
- `TextureLoader` never calls `onProgress`. For a byte-level bar, pass `onProgress` to `GLTFLoader.loadAsync`, which reports the main file's `ProgressEvent`.
- r187 added `loader.setPriority("high" | "low" | "auto")` and `loader.setCache(fetchCacheMode)`; GLTFLoader forwards both to the loaders it creates internally.

Parallel loading, `parseAsync`, blob URLs, `setPath` / `setResourcePath`, URL rewriting, `THREE.Cache` and post-load processing (shadows, centring, unit scaling) are in `references/loading-patterns.md`.

## Textures and HDR Files

Only the loading call lives here; wrap, filtering, anisotropy, flipY and UV channels are `threejs-textures`, and environment intensity, rotation and PMREM are `threejs-lighting`.

```javascript
import { HDRLoader } from "three/addons/loaders/HDRLoader.js";

const map = await new THREE.TextureLoader().loadAsync("color.jpg");
map.colorSpace = THREE.SRGBColorSpace; // color maps only; normal, roughness, metalness, ao keep NoColorSpace
material.map = map;
material.needsUpdate = true; // only when the material already rendered without a map

const env = await new HDRLoader().loadAsync("environment.hdr"); // EXRLoader and UltraHDRLoader work the same way
env.mapping = THREE.EquirectangularReflectionMapping; // without it a background draws as a flat screen quad
scene.environment = env; // both renderers prefilter an equirect environment themselves
scene.background = env;
```

- GLTFLoader sets color spaces and `flipY = false` on the textures it loads. A texture you load yourself for a glTF material needs `map.flipY = false` too.
- HDRLoader, EXRLoader and UltraHDRLoader return `LinearSRGBColorSpace` half-float textures by default (`setDataType(THREE.FloatType)` for full float).
- `CubeTextureLoader` sets `SRGBColorSpace` itself; faces go in +X, −X, +Y, −Y, +Z, −Z order.

## Cloning and Disposal

`Object3D.clone()` on a skinned model shares the skeleton, so every copy stays bound to the original's bones. Clone characters with `SkeletonUtils.clone()` (playing clips on the clones is `threejs-animation`):

```javascript
import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js";

const copy = SkeletonUtils.clone(gltf.scene); // its own bones; geometry and materials stay shared
scene.add(copy);
```

`Object3D.dispose()` only fires the object's `dispose` event (renderer-side state such as InstancedMesh buffers); it does not visit children or free geometry, materials or textures, and `material.dispose()` does not free the material's textures. Free a loaded model you will not use again:

```javascript
function disposeModel(root) {
  root.removeFromParent();
  root.traverse((child) => {
    child.dispose();
    child.geometry?.dispose();
    for (const material of [child.material ?? []].flat()) {
      for (const value of Object.values(material)) if (value?.isTexture) value.dispose();
      material.dispose();
    }
  });
}
```

Do not run this on a model whose geometry or materials are shared with clones still in the scene.

## Common Mistakes

- `import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js"` → `"three/addons/loaders/GLTFLoader.js"`: the examples/jsm specifier does not resolve under the import map three.js documents.
- `new RGBELoader()` → `new HDRLoader()` from `three/addons/loaders/HDRLoader.js`: renamed in r180; the old class still exists but warns on every construction.
- `new USDZLoader()` → `new USDLoader()`: deprecated in r179; USDLoader reads USD, USDA, USDC and USDZ.
- `new Promise((res, rej) => loader.load(url, res, undefined, rej))` → `loader.loadAsync(url)`.
- `dracoLoader.setDecoderPath("https://www.gstatic.com/draco/v1/decoders/")` → `new DRACOLoader()`: since r185 the decoder ships with three; a path to another build can mismatch the loader.
- `dracoLoader.setDecoderConfig({ type: "js" })` → `new DRACOLoader()`: deprecated in r185 (warns that it will be removed in r194); WASM is used wherever WebAssembly exists.
- `ktx2Loader.detectSupportAsync(renderer)` → `await renderer.init(); ktx2Loader.detectSupport(renderer);`: deprecated in r181.
- A new `KTX2Loader` per model → one shared instance: each active instance loads its own transcoder and worker pool.
- `material.map = new THREE.TextureLoader().load("color.jpg")` with no color space → set `map.colorSpace = THREE.SRGBColorSpace`: loaded images default to NoColorSpace and color maps render washed out.
- `texture.encoding = THREE.sRGBEncoding` → `texture.colorSpace = THREE.SRGBColorSpace`: `encoding` and `sRGBEncoding` no longer exist.
- `scene.environment = pmrem.fromEquirectangular(hdr).texture` for a file → set `hdr.mapping = THREE.EquirectangularReflectionMapping` and assign it: both renderers prefilter it themselves.
- `gltf.scene.clone()` for a second character → `SkeletonUtils.clone(gltf.scene)`.
- `gltf.scene.dispose()` to free a model → traverse and dispose geometry, materials and textures as above.
- A PLY scan with `double` properties under WebGLRenderer → convert Float64 attributes to `Float32BufferAttribute` first: WebGLRenderer throws `Unsupported buffer data format` (see `references/formats.md`).
- `fetch(url, { signal })` to cancel a model → `manager.abort()` on the loader's LoadingManager.

## Reference Files

- `references/formats.md` — open for OBJ/MTL, FBX, STL, PLY, USD, Gaussian splats, CubeTextureLoader, UltraHDR/JXL and the list of other bundled loaders.
- `references/loading-patterns.md` — open for parallel loading, parsing buffers and blobs, path and URL rewriting, caching, abort with timeouts, request priority, and processing a loaded scene (shadows, named nodes, centring and scaling, placeholders).
- `references/webgpu.md` — open when the renderer is WebGPURenderer: init ordering for KTX2 and PMREM, anisotropy, node materials, splats.

## See Also

- `threejs-asset-pipeline` — producing the Draco, meshopt and KTX2 files these decoders read.
- `threejs-animation` — AnimationMixer and clips from `gltf.animations`.
- `threejs-textures` — texture properties after loading; `threejs-lighting` — HDR environments and IBL.
- `threejs-fundamentals` — renderer, camera and loop around loaded assets.
- `threejs-react-three-fiber` — `useGLTF` and Suspense in React.
