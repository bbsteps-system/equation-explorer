# Spec: Drag-and-Drop Linear Equation Solver

**Status:** Implemented through M4 with automated verification complete. M3's human acceptance gate and the M4 human release gate remain open; no milestone is complete until its human gate passes.

## Objective
Create a lightweight, responsive, playful browser application that helps children practise solving a one-variable linear equation by constructing each equivalent equation from draggable mathematical tokens. A parent, teacher, or child enters an equation such as `2y - 3 = 8`; the app records it as Step 1, then lets the child build and check later steps until the variable is isolated.

Success means a child can use the app without an account or installation, see a clear history of their work, receive encouraging and actionable feedback, and finish with a mathematically verified `y = value` result.

### User stories
- As a learner, I can enter a supported equation and begin with an uneditable record of it.
- As a learner, I can compose each side of my next equation by dragging, tapping, reordering, or deleting tokens.
- As a learner, I receive a simple explanation when an expression is incomplete or changes the answer.
- As a learner, I can celebrate only after the application verifies that I have isolated `y` correctly.
- As an adult supporting the learner, I can use the app entirely offline after loading its single file and understand why unsupported equations are declined.

## Assumptions and scope
- The deliverable is a standalone `index.html`, with inline CSS and JavaScript, opened directly or served as static content.
- Supported inputs describe either a linear equation in exactly one variable, `y`, with one unique real solution, or the limited direct-reciprocal source form defined below.
- Accepted notation: integers; terminating decimals; `+`, `-`, `×`/`*`, `÷`/`/`; parentheses; `y`; unary signs; and implicit multiplication (`2y`, `2(y + 1)`). Whitespace is ignored except where used to recognize a mixed-fraction display value.
- In addition to linear sources, setup accepts a direct reciprocal equation `c / y = d` or `d = c / y`, where `c` and `d` are nonzero constant expressions. It solves exactly as `y = c / d` with the retained domain restriction `y ≠ 0`. This narrow exception does not permit other variable denominators such as `2 / (y + 1)`, `1 / y + 1 / (y - 1)`, or reciprocal expressions on both sides.
- The starter equation may contain fractions and parentheses if it remains linear, or if they occur within the documented direct-reciprocal constants. Equations that are nonlinear (`y²`, `y × y`), contain unsupported characters or variables, use an unsupported variable denominator, or reduce to an identity/contradiction are rejected before a session begins.
- A checked equation must have the exact same unique solution as the initial equation. The app does not require one prescribed school-method transformation, but detects simple asymmetric transformations only when that diagnosis is certain.
- No backend, account, persistence, tracking, external asset, third-party library, or build tooling is included.

## Tech Stack
- HTML5 semantic markup in `index.html`
- CSS3 custom properties, flexbox/grid, responsive media queries, animations, and `prefers-reduced-motion`
- Vanilla ES2020+ JavaScript using Pointer Events for touch/mouse token manipulation
- Web Audio API for an optional synthesized celebration sound
- Canvas or DOM/CSS particles for a dependency-free confetti effect
- Custom tokenizer, recursive-descent parser, affine-expression reducer, and exact rational-number arithmetic

## Commands
```sh
# Serve the standalone app locally (recommended for consistent browser behavior)
python3 -m http.server 8000 --directory .

# Open the application
open http://localhost:8000/

# Check repository state when the project is initialized as Git
git status --short
```

There is no package installation, build, lint, or automated-test command in the dependency-free release. Every milestone supplies an executable browser demo checklist; the final manual acceptance pass is defined below.

## Project Structure
```text
index.html                                      # Complete shipped application: markup, styles, behavior
CLAUDE.md                                       # Persistent project conventions for future contributors
docs/
  specs/
    linear-equation-drag-solver.md              # This umbrella product specification
    milestones/
      00-foundations-ui-shell.md                # Responsive visual shell and setup flow
      01-exact-linear-math-engine.md            # Parser, affine reduction, exact solution validation
      02-touch-token-workspace.md               # Pointer/tap/keyboard token editing
      03-guided-validation-history.md           # Checked steps, feedback, immutable history
      04-celebration-accessibility-hardening.md # Win experience and release validation
```

