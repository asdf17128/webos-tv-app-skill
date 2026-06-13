# JS Service + Local HTTP Proxy

The background service is the heart of a webOS app: it bypasses CORS for API calls and serves media that browser elements can't otherwise reach. One Node.js process, two responsibilities.

> **Runtime:** webOS 24 ships Node.js v16. Use **CommonJS** (`require`), not ESM. The `webos-service` and `webOSTV.js` libraries are provided by the platform / bundled with your app.

## Packaging layout

```
service/com.you.app.service/
├── service.js        # the code below
├── services.json     # declares the service name
└── package.json      # { "name": "...", "main": "service.js", "version": "..." }
```

`services.json`:

```json
{
  "id": "com.you.app.service",
  "description": "API proxy service",
  "services": [{ "name": "com.you.app.service", "description": "API proxy service" }]
}
```

The service is packaged into the same `.ipk` as the web app (`ares-package . ../service/com.you.app.service`), so the user installs one app and never sees the split.

## Full service skeleton

```js
// service.js — Node.js v16, CommonJS only
var Service = require('webos-service');
var https = require('https');
var http = require('http');
var zlib = require('zlib');
var fs = require('fs');
var path = require('path');

var service = new Service('com.you.app.service');

// Reuse TLS connections to upstream hosts. A fresh TLS handshake per media
// segment is expensive on TV CPUs; keep-alive cuts initial-load + seek latency.
var keepAliveAgent = new https.Agent({ keepAlive: true, maxSockets: 8, keepAliveMsecs: 15000 });

// ---- Cookie persistence (packaged apps have no browser cookie jar) ----
var COOKIE_FILE = path.join('/media/internal', 'app_cookies.json');
var storedCookies = {};
try { if (fs.existsSync(COOKIE_FILE)) storedCookies = JSON.parse(fs.readFileSync(COOKIE_FILE, 'utf-8')); } catch (e) {}
function saveCookies() { try { fs.writeFileSync(COOKIE_FILE, JSON.stringify(storedCookies)); } catch (e) {} }
function serializeCookies(c) { return Object.keys(c).map(function (k) { return k + '=' + c[k]; }).join('; '); }

// ---- Host allowlist — both channels MUST gate on this ----
function isAllowedHost(host) {
  var allowed = ['api.example.com', 'passport.example.com'];
  for (var i = 0; i < allowed.length; i++) if (host === allowed[i]) return true;
  // suffix matches for CDN families
  return host.indexOf('.examplecdn.com') >= 0 || host.indexOf('.akamaized.net') >= 0;
}

// ---- One HTTPS helper for both channels ----
// forceIdentity: never accept gzip/deflate. REQUIRED on the binary/segment path,
// which pipes bytes raw without re-emitting Content-Encoding — compressed bytes
// there corrupt Range/Content-Length and break the media player's range reads.
function makeRequest(parsed, method, body, contentType, range, forceIdentity, cb) {
  var hostname = parsed.hostname;
  var port = parsed.port ? parseInt(parsed.port) : 443;
  var isCDN = hostname.indexOf('examplecdn') >= 0 || hostname.indexOf('akamaized') >= 0;

  var headers = {
    'User-Agent': 'Mozilla/5.0 ... Chrome/120.0.0.0 Safari/537.36',
    'Referer': 'https://www.example.com/',
    'Accept': isCDN ? '*/*' : 'application/json, text/plain, */*',
    'Accept-Encoding': (isCDN || forceIdentity) ? 'identity' : 'gzip, deflate',
    'Cookie': serializeCookies(storedCookies)
  };
  // Many CDNs 403 if you send an Origin header — only send it to the API.
  if (!isCDN) headers['Origin'] = 'https://www.example.com';
  if (contentType) headers['Content-Type'] = contentType;
  if (range) headers['Range'] = range;

  var options = {
    hostname: hostname, port: port,
    path: parsed.pathname + (parsed.search || ''),
    method: method || 'GET', headers: headers,
    rejectUnauthorized: false, agent: keepAliveAgent
  };

  var done = false;
  var req = https.request(options, function (res) {
    if (done) return; done = true;
    // Capture Set-Cookie so login persists.
    var sc = res.headers['set-cookie'];
    if (sc) {
      sc.forEach(function (line) {
        var kv = line.split(';')[0]; var i = kv.indexOf('=');
        if (i > 0) storedCookies[kv.substring(0, i).trim()] = kv.substring(i + 1).trim();
      });
      saveCookies();
    }
    cb(null, res);
  });

  // Without a timeout, a stuck upstream socket hangs forever and a media
  // segment never returns — playback freezes permanently. Bail so the
  // caller can fail the request and the player can retry.
  req.setTimeout(10000, function () { req.destroy(new Error('Upstream timeout')); });
  req.on('error', function (err) { if (done) return; done = true; cb(err); });
  if (body) req.write(body);
  req.end();
}

// ---- Decompress for the JSON channel. Note: some servers send raw deflate,
// not zlib-wrapped — try inflate, then inflateRaw as a fallback. ----
function decompress(res, cb) {
  var chunks = [];
  res.on('data', function (c) { chunks.push(c); });
  res.on('end', function () {
    var buf = Buffer.concat(chunks);
    var enc = res.headers['content-encoding'];
    if (enc === 'gzip') zlib.gunzip(buf, function (e, r) { cb(e ? buf : r); });
    else if (enc === 'deflate') zlib.inflate(buf, function (e, r) {
      if (!e) return cb(r);
      zlib.inflateRaw(buf, function (e2, r2) { cb(e2 ? buf : r2); });
    });
    else cb(buf);
  });
}

// ==================== Channel 1: Luna bus methods ====================
service.register('fetch', function (message) {
  var targetUrl = message.payload.url;
  var parsed;
  try { parsed = new URL(targetUrl); }
  catch (e) { return message.respond({ returnValue: false, error: 'Invalid URL' }); }
  if (!isAllowedHost(parsed.hostname))
    return message.respond({ returnValue: false, error: 'Host not allowed' });

  makeRequest(parsed, message.payload.method, message.payload.body,
    message.payload.contentType, message.payload.range, false, function (err, res) {
      if (err) return message.respond({ returnValue: false, error: err.message });
      decompress(res, function (data) {
        var ct = res.headers['content-type'] || '';
        if (/json|text|xml/.test(ct)) {
          message.respond({ returnValue: true, status: res.statusCode, contentType: ct,
            body: data.toString('utf-8'), newCookies: storedCookies });
        } else {
          message.respond({ returnValue: true, status: res.statusCode, contentType: ct,
            bodyBase64: data.toString('base64'), bodyLength: data.length });
        }
      });
    });
});

service.register('getCookies',   function (m) { m.respond({ returnValue: true, cookies: storedCookies }); });
service.register('setCookies',   function (m) {
  var c = m.payload.cookies || {}; Object.keys(c).forEach(function (k) { storedCookies[k] = c[k]; });
  saveCookies(); m.respond({ returnValue: true });
});
service.register('clearCookies', function (m) { storedCookies = {}; saveCookies(); m.respond({ returnValue: true }); });
service.register('ping',         function (m) {
  m.respond({ returnValue: true, status: 'ok', nodeVersion: process.version, localProxyPort: LOCAL_PROXY_PORT });
});

// ==================== Channel 2: Local HTTP proxy ====================
// For <video>, <img>, HLS — anything the browser must fetch by URL.
// URL shape: http://127.0.0.1:PORT/proxy/{host[:port]}/{path}
var LOCAL_PROXY_PORT = 7654;

var localServer = http.createServer(function (req, res) {
  if (!req.url.startsWith('/proxy/')) { res.writeHead(404); return res.end('Not found'); }
  var rest = req.url.slice('/proxy/'.length);
  var slash = rest.indexOf('/');
  var hostWithPort = slash > 0 ? rest.slice(0, slash) : rest;
  var apiPath = slash > 0 ? rest.slice(slash) : '/';
  var hostname = hostWithPort.split(':')[0];

  if (!isAllowedHost(hostname)) { res.writeHead(403); return res.end('Host not allowed'); }

  var parsed;
  try { parsed = new URL('https://' + hostWithPort + apiPath); }
  catch (e) { res.writeHead(400); return res.end('Bad URL'); }

  // forceIdentity=true here — pipe raw bytes, never decompress on this path.
  makeRequest(parsed, req.method, null, null, req.headers['range'], true, function (err, up) {
    if (err) { if (!res.headersSent) { res.writeHead(502); res.end(err.message); } return; }
    var h = {
      'Access-Control-Allow-Origin': '*',
      'Content-Type': up.headers['content-type'] || 'application/octet-stream'
    };
    if (up.headers['content-range'])  h['Content-Range']  = up.headers['content-range'];
    if (up.headers['content-length']) h['Content-Length'] = up.headers['content-length'];
    if (up.headers['accept-ranges'])  h['Accept-Ranges']  = up.headers['accept-ranges'];
    res.writeHead(up.statusCode, h);
    up.pipe(res);
    // A half-delivered segment is exactly what an MSE player chokes on — tear
    // down on upstream error; free the upstream socket if the client leaves.
    up.on('error', function () { res.destroy(); });
    res.on('close',  function () { up.destroy(); });
  });
});

localServer.on('error', function (err) {
  if (err.code === 'EADDRINUSE') { LOCAL_PROXY_PORT++; localServer.listen(LOCAL_PROXY_PORT, '127.0.0.1'); }
});
localServer.listen(LOCAL_PROXY_PORT, '127.0.0.1', function () {
  console.log('[Service] local proxy on ' + LOCAL_PROXY_PORT);
});

// A crash kills the service — keep it alive past unexpected errors.
process.on('uncaughtException', function (e) { console.error('uncaught', e && e.message); });

// ==================== Keep the background service alive ====================
var keepAlive;
service.activityManager.create('keepAlive', function (activity) { keepAlive = activity; });
```

## Why each defensive measure exists

| Measure | Without it |
|---|---|
| `activityManager` keepAlive | webOS reaps the idle service; the next Luna call has nothing to answer |
| `process.on('uncaughtException')` | one bad upstream response crashes the whole proxy |
| host allowlist (both channels) | your app becomes an open proxy for any host on the network |
| `forceIdentity` on the byte path | gzip corrupts Range/Content-Length → "Payload length does not match" → media error |
| keep-alive `https.Agent` | a TLS handshake per segment; visible stutter on weak TV CPUs |
| `req.setTimeout` + `destroy` | a stuck socket freezes playback forever with no recovery |
| `res.on('close') → up.destroy()` | leaked upstream sockets when the user seeks/leaves mid-segment |

## Dev parity

In browser dev you can't use the Luna bus or a TV localhost service. Run a tiny Node proxy on the Mac (e.g. `:9527`) exposing the **same two shapes** — an `/api?url=` endpoint and a `/proxy/{host}/{path}` endpoint — so the web app's `apiFetch`/`mediaUrl` helpers work unchanged in both environments. Detect the environment once (`typeof window.webOS !== 'undefined'`) and branch there only.
