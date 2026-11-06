# Milestone 3 Todo — Guided Validation and History

**Spec:** [`docs/specs/milestones/03-guided-validation-history.md`](../docs/specs/milestones/03-guided-validation-history.md)  
**Plan:** [`tasks/03-plan.md`](03-plan.md)  
**Status:** Tasks 0–7 complete and automatically verified. Task 8 remains open for the human browser acceptance demo.

## Prerequisites

- [x] **Task 0: Approve the M3 implementation contract**
  - **Depends on:** Accepted M2; M3 spec reviewed.
  - **Acceptance:** The M3 spec incorporates or explicitly approves: state/history schema and mode transitions, `analyzeCandidateEquation` result contract, shared final-form predicate, duplicate-activation rule, focus targets, history rendering, and the `x = 11/2` vs `x = 11 ÷ 2` test route. The reviewer approves this plan and task list.
  - **Verify:** Review `tasks/03-plan.md` and the amended M3 spec; confirm no open decisions remain in the plan’s approval list.
  - **Files:** `docs/specs/milestones/03-guided-validation-history.md`, `tasks/03-plan.md`, `tasks/03-todo.md`.

## Implementation tasks

- [x] **Task 1: Add pure side parsing and candidate validation**
  - **Depends on:** Task 0.
  - **Acceptance:** `index.html` exposes pure `parseExpressionTokens(tokens)` and `analyzeCandidateEquation(leftTokens, rightTokens, sourceEquation)`. The verdict is `error` (with side and M1 code), `inequivalent`, or `accepted` (with exact solution and display equation; the final-form flag is wired in Task 2). It reuses `serializeTokens`, `tokenizeEquation`, `reduceAffine`, `sameRational`, and `formatAst`; applies empty-side, left-before-right, identity/contradiction, limit, and reciprocal-rejection rules; and never reads or mutates DOM/state.
  - **Verify:** Add deterministic `runMathSelfChecks()` cases for accepted `2x = 11`, inequivalent `2x - 3 = 9`, empty/malformed sides, identity, contradiction, nonlinear, divide-by-zero, reciprocal candidates, side attribution, and input limits. Confirm all candidate cases pass and no learner UI changes.
  - **Files:** `index.html`, `tasks/03-todo.md`.

- [x] **Task 2: Add the final-form predicate, candidate display, and final self-checks**
  - **Depends on:** Task 1.
  - **Acceptance:** A pure `isFinalFormCandidate(...)` returns true only for exactly one canonical `x` token on the left, an x-free right side reducing to the source solution, and an exact rational match. Locked-row display comes from `formatAst(leftAst) = formatAst(rightAst)`. Self-checks cover `x = 11 ÷ 2`, `x = 5.5`, `x = 11/2` setup normalization, `x = 0`, reciprocal-source `x = 1 ÷ 6`, and rejection of `1x`, `(x)`, `x + 0`, `2x ÷ 2`, `2x = 11`, and `x = 11 ÷ 2 + x − x`.
  - **Verify:** `runMathSelfChecks()` final-form cases pass with exact numerator/denominator assertions; display strings match expected normalized forms.
  - **Files:** `index.html`, `tasks/03-todo.md`.

- [x] **Task 3: Extend state, immutable history snapshots, and start/reset transitions**
  - **Depends on:** Task 2.
  - **Acceptance:** State adds `lockedRows`, `mode: 'completed'`, structured feedback side, and `lastAcceptedCheck`. `startSession()` initializes an empty history and one active row; `resetSession()` clears history, active row, selection, drag, pending focus, guard, error side, and completion state. Locked snapshots copy token arrays, row/token IDs are monotonic, and new active rows receive fresh token IDs from accepted symbols. Step numbering derives from `lockedRows.length + 2`.
  - **Verify:** Start/reset repeatedly in the browser and inspect state: no stale history, no duplicate IDs, no selection referencing a locked token, and no active row after reset. Confirm the existing M2 editing paths still work.
  - **Files:** `index.html`, `tasks/03-todo.md`.

- [x] **Task 4: Implement the accept path — Check, lock, append history, spawn one successor**
  - **Depends on:** Task 3.
  - **Acceptance:** The active editor has a semantic Check button. Checking a valid non-final equation preserves exact solution equality, locks the row with `✓ Locked`, appends one immutable history entry with formatted display, clears selection, creates exactly one new active row prefilled from accepted tokens, updates step numbering, shows `"Great equivalent equation!"`, and focuses the new LHS. History rows render as static text with no interactive tokens; the source Step 1 stays locked.
  - **Verify:** Start `2x - 3 = 8`, build `2x = 11`, Check. Confirm one locked Step 2, one active Step 3 prefilled `2x = 11`, correct step labels, success feedback, focus location, and clean console. Repeat via keyboard only.
  - **Files:** `index.html`, `tasks/03-todo.md`.

- [x] **Task 5: Implement the rejection path with recoverable feedback**
  - **Depends on:** Task 4.
  - **Acceptance:** Empty side, malformed syntax, nonlinear, divide-by-zero, unsupported rational, identity, contradiction, and inequivalence keep all tokens, selection, and scroll state; append no history; create no successor; set `error` feedback with affected side; mark the side/row with text/icon plus color; and clear error styling on the next edit or check. `2x −` gives syntax feedback; `2x - 3 = 9` gives the exact answer-preservation message; identity/contradiction share the M1 non-unique message while retaining distinct internal codes.
  - **Verify:** M3 demo steps 3–4 plus one keyboard-only correction path. Confirm no row is added, no data is lost, live feedback is announced, and reduced motion keeps the message and icon understandable without shake.
  - **Files:** `index.html`, `tasks/03-todo.md`.

