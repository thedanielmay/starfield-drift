# Mobile Touch Controls Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add touch controls to `index.html` so the game is playable on iPhone Safari, without regressing the desktop keyboard experience.

**Architecture:** Single-file HTML game. All changes land in `index.html` — new CSS in the existing `<style>` block, new HTML in the existing cabinet markup, new JS in the existing IIFE. Touch handlers mutate the same `keys` object the game loop already polls, so the game loop is untouched. A new `touch` controller object holds all new state. A `body.touch-mode` class toggles mobile-specific layout. Mode detection uses `matchMedia('(pointer: coarse)')` + `'ontouchstart' in window` as initial, reactively overridden by first real input event.

**Tech Stack:** Vanilla HTML/CSS/JS. No build step. No test framework. No dependencies.

**Testing approach:** This project has no JS test framework and adding one is out of scope. Verification is **manual browser-based**, with explicit steps per task (what to open, what to do, what to observe). Each task ends with a manual-verification block before the commit. The final task adds a written manual-test checklist inside `index.html` as a code comment for future maintainers.

**Spec:** `docs/superpowers/specs/2026-04-23-mobile-touch-controls-design.md`

---

## File structure

Only `index.html` is modified. Changes are grouped into five logical sections to keep each task's diff focused and reviewable:

1. **Viewport meta + root CSS custom properties** (near line 5 and inside `<style>`) — safe-area props, unconditional page-level rules.
2. **Touch-mode layout CSS** (inside `<style>`) — `body.touch-mode` rules for cabinet passthrough, overlay fixed-positioning, control overlay layout.
3. **Touch overlay DOM** (inside the existing `.screen` container in HTML) — joystick base/knob, four buttons (THRUST/FIRE/WARP/PAUSE), pause-resume overlay, portrait tip.
4. **`touch` controller object + mode detection + lifecycle** (inside the IIFE, near line 425) — state, initial mode, reactive override, debounce.
5. **Pointer handlers + pause hooks** (inside the IIFE) — joystick pointer lifecycle, button handlers, scroll suppression, `visibilitychange`, pause flag in `update()`/`render()`, `prefers-reduced-motion` flag.

## Task-order rationale

Tasks are ordered so that each one is independently verifiable without the next one existing. CSS and DOM come before handlers so there's a visible overlay to poke at. Mode detection lands before handlers so the overlay can actually show/hide. Pause lands near the end because it depends on state mutations that the earlier tasks don't yet perform.

---

### Task 1: Viewport meta, root custom properties, and page-level CSS

**Files:**
- Modify: `index.html:5` (viewport meta)
- Modify: `index.html:11-26` (`:root` block — add safe-area custom properties)
- Modify: `index.html:~38` (after the existing `body` rule — add `overscroll-behavior` and `touch-callout`)

**Rationale:** Establishes the page-level defaults the rest of the plan depends on. Must land first because `--sat` / `--sab` are referenced in later CSS and JS.

- [ ] **Step 1: Update the viewport meta tag**

Replace `index.html:5`:

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
```

- [ ] **Step 2: Add safe-area custom properties to the `:root` block**

Inside the existing `:root { ... }` block at `index.html:11-26`, after the last `--chrome:#d6c8ff;` line, add:

```css
    --sat: env(safe-area-inset-top);
    --sar: env(safe-area-inset-right);
    --sab: env(safe-area-inset-bottom);
    --sal: env(safe-area-inset-left);
```

- [ ] **Step 3: Add unconditional page-level rules**

After the existing `body { ... }` rule (the one that ends with `-webkit-user-select:none;`), add a new rule:

```css
  html, body { overscroll-behavior: none; }
  body { -webkit-touch-callout: none; }
```

- [ ] **Step 4: Manually verify desktop is unchanged**

Open `index.html` in desktop Chrome or Safari. Observe:
- The arcade cabinet renders exactly as before (no visible layout shift).
- Keyboard gameplay works: arrow keys rotate/thrust, space fires, H hyperjumps, start/replay buttons work.
- No console errors.

Expected: indistinguishable from the pre-change behaviour. If the cabinet shifts or anything breaks, the `overscroll-behavior` rule is likely colliding with the existing body rule — inspect the computed style.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "touch: add viewport-fit, safe-area custom properties, and page-level touch CSS

Prepares the page for touch-mode. Desktop behaviour is unchanged:
- viewport-fit=cover unlocks env() safe-area insets
- --sat/--sar/--sab/--sal custom props referenced by later CSS/JS
- overscroll-behavior:none and -webkit-touch-callout:none are harmless on desktop"
```

---

### Task 2: Add `touch-mode` CSS for cabinet passthrough and overlay repositioning

**Files:**
- Modify: `index.html` — add new CSS inside the existing `<style>` block, after the `.cabinet` rule (around line 80-ish, wherever feels natural alphabetically/grouped)

**Rationale:** Defines what `body.touch-mode` does visually. Lands before the DOM/JS that toggles the class so we can manually verify by temporarily adding the class in DevTools.

- [ ] **Step 1: Add the cabinet-passthrough and touch-mode layout CSS**

Inside the `<style>` block, after the existing `.cabinet { ... }` rule, add:

```css
  /* ---------- Touch mode ---------- */
  body.touch-mode { touch-action: manipulation; }        /* suppresses double-tap zoom */
  body.touch-mode canvas { touch-action: none; }         /* no pinch/pan on gameplay */

  body.touch-mode .room::after { display: none; }        /* hide floor grid */

  body.touch-mode .cabinet {
    width: 100dvw;
    height: 100dvh;
    padding: 0;
    border-radius: 0;
    background: none;
    box-shadow: none;
  }

  body.touch-mode .deck,
  body.touch-mode .bezel { display: none; }

  body.touch-mode .screen {
    width: 100dvw;
    height: 100dvh;
    border-radius: 0;
  }

  body.touch-mode #title,
  body.touch-mode #gameover {
    position: fixed;
    inset: 0;
    z-index: 9999;
  }
