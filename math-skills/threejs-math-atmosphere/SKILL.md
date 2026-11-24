---
name: threejs-math-atmosphere
description: "Atmosphere and volumetrics for three.js: Rayleigh and Mie scattering with their phase functions, the Sky and SkyMesh addons, Beer-Lambert transmittance, raymarched fog and clouds with light marching, god rays, distance and height fog including scene fog and the TSL fog nodes, linear-workflow tone mapping with ACES, AgX and Neutral, and jitter against banding. Use when building skies, sunsets, fog, haze, clouds, light shafts or any participating medium, or when a bright scene clips or bands."
---

# Three.js Math: Atmosphere

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

## Quick Start

A late-afternoon sun in the `Sky` addon, a directional light along the same direction, and exponential fog that fades distant boxes into the horizon:

```javascript
import * as THREE from "three";
import { Sky } from "three/addons/objects/Sky.js";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.toneMapping = THREE.ACESFilmicToneMapping; // the sky is HDR: with NoToneMapping it clips to white
renderer.toneMappingExposure = 0.5; // a starting point, retune per scene
document.body.appendChild(renderer.domElement);
const scene = new THREE.Scene();
scene.fog = new THREE.FogExp2(0xc9b8a6, 2.146 / 400); // 99% fog at 400 units; hex = on-screen horizon color
const camera = new THREE.PerspectiveCamera(55, window.innerWidth / window.innerHeight, 0.5, 2000);
camera.position.set(0, 6, 30);
camera.lookAt(0, 4, 0);
const sky = new Sky();
sky.scale.setScalar(10000); // depth is forced to the far plane, so camera.far does not clip it
scene.add(sky);
const skyUniforms = sky.material.uniforms;
skyUniforms.turbidity.value = 6;
skyUniforms.rayleigh.value = 2;
const elevation = 14; // degrees above the horizon; phi below is measured from +Y
const azimuth = 215; // degrees around +Y; 180 is straight ahead of this camera (-Z)
const sunDir = new THREE.Vector3().setFromSphericalCoords(1, THREE.MathUtils.degToRad(90 - elevation), THREE.MathUtils.degToRad(azimuth));
skyUniforms.sunPosition.value.copy(sunDir); // a direction: the shader normalizes it
const sunLight = new THREE.DirectionalLight(0xffd8b0, 5);
sunLight.position.copy(sunDir).multiplyScalar(100); // shines from the sun toward the origin
scene.add(sunLight, new THREE.HemisphereLight(0x9fb8d8, 0x40362c, 2.5));
const ground = new THREE.Mesh(new THREE.PlaneGeometry(4000, 4000), new THREE.MeshStandardMaterial({ color: 0x7a8459 }));
ground.rotation.x = -Math.PI / 2;
scene.add(ground);
const boxGeometry = new THREE.BoxGeometry(6, 8, 6);
const boxMaterial = new THREE.MeshStandardMaterial({ color: 0x8a8f99 });
for (let i = 0; i < 16; i++) {
  const box = new THREE.Mesh(boxGeometry, boxMaterial);
  box.position.set((i % 2 ? 1 : -1) * (6 + i), 4, -i * 30); // receding into the fog
  scene.add(box);
}
const timer = new THREE.Timer();
timer.connect(document);
renderer.setAnimationLoop(animate);
function animate() {
  timer.update();
  skyUniforms.time.value = timer.getElapsed(); // the built-in cloud layer moves only through this uniform
  renderer.render(scene, camera);
}
```

`Sky` is not fogged (its `ShaderMaterial` has `fog` false), so only the meshes fade. Pick the fog hex from the rendered horizon, not from the sky parameters (see Fog).

## Light in a Medium

A participating medium (air, haze, fog, cloud, smoke) changes radiance along a ray in four ways, with coefficients per unit length (1/m, or 1/world unit):

- Absorption `σa` (light turned into heat) and out-scattering `σs` (light deflected out of the ray) add to the extinction `σt = σa + σs`. Mean free path: `1 / σt`.
- In-scattering adds light from other directions, `σs · phase · L_incoming`; emission adds fire or glowing gas.
- Single-scattering albedo `σs / σt`: about 1 for clouds and fog droplets (white), lower for smoke and soot. Everything below is single scattering: sunlight reaches a point (attenuated on the way), scatters once toward the camera and is attenuated again. Multiple scattering is approximated, never traced.

## Beer–Lambert Transmittance

- Transmittance along a path: `T = exp(-τ)` with optical depth `τ = ∫ σt ds` (dimensionless). Homogeneous medium of length `d`: `T = exp(-σt · d)`.
- Per march step of length `Δs`: `T *= exp(-σt · Δs)`. Exact for a constant `σt` across the step, so a homogeneous medium gives the same result at any step count. `T *= 1.0 - σt * Δs` is its first-order expansion: it goes negative once `σt · Δs > 1`.
- Colored media: make `σt` a `vec3`; `exp` works per channel. Blue-heavy extinction turns transmitted light red, which is the sunset. Radiance reaching the camera: `L = ∫ T(0→s) · S(s) ds + T_total · L_background`, where `S = σs · phase · L_in` is the in-scattered radiance per unit length.

## Phase Functions

