# Curves, Frames and Sweeps (Tube, Loft, Lathe, Extrude)

Purpose: custom curves, how `computeFrenetFrames` really builds its frames, sweeping profiles with TubeGeometry and the LoftGeometry addon, and lathe/extrude from computed profiles (Curve, TubeGeometry, ExtrudeGeometry and LoftGeometry checked against the r187 source). Plain constructor signatures of the built-in sweeps are in `threejs-geometry`.

## Curves

A curve is a subclass of `THREE.Curve` implementing `getPoint(t, target)`; `getPointAt`, `getPoints`, `getSpacedPoints`, `getLength`, `getTangent`, `getTangentAt` and `computeFrenetFrames` come with it.

```javascript
class Lissajous extends THREE.Curve {
  getPoint(t, target = new THREE.Vector3()) {
    const a = t * Math.PI * 2;
    return target.set(Math.sin(3 * a), Math.sin(2 * a), Math.cos(5 * a));
  }
}
const path = new Lissajous();
const uniformInT = path.getPoints(200); // uniform in the parameter t
const uniformInDistance = path.getSpacedPoints(200); // uniform in arc length: use this for rings and spacing
```

- Arc-length lookups are cached (`arcLengthDivisions`, default 200). After changing a curve's control data, set `curve.needsUpdate = true` or call `curve.updateArcLengths()` before the next `getPointAt` / `getSpacedPoints`.
- `three/addons/curves/CurveExtras.js` ships ready-made curves: `GrannyKnot`, `HeartCurve`, `VivianiCurve`, `KnotCurve`, `HelixCurve`, `TrefoilKnot`, `TorusKnot`, `CinquefoilKnot`, `TrefoilPolynomialKnot`, `FigureEightPolynomialKnot` and `DecoratedTorusKnot4a/4b/5a/5c`. NURBS curves and surfaces live in `three/addons/curves/NURBSCurve.js` and `NURBSSurface.js`. Timing along a curve by arc length is in `threejs-math-motion`.

## Frames Along a Curve

The Frenet frame is `T = P'/|P'|`, `N = T'/|T'|`, `B = T x N`. `N` is undefined where the curve is straight and flips 180 degrees at inflection points, which twists or pinches a swept profile. Do not build a sweep from the second derivative.

`curve.computeFrenetFrames(segments, closed = false)` returns `{ tangents, normals, binormals }`, each `segments + 1` `Vector3`. Despite the name it does not compute Frenet frames. It picks an initial normal perpendicular to the first tangent (along the smallest tangent component), then rotates the previous normal by the rotation that takes `T[i-1]` to `T[i]` (axis `T[i-1] x T[i]`, angle `acos(clamp(dot))`): parallel transport, a rotation-minimizing frame. `B = T x N`, so `N x B = T`. Tangents come from `getTangentAt`, so samples are uniform in arc length.

- On a closed curve pass `closed = true`: the code then spreads the leftover twist evenly along the frames so the last normal meets the first. On a `TrefoilKnot` with 100 segments the last normal sits about 3.07 rad from the first without the flag and about 0.002 rad with it.
- A sharp corner still pinches the profile: raise `segments` or round the path. The initial normal can jump when the start direction changes, so an animated path may spin as a whole. Frames also depend on `segments`; re-sampling changes the twist slightly.
- A ring point is `P + (N cos a + B sin a) * r`. To orient an object along the path, use the frame as a basis:

```javascript
const frames = path.computeFrenetFrames(100, true);
const points = path.getSpacedPoints(100); // same arc-length samples as the frames
const marker = new THREE.Mesh(new THREE.ConeGeometry(0.05, 0.2, 8).rotateX(Math.PI / 2), new THREE.MeshNormalMaterial());
marker.position.copy(points[10]);
const basis = new THREE.Matrix4();
marker.quaternion.setFromRotationMatrix(basis.makeBasis(frames.normals[10], frames.binormals[10], frames.tangents[10])); // local +Z follows the tangent
scene.add(marker);
```