```

(The `.room::after` floor-grid hide is extrapolated from the existing code at `index.html:50-61`; if that selector doesn't exist in your tree, skip that one rule.)

- [ ] **Step 2: Add reduced-motion CSS**

Immediately after the touch-mode block above, add:

```css
  @media (prefers-reduced-motion: reduce) {
    .starfield, .room::after { animation: none !important; }
    #touch-controls button:active { transform: none; }
  }
```

- [ ] **Step 3: Manually verify the class toggle in DevTools**

Open `index.html` in a desktop browser. In the DevTools Elements panel:
1. Select the `<body>` element.
2. Add the class `touch-mode` (edit element → add class, or `document.body.classList.add('touch-mode')` in the console).
3. Observe: the arcade cabinet chrome collapses — no bezel, no deck, canvas fills the viewport. Remove the class; cabinet returns.

Expected: clean, reversible toggle with no console errors. If anything is stuck, check for CSS specificity battles with the existing `.cabinet` rule.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "touch: add body.touch-mode CSS for cabinet passthrough and overlay positioning

- Cabinet becomes a visual passthrough (dimensions preserved, decoration stripped)
- .deck / .bezel hidden in touch mode
- #title / #gameover use position:fixed to escape stacking context
- touch-action scoped: manipulation on body (double-tap), none on canvas (gameplay)
- prefers-reduced-motion disables decorative animations"
```

---

### Task 3: Add the touch overlay DOM

**Files:**
- Modify: `index.html:~312` (inside `.screen`, after the `<canvas id="game">` and before any other children)

**Rationale:** Plain markup — no handlers yet — so it renders inert. Verifying at this step confirms the CSS hooks are correct before we write any JS.

- [ ] **Step 1: Add the touch overlay CSS rules**

Inside the `<style>` block, after the reduced-motion rule from Task 2, add:

```css
  /* ---------- Touch overlay ---------- */
  #touch-controls {
    display: none;                       /* shown via body.touch-mode below */
    position: absolute;
    inset: 0;
    pointer-events: none;                /* empty space doesn't eat clicks */
    padding: var(--sat) var(--sar) calc(var(--sab) + 12px) var(--sal);
    z-index: 500;
    font-family: "JetBrains Mono", monospace;
  }
  body.touch-mode #touch-controls { display: block; }

  #touch-controls .stick-base {
    position: absolute;
    width: 100px; height: 100px;
    border-radius: 50%;
    background: radial-gradient(circle, rgba(123,75,255,0.18), rgba(123,75,255,0.05));
    border: 2px dashed rgba(59,240,255,0.5);
    pointer-events: none;
    transform: translate(-50%,-50%);
    opacity: 0;
    transition: opacity 0.15s;
  }
  #touch-controls .stick-base.visible { opacity: 1; }
  #touch-controls .stick-knob {
    position: absolute;
    left: 50%; top: 50%;
    width: 44px; height: 44px;
    border-radius: 50%;
    background: radial-gradient(circle, rgba(180,220,255,0.8), rgba(123,75,255,0.5));
    border: 2px solid rgba(214,200,255,0.9);
    transform: translate(-50%,-50%);
    box-shadow: 0 0 12px rgba(123,75,255,0.6);
    transition: background 0.1s, box-shadow 0.1s;
  }
  #touch-controls .stick-knob.active {
    background: radial-gradient(circle, rgba(59,240,255,0.95), rgba(217,70,255,0.6));
    box-shadow: 0 0 20px rgba(59,240,255,0.8);
  }

  #touch-controls .btn-col {
    position: absolute;
    right: calc(var(--sar) + 20px);
    bottom: calc(var(--sab) + 20px);
    display: flex;
    flex-direction: column;
    gap: 12px;
    align-items: center;
    pointer-events: none;
  }
  #touch-controls button {
    pointer-events: auto;
    border: 2px solid currentColor;
    border-radius: 50%;
    background: rgba(0,0,0,0.4);
    color: var(--ink);
    font-family: inherit;
    font-weight: bold;
    letter-spacing: 1px;
    cursor: pointer;
    position: relative;
    user-select: none;
    -webkit-user-select: none;
    touch-action: none;
  }
  #touch-controls button:active { transform: scale(0.93); }
  #touch-controls button .chip {
    position: absolute;
    inset: 4px;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    background: rgba(0,0,0,0.55);
    color: #fff;
    font-size: 10px;
    letter-spacing: 1px;
  }
  #touch-controls .btn-thrust { width: 64px; height: 64px; color: var(--amber); }
  #touch-controls .btn-fire   { width: 56px; height: 56px; color: var(--danger); }
  #touch-controls .btn-warp   { width: 48px; height: 48px; color: var(--magenta); }

  #touch-controls .btn-warp .fill {
    position: absolute;
    inset: -2px;
    border-radius: 50%;
    border: 2px solid var(--cyan);
    clip-path: inset(100% 0 0 0);
    pointer-events: none;
  }
  #touch-controls .btn-warp.charging .fill {
    animation: warpFill 100ms linear forwards;
  }
  @keyframes warpFill { to { clip-path: inset(0 0 0 0); } }

  #touch-controls .btn-pause {
    pointer-events: auto;
    position: absolute;
    top: calc(var(--sat) + 10px);
    left: 50%;
    transform: translateX(-50%);
    width: 32px; height: 32px;
    border: 2px solid var(--chrome);
    border-radius: 50%;
    background: rgba(0,0,0,0.4);
    color: var(--chrome);
    font-size: 11px;
    font-family: inherit;
    cursor: pointer;
    user-select: none;
    -webkit-user-select: none;
    touch-action: none;
  }

  #pause-screen {
    display: none;
    position: fixed; inset: 0;
    background: rgba(7,3,15,0.8);
    z-index: 9998;
    align-items: center;
    justify-content: center;
    color: var(--ink);
    font-family: "Orbitron", sans-serif;
    font-size: 20px;
    letter-spacing: 4px;
  }
  #pause-screen.visible { display: flex; }

  #portrait-tip {
    display: none;
    position: fixed;
    top: calc(var(--sat) + 48px);
    left: 50%;
    transform: translateX(-50%);
    background: rgba(0,0,0,0.6);
    border: 1px solid var(--cyan);
    color: var(--cyan);
    padding: 8px 14px;
    border-radius: 4px;
    font-size: 12px;
    letter-spacing: 1px;
    z-index: 9997;
    pointer-events: auto;
  }
  body.touch-mode.portrait #portrait-tip { display: block; }
```

