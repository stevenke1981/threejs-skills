---
name: threejs-math-sdf-raymarching
description: "Signed distance fields and sphere tracing in three.js r187: exact SDF primitives, transforms, repetition, booleans and smooth minimum, a raymarcher on a full-screen quad or proxy box, normals, soft shadows, AO, Mandelbulb, depth compositing with meshes, step budgets, GLSL and TSL. Use for implicit or blobby shapes, fractals, shader-only scenes, holes or speckles in a raymarch, SDF not lining up with meshes. Not for fog, clouds or volume marching (use threejs-math-atmosphere), SDF to mesh (use threejs-math-procedural-geometry) or the noise itself (use threejs-math-noise). 中文: 距離場, 光線步進, 符號距離函數, 平滑聯集, 碎形, 軟陰影"
---

# Three.js Math: SDF Raymarching

Targets three.js r187 (`three@0.187.0`). Examples use WebGLRenderer; the TSL marcher in `references/compositing-and-webgpu.md` uses WebGPURenderer. Nothing this skill relies on changed between r186 and r187 (the `RaymarchingBox` addon only switched to the compact layout form; `ShaderMaterial` uniform prefixes, the `packing` chunk and `FullScreenQuad` are unchanged).

## Quick Start

three ships no SDF library and no raymarching material for WebGL: the marcher is a `ShaderMaterial` on a mesh that covers the screen. A sphere smooth-unioned with a rounded box over a ground plane, lit by one sun with a soft shadow:

```javascript
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5)); // every pixel runs the full march
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(3, 2.5, 4);
const controls = new OrbitControls(camera, renderer.domElement);
controls.target.set(0, 0.7, 0);
controls.update();

const vertexShader = /* glsl */ `
  varying vec3 vRayDir;

  void main() {
    gl_Position = vec4(position.xy, 0.0, 1.0); // PlaneGeometry(2, 2) covers clip space
    vec4 viewPos = inverse(projectionMatrix) * vec4(position.xy, -1.0, 1.0); // a point on the near plane
    vRayDir = transpose(mat3(viewMatrix)) * (viewPos.xyz / viewPos.w); // view to world: viewMatrix is rigid
  }
`;

const fragmentShader = /* glsl */ `
  uniform float time;
  varying vec3 vRayDir;

  float sdSphere(vec3 p, float r) { return length(p) - r; }
  float sdBox(vec3 p, vec3 b) {
    vec3 q = abs(p) - b;
    return length(max(q, 0.0)) + min(max(q.x, max(q.y, q.z)), 0.0);
  }
  float sdRoundBox(vec3 p, vec3 b, float r) { return sdBox(p, b - r) - r; }
  float smin(float a, float b, float k) { // quadratic; k = blend width, k > 0
    float h = max(k - abs(a - b), 0.0) / k;
    return min(a, b) - h * h * k * 0.25;
  }

  float map(vec3 p) {
    float sphere = sdSphere(p - vec3(0.9 * sin(time), 1.2, 0.0), 0.45);
    float box = sdRoundBox(p - vec3(0.0, 0.7, 0.0), vec3(0.5), 0.08);
    return min(smin(sphere, box, 0.35), p.y); // p.y: the ground plane y = 0
  }

  float trace(vec3 ro, vec3 rd) {
    float t = 0.0;
    for (int i = 0; i < 96; i++) {
      float d = map(ro + rd * t);
      if (d < 0.0005 * t + 0.0001) return t; // one pixel covers more world far away
      t += d;
      if (t > 50.0) break;
    }
    return -1.0; // miss, including running out of steps
  }

  vec3 calcNormal(vec3 p) { // tetrahedron technique: 4 map calls
    const vec2 k = vec2(1.0, -1.0);
    const float h = 0.0005;
    return normalize(k.xyy * map(p + k.xyy * h) + k.yyx * map(p + k.yyx * h) +
                     k.yxy * map(p + k.yxy * h) + k.xxx * map(p + k.xxx * h));
  }

  float softShadow(vec3 ro, vec3 rd, float tmin, float tmax, float k) { // 0 shadowed, 1 lit
    float res = 1.0;
    float t = tmin;
    for (int i = 0; i < 48 && t < tmax; i++) {
      float h = map(ro + rd * t);
      if (h < 0.001) return 0.0;
      res = min(res, k * h / t);
      t += clamp(h, 0.01, 0.25);
    }
    return res;
  }

  void main() {
    vec3 ro = cameraPosition;
    vec3 rd = normalize(vRayDir); // interpolated vectors are shorter than unit length
    vec3 sky = vec3(0.55, 0.7, 0.9), col = sky;
    float t = trace(ro, rd);
    if (t > 0.0) {
      vec3 p = ro + rd * t;
      vec3 n = calcNormal(p);
      vec3 sunDir = normalize(vec3(0.6, 0.8, 0.4));
      float shadow = softShadow(p + n * 0.002, sunDir, 0.02, 10.0, 16.0);
      vec3 albedo = p.y < 0.01 + 0.0005 * t ? vec3(0.45) : vec3(0.9, 0.35, 0.15); // same tolerance as trace()
      col = albedo * (max(dot(n, sunDir), 0.0) * shadow * vec3(1.0, 0.95, 0.85) + 0.2 * sky);
    }
    gl_FragColor = vec4(col, 1.0);
    #include <tonemapping_fragment>
    #include <colorspace_fragment>
  }
