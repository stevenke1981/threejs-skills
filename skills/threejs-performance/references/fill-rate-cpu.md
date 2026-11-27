# Fill Rate, Shadows, CPU Work and Shader Compile Stalls

Purpose: GPU fragment cost (pixel ratio, overdraw, shadow maps), per-frame CPU costs (matrix updates, raycasting, allocations) and avoiding first-use shader compile hitches (WebGLRenderer behaviour checked against the r187 source).

## Pixel Ratio

The drawing buffer is `width × ratio` by `height × ratio` pixels, so shading cost scales with the pixel ratio squared: a 3× display shades 9× the pixels of 1×. Cap it:

```javascript
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
```

Adaptive resolution (lower the ratio while frame time is over budget, raise it when idle) is the next step for heavy scenes; post-processing passes scale with it too (`threejs-postprocessing`).

## Overdraw and Transparency

WebGLRenderer draws opaque objects first (sorted by group order, render order, material, then distance front to back), transmissive next, transparent last, back to front so blending composes correctly. Stacked transparent layers cannot reject each other: every layer behind another transparent surface still shades its pixels, and a full-screen transparent overlay shades every pixel once more. Prefer opaque or `alphaTest` / `alphaHash` cut-outs where the look allows, and keep transparent quads few and small. `renderer.info` does not show overdraw; only frame time moves.

## Shadow Maps

Each shadow-casting light renders its casters into a depth map every frame while shadow updates are on. The default `shadow.mapSize` is 512 × 512; raise it knowingly, and freeze it for static scenery:

```javascript
const sun = new THREE.DirectionalLight(0xffffff, 2);
sun.position.set(3, 6, 4);
sun.castShadow = true;
sun.shadow.mapSize.set(1024, 1024);
scene.add(sun);

renderer.shadowMap.enabled = true;

// static scenery: draw the shadow maps once instead of every frame
renderer.shadowMap.autoUpdate = false;
renderer.shadowMap.needsUpdate = true; // render them one more time, then freeze

// per light instead of globally: sun.shadow.autoUpdate = false; sun.shadow.needsUpdate = true;
```

- Each shadow-casting light adds its own pass, and those draws appear in `info.render.calls`: it re-draws every `castShadow` object inside its shadow camera's frustum, a PointLight once per cube face the object falls in (up to six).
- Tight shadow camera bounds (`sun.shadow.camera.left`, `right`, `top`, `bottom`, then `sun.shadow.camera.updateProjectionMatrix()`) cut the casters drawn; the map size sets each pass's fill cost.
- Turn `castShadow` off for small or distant objects whose shadow nobody sees. Shadow types, bias and cascades belong to `threejs-lighting`.

## Static Transforms

Every `render()` calls `scene.updateMatrixWorld()` (while `scene.matrixWorldAutoUpdate` is true), which visits every object and recomposes its local matrix from position, rotation and scale when `matrixAutoUpdate` is true (the default). For scenery that never moves, compose once:

```javascript
mesh.matrixAutoUpdate = false;
mesh.updateMatrix(); // bakes position/rotation/scale (and pivot) into mesh.matrix and flags the world matrix

// later, when this object does move:
mesh.position.set(1, 2, 3);
mesh.updateMatrix(); // manual now: the renderer no longer recomposes it
```

If you write `mesh.matrix` directly instead of calling `updateMatrix()`, also set `mesh.matrixWorldNeedsUpdate = true`: since r185 `updateWorldMatrix()` honours that flag.

## Raycasting

`intersectObjects(objects, recursive = true)` walks every descendant of every object and tests each: a layers test first, then the bounding sphere, then triangles. Narrow the input instead of filtering the output:

```javascript
const raycaster = new THREE.Raycaster();
raycaster.layers.set(1); // only objects enabled on layer 1 are tested, before any geometry work

const pickables = [];
scene.traverse((child) => {
  if (child.isMesh && child.userData.pickable) {
    child.layers.enable(1); // adds layer 1 while keeping layer 0 membership
    pickables.push(child);
  }
});

const hits = raycaster.intersectObjects(pickables, false); // an explicit list, no recursion
```

A pointermove handler that raycasts runs on every event; throttle it to once per frame. For large static meshes the `three-mesh-bvh` package accelerates the triangle tests. Picking APIs: `threejs-interaction`.

## No Allocations in the Loop

Objects created per frame become garbage, and collection pauses show up as periodic frame spikes. Reuse vectors and matrices declared outside the loop:

```javascript
const timer = new THREE.Timer();
timer.connect(document);

const step = new THREE.Vector3(); // allocated once, reused every frame
const target = new THREE.Vector3(0, 1, 0);

function animate() {
  timer.update();
  step.subVectors(target, mesh.position).multiplyScalar(Math.min(timer.getDelta() * 2, 1));
  mesh.position.add(step);
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

## Shader Compile Stalls

Programs build lazily: the first frame a material renders under conditions it has not seen before (a different light count, fog switched on, shadows enabled) stalls while its shader compiles. Build them during loading:

```javascript
// during loading, before the first frame: compile everything the scene will render with
await renderer.compileAsync(scene, camera); // polls KHR_parallel_shader_compile where available

renderer.setAnimationLoop(() => {
  renderer.render(scene, camera);
});
```

- `renderer.compile(scene, camera)` issues the same builds but returns at once, so the first render still waits for any program that has not finished. Prefer the async form.
- To warm up an object before adding it, pass the scene it will join: `await renderer.compileAsync(object, camera, scene)` so its programs see that scene's lights.
- WebGPURenderer also has `compileAsync(scene, camera, targetScene)`; r187 made it compile pipelines in parallel, compile with the render configuration and preserve the material side, so its pipelines match the ones the real render uses (release notes #34816, #35168, #35016).
- Changing a setting that is compiled into the shader (`transparent`, `side`, `blending`, `flatShading`, `vertexColors`, `dithering`, `premultipliedAlpha`, `alphaHash`, or adding or removing a map) needs `material.needsUpdate = true` and builds a new program, so avoid flipping these at runtime (`threejs-materials` lists them). Toggling lights or a light's `castShadow` also changes the program of every lit material.