- [ ] **Step 2: Add the overlay markup inside `.screen`**

At `index.html:~312`, immediately after `<canvas id="game"></canvas>` and before the existing overlays, add:

```html
        <div id="touch-controls" aria-hidden="true">
          <div class="stick-base"></div>
          <div class="stick-knob"></div>
          <button class="btn-thrust" aria-label="Thrust"><span class="chip">THRUST</span></button>
          <div class="btn-col">
            <button class="btn-warp" aria-label="Warp (hold briefly)"><span class="chip">WARP</span><span class="fill"></span></button>
            <button class="btn-fire" aria-label="Fire"><span class="chip">FIRE</span></button>
          </div>
          <button class="btn-pause" aria-label="Pause">II</button>
        </div>
        <div id="pause-screen">PAUSED — TAP TO RESUME</div>
        <div id="portrait-tip">↻ ROTATE FOR BEST VIEW</div>
```

(THRUST sits free-positioned top-right via JS in a later task; FIRE + WARP are stacked on the right via the `.btn-col` flex column.)

Actually — correcting mid-step for clarity: THRUST also lives in the right column below WARP/FIRE? No. Per spec: THRUST top (64px), FIRE middle (56px), WARP bottom (48px). All three stack right. Replace the markup with:

```html
        <div id="touch-controls" aria-hidden="true">
          <div class="stick-base"></div>
          <div class="stick-knob"></div>
          <div class="btn-col">
            <button class="btn-thrust" aria-label="Thrust"><span class="chip">THRUST</span></button>
            <button class="btn-fire" aria-label="Fire"><span class="chip">FIRE</span></button>
            <button class="btn-warp" aria-label="Warp (hold briefly)"><span class="chip">WARP</span><span class="fill"></span></button>
          </div>
          <button class="btn-pause" aria-label="Pause">II</button>
        </div>
        <div id="pause-screen">PAUSED — TAP TO RESUME</div>
        <div id="portrait-tip">↻ ROTATE FOR BEST VIEW</div>
```

- [ ] **Step 3: Add ARIA label to the canvas**

At `index.html:312` (the `<canvas>` element), change:

```html
        <canvas id="game"></canvas>
```

to:

```html
        <canvas id="game" role="application" aria-label="Starfield Drift — arcade action game, not accessible to screen readers"></canvas>
```

- [ ] **Step 4: Manually verify the overlay renders only in touch mode**

Open `index.html` in desktop Chrome:
1. Confirm no overlay visible by default. Cabinet looks normal. Keyboard still works.
2. In DevTools console: `document.body.classList.add('touch-mode')`.
3. Confirm: cabinet collapses, canvas fills viewport, the three right-side buttons are visible stacked vertically, the PAUSE button is at top-centre, joystick base/knob are hidden (no `.visible` class yet).
4. Remove the class: overlay disappears cleanly.

Expected: overlay renders with correct positioning and is inert (buttons do nothing yet). No console errors.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "touch: add touch overlay DOM + CSS (inert, no handlers yet)

Introduces joystick base/knob, THRUST/FIRE/WARP button column, PAUSE button,
pause screen, and portrait tip. All hidden by default; visible via body.touch-mode.
Canvas gets role=application + aria-label for honest screen-reader exclusion.
No interactive behaviour yet — handlers land in later tasks."
```

---

### Task 4: Add the `touch` controller object and mode detection

**Files:**
- Modify: `index.html:~425` (immediately before the existing `const keys = Object.create(null);` line)

**Rationale:** Creates the state container and sets the initial mode class before any handlers try to read it. Mode flips are wired but do nothing user-visible beyond toggling the `touch-mode` class.

- [ ] **Step 1: Add the `touch` controller object**

Immediately before the `// ---------- Input ----------` section header at `index.html:~424`, add:

```js
  // ---------- Touch controller (mode + state) ----------
  const touch = {
    mode: 'keyboard',
    lastModeFlip: 0,
    reducedMotion: matchMedia('(prefers-reduced-motion: reduce)').matches,
    overlay: null,     // filled in next task
    pauseScreen: null, // filled in next task
    portraitTip: null, // filled in next task
    joystick: {
      base: null,
      knob: null,
      activePointerId: null,
      originX: 0, originY: 0,
      radius: 50,
      deadzone: 8,
      active: false,
      proximityAlpha: 1,
    },
    buttons: {
      thrust: null, fire: null, warp: null, pause: null,
      thrustPointerId: null, firePointerId: null, warpPointerId: null,
      fireIntervalId: null, warpHoldTimerId: null,
    },
  };

  function setInputMode(next, now) {
    if (next === touch.mode) return;
    if (now - touch.lastModeFlip < 1500) return; // debounce hybrid-device flicker
    touch.mode = next;
    touch.lastModeFlip = now;
    document.body.classList.toggle('touch-mode', next === 'touch');
  }

  // initial mode: coarse pointer OR touchstart capability (WKWebView fallback)
  {
    const touchFirst = matchMedia('(pointer: coarse)').matches || 'ontouchstart' in window;
    touch.mode = touchFirst ? 'touch' : 'keyboard';
    document.body.classList.toggle('touch-mode', touch.mode === 'touch');
  }

  // reactive mode override on first real input
  window.addEventListener('keydown', e => {
    if (['ArrowUp','ArrowDown','ArrowLeft','ArrowRight',' ','h','H'].includes(e.key)) {
      setInputMode('keyboard', performance.now());
    }
  }, true); // capture phase so this runs before the existing game handler

  window.addEventListener('pointerdown', e => {
    if (e.pointerType === 'touch') setInputMode('touch', performance.now());
  }, true);

  matchMedia('(prefers-reduced-motion: reduce)').addEventListener('change', e => {
    touch.reducedMotion = e.matches;
  });

  // portrait detection → class on body drives #portrait-tip visibility
  const portraitMQ = matchMedia('(orientation: portrait)');
  const syncPortrait = () => document.body.classList.toggle('portrait', portraitMQ.matches);
  portraitMQ.addEventListener('change', syncPortrait);
  syncPortrait();
```

- [ ] **Step 2: Fix the stale-DPR bug while we're here**

Per spec: `DPR` is captured once at boot (line 400) and becomes stale on mode changes. Replace `index.html:400-407`:

```js
  // ---------- Canvas sizing ----------
  let W = 0, H = 0, DPR = 1;
  function resize() {
    DPR = Math.min(window.devicePixelRatio || 1, 2);   // re-read every resize
    const rect = canvas.getBoundingClientRect();
    W = rect.width; H = rect.height;
    canvas.width  = Math.floor(W * DPR);
    canvas.height = Math.floor(H * DPR);
    ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
  }
```

- [ ] **Step 3: Wire the overlay element refs at startup**

After the existing `const againBtn = document.getElementById('againBtn');` at `index.html:397`, add:

```js
  touch.overlay = document.getElementById('touch-controls');
  touch.pauseScreen = document.getElementById('pause-screen');
  touch.portraitTip = document.getElementById('portrait-tip');
  touch.joystick.base = touch.overlay.querySelector('.stick-base');
  touch.joystick.knob = touch.overlay.querySelector('.stick-knob');
  touch.buttons.thrust = touch.overlay.querySelector('.btn-thrust');
  touch.buttons.fire   = touch.overlay.querySelector('.btn-fire');
  touch.buttons.warp   = touch.overlay.querySelector('.btn-warp');
  touch.buttons.pause  = touch.overlay.querySelector('.btn-pause');
```

- [ ] **Step 4: Manually verify mode detection**

1. Open `index.html` in desktop Chrome. Inspect `<body>`: should NOT have `touch-mode` class. Cabinet renders normally. `touch` object is visible in console with `touch.mode === 'keyboard'`.
2. Enable touch emulation in DevTools (Cmd+Shift+M → toolbar phone icon → "Responsive"). Reload. `<body>` should have `touch-mode`. Overlay DOM renders over the canvas.
3. Disable emulation, reload. `touch-mode` gone again.
4. On an actual iPhone: open the file (via a local server or deployed), confirm `touch-mode` is present from first paint.
5. With DevTools emulation on, press an arrow key on your real keyboard: after 1.5s debounce, `touch-mode` should strip. Click inside the viewport with touch emulation: mode flips back to touch.

Expected: mode is correct from first paint, reactive override works, debounce prevents rapid flicker. DPR still behaves sanely on resize.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "touch: add touch controller + mode detection + DPR re-read on resize

Introduces the 'touch' state object (no handlers yet), initial mode from
(pointer:coarse) || ontouchstart, reactive override on first real
keydown/pointerdown(touch), 1500ms debounce for hybrid-device flicker, and
reduced-motion + portrait media-query sync.

