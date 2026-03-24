# VFX Patterns — Advanced Recipes

## 1. Directional Marquee (Observer Plugin)

> **Source**: [GSAP Official CodePen](https://codepen.io/GreenSock/pen/rNYqEYe)
> **Plugins**: Observer, horizontalLoop helper
> **Key pattern**: Horizontal infinite loop that reverses direction and adjusts speed based on scroll/swipe velocity

Unlike the basic marquee in SKILL.md (simple `gsap.to` with `repeat: -1`), this uses the `horizontalLoop` helper for seamless bidirectional looping and Observer to react to scroll direction.

### HTML Structure

```html
<div class="scrolling-text">
  <div class="rail">
    <h4>Animate Anything...</h4>
    <h4>Delivering silky-smooth performance</h4>
    <h4>so you can focus on the fun stuff.</h4>
  </div>
</div>
```

### CSS

```css
.scrolling-text {
  overflow: hidden;
  width: 100%;
  height: 100vh;
  display: flex;
  align-items: center;
}

.scrolling-text .rail {
  display: flex;
}

.scrolling-text .rail h4 {
  white-space: nowrap;
  font-size: 100px;
  font-weight: 900;
  line-height: 1em;
  margin: 0 30px 0 0;
}
```

### Script

```js
gsap.registerPlugin(Observer);

const scrollingText = gsap.utils.toArray('.rail h4');

// Create the infinite horizontal loop timeline
const tl = horizontalLoop(scrollingText, {
  repeat: -1,
  paddingRight: 30,
});

// Observer watches scroll direction and adjusts timeScale
Observer.create({
  onChangeY(self) {
    let factor = 2.5;
    if (self.deltaY < 0) {
      factor *= -1; // Reverse direction on scroll up
    }
    gsap.timeline({
      defaults: { ease: "none" }
    })
      .to(tl, { timeScale: factor * 2.5, duration: 0.2, overwrite: true })
      .to(tl, { timeScale: factor / 2.5, duration: 1 }, "+=0.3");
  }
});
```

### horizontalLoop Helper (GSAP Official)

This is the official GSAP `horizontalLoop` utility. It creates a seamless, responsive loop along the x-axis using `xPercent` so it handles window resizes.

```js
function horizontalLoop(items, config) {
  items = gsap.utils.toArray(items);
  config = config || {};
  let tl = gsap.timeline({
      repeat: config.repeat,
      paused: config.paused,
      defaults: { ease: "none" },
      onReverseComplete: () => tl.totalTime(tl.rawTime() + tl.duration() * 100)
    }),
    length = items.length,
    startX = items[0].offsetLeft,
    times = [],
    widths = [],
    xPercents = [],
    curIndex = 0,
    pixelsPerSecond = (config.speed || 1) * 100,
    snap = config.snap === false ? v => v : gsap.utils.snap(config.snap || 1),
    totalWidth, curX, distanceToStart, distanceToLoop, item, i;

  gsap.set(items, {
    xPercent: (i, el) => {
      let w = widths[i] = parseFloat(gsap.getProperty(el, "width", "px"));
      xPercents[i] = snap(
        parseFloat(gsap.getProperty(el, "x", "px")) / w * 100 +
        gsap.getProperty(el, "xPercent")
      );
      return xPercents[i];
    }
  });
  gsap.set(items, { x: 0 });

  totalWidth = items[length - 1].offsetLeft +
    xPercents[length - 1] / 100 * widths[length - 1] - startX +
    items[length - 1].offsetWidth * gsap.getProperty(items[length - 1], "scaleX") +
    (parseFloat(config.paddingRight) || 0);

  for (i = 0; i < length; i++) {
    item = items[i];
    curX = xPercents[i] / 100 * widths[i];
    distanceToStart = item.offsetLeft + curX - startX;
    distanceToLoop = distanceToStart + widths[i] * gsap.getProperty(item, "scaleX");
    tl.to(item, {
        xPercent: snap((curX - distanceToLoop) / widths[i] * 100),
        duration: distanceToLoop / pixelsPerSecond
      }, 0)
      .fromTo(item,
        { xPercent: snap((curX - distanceToLoop + totalWidth) / widths[i] * 100) },
        {
          xPercent: xPercents[i],
          duration: (curX - distanceToLoop + totalWidth - curX) / pixelsPerSecond,
          immediateRender: false
        },
        distanceToLoop / pixelsPerSecond
      )
      .add("label" + i, distanceToStart / pixelsPerSecond);
    times[i] = distanceToStart / pixelsPerSecond;
  }

  function toIndex(index, vars) {
    vars = vars || {};
    (Math.abs(index - curIndex) > length / 2) &&
      (index += index > curIndex ? -length : length);
    let newIndex = gsap.utils.wrap(0, length, index),
      time = times[newIndex];
    if (time > tl.time() !== index > curIndex) {
      vars.modifiers = { time: gsap.utils.wrap(0, tl.duration()) };
      time += tl.duration() * (index > curIndex ? 1 : -1);
    }
    curIndex = newIndex;
    vars.overwrite = true;
    return tl.tweenTo(time, vars);
  }

  tl.next = vars => toIndex(curIndex + 1, vars);
  tl.previous = vars => toIndex(curIndex - 1, vars);
  tl.current = () => curIndex;
  tl.toIndex = (index, vars) => toIndex(index, vars);
  tl.times = times;
  tl.progress(1, true).progress(0, true); // pre-render for performance
  if (config.reversed) {
    tl.vars.onReverseComplete();
    tl.reverse();
  }
  return tl;
}
```

### Key Patterns

- **`horizontalLoop`** uses `xPercent` instead of `x` for responsive behavior across resize
- **`Observer.onChangeY`** detects scroll direction without needing ScrollTrigger
- **`timeScale` animation** creates smooth acceleration/deceleration when direction changes — short `.to()` for the burst, longer `.to()` to settle back
- **`overwrite: true`** on the timeScale tween prevents competing tweens from stacking
- **`snap` rounding** prevents sub-pixel jitter in flex layouts
- **`onReverseComplete` trick** — `tl.totalTime(tl.rawTime() + tl.duration() * 100)` prevents the timeline from stopping when it reverses past time 0

### Vue 3 Adaptation Notes

```js
// In <script setup>
import { onMounted, onUnmounted, ref } from 'vue'

const railRef = ref(null)
let ctx

onMounted(() => {
  ctx = gsap.context(() => {
    const items = gsap.utils.toArray(railRef.value.querySelectorAll('h4'))
    const tl = horizontalLoop(items, { repeat: -1, paddingRight: 30 })

    Observer.create({
      onChangeY(self) {
        let factor = self.deltaY < 0 ? -2.5 : 2.5
        gsap.timeline({ defaults: { ease: 'none' } })
          .to(tl, { timeScale: factor * 2.5, duration: 0.2, overwrite: true })
          .to(tl, { timeScale: factor / 2.5, duration: 1 }, '+=0.3')
      }
    })
  }, railRef.value)
})

onUnmounted(() => ctx?.revert())
```

---

## 2. Footer Bounce (MorphSVG + ScrollTrigger Velocity)

> **Source**: [GSAP Official CodePen](https://codepen.io/GreenSock/pen/KKGjNqp)
> **Plugins**: ScrollTrigger, MorphSVGPlugin
> **Key pattern**: Footer reveal with elastic bounce whose intensity scales with scroll velocity

### HTML Structure

```html
<p>scroll down</p>
<div class="footer">
  <svg preserveAspectRatio="none" id="footer-img"
       xmlns="http://www.w3.org/2000/svg" viewBox="0 0 2278 683">
    <defs>
      <linearGradient id="grad-1" x1="0" y1="0" x2="2278" y2="683"
                      gradientUnits="userSpaceOnUse">
        <stop offset="0.2" stop-color="#fec5fb"></stop>
        <stop offset="0.8" stop-color="#00bae2"></stop>
      </linearGradient>
    </defs>
    <path class="footer-svg" id="bouncy-path" fill="url(#grad-1)"
          d="M0-0.3C0-0.3,464,156,1139,156S2278-0.3,2278-0.3V683H0V-0.3z"/>
  </svg>
</div>
```

### CSS

```css
html {
  overscroll-behavior: none;
}

body {
  height: 250vh;
  position: relative;
  margin: 0;
}

.footer {
  position: absolute;
  width: 100%;
  bottom: 0;
}

#footer-img {
  height: 100%;
  width: 100%;
  display: block;
  overflow: visible;
}
```

### Script

```js
gsap.registerPlugin(ScrollTrigger, MorphSVGPlugin);

// Two SVG path states: curved (down) and flat (center)
const down   = 'M0-0.3C0-0.3,464,156,1139,156S2278-0.3,2278-0.3V683H0V-0.3z';
const center = 'M0-0.3C0-0.3,464,0,1139,0s1139-0.3,1139-0.3V683H0V-0.3z';

ScrollTrigger.create({
  trigger: '.footer',
  start: 'top bottom',
  toggleActions: 'play pause resume reverse',
  onEnter: self => {
    const velocity = self.getVelocity();
    const variation = velocity / 10000;

    gsap.fromTo('#bouncy-path', {
      morphSVG: down           // Start from curved/bulging state
    }, {
      duration: 2,
      morphSVG: center,        // Morph to flat state
      ease: `elastic.out(${1 + variation}, ${1 - variation})`,
      overwrite: 'true'
    });
  }
});
```

### Key Patterns

- **Velocity-driven elastic ease**: `self.getVelocity()` from ScrollTrigger feeds into the `elastic.out` amplitude and frequency — faster scroll = more dramatic bounce
- **Two SVG path states**: `down` (curved/bulging) and `center` (flat) — the morph goes from exaggerated to resting
- **`overwrite: true`** prevents stacking if user scrolls in/out quickly
- **`preserveAspectRatio="none"`** on the SVG allows it to stretch full-width
- **`overflow: visible`** on the SVG prevents clipping during elastic overshoot
- **`overscroll-behavior: none`** prevents pull-to-refresh from interfering

### Vue 3 Adaptation Notes

```vue
<script setup>
import { onMounted, onUnmounted, ref } from 'vue'

const footerRef = ref(null)
let ctx

const down   = 'M0-0.3C0-0.3,464,156,1139,156S2278-0.3,2278-0.3V683H0V-0.3z'
const center = 'M0-0.3C0-0.3,464,0,1139,0s1139-0.3,1139-0.3V683H0V-0.3z'

onMounted(() => {
  ctx = gsap.context(() => {
    ScrollTrigger.create({
      trigger: footerRef.value,
      start: 'top bottom',
      toggleActions: 'play pause resume reverse',
      onEnter: self => {
        const variation = self.getVelocity() / 10000
        gsap.fromTo('#bouncy-path',
          { morphSVG: down },
          {
            duration: 2,
            morphSVG: center,
            ease: `elastic.out(${1 + variation}, ${1 - variation})`,
            overwrite: true
          }
        )
      }
    })
  }, footerRef.value)
})

onUnmounted(() => ctx?.revert())
</script>

<template>
  <div ref="footerRef" class="footer">
    <svg preserveAspectRatio="none" viewBox="0 0 2278 683">
      <defs>
        <linearGradient id="grad-1" x1="0" y1="0" x2="2278" y2="683"
                        gradientUnits="userSpaceOnUse">
          <stop offset="0.2" stop-color="#fec5fb" />
          <stop offset="0.8" stop-color="#00bae2" />
        </linearGradient>
      </defs>
      <path id="bouncy-path" fill="url(#grad-1)"
            d="M0-0.3C0-0.3,464,156,1139,156S2278-0.3,2278-0.3V683H0V-0.3z" />
    </svg>
  </div>
</template>
```
