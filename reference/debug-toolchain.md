# On-Device Debug Toolchain

> **Golden rule: debug-first.** Pull logs / metrics / network from the real TV *before* writing a fix. "It feels laggy" / "it froze" has a dozen possible causes; only data narrows it down. Blind code-and-redeploy loops are the slowest way to develop a TV app.

Everything here works the same way: **SSH tunnel into the TV → reach its Chrome DevTools Protocol endpoint at `127.0.0.1:9998` → drive it over a WebSocket.** webOS launches packaged web apps with `--remote-debugging-port=9998` automatically.

## Connection prerequisites

- TV in **Developer Mode** (LG Developer Mode app installed and active).
- SSH details from that app: host IP, port (commonly `9922`), user (`prisoner`), and a private key (`~/.ssh/tv_webos`) whose **passphrase is shown in the Developer Mode app**.
- The passphrase **rotates** when the dev session is renewed — pass it as a CLI arg, never hard-code it. On auth failure, re-read it and re-fetch the key (`ares-novacom --device tv --getkey`).
- The TV presents an `ssh-rsa` host key → set `algorithms: { serverHostKey: ['ssh-rsa'] }` in `ssh2`.
- Regular `ssh -L` often fails on the encrypted key; the `ssh2` Node library handles the passphrase cleanly, so all tools below use it.

## deploy.mjs — install over SSH (bypasses ares-cli)

The official `@webos-tools/cli` only supports Node 14–16 and throws on modern Node (`isDate is not a function`). Do it directly:

```js
import { Client } from 'ssh2';
import { readFileSync, readdirSync } from 'fs';

const TV = { host: '192.168.x.x', port: 9922, user: 'prisoner' };
const KEY = process.env.HOME + '/.ssh/tv_webos';
const APP_ID = 'com.you.app';
const REMOTE = '/media/developer/temp/';
const ipk = 'app/dist/' + readdirSync('app/dist').filter(f => f.endsWith('.ipk')).sort().pop();

const exec = (c, cmd) => new Promise((res, rej) => c.exec(cmd, (e, s) => {
  if (e) return rej(e); let o = '', err = '';
  s.on('data', d => o += d); s.stderr.on('data', d => err += d);
  s.on('close', code => res({ stdout: o, stderr: err, code }));
}));

const conn = new Client();
conn.on('ready', async () => {
  // 1. sftp the ipk up
  await new Promise(r => conn.sftp((e, sftp) => sftp.fastPut(ipk, REMOTE + 'app.ipk', r)));
  // 2. install via luna-send
  await exec(conn, `luna-send-pub -n 6 -f luna://com.webos.appInstallService/dev/install '{"id":"${APP_ID}","ipkUrl":"${REMOTE}app.ipk","subscribe":true}'`);
  await new Promise(r => setTimeout(r, 3000));
  // 3. launch
  await exec(conn, `luna-send-pub -n 1 -f luna://com.webos.service.applicationmanager/launch '{"id":"${APP_ID}"}'`);
  conn.end();
});
conn.connect({
  host: TV.host, port: TV.port, username: TV.user,
  privateKey: readFileSync(KEY), passphrase: process.argv[2] || '',
  algorithms: { serverHostKey: ['ssh-rsa'] },
});
```

`build.sh` ties it together: `vite build` → `cp webos-meta/* dist/` → `ares-package --no-minify . ../service/...` → `node tools/deploy.mjs "$PASS"`.

## The tunnel + CDP pattern (shared by every debug tool)

```js
import { Client } from 'ssh2';
import { readFileSync } from 'fs';
import http from 'http'; import net from 'net'; import { WebSocket } from 'ws';

const conn = new Client();
conn.on('ready', () => {
  // forward a local port to the TV's 127.0.0.1:9998
  const server = net.createServer(s =>
    conn.forwardOut('127.0.0.1', 0, '127.0.0.1', 9998, (e, rs) => e ? s.end() : s.pipe(rs).pipe(s)));
  server.listen(19998, '127.0.0.1', () => {
    // ask DevTools for the page list, find our app, connect its WS
    http.get('http://127.0.0.1:19998/json', res => {
      let d = ''; res.on('data', c => d += c);
      res.on('end', () => {
        const app = JSON.parse(d).find(p => p.url?.includes('com.you.app') || p.title?.includes('Your App'));
        const ws = new WebSocket(app.webSocketDebuggerUrl.replace(/127\.0\.0\.1:\d+/, '127.0.0.1:19998'));
        // ... send CDP commands ...
      });
    });
  });
});
conn.connect({ host, port, username, privateKey: readFileSync(KEY),
  passphrase: process.argv[2], algorithms: { serverHostKey: ['ssh-rsa'] } });
```

CDP domains used:

- `Console.enable` — `Console.messageAdded` events (your `console.log/warn/error`). Note: it **flushes recent history** on enable, so old lines reappear; only lines after a fresh action are current.
- `Runtime.enable` — `Runtime.exceptionThrown` (uncaught errors).
- `Runtime.evaluate` — run diagnostic JS in the live app.
- `Performance.enable` + `Performance.getMetrics` — `JSHeapUsedSize`, `Nodes`, `LayoutCount`, `RecalcStyleCount`, `TaskDuration`.
- `Network.enable` — request/response events (but see the HLS caveat below).
- `Page.captureScreenshot` — PNG of the page (but see the video-plane caveat).
- `Input.dispatchKeyEvent` — inject remote-control keys (self-drive).

## debug.mjs — health snapshot

Enable Console + Runtime + Performance, wait a couple seconds, then evaluate a DOM-counts expression and dump metrics. This is the first thing to run when anything seems off.

```js
ws.send(JSON.stringify({ id: 100, method: 'Runtime.evaluate', params: { expression: `
  JSON.stringify({
    focusRegistry: document.querySelectorAll('[data-focus-id]').length,
    domNodes: document.querySelectorAll('*').length,
    images: document.querySelectorAll('img').length,
  })` }}));
// later: { id, method: 'Performance.getMetrics' } and filter the metric names you care about
```

## watch.mjs — long playback monitor

Stream console + exceptions for N seconds, **and poll `<video>` state every 5s** so you see exactly when playback freezes. Flag stall keywords (`payload length`, `shaka error`, `watchdog`, `stall`, `buffer`).

```js
setInterval(() => ws.send(JSON.stringify({ id: 200, method: 'Runtime.evaluate', params: { expression:
  `(function(){var v=document.querySelector('video');return v?JSON.stringify({t:+v.currentTime.toFixed(1),paused:v.paused,ended:v.ended,ready:v.readyState,buffered:v.buffered.length?+v.buffered.end(v.buffered.length-1).toFixed(1):0}):'no-video';})()` }})), 5000);
```

## netcap.mjs — per-request timing via the proxy

Enable `Network` and watch `requestWillBeSent` / `loadingFinished` / `loadingFailed`. Filter to `/proxy/` URLs, parse the upstream host out of the path, and log `done <ms> <KB> <host> <Range>` (or `FAILED … errorText`). This is how you discover *which* CDN host is slow/failing and how long each segment takes.

```js
if (msg.method === 'Network.requestWillBeSent' && msg.params.request.url.includes('/proxy/')) {
  const host = msg.params.request.url.match(/\/proxy\/([^/]+)/)?.[1];
  reqHost[msg.params.requestId] = host;
  reqStart[msg.params.requestId] = msg.params.timestamp * 1000;
}
if (msg.method === 'Network.loadingFinished') {
  const dur = Math.round(msg.params.timestamp * 1000 - reqStart[msg.params.requestId]);
  console.log(`done ${dur}ms ${Math.round(msg.params.encodedDataLength/1024)}KB ${reqHost[msg.params.requestId]}`);
}
```

## drive.mjs — self-driving + screenshot (no human needed)

Inject D-pad keys with `Input.dispatchKeyEvent`, let the UI settle, read state, then screenshot. This lets an agent operate the app and verify a change end-to-end.

```js
const KEYMAP = {
  up:{key:'ArrowUp',code:'ArrowUp',vk:38}, down:{key:'ArrowDown',code:'ArrowDown',vk:40},
  left:{key:'ArrowLeft',code:'ArrowLeft',vk:37}, right:{key:'ArrowRight',code:'ArrowRight',vk:39},
  ok:{key:'Enter',code:'Enter',vk:13}, back:{key:'Backspace',code:'Backspace',vk:8},
};
for (const k of keys) {
  const m = KEYMAP[k];
  await call('Input.dispatchKeyEvent', { type:'keyDown', key:m.key, code:m.code, windowsVirtualKeyCode:m.vk, nativeVirtualKeyCode:m.vk });
  await call('Input.dispatchKeyEvent', { type:'keyUp',   key:m.key, code:m.code, windowsVirtualKeyCode:m.vk, nativeVirtualKeyCode:m.vk });
  await sleep(450);
}
await sleep(1200);
// read state: focused id, video time, broken images
const diag = await call('Runtime.evaluate', { returnByValue: true, expression: `JSON.stringify({
  focus: document.querySelector('[data-focus-id].focused')?.getAttribute('data-focus-id') || null,
  v: (function(){var v=document.querySelector('video');return v?{t:+v.currentTime.toFixed(1),ready:v.readyState,paused:v.paused}:null;})(),
  brokenImgs: Array.from(document.querySelectorAll('img')).filter(i=>i.complete&&i.naturalWidth===0).length,
})` });
const shot = await call('Page.captureScreenshot', { format: 'png' });
```

Usage: `node tools/drive.mjs "down,down,right,ok" out.png`. Map a `home` token to several `left` presses to return to the sidebar.

## sh.mjs — arbitrary shell on the TV

```js
conn.exec(process.argv[2], (e, s) => { s.on('data', d => process.stdout.write(d)); s.on('close', () => conn.end()); });
```

`node tools/sh.mjs "journalctl -n 100"` etc. **Note:** a JS service's `console.log` does **not** reach `journald` — capture service-side logs over CDP from the app, or log to a file the proxy can serve.

## verify.sh — the full closed loop

`build → deploy → sleep → CDP-assert (DOM/focus/sidebar present) → screenshot`. Run this as the end-to-end gate after a change. It asserts the app actually rendered (`focus > 0 && hasSidebar`) and drops a `verify-screenshot.png`.

## Two CDP limitations to internalize

1. **`Page.captureScreenshot` can't capture the hardware video plane.** Decoded video lives on a separate overlay below the web layer, so screenshots show white/black where the video is. **Verify playback by polling `<video>.currentTime` advancing**, not by looking at a screenshot.
2. **Native HLS playback bypasses the CDP Network domain.** When the TV uses its native media pipeline for HLS, segment fetches happen below Chromium and never surface as `Network.*` events. **Route HLS through your local proxy** and read timing from the proxy logs (`netcap.mjs`) instead.

---

## Media / streaming gotchas (diagnosed with the tools above)

- **Chromium on TV plays neither DASH nor HLS natively.** DASH → an MSE player (Shaka Player). HLS → through the local proxy into a `<video>`-friendly stream.
- **No exponential backoff in the player's retry config.** `baseDelay:500, backoffFactor:2, maxAttempts:5` makes every load that hits one transient failure wait ~7.5s (500+1000+2000+4000) in the manifest phase. Use flat short delays: `backoffFactor:1, baseDelay:150–200`. Cut loads from 5–9s to <1s. (A Shaka `onstatechange` listener logging `e.state` + elapsed ms reveals per-phase timing: `manifest-parser` / `manifest` / `drm-engine` / `load`.)

  ```js
  player.configure({
    abr: { enabled: false },                                   // you have a manual quality selector
    manifest:  { retryParameters: { maxAttempts: 4, baseDelay: 150, backoffFactor: 1, timeout: 15000 } },
    streaming: { retryParameters: { maxAttempts: 6, baseDelay: 200, backoffFactor: 1, fuzzFactor: 0.5, timeout: 20000 } },
  });
  ```

- **Prefer the stable origin CDN over flaky P2P/PCDN nodes.** A manifest's primary `baseUrl` is often a fast-but-unreliable PCDN/P2P node that returns short/bad data on range requests → "Payload length does not match range requested bytes" → fatal error / black load. When you build the DASH MPD, emit primary **+ all backup URLs** as multiple `<BaseURL>` and **order the stable origin CDN (e.g. `*.examplecdn.com:443`) first**, PCDN last, so the player prefers reliable hosts and only falls back when needed. Origin throughput is fine; origin is reliable, PCDN is fast-but-broken.

  ```js
  // inside buildMPD: collect [primary, ...backupUrl], sort origin-first, dedup
  const urls = [rep.baseUrl, ...(rep.backupUrl || [])].filter(Boolean);
  urls.sort((a, b) => isOrigin(b) - isOrigin(a));   // origin first
  const baseUrls = urls.map(u => `<BaseURL>${escapeXml(u)}</BaseURL>`).join('\n');
  ```

- **Resume via `player.load(url, startTime)`, not load-at-0-then-seek** (the latter double-buffers).
- **Stall watchdog:** if playback hasn't advanced for >8s, nudge `currentTime` to recover from soft freezes.
- For a manual quality selector, the MPD only needs the single highest-bitrate video+audio rep; re-fetch the play URL on quality change rather than packing every representation into one manifest.
