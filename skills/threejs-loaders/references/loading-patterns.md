# Loading Patterns

Purpose: async and source-handling recipes around the loaders (parallel loads, buffers, URLs, cache, abort, priority) and what to do with a glTF scene once it has loaded. Verified against three.js r187 `src/loaders/` and `GLTFLoader.js`.

## Parallel Loading

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { HDRLoader } from "three/addons/loaders/HDRLoader.js";

const [model, environment, colorMap] = await Promise.all([
  new GLTFLoader().loadAsync("model.glb"),
  new HDRLoader().loadAsync("environment.hdr"),
  new THREE.TextureLoader().loadAsync("color.jpg"),
]);

environment.mapping = THREE.EquirectangularReflectionMapping;
colorMap.colorSpace = THREE.SRGBColorSpace;

scene.add(model.scene);
scene.environment = environment;
```

## Full LoadingManager

```javascript
const manager = new THREE.LoadingManager();

manager.onStart = (url, loaded, total) => console.log(`Started: ${url}`);
manager.onProgress = (url, loaded, total) => console.log(`${loaded} / ${total} items`);
manager.onLoad = () => console.log("All assets loaded");
manager.onError = (url) => console.error(`Error loading: ${url}`);

const gltfLoader = new GLTFLoader(manager);
const textureLoader = new THREE.TextureLoader(manager);
```

`onProgress` counts items (files started and finished), not bytes. Items a loader discovers while parsing (a glTF's external textures) are added to `total` as they start, so the fraction can move backwards.

## Abort with a Timeout

`manager.abort()` aborts the manager's AbortController; FileLoader and ImageBitmapLoader combine that signal with their own (`AbortSignal.any`). Loaders that use an `<img>` element (TextureLoader, ImageLoader) ignore it.

```javascript
const manager = new THREE.LoadingManager();
const timeoutId = setTimeout(() => manager.abort(), 30000);

try {
  const gltf = await new GLTFLoader(manager).loadAsync("model.glb");
  scene.add(gltf.scene);
} catch (error) {
  console.error(error); // an aborted request rejects here
} finally {
  clearTimeout(timeoutId);
}
```

`loader.abort()` cancels the requests of one loader that implements it (FileLoader, ImageBitmapLoader).

## Request Priority and HTTP Cache (r187)

```javascript
const heroLoader = new GLTFLoader().setPriority("high"); // fetch priority hint; images get HTMLImageElement.fetchPriority
const backgroundLoader = new GLTFLoader().setPriority("low");
const fresh = new GLTFLoader().setCache("no-cache"); // fetch() cache mode: revalidate with the server
```

- `setCache` takes the `fetch()` cache modes `"default"`, `"no-store"`, `"reload"`, `"no-cache"`, `"force-cache"`. It does not affect `THREE.Cache` or `<img>` requests.
- GLTFLoader passes priority, cache and `withCredentials` on to the file and texture loaders it creates. Loaders you supply (DRACOLoader, KTX2Loader) keep their own settings.

## Parsing Data You Already Hold

```javascript
const buffer = await (await fetch("model.glb")).arrayBuffer(); // or a File from <input type="file">: await file.arrayBuffer()
const gltf = await new GLTFLoader().parseAsync(buffer, ""); // second argument: base path for external resources
scene.add(gltf.scene);
```

Blob and data URLs work wherever a URL is accepted:

```javascript
async function loadTextureFromBlob(blob) {
  const url = URL.createObjectURL(blob);
  try {
    return await new THREE.TextureLoader().loadAsync(url);
  } finally {
    URL.revokeObjectURL(url);
  }
}
```

A multi-file `.gltf` dropped by the user references its `.bin` and textures by relative name; map those names to blob URLs with `manager.setURLModifier`.

## Paths and URL Rewriting

```javascript
const loader = new GLTFLoader();
loader.setPath("assets/models/"); // prefix for the URL passed to load
loader.setResourcePath("assets/textures/"); // where the file's own references (buffers, images) resolve
const gltf = await loader.loadAsync("model.glb"); // fetches assets/models/model.glb

const manager = new THREE.LoadingManager();
manager.setURLModifier((url) => url.replace("assets/", "https://cdn.example.com/assets/"));
const cdnGltf = await new GLTFLoader(manager).loadAsync("assets/models/model.glb");
```

`setCrossOrigin("anonymous" | "use-credentials")` and `setWithCredentials(true)` control cookies and CORS; `setRequestHeader({ ... })` adds headers to FileLoader requests.

## Memory Cache

```javascript
THREE.Cache.enabled = true; // FileLoader and image loads of the same URL are served from memory
THREE.Cache.clear(); // force the next load to fetch again
```

Off by default. It caches the raw response, not the parsed result; loading the same `.glb` twice still parses it twice — clone the first result instead (`SkeletonUtils.clone` for skinned models).

## Processing a Loaded glTF Scene

```javascript
const gltf = await new GLTFLoader().loadAsync("model.glb");
const model = gltf.scene;

model.traverse((child) => {
  if (child.isMesh) {
    child.castShadow = true; // also needs renderer.shadowMap.enabled and a light with castShadow (threejs-lighting)
    child.receiveShadow = true;
  }
});

const head = model.getObjectByName("Head"); // node.name is the glTF name sanitized for track binding and made unique; the original is in userData.name
const extras = gltf.userData; // glTF extras of the root, and gltfExtensions for unknown extensions

// Centre the model and scale it into a unit box
const box = new THREE.Box3().setFromObject(model);
const center = box.getCenter(new THREE.Vector3());
const size = box.getSize(new THREE.Vector3());
const scale = 1 / Math.max(size.x, size.y, size.z);
model.scale.setScalar(scale);
model.position.copy(center).multiplyScalar(-scale); // the offset is scaled too

scene.add(model);
```

## Placeholders

Show a cheap stand-in (a box with the model's expected bounds, or a low-poly LOD) and swap the real model in when `loadAsync` resolves. Keep payloads small with the codecs from `threejs-asset-pipeline`: Draco or meshopt for geometry, KTX2 for GPU memory, WebP/AVIF for download size.
