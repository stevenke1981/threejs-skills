---
name: threejs-math-sdf-raymarching
description: "Signed distance fields and sphere tracing in three.js: SDF primitives, transforms and repetition, booleans and smooth minimum, a raymarcher in a full-screen quad or bounding box, gradient normals, soft shadows, ambient occlusion, fractals such as the Mandelbulb, depth compositing with rasterized meshes, and step budgets, in GLSL and TSL. Use when rendering implicit surfaces, blobby or fractal shapes, or shader-only scenes."
---

# Three.js Math: SDF Raymarching

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

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
  float smin(float a, float b, float k) {
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
    return -1.0; // miss
  }

  vec3 calcNormal(vec3 p) {
    const vec2 k = vec2(1.0, -1.0);
    const float h = 0.0005;
    return normalize(k.xyy * map(p + k.xyy * h) + k.yyx * map(p + k.yyx * h) +
                     k.yxy * map(p + k.yxy * h) + k.xxx * map(p + k.xxx * h));
  }

  float softShadow(vec3 ro, vec3 rd, float tmin, float tmax, float k) {
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

`map(p)` returns a signed distance: positive outside, negative inside, and never larger than the true distance to the nearest surface. A sphere of radius `map(p)` around `p` is therefore empty, so the ray can advance by that amount without passing through anything. `trace()` above repeats that step:

- Hit: the distance drops below a threshold. The threshold grows with `t` (`0.0005 * t + 0.0001`) because one pixel covers more world far away; a fixed tiny epsilon wastes steps in the distance and still leaves holes at grazing angles.
- Miss: `t` passes the far limit, or the step count runs out. Running out usually means a ray skimming a silhouette; treat it as a miss (or fade it), never shade it as a hit at whatever `t` it reached.
- Relaxation: when `map` is only an estimate (warps, displacement, fractals) take shorter steps, `t += d * 0.7;`, so the ray does not jump past thin parts.

Fixed-step marching through a volume (fog, clouds, `RaymarchingBox`) is a different algorithm; see `threejs-math-atmosphere`.

## Camera Rays

Build rays from the camera's own matrices. The Quick Start vertex shader unprojects each corner of the quad onto the near plane with `inverse(projectionMatrix)` and rotates it to world space with `transpose(mat3(viewMatrix))` (a rotation's inverse is its transpose). Any clip-space z lies on the same eye ray for a perspective camera; -1 keeps the four corners on one plane, so linear interpolation across the quad is exact. Field of view, aspect and orientation all come from the matrices, so the SDF scene lines up with rasterized meshes and follows `camera.fov`, resizes and OrbitControls.

- A mesh placed in the scene receives the scene camera's `projectionMatrix`, `viewMatrix` and `cameraPosition`.
- The ShaderMaterial fragment prefix declares only `viewMatrix`, `cameraPosition` and `isOrthographic`. A fragment shader that needs `projectionMatrix` declares `uniform mat4 projectionMatrix;` itself and the renderer fills it.
- `mesh.frustumCulled = false;` is required for a clip-space quad: culling tests the plane's object-space bounding sphere at the mesh position.
- `FullScreenQuad` from `three/addons/postprocessing/Pass.js` renders with its own `OrthographicCamera(-1, 1, 1, -1, 0, 1)`, so inside it the built-in camera uniforms describe that camera. Pass the scene camera's matrices as your own uniforms (see Mixing with Rasterized Meshes).

Orthographic camera: all rays are parallel and the origin moves across the near plane.

```glsl
vec3 rd = -transpose(mat3(viewMatrix))[2]; // the camera's forward axis in world space
vec3 ro = (inverse(viewMatrix) * vec4(viewPos.xyz / viewPos.w, 1.0)).xyz; // pass ro as a varying
```

## SDF Primitives

`p` is already in the primitive's local space. All of these are exact distances.

