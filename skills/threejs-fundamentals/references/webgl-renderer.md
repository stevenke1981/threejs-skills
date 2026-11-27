# WebGLRenderer Configuration

Purpose: WebGLRenderer constructor options, output settings, clear colour and the `renderer.info` counters in three.js r187.

## Constructor and settings

```javascript
const renderer = new THREE.WebGLRenderer({
  canvas: document.querySelector("#canvas"), // reuse an existing canvas; omit to create one
  antialias: true, // MSAA on the default framebuffer
  alpha: true, // let the page background show through the canvas
  stencil: true, // stencil is off by default; pass this for stencil techniques
  powerPreference: "high-performance",
  preserveDrawingBuffer: true, // keep the drawing buffer for readPixels / toDataURL screenshots
});

renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // cap the fill rate on high-DPI displays
renderer.setSize(window.innerWidth, window.innerHeight); // also sets the canvas CSS size; pass false as a third argument to leave CSS alone

renderer.toneMapping = THREE.ACESFilmicToneMapping; // or AgXToneMapping, NeutralToneMapping; NoToneMapping is the default
renderer.toneMappingExposure = 1.0;
// renderer.outputColorSpace is SRGBColorSpace by default; leave it alone

renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap; // the default, already soft; VSMShadowMap and BasicShadowMap are the alternatives

renderer.setClearColor(0x000000, 0); // clear alpha 0 keeps the alpha: true canvas transparent
renderer.render(scene, camera);
```

- Constructor options (`antialias`, `alpha`, `stencil`, `powerPreference`, `preserveDrawingBuffer`, `depth`, `premultipliedAlpha`) are WebGL context attributes; changing them later has no effect, so recreate the renderer instead.
- `setViewport()` and `setScissor()` take CSS pixels and multiply by the pixel ratio for the canvas. Since r187 they no longer scale while a render target is bound; pass render-target pixels there.
- Shadow quality and light setup: `threejs-lighting`. Tone-mapping operator maths and banding: `threejs-math-atmosphere`.

## Scene background and environment

```javascript
scene.background = new THREE.Color(0x111318); // solid colour
scene.fog = new THREE.Fog(0x111318, 1, 100); // linear fog: near, far
scene.fog = new THREE.FogExp2(0x111318, 0.02); // exponential fog: density (pick one)
```

- `scene.background` also accepts a `CubeTexture`, or a `Texture` whose `mapping` is `THREE.EquirectangularReflectionMapping`; a plain unmapped texture is drawn as a flat screen-filling quad, not a skybox.
- `scene.environment` lights `MeshStandardMaterial` and `MeshPhysicalMaterial` on both renderers, and `MeshLambertMaterial` and `MeshPhongMaterial` on WebGLRenderer only (their node versions use only `material.envMap` / `envNode`). `scene.environmentIntensity` scales it; `material.envMapIntensity` scales only `material.envMap`. HDR loading: `threejs-loaders`; IBL setup: `threejs-lighting`.

## Draw-call and memory counters

- `renderer.info.render.calls` counts the draw calls of the last `render()` call; with several passes per frame, set `renderer.info.autoReset = false` and call `renderer.info.reset()` once per frame.
- `renderer.info.memory.geometries` and `.textures` count live GPU resources; check them after disposing objects to find leaks.
- How to act on the numbers (merging, instancing, LOD, fill rate): `threejs-performance`.
