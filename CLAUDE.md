# Solve Equation — Project Guidance

## Purpose
Build a standalone, kid-friendly browser game that teaches solving single-variable linear equations with a touch-friendly token workspace.

## Architecture
- Keep the application self-contained in `index.html`: semantic HTML, embedded CSS, and vanilla JavaScript.
- Do not add a backend, build system, package manager, CDN, analytics, or third-party dependency without approval.
- Use Pointer Events for drag/reorder/delete interactions so the app works on mouse and touch input.
- Parse and evaluate mathematical input with project code; never use `eval`, `Function`, or a third-party expression evaluator.
- Represent linear expressions and values with exact rational arithmetic, not binary floating point.

## Scope
- Support one-variable, unique-solution linear equations, plus the approved source-only direct-reciprocal exception `c / x = d` or `d = c / x` when `c` and `d` are nonzero constant expressions. Keep the exception narrow: do not accept general rational equations or other variable denominators.
- Accepted notation: integers, terminating decimals, fractions using `÷` or `/`, parentheses, `+`, `-`, `×`/`*`, implicit multiplication such as `2x`, and `x`.
- Treat unsupported, nonlinear, unsupported-rational, identity, and contradiction equations as invalid and explain the limitation clearly.

## Quality Requirements
- Preserve each accepted equation as locked history; only the active row is editable.
- Never permit an active step that changes the initial equation’s exact solution.
- Provide keyboard-accessible token insertion and deletion in addition to pointer interactions.
- Maintain high contrast, visible focus states, responsive layout, and reduced-motion behavior.
- Run the manual browser acceptance checklist in `docs/specs/linear-equation-drag-solver.md` after changes.

## Commands
- Run locally: `python3 -m http.server 8000 --directory .`
- Open: `http://localhost:8000/`
- Optional static syntax check: `node --check index.html` is not applicable to inline scripts; extract only temporarily for diagnosis and do not commit generated files.
