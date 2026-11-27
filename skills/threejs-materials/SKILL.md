---
name: threejs-materials
description: "Choosing and configuring three.js built-in materials (Basic through Physical, Toon, Matcap, Points, Line) and their node-material equivalents: PBR parameters, transparency, blending, stencil, multi-material groups, envMap response, needsUpdate and disposal. Use for \"which material\", glass or clearcoat, mesh renders black, transparency sorts wrong, thick lines, MeshStandardNodeMaterial. Not for custom GLSL or TSL logic (use threejs-shaders), texture color space or UVs (use threejs-textures), lights and IBL setup (use threejs-lighting). 中文: 材質, 材質選擇, PBR材質, 玻璃材質, 透明排序, 混合模式, 節點材質"
---

# Three.js Materials

Targets three.js r187 (npm `three@0.187.0`). Examples use WebGLRenderer unless they import from `three/webgpu`.

## Quick Start

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 3);

// PBR material - the default choice for lit, realistic surfaces
const material = new THREE.MeshStandardMaterial({ color: 0x00aaff, roughness: 0.35, metalness: 0.1 });
const mesh = new THREE.Mesh(new THREE.SphereGeometry(1, 48, 32), material);
scene.add(mesh);

scene.add(new THREE.HemisphereLight(0xffffff, 0x444444, 1)); // lit materials render black without lights
const light = new THREE.DirectionalLight(0xffffff, 2);
light.position.set(3, 3, 3);
scene.add(light);

renderer.setAnimationLoop(() => {
  mesh.rotation.y += 0.01;
  renderer.render(scene, camera);
});
```

## Choosing a Material

| Material | Use case | Lighting |
| --- | --- | --- |
| MeshBasicMaterial | Unlit color, textures, wireframes, UI | None |
| MeshLambertMaterial | Matte diffuse surfaces, cheapest lit | Diffuse |
| MeshPhongMaterial | Shiny plastic, cheap specular | Diffuse + specular |
| MeshStandardMaterial | PBR metal/rough workflow (glTF default) | PBR |
| MeshPhysicalMaterial | PBR plus clearcoat, transmission, sheen, iridescence, dispersion, rough diffuse | PBR |
| MeshToonMaterial | Cel shading | Toon steps |
| MeshMatcapMaterial | Lit look baked into a matcap texture, no lights needed | Baked |
| ShadowMaterial | Transparent surface that shows only received shadows | Shadows only |
| MeshNormalMaterial / MeshDepthMaterial | Debug views, depth packing | None |
| PointsMaterial | Point clouds | None |
| LineBasicMaterial / LineDashedMaterial | 1 px lines | None |

Decision guide:

1. Needs lights to look right and must match glTF/PBR assets: `MeshStandardMaterial`. Add `MeshPhysicalMaterial` only for the features it adds (each enabled feature costs shader time).
2. Lit but many objects on weak hardware: Lambert, then Phong.
3. Must look identical regardless of lights (UI, sky, video screen): `MeshBasicMaterial`, often with `toneMapped: false`.
4. Custom shading: under WebGLRenderer a ShaderMaterial, under WebGPURenderer a node material with TSL - both are authored in `threejs-shaders`.

Color textures (`map`, `emissiveMap`, `specularMap`, `sheenColorMap`, `specularColorMap`, LDR matcap) need `texture.colorSpace = THREE.SRGBColorSpace`; data maps keep the default. Full rules, UV channels (`texture.channel`) and filtering: `threejs-textures`.

## Properties Every Material Has

```javascript
material.transparent = false; // true for opacity < 1 or blended alpha; sorted back-to-front per object
material.opacity = 1.0;
material.alphaTest = 0; // > 0 discards below the value and keeps the material in the opaque pass
material.side = THREE.FrontSide; // BackSide, DoubleSide
material.depthTest = true;
material.depthWrite = true; // set false on transparent layers that should not occlude each other
material.blending = THREE.NormalBlending; // AdditiveBlending, SubtractiveBlending, MultiplyBlending, CustomBlending
material.polygonOffset = false; // with polygonOffsetFactor/Units, fixes z-fighting decals
material.toneMapped = true; // false keeps exact colors (UI overlays)
material.dithering = false; // true reduces banding in gradients
material.vertexColors = false;
material.visible = true;
```

Stencil, blending details and multi-material geometry groups: `references/blending-stencil-groups.md`.

## Environment Response

- `scene.environment` lights MeshStandardMaterial and MeshPhysicalMaterial on both renderers. WebGLRenderer also applies it to MeshLambertMaterial and MeshPhongMaterial; WebGPURenderer does not (their node versions read only their own `envMap`).
- `material.envMap` overrides `scene.environment` for that material; `material.envMapIntensity` and `material.envMapRotation` affect only that own envMap. When the material falls back to `scene.environment`, WebGLRenderer uploads `scene.environmentIntensity` and `scene.environmentRotation` instead.
- Basic, Lambert and Phong blend their envMap with `combine` (`MultiplyOperation` default, `MixOperation`, `AddOperation`) and `reflectivity`; `specularMap` masks it.

Setting up `scene.environment`, HDR environments and dimming IBL: `threejs-lighting`. Environment texture objects, `mapping` and PMREM internals: `threejs-textures`.

## Runtime Changes, Cloning, Disposal

```javascript
mesh.material.color.set(0xff0000); // uniform values: no recompile
mesh.material.roughness = 0.8;

// Needs material.needsUpdate = true after the first render (compiled into the shader):
// transparent, side, blending, flatShading, vertexColors, dithering, premultipliedAlpha, alphaHash,
// adding or removing a map slot. alphaTest crossing 0 and envMap changes recompile by themselves.
mesh.material.map = newMap;
mesh.material.needsUpdate = true;

