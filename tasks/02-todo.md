# Milestone 2 Todo — Touch-Friendly Token Workspace

**Spec:** [`docs/specs/milestones/02-touch-token-workspace.md`](../docs/specs/milestones/02-touch-token-workspace.md)  
**Plan:** [`tasks/02-plan.md`](02-plan.md)  
**Status:** Complete — the reviewer confirmed the M2 human browser acceptance gate passed. Detailed browser/version and viewport evidence was not captured in this session. Post-acceptance floating-toolbox and red-variable enhancements were added on 2026-11-07; their automated verification passed and a human visual check remains pending.

## Prerequisites

- [x] **Task 0: Record prerequisite human acceptance gates**
  - **Completion:** The reviewer confirmed M0/M1 browser acceptance. `tasks/00-todo.md` Task 5 and `tasks/01-todo.md` Task 6 now record that confirmation; detailed browser, viewport, and direct-file/static-server evidence was not retained in this session. The M2 spec and plan are approved.
  - **Files:** `tasks/00-todo.md`, `tasks/01-todo.md`, `tasks/02-plan.md`, `tasks/02-todo.md`.

- [x] **Task 1: Resolve M2 interaction contract in the specification**
  - **Completion:** The approved interaction contract in the M2 spec now defines token/selection/drag state, canonical conversion and serialization, thresholds and scroll policy, wrapped insertion, keyboard/focus behavior, and cancellation/reset cleanup.
  - **Files:** `docs/specs/milestones/02-touch-token-workspace.md`, `tasks/02-plan.md`, `tasks/02-todo.md`.

## Implementation tasks

- [x] **Task 2: Add pure token model, source conversion, and serialization**
  - **Completion:** `index.html` implements stable `{ id, symbol, label }` tokens, independent active-row sides, state-based selection, deterministic display-token conversion, and whitespace-free serialization for linear and direct-reciprocal sources.
  - **Verification:** Token-model cases cover `2x − 3 = 8`, `2(x + 1) = 6`, and `2 ÷ x = 12`, including IDs, equality removal, and exact serialization.
  - **Files:** `index.html`, `tasks/02-todo.md`.

- [x] **Task 3: Implement active-row rendering, selection, and non-drag editing**
  - **Completion:** The active editor provides side selection, palette insertion, semantic selected-token controls, deletion, reordering, cross-side movement, focus management, and live feedback. Locked Step 1 has no mutation controls.
  - **Verification:** The reviewer confirmed the keyboard-only editing path as part of the M2 human acceptance gate; detailed key-by-key notes were not retained.
  - **Files:** `index.html`, `tasks/02-todo.md`.

- [x] **Task 4: Implement atomic Pointer Events drag and drop**
  - **Completion:** Palette and existing-token drags use Pointer Events, thresholds, persistent-container capture, preview/marker rendering, wrapped-row insertion, atomic valid-drop commits, and no-op cancellation/outside handling. Palette follow-up clicks are suppressed after a successful drag.
  - **Verification:** The reviewer confirmed mouse and touch/pointer drag paths as part of the M2 human acceptance gate; detailed device notes were not retained.
  - **Files:** `index.html`, `tasks/02-todo.md`.

- [x] **Task 5: Integrate accessibility, responsive behavior, and lifecycle cleanup**
  - **Completion:** Semantic controls, readable labels, live announcements, visible selection/focus/drop states, ≥44px controls, responsive wrapping, reduced-motion rules, and reset/session drag cleanup are implemented.
  - **Verification:** The reviewer confirmed responsive, accessibility, touch-scroll, cancellation, and reset behavior as part of the M2 human acceptance gate; detailed observations were not retained.
  - **Files:** `index.html`, `tasks/02-todo.md`.

- [x] **Task 6: Add deterministic token checks and run static validation**
  - **Completion:** `runMathSelfChecks()` includes pure source-conversion, serialization, insertion, reorder, transfer, deletion, ID, boundary, and cancellation checks. The inline JavaScript syntax check passed; deterministic self-checks passed **51/51** with zero failures.
  - **Files:** `index.html`, `tasks/02-todo.md`.

