# Design Document

## Overview

The Interactive Cat Pet App is a self-contained single-page web application (SPA) delivered as a single `index.html` file. It features an animated, interactive SVG cat character that responds to user clicks and taps with randomized animations, floating emoji particles, and optional sound effects. A glassmorphism UI wraps a Happiness Meter and Pet Counter to give users a sense of progression.

The entire implementation uses:
- **HTML5** for markup structure
- **Tailwind CSS via CDN** for utility-based styling and responsive layout
- **Vanilla JavaScript (ES2020+)** for all interactivity, animation orchestration, and state management
- **CSS keyframe animations** and the **Web Animations API** for smooth, interruptible animations
- **Web Audio API** for sound effects, initialized lazily on first interaction

No build tools, bundlers, or server-side components are required. The app is fully functional from a local file or static host.

---

## Architecture

The app is organized into four loosely coupled logical modules, all living within the single `index.html` file (or split into inline `<script>` sections for clarity):

```
┌─────────────────────────────────────────────────────────┐
│                       index.html                        │
│                                                         │
│  ┌──────────────┐    ┌───────────────────────────────┐  │
│  │  AppState    │◄───│         EventBus              │  │
│  │  (singleton) │    │  (click/keyboard dispatcher)  │  │
│  └──────┬───────┘    └───────────────────────────────┘  │
│         │                                               │
│         ▼                                               │
│  ┌──────────────────────────────────────────────┐       │
│  │             AnimationEngine                  │       │
│  │  - Reaction Pool management                  │       │
│  │  - CSS / WAAPI animation orchestration       │       │
│  │  - Particle / floating-label spawner         │       │
│  │  - Idle animation loop                       │       │
│  └──────────────────────────────────────────────┘       │
│                                                         │
│  ┌──────────────────────┐  ┌────────────────────────┐   │
│  │    SoundManager      │  │    UIRenderer          │   │
│  │  - AudioContext lazy │  │  - Happiness Meter     │   │
│  │  - Clip loading      │  │  - Pet Counter         │   │
│  │  - Mute toggle       │  │  - Mute button         │   │
│  └──────────────────────┘  └────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Data Flow

1. User clicks/taps Cat or presses Enter/Space while Cat is focused → **EventBus** fires `petAction` event.
2. **AppState** increments `petCount` and `happinessValue` (capped at 100), checks for celebration milestone.
3. **AnimationEngine** picks the next Reaction (avoiding repeat), spawns particles/labels, plays animation.
4. **SoundManager** plays the matching audio clip (if not muted and AudioContext is initialized).
5. **UIRenderer** updates the progress bar and counter displays within 200ms.

---

## Components and Interfaces

### AppState

A plain JavaScript object acting as the single source of truth for runtime state.

```js
// Conceptual interface
AppState = {
  petCount: number,           // 0 – 999,999
  happinessValue: number,     // 0 – 100
  isMuted: boolean,
  lastReactionIndex: number,  // -1 when no previous reaction
  isAnimating: boolean,
}
```

Methods:
- `AppState.pet()` — increments counters, returns updated state snapshot
- `AppState.toggleMute()` — flips `isMuted`, returns new value
- `AppState.reset()` — resets to initial values (page load state)

---

### AnimationEngine

Responsible for all visual animation on the cat and its surrounding area.

```js
// Conceptual interface
AnimationEngine = {
  reactionPool: Reaction[],       // at least 6 entries
  currentAnimation: Animation | null,

  playReaction(reactionIndex: number): void,
  stopCurrentAnimation(): void,
  playIdle(): void,
  spawnParticle(emoji: string, catCenterX: number, catCenterY: number): void,
  spawnFloatingLabel(label: string, catCenterX: number, catCenterY: number, offset: {x,y}): void,
  selectNextReaction(lastIndex: number): number,
}
```

**Reaction object shape:**
```js
Reaction = {
  id: string,           // e.g. "purr", "heart-eyes"
  label: string,        // emoji/text for floating label, e.g. "❤️"
  emoji: string,        // particle emoji, e.g. "💕"
  cssClass: string,     // applied to cat SVG during reaction
  durationMs: number,   // total animation duration
  soundId: string,      // key into SoundManager clip map
}
```

**Reaction selection algorithm:**
```
selectNextReaction(lastIndex):
  if reactionPool.length === 0: return -1
  if reactionPool.length === 1: return 0
  candidates = [0..pool.length-1] excluding lastIndex
  return random pick from candidates
