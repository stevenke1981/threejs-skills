# drei Helpers

Purpose: the subset of @react-three/drei 10.7.9 helpers this skill has checked (loading, controls, scene dressing, instancing, LOD, adaptive performance), with their real defaults. drei ships many more; check its docs for the rest. Every helper imports from `@react-three/drei`.

## Loading assets

Loading hooks suspend; put the boundary above them and the fallback shows while they load.

```jsx
import { Suspense } from "react";
import { Canvas } from "@react-three/fiber";
import { useGLTF, useProgress, Environment } from "@react-three/drei";

function Model({ url }) {
  const { scene, animations, nodes, materials } = useGLTF(url);
  return <primitive object={scene} />;
}

function Hud() {
  const { active, progress, errors } = useProgress(); // tracks three's DefaultLoadingManager
  return <div>{active ? `${progress.toFixed(0)}%` : errors.length > 0 ? "failed" : "done"}</div>;
}

export function App() {
  return (
    <>
      <Hud />
      <Canvas camera={{ position: [3, 2, 4], fov: 50 }}>
        <Suspense fallback={null}>
          <Environment preset="city" />
          <Model url="model.glb" />
        </Suspense>
      </Canvas>
    </>
  );
}
```

- `useGLTF(url, useDraco = true, useMeshopt = true, extendLoader?)` loads through three-stdlib's GLTFLoader (not `three/addons`). Draco decodes from `https://www.gstatic.com/draco/versioned/decoders/1.5.5/` by default: `useGLTF.setDecoderPath(path)` retargets it globally, a string as the second argument retargets one call, `false` disables Draco. `extendLoader(loader)` runs before the decoders are attached (for KTX2: `loader.setKTX2Loader(...)`). `useGLTF.preload(url)` warms the cache, `useGLTF.clear(url)` drops it. The result carries `scene`, `animations`, `cameras` plus `nodes` / `materials` / `meshes` lookup maps.
- Several copies of a skinned model: clone once per component with `useMemo(() => SkeletonUtils.clone(scene), [scene])` (`import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js"`) and mount each clone in its own primitive. drei's `<Clone object={scene} />` and `<Gltf src={url} />` do the cloning for you.
- `useTexture(input)` loads one URL, an array (`const [a, b] = useTexture([x, y])`) or a keyed object, and uploads each texture with `gl.initTexture` on WebGLRenderer. Colour space comes from fiber: when the texture is assigned through `map`, `emissiveMap`, `sheenColorMap`, `specularColorMap` or `envMap` props and is RGBA8, fiber sets `SRGBColorSpace`; scalar maps keep NoColorSpace. Assigned imperatively, set `colorSpace` yourself.
- `useProgress` is a plain store over `THREE.DefaultLoadingManager`, so the Hud above reads it outside the Canvas. Loaders built with their own LoadingManager do not report to it.
- `<Preload all />` calls `gl.compile` on the scene once when it mounts (with `all`, hidden objects too) and renders it into a 128 px cube target to upload textures, trading a one-time stall for no first-view hitches. Put it inside the Suspense boundary after the assets, or it runs before they load.

## Controls

`OrbitControls` constructs three-stdlib's OrbitControls, connects it to the event source (`events.connected`, the Canvas wrapper by default), updates it every frame at priority −1 (before user callbacks), calls `invalidate()` on change (so it works with `frameloop="demand"`), disposes it on unmount and turns damping on by default. `makeDefault` stores it in `state.controls` (Bounds and other helpers pick it up); `regress` calls `performance.regress()` on every change. Props pass straight through to the controls instance; `onChange`, `onStart`, `onEnd` subscribe to its events.

```jsx
import { OrbitControls } from "@react-three/drei";

<OrbitControls
  makeDefault
  enableDamping
  dampingFactor={0.1}
  target={[0, 1, 0]}
  minDistance={2}
  maxDistance={12}
  maxPolarAngle={Math.PI / 2}
  autoRotate
  autoRotateSpeed={0.5}
/>
```

`CameraControls` wraps yomotsu's camera-controls library the same way (ref methods such as `fitToBox`, `setLookAt` come from that library; it updates with the frame delta). Raw controls classes and their options: `threejs-interaction`.

## Scene dressing

```jsx
import { ContactShadows, Environment, Lightformer, Text, Html, Center, Bounds } from "@react-three/drei";

export function Dressing({ overlay }) { // overlay: a ref to a DOM element laid over the Canvas
  return (
    <group>
      <Environment resolution={256}>
        <Lightformer form="rect" intensity={2} position={[0, 4, 3]} scale={[8, 2, 1]} />
      </Environment>
      <ContactShadows position={[0, -1, 0]} opacity={0.6} scale={12} blur={2.4} far={4} />
      <Bounds fit clip observe margin={1.5}>
        <mesh>
          <sphereGeometry args={[1, 32, 16]} />
          <meshStandardMaterial metalness={0.9} roughness={0.2} />
        </mesh>
        <Center top position={[2.5, -1, 0]}>
          <mesh>
            <boxGeometry args={[1, 0.6, 1]} />
            <meshStandardMaterial />
          </mesh>
        </Center>
      </Bounds>
      <Text position={[0.75, 1.1, 0]} fontSize={0.4} anchorX="center" anchorY="bottom" color="#e8b4a0" outlineWidth={0.02}>
        three.js
      </Text>
      <Html portal={overlay} position={[2.5, 0.2, 0]} center distanceFactor={8} occlude="raycast">
        <div>dom overlay anchored to the scene</div>
      </Html>
    </group>
  );
}
```

