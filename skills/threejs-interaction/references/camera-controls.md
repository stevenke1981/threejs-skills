# Camera Controls Addons

Purpose: the shared `Controls` lifecycle and per-class setup for every camera controls addon in `three/addons/controls/` (checked against the r187 source).

## Shared lifecycle (`Controls` base class)

- Constructor: `new XControls(camera, domElement = null)`. When `domElement` is given the constructor calls `connect(domElement)` itself. DragControls takes its array of objects first; ArcballControls takes an optional `scene` third.
- `connect(element)` disconnects from a previous element, stores the new one and adds the listeners. Calling it without an element throws a TypeError.
- `disconnect()` removes the listeners; `dispose()` calls `disconnect()` (TransformControls also disposes its gizmo).
- `enabled = false` keeps the listeners but ignores input.
- `controls.object` is the camera being driven; it renders without being in the scene, so `scene.add(controls.object)` is needed only when the camera has children.
- `update(delta)` matters only for OrbitControls/MapControls, FlyControls and FirstPersonControls (pass `timer.getDelta()`); TrackballControls and ArcballControls take no argument; PointerLock, Drag and Transform controls have no per-frame update.

## OrbitControls

```javascript
import { OrbitControls } from "three/addons/controls/OrbitControls.js";

const controls = new OrbitControls(camera, renderer.domElement);

controls.enableDamping = true; // requires update() every frame
controls.dampingFactor = 0.05; // default

controls.minDistance = 2;
controls.maxDistance = 50;
controls.minPolarAngle = 0.1;          // vertical limits, radians
controls.maxPolarAngle = Math.PI / 2;  // stop at the horizon
controls.minAzimuthAngle = -Math.PI / 4;
controls.maxAzimuthAngle = Math.PI / 4;

controls.target.set(0, 1, 0);  // the point orbited around; call update() after changing it
controls.zoomToCursor = true;  // wheel and (since r187, fixed) middle-drag dolly zoom toward the pointer

controls.autoRotate = true;      // requires update() every frame; stops at an azimuth limit
controls.autoRotateSpeed = 2.0;  // default; one full turn per 30 s at 60 fps

const timer = new THREE.Timer();
timer.connect(document);

function animate() {
  timer.update();
  controls.update(timer.getDelta()); // the delta makes autoRotate frame-rate independent
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

- Defaults: left drag rotates, middle drag dollies, right drag pans; left drag with ctrl/meta/shift pans. Touch: one finger rotates, two fingers dolly and pan.
- Events: `change`, `start`, `end`. With on-demand rendering, render on `change` instead of every frame.
- Arrow-key panning needs `controls.listenToKeyEvents(window)`; `stopListenToKeyEvents()` undoes it.
- `controls.saveState()` / `controls.reset()` store and restore target, position and zoom.
- r187 changes: middle-mouse dolly honours `zoomToCursor`; disabling rotate or pan for the one-finger touch action now resets the state instead of leaving it stuck; key interception listens on `domElement.ownerDocument` again (r186 used `getRootNode()`).

## MapControls

```javascript
import { MapControls } from "three/addons/controls/MapControls.js";

// Left drag pans, right drag (or left drag with ctrl/meta/shift) orbits,
// middle button or wheel zooms. screenSpacePanning is false, so panning
// stays parallel to the ground plane (normal = camera.up). Every OrbitControls option applies.
const controls = new MapControls(camera, renderer.domElement);
controls.enableDamping = true;
```

Touch: one finger pans, two fingers dolly and rotate.

## FlyControls

```javascript
import { FlyControls } from "three/addons/controls/FlyControls.js";

const controls = new FlyControls(camera, renderer.domElement);
controls.movementSpeed = 10; // default 1
controls.rollSpeed = Math.PI / 24; // default 0.005
controls.dragToLook = true; // steer only while a mouse button is held
controls.autoForward = false;

