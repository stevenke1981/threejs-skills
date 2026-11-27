# Node materials (WebGPURenderer)

Purpose: which node material class replaces which built-in material, which node slots each one has, and the WebGPU-specific material gotchas (r187); open when the project imports `three/webgpu`. Writing the node graphs themselves (Fn, uniforms, varyings) is in `threejs-shaders`.

## Setup

```javascript
import * as THREE from "three/webgpu"; // renderer, core classes and node materials
import { texture, mix, color, float, time, positionLocal, normalLocal, sin } from "three/tsl"; // TSL functions

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before sync calls such as compile() or hasFeature(); setAnimationLoop also initializes
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111111); // the WebGPU canvas is transparent by default
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 3);

renderer.setAnimationLoop(() => renderer.render(scene, camera));
```

Renderer creation and the WebGL 2 fallback belong to `threejs-fundamentals`.

## Class mapping

Built-in materials are converted automatically; create the node class yourself only when you want node slots.

| Built-in | Node class (from `three/webgpu`) |
| --- | --- |
| MeshBasicMaterial | MeshBasicNodeMaterial |
| MeshLambertMaterial | MeshLambertNodeMaterial |
| MeshPhongMaterial | MeshPhongNodeMaterial |
| MeshStandardMaterial | MeshStandardNodeMaterial |
| MeshPhysicalMaterial | MeshPhysicalNodeMaterial |
| MeshToonMaterial | MeshToonNodeMaterial |
| MeshMatcapMaterial | MeshMatcapNodeMaterial |
| MeshNormalMaterial | MeshNormalNodeMaterial |
| PointsMaterial | PointsNodeMaterial |
| SpriteMaterial | SpriteNodeMaterial |
| LineBasicMaterial / LineDashedMaterial | LineBasicNodeMaterial / LineDashedNodeMaterial |
| ShadowMaterial | ShadowNodeMaterial |
| LineMaterial (addon) | Line2NodeMaterial |
| ShaderMaterial, RawShaderMaterial, MeshDepthMaterial, MeshDistanceMaterial | none - logs `NodeBuilder: Material "..." is not compatible.` and renders with a plain `NodeMaterial` fallback |

Node-only classes: `MeshSSSNodeMaterial` (subsurface scattering), `VolumeNodeMaterial` (ray-marched volumes), and the base `NodeMaterial`.

## Node slots

All node materials (base `NodeMaterial`): `colorNode`, `opacityNode`, `normalNode`, `positionNode`, `alphaTestNode`, `maskNode`, `depthNode`, `outputNode`, `fragmentNode` (replaces the whole fragment output), `vertexNode`, `envNode`, `aoNode`, `lightsNode`, `backdropNode`, `backdropAlphaNode`, `castShadowNode`, `receivedShadowNode`, `mrtNode`, `geometryNode`, `contextNode`, plus the shadow-position hooks `castShadowPositionNode`, `receivedShadowPositionNode` and `maskShadowNode`.

Lit materials add:

- `emissiveNode` - MeshStandardNodeMaterial and MeshPhysicalNodeMaterial; since r187 also MeshLambertNodeMaterial, MeshPhongNodeMaterial and MeshToonNodeMaterial. MeshBasicNodeMaterial has none (it is unlit; brighten `colorNode` instead).
- MeshStandardNodeMaterial: `roughnessNode`, `metalnessNode`.
- MeshPhysicalNodeMaterial: `clearcoatNode`, `clearcoatRoughnessNode`, `clearcoatNormalNode`, `sheenNode`, `sheenRoughnessNode`, `iridescenceNode`, `iridescenceIORNode`, `iridescenceThicknessNode`, `specularIntensityNode`, `specularColorNode`, `iorNode`, `transmissionNode`, `thicknessNode`, `attenuationDistanceNode`, `attenuationColorNode`, `dispersionNode`, `anisotropyNode`, `retroreflectivityNode`, `diffuseRoughnessNode`.
- SpriteNodeMaterial: `rotationNode`, `scaleNode`; PointsNodeMaterial (which extends it) adds `sizeNode`.

The classic properties (`color`, `map`, `roughness`, `normalMap`, `envMap`, `transparent`, `side` ...) still work on node materials; a slot, when set, replaces the value the property would produce.

```javascript
// Unlit: the node equivalent of a color ShaderMaterial
const unlit = new THREE.MeshBasicNodeMaterial();
unlit.colorNode = mix(color(0x0066ff), color(0xff0066), sin(time.mul(0.5)).mul(0.5).add(0.5));

// Lit: node slots replace the fixed properties; needs lights or scene.environment
const map = await new THREE.TextureLoader().loadAsync("color.jpg");
map.colorSpace = THREE.SRGBColorSpace;
const lit = new THREE.MeshStandardNodeMaterial();
lit.colorNode = texture(map);
lit.roughnessNode = float(0.2);

// Vertex displacement on positionLocal, which already carries skinning, morphs and instancing
const shell = new THREE.MeshStandardNodeMaterial();
shell.positionNode = positionLocal.add(normalLocal.mul(sin(positionLocal.y.mul(6))).mul(0.05));
```

## Environment differences

- MeshStandardNodeMaterial and MeshPhysicalNodeMaterial use `material.envMap`, else `scene.environment`.
- MeshLambertNodeMaterial and MeshPhongNodeMaterial use only their own `envMap` - unlike WebGLRenderer, `scene.environment` does not light them.

## Points larger than 1 px

The WebGPU backend draws point primitives at 1 px whatever `size` says. Draw larger points as instanced sprite quads:

```javascript
import * as THREE from "three/webgpu";
import { instancedBufferAttribute } from "three/tsl";

const positions = new THREE.InstancedBufferAttribute(new Float32Array(1500), 3); // fill with x, y, z per point
const points = new THREE.Sprite(
  new THREE.PointsNodeMaterial({ positionNode: instancedBufferAttribute(positions), size: 4, sizeAttenuation: false }), // 4 px squares
);
points.count = 500; // one quad per point
points.frustumCulled = false; // culling would test only the sprite's own small bounds at its origin
scene.add(points);
```

## Thick lines

`LineMaterial` from `three/addons/lines/` is a ShaderMaterial; under WebGPURenderer import `Line2` from `three/addons/lines/webgpu/Line2.js` (also `LineSegments2.js`, `Wireframe.js` in the same folder), which defaults to `Line2NodeMaterial` from `three/webgpu`. The geometry is still `LineGeometry` from `three/addons/lines/LineGeometry.js`.

## Node materials on WebGLRenderer

WebGLRenderer can draw node materials through the `WebGLNodesHandler` addon, a migration aid:

```javascript
import * as THREE from "three";
import { MeshStandardNodeMaterial } from "three/webgpu";
import { WebGLNodesHandler } from "three/addons/tsl/WebGLNodesHandler.js";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setNodesHandler(new WebGLNodesHandler());
```

Documented limitations in the r187 source: no VSM shadows, no MRT, no transmission, no WebGPU post-processing stack, no storage textures, fog and environment changes need a material `dispose()` to update, and instanced mesh geometry cannot be shared.