- `ContactShadows` renders the scene from below into a depth target and blurs it: a soft contact-shadow plane with no shadow-map setup. `frames` defaults to `Infinity`; `frames={1}` bakes it once for still scenes.
- `Lightformer` is a card of light (`form`: `"rect"` default, `"circle"`, `"ring"`, `"box"`) meant to sit inside an `<Environment>` children scene that bakes into reflections.
- `Text` renders SDF text (troika) with layout props `maxWidth`, `textAlign`, `lineHeight`, `letterSpacing`, `characters`; it suspends while the font loads.
- `Html` projects a DOM node into the scene; `transform` renders it as a 3D-transformed overlay instead of a screen-facing one. `occlude` (`true` or `"raycast"`) hides it while geometry blocks the ray to its anchor, so never anchor it inside a mesh; `occlude="blending"` hides it by depth through a ShaderMaterial (WebGL only). Without `portal` it mounts into `events.connected` or the canvas's parent. Observed with fiber 9.8, drei 10.7 and React 19.2: an `Html` that mounts in the same commit as a default (WebGLRenderer) Canvas comes up empty, and so does one that remounts with its Canvas; passing `portal` a ref to a DOM element laid over the canvas avoids it, as the Dressing does: `<div ref={overlay} style={{ position: "absolute", inset: 0, pointerEvents: "none" }} />` after the Canvas, inside a `position: relative` wrapper.
- `Center` recentres children on their bounding box (`top` / `bottom` / `left` / `right` / `front` / `back` align to an edge; `precise` defaults to `true` and measures every vertex, `precise={false}` uses per-object bounds; `onCentered` reports the box). It measures vertex positions, so it misplaces drei `Text`, whose geometry is a unit quad; place Text with `anchorX`, `anchorY` and `position` instead.
- `Bounds` fits the camera around its children (`fit`, `clip` adjusts near/far, `observe` refits on resize, `margin` default 1.2); `useBounds()` returns its imperative API (`refresh(object?)`, `reset`, `moveTo`, `lookAt`, `fit`, `clip`; `to` is deprecated in favour of `moveTo` + `lookAt`).

## Instancing and LOD

Many copies of one shape share one draw call through `Instances`; each `Instance` is a declarative transform (and optional `color`) that composes into the instanced matrix every frame (`frames` default `Infinity`):

```jsx
import { Instance, Instances } from "@react-three/drei";

function Crowd({ count = 200 }) {
  return (
    <Instances limit={count} castShadow>
      <capsuleGeometry args={[0.25, 0.8, 4, 12]} />
      <meshStandardMaterial />
      {Array.from({ length: count }, (_, i) => (
        <Instance
          key={i}
          position={[(i % 15) - 7, 0.65, Math.floor(i / 15) - 7]}
          color={`hsl(${(i * 37) % 360}, 60%, 55%)`}
        />
      ))}
    </Instances>
  );
}
```

`limit` (default 1000) caps the instance buffer; `range` draws fewer. For thousands of static copies, set matrices once on a plain `<instancedMesh>` instead (`threejs-geometry`).

`Detailed` picks one of its children by camera distance (three's `LOD`): child i shows from `distances[i]` on (the first entry is 0), and `hysteresis` (default 0) is a fraction of that distance that delays the switch back to finer detail:

```jsx
import { Detailed } from "@react-three/drei";

<Detailed distances={[0, 15, 40]} hysteresis={0.1}>
  <mesh><sphereGeometry args={[1, 64, 32]} /><meshStandardMaterial /></mesh>
  <mesh><sphereGeometry args={[1, 16, 8]} /><meshStandardMaterial /></mesh>
  <mesh><sphereGeometry args={[1, 8, 4]} /><meshStandardMaterial /></mesh>
</Detailed>
```

## Adaptive resolution

Controls with `regress` drop `performance.current` to `performance.min` (default 0.5) while they move and restore `performance.max` after `performance.debounce` (default 200 ms). `AdaptiveDpr` sets the pixel ratio to `current × viewport.initialDpr` (the first ratio the Canvas resolved), so every drag ends at that first ratio; `pixelated` switches `image-rendering` while lowered. `PerformanceMonitor` samples real frame rates and reports through `onIncline`, `onDecline`, `onChange` and `onFallback` (after `flipflops` changes, default `Infinity`); it does not touch the pixel ratio itself. Drive the Canvas `dpr` prop from its callbacks through React state, because the Canvas re-applies that prop when it re-renders, which undoes a `setDpr` call made from inside it; keep `AdaptiveDpr` out of that Canvas, or the next drag restores the first ratio over a lowered `dpr`:

```jsx
import { useState } from "react";
import { Canvas } from "@react-three/fiber";
import { OrbitControls, AdaptiveDpr, PerformanceMonitor } from "@react-three/drei";

// lower the resolution while the controls move
export function DragTunedApp() {
  return (
    <Canvas dpr={[1, 2]}>
      <OrbitControls regress />
      <AdaptiveDpr pixelated />
      {/* scene contents */}
    </Canvas>
  );
}

// let the measured frame rate pick the resolution through the dpr prop
export function MonitoredApp() {
  const [dpr, setDpr] = useState([1, 2]);
  return (
    <Canvas dpr={dpr}>
      <OrbitControls />
      <PerformanceMonitor
        onDecline={() => setDpr(1)}
        onIncline={() => setDpr([1, 2])}
        flipflops={3}
        onFallback={() => setDpr(1)}
      />
      {/* scene contents */}
    </Canvas>
  );
}
```

Beyond fiber-specific tools (capping `dpr`, KTX2 textures, merging, instancing, disposal): `threejs-performance`.