Also fixes stale-DPR: DPR is now re-read inside resize() on every callback,
so display-density changes (mode flip on docked hybrid devices) are honoured."
```

---

### Task 5: Implement the joystick pointer handlers

**Files:**
- Modify: `index.html` — add new code after the touch-controller setup from Task 4 (still inside the IIFE)

**Rationale:** First interactive piece. Independently testable: turn rotation on mobile, nothing else yet.

- [ ] **Step 1: Add the helper clamps and the joystick handlers**

Immediately after the block you added in Task 4 Step 3 (the element refs), add:

```js
  // ---------- Touch: joystick ----------
  const clamp = (v, lo, hi) => Math.max(lo, Math.min(hi, v));

  function getSafeAreaTopPx() {
    return parseFloat(getComputedStyle(document.documentElement).getPropertyValue('--sat')) || 0;
  }
  function getSafeAreaBottomPx() {
    return parseFloat(getComputedStyle(document.documentElement).getPropertyValue('--sab')) || 0;
  }

  function anchorJoystick(clientX, clientY) {
    // compensate for visualViewport offset during iOS address-bar transitions
    const vv = window.visualViewport;
    const offX = vv ? vv.offsetLeft : 0;
    const offY = vv ? vv.offsetTop : 0;
    let x = clientX - offX;
    let y = clientY - offY;

    // clamp so the 100px base stays on-screen (50px half-base margin)
    const maxX = window.innerWidth * 0.45 - 50;
    const minX = 50;
    const minY = getSafeAreaTopPx() + 50;
    const maxY = window.innerHeight - getSafeAreaBottomPx() - 62;
    x = clamp(x, minX, maxX);
    y = clamp(y, minY, maxY);

    touch.joystick.originX = x;
    touch.joystick.originY = y;
    touch.joystick.base.style.left = x + 'px';
    touch.joystick.base.style.top  = y + 'px';
    touch.joystick.knob.style.left = x + 'px';
    touch.joystick.knob.style.top  = y + 'px';
    touch.joystick.base.classList.add('visible');
  }

  function resetJoystick() {
    touch.joystick.activePointerId = null;
    touch.joystick.active = false;
    touch.joystick.knob.classList.remove('active');
    touch.joystick.base.classList.remove('visible');
    touch.joystick.knob.style.left = touch.joystick.originX + 'px';
    touch.joystick.knob.style.top  = touch.joystick.originY + 'px';
    keys['ArrowLeft']  = false;
    keys['ArrowRight'] = false;
  }

  // listen on the overlay for touchdowns in the left 45% of the viewport
  touch.overlay.addEventListener('pointerdown', e => {
    if (e.pointerType !== 'touch') return;
    if (touch.joystick.activePointerId !== null) return; // already tracking
    if (e.clientX > window.innerWidth * 0.45) return;    // right-side is button territory
    if (e.target.closest('button')) return;              // let buttons handle their own
    if (e.clientY < getSafeAreaTopPx()) return;          // reject Dynamic Island zone

    touch.joystick.activePointerId = e.pointerId;
    anchorJoystick(e.clientX, e.clientY);
    // capture on the knob element so we keep getting events even if finger drags off
    touch.joystick.knob.setPointerCapture(e.pointerId);
    e.preventDefault();
  });

  touch.joystick.knob.addEventListener('pointermove', e => {
    if (e.pointerId !== touch.joystick.activePointerId) return;
    const vv = window.visualViewport;
    const offX = vv ? vv.offsetLeft : 0;
    const offY = vv ? vv.offsetTop : 0;
    const dx = (e.clientX - offX) - touch.joystick.originX;
    const dy = (e.clientY - offY) - touch.joystick.originY;
    const mag = Math.hypot(dx, dy);
    const k = mag > touch.joystick.radius ? touch.joystick.radius / mag : 1;
    const kx = dx * k, ky = dy * k;
    touch.joystick.knob.style.left = (touch.joystick.originX + kx) + 'px';
    touch.joystick.knob.style.top  = (touch.joystick.originY + ky) + 'px';

    const past = mag > touch.joystick.deadzone;
    touch.joystick.active = past;
    touch.joystick.knob.classList.toggle('active', past);
    if (past) {
      keys['ArrowLeft']  = dx < -touch.joystick.deadzone;
      keys['ArrowRight'] = dx >  touch.joystick.deadzone;
    } else {
      keys['ArrowLeft']  = false;
      keys['ArrowRight'] = false;
    }
  });

  // release: same handler block for up AND cancel (NOT pointerleave — Safari
  // suppresses it on captured pointers, so listening for it is a trap)
  function onJoystickRelease(e) {
    if (e.pointerId !== touch.joystick.activePointerId) return;
    resetJoystick();
  }
  touch.joystick.knob.addEventListener('pointerup', onJoystickRelease);
  touch.joystick.knob.addEventListener('pointercancel', onJoystickRelease);
```

- [ ] **Step 2: Manually verify joystick on mobile emulation**

1. Open `index.html` in Chrome with DevTools touch emulation on, viewport something like iPhone 12 (390×844, landscape).
2. Start a game (click the start button).
3. Tap-and-drag from the left half of the screen: dashed joystick base appears at the touchdown point, knob follows your drag within a 50px radius.
4. Drag the knob left past 8px: ship rotates left. Drag right: ship rotates right. Release to centre: ship stops rotating.
5. Drag-off: release finger anywhere; knob springs back, base fades out.
6. On a real iPhone: confirm the same, plus confirm the joystick feels correct when dragging past the viewport edge.

Expected: smooth rotation control. No stuck-key state after release. `keys.ArrowLeft`/`ArrowRight` in console show correct true/false transitions.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "touch: implement floating joystick (rotation only)

- Anchor on first pointerdown in left 45% of viewport
- visualViewport offset correction for iOS address-bar transitions
- Edge-clamp so the 100px base never clips off-screen
- Dynamic Island / top safe-area rejection via --sat custom property
- 50px clamp radius (matches base edge), 8px deadzone
- Active-state visual feedback on knob past deadzone
- pointerup + pointercancel release (pointerleave deliberately NOT used —
  Safari suppresses it on captured pointers)
- Keys cleared on release; no stuck input state"
```

---

### Task 6: Implement the THRUST, FIRE, and WARP button handlers

**Files:**
- Modify: `index.html` — add after the joystick handlers from Task 5

**Rationale:** Completes the core control set. After this task, the game is playable on touch end-to-end.

