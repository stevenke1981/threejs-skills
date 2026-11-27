# Built-in Geometries, Text, Edges and Wireframes

Purpose: full constructor signatures of every core geometry class, the path-based shapes, 3D text, and edge/wireframe helpers (checked against the r187 source).

All parameters are optional; the comments list them in order with the r187 defaults where they matter. Segment counts trade smoothness for vertex count: the sphere default of 32 by 16 suits most uses, go higher only for close-ups.

## Basic Shapes

```javascript
new THREE.BoxGeometry(1, 1, 1); // width=1, height=1, depth=1, widthSegments=1, heightSegments=1, depthSegments=1
new THREE.SphereGeometry(1, 32, 16); // radius=1, widthSegments=32, heightSegments=16, phiStart=0, phiLength=2π, thetaStart=0, thetaLength=π
new THREE.SphereGeometry(1, 32, 16, 0, Math.PI); // half sphere (phi sweep, cut through the poles)
new THREE.SphereGeometry(1, 32, 16, 0, Math.PI * 2, 0, Math.PI / 2); // dome (cut at the equator)
new THREE.PlaneGeometry(10, 10, 1, 1); // width=1, height=1, widthSegments=1, heightSegments=1
new THREE.CircleGeometry(1, 32); // radius=1, segments=32, thetaStart=0, thetaLength=2π
new THREE.CylinderGeometry(1, 1, 2, 32); // radiusTop=1, radiusBottom=1, height=1, radialSegments=32, heightSegments=1, openEnded=false, thetaStart=0, thetaLength=2π
new THREE.CylinderGeometry(1, 1, 2, 6); // hexagonal prism
new THREE.ConeGeometry(1, 2, 32); // radius=1, height=1, radialSegments=32, heightSegments=1, openEnded=false, thetaStart=0, thetaLength=2π
new THREE.TorusGeometry(1, 0.4, 16, 100); // radius=1, tube=0.4, radialSegments=12, tubularSegments=48, arc=2π, thetaStart=0, thetaLength=2π
new THREE.TorusGeometry(1, 0.4, 16, 100, Math.PI); // half arc
new THREE.TorusKnotGeometry(1, 0.4, 100, 16, 2, 3); // radius=1, tube=0.4, tubularSegments=64, radialSegments=8, p=2, q=3
new THREE.RingGeometry(0.5, 1, 32, 1); // innerRadius=0.5, outerRadius=1, thetaSegments=32, phiSegments=1, thetaStart=0, thetaLength=2π
```

## Capsule and Polyhedra

```javascript
// radius=1, height=1 (the middle section only), capSegments=4, radialSegments=8, heightSegments=1
// r176 renamed the second parameter from `length` to `height` and added heightSegments
new THREE.CapsuleGeometry(0.5, 1, 4, 8, 1);

// radius=1, detail=0 - each detail level subdivides every face
new THREE.IcosahedronGeometry(1, 0); // 20 faces at detail 0
new THREE.DodecahedronGeometry(1, 0);
new THREE.OctahedronGeometry(1, 0);
new THREE.TetrahedronGeometry(1, 0);

// Polyhedron - flat vertices array, indices, radius=1, detail=0
const vertices = [1, 1, 1, -1, -1, 1, -1, 1, -1, 1, -1, -1];
const indices = [2, 1, 0, 0, 3, 2, 1, 3, 0, 2, 3, 1];
new THREE.PolyhedronGeometry(vertices, indices, 1, 0);
```

## Path-Based Shapes

The constructors below take a fixed profile or path. For profiles and paths computed from equations, rotation-minimizing frames, a radius that varies along a tube, `LoftGeometry` and `extrudePath` behaviour, see `threejs-math-procedural-geometry`.

