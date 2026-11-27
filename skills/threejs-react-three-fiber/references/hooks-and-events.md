# Hooks, Frame Loop and Pointer Events

Purpose: `useFrame`, `useThree`, `useLoader`, `useGraph`, manual frame loops, on-demand rendering and pointer events in @react-three/fiber 9.8.2 (read from the package source).

## useFrame

`useFrame(callback, renderPriority = 0)` subscribes to the render loop; callbacks run before the scene renders and receive `(state, delta, xrFrame)`:

```jsx
import { useRef } from "react";
import { useFrame } from "@react-three/fiber";

function Spinner({ radius = 2 }) {
  const ref = useRef();
  useFrame((state, delta) => {
    ref.current.rotation.y += delta;
    ref.current.position.x = Math.sin(state.clock.elapsedTime) * radius;
  });
  return (
    <mesh ref={ref}>
      <sphereGeometry args={[0.4, 32, 16]} />
      <meshBasicMaterial color="#e8b4a0" />
    </mesh>
  );
}
```

- Subscribers run in ascending priority order. Any subscriber with a positive `renderPriority` takes rendering into its own hands: the Canvas stops calling `gl.render`, and some callback must call `state.gl.render(state.scene, state.camera)`. At default priority, calling `gl.render` yourself double-renders.
- `state.clock` is a `THREE.Clock` created by fiber (three logs its r183 deprecation warning once per Canvas). Read `state.clock.elapsedTime`; `getElapsedTime()` calls `getDelta()` and steals time from the next frame's delta.
- Never call `setState` in `useFrame`; mutate objects through refs. A store or `flushSync` covers the rare case the DOM must follow per-frame values.

## useThree

`useThree(selector?)` reads Canvas state reactively; with no selector it returns the whole state (`gl`, `camera`, `scene`, `raycaster`, `pointer`, `clock`, `size`, `viewport`, `invalidate`, `setDpr`, `setFrameloop`, `setSize`, `setEvents`, `events`, `controls`, `performance`, `get`, `set`):

```jsx
import { useThree } from "@react-three/fiber";

const camera = useThree((state) => state.camera);
const invalidate = useThree((state) => state.invalidate);
```

Use a selector so the component re-renders only when that slice changes. Inside `useFrame` use the `state` argument instead; it is the same state without a re-render.

## useLoader and useGraph

`useLoader(LoaderClassOrInstance, url | url[], extensions?, onProgress?)` suspends until the loader resolves and caches by loader and URL; its caller must sit under a Suspense boundary. A class is constructed once and memoized, so a setting the extensions callback makes (a path, a decoder) applies to every later `useLoader` call with that class. A result with a `scene` (glTF) also gets `nodes`, `materials` and `meshes` lookup maps.

```jsx
import { useGraph, useLoader } from "@react-three/fiber";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

function Model({ url }) {
  const gltf = useLoader(GLTFLoader, url, (loader) => loader.setPath("assets/"));
  const { nodes } = useGraph(gltf.scene); // { nodes, materials, meshes } keyed by name
  return <primitive object={gltf.scene} />;
}
```

`useLoader.preload(Loader, url, extensions?)` warms the cache outside React; `useLoader.clear(Loader, url)` drops an entry. Load failures reject with `Could not load <url>: <message>` and reach the nearest error boundary.

## On-demand and manual loops

Under `<Canvas frameloop="demand">` React prop changes repaint by themselves; imperative changes need `invalidate()`:

```jsx
import { useEffect, useRef } from "react";
import { useThree } from "@react-three/fiber";

function OnDemand() {
  const ref = useRef();
  const invalidate = useThree((state) => state.invalidate);
  useEffect(() => {
    const id = setInterval(() => {
      ref.current.rotation.y += 0.5;
      invalidate(); // request one frame for the imperative change
    }, 100);
    return () => clearInterval(id);
  }, [invalidate]);
  return (
    <mesh ref={ref}>
      <boxGeometry args={[1, 1, 1]} />
      <meshNormalMaterial />
    </mesh>
  );
}
```

- `invalidate()` called inside a `useFrame` callback requests one more frame.
- Since 9.8.2 removing an object under `frameloop="demand"` requests a frame, so it disappears immediately.
- `frameloop="never"` hands the loop over: call the module-level `advance(timestamp)` export with a time in **seconds**. In this mode the delta is the difference between successive timestamps, so milliseconds make it 1000 times too large.

## Pointer events

Objects with handlers are raycast on each pointer event (pointer moves test only objects with move, over, out, enter or leave handlers); handlers fire nearest first, bubbling from the hit object up its parents. The event carries the intersection plus helpers: `object`, `eventObject` (the object that registered the handler), `point`, `distance`, `face`, `instanceId`, `intersections`, `ray`, `camera`, `unprojectedPoint`, `delta` (pixels travelled since pointerdown, for click-like events), `nativeEvent`, `stopPropagation()`.

Handlers: `onClick`, `onDoubleClick`, `onContextMenu`, `onWheel`, `onPointerUp`, `onPointerDown`, `onPointerMove`, `onPointerOver`, `onPointerOut`, `onPointerEnter`, `onPointerLeave`, `onPointerMissed`. `onPointerCancel` and `onLostPointerCapture` are typed but fiber never calls them on objects; it uses the DOM events internally to cancel hover and capture state.

```jsx
import { useState } from "react";

function Pickable() {
  const [hovered, setHovered] = useState(false);
  return (
    <mesh
      onClick={(event) => {
        event.stopPropagation(); // objects behind this one along the ray no longer receive it
        console.log(event.object.name, event.distance, event.point, event.instanceId);
      }}
      onPointerOver={(event) => {
        event.stopPropagation();
        setHovered(true);
      }}
      onPointerOut={() => setHovered(false)}
      onPointerDown={(event) => {
        event.target.setPointerCapture(event.pointerId); // this pointer's events reach the mesh off its surface until release
      }}
      onPointerUp={(event) => {
        event.target.releasePointerCapture(event.pointerId);
      }}
      onPointerMissed={() => console.log("clicked elsewhere")}
    >
      <sphereGeometry args={[0.75, 32, 16]} />
      <meshStandardMaterial color={hovered ? "#e8b4a0" : "#42699c"} />
    </mesh>
  );
}
```

- The default raycaster tests CPU geometry with `intersectObject(object, true)`, under any renderer; GPU displacement (`positionNode`) is invisible to it.
- Tune it through the Canvas: `raycaster={{ params: { Line: { threshold: 0.1 } } }}`.
- An object with `raycast={() => null}` is skipped by picking.
- For raycasting outside the event system (custom tools, gameplay rays), use `state.raycaster` and the patterns in `threejs-interaction`.
