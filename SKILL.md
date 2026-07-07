---
name: webos-tv-app
description: Use when building, debugging, or optimizing React apps for LG webOS Smart TVs (e.g. 2024 LG C4) — covers the Luna service + local-proxy architecture, D-pad focus without re-renders, TV performance rules, ares-package/SSH deploy, and on-device CDP debugging.
---

# Building React Apps for LG webOS Smart TVs

各位开发者，这是一套从真实 webOS TV 项目里打磨出来的方法论。Target hardware: LG webOS 24 (Chromium 108, Node.js v16 for services), 1080p, driven by a D-pad remote. The patterns below assume that constraint set; they are what makes a DOM-based TV app actually usable on weak TV silicon.

Read this file first. It is a map. The longer code lives in `reference/`:

- `reference/service-and-proxy.md` — Luna JS service + local HTTP proxy, full skeleton
- `reference/focus-system.md` — zero-render D-pad focus engine
- `reference/performance.md` — the TV performance rule set, with CSS/JSX
- `reference/debug-toolchain.md` — CDP-over-SSH debugging, self-driving, screenshots, netcap
- `reference/testing.md` — release pipeline, old-device (Node 8) testing, verification discipline

---

## 1. Architecture: two channels, not one

A packaged webOS app runs from a `file://` / `null` origin. External APIs do **not** return CORS headers for it, and there is no cookie jar. You cannot fetch your backend directly from the web app. So a webOS app is really **two processes** shipped in one `.ipk`:

```
On TV:
  Web App (React/Chromium) ──Luna bus──▶ JS Service (Node.js) ──HTTPS──▶ API
                           ◀───────────
  <video>/<img>/HLS ──HTTP──▶ Local Proxy (127.0.0.1:PORT) ──HTTPS──▶ CDN

In dev (browser):
  Web App ──HTTP──▶ Mac dev proxy ──HTTPS──▶ API/CDN   (mirrors the two paths)
```

**Why both channels are needed:**

- **Luna bus → JS Service.** The service is plain Node.js with no CORS, no origin restriction, and a writable filesystem. Use it for all JSON/API calls, login, and cookie persistence. Bytes cross the bus as JSON (base64 for binary), so it is great for data but useless for media.
- **Local HTTP proxy (127.0.0.1).** `<video>`, `<img>`, and HLS/`<source>` elements can only load from a URL — they cannot consume Luna responses. So the same service also opens a localhost HTTP server. The web app points media at `http://127.0.0.1:PORT/proxy/{host}/{path}`, and the service streams the upstream HTTPS response back (adding `Referer`, cookies, range support, CORS).

Key service requirements (full code in `reference/service-and-proxy.md`):

- `webos-service` + `activityManager.create('keepAlive', ...)` so the background service is not reaped.
- A **host allowlist** on both the Luna `fetch` method and the proxy — never proxy arbitrary hosts.
- On the **binary/segment path, force `Accept-Encoding: identity`.** The proxy pipes bytes raw without re-emitting `Content-Encoding`; gzip there corrupts `Content-Range`/`Content-Length` and breaks the media player's range requests.
- A **keep-alive `https.Agent`** (TLS reuse is a big latency win on TV CPUs) plus an **upstream timeout** and socket teardown on client disconnect, so one stuck segment cannot hang playback forever.
- **CommonJS only** (`require`, not `import`) — the webOS Node runtime is v16.

