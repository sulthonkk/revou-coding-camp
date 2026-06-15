# Implementation Plan: Interactive Cat Pet App

## Overview

Implement a self-contained single-page web application delivered as a single `index.html` file. The build uses Vanilla JavaScript (ES2020+), Tailwind CSS via CDN, CSS keyframe animations, the Web Animations API, and the Web Audio API. Implementation is incremental: core state and structure first, then animation, then sound, then UI polish, then accessibility and testing.

---

## Tasks

- [ ] 1. Scaffold project structure and core HTML skeleton
  - Create `index.html` with `<!DOCTYPE html>`, `<html lang="en">`, `<head>` (charset, viewport, title), and `<body>`
  - Add Tailwind CSS CDN `<script>` tag — no local copy (Requirement 1.4)
  - Add background gradient and two decorative blob `<div>` elements
  - Add the main glassmorphism glass-panel `<div>` (`backdrop-filter: blur ≥ 8px`, `background-color alpha ≤ 0.4`, `1px border alpha ≤ 0.3`) (Requirement 1.3)
  - Inside the panel: add placeholder elements for mute button (`#mute-btn`), cat stage (`#cat-stage`), happiness meter (`#happiness-meter`, `#progress-bar`, `#happiness-value`), and pet counter (`#pet-count`)
  - Verify the page renders without horizontal overflow between 320px and 1920px viewport widths (Requirement 1.5)
  - _Requirements: 1.1, 1.3, 1.4, 1.5_

- [ ] 2. Implement AppState module
  - [ ] 2.1 Define and initialise the `AppState` singleton object
    - Fields: `petCount` (0), `happinessValue` (0), `isMuted` (false), `lastReactionIndex` (-1), `isAnimating` (false)
    - Implement `AppState.pet()` — increments `petCount` up to 999,999 and `happinessValue` up to 100, returns updated state snapshot
    - Implement `AppState.toggleMute()` — flips `isMuted`, returns new value
    - Implement `AppState.reset()` — restores all fields to initial values
    - _Requirements: 5.2, 6.2, 6.3, 6.5, 4.3_

  - [ ]* 2.2 Write property test — pet counter increments monotonically to cap (Property 3)
    - Use fast-check; generate `n ∈ [0, 100]` pet actions; assert `petCount === Math.min(n, 999999)` after all actions
    - **Property 3: Pet counter increments monotonically to cap**
    - **Validates: Requirements 6.2, 6.5**

  - [ ]* 2.3 Write property test — happiness value always bounded in [0, 100] (Property 4)
    - Use fast-check; generate `n ∈ [0, 200]` pet actions; assert `0 ≤ happinessValue ≤ 100` after every action
    - **Property 4: Happiness value is always bounded in [0, 100]**
    - **Validates: Requirements 5.2**

  - [ ]* 2.4 Write property test — mute toggle state transitions are correct (Property 10)
    - Use fast-check; generate arbitrary sequences of `setMuted(bool)` calls; assert `isMuted` equals the argument of the last call
    - **Property 10: Mute toggle state transitions are correct**
    - **Validates: Requirements 4.3, 4.4**

- [ ] 3. Implement EventBus
  - Create `EventBus` using native `CustomEvent` on `document`
  - Expose `EventBus.emit(eventName, detail?)` and `EventBus.on(eventName, handler)` helpers
  - Wire the cat element's `click`, `keydown` (Enter/Space) event listeners to emit `petAction` via EventBus
  - `keydown` handler must call `event.preventDefault()` for Space to prevent page scroll (Requirement 8.3)
  - _Requirements: 2.1, 8.2, 8.3_

- [ ] 4. Implement the SVG Cat element and idle animation
  - [ ] 4.1 Create the inline SVG Cat element
    - Author the SVG with ≤ 20 distinct visual elements (Requirement 1.2)
    - Set `id="cat"`, `role="button"`, `tabindex="0"`, `aria-label="Pet the cat"` (Requirements 8.2, 8.4)
    - Apply `cat--idle` CSS class by default on page load
    - Center the cat horizontally ±10px and vertically ±10px of the viewport (Requirement 1.1)
    - _Requirements: 1.1, 1.2, 8.2, 8.4_

  - [ ] 4.2 Define idle animation CSS
    - Write `@keyframes catIdle` with a looping cycle of 1500ms–4000ms (Requirement 3.4)
    - Define `.cat--idle { animation: catIdle … infinite }` CSS rule
    - Verify idle animation runs continuously after `DOMContentLoaded`
    - _Requirements: 3.4_

