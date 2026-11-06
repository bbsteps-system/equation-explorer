# Milestone 3 Spec: Guided Validation and History

**Status:** In review — implementation and automated verification are complete; the human demo below remains the acceptance gate. Depends on accepted M0, M1, and M2.

## Objective
Connect the token editor to the exact math engine, teaching learners through a repeatable check-feedback-next-step loop. Accepted equations become immutable history; malformed or answer-changing equations remain safely editable with helpful feedback.

## Scope
- Add Check / Next Step behavior to the active row.
- Convert the active LHS/RHS token arrays to expressions and validate syntax via M1 parser/reducer.
- Solve the linear candidate equation and compare its exact rational solution to the stored source solution, including a solution retained from M1’s limited direct-reciprocal source form.
- On valid, non-final input: lock the row, append it to chronological history, show success feedback, and create exactly one editable row prefilled from the accepted token state.
- On an active row that is exactly the previous step, whether the tokens are identical or differ only in notation that normalizes to the same displayed equation, reject the check without adding history or creating a successor.
- On invalid syntax or changed solution: preserve all active tokens, apply accessible error state/brief shake subject to reduced motion, and show a child-friendly corrective message.
- Focus feedback or affected editor in a manner that lets keyboard and screen-reader users understand and recover.
- Detect a final-form candidate for M4, but do not implement confetti/audio/praise yet. M3 may present a neutral “You found `x = …`—celebration comes next” development placeholder only if it does not falsely mark M4 complete.

## Approved implementation contract
Approved with the M3 plan in `tasks/03-plan.md`; this section records the decisions the implementation relies on.

- **State and modes:** the existing M2 state gains `mode: 'landing' | 'started' | 'completed'`, `lockedRows`, `activeRow`, `lastAcceptedCheck`, and `sourceStepSignature`. Step 1 remains the static source row; its serialized LHS|RHS token signature is retained because it is not stored in `lockedRows`. Accepted steps are append-only `lockedRows` entries `{ id, leftTokens, rightTokens, displayEquation, isFinal }`; token arrays are snapshots with fresh IDs, and only `activeRow` is editable. `completed` clears `activeRow`, keeps history visible, and shows the neutral handoff.
- **Candidate API:** one pure function `analyzeCandidateEquation(leftTokens, rightTokens, sourceEquation)` returns `{ kind: 'error', side, error }`, `{ kind: 'inequivalent' }`, or `{ kind: 'accepted', candidate }`. It reuses `serializeTokens`, `tokenizeEquation`, a side-expression parser, `reduceAffine`, exact rational equality, and `formatAst`; it never reads DOM or state.
- **Final-form predicate:** a shared pure predicate requires exactly one canonical `x` token on the left, an `x`-free right side that reduces to the source solution, and exact rational equality. Textual forms such as `1x`, `(x)`, `x + 0`, and `2x ÷ 2` do not qualify; `x = 11 ÷ 2`, `x = 5.5`, and a reciprocal-source `x = 1 ÷ 6` do.
- **History display:** a locked row renders static text from `formatAst(leftAst) = formatAst(rightAst)`; the `11/2` setup route normalizes to the `11 ÷ 2` display form because the palette is division-only.
- **Check lifecycle:** disable Check synchronously at handler entry; reject an identical serialized LHS|RHS signature accepted within `DUPLICATE_CHECK_WINDOW_MS = 500`; reject an active row that is exactly the previous step with “That is exactly the same as the previous step. Change something before checking!” and no history/successor. The previous step is the source Step 1 for the first active row, otherwise the latest locked row. Guard first on the exact token signature (this also handles an unchanged reciprocal source before parsing), then, after successful analysis, guard on the candidate’s canonical `displayEquation` matching the previous step’s display (this catches notation-only changes such as `2x` versus `2 × x`). Accepted steps copy tokens into one locked snapshot and create exactly one fresh active row; final form creates no successor. The rule does not reject genuine mathematical alternatives such as `x + 0 = 11` when their display differs.
- **Focus contract:** side-local syntax/reduction errors focus the affected token side; row-level errors focus the session live region; accepted steps focus the new LHS; final handoff focuses the final-status element. Focus is never trapped, and locked rows stay non-interactive.
- **Start/reset:** start initializes empty history, a fresh active row, the source-step signature, and no duplicate guard; reset clears history, active row, selection, drag, pending focus, duplicate guard, source-step signature, error state, and completion state, then returns to landing.
- **Feedback regions:** `#feedback` is the setup live region and `#session-feedback` is the session live region; feedback carries a `scope` so candidate errors never mark the setup input invalid.
- **M4 supersession:** M4 replaces the neutral `#final-status` placeholder and its focus target with the semantic `#completion-panel`. M3's final-form predicate, final lock, immutable history row, and `completed` mode remain authoritative; M4 consumes them without adding another win path.

