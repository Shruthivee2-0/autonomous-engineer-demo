# Autonomous Engineer — Context Orb Demo

A self-contained landing-page demo built around a scroll-driven 3D context-graph orb. The animation progresses from scattered translucent colored nodes → a rotating wireframe sphere → a rounded rectangle that frames the orb → a play button overlay.

**Live demo:** https://shruthivee2-0.github.io/autonomous-engineer-demo/

---

## Files

```
.
├── index.html   Everything — markup, CSS, and Canvas 2D animation, all inline
└── README.md    This file
```

No build step. No dependencies. Open `index.html` in any modern browser.

---

## Page structure (top to bottom)

| Section | Height | Purpose |
|--|--|--|
| `<section class="hero">` | 100vh | Heading, subtext, CTAs. Scattered colored nodes float behind |
| `<div class="spacer">` | 220vh | Drives the build animation as the user scrolls through it |
| `<section class="video-section">` | 100vh | Scroll buffer for the formed-orb state. Empty visually |
| `<button class="play-btn">` | — | Fixed at viewport center; fades in once the build completes |

A single fixed-position `<canvas>` (`#bg`) covers the viewport and draws every frame of the animation. The body has a soft 4-corner gradient backdrop (lavender + peach + white).

---

## How the animation works

### Scroll → progress

`updateProgress()` measures `getBoundingClientRect()` of `.spacer` and computes a `progress` value 0.0 → 1.0 spanning the entire spacer scroll range. All animation phases derive from this single value.

### Phases

Each phase is a `smoothstep(a, b, progress)` so values ease in and out smoothly.

| Phase | Range | What happens |
|--|--|--|
| `tMove` | 0.00 → 0.42 | Nodes drift from scatter positions onto the Fibonacci sphere |
| `tEdge` | 0.12 → 0.52 | Wireframe edges fade in |
| `tFrameStroke` | 0.52 → 0.78 | Rounded rectangle outlines the orb |
| `tFrameFill` | 0.75 → 0.96 | Frame fills with light gray (the video container) |
| Play button | ≥ 0.94 | `.visible` class toggled; CSS handles the opacity fade |

### Canvas technique

- **115 nodes** with 3D positions generated via a Fibonacci sphere (`fibonacciSphere(N)`)
- Each frame: rotate the sphere → project to 2D (orthographic) → render
- **Edges** connect each node to its 5 nearest 3D neighbours (`nearestNeighbourEdges(nodes, K)`)
- Edges are **shrunk** at each endpoint to stop at the node's circle boundary
- After edges draw, a `globalCompositeOperation = 'destination-out'` pass **erases** the canvas inside every node disc, so any line that crosses a non-endpoint node disappears cleanly
- Nodes are then drawn (back-to-front by depth) on top of the cleared discs
- `requestAnimationFrame` loop — typically 60 fps; only the canvas redraws each frame

### Node hierarchy

| Tier | Count | Base radius (px) | Purpose |
|--|--|--|--|
| Primary (2) | 8 | 12–16 | Anchor nodes, one per distinct team color |
| Secondary (1) | ~25 | 7–9 | Medium nodes |
| Tertiary (0) | rest | 4–5.8 | Small dots |

### Color palette (`COLS`)

Each team category is `{ f: pastel-fill, s: saturated-stroke }`. Current set is warm — three pinks, three oranges/corals, plus violet and amber:

```js
agent:    {f:'#fcd9c9', s:'#E0531F'},  // burnt orange (matches CTA)
api:      {f:'#fee5d0', s:'#FB923C'},  // tangerine
workflow: {f:'#fde5cf', s:'#F97316'},  // medium orange
incident: {f:'#fbd5e6', s:'#EC4899'},  // hot pink
knowledge:{f:'#e0d4f8', s:'#8B5CF6'},  // violet
infra:    {f:'#fce7f3', s:'#F472B6'},  // light pink
observ:   {f:'#fbe9c4', s:'#F59E0B'},  // amber
```

Each node is colored uniformly with its category's pastel fill + saturated stroke, no per-node-position color mixing.