```

---

### SoundManager

Manages all audio, initialized lazily on the first pet action.

```js
// Conceptual interface
SoundManager = {
  audioContext: AudioContext | null,
  clips: Map<string, AudioBuffer>,
  isMuted: boolean,

  init(): Promise<void>,          // called once on first pet action
  loadClip(id, url): Promise<void>,
  play(id: string): void,         // no-op if muted or clip missing
  setMuted(value: boolean): void,
}
```

All clip loading errors are caught silently; failed clips are simply absent from the map and `play()` is a no-op for missing keys.

---

### UIRenderer

Handles DOM updates for the Happiness Meter, Pet Counter, and mute button.

```js
// Conceptual interface
UIRenderer = {
  updatePetCounter(value: number): void,
  updateHappinessMeter(value: number): void,   // triggers CSS transition on progress bar
  setMuteButtonState(isMuted: boolean): void,  // swaps icon/label
}
```

Updates are synchronous DOM writes, relying on the browser's next paint to reflect changes. The progress bar uses a CSS `transition: width 200ms ease` to satisfy the ≤200ms visual update requirement.

---

### EventBus

A lightweight publish/subscribe dispatcher using native DOM `CustomEvent` on `document`.

```js
EventBus.emit('petAction');
EventBus.on('petAction', handler);
```

This decouples the cat click handler from the animation and sound modules.

---

## Data Models

### ReactionPool (static, defined at startup)

| id          | label | emoji | cssClass              | durationMs | soundId     |
|-------------|-------|-------|-----------------------|------------|-------------|
| purr        | 😸    | 💕    | cat--purr             | 1200       | purr        |
| headTilt    | 🤔    | ✨    | cat--head-tilt        | 1000       | tilt        |
| blink       | 😌    | 💫    | cat--blink            | 800        | blink       |
| stretch     | 😺    | ⭐    | cat--stretch          | 1500       | stretch     |
| spin        | 😵    | 🌀    | cat--spin             | 1000       | spin        |
| heartEyes   | 😍    | ❤️    | cat--heart-eyes       | 1200       | heartEyes   |
| sleepy      | 😴    | 💤    | cat--sleepy           | 1400       | sleepy      |

A `celebration` reaction (triggered at happiness multiples of 10) is defined separately with a unique animation class `cat--celebrate` and duration 2000ms, not present in the standard pool.

---

### AppState (runtime)

| Field              | Type    | Initial | Range       |
|--------------------|---------|---------|-------------|
| petCount           | integer | 0       | 0–999,999   |
| happinessValue     | integer | 0       | 0–100       |
| isMuted            | boolean | false   | —           |
| lastReactionIndex  | integer | -1      | -1 to pool.length-1 |
| isAnimating        | boolean | false   | —           |

---

### CSS Animation Classes

Each reaction maps to a CSS `@keyframes` animation. Reactions are applied to the cat SVG element via class toggling:

```
.cat--purr       { animation: catPurr 1200ms ease forwards }
.cat--head-tilt  { animation: catHeadTilt 1000ms ease forwards }
.cat--blink      { animation: catBlink 800ms ease forwards }
.cat--stretch    { animation: catStretch 1500ms ease-in-out forwards }
.cat--spin       { animation: catSpin 1000ms linear forwards }
.cat--heart-eyes { animation: catHeartEyes 1200ms ease forwards }
.cat--sleepy     { animation: catSleepy 1400ms ease forwards }
.cat--celebrate  { animation: catCelebrate 2000ms ease forwards }
.cat--idle       { animation: catIdle 2500ms ease-in-out infinite }
.cat--bounce     { animation: catBounce 200ms ease-out forwards }
```

The `.cat--bounce` class is the immediate scale/bounce response applied to satisfy the ≤50ms / ≤300ms requirements (Requirement 3.5). It is layered on top of the reaction animation using separate CSS custom properties to avoid conflicts.

---

### DOM Structure (simplified)

```html
<body class="bg-gradient min-h-screen flex items-center justify-center">
  <!-- Background decorative blobs -->
  <div class="blob blob-1"></div>
  <div class="blob blob-2"></div>

  <!-- Main glass panel -->
  <div class="glass-panel flex flex-col items-center gap-6 p-8">

    <!-- Mute button (top-right) -->
    <button id="mute-btn" aria-label="Toggle sound" class="...">🔊</button>

    <!-- Cat stage -->
    <div id="cat-stage" class="relative">
      <svg id="cat" role="button" tabindex="0" aria-label="Pet the cat"
           class="cat cat--idle" ...>
        <!-- max 20 SVG elements -->
      </svg>
      <!-- Particles / floating labels injected here dynamically -->
    </div>

    <!-- Happiness Meter -->
    <div id="happiness-meter" aria-label="Happiness meter">
      <span id="happiness-value">0</span>
      <div class="progress-track">
        <div id="progress-bar" style="width: 0%"></div>
      </div>
    </div>

    <!-- Pet Counter -->
    <div id="pet-counter">
      Pets: <span id="pet-count">0</span>
    </div>

  </div>
