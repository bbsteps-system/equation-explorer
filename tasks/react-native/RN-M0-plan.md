# RN-M0 Implementation Plan: Baseline Preservation and Repository Layout

**Status:** Draft for human review. Do not implement until this plan and `RN-M0-todo.md` are approved.
**Date:** 2026-11-10
**Milestone:** RN-M0
**Governing specs:**
- `docs/specs/react-native/milestones/RN-M0-baseline-and-repo-layout.md` (primary)
- `docs/specs/react-native/milestones/README.md` (roadmap)
- `docs/specs/linear-equation-drag-solver.md` (web product spec)
- `docs/react-native-implementability-review.md` (F01, F10, F17)
- `CLAUDE.md`

---

## Outcome

RN-M0 leaves the repository in this state:

- The accepted web app is preserved at a tagged, checksummed baseline.
- The web app lives at `web/index.html` and is byte-identical to the tagged baseline.
- Baseline evidence exists: the 12-step web checklist, environment details, screenshots, and known limitations.
- `README.md`, `CLAUDE.md`, and the authoritative product spec point to `web/` correctly.
- `docs/adr/ADR-011-repository-layout.md` records the `web/` + `mobile/` layout and the PB-02/PB-10 resolution.
- `tasks/react-native/README.md` documents the native spec/plan/todo/evidence conventions.
- No `mobile/`, no React Native dependency, no package manager, and no web behavior change exists.

RN-M1 can then create `mobile/` without touching or confusing the web app.

---

## Architecture decisions

1. **Backup before move.** An annotated tag, a backup branch, and a SHA-256 checksum are created at the pre-move commit. The tag is the durable reference; the branch is the easy checkout. A remote push is an approval-gated follow-up, not a prerequisite for the local move.
2. **Evidence before move.** The 12-step checklist is run against the root `index.html` before the move. The evidence file records what was actually observed. A missing screen reader, touch device, or audio device is recorded as a limitation, never invented.
3. **Byte-identical move.** `git mv index.html web/index.html` is the only change to the web app. Pre-move and post-move SHA-256 must match. Any content difference is a defect, not a refactor opportunity.
4. **Two documentation classes.** Authoritative current-state docs (`README.md`, `CLAUDE.md`, `docs/specs/linear-equation-drag-solver.md`) are corrected for the new path. Historical web milestone/task files (`docs/specs/milestones/00`–`04`, `tasks/00`–`04`) stay unchanged; `x` examples and root paths there are recorded as historical drift.
5. **No native scaffolding.** `mobile/` is created in RN-M1, not RN-M0. RN-M0 installs no dependency and adds no build system.
6. **Hosting is external.** The repository has no hosting configuration. RN-M0 documents that the static host must serve `web/` or redirect the root; it does not change DNS, CDN, or deployment settings.
7. **Commit boundaries.** The planning docs are committed before tagging. The move is committed separately from the documentation/evidence changes when practical, so the rename is reviewable.

### Spec correction discovered during planning

The original RN-M0 spec said the product spec already used `y`, which is true, but it did not account for root-path references in that same authoritative document. `docs/specs/linear-equation-drag-solver.md` still says `index.html` and `--directory .` at lines 18, 27, 37, 50, and 185 (or any other root-path reference). The RN-M0 spec has been corrected: the product spec's **paths** are updated, but its requirements, notation, and examples are not. This is a documentation correction, not a scope expansion.

---

## Dependency graph

```text
Approved RN-M0 spec + plan/todo
  └─ T1 → T2 → T3 → ◆ Checkpoint A
                       └─ T4
                            ├─ T5 ─┐
                            ├─ T6 ─┤
                            ├─ T7 ─┼─ T9 → ◆ Checkpoint B → T10 → T11 → ◆ Checkpoint C
                            └─ T8 ─┘
```

**Must be sequential:** T1–T4 (shared repository state and the baseline invariant), T9–T11 (evidence and verification depend on all edits).

**Safe to parallelize with multiple sessions:** T5, T6, T7, and T8 touch different files and can run concurrently after T4. T9 must wait for all four. With one session, run them in order.

---

## Implementation phases

### Phase 1: Baseline capture (T1–T3)

**Goal:** Make the accepted web baseline reproducible and evidenced before anything moves.

**T1 — Pre-flight and planning-doc commit.**
Confirm the web app is unmodified (`git diff --exit-code -- index.html`), confirm there is no `web/`, `mobile/`, `package.json`, or `node_modules/`, and commit the approved RN planning/spec/task docs. Record the resulting HEAD as the baseline candidate. If the docs are already committed, skip the commit and record HEAD.

