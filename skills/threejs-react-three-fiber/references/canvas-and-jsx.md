# Canvas Props, JSX Elements and Disposal

Purpose: every `<Canvas>` prop in @react-three/fiber 9.8.2, how JSX maps onto three.js objects, and what fiber disposes on unmount (read from the 9.8.2 package source).

## Canvas props

```jsx
export function App() {
  return (
    <Canvas
      shadows="variance"
      dpr={[1, 2]}
      frameloop="demand"
      flat
      camera={{ position: [0, 2, 6], fov: 45 }}
      gl={{ powerPreference: "high-performance" }}
      onCreated={(state) => state.gl.setClearColor("#111827")}
    >
      {/* scene contents */}
    </Canvas>
  );
}
```

- `camera`: props for the default camera, or a camera instance. Without it: `PerspectiveCamera` fov 75, near 0.1, far 1000, at z 5 looking at the origin (unless you pass `rotation`). Fiber updates aspect (or the orthographic frustum with `orthographic`) and the projection matrix on every resize. Passing `aspect`, `left`, `right`, `top` or `bottom` marks the camera manual; `camera={{ manual: true }}` does the same explicitly.
- `orthographic`: the default camera becomes an OrthographicCamera.
- `dpr`: a number, or `[min, max]` (default `[1, 2]`) clamped against `window.devicePixelRatio`. Since 9.8.2 a range also follows display changes and browser zoom; a fixed number never re-resolves.
- `shadows`: `true` enables shadow maps; a string picks the type (`"basic"` BasicShadowMap, `"percentage"` PCFShadowMap, `"soft"` PCFSoftShadowMap, `"variance"` VSMShadowMap); an object merges into `gl.shadowMap`. `true`, `"soft"`, an unknown string and the default `false` all write PCFSoftShadowMap, which three swaps for PCFShadowMap with a console warning (WebGLRenderer when shadows render, WebGPURenderer on every Canvas).
- `flat`: NoToneMapping instead of fiber's default ACESFilmicToneMapping. `linear`: LinearSRGBColorSpace output instead of sRGB, and no automatic sRGB on colour maps. `legacy`: sets `THREE.ColorManagement.enabled = false`. Since 9.8.1 changing `flat` or `linear` after creation updates the renderer, unless `gl` sets `toneMapping` or `outputColorSpace` explicitly.
- `gl`: props merged into the default `new WebGLRenderer({ canvas, antialias: true, alpha: true, powerPreference: "high-performance" })`, a renderer instance, or a factory function receiving those defaults (sync or async; the WebGPU path in `webgpu.md`).
- `frameloop`: `"always"` (default) renders every frame, `"demand"` only after React changes or `invalidate()`, `"never"` hands the loop to you (`advance`, see `hooks-and-events.md`).
- `events` replaces the event manager (the default maps DOM pointer events to raycasts). `eventSource` is the element (or ref) events attach to, default the Canvas wrapper div; with an `eventSource` the wrapper gets `pointer-events: none`. `eventPrefix` (`"offset"`, `"client"`, `"page"`, `"layer"`, `"screen"`) picks which `event[prefix + "X"]` pair is mapped to the pointer; without it fiber uses `offsetX/offsetY`.
- `raycaster`: props merged into the default Raycaster, e.g. `raycaster={{ params: { Line: { threshold: 0.1 } } }}`.
- `scene`: props for the default scene, or a Scene instance.
- `performance`: `{ current, min, max, debounce }` for adaptive regression (`drei.md`).
- `resize`: options for react-use-measure (`{ scroll: true, debounce: { scroll: 50, resize: 0 } }` by default).
- `fallback`: DOM content rendered inside the `<canvas>` element for browsers without canvas support.
- `onCreated(state)` runs once the renderer and scene exist, before the first frame; `onPointerMissed` fires on clicks that hit no object.
- Extra DOM props (`style`, `className`, `id`) go to the wrapper div, which is `position: relative; width: 100%; height: 100%; overflow: hidden`.