A phase function gives the fraction of scattered light going into each direction, in 1/steradian, and integrates to 1 over the sphere. Both below were checked to integrate to 1.

```glsl
const float PI = 3.141592653589793;
float phaseIsotropic() { return 1.0 / (4.0 * PI); }
float phaseRayleigh(float cosTheta) { return 3.0 / (16.0 * PI) * (1.0 + cosTheta * cosTheta); }
float phaseHG(float cosTheta, float g) { // Henyey-Greenstein; g in (-1, 1): 0 isotropic, > 0 forward
  float g2 = g * g;
  return (1.0 - g2) / (4.0 * PI * pow(1.0 + g2 - 2.0 * g * cosTheta, 1.5));
}
float phaseClouds(float cosTheta) { // dual lobe: silver lining plus some back-scatter, still normalized
  return mix(phaseHG(cosTheta, -0.3), phaseHG(cosTheta, 0.8), 0.7);
}
```

- Sign convention: `cosTheta = dot(rd, sunDir)`, with `rd` the view ray (camera to sample) and `sunDir` pointing from the sample toward the sun. Forward scattering (`g > 0`) is then brightest when looking toward the sun. Using the direction the light travels (`-sunDir`) puts the glow opposite the sun.
- In-scattering from a directional sun: `S = σs · phase(cosTheta) · sunIrradiance · sunTransmittance`. From a uniform sky of radiance `Lsky` the phase integrates out: `S = σs · Lsky`. Dropping the `1 / 4π` scatters about 12.6 times too much light.
- Typical `g`: clouds 0.8–0.9 forward (with a second lobe), haze and Mie aerosols about 0.76 (`Sky` defaults `mieDirectionalG` to 0.8), dust and smoke 0.3–0.6. `|g| = 1` is singular; clamp to about ±0.99. `Sky.js` evaluates its Rayleigh phase at `cosTheta * 0.5 + 0.5`, an artistic variation; the formula above takes `cosTheta` itself.

## Rayleigh and Mie in the Sky

- Rayleigh scattering (molecules, far smaller than the wavelength) scales with `1 / λ⁴`. Sea-level coefficients in common use: `(5.8e-6, 13.5e-6, 33.1e-6)` 1/m for 680/550/440 nm; `Sky.js` uses `(5.8045e-6, 1.3563e-5, 3.0266e-5)` for 680/550/450 nm. Blue scatters 5 to 6 times more than red: the sky overhead is blue, and at sunset the long path through air strips blue from the direct beam and leaves red.
- Mie scattering (aerosols, sizes near the wavelength) is nearly wavelength-independent (white or grey haze) and strongly forward: the bright glow around the sun. Clear air is around `21e-6` 1/m at sea level; treat it as a tunable.
- Density falls with altitude: `ρ(h) = exp(-h / H)`, scale height `H` about 8000 m for Rayleigh and 1200 m for Mie. `Sky.js` folds these into zenith optical lengths of 8400 m and 1250 m. The coefficients are per metre. With 1 unit ≠ 1 m, multiply them by metres per unit; in a 1 unit = 1 km scene Rayleigh blue is `0.0331` per unit.
- A full single-scattering planet atmosphere is a fragment shader that intersects the view ray with two spheres (ground at 6371 km, top near 6471 km), takes around 16 samples along it and, at each, around 8 samples toward the sun for the light's optical depth, summing Rayleigh and Mie with their own phase functions. three ships no such shader and no precomputed-LUT atmosphere; `Sky` and `SkyMesh` are the shipped solution.

## Sky and SkyMesh

`Sky` (`three/addons/objects/Sky.js`, WebGLRenderer only) is a `Mesh` with a unit `BoxGeometry`, `BackSide`, `depthWrite: false` and a vertex shader that forces depth to the far plane. It evaluates the Preetham analytic daylight model; it has no stars, moon or geographic sun position. Uniforms in `sky.material.uniforms`, set through `.value`:

| Uniform | Default | Effect |
| --- | --- | --- |
| `turbidity` | 2 | haze amount (about 1–20); raises Mie scattering, whiter horizon |
| `rayleigh` | 1 | blue scattering strength (0–4); higher is deeper blue and redder sunsets |
| `mieCoefficient` | 0.005 | size and brightness of the sun glow (0–0.1) |
| `mieDirectionalG` | 0.8 | tightness of the glow (HG `g`) |
| `sunPosition` | `(0, 0, 0)` | direction toward the sun, normalized in the shader |
| `cloudCoverage`, `cloudDensity`, `cloudElevation` | 0.4, 0.4, 0.5 | built-in 2D cloud layer, on by default; `cloudCoverage = 0` removes it |
| `cloudScale`, `cloudSpeed`, `time` | 0.0002, 0.00002, 0 | cloud size and drift; nothing advances `time` for you |
| `showSunDisc` | 1 | 0 hides the disc |

- Sun elevation drives the look: near 0° gives sunset colors, below the horizon the sky darkens. Keep the `DirectionalLight` on the same direction; tint it warmer and dimmer as elevation drops.
- The sky ends with the two output includes, so it follows `renderer.toneMapping` and `toneMappingExposure`. Its output is HDR (the sun disc is far above 1): with ACES, exposure around 0.3–0.6 is a starting range.

