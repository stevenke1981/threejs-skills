---
name: threejs-shaders
description: "Custom three.js shaders: ShaderMaterial and RawShaderMaterial GLSL, uniforms, varyings, shader chunks, onBeforeCompile on built-in materials, instancing, tone-mapped output, and custom TSL node logic (Fn, uniform, varying) for WebGPURenderer. Use when writing a shader, on a shader compile error, porting GLSL to TSL, or when ShaderMaterial logs \"is not compatible\". Not for picking built-in materials (use threejs-materials), full-screen passes (use threejs-postprocessing), noise or SDF maths (use threejs-math-noise, threejs-math-sdf-raymarching). 中文: 著色器, 自訂著色器, GLSL, TSL, 頂點位移, 著色器編譯錯誤"
---

# Three.js Shaders

Targets three.js r187 (npm `three@0.187.0`). GLSL examples use WebGLRenderer; TSL examples import from `three/webgpu` and `three/tsl` and run on WebGPURenderer.

## Quick Start (GLSL, WebGLRenderer)

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 2);

const material = new THREE.ShaderMaterial({
  uniforms: {
    time: { value: 0 },
    color: { value: new THREE.Color(0x0066ff) }, // hex converted to the linear working space
  },
  vertexShader: /* glsl */ `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    uniform float time;
    uniform vec3 color;
    varying vec2 vUv;
    void main() {
      float wave = sin(vUv.x * 10.0 + time * 2.0) * 0.5 + 0.5;
      gl_FragColor = vec4(color * wave, 1.0);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
    }
  `,
});

scene.add(new THREE.Mesh(new THREE.PlaneGeometry(2, 2), material));

const timer = new THREE.Timer(); // Clock is deprecated since r183
timer.connect(document);

renderer.setAnimationLoop(() => {
  timer.update(); // first thing every frame
  material.uniforms.time.value = timer.getElapsed();
  renderer.render(scene, camera);
});
```

## Choosing the Approach

| Need | Use | Renderer |
| --- | --- | --- |
| Full custom look, three.js declares matrices/attributes for you | `ShaderMaterial` | WebGLRenderer only |
| Complete control of the source, nothing injected | `RawShaderMaterial` | WebGLRenderer only |
| Keep a built-in material's lighting/shadows, change a few lines | `material.onBeforeCompile` | WebGLRenderer only |
| Anything under WebGPURenderer, or one code path for WebGPU + WebGL 2 | node material + TSL | WebGPURenderer (WebGL 2 fallback built in) |
| Whole-screen effect (bloom, color grade) | a post-processing pass | see `threejs-postprocessing` |

Under WebGPURenderer, ShaderMaterial and RawShaderMaterial log `NodeBuilder: Material "ShaderMaterial" is not compatible.` and render with a plain fallback, and `onBeforeCompile` is never called. Port to TSL instead (`references/tsl.md`).

## Rules That Break Shaders Most Often

1. ShaderMaterial always compiles as GLSL ES 3.00 (`#version 300 es` is prepended). `texture()`, `textureLod()`, `dFdx`, `gl_FragDepth` work with no flags, and `texture2D`, `textureCube`, `attribute`, `varying`, `gl_FragColor` remain as `#define` aliases. Leave `glslVersion` unset: `THREE.GLSL3` only removes the `gl_FragColor` alias.
2. RawShaderMaterial gets nothing but `SHADER_TYPE`/`SHADER_NAME`/your `defines`: set `glslVersion: THREE.GLSL3` (otherwise GLSL ES 1.00, no `texture()`), write `precision highp float;`, `in`/`out`, your own output variable, and declare every matrix and attribute.
3. Built-in matrix uniforms (`modelMatrix`, `modelViewMatrix`, `projectionMatrix`, `viewMatrix`, `normalMatrix`, `cameraPosition`) are set by the renderer for any program that declares them - never pass them through `uniforms`, and never redeclare them in a ShaderMaterial (already declared → compile error).
4. ShaderMaterial declares `position`, `normal`, `uv`, `instanceMatrix` (InstancedMesh), `instanceColor` (after `setColorAt`), `color` (with `vertexColors: true`). `uv1`-`uv3`, `tangent` and custom attributes need your own `attribute` declaration.
5. Output is written as-is. End the fragment shader with `#include <tonemapping_fragment>` and `#include <colorspace_fragment>`, each on its own line, or colors look darker than the same hex on built-in materials. Into a render target both leave the color linear, as built-in materials do.
6. ShaderMaterial does not upload `material.opacity`, `color` or any material property - everything the shader reads is a uniform you declare.
7. On an InstancedMesh the shader must apply `instanceMatrix` itself: `projectionMatrix * modelViewMatrix * instanceMatrix * vec4(position, 1.0)`.

Full declaration lists, uniform types and color handling: `references/glsl-reference.md`.

## Extending Built-in Materials (summary)