**T2 — Baseline tag, backup branch, checksum.**
Create `web-baseline-2026-11-11` (annotated) and `backup/web-baseline-2026-11-11` at the T1 commit. Verify both resolve to the same SHA. Record the SHA and `shasum -a 256 index.html` in the evidence skeleton. The evidence file is created after the tag, so it is intentionally not part of the tag; the tag preserves the web app and planning docs.

**T3 — Pre-move checklist and evidence.**
Serve the root app with `python3 -m http.server 8000 --directory .` and run all 12 steps of the product spec's final release checklist. Capture the environment (date, macOS, browser and version, viewports 320/768/1024/1280, console, network, keyboard, reduced motion, screen reader, audio) and per-step results. Store screenshots under `docs/specs/react-native/evidence/assets/RN-M0/`. Do not edit `index.html`.

**◆ Checkpoint A — after T3.**
The human reviewer reads the evidence file and accepts the baseline. If a checklist step fails, stop and decide whether the failure is a known baseline limitation or a defect that must be recorded before the move. Do not move the file until the evidence is accepted.

---

### Phase 2: Move and documentation (T4–T9)

**Goal:** Relocate the web app without changing it, then make every authoritative document tell the truth about the new layout.

**T4 — Move to `web/`.**
`mkdir -p web && git mv index.html web/index.html`. Add `web/README.md` with the web app's role (parity baseline, still live), the run command, and the no-build/no-dependency constraint. Verify the SHA-256 matches the tag. Serve with `python3 -m http.server 8000 --directory web` and run the 5-step post-move smoke flow. Commit the move separately if practical: `chore(repo): move web app into web/`.

**T5 — README update.**
Correct the stale status (the web app is implemented through M4 and accepted as the parity baseline), change the run command to `--directory web`, document that the static host must serve `web/` or redirect the root, link the RN roadmap and RN-M0 evidence, and keep the web constraints visible. Add a short repository-layout section.

**T6 — CLAUDE.md update.**
Split the guidance into web and native rules. Web: `web/index.html`, self-contained, no build/package manager/CDN/dependencies, Pointer Events, no `eval`/`Function`, exact rationals, browser checklist. Native: `mobile/` (created in RN-M1), Expo/TypeScript per the RN specs, no WebView/backend/auth/persistence/analytics, local builds, no EAS. Make explicit that the web no-build rule does not apply to `mobile/`.

**T7 — ADR-011.**
Record context, decision (`web/` + `mobile/`), consequences, alternatives (web at root, separate repos), the PB-02/PB-10 resolution ("native is the intended successor; the web stays live and maintained until a separate retirement decision"), the hosting follow-up, and the historical-docs policy. Status: Proposed.

**T8 — Product spec path update.**
Change only the paths in `docs/specs/linear-equation-drag-solver.md`: `index.html` → `web/index.html`, `--directory .` → `--directory web`. Review the diff to confirm no requirement, notation, example, or acceptance behavior changed. The product spec remains the web product spec.

**T9 — Task conventions and evidence finalization.**
Create `tasks/react-native/README.md` with the naming conventions: specs `docs/specs/react-native/milestones/RN-M<N>-<slug>.md`, plans `tasks/react-native/RN-M<N>-plan.md`, todos `RN-M<N>-todo.md`, evidence `docs/specs/react-native/evidence/RN-M<N>-<slug>.md`, assets under `evidence/assets/RN-M<N>/`. Add the post-move smoke results, documentation reconciliation, historical-drift note, and push-status placeholder to the evidence file.

**◆ Checkpoint B — after T9.**
The human reviewer checks the move (checksum + smoke), the README/CLAUDE/product-spec diffs, and ADR-011. No documentation contradiction should remain.

---

### Phase 3: Verification and exit (T10–T11)

**Goal:** Prove the move changed nothing, record the result, and hand off to RN-M1.

**T10 — Full verification.**
Run the checks in the RN-M0 spec §8.3: tag/branch equality, checksum equality, rename detection, post-move smoke, stale-path grep, and the absence of `mobile/`, `package.json`, `node_modules/`, `ios/`, and `android/`. Record the results in the evidence file. Any failure blocks T11.

**T11 — Commit, optional remote backup, exit gate.**
Commit the documentation/evidence/ADR changes (separate from the move commit if practical). Review `git log`, `git status`, `git tag`, and `git branch`. Ask for approval before pushing the tag and backup branch to `origin`; if approved, push both and record the result. If not approved, record "local-only backup." The human reviewer signs the M0 exit gate. Do not start RN-M1 in this task.

