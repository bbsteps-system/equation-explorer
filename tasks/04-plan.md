# Milestone 4 Plan — Celebration, Accessibility, and Release Hardening

**Spec:** [`docs/specs/milestones/04-celebration-accessibility-hardening.md`](../docs/specs/milestones/04-celebration-accessibility-hardening.md)  
**Task list:** [`tasks/04-todo.md`](04-todo.md)  
**Status:** Implemented — Tasks 1–6 are complete and automatically verified. The M3 human acceptance gate (Task 0) and the M4 human release gate (Task 7) remain open.

## Outcome

Replace M3's neutral final handoff with a complete, accessible win experience: a verified-answer completion panel with a focused “Solve another!” path, one reduced-motion-safe confetti burst, and one short synthesized chime that never blocks completion when audio is unavailable. Then run the final responsive, keyboard, direct-file, local-server, reduced-motion, and browser release checks.

M4 does **not** add a new win detector. M3 already implements the shared structural predicate `isFinalFormCandidate` and passes `candidate.isFinalForm` through `analyzeCandidateEquation`. M4 consumes that result and must not duplicate or weaken it.

The milestone remains one vertical behavior delivered in slices:

```text
valid final candidate (M3 predicate)
  → lock final row, preserve history
  → mode = 'completed'
  → render completion panel with verified answer
  → focus panel; launch one confetti burst + one chime
  → Solve another / reset → clear every effect, timer, node, and audio resource
```

## Preconditions

1. **M3 human acceptance must pass.** M3 Tasks 0–7 are complete and automatically verified, but `tasks/03-todo.md` Task 8 remains open. M4's entry gate requires accepted M0–M3. This plan may be reviewed now; implementation must not start until the M3 demo and screen-reader check are signed off.
2. The M4 spec is approved for implementation.
3. The contract decisions below are approved and folded into the M4 spec. The consequential decisions are:
   - completion panel semantics, markup ownership, and focus target;
   - removal of the M3 `#final-status` placeholder;
   - DOM/CSS confetti with a fixed layer and centralized cleanup timer;
   - audio policy: no visible sound toggle in this release;
   - one-time celebration flag reset only by start/reset;
   - effect cleanup during reset and completed-session teardown.
4. No dependency, build step, network request, font, image, analytics, or persistence is added.

## Assumptions

These are treated as fixed unless a reviewer says otherwise:

1. M1–M3 math, token, history, state, focus, and duplicate-submission contracts remain unchanged.
2. The final row's verified answer is already available as `lockedRow.displayEquation` (for example `x = 11 ÷ 2`, `x = 5.5`, or `x = 0`); M4 does not reformat it independently.
3. The application remains one `index.html`; all implementation tasks touch that file plus task evidence.
4. M4 remains synchronous except for promise-returning audio/cleanup calls, which are caught and ignored.
5. `prefers-reduced-motion` is re-read at completion time; no persistent media-query listener is required.

## Proposed implementation contract

### State and one-time semantics

Add the smallest possible M4 state:

```js
const state = {
  // Existing M3 fields remain.
  mode: 'landing' | 'started' | 'completed',
  // ...
  celebrationPlayed: false, // true after the one allowed effect launch in a completed session
};

let celebrationTimer = null; // one cleanup timer for the confetti layer
let audioContext = null;     // lazily created, closed on reset
```

Rules:

- `celebrationPlayed` starts `false`, becomes `true` immediately before launching effects, and is reset to `false` only by `startSession()` and `resetSession()`.
- Completion is still entered only through the M3 final branch of `lockActiveRow`; there is no second win path.
- `render()` remains a pure state-to-DOM projection; it never launches effects.
- Effects are launched after the completed DOM is rendered, so a focus or render pass cannot replay them.

### Completion panel

Replace the M3 `<p id="final-status" class="final-status" …>` placeholder with one semantic panel. The old ID, CSS class, and neutral text are removed.

