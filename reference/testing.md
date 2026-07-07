# Testing & verification for webOS TV apps

How to know a build actually works — on the TV in front of you AND on the
five-year-old TV you don't own. Distilled from shipping fixes to real users on
webOS 5/6/24 hardware, including two "verified" fixes that turned out to be
nothing of the sort until the discipline below was applied.

---

## 1. The release pipeline

One command, layered fail-fast, run before every release:

```
[1] syntax   — every service file parses as ES2017 (webOS 5 service = Node 8)
[2] node8    — run the REAL service under docker node:8, drive its handlers
[3] build    — vite production build
[4] deploy   — package + install + relaunch on the reference TV
[5] device   — CDP assertions on the live app (rendered? broken images? focusables?)
[+] smoke    — optional full UI drive-through (keys + DOM asserts per screen)
```

Layers 1–3 need no TV (`--no-tv` mode); 4–5 verify on real hardware. Example
skeleton:

```bash
#!/bin/bash
set -e
# [1] Node-8 syntax gate (acorn: ES2017 ≈ Node 8)
for f in service/*/[!n]*.js service/*/cast/*.js; do
  npx --yes acorn --ecma2017 --silent "$f" || { echo "SYNTAX-FAIL $f"; exit 1; }
done
# [2] real Node 8 (see §2)
bash tools/test-node8/test.sh
# [3] build
(cd app && npx vite build)
# [4] deploy + relaunch (ssh2 + luna-send-pub, see debug-toolchain.md)
bash build.sh && node tools/launch.mjs com.you.app
# [5] on-device assertions via CDP Runtime.evaluate
node tools/eval.mjs "(function(){
  var cards=document.querySelectorAll('[data-focus-id]').length;
  var broken=[].slice.call(document.querySelectorAll('img'))
    .filter(function(i){return i.complete && i.naturalWidth===0;}).length;
  return (cards>5 && broken===0 ? 'PASS' : 'FAIL')+' cards='+cards+' broken='+broken;
})()"
```

---

## 2. Old-device testing without old devices

**The problem.** Your dev TV is new (Node 16 / Chromium 108), your users' TVs
are not (webOS 5 = **Node 8** / Chromium 68; webOS 6 = Chromium 79). Official
options don't save you:

- LG's VirtualBox **Emulator** only covers webOS TV ≤ 6.0 and is **x86-only** —
  it will not run on Apple Silicon at all.
- The newer native **Simulator** (macOS arm64) only covers webOS 22+.
- UTM/QEMU x86 emulation technically works and practically doesn't (slow, fragile).

**The solution for the service layer** (where old-webOS breakage almost always
lives): run the *real* service under **docker `node:8`** with a stubbed
`webos-service`, and drive its registered handlers end-to-end.

```js
// stub/webos-service/index.js — minimal Luna stand-in
function Service(name) {
  this.name = name; this.methods = {};
  this.activityManager = { create: function () {} };
  module.exports.last = this;               // expose the instance to the test
}
Service.prototype.register = function (m, cb) { this.methods[m] = cb; };
module.exports = Service;
```

```js
// run8.js — evaluate the REAL service.js on real Node 8, drive the hot path
require('./service.js');
var svc = require('webos-service').last;
svc.methods['fetch']({
  payload: { url: 'https://api.example.com/health' },
  respond: function (r) {
    if (r.returnValue === false) { console.log('FAIL:', r.error); process.exit(1); }
    console.log('fetch OK on real Node 8: status=' + r.status);
    process.exit(0);
  }
});
setTimeout(function () { console.log('TIMEOUT'); process.exit(1); }, 20000);
```

```bash
docker run --rm --platform linux/amd64 -v "$WORK":/svc -w /svc node:8 node run8.js
```

**Node 8 forbidden list** (each of these shipped a real outage or would have):

| Missing on Node 8 | Use instead |
|---|---|
| `URL` / `URLSearchParams` globals (Node 10+) | `require('url').URL` (exists since 6.13) |
| `globalThis` (Node 12+) | `global` in services |
| `?.` / `??` / optional `catch {}` (Node 14/10) | ES2017 syntax only |
| `ws@8` (needs Node 14) | pin `ws@^7.5` |

The `URL` one is instructive: `new URL(...)` sat inside a `try/catch`, so on
Node 8 every request "politely" failed with *Invalid URL* — the service stayed
up, the app showed no errors, and every screen was just empty. Nothing in
app-side inspection (`ares-inspect`) showed anything. Only running the real
service under real Node 8 (or a user's photo of an error-text-bearing UI)
catches this class.

For the **app layer** on old TVs, `@vitejs/plugin-legacy` handles syntax; the
residual risks are missing *globals* (`globalThis` needs a polyfill on
Chromium 68) and too-new Web APIs — grep for them when you target old webOS.

---

## 3. Verification discipline

Rules that exist because their violations each burned a release:

1. **Positive control: a fix is "verified" only if the same harness reproduces
   the bug on the broken build first.** Check out the pre-fix revision of just
   the touched files, deploy, run the test (it must FAIL/reproduce), restore,
   re-run (it must pass). Without this you are testing your test's inability
   to see the bug — one edge-scroll bug here survived two consecutive
   "verified" fixes exactly this way.

2. **Test the failure path, not just the happy path.** Especially for
   diagnostics/error-surfacing features: cut the network (Playwright
   `route.abort()`), and assert the *error text* reaches the screen. A
   diagnostics page that's only been seen all-green is untested.

3. **When results look wrong, verify the instrument before the product.**
   CDP `Input.dispatchMouseEvent` on a TV can silently stop delivering events
   (mousemove count: zero) while everything else still works. Hook counters in
   the page (`document.addEventListener('mousemove', ..., true)`), inject, and
   read the counters — that one check separates "app broke" from "harness
   broke" and saves hours of chasing ghosts.

4. **Interaction semantics need a trusted input pipeline.** Don't trust
   synthetic `dispatchEvent` or TV-side CDP injection for hover/wheel/click
   testing — React's enter/leave synthesis and the browser's hit-testing only
   behave fully under real input. Recipe: `vite dev` + Playwright
   (`page.mouse` / `page.keyboard` go through Chromium's real input pipeline),
   `addInitScript` to disable the TV bridge (`window.webOS`) so the app uses
   the dev proxy, `page.route` to mock the data feed with deterministic items.
   Same engine family as the TV, fully controllable, CI-able.

5. **Deterministic logic gets a deterministic test.** Player decision logic
   (what plays next: playlist vs multi-part), format math, parsers — extract
   and run them in plain Node with real captured data. On-device playback runs
   are flaky and slow; save them for what only a device can tell you.

---

## 4. Assorted verification gotchas

- `Page.captureScreenshot` shows white/black where video (and some GPU-composited
  surfaces) render — assert via DOM/`<video>.currentTime`, not pixels.
- Prefilled-GitHub-issue URLs (a zero-server "scan to report" channel): keep the
  report **ASCII-only** — a percent-encoded CJK character is 9 bytes and can make
  a QR too dense to scan off a TV screen. Verify by decoding the QR from a real
  device screenshot (jsQR) and opening the decoded URL.
- `luna-send` (private bus) is permission-denied for the dev-mode user on TV;
  `luna-send-pub` works — use it for launch/relaunch in scripts.
- Launching an already-running app only foregrounds it (state persists). For a
  clean-state test, close it first or reload via CDP `Page.reload`.
- Long-lived test scripts must use ephemeral local ports for SSH tunnels — a
  fixed port + a lingering process = every later run dies confusingly.