</body>
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Reaction selection never repeats immediately

*For any* Reaction Pool of size ≥ 2 and any valid previous reaction index, calling `selectNextReaction(lastIndex)` SHALL return an index that is both a valid pool index and differs from `lastIndex`.

**Validates: Requirements 2.5**

---

### Property 2: Single-entry pool always returns index 0

*For any* Reaction Pool containing exactly one entry, and *for any* value of the previous reaction index (including -1), calling `selectNextReaction` SHALL always return 0.

**Validates: Requirements 2.5**

---

### Property 3: Pet counter increments monotonically to cap

*For any* sequence of n Pet Actions (n ≥ 0) starting from `petCount = 0`, after all actions have been processed, `petCount` SHALL equal `min(n, 999999)`.

**Validates: Requirements 6.2, 6.5**

---

### Property 4: Happiness value is always bounded in [0, 100]

*For any* sequence of Pet Actions of any length, `happinessValue` SHALL always remain in the inclusive range [0, 100]; it SHALL never be negative or exceed 100.

**Validates: Requirements 5.2**

---

### Property 5: Celebration triggers exactly at happiness milestones

*For any* sequence of Pet Actions that causes `happinessValue` to transition across a positive multiple of 10 (i.e., 10, 20, …, 100), the celebration Reaction SHALL be triggered exactly once per crossing, and it SHALL use the `cat--celebrate` animation class which is absent from the standard Reaction Pool.

**Validates: Requirements 5.4**

---

### Property 6: Happiness meter DOM always reflects current value

*For any* `happinessValue` v in [0, 100], after `UIRenderer.updateHappinessMeter(v)` is called, (a) the progress bar's CSS width value SHALL equal `v`% and (b) the numeric text display SHALL show the integer `v`.

**Validates: Requirements 5.3, 5.5**

---

### Property 7: Concurrent floating labels are spatially distinct and proximate

*For any* k ≥ 2 floating labels spawned within the same 800–1200ms animation window, (a) no two labels SHALL share the same initial pixel coordinates, and (b) every label's initial position SHALL be within 80px of the Cat element's geometric center as measured at spawn time.

**Validates: Requirements 7.3, 7.4**

---

### Property 8: Floating labels are removed from DOM after animation

*For any* floating label element appended to the DOM, that element SHALL no longer be present in the DOM after its animation completes (within animation duration + 200ms grace period).

**Validates: Requirements 7.5**

---

### Property 9: Muted state suppresses all audio output

*For any* audio clip id, *for any* number of `SoundManager.play(id)` calls made while `isMuted === true`, no `AudioBufferSourceNode.start()` SHALL be invoked.

**Validates: Requirements 4.3**

---

### Property 10: Mute toggle state transitions are correct

*For any* current mute state `s` ∈ {true, false}, calling `setMuted(!s)` SHALL result in `isMuted === !s`; calling `setMuted(s)` SHALL leave `isMuted === s` unchanged. Any sequence of toggle calls SHALL leave the state equal to the argument of the last call.

**Validates: Requirements 4.3, 4.4**

---

### Property 11: Keyboard pet action is equivalent to mouse click

*For any* focused state of the cat element, pressing Enter or Space SHALL produce an increment to `petCount` and `happinessValue` identical to that produced by a mouse click event on the same element.

**Validates: Requirements 8.3**

---

### Property 12: Bounce animation applied to every pet action

*For any* Pet Action (click, tap, or keyboard trigger), the `cat--bounce` CSS class (or equivalent bounce keyframe) SHALL be applied to the cat element within the same JavaScript event handler execution tick, before the next browser paint.

**Validates: Requirements 3.5**

---

## Error Handling

### Audio Failures

All audio operations are wrapped in try/catch blocks. If `AudioContext` creation, clip loading, or `buffer.start()` fails, the error is logged to `console.warn` and execution continues. The visual reaction proceeds uninterrupted (Requirement 4.5).

