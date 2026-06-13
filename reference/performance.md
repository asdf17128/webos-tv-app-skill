# TV Performance Rules

The mental model: **YouTube on a TV uses a native C++/WebGL renderer. You're using DOM.** DOM on TV silicon has a low performance ceiling, so you must spend frames deliberately. Chromium 108, weak GPU/CPU, 1080p. Every layout/paint you avoid is a frame you keep.

Rules in priority order. The first three are the ones that actually decide whether the app feels native.

## 1. Focus = direct DOM, never `setState`

See `focus-system.md`. A `setState` per arrow press re-renders the focused subtree and diffs the virtual DOM — visibly laggy on TV. Move the `.focused` class with `classList`, zero React.

## 2. Scroll with `transform: translateY`, not `overflow: scroll`

Native scroll triggers layout reflow on the TV. A GPU-composited transform does not.

```jsx
// A scroll container that moves via transform.
function Grid({ rows, scrollRow }) {
  const offsetY = scrollRow * ROW_HEIGHT;   // computed from focused row
  return (
    <div className="viewport">{/* fixed height, overflow:hidden */}
      <div className="track" style={{ transform: `translateY(${-offsetY}px)` }}>
        {rows}
      </div>
    </div>
  );
}
```

```css
.viewport { height: 100vh; overflow: hidden; }
.track    { will-change: transform; transition: transform 200ms ease; }
```

## 3. Animate ONLY `transform` and `opacity`

These two properties are GPU-composited and skip layout + paint. Anything else (`width`, `height`, `top`, `margin`, `box-shadow` size, `filter`) forces layout/paint and tanks the frame rate. Audit **every** `transition`/`animation` in the codebase — one careless `width` transition on a focused card is enough to drop frames.

```css
/* GOOD: scale + opacity only */
.card.focused { transform: scale(1.06); opacity: 1; }

/* BAD: animating layout properties */
.card.focused { width: 320px; height: 200px; }  /* don't */
```

For scrolling text/marquee and danmaku-style overlays, use CSS `@keyframes` that only translate, and let the compositor handle it.

## 4. `content-visibility: auto` + `contain: content`

Lets the browser skip rendering work for off-screen elements. Put it on list cards.

```css
.card {
  content-visibility: auto;
  contain: content;
  contain-intrinsic-size: 320px 220px;  /* reserve size so scroll height is stable */
}
```

## 5. `React.memo` every list component

Without it, a parent state change re-renders the entire grid. Memoize cards and rows; pass stable callbacks (`useCallback`) so memo isn't defeated.

```jsx
const VideoCard = React.memo(function VideoCard({ item, index, onOpen }) {
  /* ... */
}, (a, b) => a.item.id === b.item.id && a.index === b.index);
```

## 6. Keep pages mounted; hide with `display: none`

Unmounting a page loses its scroll position and component state, and forces an expensive re-fetch + re-render when the user returns. Keep pages alive and toggle visibility. The player **overlays** the home page rather than replacing it.

```jsx
<div style={{ display: route === 'home'   ? 'block' : 'none' }}><HomePage /></div>
<div style={{ display: route === 'search' ? 'block' : 'none' }}><SearchPage /></div>
{playing && <div className="player-overlay"><PlayerPage /></div>}
```

## 7. Resize images at the CDN; prefer webp

Downloading and decoding full-res images is a real cost on TV. If your image CDN supports a resize suffix/param, request thumbnail-sized webp (e.g. `…@672w_420h.webp`). Decode time and bandwidth both drop.

## 8. Preload + progressive render

Fetch in batches (~20 items), render only the visible 6–8, and serve the rest from an in-memory cache as the user scrolls. De-dup with a `Set` of ids so infinite-load doesn't repeat items.

## 9. Build target `chrome108`

```js
// vite.config.js
build: { target: 'chrome108', base: './' }
```

Newer JS/CSS that your toolchain emits by default can silently fail on the TV's Chromium. Pin the target. Use `base: './'` so assets resolve under `file://` packaging. Split big deps (player, react) into their own chunks so the initial parse is smaller.

## Measuring it

Don't guess — pull metrics from the device (see `debug-toolchain.md`). A useful diagnostic expression over CDP:

```js
JSON.stringify({
  focusables: document.querySelectorAll('[data-focus-id]').length,
  domNodes:   document.querySelectorAll('*').length,
  images:     document.querySelectorAll('img').length,
  animations: document.getAnimations?.().length || 0,
})
```

And `Performance.getMetrics` → watch `Nodes`, `LayoutCount`, `RecalcStyleCount`, `TaskDuration`, `JSHeapUsedSize`. Keep a baseline so a regression is obvious. (A well-optimized home screen in the source project sat around ~184 DOM nodes, ~21 focusables, ~1.8 MB heap — your numbers will differ, but the point is to *have* a number.)

A high `LayoutCount` that climbs while merely moving focus = you're animating a layout property or scrolling with `overflow`. A high `Nodes` count = render too many cards; lean on progressive rendering + `content-visibility`.
