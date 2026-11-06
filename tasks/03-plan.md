# Milestone 3 Plan — Guided Validation and History

**Spec:** [`docs/specs/milestones/03-guided-validation-history.md`](../docs/specs/milestones/03-guided-validation-history.md)  
**Task list:** [`tasks/03-todo.md`](03-todo.md)  
**Status:** Approved and implemented — Tasks 1–7 are complete. Automated self-checks, syntax validation, headless UI flows, 320/500/1280 layout checks, reduced-motion checks, and keyboard-only checks pass. The human M3 demo in `tasks/03-todo.md` Task 8 remains the acceptance gate.

## Outcome

Connect the M2 token editor to the M1 exact math engine so a learner can check each constructed equation. An accepted equation locks, enters chronological history, and produces exactly one editable successor; malformed, nonlinear, identity/contradiction, and answer-changing equations remain editable with recoverable feedback. A final-form candidate (`x = constant`) is detected and locked with a neutral M3 handoff message. Confetti, sound, praise, and the completion panel remain M4.

The milestone is one vertical behavior delivered in thin slices:

```text
start source
  → edit active row (M2)
  → Check
      ├─ invalid / non-unique / inequivalent → feedback, row stays editable
      ├─ equivalent, non-final             → lock, append history, spawn one row
      └─ equivalent, final form            → lock, append history, neutral handoff
```

## Assumptions

These are treated as fixed unless a reviewer says otherwise:

1. M2 is accepted and its token shape, serialization, and interaction paths remain unchanged.
2. M1's exact rational, tokenizer, parser, reducer, and formatter are reused; no grammar or math rule is duplicated.
3. Step 1 remains the static locked source rendering built from `sourceEquation.displayEquation`; `lockedRows` holds accepted steps after Step 1.
4. No dependency, build step, persistence, network behavior, or multi-file split is introduced.
5. The application remains one `index.html`; all slices are sequenced to avoid concurrent edits to that file.

## Preconditions

1. M2 human acceptance is recorded (it is: `tasks/02-todo.md`).
2. The M3 specification is approved for implementation.
3. The contract decisions in this plan are approved and folded into the M3 spec, mirroring the M2 precedent. The most consequential are:
   - history/state schema and mode transitions;
   - pure candidate-validation API and side-aware error result;
   - shared final-form predicate for M3/M4;
   - focus and duplicate-submission rules;
   - how `x = 11/2` is exercised through a `÷`-only workspace.
4. No M4 work (confetti, audio, praise panel, completion CTA) is started.

## Proposed implementation contract

### State and invariants

Extend the existing state rather than replacing it:

```js
const state = {
  // Existing M0/M1/M2 fields remain.
  mode: 'landing' | 'started' | 'completed', // 'completed' is M3's neutral final handoff
  draftEquation: '',
  submittedEquation: '',
  sourceEquation: SourceEquation | null,
  lockedRows: LockedRow[],                    // accepted steps after Step 1, append-only
  activeRow: ActiveRow | null,                // the only editable row while started
  selectedSide: 'left' | 'right',
  selectedToken: { side, tokenId } | null,
  drag: DragState | null,
  feedback: {
    kind: 'info' | 'error',
    message: string,
    side: 'left' | 'right' | null,            // side-local error when applicable
    scope: 'setup' | 'session',               // routes the setup vs session live region
  },
  lastAcceptedCheck: { signature: string, at: number } | null, // duplicate-activation guard
};

/** Locked accepted row; token arrays are snapshots and are never edited. */
const lockedRow = {
  id: string,                 // monotonic row id, e.g. 'row-7'
  leftTokens: Token[],
  rightTokens: Token[],
  displayEquation: string,    // normalized display derived from the parsed candidate
  isFinal: boolean,
};

const activeRow = {
  id: string,
  leftTokens: Token[],
  rightTokens: Token[],
};
```

Invariants:

- **landing:** `sourceEquation === null`, `lockedRows === []`, `activeRow === null`.
- **started:** `sourceEquation !== null`, `activeRow !== null`; `lockedRows` is append-only.
- **completed:** `activeRow === null`; the last `lockedRows` entry has `isFinal: true`.
- Step 1 is the source: `state.submittedEquation` / `state.sourceEquation.displayEquation`. Locked steps begin at Step 2.
- The active step number is `lockedRows.length + 2`.
- A locked row is never rendered as interactive workspace tokens and is never a drop target.
- Locking copies arrays into `lockedRows`; the new active row receives **fresh token IDs** copied from accepted symbols, so selection/focus IDs cannot dangle into history.
- Row and token IDs use monotonic counters, not `Date.now()`.

