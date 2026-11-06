# Milestone 1 Plan — Exact Linear Math Engine

**Spec:** [`docs/specs/milestones/01-exact-linear-math-engine.md`](../docs/specs/milestones/01-exact-linear-math-engine.md)  
**Status:** Revised after the approved direct-reciprocal source extension. Existing affine work is complete, but source analysis, self-checks, UI handoff, and release validation must be repeated. Implementation remains blocked until M0 Task 5 is human-approved.

## Outcome

Replace M0's nonblank mock setup rule with a deterministic, dependency-free source-equation analyzer. It must accept documented unique-solution linear equations and the narrow direct-reciprocal source forms `c / x = d` and `d = c / x`, retain their exact private source representation and solution (including `x ≠ 0` for reciprocal sources), render a safe normalized Step 1, and preserve M0's existing preview-only workspace and accessibility behavior.

## Preconditions

1. Complete and record the M0 human browser acceptance gate in `tasks/00-todo.md`.
2. Treat the approved M1 specification as the immutable implementation contract. Any grammar, mathematical, UX, dependency, or architectural change requires a spec update and human approval before implementation.

## Dependency graph and delivery slices

```text
M0 human acceptance gate
          │
          ▼
M1.1 Math foundation (Rational + typed results)
          │
          ▼
M1.2 Tokenizer and parser (tokens → AST)
          │
          ▼
M1.3 Structural direct-reciprocal recognition
          │                         │
          │ reciprocal source       │ ordinary source
          ▼                         ▼
M1.4 Exact reciprocal analysis   M1.5 Affine reducer and solver
          └────────────┬────────────┘
                       ▼
        M1.6 Formatter, self-checks, and M0 UI handoff
                       │
                       ▼
              M1.7 Integrated validation
```

The recognizer is a source-analysis branch, not a general expression reducer. M1.6 changes shared `index.html` UI and test code, so perform its formatter, self-check, and handoff updates as one coordinated change after both analysis paths work. M1.7 is the release slice and runs only after all prior slices.

## Implementation order

### 1. Establish pure exact-math primitives

Add constants and pure helper functions near the beginning of the inline script. Implement normalized immutable-shaped `Rational` values using `BigInt`, plus typed result/error constructors and rational operations. Make all internal failures map to documented `MathError` results at public boundaries.

**Checkpoint:** exact integer, decimal, fraction, and mixed-number values can be represented without `Number` or floating-point arithmetic; zero denominators are rejected predictably.

### 2. Build bounded lexical and syntax analysis

Implement a tokenizer that enforces the 512-code-unit and 160-token limits, recognizes documented operator aliases, identifies unsupported symbols/variables/powers, and recognizes only valid mixed-number forms. Implement a recursive-descent parser that yields an AST with source spans, one top-level equality, unary signs, grouping, left-associative multiplication/division, and explicitly limited implicit multiplication.

**Checkpoint:** parsing produces no DOM effects and distinguishes malformed input categories without throwing into UI event handlers.

### 3. Recognize only the direct-reciprocal source structure

After parsing an equation and before affine reduction, inspect the top-level AST shape. Accept only one side that is exactly a binary division whose right operand is the bare variable `x`, paired with a side that reduces to a constant; reduce the numerator and opposite side independently as constants. Require both constants to be nonzero. Support both `c / x = d` and `d = c / x`; reject all other variable denominators, additional reciprocal terms, and reciprocal expressions on both sides with `unsupported-rational`.

**Checkpoint:** `2/x = 12` and `12 = 2/x` return an exact `1/6` solution, `kind: 'direct-reciprocal'`, and private `x ≠ 0` metadata, while `2/(x+1)=3`, `1/x + 1/(x-1)=2`, and `x/x=1` are rejected predictably.

### 4. Reduce ordinary ASTs to affine expressions and solve exactly

Keep AST-to-affine reduction for ordinary linear sources using the documented component-wise and product/division rules. Reject nonlinear multiplication and ordinary variable denominators; classify zero denominator, no-solution, and not-unique outcomes correctly. Expose `analyzeSourceEquation(input)` as the single pure public analysis boundary returning either `SourceEquation` or `MathError`.

**Checkpoint:** all documented linear examples resolve to the required exact rational numerator/denominator pairs, while identities and contradictions remain distinguishable internally.