### Empty Reaction Pool

If `reactionPool.length === 0` at the time of a Pet Action, the `petAction` handler returns early without modifying any state or throwing. This satisfies Requirement 2.6.

### Animation Interruption

When a new Pet Action arrives while `isAnimating === true`, `AnimationEngine.stopCurrentAnimation()` cancels the Web Animations API `Animation` object (`.cancel()`) and removes any in-progress CSS animation classes before applying new ones. This prevents partially-transformed visual states (Requirements 2.3, 3.6).

### DOM Cleanup for Particles

Floating labels and particles are removed via an `animationend` event listener. A fallback `setTimeout` (duration + 200ms) guards against browsers where `animationend` may not fire reliably, ensuring no DOM leak.

### Keyboard Accessibility Edge Cases

The cat element has `tabindex="0"` so it participates in normal tab order. `keydown` handlers check for `event.key === 'Enter' || event.key === ' '` and call `preventDefault()` on Space to prevent page scroll.

---

## Testing Strategy

### Dual Testing Approach

The testing strategy combines **unit/example-based tests** for deterministic logic and **property-based tests** for universal invariants across randomized inputs.

### Unit / Example-Based Tests

Target the following specific behaviors:

| Test Area | Examples Covered |
|---|---|
| Cat renders centered on load | DOM position within ±10px of viewport center |
| Idle animation applied on load | `cat--idle` class present on `#cat` after DOMContentLoaded |
| Pet counter starts at 0 | `petCount === 0` on page load |
| Celebration triggers at multiples of 10 | happiness reaches 10, 20, 30 |
| Mute button toggles icon | icon changes after click |
| Reaction pool has ≥ 6 entries | `reactionPool.length >= 6` |
| Audio fails gracefully | SoundManager.play with broken URL doesn't throw |
| Keyboard Enter/Space triggers pet action | Simulated keydown events increment counter |
| Floating labels removed after animation | DOM node absent after animationend |

### Property-Based Tests

Use **fast-check** (JavaScript property-based testing library) configured to run **100 iterations minimum** per property.

Each property test references its design property via a comment tag:

```js
// Feature: interactive-cat-pet-app, Property 1: Reaction selection never repeats immediately
```

#### Property Tests Summary

| # | Property | Generator Inputs | Assertion |
|---|---|---|---|
| P1 | No immediate repeat | Pool size ∈ [2, 20], random lastIndex ∈ [0, size-1] | result !== lastIndex and result is a valid index |
| P2 | Single-entry pool always returns 0 | lastIndex ∈ any integer | result === 0 |
| P3 | Pet counter increments monotonically to cap | n ∈ [0, 100] pet actions | petCount === min(n, 999999) |
| P4 | Happiness stays in [0, 100] | n ∈ [0, 200] pet actions | 0 ≤ happinessValue ≤ 100 |
| P5 | Celebration triggers at milestones | n actions crossing multiples of 10 | celebration class applied exactly once per crossing |
| P6 | Happiness meter DOM reflects value | v ∈ [0, 100] | width% === v and textContent === v |
| P7 | Concurrent floating labels spatially distinct & proximate | k ∈ [2, 10] simultaneous labels | all initial positions unique; all within 80px of cat center |
| P8 | Floating labels removed after animation | k ∈ [1, 10] labels, animationend fired | no label nodes in DOM after animation + grace period |
| P9 | Muted state suppresses all audio | any clip id, muted=true, n ∈ [1, 20] play calls | AudioBufferSourceNode.start() never called |
| P10 | Mute toggle state transitions | sequences of setMuted(bool) calls | isMuted equals argument of last call |
| P11 | Keyboard trigger equals mouse click | Enter or Space keydown on focused cat | petCount and happinessValue increment identically to click |
| P12 | Bounce applied to every pet action | any pet action trigger (click/keyboard) | cat--bounce class applied synchronously before next paint |

### Performance & Accessibility Checks

- **Lighthouse audit**: Desktop performance score ≥ 80, total asset weight ≤ 2MB.
- **Manual accessibility**: Tab focus lands on cat, Enter/Space triggers pet action; mute button has `aria-label`; cat has `role="button"` and `aria-label="Pet the cat"`.
- **Frame rate**: Verified via Chrome DevTools Performance panel on animation-heavy interactions (≥ 60fps target).
- **Responsive layout**: Manual resize tests from 320px to 1920px viewport width.