```html
<section
  id="completion-panel"
  class="completion-panel"
  role="region"
  aria-labelledby="completion-heading"
  aria-describedby="completion-praise completion-answer"
  tabindex="-1"
  hidden
>
  <p class="completion-mark" aria-hidden="true">🎉 ✓</p>
  <h2 id="completion-heading">You solved it!</h2>
  <p id="completion-praise">Amazing work — you kept the equation balanced all the way to the answer.</p>
  <p id="completion-answer" class="completion-answer"></p>
  <button id="solve-another-button" class="primary-button" type="button">Solve another!</button>
</section>
```

Contract:

- `#completion-answer` text is set to the final locked row's `displayEquation`, prefixed by accessible prose only if it reads naturally. Recommended: `The answer is x = 11 ÷ 2.` as the element's full text.
- The fixed praise text is static, not interpolated from user input except the verified equation display.
- The panel is visible only in `mode === 'completed'`; the active editor stays hidden and history stays visible.
- The panel is the focus target, not the heading. `tabindex="-1"`, `aria-labelledby`, and `aria-describedby` let a screen reader read the heading, praise, and answer immediately without a duplicate live-region announcement.
- `#session-feedback` is cleared on completion, so the panel is the only success narration.
- `Solve another!` calls the existing `resetSession()` path. The top “New equation / Reset” control remains available as an alternative reset path.
- The toolbox remains visible but disabled, matching M3's completed state. No new layout mode is introduced.
- The panel uses the existing green success surface, visible border, and high-contrast text; color is never the only signal. The static mark and “You solved it!” text carry the result.

### Rendering changes

- Add DOM queries for the panel, answer element, and Solve another button; remove `finalStatusElement`.
- `render()`:
  - toggles `completionPanelElement.hidden` from `mode === 'completed'`;
  - writes `completionAnswerElement.textContent` from the last final locked row;
  - leaves the active step hidden and Check disabled in completed mode;
  - does not write any live-region text for the completion.
- Replace `finalHandoffMessage()` with a small `finalLockedRow()` / `renderCompletionPanel()` helper.
- In the final branch of `lockActiveRow()`: set `mode = 'completed'`, clear `activeRow`/selection/duplicate guard, clear session feedback, render, focus the panel, then launch effects once.
- Reset and start must run `clearCelebration()` before the normal state reset.

### Confetti design

Use a fixed-position, pointer-transparent DOM layer, not canvas and not requestAnimationFrame.

```html
<div id="confetti-layer" aria-hidden="true"></div>
```

CSS:

```css
#confetti-layer {
  position: fixed;
  inset: 0;
  z-index: 20;
  overflow: hidden;
  pointer-events: none;
}

.confetti-piece {
  position: absolute;
  top: -1rem;
  left: var(--confetti-x);
  width: .55rem;
  height: .9rem;
  border-radius: .15rem;
  background: var(--confetti-color);
  animation: confetti-fall var(--confetti-duration) var(--confetti-delay) ease-in forwards;
}
```

Behavior:

- `CONFETTI_PIECE_COUNT = 42`; `CONFETTI_LIFETIME_MS = 3200`.
- Use a deterministic spread derived from the piece index (left percentage, horizontal drift, rotation, delay, duration, color from a fixed palette). No layout measurement and no random flakiness.
- Create all pieces in one `DocumentFragment`, append once, and clear them with one fallback timer.
- The layer and pieces are `aria-hidden`; they never receive focus or pointer events.
- If `matchMedia('(prefers-reduced-motion: reduce)').matches` is true, create **no** layer and no pieces. The static completion mark, heading, praise, and answer remain.
- `clearCelebration()` removes the layer, clears the timer, and resets handles; it is idempotent and safe during animation.
- The effect must not change document height, width, scroll position, or focus order.

### Audio design

Use one short synthesized fanfare, lazily created from the Check user gesture. No toggle in M4; the spec marks one optional.