Lighting the scene from the sky: prefilter it into an environment map with the disc hidden, because a tiny very bright disc turns into fireflies in the PMREM. Details on PMREM and shadows: `threejs-lighting`.

```javascript
import * as THREE from "three";
const pmrem = new THREE.PMREMGenerator(renderer);
const envScene = new THREE.Scene();
let envTarget = null;

function updateSkyEnvironment() { // call again after moving the sun
  sky.material.uniforms.showSunDisc.value = 0;
  envScene.add(sky); // an object has one parent: this removes it from scene
  envTarget?.dispose();
  envTarget = pmrem.fromScene(envScene); // (scene, sigma = 0, near = 0.1, far = 100, { size: 256 })
  scene.add(sky);
  sky.material.uniforms.showSunDisc.value = 1;
  scene.environment = envTarget.texture;
}
```

## Fog

| Fog | Factor `f` (0 = clear) | Notes |
| --- | --- | --- |
| `new THREE.Fog(color, near, far)` | `smoothstep(near, far, d)` | a smoothstep, not a linear ramp |
| `new THREE.FogExp2(color, density)` | `1 - exp(-(density · d)²)` | squared exponential, not Beer–Lambert; `f = 0.99` at `d = 2.146 / density` |
| homogeneous medium | `1 - exp(-σt · d)` | Beer–Lambert with radial distance; write it yourself |

- `d` in three's fog is view depth (`-mvPosition.z` in `fog_vertex`), not distance from the camera. Fog at the screen edges is thinner than in the center at equal distance and shifts as the camera turns; a custom fog can use `length(viewPosition)`.
- Composite: `color = mix(color, fogColor, f)`. That is the in-scattering integral for a uniformly lit medium with `fogColor` as the in-scattered radiance. Directional haze: `fogColor = mix(hazeColor, sunColor, pow(max(dot(rd, sunDir), 0.0), 8.0))`.
- WebGL built-in materials apply fog after tone mapping and color space conversion (`opaque_fragment`, `tonemapping_fragment`, `colorspace_fragment`, `fog_fragment`). On the canvas the fog color is converted to the output color space, so the hex is the on-screen color and ignores `toneMappingExposure`: match it to the sky's on-screen horizon color. Inside a render target (EffectComposer) the mix happens in linear space before `OutputPass` tone-maps, so the same hex looks different. Under WebGPURenderer fog is part of the material output, mixed before the renderer tone-maps. `material.fog = false` excludes a built-in material (sun sprites, sky domes). `ShaderMaterial` and `SkyMesh` default to no fog.

A `ShaderMaterial` gets scene fog only with `fog: true`, the fog uniforms and the four chunks:

```javascript
import * as THREE from "three";
const fogged = new THREE.ShaderMaterial({
  fog: true,
  uniforms: THREE.UniformsUtils.merge([THREE.UniformsLib.fog, { uColor: { value: new THREE.Color(0x8899aa) } }]),
  vertexShader: /* glsl */ `
    #include <fog_pars_vertex>
    void main() {
      vec4 mvPosition = modelViewMatrix * vec4(position, 1.0); // fog_vertex reads this name
      gl_Position = projectionMatrix * mvPosition;
      #include <fog_vertex>
    }`,
  fragmentShader: /* glsl */ `
    #include <fog_pars_fragment>
    uniform vec3 uColor;
    void main() {
      gl_FragColor = vec4(uColor, 1.0);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
      #include <fog_fragment>
    }`,
});
```

Analytic height fog: density `σ(y) = a · exp(-b · y)` has a closed-form optical depth along a ray from `ro`, unit direction `rd`, length `t`. `a` is the density at y = 0 (1/unit), `1 / b` the height at which it drops to 37%.

```glsl
float heightFogOpticalDepth(vec3 ro, vec3 rd, float t, float a, float b) {
  float k = b * rd.y;
  float base = a * exp(-b * ro.y);
  if (abs(k) < 1e-4) return base * t; // horizontal ray: the limit of the formula below, avoids 0 / 0
  return base * (1.0 - exp(-max(k * t, -60.0))) / k; // clamp: a long downward ray overflows exp
}
```

On a built-in material, add it with `onBeforeCompile` (pattern in `threejs-shaders`) and mix before tone mapping so the fog color is linear radiance, as on WebGPU. Set `scene.fog = null` so the two fogs do not stack. Uniforms `uFogA`, `uFogB`, `uFogColor` go into `shader.uniforms`:

```glsl
// vertex, after #include <project_vertex>:
vFogWorld = (modelMatrix * vec4(transformed, 1.0)).xyz; // declare varying vec3 vFogWorld; (ignores instanceMatrix)
// fragment, replace "#include <opaque_fragment>" with:
#include <opaque_fragment>
vec3 toFrag = vFogWorld - cameraPosition;
float dist = length(toFrag);
float f = 1.0 - exp(-heightFogOpticalDepth(cameraPosition, toFrag / dist, dist, uFogA, uFogB));
gl_FragColor.rgb = mix(gl_FragColor.rgb, uFogColor, f);
```

## Raymarched Volumes