function animate() {
  timer.update();
  controls.update(timer.getDelta()); // movement and roll scale by the delta; without it the position becomes NaN
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

WASD move, R/F up and down, Q/E roll, arrow keys pitch and yaw. Shift sets an internal `movementSpeedMultiplier` that `update()` never reads in r187, so it has no effect; scale `movementSpeed` yourself for a sprint key. Key listeners go on `window`.

## FirstPersonControls

Rewritten in r184. Press and drag to look: look speed grows with the drag distance and eases through `dampingFactor` (default 0.1; 1 disables the easing, which also applies to movement). WASD or arrows move in the horizontal plane from the camera's yaw, R and F move along world Y, holding the left button flies forward along the view direction and the right button backward. Touch: one finger flies forward, two fingers backward. Keys listen on `window`.

```javascript
import { FirstPersonControls } from "three/addons/controls/FirstPersonControls.js";

const controls = new FirstPersonControls(camera, renderer.domElement);
controls.movementSpeed = 10;   // default 1
controls.lookSpeed = 0.005;    // default; 0 disables look
controls.dampingFactor = 0.1;  // default
controls.lookVertical = true;  // default
controls.constrainVertical = true; // default false
controls.verticalMin = Math.PI / 4;       // default 0
controls.verticalMax = (Math.PI * 3) / 4; // default Math.PI
controls.lookAt(0, 0, 0); // use this rather than camera.lookAt, so the controls resync their angles

function animate() {
  timer.update();
  controls.update(timer.getDelta()); // the delta is required; without it the position becomes NaN
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

There is no `activeLook` property, and `handleResize()` only logs that it was removed.

## PointerLockControls

```javascript
import { PointerLockControls } from "three/addons/controls/PointerLockControls.js";

const controls = new PointerLockControls(camera, document.body);
controls.pointerSpeed = 1.0; // mouse sensitivity (default)
controls.minPolarAngle = 0; // default; clamp vertical look with these two
controls.maxPolarAngle = Math.PI;

controls.addEventListener("lock", () => console.log("locked"));
controls.addEventListener("unlock", () => console.log("unlocked"));
document.body.addEventListener("click", () => controls.lock()); // needs a user gesture; lock(true) asks for unadjusted movement

const velocity = new THREE.Vector3();
let moveForward = false; // let: the key handlers reassign these
let moveBackward = false;

document.addEventListener("keydown", (event) => {
  if (event.code === "KeyW") moveForward = true;
  if (event.code === "KeyS") moveBackward = true;
});
document.addEventListener("keyup", (event) => {
  if (event.code === "KeyW") moveForward = false;
  if (event.code === "KeyS") moveBackward = false;
});

const timer = new THREE.Timer();
timer.connect(document);

function animate() {
  timer.update();
  const delta = timer.getDelta();
  if (controls.isLocked) {
    // ease toward the target speed, then move the camera forward parallel to the ground (xz-plane)
    velocity.z = THREE.MathUtils.damp(
      velocity.z,
      10 * (Number(moveForward) - Number(moveBackward)),
      10,
      delta,
    );
    controls.moveForward(velocity.z * delta);
    // controls.moveRight(distance) does the same along the camera's right axis
  }
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

The controls only rotate the camera; movement, collision and gravity are yours (`fps-shooter`, `physics-tuning` for the design side). Events: `change`, `lock`, `unlock`.

## TrackballControls

```javascript
import { TrackballControls } from "three/addons/controls/TrackballControls.js";

const controls = new TrackballControls(camera, renderer.domElement);
controls.rotateSpeed = 2.0; // default 1
controls.zoomSpeed = 1.2;   // default
controls.panSpeed = 0.8;    // default 0.3
controls.staticMoving = true; // no inertia; otherwise dynamicDampingFactor (default 0.2) applies

window.addEventListener("resize", () => controls.handleResize()); // re-reads the element's screen rect

function animate() {
  controls.update(); // no delta argument; must run every frame
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

The constructor calls `handleResize()` only when it receives the element; after a late `connect(element)` call `handleResize()` yourself. Default buttons: left rotate, middle zoom, right pan; holding A, S or D switches to rotate, zoom or pan.

## ArcballControls

```javascript
import { ArcballControls } from "three/addons/controls/ArcballControls.js";

const controls = new ArcballControls(camera, renderer.domElement, scene); // scene is needed to show the gizmos
controls.setGizmosVisible(true);
controls.cursorZoom = true;       // default false: zoom toward the cursor
controls.adjustNearFar = true;    // default false: adjust near/far on zoom to keep the initial visible range (perspective only)
controls.enableAnimations = true; // default: inertia and focus animations, driven by its own requestAnimationFrame
controls.addEventListener("change", () => renderer.render(scene, camera)); // or render in your loop
```

Events: `change`, `start`, `end`. Without `scene`, the gizmos cannot be shown.