```glsl
float sdSphere(vec3 p, float r) { return length(p) - r; }

float sdBox(vec3 p, vec3 b) { // b = half extents
  vec3 q = abs(p) - b;
  return length(max(q, 0.0)) + min(max(q.x, max(q.y, q.z)), 0.0);
}

float sdRoundBox(vec3 p, vec3 b, float r) { return sdBox(p, b - r) - r; } // outer size stays b

float sdTorus(vec3 p, float R, float r) { // ring in the xz plane
  return length(vec2(length(p.xz) - R, p.y)) - r;
}

float sdCapsule(vec3 p, vec3 a, vec3 b, float r) {
  vec3 pa = p - a, ba = b - a;
  float h = clamp(dot(pa, ba) / dot(ba, ba), 0.0, 1.0);
  return length(pa - ba * h) - r;
}

float sdCylinder(vec3 p, float r, float h) { // axis y, half height h
  vec2 d = abs(vec2(length(p.xz), p.y)) - vec2(r, h);
  return min(max(d.x, d.y), 0.0) + length(max(d, 0.0));
}

float sdPlane(vec3 p, vec3 n, float h) { return dot(p, n) + h; } // n must be unit length
```

- The second term of `sdBox` is the interior distance (negative inside). Without it the field is 0 everywhere inside: fine for tracing from outside, wrong for subtraction, AO and shells.
- `sd - r` grows any shape by `r` and rounds its convex edges. `abs(sd) - thickness` turns a solid into a shell.

## Transforms

A shape is moved by evaluating it at the inverse-transformed point: `sdBox(transpose(R) * (p - center), b)` for a rotation `R`. To drive an SDF shape from an `Object3D`, upload the inverse of its world matrix:

```javascript
import * as THREE from "three";

const proxy = new THREE.Object3D(); // position and rotate it like any object
const uInvModel = { value: new THREE.Matrix4() }; // add to the material's uniforms
proxy.updateMatrixWorld(); // each frame, after moving proxy
uInvModel.value.copy(proxy.matrixWorld).invert();
// GLSL: float d = sdBox((uInvModel * vec4(p, 1.0)).xyz, vec3(0.5)); valid for translation and rotation only
```

- Uniform scale: `sdShape(p / s) * s`. Without the trailing `* s` the distance is off by the factor `s`; the ray crawls (s > 1) or overshoots (s < 1).
- Mirror: `p.x = abs(p.x);` evaluates one half and gets the other for free.
- Infinite repetition with cell size `c`: `vec3 q = p - c * round(p / c);`, one copy centered on the origin. Limited repetition: `vec3 q = p - c * clamp(round(p / c), -l, l);` with `l` copies per side. Both are correct only when the shape fits inside its cell and is symmetric about the cell center; otherwise the nearest surface can be in a neighbor cell, the distance is overestimated and holes appear. Shrink the step or evaluate the neighbor cells too.
- Non-uniform scale, twist, bend and noise displacement stretch space, so the result is no longer a distance, only a bound after scaling. Multiply the step by a factor below 1 (the reciprocal of the largest stretch), or the ray tunnels through thin parts. A twist of `k` radians per unit stretches by `(k * r + sqrt(k * k * r * r + 4.0)) / 2.0` at distance `r` from the axis (`sqrt(1.0 + k * k * r * r)` is too small); use the shape's largest `r`:

```glsl
vec3 opTwist(vec3 p, float k) { // rotates the xz plane by k radians per unit of y
  float c = cos(k * p.y), s = sin(k * p.y);
  return vec3(mat2(c, -s, s, c) * p.xz, p.y).xzy;
}
// map: return sdBox(opTwist(p, 2.0), vec3(0.3, 1.0, 0.3)); trace: t += d * 0.6; (bound 1.5 at r = 0.42)
```

## Combining Shapes

```glsl
float opUnion(float a, float b) { return min(a, b); }
float opIntersect(float a, float b) { return max(a, b); }
float opSubtract(float a, float b) { return max(a, -b); } // a with b carved out

// k = blend width in distance units; the field drops by at most k / 4 (where a == b), so the blend adds material
float smin(float a, float b, float k) {
  float h = max(k - abs(a - b), 0.0) / k;
  return min(a, b) - h * h * k * 0.25;
}

float smax(float a, float b, float k) { return -smin(-a, -b, k); }
```

