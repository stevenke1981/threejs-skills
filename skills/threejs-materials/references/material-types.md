# Material types in detail

Purpose: per-class property walkthroughs for the built-in materials (r187); open when configuring one specific class.

Texture color-space rule used below: color maps get `texture.colorSpace = THREE.SRGBColorSpace`, data maps (normal, bump, displacement, roughness, metalness, ao, alpha, gradient) keep the default `NoColorSpace`. Details in `threejs-textures`.

## MeshBasicMaterial

No lighting calculations. Fast, always visible. Supports `map`, `alphaMap`, `aoMap`, `lightMap`, `specularMap`, `envMap` with `reflectivity` (0 to 1) and `combine` (`THREE.MultiplyOperation` default, `THREE.MixOperation`, `THREE.AddOperation`); `specularMap` (sRGB) masks the env-map contribution.

```javascript
const map = await new THREE.TextureLoader().loadAsync("color.jpg");
map.colorSpace = THREE.SRGBColorSpace;

const material = new THREE.MeshBasicMaterial({
  color: 0xffffff, // multiplied with the map
  map,
  transparent: true,
  opacity: 0.5,
  side: THREE.DoubleSide, // FrontSide (default), BackSide, DoubleSide
  fog: true, // affected by scene fog
});
```

## MeshLambertMaterial and MeshPhongMaterial

Lambert is diffuse-only. Phong adds a specular highlight; it is the cheap alternative when Standard is too heavy. Both shade per fragment.

```javascript
const lambert = new THREE.MeshLambertMaterial({
  color: 0x00ff00,
  emissive: 0x111111, // self-illumination color
  emissiveIntensity: 1,
});

const phong = new THREE.MeshPhongMaterial({
  color: 0x0000ff,
  specular: 0x111111, // highlight color
  shininess: 30, // higher = sharper highlight
  flatShading: false, // flat vs smooth normals
});
```

Both also support `map`, `emissiveMap`, `specularMap` (sRGB; masks the env-map reflection, and the highlight on Phong), `normalMap`, `bumpMap`, `displacementMap`, `aoMap`, `lightMap`, `alphaMap`, `envMap`, `envMapIntensity`, `envMapRotation`, `combine` and `reflectivity`.

## MeshStandardMaterial (PBR)

Physically based metal/roughness workflow; what GLTFLoader creates for most glTF materials.

```javascript
const map = await new THREE.TextureLoader().loadAsync("color.jpg");
map.colorSpace = THREE.SRGBColorSpace;
const ormMap = await new THREE.TextureLoader().loadAsync("orm.jpg"); // glTF ORM: ao red, roughness green, metalness blue; data: keep default

const material = new THREE.MeshStandardMaterial({
  color: 0xffffff, // multiplied with the map
  roughness: 1.0, // 0 = mirror, 1 = fully diffuse; multiplied by roughnessMap (green)
  metalness: 1.0, // 0 = dielectric, 1 = metal; multiplied by metalnessMap (blue), so keep 1 with a map
  map,
  roughnessMap: ormMap,
  metalnessMap: ormMap,
  aoMap: ormMap, // ambient occlusion (red); strength with aoMapIntensity
  normalMap: null, // surface detail; scale with normalScale (a Vector2)
  displacementMap: null, // moves vertices; displacementScale, displacementBias; needs a subdivided mesh
  emissive: 0x000000,
  emissiveIntensity: 1,
  emissiveMap: null, // sRGB; contributes only when emissive is not black
  envMap: null, // or rely on scene.environment
  fog: true,
});
```

`lightMap` (with `lightMapIntensity`) bakes indirect light the way aoMap bakes occlusion; an HDR or EXR lightMap is `LinearSRGBColorSpace`, which those loaders set.

aoMap and lightMap sample the UV set selected by `texture.channel` (0 = `uv`, the default) - no second UV set is needed unless the bake used its own unwrap, which goes in `uv1` with `channel = 1`. See `threejs-textures`.

## MeshPhysicalMaterial (advanced PBR)

Extends MeshStandardMaterial. Each feature compiles in only while its strength is above 0, so leave unused ones at 0. Texture variants: `clearcoatMap`, `clearcoatRoughnessMap`, `clearcoatNormalMap`, `transmissionMap`, `thicknessMap`, `iridescenceMap`, `iridescenceThicknessMap`, `sheenColorMap`, `sheenRoughnessMap`, `anisotropyMap`, `specularIntensityMap`, `specularColorMap`, `diffuseRoughnessMap`.

```javascript
const material = new THREE.MeshPhysicalMaterial({
  // Clearcoat (car paint, lacquer)
  clearcoat: 1.0, // 0 = none, 1 = full coat
  clearcoatRoughness: 0.1,
  // Transmission (glass, water) - refracts the opaque objects and scene.background behind it
  transmission: 1.0, // 0 = opaque, 1 = fully transmissive; keep opacity at 1
  thickness: 0.5, // volume thickness for refraction
  attenuationDistance: 1, // world units; Infinity (the default) = no absorption
  attenuationColor: new THREE.Color(0xffaa66), // white light turns this color after attenuationDistance
  ior: 1.5, // index of refraction, 1.0 to 2.333; reflectivity is derived from it
  // Sheen (fabric, velvet)
  sheen: 1.0,
  sheenColor: new THREE.Color(0xffffff), // default black: set it, or sheen shows nothing
  sheenRoughness: 0.5,
  // Iridescence (soap bubbles, oil slicks)
  iridescence: 1.0,
  iridescenceIOR: 1.3,
  iridescenceThicknessMinimum: 100, // nanometers; r187 replacement for iridescenceThicknessRange[0]
  iridescenceThicknessMaximum: 400, // r187 replacement for iridescenceThicknessRange[1]
  // Anisotropy (brushed metal); direction with anisotropyRotation (radians)
  anisotropy: 1.0,
  // Dispersion (prismatic glass, used with transmission); retroreflectivity (road signs, safety gear)
  dispersion: 0.0,
  retroreflectivity: 0.0,
  // Rough diffuse: 0 = Lambertian; above 0 uses the EON energy-preserving rough diffuse model (clay, plaster)
  diffuseRoughness: 0.0,
  // Specular overrides for dielectrics
  specularIntensity: 1.0,
  specularColor: new THREE.Color(0xffffff),
});
```

