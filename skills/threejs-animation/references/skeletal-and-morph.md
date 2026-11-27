# Skeletons, Retargeting and Morph Targets

Purpose: working with bones and skinned meshes, cloning and retargeting characters with SkeletonUtils, building morph targets by hand, and the culling and performance issues of animated meshes. Verified against three.js r187 `src/objects/SkinnedMesh.js`, `src/animation/` and `examples/jsm/utils/SkeletonUtils.js`.

## Inspecting a Rig

```javascript
const skinnedMesh = model.getObjectByProperty("isSkinnedMesh", true); // first SkinnedMesh
const skeleton = skinnedMesh.skeleton;
console.log(skeleton.bones.map((b) => b.name));

const helper = new THREE.SkeletonHelper(model); // lines between bones; add it to the scene
scene.add(helper);
```

Bone names in glTF are sanitized for track binding (spaces become underscores, `.`, `:`, `/`, `[` and `]` are removed), so look them up by the name you see in `skeleton.bones`, not the one in the authoring tool.

## Attaching Objects to Bones

```javascript
const handBone = model.getObjectByName("RightHand");

const weapon = new THREE.Mesh(new THREE.BoxGeometry(0.08, 0.08, 0.5), new THREE.MeshNormalMaterial());
if (handBone) {
  handBone.add(weapon); // follows the bone's world transform
  weapon.position.set(0, 0, 0.25); // offset in the bone's local space
  weapon.rotation.set(0, Math.PI / 2, 0);
}
```

Bone scale applies to children; if the rig was exported with scaled bones (common from FBX, cm units), compensate on the attached object. `object.attach(child)` keeps the child's current world transform when re-parenting, `add` keeps its local one.

## SkeletonUtils

```javascript
import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js";

const copy = SkeletonUtils.clone(model); // clones the hierarchy and rebinds each SkinnedMesh to the copy's own bones
```

- Geometry and materials stay shared between clones; clone a material before changing it per character.
- Each clone needs its own `AnimationMixer`; the same `AnimationClip` objects can drive all of them.
- `SkeletonUtils.retargetClip(target, source, clip, options)` converts a clip authored for the `source` rig into one for the `target` rig; `SkeletonUtils.retarget(target, source, options)` poses `target` like `source` once. `options.names` maps target bone names to source bone names (or pass `options.getBoneName(targetBone)`), `options.hip` names the source hip bone (default `"hip"`), `options.scale` scales hip motion, and `options.useFirstFramePosition`, `options.preserveBoneMatrix`, `options.fps` tune the result. Both rigs should share a rest pose (T-pose to T-pose) or the result is skewed.

## Skinned Bounds and Culling

A SkinnedMesh computes its bounding sphere once, the first time frustum culling or raycasting needs it, from the current pose. A clip that moves the mesh far from that pose (a jump, a long attack) can get it culled while still on screen.

```javascript
mixer.update(dt);
mixer.getRoot().updateMatrixWorld();
skinnedMesh.computeBoundingSphere(); // CPU pass over every vertex with the current bone matrices
// or, cheaper: skinnedMesh.frustumCulled = false;
```

Raycasting a SkinnedMesh tests the skinned (posed) vertices on the CPU; for many characters, raycast simple proxy colliders instead.

## Morph Targets by Hand

```javascript
const geometry = mesh.geometry;

// A morph target is a full-size position (or normal) attribute beside the base one
const raised = geometry.attributes.position.array.slice(); // same vertex count
for (let i = 1; i < raised.length; i += 3) raised[i] += 0.5; // y is every third component

const target = new THREE.Float32BufferAttribute(raised, 3);
target.name = "raised"; // named targets are addressable from tracks and the dictionary

geometry.morphAttributes.position = [target];
geometry.morphTargetsRelative = false; // default: targets hold absolute positions, not deltas
mesh.updateMorphTargets(); // rebuilds morphTargetInfluences and morphTargetDictionary

mesh.morphTargetInfluences[mesh.morphTargetDictionary.raised] = 1;
```

- glTF morph targets load with `morphTargetsRelative = true` (deltas), as the format stores them.
- `AnimationClip.CreateFromMorphTargetSequence("walk", targets, 30)` builds a flip-book clip that steps through targets in order.

## Performance

- Share clips across mixers; never copy keyframe arrays per character.
- Skip `mixer.update()` for characters that are off screen or far away (an un-updated mixer holds its last pose), or update distant ones every other frame with the accumulated delta.
- Skinning runs per vertex on the GPU, and the mixer's per-bone sampling runs on the CPU; for crowds of skinned characters, measure first and see `threejs-performance` for strategies.