Why Step 1 stays outside `lockedRows`: the existing static Step 1 article and `submittedEquationElement` already encode the immutable source. Keeping it separate avoids rewriting accepted M2 markup and keeps reset semantics obvious. The visible history is still chronological: source, then `lockedRows`, then the active row.

### Candidate validation API

Add one pure function boundary in the math section. It never reads the DOM, state, focus, or feedback.

```js
/**
 * @returns
 *   { kind: 'error', side: 'left'|'right'|null, error: MathError }
 * | { kind: 'inequivalent' }
 * | { kind: 'accepted', candidate: Candidate }
 *
 * Candidate = {
 *   displayEquation: string,   // formatAst(leftAst) + ' = ' + formatAst(rightAst)
 *   solution: Rational,
 *   isFinalForm: boolean,
 * }
 */
function analyzeCandidateEquation(leftTokens, rightTokens, sourceEquation) { /* ... */ }
```

Reuse the existing pure pieces:

- `serializeTokens(tokens)` for each side;
- `tokenizeEquation(text)` for the M1 character/token limits;
- a new `parseExpressionTokens(tokens)` that parses one side and rejects trailing tokens;
- `reduceAffine(ast)` for exact reduction;
- `sameRational(...)` for solution equality;
- `formatAst(...)` for the locked-row display.

`parseExpressionTokens` must not call the direct-reciprocal recognizer. That recognizer is source-only (`analyzeSourceEquation`), so a workspace candidate containing `÷x` returns M1's `unsupported-rational`.

Validation order is deterministic:

1. Empty left side → `error` with `side: 'left'`.
2. Left tokenize/parse → left-side M1 error.
3. Left reduction → left-side error (`nonlinear`, `divide-by-zero`, `unsupported-rational`).
4. Empty right side → `error` with `side: 'right'`.
5. Right tokenize/parse/reduction → right-side error.
6. Candidate coefficient zero → `not-unique` (identity) or `no-solution` (contradiction), both `side: null`. Preserve the distinct M1 codes; use the shared child-facing message.
7. Candidate solution differs from `sourceEquation.solution` → `kind: 'inequivalent'`.
8. Otherwise → `kind: 'accepted'` with exact solution, display, and final-form flag.

Additional rules:

- Compare only against `sourceEquation.solution`; never recompute the source solution from Step 1 tokens or displayed text.
- The source may be direct-reciprocal; every candidate must reduce as an ordinary linear equation. A candidate copied unchanged from `2 ÷ x = 12` is rejected as `unsupported-rational`.
- Each side goes through the same 512-code-unit / 160-token limits as M1; map `input-too-long` to the affected side and keep the row editable.
- Error start/end spans are internal and side-local; the UI does not display offsets.
- Repeated equivalent steps are allowed by this milestone. No hidden "must change" rule is added; the duplicate guard below is only an anti-double-activation safeguard.

### Final-form predicate (shared with M4)

M3 must decide whether to spawn a successor, so the predicate lives in M3 and M4 consumes it rather than reimplementing it.

```js
function isFinalFormCandidate(leftTokens, rightTokens, rightAffine, sourceSolution) {
  return leftTokens.length === 1
    && leftTokens[0].symbol === 'x'
    && !rightTokens.some((token) => token.symbol === 'x')
    && sameRational(rightAffine.constant, sourceSolution);
}
```

Proposed policy: the right side must be **syntactically x-free**, not merely reduce to a constant. This rejects `x = 11 ÷ 2 + x − x` while accepting `x = 5.5`, `x = 11 ÷ 2`, `x = 11/2` (normalized to `÷`), `x = (11 ÷ 2)`, and `x = 0`. This is stricter than M4's current "reduces to a constant" wording; M4 must be aligned when that spec is amended. This is the main decision requiring explicit reviewer approval.

Canonical left side is an exact token-array predicate, never a display-string match:

- accepted: exactly one token with symbol `x`;
- rejected as final: `1x`, `(x)`, `x + 0`, `2x ÷ 2`, `−x` with a coefficient adjustment, and any other equivalent form.