## Validation rules
1. Reject an active row that is the previous step exactly, either by serialized LHS|RHS token sequence or by matching canonical display, keeping it editable with same-step feedback.
2. Reject empty LHS or RHS before invoking the parser with “Let’s put something on both sides of the equals sign.”
3. Parse each serialized side. Return the most actionable M1 error category and identify the affected side where possible.
4. Form the candidate equation from valid affine values. Reject candidate identities/contradictions as non-unique.
5. Compare candidate and source solutions by normalized BigInt rational equality only.
6. If the solutions differ, keep the row active and show: “That changes the answer for `x`. Try doing the same operation to both sides!”
7. If solutions match, lock the row. Create one, and only one, subsequent active row unless final-form detection says M4 should complete the session.

A valid equation need not be textually identical to the source or arise from a recognized symmetric operation. Equivalence is defined by same unique solution. The app must not state that it observed “the same operation” unless a future enhancement proves that fact.

## Feedback contract
- Success includes text (“Great equivalent equation!”), a check icon, and a nonessential green/scale animation that can be disabled by `prefers-reduced-motion`.
- Syntax feedback names a correction, such as “This side needs a number or `x` after that operation.” or “Let’s close each set of parentheses.”
- Error presentation uses text/icon/status in addition to color and never erases learner work.
- Checking an unchanged active row says: “That is exactly the same as the previous step. Change something before checking!” and leaves the row editable.
- The Check control prevents duplicate submissions while handling one check, so a double click/Enter cannot create duplicate history rows.
- Feedback uses the existing polite live region; use focus deliberately but do not trap it.

## Non-goals
- Confetti, sound, final praise/call-to-action, and final release hardening (M4).
- Persisting session history.
- Enforcing a single recommended algebraic route or offering an automatic solver/hint sequence.
- Automatically repairing malformed token expressions.

## Acceptance criteria
Automated evidence is recorded in `tasks/03-todo.md`; these boxes remain unchecked until the human demo and review sign-off.

- [ ] Checking `2x −` on one side reports syntax feedback, keeps all tokens editable, adds no history row, and gives a recovery target.
- [ ] Checking an equation with changed answer, such as `2x - 3 = 9` after starting `2x - 3 = 8`, is rejected with answer-preservation feedback and no data loss.
- [ ] Starting `2x - 3 = 8`, then checking `2x = 11`, locks that row and creates exactly one editable next row prefilled with `2x = 11`.
- [ ] Checking an active row that is exactly the same as Step 1 or the latest locked step is rejected with same-step feedback, no new history, and no successor, including when only tokenization differs (`2x` versus `2 × x`); a genuinely different equivalent step is still accepted.
- [ ] Checking the same row twice rapidly cannot create two new active rows or two copies in history.
- [ ] Locked accepted rows cannot be modified, while only the latest active row remains editable through all M2 input paths.
- [ ] Exact-equivalent forms (`x = 11 ÷ 2`, `x = 5.5`, and `x = 11/2`) are accepted when tokenized correctly; comparison does not fail from decimal rounding.
- [ ] An expression that parses but is nonlinear, divides by zero, or produces no unique candidate solution receives safe feedback and remains editable.
- [ ] Success/error state remains understandable with reduced motion; feedback is accessible by keyboard and announced in the live region.

## Human demo script
1. Start `2x - 3 = 8`. In Step 2, use M2 controls to make LHS `2x` and RHS `11`; select Check. Confirm a locked Step 2, success feedback, and one active Step 3 copied from `2x = 11`.
2. In a fresh session, leave Step 2 exactly as Step 1 and Check. Confirm same-step feedback, no new history, and no Step 3; then change it to `2x = 11` and confirm it is accepted.
3. On Step 3, make `x = 11 ÷ 2` and Check. Confirm exact-equivalence success and final-form handoff state; repeat fresh sessions with `x = 5.5` and `x = 11/2`.
4. Start a fresh `2x - 3 = 8`; leave LHS as `2x −` and Check. Confirm a syntax message, shake/error treatment if motion is enabled, retained tokens, and no Step 3.
5. Start fresh and change RHS to `9`; Check. Confirm the inequivalence message, retained editable row, and no new row.
6. Double-click Check or press its activation key twice quickly on a valid row. Confirm only one row is locked and exactly one successor appears.
7. Tab to controls in a failed row and correct the error without a pointer. Confirm feedback/state can be understood with a screen reader if available.
8. Enable reduced motion and repeat one success and one error case; confirm no essential result depends on motion.

## Verification evidence
- Human demonstration captures valid, syntax-error, inequivalence, rapid-repeat, and keyboard cases.
- Console stays free of uncaught errors during checks.
- Code review verifies that M3 uses M1's exact math functions, rather than a new floating-point or DOM-text solution path.

## Exit gate and dependencies
**Entry:** M0–M2 are accepted; this spec is approved.

**Done:** Learners can reliably progress through mathematically equivalent token-built equations while receiving recoverable feedback for wrong work. M4 begins only once history immutability, single-successor behavior, and exact candidate validation are accepted.
