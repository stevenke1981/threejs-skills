# WebGPU RenderPipeline and TSL Effect Nodes

Purpose: post-processing under WebGPURenderer: `RenderPipeline` semantics, the effect-node catalogue with import paths and signatures, and recipes for DOF + FXAA, GTAO/SSAO, custom TSL effects and selective bloom. Targets three.js r187 (all signatures checked against `examples/jsm/tsl/display/` at the r187 tag).

## RenderPipeline

`RenderPipeline` (from `three/webgpu`; renamed from `PostProcessing` in r183, which survives as a deprecated subclass that warns) renders a full-screen quad whose fragment is `renderPipeline.outputNode`.

- `outputColorTransform` (default `true`) wraps the output in `renderOutput(outputNode, renderer.toneMapping, renderer.outputColorSpace)`. Set it to `false` when an effect needs sRGB input (FXAA) and apply `renderOutput()` yourself earlier in the graph.
- `needsUpdate` must be set to `true` after changing `outputNode` or `outputColorTransform` once rendering has started. A change of `renderer.toneMapping` or `outputColorSpace` is detected automatically.
- `render()` replaces `renderer.render()` in the loop. `renderAsync()` is deprecated since r181: `await renderer.init()` once, then render synchronously.
- `pass(scene, camera, options)` renders the scene into a target that follows the renderer's drawing-buffer size, so the pipeline needs no resize call. Its MSAA sample count is `options.samples` or, by default, the renderer's (`antialias: true` gives 4).
- `DirectRenderPipeline(renderer)` applies tone mapping and color space inside each material instead of in a separate output pass (`renderPipeline.render(scene, camera)`): cheaper, but it changes blending, breaks materials that sample the framebuffer (transmission), and ignores `antialias` on the WebGL backend.
- EffectComposer and its passes run only under WebGLRenderer.

## Effect-node catalogue

Import each from `three/addons/tsl/display/<File>.js`. "Adds" means the node returns only its contribution, which you combine with the scene color; "replaces" means it returns the finished image.

| Node (file) | Signature | Result |
| --- | --- | --- |
| `bloom` (BloomNode) | `(node, strength = 1, radius = 0, threshold = 0)` | adds: glow only |
| `dof` (DepthOfFieldNode) | `(node, viewZNode, focusDistance = 1, focalLength = 1, bokehScale = 1)` | replaces |
| `fxaa` (FXAANode) | `(node)` | replaces; needs sRGB input |
| `smaa` (SMAANode) | `(node)` | replaces; linear input, before sRGB conversion |
| `traa` (TRAANode) | `(beautyNode, depthNode, velocityNode, camera)` | replaces; MSAA must be off |
| `ao` (GTAONode) | `(depthNode, normalNode, camera)` | AO in `.r` |
| `ssao` (SSAONode) | `(depthNode, normalNode, camera)` | AO in `.r`, self-denoised |
| `outline` (OutlineNode) | `(scene, camera, { selectedObjects, edgeThickness, edgeGlow, downSampleRatio })` | adds: `.visibleEdge`, `.hiddenEdge` masks |
| `film` (FilmNode) | `(inputNode, intensityNode?, uvNode?)` | replaces |
| `rgbShift` (RGBShiftNode) | `(node, amount, angle)` | replaces |
| `chromaticAberration` (ChromaticAberrationNode) | `(node, strength = 1, center = null, scale = 1.1)` | replaces |
| `afterImage` (AfterImageNode) | `(node, damp = 0.96)` | replaces |
| `dotScreen` (DotScreenNode) | `(node, angle, scale)` | replaces |
| `lut3D` (Lut3DNode) | `(node, lut, size, intensity)` | replaces |
| `pixelationPass` (PixelationPassNode) | `(scene, camera, pixelSize = 6, normalEdgeStrength = 0.3, depthEdgeStrength = 0.4)` | renders the scene itself; `pixelSize` is a plain number since r187 |
| `godrays` (GodraysNode) | `(depthNode, camera, light)` | see `threejs-math-atmosphere` |

Also in the folder: `gaussianBlur`, `bilateralBlur`, `boxBlur`, `hashBlur`, `radialBlur`, `denoise`, `ssr`, `ssgi`, `sss`, `taau`, `fsr1`, `sharpen`, `sobel`, `lensflare`, `lensDirt` (new in r187), `motionBlur`, `transition`, `anaglyphPass`, `ssaaPass` and others. `three/tsl` itself exports only the plumbing: `pass`, `mrt`, `renderOutput`, `output`, `emissive`, `velocity`, `normalView`, `builtinAOContext` and so on.

## Bloom (quick start)

```javascript
import * as THREE from "three/webgpu";
import { pass } from "three/tsl";
import { bloom } from "three/addons/tsl/display/BloomNode.js";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x111111); // an opaque background avoids premultiplied-alpha surprises
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 5);
scene.add(new THREE.Mesh(new THREE.SphereGeometry(1, 32, 16), new THREE.MeshBasicNodeMaterial({ color: 0xffffff })));

const renderPipeline = new THREE.RenderPipeline(renderer);
const scenePass = pass(scene, camera);
const scenePassColor = scenePass.getTextureNode("output");

// bloom() returns only the glow, so add it to the scene color
renderPipeline.outputNode = scenePassColor.add(bloom(scenePassColor, 1.5, 0.4, 0.85));

function animate() {
  renderPipeline.render(); // instead of renderer.render()
}

renderer.setAnimationLoop(animate);
```

