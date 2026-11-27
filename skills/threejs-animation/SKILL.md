---
name: threejs-animation
description: "three.js r187 clip animation: AnimationMixer, AnimationClip and keyframe tracks, AnimationAction play, loop, fades and crossfades, weight and additive blending, glTF clips, skeletons and bones, morph targets, SkeletonUtils clone and retarget, TSL time on WebGPU. Use when a glTF animation will not play, crossFadeTo, blending idle and run, a cloned character stuck or following the original, attaching a weapon to a bone, driving shape keys. Not for easing, damping or springs (use threejs-math-motion), loading the file (use threejs-loaders), custom vertex shaders (use threejs-shaders). 中文: 動畫, 骨骼動畫, 動作混合, 動畫淡入淡出, 變形目標, 角色動畫"
---

# Three.js Animation

Targets three.js r187. Examples use WebGLRenderer; the animation system runs on the CPU and drives models identically under WebGPURenderer (`references/webgpu-tsl.md` covers GPU-side motion in TSL). Procedural maths — easing, damping, springs, shake — is `threejs-math-motion`.

## Quick Start: Play a glTF Clip

```javascript
import * as THREE from "three";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
const sun = new THREE.DirectionalLight(0xffffff, 2); // glTF materials are lit: no light, black model
sun.position.set(3, 5, 2);
scene.add(sun, new THREE.HemisphereLight(0xffffff, 0x444444, 1.5));

const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 1.5, 4);

const gltf = await new GLTFLoader().loadAsync("character.glb");
scene.add(gltf.scene);

const mixer = new THREE.AnimationMixer(gltf.scene); // one mixer per animated root
const clip = THREE.AnimationClip.findByName(gltf.animations, "Walk") ?? gltf.animations[0];
const walk = mixer.clipAction(clip); // cached: the same clip and root return the same action
walk.play();

const timer = new THREE.Timer(); // core since r179; THREE.Clock is deprecated (r183) and warns
timer.connect(document); // delta is 0 while the tab is hidden

renderer.setAnimationLoop(() => {
  timer.update(); // first thing each frame
  mixer.update(timer.getDelta()); // nothing moves without this
  renderer.render(scene, camera);
});
```

## How the System Fits Together

- **AnimationClip** — data: a name, a duration and an array of KeyframeTracks. No playback state; one clip can feed any number of mixers.
- **KeyframeTrack** — times plus values for one property path (`"Hips.quaternion"`, `".morphTargetInfluences[smile]"`).
- **AnimationMixer** — plays clips on one root and its descendants; `update(dt)` samples every active action and writes plain object properties.
- **AnimationAction** — playback state of one clip on one mixer: time, weight, time scale, loop mode, fades.

Track names follow `nodeName.objectName[objectIndex].propertyName[propertyIndex]`; only the property is required. `.position` binds the mixer root, `Hips.position` a named descendant (that is how bones are addressed), `.material.opacity` a material property, `.position[y]` one component, `.morphTargetInfluences[smile]` one morph target by name. The supported object names are `material`, `materials`, `bones` and `map`. Building clips by hand is in `references/keyframe-tracks.md`.

## Actions: the Calls You Need

```javascript
action.play(); // schedule; returns the action
action.reset(); // time 0, cancel fades, re-enable a finished one-shot
action.stop(); // unschedule now; properties no other action animates go back to their original values
action.setLoop(THREE.LoopOnce, 1); // LoopRepeat (default, Infinity), LoopOnce, LoopPingPong
action.clampWhenFinished = true; // hold the last frame of a LoopOnce action instead of snapping back
action.timeScale = -1; // negative plays backward; 0 or paused = true freezes
action.setEffectiveWeight(0.5); // blend share

mixer.addEventListener("finished", (e) => console.log(e.action.getClip().name)); // LoopOnce or finite repetitions only
```

### Crossfades

```javascript
function fadeTo(from, to, seconds) {
  to.reset() // re-enables an action that an earlier fade-out disabled
    .setEffectiveWeight(1) // fades multiply action.weight, which a weight blend may have left at 0
    .play();
  from.crossFadeTo(to, seconds, true); // true also warps time scales so cycles line up during the fade
}

fadeTo(idle, walk, 0.4);
```

`a.crossFadeTo(b, duration, warp = false)` is `a.fadeOut(duration)` plus `b.fadeIn(duration)`, with an optional time-scale warp. A fade ramps a 0→1 or 1→0 multiplier on `action.weight`, and an action that finishes fading out is disabled until `reset()`. Both actions must be playing for the fade to show. Weight blending, additive layers, `halt`, `warp`, `syncWith` and mixer-wide time control are in `references/blending-and-actions.md`.

## Skeletons and Bones

```javascript
const skinnedMesh = gltf.scene.getObjectByProperty("isSkinnedMesh", true);
const bones = skinnedMesh.skeleton.bones; // bones are Object3Ds
const head = gltf.scene.getObjectByName("Head");

scene.add(new THREE.SkeletonHelper(gltf.scene)); // visualise the rig

renderer.setAnimationLoop(() => {
  timer.update();
  mixer.update(timer.getDelta()); // clips write the bone pose here
  if (head) head.rotation.y = Math.sin(timer.getElapsed()) * 0.3; // procedural override goes AFTER update()
  renderer.render(scene, camera);
});

const hand = gltf.scene.getObjectByName("RightHand");
hand?.add(sword); // attach = parent to the bone; sword's position/rotation are in the bone's space
```

