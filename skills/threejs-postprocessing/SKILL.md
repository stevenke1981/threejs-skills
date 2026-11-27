---
name: threejs-postprocessing
description: "Three.js r187 post-processing: EffectComposer pass chains and OutputPass, renderer.setEffects, and the WebGPU RenderPipeline with TSL effect nodes: bloom, selective bloom, depth of field, SSAO/GTAO, outline, FXAA/SMAA/TRAA, pixelation, custom screen shaders. Use for adding glow, colors washed out or dark after EffectComposer, anti-aliasing a composer, or porting passes to WebGPU. Not for the renderer's tone-mapping setting (use threejs-fundamentals), writing shader code (use threejs-shaders), god rays or volumetric fog (use threejs-math-atmosphere). 中文: 後處理, 泛光, 景深, 環境光遮蔽, 抗鋸齒, 描邊, 畫面特效"
---

# Three.js Post-Processing

Targets three.js r187. Examples use WebGLRenderer; `references/webgpu-tsl-pipeline.md` covers WebGPURenderer.

## Pick the Path

| Renderer | Path | When |
| --- | --- | --- |
| WebGLRenderer | `EffectComposer` + passes, `OutputPass` last | the general case; any number of passes, multiple composers, time-driven passes |
| WebGLRenderer | `renderer.setEffects([...])` with `outputBufferType: THREE.HalfFloatType` | a few static effects (bloom) with no composer, renderer MSAA kept |
| WebGPURenderer | `RenderPipeline` + TSL effect nodes from `three/addons/tsl/display/` | always: EffectComposer does not run on WebGPURenderer |

## Quick Start (WebGL)

```javascript
import * as THREE from "three";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 0, 5);

// A bright emissive sphere gives bloom something to pick up
const mesh = new THREE.Mesh(
  new THREE.SphereGeometry(1, 32, 16),
  new THREE.MeshStandardMaterial({ color: 0x222222, emissive: 0xffffff, emissiveIntensity: 2 }),
);
scene.add(mesh);

// The composer replaces renderer.render(). Render the scene, then effects,
// then OutputPass last: it applies renderer.toneMapping and outputColorSpace once.
const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
// UnrealBloomPass(resolution, strength = 1, radius, threshold)
const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
// bloomPass.strength, .radius and .threshold are live properties; threshold is the luminance above which things glow
composer.addPass(bloomPass);
composer.addPass(new OutputPass());

function animate() {
  composer.render(); // not renderer.render()
}

renderer.setAnimationLoop(animate);
```

## Pass Order

The composer's buffers hold linear HDR colors (HalfFloatType by default); tone mapping and the sRGB conversion happen once, in `OutputPass`, using the renderer's `toneMapping`, `toneMappingExposure` and `outputColorSpace`. Build chains in this order:

`RenderPass` → linear-space effects (bloom, depth of field, SSAO/GTAO, outline, film, custom) → `SMAAPass` → `OutputPass` → `FXAAPass` (only when FXAA is used, because FXAA needs sRGB input).

- Without FXAA, `OutputPass` is the last pass. `new SMAAPass()` and `new FXAAPass()` take no arguments; the composer sizes them.
- `SSAOPass` and `GTAOPass` darken the image produced by the previous pass, so a `RenderPass` must come before them. `RenderPixelatedPass` renders the scene itself and replaces `RenderPass`.
- `pass.enabled = false` skips a pass without rebuilding the chain.
- Which tone-mapping curve to choose is `threejs-fundamentals`; the curve maths for HDR skies is `threejs-math-atmosphere`.

## Composer Setup

- The composer copies the renderer's pixel ratio at construction. Call `renderer.setPixelRatio()` before constructing it, or `composer.setPixelRatio()` afterwards.
- The composer's buffers are not multisampled; `antialias: true` on the renderer does not reach them. For MSAA on the scene render and the composer's buffers, pass your own buffer with `samples` and size the composer once:

