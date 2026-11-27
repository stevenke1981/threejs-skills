# Camera rays, depth compositing, budgets and WebGPU

Purpose: building rays from the three.js camera (perspective and orthographic), writing `gl_FragDepth` so SDF shapes and meshes occlude each other, stopping rays at the scene depth in a post pass, performance levers, and the TSL marcher for WebGPURenderer with `depthNode`. Open it when the SDF scene must line up with rasterized meshes, run fast, or run under WebGPU.

Checked against the r187 sources: `src/renderers/webgl/WebGLProgram.js` (built-in uniform declarations), `src/renderers/shaders/ShaderChunk/packing.glsl.js` (`perspectiveDepthToViewZ`), `examples/jsm/postprocessing/Pass.js` (`FullScreenQuad`), `src/renderers/WebGLRenderer.js` (`reversedDepthBuffer`), `src/materials/nodes/NodeMaterial.js` (`depthNode`), `src/nodes/display/ViewportDepthNode.js` and `examples/jsm/tsl/utils/Raymarching.js`.

## Camera rays

The Quick Start vertex shader unprojects each corner of the quad onto the near plane with `inverse(projectionMatrix)` and rotates it to world space with `transpose(mat3(viewMatrix))` (a rotation's inverse is its transpose). Any clip-space z lies on the same eye ray for a perspective camera; -1 keeps the four corners on one plane, so linear interpolation across the quad is exact. Field of view, aspect and orientation all come from the matrices, so the SDF scene lines up with rasterized meshes and follows `camera.fov`, resizes and OrbitControls.

- A mesh placed in the scene receives the scene camera's `projectionMatrix`, `viewMatrix` and `cameraPosition`.
- The ShaderMaterial vertex prefix declares `modelMatrix`, `modelViewMatrix`, `projectionMatrix`, `viewMatrix`, `normalMatrix`, `cameraPosition` and `isOrthographic`; the fragment prefix declares only `viewMatrix`, `cameraPosition` and `isOrthographic`. A fragment shader that needs `projectionMatrix` declares `uniform mat4 projectionMatrix;` itself and the renderer fills it.
- `mesh.frustumCulled = false;` is required for a clip-space quad: culling tests the plane's object-space bounding sphere at the mesh position.
- `FullScreenQuad` from `three/addons/postprocessing/Pass.js` draws one oversized triangle with its own `OrthographicCamera(-1, 1, 1, -1, 0, 1)`, so inside it the built-in camera uniforms describe that camera. Pass the scene camera's matrices as your own uniforms (below).
- Interpolated unit vectors are shorter than unit length: normalize `vRayDir` in the fragment shader.

Orthographic camera: all rays are parallel and the origin moves across the near plane.

```glsl
vec3 rd = -transpose(mat3(viewMatrix))[2]; // the camera's forward axis in world space
vec3 ro = (inverse(viewMatrix) * vec4(viewPos.xyz / viewPos.w, 1.0)).xyz; // pass ro as a varying
```

## Writing depth

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

- `gl_FragDepth` is core in GLSL ES 3.00: no extension line, no `extensions` key. Once a shader writes it on one path it must write it on every non-discarded path; otherwise the value on the other paths is undefined.
- The formula is for the default depth buffer only. `WebGLRenderer({ logarithmicDepthBuffer: true })` and `({ reversedDepthBuffer: true })` (needs `EXT_clip_control`) change what the buffer stores; three's shaders see them as `USE_LOGARITHMIC_DEPTH_BUFFER` and `USE_REVERSED_DEPTH_BUFFER` defines.
- Writing depth disables early depth rejection for that draw, so the full march runs for every covered pixel. When the content is local, draw the marcher on a bounding box instead of the whole screen.
- Meshes get no shadows from SDF shapes and the shadow map does not contain them; shadow the meshes yourself or leave them unshadowed.

## Reading scene depth in a pass

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
- `perspectiveDepthToViewZ` in the `packing` chunk already handles `USE_REVERSED_DEPTH_BUFFER`; a logarithmic depth buffer needs a different conversion.
- The scene in the target is linear and not tone mapped; the two includes apply tone mapping and the output color space when the pass draws to the canvas. Inside an EffectComposer leave them in and let `OutputPass` encode. Update `cameraNear` and `cameraFar` if the camera's planes change, and resize the target with the canvas.

## Performance budgets

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
await renderer.init(); // a render() outside setAnimationLoop throws before init (r186, r187)
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
- The box is drawn with its back faces, so the depth buffer holds the box, not the hit. `material.depthNode` sets the fragment depth instead; for a perspective camera and the default depth buffer the value is `viewZToPerspectiveDepth(viewZ, cameraNear, cameraFar)` (all three from `three/tsl`) with `viewZ` from `cameraViewMatrix.mul(vec4(p, 1)).z`. Writing depth costs early-z here too.
- A plain `Fn(([p]) => ...)` without a layout is inlined at every call, so `calcNormal` expands `map` four times; give heavy functions a layout (`Fn(fn, { p: 'vec3', return: 'float' })`, the form r187 uses throughout its own nodes) to emit a real shader function.
- `RaymarchingBox(steps, ({ positionRay, stepSize }) => { ... })` from `three/addons/tsl/utils/Raymarching.js` is a fixed-step march through the unit box `[-0.5, 0.5]^3` in object space, for volumes, not a sphere tracer; see `threejs-math-atmosphere`.