```javascript
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });

material.onBeforeCompile = (shader) => {
  shader.uniforms.time = { value: 0 };
  material.userData.shader = shader; // keep the reference for per-frame updates
  shader.vertexShader = "uniform float time;\n" + shader.vertexShader.replace(
    "#include <begin_vertex>",
    "#include <begin_vertex>\n  transformed.y += sin(position.x * 10.0 + time) * 0.1;",
  );
};
// per frame: if (material.userData.shader) material.userData.shader.uniforms.time.value = timer.getElapsed();
```

Injection points, `customProgramCacheKey()` for per-material variants, and the chunk list: `references/glsl-patterns.md`.

## TSL in One Example (WebGPURenderer)

```javascript
import * as THREE from "three/webgpu";
import { color, mix, sin, time, uv, uniform } from "three/tsl";

const speed = uniform(1.0); // change speed.value from JS; no recompile
const material = new THREE.MeshBasicNodeMaterial(); // unlit; MeshStandardNodeMaterial is lit
material.colorNode = mix(color(0x004466), color(0x66ddff), sin(uv().x.mul(20).add(time.mul(speed))).mul(0.5).add(0.5));
```

`time` updates itself every frame. Reusable logic is a `Fn`, vertex-to-fragment values use `varying(node, "name")`, vertex movement goes in `positionNode` built on `positionLocal`. Which node material class and slot to use: `threejs-materials` (`references/node-materials.md`). Authoring patterns, GLSL-to-TSL mapping and renamed functions: `references/tsl.md`.

## Common Mistakes

- `"#include <output_fragment>"` → `"#include <opaque_fragment>"`: output_fragment was renamed in r154, so a string replace on it silently matches nothing.
- A fragment shader without the two output includes → add `#include <tonemapping_fragment>` and `#include <colorspace_fragment>` after writing `gl_FragColor`.
- `new THREE.ShaderMaterial({ glslVersion: THREE.GLSL3, ... })` with `gl_FragColor` → drop `glslVersion`: GLSL3 removes the alias and the shader stops compiling.
- `new THREE.RawShaderMaterial({ vertexShader, fragmentShader })` → add `glslVersion: THREE.GLSL3`: otherwise it is GLSL ES 1.00 with no `texture()`.
- `extensions: { derivatives: true, fragDepth: true }` → remove: the WebGL 1 extension keys went away with WebGL 1 support (r163); `extensions` holds only `clipCullDistance` and `multiDraw`.
- `uniforms: { modelViewMatrix: { value: new THREE.Matrix4() } }` → declare `uniform mat4 modelViewMatrix;` in a RawShaderMaterial only: the renderer overwrites it every draw.
- `uniforms: { position: ... }` → rename (`offset`): collides with the declared attribute.
- `new THREE.ShaderMaterial({ transparent: true, opacity: 0.5 })` → add `uniforms: { opacity: { value: 0.5 } }` and write it to alpha.
- `blending: THREE.MultiplyBlending` → add `premultipliedAlpha: true`, and premultiply the output with `#include <premultiplied_alpha_fragment>`.
- `varying vec2 vUv1; void main() { vUv1 = uv1; }` → add `attribute vec2 uv1;`.
- `dot(normalize(cameraPosition - vWorldPosition), normalize(normalMatrix * normal))` → use `mat3(modelMatrix) * normal`: normalMatrix gives view-space normals, cameraPosition is world space.
- Same `onBeforeCompile` producing different shaders per material → also define `material.customProgramCacheKey = () => variantKey;` or all share one cached program.
- `new THREE.ShaderMaterial(...)` under WebGPURenderer → a node material with TSL.
- `import { time } from "three/nodes"` → `import { time } from "three/tsl"`.
- `material.positionNode = positionGeometry.add(offset)` → `positionLocal.add(offset)`: positionGeometry drops skinning, morphs, displacement, batching and instancing.
- `toVarying(node)` imported from `three/tsl` → `varying(node, "vName")` (or the method `node.toVarying("vName")`): `toVarying` is not exported.
- `glslFn(glslSource)` for a portable effect → rewrite as a TSL `Fn`: native-code nodes compile only for their own backend.

## Reference Files

- `references/glsl-reference.md` - what ShaderMaterial and RawShaderMaterial prepend, GLSL ES 3.00 aliases, uniform types, output color, video textures. Open when a shader fails to compile or colors are off.
- `references/glsl-patterns.md` - vertex displacement, fresnel and rim, dissolve, instanced and custom attributes, onBeforeCompile injection points, shader chunks, debugging. Open when writing a specific effect.
- `references/tsl.md` - TSL authoring: uniforms and time, textures and discard, displacement, Fn and varyings, GLSL-to-TSL mapping, renamed and removed TSL names, r187 type coercion, debugging generated code. Open for any WebGPURenderer shader work.

## See Also

- `threejs-materials` - built-in materials and node material classes, when a standard material already does the job
- `threejs-textures` - color spaces, UV channels and render targets sampled in shaders
- `threejs-postprocessing` - full-screen passes instead of per-object shaders
- `threejs-math-noise`, `threejs-math-sdf-raymarching` - the maths inside procedural shaders
- `shader-programming` - engine-neutral shader concepts