### 5. Add semantics-preserving display, deterministic self-checks, and M0 handoff

Generate `displayEquation` from the parsed AST/tokens, with normalized glyphs and parentheses sufficient to preserve precedence, associativity, unary binding, and mixed-number meaning. Store source `kind`, solution, and any `domainRestriction` privately. Add side-effect-free `window.runMathSelfChecks()` with stable named cases covering rational arithmetic, grammar, linear and reciprocal exact solutions, unsupported-rational boundaries, and length limits. Route the setup form through `analyzeSourceEquation`; on success use only normalized `displayEquation` in Step 1, and on failure keep the landing form editable while clearing stale source state.

**Checkpoint:** all self-check cases pass without reading or mutating DOM/application state; valid direct reciprocal sources begin the neutral preview session without revealing the solution, while invalid rational structures do not.

### 6. Validate the integrated milestone and record the human gate

Run the inline-script syntax extraction command, execute `runMathSelfChecks()` in a current browser, and execute each M1 human acceptance case through both `file://` and a static server. Include both direct reciprocal orientations and the documented unsupported-rational rejection cases. Recheck M0 responsive, keyboard, focus, live-region, reduced-motion, and reset behavior at required viewport sizes. Record browser/version, widths, and results in the completion task before marking M1 complete.

**Checkpoint:** all M1 acceptance criteria and manual gate cases pass with no uncaught console error or failed resource.

## Risks and mitigations

| Risk | Consequence | Mitigation |
| --- | --- | --- |
| Implicit multiplication or mixed-number tokenization becomes ambiguous | Valid math can be parsed incorrectly | Tokenize mixed numbers only in the exact documented shape; reject all other numeric adjacency. |
| A direct-reciprocal recognizer accepts a broader rational expression | Unsupported equations can appear valid or have missed domain constraints | Match only one top-level `constant ÷ x` term paired with a nonzero constant; reject every other variable denominator as `unsupported-rational`. |
| Reciprocal domain metadata is discarded | Later validation cannot preserve the source domain contract | Store `kind` and `domainRestriction` with each accepted `SourceEquation`, outside the DOM. |
| A reducer uses `Number` indirectly | Fraction/decimal comparisons become inaccurate | Keep values as normalized `BigInt` rationals; self-check numerator/denominator pairs directly. |
| Division or formatter loses AST grouping | A displayed equation can mean something different | Parse `*` and `/` left-to-right; formatter adds grouping parentheses in every required case. |
| Failed analysis leaves prior successful source state | An invalid setup could enter or contaminate a session | Clear `sourceEquation` before/while reporting every failed setup; assert this in self-check/manual validation. |
| Math changes regress M0 accessibility | Keyboard users lose setup or focus feedback | Keep DOM rendering isolated from pure math and repeat the M0 regression scenarios in the M1 gate. |
| No test framework exists | Regressions lack repeatable coverage | Implement the required browser-callable deterministic self-check and retain a detailed manual browser gate. |

## File-impact plan

| File | Planned change |
| --- | --- |
| `index.html` | Add bounded direct-reciprocal source recognition before affine analysis, exact source metadata, formatter/self-check coverage, and M1 setup-state/UI integration while retaining the one-file architecture. |
| `tasks/01-todo.md` | Mark tasks as they complete and record validation evidence. |
| `docs/specs/milestones/01-exact-linear-math-engine.md` | Update only if implementation reveals an approved contract change; do not change it to compensate for code. |

## Verification checkpoints

1. **Pure math checkpoint:** browser self-check proves exact rational behavior, parser/reducer cases, reciprocal-source metadata, and unsupported-rational boundaries before UI wiring.
2. **Setup-flow checkpoint:** valid linear and direct-reciprocal inputs plus invalid rational paths prove state isolation, normalized display, error association, focus, and no answer disclosure.
3. **Release checkpoint:** script syntax check, `runMathSelfChecks()`, local-file/static-server checks, and the complete M1 human acceptance gate pass in a documented current browser.

## Completion rule

Do not mark M1 complete merely because code exists. It completes only after its self-check suite and every M1 human acceptance-gate step pass, including the direct-reciprocal cases and M0 regression checks, and the reviewer records the result in `tasks/01-todo.md`.
