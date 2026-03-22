# Interaction Patterns Reference

## Table of Contents
- [Spring Physics (Elastic Cursor Followers)](#spring-physics-elastic-cursor-followers)
- [Spotlight Cursor (Clip-Path Circle)](#spotlight-cursor-clip-path-circle)
- [Circuit Glow (CSS Custom Properties)](#circuit-glow-css-custom-properties)

---

## Spring Physics (Elastic Cursor Followers)

Multiple blobs chase the cursor with different durations for a staggered, physics-like feel. Rest positions are **percentage-based** relative to the container.

```js
const blobs = [
  { x: 15, y: 40, size: 90,  dur: 0.9, colorStop: 'rgba(20,184,166,0.4)' },
  { x: 30, y: 55, size: 55,  dur: 0.6, colorStop: 'rgba(var(--glow-secondary-rgb),0.35)' },
  { x: 55, y: 30, size: 120, dur: 1.1, colorStop: 'rgba(var(--glow-primary-rgb),0.3)' },
  { x: 68, y: 65, size: 60,  dur: 0.7, colorStop: 'rgba(6,182,212,0.35)' },
  { x: 80, y: 25, size: 75,  dur: 0.8, colorStop: 'rgba(20,184,166,0.35)' },
]
```

### Initial placement — percentage of container

```js
const setupVisuals = () => {
  const cw = container.offsetWidth
  const ch = container.offsetHeight
  blobRefs.value.forEach((blob, i) => {
    if (!blob) return
    const b = blobs[i]
    blob.style.width = `${b.size}px`
    blob.style.height = `${b.size}px`
    blob.style.background = `radial-gradient(circle, ${b.colorStop}, transparent)`
    blob.style.filter = `blur(${Math.round(b.size / 4)}px)`
    gsap.set(blob, { xPercent: -50, yPercent: -50, x: (cw * b.x) / 100, y: (ch * b.y) / 100 })
  })
}
```

### moveBlobs + resetBlobs

```js
onMounted(() => {
  ctx = gsap.context((self) => {
    setupVisuals()

    self.add('moveBlobs', (mx, my) => {
      blobRefs.value.forEach((blob, i) => {
        if (!blob) return
        gsap.to(blob, {
          x: mx, y: my,
          duration: blobs[i].dur,
          ease: 'elastic.out(1.2, 0.3)',
          overwrite: 'auto', force3D: true,
        })
      })
    })

    self.add('resetBlobs', () => {
      const cw = container.offsetWidth
      const ch = container.offsetHeight
      blobRefs.value.forEach((blob, i) => {
        if (!blob) return
        const b = blobs[i]
        gsap.to(blob, {
          x: (cw * b.x) / 100, y: (ch * b.y) / 100,
          duration: 1.6, ease: 'elastic.out(1, 0.4)', force3D: true,
        })
      })
    })
  }, containerRef.value?.closest('section'))
})
```

**Key rules**: rest positions are **percentage-based** (`cw * b.x / 100`) | on mousemove blobs go to **exact** cursor position | unique `duration` per blob creates stagger | `elastic.out(1.2, 0.3)` for overshoot on move, softer on reset

---

## Spotlight Cursor (Clip-Path Circle)

Reveal a hidden layer by moving a `clipPath` circle to follow the cursor.

```js
onMounted(() => {
  ctx = gsap.context((self) => {
    const spotlight = spotlightRef.value
    gsap.set(spotlight, { clipPath: 'circle(0px at 50% 50%)' })

    self.add('moveSpotlight', (x, y) => {
      gsap.to(spotlight, {
        clipPath: `circle(130px at ${x}px ${y}px)`,
        duration: 0.25, ease: 'power2.out', overwrite: 'auto',
      })
    })

    self.add('hideSpotlight', () => {
      gsap.to(spotlight, {
        clipPath: 'circle(0px at 50% 50%)',
        duration: 0.4, ease: 'power2.inOut', overwrite: 'auto',
      })
    })
  }, scopeRef.value)
})
```

**Key rules**: `overwrite: 'auto'` prevents pile-up | initial state via `gsap.set` for clean revert | short duration (0.25s) for responsive feel

---

## Circuit Glow (CSS Custom Properties)

Pipe mouse position directly to CSS custom properties without GSAP tweening. Uses a **real div element** with `mask-image` to reveal a glow layer.

### Template

```vue
<div class="relative w-full" @mousemove="onMouseMove">
  <div ref="circuitWrap" class="relative z-0" />
  <div ref="glowRef" class="circuit-glow absolute inset-0 z-[1]" style="--glow-x: 0.5; --glow-y: 0.5" />
</div>
```

### JS — unitless ratios (0-1), not pixels

```js
const onMouseMove = (e) => {
  if (!glowRef.value) return
  const rect = e.currentTarget.getBoundingClientRect()
  glowRef.value.style.setProperty('--glow-x', (e.clientX - rect.left) / rect.width)
  glowRef.value.style.setProperty('--glow-y', (e.clientY - rect.top) / rect.height)
}
```

### CSS

```css
.circuit-glow {
  background-image: url('~/assets/images/circuit-board.svg');
  @apply bg-cover bg-center bg-no-repeat opacity-0;
  filter: invert(1) sepia(1) saturate(5) hue-rotate(130deg) brightness(0.8);
  mask-image: radial-gradient(
    circle 350px at calc(var(--glow-x) * 100%) calc(var(--glow-y) * 100%),
    black 0%, transparent 100%
  );
  -webkit-mask-image: radial-gradient(
    circle 350px at calc(var(--glow-x) * 100%) calc(var(--glow-y) * 100%),
    black 0%, transparent 100%
  );
}
div:hover .circuit-glow {
  @apply opacity-35 transition-opacity duration-500;
}
```

**Key rules**: real `<div>` (not `::after`) | stores **unitless ratios** (0-1), CSS uses `calc(var * 100%)` | opacity is CSS-transition-based, no GSAP | direct `style.setProperty` for maximum speed