- [x] **Task 7: Execute and record the M2 human acceptance gate**
  - **Completion:** The reviewer confirmed the M2 human browser acceptance gate passed, including the documented mouse, touch/pointer, keyboard, responsive, accessibility, cancellation/reset, and locked-history paths. Detailed browser/version, viewport, and console/network notes were not retained in this session.
  - **Files:** `tasks/02-todo.md`.

## Post-acceptance enhancement: Floating toolbox (2026-11-07)

- [x] **Task 8: Keep the toolbox reachable while the solving path scrolls**
  - **Completion:** `.toolbox-card` is sticky beneath the top edge while `mode === 'started'`, with safe-area-aware top offset and `z-index: 6`; it returns to the document flow at the top of the session and becomes static in `completed` mode. Below 600px it uses compact padding, hides the eyebrow/helper line, and retains its heading. A `ResizeObserver` publishes `--toolbox-sticky-height`; `scroll-padding-top` and `scroll-margin-top` reserve that height so focused tokens, selectors, feedback, and completion content remain clear. Post-edit focus uses `focus({ preventScroll: true })` plus `scrollIntoView({ block: 'nearest' })`.
  - **Verification:** A 35-check headless Chrome/CDP suite passed at 320×600, 500×700, and 1280×700 CSS px. It confirmed sticky height/top values, compact vs. full header, no horizontal overflow, a real palette-to-side drag while pinned, focused inserted-token visibility, release to static completion mode, zero sticky reservation when complete, same-step regression (24/24), fanfare regression (15/15), syntax check, and `runMathSelfChecks()` 70/70 with no console errors. A human visual check is still pending.
  - **Files:** `index.html`, `docs/specs/milestones/02-touch-token-workspace.md`, `docs/specs/linear-equation-drag-solver.md`, `tasks/02-todo.md`.

- [x] **Task 9: Distinguish the variable `x` from multiplication `×`**
  - **Completion:** The variable renders with a dedicated red (`--variable: #b91c1c`) and heavier weight in palette buttons, active workspace tokens, the submitted source display, every locked equation row, the orientation hint, and the completion answer. Multiplication remains the existing dark text color. Equation display strings are rebuilt with a text-preserving span helper, so layout, `textContent`, copy behavior, and accessible names are unchanged.
  - **Verification:** A 12-check headless Chrome/CDP test confirmed the red CSS token, red palette and workspace `x`, dark `×`, unchanged source/locked equation text, red spans in displayed equations, the orientation hint, and `runMathSelfChecks()` 70/70 with no console errors. Same-step (24/24) and sticky-toolbox (35/35) regressions still pass.
  - **Files:** `index.html`, `docs/specs/milestones/02-touch-token-workspace.md`, `docs/specs/linear-equation-drag-solver.md`, `tasks/02-todo.md`.

## Completion record

The reviewer confirmed the M2 human acceptance gate passed. Detailed environment and observation notes were not retained in this session.

- **Browser(s)/version(s):** Passed; browser/version not recorded.
- **Viewport widths tested:** Passed, including the required responsive coverage; exact widths not recorded.
- **Local-file result:** Passed; detailed observation not recorded.
- **Static-server result:** Passed; detailed observation not recorded.
- **Mouse interaction result:** Passed.
- **Touch/pointer-emulation result:** Passed.
- **Keyboard-only result:** Passed.
- **Accessibility/live-feedback observations:** Passed; detailed observations not recorded.
- **`runMathSelfChecks()` result:** Pure Node harness passed **51/51** with zero failures; browser DevTools result was reviewer-confirmed but not separately recorded.
- **Canceled-drag/reset result:** Passed.
- **Floating-toolbox result (2026-11-07):** Automated 35/35 pass at 320, 500, and 1280 CSS px, including pinned palette drag, focus clearance, compact mobile header, completion release, and no horizontal overflow; human visual confirmation pending.
- **Red-variable result (2026-11-07):** Automated 12/12 pass confirming red `x` in palette, workspace, source/locked equations, orientation, and completion answer; `×` remains dark; text content unchanged; self-checks 70/70. Human visual confirmation pending.
- **Reviewer decision:** Passed — M2 accepted. The floating-toolbox and red-variable enhancements are documented and awaiting a human visual check.
