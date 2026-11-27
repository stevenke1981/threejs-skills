# Actions, Blending and the Mixer

Purpose: the full AnimationAction and AnimationMixer API — playback control, weight blending, additive layers, fades and warps, events, time control and freeing a mixer. Verified against three.js r187 `src/animation/AnimationAction.js`, `AnimationMixer.js` and `AnimationUtils.js`.

## AnimationMixer

| Call | Effect |
|---|---|
| `new AnimationMixer(root)` | Plays clips on `root` and its descendants |
| `mixer.clipAction(clip, optionalRoot, blendMode)` | Returns the cached action for that clip and root, creating it once. `clip` may also be a clip name |
| `mixer.existingAction(clip, optionalRoot)` | The cached action or `null` |
| `mixer.update(deltaSeconds)` | Advances every active action and writes the blended values |
| `mixer.setTime(seconds)` | Zeroes the mixer's and every action's time, then calls `update(seconds)` |
| `mixer.timeScale` | Global speed multiplier; 0 pauses everything |
| `mixer.stopAllAction()` | Stops every scheduled action |
| `mixer.uncacheClip(clip)`, `uncacheRoot(root)`, `uncacheAction(clip, root)` | Free cached actions and property bindings |
| events `"finished"`, `"loop"` | `e.action`, plus `e.direction` (finished) / `e.loopDelta` (loop) |

## AnimationAction

```javascript
const action = mixer.clipAction(clip);

action.play(); // schedule and start
action.stop(); // unschedule immediately
action.reset(); // time 0, enabled = true, paused = false, cancel fades and warps
action.isRunning(); // active in the mixer, started (startAt reached), enabled, not paused, timeScale != 0
action.isScheduled(); // play() called and not stopped

action.time = 0.5; // scrub within the clip
action.timeScale = -1; // negative plays backward
action.paused = true;
action.startAt(mixer.time + 1); // delay the start by one second of mixer time
action.setDuration(1.5); // set timeScale so one cycle takes 1.5 s
action.syncWith(otherAction); // copy time and time scale (keep cycles aligned)

action.setLoop(THREE.LoopOnce, 1);
action.setLoop(THREE.LoopRepeat, Infinity); // default
action.setLoop(THREE.LoopPingPong, 4); // alternate direction each pass
action.clampWhenFinished = true; // hold the last frame when a finite loop ends

action.zeroSlopeAtStart = true; // default: smooth start/end for InterpolateSmooth tracks
action.zeroSlopeAtEnd = true;
```

`stop()` restores, for every property no other scheduled action still animates, the value it had before the mixer first bound it.

## Fades, Warps and Halts

```javascript
walk.reset().fadeIn(0.4).play(); // weight multiplier ramps 0 → 1
idle.fadeOut(0.4); // ramps 1 → 0, then the action is disabled
idle.crossFadeTo(walk, 0.4, false); // the two lines above in one call; true also warps time scales

action.warp(1, 0.2, 0.5); // ramp the time scale from 1 to 0.2 over 0.5 s
action.halt(0.3); // decelerate to a stop over 0.3 s (warp to 0)
action.stopFading();
action.stopWarping();
```

Fades multiply `action.weight`; they never raise it above what `setEffectiveWeight` set. An action whose fade-out completed has `enabled = false` until `reset()` or `enabled = true`.

## Weight Blending (Blend Trees)

```javascript
const idle = mixer.clipAction(idleClip).play();
const walk = mixer.clipAction(walkClip).play();
const run = mixer.clipAction(runClip).play();

// 1D blend by speed: all three sample every frame, the weights mix their poses
function setLocomotion(speed) {
  const w = (a, b) => THREE.MathUtils.clamp((speed - a) / (b - a), 0, 1);
  const toWalk = w(0, 2); // 0 → 2 m/s: idle → walk
  const toRun = w(2, 5); // 2 → 5 m/s: walk → run
  idle.setEffectiveWeight(1 - toWalk);
  walk.setEffectiveWeight(toWalk * (1 - toRun));
  run.setEffectiveWeight(toRun);
}
```

- Weights of actions animating the same property are normalised when they sum to more than 1; when they sum to less, the remainder blends toward the property's original value.
- Walk and run cycles of different lengths drift apart; keep them in phase with `run.syncWith(walk)` plus `setDuration`, or scale `timeScale` by the speed ratio.
- Smooth `speed` before blending (`threejs-math-motion`), or weights flicker with noisy input.

## Additive Layers

```javascript
// Re-base every track on a reference pose and mark the clip additive
THREE.AnimationUtils.makeClipAdditive(breathClip); // (targetClip, referenceFrame = 0, referenceClip = targetClip, fps = 30)

mixer.clipAction(walkClip).play(); // normal base layer
const breath = mixer.clipAction(breathClip).play(); // adds its offsets on top of the base pose
breath.setEffectiveWeight(0.5); // half-strength layer
```

- `makeClipAdditive` modifies the clip in place and sets `clip.blendMode = THREE.AdditiveAnimationBlendMode`; clone first if you also need the normal version.
- Typical use: a pose-offset clip (aim, lean, breathing) authored relative to the base clip's first frame. Pass `referenceClip` and `referenceFrame` to subtract a pose from another clip instead.
- r187 fixed the reference sampling for glTF cubic-spline tracks when the reference frame falls between keys.

## Releasing a Mixer

```javascript
mixer.stopAllAction();
mixer.uncacheRoot(model); // releases actions and bindings created for this root
for (const clip of clips) mixer.uncacheClip(clip); // when the clips themselves are discarded
```

An un-updated mixer costs nothing per frame; uncaching matters only when models are created and destroyed repeatedly.
