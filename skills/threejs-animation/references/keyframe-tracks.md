# Keyframe Tracks and Clip Utilities

Purpose: building AnimationClips in code — track types, value layout, interpolation — and the clip utilities. Verified against three.js r187 `src/animation/` and `src/math/interpolants/`.

## Building a Clip

```javascript
// One track per animated property. values.length must be times.length * valueSize.
const times = [0, 1, 2]; // seconds, ascending
const values = [0, 1, 0]; // one scalar per keyframe for a Number track

const track = new THREE.NumberKeyframeTrack(".position[y]", times, values); // the mixer root's position.y
const clip = new THREE.AnimationClip("bounce", -1, [track]); // -1 derives the duration from the tracks (2 s)
```

`new AnimationClip(name = "", duration = -1, tracks = [], blendMode = THREE.NormalAnimationBlendMode)`. A KeyframeTrack throws immediately if its name is undefined or it has no times.

## Track Types

```javascript
// Number (value size 1); create the material with transparent: true, or opacity is ignored
new THREE.NumberKeyframeTrack(".material.opacity", [0, 1], [1, 0]);

// Vector (value size = vector length: 3 for position and scale)
new THREE.VectorKeyframeTrack(".position", [0, 1, 2], [0, 0, 0, 1, 2, 0, 0, 0, 0]);

// Quaternion (value size 4: x, y, z, w per keyframe — never Euler angles); slerps between keys
const q2 = new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0, 1, 0), Math.PI);
new THREE.QuaternionKeyframeTrack(".quaternion", [0, 1], [0, 0, 0, 1, q2.x, q2.y, q2.z, q2.w]);

// Color (value size 3: r, g, b in the working color space)
new THREE.ColorKeyframeTrack(".material.color", [0, 1], [1, 0, 0, 0, 1, 0]);

// Boolean and String: always discrete (step)
new THREE.BooleanKeyframeTrack(".visible", [0, 1], [true, false]);
new THREE.StringKeyframeTrack(".name", [0, 1], ["a", "b"]);
```

Rotation through `.rotation[y]` with a Number track works for a single axis; for full orientations use a Quaternion track on `.quaternion`.

## Interpolation

```javascript
track.setInterpolation(THREE.InterpolateLinear); // default for Number, Vector, Color, Quaternion
track.setInterpolation(THREE.InterpolateSmooth); // cubic through the keyframes
track.setInterpolation(THREE.InterpolateDiscrete); // step, no in-between values
```

### Bezier tangents

`InterpolateBezier` reads absolute `(time, value)` control points from `track.settings.inTangents` and `track.settings.outTangents`. For N keyframes of value size S, each array holds `N * S * 2` numbers: per keyframe, per component, a time then a value. Segment k→k+1 uses keyframe k's out-tangent and keyframe k+1's in-tangent. Without tangents the interpolant falls back to linear.

```javascript
const morphTrack = new THREE.NumberKeyframeTrack(".morphTargetInfluences[smile]", [0, 1], [0, 1]);
morphTrack.setInterpolation(THREE.InterpolateBezier);
morphTrack.settings = {
  inTangents: [0, 0, 1, 1], // key 0: (0, 0)  key 1: (1, 1)
  outTangents: [0, 1, 1, 0], // key 0: (0, 1)  key 1: (1, 0) — curve 0→1 rises fast, then eases into 1
};
```

The tangents are copied into the interpolant when the mixer binds the track, so set `settings` before the first `play()`. In r187 `track.shift()` and `track.scale()` move the tangent times along with the keys (`shift()` ignored them before r187). ColladaLoader emits this format for Bezier curves (FBXLoader does not use it).

## Clip Utilities

```javascript
const clip = new THREE.AnimationClip("walk", 2, [
  new THREE.NumberKeyframeTrack(".rotation[y]", [0, 1, 2], [0, Math.PI, Math.PI * 2]),
]);

const walk = THREE.AnimationClip.findByName([clip], "walk"); // by name; also accepts an object with .animations
const copy = clip.clone(); // deep copy, tangents included

// Keeps existing keys in frames [startFrame, endFrame) at fps (default 30); never resamples
const cut = THREE.AnimationUtils.subclip(clip, "first-second", 0, 31, 30); // keeps the keys at 0 s and 1 s

clip.resetDuration(); // duration = longest track's last time; it does not seek or restart anything
clip.trim(); // cut every track to [0, duration]
clip.optimize(); // removes keys at duplicate times and keys equal to both neighbours (not on InterpolateSmooth tracks)
clip.validate(); // false (and logs) for a fractional values/times ratio, an empty track, NaN or unsorted times
```

- `optimize()` on an `InterpolateBezier` track can remove keys but leaves `settings` tangent arrays at their old length, which misaligns them with the remaining keys. Do not optimize Bezier clips.
- `AnimationClip.CreateFromMorphTargetSequence(name, morphTargets, fps, noLoop)` and `CreateClipsFromMorphTargetSequences(morphTargets, fps, noLoop)` build flip-book clips from morph targets named like `walk_000`, `walk_001`.
- `AnimationClip.parse(json)` / `AnimationClip.toJSON(clip)` (and `clip.toJSON()`) round-trip clips, tangents included.
- A track must keep `times` ascending; `AnimationUtils.sortedArray` and `getKeyframeOrder` help when building them from unsorted data.