```javascript
// Lathe - a Vector2 profile rotated around the Y axis: points, segments=12, phiStart=0, phiLength=2π
const profile = [
  new THREE.Vector2(0, 0), new THREE.Vector2(0.5, 0), new THREE.Vector2(0.5, 1), new THREE.Vector2(0, 1),
];
new THREE.LatheGeometry(profile, 32);

// Shape - a flat 2D outline: shapes, curveSegments=12
const shape = new THREE.Shape();
shape.moveTo(0, 0);
shape.lineTo(1, 0);
shape.lineTo(1, 1);
shape.lineTo(0, 1);
new THREE.ShapeGeometry(shape);

// Extrude - a 2D Shape pushed along Z. Defaults: steps 1, depth 1, bevelEnabled TRUE,
// bevelThickness 0.2, bevelSize bevelThickness - 0.1, bevelOffset 0, bevelSegments 3, curveSegments 12
new THREE.ExtrudeGeometry(shape, {
  steps: 1, depth: 1, bevelEnabled: true,
  bevelThickness: 0.1, bevelSize: 0.1, bevelSegments: 3,
});
new THREE.ExtrudeGeometry(shape, { depth: 1, bevelEnabled: false }); // a plain prism

// Tube - a circular sweep along a curve: path, tubularSegments=64, radius=1, radialSegments=8, closed=false
const curve = new THREE.CatmullRomCurve3([
  new THREE.Vector3(-1, 0, 0), new THREE.Vector3(0, 1, 0), new THREE.Vector3(1, 0, 0),
]);
new THREE.TubeGeometry(curve, 64, 0.2, 8, false);
```

## Text Geometry

`TextGeometry` and `FontLoader` are addons, not part of the `THREE` namespace.

```javascript
import { FontLoader } from "three/addons/loaders/FontLoader.js";
import { TextGeometry } from "three/addons/geometries/TextGeometry.js";

// A .typeface.json font, e.g. from examples/fonts in the three.js repository
const font = await new FontLoader().loadAsync("fonts/helvetiker_regular.typeface.json");
const geometry = new TextGeometry("Hello", {
  font: font,
  size: 1, // text size in world units; default 100
  depth: 0.2, // extrusion depth; default 50 (renamed from `height` in r163)
  curveSegments: 12,
  bevelEnabled: true, // default false for text (ExtrudeGeometry defaults to true)
  bevelThickness: 0.03, // default 10
  bevelSize: 0.02, // default 8
  direction: "ltr", // 'ltr', 'rtl' (right to left) or 'tb' (top to bottom)
});

geometry.center(); // recentre the glyphs on the origin
scene.add(new THREE.Mesh(geometry, new THREE.MeshNormalMaterial()));
```

The defaults are sized for fonts in font units, so a TextGeometry with only `font` set is 100 units tall and 50 deep: always pass `size` and `depth` in world units. TextGeometry is an ExtrudeGeometry, so `steps`, `bevelOffset`, `bevelSegments`, `extrudePath` and `UVGenerator` also apply.

## Edges and Wireframes

```javascript
const boxGeometry = new THREE.BoxGeometry(1, 1, 1);
// Edges where the two faces' normals differ by at least thresholdAngle (degrees, default 1; 15 here), plus open boundary edges
const edges = new THREE.EdgesGeometry(boxGeometry, 15);
const edgeLines = new THREE.LineSegments(edges, new THREE.LineBasicMaterial({ color: 0xffffff }));
// Every triangle edge, diagonals included; an edge shared by two triangles is drawn once
const wireframe = new THREE.WireframeGeometry(boxGeometry);
const wireLines = new THREE.LineSegments(wireframe, new THREE.LineBasicMaterial({ color: 0xffffff }));
scene.add(edgeLines, wireLines);
```

## Other Geometry Addons

`three/addons/geometries/` also ships `RoundedBoxGeometry`, `ConvexGeometry`, `DecalGeometry`, `TeapotGeometry`, `BoxLineGeometry`, `ParametricGeometry` and `LoftGeometry` (the last two are covered in `threejs-math-procedural-geometry`).