- [ ] **Step 1: Add the button handlers**

Immediately after the joystick handlers you added in Task 5, add:

```js
  // ---------- Touch: buttons ----------
  function bindThrust() {
    touch.buttons.thrust.addEventListener('pointerdown', e => {
      if (touch.buttons.thrustPointerId !== null) return;
      touch.buttons.thrustPointerId = e.pointerId;
      touch.buttons.thrust.setPointerCapture(e.pointerId);
      keys['ArrowUp'] = true;
      e.preventDefault();
    });
    const release = e => {
      if (e.pointerId !== touch.buttons.thrustPointerId) return;
      touch.buttons.thrustPointerId = null;
      keys['ArrowUp'] = false;
    };
    touch.buttons.thrust.addEventListener('pointerup', release);
    touch.buttons.thrust.addEventListener('pointercancel', release);
  }

  function bindFire() {
    touch.buttons.fire.addEventListener('pointerdown', e => {
      if (touch.buttons.firePointerId !== null) return;
      if (state.mode !== 'playing' || !state.ship || state.ship.dead) return;
      touch.buttons.firePointerId = e.pointerId;
      touch.buttons.fire.setPointerCapture(e.pointerId);
      shoot();
      touch.buttons.fireIntervalId = setInterval(() => {
        if (state.mode === 'playing' && state.ship && !state.ship.dead) shoot();
      }, 100);
      e.preventDefault();
    });
    const release = e => {
      if (e.pointerId !== touch.buttons.firePointerId) return;
      touch.buttons.firePointerId = null;
      if (touch.buttons.fireIntervalId !== null) {
        clearInterval(touch.buttons.fireIntervalId);
        touch.buttons.fireIntervalId = null;
      }
    };
    touch.buttons.fire.addEventListener('pointerup', release);
    touch.buttons.fire.addEventListener('pointercancel', release);
  }

  function bindWarp() {
    touch.buttons.warp.addEventListener('pointerdown', e => {
      if (touch.buttons.warpPointerId !== null) return;
      if (state.mode !== 'playing' || !state.ship || state.ship.dead) return;
      touch.buttons.warpPointerId = e.pointerId;
      touch.buttons.warp.setPointerCapture(e.pointerId);
      touch.buttons.warp.classList.add('charging');
      touch.buttons.warpHoldTimerId = setTimeout(() => {
        // timer fired, button was still held → warp once
        if (state.mode === 'playing' && state.ship && !state.ship.dead) hyperjump();
        touch.buttons.warpHoldTimerId = null;
        // do not retrigger until release
      }, 100);
      e.preventDefault();
    });
    const release = e => {
      if (e.pointerId !== touch.buttons.warpPointerId) return;
      touch.buttons.warpPointerId = null;
      touch.buttons.warp.classList.remove('charging');
      if (touch.buttons.warpHoldTimerId !== null) {
        clearTimeout(touch.buttons.warpHoldTimerId);
        touch.buttons.warpHoldTimerId = null;
      }
    };
    touch.buttons.warp.addEventListener('pointerup', release);
    touch.buttons.warp.addEventListener('pointercancel', release);
  }

  bindThrust();
  bindFire();
  bindWarp();
```

- [ ] **Step 2: Manually verify all three buttons on emulation + real device**

Emulation path (DevTools touch):
1. Start a game. Tap-and-hold THRUST: ship accelerates; release: ship coasts and decelerates.
2. Tap FIRE once: single shot. Tap-and-hold: continuous shots at ~10/sec.
3. Quick tap WARP (<100ms): nothing happens, fill animation cancels before completing. Tap-and-hold for >100ms: fill completes, ship warps once. Keep holding: does NOT retrigger — you must release and press again.
4. Joystick + THRUST + FIRE simultaneously: all three should work in parallel (multi-touch).

Real device (iPhone) path:
- Same as above. Pay particular attention to: (a) no stuck `keys.ArrowUp` after releasing THRUST near a screen edge, (b) FIRE stops when finger lifts even if lifted off the button visually, (c) WARP fill animation is visible.

Expected: fully playable on touch. No stuck state. No double-fires. No phantom warps.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "touch: implement THRUST / FIRE / WARP button handlers

- Per-button pointer-capture + activePointerId tracking (correct multi-touch)
- THRUST: hold = keys.ArrowUp; release = release
- FIRE: immediate shot + 100ms auto-fire interval; cleared on release
- WARP: 100ms hold timer + border-fill animation; fires hyperjump() once;
  no retrigger while held
- All three bail out if state.mode !== 'playing' or ship is dead"
```

---

### Task 7: Scroll suppression, pause mechanism, and `visibilitychange` hygiene

**Files:**
- Modify: `index.html` — add after the button handlers from Task 6
- Modify: `index.html:704` (top of `update()`)
- Modify: `index.html:917` (top of `render()`)

**Rationale:** This is where the `state.paused` flag is actually consulted. Adding it earlier would be inert; adding it later means the game can't actually be paused.

- [ ] **Step 1: Add scroll suppression for touch mode**

Immediately after the button handlers you added in Task 6, add:

```js
  // ---------- Touch: rubber-band suppression ----------
  // Pointer Events can't preventDefault the underlying scroll; we hook the
  // native touch events in parallel, gated by body.touch-mode.
  const suppressScroll = e => {
    if (document.body.classList.contains('touch-mode')) e.preventDefault();
  };
  document.addEventListener('touchstart', suppressScroll, { passive: false });
  document.addEventListener('touchmove',  suppressScroll, { passive: false });
