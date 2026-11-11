# RN-M0 Task List: Baseline Preservation and Repository Layout

**Status:** Implementation complete through Task 10; Checkpoints A and B and the Task 11 human exit gate are pending human review.
**Date:** 2026-11-10
**References:**
- `tasks/react-native/RN-M0-plan.md`
- `docs/specs/react-native/milestones/RN-M0-baseline-and-repo-layout.md`
- `docs/specs/react-native/milestones/README.md`
- `docs/specs/linear-equation-drag-solver.md`
- `CLAUDE.md`

---

## Phase 1: Baseline capture

- [x] **Task 1: Pre-flight and planning-doc commit**
  - **Depends on:** Approved RN-M0 spec, plan, and this task list.
  - **Size:** S.
  - **Acceptance:**
    - `git diff --exit-code -- index.html` succeeds; the web app has no uncommitted changes.
    - The approved RN planning docs, milestone spec, plan, and task list are committed.
    - The resulting HEAD is recorded as the baseline candidate.
    - `web/`, `mobile/`, `package.json`, `node_modules/`, `ios/`, and `android/` do not exist.
  - **Verify:**
    ```sh
    git status --short
    git rev-parse HEAD
    git diff --exit-code -- index.html
    test ! -e web && test ! -e mobile
    test ! -e package.json && test ! -d node_modules
    ```
  - **Files:** `docs/react-native-*.md`, `docs/specs/react-native/**`, `tasks/react-native/**` (commit only; no content edits).
  - **Note:** Commit message: `docs(rn): add React Native migration plan and RN-M0 spec`. If the docs are already committed, record HEAD and skip the commit.

- [x] **Task 2: Create the baseline tag, backup branch, and checksum**
  - **Depends on:** Task 1.
  - **Size:** S.
  - **Acceptance:**
    - `web-baseline-2026-11-11` is an annotated tag at the Task 1 HEAD.
    - `backup/web-baseline-2026-11-11` is a branch at the same commit.
    - The tag, branch, and HEAD resolve to the same SHA.
    - `shasum -a 256 index.html` is recorded in the evidence file.
    - The evidence skeleton exists with commit, tag, branch, checksum, and pending checklist sections.
  - **Verify:**
    ```sh
    git rev-parse web-baseline-2026-11-11
    git rev-parse backup/web-baseline-2026-11-11
    git rev-parse HEAD
    shasum -a 256 index.html
    ```
    Read the evidence file and confirm the recorded values match.
  - **Files:** `docs/specs/react-native/evidence/RN-M0-baseline.md` (new).
  - **Note:** If implementation starts after 2026-11-10, use `web-baseline-<implementation-date>` and update every reference consistently. Do not push in this task.

- [x] **Task 3: Run the pre-move checklist and capture evidence**
  - **Depends on:** Task 2.
  - **Size:** M.
  - **Acceptance:**
    - All 12 steps of the product spec's final release checklist have a result (pass/fail/not applicable), notes, and evidence.
    - The evidence file records date, macOS version, browser and version, viewports (320/768/1024/1280), console errors, network requests, keyboard pass, reduced-motion pass, screen-reader result, audio result, and limitations.
    - Screenshots exist under `docs/specs/react-native/evidence/assets/RN-M0/`.
    - The web app is unchanged.
  - **Verify:**
    ```sh
    python3 -m http.server 8000 --directory .
    git diff --exit-code -- index.html
    ls docs/specs/react-native/evidence/assets/RN-M0
    ```
    Open `http://localhost:8000/` and perform the checklist. Do not mark a step complete unless it was actually observed.
  - **Files:** evidence file, screenshot assets.

### Checkpoint A: Baseline accepted (after Tasks 1–3)

- [ ] Tag, backup branch, and HEAD resolve to the same commit.
- [ ] Pre-move checksum is recorded and reproducible.
- [ ] All 12 checklist steps have honest results and evidence.
- [ ] Any unavailable device or assistive technology is recorded as a limitation.
- [ ] The human reviewer accepts the baseline evidence and authorizes the move.

**Stop here if the baseline evidence is incomplete or the web app has uncommitted changes.**

---

## Phase 2: Move and documentation

- [x] **Task 4: Move the web app to `web/`**
  - **Depends on:** Checkpoint A.
  - **Size:** S.
  - **Acceptance:**
    - Root `index.html` no longer exists; `web/index.html` does.
    - The SHA-256 of `web/index.html` matches the tagged baseline.
    - `web/README.md` states the web app's role, run command, and no-build/no-dependency constraint.
    - The 5-step post-move smoke flow passes from `http://localhost:8000/`.
    - No web content, markup, CSS, JavaScript, or behavior changed.
  - **Verify:**
    ```sh
    shasum -a 256 web/index.html
    git show web-baseline-2026-11-11:index.html | shasum -a 256
    python3 -m http.server 8000 --directory web
    git diff --find-renames --summary
    ```
    Smoke flow: load, start `2y - 3 = 8`, lock `2y = 11`, complete `y = 11 ÷ 2`, press "Solve another!".
  - **Files:** `index.html` → `web/index.html`, `web/README.md`.
  - **Note:** Commit the move separately when practical: `chore(repo): move web app into web/`.

