# 10-foot UI design for webOS TV apps

Distilled from [Apple tvOS HIG](https://developer.apple.com/design/human-interface-guidelines/designing-for-tvos)
(typography ≥29pt body / ≥48pt titles, 60/80px safe areas, unmistakable focus) and
[LG webOS TV Design Principles](https://webostv.developer.lge.com/develop/guides/design-principles)
(≥20px comfortable reading, high contrast, card layout, restraint), then hardened by
shipping to real living rooms. The numbers below are the pragmatic engineering scale we
landed on for a dense, information-rich app (a video platform client); adjust upward for
sparser apps, never downward.

## 1. Typography (at 1080p fullscreen)

The couch is ~3m away. Multiply phone sizes by ~2.5–3x. Our working scale:

| Tier | Size | Use |
|---|---|---|
| Display | 30–32 | Page titles, player title (28–30) |
| Title | 22–24 | Card titles, featured-item titles |
| Body | 20 | Buttons, menu items, tabs, hints, time readouts |
| Caption | **18 — absolute floor for anything the user must read** | Secondary info, metadata |
| Badge | 16 (glanceable elements ONLY) | Duration badges, status chips; never sentences |

**Nothing visible below 16px. Ever.** When in doubt, test on the actual TV from the couch —
a dev monitor at 60cm lies to you by a factor of ~4.

Real-world calibration: our first "tasteful" design pass used 14–15px hints — the owner's
immediate reaction was "怎么给沙发用户看?" (how is a couch user supposed to read this?).
Both directions fail: oversized loud chrome reads as ugly, undersized reads as invisible.

## 2. Color

- Backgrounds: deep, low-luminance (e.g. `#0a0d1a` page / `#101425` cards / `#0d1020` overlays)
- Text: primary `#f0f0f0` · secondary `#9aa0a8` · muted `#8a8f98` (captions only, never key info)
- One accent color for focus/selection/links; a soft red for destructive/danger
- Secondary text vs background contrast ≥ 4.5:1
- Avoid large pure-white areas — TVs are bright and viewed in dark rooms

## 3. Focus (the make-or-break of TV UX)

- A standard, unmistakable focus treatment everywhere — e.g. `outline: 4px solid <accent>`
  plus a glow when needed. tvOS uses scale (1.05–1.1x) + elevation; either works, but it
  must be instantly readable from 3m.
- **There is always exactly one focused element on screen.** A screen where focus can
  vanish (modal islands, overlays that trap the D-pad) is broken — keep every region
  reachable by arrows: grid ↕ tabs ↕ control bar, never a focus island.
- Animate only `transform`/`opacity` (GPU), 150–250ms.

## 4. Layout & safe area

- Safe area: keep critical content ≥60px from top/bottom, ≥80px from sides
  (tvOS standard; older TVs overscan).
- 8px base spacing grid; 8–12px card radius; ~24px grid gaps.
- **Floating UI (bubbles/popovers) must be mounted at the root**, not inside a scrolling
  container — `overflow` on an ancestor silently clips anything extending past it, and
  `getBoundingClientRect` will NOT reveal the clipping (it reports layout, not visibility).
  Screenshot-verify floating UI; a rect check once "passed" a preview bubble that was in
  fact three-quarters invisible.

## 5. Restraint (webOS official principle: fewer, simpler)

- One primary action per screen; drop decorative chrome (borders/badges/glows) by default.
- Progress/countdown affordances: a thin filling line with linear easing beats big numbers
  and colored circles. (A 64px countdown circle on a video cover was vetoed on sight.)
- Overlay hint lines: `primary info · secondary action · secondary action` at Body size in
  secondary color.

## 6. Old-device compatibility floor

webOS 5 = Chromium 68, webOS 6 = Chromium 79. CSS/JS that silently breaks there:

- **`aspect-ratio` (Chrome 88+)** — covers collapse to zero height. Use the padding-top
  ratio hack — and remember the child then needs `position: absolute; inset: 0`, or images
  render 0-height (this exact miss shipped a black-covers regression).
- `globalThis` needs a polyfill on 68; check caniuse for anything newer than 2019 before use.

## 7. Enforce the spec mechanically

A spec nobody enforces decays. Cheap static gates in the release pipeline keep it honest:

```bash
# no sub-16px inline text
grep -rn "fontSize: 1[0-5]\b" app/src --include="*.jsx" && exit 1
# no aspect-ratio CSS (Chromium 68/79 targets)
grep -rn "aspectRatio" app/src --include="*.jsx" && exit 1
```

Both of these greps exist because each rule was violated once, shipped, and caught by a
human. The gate makes the second occurrence impossible.
