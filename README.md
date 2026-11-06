# Equation Explorer

A lightweight, kid-friendly browser app for practising single-variable linear equations step by step. Learners enter an equation, then—over later milestones—will use touch-friendly math tokens to build equivalent steps until `x` is isolated.

## Current status
Milestone 0 (responsive UI shell) is implemented. Its final visual/accessibility acceptance review is pending. Milestone 1 (exact math engine) is specified but not implemented.

The current app is a **UI-only preview**: it accepts non-blank text to demonstrate the layout but does not yet validate mathematics or solve equations.

## Run locally
No installation or build step is required. Open `index.html` directly in a modern browser, or serve it locally:

```sh
python3 -m http.server 8000 --directory .
```

Then browse to [http://localhost:8000/](http://localhost:8000/).

## Deployment
The finished application is static client-side HTML, CSS, and JavaScript. It can be hosted on Amazon S3 static hosting or behind CloudFront, with no backend, database, account system, or API required.

## Roadmap
- **M0 — Foundations and UI shell:** implemented; human browser acceptance pending.
- **M1 — Exact linear math engine:** safe parser, exact rational arithmetic, and source-equation validation.
- **M2 — Touch token workspace:** pointer, tap, keyboard, reorder, move, and delete controls.
- **M3 — Guided validation and history:** equivalent-step checks, feedback, and locked equation history.
- **M4 — Celebration and hardening:** final-answer detection, confetti/audio, accessibility, and release acceptance.

## Documentation
- [Product specification](docs/specs/linear-equation-drag-solver.md)
- [M0: Foundations and UI shell](docs/specs/milestones/00-foundations-ui-shell.md)
- [M1: Exact linear math engine](docs/specs/milestones/01-exact-linear-math-engine.md)
- [M0 implementation plan](tasks/00-plan.md)
- [M0 task list and status](tasks/00-todo.md)
- [Project engineering guidance](CLAUDE.md)

## Project constraints
The app remains a standalone, dependency-free `index.html`. It uses vanilla JavaScript, works offline, and must never evaluate equations with `eval()` or `Function()`. Mathematical validation will use exact `BigInt` rational arithmetic.
