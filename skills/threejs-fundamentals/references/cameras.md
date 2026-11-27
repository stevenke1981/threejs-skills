# Cameras

Purpose: construction and update rules for PerspectiveCamera, OrthographicCamera, ArrayCamera and CubeCamera in three.js r187.

## PerspectiveCamera

```javascript
// PerspectiveCamera(fov in degrees, aspect, near, far)
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 5, 10);
camera.lookAt(0, 0, 0); // rotates the camera to face the target

camera.fov = 60;
camera.updateProjectionMatrix(); // required after changing fov, aspect, near or far
```

## OrthographicCamera

```javascript
// OrthographicCamera(left, right, top, bottom, near, far); frustum size in world units
const aspect = window.innerWidth / window.innerHeight;
const frustumHeight = 10;
const camera = new THREE.OrthographicCamera(
  (-frustumHeight * aspect) / 2,
  (frustumHeight * aspect) / 2,
  frustumHeight / 2,
  -frustumHeight / 2,
  0.1,
  1000,
);

window.addEventListener("resize", () => {
  const nextAspect = window.innerWidth / window.innerHeight;
  camera.left = (-frustumHeight * nextAspect) / 2;
  camera.right = (frustumHeight * nextAspect) / 2;
  camera.updateProjectionMatrix();
});
```

Zoom an orthographic camera with `camera.zoom`, followed by `updateProjectionMatrix()`; moving it along its view axis changes nothing on screen.

## ArrayCamera

Renders the scene once per sub-camera, each into its own viewport (split screen, multi-view).

```javascript
// Viewport units: WebGLRenderer takes drawing-buffer pixels (CSS size × pixel ratio);
// WebGPURenderer takes CSS pixels and multiplies by the pixel ratio itself
// (by 1 when it renders into a render target).
const width = window.innerWidth;
const height = window.innerHeight;
const dpr = renderer.getPixelRatio();
const cameras = [];
for (let i = 0; i < 4; i++) {
  const subCamera = new THREE.PerspectiveCamera(75, width / height, 0.1, 100);
  subCamera.viewport = new THREE.Vector4(
    ((i % 2) * width * dpr) / 2,
    (Math.floor(i / 2) * height * dpr) / 2,
    (width * dpr) / 2,
    (height * dpr) / 2,
  ); // WebGLRenderer values; drop the "* dpr" factors under WebGPURenderer
  subCamera.position.set(-2 + (i % 2) * 4, 1.5, 6);
  subCamera.lookAt(0, 0, 0);
  subCamera.updateMatrixWorld(); // sub-cameras are not scene-graph children; update them when they move
  cameras.push(subCamera);
}
const arrayCamera = new THREE.ArrayCamera(cameras);
renderer.render(scene, arrayCamera);
```

Culling (r187): both renderers build a `FrustumArray` from the sub-cameras and keep an object when it intersects any sub-camera frustum, so the ArrayCamera's own position and fov no longer matter for culling (up to r186 WebGLRenderer culled with the ArrayCamera's own frustum). On WebGPURenderer, `arrayCamera.perCameraFrustumCulled = true` (default `false`) also skips each draw for sub-cameras that cannot see the object; it costs one frustum test per sub-camera, so enable it only when the frustums barely overlap. WebGLRenderer ignores the flag.

## CubeCamera

```javascript
const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256);
const cubeCamera = new THREE.CubeCamera(0.1, 1000, cubeRenderTarget);
scene.add(cubeCamera);

const chrome = new THREE.Mesh(
  new THREE.SphereGeometry(1, 32, 16),
  new THREE.MeshStandardMaterial({ color: 0xffffff, metalness: 1, roughness: 0 }),
);
chrome.material.envMap = cubeRenderTarget.texture;
scene.add(chrome);

// every frame: hide the reflective mesh, or it is drawn into the map it samples
chrome.visible = false;
cubeCamera.position.copy(chrome.position);
cubeCamera.update(renderer, scene);
chrome.visible = true;
```

With WebGPURenderer, build the target with `CubeRenderTarget` from `three/webgpu`; `WebGLCubeRenderTarget` cannot be used with WebGPURenderer since r183. Static environment maps and PMREM belong to `threejs-lighting` and `threejs-textures`.
