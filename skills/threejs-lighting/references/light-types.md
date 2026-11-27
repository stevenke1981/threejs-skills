# Light Types: Full API and Recipes

Purpose: constructor signatures, per-light details (SunLight, CSM, RectAreaLight on both renderers), helpers, classic rigs and animated lights. Targets three.js r187.

## Ambient and Hemisphere

```javascript
// AmbientLight(color, intensity) — a flat multiplier on every lit material
const ambient = new THREE.AmbientLight(0x404040, 0.5);
scene.add(ambient);

// HemisphereLight(skyColor, groundColor, intensity)
const hemi = new THREE.HemisphereLight(0x87ceeb, 0x8b6f47, 0.6);
scene.add(hemi);
```

Both are unit-less multipliers and cast no shadows. Prefer `scene.environment` (see `environment-and-probes.md`) when the scene uses MeshStandardMaterial or MeshPhysicalMaterial: an environment gives directional variation and reflections, an ambient light gives neither.

## DirectionalLight

```javascript
// DirectionalLight(color, intensity): parallel rays from position towards target; the distance only matters to the shadow camera
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
dirLight.position.set(5, 10, 5);

// Shines from its position towards target; add the target to the scene when you move it
dirLight.target.position.set(0, 0, 0);
scene.add(dirLight.target);
scene.add(dirLight);
```

## SunLight: automatic cascaded shadows

`SunLight` (`three/addons/lights/SunLight.js`) is a parallel light that casts two cascaded shadow maps, suited to large scenes. It has no target: like HemisphereLight its direction comes from its position, shining towards the origin, straight down by default. With WebGLRenderer it works out of the box; under WebGPURenderer call `SunLight.registerNode(SunLightNode)` once first (see `webgpu-tsl.md`).

```javascript
import { SunLight } from "three/addons/lights/SunLight.js";

const sun = new SunLight(0xfff2e3, 3);
sun.position.set(100, 160, 60);
sun.castShadow = true;
scene.add(sun);
```

- The cascades are fitted automatically to slices of the camera's view frustum, up to `sun.shadow.camera.far` (default 500) or the view camera's far plane, whichever is smaller. `shadow.camera.left/right/top/bottom` are ignored; the default `mapSize` is 1024 × 1024 per cascade. There is no shadow frustum to hand-tighten.
- Adjacent cascades blend over 10% of each cascade's depth range.
- Not for small bounded scenes (a diorama, one room, a model viewer): the r187 docs warn that most content then lands in the last, lowest-resolution cascade. Use a DirectionalLight with a tight shadow frustum there.
- `LightProbeGrid.bake()` temporarily swaps shadow-casting SunLights for equivalent directional lights, because view-fitted cascades cannot be frozen across probe renders.

### CSM addon (older, configurable)

`CSM` (`three/addons/csm/CSM.js`, WebGLRenderer) and `CSMShadowNode` (`three/addons/csm/CSMShadowNode.js`, WebGPURenderer) still ship in r187. CSM creates one DirectionalLight per cascade and offers `cascades` (default 3), `maxFar`, split `mode` (`'practical'` default, `'uniform'`, `'logarithmic'`, `'custom'`) and `fade`; with the WebGL `CSM` every receiving material must be patched with `csm.setupMaterial(material)` and `csm.update()` called each frame. `CSMShadowNode(light, data)` takes the DirectionalLight it shadows. Prefer SunLight for new code unless you need more than two cascades or custom splits.

## PointLight and SpotLight

```javascript
// PointLight(color, intensity in candela, distance, decay)
const bulb = new THREE.PointLight(0xffdcb0, 100, 20, 2);
bulb.position.set(0, 4, 0);
scene.add(bulb);

// SpotLight(color, intensity in candela, distance, angle, penumbra, decay)
const deskLamp = new THREE.SpotLight(0xffffff, 250, 30, Math.PI / 6, 0.4, 2);
deskLamp.position.set(2, 5, 0);
deskLamp.target.position.set(0, 0, 0);
scene.add(deskLamp.target);
scene.add(deskLamp);

// Prefer lumens: power writes the candela intensity for you
bulb.power = 800; // PointLight: intensity = power / (4π)
```

