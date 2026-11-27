---
name: threejs-lighting
description: "Three.js r187 lights and shadows: light types and physical units, shadow maps and bias, SunLight cascades, RectAreaLight, HDR environment lighting (IBL), light probes and probe grids, on WebGL or WebGPU/TSL. Use for adding or tuning lights, shadows not showing, shadow acne, a PointLight too dim, lighting from an HDR, or dimming an environment map. Not for material response (use threejs-materials), loading HDR files (use threejs-loaders), sky or fog (use threejs-math-atmosphere), bloom (use threejs-postprocessing), exposure (use threejs-fundamentals). 中文: 燈光, 光源, 陰影, 陰影痤瘡, 環境光照, HDR環境貼圖, 光照探針"
---

# Three.js Lighting

Targets three.js r187. Examples use WebGLRenderer; `references/webgpu-tsl.md` covers WebGPURenderer.

## Quick Start

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();

const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(4, 3, 8);

// A single directional light: intensity is a plain multiplier, no distance falloff
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
dirLight.position.set(5, 10, 5);
scene.add(dirLight);

const floor = new THREE.Mesh(
  new THREE.PlaneGeometry(20, 20),
  new THREE.MeshStandardMaterial({ color: 0x808080 })
);
floor.rotation.x = -Math.PI / 2;
scene.add(floor);

const mesh = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0x3a76c4 })
);
scene.add(mesh);

// Shadows need three switches: renderer, light, and the meshes that cast and receive
renderer.shadowMap.enabled = true;
dirLight.castShadow = true;
mesh.castShadow = true;
floor.receiveShadow = true;

