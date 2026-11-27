# TSL authoring (WebGPURenderer)

Purpose: writing custom shader logic with TSL node functions for WebGPURenderer in r187 - uniforms, textures, displacement, reusable functions, porting GLSL, and the renamed names that break old code; open for any shader work under WebGPURenderer. Node material classes and their slots are listed in `threejs-materials` (`references/node-materials.md`); renderer creation is in `threejs-fundamentals`.

Imports: node materials and the renderer from `three/webgpu`, TSL functions from `three/tsl`. There is no `three/nodes` module. WebGPURenderer falls back to its WebGL 2 backend when WebGPU is unavailable, so one TSL code path covers both. `ShaderMaterial`, `RawShaderMaterial` and `onBeforeCompile` do not work under WebGPURenderer.

## Minimal scene

```javascript
import * as THREE from "three/webgpu";
import { color, mix, sin, time, uv } from "three/tsl";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init();
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111827); // the WebGPU canvas is transparent by default
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 2);

const material = new THREE.MeshBasicNodeMaterial(); // unlit; MeshStandardNodeMaterial is lit and black without lights
material.colorNode = mix(color(0x004466), color(0x66ddff), sin(uv().x.mul(20).add(time)).mul(0.5).add(0.5));
scene.add(new THREE.Mesh(new THREE.PlaneGeometry(2, 2), material));

renderer.setAnimationLoop(() => renderer.render(scene, camera));
```

A ShaderMaterial maps to `MeshBasicNodeMaterial` (unlit, `colorNode`/`opacityNode`), to the slots of `MeshStandardNodeMaterial` (lit: `colorNode`, `roughnessNode`, `emissiveNode`, `normalNode` ...), or to `fragmentNode` when you want to replace the whole fragment output including lighting. Output color goes through the renderer's tone mapping and color space conversion automatically - no includes needed.

## Uniforms and time

`uniform(value)` creates an updatable uniform; assign `.value` from JS without rebuilding the shader. `time` (seconds) and `deltaTime` are built-in nodes updated every frame.

```javascript
import * as THREE from "three/webgpu";
import { abs, mx_fractal_noise_float, time, uniform, uv, vec3 } from "three/tsl";

const amplitude = uniform(0.5);
const tint = uniform(new THREE.Color(0x66ccff)); // Color, Vector2/3/4 and Matrix values work too

const material = new THREE.MeshBasicNodeMaterial();
material.colorNode = vec3(abs(mx_fractal_noise_float(uv().mul(10).add(time.mul(0.2))))).mul(amplitude).mul(tint);

amplitude.value = 1.0; // from JS, any time
```

`uniformArray([...])` holds an array of values (renamed from `uniforms()` in r168).

## Textures and discard

`texture(map)` samples with `uv(map.channel)` plus the texture's own offset/repeat/rotation; `texture(map, uvNode)` takes explicit coordinates and ignores that transform. `texture(map).sample(uvNode)` resamples an existing texture node (renamed from `.uv()` in r172). `.discard()` on a boolean node drops the fragment, as `discard` does in GLSL (`Discard(cond)` is the function form).

```javascript
import * as THREE from "three/webgpu";
import { Fn, color, mix, smoothstep, texture, uniform, uv } from "three/tsl";

const noiseMap = await new THREE.TextureLoader().loadAsync("textures/noise.png"); // data: keep NoColorSpace
const progress = uniform(0); // animate progress.value from 0 to 1

const material = new THREE.MeshBasicNodeMaterial();
material.colorNode = Fn(() => {
  const n = texture(noiseMap, uv()).r;
  n.lessThan(progress).discard();
  return mix(color(1, 0.5, 0), color(0.5, 0.5, 0.5), smoothstep(progress, progress.add(0.1), n));
})();
```

Color textures sampled with `texture()` are decoded to linear like in built-in materials (set `colorSpace = SRGBColorSpace` on them).

## Vertex displacement

`positionNode` replaces the local position after three.js has applied morph targets, skinning, displacement maps, batching and InstancedMesh transforms. Build it on `positionLocal`, which holds that result; `positionGeometry` is the raw attribute, so building on it discards all of those. Displacement needs a subdivided mesh.

```javascript
import * as THREE from "three/webgpu";
import { normalLocal, positionLocal, sin, time } from "three/tsl";

const sun = new THREE.DirectionalLight(0xffffff, 2);
sun.position.set(1, 2, 3);
scene.add(sun, new THREE.HemisphereLight(0xffffff, 0x223344, 1)); // lit material: black without lights

const material = new THREE.MeshStandardNodeMaterial({ flatShading: true }); // normals from the displaced surface; vertex normals ignore positionNode
material.positionNode = positionLocal.add(normalLocal.mul(sin(positionLocal.y.mul(4).add(time))).mul(0.2));

const mesh = new THREE.Mesh(new THREE.PlaneGeometry(2, 2, 96, 96), material);
mesh.rotation.x = -Math.PI / 3; // tilt so the displacement along the normal is visible
scene.add(mesh);
```

## Reusable logic, variables and control flow

Write reusable logic as a TSL `Fn`; it compiles to both WGSL and GLSL. Parameters arrive as an array (`Fn(([a, b]) => ...)`) or an object (`Fn(({ a, b }) => ...)`). `varying(node, "name")` (or `node.toVarying("name")`) computes a value in the vertex stage and interpolates it to the fragment stage.

