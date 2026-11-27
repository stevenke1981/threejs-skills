# WebGL Passes: Recipes

Purpose: EffectComposer recipes for WebGLRenderer: selective bloom, depth of field, SSAO and GTAO, outline, filter passes, pixelation, custom ShaderPass, and the composer-free `renderer.setEffects()`. Targets three.js r187. Every import below is from `three/addons/postprocessing/` or `three/addons/shaders/`.

## Selective Bloom

Bloom only meshes marked with a layer: paint everything else black for a bloom-only composer that renders off screen, then mix its texture into a final composer that renders the clean scene. `scene.background` is not a mesh: set it to `null` around `bloomComposer.render()`, or it is added to the final image a second time; on resize, call `setSize` on both composers.

```javascript
import * as THREE from "three";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";

const BLOOM_LAYER = 1;
const bloomLayer = new THREE.Layers();
bloomLayer.set(BLOOM_LAYER); // test mask: matches any object enabled on this layer

mesh.layers.enable(BLOOM_LAYER); // add the layer; set() would remove the mesh from layer 0

// Paint non-bloomed meshes black, then restore them
const darkMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 });
const materials = new Map();

function darkenNonBloomed(object) {
  if (object.isMesh && bloomLayer.test(object.layers) === false) {
    materials.set(object, object.material);
    object.material = darkMaterial;
  }
}

function restoreMaterial(object) {
  if (materials.has(object)) {
    object.material = materials.get(object);
    materials.delete(object);
  }
}

// Bloom-only composer; with renderToScreen off, its result stays in renderTarget2
const bloomComposer = new EffectComposer(renderer);
bloomComposer.renderToScreen = false;
bloomComposer.addPass(new RenderPass(scene, camera));
bloomComposer.addPass(new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85));

// Final composer: the clean scene plus the bloom texture, mixed by a ShaderPass
const finalComposer = new EffectComposer(renderer);
finalComposer.addPass(new RenderPass(scene, camera));
finalComposer.addPass(
  new ShaderPass(
    new THREE.ShaderMaterial({
      uniforms: {
        baseTexture: { value: null }, // filled from the previous pass
        bloomTexture: { value: bloomComposer.renderTarget2.texture },
      },
      vertexShader: /* glsl */ `
        varying vec2 vUv;
        void main() {
          vUv = uv;
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
      `,
      fragmentShader: /* glsl */ `
        uniform sampler2D baseTexture;
        uniform sampler2D bloomTexture;
        varying vec2 vUv;
        void main() {
          gl_FragColor = texture2D(baseTexture, vUv) + texture2D(bloomTexture, vUv);
        }
      `,
    }),
    "baseTexture",
  ),
);
finalComposer.addPass(new OutputPass());

function animate() {
  const background = scene.background;
  scene.background = null; // not a mesh: it would bloom and be added twice
  scene.traverse(darkenNonBloomed); // black scene: only marked meshes bloom
  bloomComposer.render();
  scene.traverse(restoreMaterial);
  scene.background = background;
  finalComposer.render(); // clean scene + bloom to the screen
}

renderer.setAnimationLoop(animate);
```

## Depth of Field

```javascript
import { BokehPass } from "three/addons/postprocessing/BokehPass.js";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera)); // BokehPass reads this color image

const bokehPass = new BokehPass(scene, camera, {
  focus: 10.0, // view-space depth that stays sharp, in world units (default 1)
  aperture: 0.025, // how far out of focus things blur (default 0.025)
  maxblur: 0.01, // cap on the blur radius (default 1)
});
composer.addPass(bokehPass);
composer.addPass(new OutputPass());

// Refocus at runtime: distanceTo matches the view-space depth for objects near the view center
bokehPass.uniforms.focus.value = camera.position.distanceTo(mesh.position);
```

## Ambient Occlusion

```javascript
import { SSAOPass } from "three/addons/postprocessing/SSAOPass.js";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";

const composer = new EffectComposer(renderer);

// SSAOPass darkens the previous image, so the scene render must come first
composer.addPass(new RenderPass(scene, camera));

// SSAOPass(scene, camera, width = 512, height = 512, kernelSize = 32)
const ssaoPass = new SSAOPass(scene, camera, window.innerWidth, window.innerHeight);
ssaoPass.kernelRadius = 16; // default 8
ssaoPass.minDistance = 0.005; // default
ssaoPass.maxDistance = 0.1; // default
composer.addPass(ssaoPass);
composer.addPass(new OutputPass());

// Debug views: ssaoPass.output = SSAOPass.OUTPUT.SSAO (or Blur, Depth, Normal); Default shows the shaded scene
```

`GTAOPass(scene, camera, width, height, parameters?, aoParameters?, pdParameters?)` is the ground-truth AO alternative with a Poisson denoiser: set `gtaoPass.output = GTAOPass.OUTPUT.Denoise` to inspect the denoised AO, tune with `updateGtaoMaterial({...})` / `updatePdMaterial({...})`, and blend with `blendIntensity` (default 1). It also follows a `RenderPass` and precedes `OutputPass`.

## Outline

```javascript
import * as THREE from "three";
import { OutlinePass } from "three/addons/postprocessing/OutlinePass.js";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));

// OutlinePass(resolution, scene, camera, selectedObjects)
const outlinePass = new OutlinePass(new THREE.Vector2(window.innerWidth, window.innerHeight), scene, camera);
outlinePass.selectedObjects = [mesh]; // only these are outlined
outlinePass.edgeStrength = 3;
outlinePass.visibleEdgeColor.set(0xffffff);
outlinePass.hiddenEdgeColor.set(0x190a05); // parts hidden behind other objects
composer.addPass(outlinePass);
composer.addPass(new OutputPass());
```

## Filter Passes

Film grain, glitch, halftone and vignette are drop-in filters for the middle of the chain. `ColorCorrectionShader` works like the vignette: a `ShaderPass` with `powRGB` / `mulRGB` / `addRGB` uniform vectors. Its default powRGB of (2, 2, 2) darkens the image; start from (1, 1, 1) for neutral.

```javascript
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";
import { FilmPass } from "three/addons/postprocessing/FilmPass.js";
import { GlitchPass } from "three/addons/postprocessing/GlitchPass.js";
import { HalftonePass } from "three/addons/postprocessing/HalftonePass.js";
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";
import { VignetteShader } from "three/addons/shaders/VignetteShader.js";

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));

const filmPass = new FilmPass(0.35, false); // (intensity = 0.5, grayscale = false); animates from the composer's delta
composer.addPass(filmPass);

const glitchPass = new GlitchPass();
glitchPass.goWild = true; // glitch every frame; false (the default) glitches at random intervals
composer.addPass(glitchPass);

const halftonePass = new HalftonePass({ shape: 1, radius: 4, scatter: 0, greyscale: false }); // shape: 1 dot, 2 ellipse, 3 line, 4 square, 5 diamond
composer.addPass(halftonePass);

const vignettePass = new ShaderPass(VignetteShader); // uniforms: offset and darkness (both default 1)
vignettePass.uniforms.offset.value = 1.0;
vignettePass.uniforms.darkness.value = 1.0;
composer.addPass(vignettePass);

composer.addPass(new OutputPass());
```

## Pixelation

`RenderPixelatedPass` renders the scene itself, so it replaces `RenderPass` instead of following it:

```javascript
import { RenderPixelatedPass } from "three/addons/postprocessing/RenderPixelatedPass.js";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";

const composer = new EffectComposer(renderer);
const pixelatedPass = new RenderPixelatedPass(6, scene, camera); // (pixelSize, scene, camera, { normalEdgeStrength, depthEdgeStrength })
pixelatedPass.normalEdgeStrength = 0.3; // edges where the surface normal changes (creases)
pixelatedPass.depthEdgeStrength = 0.4; // depth-discontinuity edges
composer.addPass(pixelatedPass);
composer.addPass(new OutputPass());
```

## Custom Passes

Wrap any `ShaderMaterial`-style shader object in a `ShaderPass`. The `tDiffuse` uniform (the default `textureID`) receives the previous pass output. Keep custom effects before `OutputPass` while they work in linear space; place one after it only if it needs sRGB input, like FXAA. GLSL authoring itself is `threejs-shaders`.

```javascript
import * as THREE from "three";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";

const WaveShader = {
  uniforms: {
    tDiffuse: { value: null },
    time: { value: 0 },
    intensity: { value: 1.0 },
  },
  vertexShader: /* glsl */ `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    uniform sampler2D tDiffuse;
    uniform float time;
    uniform float intensity;
    varying vec2 vUv;
    void main() {
      vec2 uv = vUv;
      uv.x += sin(uv.y * 10.0 + time) * 0.01 * intensity; // wave distortion
      gl_FragColor = texture2D(tDiffuse, uv);
    }
  `,
};

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
const wavePass = new ShaderPass(WaveShader);
composer.addPass(wavePass);
composer.addPass(new OutputPass());

// Drive time uniforms from a Timer
const timer = new THREE.Timer();
timer.connect(document); // keeps deltas sane in inactive tabs

function animate() {
  timer.update();
  wavePass.uniforms.time.value = timer.getElapsed();
  composer.render();
}

renderer.setAnimationLoop(animate);
```

A pass that renders into a target with its own viewport or scissor: since r187, `renderer.setViewport()` and `setScissor()` no longer scale by the pixel ratio while a render target is bound, so pass render-target pixels there.

## Renderer Built-In Effects (No Composer)

`renderer.setEffects()` runs ordinary passes on internal HDR buffers after a plain `renderer.render()` — no composer, and tone mapping and output color space are applied automatically after the effects. It requires `outputBufferType: THREE.HalfFloatType` (or `FloatType`) on the WebGLRenderer, takes no `OutputPass` (it warns and the output transform would apply twice) or `FXAAPass`, anti-aliases through `antialias: true`, and resizes with `renderer.setSize()`. Effects receive no delta time here: FilmPass's time uniform becomes NaN and its grain is lost, so run time-driven passes through EffectComposer.

```javascript
import * as THREE from "three";
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";

const renderer = new THREE.WebGLRenderer({ antialias: true, outputBufferType: THREE.HalfFloatType });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
renderer.setEffects([bloomPass]);

function animate() {
  renderer.render(scene, camera); // bloom runs on the internal HDR buffers after the scene render
}

renderer.setAnimationLoop(animate);
```

With the default `UnsignedByteType` output buffer, `setEffects()` logs an error and applies nothing.
