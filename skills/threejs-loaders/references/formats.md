# Non-glTF Formats

Purpose: loading code for every format other than glTF, plus the less common texture loaders. Verified against three.js r187 `examples/jsm/loaders/`.

Lit materials render black without lights or an environment; each snippet that creates its own material adds lights for that reason.

## OBJ + MTL

```javascript
import { MTLLoader } from "three/addons/loaders/MTLLoader.js";
import { OBJLoader } from "three/addons/loaders/OBJLoader.js";

const materials = await new MTLLoader().loadAsync("model.mtl");
materials.preload();

const object = await new OBJLoader().setMaterials(materials).loadAsync("model.obj");
scene.add(object); // a Group; without an MTL each mesh gets a MeshPhongMaterial
```

r187 fixed parsing of OBJ `l` (line) and `p` (point) statements.

## FBX

```javascript
import { FBXLoader } from "three/addons/loaders/FBXLoader.js";

const object = await new FBXLoader().loadAsync("model.fbx");
object.scale.setScalar(0.01); // FBX files are often exported in centimetres

const mixer = new THREE.AnimationMixer(object); // update it every frame with timer.getDelta(); see threejs-animation
if (object.animations.length > 0) mixer.clipAction(object.animations[0]).play();

scene.add(object);
```

Converting FBX to glTF offline (Blender, FBX2glTF) usually gives smaller files and fewer material surprises than loading FBX at runtime.

## STL

```javascript
import { STLLoader } from "three/addons/loaders/STLLoader.js";

const geometry = await new STLLoader().loadAsync("model.stl"); // a BufferGeometry
scene.add(new THREE.Mesh(geometry, new THREE.MeshStandardMaterial({ color: 0x888888 })));
scene.add(new THREE.HemisphereLight(0xffffff, 0x444444, 1));
scene.add(new THREE.DirectionalLight(0xffffff, 2)); // shines down from (0, 1, 0)
```

## PLY

PLYLoader honours the file's data types: `double` / `float64` properties become attributes backed by a `Float64Array`, which WebGLRenderer cannot upload (it throws `THREE.WebGLAttributes: Unsupported buffer data format`). Convert them before rendering:

```javascript
import { PLYLoader } from "three/addons/loaders/PLYLoader.js";

const geometry = await new PLYLoader().loadAsync("model.ply");

for (const name of Object.keys(geometry.attributes)) {
  const attribute = geometry.getAttribute(name);
  if (attribute.array instanceof Float64Array) {
    geometry.setAttribute(name, new THREE.Float32BufferAttribute(attribute.array, attribute.itemSize));
  }
}

if (!geometry.hasAttribute("normal")) geometry.computeVertexNormals(); // keep the file's own normals when present
const material = new THREE.MeshStandardMaterial({ vertexColors: geometry.hasAttribute("color") }); // vertexColors without a color attribute renders black
scene.add(new THREE.Mesh(geometry, material));
scene.add(new THREE.HemisphereLight(0xffffff, 0x444444, 1));
scene.add(new THREE.DirectionalLight(0xffffff, 2));
```

r187 fixed quad faces: they now split as (0, 1, 2) and (0, 2, 3) per the PLY spec, and per-face colors and texture coordinates on quads are kept.

## USD

```javascript
import { USDLoader } from "three/addons/loaders/USDLoader.js";

const model = await new USDLoader().loadAsync("model.usdz"); // USD, USDA, USDC and USDZ archives
scene.add(model);
```

`USDZLoader` still exists as a subclass of USDLoader that warns on construction (deprecated in r179).

## Gaussian Splats (WebGPURenderer only)

```javascript
import * as THREE from "three/webgpu";
import { SPLATLoader } from "three/addons/loaders/SPLATLoader.js";
import { GaussianSplat } from "three/addons/objects/GaussianSplat.js";

const data = await new SPLATLoader().loadAsync("scene.splat");
scene.add(new GaussianSplat(data));
```

- `SPZLoader` (`.spz`), `KSPLATLoader` (`.ksplat`) and `GaussianSplatPLYLoader` (3DGS `.ply`) return the same kind of data.
- glTF files with `KHR_gaussian_splatting` need the plugin registered by hand, because GaussianSplat requires WebGPURenderer:

```javascript
import { GLTFGaussianSplatLoaderExtension } from "three/addons/loaders/GLTFGaussianSplatLoaderExtension.js";

const loader = new GLTFLoader();
loader.register((parser) => new GLTFGaussianSplatLoaderExtension(parser));
```

## Cube Maps

Six faces in +X, −X, +Y, −Y, +Z, −Z order; the loader sets `SRGBColorSpace` on the result.

```javascript
const cubeTexture = await new THREE.CubeTextureLoader()
  .setPath("textures/cube/")
  .loadAsync(["px.jpg", "nx.jpg", "py.jpg", "ny.jpg", "pz.jpg", "nz.jpg"]);

scene.background = cubeTexture;
scene.environment = cubeTexture;
```

## HDR Image Loaders

All live in `three/addons/loaders/` and return linear-sRGB textures; set `mapping = THREE.EquirectangularReflectionMapping` before using one as an environment or background (configuration is `threejs-lighting`).

- `HDRLoader` — Radiance `.hdr`. Renamed from `RGBELoader` in r180; `RGBELoader` remains as a deprecated subclass that warns.
- `EXRLoader` — OpenEXR. Half float by default; `setDataType(THREE.FloatType)` for full float.
- `UltraHDRLoader` — JPEG with an HDR gain map; HDR range at JPEG file size. Half float by default.
- `JXLLoader` (new in r187) — HDR JPEG XL through the browser's native decoder; needs float16 Canvas2D support, the `srgb-linear` canvas color space and `globalHDRHeadroom`, so check browser support before relying on it.
- `HDRCubeTextureLoader` — six `.hdr` faces as a cube texture.

```javascript
import { UltraHDRLoader } from "three/addons/loaders/UltraHDRLoader.js";

const env = await new UltraHDRLoader().loadAsync("environment.jpg");
env.mapping = THREE.EquirectangularReflectionMapping;
scene.environment = env;
```

## Other Bundled Loaders

Also in `examples/jsm/loaders/` at r187: 3DMLoader (Rhino), 3MFLoader, AMFLoader, BVHLoader (motion capture to skeleton + clip), ColladaLoader, DDSLoader, KTXLoader (KTX1), PVRLoader, TGALoader, TIFFLoader, FontLoader and TTFLoader (for TextGeometry), GCodeLoader, IESLoader (light profiles), KMZLoader, LDrawLoader, LUT3dlLoader / LUTCubeLoader / LUTImageLoader (color grading LUTs), LWOLoader, MD2Loader, MDDLoader, MaterialXLoader, NRRDLoader, PCDLoader, PDBLoader, SVGLoader, TDSLoader (3DS), VOXLoader, VRMLLoader, VTKLoader, XYZLoader. All follow the same `new XLoader(manager).loadAsync(url)` pattern.