`;

const material = new THREE.ShaderMaterial({ uniforms: { time: { value: 0 } }, vertexShader, fragmentShader });
const quad = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), material);
quad.frustumCulled = false; // the plane's bounds sit at the origin, not on screen
scene.add(quad);

const timer = new THREE.Timer();
timer.connect(document);

renderer.setAnimationLoop(animate);

function animate() {
  timer.update();
  material.uniforms.time.value = timer.getElapsed();
  renderer.render(scene, camera);
}
```

## How Sphere Tracing Works

`map(p)` returns a signed distance: positive outside, negative inside, and never larger than the true distance to the nearest surface. A sphere of radius `map(p)` around `p` is therefore empty, so the ray can advance by that amount without passing through anything.

- Hit: the distance drops below a threshold that grows with `t` (`0.0005 * t + 0.0001`), because one pixel covers more world far away. A fixed tiny epsilon wastes steps in the distance and still leaves holes at grazing angles.
- Miss: `t` passes the far limit, or the step count runs out. Running out usually means a ray skimming a silhouette; treat it as a miss (or fade it), never shade it as a hit at whatever `t` it reached.
- Relaxation: when `map` is only a bound or an estimate that can overshoot (twist, bend, non-uniform scale, displacement, fractals), take shorter steps, `t += d * 0.5` to `0.8`, so the ray does not jump past thin parts.

Fixed-step marching through a volume (fog, clouds, `RaymarchingBox`) is a different algorithm; see `threejs-math-atmosphere`.

## Which Operations Keep a Distance

| Operation | Result | March with |
| --- | --- | --- |
| Primitives in `references/primitives-and-operators.md`, translation, rotation | exact | `t += d` |
| Uniform scale `sd(p / s) * s`, mirror `abs(p.x)` | exact | `t += d` |
| `min` union | exact outside, bound inside | `t += d`; AO and shells see the broken interior |
| `max` intersection and subtraction | bound | `t += d` (safe, slower) |
| Quadratic `smin` / `smax` | exact outside the blend, bound inside (never overestimates) | `t += d` |
| Repetition `p - c * round(p / c)` | exact only for shapes symmetric inside their cell | check neighbor cells, or shorten the step |
| Twist `k` rad/unit at radius `r` | stretches by `(k r + sqrt(k² r² + 4)) / 2` | `t += d / stretch` |
| Non-uniform scale, bend, noise displacement | neither | `t += d * 0.5..0.8`, tune until holes vanish |
| Mandelbulb and other escape-time estimators | estimate | `t += d * 0.5..0.8`, more steps |

Sources for these classes: Inigo Quilez's "Distance functions", "Smooth minimum", "Interior SDFs" and "Domain repetition" articles on iquilezles.org.

## Camera Rays in Brief

Build rays from the camera's own matrices, as the Quick Start does: `inverse(projectionMatrix)` to a near-plane point, `transpose(mat3(viewMatrix))` to world space, origin `cameraPosition`. Then FOV, aspect, resizes and OrbitControls all follow, and the SDF scene lines up with rasterized meshes. Details, the orthographic case, and passing matrices into a `FullScreenQuad` (which renders with its own orthographic camera): `references/compositing-and-webgpu.md`.

## Common Mistakes