- [ ] 5. Implement AnimationEngine — Reaction Pool and selection
  - [ ] 5.1 Define the ReactionPool array
    - Include at least 7 entries: `purr`, `headTilt`, `blink`, `stretch`, `spin`, `heartEyes`, `sleepy` with their `id`, `label`, `emoji`, `cssClass`, `durationMs`, `soundId` fields (Requirement 2.4)
    - Define the separate `celebration` reaction (`cat--celebrate`, 2000ms) outside the standard pool (Requirement 5.4)
    - _Requirements: 2.4, 5.4_

  - [ ] 5.2 Implement `selectNextReaction(lastIndex)` function
    - If pool is empty return -1; if pool has 1 entry return 0; otherwise pick uniformly at random from `[0..pool.length-1]` excluding `lastIndex` (Requirement 2.5)
    - _Requirements: 2.5_

  - [ ]* 5.3 Write property test — reaction selection never repeats immediately (Property 1)
    - Use fast-check; generate pool size `∈ [2, 20]` and random `lastIndex ∈ [0, size-1]`; assert `result !== lastIndex && result >= 0 && result < poolSize`
    - **Property 1: Reaction selection never repeats immediately**
    - **Validates: Requirements 2.5**

  - [ ]* 5.4 Write property test — single-entry pool always returns index 0 (Property 2)
    - Use fast-check; generate any integer `lastIndex`; assert `selectNextReaction(lastIndex)` with a size-1 pool returns 0
    - **Property 2: Single-entry pool always returns index 0**
    - **Validates: Requirements 2.5**

- [ ] 6. Implement AnimationEngine — reaction and bounce animations
  - [ ] 6.1 Define all reaction CSS keyframe animations
    - Write `@keyframes` blocks and corresponding `.cat--*` classes for all reactions and the celebration reaction (Requirement 3.1)
    - Write `@keyframes catBounce` and `.cat--bounce` — bounce must begin within 50ms and complete within 300ms (Requirement 3.5)
    - Write `@keyframes catCelebrate` for the celebration class with a duration distinct from standard reactions (Requirement 5.4)
    - _Requirements: 3.1, 3.5, 5.4_

  - [ ] 6.2 Implement `AnimationEngine.playReaction(reactionIndex)` and `stopCurrentAnimation()`
    - `stopCurrentAnimation()` cancels the WAAPI `Animation` object and removes all `.cat--*` reaction classes before applying new ones (Requirements 2.3, 3.6)
    - `playReaction()` applies `.cat--bounce` synchronously (same event-handler tick) and the reaction class immediately after (Requirements 2.2, 3.5)
    - After `durationMs` elapses, remove the reaction class and re-apply `.cat--idle` within 300ms (Requirement 3.2)
    - Handle empty pool: return early without modifying state or throwing (Requirement 2.6)
    - _Requirements: 2.2, 2.3, 2.6, 3.2, 3.5, 3.6_

  - [ ]* 6.3 Write property test — bounce animation applied to every pet action (Property 12)
    - Use fast-check; for any pet action trigger (click or keyboard), assert `.cat--bounce` is applied to `#cat` synchronously within the same event-handler tick before next paint
    - **Property 12: Bounce animation applied to every pet action**
    - **Validates: Requirements 3.5**

  - [ ]* 6.4 Write property test — celebration triggers exactly at happiness milestones (Property 5)
    - Use fast-check; generate sequences of pet actions causing happiness to cross multiples of 10; assert `.cat--celebrate` is applied exactly once per crossing and is absent from the standard pool
    - **Property 5: Celebration triggers exactly at happiness milestones**
    - **Validates: Requirements 5.4**

