# Lighting under WebGPURenderer and TSL

Purpose: what changes for lights, shadows and environments on WebGPURenderer: setup, light-node registration, alternative lighting systems, per-material lights, colored shadows. Targets three.js r187.

Light classes, shadow settings and the scene's environment properties carry over to WebGPURenderer, except these WebGL-only pieces:

| WebGL | WebGPU replacement |
| --- | --- |
| `RectAreaLightUniformsLib.init()` | `THREE.RectAreaLightNode.setLTC(RectAreaLightTexturesLib.init())` |
| `WebGLCubeRenderTarget` | `THREE.CubeRenderTarget` (from `three/webgpu`) |
| `LightProbeGridWebGL` | `LightProbeGrid` |
| `renderer.shadowMap.autoUpdate` | none: per light, `shadow.autoUpdate` / `shadow.needsUpdate` |
| `CSM` | `CSMShadowNode` |
| `scene.environment` lighting Lambert / Phong | not supported: give those materials their own `envMap` |

## Setup

```javascript
import * as THREE from "three/webgpu";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x101010); // an opaque background avoids premultiplied-alpha blending surprises

const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(4, 3, 8);

renderer.shadowMap.enabled = true;
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
dirLight.castShadow = true;
scene.add(dirLight);

function animate() {
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

## Registrations and lighting systems

```javascript
import * as THREE from "three/webgpu";
import { RectAreaLightTexturesLib } from "three/addons/lights/RectAreaLightTexturesLib.js";
import { SunLight } from "three/addons/lights/SunLight.js";
import { SunLightNode } from "three/addons/lights/SunLightNode.js";
import { ClusteredLighting } from "three/addons/lighting/ClusteredLighting.js";

// RectAreaLight: register the LTC textures once (replaces RectAreaLightUniformsLib)
THREE.RectAreaLightNode.setLTC(RectAreaLightTexturesLib.init());

// SunLight (and any custom light class): assign its node implementation once
SunLight.registerNode(SunLightNode);

// Many point and spot lights: clustered Forward+ shading
renderer.lighting = new ClusteredLighting(); // (maxLights = 1024, tileSize = 32, zSlices = 24, maxLightsPerCluster = 64)
```

- `Light.registerNode(LightNodeClass)` is new in r187. The old `renderer.library.addLight(SunLightNode, SunLight)` still works but logs a deprecation warning.
- `ClusteredLighting` clusters point lights and, since r187, spot lights. Lights that cast shadows, have a `colorNode`, or are spot lights with a `map` (and subclasses with their own node, such as IESSpotLight or ProjectorLight) keep the default per-light path.
- `DynamicLighting` (`three/addons/lighting/DynamicLighting.js`) batches ambient, hemisphere, directional, point and spot lights into uniform arrays, so adding or removing them does not recompile materials. Options: `maxDirectionalLights` 8, `maxPointLights` 16, `maxSpotLights` 16, `maxHemisphereLights` 4. Shadow-casting lights, spot lights with a `map` or `colorNode`, and node lights are not batched.
- `renderer.lighting` holds one system at a time: `renderer.lighting = new DynamicLighting({ maxPointLights: 64 })` replaces the default (or a ClusteredLighting set earlier).

## Per-material light selection

```javascript
import * as THREE from "three/webgpu";
import { lights } from "three/tsl";

const keyLight = new THREE.PointLight(0x66aaff, 120, 20);
scene.add(keyLight);

// A material lit only by keyLight, ignoring every other light in the scene
const material = new THREE.MeshStandardNodeMaterial();
material.lightsNode = lights([keyLight]);
```

Under WebGLRenderer the same works for node materials through the compatibility addon: `renderer.setNodesHandler(new WebGLNodesHandler())` with `WebGLNodesHandler` from `three/addons/tsl/WebGLNodesHandler.js`.

## Colored and translucent shadows

With `renderer.shadowMap.transmitted = true`, casters write color and alpha into the shadow map, from `material.castShadowNode` (a vec4) or, without one, from the alpha of `map` and `colorNode`. Setting `castShadowNode` without the flag warns. `material.receivedShadowNode` (an Fn of the shadow value) changes how a receiver applies its shadows and works without the flag. `material.maskShadowNode` discards caster fragments from the shadow pass.

## Shadow notes specific to WebGPU

- `PCFSoftShadowMap` was removed from WebGPURenderer in r186 (warns, uses `PCFShadowMap`).
- Shadows were reworked in r183: bias values tuned for WebGL are often too large here.
- Point shadows under `VSMShadowMap` use the PCF point filter instead of VSM.
