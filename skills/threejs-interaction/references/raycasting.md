# Raycasting and Coordinate Conversion

Purpose: what a Raycaster intersection contains, how to configure range, precision and layers, dragging on a plane, and converting between screen and world coordinates (three.js r187).

## What an intersection contains

```javascript
const raycaster = new THREE.Raycaster();
const clickables = [mesh]; // the smallest list you can: the meshes, not scene.children
const hits = raycaster.intersectObjects(clickables, false); // sorted, closest first
// Each hit:
// {
//   distance: number,                          // from the ray origin, world units
//   point: THREE.Vector3,                      // hit point in world space
//   face: { a, b, c, normal, materialIndex },  // plain object; a, b, c are vertex indices; normal in object space
//   faceIndex: number,
//   object: THREE.Object3D,
//   uv: THREE.Vector2,                         // undefined when the geometry has no uv
//   uv1: THREE.Vector2,                        // second UV set, when the geometry has one
//   normal: THREE.Vector3,                     // interpolated vertex normal in object space, flipped to face the ray
//   barycoord: THREE.Vector3,                  // barycentric coordinates of the hit point
//   instanceId: number,                        // set on InstancedMesh hits
//   batchId: number,                           // set on BatchedMesh hits
//   distanceToRay: number                      // set on Points hits
// }
```

- `intersectObject(object, recursive = true)` and `intersectObjects(objects, recursive = true)`: recursion is on by default; pass `false` when the list already holds the meshes.
- Faces pointing away from the ray origin are not hit; set `material.side = THREE.DoubleSide` to catch both sides. A ray that grazes an edge shared by two faces can produce two coincident hits.
- Invisible meshes are still raycast: the Raycaster checks `object.layers`, not `visible`.
- To turn the object-space `face.normal` into world space: `hit.face.normal.clone().transformDirection(hit.object.matrixWorld)`.

## Raycaster configuration

```javascript
const raycaster = new THREE.Raycaster();

// From any origin and direction (normalize the direction first)
raycaster.set(new THREE.Vector3(0, 2, 0), new THREE.Vector3(1, 0, 0).normalize());

// Range: hits closer than near or farther than far are discarded
raycaster.near = 0.1;
raycaster.far = 10;

// Precision for Points and Line, in world units (both default to 1)
raycaster.params.Points.threshold = 0.2;
raycaster.params.Line.threshold = 0.2;

// Restrict picking to layer 1. Objects join with enable; set would also remove
// them from layer 0, where the default camera looks, so use set only on a pick-only proxy.
mesh.layers.enable(1); // stays on layer 0 too, so the camera still renders the mesh
raycaster.layers.set(1); // tests layer 1 only; raycaster.layers.enable(1) would test layers 0 and 1
```

`raycaster.setFromCamera(ndc, camera)` works for perspective and orthographic cameras.

## Dragging on a plane

```javascript
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();
const groundPlane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0); // normal, constant
const hit = new THREE.Vector3();

renderer.domElement.addEventListener("pointermove", (event) => {
  const rect = renderer.domElement.getBoundingClientRect();
  pointer.set(
    ((event.clientX - rect.left) / rect.width) * 2 - 1,
    -((event.clientY - rect.top) / rect.height) * 2 + 1,
  );
  raycaster.setFromCamera(pointer, camera);
  // null when the ray is parallel to the plane or points away from it
  if (event.buttons !== 0 && raycaster.ray.intersectPlane(groundPlane, hit) !== null) {
    mesh.position.copy(hit); // while a button is held, drag the mesh along the ground
  }
});
```

Disable camera controls while dragging (`controls.enabled = false` on pointerdown over the object, `true` on pointerup), or the orbit and the drag fight.

## Screen and world coordinates

```javascript
// World to screen: project, then map NDC (-1..1) to CSS pixels of the canvas
const rect = renderer.domElement.getBoundingClientRect();
const ndc = mesh.getWorldPosition(new THREE.Vector3()).project(camera);
const screenX = ((ndc.x + 1) / 2) * rect.width + rect.left;
const screenY = ((-ndc.y + 1) / 2) * rect.height + rect.top;
// ndc.z outside [-1, 1] means the point is outside the near-to-far range (behind the camera included)
```

```javascript
// Screen to world: unproject into a ray, then walk it to the plane z = 0
const worldPoint = new THREE.Vector3(0.1, 0.2, 0.5).unproject(camera); // NDC x, y; z between -1 and 1
const direction = worldPoint.sub(camera.position).normalize();
const distance = -camera.position.z / direction.z; // the ray misses the plane when direction.z is 0
const world = camera.position.clone().addScaledVector(direction, distance);
// The same result comes from a Raycaster built with setFromCamera and ray.intersectPlane
```

This unproject-and-subtract form assumes a perspective camera; for an orthographic camera use `raycaster.setFromCamera`, which builds the correct parallel ray.