- Prefer `window.AudioContext || window.webkitAudioContext`; return silently when neither exists.
- `playCelebrationChime()` is wrapped in `try/catch`; the promise from `resume()` is also caught. Audio failure never changes UI state or logs an error.
- Signal: a bright C-major arpeggio (C5, E5, G5, C6) with a final C-major chord (C6/E6/G6), using triangle waves with exponential gain ramps. It is noticeably louder than the initial quiet chime but keeps combined peak gain below clipping, with a total duration under ~800 ms.
- Create nodes per chime; close or disconnect them as the sound finishes. Oscillators stop automatically.
- Keep a single lazily created `AudioContext` for the session. `clearCelebration()` calls `close()` on reset, catches rejection, and sets the reference to `null`.
- Launch the chime only from the final Check flow, never from render, load, self-checks, or reset.

### Lifecycle and cleanup

```js
function clearCelebration() {
  if (celebrationTimer !== null) {
    clearTimeout(celebrationTimer);
    celebrationTimer = null;
  }
  document.querySelector('#confetti-layer')?.remove();

  if (audioContext && audioContext.state !== 'closed') {
    audioContext.close().catch(() => {});
  }
  audioContext = null;
}
```

- `startSession()` and `resetSession()` call `clearCelebration()` and set `state.celebrationPlayed = false`.
- `resetSession()` continues to clear history, active row, selection, drag, pending focus, duplicate guard, completion mode, and feedback, then focuses the setup input.
- Reset during confetti must remove the layer immediately; lingering timer callbacks must be harmless.
- Re-activating Check or any residual control after completion cannot call `lockActiveRow` again because `mode !== 'started'`, the active row is null, and the final branch is not reachable.
- The one-time guard is defensive rather than the only protection: both mode and `celebrationPlayed` must agree before effects launch.

### Accessibility contract

- Completion panel: named region, heading, praise, verified answer, and a focusable button. Focus lands on the panel; Tab reaches “Solve another!” next.
- No focus trap. Shift+Tab can leave the panel.
- Confetti is decorative and hidden from assistive technology.
- Audio is not the only success signal; visual text and the static mark always communicate success.
- Reduced motion removes moving particles but not success text, answer, or button.
- Error and success live-region behavior from M3 is retained; completion does not double-announce by also setting a status region.
- Touch targets remain at least 44×44 px; the completion button follows the existing primary-button dimensions.
- The panel wraps at 320px without horizontal overflow; the fixed confetti layer cannot widen the document.

### Browser and release hardening

- No new remote resource, `fetch`, `eval`, `Function`, module script, or dependency.
- Use CSS custom properties, CSS keyframes, `DocumentFragment`, `replaceChildren`, optional chaining, and `AudioContext` fallbacks; these match the existing evergreen-browser baseline.
- Check Web Audio naming and catch behavior in Safari; check animation/cleanup behavior in Firefox if available.
- Verify `file://` and `python3 -m http.server 8000 --directory .` operation; audio from `file://` must remain best-effort.
- Verify no uncaught exceptions when AudioContext construction, `resume()`, or `close()` fails.
- Re-run `node --check` on the extracted inline script and `runMathSelfChecks()`.

## Implementation order and slices

The file is edited sequentially; slices are ordered to keep the app working after each task.

1. **Completion panel first, effects off.** Replace the placeholder, render the answer, focus the panel, make Solve another reset. This delivers the accessible outcome even if every effect fails.
2. **Confetti second.** Add the fixed layer, deterministic pieces, timer, and reduced-motion skip. The panel remains functional without it.
3. **Audio third.** Add the lazy chime and failure containment. The panel remains functional without it.
4. **Lifecycle fourth.** Wire one-time semantics, reset cleanup, and start cleanup; test reset during animation/audio.
5. **Polish fifth.** Audit focus, names, contrast, 320px wrapping, and reduced motion.
6. **Evidence sixth.** Automated checks and the final human release checklist.