Clouds, local fog banks and smoke are marched at fixed steps through their bounds (sphere tracing is for surfaces: `threejs-math-sdf-raymarching`). Each step adds the light scattered there and attenuates what lies behind. Use the analytic step integral (Hillaire): the in-scattered term `S · exp(-σt x)` integrated over the step is `S · (1 - exp(-σt Δs)) / σt`. The Riemann sum `radiance += T * S * Δs` over-brightens dense media and changes brightness with the step count; the integral does not.

```glsl
vec2 intersectBox(vec3 ro, vec3 rd, vec3 bmin, vec3 bmax) { // x > y: miss
  vec3 inv = 1.0 / rd; // a zero component gives +/-inf, which min and max handle
  vec3 t0 = (bmin - ro) * inv, t1 = (bmax - ro) * inv;
  vec3 tmin = min(t0, t1), tmax = max(t0, t1);
  return vec2(max(max(tmin.x, tmin.y), max(tmin.z, 0.0)), min(tmax.x, min(tmax.y, tmax.z)));
}
vec4 marchVolume(vec3 ro, vec3 rd, float tNear, float tFar, float jitter) {
  const int STEPS = 48; // a starting point
  float stepSize = (tFar - tNear) / float(STEPS);
  float t = tNear + stepSize * jitter; // jitter in [0, 1): slices become noise
  float phase = phaseClouds(dot(rd, uSunDir)); // constant along the ray
  vec3 radiance = vec3(0.0);
  float transmittance = 1.0;
  for (int i = 0; i < STEPS; i++) {
    vec3 p = ro + rd * t;
    float density = sampleDensity(p);
    if (density > 0.001) { // skip the light march in empty space
      float sigmaT = density * uExtinction;
      vec3 S = sigmaT * ALBEDO * (uSunColor * phase * lightTransmittance(p) + uAmbient);
      float stepT = exp(-sigmaT * stepSize);
      radiance += transmittance * (S - S * stepT) / sigmaT;
      transmittance *= stepT;
      if (transmittance < 0.01) break; // nothing behind contributes visibly
    }
    t += stepSize;
  }
  return vec4(radiance, transmittance);
}
```

- With `S = σs · L_in` the step contributes `albedo · L_in · (1 - stepT)`: a dense step saturates at the in-scattered light instead of growing without bound. `uSunColor` is the sun's irradiance (phase turns it into radiance); `uAmbient` is sky radiance, which needs no phase.
- Composite: `color = radiance + transmittance * sceneColor`. As a blended material, output `vec4(radiance, 1.0 - transmittance)` with `blending: THREE.CustomBlending`, `blendSrc: THREE.OneFactor`, `blendDst: THREE.OneMinusSrcAlphaFactor` (premultiplied).
- Work in linear radiance, before tone mapping: a volume drawn to the canvas ends with the two output includes, one inside EffectComposer stays linear for `OutputPass`. `tFar = min(tFar, tScene)` stops the ray at opaque geometry; if `tFar <= tNear` there is nothing to march.

As a full-screen pass over the scene's color and depth. The camera ray and scene distance are the recipe from `threejs-math-sdf-raymarching` (five lines in `main`):

```javascript
import * as THREE from "three";
import { FullScreenQuad } from "three/addons/postprocessing/Pass.js";
const w = Math.floor(window.innerWidth * renderer.getPixelRatio());
const h = Math.floor(window.innerHeight * renderer.getPixelRatio());
const sceneTarget = new THREE.WebGLRenderTarget(w, h, { type: THREE.HalfFloatType, depthTexture: new THREE.DepthTexture(w, h) });
const volumePass = new FullScreenQuad(new THREE.ShaderMaterial({
  uniforms: {
    tColor: { value: sceneTarget.texture }, tDepth: { value: sceneTarget.depthTexture }, // linear HDR color
    uProjectionMatrixInverse: { value: camera.projectionMatrixInverse }, uCameraMatrixWorld: { value: camera.matrixWorld },
    cameraNear: { value: camera.near }, cameraFar: { value: camera.far }, uTime: { value: 0 },
    uSunDir: { value: sunDir }, // shared with the Sky
    uSunColor: { value: new THREE.Color(1.0, 0.85, 0.7).multiplyScalar(6) },
    uAmbient: { value: new THREE.Color(0.35, 0.45, 0.6) },
    uCoverage: { value: 0.45 }, uExtinction: { value: 0.08 }, // extinction per world unit at density 1
    uBoxMin: { value: new THREE.Vector3(-600, 80, -600) }, uBoxMax: { value: new THREE.Vector3(600, 140, 600) },
  },
  vertexShader: /* glsl */ `varying vec2 vUv;
    void main() { vUv = uv; gl_Position = vec4(position.xy, 0.0, 1.0); }`,
  fragmentShader: /* glsl */ `
    #include <packing>
    uniform sampler2D tColor, tDepth;
    uniform mat4 uProjectionMatrixInverse, uCameraMatrixWorld;
    uniform float cameraNear, cameraFar, uTime, uCoverage, uExtinction;
    uniform vec3 uSunDir, uSunColor, uAmbient, uBoxMin, uBoxMax;
    varying vec2 vUv;
    const float ALBEDO = 1.0;
    // paste in this order: PI, phaseHG, phaseClouds; pcg, hash31, valueNoise3, fbm3, remap,
    // sampleDensity, lightTransmittance (Clouds); intersectBox, marchVolume; ign (Banding)
    void main() {
      vec4 viewPos = uProjectionMatrixInverse * vec4(vUv * 2.0 - 1.0, -1.0, 1.0);
      vec3 rdView = normalize(viewPos.xyz / viewPos.w);
      vec3 ro = uCameraMatrixWorld[3].xyz;
      vec3 rd = normalize(mat3(uCameraMatrixWorld) * rdView);
      float tScene = perspectiveDepthToViewZ(texture(tDepth, vUv).r, cameraNear, cameraFar) / rdView.z;
      vec3 col = texture(tColor, vUv).rgb;
      vec2 tb = intersectBox(ro, rd, uBoxMin, uBoxMax);
      float tFar = min(min(tb.y, tScene), tb.x + 400.0); // cap grazing paths toward the horizon
      if (tFar > tb.x) {
        vec4 v = marchVolume(ro, rd, tb.x, tFar, ign(gl_FragCoord.xy));
        col = v.rgb + v.a * col;
      }
      gl_FragColor = vec4(col, 1.0);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
    }`,
}));
function renderFrame() { // replaces renderer.render in animate(), after timer.update()
  volumePass.material.uniforms.uTime.value = timer.getElapsed();
  renderer.setRenderTarget(sceneTarget);
  renderer.render(scene, camera);
  renderer.setRenderTarget(null);
  volumePass.render(renderer);
}
```