- `smin(a, b, k)` equals `min(a, b)` wherever `|a - b| >= k`. `k` must be above 0: `k = 0` divides by zero and returns NaN. Smooth subtraction is `smax(a, -b, k)`, smooth intersection `smax(a, b, k)`.
- Intersection, subtraction and both smooth operators return a lower bound, not the exact distance. They are safe to trace; do not read the value as a physical distance away from the surface.
- The exponential smooth minimum overflows in float for large `k * distance`; the polynomial form above does not.
- Blend a color or material across the seam with a matching weight, 1 where `a` is closer: `float h = clamp(0.5 + 0.5 * (b - a) / k, 0.0, 1.0); vec3 col = mix(colB, colA, h);`.

## Normals

The normal is the normalized gradient of `map`. The Quick Start's `calcNormal` samples the four corners of a tetrahedron: 4 `map` calls. Central differences cost 6:

```glsl
vec3 calcNormalCentral(vec3 p) {
  const vec2 e = vec2(0.0005, 0.0);
  return normalize(vec3(map(p + e.xyy) - map(p - e.xyy),
                        map(p + e.yxy) - map(p - e.yxy),
                        map(p + e.yyx) - map(p - e.yyx)));
}
```

- Forward differences (`map(p + e) - map(p)`, also 4 calls) are biased and visibly worse than the tetrahedron.
- Epsilon too small: float noise and speckled normals far from the origin, since float32 keeps about 7 digits: the differences get noisy once `|p|` is around 1e5 times `h`, and `p + h == p` near 1e7 times `h`. Too large: edges look rounded. For large scenes scale `h` with `t` like the hit threshold.
- Start secondary rays offset along the normal (`p + n * 0.002`), or they hit the surface they start on.

## Soft Shadows and Ambient Occlusion

