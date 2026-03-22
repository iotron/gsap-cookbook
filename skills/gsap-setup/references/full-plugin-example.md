# Full Plugin Examples

## Table of Contents
- [Nuxt 3 Plugin with Lazy Loading](#nuxt-3-plugin-with-lazy-loading)

---

## Nuxt 3 Plugin with Lazy Loading

Complete plugin with registered effects, lazy loaders, and ScrollTrigger sort.

```js
// plugins/gsap.js
import { gsap } from 'gsap'
import ScrollTrigger from 'gsap/ScrollTrigger'
import TextPlugin from 'gsap/TextPlugin'

/** Lazy-loader factory: import once, register once, cache forever. */
function createLazyLoader(importFn, exportKey = 'default') {
  let cached = null
  return async () => {
    if (!cached) {
      const mod = await importFn()
      cached = mod[exportKey]
      gsap.registerPlugin(cached)
    }
    return cached
  }
}

export default defineNuxtPlugin(() => {
  if (process.client) {
    gsap.registerPlugin(ScrollTrigger, TextPlugin)

    gsap.defaults({ overwrite: 'auto' })
    gsap.ticker.lagSmoothing(500, 33)

    // ── Registered effects ──

    gsap.registerEffect({
      name: 'reveal',
      effect: (targets, config) =>
        gsap.fromTo(targets,
          { y: 20, autoAlpha: 0 },
          { y: 0, autoAlpha: 1, ...config },
        ),
      defaults: { duration: 0.8, ease: 'power2.out', stagger: 0 },
      extendTimeline: true,
    })

    gsap.registerEffect({
      name: 'textReveal',
      effect: (targets, config) =>
        gsap.to(targets, { y: '0%', ...config }),
      defaults: { duration: 0.6, ease: 'power3.out', stagger: 0.06, force3D: true },
      extendTimeline: true,
    })

    // Auto-sort ScrollTriggers by DOM position before every refresh
    ScrollTrigger.addEventListener('refreshInit', () => ScrollTrigger.sort())
  }

  // Lazy loading helpers for premium/heavy plugins
  const lazyLoadDrawSVG   = createLazyLoader(() => import('gsap/DrawSVGPlugin'))
  const lazyLoadMorphSVG  = createLazyLoader(() => import('gsap/MorphSVGPlugin'))
  const lazyLoadScramble  = createLazyLoader(() => import('gsap/ScrambleTextPlugin'))
  const lazyLoadSplitText = createLazyLoader(() => import('gsap/SplitText'), 'SplitText')

  return {
    provide: {
      gsap,
      ScrollTrigger,
      lazyLoadDrawSVG,
      lazyLoadMorphSVG,
      lazyLoadScramble,
      lazyLoadSplitText,
    },
  }
})
```

**Key patterns:** default imports (GSAP v3 uses default exports), `createLazyLoader` factory for premium plugins (dynamic import on first use, keeps initial bundle small), `ScrollTrigger.sort()` on `refreshInit` (pins refresh before downstream triggers). Provides: `$gsap`, `$ScrollTrigger`, plus four lazy loaders (`$lazyLoadSplitText`, etc.).