Future modularization requires approval and should move code into `src/` and introduce a browser test runner before adding any production dependency.

## Milestone roadmap
The product is delivered in five sequential, independently reviewable milestones. Each milestone has a detailed scoped specification, a human demo script, and an explicit exit gate. A later milestone may refine implementation details but may not weaken a prior milestone's acceptance criteria without updating this umbrella spec and receiving approval.

| Milestone | Outcome | Human acceptance gate |
| --- | --- | --- |
| [M0: Foundations and UI shell](milestones/00-foundations-ui-shell.md) | Responsive, accessible game shell and setup experience | A reviewer can start/reset a visually complete mock session at 320px, tablet, and desktop widths. |
| [M1: Exact linear math engine](milestones/01-exact-linear-math-engine.md) | Safe parser and exact affine/rational model wired to setup validation | A reviewer can enter the supplied valid and invalid equations and observe each expected acceptance/rejection result. |
| [M2: Touch token workspace](milestones/02-touch-token-workspace.md) | Pointer, tap, keyboard, reorder, move, and trash interactions | A reviewer can construct and edit expressions with mouse, touch emulation, and keyboard alone. |
| [M3: Guided validation and history](milestones/03-guided-validation-history.md) | Equivalence checks, feedback, locked rows, and next-step creation | A reviewer can complete valid steps and see malformed/inequivalent work rejected without data loss. |
| [M4: Celebration and accessibility hardening](milestones/04-celebration-accessibility-hardening.md) | Verified win state, confetti/audio, reduced-motion behavior, release pass | A reviewer can finish integer, decimal, and fraction examples and pass the final accessibility/responsiveness checklist. |

**Dependency order:** M0 establishes the interface and state seams; M1 provides the trusted validation core; M2 populates editable expressions; M3 connects expressions to mathematical feedback and history; M4 completes the final outcome and release-quality checks. No milestone is complete until its human gate passes.

## Draft UI and flow

### Desktop draft
```text
┌───────────────────────────────────────────────────────────────────┐
│  🧮 Equation Explorer                         ⭐ Solve it step by step │
│  Enter an equation: [ 2y - 3 = 8                         ] [Start] │
│  Tip: Whatever you do to one side, do to the other side!           │
├───────────────────────────────────────────────────────────────────┤
│  TOKEN TOOLBOX                                      🗑 Drop to erase │
│  [0][1][2][3][4][5][6][7][8][9][.]  [+][-][×][÷][(][)][y]          │
├───────────────────────────────────────────────────────────────────┤
│  YOUR SOLVING PATH                                                 │
│  Step 1  🔒              2y − 3       =       8                   │
│  Step 2                  [ drop tokens ] = [ drop tokens ] [↺][Check] │
│                            ↑ selected/focused side                │
├───────────────────────────────────────────────────────────────────┤
│  💡 Build an equation that still has the same answer for y.        │
│  Progress: ● ○ ○     Accessible help / reset                       │
└───────────────────────────────────────────────────────────────────┘
```

The visual language uses a warm off-white page background, bright indigo/teal/coral accents, large rounded cards and buttons, dark text on light surfaces, icon-and-text controls, and minimum 44×44px touch targets. No color is the sole conveyer of state; icons, text, and ARIA announcements accompany success and error states.

### Mobile draft
On narrow screens, the header remains compact; equation input and Start button stack; the toolbox wraps into large tap-friendly rows and pins beneath the top of the viewport while solving; each active equation row stacks LHS, `=`, RHS, then the Check button. History stays chronological and horizontally safe—tokens wrap inside their side rather than forcing page overflow.

