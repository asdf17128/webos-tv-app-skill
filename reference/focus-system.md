# Zero-Render D-pad Focus System

A TV app is navigated entirely by a D-pad remote (up/down/left/right/OK/Back). There is no pointer hover that the platform tracks for you — **you** own the focus model.

The naive React approach (`const [focusedId, setFocusedId] = useState()`, then conditionally apply a class) re-renders on every single arrow press. On TV hardware the virtual-DOM diff for a grid of cards is slow enough to feel laggy. The fix: **focus changes touch the DOM directly and never go through React.**

## Design

- A module-level `Map` registry: `id -> { row, col, group, onSelect }`.
- Each focusable mounts via a `useFocusable` hook → registers itself, renders `data-focus-id` + a stable click/hover handler, and unregisters on unmount.
- Focus id format: **`{group}-{row}-{col}`** (e.g. `sidebar-0-0`, `content-1-2`). This encoding is what makes navigation O(1).
- Focus move = remove `.focused` from the old element, add it to the new — pure `classList`, no render.
- One global `keydown` listener does navigation + Enter + Back.

## Full hook + engine

```js
import { useEffect, useCallback, useRef } from 'react';

const focusRegistry = new Map();   // id -> { row, col, group, onSelect }
let currentFocusId = null;
let lastSidebarFocus = 'sidebar-0-0';   // remember where we left the sidebar

// The only place focus visually changes — direct DOM, no React.
function applyFocus(newId) {
  const prevId = currentFocusId;
  currentFocusId = newId;
  if (newId && newId.startsWith('sidebar-')) lastSidebarFocus = newId;

  if (prevId) {
    const prev = document.querySelector(`[data-focus-id="${prevId}"]`);
    if (prev) prev.classList.remove('focused');
  }
  if (newId) {
    const el = document.querySelector(`[data-focus-id="${newId}"]`);
    if (el) { el.classList.add('focused'); el.scrollIntoView({ block: 'nearest' }); }
  }
  globalListeners.forEach(fn => fn(newId));   // e.g. sidebar expand on focus
}

export function registerFocusable(id, data) { focusRegistry.set(id, data); }
export function unregisterFocusable(id) {
  focusRegistry.delete(id);
  if (currentFocusId === id) currentFocusId = null;
}
export function setFocus(id) {
  if (!focusRegistry.has(id) || id === currentFocusId) return;
  applyFocus(id);
}
export function getCurrentFocusId() { return currentFocusId; }

const globalListeners = new Set();
export function onFocusChange(fn) { globalListeners.add(fn); return () => globalListeners.delete(fn); }

// ---- O(1) navigation: compute the target id by arithmetic, then a .has() check ----
function navigateGrid(fromId, direction) {
  const from = focusRegistry.get(fromId);
  if (!from) return null;
  let { row, col, group } = from, tr = row, tc = col;
  if (direction === 'up') tr--;
  else if (direction === 'down') tr++;
  else if (direction === 'left') tc--;
  else if (direction === 'right') tc++;

  const target = `${group}-${tr}-${tc}`;
  if (focusRegistry.has(target)) return target;

  // Vertical move into a shorter row: fall back to the nearest existing column.
  if (direction === 'down' || direction === 'up') {
    for (let c = col; c >= 0; c--) {
      const id = `${group}-${tr}-${c}`;
      if (focusRegistry.has(id)) return id;
    }
  }
  return null;
}

// Find any focusable in a group near a preferred row (used when crossing groups).
function findInGroup(group, preferRow) {
  const exact = `${group}-${preferRow}-0`;
  if (focusRegistry.has(exact)) return exact;
  for (let d = 1; d <= 8; d++) {
    if (focusRegistry.has(`${group}-${preferRow - d}-0`)) return `${group}-${preferRow - d}-0`;
    if (focusRegistry.has(`${group}-${preferRow + d}-0`)) return `${group}-${preferRow + d}-0`;
  }
  for (const [id, data] of focusRegistry) if (data.group === group) return id;
  return null;
}

// ---- Global keyboard handler ----
let keyHandler = null;
let customKeyHandler = null;
// Pages (e.g. the player) can intercept keys first; return true to consume.
export function setCustomKeyHandler(handler) { customKeyHandler = handler; }

export function initKeyboardNav() {
  if (keyHandler) return;
  keyHandler = (e) => {
    if (customKeyHandler && customKeyHandler(e)) return;
    const key = e.key;

    // webOS Back is keyCode 461 — NOT Backspace. Route to your router.
    if (e.keyCode === 461 || key === 'Backspace' || key === 'GoBack') {
      e.preventDefault(); e.stopPropagation();
      window.dispatchEvent(new CustomEvent('tv-back'));
      return;
    }
    if (!['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight', 'Enter'].includes(key)) return;
    e.preventDefault();

    if (key === 'Enter') { if (currentFocusId) focusRegistry.get(currentFocusId)?.onSelect?.(); return; }
    if (!currentFocusId) return;
    const from = focusRegistry.get(currentFocusId);
    if (!from) return;

    const dir = { ArrowUp: 'up', ArrowDown: 'down', ArrowLeft: 'left', ArrowRight: 'right' }[key];

    if (dir === 'up' || dir === 'down') {
      const next = navigateGrid(currentFocusId, dir);
      if (next) setFocus(next);
      return;
    }

    // Horizontal: try within-group first, then cross sidebar <-> content at edges.
    let next = navigateGrid(currentFocusId, dir);
    if (!next) {
      if (dir === 'left' && from.group !== 'sidebar') {
        next = lastSidebarFocus || 'sidebar-0-0';
        if (!focusRegistry.has(next)) next = findInGroup('sidebar', 0);
      } else if (dir === 'right' && from.group === 'sidebar') {
        next = 'content-0-0';
        if (!focusRegistry.has(next)) next = findInGroup('content', 0);
      }
    }
    if (next) setFocus(next);
  };
  window.addEventListener('keydown', keyHandler);
}

// ---- The hook each focusable element uses ----
export function useFocusable({ id, row = 0, col = 0, group = 'content', onSelect }) {
  // Keep the latest callback in a ref so changing it never re-registers
  // (callback in a useEffect dep array => infinite re-register loop).
  const onSelectRef = useRef(onSelect);
  onSelectRef.current = onSelect;

  useEffect(() => {
    registerFocusable(id, { row, col, group, onSelect: () => onSelectRef.current?.() });
    return () => unregisterFocusable(id);
  }, [id, row, col, group]);   // NOT onSelect

  const handleClick = useCallback((e) => { e.preventDefault(); setFocus(id); onSelectRef.current?.(); }, [id]);
  const handleMouseEnter = useCallback(() => setFocus(id), [id]);

  return {
    isFocused: currentFocusId === id,   // accurate only at render time, not reactive
    props: {
      'data-focus-id': id,
      onClick: handleClick,
      onMouseEnter: handleMouseEnter,
      style: { cursor: 'pointer' },
    },
  };
}
```