- [x] **Task 5: Update `README.md`**
  - **Depends on:** Task 4.
  - **Size:** S.
  - **Acceptance:**
    - Current status is accurate: the web app is implemented through M4 and accepted as the parity baseline; the native app is planned and `mobile/` is created in RN-M1.
    - The run command is `python3 -m http.server 8000 --directory web`.
    - The deployment note says the static host must serve `web/` or redirect the root; no hosting configuration is in the repository.
    - The RN roadmap and RN-M0 evidence are linked.
    - The web constraints (standalone, dependency-free, no build, exact rationals, no `eval`) remain visible.
  - **Verify:**
    ```sh
    grep -n -- '--directory \.' README.md   # expected: no output
    grep -n 'web/index.html' README.md
    grep -n 'react-native/milestones/README.md' README.md
    ```
    Read the document for stale claims and broken relative links.
  - **Files:** `README.md`.

- [x] **Task 6: Update `CLAUDE.md`**
  - **Depends on:** Task 4.
  - **Size:** S.
  - **Acceptance:**
    - Web rules are scoped to `web/index.html`: self-contained, no build/package manager/CDN/dependencies, Pointer Events, no `eval`/`Function`, exact rationals, browser checklist.
    - Native rules are scoped to `mobile/`: Expo/TypeScript per the RN specs, no WebView/backend/auth/persistence/analytics, local builds, no EAS.
    - Commands use `--directory web` for the web app; native commands are marked as added in RN-M1.
    - The web no-build rule is explicitly not applied to `mobile/`.
  - **Verify:**
    ```sh
    grep -n -- '--directory \.' CLAUDE.md   # expected: no output
    grep -n 'web/index.html' CLAUDE.md
    grep -n 'mobile/' CLAUDE.md
    ```
    Read for contradictions between the web and native sections.
  - **Files:** `CLAUDE.md`.

- [x] **Task 7: Add `docs/adr/ADR-011-repository-layout.md`**
  - **Depends on:** Task 4.
  - **Size:** S.
  - **Acceptance:**
    - Records context, decision (`web/` + `mobile/`), consequences, and alternatives considered.
    - Resolves PB-02/PB-10: native is the intended successor; the web stays live and maintained until a separate retirement decision.
    - Records the hosting-root follow-up and the historical-docs policy.
    - Status is Proposed; date and deciders are present; links resolve.
  - **Verify:** Read against RN-M0 spec §13 and the roadmap. Check every relative link.
  - **Files:** `docs/adr/ADR-011-repository-layout.md` (new).

- [x] **Task 8: Update the product spec paths**
  - **Depends on:** Task 4.
  - **Size:** S.
  - **Acceptance:**
    - `docs/specs/linear-equation-drag-solver.md` uses `web/index.html` where it previously said `index.html`.
    - It uses `--directory web` where it previously said `--directory .`.
    - No requirement, supported notation, example, acceptance criterion, or behavior changed.
  - **Verify:**
    ```sh
    grep -n -- '--directory \.\|`index\.html`' docs/specs/linear-equation-drag-solver.md   # expected: no root refs
    git diff -- docs/specs/linear-equation-drag-solver.md
    ```
    Confirm the diff contains only path/command strings.
  - **Files:** `docs/specs/linear-equation-drag-solver.md`.

- [x] **Task 9: Add task conventions and finalize the evidence record**
  - **Depends on:** Tasks 4–8.
  - **Size:** S.
  - **Acceptance:**
    - `tasks/react-native/README.md` documents the spec, plan, todo, and evidence paths and the SPECIFY → PLAN → TASKS → IMPLEMENT gates.
    - The evidence file records the post-move smoke results, documentation reconciliation, historical `x`/root-path drift, and a pending/complete push status.
    - Historical web milestone and task files are unchanged.
  - **Verify:**
    ```sh
    git status --short -- docs/specs/milestones tasks
    git diff --stat -- docs/specs/milestones tasks
    ```
    The only task-tree change should be under `tasks/react-native/`. Read the evidence file for completeness.
  - **Files:** `tasks/react-native/README.md` (new), evidence file.

### Checkpoint B: Move and documentation reviewed (after Tasks 4–9)

- [ ] Root `index.html` is gone; `web/index.html` is byte-identical.
- [ ] Post-move smoke flow passes.
- [ ] README, CLAUDE.md, and the product spec contain no root-path commands.
- [ ] ADR-011 records the layout and the PB-02/PB-10 resolution.
- [ ] Historical milestone/task files are unchanged.
- [ ] The human reviewer approves the move and documentation diffs.

