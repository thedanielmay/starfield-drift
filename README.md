# Starfield Drift

A neon arcade vector shooter — Asteroids-style gameplay wrapped in a pixel-perfect retro cabinet, playable in any modern browser.

## Overview

Starfield Drift is a single-file browser game that recreates the feel of a classic coin-op cabinet. You pilot a vector-drawn ship through waves of neon asteroids, rotating and thrusting with inertia, firing bullets, and hyperjumping to escape tight spots. Waves grow progressively harder; your high score persists via `localStorage`.

The entire game — engine, physics, audio, HUD, and UI — lives in one `index.html` file with no build step and no external runtime dependencies.

## Tech Stack

| Layer | Detail |
|---|---|
| Rendering | HTML5 Canvas 2D API (device-pixel-ratio aware) |
| Audio | Web Audio API — procedural synth beeps and noise bursts |
| Input | Keyboard (desktop) + Pointer Events floating joystick (touch) |
| Styling | Vanilla CSS with CSS custom properties, `oklch()` colours, and `@keyframes` |
| Fonts | Google Fonts — Orbitron (arcade display), JetBrains Mono (HUD) |
| Storage | `localStorage` for the persistent high score |
| Dependencies | None — zero npm, zero bundler |

## How to Run

Open `index.html` directly in a browser — no server required:

```
open index.html          # macOS
xdg-open index.html      # Linux
# or drag the file into any Chromium / Firefox / Safari window
```

The game resizes automatically to fill the available viewport. On a touch device it drops the cabinet chrome and fills the screen with the touch overlay.

## Controls

### Keyboard (desktop)

| Key | Action |
|---|---|
| `←` / `→` | Rotate ship |
| `↑` | Thrust |
| `Space` | Fire |
| `H` | Hyperjump (teleport to a random position) |
| `Space` on title/game-over | Start / replay |

### Touch (mobile / tablet)

| Control | Action |
|---|---|
| Left-side floating joystick | Rotate ship (tap anywhere in left 45 % to anchor) |
| THRUST button | Continuous thrust while held |
| FIRE button | Tap to fire; hold for auto-fire (~10 shots/sec) |
| WARP button | Hold briefly to hyperjump (hold >= 100 ms, fires once per press) |
| Pause button (top-centre) | Pause / resume |

Backgrounding the tab on a touch device pauses automatically.

## Notable Features

- **Arcade cabinet shell** — neon glow, scanline overlay, floor-grid perspective, and a decorative control deck render in CSS around the canvas.
- **Wave system** — each cleared wave spawns more asteroids (capped at 11); asteroids split into two smaller pieces on hit (large → medium → small).
- **Scoring** — large asteroids: 20 pts · medium: 50 pts · small: 100 pts. High score saved across sessions.
- **Procedural audio** — Web Audio oscillators and noise buffers produce shoot, explosion, and hyperjump sounds without any audio files.
- **Particle system** — thrust exhaust, bullet trails, and explosion bursts all emit coloured particles with velocity decay and alpha fade.
- **Screen shake** — asteroid hits and ship death trigger a canvas-translate shake that decays over time; suppressed under `prefers-reduced-motion`.
- **Adaptive input** — detects touch vs. keyboard at startup and switches dynamically on hybrid devices (1.5 s debounce to suppress flicker).
- **Safe-area support** — `env(safe-area-inset-*)` keeps touch controls clear of Dynamic Island and home-indicator on iPhone.
- **DPR-aware canvas** — re-reads `devicePixelRatio` on every resize; caps at 2× to balance sharpness and fill-rate.

## Status

Active. The main branch ships keyboard gameplay and a full mobile touch-control layer (floating joystick + action buttons). Specs and implementation notes live in `docs/superpowers/`.
