# Milestone 1 Spec: Exact Linear Math Engine

**Status:** Revised draft — requires human approval before implementation. Depends on accepted M0.

## Objective
Make Equation Explorer's setup flow mathematically trustworthy. M1 replaces M0's “any non-blank text” preview rule with an offline, deterministic parser, exact affine-arithmetic engine, and narrowly bounded direct-reciprocal source analyzer. A session starts only when the entered equation uses the supported notation and either is linear in exactly one lowercase `x` with one unique real solution, or matches the documented direct-reciprocal source form with one unique nonzero solution.

A successful M1 session shows a normalized, read-only Step 1 and the existing disabled M0 preview workspace. It deliberately does **not** reveal the answer or enable token editing/checking; those belong to M2 and M3.

## Decisions and scope
- Keep the single dependency-free `index.html`; use strict-mode vanilla JavaScript and no network requests.
- Parse equation text with project code only. `eval()`, `Function()`, dynamic imports, third-party evaluators, and binary floating-point arithmetic are prohibited.
- Use `BigInt` rational values throughout parsing, affine reduction, and solution comparison.
- Support exactly one equation and only the lowercase variable name `x`; `x` may occur any number of times in a linear expression. After exact affine reduction, the equation must have a non-zero coefficient of `x` and therefore one unique real solution. Constant-only equations and equations whose variable terms cancel are rejected as no-solution or not-unique, as applicable.
- Also support the direct-reciprocal **source-only** forms `c / x = d` and `d = c / x`, where `c` and `d` are nonzero constant expressions. Analyze these structurally after parsing and before ordinary affine reduction: retain the exact solution `c / d` and the domain restriction `x ≠ 0`. This is an exception for one reciprocal term, not general rational-equation support.
- M1 owns source-input validation, normalized Step 1 rendering, source-equation state (including reciprocal domain restrictions), parser/rational self-checks, and child-friendly setup errors.
- M1 must preserve the M0 layout, reset lifecycle, responsive behavior, focus behavior, and disabled preview-only palette/workspace. It must not add drag, token editing, later-step validation, history advancement, sound, or confetti.

## Non-goals
- Variables other than lowercase `x`; inequalities; systems; exponent syntax; scientific/repeating decimals; functions; constants such as `π`; percentages; or units.
- General rational equations or variable denominators beyond the direct-reciprocal source forms: reject `2 / (x + 1) = 3`, `1 / x + 1 / (x - 1) = 2`, `x / (x + 1) = 2`, `x / x = 1`, and reciprocal expressions on both sides.
- Equations with no solution or infinitely many solutions, even when their algebra is otherwise valid.
- Revealing or rendering the computed answer in visible learner UI before M3 validates a learner-created final step.
- A build tool, package manager, external test runner, external math/font/CDN asset, persistence, or server API.

## Commands
```sh
# Serve the standalone app for browser review
python3 -m http.server 8000 --directory .

# Open the served app on macOS
open http://localhost:8000/

# Inspect the inline script syntax without changing project files
python3 - <<'PY' > /tmp/equation-explorer-m1.js
from pathlib import Path
html = Path('index.html').read_text()
script = html.rsplit('<script>', 1)[1].split('</script>', 1)[0]
Path('/tmp/equation-explorer-m1.js').write_text(script)
PY
node --check /tmp/equation-explorer-m1.js
```