### Display and history rendering

- `displayEquation` for a locked step is `formatAst(leftAst) = formatAst(rightAst)` from the accepted parse. It is display-only and is never reparsed for math.
- Step 1 remains `state.sourceEquation.displayEquation` rendered through `textContent`.
- Locked rows render as static `article.step-row.step-row--locked` elements with visible step label and equation text. They contain no `.workspace-token` buttons and no pointer handlers.
- Step labels:
  - source: `Step 1 🔒 Locked`;
  - accepted step: `Step N ✓ Locked`;
  - final: `Step N ✓ Solved`.
- The active row label is updated to `Step ${lockedRows.length + 2}` on every render.
- New locked rows receive `data-just-locked="true"` for one success animation; reduced motion suppresses movement while retaining the check icon and text.
- On completion (M3 handoff), the active editor is hidden, the final row is in history, and palette/side/toolbar/trash controls are disabled or hidden. No successor is created.

### Check interaction contract

Add one enabled `Check` button to the active editor (44px minimum, visible text, `type="button"`, accessible label such as "Check this equation").

Handler order:

```js
function checkActiveEquation() {
  if (state.mode !== 'started' || !state.activeRow) return;
  cleanupDrag();
  const rowId = state.activeRow.id;
  const signature = serializeTokens(state.activeRow.leftTokens) + '|'
    + serializeTokens(state.activeRow.rightTokens);

  if (isDuplicateActivation(signature)) return;

  const verdict = analyzeCandidateEquation(
    state.activeRow.leftTokens,
    state.activeRow.rightTokens,
    state.sourceEquation,
  );

  if (state.mode !== 'started' || state.activeRow?.id !== rowId) return; // stale event
  // Apply verdict: failure, inequivalent, accepted, or final handoff.
}
```

Duplicate activation: a synchronous "in-flight" flag alone cannot stop the second click of a double-click, because checking is synchronous and the button is still under the pointer. The plan therefore uses both:

1. disable the Check control at handler entry and restore it after render;
2. reject an identical `signature` within `DUPLICATE_CHECK_WINDOW_MS = 500` of the last accepted check, with a silent no-op.

The window covers double-clicks and rapid keyboard repeats. A deliberate later check of a repeated equivalent equation is still permitted after the window. Reset clears `lastAcceptedCheck`.

Failure behavior:

- Preserve every active token, selection, side, and scroll position.
- Set `feedback.kind = 'error'`, `feedback.side` when known, and the M1 message (or the M3 inequivalence message).
- Mark the affected side (`data-error="true"`) or the active row for row-level errors. Use an icon and text in addition to color.
- Do not append history, lock anything, or create a successor.
- Clear error styling on the next edit or next check.
- Trigger a brief shake when selected and not `prefers-reduced-motion`; motion is never the only signal.

Success behavior (non-final):

1. Snapshot the accepted token arrays into `lockedRows` with the formatted display.
2. Clear `selectedToken`; keep `selectedSide` valid.
3. Copy accepted symbols into a new `activeRow` with fresh token IDs.
4. Set success feedback `"Great equivalent equation!"`; render the new locked row and new active step.
5. Scroll the new active row into view with `block: 'nearest'` (no smooth scrolling) and focus its LHS container.

Final behavior:

1. Append the final row with `isFinal: true`.
2. Set `mode = 'completed'`, `activeRow = null`, clear selection and duplicate guard.
3. Render the neutral handoff (for example: `"You found x = 11 ÷ 2. Celebration comes next."`). This is M3's placeholder, not M4 praise.
4. Hide the active editor and disable palette/side/toolbar/trash controls; keep Step 1 and the final row visible.
5. Focus the neutral final-status element. M4 replaces this presentation without changing the detector.

### Focus and announcements

| Event | Focus target | Announcement |
| --- | --- | --- |
| Side-local syntax/reduction error | Affected `.token-side` container, `tabindex="-1"`, `aria-describedby="feedback"` | Error message in the existing polite live region |
| Row-level error (`not-unique`, `no-solution`, `inequivalent`) | `#session-feedback`, `tabindex="-1"` | Same error message, set before focus |
| Accepted non-final step | New active row LHS container, `tabindex="-1"` | Success message |
| Final handoff | Final-status element, `tabindex="-1"` | Neutral final message |

