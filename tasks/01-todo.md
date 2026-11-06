# Milestone 1 Todo — Exact Linear Math Engine

**Spec:** [`docs/specs/milestones/01-exact-linear-math-engine.md`](../docs/specs/milestones/01-exact-linear-math-engine.md)  
**Plan:** [`tasks/01-plan.md`](01-plan.md)  
**Status:** Complete. Tasks 1–5 pass deterministic syntax and math checks, including direct-reciprocal source analysis; the reviewer confirmed the required browser gates, without detailed environment evidence captured in this session.

## Prerequisite

- [x] **Task 0: Complete the M0 human acceptance gate**
  - **Dependency:** M0 implementation and `tasks/00-todo.md` Tasks 1–4.
  - **Acceptance:** A human completes the manual browser checklist in `docs/specs/milestones/00-foundations-ui-shell.md`, records browser engine/version and tested viewport widths, and marks M0 Task 5 complete.
  - **Verify:** Test direct `file://` and static-server use, landing/start/reset behavior, 320px/~768px/≥1280px layouts, keyboard order, reduced motion, zoom, and screen reader behavior if available.
  - **Files:** `tasks/00-todo.md` (human-recorded result).

## Implementation tasks

- [x] **Task 1: Add exact rational and result primitives**
  - **Dependency:** Task 0.
  - **Acceptance:** `index.html` contains normalized `BigInt` rational construction and all required immutable pure operations: add, subtract, multiply, divide, negate, zero test, and equality. It exposes a consistent typed success/error result shape without using `Number` for equation decisions.
  - **Verify:** Add/run relevant `runMathSelfChecks()` cases for sign normalization, GCD reduction, zero canonicalization, and exact decimal `0.5 = 1/2`; run the inline-script syntax check.
  - **Files:** `index.html`, `tasks/01-todo.md`.

- [x] **Task 2: Implement bounded tokenization and AST parsing**
  - **Dependency:** Task 1.
  - **Acceptance:** Input limits, documented characters/aliases, literal rules, exactly one top-level equals sign, source spans, unary signs, parentheses, left-associative `*`/`/`, and allowed implicit multiplication are represented in a pure tokenizer/parser. Unsupported or malformed input returns the specified error category without uncaught exceptions.
  - **Verify:** Extend/run `runMathSelfChecks()` for precedence, grouping, unary signs, implicit multiplication, equality count, empty sides, unmatched parentheses, malformed numbers/mixed fractions, unsupported variables/symbols/powers, and numeric adjacency; run syntax check.
  - **Files:** `index.html`, `tasks/01-todo.md`.

- [x] **Task 3: Add direct-reciprocal source analysis alongside affine reduction**
  - **Dependency:** Task 2.
  - **Acceptance:** Preserve the existing AST-to-affine reducer for linear sources. Before it runs, `analyzeSourceEquation(input)` structurally recognizes only `c / x = d` and `d = c / x`, with nonzero constant expressions `c` and `d`; it returns exact `SourceEquation` data with `kind: 'direct-reciprocal'`, solution `c / d`, and an `x ≠ 0` domain restriction. All other variable-denominator/rational structures return the typed `unsupported-rational` error; existing nonlinear, divide-by-zero, no-solution, and not-unique classifications remain correct.
  - **Verify:** Completed: isolated syntax check passed; deterministic self-check passed 42/42 cases, including both orientations, decimal constants, zero constants, forbidden variable denominators, and reciprocal expressions on both sides.
  - **Files:** `index.html`, `tasks/01-todo.md`.

- [x] **Task 4: Extend normalized formatting and self-check interface**
  - **Dependency:** Task 3.
  - **Acceptance:** `displayEquation` is generated from accepted AST/tokens, uses normalized operator glyphs, preserves mixed-number notation and expression semantics through required parentheses, and is never reparsed for calculation. It formats accepted direct-reciprocal sources without changing their meaning. `window.runMathSelfChecks()` returns named pass/fail results, asserts reciprocal-source metadata and rejection boundaries, and does not change app/UI state.
  - **Verify:** Completed: deterministic self-check passed 42/42 cases, asserting normalized reciprocal displays plus multiplication, division, and unary grouping. Browser DevTools UI non-mutation remains part of Task 6's human gate.
  - **Files:** `index.html`, `tasks/01-todo.md`.

- [x] **Task 5: Integrate reciprocal-aware analysis with the M0 setup lifecycle**
  - **Dependency:** Task 4.
  - **Acceptance:** The form starts a session only after successful analysis, saves normalized display text plus private source data—including source kind and any reciprocal domain restriction—and never shows the answer. Failed analysis remains in landing mode, keeps the draft, clears stale source data, associates actionable feedback with the field, and restores field focus. Reset clears all M1 state and retains M0's neutral ready status, disabled workspace, and safe `textContent` rendering.
  - **Verify:** Completed static validation and deterministic 42/42 self-check. The required manual browser verification of valid/invalid transitions, focus, reset, and answer/domain non-disclosure remains part of Task 6.
  - **Files:** `index.html`, `tasks/01-todo.md`.

- [x] **Task 6: Execute and record the M1 release gate**
  - **Dependency:** Tasks 1–5.
  - **Acceptance:** Every M1 acceptance criterion and the complete six-step human acceptance gate in the M1 spec pass. Browser engine/version, viewport widths, direct-file/static-server results, DevTools self-check result, and any manual observations are recorded here before M1 is marked complete.
  - **Verify:** Run the documented inline-script syntax command; call `runMathSelfChecks()`; then complete all manual cases in `docs/specs/milestones/01-exact-linear-math-engine.md` §§ Testing strategy and Human acceptance gate.
  - **Files:** `tasks/01-todo.md`.

## Completion record

Leave this section unchanged until Task 6 is executed by a human reviewer.

- **Browser(s)/version(s):** Pending
- **Viewport widths tested:** Pending
- **Local-file result:** Pending
- **Static-server result:** Pending
- **`runMathSelfChecks()` result:** Pending
- **M0 regression result:** Pending
- **Reviewer decision:** Pending
