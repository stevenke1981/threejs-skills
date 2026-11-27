# Object3D, Hierarchy and Core Math

Purpose: Object3D transforms and hierarchy, Group and multi-material Mesh, matrix updates, culling bounds, and cheat sheets for Vector3, Matrix4, Quaternion, Euler, Color and MathUtils (three.js r187).

## Object3D

Base class of Mesh, Group, Light and Camera.

```javascript
const obj = new THREE.Object3D();
const child = new THREE.Object3D();

obj.position.set(1, 2, 3); // translation
obj.rotation.set(0, Math.PI / 4, 0); // Euler angles in radians; kept in sync with quaternion
obj.quaternion.setFromAxisAngle(new THREE.Vector3(0, 1, 0), Math.PI / 4); // the same rotation as a quaternion
obj.scale.set(1, 1, 1);
obj.pivot = new THREE.Vector3(0, -0.5, 0); // optional: rotate and scale around this local point (default null)

obj.getWorldPosition(new THREE.Vector3()); // world-space queries take a target vector to write into
obj.getWorldQuaternion(new THREE.Quaternion());
obj.getWorldDirection(new THREE.Vector3()); // the +Z axis in world space; cameras return their view direction (−Z)

obj.add(child); // reparents child; also removes it from its previous parent
obj.attach(child); // reparents and keeps child's world transform; add() keeps the local one
obj.remove(child);
obj.visible = false; // hides the whole subtree

obj.layers.enable(1); // ADD layer 1; set(1) would replace the mask and leave layer 0
obj.layers.disable(1);
obj.layers.test(camera.layers); // true when the object shares a layer with the camera

obj.traverse((descendant) => {
  if (descendant.isMesh) descendant.castShadow = true; // visits obj itself, then all descendants
});

// matrix updates run automatically while matrixAutoUpdate is true (the default)
obj.matrixAutoUpdate = false;
obj.updateMatrix(); // composes obj.matrix and flags the world matrix dirty
// when writing obj.matrix directly instead, flag the world matrix yourself:
obj.matrixWorldNeedsUpdate = true;
```

- `position`, `rotation`, `quaternion` and `scale` are read-only properties holding objects: mutate them (`set`, `copy`), never assign a new object.
- `pivot` is serialized by `toJSON()` and copied by `clone()`; r187 fixed `applyMatrix4()` so it honours the pivot when decomposing.
- `obj.dispose()` (r186+) dispatches a `dispose` event so the renderer frees per-object state (WebGPURenderer bindings, WebGLRenderer instanced-mesh buffers); it does not touch children, geometry, materials or textures.

## Group and Mesh

```javascript
const group = new THREE.Group(); // an Object3D that exists to hold children
const ball = new THREE.Mesh(new THREE.SphereGeometry(0.5), new THREE.MeshStandardMaterial());
group.add(ball);
group.rotation.y = Math.PI / 4; // transforms the whole subtree
scene.add(group);

// one material per geometry group; BoxGeometry has six (+x, −x, +y, −y, +z, −z)
// a group whose materialIndex has no material is not drawn
const faceColors = [0xff0000, 0x00ff00, 0x0000ff, 0xffff00, 0xff00ff, 0x00ffff];
const dice = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  faceColors.map((color) => new THREE.MeshStandardMaterial({ color })),
);

dice.castShadow = true; // this object casts shadows
dice.receiveShadow = true; // this object shows shadows
dice.frustumCulled = true; // default: skipped when its bounding sphere leaves the camera frustum
dice.renderOrder = 10; // renders later within the same transparency bucket
```

## Culling bounds

Frustum culling tests a bounding sphere, not the bounding box: `geometry.boundingSphere` for a plain mesh, the object's own `boundingSphere` for InstancedMesh, SkinnedMesh and BatchedMesh. Both are computed once, so after editing positions call `geometry.computeBoundingSphere()`, and after moving instances or bones far from where they started call `object.computeBoundingSphere()`, or visible objects get culled. Instancing, merging and LOD: `threejs-geometry` and `threejs-performance`.

## Coordinate system