### Interaction flow
1. **Landing/setup:** Show a friendly title, example placeholder, a short equivalence tip, and an input with Start Solving disabled until non-empty.
2. **Start:** Parse and validate the entered equation. On success, normalize display symbols, add locked Step 1, expose the toolbox/workspace, create an active Step 2 prefilled with token copies of Step 1, and focus its LHS. On failure, retain input and give an inline, specific correction message.
3. **Build:** A learner drags a palette token into either active side, drags existing tokens to reorder or move across sides, or drags a token into the labeled trash zone. Tapping/clicking a palette token inserts it at the end of the selected side; tapping a token selects it for a clearly labeled Delete selected token action. This makes all core behavior usable without a drag gesture. If edits leave the active row messy, a Restore previous step button beside Check replaces both active sides with fresh token copies of the latest locked row (Step 1 for the first editable row); it never affects history.
4. **Check:** The app rejects an active row that is exactly the previous step—token-for-token, or differing only by notation that normalizes to the same displayed equation—keeping it editable with same-step feedback. Otherwise it validates each side’s syntax, then reduces both sides to exact affine forms and compares the resulting unique solution to Step 1. Valid rows lock with a green check animation and create a duplicated, editable next row. Invalid rows remain editable, shake gently, describe the relevant issue, and set focus to the affected side or feedback text.
5. **Win:** When the valid active equation is exactly `y = rational value` (coefficient `1` on the left and no `y` on the right), lock it, announce the answer, show confetti and praise, and play a safely-caught short Web Audio chime. The celebration panel offers “Solve another!” to reset safely.

## Functional Requirements

### Equation setup
- Input has a clear label, an example (`2y - 3 = 8`), Enter-key support, and a reset control once a session is underway.
- Initial input must contain exactly one top-level equality sign and pass supported-linear-equation validation.
- Step 1 displays the normalized input read-only. The source equation and its exact solution are retained in application state, never reconstructed from DOM text.

### Token workspace
- Palette provides `0`–`9`, decimal point (`.`), `+`, `−`, `×`, `÷`, `(`, `)`, and `y`.
- A token is a semantic button/list item with its token type and a readable aria-label (for example, “multiply” rather than only `×`).
- The variable `y` is red in the palette, active tokens, and rendered equations; multiplication `×` remains dark so the similar glyphs are easier to distinguish. Shape and accessible labels remain the primary distinction.
- Active LHS and RHS are independently labeled droppable token lists separated by a static equality sign. They begin as tokenized copies of the latest accepted row.
- A visible trash zone accepts dragging; keyboard/tap deletion is an equivalent path. The Delete selected token button sits beside the trash zone and before Restore previous step. Deleted items disappear only from the active row.
- While a session is active, the toolbox pins beneath the top of the viewport during vertical scrolling so the palette stays reachable; it returns to normal flow at the top and becomes static once the session is complete. On screens narrower than 600px it compacts its padding and helper text. The app reserves the card's measured height in scroll offsets so focused tokens, side selectors, and feedback are not obscured.
- Prior rows are immutable and cannot be edited, reordered, or deleted.
- A Restore previous step button beside Check replaces both active side lists with fresh token copies of the most recent locked step (Step 1 for the first editable row). It is disabled when the active row already matches that step, clears any selected token, and leaves locked history untouched.
- Each active side has a small Clear (×) button in its top-right corner that empties only that side and is disabled while the side is already empty. Clearing clears any selected token and never alters locked history.
- A Flip sides button beside Restore previous step swaps the complete token contents of the two active sides. It is disabled when both sides are empty, clears any selected token, and never alters locked history.

### Parsing and validation
- Token strings are joined and parsed with precedence: parentheses; unary `+`/`-`; multiplication/division including implicit multiplication; addition/subtraction.
- The parser must reject empty sides, unmatched parentheses, adjacent invalid operands/operators, unknown symbols, division by zero, malformed decimals/fractions, nonlinear products, and divisions involving `y` except for the documented direct-reciprocal source form.
- Reduce linear expressions to `a*y + b`, where `a` and `b` are normalized exact rationals `{ numerator: bigint, denominator: bigint }`, denominator positive and fraction reduced by GCD. Analyze the direct reciprocal source form separately and retain its nonzero-domain restriction.
- A linear initial equation solves as `y = (right.b - left.b) / (left.a - right.a)`. A direct reciprocal source solves as `y = c / d`; reject zero constants or unsupported reciprocal structures because they do not satisfy the documented unique-solution form.
- Each checked row must be a valid, unique-solution linear equation whose exact reduced solution equals the initial solution. This permits a learner who starts from a direct reciprocal source to build an equivalent linear form such as `y = 1 ÷ 6`. Rational cross multiplication—not floating point—is the equality comparison.
- An active row that is exactly the previous step (Step 1 for the first editable row, otherwise the latest locked row) must not lock. Keep it editable, add no history, create no successor, and show same-step feedback. Compare the exact token sequence and, after parsing, the candidate’s canonical display; the latter catches notation-only variants such as `2y` versus `2 × y`. A genuinely different equivalent step still advances.
- For the isolated win form, left side must reduce structurally to exactly `y` (not merely `1y` or `(y)` after arbitrary formatting unless the UI explicitly canonicalizes it), right side must be a constant rational, and that exact constant must equal the source solution.
- Accepted result displays use canonical integer, improper fraction (`11/2`), or terminating decimal (`5.5`) notation. A mixed fraction such as `5 1/2` is accepted in setup input; learners can express its equivalent through `11 ÷ 2` from the palette.

