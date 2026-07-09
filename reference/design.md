# 10-foot UI design for webOS TV apps

Distilled from [Apple tvOS HIG](https://developer.apple.com/design/human-interface-guidelines/designing-for-tvos)
(typography ≥29pt body / ≥48pt titles, 60/90px safe zone, unmistakable focus) and
[LG webOS TV Design Principles](https://webostv.developer.lge.com/develop/guides/design-principles)
(≥20px comfortable reading, high contrast, card layout, restraint), then hardened by
shipping to real living rooms. The numbers below are the pragmatic engineering scale we
landed on for a dense, information-rich app (a video platform client); adjust upward for
sparser apps, never downward.

A 2026-07 research pass through the archived official tvOS HIG pages and LG's own
Sandstone design system (the Enact component library that renders the webOS system UI —
its tokens are authored at 4K, ÷2 = 1080p px) corrected two folklore numbers and added
the platform-native focus spec; those findings are folded in below and tagged **[2026-07]**.

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

**[2026-07] Official floors, for calibration:** LG's Self-Checklist (QA recommendation tier)
wants **all text ≥20px at 1080p**; tvOS's smallest system text style is 23pt, body is
29pt Medium (tvOS biases *heavier weights* for distance). Sandstone's 1080p scale: body 30,
list-item 30, caption/label 24, button label 39, panel title 54 — and focused text bumps
weight 400→600. For a store submission, treat **20px as the floor** instead of 16, and copy
the weight bump on focus. Font family on LG TVs: `"LG Smart UI", "Museo Sans", sans-serif`
(LG Smart UI ships on the TV itself).

Real-world calibration: our first "tasteful" design pass used 14–15px hints — the owner's
immediate reaction was "怎么给沙发用户看?" (how is a couch user supposed to read this?).
Both directions fail: oversized loud chrome reads as ugly, undersized reads as invisible.

## 2. Color

- Backgrounds: deep, low-luminance (e.g. `#0a0d1a` page / `#101425` cards / `#0d1020` overlays)
- Text: primary `#f0f0f0` · secondary `#9aa0a8` · muted `#8a8f98` (captions only, never key info)
- One accent color for focus/selection/links; a soft red for destructive/danger
- Secondary text vs background contrast ≥ 4.5:1
- Avoid large pure-white areas — TVs are bright and viewed in dark rooms
- **[2026-07]** The system's own palette (Sandstone dark skin): bg `#000`, text `#e6e6e6`
  (note: **off-white, never pure #fff**), secondary `#abaeb3`, selected-but-unfocused
  `#3e454d`, scrim `rgba(0,0,0,0.6)`. Attribution correction: "avoid pure white" is TV-industry
  practice and matches LG's system palette, but it is NOT an Apple HIG rule — don't cite Apple for it.

## 3. Focus (the make-or-break of TV UX)

- A standard, unmistakable focus treatment everywhere — e.g. `outline: 4px solid <accent>`
  plus a glow when needed. tvOS uses scale (1.05–1.1x) + elevation; either works, but it
  must be instantly readable from 3m.
- **[2026-07] The platform-native treatment is a light fill, not an outline.** Sandstone
  spotlight: bg `#e6e6e6`, text flips dark `#4c5059`, weight 400→600, **200ms**, drop
  shadow `0 18px 18px rgba(0,0,0,0.3)` (1080p), buttons `scale(1.05)`, images `scale(1.2)`.
  Shipped in the IPTV app; reads unmistakably at 3m and frees the accent color to mean
  "playing / live now" instead of "where the cursor is". Prefer this for new apps.
- **[2026-07] Store-blocking:** LG Self-Checklist mandatory item — "all of the objects in
  your app MUST ALWAYS provide some sort of selection effect with NO EXCEPTION." Invisible
  focus is a rejection, not a style choice.
- **There is always exactly one focused element on screen.** A screen where focus can
  vanish (modal islands, overlays that trap the D-pad) is broken — keep every region
  reachable by arrows: grid ↕ tabs ↕ control bar, never a focus island.
- Animate only `transform`/`opacity` (GPU), 150–250ms; tvOS reserves ~100pt vertical gaps
  between unfocused rows so focus growth never overlaps.

## 4. Layout & safe area

- Safe area: keep critical content ≥60px from top/bottom, **≥90px from sides**
  ([2026-07] correction: the tvOS HIG says 90, not the commonly-quoted 80; older TVs
  overscan). LG's own hard minimum is smaller — **20px on every edge** at 1080p
  (official overscan guide) with Sandstone using a 24px app keep-out — so 60/90 for
  full-bleed overlays, ≥20–24px absolute floor for panel layouts.
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

## 7. LG Content Store UX mandatories [2026-07]

From the [Self-Checklist](https://webostv.developer.lge.com/distribute/app-self-checklist)
(submitting it is itself mandatory). Fail any of these = QA rejection:

1. **5-way operability** — every selectable element reachable and usable with
   arrows + OK + Back. Colored-key (R/G/Y/B) shortcuts are exempt from reachability,
   but whatever they open must itself be 5-way navigable.
2. **Pointer operability** — the same UI must also work with the Magic Remote
   cursor (hover moves focus, click selects).
3. **Back on the entry page leaves the app** — `webOS.platformBack()` (webOS 6+
   shows the system exit popup) or your own exit popup + `window.close()`. An app
   you can't exit with the remote is a rejection.
4. **Visible selection effect on every object** (see §3).

Recommended tier (QA flags, won't always reject): text ≥20px, buttons ≥75×75px,
a visible loading indicator (silent black screen at launch is a classic real-world
rejection), wheel scrolls lists, media overlays auto-hide ~5s (Sandstone default),
jump-seek 30s per press.

## 8. Enforce the spec mechanically

A spec nobody enforces decays. Cheap static gates in the release pipeline keep it honest:

```bash
# no sub-16px inline text
grep -rn "fontSize: 1[0-5]\b" app/src --include="*.jsx" && exit 1
# no aspect-ratio CSS (Chromium 68/79 targets)
grep -rn "aspectRatio" app/src --include="*.jsx" && exit 1
```

Both of these greps exist because each rule was violated once, shipped, and caught by a
human. The gate makes the second occurrence impossible.
