# Texture types and compressed textures

Purpose: code for every texture class you create yourself in r187 (image, data, canvas, video, WebCodecs, HTML, KTX2) plus atlases and material slot channel conventions; open when creating textures from code or setting up KTX2.

## Texture from an image element

```javascript
const image = new Image();
image.src = "sprite.png";

const imageTexture = new THREE.Texture(image);
imageTexture.colorSpace = THREE.SRGBColorSpace;
image.onload = () => (imageTexture.needsUpdate = true); // upload once the pixels exist
```

Loading with `TextureLoader` is simpler (`load()` returns the texture immediately and fills it when the image arrives; `loadAsync()` returns a promise). TextureLoader's `onProgress` argument is unsupported; use a `LoadingManager` (see `threejs-loaders`).

## DataTexture

Raw typed-array pixels. Defaults differ from `Texture`: `NearestFilter` for both filters, `flipY` false, `generateMipmaps` false, `unpackAlignment` 1.

```javascript
const size = 256;
const data = new Uint8Array(size * size * 4); // RGBA, one byte per channel
for (let i = 0; i < size; i++) {
  for (let j = 0; j < size; j++) {
    const index = (i * size + j) * 4;
    data[index] = i;
    data[index + 1] = j;
    data[index + 2] = 128;
    data[index + 3] = 255;
  }
}
const dataTexture = new THREE.DataTexture(data, size, size); // RGBAFormat, UnsignedByteType by default
dataTexture.needsUpdate = true; // also after every later change to data
```

Float data: `new THREE.DataTexture(new Float32Array(size * size * 4), size, size, THREE.RGBAFormat, THREE.FloatType)`. Volumes and layers: `Data3DTexture` and `DataArrayTexture`.

## CanvasTexture

Sets `needsUpdate` on construction.

```javascript
const canvas = document.createElement("canvas");
canvas.width = 256;
canvas.height = 256;
const ctx = canvas.getContext("2d");
ctx.fillStyle = "#404040";
ctx.fillRect(0, 0, 256, 256);
ctx.fillStyle = "#ffffff";
ctx.font = "48px sans-serif";
ctx.fillText("Hello", 40, 150);

const canvasTexture = new THREE.CanvasTexture(canvas);
canvasTexture.colorSpace = THREE.SRGBColorSpace; // canvas colors are sRGB

// after each redraw of the canvas:
canvasTexture.needsUpdate = true;
```

## VideoTexture

For a playing `HTMLVideoElement`. Mipmaps off, Linear filters; it flags itself for upload on each new video frame (via `requestVideoFrameCallback` where available).

```javascript
const video = document.createElement("video");
video.src = "video.mp4";
video.loop = true;
video.muted = true; // autoplay policies allow muted playback
video.playsInline = true;
await video.play();

const videoTexture = new THREE.VideoTexture(video);
videoTexture.colorSpace = THREE.SRGBColorSpace;
```

Under WebGLRenderer a video texture is uploaded without sRGB decoding and built-in materials decode it in the shader; a custom ShaderMaterial must decode it itself (see `threejs-shaders`).

## VideoFrameTexture (WebCodecs)

Each decoded frame is pushed by hand:

```javascript
const frameTexture = new THREE.VideoFrameTexture();
frameTexture.colorSpace = THREE.SRGBColorSpace;
const decoder = new VideoDecoder({
  output(frame) {
    if (frameTexture.image instanceof VideoFrame) frameTexture.image.close(); // free the replaced frame, or decoding stalls
    frameTexture.setFrame(frame); // sets image and needsUpdate
  },
  error: (e) => console.error(e),
});
```

## HTMLTexture

`new THREE.HTMLTexture(element)` draws an HTML element through the browser's HTML-in-Canvas API: the renderer moves the element into its own canvas (setting `layoutsubtree` on it) and re-uploads on the canvas's paint events. Where the browser lacks the API the texture stays empty. Mipmaps are off.

## Compressed textures (KTX2)

```javascript
import { KTX2Loader } from "three/addons/loaders/KTX2Loader.js";

const ktx2Loader = new KTX2Loader();
// No setTranscoderPath call needed: by default the Basis transcoder (basis_transcoder.js/.wasm)
// loads from examples/jsm/libs/basis/ next to KTX2Loader.js via import.meta.url.
ktx2Loader.detectSupport(renderer); // required before loading; works with WebGLRenderer and WebGPURenderer

const compressedMap = await ktx2Loader.loadAsync("diffuse.ktx2");
// colorSpace comes from the file's transfer function; nothing to set by hand
```

- Self-hosting the decoders: `ktx2Loader.setTranscoderPath("/basis/")` pointing at a copy of `examples/jsm/libs/basis/` from the same release, for example `https://cdn.jsdelivr.net/npm/three@0.187.0/examples/jsm/libs/basis/`. A transcoder from another release can fail to decode.
- Under WebGPURenderer call `detectSupport` after `await renderer.init()`; `detectSupportAsync()` is deprecated (r181).
- Basis Universal KTX2 transcodes to a format the GPU supports (ASTC, BC7, ETC2, S3TC, PVRTC ...), falling back to uncompressed RGBA where none is available. It stays compressed in GPU memory, unlike PNG/JPEG.
- Compressed textures cannot be flipped or mipmapped at upload: `flipY` stays false (author the file in the orientation your UVs expect) and mipmaps must be embedded in the file.
- Wiring KTX2 into GLTFLoader (`gltfLoader.setKTX2Loader(ktx2Loader)`): `threejs-loaders`. Encoding textures to KTX2: `threejs-asset-pipeline`.
- r187: WebGPURenderer now also supports `CompressedCubeTexture` and uncompressed data inside compressed textures; `CompressedTexture.mipmaps` defaults to an empty array.

## Texture atlas

One image, many sprites, each selected with offset and repeat:

```javascript
const atlas = await new THREE.TextureLoader().loadAsync("atlas.png");
atlas.colorSpace = THREE.SRGBColorSpace;
// filtering and mipmaps sample across sprite borders: leave padding around each sprite

// one clone per sprite: clones share the image source, so the renderer uploads it once
function spriteTexture(row, col, gridSize = 2) {
  const sprite = atlas.clone();
  sprite.offset.set(col / gridSize, 1 - (row + 1) / gridSize);
  sprite.repeat.set(1 / gridSize, 1 / gridSize);
  return sprite;
}

mesh.material.map = spriteTexture(0, 0); // top-left sprite of a 2x2 sheet
```

For many sprites in one draw, a `DataArrayTexture` or a TSL/GLSL UV offset per instance avoids one material per sprite.

## Material slot channel conventions

- `aoMap` reads red, `roughnessMap` green, `metalnessMap` blue - one packed occlusion-roughness-metalness (ORM) image can fill all three slots.
- `alphaMap` reads green; it takes effect with `transparent: true` (blended) or `alphaTest` (cut out).
- `emissiveMap` contributes only when `emissive` is not black.
- `displacementMap` moves vertices (`displacementScale`, `displacementBias`) and wants a subdivided mesh.
- `normalMapType` defaults to `TangentSpaceNormalMap`; set `ObjectSpaceNormalMap` for object-space maps. `normalScale` (a Vector2) scales the effect; a map baked with the opposite green-channel convention (DirectX-style, green down) looks inverted and is fixed with `normalScale.y = -1`.

Which material has which slot: `threejs-materials`.
