# Milestone 4 Todo — Celebration, Accessibility, and Release Hardening

**Spec:** [`docs/specs/milestones/04-celebration-accessibility-hardening.md`](../docs/specs/milestones/04-celebration-accessibility-hardening.md)  
**Plan:** [`tasks/04-plan.md`](04-plan.md)  
**Status:** Tasks 1–6 implemented and automatically verified. Task 0's M3 human acceptance prerequisite and Task 7's human release gate remain open.

## Prerequisites

- [ ] **Task 0: Confirm M3 acceptance and approve the M4 contract**
  - **Depends on:** M3 implementation (Tasks 0–7) and the M3 human demo/screen-reader gate.
  - **Acceptance:** `tasks/03-todo.md` Task 8 is signed off; the M4 spec incorporates or explicitly approves the completion-panel markup/focus target, removal of `#final-status`, DOM/CSS confetti and cleanup, no-toggle audio policy, one-time celebration guard, and reset cleanup. The M3 spec notes that M4 replaces the neutral final placeholder and its focus target. The reviewer approves this plan and task list.
  - **Verify:** Review `tasks/04-plan.md`, the amended M4 spec, the M3 supersession note, and the completed M3 Task 8 record; confirm no open decisions remain.
  - **Files:** `docs/specs/milestones/04-celebration-accessibility-hardening.md`, `docs/specs/milestones/03-guided-validation-history.md`, `tasks/03-todo.md`, `tasks/04-plan.md`, `tasks/04-todo.md`.
  - **Implementation note (2026-11-07):** The M4 contract was approved by the explicit implementation request and folded into the M4 spec. The M3 human demo/screen-reader sign-off is still pending, so this prerequisite task remains unchecked.

## Implementation tasks

- [x] **Task 1: Replace the neutral handoff with the completion panel**
  - **Depends on:** Task 0.
  - **Acceptance:** The M3 `#final-status` placeholder is removed. A semantic `#completion-panel` with labelled region, static praise, verified answer from the final locked row, and “Solve another!” is the only completed-state outcome. The panel is hidden outside `completed`, focus lands on the panel, Tab reaches the reset button, and Solve another calls the existing reset path and focuses the setup input. No confetti or audio is added yet. M3 non-final acceptance/rejection behavior is unchanged.
  - **Verify:** Complete `x = 11 ÷ 2` and `x = 5.5`; confirm one panel, answer text, focus, hidden active editor, disabled palette, Solve another reset, and setup focus. Re-run M3 accepted/rejected/final cases and `runMathSelfChecks()`.
  - **Files:** `index.html`, `tasks/04-todo.md`.

- [x] **Task 2: Add reduced-motion-safe DOM/CSS confetti with cleanup**
  - **Depends on:** Task 1.
  - **Acceptance:** Completing a session creates one fixed, pointer-transparent, `aria-hidden` confetti layer with a deterministic set of pieces. No layer or pieces are created when `prefers-reduced-motion: reduce` matches. The layer cannot widen the page, change scroll position, or receive focus. Pieces and the cleanup timer are removed after `CONFETTI_LIFETIME_MS` and immediately on reset. Re-rendering does not create a second layer.
  - **Verify:** Complete a session and inspect layer/piece count; wait past 3.2 s and confirm removal; reset during the animation and confirm immediate removal; emulate reduced motion and confirm zero pieces while the panel remains readable; measure 320/desktop `scrollWidth`.
  - **Files:** `index.html`, `tasks/04-todo.md`.

- [x] **Task 3: Add the resilient Web Audio celebration chime**
  - **Depends on:** Task 1.
  - **Acceptance:** The final Check flow makes one short, low-volume synthesized chime attempt using `AudioContext`/`webkitAudioContext`. Construction, `resume()`, scheduling, and `close()` failures are caught silently; no error UI appears and success still completes. Audio is never attempted from load, render, reset, or self-checks. No visible sound toggle is added. Reset closes the retained context and clears the reference.
  - **Verify:** Complete normally with audio available; repeat with a stubbed constructor that throws, with `resume()` rejecting, and with the API removed; confirm the panel, confetti, reset, and console remain correct. Inspect the reset path for a closed/null context.
  - **Files:** `index.html`, `tasks/04-todo.md`.

- [x] **Task 4: Enforce one-time celebration and reset lifecycle**
  - **Depends on:** Tasks 1–3.
  - **Acceptance:** One completed session launches at most one confetti layer and one chime. Completed mode, null active row, and `celebrationPlayed` all prevent replay from rapid click, keyboard repeat, or programmatic activation. `startSession()` and `resetSession()` clear the timer, layer, audio context, flag, completion panel, history, selection, drag, focus, and feedback without stale callbacks or duplicate handlers. Reset may run at any point during confetti/audio.
  - **Verify:** Double-click and rapid-Enter the final Check; attempt a second Check after completion; reset immediately during confetti; start a new equation from the completed setup form; complete two sessions back-to-back and confirm one effect each.
  - **Files:** `index.html`, `tasks/04-todo.md`.

- [x] **Task 5: Audit completion accessibility and responsive behavior**
  - **Depends on:** Task 4.
  - **Acceptance:** The completion panel has a visible heading, high-contrast text, static success mark, accessible name/description, and a 44px-minimum reset button. Focus is not trapped; Tab order is sensible; the panel does not double-announce alongside `#session-feedback`. Confetti is hidden from assistive technology. At 320px the panel and answer wrap without horizontal scrolling, and reduced motion retains all success meaning.
  - **Verify:** Keyboard-only completion and reset; inspect focus/accessible names; emulate reduced motion; check 320, tablet, and desktop widths; take screenshots for visual review. Screen-reader confirmation remains part of Task 7.
  - **Files:** `index.html`, `tasks/04-todo.md`.

