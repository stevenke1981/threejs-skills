---
name: threejs-textures
description: "Three.js texture objects: color space, wrap, repeat, filters, mipmaps, anisotropy, UV channels, Data/Canvas/Video/KTX2 compressed textures, cube and equirect env textures, PMREM mechanics, render targets, CubeCamera, texture memory and disposal. Use when a texture looks washed out or blurry, tiling, aoMap and uv2, render to texture, KTX2Loader setup. Not for the model and HDR loading flow (use threejs-loaders), scene.environment lighting (use threejs-lighting), material slots (use threejs-materials), offline compression (use threejs-asset-pipeline). 中文: 貼圖, 紋理, 色彩空間, 貼圖模糊, 貼圖重複, 渲染目標, 壓縮貼圖"
---

# Three.js Textures

Targets three.js r187 (npm `three@0.187.0`). Examples use WebGLRenderer unless they import from `three/webgpu`.

## Quick Start

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 2);

// load() returns the texture at once; the image arrives later. loadAsync() when you need it loaded.
const texture = new THREE.TextureLoader().load("diffuse.jpg");
texture.colorSpace = THREE.SRGBColorSpace; // color image: without this it renders washed out
texture.anisotropy = renderer.capabilities.getMaxAnisotropy(); // sharp at grazing angles

const mesh = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), new THREE.MeshBasicMaterial({ map: texture }));
scene.add(mesh);

renderer.setAnimationLoop(() => {
  mesh.rotation.z += 0.01;
  renderer.render(scene, camera);
});
```

Loader choice, `LoadingManager` progress and GLTF texture handling: `threejs-loaders`.

## Color Space

Color management is on by default and every texture starts at `NoColorSpace`:

| Texture | `colorSpace` |
| --- | --- |
| LDR color images: `map`, `emissiveMap`, `specularMap`, `sheenColorMap`, `specularColorMap`, LDR matcap, canvas or video used as color | `THREE.SRGBColorSpace` (set it yourself) |
| Data: normal, bump, displacement, roughness, metalness, ao, alpha, gradient, noise, packed ORM | `THREE.NoColorSpace` (default; leave it) |
| HDR / EXR / Ultra HDR environments and light maps | `THREE.LinearSRGBColorSpace` (the loaders set it) |
| `CubeTextureLoader` results | `SRGBColorSpace` (the loader sets it) |
| GLTFLoader and KTX2Loader textures | set by the loader from the file |

`texture.encoding` and `THREE.sRGBEncoding` were removed (r152); `colorSpace` replaced them.

## Wrap, Repeat, Filters, Mipmaps

```javascript
texture.wrapS = THREE.RepeatWrapping; // tiling needs repeat wrapping; default ClampToEdgeWrapping
texture.wrapT = THREE.RepeatWrapping; // MirroredRepeatWrapping flips each repeat
texture.repeat.set(4, 4);
texture.offset.set(0.5, 0.5); // UV units
texture.center.set(0.5, 0.5); // pivot for rotation
texture.rotation = Math.PI / 4; // radians

