---
name: threejs-react-three-fiber
description: "React Three Fiber 9 and drei 10 for three.js r187: Canvas props, the scene graph as JSX, useFrame, useThree and useLoader, pointer events, useGLTF and drei helpers, on-demand rendering, disposal, and WebGPURenderer through an async gl factory. Use when writing React components that render 3D, a Canvas is blank or black, useFrame re-renders, useGLTF clones share a skeleton, or setting up WebGPU in R3F. Not for plain three.js setup (use threejs-fundamentals), gltfjsx or compression (use threejs-asset-pipeline), TSL node code (use threejs-shaders). 中文: React 3D, R3F, drei, 宣告式場景, 載入模型, 指標事件"
---

# Three.js React Three Fiber

Targets `@react-three/fiber` 9.8.2 (peer: React `>=19 <19.4`, three `>=0.156`), `@react-three/drei` 10.7.9 and three.js r187. The Canvas creates a WebGLRenderer by default; WebGPURenderer setup is below and in `references/webgpu.md`.

## Quick Start

The Canvas creates and owns a renderer, a scene and a camera; three.js classes become lowercase JSX elements. The Canvas fills its parent, so give the parent a height.

```jsx
import { useRef } from "react";
import { Canvas, useFrame } from "@react-three/fiber";
import { OrbitControls } from "@react-three/drei";

function Box() {
  const ref = useRef();
  useFrame((_, delta) => {
    ref.current.rotation.y += delta;
  });
  return (
    <mesh ref={ref} position={[0, 0.5, 0]} castShadow>
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color="#d9915b" />
    </mesh>
  );
}

export default function App() {
  return (
    <Canvas shadows="percentage" camera={{ position: [3, 2, 4], fov: 50 }}>
      <color attach="background" args={["#111827"]} />
      <ambientLight intensity={0.4} />
      <directionalLight position={[3, 4, 2]} intensity={2} castShadow />
      <Box />
      <mesh rotation-x={-Math.PI / 2} receiveShadow>
        <planeGeometry args={[10, 10]} />
        <meshStandardMaterial color="#20293a" />
      </mesh>
      <OrbitControls makeDefault />
    </Canvas>
  );
}
```

## Where Fiber Differs From Plain three.js

- **Tone mapping is on**: ACESFilmicToneMapping unless the Canvas is `flat` (NoToneMapping). Output is sRGB unless `linear`. Explicit `gl={{ toneMapping, outputColorSpace }}` wins over both shorthands.
- **Colour maps get sRGB automatically**: an RGBA8 texture assigned through `map`, `emissiveMap`, `sheenColorMap`, `specularColorMap` or `envMap` props gets `SRGBColorSpace` (skipped when the Canvas is `linear`).
- **Renderer defaults**: `new WebGLRenderer({ canvas, antialias: true, alpha: true, powerPreference: "high-performance" })`, `dpr={[1, 2]}`, `frameloop="always"`.
- **Default camera**: PerspectiveCamera fov 75, near 0.1, far 1000, at z 5 looking at the origin; aspect follows the canvas size unless you pass `aspect`/frustum values or `manual: true`.
- **`shadows`**: `true` (and the default `false`) select PCFSoftShadowMap, which three r182+ replaces with PCFShadowMap plus a console warning. Use `shadows="percentage"` (PCF), `"basic"` or `"variance"`.
- **Clock warning**: `state.clock` is a `THREE.Clock` fiber 9.8 creates, so every Canvas logs three's r183 Clock deprecation warning. It is not from your code; ignore it.
- Every Canvas prop, including `events`, `eventSource` and `eventPrefix`: `references/canvas-and-jsx.md`.

## JSX Rules