```

- [ ] **Step 2: Add the pause button handler + pause screen tap-to-resume**

Directly after the scroll-suppression block, add:

```js
  // ---------- Touch: pause ----------
  function setPaused(next) {
    state.paused = next;
    touch.pauseScreen.classList.toggle('visible', !!next);
    if (next) {
      // clear any in-flight input so we don't resume into a stuck state
      if (touch.buttons.fireIntervalId !== null) {
        clearInterval(touch.buttons.fireIntervalId);
        touch.buttons.fireIntervalId = null;
      }
      if (touch.buttons.warpHoldTimerId !== null) {
        clearTimeout(touch.buttons.warpHoldTimerId);
        touch.buttons.warpHoldTimerId = null;
        touch.buttons.warp.classList.remove('charging');
      }
      touch.buttons.thrustPointerId = null;
      touch.buttons.firePointerId   = null;
      touch.buttons.warpPointerId   = null;
      touch.joystick.activePointerId = null;
      touch.joystick.base.classList.remove('visible');
      touch.joystick.knob.classList.remove('active');
      keys['ArrowUp']    = false;
      keys['ArrowLeft']  = false;
      keys['ArrowRight'] = false;
    }
  }

  touch.buttons.pause.addEventListener('pointerdown', e => {
    setPaused(!state.paused);
    e.preventDefault();
  });
  touch.pauseScreen.addEventListener('pointerdown', e => {
    if (state.paused) setPaused(false);
    e.preventDefault();
  });

  document.addEventListener('visibilitychange', () => {
    if (document.hidden) setPaused(true);
  });
