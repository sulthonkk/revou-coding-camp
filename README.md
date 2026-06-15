# 🐱 Interactive Cat Pet App

A delightful single-page web app where you click a cute animated cat to trigger randomized reactions, floating emoji particles, and synthesized sound effects — all wrapped in a glassmorphism UI.

## Demo

Open `index.html` directly in any modern browser. No build step, no server required.

## Features

- **Animated SVG cat** with 7 unique reactions (purr, head-tilt, blink, stretch, spin, heart-eyes, sleepy) plus a special celebration animation
- **Happiness Meter** — fills up as you pet the cat; triggers a celebration burst every 10 points
- **Pet Counter** — tracks total pets in the current session (up to 999,999)
- **Floating emoji particles** spawn near the cat on every interaction with independent animation timelines
- **Synthesized sound effects** via Web Audio API — no external audio files needed, initialized lazily on first click
- **Mute/unmute toggle** with persistent state for the session
- **Glassmorphism UI** with animated gradient background and decorative blobs
- **Keyboard accessible** — Tab to focus the cat, Enter or Space to pet it
- **Responsive** — works from 320px to 1920px viewport widths

## Tech Stack

| Layer | Technology |
|---|---|
| Markup | HTML5 |
| Styling | Tailwind CSS via CDN + custom CSS keyframes |
| Interactivity | Vanilla JavaScript (ES2020+) |
| Animations | CSS `@keyframes` + Web Animations API |
| Audio | Web Audio API (procedural synthesis) |

No build tools, bundlers, or dependencies to install.

## Architecture

The app is organized into four loosely coupled modules inside `index.html`:

```
AppState       — singleton source of truth (petCount, happinessValue, isMuted, …)
EventBus       — lightweight pub/sub using native CustomEvent on document
AnimationEngine — reaction pool, CSS class toggling, particle spawner, idle loop
SoundManager   — lazy AudioContext init, synthesized clip generation, mute control
UIRenderer     — DOM updates for progress bar, pet counter, mute button
```

Data flow on each click/tap/keypress:
1. Cat click → `EventBus.emit('petAction')`
2. `AppState.pet()` increments counters, checks celebration milestone
3. `AnimationEngine.playReaction()` picks next reaction (no immediate repeat), spawns particles
4. `SoundManager.play()` plays the matching synthesized clip (if unmuted)
5. `UIRenderer` updates the progress bar and counter within 200ms

## Reactions

| Reaction | Emoji | Duration |
|---|---|---|
| Purr | 😸 💕 | 1200ms |
| Head Tilt | 🤔 ✨ | 1000ms |
| Blink | 😌 💫 | 800ms |
| Stretch | 😺 ⭐ | 1500ms |
| Spin | 😵 🌀 | 1000ms |
| Heart Eyes | 😍 ❤️ | 1200ms |
| Sleepy | 😴 💤 | 1400ms |
| **Celebrate** *(milestone)* | 🎉 🌟 | 2000ms |

## Usage

```bash
# Just open the file — no install needed
start index.html        # Windows
open index.html         # macOS
xdg-open index.html     # Linux
```

Or serve it with any static file server:

```bash
npx serve .
# then visit http://localhost:3000
```

## Accessibility

- Cat element has `role="button"`, `tabindex="0"`, and `aria-label="Pet the cat"`
- Mute button has `aria-label` that updates on toggle
- Happiness meter uses `role="progressbar"` with `aria-valuenow`
- Space key calls `preventDefault()` to suppress page scroll
- `prefers-reduced-motion` media query disables non-essential animations

## Performance

- Total file size: ~34 KB (well under the 2 MB budget)
- No external fonts, images, or audio files
- All audio is synthesized at runtime via Web Audio API
- Targets 60fps in Chrome 120+, Firefox 121+, Safari 17+
