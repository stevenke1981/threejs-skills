# Blending, stencil and multi-material groups

Purpose: the material settings for blend modes, stencil masking and per-group materials (r187); open for masks, portals, additive glow or per-face materials.

## Blending

```javascript
material.blending = THREE.NormalBlending; // default; alpha-over with transparent: true
material.blending = THREE.AdditiveBlending; // glow, fire, light beams; usually with depthWrite: false
material.blending = THREE.NoBlending; // ignore alpha completely

// Multiply and Subtractive require premultiplied alpha on both renderers (since r178)
const multiply = new THREE.MeshBasicMaterial({ color: 0x8844ff, transparent: true });
multiply.blending = THREE.MultiplyBlending; // same requirement for THREE.SubtractiveBlending
multiply.premultipliedAlpha = true; // without it the renderer logs an error and sets no blend function

// Custom: pick the equation and factors yourself
const custom = new THREE.MeshBasicMaterial({ transparent: true });
custom.blending = THREE.CustomBlending;
custom.blendEquation = THREE.AddEquation;
custom.blendSrc = THREE.SrcAlphaFactor;
custom.blendDst = THREE.OneMinusSrcAlphaFactor;
```

A ShaderMaterial that uses premultiplied blending must premultiply its own output (`#include <premultiplied_alpha_fragment>`); see `threejs-shaders`.

Transparency checklist:

- `transparent: true` puts the object in the transparent list, sorted back to front by object position, drawn after opaque objects.
- Transparent objects still write depth unless `depthWrite: false`; stacked layers then hide each other.
- Intersecting transparent meshes cannot be sorted correctly per object; split them, use `alphaTest` / `alphaHash` cutouts, or set `renderOrder`.
- `alphaTest` keeps the material in the opaque pass, correct in any order.

## Stencil

Stencil is off by default in both renderers: create the renderer with `{ stencil: true }`.

```javascript
const renderer = new THREE.WebGLRenderer({ antialias: true, stencil: true });

// 1. The mask writes 1 into the stencil buffer and nothing into color or depth
const maskMaterial = new THREE.MeshBasicMaterial({ colorWrite: false, depthWrite: false });
maskMaterial.stencilWrite = true; // master switch for the stencil test and write
maskMaterial.stencilRef = 1;
maskMaterial.stencilFunc = THREE.AlwaysStencilFunc;
maskMaterial.stencilZPass = THREE.ReplaceStencilOp; // writes 1 where the mask draws; also stencilFail, stencilZFail

// 2. The content draws only where the mask wrote 1
const contentMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000, stencilWrite: true, stencilRef: 1 });
contentMaterial.stencilFunc = THREE.EqualStencilFunc;
contentMaterial.stencilFuncMask = 0xff; // compare mask; stencilWriteMask masks the write (both default 0xff)

// 3. Draw the mask first
maskMesh.renderOrder = 1;
contentMesh.renderOrder = 2;
```

There is no `stencilMask` property; it is `stencilFuncMask` (compare) and `stencilWriteMask` (write). A render target needs `stencilBuffer: true` for stencil to work inside it.

## Multiple Materials per Mesh

```javascript
// One material per geometry group; BoxGeometry has six, in the order +x, -x, +y, -y, +z, -z
const geometry = new THREE.BoxGeometry(1, 1, 1);
const colors = [0xff0000, 0x00ff00, 0x0000ff, 0xffff00, 0xff00ff, 0x00ffff];
const mesh = new THREE.Mesh(geometry, colors.map((color) => new THREE.MeshBasicMaterial({ color })));

// Custom groups: start, count, material index. Indexed geometry counts indices (BoxGeometry has 36),
// non-indexed geometry counts vertices; ranges outside every group are not drawn
geometry.clearGroups();
geometry.addGroup(0, 18, 0);
geometry.addGroup(18, 18, 1);
```

Each group is a separate draw call. `BufferGeometryUtils.mergeGeometries(geometries, true)` creates one group per input geometry so a material array still applies (see `threejs-geometry`).
