# Shadows: Types, Tuning and Tricks

Purpose: shadow map types, filtering, directional/spot/point shadow cameras, acne and bias, caster filtering by layers, static shadows, contact shadows. Targets three.js r187.

## The three switches and the map types

```javascript
// 1. Renderer: shadow maps are off by default
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap; // the default

// 2. Light
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
dirLight.position.set(5, 10, 5);
dirLight.castShadow = true;
scene.add(dirLight);

// 3. Objects
mesh.castShadow = true;
mesh.receiveShadow = true;
```

| `renderer.shadowMap.type` | Notes |
| --- | --- |
| `BasicShadowMap` | unfiltered, hard edges, fastest; `shadow.radius` has no effect |
| `PCFShadowMap` (default) | percentage-closer filtering, soft; `shadow.radius` widens the filter |
| `VSMShadowMap` | variance shadow maps, blurred map; `shadow.radius` and `shadow.blurSamples` (default 8) control the blur; every receiving mesh also casts; not supported for PointLight |
| `PCFSoftShadowMap` | deprecated: WebGLRenderer since r182, removed from WebGPURenderer in r186. In r187 both renderers warn "has been removed" and fall back to `PCFShadowMap`, which is already soft |

With `VSMShadowMap`, a PointLight shadow is skipped with a warning under WebGLRenderer; WebGPURenderer falls back to its PCF point-shadow filter for that light.

## Tuning a directional shadow

```javascript
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
dirLight.position.set(5, 10, 5); // every caster must lie between near and far from here
dirLight.castShadow = true;

// The shadow camera sits at the light's position and looks at its target: near and far
// are distances from the light; left/right/top/bottom frame the area around the target
const shadowCam = dirLight.shadow.camera;
shadowCam.left = -10;
shadowCam.right = 10;
shadowCam.top = 10;
shadowCam.bottom = -10;
shadowCam.near = 0.5;
shadowCam.far = 50;
shadowCam.updateProjectionMatrix(); // needed whenever these change after the first render

dirLight.shadow.mapSize.set(2048, 2048); // default 512 × 512; powers of two

// Acne: normalBias offsets the lookup along the surface normal (light grazing a surface);
// bias offsets the depth compared against the map — start tiny, negative fights acne
dirLight.shadow.normalBias = 0.02;
dirLight.shadow.bias = -0.0001;

// Lighten shadows without adding fill lights: 0 removes, 1 (default) is full darkness
dirLight.shadow.intensity = 0.7;
```

- Shadow resolution per world unit is `mapSize / (right − left)`: halving the frustum width doubles the sharpness. Fit the frustum to what the camera sees, or use SunLight (see `light-types.md`).
- Acne (stripes on lit surfaces): raise `normalBias` first, then a small negative `bias`. Peter-panning (shadow detached from the caster): too much bias; reduce it.
- WebGPURenderer's shadows were reworked in r183 and usually need less bias than WebGL: remove or reduce bias values carried over.
- Visualize the frustum with `scene.add(new THREE.CameraHelper(dirLight.shadow.camera))` and call the helper's `update()` after changing the camera.

## Point and spot shadow cameras are derived

The renderer rebuilds these cameras on every shadow pass: a spot light's shadow fov is `2 × angle × shadow.focus` (in degrees) and its far is `light.distance || shadow.camera.far`; a point light's far is also `light.distance || shadow.camera.far`. The fov is always overwritten, and a manual `far` survives only while `distance` is 0. Control the cone with `angle` and `shadow.focus` (default 1), the range with `distance`.

```javascript
const deskLamp = new THREE.SpotLight(0xffffff, 250, 30);
deskLamp.castShadow = true;
deskLamp.shadow.mapSize.set(1024, 1024);
deskLamp.shadow.focus = 0.9; // shadow fov as a fraction of the light's cone
deskLamp.shadow.bias = -0.0001;
```

A point light renders six shadow faces per update: it is the most expensive shadow caster.

## Choosing casters with layers

Casters are filtered by `shadow.camera.layers` as soon as it enables any layer other than 0; otherwise the view camera's layers are used (WebGPURenderer already did this; WebGLRenderer since r187). To keep an object visible but out of one light's shadow map:

```javascript
dirLight.shadow.camera.layers.enable(1); // this light now draws only casters on layer 0 or 1
dirLight.shadow.camera.layers.disable(0); // ...and with this, only layer 1
heroMesh.layers.enable(1); // casts into this light's map; still rendered by a camera on layer 0
```

Lights themselves are not filtered per mesh by layers (see SKILL.md); per-material light selection is `material.lightsNode` (see `webgpu-tsl.md`).

## Static shadows

Render a light's shadow on demand: `light.shadow.autoUpdate = false; light.shadow.needsUpdate = true;` and set `light.shadow.needsUpdate = true` again after something moves. This works on both renderers. WebGLRenderer also has the global `renderer.shadowMap.autoUpdate` / `renderer.shadowMap.needsUpdate`; WebGPURenderer's `shadowMap` has only `enabled`, `transmitted` and `type`.

## Colored and translucent shadows

WebGPURenderer only: see `webgpu-tsl.md` (`renderer.shadowMap.transmitted`, `castShadowNode`, `receivedShadowNode`).

## Contact shadows (fake, cheap)

A shadow map per light is overkill for a grounded look; a radial-gradient blob costs one textured quad.

```javascript
// A soft blob under the object: no shadow map, works under both renderers
const size = 128;
const canvas = document.createElement("canvas");
canvas.width = canvas.height = size;
const ctx = canvas.getContext("2d");
const gradient = ctx.createRadialGradient(size / 2, size / 2, 0, size / 2, size / 2, size / 2);
gradient.addColorStop(0, "rgba(0,0,0,0.6)");
gradient.addColorStop(1, "rgba(0,0,0,0)");
ctx.fillStyle = gradient;
ctx.fillRect(0, 0, size, size);

const shadowTexture = new THREE.CanvasTexture(canvas);
shadowTexture.colorSpace = THREE.SRGBColorSpace; // a canvas used as a color map is sRGB

const blob = new THREE.Mesh(
  new THREE.PlaneGeometry(2, 2),
  new THREE.MeshBasicMaterial({ map: shadowTexture, transparent: true, depthWrite: false })
);
blob.rotation.x = -Math.PI / 2;
blob.position.y = 0.001; // just above the floor to avoid z-fighting
scene.add(blob);
```

For a blob that follows the object's silhouette, render depth from below into a render target and blur it — the official `webgl_shadow_contact` example does this with `HorizontalBlurShader` and `VerticalBlurShader` from `three/addons/shaders/`. three.js has no `ContactShadows` class; that is a drei component (`threejs-react-three-fiber`).