The `Sky` writes no depth, so sky pixels read depth 1 and `tScene` reaches the far plane; keep the box inside `camera.far`. Built-in materials skip tone mapping inside a target, so `tColor` is linear HDR. Resize `sceneTarget` with the canvas.

## Clouds

Density is noise thresholded by coverage and shaped by a height gradient. The hash is copied from `threejs-math-noise`; `valueNoise3` is the 3D form of its `valueNoise`.

```glsl
uint pcg(uint v) {
  uint state = v * 747796405u + 2891336453u;
  uint word = ((state >> ((state >> 28u) + 4u)) ^ state) * 277803737u;
  return (word >> 22u) ^ word;
}
float hash31(vec3 p) {
  uvec3 q = uvec3(ivec3(floor(p)));
  return float(pcg(q.x + pcg(q.y + pcg(q.z))) >> 8u) * (1.0 / 16777216.0);
}
float valueNoise3(vec3 p) { // [0, 1)
  vec3 i = floor(p), f = fract(p);
  vec3 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);
  vec2 e = vec2(0.0, 1.0);
  float x00 = mix(hash31(i + e.xxx), hash31(i + e.yxx), u.x), x10 = mix(hash31(i + e.xyx), hash31(i + e.yyx), u.x);
  float x01 = mix(hash31(i + e.xxy), hash31(i + e.yxy), u.x), x11 = mix(hash31(i + e.xyy), hash31(i + e.yyy), u.x);
  return mix(mix(x00, x10, u.y), mix(x01, x11, u.y), u.z);
}
float fbm3(vec3 p) { // about [0, 1]
  float sum = 0.0, amp = 0.5;
  for (int i = 0; i < 4; i++) { sum += amp * valueNoise3(p); p = p * 2.03 + 17.0; amp *= 0.5; }
  return sum / 0.9375; // divide by the amplitude sum
}
float remap(float x, float a, float b, float c, float d) { return c + (x - a) * (d - c) / (b - a); }
float sampleDensity(vec3 p) {
  float h = (p.y - uBoxMin.y) / (uBoxMax.y - uBoxMin.y); // 0 at the cloud base, 1 at the top
  float shape = smoothstep(0.0, 0.15, h) * (1.0 - smoothstep(0.6, 1.0, h)); // edge0 < edge1 in both
  float n = fbm3(p * 0.008 + vec3(4.0, 0.0, 1.0) * 0.01 * uTime); // scale and wind: tune per scene
  return clamp(remap(n, 1.0 - uCoverage, 1.0, 0.0, 1.0), 0.0, 1.0) * shape;
}
float lightTransmittance(vec3 p) { // optical depth toward the sun
  const int LIGHT_STEPS = 6;
  float ds = 4.0, tau = 0.0; // first step in world units; the steps grow, covering about 26 times this
  for (int j = 0; j < LIGHT_STEPS; j++) {
    tau += sampleDensity(p + uSunDir * (0.5 * ds)) * ds; // midpoint of the step
    p += uSunDir * ds;
    ds *= 1.6; // nearby occlusion matters most
  }
  return exp(-tau * uExtinction);
}
```

- Cost per pixel is STEPS × (1 + LIGHT_STEPS) density evaluations: 48 × 7 = 336, each 4 octaves × 8 hashes here. That is why volumes are rendered at half or quarter resolution and upsampled, and why density should come from a tiling `Data3DTexture` (`threejs-textures`) once the counts grow. A slab seen toward the horizon has a very long path: cap `tFar` (as above) and fade density with distance.
- Erosion: subtract a higher-frequency Worley-based noise (`1.0 - F1`, from `threejs-math-noise`) near low densities for billowed edges.
- Beer–powder: `float sunT = exp(-tau) * 2.0 * (1.0 - exp(-2.0 * tau));` darkens thin regions on the lit side, the "powdered sugar" look of cumulus seen away from the sun. It is an artistic approximation of multiple scattering, not physics; `Sky.js` uses the same idea for its 2D clouds. Blend it in with a factor rather than replacing `exp(-tau)` everywhere.
- Single scattering with albedo 1 still looks too dark and grey, because real clouds are lit by many bounces. Cheap approximations: an ambient term from the sky color, stronger near the top (`uAmbient * mix(0.5, 1.0, h)`); and summing 2–3 octaves of the sun term, octave `n` with extinction `× 0.5^n`, contribution `× 0.5^n` and `g × 0.5^n`.