```js
// Web app side — one helper picks Luna on TV, proxy in dev
const onTV = typeof window.webOS !== 'undefined';
async function apiFetch(url, opts) {
  if (onTV) return lunaCall('com.you.app.service', 'fetch', { url, ...opts });
  return fetch(`http://localhost:9527/api?url=${encodeURIComponent(url)}`, opts);
}
// Media URL builder — same shape in both environments
const mediaUrl = (raw) => {
  const u = new URL(raw);
  return onTV
    ? `http://127.0.0.1:${proxyPort}/proxy/${u.host}${u.pathname}${u.search}`
    : `http://localhost:9527/proxy/${u.host}${u.pathname}${u.search}`;
};
```

---

## 2. D-pad focus without React re-renders

The single biggest perf decision. On TV hardware, a React `setState` on every focus move (which re-renders the focused card, its siblings, and diffs the virtual DOM) is visibly laggy. **Move focus by mutating the DOM directly, with zero React involvement.**

The system (full code in `reference/focus-system.md`):

1. A module-level `Map` registry: `id -> { row, col, group, onSelect }`. Each focusable registers on mount via a `useFocusable` hook and unregisters on unmount.
2. Each focusable renders `data-focus-id="{group}-{row}-{col}"` and a CSS `.focused` style.
3. Focus change = `querySelector('[data-focus-id=prev]').classList.remove('focused')` + `add('focused')` on the new one. No state, no render.
4. **O(1) grid navigation:** the next id is computed by arithmetic on `{group}-{row}-{col}` (e.g. down = `row+1`), then a registry `.has()` check — never iterate the registry. Edge cases (column gaps, crossing from a sidebar group to a content group) are a couple of bounded fallbacks.
5. One global `keydown` listener maps arrows → navigation, Enter → the focused element's `onSelect`. The webOS **Back** button is `keyCode 461` (not Backspace) — handle it explicitly.

```js
function applyFocus(newId) {
  const prev = document.querySelector('[data-focus-id].focused');
  if (prev) prev.classList.remove('focused');
  const el = document.querySelector(`[data-focus-id="${newId}"]`);
  if (el) { el.classList.add('focused'); el.scrollIntoView({ block: 'nearest' }); }
}
```

The `onSelect` callback is stored in a `useRef` and read through a stable wrapper, so changing the callback never re-registers and a stale closure never fires (a real bug class: putting `onSelect` directly in a `useEffect` dependency array causes infinite re-register loops).

---

## 3. TV performance rules

You are rendering a 10-foot UI in DOM on a Chromium 108 device with weak GPU/CPU. Every frame counts. Full examples in `reference/performance.md`. The rules, by impact:

1. **Focus = direct DOM classList, never setState.** (See §2.)
2. **Scroll with `transform: translateY(...)`, not `overflow: scroll`.** Native scroll triggers layout reflow; transform is GPU-composited.
3. **Animate only `transform` and `opacity`.** These two skip layout/paint and go straight to GPU compositing. A stray `width`/`height`/`top` transition will tank the frame rate.
4. **`content-visibility: auto` + `contain: content`** on list cards so the browser skips off-screen work.
5. **`React.memo` every list component** so a parent state change doesn't re-render the whole grid.
6. **Keep pages mounted; hide with `display: none`.** Unmounting loses state and forces an expensive re-fetch/re-render on return. The player overlays the home page rather than replacing it.
7. **Request resized thumbnails** (e.g. a `@WxH` webp suffix if your image CDN supports it) — less to download and decode.
8. **Preload + progressively render:** fetch ~20 items, render the visible 6–8, pull the rest from cache as the user scrolls.
9. **Build target `chrome108`** in Vite — newer syntax/features silently break on the TV.

```js
// vite.config.js
export default defineConfig({
  plugins: [react()],
  base: './',                          // relative paths for file:// packaging
  build: {
    target: 'chrome108',
    rollupOptions: { output: { manualChunks: {
      shaka: ['shaka-player'], 'react-vendor': ['react', 'react-dom'],
    } } },
  },
});
```

---

## 4. Build, package, deploy

```bash
# 1. Build the web app (relative base, chrome108 target)
npx vite build
# 2. Copy webOS metadata (appinfo.json + icons) into the build output
cp webos-meta/* dist/
# 3. Package app + service into one .ipk
cd dist && ares-package --no-minify . ../service/com.you.app.service
# 4. Install + launch over SSH (see below)
node tools/deploy.mjs
```

`appinfo.json` essentials:

```json
{
  "id": "com.you.app", "type": "web", "main": "index.html",
  "title": "Your App", "icon": "icon.png", "largeIcon": "largeIcon.png",
  "resolution": "1920x1080",
  "disableBackHistoryAPI": true,
  "requiredPermissions": ["all"]
}
```

- **`disableBackHistoryAPI: true`** is mandatory for an SPA — otherwise webOS handles Back itself and exits the app instead of letting your router navigate.
- Icons must be real PNGs (`icon.png` 80×80, `largeIcon.png` 130×130). Generate them with a real image tool (`sips`/ImageMagick); hand-rolled PNG bytes get rejected.

**Deploy gotchas:**

- The official `@webos-tools/cli` only supports Node 14–16; on a modern Node it throws (e.g. `isDate is not a function`). The robust path is a custom `deploy.mjs` using the `ssh2` library: SSH in, `sftp` the `.ipk` up, then `luna-send-pub` `dev/install` followed by `applicationmanager/launch`. Full script in `reference/debug-toolchain.md`.
- **Developer Mode passphrase rotates.** The SSH key passphrase comes from the TV's Developer Mode app and changes when the dev session is renewed. If SSH auth fails, re-read the current passphrase from the app and re-fetch the key (`ares-novacom --device tv --getkey`). Pass it as a CLI arg so it's never hard-coded.
- The TV uses an `ssh-rsa` host key — set `algorithms: { serverHostKey: ['ssh-rsa'] }`.

---

## 5. Debug-first methodology

**Always pull logs / metrics / network from the real device before writing a fix.** Blind code-and-redeploy cycles waste enormous time on TV. "It feels laggy" has a dozen causes (too many DOM nodes, too many live animations, React re-renders, image decode, a flaky CDN). Only data tells you which.

The legibility loop you build once and reuse:

```bash
node tools/debug.mjs   "<pass>"            # console + exceptions + perf metrics + DOM counts
node tools/watch.mjs   "<pass>" 300        # long monitor; polls <video> state every 5s
node tools/netcap.mjs  "<pass>" 70         # per-request timing/host/status via the proxy
node tools/drive.mjs   "down,down,ok" out.png  # inject key events, then screenshot
node tools/sh.mjs      "journalctl -n 100" "<pass>"  # arbitrary shell on the TV
bash   tools/verify.sh "<pass>"            # build → deploy → wait → CDP assert → screenshot
```

All of these connect the same way: **SSH tunnel → TV's `127.0.0.1:9998` (Chrome DevTools Protocol) → WebSocket.** Details and full scripts in `reference/debug-toolchain.md`. Highlights:

- `Performance.getMetrics` → `JSHeapUsedSize`, `Nodes`, `LayoutCount`, `RecalcStyleCount`, `TaskDuration`. Keep a baseline (an optimized home screen here was ~184 DOM nodes, ~21 focusables, ~1.8 MB heap) so regressions are obvious.
- `Runtime.evaluate` runs diagnostic JS in the live app — count focusables, find broken images, read `<video>` `currentTime`/`readyState`/`buffered`.
- **`Input.dispatchKeyEvent`** injects D-pad keys so an agent can self-drive the app (navigate, open a video) and then screenshot to verify — no human in the loop.

Two CDP limitations to know:

- **`Page.captureScreenshot` cannot capture the hardware video plane** — the decoded video sits on a separate overlay, so the screenshot shows white/black where the video is. Verify playback via `<video>.currentTime` advancing, not the screenshot.
- **Native HLS playback bypasses the CDP Network domain** — the TV's native media pipeline fetches segments below Chromium, so `Network.*` events won't show them. Route HLS through your local proxy and read timing from the proxy logs (`netcap.mjs`) instead.

Note: a JS service's `console.log` does **not** land in `journald` — capture it over CDP from the app side, or log to a file the proxy serves.

---

## 6. Media player pitfalls

Hard-won, from `reference/debug-toolchain.md` (streaming section):

- **Chromium on TV plays neither DASH nor HLS natively.** DASH needs an MSE player — use **Shaka Player**. For HLS, route through the local proxy and feed a `<video>`-compatible stream.
- **Do NOT use exponential backoff in the player's retry config.** A `baseDelay:500, backoffFactor:2, maxAttempts:5` makes *every* load that hits one transient failure wait ~7.5s (500+1000+2000+4000) in the manifest phase. Use **flat short delays** (`backoffFactor:1, baseDelay:150–200`). This cut loads from 5–9s to <1s.
- **Prefer the stable origin CDN over flaky P2P/PCDN nodes.** A manifest's primary `baseUrl` is often a fast-but-unreliable PCDN/P2P node that returns short/bad data on range requests (→ "Payload length does not match range requested bytes" → fatal stream error). Emit **all** backup URLs as multiple `<BaseURL>` entries and **order the stable origin CDN first** so the player fails over to reliable hosts.
- **Resume via `player.load(url, startTime)`, not load-at-0-then-seek** — the latter double-buffers and wastes bandwidth.
- A **stall watchdog** (nudge `currentTime` if playback hasn't advanced for >8s) recovers from soft freezes.
- Disable the player's ABR if you have a manual quality selector (`abr: { enabled: false }`); re-fetch the play URL on quality change rather than packing every rep into one manifest.

---

## 7. Testing & verification

Full treatment in `reference/testing.md`. The parts people skip and regret:

- **Release pipeline, layered fail-fast:** ES2017 syntax gate on service files →
  run the *real* service under **docker `node:8`** (webOS 5's actual service
  runtime; LG's emulator only covers ≤6.0, is x86-only, and won't run on Apple
  Silicon) → build → deploy → CDP assertions on the live app.
- **Node 8 forbidden list** for services: `URL`/`URLSearchParams` globals
  (use `require('url').URL`), `globalThis`, `?.`/`??`, `ws@8`. A `new URL`
  inside try/catch once turned every request on webOS 5 into a silent
  "Invalid URL" — service up, app error-free, all screens empty.
- **Positive control:** a fix is verified only if the same harness reproduces
  the bug on the pre-fix build first. Two consecutive "verified" fixes shipped
  broken here before this rule existed.
- **Failure paths are part of the spec** — cut the network and assert the error
  text reaches the screen, especially for diagnostics features.
- **Verify the instrument before the product** when results look impossible —
  TV-side CDP mouse injection can silently die; in-page event counters expose it.
- **Interaction semantics** (hover/wheel/click) want a trusted input pipeline:
  desktop Playwright (`page.mouse`) + disabled TV bridge + mocked feeds, not
  synthetic `dispatchEvent`.

---

## Common pitfalls (quick table)

| Symptom | Cause | Fix |
|---|---|---|
| App exits on Back press | webOS owns Back by default | `disableBackHistoryAPI: true`; handle keyCode 461 |
| Blank page after navigation | component unmounted, state lost | `display:none` instead of conditional unmount |
| Focus jumps around | non-contiguous focus ids | continuous `{group}-{row}-{col}` ids |
| Sidebar page switch janks | every focus move triggers an API call | separate "hover-highlight" from "load data" |
| Service dies after a while | uncaught exception / no keepAlive | `process.on('uncaughtException')` + `activityManager` keepAlive |
| Media range/length corrupt | proxy forwarded gzip | force `Accept-Encoding: identity` on the byte path |
| Deploy auth fails | Dev Mode passphrase rotated | re-read passphrase, re-fetch SSH key |
| Hook infinite loop | callback in `useEffect` deps | store callback in `useRef` |
| CDN 403 | sent `Origin` header to CDN | drop `Origin` on CDN requests |
