# GLSL hashes and noise bases

Purpose: the full GLSL for hashes, value noise, gradient noise, 3D value noise and Worley noise, with the reasoning behind each choice. Open it when writing a noise basis by hand.

All snippets are GLSL ES 3.00 (what `ShaderMaterial` compiles to under WebGLRenderer). They build on each other in the order given.

## Integer hash (PCG)

An integer hash returns the same bits on every GPU, because GLSL ES 3.00 integer arithmetic is exact 32-bit. The function is the PCG-based hash that three.js also uses in `src/nodes/math/Hash.js` (same constants 747796405, 2891336453, 277803737 and shifts 28, 4, 22; that file credits shadertoy.com/view/XlGcRh and pcg-random.org).

```glsl
// Integer hash (PCG). Same result on every GPU.
uint pcg(uint v) {
  uint state = v * 747796405u + 2891336453u;
  uint word = ((state >> ((state >> 28u) + 4u)) ^ state) * 277803737u;
  return (word >> 22u) ^ word;
}

float hash21(vec2 p) {
  uvec2 q = uvec2(ivec2(floor(p))); // through ivec2: float-to-uint of a negative number is undefined
  return float(pcg(q.x + pcg(q.y)) >> 8u) * (1.0 / 16777216.0); // 24 bits: exactly representable, result in [0, 1)
}

float hash31(vec3 p) {
  uvec3 q = uvec3(ivec3(floor(p)));
  return float(pcg(q.x + pcg(q.y + pcg(q.z))) >> 8u) * (1.0 / 16777216.0);
}
```

- Multi-dimensional inputs are combined by nesting, `pcg(x + pcg(y))`.
- Dividing the full 32-bit result by 2^32 in float can round up to exactly 1.0 (a float has a 24-bit significand), which breaks `floor(r * count)` indexing. That is why the code keeps 24 bits.
- The TSL `hash(seed)` node (exported from `three/tsl`, seed converted to `uint`) does divide all 32 bits by 2^32, so on the GPU it can return 1.0 for the largest outputs; its JSDoc says `[0, 1]`. Clamp before using it as an index. (Read from the r187 source; not measured on a GPU.)
- Hash lattice coordinates (integers) with these. They floor their input, so a continuously changing value such as `time` repeats the same hash until it crosses an integer.

## Float hash for non-integer inputs

```glsl
// Sin-free float hash (Dave Hoskins, "Hash without Sine"). For non-integer inputs.
float hash12(vec2 p) {
  vec3 p3 = fract(vec3(p.xyx) * 0.1031);
  p3 += dot(p3, p3.yzx + 33.33);
  return fract((p3.x + p3.y) * p3.z);
}
```

It loses quality beyond inputs of a few thousand. `fract(sin(dot(p, k)) * 43758.5453)` is not used here: its result depends on the precision of the GPU's `sin`, so the pattern differs between devices, bands for large inputs and degrades on mediump mobile GPUs. The `hash2d`/`hash3d` helpers inside `three/addons/tsl/math/voronoiNoise.js` are of that sin type.

## Value noise

Value noise interpolates random values at lattice corners. The quintic fade `6t^5 - 15t^4 + 10t^3` has continuous first and second derivatives; the cubic `3t^2 - 2t^3` does not, and its creases show once the noise feeds normals.

```glsl
float valueNoise(vec2 p) { // [0, 1)
  vec2 i = floor(p);
  vec2 f = fract(p);
  vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);
  return mix(mix(hash21(i), hash21(i + vec2(1.0, 0.0)), u.x),
             mix(hash21(i + vec2(0.0, 1.0)), hash21(i + vec2(1.0, 1.0)), u.x), u.y);
}

float valueNoise3(vec3 p) { // signed, [-1, 1): a potential for curl3 or any signed 3D basis
  vec3 i = floor(p);
  vec3 f = fract(p);
  vec3 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);
  float n000 = hash31(i);
  float n100 = hash31(i + vec3(1.0, 0.0, 0.0));
  float n010 = hash31(i + vec3(0.0, 1.0, 0.0));
  float n110 = hash31(i + vec3(1.0, 1.0, 0.0));
  float n001 = hash31(i + vec3(0.0, 0.0, 1.0));
  float n101 = hash31(i + vec3(1.0, 0.0, 1.0));
  float n011 = hash31(i + vec3(0.0, 1.0, 1.0));
  float n111 = hash31(i + vec3(1.0, 1.0, 1.0));
  float v = mix(mix(mix(n000, n100, u.x), mix(n010, n110, u.x), u.y),
                mix(mix(n001, n101, u.x), mix(n011, n111, u.x), u.y), u.z);
  return v * 2.0 - 1.0;
}
```

Value noise in [0, 1) is not signed, so `abs()` on it folds nothing; map it to [-1, 1) first when a signed basis is needed.

## Gradient (Perlin) noise

`gradientNoise` and `grad2` are in the SKILL.md Quick Start. Gradient noise interpolates the dot product of a random unit gradient with the offset to each corner.

- Signed, about [-0.707, 0.707] in 2D (the bound is `sqrt(N) / 2` for N dimensions with unit gradients), and exactly 0 at every integer lattice point. Multiply by about 1.4 to get roughly [-1, 1].
- Gradients must be unit length (`grad2` is): random vectors of uneven length make some cells visibly stronger.
- Cost in 2D: 4 hashes, 4 dot products, 3 mixes.

## Simplex

Use a tested implementation instead of a hand port. On the CPU, the `SimplexNoise` addon (`references/tsl-and-cpu.md`). In TSL, `snoise` from `three/addons/tsl/math/curlNoise.js`. A GLSL 3D simplex (the Ashima Arts / Stefan Gustavson form) is about 40 lines; `examples/jsm/tsl/math/curlNoise.js` is a line-by-line TSL port of that structure and can be read as a template. Simplex evaluates 3 corners in 2D and 4 in 3D, against 4 and 8 for lattice gradient noise.

## Worley (cellular) noise

Worley noise returns distances to jittered feature points, one per cell:

```glsl
// returns vec2(F1, F2): distances to the nearest and second-nearest feature point
vec2 worley(vec2 p) {
  vec2 i = floor(p);
  vec2 f = fract(p);
  float f1 = 8.0, f2 = 8.0;
  for (int y = -1; y <= 1; y++) {
    for (int x = -1; x <= 1; x++) {
      vec2 g = vec2(float(x), float(y));
      vec2 o = vec2(hash21(i + g), hash21(i + g + vec2(37.0, 17.0))); // feature point inside the cell
      vec2 d = g + o - f;
      float dist = dot(d, d);
      if (dist < f1) { f2 = f1; f1 = dist; } else if (dist < f2) { f2 = dist; }
    }
  }
  return sqrt(vec2(f1, f2));
}
```

- Compare squared distances and take one `sqrt` at the end.
- The 3x3 neighborhood (3x3x3 = 27 cells in 3D) is the standard search, not an exact one: with full jitter a feature point two cells away can occasionally be nearer. The error is rare for F1 and more visible for F2. A 2x2 search is visibly wrong at full jitter.
- `F2 - F1` is zero on cell borders: cracks, scales, cell walls. `1.0 - F1` gives billowy blobs for clouds.
- The TSL equivalents are `mx_worley_noise_float` (F1, Euclidean) and `mx_worley_noise_vec2` (F1, F2 as squared distances); see `references/tsl-and-cpu.md`.
