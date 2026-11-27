# Render targets and CubeCamera

Purpose: rendering into textures in r187 - render targets, depth textures, MSAA, multiple attachments, cube targets for dynamic reflections, and the WebGPU classes; open for any offscreen rendering. Full-screen effect chains are in `threejs-postprocessing`.

## Classes

| Class | Import | Use |
| --- | --- | --- |
| `WebGLRenderTarget` | `three` (also re-exported by `three/webgpu`) | 2D target |
| `WebGLCubeRenderTarget` | `three` only | cube target for WebGLRenderer / CubeCamera |
| `WebGLArrayRenderTarget`, `WebGL3DRenderTarget` | `three` | layered and volume targets |
| `RenderTarget`, `RenderTarget3D` | `three` and `three/webgpu` | renderer-agnostic base classes; the usual 2D target under WebGPURenderer |
| `CubeRenderTarget` | `three/webgpu` | cube target for WebGPURenderer (required since r183) |

## Render to texture

```javascript
const renderTarget = new THREE.WebGLRenderTarget(512, 512, {
  type: THREE.HalfFloatType, // linear HDR content without 8-bit banding; default UnsignedByteType
  samples: 4, // MSAA, resolved into .texture after rendering; default 0
});
const rtScene = new THREE.Scene(); // must not contain the mesh that samples the target
rtScene.add(new THREE.Mesh(new THREE.BoxGeometry(), new THREE.MeshNormalMaterial())); // unlit, visible without lights
const rtCamera = new THREE.PerspectiveCamera(50, 1, 0.1, 10);
rtCamera.position.z = 3;

renderer.setRenderTarget(renderTarget);
renderer.render(rtScene, rtCamera);
renderer.setRenderTarget(null); // back to the canvas

screenMesh.material.map = renderTarget.texture; // a texture like any other
```

- Content rendered into a target is linear (working color space, no tone mapping); leave `renderTarget.texture.colorSpace` at its default and let the final on-screen material or `OutputPass` encode it.
- Options include `minFilter`/`magFilter` (Linear), `generateMipmaps` (false), `mipmapsAutoUpdate` (true, r187), `format`, `type`, `colorSpace`, `depthBuffer` (true), `stencilBuffer` (false), `depthTexture`, `samples`, `count`, `depth`.
- `renderTarget.setSize(w, h)` on window resize; `renderTarget.dispose()` when done.
- r187: `WebGLRenderer.setViewport()` and `setScissor()` no longer scale by the pixel ratio while a render target is bound - pass target pixels.
- `renderTarget.clone()` makes a full copy with its own textures (since r174); since r187 each attachment also has its own `TextureSource`.
- Reading pixels back without a sync stall: WebGLRenderer `await renderer.readRenderTargetPixelsAsync(renderTarget, x, y, w, h, buffer)` fills your typed array (RGBA, UnsignedByte or the implementation format); WebGPURenderer `await renderer.readRenderTargetPixelsAsync(renderTarget, x, y, w, h, textureIndex, faceIndex)` returns a new typed array. The parameter lists differ.

## Depth texture

```javascript
const depthTarget = new THREE.WebGLRenderTarget(512, 512);
depthTarget.depthTexture = new THREE.DepthTexture(512, 512); // default type UnsignedIntType, NearestFilter
// FloatType for a 32-bit float depth; DepthStencilFormat + UnsignedInt248Type to include stencil
```

Sample it in a shader and convert with `perspectiveDepthToViewZ` (GLSL `#include <packing>`, or the TSL function of the same name); see `threejs-shaders`. WebGPURenderer (r187) also supports depth-only targets: `new THREE.RenderTarget(w, h, { count: 0, depthTexture: new THREE.DepthTexture(w, h) })`.

## Multiple render targets (MRT)

```javascript
const mrtTarget = new THREE.WebGLRenderTarget(512, 512, { count: 2 }); // WebGLMultipleRenderTargets was removed in r162
const first = mrtTarget.textures[0]; // mrtTarget.texture is textures[0]
const second = mrtTarget.textures[1];
```

With WebGLRenderer the fragment shader writes each attachment (`layout(location = 1) out vec4 ...` in a ShaderMaterial with `glslVersion: THREE.GLSL3`); with WebGPURenderer use the TSL `mrt()` node (see `threejs-postprocessing`).

## Dynamic reflections with CubeCamera (WebGLRenderer)

```javascript
const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256, {
  type: THREE.HalfFloatType,
  generateMipmaps: true,
  minFilter: THREE.LinearMipmapLinearFilter,
});
const cubeCamera = new THREE.CubeCamera(0.1, 1000, cubeRenderTarget);
scene.add(cubeCamera);

reflectiveMesh.material.envMap = cubeRenderTarget.texture;

// each frame: hide the reflective mesh so it does not render into its own reflection
reflectiveMesh.visible = false;
cubeCamera.position.copy(reflectiveMesh.position);
cubeCamera.update(renderer, scene);
reflectiveMesh.visible = true;
```

Six renders per update: update every few frames, at low resolution, or only when something moves. `WebGLCubeRenderTarget.fromEquirectangularTexture(renderer, texture)` converts an equirect image into the cube target.

## WebGPURenderer equivalents

```javascript
import * as THREE from "three/webgpu";

const target = new THREE.RenderTarget(512, 512, { samples: 4 }); // 2D

const cubeRenderTarget = new THREE.CubeRenderTarget(256); // replaces WebGLCubeRenderTarget
const cubeCamera = new THREE.CubeCamera(0.1, 1000, cubeRenderTarget);
scene.add(cubeCamera);
reflectiveMesh.material.envMap = cubeRenderTarget.texture;

reflectiveMesh.visible = false; // hide so the mesh does not reflect itself
cubeCamera.position.copy(reflectiveMesh.position);
cubeCamera.update(renderer, scene);
reflectiveMesh.visible = true;
```

In TSL, a render target texture is sampled like any texture (`texture(target.texture)`); `rtt()` / `convertToTexture()` create render-to-texture nodes inside a node graph.
