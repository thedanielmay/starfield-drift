# Mobile touch controls — design spec

**Date:** 2026-04-23
**Status:** Approved (after adversarial panel review)
**Scope:** Make `index.html` playable on iPhone Safari and equivalent touch browsers without regressing the desktop keyboard experience.

## Problem

The game at `index.html` is a single-file 985-line HTML5 canvas arcade shooter. Input is keyboard-only: arrow keys rotate the ship and apply thrust, space fires, `H` hyperjumps. Opened on iPhone Safari, there are no input affordances at all — the game is unplayable on mobile.

## Goal

Detect touch-first devices and present an on-screen control overlay that drives the existing game loop without changing gameplay, while leaving the desktop keyboard path unchanged.

## Non-goals

- Haptic feedback (iOS Safari ignores `navigator.vibrate()` in 2026).
- PWA install prompts, fullscreen API shims, orientation locking.
- Tilt / gyroscope controls.
- Screen-reader-accessible gameplay (graceful exclusion only).
- A first-run tutorial or control hints beyond the visible overlay.
- Reskinning or redesigning the arcade cabinet chrome for mobile — it is simply hidden in touch mode.

## Architecture

### Single-file integrity

All changes stay within `index.html`. Revised growth estimate: ~280 lines (+28%), taking the file to approximately 1265 lines. This is acceptable for now. If a subsequent feature pushes the file past 1400 lines, splitting into separate files becomes warranted — that decision is out of scope for this spec.

**Rollback:** single-file and no build step, so `git revert` of the implementation commit is the rollback path. No database migrations, no cache busts, no asset pipeline.

### State consolidation

All new touch-related state is grouped on one module-scoped object:

```js
const touch = {
  mode: 'keyboard',          // 'keyboard' | 'touch'
  overlay: null,             // root overlay element
  lastModeFlip: 0,           // timestamp; debounce mode changes
  joystick: {
    base: null,              // DOM ref
    knob: null,              // DOM ref
    activePointerId: null,
    originX: 0, originY: 0,  // anchor point of the current touch (CSS px, visualViewport-corrected)
    radius: 50,              // max knob travel in CSS px (matches base visual edge)
    deadzone: 8,             // centre dead-zone in CSS px (small enough for precise aim, large enough for thumb tremor)
    active: false,           // true when magnitude past deadzone — drives knob visual state
  },
  buttons: {
    thrust: null,            // DOM ref
    fire: null,              // DOM ref
    warp: null,              // DOM ref
    pause: null,             // DOM ref
    thrustPointerId: null,   // per-button pointer-capture tracking
    firePointerId: null,
    warpPointerId: null,
    fireIntervalId: null,    // cleared on pointerup AND on visibilitychange hide
    warpHoldTimerId: null,   // 100ms hold timer; cleared on pointerup/cancel
  },
};
```

No new loose globals. Existing module-scope names (`keys`, `state`, `AC`, `W`, `H`, `DPR`, `startBtn`, `againBtn`) are untouched.

### Input flow

Touch handlers mutate the existing `keys` object the game loop already polls at index.html:722–725. The game loop does not need to know input changed.

Each button tracks its own `pointerId` and calls `setPointerCapture` on `pointerdown`. This allows multi-touch — joystick + THRUST + FIRE + WARP can all be active simultaneously, each pointer routed to its originating element. When a `pointerup` or `pointercancel` fires, the handler compares `e.pointerId` against the stored id and only releases the matching button's state.

- Joystick horizontal component → `keys.ArrowLeft` / `keys.ArrowRight` when magnitude exceeds the deadzone. The joystick's vertical component is **ignored** — rotation is the only thing the stick does. The `joystick.active` flag is true while magnitude > deadzone, driving a visual state change on the knob (a subtle glow / colour shift) so users can see when input is registering vs still in dead-zone.
- THRUST button `pointerdown` → capture pointer, record `thrustPointerId`, `keys.ArrowUp = true`; matching `pointerup`/`pointercancel` → clear both.
- FIRE button `pointerdown` → capture pointer, record `firePointerId`, call `shoot()`, then `setInterval(shoot, 100)` and record `fireIntervalId` (100ms = 10 shots/sec, chosen to approximate typical macOS key-repeat without being faster than the keyboard path). Matching `pointerup`/`pointercancel` → `clearInterval(fireIntervalId)`, clear both id fields.
- WARP button `pointerdown` → capture pointer, record `warpPointerId`, start a **100ms** hold timer, and begin a 100ms fill animation on the button border. On matching `pointerup`/`pointercancel` before the timer elapses, clear the timer, cancel the animation, discard (treated as accidental brush). On timer elapse while still held, call `hyperjump()` exactly once. Does not retrigger while the button remains held — the player must release and press again. The 100ms threshold is a compromise between Tomasz's panic-proofing (accidental brushes are typically <50ms) and Priya's dexterity concern (>80ms needed for reliable activation by users with tremor).