`iridescenceThicknessRange` still works in r187 as a deprecated getter/setter that logs a warning and forwards to the two new properties.

Glass - transmission with a smooth surface and something behind it to refract:

```javascript
const glass = new THREE.MeshPhysicalMaterial({
  color: 0xffffff,
  metalness: 0,
  roughness: 0,
  transmission: 1,
  thickness: 0.5,
  ior: 1.5,
});
// transmission refracts the opaque objects and background behind it; scene.environment adds its reflections
```

Transmission renders the opaque scene into an extra buffer first, so it costs a second pass. With `WebGLNodesHandler` (node materials on WebGLRenderer) transmission is not supported.

## MeshToonMaterial

Cel shading. The gradient map is non-color data - keep its default color space.

```javascript
const steps = new Uint8Array([120, 200]); // two shading steps
const gradientMap = new THREE.DataTexture(steps, 2, 1, THREE.RedFormat);
gradientMap.minFilter = THREE.NearestFilter; // hard steps; LinearFilter blends them
gradientMap.magFilter = THREE.NearestFilter;
gradientMap.needsUpdate = true;

const material = new THREE.MeshToonMaterial({ color: 0x00ff00, gradientMap });
```

## MeshMatcapMaterial and ShadowMaterial

```javascript
const matcap = await new THREE.TextureLoader().loadAsync("matcap.png");
matcap.colorSpace = THREE.SRGBColorSpace; // LDR matcap is color data
const clay = new THREE.MeshMatcapMaterial({ matcap }); // looks lit with no lights in the scene

const shadowCatcher = new THREE.ShadowMaterial({ opacity: 0.4 }); // ground plane that shows only received shadows
// the mesh using it needs receiveShadow = true; shadow setup lives in threejs-lighting
```

## MeshNormalMaterial and MeshDepthMaterial

```javascript
// Normals encoded as RGB - debugging and flat illustrative looks
const normals = new THREE.MeshNormalMaterial({ flatShading: false, wireframe: false });

// Depth as grayscale (BasicDepthPacking, the default); RGBADepthPacking packs it for reading back in a shader
const depth = new THREE.MeshDepthMaterial();
```

MeshDepthMaterial and MeshDistanceMaterial have no node equivalent under WebGPURenderer.

## PointsMaterial

For point clouds. `size` is in pixels; with `sizeAttenuation` (the default, perspective cameras only) it is multiplied by half the canvas height and divided by the point's view-space depth.

```javascript
const geometry = new THREE.BufferGeometry();
geometry.setAttribute("position", new THREE.Float32BufferAttribute([0, 0, 0, 1, 0.5, 0, 0.5, 1, 0], 3));

const material = new THREE.PointsMaterial({
  color: 0xffffff,
  size: 0.1,
  sizeAttenuation: true, // scale with distance
  alphaTest: 0.5, // with a map or alphaMap: discard fragments below this alpha
});

scene.add(new THREE.Points(geometry, material));
```

Under WebGPURenderer, point primitives are always 1 px; use `PointsNodeMaterial` on a `Sprite` (see `node-materials.md`).

## LineBasicMaterial and LineDashedMaterial

WebGL and WebGPU always draw line primitives 1 px wide; `linewidth`, `linecap` and `linejoin` only affect the SVGRenderer.

```javascript
const geometry = new THREE.BufferGeometry();
geometry.setAttribute("position", new THREE.Float32BufferAttribute([-1, 0, 0, 1, 0, 0], 3));

const solid = new THREE.LineBasicMaterial({ color: 0xffffff }); // always 1 px

const dashed = new THREE.LineDashedMaterial({ color: 0xffffff, dashSize: 0.5, gapSize: 0.25 });
const line = new THREE.Line(geometry, dashed);
line.computeLineDistances(); // required, or the line draws solid; non-indexed geometry only
scene.add(line);
```

Thick lines under WebGLRenderer: `Line2` with `LineGeometry` and `LineMaterial` from `three/addons/lines/` (`linewidth` in pixels by default). `LineMaterial` is a ShaderMaterial, so under WebGPURenderer import `Line2` from `three/addons/lines/webgpu/Line2.js`, which uses `Line2NodeMaterial` from `three/webgpu`.

```javascript
import { Line2 } from "three/addons/lines/Line2.js";
import { LineGeometry } from "three/addons/lines/LineGeometry.js";
import { LineMaterial } from "three/addons/lines/LineMaterial.js";

const thick = new Line2(
  new LineGeometry().setPositions([-1, 0, 0, 1, 0, 0]), // a LineGeometry, not a BufferGeometry
  new LineMaterial({ color: 0xffffff, linewidth: 5 }),
);
scene.add(thick);
```
