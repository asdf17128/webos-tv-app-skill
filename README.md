# webos-tv-app — a Claude Code Skill

A practical, code-first methodology for building **React apps for LG webOS Smart TVs** (e.g. the 2024 LG C4 — webOS 24, Chromium 108, Node.js v16, D-pad remote).

It is distilled from a real, shipped webOS TV app and captures the parts that aren't in any official doc:

- **Dual-channel architecture** — a Luna JS service for JSON/CORS/cookies, plus a local `127.0.0.1` HTTP proxy for `<video>`/`<img>`/HLS that can't ride the Luna bus.
- **Zero-render D-pad focus** — move focus by mutating the DOM directly (O(1) `{group}-{row}-{col}` navigation), because a `setState` per arrow press is too slow on TV hardware.
- **TV performance rules** — transform-based GPU scrolling, transform/opacity-only animations, `content-visibility`, `React.memo`, keep pages mounted, `chrome108` build target.
- **Build / package / deploy** — `vite build` → `ares-package` → install over SSH (works around the `@webos-tools/cli` Node-version breakage), plus the Developer-Mode passphrase rotation gotcha.
- **On-device debugging** — Chrome DevTools Protocol over an SSH tunnel: console, exceptions, performance metrics, network timing through the proxy, and **self-driving** the app with injected key events + screenshots so an agent can operate and verify without a human.
- **Media pitfalls** — DASH needs an MSE player (Shaka), flat retry delays instead of exponential backoff, prefer stable origin CDN over flaky PCDN, resume via `load(startTime)`.

## Layout

```
webos-tv-skill/
├── SKILL.md                          # the skimmable guide (start here)
├── reference/
│   ├── service-and-proxy.md          # Luna service + local proxy, full skeleton
│   ├── focus-system.md               # zero-render D-pad focus engine
│   ├── performance.md                # the TV performance rule set
│   └── debug-toolchain.md            # CDP-over-SSH debugging, deploy, streaming
├── README.md
└── LICENSE
```

`SKILL.md` is the entry point and stays short; depth lives in `reference/`.

## Install as a Claude Code skill

**Personal (all your projects):** drop the folder into your skills directory.

```bash
cp -r webos-tv-skill ~/.claude/skills/webos-tv-app
```

**Per-project (checked into a repo):**

```bash
cp -r webos-tv-skill /path/to/your/project/.claude/skills/webos-tv-app
```

**As part of a plugin:** place this directory under your plugin's `skills/` folder. The skill is identified by the `name:` in `SKILL.md`'s YAML frontmatter (`webos-tv-app`).

Claude Code auto-discovers skills in these locations. Once installed, Claude will reach for it whenever you ask it to build, debug, or optimize a webOS TV app — or you can invoke it explicitly. The skill body links out to the `reference/` files, which Claude reads on demand.

## Who this is for

各位开发者 building 10-foot, remote-driven apps on LG webOS. The patterns assume the webOS 24 constraint set (Chromium 108 web layer, Node.js v16 services, weak TV GPU/CPU). They generalize to any backend/CDN — the original project's app-specific bits have been factored out.

## Provenance

Distilled from a production Bilibili client for LG webOS TV and its accumulated debugging notes. The code patterns are real (focus engine, service/proxy, deploy and CDP tooling); only the service-specific endpoints and host names were generalized.

## License

MIT — see [LICENSE](./LICENSE).