Fire and warp call the same `shoot()` and `hyperjump()` functions used by the existing keyboard handlers (index.html:429–435). The keyboard path is not modified.

### Mode detection

**Initial:** `touch.mode = (matchMedia('(pointer: coarse)').matches || 'ontouchstart' in window) ? 'touch' : 'keyboard'`. Applied as `body.classList.toggle('touch-mode', touch.mode === 'touch')` before first paint. The `ontouchstart` fallback covers WKWebView (Instagram / X in-app browser) where `(pointer: coarse)` is unreliable on some iOS 17 shell versions.

**Reactive override** via event listeners added once at init:

- First `keydown` on arrow/space/H → flip to `'keyboard'`.
- First `pointerdown` where `e.pointerType === 'touch'` → flip to `'touch'`. Stylus and mouse pointer events do **not** trigger a flip; they are ignored for mode-detection purposes.
- Debounce: a mode flip is ignored if less than 1500ms has passed since the last flip. This prevents Surface-style flicker when the user alternates input sources rapidly.

### Rubber-band and gesture suppression

Pointer Events derived from touch cannot call `preventDefault()` on the underlying scroll gesture. To suppress overscroll bounce and rubber-band:

- Parallel `touchstart` and `touchmove` listeners on `document` with `{ passive: false }` that call `e.preventDefault()` only when `document.body.classList.contains('touch-mode')`.
- Pointer Events remain the primary input mechanism for game logic; touch events exist solely for scroll suppression.

## Control overlay

Shown only when `body.touch-mode` is set. All values are CSS pixels.

### Joystick (left side)

- **Floating, not fixed.** On `pointerdown` anywhere in the left 45% of the viewport, anchor the joystick base at the touchdown point.
- **Anchor position is corrected for `visualViewport` offsets** (iOS address-bar transition can shift `clientX/Y` away from the actual fingertip by up to 52px): `originX = e.clientX - window.visualViewport.offsetLeft; originY = e.clientY - window.visualViewport.offsetTop;`.
- **Safe-area rejection:** before anchoring, reject the touch if `e.clientY < parseInt(getComputedStyle(document.documentElement).getPropertyValue('--sat'))` (the top safe-area inset, including Dynamic Island exclusion zone). A CSS custom property `--sat` is populated via `:root { --sat: env(safe-area-inset-top); }`. Same rejection against `--sab` (bottom) + 12px.
- **Edge-clamp:** before committing the anchor, clamp so the 100px base stays on-screen: `originX = clamp(originX, 50, viewportLeft45pct - 50); originY = clamp(originY, safeTop + 50, viewportHeight - safeBottom - 62)`. Prevents half-joysticks at screen corners.
- Base is a `100px` dashed circle rendered at the anchor; knob is a `44px` filled circle starting at the anchor. Knob clamp radius is **50px** — matches the base visual edge so the knob never overhangs the ring.
- On `pointermove`, compute delta from origin, clamp magnitude to 50px, move the knob visually. Write to `keys.ArrowLeft/Right` based on sign of horizontal delta, subject to the 8px centre deadzone. Set `touch.joystick.active = magnitude > deadzone`; knob CSS applies a glow/colour shift when active.
- Call `setPointerCapture(e.pointerId)` on `pointerdown`. Release and spring knob + clear `keys.ArrowLeft` and `keys.ArrowRight` on **`pointerup` and `pointercancel`** (same handler block, cancel branch written first, not later). `pointerleave` is **not** used — on captured pointers Safari suppresses it, so listening for it is a maintenance trap.
- **Ship-proximity alpha:** the joystick element's opacity fades linearly from 100% at a 120-CSS-px distance to 30% at 60 CSS px between the ship's screen position and the joystick origin. CSS-px, not canvas-px. Linear fade avoids a jarring hard cutoff.

### Right-side button stack

Three buttons, stacked vertically with **12px minimum gap between button edges**, each with a transparent padding-halo extending the hit rect beyond the visual:

- **THRUST** — top, 64px, held = continuous thrust.
- **FIRE** — middle, 56px, hold-to-auto-fire at 100ms (10 shots/sec). Label "FIRE".
- **WARP** — bottom, 48px, visually distinct colour, requires 100ms hold to activate. Label "WARP" (renamed from "HYPR"). While the hold timer is running, a border-fill animation fills over 100ms to telegraph the hold; cancelled if released early.

Bottom of the WARP button sits at least `env(safe-area-inset-bottom) + 12px` above the viewport bottom. Verified via computed hit rect, not just padding.

**Visual feedback on press** (all three plus the PAUSE button):

```css
#touch-controls button:active { transform: scale(0.93); background: var(--btn-active); }
```

**Label contrast** must meet WCAG 1.4.3 AA — minimum 4.5:1 between label text and the button background. Because buttons are semi-transparent over a dark starfield, specify label text as pure white (`#ffffff`) on a solid dark chip (`rgba(0, 0, 0, 0.55)`) inside each button, rather than relying on the overall button background alone.

### Pause button

A 32px PAUSE button lives at the top-centre of the touch overlay. `pointerdown` toggles `state.paused`. No hold timer, no auto-repeat. When paused, a semi-transparent overlay + "PAUSED — tap to resume" text covers the canvas; tapping anywhere on that overlay resumes. Keeps in-game pause reachable without backgrounding the tab.

### Start and replay buttons

Currently inside `.cabinet` (index.html:396–397). **No reparenting** — the DOM stays as-is so the desktop arcade-cabinet aesthetic is preserved. In touch mode, the `#title` and `#gameover` overlays use `position: fixed; inset: 0; z-index: 9999` to escape the cabinet's stacking context visually and fill the viewport. On desktop they remain positioned within the cabinet as now. `click` handlers are unchanged and already work on tap.

## Layout & viewport

### Viewport meta

Update index.html:5 to:

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

Note: `user-scalable=no` is **deliberately not included** (WCAG 1.4.4 — users may need to zoom UI labels). Pinch-zoom during gameplay is prevented at the canvas level via `touch-action`.

### CSS

Added unconditionally (harmless on desktop):

```css
html, body { overscroll-behavior: none; }
body { -webkit-touch-callout: none; }
:root { --sat: env(safe-area-inset-top); --sab: env(safe-area-inset-bottom); }
```

Added only in touch mode:

```css
body.touch-mode { touch-action: manipulation; }                 /* suppress double-tap zoom, scoped */
body.touch-mode canvas { touch-action: none; }                  /* no pinch/pan on gameplay */
body.touch-mode .cabinet {                                      /* passthrough, not display:contents */
  display: block;
  width: 100dvw; height: 100dvh;
  padding: 0; border: 0; border-radius: 0;
  background: none; box-shadow: none;
}
body.touch-mode .deck, body.touch-mode .bezel { display: none; }
body.touch-mode .screen { width: 100dvw; height: 100dvh; }
body.touch-mode #title, body.touch-mode #gameover { position: fixed; inset: 0; z-index: 9999; }
#touch-controls { padding: env(safe-area-inset-top) env(safe-area-inset-right) calc(env(safe-area-inset-bottom) + 12px) env(safe-area-inset-left); }
```

**Why not `display: contents` on `.cabinet`:** it removes the box entirely, which collapses all cabinet CSS (width/height/padding/background/box-shadow) and disrupts any child sizing that depended on cabinet dimensions. The passthrough approach above preserves the box but strips the decorative presentation — safer, and recoverable if a mode-flip transiently adds/removes `touch-mode`.

`100dvh`/`100dvw` are used instead of `100vh`/`100vw` to track the dynamic viewport as the iOS address bar shows/hides. `dvh` is supported in Safari 16+ and all current evergreen browsers.

### Canvas sizing regression

The existing ResizeObserver at index.html:400–409 currently captures `DPR` once at boot with `let DPR = Math.min(window.devicePixelRatio || 1, 2)`. This becomes stale when the display context changes (mode flip on a docked hybrid device). The fix is to move the `DPR` read inside the `resize()` callback so it is re-evaluated on every resize event. The existing `getBoundingClientRect()` logic is unchanged.

### Reduced motion

Enumerate exactly what is suppressed when `(prefers-reduced-motion: reduce)` matches. Decorative motion is disabled; gameplay motion (ship, asteroids, bullets, explosions) is untouched — they are functional, not decorative.

**CSS:**
```css
@media (prefers-reduced-motion: reduce) {
  .starfield { animation: none !important; }          /* background parallax scroll */
  #touch-controls button:active { transform: none; }  /* remove press-scale animation */
  /* Note: hyperjump flash overlay's transition is driven by JS; see below. */
}
```