Right-handed, +Y up: +X right, +Y up, +Z toward the viewer. A camera at the origin with no rotation looks down −Z.

```javascript
scene.add(new THREE.AxesHelper(5)); // red X, green Y, blue Z; 5 = axis length
```

## Vector3

```javascript
const a = new THREE.Vector3(1, 0, 0);
const b = new THREE.Vector3(0, 1, 0);
const c = a.clone(); // new vector with the same values

a.copy(b); // a becomes b
a.add(b); // in place: a = a + b
a.sub(b);
a.multiplyScalar(2);
a.divideScalar(2);
a.lerp(b, 0.5); // move halfway toward b
a.normalize(); // unit length
a.negate();
a.cross(b); // cross product, written into a (right-hand rule)

const len = a.length();
const lenSq = a.lengthSq(); // compare lengths without a square root
const dist = a.distanceTo(b);
const dot = a.dot(b); // > 0 same half-space, < 0 opposite, 0 perpendicular
const angle = a.angleTo(b); // radians

a.applyQuaternion(new THREE.Quaternion());
a.applyMatrix4(new THREE.Matrix4());
a.project(camera); // world position to normalized device coordinates
a.unproject(camera); // normalized device coordinates back to world
```

Almost every method mutates `this` and returns it; `clone()` first when the original must survive.

## Matrix4, Quaternion, Euler

```javascript
const m = new THREE.Matrix4();
m.identity();
m.makeTranslation(1, 2, 3);
m.makeRotationY(Math.PI / 4);
m.makeScale(2, 2, 2);

const position = new THREE.Vector3();
const quaternion = new THREE.Quaternion();
const scale = new THREE.Vector3(1, 1, 1);
m.compose(position, quaternion, scale); // build from transform components
m.decompose(position, quaternion, scale); // read them back

m.multiply(new THREE.Matrix4()); // m = m * other (other applied first)
m.premultiply(new THREE.Matrix4()); // m = other * m
m.invert();
m.lookAt(new THREE.Vector3(0, 0, 1), new THREE.Vector3(), new THREE.Vector3(0, 1, 0)); // eye, target, up

const q = new THREE.Quaternion();
q.setFromEuler(new THREE.Euler(0, Math.PI / 2, 0, "YXZ")); // Euler order matters, XYZ is the default
q.setFromAxisAngle(new THREE.Vector3(0, 1, 0), Math.PI / 4);
q.setFromRotationMatrix(m);
q.slerp(new THREE.Quaternion(), 0.5); // spherical interpolation, written into q
q.invert();
```

Frame-rate-independent smoothing, springs and slerp-based camera motion: `threejs-math-motion`.

## Color

```javascript
const color = new THREE.Color(0xff0000); // hex number
const sky = new THREE.Color("#87ceeb"); // also colour names and "rgb(255, 0, 0)"

color.setHex(0x0000ff);
color.setRGB(1, 0.5, 0); // 0–1 in the linear working colour space; hex and CSS strings are sRGB
color.setRGB(1, 128 / 255, 0, THREE.SRGBColorSpace); // sRGB components, the same colour as 0xff8000
color.setHSL(0.5, 1, 0.5); // hue, saturation, lightness in 0–1, also linear unless a colour space is passed
color.lerp(sky, 0.5); // interpolate toward the target, written into color
color.multiplyScalar(2); // scale the components
```

Texture colour spaces: `threejs-textures`.

## MathUtils

```javascript
const clamped = THREE.MathUtils.clamp(15, 1, 10); // 10
const mixed = THREE.MathUtils.lerp(0, 10, 0.5); // 5
const mapped = THREE.MathUtils.mapLinear(0.5, 0, 1, 0, 100); // 50
const radians = THREE.MathUtils.degToRad(90);
const degrees = THREE.MathUtils.radToDeg(Math.PI);
const random = THREE.MathUtils.randFloat(1, 10);
const eased = THREE.MathUtils.smoothstep(0.5, 0, 1); // eased 0–1 curve
const damped = THREE.MathUtils.damp(0, 10, 4, 1 / 60); // frame-rate-independent approach: current, target, lambda, dt
```
