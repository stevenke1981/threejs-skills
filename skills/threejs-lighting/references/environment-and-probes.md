# Environment Lighting (IBL) and Light Probes

Purpose: lighting a scene from an HDR or a generated room, PMREM, cube environments, baked spherical-harmonic probes and probe grids. Targets three.js r187. Loading the files themselves is `threejs-loaders`; texture settings are `threejs-textures`.

## HDR environment

```javascript
import { HDRLoader } from "three/addons/loaders/HDRLoader.js";

// Both renderers prefilter an equirect environment themselves: assign it directly
const hdr = await new HDRLoader().loadAsync("sunset.hdr");
hdr.mapping = THREE.EquirectangularReflectionMapping;

scene.environment = hdr; // lights the scene
scene.background = hdr; // optional: also draw it behind the scene
scene.environmentIntensity = 1; // scales the lighting contribution only
scene.environmentRotation.set(0, Math.PI / 4, 0); // rotates the lighting
scene.backgroundBlurriness = 0; // blurs the background, not the lighting
scene.backgroundIntensity = 1;
scene.backgroundRotation.set(0, 0, 0); // rotates the background independently
```

- Which materials it lights: both renderers light MeshStandardMaterial and MeshPhysicalMaterial (and their node versions) from `scene.environment`. WebGLRenderer also lights MeshLambertMaterial and MeshPhongMaterial from it when they have no `envMap` of their own; under WebGPURenderer the Lambert and Phong node materials use only their own `material.envMap`.
- `material.envMapIntensity` scales only a material's own `envMap`, not `scene.environment`; dim the scene-wide lighting with `scene.environmentIntensity`.
- Without the mapping line, HDRLoader's texture has UV mapping and the background draws as a flat screen-space image.
- `RGBELoader` was renamed `HDRLoader` in r180.
- r184 aligned environment and background rotation with how 3D objects rotate; rotation values written for r183 or older may need retuning.

## PMREMGenerator: for generated environments

Keep `PMREMGenerator` for environments generated from a scene; on an equirect file it only repeats the prefiltering the renderer already does.

```javascript
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";

const pmrem = new THREE.PMREMGenerator(renderer);
const room = new RoomEnvironment();
scene.environment = pmrem.fromScene(room, 0.04).texture; // (scene, sigma = 0, near = 0.1, far = 100, { size = 256, position })
room.dispose(); // frees the room's geometries and materials
pmrem.dispose();
```

- r187 changed PMREMs to cube render targets and removed `CubeUVReflectionMapping`; prefiltered environments look slightly different from r186. Code that tested `texture.mapping === THREE.CubeUVReflectionMapping` must go.
- `options.size` is rounded down to a power of two, minimum 256; `options.position` moves the internal cube camera.
- Under WebGPURenderer use `THREE.PMREMGenerator` from `three/webgpu` (same API). A sky dome as environment: see `threejs-math-atmosphere`.

## Cube texture environment

A cube texture works as an environment without any mapping line:

```javascript
const cubeEnv = await new THREE.CubeTextureLoader().loadAsync([
  "px.jpg",
  "nx.jpg",
  "py.jpg",
  "ny.jpg",
  "pz.jpg",
  "nz.jpg",
]);
// CubeTextureLoader already sets colorSpace to SRGBColorSpace

scene.environment = cubeEnv;
scene.background = cubeEnv;
```

## Light probes (spherical harmonics)

```javascript
import { LightProbeGenerator } from "three/addons/lights/LightProbeGenerator.js";

// Bake ambient spherical harmonics from what a cube camera sees
const cubeCamera = new THREE.CubeCamera(0.1, 100, new THREE.WebGLCubeRenderTarget(256));
cubeCamera.update(renderer, scene);

const lightProbe = await LightProbeGenerator.fromCubeRenderTarget(renderer, cubeCamera.renderTarget);
scene.add(lightProbe);
```

- `fromCubeRenderTarget` is async (since r169) and returns a Promise; `LightProbeGenerator.fromCubeTexture(cubeTexture)` derives a probe synchronously from an already-loaded cube texture.
- Under WebGPURenderer build the target with `new THREE.CubeRenderTarget(256)` from `three/webgpu`; `WebGLCubeRenderTarget` is not usable there since r183.
- Upcoming: the migration guide's 187 → 188 section (unreleased dev at the time of r187) deprecates `LightProbe`, `LightProbeGenerator` and `LightProbeHelper`, with a 1 × 1 × 1 probe grid as the replacement. For code that should survive the next upgrade, prefer a probe grid or `scene.environment` now.

## Probe grids: position-dependent ambient

A 3D grid of L2 SH irradiance probes, baked from the scene. `LightProbeGridWebGL` is for WebGLRenderer, `LightProbeGrid` for WebGPURenderer (`three/addons/lighting/`; r186 renamed the WebGL one and gave the plain name to the WebGPU version). Both are lights: adding the grid to the scene applies its baked irradiance to every lit material.

```javascript
import { LightProbeGridWebGL } from "three/addons/lighting/LightProbeGridWebGL.js";

// (width, height, depth, widthProbes?, heightProbes?, depthProbes?): a volume centred on .position
const probeGrid = new LightProbeGridWebGL(10, 4, 10); // 10 × 4 × 10 probes by default (see below)
probeGrid.position.set(0, 2, 0);
probeGrid.bake(renderer, scene, { cubemapSize: 32, near: 0.05, far: 20 }); // bake again when the static lighting changes
scene.add(probeGrid); // the official examples bake first, then add
```

- r187 changed the default probe count to `Math.max(1, Math.round(size))` per axis, one probe at the centre of each cell (r186 placed `round(size) + 1` probes, at least 2, on the volume's edges). Pass the probe counts explicitly when the volume is not in world units.
- Bake options: `cubemapSize` (default 8; a multiple of 4 for the WebGPU grid), `near` 0.1, `far` 100, `bounces` 0 (extra GI bounces; whole-grid bakes only), and `start`/`count`/`pass` for spreading a bake over frames.
- Helpers: `LightProbeGridHelperWebGL` / `LightProbeGridHelper` in `three/addons/helpers/`.
- WebGPURenderer also has `LightMapper` (`three/addons/lighting/LightMapper.js`, new in r187), which bakes irradiance from a probe grid or VXGI into a `lightMap` on meshes with `uv1` light-map UVs.
