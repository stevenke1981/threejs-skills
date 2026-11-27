# Normals, soft shadows, AO and fractals

Purpose: shading a sphere-traced hit (gradient normals and their epsilon, soft shadows including the improved variant, ambient occlusion) and fractal distance estimators (Mandelbulb, folding fractals). Open it once `trace()` hits and the surface needs lighting or more detail.

Sources: Inigo Quilez, "Normals for an SDF" (iquilezles.org/articles/normalsSDF), "Soft shadows in raymarched SDFs" (/articles/rmshadows), "Interior SDFs" (/articles/interiordistance); the Mandelbulb distance estimator is the widely used escape-time form (White/Nylander power-8 formula with the running derivative); it was checked algebraically, not re-fetched from a source for this edit.

## Normals

The normal is the normalized gradient of `map`. The Quick Start's `calcNormal` samples the four corners of a tetrahedron, `k.xyy`, `k.yyx`, `k.yxy`, `k.xxx` with `k = vec2(1, -1)`: 4 `map` calls (Quilez's tetrahedron technique). Central differences cost 6:

```glsl
vec3 calcNormalCentral(vec3 p) {
  const vec2 e = vec2(0.0005, 0.0);
  return normalize(vec3(map(p + e.xyy) - map(p - e.xyy),
                        map(p + e.yxy) - map(p - e.yxy),
                        map(p + e.yyx) - map(p - e.yyx)));
}
```

- Forward differences (`map(p + e) - map(p)`, also 4 calls) are biased and visibly worse than the tetrahedron.
- Epsilon too small: float noise and speckled normals far from the origin, since float32 keeps about 7 significant digits: the differences get noisy once `|p|` is around 1e5 times `h`, and `p + h == p` near 1e7 times `h`. Too large: edges look rounded. For large scenes scale `h` with `t` like the hit threshold. Quilez uses `h = 0.0001` as a placeholder "to replace by an appropriate value".
- A `map` with many primitives inlines four (or six) copies of itself into the shader; Quilez's article shows a loop form `for (int i = 0; i < 4; i++)` that keeps the compiler from inlining it four times.
- Start secondary rays offset along the normal (`p + n * 0.002`), or they hit the surface they start on.

## Soft shadows

`softShadow` in the Quick Start is Quilez's form, with the step clamped: it returns 0 in shadow and 1 when lit. Along the ray toward the light (`rd` = unit vector to the light) it tracks the smallest ratio of distance-to-surface over distance travelled, `k * h / t`: a ray that passes close to an occluder early is darker.

- `k` sets the penumbra; larger is sharper. Quilez's comparison images use 8, 32 and 128; it behaves like the inverse of the light's angular size.
- The `clamp(h, 0.01, 0.25)` step bounds both the cost and how thin an occluder can be and still be found; widen it for large scenes. The unclamped original uses `t += h` and up to 256 steps.

Improved penumbra (Sebastian Aaltonen's estimate, as given in the same article), which reduces banding from sharp-cornered casters by estimating the closest point between two consecutive samples. `w` is the light size (about `1 / k`):

```glsl
float softShadowImproved(vec3 ro, vec3 rd, float tmin, float tmax, float w) {
  float res = 1.0;
  float ph = 1e20; // previous h
  float t = tmin;
  for (int i = 0; i < 64 && t < tmax; i++) {
    float h = map(ro + rd * t);
    if (h < 0.001) return 0.0;
    float y = h * h / (2.0 * ph);
    float d = sqrt(h * h - y * y);
    res = min(res, d / (w * max(0.0, t - y)));
    ph = h;
    t += h;
  }
  return res;
}
```

## Ambient occlusion

Sample the field a few times along the normal and compare it with the free-space distance:

```glsl
float ambientOcclusion(vec3 p, vec3 n) {
  float occ = 0.0, scale = 1.0;
  for (int i = 0; i < 5; i++) {
    float h = 0.01 + 0.12 * float(i) / 4.0;
    occ += (h - map(p + n * h)) * scale; // how much closer geometry is than free space would be
    scale *= 0.95;
  }
  return clamp(1.0 - 3.0 * occ, 0.0, 1.0);
}
```

Multiply ambient light by it, not direct light. AO needs correct interior distances and a field close to a true distance; over unions (bound inside), displaced or heavily relaxed fields it darkens too much.

## Fractals

The power-8 Mandelbulb distance estimator iterates `z -> z^8 + p` in spherical coordinates and tracks the running derivative `dr`:

```glsl
float sdMandelbulb(vec3 p) {
  vec3 z = p;
  float dr = 1.0;
  float r = length(z);
  for (int i = 0; i < 8; i++) {
    r = length(z);
    if (r > 2.0) break;
    float theta = acos(clamp(z.z / max(r, 1e-6), -1.0, 1.0));
    float phi = atan(z.y, z.x);
    dr = 8.0 * pow(r, 7.0) * dr + 1.0;
    float zr = pow(r, 8.0);
    z = zr * vec3(sin(theta * 8.0) * cos(phi * 8.0), sin(theta * 8.0) * sin(phi * 8.0), cos(theta * 8.0)) + p;
  }
  return 0.5 * log(max(r, 1e-6)) * r / dr;
}
```

- `0.5 * log(r) * r / dr` is the standard escape-time distance estimate; some shaders write it as `0.25 * log(m) * sqrt(m) / dr` with `m = r * r`, which is the same value.
- The guards matter: at the origin `z.z / r` is 0/0 and `log(0)` is -inf. One NaN pixel spreads through later blur or bloom passes as black blocks. `atan(y, x)` with both arguments zero is undefined in GLSL; drivers return a finite value, but do not depend on it.
- This is an estimate, not a guaranteed bound: march it with `t += d * 0.5` to `0.8` and a higher step cap. The iteration count (6–10) sets detail and cost.
- The power-8 bulb fits inside a sphere of radius about 1.2: intersect the ray with a bounding sphere (radius 1.25 is a safe choice) first and march only inside it.

Folding fractals (Menger sponge, Mandelbox, kaleidoscopic IFS) build detail by repeatedly folding space and scaling it: `p = abs(p)` mirrors into one octant, `p = p - c * round(p / c)` or a box fold `clamp(p, -1.0, 1.0) * 2.0 - p` folds into a cell, then `p = p * s + offset`. Track the accumulated scale and divide the final distance by it, exactly as with uniform scaling.
