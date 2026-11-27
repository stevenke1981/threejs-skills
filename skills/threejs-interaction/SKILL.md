---
name: threejs-interaction
description: "Three.js r187 interaction: raycasting and picking, hover and click on 3D objects, camera controls addons (Orbit, Map, Fly, FirstPerson, PointerLock, Trackball, Arcball), TransformControls gizmos, DragControls, box selection and screen/world conversion, same on WebGL and WebGPU. Use for \"click a mesh\", raycast misses, OrbitControls setup, gizmo not showing, drag objects, pointer lock FPS look. Not for camera feel or follow design (use camera-systems), outline highlights (use threejs-postprocessing), React pointer props (use threejs-react-three-fiber). 中文: 射線拾取, 點選物件, 滑鼠互動, 軌道控制, 拖曳物件, 框選, 變換控制器"
---

# Three.js Interaction

Targets three.js r187 (`three@0.187.0`). Controls and Raycaster are renderer-neutral, so everything here runs unchanged under WebGLRenderer and WebGPURenderer. Scene, renderer and loop setup: `threejs-fundamentals`.

## Quick Start

```javascript
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111318);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 1.5, 5);

const mesh = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0x3a86ff }),
);
scene.add(mesh);
const light = new THREE.DirectionalLight(0xffffff, 2);
light.position.set(3, 5, 4);
scene.add(light, new THREE.AmbientLight(0xffffff, 0.4));

const controls = new OrbitControls(camera, renderer.domElement); // connects itself
controls.enableDamping = true; // requires controls.update() every frame

const timer = new THREE.Timer();
timer.connect(document); // freezes the delta while the tab is hidden

const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();

function setPointer(event) {
  // NDC relative to the canvas, so the page can scroll or embed it
  const rect = renderer.domElement.getBoundingClientRect();
  pointer.set(
    ((event.clientX - rect.left) / rect.width) * 2 - 1,
    -((event.clientY - rect.top) / rect.height) * 2 + 1,
  );
}

renderer.domElement.addEventListener("pointerdown", (event) => {
  setPointer(event);
  raycaster.setFromCamera(pointer, camera);
  const hits = raycaster.intersectObject(mesh); // recursive by default, as a loaded model needs
  if (hits.length > 0) console.log("clicked", hits[0].object);
});

function animate() {
  timer.update();
  controls.update(timer.getDelta());
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

Pointer events (`pointerdown`, `pointermove`, `pointerup`) cover mouse, touch and pen with one listener set.

## Picking Rules

- Compute NDC from `getBoundingClientRect()` of the canvas, not from `window.innerWidth`, unless the canvas is exactly full-window.
- Pass the smallest list you can (`intersectObjects(clickables, false)`), not `scene.children` recursively; hits come back sorted, closest first.
- Back faces are not hit unless `material.side = THREE.DoubleSide`. Invisible objects are still hit: the Raycaster tests layers, not `visible`, so an invisible low-poly proxy can stand in for a heavy mesh.
- The Raycaster tests CPU-side geometry. GPU displacement (`positionNode`, vertex-shader morphing, skinning done in a TSL node) is invisible to it; pick against an undisplaced proxy.
- Points and Line picking use `raycaster.params.Points.threshold` / `Line.threshold`, both 1 world unit by default, which is often far too coarse.
- Intersection fields (`point`, `face`, `uv`, `normal`, `instanceId`, `batchId`, ...), layers, plane dragging and screen/world conversion: `references/raycasting.md`.

## Choosing Camera Controls

| Need | Class | `update()` |
| --- | --- | --- |
| Inspect an object, orbit a target | `OrbitControls` | every frame with damping or autoRotate; pass the delta |
| Map, RTS or top-down pan | `MapControls` (OrbitControls subclass) | as OrbitControls |
| Free rotation without a fixed up axis | `TrackballControls` | every frame, no arguments |
| Gizmo-driven orbit for viewers | `ArcballControls` | no arguments |
| Free flight with roll | `FlyControls` | every frame with the delta (required) |
| Drag-to-look walk/fly | `FirstPersonControls` | every frame with the delta (required) |
| Mouse-look FPS | `PointerLockControls` | none; move the camera yourself |

All extend `Controls`: `new XControls(camera, domElement)` wires the listeners when the element is given; `connect(element)` attaches later, `disconnect()` detaches, `dispose()` calls `disconnect()`, and `enabled = false` ignores input. Per-class code and options: `references/camera-controls.md`. Camera design questions (follow, deadzone, collision, feel) belong to `camera-systems`.

## Gizmos, Dragging and Selection

- `TransformControls` is not an Object3D: `scene.add(transformControls.getHelper())`, then `attach(mesh)`. Pause orbiting on `dragging-changed`.
- `DragControls(objects, camera, domElement)` drags in a camera-facing plane; right drag rotates.
- `SelectionBox` + `SelectionHelper` select by screen rectangle; the helper's div needs your CSS.
- Code for all three plus a hover/click selection pattern: `references/gizmos-drag-selection.md`. Glow or outline on the selected object: `threejs-postprocessing`.

## Teardown

Keep named handler references so you can `removeEventListener` them; call `dispose()` on every controls instance (it removes their listeners and resets the `touch-action` style the pointer-driven ones set), `helper.dispose()` on a SelectionHelper, and `timer.dispose()`.

## Common Mistakes

- `scene.add(transformControls)` → `scene.add(transformControls.getHelper())`: TransformControls extends Controls (an EventDispatcher) since r169, so `add` logs "object not an instance of THREE.Object3D" and no gizmo appears.
- `new OrbitControls(camera)` then nothing → `new OrbitControls(camera, renderer.domElement)` or a later `controls.connect(renderer.domElement)`: without an element no listeners exist; `connect()` without an argument throws a TypeError.
- `controls.update(clock.getDelta())` → `timer.update(); controls.update(timer.getDelta())`: Clock is deprecated since r183.
- `flyControls.update()` / `firstPersonControls.update()` → pass `timer.getDelta()`: both multiply by the delta, so without it the camera position becomes NaN.
- `orbitControls.update()` with autoRotate → `orbitControls.update(timer.getDelta())`: without a delta autoRotate assumes 60 fps and its speed follows the refresh rate.
- Damping on, no per-frame `update()` → call `controls.update()` in the loop: damped motion stalls the moment input stops.
- `import ... from "three/examples/jsm/controls/OrbitControls.js"` → `"three/addons/controls/OrbitControls.js"`: the documented import maps define only `three/addons/`.
- `requestAnimationFrame(animate)` → `renderer.setAnimationLoop(animate)`: both renderers' docs advise it, and it hands frames to a WebXR session.
- Mouse coordinates from `window.innerWidth` on a canvas that is not full-window → use the canvas rect (`setPointer` above): picks land offset.
- `mesh.layers.set(1)` → `mesh.layers.enable(1)`: `set` removes the mesh from layer 0, so the default camera stops rendering it.
- `hit.face instanceof THREE.Face3` → `hit.face.normal`: the intersection's face is a plain `{ a, b, c, normal, materialIndex }` object; Face3 no longer exists.
- `const moveForward = false` in a key handler → `let moveForward = false`: reassigning a const throws a TypeError.
- `controls.getObject()` → `controls.object`: PointerLockControls.getObject() was removed in r168.
- `dragControls.getObjects()` / `getRaycaster()` → `dragControls.objects` / `dragControls.raycaster`: removed in r168.
- `dragControls.activate()` / `deactivate()` → `connect(renderer.domElement)` / `disconnect()`: renamed in r168.
- `firstPersonControls.activeLook = false` → `firstPersonControls.lookSpeed = 0`: FirstPersonControls (rewritten in r184) has no activeLook property.
- `firstPersonControls.handleResize()` → delete the call: it was removed and only logs a warning; TrackballControls still has `handleResize()`.

## Reference Files

- `references/raycasting.md` — intersection object fields, Raycaster configuration and layers, dragging on a plane, world-to-screen and screen-to-world conversion. Open when a pick returns the wrong thing or you need hit data.
- `references/camera-controls.md` — Controls base API, then OrbitControls, MapControls, FlyControls, FirstPersonControls, PointerLockControls, TrackballControls and ArcballControls with their options and r187 behaviour. Open when setting up or tuning any controls addon.
- `references/gizmos-drag-selection.md` — TransformControls, DragControls, click/hover selection, SelectionBox and SelectionHelper. Open for editor-style manipulation or multi-select.

## See Also

- `threejs-fundamentals` — scene, camera, renderer and the animation loop.
- `threejs-math-motion` — damping, springs, camera rails and shake for custom camera motion.
- `threejs-postprocessing` — outline and bloom passes for selection highlights.
- `threejs-loaders` — loading the glTF models you make interactive.
- `threejs-geometry` — InstancedMesh and BatchedMesh, whose picks report `instanceId` / `batchId`.
- `threejs-react-three-fiber` — pointer-event props and drei controls when the scene is written in React.
- `camera-systems`, `input-systems` — engine-neutral camera design and input mapping.