No dependency installation, build, lint, or automated-test command is approved for this milestone. `runMathSelfChecks()` is the required developer-facing deterministic check described in [Testing strategy](#testing-strategy).

## Project structure
```text
index.html                                      # Application, parser, exact math, and M0/M1 UI
CLAUDE.md                                       # Project-wide constraints
docs/specs/
  linear-equation-drag-solver.md                # Umbrella product specification
  milestones/
    00-foundations-ui-shell.md                  # Accepted M0 contract
    01-exact-linear-math-engine.md              # This M1 contract
tasks/
  00-plan.md, 00-todo.md                        # M0 execution record
```

M1 may reorganize sections inside the inline script but must not split the application into files.

## Supported input contract

### Characters and normalization
The lexer accepts ASCII digits `0`–`9`; ASCII whitespace; `x`; parentheses; `=`; `+`; ASCII hyphen-minus `-` or Unicode minus `−`; ASCII asterisk `*` or multiplication sign `×`; and slash `/` or division sign `÷`. All other characters are rejected before parsing, except Unicode superscript two (`²`), which receives the specific unsupported-power/nonlinear diagnostic below.

Whitespace is insignificant except inside an explicitly recognized mixed-number literal. Input is otherwise case-sensitive: `X` and `y` are unsupported variables.

### Grammar
```text
equation        := expression "=" expression
expression      := term (("+" | "-") term)*
term            := unary (("*" | "×" | "/" | "÷" | implicitMultiply) unary)*
unary           := ("+" | "-")* primary
primary         := number | "x" | "(" expression ")"
number          := integer | decimal | mixedNumber
integer         := digit+
decimal         := digit+ "." digit+
mixedNumber     := integer whitespace+ integer "/" integer
implicitMultiply := allowed adjacent primary pair
```

A slash or division sign is an infix division operator; `11/2` therefore parses through ordinary division and exactly equals `11 ÷ 2`. Multiplication and division share one precedence level and are parsed strictly left-to-right: `8/2*3` means `(8/2)*3`, and `8/2/2` means `(8/2)/2`. Mixed numbers are a distinct lexical form only when the exact shape `whole whitespace numerator/denominator` occurs, e.g. `5 1/2`. A mixed-number denominator cannot be zero and the fractional component must be proper (`0 <= numerator < denominator`). Thus `5 3/2` is rejected rather than silently interpreted as multiplication.

Decimal literals require digits on both sides of the decimal point: `0.5` is valid; `.5`, `5.`, and `1.2.3` are malformed. Leading zeroes are permitted. Unary signs belong to the expression grammar, so `-5.5`, `-(x + 1)`, and `--x` are valid.

Implicit multiplication is inserted only for these adjacent primary-token boundaries, with whitespace allowed except where a mixed number has already consumed it:

| Left primary ends with | Right primary starts with | Example | Result |
| --- | --- | --- | --- |
| number | `x` or `(` | `2x`, `2 (x+1)` | multiply |
| `x` | `(` | `x(x+1)` | multiply, then nonlinear rejection |
| `)` | number, `x`, or `(` | `(x+1)2`, `(x+1)x` | multiply, then reduce/reject as applicable |

Adjacent numeric primaries such as `2 3`, `2.5 4`, and `x 2` are rejected as missing an operator rather than guessed. A `)` followed by a signed/unary operator is handled normally, not as implicit multiplication. Input may contain exactly one `=`; any `=` inside parentheses is also invalid because equality is not an expression token.

### Examples

| Input | Expected result |
| --- | --- |
| `2x - 3 = 8` | accepted; solution `11/2` held privately |
| `3(x + 2) = 15` | accepted; solution `3` held privately |
| `x/2 + 1 = 3` | accepted; solution `4` held privately |
| `0.5x = 1.25` | accepted; solution `5/2` held privately |
| `2x = 11/2` | accepted; solution `11/4` held privately |
| `x = 5 1/2` | accepted; solution `11/2` held privately |
| `2 x = 4` | accepted; whitespace does not disable `2x` |
| `2/x = 12` | accepted direct reciprocal source; solution `1/6` and domain `x ≠ 0` held privately |
| `12 = 2/x` | accepted direct reciprocal source; solution `1/6` and domain `x ≠ 0` held privately |
| `2 3 = 6` | rejected; numeric values need an operator |
| `x² = 4`, `x*x = 4`, `x/x = 1`, `2/(x+1)=3` | rejected; nonlinear, unsupported power, or unsupported variable denominator |

## Mathematical model

### Exact rational values
A `Rational` is an immutable-shaped value:

```js
/** A reduced exact fraction; denominator is always positive and non-zero. */
{ numerator: bigint, denominator: bigint }
```

`normalizeRational(numerator, denominator)` is the sole rational constructor. It throws only for zero denominator internally; public analysis functions convert that failure into a result error. It moves a negative sign to the numerator, reduces by non-negative GCD, and represents zero exactly as `{ numerator: 0n, denominator: 1n }`.

Required pure helpers are `addRationals`, `subtractRationals`, `multiplyRationals`, `divideRationals`, `negateRational`, `isZeroRational`, and `sameRational`. No helper may convert a value to `Number`, stringify it to calculate, or mutate an argument.

- An integer `digits` becomes `BigInt(digits) / 1`.
- A decimal with `n` fractional digits becomes `BigInt(joinedDigits) / 10n ** BigInt(n)` and is normalized.
- A mixed number is `whole + numerator / denominator`; its lexical constraints above apply before arithmetic.

### Direct-reciprocal source recognition

Before ordinary affine reduction, source analysis examines the parsed equation for exactly one documented reciprocal shape. It accepts only:

```text
constantExpression / x = constantExpression
constantExpression = constantExpression / x
```

The numerator `c` and opposite side `d` are each reduced with the ordinary affine reducer and must be constants (`x` coefficient zero) with nonzero values. The reciprocal denominator must be the bare variable AST node `x`; grouping, arithmetic, signs, or any other expression around it are not accepted as a direct reciprocal denominator. The result is exact:

```text
solution = c / d
domainRestriction = x ≠ 0
```

This recognition is source-only and structural. It must run after parsing but before normal affine source reduction. It does not permit `2 / (x + 1) = 3`, `1 / x + 1 / (x - 1) = 2`, `x / (x + 1) = 2`, `x / x = 1`, or a reciprocal expression on both sides. These return `unsupported-rational`, rather than being broadened into general rational-equation solving. A zero `c` or `d` also does not satisfy the supported direct-reciprocal contract and returns `unsupported-rational`.

### Affine reduction

An `Affine` value is `{ x: Rational, constant: Rational }`, denoting `x * xCoefficient + constant`. The reducer returns an affine value or a typed error.

| Operator | Rule |
| --- | --- |
| `+`, `-`, unary `-` | Apply the matching rational operation component-wise to `x` and `constant`. |
| `left * right` | Let `left = (a, b)` and `right = (c, d)`, where each pair is `(x, constant)`. If `a` and `c` are both non-zero, return `nonlinear`. If `a` is zero, return `(b*c, b*d)`; otherwise return `(a*d, b*d)`. This also correctly handles two constant operands. |
| `left / right` | Let `right = (c, d)`. If `c` is non-zero, return `unsupported-rational`; if `d` is zero, return `divide-by-zero`. Otherwise, for `left = (a, b)`, return `(a/d, b/d)`. |

All zero-coefficient tests occur after rational normalization. Consequently, an expression that algebraically cancels to a constant, such as `(x - x) * 3`, is a valid affine expression; it is accepted or rejected only by the final unique-solution test. This is intentional exact algebraic reduction, not a syntactic ban on mentioning `x` more than once.

The parser must represent precedence before reduction; it must not use regex substitution or evaluate source text directly. An internal AST is recommended and must at minimum retain source spans for diagnostics:

```js
// Illustrative shape; implementation may use equivalent immutable objects.
{ type: 'number', value: Rational, start, end }
{ type: 'variable', start, end }
{ type: 'unary', operator, operand, start, end }
{ type: 'binary', operator, left, right, start, end }
```

For an ordinary affine parsed `left = right`, calculate exactly:

```text
coefficient = left.x - right.x
constant    = right.constant - left.constant
solution    = constant / coefficient
```

If `coefficient` is zero, reject the source: when `constant` is zero its category is `not-unique` (identity/infinite solutions); otherwise it is `no-solution` (contradiction). Both are displayed to learners as “This equation does not have one answer for `x` yet.” The internal category remains distinct for developer self-checks.

## Public M1 interface and M0 handoff

All math functions are pure: they accept strings/tokens/AST values and return data. They never query or modify the DOM, state, focus, console, or live region.

The single public setup-analysis boundary is:

```js
// Result is either { ok: true, value: SourceEquation } or { ok: false, error: MathError }.
function analyzeSourceEquation(input) { /* ... */ }
```

`SourceEquation` contains at least:

```js
{
  rawInput: string,              // trimmed user input, retained for diagnostics only
  displayEquation: string,       // normalized safe Step 1 display
  kind: 'linear' | 'direct-reciprocal',
  left: Affine | null,           // affine data for linear sources
  right: Affine | null,          // affine data for linear sources
  solution: Rational,            // never displayed by M1 UI
  domainRestriction: null | { variable: 'x', excluded: Rational },
}
```

A direct-reciprocal source has `kind: 'direct-reciprocal'`, `left` and `right` set to `null`, and `domainRestriction` `{ variable: 'x', excluded: 0 }`. A linear source has `kind: 'linear'` and `domainRestriction: null`.

`MathError` contains `{ code, message, start?, end? }`, where `code` is one of `empty-side`, `equality-count`, `unexpected-token`, `missing-operand`, `unmatched-parenthesis`, `malformed-number`, `unsupported-symbol`, `unsupported-variable`, `unsupported-power`, `divide-by-zero`, `nonlinear`, `unsupported-rational`, `not-unique`, `no-solution`, or `input-too-long`.

M1 extends—not replaces—the M0 state contract:

```js
const state = {
  mode: 'landing',
  draftEquation: '',
  submittedEquation: '',         // normalized displayEquation on success
  sourceEquation: null,          // SourceEquation only after successful analysis
  feedback: { kind: 'info', message: '' },
};
```

On successful analysis, set `submittedEquation` to `displayEquation`, set `sourceEquation`, transition to `started`, and render the existing neutral M0 started shell. On failure, retain `draftEquation`, keep `mode: 'landing'`, set `sourceEquation` to `null`, associate an error with the input, and focus the input after rendering. Reset clears all fields, including `sourceEquation`.

`displayEquation` is generated from accepted parsed tokens/AST—not from untrusted HTML—and uses `−`, `×`, and `÷` for binary operators with readable spacing around `+`, `−`, and `=`. The formatter must preserve AST meaning: it emits parentheses whenever omitting them would change precedence, grouping, unary-sign binding, or left-associative division (for example, `2 × (x + 1)`, `x ÷ (2 ÷ 3)`, and `-(x + 1)`). A recognized mixed number remains visibly `5 1/2` so its meaning is not made ambiguous. Formatting is display-only: M1 stores analyzed source data rather than reparsing display text. It is inserted with `textContent` only.

## Limits and diagnostics

- Reject an input longer than **512 Unicode code units** or producing more than **160 lexical tokens** with `input-too-long`; do not attempt unbounded parsing or BigInt construction.
- Token positions are zero-based internal offsets. UI wording need not expose positions, but an error should point conceptually to the offending construct by keeping the input focused and marked invalid.
- A successful setup status says the first equation is ready and does **not** say it was solved, give the numeric answer, or use a success check that implies the learner completed work.
- Error messages are friendly, actionable, and stable by error code. Required message intent:

| Code/category | Learner-facing wording intent |
| --- | --- |
| equality count / empty side | “Use one equals sign, with an expression on both sides.” |
| missing operand / unexpected token | “This side needs a number, `x`, or parentheses after that operation.” |
| unmatched parenthesis | “Let’s close each set of parentheses.” |
| malformed number | “That number needs a little fixing. Try digits like `0.5` or `11/2`.” |
| unsupported character/variable/power | “This game solves equations with one lowercase `x` and no powers.” |
| divide by zero | “We can’t divide by zero.” |
| nonlinear | “Try multiplying `x` by a number, not by another `x`.” |
| unsupported rational structure | “This game can use `x` in a denominator only in a simple equation like `2 ÷ x = 12`.” |
| identity/contradiction | “This equation does not have one answer for `x` yet.” |
| input too long | “Let’s try a shorter equation.” |

Do not claim that both sides need “the same operation” during source setup; no learner transformation has occurred yet.

## Code style

Keep the inline script organized in this order: constants/types; exact rational helpers; tokenizer; parser; direct-reciprocal recognizer; AST-to-affine reducer; source analysis and display formatting; self-check cases; state/rendering; DOM event handlers. Only DOM/render functions may interact with elements.

```js
function analyzeSourceEquation(input) {
  const tokenResult = tokenizeEquation(input);
  if (!tokenResult.ok) return tokenResult;

  const parseResult = parseEquationTokens(tokenResult.value);
  if (!parseResult.ok) return parseResult;

  const reciprocalResult = analyzeDirectReciprocalSource(parseResult.value);
  if (reciprocalResult.matched) return reciprocalResult.result;

  return reduceAndSolveEquation(parseResult.value);
}
```

The recognizer must distinguish “not this exact source shape” from an unsupported variable-denominator structure so the affine path cannot accidentally accept or misclassify rational sources.

Use two-space indentation; `const` by default; camelCase values/functions; `UPPER_SNAKE_CASE` fixed constants; and JSDoc for non-obvious values. Never use user input with `innerHTML`, derive math from rendered text, catch all errors silently, or duplicate source validation in UI handlers.

## Testing strategy

### Developer self-check

Expose a non-mutating `window.runMathSelfChecks` for this dependency-free milestone. It returns an object such as `{ passed, failed, cases }`, where each case has a stable name and outcome; it may write a compact summary with `console.table`, but must not change app state or render learner-visible content. Include a nearby comment documenting invocation: `runMathSelfChecks()` in browser DevTools.

The suite must assert exact numerator/denominator pairs—not decimal approximations—and include at least:

1. Rational normalization signs/zero/GCD and decimal `0.5 === 1/2`.
2. Parser precedence: `2 + 3 * 4`, parentheses, unary signs, and implicit `2x` / `2 (x + 1)`.
3. Valid linear sources and exact solutions: `2x - 3 = 8 → 11/2`; `3(x + 2) = 15 → 3`; `x/2 + 1 = 3 → 4`; `0.5x = 1.25 → 5/2`; `2x = 11/2 → 11/4`; `x = 5 1/2 → 11/2`.
4. Valid direct-reciprocal sources: `2/x = 12 → 1/6` and `12 = 2/x → 1/6`, each with `kind: 'direct-reciprocal'` and `x ≠ 0`; include nonzero decimal or fraction constants.
5. Accepted whitespace implicit multiplication: `2 x = 4`.
6. Rejections: empty/multiple equals, empty side, unmatched parentheses, malformed decimals, malformed mixed fractions, `2 3 = 6`, unsupported variable, unsupported symbol, `x²`, `x*x`, `x/0`, identity, contradiction, zero reciprocal constants, `x/x`, `2/(x+1)=3`, `1/x + 1/(x-1)=2`, and reciprocal expressions on both sides.
7. Limit boundaries: an input at the configured maximum succeeds or produces its normal semantic result; one beyond it returns `input-too-long` without throwing.

### Human acceptance gate

1. Open `index.html` directly and through `python3 -m http.server 8000 --directory .`; check the console has no uncaught errors or failed resources.
2. Enter `2x - 3 = 8`, `3(x + 2) = 15`, `x/2 + 1 = 3`, `0.5x = 1.25`, `2x = 11/2`, `x = 5 1/2`, `2/x = 12`, and `12 = 2/x`, resetting between cases. Each must begin the neutral M1 started session, display a normalized read-only Step 1, and never display an answer or reciprocal domain metadata.
3. Enter `x² = 4`, `x*x = 4`, `x/x = 1`, `2/(x + 1) = 3`, `1/x + 1/(x - 1) = 2`, `2/x = 0`, `x = x`, `x = x + 1`, `2x =`, `x/0 = 1`, `2y = 3`, `2x == 3`, `.5x = 1`, `x = 5 3/2`, and `2 3 = 6`. Each must remain editable, start no session, show a category-appropriate message, and keep focus on the invalid input.
4. In DevTools, run `runMathSelfChecks()` and verify all named cases pass. Confirm it does not transition the visible UI or reveal the stored solution.
5. Re-run one valid linear case, one valid direct-reciprocal case, and one invalid rational case at 320px and 1280px, by mouse and keyboard Enter, to confirm M0's responsive/focus/live-feedback behavior persists.
6. Reset after a successful setup; confirm raw/display/source solution and reciprocal-domain state are cleared and the M0 neutral ready status and input focus return.

## Boundaries

- **Always:** preserve M0 accessibility/responsiveness; use exact `BigInt` rational arithmetic; keep parser/reducer pure; enforce documented grammar and limits; retain reciprocal domain metadata privately; provide stable actionable errors; safely render only `displayEquation` with `textContent`; run self-checks and the human gate before declaring M1 complete.
- **Ask first:** add dependencies, a package/test/build system, a new notation or numeric form, a new variable, broader rational-equation support, answer display before M3, persistence/network behavior, a multi-file architecture, commit, or publish changes.
- **Never:** use `eval`, `Function`, `Number`/floating-point math for equation decisions, regex-only algebra evaluation, unbounded input processing, general rational-equation solving, learner-visible answer/domain disclosure in M1, dynamic HTML insertion of input, or duplicate M3 transformation validation.

## Acceptance criteria

- [ ] Only documented unique-solution linear equations and the two documented direct-reciprocal orientations transition from landing to started; all accepted source data is exact and retained outside the DOM.
- [ ] Direct-reciprocal sources retain `kind: 'direct-reciprocal'`, exact `c / d` solution, and private `x ≠ 0` domain metadata; unsupported variable-denominator structures and zero reciprocal constants are rejected.
- [ ] The valid examples in the self-check and human gate succeed with their stated exact solutions, while the UI never exposes those solutions or domain metadata.
- [ ] Malformed, unsupported, nonlinear, unsupported-rational, identity, and contradiction examples are rejected without exceptions, session start, or stale prior `sourceEquation` state.
- [ ] Decimal, fraction, and mixed-number semantics are deterministic and use no binary floating point.
- [ ] Input/rendering limits protect the parser without changing valid normal examples.
- [ ] Step 1 shows a safe normalized display form; mixed fractions remain unambiguous; all user-controlled text reaches the DOM through `textContent`.
- [ ] `runMathSelfChecks()` is documented, side-effect free to learner UI, and reports every required case passing.
- [ ] M0 reset, responsive layout, native form/Enter behavior, error association, focus, live feedback, disabled preview workspace, local-file operation, and no-network policy remain intact.

## Exit gate and dependencies

**Entry:** M0's human acceptance gate is complete and this revised M1 spec is approved.

**Done:** Every acceptance criterion, `runMathSelfChecks()` case, and human-gate step passes in a current Chromium or Safari browser (Firefox recommended). The reviewer accepts the grammar, direct-reciprocal boundary, diagnostics, exact data contract, private domain restriction, and no-answer-disclosure behavior. M2 may then replace the disabled workspace controls with token editing; M3 must reuse M1 analysis/reduction functions rather than reimplementing equation mathematics.

## Open questions

None. This milestone intentionally chooses strict, predictable literal rules (including `.5` rejection, proper mixed-fraction components, and one structural reciprocal exception) over permissive guessing. Any change to these rules requires a spec update and approval before implementation.
