# Environment textures and PMREM

Purpose: the texture side of environment maps in r187 - cube textures, equirectangular mapping, HDR formats, what the renderers do with them (PMREM), and their memory; open when building or debugging environment and reflection textures. Assigning `scene.environment`, intensity, rotation and IBL choices belongs to `threejs-lighting`; loader options to `threejs-loaders`.

## Cube textures

```javascript
const cubeTexture = new THREE.CubeTextureLoader().load([
  "px.jpg", "nx.jpg", // +X, -X
  "py.jpg", "ny.jpg", // +Y, -Y
  "pz.jpg", "nz.jpg", // +Z, -Z
]); // the loader sets colorSpace = SRGBColorSpace and mapping = CubeReflectionMapping

scene.background = cubeTexture;
material.envMap = cubeTexture; // or scene.environment (see threejs-lighting)
```

Refraction instead of reflection on Basic/Lambert/Phong: `cubeTexture.mapping = THREE.CubeRefractionMapping` plus `material.refractionRatio`. PBR refraction uses `MeshPhysicalMaterial.transmission` instead.

## Equirectangular textures

```javascript
import { HDRLoader } from "three/addons/loaders/HDRLoader.js"; // renamed from RGBELoader in r180

const env = await new HDRLoader().loadAsync("environment.hdr"); // colorSpace = LinearSRGBColorSpace, set by the loader
env.mapping = THREE.EquirectangularReflectionMapping; // without it, as a background it draws as a flat screen quad
scene.background = env;
```

`EXRLoader` (`three/addons/loaders/EXRLoader.js`) loads `.exr` and `UltraHDRLoader` loads Ultra HDR JPEGs the same way; all three set `LinearSRGBColorSpace`. `RGBELoader` still works in r187 but logs a deprecation warning. An LDR equirect JPEG works too, with `colorSpace = SRGBColorSpace` set by you.

## What the renderers do with an environment texture

WebGLRenderer (r187 source, `WebGLEnvironments`):

- For MeshStandardMaterial / MeshPhysicalMaterial (and Lambert/Phong using `scene.environment`), and for a background with `backgroundBlurriness > 0`, an equirect or cube texture is converted once into a PMREM: a half-float `WebGLCubeRenderTarget` whose mip levels hold GGX-prefiltered radiance, roughness 0 at level 0 to roughness 1. The PMREM is cached per texture and rebuilt when the source texture is disposed or (for render-target textures) when `pmremVersion` changes.
- For a sharp background and for Basic/Lambert/Phong with their own `envMap`, an equirect texture is converted to a plain cube map (`WebGLCubeRenderTarget` with face size = image height) instead.

WebGPURenderer builds its PMREM through the `PMREMGenerator` from `three/webgpu` in the same way, so in both renderers a loaded file needs only `mapping` set.

r187 changed the PMREM implementation (PR #34585): it now uses cube render targets instead of the old CubeUV atlas, `CubeUVReflectionMapping` was removed, and roughness filtering improved - PBR reflections look slightly different from r186. Code that checked `texture.mapping === THREE.CubeUVReflectionMapping` must check `texture.isPMREMTexture` instead.

## PMREMGenerator

Use it for environments you generate, not for files you load (the renderer already prefilters those).

```javascript
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";

const pmrem = new THREE.PMREMGenerator(renderer);
const room = new RoomEnvironment();
const envTarget = pmrem.fromScene(room, 0.04); // (scene, sigma blur in radians, near, far, { size, position })
scene.environment = envTarget.texture; // isPMREMTexture === true
room.dispose(); // frees the room's geometries and materials
pmrem.dispose(); // frees the generator's internals; the returned PMREM belongs to you
// later, when replacing the environment: envTarget.dispose();
```

- r187 `fromScene` options: `size` (default 256, rounded down to a power of two, at least 256) and `position` of the internal cube camera.
- `fromEquirectangular(texture, renderTarget?)` and `fromCubemap(cubeTexture, renderTarget?)` exist for explicit control (for example re-filtering into the same target each frame).
- PMREM face size for an equirect source is `image.width / 4`, floored to a power of two, minimum 256.
- `compileEquirectangularShader()` / `compileCubemapShader()` pre-compile while the file downloads.
- WebGPURenderer: import `PMREMGenerator` from `three/webgpu`; the `*Async` variants are deprecated since r181 - `await renderer.init()` once, then call `fromScene()` etc.

## Dynamic reflections

A `CubeCamera` renders the scene into a cube render target each frame; see `render-targets.md`. Its texture can be assigned to `envMap` directly: `cubeCamera.update()` sets `texture.needsPMREMUpdate = true`, so WebGLRenderer re-filters the PMREM of PBR materials after each update.

## Memory

- A PMREM of face size N stores six half-float RGBA faces plus a mip chain: about 6 x N² x 8 bytes x 1.33. A 2048 x 1024 HDR gives N = 512, about 16 MiB.
- The plain cube conversion uses faces of the image height: a 2048 x 1024 equirect becomes six 1024² faces, three times the equirect's pixels.
- When memory matters, ship a smaller equirect (1k-2k wide is usually enough for lighting) and use a separate sharp texture for the background if needed.
- `texture.dispose()` on the source also releases the cached PMREM and cube conversions for it.
