# gltfjsx

Purpose: the gltfjsx 6.5.3 command line and what `--transform` does to the asset. Checked against the published `cli.js` and `src/utils/transform.js` of gltfjsx 6.5.3. The React side (Canvas, Suspense, `useGLTF`, drei) belongs to `threejs-react-three-fiber`.

```bash
npx gltfjsx model.glb --transform --types --shadows
```

Output: `Model.jsx` (the input name with a capital first letter; `.tsx` with `--types` or when `--output` ends in `.tsx`) and, with `--transform`, `model-transformed.glb` beside the output, which the component loads.

## Flags

| Flag | Effect |
|---|---|
| `--output, -o` | Output file name/path |
| `--types, -t` | TypeScript definitions |
| `--keepnames, -k` | Keep original names |
| `--keepgroups, -K` | Keep empty groups, disable pruning |
| `--bones, -b` | Lay out bones declaratively |
| `--meta, -m` | Include metadata as userData |
| `--shadows, -s` | Meshes cast and receive shadows |
| `--precision, -p` | Fractional digits (default 3) |
| `--draco, -d` | Draco binary path for the generated `useGLTF` call |
| `--instance, -i` / `--instanceall, -I` | Instance recurring geometry / every geometry |
| `--exportdefault, -E` | Default export |
| `--transform, -T` | Rewrite the asset for the web |
| `--resolution, -R` | Texture size cap with `--transform` (default 1024) |
| `--format, -f` | Texture format with `--transform` (default `webp`) |
| `--keepmeshes, -j` | Do not join compatible meshes |
| `--keepmaterials, -M` | Do not palette-join materials |
| `--simplify, -S` with `--ratio` (default 0) and `--error` (default 0.0001) | Mesh simplification |
| `--console, -c` | Print the JSX instead of writing a file |

`--instance` and `--instanceall` also run the transform step.

## What `--transform` Runs

In order: unpartition, palette (unless `--keepmaterials`), reorder, dedup, flatten, dequantize, join (unless `--keepmeshes`), weld, simplify (only with `--simplify`), resample, prune, sparse, texture compression to `--format` at `--resolution` for every slot except normal maps, normal maps re-encoded as JPEG at `max(--resolution, 2048)`, and Draco compression last.

The output therefore always needs a DRACOLoader at runtime; drei's `useGLTF` wires one by default (see `threejs-react-three-fiber`). For meshopt or KTX2 output, run gltf-transform yourself and point gltfjsx (without `--transform`) at the result.
