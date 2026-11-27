# WebGPU and TSL in React Three Fiber

Purpose: running a fiber 9.8.2 Canvas on three.js r187 WebGPURenderer, passing TSL nodes through props, colour behaviour, and which fiber and drei 10.7.9 features work on each backend.

## Setup

The `gl` prop takes an async factory. Fiber calls it once per root with its defaults (`{ canvas, antialias: true, alpha: true, powerPreference: "high-performance" }`); construct the WebGPURenderer with those props, await its initialization, and return it. Fiber suspends the Canvas internally until the promise resolves and then mounts the children; a Suspense fallback around the Canvas does not show meanwhile, and a factory that throws reaches the nearest error boundary (9.8.0+).

Because the Canvas seeds its element catalogue from the plain `three` namespace (which has no node materials), merge the `three/webgpu` namespace in with `extend` before using node-material elements. TSL nodes from `three/tsl` pass through props the same way, but a material compiles its nodes once: build nodes once (module scope or `useMemo`) and animate through `uniform()` values, because a new node passed on a later render is assigned without a recompile until you set `material.needsUpdate = true`.

```jsx
import * as THREE from "three/webgpu";
import { color } from "three/tsl";
import { Canvas, extend, useFrame } from "@react-three/fiber";
import { useRef } from "react";

extend(THREE); // node materials and WebGPU classes join the element catalogue

function Knot() {
  const ref = useRef();
  useFrame((_, delta) => {
    ref.current.rotation.y += delta;
  });
  return (
    <mesh ref={ref}>
      <torusKnotGeometry args={[0.8, 0.3, 128, 24]} />
      <meshBasicNodeMaterial colorNode={color(0x42699c)} />
    </mesh>
  );
}

export default function App() {
  return (
    <Canvas
      camera={{ position: [3, 2, 4], fov: 50 }}
      gl={async (props) => {
        const renderer = new THREE.WebGPURenderer(props); // add forceWebGL: true to test the fallback
        await renderer.init();
        return renderer;
      }}
    >
      <color attach="background" args={["#111827"]} />
      <Knot />
    </Canvas>
  );
}
```

- Await `init()` inside the factory: fiber renders as soon as the promise resolves, and `WebGPURenderer.render()` throws before init. On unmount fiber skips `dispose()` for a renderer that never initialized.
- Fiber disposes a factory-built renderer on unmount and awaits its async `dispose()` (9.8.1+).
- Under a bundler, `three` and `three/webgpu` share `build/three.core.js`, so fiber's `three` import and your `three/webgpu` import see the same core classes. Without a bundler use the four-key import map from `threejs-fundamentals`.

## Colour

Fiber applies its colour defaults to a factory's renderer too: ACESFilmicToneMapping (NoToneMapping with `flat`) and sRGB output. WebGPURenderer tone-maps the whole frame, background included, so the same `<color attach="background">` renders darker than under WebGLRenderer unless the Canvas is `flat`.

## What works on each backend

Checked by rendering under WebGPURenderer on its WebGPU backend and with `forceWebGL: true` (fiber 9.8.2, drei 10.7.9, three r186; not re-run on r187).

Working on both backends: the scene graph, hooks, events (the raycaster tests CPU geometry, so `positionNode` displacement is invisible to it), `useGLTF` (loaded materials convert to node materials), `useTexture`, `Preload`, `Center`, `Bounds`, `Instances`, `Detailed`, `OrbitControls`, `CameraControls`, `Html` without `occlude="blending"`, and `Environment` with `preset`, `map` or `.hdr` files. `useProgress`, `AdaptiveDpr` and `PerformanceMonitor` call no renderer API (read in their source).

Not WebGPU-safe:

- `Text` draws one untextured quad instead of glyphs: troika injects its SDF shader through `onBeforeCompile`.
- `ContactShadows` uses an `onBeforeCompile` depth material and blur `ShaderMaterial`s, which three reports as not compatible.
- `Html` with `occlude="blending"` (ShaderMaterial).
- `Environment` with `ground`, or with `.jpg` / `.webp` gain-map `files`: both draw through ShaderMaterials (read in the source).
- `Environment` with children bakes through a cube camera that gives reflections on the WebGPU backend but leaves them black on the WebGL 2 fallback, with no error. Prefer `files="env.hdr"` or a `preset`, which only assign a texture.

TSL material slots (`colorNode`, `positionNode`, ...) behave exactly as in `threejs-shaders`; write `Fn` code, not GLSL. WebGPU post-processing: `threejs-postprocessing`.