- [x] **Task 6: Run deterministic checks, syntax validation, and headless regression flows**
  - **Depends on:** Tasks 1–5.
  - **Acceptance:** The extracted inline script passes `node --check`; `runMathSelfChecks()` still passes all M0–M3 cases with zero failures; headless Chrome verifies completion, non-final acceptance, rejection, duplicate activation, one-time effects, reset during confetti, audio failure, reduced motion, and 320/500/1280 no-overflow; console has no uncaught exceptions; direct-file and static-server loading both work.
  - **Verify:** Run the commands below and record evidence in this file. Do not commit temporary extraction or test files.
  - **Files:** `index.html`, `tasks/04-todo.md`.

- [ ] **Task 7: Complete the human release gate**
  - **Depends on:** Task 6.
  - **Acceptance:** All M4 acceptance criteria and the eight-step M4 human demo pass, plus the umbrella final release checklist. The run includes integer, decimal, and fraction completions; noncanonical `1x = 11/2` non-win; audio blocked; reduced motion; direct `file://` and Python server; 320/tablet/desktop; keyboard traversal; invalid-source and invalid-step regressions. Current Chromium and at least one of Safari/Firefox are tested when available. Browser versions, widths, unavailable browsers, and any screen-reader limitation are recorded.
  - **Verify:** Execute the M4 demo and umbrella checklist, complete the completion record below, and obtain reviewer sign-off. No commit, publish, or deploy without a separate explicit request.
  - **Files:** `docs/specs/milestones/04-celebration-accessibility-hardening.md`, `tasks/04-todo.md`.

## Checkpoints

- **After Task 1:** Panel/focus/reset behavior is complete without effects; M3 regressions pass.
- **After Task 3:** Confetti and audio are independently suppressible and cannot block completion.
- **After Task 4:** Reset, rapid activation, and repeated sessions leave no stale state.
- **After Task 7:** Release gate complete.

## Verification commands

```sh
# Serve the standalone app
python3 -m http.server 8000 --directory .

# Extract the inline script temporarily for a syntax check (do not commit /tmp output)
python3 - <<'PY' > /tmp/equation-explorer-m4.js
from pathlib import Path
html = Path('index.html').read_text()
script = html.rsplit('<script>', 1)[1].split('</script>', 1)[0]
Path('/tmp/equation-explorer-m4.js').write_text(script)
PY
node --check /tmp/equation-explorer-m4.js
```

In the browser DevTools console:

```js
runMathSelfChecks()
```

## Completion record

Automated evidence collected 2026-11-07. Task 7's human release gate is still required.

- Browser(s)/version(s): Google Chrome 157.0.8103.38 (headless new + CDP) on macOS 26.4.1. Safari and Firefox were not available.
- Viewport widths tested: 320, 500, and 1280 CSS px. The completion panel stayed inside the viewport, its button measured 44.78 px high, and `documentElement.scrollWidth === clientWidth` at every width.
- Local-file result: `file://` load, full main flow, decimal/fraction completions, reduced-motion, audio-failure, and regression flows all passed.
- Static-server result: `python3 -m http.server 8123 --bind 127.0.0.1 --directory .` served the app; `x = 11/2` completed with the panel, answer, and one 42-piece layer. The inline data-URI favicon removed the previous `/favicon.ico` 404 console error.
- Mouse/touch result: Real CDP mouse selection/deletion and real palette→side and token→trash drags passed. A physical touch device was not available; Pointer Events paths are unchanged.
- Keyboard-only result: Typed `x = 5.5`, submitted with Enter, tabbed `select-left → select-right → x token → 5.5 token → Check`, pressed Enter, landed on the completion panel, tabbed to “Solve another!”, and pressed Enter to reset and focus the input.
- Screen-reader result: Not tested with an actual screen reader. Automated inspection confirmed a named `region`, labelled/described content, panel focus, decorative confetti hidden from assistive technology, and no duplicate completion live-region announcement.
- Reduced-motion result: Emulated `prefers-reduced-motion: reduce` produced zero confetti nodes/pieces while the static mark, heading, praise, answer, and button remained; error text/state remained and animation duration was `1e-05s`.
- Audio-available result: Real `AudioContext` completion and reset completed without console errors.
- Audio-blocked/failure result: Throwing constructor, rejected `resume()`, and missing audio API each left the panel, confetti, reset, and history intact with no console errors.
- Console result: No uncaught exceptions or console errors across main, edge, regression, keyboard, direct-file, and static-server passes. No failed local network requests.
- `runMathSelfChecks()` result: 70 passed, 0 failed.
- One-time celebration result: Repeated/programmatic final Check attempts kept one layer, one chime attempt, and two locked rows. Back-to-back completed sessions each received one fresh layer.
- Reset/“Solve another!” result: Reset during confetti removed the layer immediately, cleared the timer's later effect, closed the session's audio context, and focused setup. Starting a new equation from the completed setup form also cleared all effects and history.
- M4 human demo result: Automated equivalents of steps 1–8 pass, including `2x = 11 → x = 11 ÷ 2`, integer `x = 0`, fresh `x = 5.5` and `x = 11/2`, noncanonical `1x = 11/2` accepted but non-win followed by canonical completion, reciprocal-source `x = 1 ÷ 6`, one-time effects, reset, reduced motion, audio failure, direct file/server, three widths, and keyboard traversal. A human should still run the visible demo and screen-reader check.
- Umbrella final release checklist result: Automated equivalents of checks 1–10 pass; the release checklist is not human-signed off.
- Reviewer decision: Pending Task 7.