### Feedback and celebration
- Syntax messages include child-friendly, actionable wording, such as “This side needs a number or `y` after that operation.”, “Let’s close each set of parentheses.”, and “We can’t divide by zero.”
- An inequivalence message says: “That changes the answer for `y`. Try doing the same operation to both sides!” Do not claim an operation was asymmetric unless positively established.
- Success feedback includes a check icon, text such as “Great equivalent equation!”, a brief confirmation toast that auto-dismisses, a short best-effort success chime, and a short animation that honors reduced motion. The final step uses the celebration chime in place of the step chime.
- Success toasts are visual reinforcement only (`aria-hidden`); the existing polite live region carries the accessible success message, avoiding duplicate announcements.
- Celebration uses no fetched assets, cannot block completion if AudioContext is unavailable, and has visible textual success independent of sound/confetti.

## Non-Functional Requirements
- **Responsive:** Support 320px-wide screens through desktop; prevent horizontal page scrolling.
- **Accessible:** Use labels, landmarks, semantic buttons, keyboard focus, logical tab order, a polite live region for feedback, and visible focus indicators. Pointer gestures must have an accessible non-drag alternative.
- **Performance:** Load as one small local file, perform parsing/checking synchronously in under 50ms for supported token limits, and avoid layout-intensive animation loops.
- **Safety/reliability:** Never use `eval()`/`Function()`. Treat all input as data. Catch parse/audio errors and show recoverable feedback.
- **Usability:** Use high contrast, readable system fonts, plain child-friendly language, at least 44px touch controls, and no punitive language.

## Code Style
Place the application in one strict-mode inline script and organize it in clearly named sections: immutable constants, state, DOM queries/rendering, token interactions, parser/rational math, validation, and effects. Prefer small pure functions and explicit state updates over DOM-derived business logic. Use two-space indentation, `const` by default, camelCase for values/functions, PascalCase only for classes, `UPPER_SNAKE_CASE` for fixed constants, and JSDoc for non-obvious math data structures.

```js
/** A normalized exact fraction with a positive denominator. */
function addRationals(left, right) {
  return normalizeRational(
    left.numerator * right.denominator + right.numerator * left.denominator,
    left.denominator * right.denominator,
  );
}

function sameSolution(candidate, sourceSolution) {
  return candidate.numerator === sourceSolution.numerator
    && candidate.denominator === sourceSolution.denominator;
}

function showFeedback(kind, message) {
  feedbackElement.dataset.kind = kind;
  feedbackElement.textContent = message;
  feedbackElement.focus();
}
```

Do not rely on implicit type conversion for mathematics, manipulate `innerHTML` with user input, or duplicate validation rules between UI handlers.

## Testing Strategy
The dependency-free release uses documented human acceptance passes. Parser/rational code must be designed as pure functions so an approved later test runner can cover it directly. Each milestone contains its own manual demo script; its test cases accumulate into the final release checklist.