## Usage

```jsx
function VideoCard({ index, item, onOpen }) {
  const { props } = useFocusable({
    id: `content-${Math.floor(index / 2)}-${index % 2}`,  // 2-column grid
    row: Math.floor(index / 2), col: index % 2, group: 'content',
    onSelect: () => onOpen(item),
  });
  return <div className="card" {...props}>{/* ... */}</div>;
}
```

```css
.card { transition: transform 120ms ease, box-shadow 120ms ease; } /* transform/opacity only */
.card.focused { transform: scale(1.06); box-shadow: 0 0 0 4px #fff; z-index: 1; }
```

## Why the id scheme matters

```
sidebar-0-0          content-0-0  content-0-1
sidebar-1-0          content-1-0  content-1-1
...                  content-2-0  content-2-1
```

- **Up/Down:** same group, `row ± 1`.
- **Left/Right:** same group, `col ± 1`; at a column edge, jump groups (sidebar ↔ content).
- The next id is a string template + one `Map.has()` — never a scan of the registry. With hundreds of cards this stays constant-time.

## Gotchas

- **Hover vs. load.** `onMouseEnter`/D-pad-hover should only *highlight*. Don't kick off an API request on every focus move (a sidebar where each highlight reloads a page feels terrible). Separate "highlight this item" from "commit / load its data" (the latter on Enter).
- **`isFocused` is not reactive.** It's correct only at render time. Anything that must react to focus should be CSS (`.focused`) or a `onFocusChange` listener, not React state.
- **Re-register on layout change.** If the grid's row/col mapping changes (new data appended), the dependency array `[id, row, col, group]` re-registers correctly — just make sure those props actually change.