**JS:** a module-scoped `reducedMotion = matchMedia('(prefers-reduced-motion: reduce)').matches` flag:
- Skips the screen-shake routine in `draw()` (whichever line adds the current shake offset — replaced with a noop when the flag is set).
- Sets the hyperjump flash duration to 0 (the instantaneous state change still happens; the animation does not).
- Re-evaluated on `change` of the media query, so toggling OS-level reduced-motion mid-session takes effect next frame.

### Pause on hide

The existing game has no pause function — the game loop runs unconditionally via `requestAnimationFrame`. This spec adds a minimal pause mechanism as part of the touch-controls change, since background-pause is a general mobile requirement.

- A `paused` boolean on the existing `state` object. The `update()` and `draw()` calls in the loop bail early when `paused` is true; `requestAnimationFrame` continues so resume is instant. Both bail-outs are added at the top of the existing `update()` call site and at the top of the `draw()` call path inside the `requestAnimationFrame` callback — implementation must cite the exact lines in the PR.
- `document.addEventListener('visibilitychange', ...)` sets `paused = document.hidden`. **In the same handler**, unconditionally clear the FIRE auto-fire interval (`clearInterval(touch.buttons.fireIntervalId)`), clear the WARP hold timer, reset all pointer-id fields, and clear `keys.ArrowUp/Left/Right`. Otherwise iOS may swallow a `pointerup` while backgrounded and leave input state stuck on resume.
- The in-game PAUSE button (see Pause button section above) toggles the same flag.

### Canvas ARIA

```html
<canvas role="application" aria-label="Starfield Drift — arcade action game, not accessible to screen readers"></canvas>
```

Honest graceful exclusion; no attempt to describe dynamic state.

### Portrait tip

If `matchMedia('(orientation: portrait)').matches` and `touch.mode === 'touch'`, show a small non-blocking overlay "↻ rotate for best view". The game still plays; the tip does not pause or block input. Dismissable by tap.

## Component boundaries

Three logical sections added to `index.html`, each prefixed with a short comment block:

1. **Mode detection** — the `touch` controller object plus mode-flip listeners and debounce.
2. **Overlay DOM + CSS** — the `#touch-controls` container, joystick base/knob, three buttons.
3. **Pointer handlers** — joystick pointer lifecycle (down/move/up/cancel/leave), button handlers, `touchstart`/`touchmove` scroll suppression.

Each section can be reasoned about in isolation. Section 1 depends on nothing; section 2 depends on CSS only; section 3 depends on sections 1 and 2 and the existing `keys` object.

## Manual verification checklist

Added as a comment block directly above the existing input section at index.html:~425:

```
// Manual test after touch-control changes:
//   Desktop: arrow keys steer; Space fires; H warps; Start/Replay click work.
//   Touch (iPhone Safari): floating joystick steers left/right; Thrust button thrusts;
//     Fire button auto-fires while held; Warp requires 80ms hold; Start/Replay tap work.
//   Hybrid (iPad + keyboard): plugging keyboard hides overlay within 1.5s of first keypress;
//     unplugging and tapping shows overlay again.
//   Backgrounding the tab pauses the game; returning resumes.
```

## Explicit follow-ups (not in this scope)

- **Left-handed mirror toggle.** Priya flagged one-handed play and handedness as accessibility concerns. The reconciled design keeps the joystick on the left and buttons on the right; a mirror toggle would cost ~15 lines of CSS plus a setting. Defer until requested or until a second mobile-related change lands.
- **Auto-thrust toggle.** Tomasz noted the dead-hand problem in sustained asteroids play. Not addressed now. Revisit if playtesting shows thumb fatigue.
- **File splitting.** If the next mobile-related feature pushes the file past ~1400 lines, split `index.html` into `index.html` + `game.js` + `touch.js` + `style.css`.

## Success criteria

- Opening the page on iPhone Safari shows the game full-screen with cabinet chrome hidden, a floating joystick area on the left, and three buttons on the right.
- The player can: rotate, thrust, fire (tap and hold), warp (held), start a game, and restart after game over — all without a physical keyboard.
- Opening the page on desktop Chrome/Firefox/Safari shows the arcade cabinet and keyboard gameplay as before, unchanged.
- Connecting or disconnecting a Bluetooth keyboard on iPad switches modes cleanly within 1.5 seconds of the first input in the new modality, without overlay flicker.
- Backgrounding the tab pauses the game.
- `prefers-reduced-motion` disables starfield parallax and screen-shake.
