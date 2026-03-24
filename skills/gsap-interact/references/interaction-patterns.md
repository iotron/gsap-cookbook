# Interaction Patterns Reference

## Table of Contents
- [Draggable with liveSnap](#draggable-with-livesnap)

---

## Draggable with liveSnap

SVG point-editing interaction using GSAP Draggable with `liveSnap` — drag handles snap to nearby points with inertia.

> Source: [Blake Bowen's CodePen](https://codepen.io/osublake/) (forked)

### HTML — SVG structure

```html
<svg id="svg" viewBox="0 0 400 400">
  <defs>
    <circle class="handle" r="10" />
    <circle class="marker" r="4" />

    <linearGradient id="grad-1" x1="0" y1="0" x2="400" y2="400" gradientUnits="userSpaceOnUse">
       <stop offset="0.2" stop-color="#00bae2"></stop>
       <stop offset="0.8" stop-color="#fec5fb"></stop>
    </linearGradient>
  </defs>

  <polygon id="star" stroke="url(#grad-1)"
    points="261,220 298,335 200,264 102,335 139,220 42,149 162,148 200,34 238,148 358,149" />
  <g id="marker-layer"></g>
  <g id="handle-layer"></g>
</svg>
```

### JS — Draggable with liveSnap and inertia

```js
var star = document.querySelector("#star");
var markerDef = document.querySelector("defs .marker");
var handleDef = document.querySelector("defs .handle");
var markerLayer = document.querySelector("#marker-layer");
var handleLayer = document.querySelector("#handle-layer");

// Build a points array from the SVG polygon's point list
var points = [];
var numPoints = star.points.numberOfItems;

for (var i = 0; i < numPoints; i++) {
  var point = star.points.getItem(i);
  points[i] = { x: point.x, y: point.y };
  createHandle(point);
}

function createHandle(point) {
  var marker = createClone(markerDef, markerLayer, point);
  var handle = createClone(handleDef, handleLayer, point);

  // Update the live SVG point on every drag frame
  var update = function () { point.x = this.x; point.y = this.y; };

  var draggable = new Draggable(handle, {
    onDrag: update,
    onThrowUpdate: update,   // Also fires during inertia throw
    inertia: true,            // Enable throw/flick behavior
    bounds: window,
    liveSnap: {
      points: points,         // Snap to any polygon vertex
      radius: 15              // Only snap within 15px proximity
    }
  });
}

// Clone an SVG template element and position it
function createClone(node, parent, point) {
  var element = node.cloneNode(true);
  parent.appendChild(element);
  gsap.set(element, { x: point.x, y: point.y });
  return element;
}
```

### Key patterns

- **`liveSnap.points`** — accepts an array of `{x, y}` objects; Draggable snaps to the nearest point within `radius` during drag
- **`liveSnap.radius`** — distance threshold (in pixels) for snapping; outside this radius, no snap occurs
- **`inertia: true`** — enables throw/flick physics; `onThrowUpdate` fires each frame during the inertia phase
- **`onDrag` + `onThrowUpdate`** — both update the SVG polygon point in real time, keeping the shape in sync with handles
- **SVG `points.getItem(i)`** — returns a live `SVGPoint` reference; mutating `.x`/`.y` immediately updates the rendered polygon
