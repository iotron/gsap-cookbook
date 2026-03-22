# Text Animation Patterns Reference

## Table of Contents
- [Combined Clip + Scramble (chainTextReveal)](#combined-clip--scramble-chaintextreveal)
- [Kinetic Character Split](#kinetic-character-split)
- [Elastic Type Assembly (Pin + Scrub)](#elastic-type-assembly-pin--scrub)

---

## Combined Clip + Scramble (chainTextReveal)

The signature production pattern: word slides up from mask AND scramble-decodes simultaneously.

Per word: slide-up tween + scramble tween at the same timeline position. Both MUST have `overwrite: false` (see learnings.md). The `chainTextReveal` function in `useReveal.js` handles this automatically.

```js
ctx = gsap.context(() => {
  const s = SplitText.create(el, { type: 'words', mask: 'words' })
  splits.push(s)
  gsap.set(s.words, { y: '100%' })
  gsap.set(el, { visibility: 'visible' })

  const CLIP  = { duration: 0.8, ease: 'power4.out', force3D: true }
  const SCRAM = { duration: 0.5, ease: 'none' }

  const tl = gsap.timeline({
    scrollTrigger: { trigger: el, start: 'top 80%', toggleActions: 'play none none reverse' },
  })

  s.words.forEach((w, i) => {
    const pos = i * 0.08
    tl.to(w, { y: '0%', overwrite: false, ...CLIP }, pos)
    tl.to(w, { overwrite: false, ...SCRAM,
      scrambleText: { text: w.textContent, chars: '01', speed: 0.5 },
    }, pos)
  })
}, scopeRef.value)
```

---

## Kinetic Character Split

Characters start at random positions and assemble into the final word.

```js
ctx = gsap.context(() => {
  const split = SplitText.create(el, { type: 'chars' })
  splits.push(split)

  gsap.set(split.chars, {
    y: () => gsap.utils.random(-200, 200), x: () => gsap.utils.random(-100, 100),
    rotation: () => gsap.utils.random(-90, 90), scale: () => gsap.utils.random(0.3, 2),
    autoAlpha: 0,
  })
  gsap.to(split.chars, {
    y: 0, x: 0, rotation: 0, scale: 1, autoAlpha: 1,
    duration: 1.4, ease: 'power4.out', force3D: true,
    stagger: { each: 0.04, from: 'random' },
    scrollTrigger: { trigger: el, start: 'top 80%', toggleActions: 'play none none reverse' },
  })
}, scopeRef.value)
```

---

## Elastic Type Assembly (Pin + Scrub)

Characters start scattered, assemble on scroll, hold in place, then scatter again.

```js
ctx = gsap.context(() => {
  const split = SplitText.create(el, { type: 'chars' })
  splits.push(split)

  gsap.set(split.chars, {
    y: () => gsap.utils.random(-300, 300), x: () => gsap.utils.random(-200, 200),
    rotation: () => gsap.utils.random(-180, 180), scale: 0, autoAlpha: 0,
  })

  const tl = gsap.timeline({
    scrollTrigger: { trigger: el, start: 'top center', end: '+=150%', pin: true, scrub: 1 },
  })

  // Phase 1: Assemble
  tl.to(split.chars, {
    y: 0, x: 0, rotation: 0, scale: 1, autoAlpha: 1, duration: 1,
    ease: 'elastic.out(1.2, 1)', stagger: { each: 0.03, from: 'random' },
  })
  // Phase 2: Hold
  tl.to({}, { duration: 0.5 })
  // Phase 3: Scatter
  tl.to(split.chars, {
    y: () => gsap.utils.random(-400, 400), x: () => gsap.utils.random(-300, 300),
    rotation: () => gsap.utils.random(-180, 180), scale: 0, autoAlpha: 0, duration: 1,
    ease: 'power2.in', stagger: { each: 0.02, from: 'edges' },
  })
}, scopeRef.value)
```
