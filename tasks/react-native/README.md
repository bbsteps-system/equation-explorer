# React Native Task Conventions

This directory holds the implementation plans and task checklists for the React
Native milestones (`RN-M0`–`RN-M8`). The native roadmap is at
[`../../docs/specs/react-native/milestones/README.md`](../../docs/specs/react-native/milestones/README.md).

## Artifact locations

| Artifact | Path | Example |
| --- | --- | --- |
| Milestone spec | `docs/specs/react-native/milestones/RN-M<N>-<slug>.md` | `RN-M0-baseline-and-repo-layout.md` |
| Implementation plan | `tasks/react-native/RN-M<N>-plan.md` | `RN-M0-plan.md` |
| Task checklist | `tasks/react-native/RN-M<N>-todo.md` | `RN-M0-todo.md` |
| Evidence | `docs/specs/react-native/evidence/RN-M<N>-<slug>.md` | `RN-M0-baseline.md` |
| Evidence assets | `docs/specs/react-native/evidence/assets/RN-M<N>/` | `assets/RN-M0/*.png` |
| Architecture decision | `docs/adr/ADR-NNN-<slug>.md` | `ADR-011-repository-layout.md` |

Use the full `RN-M<N>` identifier in every filename. Do not reuse the historical
web milestone prefixes (`00`–`04`) for native work.

## Workflow

Each milestone is a self-contained spec-driven unit:

```text
SPECIFY ──→ PLAN ──→ TASKS ──→ IMPLEMENT ──→ HUMAN GATE
```

1. **Specify:** write the milestone spec and get human approval.
2. **Plan:** write `RN-M<N>-plan.md` and `RN-M<N>-todo.md`; get human approval.
3. **Tasks:** implement tasks in dependency order, updating the todo as work
   completes.
4. **Implement:** one focused task at a time; run the verification step named in
   the task.
5. **Human gate:** the reviewer signs the milestone exit gate. Agents must not
   mark a human-gate checkbox complete.

Do not start a milestone until its predecessor's gate is signed.

## Status conventions

- `- [ ]` means not done; `- [x]` means done and verified.
- Keep the document's `**Status:**` line accurate (Draft, In progress, Complete
  pending human gate, Accepted).
- Mark a task complete only after its stated verification was actually run.
- Human review and sign-off checkboxes are never checked by an agent; record the
  evidence and leave the gate for the reviewer.
- If a task is intentionally skipped or deferred, record why in the todo and in
  the milestone evidence.

## Evidence rules

- Record actual commands, environment versions, viewports, and observed results.
- Never claim a manual check passed if it was not performed. Record unavailable
  devices, screen readers, or audio as limitations.
- Note deviations from the product spec or milestone spec separately from
  failures; do not silently fix out-of-scope behavior.
- Keep screenshots small and limited to what the milestone spec requests.
- Do not put learner data, secrets, or environment files in evidence.

## Scope boundaries

- Native work stays in `mobile/` once RN-M1 creates it.
- The web app stays at `web/`; do not change it in a native milestone without an
  approved exception.
- The web app's no-build/no-dependency rule does not apply to `mobile/`; use the
  approved Expo/TypeScript toolchain documented in the RN specs.