```javascript
const composer = new EffectComposer(
  renderer,
  new THREE.WebGLRenderTarget(window.innerWidth, window.innerHeight, {
    type: THREE.HalfFloatType,
    samples: 4, // MSAA on the composer's two buffers
  }),
);
composer.setSize(window.innerWidth, window.innerHeight); // scales the buffer by the pixel ratio
```

- On resize: `camera.aspect = w / h; camera.updateProjectionMatrix(); renderer.setSize(w, h); composer.setSize(w, h);` — the composer resizes its buffers and every pass, pixel ratio included. Never resize a pass by hand.
- `composer.render()` drives time-based passes (FilmPass) from its own internal `Timer`; pass a delta in seconds only if you manage time yourself.
- The composer assigns `renderToScreen` to the last enabled pass every frame. Set `composer.renderToScreen = false` only to keep the result off screen (selective bloom does this).

## Quick Start (WebGPU)

```javascript
import * as THREE from "three/webgpu";
import { pass } from "three/tsl";
import { bloom } from "three/addons/tsl/display/BloomNode.js";

const renderPipeline = new THREE.RenderPipeline(renderer); // renderer: an initialized WebGPURenderer
const scenePass = pass(scene, camera);
const scenePassColor = scenePass.getTextureNode("output");
renderPipeline.outputNode = scenePassColor.add(bloom(scenePassColor, 1.5, 0.4, 0.85)); // bloom returns only the glow

renderer.setAnimationLoop(() => renderPipeline.render()); // instead of renderer.render()
```

- Effects are nodes, not passes: "adding" nodes (`bloom`, `outline` edges) return a contribution you add to the scene color; "filter" nodes (`dof`, `fxaa`, `smaa`, `film`, ...) return the finished image and replace it; AO nodes return occlusion in `.r`.
- `RenderPipeline` tone-maps and converts color once at the end (`outputColorTransform`, default true). FXAA needs `outputColorTransform = false` plus `renderOutput()` before it.
- After the first frame, set `renderPipeline.needsUpdate = true` whenever you swap `outputNode` or `outputColorTransform`.
- `pass()` uses the renderer's MSAA samples and follows `renderer.setSize()`. TRAA needs MSAA off; AO pre-passes use `pass(scene, camera, { samples: 0 })`.

Catalogue with signatures, DOF + FXAA, GTAO/SSAO wiring, selective bloom with MRT, outline and custom TSL effects: `references/webgpu-tsl-pipeline.md`.

## Common Mistakes