`mixer.update()` overwrites every property the playing clips animate, so a value set before it is lost. After it, `=` replaces the clip's value for that frame. To layer an offset on top of the clip, `+=` works only on a property the clip rewrites every frame; on a property no clip touches, `+=` accumulates frame after frame.

### Several Characters from One File

```javascript
import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js";

const copy = SkeletonUtils.clone(gltf.scene); // its own bones; Object3D.clone() would share the skeleton
scene.add(copy);
const copyMixer = new THREE.AnimationMixer(copy); // its own mixer, same clips
copyMixer.clipAction(clip).play();
```

Update every mixer each frame. Retargeting clips between rigs (`SkeletonUtils.retargetClip`), bone attachment details and the skinned bounding-sphere culling problem are in `references/skeletal-and-morph.md`.

## Morph Targets

```javascript
const face = gltf.scene.getObjectByName("Face");
const index = face.morphTargetDictionary["Smile"]; // name -> index
if (index !== undefined) face.morphTargetInfluences[index] = 0.8; // 0 = base shape, 1 = full target

// Keyframed: a Number track by target name
const smile = new THREE.NumberKeyframeTrack(".morphTargetInfluences[Smile]", [0, 0.5, 1], [0, 1, 0]);
new THREE.AnimationMixer(face).clipAction(new THREE.AnimationClip("smile", -1, [smile])).play();
```

Adding morph targets by hand and `AnimationClip.CreateFromMorphTargetSequence` are in `references/skeletal-and-morph.md`.

## Things That Go Wrong

- Nothing moves → `mixer.update(delta)` is missing, the delta is always 0 (Timer never updated), or the mixer was created on a different root than the one the clip's node names resolve under.
- A one-shot snaps back to frame 0 → `action.clampWhenFinished = true` with `LoopOnce`.
- A second crossfade does nothing → the target action finished or faded to weight 0; `reset()` it before fading in.
- A clone stands still or mirrors the original → it was made with `Object3D.clone()`; use `SkeletonUtils.clone()` and give it its own mixer.
- A character vanishes near the screen edge → the SkinnedMesh bounding sphere was computed once in the bind pose; see `references/skeletal-and-morph.md`.
- The page hangs on `play()` → a track's `values.length` is not a whole multiple of `times.length`; the fractional value size makes the mixer loop forever (`clip.validate()` returns false and logs `Invalid value size in track`).
- Opacity track has no effect → the material needs `transparent: true`.

## Common Mistakes

- `const clock = new THREE.Clock()` → `const timer = new THREE.Timer(); timer.connect(document);`: Clock is deprecated since r183 and warns on construction.
- `clock.getElapsedTime(); clock.getDelta();` → `timer.update(); timer.getDelta(); timer.getElapsed();`: Clock's getElapsedTime() calls getDelta() internally, so the second read is about zero and the mixer stalls.
- `requestAnimationFrame(animate)` → `renderer.setAnimationLoop(animate)`: it also hands frames to a WebXR session.
- `headBone.rotation.y = x; mixer.update(dt);` → `mixer.update(dt); headBone.rotation.y = x;`.
- `new THREE.StringKeyframeTrack(".morphTargetInfluences[smile]", …)` → `NumberKeyframeTrack`: influences are numbers.
- `new THREE.NumberKeyframeTrack(".material.opacity", [0, 1, 2], [1, 0])` → as many values as `times.length × valueSize`.
- `clip.resetDuration()` to restart → `action.reset()`: resetDuration only recomputes the clip length.
- `const model2 = model.clone()` → `SkeletonUtils.clone(model)`; `import { clone } from …SkeletonUtils.js` → `import * as SkeletonUtils`, so the call cannot be mistaken for `model.clone()`.
- `import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js"` → `"three/addons/loaders/GLTFLoader.js"`.
- `mesh.rotation.y = THREE.MathUtils.damp(…)` and other smoothing questions → `threejs-math-motion`.

## Reference Files

- `references/keyframe-tracks.md` — open to build clips in code: track types and value sizes, interpolation (linear, smooth, discrete, Bezier tangents), `subclip`, `optimize`, `validate`, `clone`, JSON.
- `references/blending-and-actions.md` — open for the full AnimationAction/AnimationMixer API: weights and blend trees, additive clips, fades, warp, halt, sync, events, `setTime`, uncaching.
- `references/skeletal-and-morph.md` — open for bones, attachments, SkeletonHelper, SkeletonUtils `clone` / `retarget` / `retargetClip`, morph targets by hand, skinned bounds and culling, mixer performance.
- `references/webgpu-tsl.md` — open for GPU-side animation with TSL `time`, `deltaTime` and oscillators, and displacing skinned meshes correctly.

## See Also

- `threejs-math-motion` — easing, damping, springs, look-at and camera paths.
- `threejs-loaders` — loading glTF/FBX files and their `animations`.
- `threejs-fundamentals` — the render loop and Timer.
- `threejs-shaders` — custom vertex displacement in GLSL or TSL.