## Depth of field followed by FXAA

```javascript
import * as THREE from "three/webgpu";
import { pass, renderOutput } from "three/tsl";
import { dof } from "three/addons/tsl/display/DepthOfFieldNode.js";
import { fxaa } from "three/addons/tsl/display/FXAANode.js";

const renderPipeline = new THREE.RenderPipeline(renderer);
const scenePass = pass(scene, camera);
const scenePassColor = scenePass.getTextureNode("output");
// A filter replaces the image: node, viewZ, focus distance, focal length, bokeh scale
const dofNode = dof(scenePassColor, scenePass.getViewZNode(), 4.5, 1.2, 2.5);

// FXAA runs in sRGB: disable the automatic output transform, then convert the DOF result before FXAA
renderPipeline.outputColorTransform = false;
renderPipeline.outputNode = fxaa(renderOutput(dofNode));

// After the first frame, a changed outputNode or outputColorTransform only takes effect with:
renderPipeline.needsUpdate = true;
```

## Ambient occlusion (GTAO or SSAO)

`ao()` and `ssao()` need depth and view-space normals from a separate pre-pass, and the AO lives in the red channel. `builtinAOContext` feeds it into the materials' ambient term of the real scene pass, so direct light stays unoccluded:

```javascript
import * as THREE from "three/webgpu";
import { pass, mrt, normalView, screenUV, builtinAOContext } from "three/tsl";
import { ao } from "three/addons/tsl/display/GTAONode.js";

const renderPipeline = new THREE.RenderPipeline(renderer);
// Pre-pass with view-space normals in the output MRT attachment
const prePass = pass(scene, camera, { samples: 0 }); // single-sampled depth for the AO node
prePass.setMRT(mrt({ output: normalView }));

const aoPass = ao(prePass.getTextureNode("depth"), prePass.getTextureNode(), camera);
aoPass.resolutionScale = 0.5; // optional: half-resolution AO (default 1)

// Apply the AO to a normal scene pass
const scenePass = pass(scene, camera);
scenePass.contextNode = builtinAOContext(aoPass.getTextureNode().sample(screenUV).r);
renderPipeline.outputNode = scenePass;
```

- GTAO parameters are uniforms: `radius` (default 0.25), `thickness` (1), `scale` (1), `samples` (16), plus `resolutionScale` and `useTemporalFiltering` (false). Set them through `.value`, e.g. `aoPass.radius.value = 0.4`.
- `distanceExponent` and `distanceFallOff` are deprecated since r186 and have no effect.
- GTAO is noisy by design; the official `webgpu_postprocessing_ao` example resolves it with `traa()` (with `scenePass.options.samples = 0`) and enables `useTemporalFiltering`. `ssao()` (`radius` 0.5, `intensity` 1, `bias` 0.025, `samples` 16, `blurSharpness` 2) blurs itself and can keep MSAA.
- The official example stores normals compactly with `mrt({ output: packNormalToRGB(normalView), velocity })`, sets `prePass.getTexture("output").type = THREE.UnsignedByteType`, and decodes them with `sample((uv) => unpackRGBToNormal(prePass.getTextureNode().sample(uv)))` before passing them to `ao()`.
- Showing the AO alone: `vec4(vec3(aoPass.getTextureNode().sample(screenUV).r), 1)` with `renderer.toneMapping = THREE.NoToneMapping`.

## Custom effect in TSL

A custom effect is plain TSL over the scene texture instead of a GLSL `ShaderPass` (TSL authoring itself: `threejs-shaders`). The wave distortion from the WebGL recipe, animated by the `time` node:

```javascript
import * as THREE from "three/webgpu";
import { pass, screenUV, sin, time, vec2 } from "three/tsl";

const renderPipeline = new THREE.RenderPipeline(renderer);
const scenePassColor = pass(scene, camera).getTextureNode("output");
const offset = vec2(sin(screenUV.y.mul(10).add(time)).mul(0.01), 0);
renderPipeline.outputNode = scenePassColor.sample(screenUV.add(offset));
```

## Selective bloom with MRT

No layer trick is needed: write the materials' emissive color into its own attachment and bloom only that.

```javascript
import { pass, mrt, output, emissive } from "three/tsl";
import { bloom } from "three/addons/tsl/display/BloomNode.js";

const scenePass = pass(scene, camera);
scenePass.setMRT(mrt({ output, emissive }));
const scenePassColor = scenePass.getTextureNode("output");
renderPipeline.outputNode = scenePassColor.add(bloom(scenePass.getTextureNode("emissive")));
```

## Outline

```javascript
import { pass, uniform } from "three/tsl";
import { outline } from "three/addons/tsl/display/OutlineNode.js";

const outlinePass = outline(scene, camera, { selectedObjects: [mesh], edgeThickness: uniform(1), edgeGlow: uniform(0) });
const visibleEdgeColor = uniform(new THREE.Color(0xffffff));
const hiddenEdgeColor = uniform(new THREE.Color(0x4e3636));
const { visibleEdge, hiddenEdge } = outlinePass;
const outlineColor = visibleEdge.mul(visibleEdgeColor).add(hiddenEdge.mul(hiddenEdgeColor)).mul(3); // 3 = edge strength
renderPipeline.outputNode = outlineColor.add(pass(scene, camera));
```