### Final release checklist
1. Open `index.html` directly and through `python3 -m http.server 8000 --directory .`; verify no console errors.
2. Start `2y - 3 = 8`; confirm a locked Step 1 and editable Step 2 copied from it.
3. Use mouse drag, touch/pointer emulation, palette tap insertion, token reordering, side-to-side move, and deletion/trash. Verify row state after each action.
4. Check a malformed side such as `2y −`; verify the row remains editable, has error styling/announcement, and receives no new history row.
5. Submit an algebraically inequivalent valid equation such as `2y - 3 = 9`; verify rejection and answer-preservation hint.
6. Submit equivalent steps for `2y - 3 = 8`, including `2y = 11`, then `y = 11 ÷ 2`; verify each success shows the confirmation toast and, when audio is available, a short success chime; verify both lock and final celebration fires once.
7. Repeat with `2y = 11`, `y = 5.5`, and `y = 11/2`; verify exact comparison rather than floating-point mismatch. Verify supported mixed-fraction setup behavior separately.
8. Start direct reciprocal sources `2/y = 12` and `12 = 2/y`; verify each begins normally without revealing the exact answer. Try invalid source equations: `y² = 4`, `y = y`, `y = y + 1`, `2y =`, `y/0 = 1`, `2/(y + 1) = 3`, `1/y + 1/(y - 1) = 2`, and inputs with unknown symbols; verify understandable rejection without page failure.
9. At 320px and desktop width, verify no horizontal scrolling and usable controls. Scroll the solving path and confirm the toolbox stays reachable without covering the active tokens, then confirm it releases to normal flow on completion. Tab through all controls and trigger non-drag alternatives; inspect focus visibility and live feedback.
10. Enable reduced motion and verify success/error/celebration remain understandable without disruptive movement. Test with audio unavailable or blocked; verify visual success still completes.
11. Before any lock, edit the active row and use Restore previous step; verify both sides return to Step 1, the button disables itself, and focus moves to the left side. Lock a step, mess up the new active row, and verify Restore previous step returns to that locked step rather than Step 1, without altering locked history.
12. Use each side's Clear (×) button and the Flip sides button; verify a clear empties only its own side, a flip swaps both sides, both actions clear the selected token, their disabled states match the contents, focus is not lost, and locked history is unchanged.

### Future automated coverage (ask first)
If a test tool is approved, add unit tests for tokenizer/parser precedence, affine reduction, fraction normalization, unique-solution solving, same-solution comparisons, malformed input, and win-form detection. Add browser interaction tests for complete successful and invalid flows.

## Boundaries
- **Always:** Keep the app client-only and dependency-free; validate and parse all equation input safely; use exact rational calculations; preserve locked history; provide non-drag input paths; run the applicable milestone human-demo checklist before claiming that milestone complete; update this spec when a decision changes.
- **Ask first:** Add dependencies/CDNs, create a build or test setup, split the single-file architecture, alter supported mathematical notation/scope, add persistence/analytics, change CI configuration, commit, or publish code.
- **Never:** Use `eval()` or `Function()` to evaluate math; send learner input to a remote service; add authentication/backend code; commit secrets; edit generated/vendor files; remove validation or accessibility behavior merely to simplify implementation; silently accept unsupported/nonlinear equations.

## Success Criteria
1. Opening the single file in a modern browser presents an attractive, responsive, keyboard-accessible equation-solving interface without installation or network requests.
2. A valid unique-solution linear equation or documented direct-reciprocal source is accepted and rendered as immutable Step 1; invalid, nonlinear, unsupported-rational, or ambiguous source equations receive a specific explanation.
3. Children can add, move, reorder, and delete all required mathematical tokens on touch and desktop, with an equivalent keyboard/tap alternative.
4. The active row rejects malformed syntax with helpful feedback and never locks or creates a next row after an invalid check.
5. The app accepts only equations with exactly the same solution as the original, using exact rational arithmetic; changed-answer equations are rejected.
6. Every valid non-final row locks, visibly succeeds, and produces exactly one editable next row prefilled from the accepted state.
7. The exact isolated form `y = value` accepts integer, decimal, and improper fraction results, then shows textual praise, reduced-motion-safe confetti, and nonessential synthesized sound.
8. Every milestone's human acceptance gate passes, followed by all final release checks in a current Chromium, Safari, or Firefox browser.

## Open Questions
None for the initial draft. New notation, a guided-hint system, saved progress, or automated-test/dependency setup requires a spec update and approval.