`softShadow` in the Quick Start (Inigo Quilez's form) returns 0 in shadow and 1 when lit. Along the ray toward the light (`rd` = unit vector to the light) it tracks the smallest ratio of distance-to-surface over distance travelled, `k * h / t`: a ray that passes close to an occluder early is darker. `k` sets the penumbra: about 8 is wide and soft, 32 nearly hard. The `clamp(h, 0.01, 0.25)` step bounds both the cost and how thin an occluder can be and still be found; widen it for large scenes.

Ambient occlusion samples the field a few times along the normal and compares it with the free-space distance:

```glsl
float ambientOcclusion(vec3 p, vec3 n) {
  float occ = 0.0, scale = 1.0;
  for (int i = 0; i < 5; i++) {
    float h = 0.01 + 0.12 * float(i) / 4.0;
    occ += (h - map(p + n * h)) * scale; // how much closer geometry is than free space would be
    scale *= 0.95;
  }
  return clamp(1.0 - 3.0 * occ, 0.0, 1.0);
}
```

Multiply ambient light by it, not direct light. AO needs correct interior distances and a field close to a true distance; over displaced or heavily relaxed fields it darkens too much.

## Fractals

The power-8 Mandelbulb distance estimator iterates `z -> z^8 + p` in spherical coordinates and tracks the running derivative `dr`:

```glsl
float sdMandelbulb(vec3 p) {
  vec3 z = p;
  float dr = 1.0;
  float r = length(z);
  for (int i = 0; i < 8; i++) {
    r = length(z);
    if (r > 2.0) break;
    float theta = acos(clamp(z.z / max(r, 1e-6), -1.0, 1.0));
    float phi = atan(z.y, z.x);
    dr = 8.0 * pow(r, 7.0) * dr + 1.0;
    float zr = pow(r, 8.0);
    z = zr * vec3(sin(theta * 8.0) * cos(phi * 8.0), sin(theta * 8.0) * sin(phi * 8.0), cos(theta * 8.0)) + p;
  }
  return 0.5 * log(max(r, 1e-6)) * r / dr;
}
```

- The guards matter: at the origin `z.z / r` is 0/0 and `log(0)` is -inf. One NaN pixel spreads through later blur or bloom passes as black blocks. `atan(y, x)` with both arguments zero is undefined in GLSL; drivers return a finite value, but do not depend on it.
- This is an estimate, not a guaranteed bound: march it with `t += d * 0.5` to `0.8` and a higher step cap. The iteration count (6–10) sets detail and cost.
- The shape fits inside a sphere of radius about 1.2: intersect the ray with that sphere first and march only inside it.

Folding fractals (Menger sponge, Mandelbox, kaleidoscopic IFS) build detail by repeatedly folding space and scaling it: `p = abs(p)` mirrors into one octant, `p = p - c * round(p / c)` or a box fold `clamp(p, -1.0, 1.0) * 2.0 - p` folds into a cell, then `p = p * s + offset`. Track the accumulated scale and divide the final distance by it, exactly as with uniform scaling above.

## Mixing with Rasterized Meshes

### Writing depth

When the SDF content is opaque, make the marcher part of the scene and let the depth buffer composite both ways: keep `depthTest` and `depthWrite` true, `discard` misses and write the hit's depth.

```glsl
uniform mat4 projectionMatrix; // not declared for the fragment stage by ShaderMaterial
varying vec3 vRayDir;
// map(), trace(), calcNormal() as in the Quick Start

void main() {
  vec3 ro = cameraPosition;
  vec3 rd = normalize(vRayDir);
  float t = trace(ro, rd);
  if (t < 0.0) discard;
  vec3 p = ro + rd * t;
  vec4 clip = projectionMatrix * viewMatrix * vec4(p, 1.0);
  gl_FragDepth = (clip.z / clip.w) * 0.5 + 0.5; // NDC z in [-1, 1] to window depth in [0, 1]
  gl_FragColor = vec4(calcNormal(p) * 0.5 + 0.5, 1.0);
  #include <tonemapping_fragment>
  #include <colorspace_fragment>
}
```

- `gl_FragDepth` is core in GLSL ES 3.00: no extension line, no `extensions` key. Once a shader writes it on one path it must write it on every non-discarded path.
- The formula is for the default depth buffer only. `WebGLRenderer({ logarithmicDepthBuffer: true })` and `({ reversedDepthBuffer: true })` change what the buffer stores.
- Writing depth disables early depth rejection for that draw, so the full march runs for every covered pixel. When the content is local, draw the marcher on a bounding box instead of the whole screen.
- Meshes still get no shadows from SDF shapes and the shadow map does not contain them; shadow the meshes yourself or leave them unshadowed.

### Reading scene depth in a pass

For a post pass, or for soft or transparent SDF content, render the scene into a target with a `DepthTexture`, then stop each ray at the scene surface. Render target and pass plumbing: `threejs-textures`, `threejs-postprocessing`.

```javascript
import * as THREE from "three";
import { FullScreenQuad } from "three/addons/postprocessing/Pass.js";

const w = Math.floor(window.innerWidth * renderer.getPixelRatio());
const h = Math.floor(window.innerHeight * renderer.getPixelRatio());
const sceneTarget = new THREE.WebGLRenderTarget(w, h, { depthTexture: new THREE.DepthTexture(w, h) });

const marchPass = new FullScreenQuad(new THREE.ShaderMaterial({
  uniforms: {
    tColor: { value: sceneTarget.texture },
    tDepth: { value: sceneTarget.depthTexture },
    uProjectionMatrixInverse: { value: camera.projectionMatrixInverse }, // the camera's own objects stay current
    uCameraMatrixWorld: { value: camera.matrixWorld },
    cameraNear: { value: camera.near },
    cameraFar: { value: camera.far },
  },
  vertexShader: /* glsl */ `varying vec2 vUv;
    void main() { vUv = uv; gl_Position = vec4(position.xy, 0.0, 1.0); }`,
  fragmentShader: /* glsl */ `
    #include <packing>
    uniform sampler2D tColor;
    uniform sampler2D tDepth;
    uniform mat4 uProjectionMatrixInverse;
    uniform mat4 uCameraMatrixWorld;
    uniform float cameraNear;
    uniform float cameraFar;
    varying vec2 vUv;

    // paste here: map() from the Quick Start, and its trace() with a third parameter tMax
    // replacing the far limit 50.0, so the ray stops at the mesh in front (does not compile without them)

    void main() {
      vec4 viewPos = uProjectionMatrixInverse * vec4(vUv * 2.0 - 1.0, -1.0, 1.0);
      vec3 rdView = normalize(viewPos.xyz / viewPos.w); // view space, z < 0
      vec3 ro = uCameraMatrixWorld[3].xyz;
      vec3 rd = normalize(mat3(uCameraMatrixWorld) * rdView);
      float viewZ = perspectiveDepthToViewZ(texture(tDepth, vUv).r, cameraNear, cameraFar); // negative
      float tScene = viewZ / rdView.z; // distance along this ray, not along the view axis
      vec3 col = texture(tColor, vUv).rgb;
      float t = trace(ro, rd, min(tScene, 50.0));
      if (t > 0.0) col = mix(col, vec3(0.2, 0.6, 1.0), 0.6); // translucent SDF over the scene
      gl_FragColor = vec4(col, 1.0);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
    }
  `,
}));

// per frame: scene into the target, then the pass to the canvas
renderer.setRenderTarget(sceneTarget);
renderer.render(scene, camera);
renderer.setRenderTarget(null);
marchPass.render(renderer);
```

- `-viewZ` is the distance along the camera's forward axis. Comparing ray length `t` with it is wrong by `1 / cos(angle from the view axis)`: right in the screen center, growing toward the corners. `viewZ / rdView.z` converts it.
- The scene in the target is linear and not tone mapped; the two includes apply tone mapping and the output color space when the pass draws to the canvas. Inside an EffectComposer leave them in and let `OutputPass` encode. Update `cameraNear` and `cameraFar` if the camera's planes change, and resize the target with the canvas.

## Performance Budgets

Cost per frame is roughly pixels × average steps × cost of `map`, plus, for hit pixels, 4 `map` calls for the normal, up to 48 for the shadow and 5 for AO. The levers, largest first:

1. Fewer pixels. Draw the marcher on a bounding box or sphere proxy instead of the whole screen. Lower `renderer.setPixelRatio` (a 2× DPR screen has 4× the pixels), or march into a half-resolution target and upsample.
2. Start and stop at the bounds. Intersect the ray with a bounding sphere or box analytically, start `t` at the entry and stop at the exit.
3. Cheaper `map`. Bound a group of primitives with one cheap distance (a sphere around them) and evaluate the group only when that distance is small.
4. Step caps, as starting points rather than measurements: 64–128 for analytic scenes, 150–250 for fractals, 24–48 for shadows, 5 for AO. Measure on the target device (`threejs-performance`).
5. Constant loop bounds, with quality levels as `#define`s in the material's `defines`, so the compiler can unroll and you can switch tiers.

## WebGPU and TSL

ShaderMaterial does not run under WebGPURenderer; the marcher is a node material. Here it is a `MeshBasicNodeMaterial` on a box with `side: THREE.BackSide`, so it still draws when the camera is inside the box. World-space rays start at `cameraPosition` and pass through `positionWorld`. Values changed inside `Loop` or `If` are `.toVar()` variables updated with `.assign()` and `.addAssign()`.

```javascript
import * as THREE from "three/webgpu";
import { Break, Fn, If, Loop, cameraPosition, dot, float, length, max, normalize, positionWorld, vec2, vec3 } from "three/tsl";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x8fb4d9); // the WebGPU canvas is transparent by default
const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 5);

const map = Fn(([p]) => length(p).sub(1.0)); // sphere of radius 1 at the origin

const calcNormal = Fn(([p]) => {
  const k = vec2(1, -1);
  const h = float(0.0005);
  return normalize(
    k.xyy.mul(map(p.add(k.xyy.mul(h))))
      .add(k.yyx.mul(map(p.add(k.yyx.mul(h)))))
      .add(k.yxy.mul(map(p.add(k.yxy.mul(h)))))
      .add(k.xxx.mul(map(p.add(k.xxx.mul(h))))),
  );
});

const material = new THREE.MeshBasicNodeMaterial({ side: THREE.BackSide });
material.colorNode = Fn(() => {
  const ro = cameraPosition;
  const rd = normalize(positionWorld.sub(cameraPosition));
  const t = float(0).toVar();
  const hit = float(0).toVar();
  Loop(96, () => {
    const d = map(ro.add(rd.mul(t)));
    If(d.lessThan(t.mul(0.0005).add(0.0001)), () => {
      hit.assign(1);
      Break();
    });
    t.addAssign(d);
    If(t.greaterThan(50), () => {
      Break();
    });
  });
  hit.lessThan(0.5).discard();
  const n = calcNormal(ro.add(rd.mul(t)));
  const light = max(dot(n, normalize(vec3(0.5, 0.8, 0.3))), 0.0);
  return vec3(light.mul(0.9).add(0.1));
})();

scene.add(new THREE.Mesh(new THREE.BoxGeometry(4, 4, 4), material)); // the SDF content must fit inside the box

renderer.setAnimationLoop(() => renderer.render(scene, camera));
```

- `MeshBasicNodeMaterial` output goes through the renderer's tone mapping and output color space conversion; there are no includes to add.
- The box is drawn with its back faces, so the depth buffer holds the box, not the hit. `material.depthNode` sets the fragment depth instead; for a perspective camera and the default depth buffer the value is `viewZToPerspectiveDepth(viewZ, cameraNear, cameraFar)` with `viewZ` from `cameraViewMatrix.mul(vec4(p, 1)).z`.
- `RaymarchingBox(steps, ({ positionRay }) => { ... })` from `three/addons/tsl/utils/Raymarching.js` is a fixed-step march through a unit box in object space, for volumes, not a sphere tracer; see `threejs-math-atmosphere`.

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
- `softShadow(p, sunDir, 0.0, 10.0, 16.0)` → `softShadow(p + n * 0.002, sunDir, 0.02, 10.0, 16.0)`: a shadow ray starting on the surface reports a hit at once and the surface shadows itself.
- `float theta = acos(z.z / r);` → `acos(clamp(z.z / max(r, 1e-6), -1.0, 1.0))`: 0/0 at the origin gives NaN, which later blur or bloom passes spread into black blocks.
- `if (t > texture(tDepth, vUv).r)` → `perspectiveDepthToViewZ(depth, cameraNear, cameraFar) / rdView.z`: the depth buffer is non-linear, and view depth is not ray length away from the screen center.
- `gl_FragDepth = t / cameraFar;` → `vec4 clip = projectionMatrix * viewMatrix * vec4(p, 1.0); gl_FragDepth = clip.z / clip.w * 0.5 + 0.5;`: the depth buffer stores projected depth, so a linear value meets meshes at the wrong place.
- `projectionMatrix` used in the fragment shader undeclared → `uniform mat4 projectionMatrix;`: ShaderMaterial declares it for the vertex stage only, so the shader fails to compile.

## See Also

- `threejs-shaders` — ShaderMaterial rules, the two output includes, TSL basics and porting GLSL to TSL.
- `threejs-postprocessing` — EffectComposer, `OutputPass` and the WebGPU render pipeline for a raymarch pass over the scene.
- `threejs-textures` — render targets and `DepthTexture` setup for depth compositing.
- `threejs-math-noise` — noise and fBm for displacing SDF surfaces.
- `threejs-math-atmosphere` — fixed-step volume marching, fog, clouds and `RaymarchingBox`.
- `threejs-math-procedural-geometry` — marching cubes, turning an SDF into a triangle mesh.
