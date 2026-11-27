# GLSL reference for ShaderMaterial and RawShaderMaterial

Purpose: exactly what three.js r187 prepends to your GLSL, how uniforms map to GLSL types, and how output color is encoded; open when a shader fails to compile or its colors are off. WebGLRenderer only.

## What ShaderMaterial prepends

Every ShaderMaterial (and every built-in material) is compiled as GLSL ES 3.00: three.js writes `#version 300 es` (WebGL 1 support was removed in r163) followed by:

- Vertex aliases: `#define attribute in`, `#define varying out`, `#define texture2D texture`.
- Fragment aliases: `#define varying in`, `layout(location = 0) out highp vec4 pc_fragColor;` with `#define gl_FragColor pc_fragColor` (both omitted when `glslVersion: THREE.GLSL3`), `gl_FragDepthEXT` → `gl_FragDepth`, `texture2D`/`textureCube` → `texture`, `texture2DLodEXT`/`textureCubeLodEXT` → `textureLod`, and the `Proj`/`Grad` variants.
- A precision statement, `SHADER_TYPE`/`SHADER_NAME` and your `defines`.
- Vertex declarations: `uniform mat4 modelMatrix, modelViewMatrix, projectionMatrix, viewMatrix; uniform mat3 normalMatrix; uniform vec3 cameraPosition; uniform bool isOrthographic;` and `attribute vec3 position; attribute vec3 normal; attribute vec2 uv;`, plus, when they apply, `attribute mat4 instanceMatrix` (InstancedMesh), `attribute vec3 instanceColor` (InstancedMesh with `instanceColor`, created by `setColorAt`), `attribute vec3 color` or `vec4 color` (`vertexColors: true`), `skinIndex`/`skinWeight` (SkinnedMesh).
- Fragment declarations: `uniform mat4 viewMatrix; uniform vec3 cameraPosition; uniform bool isOrthographic;`, the tone-mapping functions when `renderer.toneMapping` is not `NoToneMapping`, and the color-space helpers (`sRGBTransferEOTF`, `sRGBTransferOETF`, `linearToOutputTexel`).

`uv1`, `uv2`, `uv3` and `tangent` are declared only when a built-in map needs them, which never happens for a ShaderMaterial - declare them yourself (`attribute vec2 uv1;`). An `#extension` line in your source fails to compile because the declarations come first; delete it, the features are core in GLSL ES 3.00. Shaders that name a variable `texture` or write `gl_FragData` also fail under 3.00.

`material.extensions` accepts only `clipCullDistance` and `multiDraw` (both default `false`).

## RawShaderMaterial

Prepended: only `#define SHADER_TYPE`, `#define SHADER_NAME` and your `defines`, plus `#version` from `glslVersion` if set. No precision statement, no declarations, no aliases.

```javascript
const material = new THREE.RawShaderMaterial({
  glslVersion: THREE.GLSL3, // without it the shader is GLSL ES 1.00, which has no texture()
  vertexShader: /* glsl */ `
    precision highp float;
    in vec3 position;
    uniform mat4 modelViewMatrix;
    uniform mat4 projectionMatrix;
    void main() {
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    precision highp float;
    out highp vec4 fragColor; // your own output variable
    void main() {
      fragColor = vec4(1.0, 0.0, 0.0, 1.0);
    }
  `,
});
```

The renderer still sets `modelMatrix`, `modelViewMatrix`, `projectionMatrix`, `viewMatrix`, `normalMatrix` and `cameraPosition` for any program that declares them. A same-named entry in `uniforms` is overwritten every draw (`modelMatrix`, `modelViewMatrix`, `normalMatrix`) or replaces the camera's value (`projectionMatrix`, `viewMatrix`, `cameraPosition`).

## Uniforms

```javascript
const map = await new THREE.TextureLoader().loadAsync("textures/diffuse.jpg");
map.colorSpace = THREE.SRGBColorSpace; // a color image: decoded to linear when sampled

const material = new THREE.ShaderMaterial({
  uniforms: {
    floatValue: { value: 1.5 },
    vec2Value: { value: new THREE.Vector2(1, 2) },
    vec3Value: { value: new THREE.Vector3(1, 2, 3) },
    colorValue: { value: new THREE.Color(0xff0000) }, // hex converted to the linear working space
    mat4Value: { value: new THREE.Matrix4() },
    mapValue: { value: map },
    floatArray: { value: [0.1, 0.2, 0.3] }, // uniform float floatArray[3];
  },
  // vertexShader and fragmentShader as usual
});
```

```glsl
uniform float floatValue;
uniform vec2 vec2Value;
uniform vec3 vec3Value;
uniform vec3 colorValue;    // a THREE.Color arrives as vec3
uniform mat4 mat4Value;
uniform sampler2D mapValue; // sample with texture(mapValue, vUv)
uniform float floatArray[3];
```

Update by assigning `.value` (`material.uniforms.floatValue.value = 2.0;`); vectors and colors mutate in place (`material.uniforms.vec3Value.value.set(1, 2, 3);`). `material.clone()` deep-clones the uniforms object, so clones animate independently. Never name a uniform `position`, `normal`, `uv`, `modelViewMatrix` or `cameraPosition` - they are already declared.

ShaderMaterial flags, all default `false`: `lights: true` makes the renderer fill the scene's light uniforms, which must already exist in `uniforms` (merge them in with `THREE.UniformsUtils.merge([THREE.UniformsLib.lights, myUniforms])`); `fog: true` does the same for `THREE.UniformsLib.fog`; `clipping: true` enables clipping planes (the shader must include the clipping chunks).

## Output color and tone mapping

`THREE.Color` converts hex input from sRGB to the linear working color space, so a color uniform holds linear values. A ShaderMaterial writes its output as-is; tone mapping and output encoding happen only if the fragment shader ends with:

```glsl
gl_FragColor = vec4(color, 1.0);
#include <tonemapping_fragment>
#include <colorspace_fragment>
```

- `tonemapping_fragment` applies `renderer.toneMapping` (a no-op with `NoToneMapping` or `toneMapped: false`).
- `colorspace_fragment` encodes to `renderer.outputColorSpace` (`SRGBColorSpace` by default).
- Both chunks write `gl_FragColor`, so they need `glslVersion` unset.
- Rendering into a render target (an EffectComposer pass, for example), both leave the color linear; the final `OutputPass` encodes it (see `threejs-postprocessing`).
- An 8-bit texture with `colorSpace = SRGBColorSpace` decodes to linear on sampling, so sampled colors combine correctly with the includes.
- A `VideoTexture` is the exception: it is uploaded without sRGB decoding and only built-in materials decode it in the shader. In your own shader decode it with `sRGBTransferEOTF(texture(videoMap, vUv))` (the ShaderMaterial prefix declares that function).
- Premultiplied blending: end with `#include <premultiplied_alpha_fragment>` (active when `premultipliedAlpha: true`).
