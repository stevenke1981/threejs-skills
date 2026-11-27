# GPU Memory: Byte Math, Compressed Textures, Disposal

Purpose: estimating VRAM, the texture levers (size, KTX2), and disposal patterns that actually lower `info.memory` (checked against the r187 source; KTX2 transcoding details are summarised here and owned by `threejs-textures` / `threejs-loaders`).

## Byte Math

`info.memory.textures` on WebGLRenderer counts textures; the bytes are yours to estimate. An RGBA8 texture costs 4 bytes per pixel, and a full mip chain adds about a third: a 2048 × 2048 color map is about 16.8 MB base, about 22 MB with mips. Float and half-float render targets cost 16 and 8 bytes per RGBA pixel. A 1024 texture is a quarter of a 2048 one: author color maps at the size they appear on screen and let mipmapping handle distance.

Geometry: `BufferGeometryUtils.estimateBytesUsed(geometry)` sums the attribute and index arrays. Quantized attributes (Int16 / Int8 normalized) and Uint16 indices cut it; see `threejs-asset-pipeline` for doing that offline.

## Compressed Textures (KTX2)

A Basis Universal KTX2 texture (ETC1S or UASTC) is transcoded at load time to a block-compressed format the GPU supports (ASTC, BC7, BC1/BC3, ETC2, ETC1 or PVRTC) and stays compressed in VRAM, instead of expanding to 4 bytes per pixel like a PNG or JPEG. ASTC is chosen only for UASTC files, ETC1 has no alpha, and PVRTC needs power-of-two sizes; when nothing fits, KTX2Loader falls back to uncompressed RGBA32.

```javascript
import { KTX2Loader } from "three/addons/loaders/KTX2Loader.js";

const ktx2Loader = new KTX2Loader().setTranscoderPath("/basis/").detectSupport(renderer); // WebGPU: after await renderer.init()
const map = await ktx2Loader.loadAsync("color.ktx2"); // a CompressedTexture
mesh.material.map = map; // KTX2Loader sets colorSpace from the file's transfer function
mesh.material.needsUpdate = true; // the material rendered without a map before
```

Loader wiring and decoder paths: `threejs-loaders`. Producing KTX2 files: `threejs-asset-pipeline`.

## Disposal

- `geometry.dispose()` frees its GPU buffers at once, even if another mesh still uses this geometry; that mesh uploads it again on its next render.
- `texture.dispose()` frees the GPU texture once no other undisposed texture shares it. A `texture.clone()` shares the image, and on WebGLRenderer also the GPU copy while wrap, filter, anisotropy, format, type, mipmap, flip, premultiply, unpack-alignment and color-space settings all match.
- `material.dispose()` releases its shader programs (deleted once no material uses them), never its textures.
- `object.dispose()` (Object3D, since r186) releases renderer state for that one object, never its children, geometry, materials or textures. `InstancedMesh.dispose()` also releases its instance matrix, color and morph data.
- Nothing reference-counts meshes: dispose only what nothing still renders.

```javascript
scene.remove(mesh);
mesh.geometry.dispose();
for (const value of Object.values(mesh.material)) {
  if (value?.isTexture) value.dispose();
}
mesh.material.dispose();
mesh.dispose();
console.log(renderer.info.memory.textures); // the count drops as each texture frees
```

To free a whole loaded subtree:

```javascript
function disposeTree(root) {
  root.traverse((child) => {
    child.geometry?.dispose();
    for (const material of [child.material ?? []].flat()) {
      for (const value of Object.values(material)) if (value?.isTexture) value.dispose();
      material.dispose();
    }
    child.dispose?.();
  });
  root.removeFromParent();
}
```

Textures shared with objects that stay in the scene (an environment map, a texture atlas) must be skipped; keep a set of shared resources and test against it.

## WebGPURenderer Memory Counters

Under WebGPURenderer `memory.geometries` and `memory.textures` stay counts, and `memory.total` plus the `*Size` fields add byte estimates (see measuring.md). Since r187 `dispose()` is supported for all buffer attributes there, including `StorageBufferAttribute` from compute (`attribute.dispose()`), which frees the GPU buffer.
