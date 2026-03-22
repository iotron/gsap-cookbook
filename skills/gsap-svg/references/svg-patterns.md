# SVG Patterns Reference

## Table of Contents
- [Circuit Tree Pattern (Triple-Layer Animation)](#circuit-tree-pattern-triple-layer-animation)
- [CTAHud 5-Layer System](#ctahud-5-layer-system)

---

## Circuit Tree Pattern (Triple-Layer Animation)

Three synced 10s timelines via `.add()` on a master with `repeatRefresh: true`. All layers target the same `allEls` array. No stagger on fade or wave layers; only colorAnime uses stagger.

```js
const COLORS = '#02f4c8|#41bbf6|#2dd4bf|#34d399|#06b6d4|#0ea5e9'
const randomColorStr = `random([${COLORS.split('|').map(c => `"${c}"`).join(',')}])`

const allEls = [...circuitWrap.value.querySelectorAll('polygon, path, rect, line, circle, polyline, ellipse')]
allEls.forEach(el => { el.style.cssText = 'fill:transparent;stroke-linecap:round;stroke-linejoin:round' })

gsap.set(allEls, { stroke: randomColorStr, strokeWidth: '0.3rem', autoAlpha: 0 })

gsap.timeline({ repeat: -1, repeatRefresh: true })
  .add(fadeAnime(), 0)
  .add(waveAnime(), 0)
  .add(colorAnime(), 3)
```

`repeatRefresh: true` lives on the **master** timeline so `randomColorStr` re-rolls every cycle.

### Layer 1: fadeAnime — breathing envelope (10s)

1s fade-in, 8s breathing glow (0.3 to 0.6, yoyo x3), 1s fade-out.

```js
function fadeAnime() {
  const tl = gsap.timeline()
  tl.fromTo(allEls, { autoAlpha: 0 }, { autoAlpha: 0.3, duration: 1, ease: 'sine.in' })
    .to(allEls, { autoAlpha: 0.6, duration: 2, ease: 'sine.inOut', yoyo: true, repeat: 3 })
    .to(allEls, { autoAlpha: 0, duration: 1, ease: 'sine.out' })
  return tl
}
```

### Layer 2: waveAnime — DrawSVG 3-phase (10s)

Three sequential tweens: build (3s), peak (4s), collapse (3s). First tween also sets `stroke: randomColorStr`.

```js
function waveAnime() {
  const tl = gsap.timeline()
  tl.fromTo(allEls, { drawSVG: 0 }, { drawSVG: '40% 50%', stroke: randomColorStr, duration: 3, ease: 'sine.in' })
    .to(allEls, { drawSVG: '73% 80%', duration: 4, ease: 'power1.inOut' })
    .to(allEls, { drawSVG: '100% 100%', strokeWidth: '0.2rem', duration: 3, ease: 'sine.out' })
  return tl
}
```

### Layer 3: colorAnime — elastic fill sweep

Positioned at `3` on master. Uses `yoyo: true, repeat: 1` — fill sweeps in then back out.

```js
function colorAnime() {
  const tl = gsap.timeline({ yoyo: true, repeat: 1 })
  tl.to(allEls, {
    fill: randomColorStr, duration: 2,
    stagger: { each: 0.005, from: 'edges' }, ease: 'elastic',
  })
  return tl
}
```

---

## CTAHud 5-Layer System

Five concurrent layers on a HUD circuit board SVG. Reveal plays once; remaining four loop.

```js
const master = gsap.timeline()
master.add(revealTl, 0)       // once
master.add(colorTl, 1.5)      // looping layers start after reveal
master.add(waveTl, 1.5)
master.add(blinkTl, 1.5)
master.add(glowTl, 1.5)
```

**Layer 1 — Reveal**: DrawSVG from center, plays once.
```js
revealTl.from(allPaths, {
  drawSVG: '50% 50%', duration: 1.5, ease: 'power2.inOut',
  stagger: { each: 0.02, from: 'edges' },
})
```

**Layer 2 — Color cycling**: random teal/cyan fills with `repeatRefresh: true`, elastic easing, `from: 'edges'` stagger.

**Layer 3 — Wave**: sweeping DrawSVG 73-80% to 100%, 5s yoyo.

**Layer 4 — Blink**: circles `scale: 1.3`, `transformOrigin: '50% 50%'`, random stagger, yoyo.

**Layer 5 — Glow**: animate `feGaussianBlur` stdDeviation via `attr: { stdDeviation: 4 }`.

```html
<filter id="hud-glow"><feGaussianBlur class="blur-node" stdDeviation="10" /></filter>
```

```js
glowTl = gsap.timeline({ repeat: -1, yoyo: true })
glowTl.to('.blur-node', { attr: { stdDeviation: 4 }, duration: 1.5, ease: 'sine.inOut' })
```