- [ ] 7. Implement AnimationEngine — floating labels and particles
  - [ ] 7.1 Implement `spawnFloatingLabel(label, catCenterX, catCenterY, offset)` and `spawnParticle(emoji, catCenterX, catCenterY)`
    - Each label must be a separate DOM element appended to `#cat-stage` with its own independent animation timeline (Requirement 7.3)
    - Initial position of every label must be within 80px of `#cat`'s geometric center (Requirements 3.3, 7.4)
    - Label offset must ensure no two concurrent labels share the same initial pixel coordinates (Requirement 7.3)
    - Upward translate + opacity-fade animation duration must be 800ms–1200ms (Requirement 7.2)
    - Remove the label element from the DOM on `animationend`; add a `setTimeout` fallback (duration + 200ms) (Requirement 7.5)
    - At least one floating emoji/particle must be spawned on every reaction (Requirement 3.3)
    - _Requirements: 3.3, 7.1, 7.2, 7.3, 7.4, 7.5_

  - [ ]* 7.2 Write property test — concurrent floating labels are spatially distinct and proximate (Property 7)
    - Use fast-check; generate `k ∈ [2, 10]` simultaneous labels; assert all initial positions are unique and all are within 80px of cat center
    - **Property 7: Concurrent floating labels are spatially distinct and proximate**
    - **Validates: Requirements 7.3, 7.4**

  - [ ]* 7.3 Write property test — floating labels removed from DOM after animation (Property 8)
    - Use fast-check; generate `k ∈ [1, 10]` label elements; fire `animationend`; assert no label node remains in the DOM after animation duration + 200ms grace period
    - **Property 8: Floating labels are removed from DOM after animation**
    - **Validates: Requirements 7.5**

- [ ] 8. Checkpoint — core animation and state
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. Implement SoundManager
  - [ ] 9.1 Implement `SoundManager` — lazy init, clip loading, and play
    - `AudioContext` is created only on the first pet action (Requirement 4.6)
    - `loadClip(id, url)` loads and decodes audio into `clips` map; all errors are caught silently — failed clips are absent from the map (Requirement 4.5)
    - `play(id)` is a no-op when `isMuted === true`, when AudioContext is not yet initialized, or when the clip is absent from the map (Requirement 4.3)
    - Each played clip must have a duration ≤ 3 seconds (Requirement 4.1)
    - Wrap `AudioBufferSourceNode.start()` in try/catch; log to `console.warn` on failure (Requirement 4.5)
    - `setMuted(value)` sets `isMuted` without creating side effects
    - _Requirements: 4.1, 4.3, 4.5, 4.6_

  - [ ]* 9.2 Write property test — muted state suppresses all audio output (Property 9)
    - Use fast-check; set `isMuted = true`; generate any `clipId` and `n ∈ [1, 20]` calls to `SoundManager.play(clipId)`; assert `AudioBufferSourceNode.start()` is never called (spy/mock)
    - **Property 9: Muted state suppresses all audio output**
    - **Validates: Requirements 4.3**

- [ ] 10. Implement UIRenderer
  - [ ] 10.1 Implement `UIRenderer.updateHappinessMeter(value)` and `updatePetCounter(value)`
    - `updateHappinessMeter(v)` sets `#progress-bar` CSS width to `v%` and sets `#happiness-value` text to integer `v` (Requirements 5.3, 5.5)
    - Progress bar must use `transition: width 200ms ease` to satisfy the ≤200ms visual update requirement (Requirement 5.3)
    - `updatePetCounter(v)` sets `#pet-count` text to `v`; counter must display values 0–999,999 (Requirements 6.1, 6.5)
    - _Requirements: 5.3, 5.5, 6.1, 6.5_

  - [ ]* 10.2 Write property test — happiness meter DOM always reflects current value (Property 6)
    - Use fast-check; generate `v ∈ [0, 100]`; call `updateHappinessMeter(v)`; assert `#progress-bar` style width equals `v%` and `#happiness-value` textContent equals `String(v)`
    - **Property 6: Happiness meter DOM always reflects current value**
    - **Validates: Requirements 5.3, 5.5**

  - [ ] 10.3 Implement `UIRenderer.setMuteButtonState(isMuted)` and the mute toggle button
    - Mute button (`#mute-btn`) is always visible and displays a distinct icon/label per state (Requirement 4.2)
    - Clicking `#mute-btn` calls `AppState.toggleMute()` and `UIRenderer.setMuteButtonState()` + `SoundManager.setMuted()`
    - Button has `aria-label="Toggle sound"` and updates on each toggle
    - _Requirements: 4.2, 4.3, 4.4_