- `#feedback` remains the setup live region; `#session-feedback` is the session live region. Feedback carries `scope` so a candidate error never marks the setup input invalid or writes into the setup region.
- Do not focus the live region for side-local errors; doing so commonly double-announces.
- Never trap focus.
- Locked-row check icons are `aria-hidden`; the visible text carries the state.
- New-step scrolling must not cause document-level horizontal movement.

### Start, reset, and lifecycle

- `startSession()`: initialize `lockedRows = []`, `lastAcceptedCheck = null`, fresh active row from source tokens, `mode = 'started'`, feedback side `null`.
- `resetSession()`: clear `lockedRows`, active row, selection, drag, pending focus, `lastAcceptedCheck`, feedback side, and completion state; return to `landing`; focus the setup input.
- All editing functions that mutate rows must guard `state.mode === 'started' && state.activeRow`; this protects the completed state and locked history.
- `cleanupDrag()` runs at Check, start, and reset.

### Code organization and constants

Keep the existing section order and extend it:

- exact math/primitives (existing);
- tokenizer/parser (add `parseExpressionTokens`);
- candidate analysis + final-form predicate (new, pure);
- source formatting/self-checks (extend);
- state/render (extend history, step labels, final state);
- token interactions (guard editing);
- Check interaction (new);
- effects (none in M3).

New constants: `DUPLICATE_CHECK_WINDOW_MS = 500`. No timers or asynchronous work are introduced in M3. Two-space indentation, `const` by default, camelCase, JSDoc for verdict/row shapes.

## Dependency graph and delivery slices

```text
M3.0 Approve contract amendment
            │
            ▼
M3.1 Pure candidate validation + expression parser
            │
            ▼
M3.2 Final-form predicate + display formatter + self-checks
            │
            ▼
M3.3 State/history snapshots + start/reset transitions
            │
            ▼
M3.4 Check action + accept path + history rendering (first vertical slice)
            │
            ├─────────────────────┐
            ▼                     ▼
M3.5 Rejection/error path   M3.6 Final-form handoff
            └──────────┬──────────┘
                       ▼
M3.7 Duplicate guard + focus/accessibility/immutability hardening
                       │
                       ▼
M3.8 Release verification and evidence
```

M3.1 and M3.2 are logically independent of UI, but they modify the same `index.html` script; run them sequentially. M3.5 and M3.6 both touch the active-row rendering and must not run concurrently. All slices are single-file changes except documentation.

## Verification checkpoints

1. **Engine checkpoint (after M3.2):** `runMathSelfChecks()` covers candidate acceptance, inequivalence, empty/malformed sides, identity/contradiction, nonlinear, divide-by-zero, reciprocal rejection, limits, all final/non-final forms, and exact rational comparison. No learner-visible behavior changes yet.
2. **Acceptance checkpoint (after M3.4):** `2x - 3 = 8` → build `2x = 11` → one locked Step 2, one active Step 3 prefilled `2x = 11`; step numbering is correct; console is clean.
3. **Rejection checkpoint (after M3.5):** malformed `2x −`, `2x - 3 = 9`, identity, contradiction, nonlinear, divide-by-zero, and reciprocal candidates keep tokens, add no history, and give keyboard-recoverable feedback.
4. **Final checkpoint (after M3.6):** `x = 11 ÷ 2`, `x = 5.5`, and a reciprocal source completed to `x = 1 ÷ 6` lock and enter the neutral handoff; `2x = 11` and `1x = 11/2` do not.
5. **Robustness checkpoint (after M3.7):** double click/Enter repeat creates exactly one row; locked rows are inert; focus lands predictably; 320px and desktop layouts do not overflow or horizontal-scroll.
6. **Release checkpoint (after M3.8):** syntax check, self-checks, full M3 demo script, acceptance checklist, reduced motion, direct file, and static server all pass and are recorded.

## Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| Double-click/Enter creates a duplicate row because the second click acts on the newly spawned identical row | Signature + 500ms post-acceptance window plus synchronous disable; explicitly tested in M3.7. |
| Focus falls to `<body>` when Check locks and replaces its row | Explicit focus targets in the contract; new active LHS or final-status element receives focus after render. |
| Final-form logic diverges between M3 and M4 | One pure detector in M3; M4 consumes it. M4 spec amendment required for the x-free RHS policy. |
| M1 exposes no side parser | Add only a thin `parseExpressionTokens` wrapper around the existing parser; do not duplicate grammar or reducer rules. |
| History stores display strings that later drift from token state | Store token snapshots plus a formatter-derived display; math always uses snapshots, never display text. |
| Locked rows accidentally remain interactive because `renderTokenSide` rebuilds them | Render locked rows with static markup and no `.workspace-token` class; guard all mutation handlers on `mode` and `activeRow`. |
| Reciprocal source users are confused when the copied reciprocal row fails Check | Document and test the explicit `unsupported-rational` candidate path; feedback names the limitation. |
| Color-only error state | Side/row text + icon + live-region message accompany color. |
| Candidate side serialization exceeds M1 limits | Run the existing tokenizer limits per side; keep the row editable with `input-too-long` feedback. |
| Self-checks mutate app state through token ID generation | Candidate self-checks use plain token literals, not `createWorkspaceToken`. |
| Mobile history growth pushes the new row off screen | Scroll the new active row into view with `block: 'nearest'`; tokens wrap inside their side. |
| Repeated equivalent rows are mistaken for a bug | Explicitly allowed by the contract; only rapid duplicate activation is blocked. |

## Acceptance-criteria mapping

| M3 acceptance criterion | Implemented by | Verified by |
| --- | --- | --- |
| Malformed `2x −` keeps tokens, adds no row, gives recovery target | M3.5 | Self-check + demo 3 |
| Changed answer `2x - 3 = 9` rejected without data loss | M3.5 | Self-check + demo 4 |
| `2x = 11` locks and creates exactly one prefilled successor | M3.4 | Engine + acceptance checkpoint + demo 1 |
| Rapid double check cannot duplicate history | M3.7 | Demo 5 |
| Locked rows immutable; only active row editable | M3.3, M3.7 | Demo 6 + code review |
| `x = 11 ÷ 2`, `x = 5.5`, `x = 11/2` accepted exactly | M3.1, M3.2, M3.6 | Self-checks + demo 2 |
| Nonlinear/divide-by-zero/no-unique candidates are safe | M3.5 | Self-checks + manual |
| Reduced-motion and live feedback remain understandable | M3.5, M3.7 | Demo 7 |

`x = 11/2` note: the workspace palette contains `÷`, and M1 normalizes `/` to `÷` in `displayEquation`. The demo therefore exercises either a setup source containing `11/2` (displayed as `11 ÷ 2`) or a workspace-built `11 ÷ 2`. This route must be stated in the amended spec so the criterion is reproducible.

## Out of scope

- Confetti, sound, final praise/call-to-action, completion panel, and final hardening (M4).
- Persistence, session history across reloads, undo, hints, automatic solving, or a single prescribed algebraic route.
- New math notation, variables, rational-equation support, or dependencies.
- Splitting `index.html` into modules.

## Open decisions requiring reviewer approval

1. **RHS final-form policy:** structural x-free RHS (recommended) versus M4's current "reduces to a constant" wording. This changes whether `x = 11 ÷ 2 + x − x` can win.
2. **Duplicate-activation policy:** 500ms silent no-op (recommended) versus a visible "already checked" message. The window length is an implementation constant.
3. **Row-level error focus:** focus `#feedback` (recommended) versus leave focus on Check and rely on the live region.
4. **Final placeholder copy:** exact neutral wording for M3; M4 will replace it.
5. **Repeated equivalent steps:** allowed (recommended, matches solution-only equivalence) versus a progress rule.
6. **Immediate final win:** a source like `x = 5` yields an unchanged Step 2; checking it should be allowed to win (recommended) rather than requiring an edit.

## Definition of ready for implementation

- [ ] M3 spec incorporates or explicitly approves the contract decisions above.
- [ ] This plan and `tasks/03-todo.md` are approved by a human reviewer.
- [ ] The one-file, dependency-free, exact-math, no-persistence boundaries remain unchanged.
- [ ] No M4 work is mixed into implementation.

## Completion rule

M3 is complete only when every M3 acceptance criterion and all seven human demo steps pass, candidate validation is proven by deterministic self-checks and code review to use M1's exact math, history is immutable, and rapid duplicate activation cannot add rows. Only then may M4 replace the neutral final handoff with the completion experience.