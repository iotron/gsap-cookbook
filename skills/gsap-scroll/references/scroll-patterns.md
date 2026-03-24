# ScrollTrigger Advanced Patterns

## Table of Contents
- [Stacking Cards (Sticky + Scrub)](#stacking-cards-sticky--scrub)
- [Scrubbed Timeline (Progress Mapping)](#scrubbed-timeline-progress-mapping)
- [Elastic Type (Pin + Scrub Assembly)](#elastic-type-pin--scrub-assembly)
- [Service Section Switching](#service-section-switching)

---

## Stacking Cards (Sticky + Scrub)

CSS sticky + GSAP scrubbed two-phase timeline: fade/scale in, then stack/scale out.

```css
.stacking-card {
  position: relative;
  transform-origin: center top;
}
@media (min-width: 1024px) {
  .stacking-card {
    position: sticky;
    top: calc(5rem + var(--card-index, 0) * 0.5rem);
  }
}
```

```js
cards.forEach((card, index) => {
  gsap.set(card, { y: 100, autoAlpha: 0, scale: 0.9 })

  const tl = gsap.timeline({
    scrollTrigger: {
      trigger: card, start: 'top 85%', end: 'top 25%', scrub: 0.5,
      invalidateOnRefresh: true,
      onEnter:     () => gsap.set(card, { willChange: 'transform, opacity' }),
      onLeave:     () => gsap.set(card, { willChange: 'auto' }),
      onEnterBack: () => gsap.set(card, { willChange: 'transform, opacity' }),
      onLeaveBack: () => gsap.set(card, { willChange: 'auto' }),
    },
  })

  // Phase 1: reveal
  tl.to(card, { y: 0, autoAlpha: 1, scale: 1, duration: 0.5, ease: 'power2.out', force3D: true })
  // Phase 2: stack with depth
  .to(card, { y: index * -15, scale: 1 - index * 0.015, duration: 0.5, force3D: true })
})
```

- `will-change` set in onEnter, released in onLeave — avoids permanent GPU cost
- `--card-index` CSS variable offsets sticky `top` so cards stack with slight gaps
- Two-phase timeline: first tween reveals, second creates stacked depth

---

## Scrubbed Timeline (Progress Mapping)

`scrub: 1` maps scroll to timeline progress. Progress bar, bouncing dots, alternating cards.

```js
// Alternating initial offsets
timelineCardRefs.value.forEach((card, i) => {
  if (card) gsap.set(card, { x: i % 2 === 0 ? -50 : 50, autoAlpha: 0, force3D: true })
})

const tl = gsap.timeline({
  scrollTrigger: { trigger: timelineRef.value, start: 'top 70%', end: 'bottom 30%', scrub: 1, invalidateOnRefresh: true },
})

// Progress bar: scaleY 0->1, linear
tl.to(timelineProgressRef.value, { scaleY: 1, ease: 'none', duration: 1 })

// Dots + cards in sequence
milestones.forEach((_, i) => {
  const offset = i * 0.14
  if (timelineDotRefs.value[i])
    tl.to(timelineDotRefs.value[i], { scale: 1, autoAlpha: 1, duration: 0.08, ease: 'back.out(3)', force3D: true }, offset)
  if (timelineCardRefs.value[i])
    tl.to(timelineCardRefs.value[i], { x: 0, autoAlpha: 1, duration: 0.14, ease: 'power2.out', force3D: true }, offset + 0.04)
})
```

- `scrub: 1` — 1s smooth catch-up (good for complex timelines)
- Position offsets (`i * 0.14`) space milestones across the timeline

---

## Elastic Type (Pin + Scrub Assembly)

Pin section, scrub three phases: assemble from random, hold, scatter outward.

```js
gsap.set(chars, {
  y: () => gsap.utils.random(-200, 200), x: () => gsap.utils.random(-100, 100),
  rotation: () => gsap.utils.random(-90, 90), scale: () => gsap.utils.random(0.4, 2.5),
  autoAlpha: 0,
})

const tl = gsap.timeline({
  scrollTrigger: { trigger: elasticPinRef.value, start: 'top top', end: '+=150%', pin: true, scrub: 1, invalidateOnRefresh: true },
})

// Phase 1: assemble
tl.to(chars, {
  y: 0, x: 0, rotation: 0, scale: 1, autoAlpha: 1,
  stagger: { each: 0.03, from: 'random' }, duration: 1, ease: 'elastic.out(1.2, 1)', force3D: true,
})
// Phase 2: hold
tl.to({}, { duration: 0.3 })
// Phase 3: scatter from edges
tl.to(chars, {
  y: () => gsap.utils.random(-300, 300), x: () => gsap.utils.random(-200, 200),
  rotation: () => gsap.utils.random(-120, 120), scale: 0, autoAlpha: 0,
  stagger: { each: 0.02, from: 'edges' }, duration: 0.8, ease: 'power2.in',
})
```

- `pin: true` — element stays fixed while scroll drives timeline
- `end: '+=150%'` — scroll distance = 150% of viewport height
- Empty tween `tl.to({}, { duration: 0.3 })` creates a hold phase

---

## Service Section Switching

`ScrollTrigger.create()` per section with callbacks — no tween attached, purely event-driven switching.

```js
const activeIdx = ref(-1)
let revealed = false

// Gate: reveal sidebar + first service when section enters
ScrollTrigger.create({
  trigger: servicesCenter.value, start: 'top 70%', fastScrollEnd: true,
  onEnter: () => { if (!revealed) { revealed = true; switchTo(0) } },
  onLeaveBack: () => { revealed = false; deactivate(activeIdx.value); activeIdx.value = -1 },
})

// Per-category triggers
services.forEach((_, index) => {
  ScrollTrigger.create({
    trigger: categoryRefs.value[index], start: 'top 55%', end: 'bottom 45%', fastScrollEnd: true,
    onEnter:     () => { if (revealed && index > 0) switchTo(index) },
    onLeave:     () => { if (revealed && index < services.length - 1) switchTo(index + 1) },
    onEnterBack: () => { if (revealed) switchTo(index) },
    onLeaveBack: () => { if (revealed && index > 0) switchTo(index - 1) },
  })
})

function switchTo(index) {
  if (activeIdx.value === index) return
  if (activeIdx.value >= 0) deactivate(activeIdx.value)
  activeIdx.value = index
  activate(index)
}

function deactivate(i) {
  gsap.to(bodyRefs.value[i], { autoAlpha: 0, y: -10, duration: 0.3, overwrite: true })
  gsap.to(illustrationRefs.value[i], { autoAlpha: 0, scale: 0.95, duration: 0.3, overwrite: true })
}

function activate(i) {
  gsap.fromTo(bodyRefs.value[i], { autoAlpha: 0, y: 20 },
    { autoAlpha: 1, y: 0, duration: 0.5, ease: 'power2.out', overwrite: true })
  gsap.fromTo(illustrationRefs.value[i], { autoAlpha: 0, scale: 0.9 },
    { autoAlpha: 1, scale: 1, duration: 0.5, ease: 'back.out(1.4)', overwrite: true, force3D: true })
  pointRefs.value[i]?.forEach((li, j) => {
    if (li) gsap.fromTo(li, { autoAlpha: 0, x: -30 },
      { autoAlpha: 1, x: 0, duration: 0.3, delay: 0.15 + j * 0.08, ease: 'power2.out', overwrite: true })
  })
}
```

- `ScrollTrigger.create()` (no tween) — purely callback-driven
- `fastScrollEnd: true` — snaps to correct state on fast scroll
- `overwrite: true` kills in-flight tweens; `fromTo` ensures correct start state
- `revealed` gate prevents activation before section enters viewport

---

## Velocity Skew on Scroll

Applies dynamic skewY transforms to elements based on scroll velocity. Uses `gsap.quickSetter` for per-frame performance.

**Source**: [CodePen eYpGLYL](https://codepen.io/GreenSock/pen/eYpGLYL)

```html
<img src="image-1.jpg" alt="" class="skewElem">
<img src="image-2.jpg" alt="" class="skewElem">
<img src="image-3.jpg" alt="" class="skewElem">
<img src="image-4.jpg" alt="" class="skewElem">
<img src="image-5.jpg" alt="" class="skewElem">
```

```css
body {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  padding-top: 20vh;
}
body > img {
  margin-bottom: 20vh;
}
img {
  object-fit: cover;
  width: 300px;
  max-width: 80vw;
  aspect-ratio: 1/1;
}
```

```js
let proxy = { skew: 0 },
    skewSetter = gsap.quickSetter(".skewElem", "skewY", "deg"), // fast — bypasses .set() overhead
    clamp = gsap.utils.clamp(-20, 20); // cap skew at ±20 degrees

ScrollTrigger.create({
  onUpdate: (self) => {
    let skew = clamp(self.getVelocity() / -300);
    // Only update if new skew is MORE severe — avoids interrupting smooth tween-back-to-zero
    if (Math.abs(skew) > Math.abs(proxy.skew)) {
      proxy.skew = skew;
      gsap.to(proxy, {
        skew: 0, duration: 0.8, ease: "power3",
        overwrite: true,
        onUpdate: () => skewSetter(proxy.skew)
      });
    }
  }
});

// Right edge anchored to scroll bar side; force3D for GPU compositing
gsap.set(".skewElem", { transformOrigin: "right center", force3D: true });
```

**Key patterns**:
- `gsap.quickSetter()` — fastest way to set a single property per frame (no tween overhead)
- `ScrollTrigger.getVelocity()` — returns scroll speed in px/s, sign indicates direction
- `gsap.utils.clamp()` — prevents extreme values on fast flicks
- Proxy object pattern — tween a plain object, apply via quickSetter in `onUpdate`
- `overwrite: true` — kills previous tween when velocity changes direction

---

## Infinite Looped Panels

Layered pinning with infinite scroll looping — panels stack on top of each other and loop seamlessly.

**Source**: [CodePen VwbywPd](https://codepen.io/GreenSock/pen/VwbywPd)

```html
<div class="description panel">
  <h1>Layered pinning with infinite looping</h1>
  <p>Use pinning to layer panels on top of each other as you scroll.</p>
</div>

<section class="panel green">
  <h2 class="panel__number">1</h2>
</section>
<section class="panel">
  <h2 class="panel__number">2</h2>
</section>
<section class="panel purple">
  <h2 class="panel__number">3</h2>
</section>
<section class="panel">
  <h2 class="panel__number">4</h2>
</section>
<section class="panel blue">
  <h2 class="panel__number">5</h2>
</section>
```

```js
gsap.registerPlugin(ScrollTrigger);

let panels = gsap.utils.toArray(".panel"),
    copy = panels[0].cloneNode(true);
panels[0].parentNode.appendChild(copy); // Clone first panel to end for seamless looping

// Pin each panel with no spacing — they stack visually
panels.forEach((panel, i) => {
  ScrollTrigger.create({
    trigger: panel,
    start: "top top",
    pin: true,
    pinSpacing: false  // Panels overlap instead of pushing content down
  });
});

let maxScroll;
let pageScrollTrigger = ScrollTrigger.create({
  // Custom snap function handles edge cases at scroll boundaries
  snap(value) {
    let snappedValue = gsap.utils.snap(1 / panels.length, value);
    if (snappedValue <= 0) {
      // Don't snap to exactly 0 — keep 1px+ from top to prevent wrap
      return 1.05 / maxScroll;
    } else if (snappedValue >= 1) {
      // Don't snap to exactly end — keep 1px+ from bottom to prevent wrap
      return maxScroll / (maxScroll + 1.05);
    }
    return snappedValue;
  }
});

function onResize() {
  maxScroll = ScrollTrigger.maxScroll(window) - 1;
}
onResize();
window.addEventListener("resize", onResize);

// Non-passive listener to preventDefault at scroll boundaries for looping
window.addEventListener("scroll", e => {
  let scroll = pageScrollTrigger.scroll();
  if (scroll > maxScroll) {
    pageScrollTrigger.scroll(1);
    e.preventDefault();
  } else if (scroll < 1) {
    pageScrollTrigger.scroll(maxScroll - 1);
    e.preventDefault();
  }
}, { passive: false });
```

**Key patterns**:
- `pinSpacing: false` — panels overlap/stack instead of creating scroll gaps
- `cloneNode(true)` on first panel appended to end — creates seamless loop illusion
- Custom `snap()` function — prevents snapping to exact 0 or 1 which would break the loop
- `pageScrollTrigger.scroll()` — programmatically jump scroll position at boundaries
- `{ passive: false }` — required for `preventDefault()` on scroll events
- `ScrollTrigger.maxScroll()` — utility to get max scrollable distance

---

## Directionally Aware Header (Show/Hide on Scroll Direction)

Header slides in when scrolling up, hides when scrolling down. Extremely common UI pattern.

**Source**: [CodePen qBawMGb](https://codepen.io/GreenSock/pen/qBawMGb)

```html
<div class="main-tool-bar">Header</div>
<div class="scrollable-area"></div>
```

```css
.main-tool-bar {
  height: 80px;
  background: linear-gradient(144deg, #00bae2 4.56%, #fec5fb 72.98%);
  color: #0e100f;
  text-align: center;
  display: flex;
  align-items: center;
  justify-content: center;
  position: fixed;
  width: 100%;
  left: 0;
  top: 0;
}
.scrollable-area {
  height: 200vh;
}
```

```js
// Create the hide animation paused, then set progress to 1 (visible state)
const showAnim = gsap.from('.main-tool-bar', {
  yPercent: -100,
  paused: true,
  duration: 0.2
}).progress(1);

ScrollTrigger.create({
  start: "top top",
  end: "max",
  onUpdate: (self) => {
    // direction: -1 = scrolling up, 1 = scrolling down
    self.direction === -1 ? showAnim.play() : showAnim.reverse();
  }
});
```

**Key patterns**:
- `gsap.from().progress(1)` — creates animation in hidden state but immediately shows it (progress=1 means "at the end" = visible)
- `paused: true` — animation only plays/reverses when explicitly called
- `self.direction` — `-1` for scrolling up, `1` for scrolling down
- `play()` / `reverse()` — smooth toggle without creating new tweens each frame
- `end: "max"` — ScrollTrigger stays active for entire page scroll
- No `trigger` element needed — uses the whole page as the scroll context