- Props set instance properties; values with `.set()` accept shorthand: `position={[1, 2, 3]}`, `scale={2}` (setScalar), `color="#d9915b"`. Dashed props pierce: `rotation-x`, `material-color`.
- `args` are constructor arguments; changing them rebuilds the object.
- Geometries and materials attach to their parent mesh automatically; `attach="material-0"` targets an array slot.
- `<primitive object={obj} />` mounts an existing object (one primitive per object; fiber never disposes it).
- Addon classes need `extend({ Water })` before `<water />`.
- Details, TypeScript tag collisions and disposal rules: `references/canvas-and-jsx.md`.

## Frame Loop and State

```jsx
useFrame((state, delta) => {
  ref.current.rotation.y += delta; // mutate through refs; never setState here
});
const invalidate = useThree((state) => state.invalidate); // selector form avoids re-renders
```

- `useFrame(callback, renderPriority = 0)`. A positive priority makes the Canvas stop rendering; your callback must then call `state.gl.render(state.scene, state.camera)`.
- Under `frameloop="demand"`, React prop changes repaint by themselves; imperative mutations need `invalidate()`.
- `useLoader(Loader, url, extensions?, onProgress?)` suspends and caches by loader and URL; put a `<Suspense>` above it.
- Hook and pointer-event details (handler list, `stopPropagation`, pointer capture): `references/hooks-and-events.md`.

## Loading Models

```jsx
import { Suspense, useMemo } from "react";
import { useGLTF } from "@react-three/drei";
import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js";

function Character({ url }) {
  const { scene } = useGLTF(url); // Draco and meshopt on by default; cached per URL
  const clone = useMemo(() => SkeletonUtils.clone(scene), [scene]); // one rebound copy per component
  return <primitive object={clone} />;
}

useGLTF.preload("/robot.glb");
// <Suspense fallback={null}><Character url="/robot.glb" /></Suspense>
```

drei helpers (`useTexture`, `Environment`, `ContactShadows`, `Text`, `Html`, `Center`, `Bounds`, `Instances`, `Detailed`, `AdaptiveDpr`, `PerformanceMonitor`, `useProgress`, `Preload`): `references/drei.md`. Generating typed components with gltfjsx and compressing files: `threejs-asset-pipeline`.

## WebGPU in Brief

```jsx
import * as THREE from "three/webgpu";
import { Canvas, extend } from "@react-three/fiber";

extend(THREE); // node materials and WebGPU classes join the JSX catalogue

<Canvas
  gl={async (props) => {
    const renderer = new THREE.WebGPURenderer(props); // props carry the Canvas's canvas
    await renderer.init();
    return renderer;
  }}
>
  {/* scene contents */}
</Canvas>;
```

Fiber waits for the promise and mounts the children once it resolves. Several drei helpers inject GLSL and break under WebGPU (`Text`, `ContactShadows`, `Html occlude="blending"`, some `Environment` modes). Full example, colour notes and the compatibility list: `references/webgpu.md`.

## Common Mistakes

