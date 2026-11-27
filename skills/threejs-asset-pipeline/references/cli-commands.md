# gltf-transform CLI Commands

Purpose: the `optimize` flag list and the single-purpose commands, checked against `packages/cli/src/cli.ts` at tag v4.5.1 of glTF-Transform. Run any command with `--help` for the full option list of your installed version.

All commands take `<input> <output>` (except `inspect`, `validate` and `gzip`, which take only the input). Global options include `--vertex-layout interleaved|separate` and `--allow-net` (read inputs from URLs).

## optimize Flags

| Flag | Default | Effect |
|---|---|---|
| `--compress draco\|meshopt\|quantize\|false` | `meshopt` | Geometry (and, for meshopt/quantize, animation) compression, applied last |
| `--meshopt-level medium\|high` | `high` | Meshopt filter aggressiveness |
| `--texture-compress ktx2\|webp\|avif\|auto\|false` | `auto` | `auto` recompresses in the original format |
| `--texture-size <px>` | `2048` | Maximum texture dimension |
| `--simplify <bool>` | `true` | meshoptimizer simplification |
| `--simplify-ratio <0–1>` | `0` | Target fraction of vertices to keep |
| `--simplify-error <fraction>` | `0.0001` | Error limit, fraction of mesh extent |
| `--simplify-lock-border <bool>` | `false` | Keep open borders fixed |
| `--instance <bool>`, `--instance-min <n>` | `true`, `5` | EXT_mesh_gpu_instancing for reused meshes |
| `--palette <bool>`, `--palette-min <n>` | `true`, `5` | Bakes solid material factors (base color, alpha, emissive, metallic, roughness) of untextured materials into palette textures, then merges the materials so more meshes can join |
| `--flatten <bool>` | `true` | Flatten the node hierarchy (skeletons and animated nodes are left in place) |
| `--join <bool>` | `true` | Join compatible primitives; requires `--flatten` |
| `--join-meshes <bool>`, `--join-named <bool>` | `true`, `true` | Also join distinct meshes / named meshes; require `--join` |
| `--weld <bool>` | `true` | Merge equivalent vertices; with `--compress draco`, `--weld false` is ignored with a warning |
| `--resample <bool>` | `true` | Losslessly remove redundant animation keyframes |
| `--prune <bool>` | `true` | Remove properties no Scene references |
| `--prune-attributes <bool>` | `true` | Also remove unused vertex attributes |
| `--prune-solid-textures <bool>` | `true` | Replace single-color textures with material factors |
| `--sparse <bool>` | `true` | Sparse accessors for mostly-zero data (morph targets) |

## Geometry Commands

```bash
npx @gltf-transform/cli draco model.glb out.glb --method edgebreaker --quantize-position 14
npx @gltf-transform/cli meshopt model.glb out.glb --level high
npx @gltf-transform/cli quantize model.glb out.glb
npx @gltf-transform/cli dequantize model.glb out.glb
npx @gltf-transform/cli gzip out.glb
```

- `draco`: `--method edgebreaker|sequential` (default edgebreaker), `--encode-speed` / `--decode-speed` 0–10, `--quantize-position|normal|color|texcoord|generic <bits>` (position default 14), `--quantization-volume mesh|scene`. Compresses indexed triangle primitives only; it skips non-indexed and non-triangle primitives with a warning, and welds before encoding.
- `meshopt`: `--level medium|high`, `--quantize-position|normal|texcoord|color|weight|generic <bits>`, `--quantization-volume`. Compresses points, lines and triangles, morph targets and animation; decodes very fast. Writes `EXT_meshopt_compression`.
- `quantize`: same `--quantize-*` options plus `--pattern <glob>` for attribute names. No runtime decoder.
- `gzip`: writes `<input>.gz` next to the input. Serve it with `Content-Encoding: gzip` (or let the server compress) and the browser decompresses transparently. It adds little on top of Draco or JPEG textures and the most on top of meshopt and quantize.

## Texture Commands