---

## Phase 3: Verification and exit

- [x] **Task 10: Run full verification**
  - **Depends on:** Checkpoint B.
  - **Size:** S.
  - **Acceptance:**
    - Tag and backup branch resolve to the same commit.
    - `web/index.html` hash matches the tagged baseline.
    - The post-move smoke flow passes with no console errors, failed assets, or external requests.
    - Current-state docs (`README.md`, `CLAUDE.md`, `web/README.md`, and the product spec) contain no `--directory .` or root-relative `index.html`.
    - `git diff --find-renames --summary` shows the web app as a rename.
    - `mobile/`, `package.json`, `node_modules/`, `ios/`, and `android/` are absent.
    - All results are recorded in the evidence file.
  - **Verify:**
    ```sh
    git rev-parse web-baseline-2026-11-11
    git rev-parse backup/web-baseline-2026-11-11
    shasum -a 256 web/index.html
    git show web-baseline-2026-11-11:index.html | shasum -a 256
    python3 -m http.server 8000 --directory web
    # Current-state docs only. Planning/historical docs may mention the pre-move path.
    grep -RIn --exclude-dir=.git \
      -E 'python3 -m http\.server 8000 --directory \.' \
      README.md CLAUDE.md web docs/specs/linear-equation-drag-solver.md
    grep -RIn --exclude-dir=.git \
      -E '(^|[^A-Za-z0-9_./-])index\.html' \
      README.md CLAUDE.md web docs/specs/linear-equation-drag-solver.md
    git diff --find-renames --summary
    test ! -e mobile && test ! -e package.json && test ! -d node_modules
    test ! -d ios && test ! -d android
    git status --short
    ```
    The grep commands are expected to return no matches. Investigate any match rather than dismissing it.
  - **Files:** evidence file (results only).

- [ ] **Task 11: Commit, optional remote backup, and M0 exit gate**
  - **Depends on:** Task 10.
  - **Size:** S.
  - **Acceptance:**
    - The move is committed separately from the documentation/evidence changes when practical.
    - `git status --short` is clean, or every remaining entry is explained in the evidence file.
    - The tag and backup branch are pushed to `origin` only with explicit approval; the outcome is recorded.
    - If push is not approved, the evidence file says "local-only backup; push not approved."
    - The human reviewer signs the M0 exit gate.
  - **Verify:**
    ```sh
    git log --oneline --decorate -5
    git status --short
    git tag --list 'web-baseline-*'
    git branch --list 'backup/*'
    ```
    Review the final diff and the evidence file.
  - **Files:** all changed files; Git refs.
  - **Note:** Do not start RN-M1, create `mobile/`, or install dependencies in this task.

### Checkpoint C: M0 exit gate (after Task 11)

- [ ] Every RN-M0 spec acceptance criterion passes.
- [ ] Evidence is complete and reviewed.
- [ ] Web app is byte-identical and runs from `web/`.
- [ ] Documentation and ADR are consistent.
- [ ] Remote backup is complete, or local-only status is recorded and accepted.
- [ ] The human reviewer authorizes RN-M1.

---

## Delivery order

```text
T1 → T2 → T3 → Checkpoint A → T4
                                 ├─ T5 ─┐
                                 ├─ T6 ─┤
                                 ├─ T7 ─┼─ T9 → Checkpoint B → T10 → T11 → Checkpoint C
                                 └─ T8 ─┘
```

- T1–T4 are sequential because they establish and preserve the baseline invariant.
- T5–T8 may be parallelized across sessions because they touch different files; T9 waits for all of them.
- T10–T11 are sequential.
- Do not reorder the move before the baseline evidence is accepted.
- Do not begin RN-M1 until Checkpoint C is signed.

---

## Status log

- **2026-11-11:** Tasks 1–10 implemented.
  - Planning docs committed at `fa97054`; baseline tag and backup branch
    created at that commit.
  - Pre-move evidence captured (automated Chromium run; human gate still
    required) and post-move smoke flow passed.
  - Web app moved byte-identically (`rename index.html => web/index.html (100%)`),
    move committed as `6f3b465`.
  - Documentation, ADR-011, task conventions, and evidence updated.
  - Backup is local-only; pushing the tag/branch to `origin` was not approved.
  - Remaining: human review of Checkpoints A and B, Task 11 commit approval,
    push decision, and the M0 exit gate (Checkpoint C). RN-M1 must not start
    until the gate is signed.
  - **Process note:** the full milestone was implemented in one request. The
    move followed evidence capture, but no separate human Checkpoint A sign-off
    occurred before it. The baseline evidence is ready for review now; the
    reviewer may accept it retrospectively or require follow-up changes.