function animate() {
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

## Light Types and Units

| Light | Shadows | Intensity | Notes |
|---|---|---|---|
| AmbientLight | no | multiplier | Flat fill from every direction |
| HemisphereLight | no | multiplier | Sky color from above, ground color from below |
| DirectionalLight | yes | multiplier | Parallel rays (sun); shines at its `target` |
| SunLight (addon) | yes, 2 cascades | multiplier | Shadow frustum fitted to the view automatically; WebGPU needs `SunLight.registerNode(SunLightNode)` |
| PointLight | yes, six faces | candela | Inverse-square falloff (`decay` 2) |
| SpotLight | yes | candela | Cone; inverse-square falloff (`decay` 2) |
| RectAreaLight | no | nits | Only Standard/Physical materials; needs LTC textures registered |
| LightProbe | no | multiplier | Spherical-harmonics ambient, usually baked |

PointLight and SpotLight intensity is candela with inverse-square falloff: a 1-candela light 5 units away contributes 1/25, so useful values depend on distance and typically run to the hundreds (the official docs' examples use 80 to 1000). `light.power` sets the same thing in lumens. `light.distance` is a smooth cutoff range (0 means no limit). Directional, ambient and hemisphere intensities are plain unit-less multipliers. RectAreaLight intensity is nits (candela per square metre). Constructors, RectAreaLight setup per renderer, helpers and rigs: `references/light-types.md`.

## Choosing a Setup

- **Product or character viewer, PBR materials:** `scene.environment` from an HDR or `RoomEnvironment`, plus one shadow-casting DirectionalLight as key. Skip ambient lights: the environment already fills.
- **Small bounded scene (room, diorama):** DirectionalLight with a shadow frustum fitted tightly to the scene. SunLight's cascades waste resolution here.
- **Large outdoor scene:** SunLight (cascades fitted to the view) plus HemisphereLight or an environment. Need more than two cascades or custom splits: the `CSM` / `CSMShadowNode` addons.
- **Lamps, torches, many small lights:** PointLight/SpotLight in candela, shadows on at most a few. Under WebGPU, `ClusteredLighting` for hundreds of unshadowed point and spot lights.
- **Soft window or strip light:** RectAreaLight (no shadows; pair it with a shadow-casting directional light if needed).
- **Ambient that varies by location:** a baked probe grid (`references/environment-and-probes.md`).
- **Matching sky and sun:** `threejs-math-atmosphere` for the sky, with the light on the same direction.

## Shadows: the Essentials

```javascript
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap; // default; BasicShadowMap hard, VSMShadowMap blurred

const shadowCam = dirLight.shadow.camera; // orthographic, at the light, looking at its target
shadowCam.left = -10;
shadowCam.right = 10;
shadowCam.top = 10;
shadowCam.bottom = -10;
shadowCam.near = 0.5;
shadowCam.far = 50;
shadowCam.updateProjectionMatrix();

dirLight.shadow.mapSize.set(2048, 2048); // default 512
dirLight.shadow.normalBias = 0.02; // acne on surfaces grazed by the light
dirLight.shadow.bias = -0.0001; // start tiny; too much detaches shadows (peter-panning)
dirLight.shadow.intensity = 0.7; // 0 removes the shadow, 1 is full darkness
```

- Nothing shadows: check all three switches, then that every caster lies between the shadow camera's `near` and `far` and inside its left/right/top/bottom box (`new THREE.CameraHelper(dirLight.shadow.camera)` shows it).
- Blurry or blocky shadows: shrink the frustum before raising `mapSize`.
- Spot and point shadow cameras are rebuilt every pass from `angle`, `shadow.focus` and `distance`; setting their `fov` or `far` by hand does nothing (a manual `far` survives only while `distance` is 0).
- Since r187 both renderers filter casters by `shadow.camera.layers` once it enables a layer other than 0.

Shadow types table, VSM caveats, layers recipe, static shadows and contact shadows: `references/shadows.md`.

## Environment Lighting (IBL)

```javascript
import { HDRLoader } from "three/addons/loaders/HDRLoader.js";

const hdr = await new HDRLoader().loadAsync("sunset.hdr");
hdr.mapping = THREE.EquirectangularReflectionMapping; // without it the background is a flat image
scene.environment = hdr; // both renderers prefilter an equirect environment themselves
scene.background = hdr;
scene.environmentIntensity = 0.5; // dims the lighting; material.envMapIntensity does not touch scene.environment
```

Keep `PMREMGenerator` for environments generated from a scene (`RoomEnvironment`, a sky). r187 rebuilt PMREM on cube render targets and removed `CubeUVReflectionMapping`. Under WebGPURenderer `scene.environment` lights only Standard/Physical node materials, not Lambert or Phong. Details, cube environments, light probes and probe grids: `references/environment-and-probes.md`.

## Performance

- Both renderers compile the set of lights into every lit material: adding or removing a light (an AmbientLight only under WebGPURenderer), or toggling `castShadow`, recompiles them. Create lights up front and switch them at runtime through `intensity`. Under WebGPURenderer, `DynamicLighting` avoids recompiles for unshadowed lights and `ClusteredLighting` scales to many point and spot lights.
- Every shadow-casting light adds depth passes (six per point light). Leave `castShadow` off where nothing needs the shadow.
- Static scenes: `light.shadow.autoUpdate = false; light.shadow.needsUpdate = true;`, and set `needsUpdate` again after something moves.
- Lights are not filtered per mesh by layers: a light's layers are tested only against the camera's. To light selected meshes only, use `material.lightsNode` on a node material.
- Measuring frame cost and draw calls: `threejs-performance`.

## Common Mistakes

- `light.layers.set(1);` → `material.lightsNode = lights([keyLight, fillLight]);`: a light's layers are tested against the camera's, so this switches the light off for the whole scene; per-material selection needs a node material, under WebGPURenderer or under WebGLRenderer through the WebGLNodesHandler addon.
- `new THREE.PointLight(0xffffff, 1, 100);` → `new THREE.PointLight(0xffffff, 100, 20);`: point and spot intensity is candela with inverse-square falloff, so 1 contributes almost nothing at range.
- `renderer.useLegacyLights = true;` → retune point and spot lights in candela: legacy light units are gone, and neither useLegacyLights nor physicallyCorrectLights does anything.
- `renderer.shadowMap.type = THREE.PCFSoftShadowMap;` → `THREE.PCFShadowMap`: deprecated for WebGLRenderer in r182 and removed from WebGPURenderer in r186; both warn and fall back to PCFShadowMap, which is already soft.
- `renderer.library.addLight(SunLightNode, SunLight);` → `SunLight.registerNode(SunLightNode);`: light-node registration moved to the light class in r187; the old call warns.
- `lightProbe.copy(LightProbeGenerator.fromCubeRenderTarget(renderer, rt));` → `lightProbe.copy(await LightProbeGenerator.fromCubeRenderTarget(renderer, rt));`: the call is async and returns a Promise.
- `new ContactShadows({ opacity: 0.5 });` → a radial-gradient plane (`references/shadows.md`): three.js has no ContactShadows class; it is a drei component.
- `scene.environment = pmremGenerator.fromEquirectangular(texture).texture;` → `texture.mapping = THREE.EquirectangularReflectionMapping; scene.environment = texture;`: the renderer prefilters an equirect environment itself.
- `scene.background = hdr;` alone → set `hdr.mapping = THREE.EquirectangularReflectionMapping` first: HDRLoader returns UV mapping, which draws as a flat screen-space image.
- `import { RGBELoader } from "three/examples/jsm/loaders/RGBELoader.js";` → `import { HDRLoader } from "three/addons/loaders/HDRLoader.js";`: renamed in r180, and addons resolve through `three/addons/`.
- `spotLight.shadow.camera.fov = 30;` → `spotLight.shadow.focus = 0.9;`: the shadow fov is recomputed each pass as 2 × angle × focus in degrees.
- `lamp.shadow.camera.far = 50;` on a PointLight with distance 20 → `new THREE.PointLight(0xffffff, 100, 50)`: a non-zero distance overwrites the shadow far on every pass.
- `material.envMapIntensity = 0.5;` → `scene.environmentIntensity = 0.5;`: envMapIntensity scales only a material's own envMap.
- `RectAreaLightUniformsLib.init();` under WebGPURenderer → `THREE.RectAreaLightNode.setLTC(RectAreaLightTexturesLib.init());`: the uniforms lib is never read there, and the light node fails until the LTC textures are registered.
- `texture.mapping === THREE.CubeUVReflectionMapping` → remove the check: the constant was removed in r187 when PMREMs moved to cube render targets.

## References

- `references/light-types.md` — every light's constructor and options, SunLight and CSM details, RectAreaLight on both renderers, helpers, three-point and outdoor rigs, animated lights. Open when writing or debugging a specific light.
- `references/shadows.md` — shadow map types, VSM limits, directional/spot/point shadow cameras, bias, caster layers, static shadows, contact shadows. Open for any shadow problem beyond the essentials above.
- `references/environment-and-probes.md` — HDR and cube environments, PMREMGenerator and the r187 PMREM change, LightProbeGenerator, probe grids and their r187 defaults, LightMapper. Open for IBL or baked ambient.
- `references/webgpu-tsl.md` — WebGPURenderer setup, `registerNode`, DynamicLighting and ClusteredLighting, `lightsNode`, colored shadows. Open when the project imports `three/webgpu`.

## See Also

- `threejs-materials` — how materials respond to light (roughness, metalness, `material.envMap`).
- `threejs-textures` — environment-map texture objects, color spaces, render targets.
- `threejs-loaders` — HDRLoader, EXRLoader and UltraHDR loading.
- `threejs-math-atmosphere` — Sky/SkyMesh, a sun direction shared with the light, fog and volumetric light.
- `threejs-postprocessing` — bloom, SSAO/GTAO and other light-adjacent effects.
- `threejs-fundamentals` — renderer setup, tone mapping and exposure for physically scaled lights.
- `threejs-performance` — profiling the cost of lights and shadows.
