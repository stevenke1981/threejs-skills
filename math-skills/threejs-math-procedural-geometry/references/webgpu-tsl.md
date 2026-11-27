# Procedural Geometry under WebGPURenderer and TSL

Purpose: running the CPU-built geometry under WebGPURenderer, and moving a surface formula into the vertex stage with `positionNode` and `normalNode` when it changes every frame (TSL names and node semantics checked against the r187 source).

Everything in the other reference files builds a plain `BufferGeometry` on the CPU and renders unchanged under WebGPURenderer: import `THREE` from `three/webgpu` and use a node material or a classic material (classic materials are converted to their node versions; `ShaderMaterial` is not). `MarchingCubes` is a `Mesh` with dynamic attributes and works the same way. No GPU marching-cubes path ships in the package.

## A Surface Evaluated in the Vertex Stage

When the surface changes every frame, evaluate the formula on the GPU instead of rebuilding buffers. A flat grid carries `uv()`; `positionNode` evaluates `P(u, v)` from it and `normalNode` the analytic normal. A pulsing torus (the tube radius is uniform over the surface, so the radial normal stays exact):

```javascript
import * as THREE from "three/webgpu";
import { TWO_PI, cos, sin, time, transformNormalToView, uv, vec3 } from "three/tsl";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111827); // the WebGPU canvas is transparent by default
const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 2, 5);
camera.lookAt(0, 0, 0);
scene.add(new THREE.DirectionalLight(0xffffff, 3), new THREE.HemisphereLight(0xffffff, 0x223344, 1));

const a = uv().x.mul(TWO_PI); // around the ring
const b = uv().y.mul(TWO_PI); // around the tube
const radius = sin(time).mul(0.1).add(0.35); // `time` is a uniform the renderer updates each frame
const ring = cos(b).mul(radius).add(1);

const material = new THREE.MeshStandardNodeMaterial({ color: 0x66aaff, roughness: 0.4 });
material.positionNode = vec3(ring.mul(cos(a)), ring.mul(sin(a)), sin(b).mul(radius)); // replaces the position outright
const normalObject = vec3(cos(b).mul(cos(a)), cos(b).mul(sin(a)), sin(b));
material.normalNode = transformNormalToView(normalObject); // normalNode is read as the view-space normal

const mesh = new THREE.Mesh(new THREE.PlaneGeometry(1, 1, 128, 32), material); // the flat grid supplies only uv
mesh.frustumCulled = false; // the bounding sphere describes the flat 1 x 1 plane, not the torus
mesh.rotation.x = -Math.PI / 3;
scene.add(mesh);

renderer.setAnimationLoop(animate);

function animate() {
  renderer.render(scene, camera);
}
```

- `positionNode = vec3(...)` is assigned after morphs, skinning, batching and the instance matrix have been applied to `positionLocal`, so it discards all of them. Build on `positionLocal` (for example `positionLocal.add(offset)`) when those must survive.
- `normalNode` is used directly as `normalView`, so a local-space normal must go through `transformNormalToView()` (local to view, including the model's rotation).
- A radius that varies over the surface needs the normal from the cross product of the partials, written out in TSL. When that is not worth it, use `flatShading: true` on the material.
- The uv grid is closed, so the duplicate seam column evaluates the same formula and shades smoothly.
- Since r187, operands and parameters of TSL math functions are implicitly coerced to a broader common type (Migration Guide 186 → 187); expressions that relied on the old narrower result type may change.
- CPU raycasting and the bounding sphere still see the flat grid; picking such a mesh needs an analytic or CPU-side copy of the surface.
