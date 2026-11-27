---
name: threejs-asset-pipeline
description: "Offline glTF optimisation for three.js with the gltf-transform 4.5 CLI and gltfjsx 6.5: inspect, validate, optimize, Draco, meshopt and quantize, KTX2 (ETC1S/UASTC), WebP and AVIF textures, simplify, weld, join, instance, command order, scripted Node pipelines. Use when a .glb is too big, \"optimize my model\", Draco vs meshopt, KTX2 vs WebP, a validation CI gate, or gltfjsx --transform. Not for runtime decoder setup (use threejs-loaders), the React component side (use threejs-react-three-fiber), measuring draw calls or GPU memory (use threejs-performance). 中文: 模型壓縮, 模型優化, 貼圖壓縮, 減面, 檔案太大, glTF 優化"
---

# Three.js Asset Pipeline

Targets three.js r187 with `@gltf-transform/cli` 4.5.1 and `gltfjsx` 6.5.3. This skill owns build-time work on `.glb` files; the loader wiring the output needs at runtime is `threejs-loaders`.

## Quick Start

Inspect, optimise, inspect again:

```bash
npx @gltf-transform/cli inspect model.glb
npx @gltf-transform/cli optimize model.glb model.opt.glb --compress draco --texture-compress webp --texture-size 2048
npx @gltf-transform/cli inspect model.opt.glb
```

Every command that writes a file logs `input (size) → output (size)` when it finishes. Draco output needs a DRACOLoader at runtime; WebP decodes in the browser:

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { DRACOLoader } from "three/addons/loaders/DRACOLoader.js";