texture.minFilter = THREE.LinearMipmapLinearFilter; // default: trilinear
texture.magFilter = THREE.LinearFilter; // default; NearestFilter (both) for pixel art
texture.generateMipmaps = false; // the only switch that stops mip generation; then use a non-mip minFilter
texture.anisotropy = renderer.capabilities.getMaxAnisotropy(); // WebGPURenderer: renderer.getMaxAnisotropy()
```

- Mipmaps are generated whenever `generateMipmaps` is true, whatever the filters (since r170).
- Non-power-of-two sizes are fine: WebGL 2 and WebGPU support mipmaps and repeat wrapping on them.
- Changing wrap, filter, `flipY` or the image after the first upload needs `texture.needsUpdate = true`; `offset`, `repeat`, `rotation` and `center` do not.
- New in r187: `texture.mipmapsAutoUpdate` (default `true`); set `false` to stop the renderer regenerating mips after uploads or render-to-texture when you write the levels yourself.

## UV Channels

Every material texture samples the UV attribute chosen by `texture.channel`: `0` reads `uv` (default), `1` reads `uv1`, `2` reads `uv2`, `3` reads `uv3`. `aoMap` and `lightMap` therefore work on the default `uv`; only a separate bake unwrap goes in `uv1`:

```javascript
const aoTexture = new THREE.TextureLoader().load("ao.jpg"); // data: keep NoColorSpace
const geometry = new THREE.BoxGeometry(1, 1, 1);
geometry.setAttribute("uv1", new THREE.BufferAttribute(new Float32Array(geometry.attributes.uv.count * 2), 2)); // fill with the bake unwrap
aoTexture.channel = 1; // sample "uv1" instead of "uv"
const aoMesh = new THREE.Mesh(geometry, new THREE.MeshStandardMaterial({ aoMap: aoTexture }));
```

GLTFLoader maps `TEXCOORD_1` to `uv1` and sets `channel` itself. Since r151 `uv2` is not special: the old "copy uv into uv2 for aoMap" step is obsolete.

## Choosing a Texture Type

| Source | Class | Notes |
| --- | --- | --- |
| Image file | `TextureLoader` → `Texture` | flipY true, mipmaps on |
| Typed array pixels | `DataTexture` | Nearest filters, flipY false, mipmaps off, `needsUpdate = true` after filling |
| 2D canvas | `CanvasTexture` | `needsUpdate = true` after each redraw |
| `<video>` | `VideoTexture` | updates itself per decoded frame, mipmaps off |
| WebCodecs `VideoFrame` | `VideoFrameTexture` | push frames with `setFrame()`, close the old one |
| HTML element | `HTMLTexture` | HTML-in-Canvas API; empty where the browser lacks it |
| GPU-compressed file | `KTX2Loader` → `CompressedTexture` | smaller memory; needs `detectSupport(renderer)` |
| Six images / equirect HDR | `CubeTextureLoader` / `HDRLoader` | environment textures |
| Rendered by three.js | `WebGLRenderTarget.texture` / `RenderTarget.texture` | render to texture |

Code for each, KTX2 setup and atlases: `references/texture-types.md`.

## Environment Textures (summary)

- An equirectangular texture must get `texture.mapping = THREE.EquirectangularReflectionMapping` before use as `scene.background`, `scene.environment` or `envMap`; without it a background draws as a flat screen quad.
- Both renderers convert equirect and cube environments to a PMREM themselves for PBR materials; do not pre-run a loaded file through `PMREMGenerator`. Use `PMREMGenerator` for generated scenes (`RoomEnvironment`).
- r187 rebuilt PMREM on cube render targets: `CubeUVReflectionMapping` is gone and PBR reflections look slightly different from r186.

Cube maps, PMREM mechanics, HDR formats and memory: `references/environment-textures.md`. Assigning `scene.environment`, intensity and rotation: `threejs-lighting`.

## Render Targets (summary)

```javascript
const renderTarget = new THREE.WebGLRenderTarget(512, 512, { samples: 4 }); // MSAA resolved into .texture
renderer.setRenderTarget(renderTarget);
renderer.render(rtScene, rtCamera); // rtScene must not contain the mesh that samples the target
renderer.setRenderTarget(null);
screenMesh.material.map = renderTarget.texture;
```

Depth textures, MRT (`count`), cube targets with CubeCamera, the WebGPU classes (`RenderTarget`, `CubeRenderTarget`) and resizing: `references/render-targets.md`.

## Disposal and Memory

`texture.dispose()` frees the GPU upload once no other undisposed texture shares the same source (clones share it); the image stays in JS memory. `material.dispose()` does not dispose its textures.

```javascript
function disposeMaterial(material) {
  for (const value of Object.values(material)) {
    if (value && value.isTexture) value.dispose(); // only if no other material still uses it
  }
  material.dispose();
}
console.log(renderer.info.memory.textures); // textures currently on the GPU (WebGLRenderer)
```

GPU memory is width x height x 4 bytes for RGBA8, x 1.33 with mipmaps, whatever the file size; a 4096² JPEG costs about 85 MiB. Use KTX2, smaller sizes, and share one texture instance across materials. Budgeting strategy: `threejs-performance`.

## WebGPU and TSL

Textures, loaders and color spaces work the same under WebGPURenderer. Differences:

- `renderer.getMaxAnisotropy()` instead of `renderer.capabilities.getMaxAnisotropy()` (no `capabilities` object).
- `KTX2Loader.detectSupport(renderer)` reads device features, so call it after `await renderer.init()`.
- Cube render targets: `CubeRenderTarget` from `three/webgpu`; `WebGLCubeRenderTarget` is not usable with WebGPURenderer (since r183).
- In TSL, `texture(map)` samples `uv(map.channel)` plus the texture's offset/repeat/rotation; `texture(map, uvNode)` takes explicit coordinates and skips that transform (`uv(1)` reads `uv1`). More TSL sampling in `threejs-shaders`.

## Common Mistakes

- Color map left at `NoColorSpace` → `texture.colorSpace = THREE.SRGBColorSpace;`: renders too bright and washed out.
- `normalMap.colorSpace = THREE.SRGBColorSpace` or `LinearSRGBColorSpace` → leave `NoColorSpace`: data maps must not be color-converted.
- `texture.encoding = THREE.sRGBEncoding` → `texture.colorSpace = THREE.SRGBColorSpace`.
- `geometry.setAttribute("uv2", geometry.attributes.uv)` for aoMap → nothing: aoMap reads `uv` unless `channel` says otherwise.
- `texture.minFilter = THREE.LinearFilter; // no mipmaps` → `texture.generateMipmaps = false;`.
- Disabling mipmaps or repeat "because the size is not a power of two" → not needed in WebGL 2 / WebGPU.
- `texture.repeat.set(4, 4)` that stretches instead of tiling → also set `wrapS`/`wrapT` to `RepeatWrapping`.
- `mesh.material.map = texture;` after the first render, map invisible → also `mesh.material.needsUpdate = true;` (WebGLRenderer compiled the shader without a map).
- Blurry texture on a floor at grazing angles → raise `anisotropy` to the hardware maximum.
- `scene.background = hdr` drawn as a flat quad → set `hdr.mapping = THREE.EquirectangularReflectionMapping` first.
- `scene.environment = pmrem.fromEquirectangular(hdr).texture` for a loaded file → assign `hdr` (with `mapping` set) directly; the renderer prefilters it.
- `ktx2Loader.detectSupportAsync(renderer)` → `await renderer.init(); ktx2Loader.detectSupport(renderer);` (async variant deprecated r181).
- Loading a `.ktx2` without `detectSupport` → throws `Missing initialization with .detectSupport( renderer )`.
- `new THREE.WebGLMultipleRenderTargets(w, h, 2)` → `new THREE.WebGLRenderTarget(w, h, { count: 2 })` (removed r162).
- `new THREE.WebGLCubeRenderTarget(256)` with WebGPURenderer → `new THREE.CubeRenderTarget(256)` from `three/webgpu`.
- CubeCamera reflection showing the reflective object itself → hide it around `cubeCamera.update(renderer, scene)`.
- `texture.anisotropy = renderer.capabilities.getMaxAnisotropy()` under WebGPURenderer → `renderer.getMaxAnisotropy()`.

## Reference Files

- `references/texture-types.md` - Texture from an image element, DataTexture, CanvasTexture, VideoTexture, VideoFrameTexture, HTMLTexture, KTX2 compressed textures, atlases, material slot channel conventions. Open when creating textures from code or setting up KTX2.
- `references/environment-textures.md` - cube textures, equirect mapping, HDR/EXR/Ultra HDR, PMREMGenerator mechanics in r187, environment texture memory. Open when building or debugging environment and reflection textures.
- `references/render-targets.md` - render-to-texture, depth textures, MSAA, MRT, depth-only targets, cube render targets and CubeCamera, WebGPU equivalents, resizing and disposal. Open for any offscreen rendering.

## See Also

- `threejs-materials` - which material slot takes which texture
- `threejs-loaders` - TextureLoader, HDR/EXR loaders, GLTFLoader with KTX2, LoadingManager progress
- `threejs-lighting` - scene.environment, IBL intensity and rotation
- `threejs-shaders` - sampling textures in GLSL and TSL
- `threejs-asset-pipeline` - encoding textures to KTX2, WebP or AVIF offline
- `threejs-performance` - texture memory budgets
