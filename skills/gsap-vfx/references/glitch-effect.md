# Glitch Effect Reference

Full cyberpunk glitch system: 1-second burst every 4 seconds with chromatic aberration, text scramble, jitter, and noise bars.

## Table of Contents
- [Timeline structure](#timeline-structure)
- [RGB chromatic aberration](#rgb-chromatic-aberration)
- [Per-word scramble](#per-word-scramble)
- [Jitter (continuous onUpdate)](#jitter-continuous-onupdate)
- [Noise bars](#noise-bars)
- [GPU lifecycle](#gpu-lifecycle)
- [Badge hue-rotate flash](#badge-hue-rotate-flash)
- [CLS-safe wrapper + cleanup](#cls-safe-wrapper--cleanup)

---

## Timeline structure

```js
const tl = gsap.timeline({ repeat: -1, repeatDelay: 4, delay: 4 })
```

Auto-plays immediately. The `delay: 4` gives the page time to load before the first burst.

## RGB chromatic aberration

Two overlay layers clipped identically to the text, offset in opposite directions.

```js
tl.to(rL, { autoAlpha: 0.8, x: -4, y: -2, skewX: 2, duration: 0.05, ease: 'steps(1)' }, 0)
tl.to(bL, { autoAlpha: 0.7, x: 4, y: 1, skewX: -1.5, duration: 0.05, ease: 'steps(1)' }, 0)
```

`ease: 'steps(1)'` creates instant frame-snapping. Note asymmetric autoAlpha (0.8 vs 0.7) and opposing skewX.

## Per-word scramble

```js
words.forEach((w, i) => {
  tl.to(w, {
    duration: 0.8, ease: 'none', overwrite: false,
    scrambleText: { text: w.textContent, chars: CHARS, revealDelay: 0.15, speed: 0.5 },
  }, i * 0.05)
})
```

`overwrite: false` prevents scramble tweens from killing each other.

## Jitter (continuous onUpdate)

A single empty-target tween that calls `applyJitter` every frame for 0.72 seconds.

```js
const rnd = (min, max) => gsap.utils.random(min, max)
const clipRnd = (max) => `inset(${rnd(0, max)}% 0 ${rnd(0, max)}% 0)`

function applyJitter() {
  gsap.set(el, { x: rnd(-4, 4), y: rnd(-2, 2), skewX: rnd(-3, 3), clipPath: clipRnd(40) })
  gsap.set(rL, { x: rnd(-8, -2), clipPath: clipRnd(60) })
  gsap.set(bL, { x: rnd(2, 8), clipPath: clipRnd(60) })
}

tl.to({}, { duration: 0.72, onUpdate: applyJitter }, 0)
```

## Noise bars

4 horizontal bars — snap on then snap off using `steps(1)`.

```js
noiseBars.forEach((bar, i) => {
  tl.to(bar, { scaleX: 1, duration: 0.03, ease: 'steps(1)' }, i * 0.06)
  tl.to(bar, { scaleX: 0, duration: 0.03, ease: 'steps(1)' }, i * 0.06 + 0.12)
})
```

## GPU lifecycle

Promote at burst start (0s), release after full resolve (1s).

```js
const glitchTargets = [el, rL, bL]
function promoteGPU() { glitchTargets.forEach(t => { t.style.willChange = 'transform, opacity, clip-path' }) }
function releaseGPU() { glitchTargets.forEach(t => { t.style.willChange = 'auto' }) }

tl.call(promoteGPU, null, 0)
tl.call(releaseGPU, null, 1)
```

## Badge hue-rotate flash

```js
const RED_FILTER = 'hue-rotate(150deg) saturate(1.5) brightness(1.2)'
tl.to(badge, { filter: RED_FILTER, duration: 0.05 }, 0)
tl.to(badge, { filter: 'none', duration: 0.1 }, 0.9)
```

## CLS-safe wrapper + cleanup

```js
// Lock height to prevent layout shift
const wrapper = el.closest('.glitch-wrapper')
if (wrapper) wrapper.style.height = `${wrapper.offsetHeight}px`

function clearJitter() {
  gsap.set([el, rL, bL], { x: 0, y: 0, skewX: 0, clipPath: 'none' })
}

tl.call(clearJitter, null, 0.9)
tl.to(rL, { autoAlpha: 0, x: 0, y: 0, skewX: 0, clipPath: 'none', duration: 0.1, overwrite: 'auto' }, 0.9)
tl.to(bL, { autoAlpha: 0, x: 0, y: 0, skewX: 0, clipPath: 'none', duration: 0.1, overwrite: 'auto' }, 0.9)
```