```bash
# KTX2 / Basis Universal: ETC1S (smaller, lossier) and UASTC (larger, cleaner)
npx @gltf-transform/cli uastc model.glb a.glb --slots "{normalTexture,occlusionTexture,metallicRoughnessTexture}" --level 4
npx @gltf-transform/cli etc1s a.glb out.glb --quality 128
# Download size
npx @gltf-transform/cli webp model.glb out.glb --quality 80
npx @gltf-transform/cli avif model.glb out.glb --quality 60
# Cap PNG/JPEG dimensions, keeping aspect ratio; never enlarges
npx @gltf-transform/cli resize model.glb out.glb --width 1024 --height 1024
```

- `etc1s`: `--quality 1–255` (higher is better and slower), `--compression`, `--max-endpoints`, `--max-selectors`, `--rdo-threshold`, `--rdo`, `--resize <px>`, `--mipmaps <bool>`, `--slots <glob>`, `--pattern <glob>`, `--jobs <n>`.
- `uastc`: `--level 0–4` (speed vs quality), `--rdo`, `--rdo-lambda`, `--zstd <level>`, plus the same `--resize`, `--mipmaps`, `--slots`, `--pattern`, `--jobs`.
- Both skip textures that are already KTX2, so a `uastc` pass restricted by `--slots` followed by a plain `etc1s` pass gives the per-slot split `optimize --texture-compress ktx2` makes.
- Both call the `ktx` executable from KTX-Software 4.4.0+; without it they fail with `Command "ktx" not found`.
- `webp`, `avif`, `png`, `jpeg`: `--quality 1–100`, `--effort 0–100`, `--lossless <bool>` (webp/avif), `--near-lossless` (webp), `--formats <glob>`, `--slots <glob>`, `--pattern <glob>`. Encoded in-process with sharp.
- `ktxdecompress` turns KTX2 textures back into PNG; `ktxfix` repairs common KTX metadata problems.

## Scene and Mesh Commands

```bash
npx @gltf-transform/cli weld model.glb out.glb
npx @gltf-transform/cli simplify model.glb out.glb --ratio 0.5 --error 0.001 --lock-border true
npx @gltf-transform/cli join model.glb out.glb
npx @gltf-transform/cli instance model.glb out.glb --min 5
npx @gltf-transform/cli dedup model.glb out.glb
npx @gltf-transform/cli prune model.glb out.glb
```

- `weld`: merges equivalent vertices and indexes the primitive. Run it before `simplify` and `meshopt`.
- `simplify`: `--ratio` (fraction of vertices to keep), `--error` (fraction of mesh radius), `--lock-border`. Lossy. It welds only primitives without indices first.
- `join`: runs a material `dedup` and `flatten` first, then joins compatible primitives. `--keepMeshes`, `--keepNamed`. In scenes that reuse one mesh heavily it can raise the vertex count — use `instance` instead.
- `instance`: writes `EXT_mesh_gpu_instancing` for meshes referenced by at least `--min` nodes (default 5); GLTFLoader turns each batch into an `InstancedMesh`. Skips animated files with a warning; instanced meshes must share materials.
- `dedup`: merges duplicate accessors, materials, meshes, skins and textures (`--accessors`, `--materials`, `--meshes`, `--skins`, `--textures` toggle each).
- `prune`: removes properties no Scene references; `--keep-attributes`, `--keep-indices`, `--keep-leaves`.
- Others: `flatten`, `center --pivot center|above|below`, `resample --tolerance`, `sparse`, `reorder`, `unweld`, `tangents`, `unwrap`, `palette`, `metalrough`, `unlit`, `merge`, `partition`, `copy`, `sequence`, `xmp`.

## Validation

```bash
npx @gltf-transform/cli validate model.glb --limit 100 --ignore ACCESSOR_WEIGHTS_NON_NORMALIZED,UNUSED_OBJECT --format csv
```

`--ignore` takes comma-separated issue codes. The command fails when an error-severity issue remains; warnings, infos and hints are reported only.
