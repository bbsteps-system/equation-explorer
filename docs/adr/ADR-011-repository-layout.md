# ADR-011: Repository Layout — `web/` and `mobile/`

- **Status:** Proposed
- **Date:** 2026-11-11
- **Deciders:** Boris (product owner), solution architect
- **Milestone:** RN-M0
- **Related:** [React Native roadmap](../specs/react-native/milestones/README.md), [RN-M0 evidence](../specs/react-native/evidence/RN-M0-baseline.md), [web app notes](../../web/README.md)

## Context

The repository originally held the browser app at the root as `index.html`.
The approved migration introduces a native iOS/Android app built with React
Native. Both codebases must coexist without path ambiguity, and the accepted web
baseline must remain runnable and identifiable during and after the migration.

The migration planning documents left the layout and the web/native lifecycle
unresolved (PB-02/PB-10). This ADR records the resolution.

## Decision

1. The web app lives at `web/index.html`, with a short `web/README.md`.
2. The native app will live in `mobile/`, created in RN-M1.
3. The web app is moved byte-identically. Its content, behavior, markup, CSS,
   JavaScript, math, and accessibility are unchanged by the move.
4. The pre-move web app is preserved by an annotated Git tag
   `web-baseline-2026-11-11`, a backup branch
   `backup/web-baseline-2026-11-11`, and a recorded SHA-256 checksum in the
   [RN-M0 evidence](../specs/react-native/evidence/RN-M0-baseline.md).
5. The web app is served from the repository root with
   `python3 -m http.server 8000 --directory web`.
6. Historical web milestone specs and task files (`docs/specs/milestones/00`–`04`,
   `tasks/00`–`04`) remain unchanged as historical records, including their `x`
   examples and root-path commands. Current-state documents use `y` and the new
   paths.
7. `docs/` and `tasks/` stay at the repository root and gain
   `docs/specs/react-native/`, `docs/adr/`, and `tasks/react-native/`.

## PB-02 / PB-10 resolution

**Native is the intended successor. The web app stays live and maintained until
a separate retirement decision.**

The move does not deprecate the web app, change its support status, or schedule
its removal. A future retirement requires its own approved decision and must
account for any external deployment and existing users.

## Consequences

- Two codebases have unambiguous locations; native work cannot accidentally edit
  the web app.
- The web app's run command, links, and documentation now include `web/`.
- External hosting must serve the `web/` directory or redirect the site root.
  The repository contains no hosting configuration, so this is an owner action.
- The baseline tag and backup branch preserve the exact pre-move file and the
  planning documents. Pushing them to `origin` is an approval-gated follow-up;
  until then the backup is local-only.
- Native scaffolding can begin in RN-M1 without changing the web app.

## Alternatives considered

- **Keep the web app at the repository root and add `mobile/` beside it.**
  Rejected: two entry points at the root invite confusion, and the native app
  would share the root namespace with the web app's assets and commands.
- **Move the web app to a separate repository.** Rejected for now: it would
  split the shared product specification, history, and review workflow, and it
  adds release overhead without a present benefit.
- **Keep a root `index.html` redirect to `web/`.** Deferred: a root redirect
  creates a second web entry point. Prefer a hosting/CDN redirect; adding a
  root file would be a separate approved change.
- **Archive the web app instead of moving it.** Rejected: the web app remains
  live and maintained, so it needs a working location, not an archive.

## Hosting follow-up

The static host (S3/CloudFront or equivalent) must be pointed at `web/` or
configured to redirect the site root. No DNS, CDN, or deployment configuration
is stored in this repository. The owner must update external infrastructure
separately and verify the deployed URL after the move.

## Historical documentation policy

Historical milestone and task records are evidence of what was planned and
delivered at the time. They are not rewritten to match current layout or
variable naming. Drift (root `index.html` paths and `x` examples) is recorded in
the RN-M0 evidence file. Authoritative current-state documents — `README.md`,
`CLAUDE.md`, `web/README.md`, `docs/specs/linear-equation-drag-solver.md`, and
this ADR — use the current paths and `y`.

## Links

- [Repository README](../../README.md)
- [Project guidance](../../CLAUDE.md)
- [Web product specification](../specs/linear-equation-drag-solver.md)
- [React Native roadmap](../specs/react-native/milestones/README.md)
- [RN-M0 baseline evidence](../specs/react-native/evidence/RN-M0-baseline.md)