```javascript
import * as THREE from "three/webgpu";
import { Fn, If, Loop, color, float, mix, positionWorld, sin, varying } from "three/tsl";

const heightFade = Fn(([y]) => mix(color(0x003344), color(0x88eeff), sin(y.mul(3)).mul(0.5).add(0.5)));

const material = new THREE.MeshBasicNodeMaterial();
const vWorldY = varying(positionWorld.y, "vWorldY"); // interpolated from vertex to fragment stage
material.colorNode = heightFade(vWorldY);

// Mutable variables need .toVar(); assign with .assign() / .addAssign(), never JS "="
const stripes = Fn(() => {
  const sum = float(0).toVar();
  Loop(4, ({ i }) => {
    sum.addAssign(sin(positionWorld.x.mul(float(i).add(1))));
  });
  If(sum.lessThan(0), () => {
    sum.assign(0);
  });
  return sum;
});
```

JS `if`/`for` run once while the graph is built; use `If`, `Loop` and `select(cond, a, b)` for per-pixel branching.

## GLSL to TSL mapping

| GLSL (ShaderMaterial) | TSL |
| --- | --- |
| `gl_FragColor = vec4(c, a)` | `material.colorNode = c; material.opacityNode = a` (or `fragmentNode`) |
| `gl_Position = ...` | `material.positionNode` (local position, keeps the built-in transforms) or `vertexNode` (replaces the built-in vertex-stage logic; a `vec4` clip-space result) |
| `uniform float t;` + `uniforms.t.value` | `const t = uniform(0); t.value = ...` |
| `time` uniform fed from JS | `time` |
| `varying vec2 vUv;` | `uv()` (already interpolated) or `varying(node, "vName")` |
| `texture(map, vUv)` | `texture(map, uvNode)` |
| `position`, `normal` | `positionLocal`, `normalLocal` (`positionGeometry`, `normalGeometry` for raw attributes) |
| `(modelMatrix * vec4(position, 1)).xyz` | `positionWorld` |
| `normalize(mat3(modelMatrix) * normal)` | `normalWorld` |
| `normalMatrix * normal` / view-space position | `normalView` / `positionView` |
| `cameraPosition` | `cameraPosition` |
| `attribute vec3 aOffset;` | `attribute("aOffset", "vec3")`; per instance: `instancedBufferAttribute(attr)` |
| `gl_InstanceID` / `gl_VertexID` | `instanceIndex` / `vertexIndex` |
| `discard` | `cond.discard()` / `Discard(cond)` |
| `if` / `for` | `If(...).ElseIf(...).Else(...)`, `Loop(n, ({ i }) => ...)` |
| `x * y + z` | `x.mul(y).add(z)` (operators are methods) |
| `#include <packing>` helpers | `perspectiveDepthToViewZ`, `viewZToPerspectiveDepth`, `packNormalToRGB`, `unpackRGBToNormal` |

## Porting GLSL strings

Rewrite GLSL functions as `Fn`s instead of pasting shader strings. `glslFn(code)` and `wgslFn(code)` wrap native code for one shading language only, so a `glslFn` node is unusable on the WebGPU backend and a `wgslFn` node on the WebGL 2 backend (inferred from the per-language code nodes; not tested here). Compute shaders (`Fn(...)().compute(count)`, `renderer.compute()`) are covered in `threejs-math-particles-fields`.

## Renamed and removed names

| Old | Current | Since |
| --- | --- | --- |
| `import ... from "three/nodes"` | `three/tsl` (functions), `three/webgpu` (classes) | r171 |
| `uniforms()` | `uniformArray()` | r168 |
| `viewportTopLeft` / `viewportBottomLeft` | `viewportUV` / `viewportUV.flipY()` | r168 |
| `burn()`, `dodge()`, `screen()`, `overlay()` | `blendBurn()`, `blendDodge()`, `blendScreen()`, `blendOverlay()` | r171 |
| `storageObject()` | `storage().setPBO(true)` | r171 |
| `texture(map).uv(uvNode)` | `texture(map).sample(uvNode)` | r172 |
| `rangeFog(color, near, far)` / `densityFog(color, d)` | `fog(color, rangeFogFactor(near, far))` / `fog(color, densityFogFactor(d))` | deprecated r172, no longer exported in r187 |
| `materialAOMap` | `materialAO` | r172 |
| `node.label("x")` | `node.setName("x")` (label still works, warns) | r179 |
| `PI2` | `TWO_PI` (PI2 still exported, deprecated) | r181 |
| `directionToColor()` / `colorToDirection()` | `packNormalToRGB()` / `unpackRGBToNormal()` (old names warn) | r185 |
| `renderer.computeAsync()`, `renderAsync()` | `await renderer.init()` once, then the sync calls | r181 |

`toVarying` and `toVertexStage` exist only as methods (`node.toVarying("name")`); the exported functions are `varying()` and `vertexStage()`.

## r187 behaviour changes

- Operands and math-function parameters are now implicitly coerced to a broader common type (migration guide, PR #33695), so generated code can differ slightly from r186 and some explicit conversions become unnecessary.
- Struct storage buffers are always treated as arrays: `storage(buffer, StructType).get("member")` becomes `.element(0).get("member")`.
- Custom light nodes register with `CustomLight.registerNode(CustomLightNode)` instead of `renderer.library.addLight(...)`.

## Debugging

`await renderer.debug.getShaderAsync(scene, camera, mesh)` returns the generated `{ vertexShader, fragmentShader }` for one mesh (WGSL on the WebGPU backend, GLSL on the WebGL 2 backend). `renderer.debug.onShaderError` is honoured only by the WebGL 2 backend. Visual debugging works as in GLSL: `material.colorNode = uv()` or `normalView.mul(0.5).add(0.5)`.