- `angle` is the cone half-angle in radians (default Math.PI / 3, upper bound Math.PI / 2); `penumbra` (0 to 1) softens the cone edge.
- `power` conversions: PointLight `intensity · 4π`, SpotLight `intensity · π`, RectAreaLight `intensity · width · height · π`.
- `distance` is a smooth cutoff range (0, the default, means no limit) and also sets the shadow camera's far plane (see `shadows.md`).

## RectAreaLight

A rectangular emitter for soft window- or strip-style lighting. It lights only MeshStandardMaterial and MeshPhysicalMaterial (and their node equivalents), and it cannot cast shadows. It emits along its local −Z axis; `lookAt()` on a light points −Z at the target.

The light needs LTC lookup textures, registered differently per renderer:

```javascript
// WebGLRenderer
import { RectAreaLightUniformsLib } from "three/addons/lights/RectAreaLightUniformsLib.js";
RectAreaLightUniformsLib.init(); // once, before the first render

// WebGPURenderer (THREE from "three/webgpu")
import { RectAreaLightTexturesLib } from "three/addons/lights/RectAreaLightTexturesLib.js";
THREE.RectAreaLightNode.setLTC(RectAreaLightTexturesLib.init()); // once
```

```javascript
import { RectAreaLightHelper } from "three/addons/helpers/RectAreaLightHelper.js";

// RectAreaLight(color, intensity in nits, width, height)
const windowLight = new THREE.RectAreaLight(0xffffff, 5, 4, 10);
windowLight.position.set(-5, 5, 0);
windowLight.lookAt(0, 0, 0); // emits from its front face (−Z) towards the origin
scene.add(windowLight);

// The class docs say: add the helper as a child of the light
const helper = new RectAreaLightHelper(windowLight);
windowLight.add(helper);
```

The helper rebuilds its world matrix from the light's in `updateMatrixWorld()` (ignoring the light's world scale), which is why the official examples can also add it to the scene directly.

## Light Helpers

```javascript
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
const hemi = new THREE.HemisphereLight(0x87ceeb, 0x8b6f47, 0.6);
const bulb = new THREE.PointLight(0xffdcb0, 100, 20);
const deskLamp = new THREE.SpotLight(0xffffff, 250, 30);
scene.add(dirLight, hemi, bulb, deskLamp);

// Built-in helpers are objects: add them to the scene
const dirHelper = new THREE.DirectionalLightHelper(dirLight, 5);
const pointHelper = new THREE.PointLightHelper(bulb, 0.5);
const spotHelper = new THREE.SpotLightHelper(deskLamp);
const hemiHelper = new THREE.HemisphereLightHelper(hemi, 5);
scene.add(dirHelper, pointHelper, spotHelper, hemiHelper);
```

Every light helper copies the light's color only in `update()`, and DirectionalLightHelper, SpotLightHelper and HemisphereLightHelper also re-aim only there: call `update()` after recoloring a light, and after moving or re-aiming one of those three.

## Common Setups

```javascript
// Three-point lighting: key, fill, rim
const keyLight = new THREE.DirectionalLight(0xffffff, 2);
keyLight.position.set(5, 6, 4);

const fillLight = new THREE.DirectionalLight(0xffffff, 0.5);
fillLight.position.set(-5, 3, 5);

const backLight = new THREE.DirectionalLight(0xffffff, 1);
backLight.position.set(-4, 5, -6);

scene.add(keyLight, fillLight, backLight);
```

```javascript
import { SunLight } from "three/addons/lights/SunLight.js";

// Outdoor daylight: sun with cascaded shadows plus a sky gradient
const sun = new SunLight(0xfff2e3, 3);
sun.position.set(100, 160, 60);
sun.castShadow = true;

const sky = new THREE.HemisphereLight(0x87ceeb, 0x8b6f47, 0.5);

scene.add(sun, sky);
```

For a physically based sky dome that matches the sun direction, see `threejs-math-atmosphere`.

## Animating Lights

```javascript
// Timer drives per-frame time: update() first, then read it
const timer = new THREE.Timer();
timer.connect(document); // avoids huge deltas after tab switches

const lamp = new THREE.PointLight(0xffd0a0, 100, 20);
scene.add(lamp);

function animate() {
  timer.update();

  const t = timer.getElapsed();
  lamp.position.set(Math.cos(t) * 4, 3, Math.sin(t) * 4);
  lamp.intensity = 100 + Math.sin(t * 2) * 30; // changing intensity never recompiles

  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```