## God Rays

Screen-space shafts blur bright pixels radially toward the sun's screen position. As a WebGL `ShaderPass` fragment shader (`threejs-postprocessing`), before `OutputPass`:

```glsl
uniform sampler2D tDiffuse; // the scene, or better an occlusion buffer: sky bright, geometry black
uniform vec2 uSunScreen;    // sun position in uv space
uniform float uDensity, uDecay, uWeight, uExposure; // 0.9 (fraction of the way to the sun), 0.95, 0.5, 0.3 * fade
varying vec2 vUv;
void main() {
  const int SAMPLES = 48;
  vec2 delta = (vUv - uSunScreen) * (uDensity / float(SAMPLES));
  vec2 uv = vUv;
  float illumination = 1.0;
  vec3 shafts = vec3(0.0);
  for (int i = 0; i < SAMPLES; i++) {
    uv -= delta; // step toward the sun
    shafts += texture(tDiffuse, uv).rgb * illumination * uWeight;
    illumination *= uDecay;
  }
  gl_FragColor = vec4(texture(tDiffuse, vUv).rgb + shafts * uExposure / float(SAMPLES), 1.0);
}
```

- Sun screen position: `const ndc = camera.position.clone().addScaledVector(sunDir, 1000).project(camera);` then `uv = ndc.xy * 0.5 + 0.5`. `project()` also returns a point for a sun behind the camera, mirrored through the center: set the exposure to 0 when `ndc.z > 1` and scale it by `Math.max(camera.getWorldDirection(v).dot(sunDir), 0)`.
- Blurring the full scene color also smears every bright non-sun pixel. The clean version renders an occlusion buffer first (sky and sun disc bright, all geometry black, for example through an override material) and adds its blurred result to the scene.
- There is no god-ray pass in the WebGL addons; this `ShaderPass` is the WebGL route. Jitter the first sample by `ign(gl_FragCoord.xy) * delta` (Banding) to hide the steps.
- The effect exists only while the sun is on or near the screen. Shafts from an off-screen sun or through windows need the shadow map marched per pixel: the TSL `godrays()` node does that (see WebGPU and TSL); on WebGLRenderer it means sampling the light's shadow map inside your own volume loop, which this skill does not implement.

## Banding and Jitter

Two separate causes. Too few march steps show slices through the volume: offset each ray's start by a per-pixel value in [0, 1) steps and the slices become high-frequency noise that TAA, a blur or the eye averages, at the same cost. 8-bit output shows smooth dark gradients (horizon at dusk, fog falloff) in steps of 1/255: dither at the very end, in display space.

```glsl
float ign(vec2 fragCoord) { // interleaved gradient noise (Jimenez), the constants of three's interleavedGradientNoise
  return fract(52.9829189 * fract(dot(fragCoord, vec2(0.06711056, 0.00583715))));
}
// march jitter:  float jitter = ign(gl_FragCoord.xy);
// temporal:      float jitter = fract(ign(gl_FragCoord.xy) + float(uFrame) * 0.61803398875);
// output dither, after #include <colorspace_fragment>:
//                gl_FragColor.rgb += (ign(gl_FragCoord.xy) - 0.5) / 255.0;
```

- Temporal variation (golden-ratio offset per frame) only helps with accumulation (TAA, an afterimage pass); without it the noise boils.
- Blue noise spreads the error better than IGN or white noise, but three ships no blue-noise texture. Supply your own 64×64 or 128×128 tile with `NearestFilter`, `RepeatWrapping`, `NoColorSpace`, sampled at `gl_FragCoord.xy / size`.
- `material.dithering = true` adds a small dither to built-in materials after output encoding. Dithering linear values before the sRGB encode makes the noise too strong in the darks. Keep intermediate buffers half float so banding is not baked in before the final pass.

## Tone Mapping and Color

Scattering is computed in linear radiance, unbounded above 1. The display transform (tone mapping, then sRGB encoding) happens once, last. The classic errors are fog or scattering added after the encode and tone mapping applied twice (in the material and again in `OutputPass`).

| `renderer.toneMapping` | Look |
| --- | --- |
| `ACESFilmicToneMapping` | filmic contrast, strong highlight desaturation, some hue shift (saturated blues toward purple or cyan, reds toward orange); three multiplies the input by `exposure / 0.6`, so it renders brighter than a textbook ACES |
| `AgXToneMapping` | softer contrast, keeps hue in very bright saturated light (sunsets, fire); flatter until graded; working range −12.47 to +4.03 EV around middle grey |
| `NeutralToneMapping` | Khronos PBR Neutral: colors unchanged below a peak of 0.76 (after a small toe offset), then compressed and desaturated toward white; least "atmospheric" |
| `ReinhardToneMapping`, `CineonToneMapping`, `LinearToneMapping` | `c / (1 + c)` with low contrast; an older filmic fit; exposure then clip |

