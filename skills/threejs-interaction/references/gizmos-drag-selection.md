# Gizmos, Dragging and Selection

Purpose: TransformControls gizmos, DragControls, click/hover selection, and rectangle selection with SelectionBox and SelectionHelper (three.js r187).

## TransformControls

A gizmo for moving, rotating and scaling objects. Since r169 the controls object extends `Controls`, not Object3D; the gizmo it renders must be added to the scene separately.

```javascript
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import { TransformControls } from "three/addons/controls/TransformControls.js";

const orbitControls = new OrbitControls(camera, renderer.domElement);

const transformControls = new TransformControls(camera, renderer.domElement);
scene.add(transformControls.getHelper()); // the gizmo; adding the controls themselves only logs an error

transformControls.attach(mesh); // show the gizmo on this object
transformControls.setMode("translate"); // "translate" | "rotate" | "scale"
transformControls.setSpace("local");    // "local" | "world"; scale always acts locally
transformControls.setSize(0.8);
transformControls.setTranslationSnap(0.5);
transformControls.setRotationSnap(THREE.MathUtils.degToRad(15));
transformControls.setScaleSnap(0.25);

transformControls.addEventListener("dragging-changed", (event) => {
  orbitControls.enabled = !event.value; // pause orbiting while the gizmo is dragged
});
transformControls.addEventListener("objectChange", () => {
  console.log(mesh.position); // fires as the gizmo moves the object
});

window.addEventListener("keydown", (event) => {
  if (event.key === "g") transformControls.setMode("translate");
  if (event.key === "r") transformControls.setMode("rotate");
  if (event.key === "s") transformControls.setMode("scale");
  if (event.key === "Escape") transformControls.detach(); // hide the gizmo
});
```

- Events: `change`, `mouseDown`, `mouseUp` (both carry `mode`), `objectChange`, and `<property>-changed` for each observed property such as `dragging-changed`, `mode-changed`, `object-changed`.
- Only the left button (`button === 0`) starts a drag.
- `transformControls.dispose()` disconnects the listeners and disposes the gizmo's geometry and materials.

## DragControls

Drags objects directly: left or middle drag moves the object in a plane parallel to the screen through the object's position, right drag rotates it around the camera's up and right axes (the rotation assumes +Y up).

```javascript
import { DragControls } from "three/addons/controls/DragControls.js";

const dragControls = new DragControls([mesh], camera, renderer.domElement);
dragControls.rotateSpeed = 1; // default
dragControls.recursive = true; // default: children of the listed objects are draggable on their own
dragControls.mouseButtons = { LEFT: THREE.MOUSE.PAN, MIDDLE: THREE.MOUSE.PAN, RIGHT: THREE.MOUSE.ROTATE }; // the default map
// transformGroup = true drags the outermost Group above the hit instead (pass objects = [group]);
// event.object is then that Group, which has no material, so use event.object.material?.emissive in the handlers

dragControls.addEventListener("dragstart", (event) => {
  event.object.material.emissive?.set(0x333333);
});
dragControls.addEventListener("drag", (event) => {
  event.object.position.y = 0; // clamp to the ground mid-drag
});
dragControls.addEventListener("dragend", (event) => {
  event.object.material.emissive?.set(0x000000);
});

// The built-in picker: dragControls.raycaster. The dragged objects: dragControls.objects.
```

- Also fires `hoveron` and `hoveroff`.
- Pair with OrbitControls by toggling `orbitControls.enabled` in `dragstart` / `dragend`.

## Click and hover selection

```javascript
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();
const selectable = [mesh];
let selected = null;

function setPointerFromEvent(event) {
  const rect = renderer.domElement.getBoundingClientRect();
  pointer.set(
    ((event.clientX - rect.left) / rect.width) * 2 - 1,
    -((event.clientY - rect.top) / rect.height) * 2 + 1,
  );
}

function onPointerDown(event) {
  setPointerFromEvent(event);
  raycaster.setFromCamera(pointer, camera);
  const hit = raycaster.intersectObjects(selectable, false)[0];
  if (selected) selected.material.emissive.set(0x000000); // deselect the previous mesh
  selected = hit ? hit.object : null;
  if (selected) selected.material.emissive.set(0x444444);
}

function onPointerMove(event) {
  setPointerFromEvent(event);
  raycaster.setFromCamera(pointer, camera);
  const hit = raycaster.intersectObjects(selectable, false)[0];
  renderer.domElement.style.cursor = hit ? "pointer" : "default"; // hover feedback
}

renderer.domElement.addEventListener("pointerdown", onPointerDown);
renderer.domElement.addEventListener("pointermove", onPointerMove);
// teardown: removeEventListener with the same named functions
```

Changing `material.emissive` affects every mesh sharing that material; clone the material per selectable mesh, or use an outline pass (`threejs-postprocessing`). Raycasting on every `pointermove` over a heavy scene is costly; throttle it to once per frame.

## Box selection

`SelectionBox` selects meshes, lines and points whose bounding-sphere centre falls inside a screen-space rectangle (for InstancedMesh and BatchedMesh it tests each instance's origin); `SelectionHelper` draws the rectangle and tracks the drag.

```javascript
import { SelectionBox } from "three/addons/interactive/SelectionBox.js";
import { SelectionHelper } from "three/addons/interactive/SelectionHelper.js";

const selectionBox = new SelectionBox(camera, scene);
const helper = new SelectionHelper(renderer, "selectBox");
// SelectionHelper appends a div with this class to the canvas's parent and positions it with
// clientX / clientY, so give .selectBox position: fixed, a border and a background in CSS, or nothing shows

function ndc(event) {
  const rect = renderer.domElement.getBoundingClientRect();
  return new THREE.Vector3(
    ((event.clientX - rect.left) / rect.width) * 2 - 1,
    -((event.clientY - rect.top) / rect.height) * 2 + 1,
    0.5,
  );
}

renderer.domElement.addEventListener("pointerdown", (event) => {
  selectionBox.startPoint.copy(ndc(event));
});

renderer.domElement.addEventListener("pointermove", (event) => {
  if (helper.isDown) selectionBox.endPoint.copy(ndc(event));
});

renderer.domElement.addEventListener("pointerup", (event) => {
  selectionBox.endPoint.copy(ndc(event));
  const selected = selectionBox.select(); // meshes; InstancedMesh and BatchedMesh ids go to selectionBox.instances / .batches, keyed by uuid
  console.log(selected);
});

// on teardown: helper.dispose() removes the helper's listeners and its rectangle element
```

- `helper.enabled = false` pauses the rectangle (for example while a camera control is dragging).
- Disable OrbitControls while box-selecting, or both react to the same left drag.