- [x] **Task 6: Implement the final-form handoff**
  - **Depends on:** Task 4.
  - **Acceptance:** An accepted final candidate locks with `✓ Solved`, appends one final history row, sets `mode = 'completed'`, clears `activeRow`/selection, creates no successor, shows the approved neutral M3 placeholder, hides the active editor, and disables palette/side/toolbar/trash controls. `2x = 11` and `1x = 11/2` never finalize; `x = 11 ÷ 2`, `x = 5.5`, and a reciprocal source completed to `x = 1 ÷ 6` do. Focus moves to the final-status element.
  - **Verify:** M3 demo step 2, plus reciprocal-source completion and the `1x = 11/2` rejection case. Confirm no duplicate panel or effect and no M4 praise/audio/confetti.
  - **Files:** `index.html`, `tasks/03-todo.md`.

- [x] **Task 7: Harden focus, announcements, duplicate activation, and locked-row immutability**
  - **Depends on:** Tasks 5 and 6.
  - **Acceptance:** Double click, rapid Enter, or programmatic repeated activation cannot create duplicate rows; the 500ms signature guard, previous-step exact-token/canonical-display guard, and synchronous Check disable are implemented. An unchanged first or later active row is rejected with same-step feedback and no history/successor, including notation-only variants such as `2x` versus `2 × x`, while a genuinely different equivalent step still advances. Side-local errors focus the affected side; row-level errors focus feedback; accepted steps focus the new LHS; final handoff focuses final status. Live messages do not double-announce, focus is never trapped, new rows scroll into view with `block: 'nearest'`, and locked rows resist click, keyboard, pointer, and programmatic mutation paths.
  - **Verify:** M3 demo steps 5–7; attempt double click and key repeat; tab through all controls; test a screen reader if available; confirm 320px and desktop layouts do not horizontally scroll.
  - **Files:** `index.html`, `tasks/03-todo.md`.

- [ ] **Task 8: Run deterministic checks, syntax validation, and the M3 human acceptance gate**
  - **Depends on:** Tasks 1–7.
  - **Acceptance:** `runMathSelfChecks()` passes all prior M0–M2 cases plus new candidate/final cases with zero failures; the extracted inline script passes `node --check`; all M3 acceptance criteria and all eight human demo steps pass at 320px, tablet, and desktop widths; direct `file://` and `python3 -m http.server 8000 --directory .` both work; console has no uncaught errors; reduced motion is verified; and evidence is recorded.
  - **Verify:** Run the syntax extraction, `runMathSelfChecks()`, and the M3 demo script. Record browser engine/version, widths, direct-file/server result, keyboard result, reduced-motion result, self-check count, and any unavailable browser.
  - **Files:** `index.html`, `tasks/03-todo.md`.

## Verification commands

```sh
# Serve the standalone app
python3 -m http.server 8000 --directory .

# Extract the inline script temporarily for a syntax check (do not commit /tmp output)
python3 - <<'PY' > /tmp/equation-explorer-m3.js
from pathlib import Path
html = Path('index.html').read_text()
script = html.rsplit('<script>', 1)[1].split('</script>', 1)[0]
Path('/tmp/equation-explorer-m3.js').write_text(script)
PY
node --check /tmp/equation-explorer-m3.js
```

In the browser DevTools console:

```js
runMathSelfChecks()
```

## Completion record

Automated evidence collected 2026-11-06. Task 8's human acceptance demo is still required.

- Browser(s)/version(s): Google Chrome 157.0.8103.38 (headless new + CDP) on macOS 26.4.1. Safari and Firefox were not available.
- Viewport widths tested: 320, 500, and 1280 CSS px. At each width `documentElement.scrollWidth === clientWidth`; the toolbox measured above the solving path at all three.
- Local-file result: `file://` load and the full start → edit → Check → final flow passed.
- Static-server result: `python3 -m http.server 8010 --directory .` returned 200 and the mouse-driven flow passed. Port 8000 was occupied by another process in this environment, so 8010 was used.
- Mouse/touch result: Real CDP mouse clicks on Start and Check passed. A physical touch device was not available; M2 pointer/touch paths are unchanged.
- Keyboard-only result: Tab from the input through Start, side controls, tokens, and Check completed `x = 11/2` and focused `#final-status`. A second keyboard-only run deleted a token, received side-local syntax feedback, recovered through the palette, and accepted the corrected equation. Token selection now restores focus after re-render.
- Reduced-motion result: `prefers-reduced-motion: reduce` reduced animations to `1e-05s`; the error text, affected-side state, and recovery focus remained present.
- Console result: No uncaught exceptions or console errors during load, Check success, rejection, reset, or final handoff.
- `runMathSelfChecks()` result: 70 passed, 0 failed, including accepted/inequivalent/error candidates, final-form positives and negatives, reciprocal sources, exact-solution assertions, and token limits.
- M3 demo script result: Automated equivalents of demo steps 1–8 pass: lock + single successor; unchanged-step rejection and recovery; final `x = 11 ÷ 2`, `x = 5.5`, and `x = 11/2`; syntax and inequivalence retention; rapid duplicate Check no-op; keyboard recovery; reduced motion. A human should still run the visible demo and screen-reader check.
- Same-step guard result (2026-11-07): A 24-check CDP browser pass confirmed unchanged Step 2 and unchanged Step 3 (after 650 ms) are rejected with same-step feedback, no history/successor, and preserved tokens; notation-only variants (`2x − 3 = 8` for source `2 × x − 3 = 8`, and `2 × x = 11` for locked `2x = 11`) are also rejected via canonical display; an unchanged reciprocal source is rejected before rational parsing; equivalent but textually different `2x + 0 = 11` is accepted; normal and reciprocal final forms still complete; reset clears the guard; console is clean; `runMathSelfChecks()` remains 70/70.
- Reviewer decision: Pending human acceptance.