- `toneMappingExposure` (default 1) is a linear multiplier before the curve: `2 ** stops`. Switching between ACES and AgX at one exposure changes overall brightness; retune it.
- WebGLRenderer tone-maps per material (`tonemapping_fragment`, skipped with `material.toneMapped = false`) and only when drawing to the canvas; inside a render target it is deferred to `OutputPass`. WebGPURenderer tone-maps the whole frame once at output.
- `scene.background` as a `Color` is a clear color in WebGLRenderer and is not tone-mapped; next to a tone-mapped sky or fog it ignores exposure. `NoToneMapping`, the default, clips an HDR sky to white.
- HDR buffers: `new THREE.WebGLRenderer({ outputBufferType: THREE.HalfFloatType })` (default `UnsignedByteType`); EffectComposer targets are half float (`threejs-postprocessing`). Bloom on the sun and fog needs that HDR input: threshold above 1, tone mapping after bloom.

## WebGPU and TSL

`Sky` and `ShaderMaterial` do not run under WebGPURenderer. `SkyMesh` has the same parameters as uniform nodes on the object, takes its cloud motion from the built-in `time` node, and sets `material.fog = false`. Its type flag is `isSkyMesh`.

```javascript
import * as THREE from "three/webgpu";
import { color, exponentialHeightFogFactor, fog, uniform } from "three/tsl";
import { SkyMesh } from "three/addons/objects/SkyMesh.js";
const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.toneMapping = THREE.AgXToneMapping;
renderer.toneMappingExposure = 0.5; // a starting point
document.body.appendChild(renderer.domElement);
const scene = new THREE.Scene(); // no scene.background needed: the sky covers every pixel
const camera = new THREE.PerspectiveCamera(55, window.innerWidth / window.innerHeight, 0.5, 2000);
camera.position.set(0, 6, 30);
const sky = new SkyMesh();
sky.scale.setScalar(10000);
sky.sunPosition.value.setFromSphericalCoords(1, THREE.MathUtils.degToRad(90 - 10), THREE.MathUtils.degToRad(180));
sky.turbidity.value = 4;
const ground = new THREE.Mesh(new THREE.PlaneGeometry(4000, 4000).rotateX(-Math.PI / 2), new THREE.MeshBasicNodeMaterial({ color: 0x55603f }));
scene.add(sky, ground);
const towerGeometry = new THREE.BoxGeometry(6, 60, 6).translate(0, 30, 0); // taller than the fog: only the base fades
const towerMaterial = new THREE.MeshBasicNodeMaterial({ color: 0x8a8f99 }); // unlit, so the fog is the only shading
for (let i = 0; i < 8; i++) scene.add(new THREE.Mesh(towerGeometry, towerMaterial).translateX((i % 2 ? 1 : -1) * (10 + i)).translateZ(-i * 30));
const fogDensity = uniform(0.0004); // per unit², see below; change later through fogDensity.value
scene.fogNode = fog(color(0x9db4c8), exponentialHeightFogFactor(fogDensity, uniform(20)));
renderer.setAnimationLoop(() => renderer.render(scene, camera));
```

- `scene.fog = new THREE.Fog(...)` or `FogExp2` is converted to a fog node automatically. Custom fog: `scene.fogNode = fog(colorNode, factorNode)` with `rangeFogFactor(near, far)`, `densityFogFactor(density)` (the FogExp2 square) or `exponentialHeightFogFactor(density, height)`, all from `three/tsl`. Pass `uniform(...)` nodes so `.value` can change later. `rangeFog()` and `densityFog()` are not exported.
- `exponentialHeightFogFactor` is `1 - exp(-density² · m²)` with `m = max(height - positionWorld.y, 0) · viewDepth`: fog only below `height`, thicker deeper below it and further away. `m` is a product of two lengths, so `density` is far smaller than a FogExp2 density: 99% at `m = 2.146 / density`. `fog()` takes any float node as factor, so a noise-modulated factor works. Node materials have `fog = true` by default; the fog node mixes into the material output before the renderer tone-maps.

`VolumeNodeMaterial` marches the whole segment between the camera and the mesh's back faces, not only the part inside the mesh (`steps` 25, `BackSide`, `transparent`, no depth test or write) and accumulates light from point, spot and rect-area lights with their shadows. Directional and ambient lights are ignored: `direct()` returns early for lights without a `distance` property. Occlusion by meshes needs `depthNode` assigned; it does not read the depth buffer by itself.

```javascript
import * as THREE from "three/webgpu";
import { mx_fractal_noise_float, screenCoordinate, time, vec3 } from "three/tsl";
import { bayer16 } from "three/addons/tsl/math/Bayer.js";
const volumeMaterial = new THREE.VolumeNodeMaterial();
volumeMaterial.steps = 12; // a starting point; the offset hides the slices
volumeMaterial.offsetNode = bayer16(screenCoordinate).r; // ray start jitter in step units, 0..1
volumeMaterial.scatteringNode = ({ positionRay }) =>
  mx_fractal_noise_float(positionRay.mul(0.2).add(vec3(0, time.mul(0.1), 0)), 3).mul(0.5).add(0.5).max(0);
const volume = new THREE.Mesh(new THREE.BoxGeometry(40, 20, 40), volumeMaterial);
volume.position.y = 10;
scene.add(volume);
const spot = new THREE.SpotLight(0xffeedd, 400, 60, Math.PI / 6, 0.5, 2); // has a distance: contributes
spot.position.set(0, 18, 0);
spot.castShadow = true; // with renderer.shadowMap.enabled = true, occluders cut shafts into the volume
scene.add(spot);
```

