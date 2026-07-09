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

## 5. Official dev loop & device matrix [2026-07]

Researched from the current SDK docs (webostv.developer.lge.com) — the tooling moved a
lot in 2024–26; the old "webOS TV CLI" and VS Code extension are deprecated.

**The fast dev loop — hosted mode (no repackaging):**

```bash
ares-launch --hosted ./appDir -d tv     # serves the dir from your machine, TV runs it
ares-launch -H ./appDir -I 192.168.x.x  # -I when multi-NIC autodetect picks wrong
ares-server ./appDir --open             # plain local server (port 7496)
```

Hosted mode auto-reloads the TV on every file save (CLI ≥1.12); a gitignore-style
`.reloadignore` filters which saves trigger it. Caveat: apps with JS services can't run
hosted; hosted apps in background get killed like any other. This beats the
package→install→launch cycle for UI iteration; still do a packaged install before sign-off
(hosted origin ≠ packaged file:// — storage and CORS behave differently).

Verified on real hardware (webOS 24, 2026-07):
- **Multi-NIC machines (VPN/Clash virtual interfaces) silently break autodetect** — the
  stub loads a dead IP and the TV sits on a blank stub forever. Always pass `-I <lan-ip>`.
- The mechanism is a stub app (`com.sdk.ares.hostedapp`) containing an **iframe** pointed
  at `http://<host-ip>:<random-port>/`. Consequences: your app's origin is that http URL —
  **localStorage/IndexedDB are separate from the packaged install** (fresh state every
  hosted session), and CORS applies as a normal http origin (packaged apps send no Origin).
  Don't sign off origin-sensitive behavior from hosted mode.
- CDP debugging: inspect `com.sdk.ares.hostedapp`, then evaluate in the **http execution
  context** (pick it from `Runtime.executionContextCreated` by origin) — the default
  context is the file:// stub where your app's globals don't exist.
- Hot reload verified: `touch` any file under the app dir → TV reloads in ~2–5s.

**Simulator (webOS 6.0, 22–26) — `ares-launch -s 24 ./appDir`:**

- Per-version desktop builds; macOS ARM64 native from webOS 22 on (the webOS 26 simulator
  is ARM64-*only*). Install via webOS Studio's Package Manager, via
  `LG_WEBOS_TV_SDK_HOME`, or point the CLI directly: `ares-launch -s 24 -sp <dir> <appDir>`.
- **Scripted download without webOS Studio** (endpoints mined from webos-studio source,
  verified 2026-07): `https://developer.lge.com/resource/tv/RetrieveToolLastVersion.dev?resourceId=RS00007585`
  lists every simulator zip with a `fileId`; then
  `.../RetrieveToolDownloadUrl.dev?fileId=<id>` returns a `gftsUrl` direct link
  (e.g. RF00020813 = webOS_TV_24 mac-arm64, 105 MB). The naive
  `RetrieveToolDownload.dev?fileName=...` URL returns an HTML error page — `file` the
  download before unzipping.
- **Not automatable**: launching the (Electron) simulator with `--remote-debugging-port`
  opens the port and prints a browser ws URL, but the DevTools HTTP/WS endpoints never
  answer — puppeteer can't attach. Treat the Simulator as a manual/visual tool; automated
  runs stay on desktop-Chromium harnesses + real-TV CDP.
- Runs the TV-matching Chromium, simulates Luna/webOSSystem (with gaps), full RCU
  including color keys with real key codes, auto-opens Web Inspector, auto-reloads.
- **Can't do: DRM, mediaOption, real A/V decode specs** — playback validation stays on
  hardware. Community warning: apps that pass the 6.0 Simulator have broken on the 6.0
  Emulator — simulator Chromium ≠ TV build; treat it as a fast UI/API harness, not proof.
- Coverage strategy by version: ≤6.x → x86 VirtualBox Emulator (or the desktop-Chromium +
  docker-Node substitutes in §2); 6.0/22+ → Simulator; always → real TV for media/perf.

**Web Inspector version-matching:** the TV's DevTools endpoint is old — LG says drive it
with Chrome v38 (webOS 3.x–5.0), v68 (6.0–24), latest (25+). Modern-Chrome frontends
half-work against old targets; raw CDP over WebSocket (debug-toolchain.md) sidesteps this.

**Perf from the CLI:** the command is **`ares-device`** in CLI v3 (`ares-device-info`
still exists as a stub but outputs nothing — a silent trap). Verified:
`ares-device -d tv -r -l -t 1 -s out.csv` → per-app `TIME,PID,ID,CPU(%),MEMORY(%),MEMORY(KB)`
rows every second; `-r -id com.your.app` filters one app. GUI: Resource Monitor
(webOS 6.0+, replaces Beanviser). LG expects a perf pass before store submission.

**Real-device farms / staged rollout:** Seller Lounge has a **Cloud Test Lab** pre-test and
**Alpha Test** (invisible publish to registered TVs) — both seller-account-gated. Third
party: Suitest (real LG TV lab + virtual remote), `appium-lg-webos-driver` (Appium over
ares+CDP). Store QA: pretest + function + content test, **every update re-enters full
review**, community-reported turnaround runs to weeks — batch your releases.

**CLI v3 notes (@webos-tools/cli):** unified multi-profile tool (`ares-config --profile tv`);
default device creds now `prisoner`@port `9922`; `--remainPlainIPK` and `--app-exclude`
on ares-package (the latter replaces hand-rolled `-e` exclude lists). Node ≥16.20 required —
the "official CLI only supports Node 14–16" era is over; if a modern Node still throws
util.isDate errors you are on the old `@webosose/ares-cli`, switch packages.