### Scatter positioning

Each node has a scatter position (where it sits at the opening) computed by **rejection sampling**:

1. Random angle + radius in normalized [0,1] coords
2. **Reject** if inside the hero text bounding box (`x: 0.27–0.73, y: 0.30–0.66`)
3. **Reject** if it overlaps a previously placed node (within `r₁ + r₂ + 14px`)
4. Up to 120 attempts; later attempts push the radius outward
5. If still no valid spot found, the node is flagged `hiddenAtScatter` and stays invisible at the opening, fading in only as it moves toward the orb

A separate ~20% of non-primary nodes are also flagged `hiddenAtScatter` deterministically, to keep the opening less dense than the orb without changing the orb's wireframe density.

---

## Tunable parameters

All in `index.html` — search by name. The most useful ones:

### Timing
- `.spacer { height: 220vh }` — total scroll length for the build
- The four `smoothstep(...)` calls in `draw()` — shift any phase earlier/later
- `rotY += 0.0009 + 0.00168 * tMove` — orb rotation speed
- `rotX = Math.sin(pulse * 0.00036) * 0.18` — wobble amplitude + frequency

### Density and sizes
- `const N = 115` in `buildNodes()` — total node count
- `PRIMARY_COUNT = 8` — how many anchor-tier nodes
- The `baseR` ternary — radius range per tier
- `K = 5` in `nearestNeighbourEdges(nodes, 5)` — edges per node

### Opacity / visibility
- `scatterAlpha = 0.50` — opening node opacity
- `formedAlpha = 1.0` — orb node opacity
- `srand(i, 7) < 0.20` — proportion of opening-hidden nodes

### Colors
- `COLS` — palette
- `rgba(170,175,190, ...)` — wireframe edge stroke color
- `rgba(170,170,185, ...)` and `rgba(240,240,243, ...)` — frame stroke/fill
- `#E0531F` / `#C44419` — primary button color + hover
- Body gradient stops: `#DDC8FF` (lavender corners) + `#FFF5F2` (peach corners)

### Frame dimensions
- `fw = Math.min(CW * 0.85, 1100)` — frame max width
- `fh = fw * 9 / 16` — aspect ratio
- `frameRadius = 18` — corner roundness

### Scatter spread
- `SCATTER_X`, `SCATTER_Y` in `buildNodes()` — how wide nodes spread on each axis
- `rad = 0.30 + srand() * 0.62` — radius range (min/max distance from center)
- `TEXT_BOX` — the rectangle that nodes avoid (matches the hero text bounding box)

---

## Browser support

- Chrome 99+, Safari 16+, Firefox 113+ (uses Canvas 2D `roundRect`, finalized 2022)
- Works as plain static HTML — no transpiler needed
- Tested at 60 fps on modern laptops; performance scales with `N` and the `K` neighbours-per-node value

---

## Integration notes

- **Embedding inside a scrolling container** instead of `window`: `updateProgress()` listens to `window.scroll` and reads `.spacer.getBoundingClientRect()`. If the page becomes a child of some other scroll context, both the listener target and the rect calculation need updating.
- **Page background**: `body { background: ...; background-attachment: fixed; }` — if you embed this inside another layout, decide whether to keep the gradient as a fixed body background or move it to a dedicated wrapper.
- **Canvas is `position: fixed; inset: 0;`** — it always covers the viewport. If you need the animation contained within a region, change to `position: absolute` inside a positioned wrapper and resize accordingly.
- **The play button is `position: fixed`** at viewport center. If you want it to live inside the video frame element specifically, move it back into the DOM tree and remove the fixed positioning.
- **Scroll-driven**: removing the `.spacer` removes the animation. If you want the animation to play automatically (no scroll), drive `progress` from a timer or `IntersectionObserver` instead of scroll position.

---

## Repo

- **Repo:** https://github.com/Shruthivee2-0/autonomous-engineer-demo
- **`main` branch** is the live demo (deployed to GitHub Pages)
- **`handoff` branch** (this branch) — same code + this README