**◆ Checkpoint C — M0 exit gate.**
All spec acceptance criteria pass, the evidence is complete, the reviewer approves, and RN-M1 is authorized.

---

## Verification plan

### Repository integrity

- `git rev-parse web-baseline-2026-11-11` equals `git rev-parse backup/web-baseline-2026-11-11`.
- `shasum -a 256 web/index.html` equals `git show web-baseline-2026-11-11:index.html | shasum -a 256`.
- `git diff --find-renames --summary` shows a rename, not a content rewrite.
- `git status --short` contains only expected changes or is clean after commits.

### Runtime

- Pre-move: all 12 checklist steps run against `--directory .`.
- Post-move: 5-step smoke flow runs against `--directory web`.
- No console errors, no failed assets, no external network requests.
- The app still opens directly from `web/index.html` if `file://` behavior is checked.

### Documentation

- No current-state doc (`README.md`, `CLAUDE.md`, `web/README.md`, product spec) says `--directory .` or root `index.html`; planning and historical docs may mention the pre-move path.
- Historical milestone/task docs are unchanged.
- README/CLAUDE/product-spec/ADR links resolve.
- The web no-build rule is scoped to `web/`; the native toolchain is scoped to `mobile/`.

### Scope

- `test ! -e mobile` passes.
- No `package.json`, `package-lock.json`, `node_modules/`, `ios/`, or `android/` is added.
- `git diff --exit-code -- web/index.html` against the tag is empty (byte identity).

---

## Risks and mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| The tag is created before the planning docs are committed, so the baseline does not include the plan | Medium | T1 commits the approved docs first; T2 tags the resulting HEAD and records the SHA. |
| The web app is accidentally edited during the move | High | Checksum before/after; `git diff` review; byte-identity blocks T10. |
| An external deployment still serves the repository root | High | ADR-011 and README record the hosting-root action; the owner updates hosting separately. |
| The product spec path update accidentally changes a requirement or example | Medium | T8 review limits the diff to path strings; `git diff` is part of Checkpoint B. |
| The evidence checklist is partially run but reported as complete | High | Per-step results, environment details, and limitations; Checkpoint A reviews the evidence. |
| Historical `x` examples or root paths are mistaken for current requirements | Low | Record the drift in the evidence file; do not rewrite historical docs. |
| The backup stays local only | Medium | T11 asks for push approval and records the outcome. |
| The move commit is mixed with docs, hiding the rename | Medium | Commit the move separately when practical; use `--find-renames`. |
| A screen reader, audio device, or touch device is unavailable | Low | Record the limitation and the manual checks that were actually performed. |

---

## Definition of ready for implementation

- [ ] RN-M0 spec reviewed and approved, including the product-spec path correction.
- [ ] This plan and `RN-M0-todo.md` reviewed and approved.
- [ ] Open questions answered or explicitly deferred with an owner.
- [ ] The baseline commit (`5c6adc9` or the T1 commit) is confirmed.
- [ ] A current browser is available for the 12-step checklist.
- [ ] The push decision (approve now, later, or local-only) is known.
- [ ] `index.html` has no uncommitted changes.

---

## Definition of done

- Every acceptance criterion in the RN-M0 spec §9 passes.
- The evidence file is complete and reviewed.
- The move is byte-identical and the post-move smoke flow passes.
- Authoritative docs and ADR-011 are consistent; historical docs are unchanged.
- No `mobile/`, dependency, or web behavior change is introduced.
- The human reviewer signs the M0 exit gate and authorizes RN-M1.

---

## Open questions

| # | Question | Recommendation | Needed by |
| --- | --- | --- | --- |
| 1 | Is the web app deployed from the repository root, and who updates the hosting origin to `web/`? | Update the hosting origin; add a redirect only if the host cannot change. | Before release; can be recorded in M0 |
| 2 | Approve pushing the tag and backup branch to `origin`? | Yes; a local-only tag is not a durable backup. | T11 |
| 3 | Are small PNG screenshots acceptable in Git? | Yes, for a small evidence set. | T3 |
| 4 | Keep historical `x` examples and root paths unchanged? | Yes; record the drift, update only authoritative docs. | T9 |
| 5 | Must the deployed root URL stay exactly `/`? | Prefer a hosting/CDN redirect over a root `index.html`. | Before release |
| 6 | Tag `5c6adc9` or the T1 commit? | Tag the T1 commit so the baseline includes the approved plan; record the SHA. | T2 |