const gltf = await new GLTFLoader().setDRACOLoader(new DRACOLoader()).loadAsync("model.opt.glb");
scene.add(gltf.scene);
```

## Inspect First

Never guess where the bytes are. `inspect` prints an overview (generator, `extensionsUsed`, `extensionsRequired`) and tables for scenes, meshes, materials, textures and animations, so a geometry-heavy file and a texture-heavy file are obvious at a glance (`--format csv` or `--format md` for other output).

```bash
npx @gltf-transform/cli validate model.glb --ignore ACCESSOR_WEIGHTS_NON_NORMALIZED
```

`validate` runs the Khronos glTF validator and prints errors, warnings, infos and hints; it throws `Validation detected errors.` when any error-severity issue is present, which makes the command fail in CI.

## Choosing a Codec

Geometry (`--compress`):

| Codec | Compresses | Runtime | Pick when |
|---|---|---|---|
| `meshopt` (default) | triangles, points, lines, morph targets, animation | `MeshoptDecoder` | Animated or morph-heavy files; serve with gzip/brotli on top |
| `draco` | triangle geometry only | `DRACOLoader` | Static meshes where the smallest download matters most |
| `quantize` | stores attributes as 8/16-bit integers (`KHR_mesh_quantization`) | none | You want smaller files and GPU buffers with no decoder |

Textures (`--texture-compress`):

| Format | Saves | Runtime | Pick when |
|---|---|---|---|
| `ktx2` | GPU memory and upload time; stays compressed on the GPU | `KTX2Loader` + `detectSupport` | Many or large textures, mobile GPUs |
| `webp` / `avif` | download size only; decoded to raw RGBA in GPU memory (4–8× the file) | none | Download size is the constraint |
| `auto` (default) | recompresses in the original format | none | A safe first pass |

KTX2 encoding shells out to the `ktx` tool from KTX-Software 4.4.0 or newer, so install it first; WebP and AVIF are encoded in-process with sharp.

## What Each Output Needs at Runtime

`extensionsRequired` in `inspect` is the runtime contract. Each codec maps to one loader call; the code, the shared-KTX2Loader rule and the exact errors are in `threejs-loaders`.

| `extensionsRequired` entry | Runtime setup |
|---|---|
| `KHR_draco_mesh_compression` | `gltfLoader.setDRACOLoader(new DRACOLoader())` |
| `EXT_meshopt_compression` | `gltfLoader.setMeshoptDecoder(MeshoptDecoder)` |
| `KHR_texture_basisu` | `gltfLoader.setKTX2Loader(new KTX2Loader().detectSupport(renderer))` |
| `KHR_mesh_quantization`, `EXT_mesh_gpu_instancing`, `EXT_texture_webp`, `EXT_texture_avif` | nothing |

## The optimize Command

`optimize` runs, in order: dedup, instance, palette, flatten, join, weld, simplify, resample, prune, sparse, texture compression, and geometry compression last. With no flags it applies meshopt and recompresses textures in their own format, capped at 2048 px. Booleans take an explicit value (`--simplify false`).

```bash
# GPU textures, keep every triangle
npx @gltf-transform/cli optimize model.glb model.opt.glb --texture-compress ktx2 --simplify false
# no geometry codec at all, textures untouched
npx @gltf-transform/cli optimize model.glb model.opt.glb --compress false --texture-compress false
```

- `--texture-compress ktx2` picks bitstreams per slot: UASTC (level 4, RDO) for normal, occlusion and metallic-roughness, ETC1S for everything else.
- `--simplify` is on by default with ratio 0 and error 0.0001 (a fraction of mesh extent): it removes only what stays within that error. Turn it off for hard-surface or rigged models you need exact.
- `--join` (needs `--flatten`) cuts draw calls but can inflate vertex counts in scenes that reuse one mesh; `--instance` handles reuse instead.

The full flag list and every single-purpose command are in `references/cli-commands.md`.

## Command Order

Every command decodes Draco and meshopt geometry when it reads a file (it warns `Decoded KHR_draco_mesh_compression. Further compression will be lossy.`), and only `draco`, `meshopt` and `optimize` encode it again. Run texture, weld, simplify and other passes first, geometry compression last:

```bash
npx @gltf-transform/cli weld model.glb a.glb
npx @gltf-transform/cli webp a.glb b.glb --quality 80
npx @gltf-transform/cli draco b.glb model.opt.glb
```

Weld before `simplify` and `meshopt`: `simplify` welds only primitives that have no indices, `meshopt` welds nothing, and its reorder step skips unindexed primitives (`No qualifying primitives found; may need to weld first.`). `draco` welds by itself.

## gltfjsx for React

```bash
npx gltfjsx model.glb --transform
```

Writes `Model.jsx` (`--types` for `.tsx`) and, with `--transform`, `model-transformed.glb`: palette materials, joined, welded and pruned geometry, Draco compression, and textures re-encoded as WebP (`--format`) capped at `--resolution` (default 1024), with normal maps kept as JPEG at up to `max(resolution, 2048)` px. `--simplify` with `--ratio` / `--error` adds simplification. Flags are in `references/gltfjsx.md`; the component side (Canvas, Suspense, useGLTF) is `threejs-react-three-fiber`.

## Common Mistakes

- `optimize model.glb out.glb --compress ktx2` → `--compress draco --texture-compress ktx2`: `--compress` takes draco, meshopt, quantize or false; texture codecs go to `--texture-compress`.
- `draco model.glb a.glb && webp a.glb out.glb` → `webp model.glb a.glb && draco a.glb out.glb`: the second command decodes the Draco geometry and writes it uncompressed.
- Running `etc1s` on every texture, normal maps included → `optimize --texture-compress ktx2`, or `uastc --slots "{normalTexture,occlusionTexture,metallicRoughnessTexture}"` followed by `etc1s`: ETC1S blocks normal maps visibly.
- `optimize` on a rigged character with default `--simplify` → add `--simplify false` when the deformation must stay exact, then compare in the viewer.
- `instance` on an animated file → it logs `Instancing is not currently supported for animated models.` and creates nothing; instanced meshes must also share materials.
- `dequantize` to "fix" a quantized file → leave it: GLTFLoader reads `KHR_mesh_quantization` natively. Dequantize only when your own code needs float vertex data in model units.
- KTX2 output that fails with `Command "ktx" not found` → install KTX-Software 4.4.0+ so `ktx` is on PATH.
- A missing decoder at runtime (`No DRACOLoader instance provided`, `setMeshoptDecoder must be called…`) → wire the loader as in `threejs-loaders`.

## Reference Files

- `references/cli-commands.md` — open for the full `optimize` flag list and the single-purpose geometry, texture and scene commands (`draco`, `meshopt`, `quantize`, `etc1s`, `uastc`, `webp`, `avif`, `resize`, `weld`, `simplify`, `join`, `instance`, `dedup`, `prune`, `gzip`).
- `references/scripting.md` — open to script the same transforms with `@gltf-transform/core` and `functions` in Node (CI, many files, custom defaults).
- `references/gltfjsx.md` — open for gltfjsx flags and what `--transform` does step by step.

## See Also

- `threejs-loaders` — DRACOLoader, MeshoptDecoder and KTX2Loader setup, and loading the result.
- `threejs-react-three-fiber` — the React app around a gltfjsx component.
- `threejs-performance` — measuring what the pipeline bought: draw calls, GPU memory, `renderer.info`.
- `create-game-assets` — producing and validating the source art before it reaches the pipeline.
