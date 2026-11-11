# Equation Explorer

A lightweight, kid-friendly browser app for practising single-variable linear
equations step by step. Learners enter an equation, then use touch-friendly math
tokens to build equivalent steps until `y` is isolated.

The project is also preparing a native iOS and Android app built with React
Native, TypeScript, and Expo. The native app is specified but not yet created.

## Current status

- **Web app:** implemented through M4 (exact math engine, token workspace,
  guided validation and history, celebration and accessibility hardening) and
  accepted as the parity baseline for the native migration. The historical M3
  human acceptance gate and M4 human release gate remain recorded as open in the
  product specification; RN-M0 captures the runtime evidence that was
  outstanding.
- **Native app:** planned. `mobile/` is created in RN-M1; no React Native code
  exists in RN-M0.

The accepted web baseline is tagged `web-baseline-2026-11-11`, with a backup
branch `backup/web-baseline-2026-11-11` and a recorded SHA-256 checksum. See the
[baseline evidence](docs/specs/react-native/evidence/RN-M0-baseline.md).

## Repository layout

```text
web/       Legacy web app (parity reference; single self-contained file)
mobile/    Future React Native app (created in RN-M1; absent in RN-M0)
docs/      Product specs, RN specs, evidence, and ADRs
tasks/     Web and RN implementation plans and task lists
```

The layout decision and the web/native lifecycle are recorded in
[ADR-011](docs/adr/ADR-011-repository-layout.md).

## Run the web app locally

No installation or build step is required. Serve the web directory:

```sh
python3 -m http.server 8000 --directory web
```

Then browse to [http://localhost:8000/](http://localhost:8000/). The app also
opens directly from `web/index.html` in a modern browser.

## Deployment

The web app is static client-side HTML, CSS, and JavaScript. It can be hosted on
Amazon S3 static hosting or behind CloudFront, with no backend, database,
account system, or API required.

The repository contains no hosting configuration. Any external static host must
serve the `web/` directory (or redirect the site root to it); changing the host
is a separate, owner-approved action.

## Roadmap

Web milestones (implemented):

- **M0 — Foundations and UI shell:** implemented.
- **M1 — Exact linear math engine:** implemented.
- **M2 — Touch token workspace:** implemented.
- **M3 — Guided validation and history:** implemented; human gate open.
- **M4 — Celebration and hardening:** implemented; human release gate open.

Native milestones (planned): see the
[React Native roadmap](docs/specs/react-native/milestones/README.md).
RN-M0 (baseline and repository layout) is complete pending its human exit gate;
RN-M1 has not started.

## Documentation

- [Product specification (web)](docs/specs/linear-equation-drag-solver.md)
- [Web app notes](web/README.md)
- [React Native roadmap](docs/specs/react-native/milestones/README.md)
- [RN-M0 baseline evidence](docs/specs/react-native/evidence/RN-M0-baseline.md)
- [ADR-011: repository layout](docs/adr/ADR-011-repository-layout.md)
- [Project engineering guidance](CLAUDE.md)
- Historical web milestones: [M0](docs/specs/milestones/00-foundations-ui-shell.md), [M1](docs/specs/milestones/01-exact-linear-math-engine.md), [M2](docs/specs/milestones/02-touch-token-workspace.md), [M3](docs/specs/milestones/03-guided-validation-history.md), [M4](docs/specs/milestones/04-celebration-accessibility-hardening.md)
- Historical web tasks: [M0 plan](tasks/00-plan.md), [M0 task list](tasks/00-todo.md)

## Web project constraints

`web/index.html` remains a standalone, dependency-free file. It uses vanilla
JavaScript, works offline, and must never evaluate equations with `eval()` or
`Function()`. Mathematical validation uses exact `BigInt` rational arithmetic.
There is no build step, package manager, CDN, or third-party dependency.

These constraints apply to the web app. The native app uses its own approved
toolchain under `mobile/`; see `CLAUDE.md`.