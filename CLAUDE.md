# Solve Equation — Project Guidance

## Purpose

Teach solving single-variable linear equations with a touch-friendly token
workspace. The project has two codebases:

- `web/` — the shipped, dependency-free browser app.
- `mobile/` — a planned React Native app (created in RN-M1; absent today).

## Repository layout

- `web/index.html` — the web app; parity reference for the native migration.
- `mobile/` — the future native app. Do not create it before RN-M1.
- `docs/specs/linear-equation-drag-solver.md` — authoritative web product spec.
- `docs/specs/react-native/` — native milestone specs, roadmap, and evidence.
- `tasks/react-native/` — native plans and task lists.
- `docs/adr/` — architecture decision records.
- `docs/specs/milestones/` and `tasks/00`–`04` — historical web records; do not
  rewrite them.

## Web app rules (`web/`)

- Keep the application self-contained in `web/index.html`: semantic HTML,
  embedded CSS, and vanilla JavaScript.
- Do not add a backend, build system, package manager, CDN, analytics, or
  third-party dependency to the web app without approval.
- Use Pointer Events for drag/reorder/delete so the app works with mouse and
  touch input.
- Parse and evaluate mathematical input with project code; never use `eval`,
  `Function`, or a third-party expression evaluator.
- Represent linear expressions and values with exact rational arithmetic, not
  binary floating point.
- Preserve each accepted equation as locked history; only the active row is
  editable, and an active step must never change the initial equation's exact
  solution.
- Provide keyboard-accessible token insertion and deletion in addition to
  pointer interactions.
- Maintain high contrast, visible focus states, responsive layout, and
  reduced-motion behavior.
- Run the manual browser acceptance checklist in
  `docs/specs/linear-equation-drag-solver.md` after web changes.
- Do not change web behavior, content, markup, CSS, JavaScript, math, or
  accessibility without an approved milestone; RN-M0 moved it byte-identically.

## Native app rules (`mobile/`)

- Create `mobile/` only in RN-M1. Use Expo, TypeScript, and Expo Router per the
  React Native specs in `docs/specs/react-native/`.
- The native app is a rebuild, not a port: no WebView, DOM components, React
  Native Web, or embedded web UI.
- No backend, authentication, persistence, analytics, crash reporting, or
  network requests. All content is bundled and the app works offline.
- Use the same exact-rational, no-`eval`/`Function` math rules as the web app.
- Build locally with Xcode and Gradle; do not use EAS build/update services.
- Keep the native toolchain scoped to `mobile/`. **The web app's no-build,
  no-dependency rule does not apply to `mobile/`.**

## Commands

```sh
# Web app (from the repository root)
python3 -m http.server 8000 --directory web
# Open http://localhost:8000/
```

The web app has no install, build, lint, or automated-test command.

Native install, run, typecheck, lint, test, and build commands are added in
RN-M1 and documented there. Do not invent or run them before `mobile/` exists.