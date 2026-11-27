# Geometry under WebGPURenderer and TSL

Purpose: what changes for geometry, points, lines and instancing when rendering with WebGPURenderer, and how per-instance data becomes TSL nodes (checked against the r187 source). Renderer setup itself is in `threejs-fundamentals`; custom node logic in `threejs-shaders`.

The geometry classes, `InstancedMesh` and `BatchedMesh` keep the API of the WebGL pages under WebGPURenderer. What differs:

- The WebGPU backend draws `Points` at one pixel whatever the material size; use an instanced `Sprite` with `PointsNodeMaterial` instead.
- `LineLoop` logs an error and is skipped (both backends of WebGPURenderer).
- `BatchedMesh`: one draw per run of consecutive instances sharing a geometry (merged into an instanced draw since r187), not a multi-draw.
- `ShaderMaterial` and `RawShaderMaterial` are not translated: the node builder logs `Material "ShaderMaterial" is not compatible` and falls back to a plain `NodeMaterial`. Per-instance data is a TSL node.
- Thick lines use `three/addons/lines/webgpu/Line2.js` with `Line2NodeMaterial` (see points-lines.md).

## Setup

```javascript
import * as THREE from "three/webgpu";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111111); // the WebGPU canvas is transparent by default (alpha: true)
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 5);
renderer.setAnimationLoop(animate);

function animate() {
  renderer.render(scene, camera);
}
```

## Sized Points as an Instanced Sprite

```javascript
import * as THREE from "three/webgpu";
import { instancedBufferAttribute } from "three/tsl";

const count = 1000;
const positions = new Float32Array(count * 3).map(() => (Math.random() - 0.5) * 10);
const material = new THREE.PointsNodeMaterial({
  positionNode: instancedBufferAttribute(new THREE.InstancedBufferAttribute(positions, 3)),
  size: 0.5, // depth-scaled like PointsMaterial; CSS pixels with sizeAttenuation: false; sizeNode for a node
});
const points = new THREE.Sprite(material);
points.count = count; // how many instances the sprite draws (WebGPURenderer only)
points.frustumCulled = false; // culling tests one ~0.71-unit sphere at the sprite origin, not the positions
scene.add(points);
```

For points written by a compute shader, `instancedArray(count, "vec3")` plus `positionNode = buffer.toAttribute()` replaces the attribute; simulation itself is in `threejs-math-particles-fields`.

## Per-Instance Attributes as Nodes

Build displacement on `positionLocal`, which at that point already carries morph targets, skinning, displacement maps, batching and the instance matrix (`NodeMaterial.setupPosition` applies them before `positionNode`):

```javascript
import * as THREE from "three/webgpu";
import { instancedBufferAttribute, positionLocal } from "three/tsl";

const count = 1000;
const jitter = new THREE.InstancedBufferAttribute(new Float32Array(count * 3).map(() => Math.random() - 0.5), 3);
const material = new THREE.MeshBasicNodeMaterial();
material.positionNode = positionLocal.add(instancedBufferAttribute(jitter).mul(0.5)); // per-instance data, no GLSL
const mesh = new THREE.InstancedMesh(new THREE.BoxGeometry(0.2, 0.2, 0.2), material, count);
const matrix = new THREE.Matrix4();
for (let i = 0; i < count; i++) {
  mesh.setMatrixAt(i, matrix.setPosition(Math.random() * 8 - 4, Math.random() * 8 - 4, Math.random() * 8 - 4));
}
scene.add(mesh);
```

- `positionGeometry` is the raw `position` attribute before any of those transforms; use it only when you want to ignore them.
- Assigning `positionNode = vec3(...)` without `positionLocal` replaces the transformed position outright, instance matrix included.
- `instancedBufferAttribute(attr)` steps once per instance; `bufferAttribute(attr)` steps once per vertex.
