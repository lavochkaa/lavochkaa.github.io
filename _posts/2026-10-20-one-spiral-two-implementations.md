---
title: "One spiral, two implementations: keeping CPU picking in sync with a GLSL shader"
date: 2026-10-20 00:05:00 +0300
categories: [Projects, Graphics]
tags: [threejs, webgl, glsl, shaders, visualization, github, bloom]
image:
  path: https://github.com/lavochkaa/githubGalaxy/raw/main/docs/overview.jpg
  alt: GitHub Galaxy — a GitHub account rendered as a spiral galaxy, one arm per repository
---

## What this is

[GitHub Galaxy](https://github.com/lavochkaa/githubGalaxy) turns a GitHub account into a spiral galaxy. Every repository is one arm, every commit is a star on that arm, and branches run as parallel strands. You can try the [live demo](https://lavochkaa.github.io/githubGalaxy/), which is a static snapshot of my own account.

It is a zero-dependency project: one Python file serves a read-only GitHub proxy, one WebGL page draws the scene with Three.js r186. The token never reaches the browser.

![Repository drill-down](https://github.com/lavochkaa/githubGalaxy/raw/main/docs/repository.jpg)
_Selecting a repository lights up its commit history along its own arm._

This post is about the one design decision that made the interaction feel right: **the arm geometry exists twice, once on the GPU and once on the CPU, and both must agree to the pixel.**

## The problem

The scene has tens of thousands of stars. They have to move every frame, so the GPU owns their positions: a vertex shader computes where each particle sits on its arm at time `t`. Nothing on the CPU knows where a given star is on screen.

But the CPU still has questions to answer:

- Which arm is under the mouse? (hover and click picking)
- Where does this repository's label go?
- Where does the camera fly when you select an arm?

The obvious fix is to read positions back from the GPU. That stalls the pipeline and is awkward with instanced particles. The other obvious fix is to raycast against geometry, but the stars are points whose positions only exist inside a shader.

## The solution: one function, two languages

Each arm is a logarithmic spiral. A point at radius `r` on arm `i` of `n` sits at angle:

```text
theta = i * 2π / n  +  cot(pitch) * ln(r / r_min)  +  pattern(t)  +  drift
```

All arms share one pitch (20°) and one phase origin, so they stay evenly interleaved. The `pattern(t)` term rotates the whole disc rigidly, one turn every 210 seconds, so arms never wind up. The y axis is squashed by an aspect factor of 0.78 to fake a tilted disc.

On the CPU (`galaxy_arm_layout.mjs`):

```js
const theta = arm * TAU / count
  + cotPitch * Math.log(adjustedRadius / GALAXY_ARMS.minRadius)
  + patternAngle(seconds) + drift;
target.set(Math.cos(theta) * adjustedRadius - Math.sin(theta) * lateral,
           (Math.sin(theta) * adjustedRadius + Math.cos(theta) * lateral) * GALAXY_ARMS.aspect, depth);
```

On the GPU (`galaxy_gpu.js`), the same thing in GLSL:

```glsl
vec3 armPointAt(float radius, float lateral) {
  float drift = armDrift(radius, aOrbit.w);
  float r = radius * exp(drift / aArm.z);
  float th = aArm.x * 6.2831853 / max(1., aArm.y)
           + aArm.z * log(r / aArm.w) + uPattern + drift;
  return vec3(cos(th) * r - sin(th) * lateral,
              (sin(th) * r + cos(th) * lateral) * ASPECT, aOrbit.z);
}
```

The shader gets `cot(pitch)` and the arm index and count as per-vertex attributes (`aArm`). Constants such as the aspect ratio are interpolated into the GLSL from the same JavaScript object that the CPU code reads, so there is a single source of truth for the numbers.

## Picking by inverting the function

For hover, the CPU never asks "where is each star". It inverts the arm function instead. Given a mouse position on the disc plane:

1. Undo the aspect squash and convert to polar `(r, angle)`.
2. For every arm, compute the angle it would have at that same radius `r`.
3. The arm with the smallest angular gap times `r`, which is arc length, wins.

That is a few multiplications per arm, so it runs on every pointer move without caring how many particles exist. Because the two implementations share one formula, the hit target is exactly where the stars are drawn.

## What went wrong along the way

- **Per-arm pitch variance made arms bunch up.** My first version gave each arm a slightly different pitch for a natural look. Spirals with different pitches drift toward each other and leave voids elsewhere. Now every arm has the same pitch, with only about ±0.15° of jitter.
- **Drift is bounded.** Stars breathe along the arm with a small time-varying offset, clamped to ±0.35 rad. That keeps them organic without moving them far enough to break picking, which uses the centerline.
- **Picking ignores drift on purpose.** It works against the idealised arm. This is the one place the two implementations are not identical, and it is a deliberate trade for simplicity.

## The grand-design density wave

Real spiral galaxies have broad bright bands across their arms. In the shader, each star's brightness is multiplied by a two-armed wave with a more open pitch than the repository arms:

```glsl
float grand(float radius) {
  float th = aArm.x * 6.2831853 / max(1., aArm.y)
           + aArm.z * log(radius / aArm.w) - 1.55 * log(radius / aArm.w);
  return pow(.5 + .5 * cos(2. * th), 2.6);
}
```

Because it is a different pitch, the bright bands cross the arms at an angle, which is exactly what sells the "grand design" look.

## The bloom pipeline

Glow is what makes it read as space rather than a scatter plot. The renderer draws into a half-float HDR target (with MSAA), then:

1. a soft-threshold extract at half resolution (threshold 0.42, knee 0.35),
2. a five-level down/up mip chain for the blur,
3. exponential tone mapping, `1 - exp(-c * exposure)`,
4. sRGB output with dithering to avoid banding in the dark gradients.

On slow GPUs the glow chain switches itself off when frame times climb, so the image stays tone-mapped but cheap. `?bloom=force` pins it on.

## Takeaways

- If the GPU owns positions, give the CPU the **same pure function** instead of reading data back.
- Share constants through one JavaScript object that also builds the GLSL string, so the two copies cannot drift apart.
- Keep the picking model simpler than the rendering model, and write down where they differ.

The source is MIT-licensed: [lavochkaa/githubGalaxy](https://github.com/lavochkaa/githubGalaxy). Feedback on how the picking and the bloom read on your display is very welcome.