- `requestAnimationFrame(tick)` → `useFrame((state, delta) => { ... })`: the Canvas already runs the loop; a second loop races it.
- `useFrame(() => { ref.current.rotation.y += 0.01; })` → use `delta`: a fixed step runs at a different speed on every refresh rate.
- `useFrame((_, delta) => setRotation((r) => r + delta))` → mutate `ref.current`: state set in the loop re-renders React every frame.
- `useFrame(cb, 1)` without rendering → drop the priority, or call `state.gl.render(state.scene, state.camera)` in `cb`: a positive priority stops the Canvas's own render and the screen goes black.
- `state.clock.getElapsedTime()` → `state.clock.elapsedTime`: `getElapsedTime()` calls `getDelta()`, which shortens the delta fiber measures for the next frame.
- `<mesh position={{ x: 1, y: 0, z: 0 }}>` → `position={[1, 0, 0]}`: `position` is a read-only property, so a plain object throws "Cannot assign to read only property".
- `import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js"` → `"three/addons/loaders/GLTFLoader.js"`: `three/addons/` resolves under bundlers and the import maps three.js documents; `examples/jsm` fails under those import maps.
- `useEffect(() => new GLTFLoader().load(url, ...), [])` → `useGLTF(url)`: the hook suspends, caches by URL and wires Draco and meshopt.
- `useGLTF(url).scene.clone()` → `useMemo(() => SkeletonUtils.clone(scene), [scene])`: `Object3D.clone()` keeps skinned copies bound to the original skeleton.
- Two `<primitive object={gltf.scene} />` → clone per instance: one object can sit in only one place in the scene graph.
- `scene.add(gltf.scene)` → `<primitive object={gltf.scene} />`: an imperatively added object gets no event props and outlives the component.
- `<water args={[geometry]} />` alone → `extend({ Water })` first: elements resolve against the registered catalogue only.
- `<line>` in TypeScript → `<threeLine>`: the bare tag collides with SVG `line`; fiber strips the `three` prefix (also `threeAudio`, `threeSource`, `threePath`).
- `<Canvas shadows>` → `<Canvas shadows="percentage">`: the boolean selects the removed PCFSoftShadowMap and triggers a warning.
- `castShadow` on the mesh only → also on the light, plus the Canvas `shadows` prop.
- `ref.current.position.x += 1` under `frameloop="demand"` → follow it with `invalidate()`.
- `gl={new THREE.WebGPURenderer()}` → the async factory above: a renderer built without fiber's canvas draws into a canvas nobody sees, and `render()` throws until `init()` resolves.
- `new OrbitControls(camera, gl.domElement)` in an effect → `<OrbitControls makeDefault />` from drei: it updates every frame, invalidates demand loops and disposes itself.
- `<meshBasicNodeMaterial>` without registration → call `extend(THREE)` with the `three/webgpu` namespace first: the Canvas seeds its catalogue from plain `three`, which has no node materials.
- `<Environment><Lightformer /></Environment>` under WebGPURenderer → `<Environment files="env.hdr" />` or a `preset`: the children form bakes through a cube camera whose reflections come out black on the WebGL 2 fallback, with no error.
- `colorNode={color(value)}` rebuilt every render → build nodes once (`useMemo` or module scope) and animate `uniform()` values: a new node is assigned without a recompile until `material.needsUpdate = true`.

## Reference Files

- `references/canvas-and-jsx.md` — every Canvas prop, JSX element and prop rules, `primitive`, `extend`, TypeScript tags, what fiber disposes and what it leaves to you. Open when configuring the Canvas or something does not mount or dispose as expected.
- `references/hooks-and-events.md` — `useFrame`, `useThree`, `useLoader`, `useGraph`, `advance` with `frameloop="never"`, pointer events and pointer capture, on-demand rendering. Open for frame-loop, state or event questions.
- `references/drei.md` — `useGLTF`, `useTexture`, `useProgress`, `Preload`, controls, `Environment`, `Lightformer`, `ContactShadows`, `Text`, `Html`, `Center`, `Bounds`, `Instances`, `Detailed`, `AdaptiveDpr`, `PerformanceMonitor`. Open when using any drei helper.
- `references/webgpu.md` — WebGPURenderer through the `gl` factory, TSL nodes in props, colour behaviour, and which fiber and drei features work on each backend. Open for any WebGPU or TSL work in R3F.

## See Also

- `threejs-fundamentals` — the renderer, scene, camera and loop the Canvas manages for you.
- `threejs-loaders` — loader configuration, decoders and runtime disposal in depth.
- `threejs-asset-pipeline` — gltfjsx components and compressing models before `useGLTF` loads them.
- `threejs-interaction` — raycasting and the controls classes behind drei's wrappers.
- `threejs-shaders` — TSL node logic and GLSL used inside fiber elements.
- `threejs-postprocessing` — post-processing passes (the `@react-three/postprocessing` package is not covered here).
- `threejs-performance` — profiling and optimisation beyond fiber's adaptive tools.
- `threejs-animation` — AnimationMixer and clips behind drei's `useAnimations`.