## Checkpoints

- **After Task 1:** Complete `x = 11 ÷ 2` and `x = 5.5`; the panel, answer, focus, and Solve another work with no confetti/audio. M3 regression cases still pass.
- **After Task 3:** Confetti and audio are independently suppressible; blocked audio and reduced motion still complete. No generated nodes survive 3.2 s or reset.
- **After Task 4:** Double activation, reset during confetti, start from completed, and top-reset all leave clean state.
- **After Task 7:** All M4 acceptance criteria, M3 regressions, and the umbrella final release checklist pass with recorded evidence.

## Risks and mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| M3 human acceptance is still pending | High — M4 entry gate unsatisfied | Keep M4 blocked until `tasks/03-todo.md` Task 8 is signed off; review this plan in parallel only |
| AudioContext creation/resume blocked or throws | Medium — could break completion | Catch synchronously and on promises; audio is optional; test with a throwing/stubbed constructor |
| Confetti timer survives reset | Medium — stray nodes or timer callbacks | One centralized cleanup function; remove layer, clear timer, close context on both reset paths |
| Duplicate celebration from retained handlers | Medium — repeated effects/history | Completed mode + null active row + `celebrationPlayed` guard; test rapid activation |
| Completion panel removed M3 focus/live behavior | High — accessibility regression | Panel is the explicit focus target with labelled/described content; remove the old status element in the same task; run keyboard/SR checks |
| Reduced-motion users still see moving particles | Medium — accessibility failure | Skip layer creation entirely when the media query matches; retain static mark/text |
| Confetti causes page overflow/scroll | Medium — release failure | Fixed layer, `inset: 0`, `overflow: hidden`, `pointer-events: none`; measure `scrollWidth` at 320/desktop |
| Audio degrades keyboard flow | Low | Chime is asynchronous and never awaited; focus is set before/independent of audio |
| Safari lacks unprefixed AudioContext or blocks resume | Medium — sound missing | `webkitAudioContext` fallback; silent catch; record human Safari result |
| New CSS overrides `hidden` on the panel | Low — invisible success | Add an explicit `[hidden] { display: none; }` rule for the completion panel and verify rendered visibility |

## Acceptance mapping

| M4 acceptance criterion | Planned evidence |
| --- | --- |
| `2x = 11` then `x = 11 ÷ 2` locks and celebrates once | UI scenario + panel text + one confetti layer + one chime attempt |
| `x = 5.5` and `x = 11/2` complete exactly | UI scenarios; self-check exact rational assertions retained |
| `2x = 11` never celebrates | Non-final accepted step remains active with no panel/effects |
| `1x = 11/2` is accepted but not final | M3 final predicate regression; panel hidden; active successor exists |
| Audio failure does not break completion/reset | Stubbed/throwing AudioContext test; console clear |
| Reduced motion retains clear success without movement | Emulated `prefers-reduced-motion: reduce`; zero confetti nodes; panel readable |
| Solve another resets and focuses input | Button click and keyboard activation; no stale nodes/timers/history |
| Final release checklist passes | Direct file + server, 320/tablet/desktop, Chrome + available Safari/Firefox, console clean |

## Approval decisions

1. M3 Task 8 human acceptance passes before M4 implementation starts.
2. Completion panel uses a named focusable region (not an auto-announced status), removes `#final-status`, and places focus on the panel.
3. Confetti uses DOM/CSS with 42 deterministic pieces, a 3.2 s cleanup timer, and no creation under reduced motion.
4. Audio uses a bright six-note Web Audio fanfare—a rising C-major arpeggio plus final chord—at a clearly audible level, lazily created, with silent failure handling; **no visible sound toggle** in this release.
5. Toolbox remains visible but disabled on completion; active editor remains hidden; top reset remains available.
6. Confetti and audio launch only after the completed DOM renders, guarded by completed mode and `celebrationPlayed`.