const clone = mesh.material.clone(); // copies values; map slots share the same textures
```

`material.dispose()` releases its programs once no other material uses them; textures are not disposed with it - call `texture.dispose()` on each texture nothing else uses.

Performance, in order of impact (strategy lives in `threejs-performance`):

1. Share one material instance across meshes; identical settings already share one compiled program, but switching instances re-uploads uniforms.
2. Prefer opaque or `alphaTest` cutouts over `transparent`: transparent objects are sorted per object, drawn after the opaque pass, overdraw, and misorder when they intersect.
3. Pick the cheapest material that looks right: Basic, Lambert, Phong, Standard, Physical.

## WebGPU and Node Materials (summary)

Under WebGPURenderer the built-in classes above work unchanged - the renderer maps each to its node twin (`MeshStandardMaterial` → `MeshStandardNodeMaterial`, and so on). ShaderMaterial, RawShaderMaterial, MeshDepthMaterial and MeshDistanceMaterial have no twin and log `NodeBuilder: Material "ShaderMaterial" is not compatible.`, then render with a plain fallback node material.

```javascript
import * as THREE from "three/webgpu"; // node materials are exported only here
import { color, texture, float } from "three/tsl";

const unlit = new THREE.MeshBasicNodeMaterial();
unlit.colorNode = color(0xff0066); // unlit: visible without lights

const lit = new THREE.MeshStandardNodeMaterial(); // lit: black without lights or scene.environment
lit.colorNode = texture(map); // map: a loaded sRGB color texture
lit.roughnessNode = float(0.2);
```

Class list, node slots per class, Points as sprites, thick lines, and running node materials on WebGLRenderer through `WebGLNodesHandler`: `references/node-materials.md`. Writing the node logic itself: `threejs-shaders`.

## Common Mistakes

- `new THREE.MeshStandardMaterial()` in a scene with no lights and no `scene.environment` → add lights or an environment, or use `MeshBasicMaterial`: lit materials render black without light.
- `material.envMapIntensity = 0.5;` to dim `scene.environment` → `scene.environmentIntensity = 0.5;`: envMapIntensity scales only the material's own `envMap`.
- `iridescenceThicknessRange: [100, 400]` → `iridescenceThicknessMinimum: 100, iridescenceThicknessMaximum: 400`: the array property is deprecated in r187 and logs a warning (it still forwards both values).
- `material.blending = THREE.MultiplyBlending;` → also set `material.premultipliedAlpha = true;`: Multiply and Subtractive blending require it on both renderers, or the renderer logs an error and sets no blend function.
- `material.stencilMask = 0xff;` → `material.stencilFuncMask = 0xff; material.stencilWriteMask = 0xff;`: `stencilMask` does not exist.
- `new THREE.WebGLRenderer({ antialias: true })` with stencil materials → `new THREE.WebGLRenderer({ antialias: true, stencil: true })`: the stencil buffer is off by default in both renderers, so stencil settings are silently ignored.
- `new THREE.LineBasicMaterial({ linewidth: 5 })` → `new Line2(new LineGeometry().setPositions([...]), new LineMaterial({ linewidth: 5 }))` from `three/addons/lines/`: WebGL and WebGPU draw line primitives 1 px wide; `linewidth` only affects SVGRenderer.
- `new THREE.LineDashedMaterial(...)` drawing solid → call `line.computeLineDistances()` once on the Line.
- `mesh.material.transparent = true;` after the first render with no effect → also set `mesh.material.needsUpdate = true;`.
- Transparent glass panes vanishing behind each other → `depthWrite: false` on the panes, or `alphaTest`/`alphaHash` for cutouts, or `renderOrder` to force the order.
- `import * as THREE from "three"; new THREE.MeshStandardNodeMaterial()` → `import * as THREE from "three/webgpu";`: node materials are exported only by `three/webgpu`.
- `import { texture } from "three/nodes";` → `import { texture } from "three/tsl";`: `three/nodes` does not exist.
- `new THREE.MeshStandardNodeMaterial()` for an unlit effect → `new THREE.MeshBasicNodeMaterial()`.
- `new THREE.PointsMaterial({ size: 5 })` under WebGPURenderer → `new THREE.Sprite(new THREE.PointsNodeMaterial({ positionNode: instancedBufferAttribute(positions), size: 5 }))` with `sprite.count` set: the WebGPU backend draws point primitives at 1 px whatever the size.
- `new THREE.ShaderMaterial(...)` under WebGPURenderer → a node material with TSL (see `threejs-shaders`).

## Reference Files

- `references/material-types.md` - per-class property walkthroughs: Basic, Lambert/Phong, Standard (ORM maps, lightMap, aoMap), Physical (glass, clearcoat, sheen, iridescence, anisotropy, dispersion, retroreflectivity, rough diffuse), Toon, Normal/Depth, Points, Lines. Open when configuring a specific class.
- `references/blending-stencil-groups.md` - blending modes and premultiplied alpha, stencil masks/portals, multi-material groups. Open for masks, additive glow, or per-face materials.
- `references/node-materials.md` - WebGPU node material classes and their slots, `WebGLNodesHandler`, Points as sprites, Line2 under WebGPU. Open when the project imports `three/webgpu`.

## See Also

- `threejs-textures` - color spaces, UV channels, filtering, texture types and render targets
- `threejs-shaders` - GLSL ShaderMaterial, onBeforeCompile and TSL node logic
- `threejs-lighting` - lights, shadows, scene.environment and IBL
- `threejs-performance` - draw calls, overdraw and shader compile stalls
