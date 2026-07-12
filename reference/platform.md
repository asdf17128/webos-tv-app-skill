# webOS TV platform APIs & engineering facts

Researched 2026-07 from webostv.developer.lge.com, LG staff forum answers, and battle-tested
community sources (jellyfin-webos, webosbrew, youtube-webos). Official unless tagged
(community). This is the "what the platform actually lets a third-party web app do" sheet.

## Engine matrix (official)

| webOS TV | Year | Web engine | Node (JS services) |
|---|---|---|---|
| 3.x | 2016–17 | Chromium 38 | v0.12.2 |
| 4.x | 2018–19 | Chromium 53 | v0.12.2 |
| 5.x | 2020 | Chromium 68 | v8.12.0 |
| 6.x | 2021 | Chromium 79 | v8.12.0 |
| 22 | 2022 | Chromium 87 | v12.21.0 |
| 23 | 2023 | Chromium 94 | v12.22.2 |
| 24 | 2024 | Chromium 108 | v16.19.1 |
| 25 | 2025 | Chromium 120 | v16.20.2 |
| 26 | 2026 | Chromium 132 | v20.12.2 |

Chromium never bumps within a TV generation. **LG upgrades older sets across webOS
generations** (a 2022 C2 may run webOS 24 after a firmware update) — detect at runtime via
`webOS.deviceInfo` `sdkVersion`, never assume from model year. For 4.0+ just check caniuse
against the Chromium version. JS services: pure-JS modules only, no C/C++ addons.

## webOSTV.js (v1.2.13, portal download — no npm)

- `webOS.service.request(uri, {method, parameters, subscribe, onSuccess, onFailure})` —
  the only Luna bridge for web apps.
- `webOS.deviceInfo(cb)` — async; `modelName`, `version`, `sdkVersion`, `screenWidth/Height`,
  **`uhd`, `uhd8K`, `oled`, `hdr10`, `dolbyVision`, `dolbyAtmos`** — the canonical runtime
  capability sniff (HDR/4K decisions belong here, not in appinfo).
- `webOS.systemInfo` — synchronous property `{country, smartServiceCountry, timezone}`.
- `webOS.platformBack()` — webOS 6.0+ shows the system exit popup; ≤5.0 jumps to Home.
- `webOS.keyboard.isShowing()` + document event `keyboardStateChange` (`e.detail.visibility`).
- `webOSDev.LGUDID({onSuccess})` (3.0+) per-device ID; `webOSDev.launchParams()`;
  `webOSDev.connection.getStatus({subscribe: true})`.

## Luna services open to third-party web apps (webOS 22–25)

Documented-callable only: activitymanager, applicationManager (`launch`,
`getForegroundAppInfo` — **`listApps` is denied**), audio (volume get/set/subscribe only),
connectionmanager, db8 (`com.palm.db`), mediadb, DRM, systemservice (time), 
tv.systemproperty (`getSystemInfo {keys:[modelName, firmwareVersion, sdkVersion, UHD]}`),
settingsservice (read-only subset: `option`/`caption`/`localeInfo`), Magic Remote sensor
(full webOS 24+), BLE GATT (24+), Keymanager3 crypto (24+).