```

- [ ] **Step 3: Initialise `state.paused` and add bail-outs in `update()` and `render()`**

Find the state object initialisation (search for `state = {` or similar around line 480-ish; the exact line varies). Add `paused: false,` to the initial state literal. If state is reinitialised in `startGame()`, add `paused: false` there too.

Then in `update()` at `index.html:704`, immediately after `function update(dt){`, add:

```js
    if (state.paused) return;
```

In `render()` at `index.html:917`, immediately after `function render(){`, add:

```js
    if (state.paused) return;
```

(Both bail out entirely — `requestAnimationFrame(frame)` continues, so resume is instant.)

- [ ] **Step 4: Make screen-shake respect reduced motion**

In `render()` at `index.html:921-925`, wrap the shake block:

```js
    // shake
    let sx=0, sy=0;
    if (state.shake > 0 && !touch.reducedMotion){
      sx = (Math.random()*2-1) * state.shake*0.6;
      sy = (Math.random()*2-1) * state.shake*0.6;
    }
```

- [ ] **Step 5: Manually verify pause + reduced-motion + backgrounding**

1. Touch mode (emulation or real): start a game. Tap PAUSE top-centre → pause screen appears, ship freezes mid-flight. Tap pause screen → resume. Tap PAUSE again → pause. All animations (rotation, asteroid drift) pause and resume cleanly.
2. Background the tab (desktop: switch tabs; iPhone: home-swipe). Return: game is paused, pause screen visible. Tap pause screen → resumes.
3. Turn on "Reduce motion" in OS settings (macOS System Settings → Accessibility → Display → Reduce motion; iOS Settings → Accessibility → Motion). Reload. Play a game; hit an asteroid: no screen shake.
4. Confirm the FIRE auto-fire interval does NOT keep ticking during pause — trigger pause mid-fire, wait 2s, resume; there should be no backlog of shots.

Expected: clean pause/resume, reduced motion honoured, no stuck state.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "touch: scroll suppression + pause mechanism + visibilitychange hygiene

- document touchstart/touchmove preventDefault (gated on body.touch-mode)
  kills iOS rubber-band + overscroll that Pointer Events can't suppress
- state.paused added; update() and render() bail early when true
- PAUSE button toggles, pause-screen tap-to-resume
- visibilitychange(hidden) pauses and fully resets pointer/key state to
  avoid resuming into a stuck ArrowUp or ghost-firing interval
- render() screen-shake skipped when prefers-reduced-motion is set"
```

---

### Task 8: Ship-proximity alpha fade on joystick

**Files:**
- Modify: `index.html` — touch into `render()` or a lightweight per-frame updater

**Rationale:** Polish. Lands last because it's non-essential — the game is fully playable without it. Separate task so a reviewer can read it in isolation.

- [ ] **Step 1: Add a per-frame proximity-alpha update**

Locate `render()` at `index.html:917`. After the `if (state.paused) return;` bail-out from Task 7, but before `ctx.clearRect(...)`, add:

```js
    // Joystick proximity alpha: fade when ship is near the joystick origin
    if (touch.mode === 'touch' && touch.joystick.activePointerId !== null && state.ship && !state.ship.dead) {
      const dx = state.ship.x - touch.joystick.originX;
      const dy = state.ship.y - touch.joystick.originY;
      const d = Math.hypot(dx, dy);
      // Linear fade: alpha 1 at 120px, alpha 0.3 at 60px or closer, alpha 1 beyond 120
      let a;
      if (d >= 120) a = 1;
      else if (d <= 60) a = 0.3;
      else a = 0.3 + ((d - 60) / 60) * 0.7;
      touch.joystick.proximityAlpha = a;
      touch.joystick.base.style.opacity = a;
      touch.joystick.knob.style.opacity = a;
    } else if (touch.joystick.proximityAlpha !== 1) {
      touch.joystick.proximityAlpha = 1;
      touch.joystick.base.style.opacity = '';  // falls back to CSS rule
      touch.joystick.knob.style.opacity = '';
    }
```

Note: `state.ship.x/y` are in canvas CSS-px coords matching `W`/`H`, which match the viewport (since cabinet is passthrough in touch mode). `touch.joystick.originX/Y` are in viewport client coords. These are the same coordinate space in touch-mode (canvas fills viewport). In keyboard mode the whole block is short-circuited by the `touch.mode === 'touch'` guard.

- [ ] **Step 2: Manually verify the fade**

1. Touch mode. Start a game. Anchor the joystick at, say, bottom-left.
2. Fly the ship down toward the joystick. As it approaches, the joystick fades to 30% opacity. Fly away: fades back to 100%.
3. No flicker. Transition is smooth (linear over 60px window).

Expected: proximity fade works in both directions, no performance hit (no dropped frames).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "touch: joystick proximity-alpha fade

When the ship approaches within 120 CSS px of the joystick origin, the
joystick fades linearly to 30% alpha at 60 CSS px. Prevents thumb occlusion
of the ship during gameplay. Only runs in touch mode while a joystick pointer
is active; otherwise falls back to CSS default opacity."
```

---

### Task 9: Manual-test comment block

**Files:**
- Modify: `index.html:~424` (immediately above the existing `// ---------- Input ----------` section header)

**Rationale:** Single permanent comment that documents every path a future maintainer needs to re-verify after touching input code. Tiny but high-value.

- [ ] **Step 1: Add the manual-test comment block**

At `index.html:~424`, immediately before the `// ---------- Touch controller (mode + state) ----------` block from Task 4, add:

```js
  // ========================================================================
  // MANUAL TEST CHECKLIST — run after any change to input or touch code
  // ------------------------------------------------------------------------
  // Desktop (keyboard-only path):
  //   - Arrow keys rotate/thrust; Space fires; H hyperjumps.
  //   - Start and Replay click/tap reach startGame().
  //   - No body.touch-mode class; arcade cabinet renders normally.
  //
  // Touch device (iPhone Safari):
  //   - Page loads with body.touch-mode; cabinet chrome hidden; canvas fills.
  //   - Floating joystick anchors in left 45% on touchdown; rotates ship
  //     left/right; releases cleanly; no stuck ArrowLeft/Right after lift.
  //   - THRUST button: hold = continuous thrust; release stops.
  //   - FIRE button: tap = one shot; hold = ~10 shots/sec.
  //   - WARP button: <100ms tap = nothing; ≥100ms hold = one warp; hold
  //     indefinitely = does not retrigger (must release to warp again).
  //   - PAUSE button pauses; tapping pause screen resumes.
  //   - Multi-touch: joystick + THRUST + FIRE + WARP all simultaneous.
  //
  // Hybrid device (iPad + Bluetooth keyboard, Surface):
  //   - First keydown flips off touch-mode within 1.5s (debounce).
  //   - First touch pointerdown flips back to touch-mode within 1.5s.
  //   - No flicker when alternating rapidly (debounce suppresses).
  //
  // OS integration:
  //   - Backgrounding the tab pauses the game and resets input state.
  //   - prefers-reduced-motion suppresses screen-shake and decorative
  //     animations; gameplay motion (ship, asteroids, bullets) unaffected.
  // ========================================================================
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "touch: add manual-test checklist comment above input section

Single-source-of-truth for what to verify after any input/touch change.
No functional change."
```

---

## Self-review

Applied the writing-plans self-review checklist:

**Spec coverage:**
- Single-file integrity / state consolidation → Task 4 (`touch` object).
- Input flow (joystick, THRUST/FIRE/WARP) → Tasks 5, 6.
- Per-button pointer-capture tracking → Task 6.
- Mode detection + 1500ms debounce + WKWebView fallback → Task 4.
- Rubber-band suppression → Task 7.
- Joystick floating anchor + edge-clamp + visualViewport + Dynamic Island → Task 5.
- Joystick deadzone + active state + proximity alpha → Tasks 5, 8.
- Button visual press + WARP fill + contrast chips → Task 3 (CSS) + Task 6 (class toggle).
- In-game PAUSE button + pause screen → Tasks 3 (DOM) + 7 (handler + `state.paused` + `update`/`render` bail).
- Start/replay overlays positioned via `position: fixed` (no reparenting) → Task 2.
- Viewport meta update + `--sat`/`--sab` root props → Task 1.
- `overscroll-behavior` + `-webkit-touch-callout` + `touch-action` scoping → Tasks 1, 2.
- Reduced motion (CSS + JS flag + screen-shake skip) → Tasks 2 (CSS), 4 (flag), 7 (shake skip).
- `visibilitychange` pause + interval/timer/pointer cleanup → Task 7.
- Canvas ARIA label → Task 3.
- Portrait detection + tip overlay → Tasks 3 (DOM + CSS) + 4 (media-query sync).
- Stale-DPR fix → Task 4.
- Manual-test comment → Task 9.

No gaps.

**Placeholder scan:** No TBDs, no "add appropriate validation", no "write tests for the above" (the plan explicitly uses manual browser verification and says so at the header and in each task).

**Type/name consistency:** `touch.joystick`/`touch.buttons` field names match across Tasks 4, 5, 6, 7, 8. `state.paused` introduced in Task 7 and referenced there (not used in earlier tasks). `touch.reducedMotion` set in Task 4 and read in Task 7. `setInputMode`, `setPaused`, `anchorJoystick`, `resetJoystick` all named consistently. `clamp` helper defined once in Task 5 and not redefined.

No issues found.