- `RaymarchingBox(steps, ({ positionRay }) => { ... })` from `three/addons/tsl/utils/Raymarching.js` marches a unit box in object space inside your own `Fn`, for 3D-texture clouds with your own compositing.
- `godrays(depthNode, camera, light)` from `three/addons/tsl/display/GodraysNode.js` marches the light's shadow map in screen space for a `DirectionalLight` or `PointLight`. It needs a full shadow setup: `renderer.shadowMap.enabled`, the light casting shadows, objects casting and receiving. Properties: `raymarchSteps` (60), `density` (0.7), `maxDensity` (0.5), `distanceAttenuation` (2) as uniforms, `resolutionScale` 0.5. The source recommends `bilateralBlur` (`three/addons/tsl/display/BilateralBlurNode.js`) on the result and compositing with `depthAwareBlend` (`three/addons/tsl/display/depthAwareBlend.js`); wire it in a `RenderPipeline` with `pass()` as in `threejs-postprocessing`.
- `radialBlur(textureNode, { center, weight, decay, count, exposure })` from `three/addons/tsl/display/radialBlur.js` is the screen-space shaft look of the GLSL pass above, with IGN jitter built in.
- Jitter: `interleavedGradientNoise(screenCoordinate)` from `three/tsl` returns a float in [0, 1); `bayer16(screenCoordinate).r` an ordered 16×16 pattern.

## Common Mistakes

- `new Sky()` under WebGPURenderer → `new SkyMesh()` from `three/addons/objects/SkyMesh.js`: Sky is a ShaderMaterial and renders only with WebGLRenderer.
- `sky.material.uniforms.turbidity.value = 4` on a SkyMesh → `sky.turbidity.value = 4`: SkyMesh exposes uniform nodes as properties and has no `material.uniforms`.
- A Sky with the default `renderer.toneMapping` → `renderer.toneMapping = THREE.ACESFilmicToneMapping; renderer.toneMappingExposure = 0.5;`: the sky outputs HDR radiance and clips to white around the sun with NoToneMapping.
- Sky clouds that never move → `sky.material.uniforms.time.value = timer.getElapsed();` each frame, or `cloudCoverage.value = 0` to remove them: nothing advances the WebGL Sky's `time` uniform.
- `sun.setFromSphericalCoords(1, THREE.MathUtils.degToRad(elevation), azimuth)` → `setFromSphericalCoords(1, THREE.MathUtils.degToRad(90 - elevation), THREE.MathUtils.degToRad(azimuth))`: phi is the polar angle from +Y, in radians.
- `transmittance *= 1.0 - sigmaT * stepSize;` → `transmittance *= exp(-sigmaT * stepSize);`: the linear form goes negative for dense media or large steps and depends on the step count.
- `radiance += transmittance * S * stepSize;` → `radiance += transmittance * (S - S * exp(-sigmaT * stepSize)) / sigmaT;`: the Riemann sum over-brightens dense regions and changes with the step count.
- `(1.0 - g * g) / pow(1.0 + g * g - 2.0 * g * c, 1.5)` → divide by `4.0 * PI` as well: an unnormalized phase function makes the medium scatter about 12.6 times too much light.
- `phaseHG(dot(rd, -sunDir), g)` with `sunDir` pointing at the sun → `phaseHG(dot(rd, sunDir), g)`: with the wrong sign the forward glow appears opposite the sun.
- `if (t > texture(tDepth, vUv).r)` or `t > -viewZ` → `tScene = perspectiveDepthToViewZ(depth, cameraNear, cameraFar) / rdView.z`: depth-buffer values are non-linear and view depth is not distance along the ray.
- `new THREE.FogExp2(color, 0.5)` in a scene hundreds of units across → `new THREE.FogExp2(color, 2.146 / visibilityDistance)`: the factor is `1 - exp(-(density·d)²)`, so 0.5 is opaque after about 4 units.
- `new THREE.VolumeNodeMaterial()` lit only by a DirectionalLight → a PointLight or SpotLight, or the `godrays()` node for a directional sun: the volumetric lighting model skips lights without a `distance`.

## See Also

- `threejs-math-sdf-raymarching` — camera-ray reconstruction, scene-depth compositing and sphere tracing of surfaces.
- `threejs-math-noise` — hashes, fBm and Worley noise for cloud density and erosion.
- `threejs-postprocessing` — EffectComposer, `ShaderPass`, `OutputPass` and the WebGPU `RenderPipeline` for volume and god-ray passes.
- `threejs-fundamentals` — renderer setup, `renderer.toneMapping`, `scene.fog` and `Timer`.
- `threejs-lighting` — PMREM environments, directional sun lights and shadows.
- `threejs-shaders` — ShaderMaterial rules, the two output includes and `onBeforeCompile`.
