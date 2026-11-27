# GLSL patterns, onBeforeCompile and chunks

Purpose: ready-made GLSL effects, custom attributes on instanced meshes, extending built-in materials and debugging compile errors (r187, WebGLRenderer); open when writing a specific effect. Every fragment shader here ends with the two output includes explained in `glsl-reference.md`.

## Vertex displacement

```javascript
const material = new THREE.ShaderMaterial({
  uniforms: {
    time: { value: 0 },
    amplitude: { value: 0.2 },
  },
  vertexShader: /* glsl */ `
    uniform float time;
    uniform float amplitude;
    void main() {
      vec3 pos = position;
      pos.z += sin(pos.x * 5.0 + time) * amplitude;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    void main() {
      gl_FragColor = vec4(0.5, 0.8, 1.0, 1.0);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
    }
  `,
});

const mesh = new THREE.Mesh(new THREE.PlaneGeometry(2, 2, 64, 64), material); // displacement needs subdivisions
```

## Fresnel and rim

Use one space consistently. World space with `cameraPosition`:

```javascript
const material = new THREE.ShaderMaterial({
  vertexShader: /* glsl */ `
    varying vec3 vWorldNormal;
    varying vec3 vWorldPosition;
    void main() {
      vWorldNormal = normalize(mat3(modelMatrix) * normal); // world space, uniform scale assumed
      vWorldPosition = (modelMatrix * vec4(position, 1.0)).xyz;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    varying vec3 vWorldNormal;
    varying vec3 vWorldPosition;
    void main() {
      vec3 viewDirection = normalize(cameraPosition - vWorldPosition);
      float fresnel = pow(1.0 - max(dot(viewDirection, normalize(vWorldNormal)), 0.0), 3.0);
      gl_FragColor = vec4(mix(vec3(0.0, 0.0, 0.5), vec3(0.5, 0.8, 1.0), fresnel), 1.0);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
    }
  `,
});
```

`normalMatrix * normal` is view space - pair it with the view-space direction to the camera, never with `cameraPosition`. Rim lighting in view space: in the vertex shader `vec4 mvPosition = modelViewMatrix * vec4(position, 1.0); vViewDir = -mvPosition.xyz; vNormal = normalMatrix * normal;`, in the fragment shader `float rim = pow(1.0 - max(dot(normalize(vViewDir), normalize(vNormal)), 0.0), 4.0);`. Non-uniformly scaled meshes need `normalMatrix` (or the inverse-transpose of `modelMatrix`) rather than `mat3(modelMatrix)`.

## Dissolve

```glsl
uniform float progress;
uniform sampler2D noiseMap; // data texture: keep NoColorSpace
varying vec2 vUv;

void main() {
  float noise = texture(noiseMap, vUv).r;
  if (noise < progress) discard;

  float edge = smoothstep(progress, progress + 0.1, noise);
  gl_FragColor = vec4(mix(vec3(1.0, 0.5, 0.0), vec3(0.5), edge), 1.0);
  #include <tonemapping_fragment>
  #include <colorspace_fragment>
}
```

Noise and hash functions to generate `noise` procedurally: `threejs-math-noise`.

## Instanced meshes and custom attributes

three.js declares `position`, `normal`, `uv`, `instanceMatrix` (InstancedMesh), `instanceColor` (once `setColorAt` has been called) and `color` (with `vertexColors: true`). Anything else - a custom per-vertex or per-instance attribute, `uv1`-`uv3`, `tangent` - you declare with `attribute` (the alias for `in`).

```javascript
const instanceCount = 100;
const geometry = new THREE.PlaneGeometry(0.2, 0.2);
const offsets = new Float32Array(instanceCount * 3); // fill with one vec3 per instance
geometry.setAttribute("aOffset", new THREE.InstancedBufferAttribute(offsets, 3));

const material = new THREE.ShaderMaterial({
  vertexShader: /* glsl */ `
    attribute vec3 aOffset;
    void main() {
      // instanceMatrix is declared by three.js; the shader must apply it
      gl_Position = projectionMatrix * modelViewMatrix * instanceMatrix * vec4(position + aOffset, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    void main() {
      gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
    }
  `,
});

const mesh = new THREE.InstancedMesh(geometry, material, instanceCount);
mesh.frustumCulled = false; // the bounds come from instanceMatrix and ignore aOffset
```

A per-vertex attribute uses `new THREE.BufferAttribute(new Float32Array(geometry.attributes.position.count * 3), 3)` with the same declaration. InstancedMesh API itself: `threejs-geometry`.

## Extending built-in materials with onBeforeCompile

`onBeforeCompile(shader, renderer)` receives the shader strings before `#include` lines are resolved and before the prefix is added, once per program. Replace an include with itself plus your code.

```javascript
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });

material.onBeforeCompile = (shader) => {
  shader.uniforms.time = { value: 0 };
  material.userData.shader = shader; // keep the reference for updates

  shader.vertexShader = "uniform float time;\n" + shader.vertexShader;
  shader.vertexShader = shader.vertexShader.replace(
    "#include <begin_vertex>",
    `
    #include <begin_vertex>
    transformed.y += sin(position.x * 10.0 + time) * 0.1;
    `,
  );
};

// in the frame loop, after timer.update():
// if (material.userData.shader) material.userData.shader.uniforms.time.value = timer.getElapsed();
```

Injection points:

| Chunk | Stage | What it provides |
| --- | --- | --- |
| `#include <begin_vertex>` | vertex | declares `vec3 transformed = vec3(position)` |
| `#include <beginnormal_vertex>` | vertex | declares `vec3 objectNormal` |
| `#include <project_vertex>` | vertex | computes `mvPosition` and sets `gl_Position` |
| `#include <color_fragment>` | fragment | `diffuseColor` after map and vertex colors |
| `#include <opaque_fragment>` | fragment | writes `gl_FragColor = vec4(outgoingLight, diffuseColor.a)`, before tone mapping |
| `#include <tonemapping_fragment>`, `<colorspace_fragment>`, `<fog_fragment>`, `<premultiplied_alpha_fragment>`, `<dithering_fragment>` | fragment | the tail, in that order |

Displacing in `begin_vertex` moves the shadow too only if the depth material gets the same change: set `mesh.customDepthMaterial` (and `customDistanceMaterial` for point-light shadows) to a MeshDepthMaterial with the same `onBeforeCompile`.

If `onBeforeCompile` produces different code for different materials (for example from `material.userData`), also define `material.customProgramCacheKey = () => "variant-" + material.userData.mode;`, or the renderer reuses one cached program for all of them.

## Shader chunks in your own shaders

`#include <chunk>` works in ShaderMaterial and RawShaderMaterial source and reuses three.js helper functions; `perspectiveDepthToViewZ` comes from the `packing` chunk.

```javascript
const fragmentShader = /* glsl */ `
  #include <packing>

  uniform sampler2D depthTexture;
  uniform float cameraNear; // set from camera.near
  uniform float cameraFar; // set from camera.far
  varying vec2 vUv;

  void main() {
    float viewZ = perspectiveDepthToViewZ(texture(depthTexture, vUv).r, cameraNear, cameraFar);
    gl_FragColor = vec4(vec3(-viewZ / cameraFar), 1.0);
  }
`;
```

Only use chunk names that exist in this release - log `Object.keys(THREE.ShaderChunk)`. Renamed chunks that old tutorials still use: `output_fragment` → `opaque_fragment` and `encodings_fragment` → `colorspace_fragment` (both r154); the `uv2_*` chunks were removed in r151. In r185 `inverseTransformDirection()` was deprecated in favour of `transformNormalByInverseViewMatrix()` and `transformDirectionByInverseViewMatrix()`.

## Debugging

```javascript
// While renderer.debug.checkShaderErrors is true (the default), a failed program logs each failing
// shader's errors with numbered source lines. A handler replaces that report and runs only on failure:
renderer.debug.onShaderError = (gl, program, vertexShader, fragmentShader) => {
  console.error(gl.getProgramInfoLog(program));
  for (const shader of [vertexShader, fragmentShader]) {
    const log = gl.getShaderInfoLog(shader);
    if (log) console.error(log, gl.getShaderSource(shader)); // the fully assembled shader
  }
};
```

`onBeforeCompile` cannot show the final code (it runs before includes and the prefix); `onShaderError` receives the assembled shaders. Debug visually by writing the value under test: `gl_FragColor = vec4(vUv, 0.0, 1.0);` for UVs, `gl_FragColor = vec4(vNormal * 0.5 + 0.5, 1.0);` for normals. Set `checkShaderErrors = false` in production to skip the synchronous error query.
