# Points and Lines

Purpose: point clouds, line primitives and thick lines with Line2, including what changes under WebGPURenderer (checked against the r187 source).

## Points

```javascript
const count = 1000;
const positions = new Float32Array(count * 3).map(() => (Math.random() - 0.5) * 10);
const geometry = new THREE.BufferGeometry();
geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));
// sizeAttenuation (default true) shrinks points with depth under a perspective camera:
// a point spans size * (canvas CSS height / 2) / depth CSS pixels
const material = new THREE.PointsMaterial({ size: 0.1, color: 0xffffff });
const points = new THREE.Points(geometry, material);
scene.add(points);
```

- Per-point colors: add a `color` attribute and set `vertexColors: true`.
- Round or textured points: `map` or `alphaMap` on `PointsMaterial` plus `transparent: true` or `alphaTest`.
- Under WebGPURenderer with the WebGPU backend, point primitives are always one pixel; larger points use an instanced `Sprite` with `PointsNodeMaterial` (see webgpu-tsl.md).
- Raycasting points uses `raycaster.params.Points.threshold` (world units); see `threejs-interaction`.

## Line Primitives

```javascript
const linePoints = [
  new THREE.Vector3(-1, 0, 0), new THREE.Vector3(0, 1, 0), new THREE.Vector3(1, 0, 0),
];
const geometry = new THREE.BufferGeometry().setFromPoints(linePoints);
const material = new THREE.LineBasicMaterial({ color: 0xff0000 });

// Line connects the points in order; LineLoop closes back to the first point
const line = new THREE.Line(geometry, material);
const loop = new THREE.LineLoop(geometry, material);
scene.add(line, loop);

// LineSegments draws independent pairs: vertex 0-1, 2-3, ...
const segmentsGeometry = new THREE.BufferGeometry().setFromPoints([linePoints[0], linePoints[1], linePoints[1], linePoints[2]]);
const segments = new THREE.LineSegments(segmentsGeometry, material);
scene.add(segments);
```

- `LineDashedMaterial` needs `line.computeLineDistances()` after the geometry is set.
- LineLoop is not supported under WebGPURenderer (either backend): it logs "Objects of type THREE.LineLoop are not supported" and draws nothing. Repeat the first point and use `Line`, or use `LineSegments`.
- `linewidth` on `LineBasicMaterial` is ignored: WebGL and WebGPU draw line primitives one pixel wide.

## Thick Lines (Line2)

```javascript
import { Line2 } from "three/addons/lines/Line2.js";
import { LineGeometry } from "three/addons/lines/LineGeometry.js";
import { LineMaterial } from "three/addons/lines/LineMaterial.js";

const lineGeometry = new LineGeometry().setPositions([-1, 0, 0, 0, 1, 0, 1, 0, 0]); // flat xyz array
const lineMaterial = new LineMaterial({ color: 0xff0000, linewidth: 4 }); // CSS pixels unless worldUnits: true
const thickLine = new Line2(lineGeometry, lineMaterial);
thickLine.computeLineDistances(); // only needed for dashed lines
scene.add(thickLine);
```

- The resolution uniform is filled from the renderer viewport in `onBeforeRender`, so it follows resizes without manual updates.
- `LineSegments2` with `LineSegmentsGeometry` draws independent segments; `Wireframe` with `WireframeGeometry2` draws a mesh wireframe with width.
- Under WebGPURenderer import `Line2` from `three/addons/lines/webgpu/Line2.js` (also `LineSegments2.js`, `Wireframe.js` in that folder) and use `Line2NodeMaterial` from `three/webgpu`; the WebGL `LineMaterial` is a ShaderMaterial and does not work there.

```javascript
import * as THREE from "three/webgpu";
import { Line2 } from "three/addons/lines/webgpu/Line2.js";
import { LineGeometry } from "three/addons/lines/LineGeometry.js";

const thickLine = new Line2(
  new LineGeometry().setPositions([-1, 0, 0, 0, 1, 0, 1, 0, 0]),
  new THREE.Line2NodeMaterial({ color: 0xff0000, linewidth: 4 }),
);
scene.add(thickLine);
```