**There is no `requiredPermissions` for TV web apps** (that's webOS OSE / JS services).
Permissions are fixed at install; anything undocumented returns "Denied method call" —
you cannot request more. Settings writes, audio routing, notifications: effectively locked.

## Lifecycle

- States: Launched (foreground) → Suspended (hidden, JS/timers/rAF paused "after a short
  time", undocumented delay). **Persist state in the `visibilitychange`-hidden handler** —
  there is no beforeunload guarantee, and suspended apps are evicted on memory pressure
  without notice. The OS will even kill the *foreground* app if media buffers balloon
  ("This app will now restart to free up memory") (community: jellyfin-webos #292).
- `webOSLaunch` / `webOSRelaunch` on `document` (register with capture=true), params in
  `e.detail`. `handlesRelaunch: true` = you stay background until you call
  `webOSSystem.activate()` (≤4.x: `PalmSystem.activate()`).
- Deep links: `ares-launch com.app -p '{"contentTarget":"..."}'` — `contentTarget` is the
  de-facto key Home/Content Store integrations pass (community: youtube-webos).
- `requiredMemory` (appinfo) is a launcher hint that may pre-kill other apps, not a quota.

## Storage — the part everyone gets wrong

- **localStorage: 16 MB cap (3.5+), survives reboot, but is DELETED when the user updates
  or removes a packaged app** (LG staff, forum #10068). Not update-safe.
- **Packaged apps have no cookies at all** (official web-engine spec). Token auth →
  localStorage/db8, never cookie sessions.
- IndexedDB: per Chromium level; same wiped-on-update fate.
- **db8 (`luna://com.palm.db`) is the update-survivable store**: `putKind {id:"com.your.app:1",
  owner:"com.your.app", private:true}` then `put/find/merge/del/watch`; `private:true`
  auto-cleans on uninstall; optional `sync:true` backs up to LG servers.
- No filesystem write API; read your own package via `webOS.fetchAppRootPath()`.

## Media element specifics

- **One media pipeline, period** (official FAQ + empirically confirmed on a C4 / webOS 24,
  2026-07): a second `<video>`/`<audio>` play() blacks out the first. Test result: 4 native
  `<video>` elements loaded real HLS, all reached readyState 4 / videoWidth set, but **only
  the last-played one advanced currentTime** — the others froze. MSE is no escape: it feeds
  the *same* single hardware decoder (a second hls.js stream steals it too). So there is no
  "1 native + N MSE" concurrency. Channel zapping = src-swap on one element + `video.load()`
  after changing src (needed on webOS to actually release/rebind the pipeline).
- **No multi-view / split-screen / PiP for a web app.** LG's consumer Multi-View is real
  (service `com.webos.service.multiviewcontroller`, methods `launchApps`/`changeMode`/
  `switchAudio`/…) but its LS2 client-permissions allowlist is **LG system components only**
  (`com.webos.app.multiviewsettings`, `/usr/bin/mvclient`, `mvwatcherclient`). A dev/third-party
  app's calls return nothing (ACG-denied); the system partition is read-only so you can't add
  yourself; and a patched TV can't be rooted to reach `com.webos.app.multiview`. W3C PiP just
  relocates the one pipeline. **A "1 big live + N small" multi-view in a web app can only be:
  1 decoded main + side tiles as logo/EPG info cards** (swap a card into the main to make it
  the live one).
- **`canvas.drawImage(video)` reads the hardware video plane as BLACK** (C4, verified: avgLum 0
  while the video decodes at 1280×720). Same reason CDP screenshots are black. So you cannot
  snapshot/mirror the live video into a canvas — the side tiles above genuinely can't show any
  video frame (not even a frozen one). Plan multi-channel UIs around info cards, not thumbnails.
- **mediaOption** (native pipeline hints, escaped into the source type):
  ```js
  var o = { option: { mediaTransportType: "HLS",
    transmission: { playTime: { start: 30000 } },        // quick-resume at 30s
    adaptiveStreaming: { maxWidth: 3840, maxHeight: 2160, bps: { start: 4000000 } } } };
  source.setAttribute('type', 'video/mp4;mediaOption=' + encodeURI(JSON.stringify(o)));
  ```
  **Pitfall: native-HLS ABR defaults cap at 1920×1080 — set maxWidth/maxHeight for UHD.**
  The Simulator ignores mediaOption entirely.
- App graphics plane is 1080p max (appinfo `resolution`: only 1920x1080/1280x720); the
  video plane still decodes 4K/HDR. HDR is signaled in-stream; gate features on
  `webOS.deviceInfo` capability flags.
- Native HLS protocol version: v3 (≤3.5), v5 (4.x), v7 (5.0+). DASH: never native — MSE only.
- MSE (community, jellyfin): buffer accounting is time-based not byte-based — high-bitrate
  streams balloon memory; keep SourceBuffer windows short, `remove()` aggressively, handle
  `QuotaExceededError` by trimming, or the OS kills the app.
- Subtitles: WebVTT only.

## Networking

- **CORS is enforced; there is no documented file:// exemption — do not rely on one.**
  Packaged apps send no usable Origin ("they do not have a CORS origin" — LG staff 2026-06),
  so plain GETs work against servers that answer `Access-Control-Allow-Origin: *` and fail
  against origin-echo/credentialed setups. The robust patterns remain: local-proxy service
  (see service-and-proxy.md) or a hosted app.
- **TLS: self-signed certs fail SILENTLY for XHR/WSS/subresources** (no interstitial outside
  navigation). Let's Encrypt trusted from webOS 5.0+ only. LAN backends: plain HTTP or a
  real cert.
- Network state: `luna://com.palm.connectionmanager` `getStatus {subscribe:true}` →
  `{isInternetConnectionAvailable, wired{...}, wifi{ssid, onInternet}}`; trust it over
  `navigator.onLine`.

## Input

- Virtual keyboard auto-shows on input focus and **cannot be suppressed**; layout follows
  input `type` (text/number/email/url/password); track via `keyboardStateChange`. Its mic
  button gives you system dictation for free —
- — because **there is no third-party mic/voice API** on webOS TV (no getUserMedia audio,
  no Web Speech backing; LG staff confirmed no SDK path).
- Gamepad: standard W3C API works; `cloudgame_active: true` (webOS 23+) for direct HID.
- USB keyboards/mice emit normal DOM events (arrows/enter/back map to remote codes).