- [ ] 11. Wire all modules together via EventBus and integrate petAction handler
  - [ ] 11.1 Connect EventBus `petAction` listener to AppState, AnimationEngine, SoundManager, and UIRenderer
    - On `petAction`: call `AppState.pet()`, then `AnimationEngine.selectNextReaction()` (passing `lastReactionIndex`), then `AnimationEngine.playReaction()`, `SoundManager.init()` (first time only) + `SoundManager.play(soundId)`, `UIRenderer.updateHappinessMeter()`, `UIRenderer.updatePetCounter()`
    - Check for happiness milestone (multiple of 10) and trigger celebration reaction accordingly (Requirement 5.4)
    - Whole handler from click timestamp to animation start must complete within 50ms (Requirements 2.2, 7.1)
    - Update `AppState.lastReactionIndex` after each selection
    - _Requirements: 2.1, 2.2, 5.4, 6.2, 7.1_

  - [ ]* 11.2 Write property test — keyboard pet action is equivalent to mouse click (Property 11)
    - Use fast-check; simulate Enter and Space keydown events on focused `#cat`; assert `petCount` and `happinessValue` increment identically to a click event
    - **Property 11: Keyboard pet action is equivalent to mouse click**
    - **Validates: Requirements 8.3**

- [ ] 12. Accessibility hardening
  - Verify `#cat` has `role="button"`, `tabindex="0"`, `aria-label="Pet the cat"` in the final DOM (Requirement 8.4)
  - Verify Tab key order lands on `#cat` before other interactive elements where appropriate (Requirement 8.2)
  - Verify `#mute-btn` has `aria-label` and the label updates on toggle
  - Verify Space key handler calls `event.preventDefault()` to suppress page scroll
  - _Requirements: 8.2, 8.3, 8.4_

- [ ] 13. Performance and asset size validation
  - Inline or minify all CSS and JavaScript so total `index.html` file size + any referenced assets stays ≤ 2MB (Requirement 8.5)
  - Verify no external fonts, images, or audio that push the total over budget
  - Add `prefers-reduced-motion` media query to disable or reduce non-essential animations for users who have opted in (good practice supporting Requirement 8.1)
  - Run a Lighthouse desktop audit and confirm performance score ≥ 80 (Requirement 8.1)
  - _Requirements: 8.1, 8.5_

- [ ] 14. Final checkpoint — Ensure all tests pass
  - Run the full fast-check property test suite (Properties 1–12, ≥ 100 iterations each)
  - Confirm all 12 property tests pass with no counterexamples
  - Ensure all tests pass, ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- The design uses Vanilla JavaScript (ES2020+) — all implementation tasks and test examples are plain JS
- Property-based tests use **fast-check** configured to run ≥ 100 iterations per property
- Each property test file should include the comment tag `// Feature: interactive-cat-pet-app, Property N: <title>` for traceability
- Checkpoints at tasks 8 and 14 ensure incremental validation before integration and at completion
- The `cat--bounce` class must be applied synchronously (same event-handler tick) — test this with Property 12

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["4.1", "5.1"] },
    { "id": 1, "tasks": ["2.1", "3", "4.2", "5.2"] },
    { "id": 2, "tasks": ["2.2", "2.3", "2.4", "5.3", "5.4", "6.1"] },
    { "id": 3, "tasks": ["6.2", "9.1", "10.1"] },
    { "id": 4, "tasks": ["6.3", "6.4", "7.1", "10.2", "10.3", "9.2"] },
    { "id": 5, "tasks": ["7.2", "7.3", "11.1"] },
    { "id": 6, "tasks": ["11.2", "12", "13"] },
    { "id": 7, "tasks": ["14"] }
  ]
}
```