- `vec3 rd = normalize(vec3(uv * 2.0 - 1.0, -1.0));` → rays from `inverse(projectionMatrix)` and `transpose(mat3(viewMatrix))`: a hard-coded pinhole ignores the camera's FOV, aspect and orientation, so the SDF scene does not line up with meshes or follow OrbitControls.
- `new FullScreenQuad(material)` with a shader reading `cameraPosition` → pass `camera.matrixWorld` and `camera.projectionMatrixInverse` as uniforms: FullScreenQuad renders with its own orthographic camera, so the built-in camera uniforms are not the scene camera's.
- `vRayDir = normalize(dir);` in the vertex shader only → also `normalize(vRayDir)` in the fragment shader: interpolated unit vectors are shorter than unit length, which shrinks every step.
- `if (d < 0.001)` → `if (d < 0.0005 * t + 0.0001)`: a fixed epsilon is too strict far away (steps run out, holes on silhouettes) and too loose up close.
- `return sdSphere(p / s, r);` → `return sdSphere(p / s, r) * s;`: scaling the domain scales the distance; without the factor the ray overshoots or crawls.
- `return length(max(abs(p) - b, 0.0));` → add `+ min(max(q.x, max(q.y, q.z)), 0.0)` with `q = abs(p) - b`: the short box is 0 everywhere inside, which breaks subtraction, shells and AO.
- `p.y < 0.01 ? ground : object` → `p.y < 0.01 + 0.0005 * t ? ground : object`, or return a material id from `map`: a hit only lands within the hit tolerance, which grows with `t`, so far ground hits fail a fixed test and draw a band of the wrong material at the horizon.
- `t += map(p);` over a twisted, bent, non-uniformly scaled or noise-displaced field → `t += map(p) * 0.6;`: those operations break the distance bound and full steps tunnel through thin features.
- `vec3 q = mod(p, c) - 0.5 * c;` → `vec3 q = p - c * round(p / c);`: the mod form puts the copies at odd multiples of `c / 2`, half a cell off the origin; round centers one copy on it.
- `smin(a, b, 0.0)` → `min(a, b)`, or keep `k > 0`: the polynomial divides by `k` and returns NaN.
- A smin snippet copied from Quilez's current article tuned with this skill's `k` → divide by 4 or use his `k *= 4.0` form consistently: his normalized `k` is the thickening, this `k` is the blend width (4 times larger).
- `softShadow(p, sunDir, 0.0, 10.0, 16.0)` → `softShadow(p + n * 0.002, sunDir, 0.02, 10.0, 16.0)`: a shadow ray starting on the surface reports a hit at once and the surface shadows itself.
- `float theta = acos(z.z / r);` → `acos(clamp(z.z / max(r, 1e-6), -1.0, 1.0))`: 0/0 at the origin gives NaN, which later blur or bloom passes spread into black blocks.
- `if (t > texture(tDepth, vUv).r)` → `perspectiveDepthToViewZ(depth, cameraNear, cameraFar) / rdView.z`: the depth buffer is non-linear, and view depth is not ray length away from the screen center.
- `gl_FragDepth = t / cameraFar;` → `vec4 clip = projectionMatrix * viewMatrix * vec4(p, 1.0); gl_FragDepth = clip.z / clip.w * 0.5 + 0.5;`: the depth buffer stores projected depth, so a linear value meets meshes at the wrong place.
- `projectionMatrix` used in the fragment shader undeclared → `uniform mat4 projectionMatrix;`: ShaderMaterial declares it for the vertex stage only, so the shader fails to compile.
- `renderer.render()` on a WebGPURenderer before `await renderer.init()` outside `setAnimationLoop` → await `init()` first: r186 and r187 throw ".render() called before the backend is initialized" (inside `setAnimationLoop` the renderer initializes itself first).

## Reference Files

- `references/primitives-and-operators.md`: exact primitives (sphere, box, round box, torus, capsule, capped cylinder, plane), driving a shape from an `Object3D`, scale, mirror, infinite and limited repetition, the twist and its stretch bound, booleans, smooth min/max, the k conventions, material blending. Open when writing `map()`.
- `references/shading-and-fractals.md`: tetrahedron and central-difference normals and epsilon choice, soft shadows including Aaltonen's improved penumbra, ambient occlusion, the Mandelbulb estimator with its NaN guards, folding fractals. Open once the march hits.
- `references/compositing-and-webgpu.md`: camera rays in full (perspective and orthographic), built-in uniforms per stage, writing `gl_FragDepth`, stopping rays at a `DepthTexture` in a `FullScreenQuad` pass, performance levers and step caps, the TSL marcher for WebGPURenderer, `depthNode`, `RaymarchingBox`.

## See Also

- `threejs-shaders`: ShaderMaterial rules, the two output includes, TSL basics and porting GLSL to TSL.
- `threejs-postprocessing`: EffectComposer, `OutputPass` and the WebGPU `RenderPipeline` for a raymarch pass over the scene.
- `threejs-textures`: render targets and `DepthTexture` setup for depth compositing.
- `threejs-math-noise`: noise and fBm for displacing SDF surfaces.
- `threejs-math-atmosphere`: fixed-step volume marching, fog, clouds and `RaymarchingBox`.
- `threejs-math-procedural-geometry`: marching cubes, turning an SDF into a triangle mesh.
- `threejs-performance`: measuring the per-pixel cost on the target device.