## Tube and Loft

`TubeGeometry(path, tubularSegments = 64, radius = 1, radialSegments = 8, closed = false)` uses those frames (exposed as `geometry.tangents`, `geometry.normals`, `geometry.binormals`). Its radius is constant. For a radius or a profile that changes along the path, build one ring of points per frame and hand the rings to `LoftGeometry`:

```javascript
import { LoftGeometry } from "three/addons/geometries/LoftGeometry.js";

function sweepTube(curve, segments, radialSegments, radiusAt, closed = false) {
  const frames = curve.computeFrenetFrames(segments, closed);
  const points = curve.getSpacedPoints(segments);
  const sections = [];
  for (let i = 0; i <= segments; i++) {
    const radius = radiusAt(i / segments);
    const ring = [];
    for (let j = 0; j < radialSegments; j++) {
      const a = (j / radialSegments) * Math.PI * 2;
      ring.push(
        points[i].clone()
          .addScaledVector(frames.normals[i], Math.cos(a) * radius)
          .addScaledVector(frames.binormals[i], Math.sin(a) * radius),
      );
    }
    sections.push(ring);
  }
  return new LoftGeometry(sections, { closed: true, capStart: !closed, capEnd: !closed });
}

const tube = sweepTube(path, 200, 16, (s) => 0.1 + 0.06 * Math.sin(s * Math.PI * 8), true);
```

`LoftGeometry(sections, { closed = true, capStart = false, capEnd = false })` takes at least two sections (arrays of `Vector3`) of equal point count, and logs an error and builds nothing otherwise. `closed` treats each section as a ring: it repeats the first point of every section and averages the seam normals. Normals point outward when each section runs counterclockwise seen from the end of the loft looking back to the start; with `a` increasing from `N` toward `B` and `N x B = T` the rings above satisfy this. If a loft shows inside out, reverse each section. For a closed path the last section duplicates the first, which is intended: it closes the tube. The `uv` follows arc length in both directions.

## Lathe and Extrude from Computed Profiles

```javascript
// Lathe: x is the radius (>= 0), y the height; the profile revolves around Y
const n = 40;
const profile = Array.from({ length: n + 1 }, (_, i) => {
  const s = i / n;
  return new THREE.Vector2(0.15 + 0.35 * Math.sin(s * Math.PI) + 0.05 * Math.sin(s * 30), s * 2);
});
const vase = new THREE.LatheGeometry(profile, 48); // points listed bottom to top face outward

// Extrude a computed outline; Shape accepts an array of Vector2
const star = Array.from({ length: 10 }, (_, i) => {
  const a = (i / 10) * Math.PI * 2;
  const rr = i % 2 === 0 ? 1 : 0.45;
  return new THREE.Vector2(Math.cos(a) * rr, Math.sin(a) * rr);
});
const prism = new THREE.ExtrudeGeometry(new THREE.Shape(star), { depth: 0.3, bevelEnabled: false });
```

- Lathe normals come from the profile's segment directions, averaged at interior points (r187 also normalizes the last point's normal). A profile that crosses the axis (negative `x`) turns inside out, and a point on the axis creates a pole.
- `ExtrudeGeometry` has `bevelEnabled: true` by default (`bevelThickness` 0.2, `bevelSize` = `bevelThickness - 0.1`): set `bevelEnabled: false` for a plain prism. Other options: `steps` (1), `depth` (1), `bevelOffset`, `bevelSegments` (3), `curveSegments` (12), `UVGenerator`.
- With `extrudePath` (a 3D curve) the shape follows `getSpacedPoints(steps)` and `computeFrenetFrames(steps, closed)`. Bevels are switched off and `depth` is not used; `steps` defaults to 1 so raise it; `closed` is read only from a `CatmullRomCurve3` (`extrudePath.closed`), any other curve is treated as open.