- `three/examples/jsm/postprocessing/EffectComposer.js` → `three/addons/postprocessing/EffectComposer.js`: the examples/jsm specifier does not resolve under the import map.
- `composer.addPass(new ShaderPass(GammaCorrectionShader))` → `composer.addPass(new OutputPass())`: GammaCorrectionShader only applies the sRGB transfer function, while OutputPass applies tone mapping, exposure and output color space.
- `composer.addPass(bloomPass); composer.render();` with no OutputPass → add `new OutputPass()` last: the composer's buffers are linear, so without it tone mapping and sRGB conversion are skipped and the image looks dark or washed out.
- `composer.addPass(fxaaPass); composer.addPass(outputPass);` → `composer.addPass(outputPass); composer.addPass(fxaaPass);`: FXAA needs sRGB input so it follows OutputPass, while SMAA works in linear space and runs before it.
- `new ShaderPass(FXAAShader)` → `new FXAAPass()`: FXAAPass sets its own resolution uniform from the composer size, pixel ratio included.
- `new FilmPass(0.35, 0.5, 648, false)` → `new FilmPass(0.35, false)`: the signature is (intensity, grayscale); the scanline arguments were removed.
- `new HalftonePass(width, height, params)` / `new SMAAPass(width, height)` → `new HalftonePass(params)` / `new SMAAPass()`: the size arguments were removed in r175.
- `new UnrealBloomPass(new THREE.Vector2(innerWidth / 2, innerHeight / 2), ...)` → full size: addPass immediately resizes the pass to the composer size, undoing the half-resolution trick; UnrealBloomPass already blurs at half resolution internally.
- `bloomPass.resolution.set(w, h)` → `composer.setSize(w, h)`: UnrealBloomPass reads `resolution` only in its constructor.
- `bloomPass.renderToScreen = false` → `bloomComposer.renderToScreen = false`: the composer reassigns renderToScreen on every pass each frame.
- `composer.render(); renderer.render(scene, camera);` → only `composer.render()`: the second call clears the canvas and erases the composer output; selective bloom uses a second composer instead.
- `renderer.setEffects([bloomPass])` on a default WebGLRenderer → create it with `outputBufferType: THREE.HalfFloatType`: otherwise setEffects logs an error and applies nothing.
- `renderer.setEffects([bloomPass, new OutputPass(), new FXAAPass()])` → `renderer.setEffects([bloomPass])`: the built-in path applies the output transform itself, so an OutputPass applies it twice (it warns) and FXAA gets linear input.
- `clock.getElapsedTime()` → `timer.getElapsed()`: Clock is deprecated since r183; create a `THREE.Timer`, call `timer.connect(document)` once and `timer.update()` each frame.
- `import { bloom } from "three/tsl"` → `import { bloom } from "three/addons/tsl/display/BloomNode.js"`: three/tsl exports the plumbing (pass, mrt, renderOutput); effect nodes live in the addons.
- `new THREE.PostProcessing(renderer)` → `new THREE.RenderPipeline(renderer)`: renamed in r183; the old name is a deprecated alias that warns.
- `renderPipeline.outputNode = bloom(scenePassColor)` → `scenePassColor.add(bloom(scenePassColor))`: bloom returns only the glow, so on its own the scene is never drawn.
- `renderPipeline.outputNode = fxaa(scenePassColor)` → `renderPipeline.outputColorTransform = false; renderPipeline.outputNode = fxaa(renderOutput(scenePass));`: FXAA needs sRGB input.
- `scenePassColor.add(dof(scenePassColor, scenePass.getViewZNode()))` → `dof(scenePassColor, scenePass.getViewZNode())`: dof returns the finished image.
- `await renderPipeline.renderAsync()` → `renderPipeline.render()`: renderAsync is deprecated since r181; `await renderer.init()` once at setup.
- `aoPass.distanceFallOff.value = 2` → `aoPass.radius.value = 0.2`: distanceExponent and distanceFallOff are deprecated since r186 and have no effect; tune radius and scale.
- `pixelationPass(scene, camera, uniform(6))` → `pixelationPass(scene, camera, 6)`: since r187 pixelSize is a plain number; node objects are not accepted.
- `new EffectComposer(webgpuRenderer)` → `new THREE.RenderPipeline(webgpuRenderer)`: EffectComposer and its passes run only under WebGLRenderer.

## References

- `references/webgl-passes.md` — WebGL recipes: selective bloom with two composers, BokehPass, SSAOPass and GTAOPass, OutlinePass, film/glitch/halftone/vignette/color correction, RenderPixelatedPass, custom ShaderPass, `renderer.setEffects()`. Open when building a WebGL effect chain.
- `references/webgpu-tsl-pipeline.md` — RenderPipeline semantics, DirectRenderPipeline, effect-node catalogue with r187 signatures, DOF + FXAA, GTAO/SSAO wiring and parameters, selective bloom with MRT, outline, custom TSL effects. Open when the project imports `three/webgpu`.

## See Also

- `threejs-fundamentals` — renderer creation, tone mapping choice and exposure, color spaces, the animation loop.
- `threejs-shaders` — GLSL ShaderMaterial authoring and the TSL node language for custom effects.
- `threejs-textures` — render targets and using a post-processed image as a texture.
- `threejs-math-atmosphere` — god rays, volumetric fog and cloud passes, banding and dithering.
- `threejs-lighting` — the lights and environment that AO and bloom react to.
- `threejs-performance` — the fill-rate cost of full-screen passes.
