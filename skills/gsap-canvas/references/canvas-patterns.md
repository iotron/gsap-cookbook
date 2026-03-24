# Canvas Patterns — Full Implementations

> Source: [GSAP Canvas Particles CodePen](https://codepen.io/GreenSock/pen/NWZRRNb)

---

## Particle Orbit System

A spiral particle system where GSAP animates plain objects and an `onUpdate` callback renders sprite images to canvas. Particles orbit inward with staggered timing, creating a continuous vortex effect.

### HTML

```html
<main>
  <canvas></canvas>
</main>
```

### CSS

```css
html, body {
  margin: 0;
  padding: 0;
}

main {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 100vh;
  background: rgb(14, 16, 15);
}
```

### JavaScript

```js
// ---------------------------------------------------------------------------
// 1. Canvas setup
// ---------------------------------------------------------------------------
// Get the canvas element and its 2D rendering context.
// Set canvas dimensions to fill the viewport.
const c = document.querySelector('canvas')
const ctx = c.getContext('2d')
let cw = (c.width = window.innerWidth)
let ch = (c.height = window.innerHeight)

// radius determines the starting distance of particles from center.
// Using the larger viewport dimension ensures particles start offscreen.
let radius = Math.max(cw, ch)

// ---------------------------------------------------------------------------
// 2. Particle array — plain JS objects, not DOM elements
// ---------------------------------------------------------------------------
// Each particle holds animated properties (x, y, scale, rotate) plus an
// Image object for its sprite. GSAP will tween the numeric properties;
// the draw function reads them every frame.
const particles = Array(99)

for (let i = 0; i < particles.length; i++) {
  particles[i] = {
    x: 0,
    y: 0,
    scale: 0,
    rotate: 0,
    img: new Image(),
  }
  // Cycle through 21 different flair sprite images (indices 2–22)
  particles[i].img.src =
    'https://assets.codepen.io/16327/flair-' + (2 + (i % 21)) + '.png'
}

// ---------------------------------------------------------------------------
// 3. Timeline with onUpdate draw loop
// ---------------------------------------------------------------------------
// The timeline animates the particle objects. The onUpdate callback fires
// every time GSAP updates any value — this is our "render loop."
//
// Key patterns:
//   - fromTo with function-based values: each particle gets a unique starting
//     position based on its index, using trigonometric functions to create
//     a spiral distribution around the center.
//   - The "from" x/y use cos/sin with angle*10 to create a multi-loop spiral
//     pattern (10 full rotations spread across all particles).
//   - Negative stagger (each: -0.05) means the last particle starts first,
//     creating an inward spiral motion.
//   - repeat: -1 on the stagger makes each particle's tween loop forever.
const tl = gsap
  .timeline({ onUpdate: draw })
  .fromTo(
    particles,
    {
      // Function-based "from" values — evaluated per particle
      x: (i) => {
        // Distribute particles in a spiral pattern using their index
        const angle = (i / particles.length) * Math.PI * 2 - Math.PI / 2
        // angle*10 creates 10 full spiral loops
        return Math.cos(angle * 10) * radius
      },
      y: (i) => {
        const angle = (i / particles.length) * Math.PI * 2 - Math.PI / 2
        return Math.sin(angle * 10) * radius
      },
      scale: 1.1,
      rotate: 0,
    },
    {
      duration: 5,
      ease: 'sine',
      x: 0,            // All particles converge to center
      y: 0,
      scale: 0,         // Shrink to nothing at center
      rotate: -3,       // ~170 degrees rotation during flight
      stagger: {
        each: -0.05,    // Negative = last particle starts first (inward spiral)
        repeat: -1,     // Each particle's individual tween repeats forever
      },
    },
    0
  )
  .seek(99) // Jump far ahead so the repeating staggers are mid-flight
            // on first render (avoids a blank start)

// ---------------------------------------------------------------------------
// 4. Draw function — the canvas rendering pipeline
// ---------------------------------------------------------------------------
// Called by GSAP on every frame via the timeline's onUpdate callback.
//
// Pattern:
//   1. Sort particles by scale for z-ordering (smaller = further away = drawn first)
//   2. Clear the entire canvas
//   3. For each particle: translate to center, apply rotation, draw the sprite
//   4. Reset transform after each particle (cheaper than save/restore)
function draw() {
  // Sort by scale so smaller (more distant) particles render behind larger ones
  particles.sort((a, b) => a.scale - b.scale)

  // Clear previous frame
  ctx.clearRect(0, 0, cw, ch)

  particles.forEach((p, i) => {
    // Move origin to canvas center — all particle positions are relative to center
    ctx.translate(cw / 2, ch / 2)

    // Apply particle rotation
    ctx.rotate(p.rotate)

    // Draw the sprite image at the particle's animated position and size.
    // p.x/p.y are offsets from center; width/height are scaled by p.scale.
    ctx.drawImage(
      p.img,
      p.x,
      p.y,
      p.img.width * p.scale,
      p.img.height * p.scale
    )

    // Reset the transform matrix — faster than ctx.save()/ctx.restore()
    ctx.resetTransform()
  })
}

// ---------------------------------------------------------------------------
// 5. Resize handler — recalculate dimensions and invalidate
// ---------------------------------------------------------------------------
// When the viewport resizes:
//   1. Update canvas dimensions (this also clears the canvas)
//   2. Recalculate the radius so particles start from the correct offscreen distance
//   3. Call tl.invalidate() — this forces GSAP to re-evaluate the function-based
//      "from" values (the trig calculations) using the new radius on the next cycle
window.addEventListener('resize', () => {
  cw = c.width = innerWidth
  ch = c.height = innerHeight
  radius = Math.max(cw, ch)
  tl.invalidate()
})

// ---------------------------------------------------------------------------
// 6. Click-to-toggle play/pause via timeScale
// ---------------------------------------------------------------------------
// Instead of tl.pause()/tl.play(), we tween the timeline's timeScale property.
// This creates a smooth ease into pause (timeScale → 0) or play (timeScale → 1)
// rather than an abrupt stop/start.
//
// tl.isActive() returns true when the timeline is playing — we toggle to 0.
// When paused (timeScale === 0, isActive() is false) — we toggle back to 1.
c.addEventListener('pointerup', () => {
  gsap.to(tl, {
    timeScale: tl.isActive() ? 0 : 1,
  })
})
```

---

## Key Patterns Summary

| Pattern | Detail |
|---|---|
| **Plain object animation** | GSAP tweens `{x, y, scale, rotate}` — never touches DOM/canvas directly |
| **onUpdate as render loop** | Timeline's `onUpdate` callback replaces manual `requestAnimationFrame` |
| **Function-based values** | `x: (i) => ...` gives each particle a unique starting position |
| **Trigonometric spiral** | `cos(angle*10)` / `sin(angle*10)` distributes particles in 10 spiral loops |
| **Negative stagger** | `each: -0.05` — last index starts first, creating inward motion |
| **seek(99)** | Jump ahead so infinite-repeating staggers are mid-animation on load |
| **Z-sort by scale** | `sort((a,b) => a.scale - b.scale)` — smaller particles drawn first (behind) |
| **resetTransform** | Faster than `save()`/`restore()` for per-particle transform resets |
| **invalidate on resize** | Re-evaluates function-based from-values with updated dimensions |
| **timeScale toggle** | `gsap.to(tl, { timeScale: 0 })` for smooth pause; `1` for smooth resume |
