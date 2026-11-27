# Scripted Pipelines with the gltf-transform SDK

Purpose: running the CLI's transforms from a Node script when the CLI stops fitting (per-project defaults, CI, batches of files). Checked against `packages/core`, `extensions` and `functions` at glTF-Transform v4.5.1.

## Draco Example

```javascript
// pipeline.mjs — deps: @gltf-transform/core @gltf-transform/extensions @gltf-transform/functions draco3dgltf
import { NodeIO } from "@gltf-transform/core";
import { ALL_EXTENSIONS } from "@gltf-transform/extensions";
import { dedup, prune, weld, draco } from "@gltf-transform/functions";
import draco3d from "draco3dgltf";

const io = new NodeIO()
  .registerExtensions(ALL_EXTENSIONS) // an unregistered extension is dropped on read, or throws if required
  .registerDependencies({
    "draco3d.decoder": await draco3d.createDecoderModule(), // reads Draco input
    "draco3d.encoder": await draco3d.createEncoderModule(), // writes Draco output
  });

const document = await io.read("model.glb");
await document.transform(
  dedup(),
  weld(), // shared vertices, before compression
  prune(), // drop what no Scene references
  draco({ method: "edgebreaker", quantizePosition: 14 }), // the defaults, spelled out
);
await io.write("model.opt.glb", document);
```

Run it with `node pipeline.mjs`. `draco()` only marks the document for Draco; the encoding happens when `io.write` runs, so it stays the last transform.

## Meshopt Instead

```javascript
import { MeshoptDecoder, MeshoptEncoder } from "meshoptimizer";
import { meshopt } from "@gltf-transform/functions";

await MeshoptDecoder.ready;
await MeshoptEncoder.ready;
io.registerDependencies({
  "meshopt.decoder": MeshoptDecoder, // reads EXT_meshopt_compression input
  "meshopt.encoder": MeshoptEncoder,
});

await document.transform(meshopt({ encoder: MeshoptEncoder, level: "high" })); // throws without an encoder
```

## Texture Compression

```javascript
import sharp from "sharp";
import { textureCompress } from "@gltf-transform/functions";

await document.transform(
  textureCompress({ encoder: sharp, targetFormat: "webp", resize: [2048, 2048] }),
);
```

Without `encoder`, `textureCompress` uses a platform-specific fallback encoder and ignores most quality and compression options; sharp works only in Node. KTX2 from a script is the CLI's `toktx` transform shelling out to the `ktx` tool; calling the CLI from the script (`npx @gltf-transform/cli uastc …`) is usually simpler.

## Notes

- Register every extension the file may use. The I/O writes only registered extensions, so a texture or attribute that depends on one it does not know about produces an invalid file.
- `simplify({ simplifier: MeshoptSimplifier, ratio, error })` needs `MeshoptSimplifier` from `meshoptimizer`; weld first.
- `document.getRoot().listTextures()`, `listMeshes()` and friends let a script report per-asset sizes or enforce budgets before writing, which is a cheap CI gate beside `validate`.