## The scene graph as JSX

Every constructor in the registered catalogue is an element: lowercase first letter, same class name (`<mesh>` → `THREE.Mesh`, `<meshStandardMaterial>` → `THREE.MeshStandardMaterial`). The Canvas registers the `three` namespace once; since 9.8.2 only constructors are registered as JSX types, so constants and `MathUtils` no longer collapse other components' prop types.

Prop rules (applyProps):

- `THREE.Color` targets accept any colour representation: `color="#d9915b"`, `color={0xff0000}`.
- Math objects with `set`/`copy` accept an array (`fromArray`), a scalar (`setScalar`) or an instance of the same class (`copy`): `position={[1, 2, 3]}`, `position={5}`, `position={new THREE.Vector3(1, 2, 3)}`.
- `Layers` targets copy the mask from a `Layers` value.
- Dashed props pierce into nested objects: `rotation-x`, `material-color`, `geometry-attributes-position`. Piercing into a missing object throws `R3F: Cannot set "...". Ensure it is an object before setting "..."`.
- `ShaderMaterial` `uniforms` props merge into the existing uniform objects, so references stay stable.
- Anything else is assigned as is; a plain object assigned to a read-only property such as `position` throws.
- Reserved props: `args`, `attach`, `object`, `dispose`, `onUpdate`, `children`, `key`, `ref`.

`args` is the constructor argument list; changing it rebuilds the object from scratch (9.8.2 keeps pointer capture working when that happens mid-drag). Geometries and materials attach to their parent mesh automatically; `attach="material-0"` attaches into an array index; `onUpdate={(self) => ...}` runs after props are applied.

```jsx
function Torus() {
  return (
    <mesh position={[0, 1, 0]}>
      <torusKnotGeometry args={[0.7, 0.25, 128, 24]} />
      <meshStandardMaterial color="#42699c" roughness={0.2} metalness={0.8} onUpdate={(self) => console.log(self.type)} />
    </mesh>
  );
}
```

Mount an object you already hold with `<primitive object={obj} />`: one object per primitive (fiber keeps its per-object state on the object, and three can parent an object only once), so clone for copies. Fiber never disposes primitives. Register classes outside the catalogue with `extend`:

```jsx
import { extend } from "@react-three/fiber";
import { Water } from "three/addons/objects/Water.js";

extend({ Water });
// then render <water args={[geometry]} /> anywhere in the Canvas
```

`extend(Class)` with a single constructor returns a generated element name to use as a component.

In TypeScript the tags `line`, `audio`, `source` and `path` collide with DOM/SVG types; write `threeLine`, `threeAudio`, `threeSource`, `threePath` (fiber strips a `three` prefix followed by a capital letter when no element of the full name exists).

## Disposal

- Fiber disposes what it creates when it unmounts, scheduled for idle time: the object behind every JSX element, with geometries, materials and textures declared as JSX children.
- Objects passed in through props (`geometry={...}`, `map={...}`) and primitives are never disposed; `dispose={null}` exempts an element and its children.
- Renderer (9.8.1+): a renderer fiber built (from defaults, `gl` props or a factory) is disposed, awaiting WebGPURenderer's async `dispose()`, and then loses its context. A renderer instance you passed in only has its render lists released and its context force-lost, so it cannot be reused.
- Since 9.8.0 unmounting is deterministic (no 500 ms timer), so a `<StrictMode>` remount no longer destroys a live root; a Canvas hidden by `<Activity>` keeps its renderer and scene (9.8.1).
- Assets loaded through `useLoader`, `useGLTF` and `useTexture` stay cached for remounts; `.clear(url)` drops only the cache entry, so dispose geometries, materials and textures you no longer need yourself.
- The module-level `dispose(object)` export calls `dispose()` on the object (skipping Scenes) and on each of its own properties that has one (a mesh's geometry and material); it does not visit children, so traverse a tree yourself.
