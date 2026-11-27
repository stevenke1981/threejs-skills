# Three.js Skills for Claude Code

English · [繁體中文](README.zh-TW.md)

A fork of [CloudAI-X/threejs-skills](https://github.com/CloudAI-X/threejs-skills) that adds the math skills in `math-skills/`, a reference example and a Traditional Chinese guide.

Skill files that give Claude Code current, verified knowledge of three.js, so the code it writes runs on the release you actually install.

The skills target **three.js r187** (`three@0.187`; `threejs-math-atmosphere`, `threejs-math-motion` and `threejs-math-particles-fields` are still on r186 pending the same update), React Three Fiber 9.8 with drei 10.7, and gltf-transform 4.5. Rendering examples use `WebGLRenderer`, directly or through React Three Fiber; the WebGPU and TSL section in every skill covers `WebGPURenderer`.

## Why

Claude Code knows three.js from its training data, which mixes many releases. A lot changed between r160 and r186: `RGBELoader` became `HDRLoader`, `Clock` gave way to `Timer`, `PostProcessing` became `RenderPipeline`, WebGL 1 support is gone, and the three.js docs now tell LLMs to use `WebGPURenderer` with TSL for custom shaders and materials. These skills give Claude Code the current API, working examples, and a Common Mistakes list of the outdated patterns models still reach for.

## Installation

With the [skills](https://skills.sh) CLI:

```bash
npx skills add stevenke1981/threejs-skills

# also the math skills in math-skills/ (the CLI only searches skills/ without --full-depth)
npx skills add stevenke1981/threejs-skills --full-depth

# every skill, user-wide, for Claude Code, Codex and Antigravity CLI at once
npx skills add stevenke1981/threejs-skills --full-depth -g -y --copy -s '*' -a claude-code -a codex -a antigravity-cli
```

The last command writes to `~/.claude/skills`, `~/.agents/skills` (Codex) and `~/.gemini/config/skills` (Antigravity CLI). `--copy` avoids symlinks, which often need admin rights on Windows.

Or manually. Claude Code loads skills from `~/.claude/skills/` (all your projects) or `.claude/skills/` (one project):

```bash
git clone https://github.com/stevenke1981/threejs-skills.git

# all your projects
mkdir -p ~/.claude/skills && cp -r threejs-skills/skills/* ~/.claude/skills/
cp -r threejs-skills/math-skills/* ~/.claude/skills/

# or one project (run from the project root)
mkdir -p .claude/skills && cp -r threejs-skills/skills/* .claude/skills/
cp -r threejs-skills/math-skills/* .claude/skills/
```

## Skills Included

| Skill                      | Covers                                                                          |
| -------------------------- | ------------------------------------------------------------------------------- |
| **threejs-fundamentals**   | Scene, cameras, renderer setup, animation loop, object hierarchy, transforms   |
| **threejs-geometry**       | Built-in shapes, BufferGeometry, merging, instancing, BatchedMesh, points, lines |
| **threejs-materials**      | Material types, PBR, blending, stencil, recompiles, node materials              |
| **threejs-lighting**       | Light types and physical units, shadows, HDR environments, light probes         |
| **threejs-textures**       | Loading, color spaces, UV channels, texture types, render targets, memory       |
| **threejs-animation**      | Keyframes, the animation mixer, skeletal and morph animation, blending          |
| **threejs-loaders**        | glTF with Draco, meshopt and KTX2, other formats, HDR, progress, disposal       |
| **threejs-shaders**        | ShaderMaterial and RawShaderMaterial, shader chunks, TSL and node materials     |
| **threejs-postprocessing** | EffectComposer, built-in renderer effects, the WebGPU RenderPipeline            |
| **threejs-interaction**    | Raycasting, camera controls, transform gizmos, drag and box selection           |
| **threejs-performance**    | Measuring frame cost, draw calls, GPU memory, fill rate, compile stalls        |
| **threejs-react-three-fiber** | React Three Fiber 9 and a verified set of drei helpers, on WebGL or WebGPU |
| **threejs-asset-pipeline** | gltf-transform: Draco, meshopt, KTX2, WebP, and the matching loader setup      |

### Math Skills

The skills in `math-skills/` cover the math behind procedural modeling and film-style effects: noise, distance fields, generated meshes, simulated particles, camera motion and light in participating media. They follow the same layout and target the same release.

| Skill                                | Covers                                                                         |
| ------------------------------------ | ------------------------------------------------------------------------------ |
| **threejs-math-noise**               | Hashes, value, gradient, simplex and Worley noise, fBm, domain warping, curl noise |
| **threejs-math-sdf-raymarching**     | Signed distance functions, smooth booleans, sphere tracing, soft shadows, AO, fractals |
| **threejs-math-procedural-geometry** | Parametric surfaces, curve frames and sweeps, supershapes, L-systems, marching cubes |
| **threejs-math-particles-fields**    | Vector fields, integrators and stability, strange attractors, GPGPU and TSL compute |
| **threejs-math-motion**              | Easing, frame-rate-independent damping, springs, quaternions, camera rails, shake |
| **threejs-math-atmosphere**          | Rayleigh and Mie scattering, Beer–Lambert, volumetric fog and clouds, god rays, tone mapping |

## How It Works

Claude Code loads a skill when your request matches its description. When you ask it to:

- create a 3D scene → `threejs-fundamentals`
- add lights and shadows → `threejs-lighting`
- load a glTF model → `threejs-loaders`
- write a custom shader or screen effect → `threejs-shaders` and `threejs-postprocessing`
- make a slow scene faster → `threejs-performance`
- build the scene with React → `threejs-react-three-fiber`
- shrink a glTF before shipping it → `threejs-asset-pipeline`
- generate terrain, clouds or marble from noise → `threejs-math-noise`
- raymarch a distance-field scene or fractal → `threejs-math-sdf-raymarching`
- build a mesh from a formula, curve or L-system → `threejs-math-procedural-geometry`
- simulate a particle flow or strange attractor → `threejs-math-particles-fields`
- move a camera along a smooth cinematic path → `threejs-math-motion`
- add a sky, volumetric fog or light shafts → `threejs-math-atmosphere`

## Examples

[`examples/cinematic-valley/index.html`](examples/cinematic-valley/index.html) combines four math skills in one page: a ridged-fBm river valley, the Sky scattering model with exponential fog, and a camera flying a closed Catmull-Rom rail at constant speed with damped look-ahead and noise shake. Its comments are in Traditional Chinese and name the skill each part comes from. Serve the repo root over HTTP (`npx serve .`) and open `/examples/cinematic-valley/`.

## Skill File Structure

Each updated skill is a short `SKILL.md` plus topic files under `references/`:

```markdown
---
name: threejs-<topic>
description: "What the skill covers. Use when ... Not for X (use other-skill). 中文: ..."
---

# Three.js <Topic>

## Quick Start          a complete, runnable example
## <decision guidance>  what to choose and what is easy to get wrong
## Common Mistakes      outdated or wrong code → the r187 fix, with the reason
## References           which references/*.md file to open for what
## See Also             related skills
```

`references/*.md` hold the API tables, secondary recipes and the WebGPU/TSL detail. The three math skills still on r186 keep the earlier single-file layout.

## Verification

The skills were originally checked against the published `three@0.186.1` package (the r187 update re-checked import paths and exported names against `three@0.187.0`) and the official Migration Guide, and against `@react-three/fiber@9.8.2`, `@react-three/drei@10.7.9`, `@gltf-transform/cli@4.5.1` and `gltfjsx@6.5.3` where they apply:

- every three.js, React Three Fiber, drei and gltf-transform import path, and every `THREE.*` class used in an example, exists in its package
- every JavaScript block parses as an ES module
- shader chunk names exist, and no removed or renamed API appears in example code
- every gltf-transform command and flag exists in the CLI, and every React Three Fiber element maps to a three.js class
- every skill was reviewed against the library source, with rendering, timing and shader claims measured where possible

The math skills had two more checks:

- every example that creates its own renderer was run in Chrome on WebGL 2, and the WebGPU and TSL examples on the WebGL 2 fallback. Each one compiled and rendered without errors or warnings, and the frames were inspected
- CPU-side numbers (integrator stability, attractor extents, frame-rate independence of damping and springs, curve frames, marching-cubes radii) were measured in Node

If you find something that is wrong on the current release, please open an issue with the skill name, the line and the three.js version you use.

## License

MIT, see [LICENSE](LICENSE).

## Acknowledgments

- [Three.js](https://threejs.org/) - the 3D library these skills document
