# RN-M0 Tech Spec: Baseline Preservation and Repository Layout

**Status:** Draft for human approval. No files may be moved until this spec is approved.
**Date:** 2026-11-10
**Owner:** Boris (product owner) + solution architect
**Milestone:** RN-M0
**Governing documents:** [`README.md` (roadmap)](./README.md), [`react-native-migration-plan.md`](../../../react-native-migration-plan.md), [`react-native-architecture-plan.md`](../../../react-native-architecture-plan.md), [`react-native-implementability-review.md`](../../../react-native-implementability-review.md), [`CLAUDE.md`](../../../../CLAUDE.md)

---

## Assumptions

These are assumptions, not decisions. Correct them before implementation; otherwise this spec proceeds with them.

1. The current tracked web application at commit `5c6adc9` (or the commit approved at implementation time) is the accepted parity baseline for PB-03.
2. The web app moves to `web/index.html`; the native app will live in `mobile/`. `mobile/` is **not** created in RN-M0.
3. A Git annotated tag plus a backup branch plus a SHA-256 checksum is sufficient backup. An external archive is optional, not required.
4. The `origin` remote is `https://github.com/bbsteps-system/equation-explorer.git`. The existing `origin/v1-html` branch is historical and must not be modified.
5. Pushing the new tag and backup branch to `origin` is desirable but requires explicit approval because it is an outward-facing action.
6. Historical web milestone specs (`docs/specs/milestones/00`–`04`) and task files (`tasks/00`–`04`) remain historical records and are **not** rewritten. The authoritative current-state docs (`README.md`, `CLAUDE.md`) are corrected.
7. The product spec `docs/specs/linear-equation-drag-solver.md` already uses `y`; no `x`-to-`y` rewrite of that document is required. Remaining `x` examples in historical milestone/task docs are recorded as drift, not changed.
8. Baseline evidence is captured in the browsers and devices available on the implementation machine. If a screen reader, physical touch device, or audio device is unavailable, the gap is recorded as a limitation.
9. The repository has no hosting configuration file. If the web app is deployed externally, the hosting root must be updated by the owner; RN-M0 documents the requirement but does not change external infrastructure.
10. The empty root file `readmemd` is unrelated and is left untouched.
11. No web behavior, content, markup, CSS, JavaScript, math, or accessibility behavior changes in RN-M0. The move must be byte-identical.

---

## 1. Objective

Preserve the accepted web baseline, capture the evidence that PB-03 assumes, and relocate the web app to `web/` so the React Native app can occupy `mobile/` without path ambiguity.

**User-visible outcome:** none. The web app behaves exactly as before, from a new path. The native project does not exist yet.

**Project outcome:** the repository has an unambiguous, documented layout, a durable baseline backup, and a recorded baseline evidence package. RN-M1 can then scaffold `mobile/` without touching or confusing the web app.

**Success looks like:** a reviewer can check out the repository, run one documented command, use the web app exactly as before, open the evidence record, and see the tagged baseline commit and checksum. The web app is not modified, only moved.

---

## 2. Scope

### In scope

- Create an annotated Git tag and a backup branch at the pre-move commit.
- Record the SHA-256 checksum of the pre-move `index.html`.
- Run the web manual acceptance checklist before the move and capture evidence.
- Move `index.html` to `web/index.html` with history preserved.
- Add a short `web/README.md` explaining the legacy web app and its run command.
- Update `README.md`, `CLAUDE.md`, and `docs/specs/linear-equation-drag-solver.md` for the new location and the web/native split. In the product spec, change only the paths and serve commands (`index.html` / `--directory .` → `web/index.html` / `--directory web`); do not change its requirements, scope, or examples.
- Add `docs/adr/ADR-011-repository-layout.md`.
- Create the RN documentation and task conventions under `docs/specs/react-native/` and `tasks/react-native/`.
- Verify byte identity, path correctness, and a smoke solve after the move.
- Record the hosting-root action item and the PB-02/PB-10 interpretation.

### Non-goals

- Creating the React Native project, `mobile/`, or any npm/Expo/Node configuration.
- Installing dependencies, changing lockfiles, or adding a build system.
- Changing the web app's behavior, content, markup, CSS, JavaScript, math, or accessibility.
- Rewriting historical web milestone specs or task evidence.
- Changing the product spec's requirements, supported notation, examples, or acceptance behavior; only its file paths and serve commands are corrected.
- Changing or deleting `origin/v1-html`, tags, or branches.
- Editing external hosting, DNS, CDN, or deployment configuration.
- Adding CI, tests, analytics, or tooling.
- Fixing the stale web acceptance gates; RN-M0 records evidence, it does not re-open product acceptance.

---

## 3. Deliverables

| # | Deliverable | Path |
| --- | --- | --- |
| 1 | Annotated baseline tag | Git ref `web-baseline-2026-11-11` |
| 2 | Backup branch | Git ref `backup/web-baseline-2026-11-11` |
| 3 | Pre-move checksum record | `docs/specs/react-native/evidence/RN-M0-baseline.md` |
| 4 | Moved web app | `web/index.html` |
| 5 | Web app readme | `web/README.md` |
| 6 | Updated root readme | `README.md` |
| 7 | Updated project guidance | `CLAUDE.md` |
| 8 | Layout decision record | `docs/adr/ADR-011-repository-layout.md` |
| 9 | Baseline evidence and screenshots | `docs/specs/react-native/evidence/RN-M0-baseline.md` and `docs/specs/react-native/evidence/assets/RN-M0/` |
| 10 | RN task convention note | `tasks/react-native/README.md` |
| 11 | Updated product spec paths | `docs/specs/linear-equation-drag-solver.md` |

---

## 4. Tech stack

RN-M0 is a repository and documentation milestone. It uses no application framework.

| Concern | Tool |
| --- | --- |
| Version control | Git 2.x (annotated tags, branches, `git mv`, `git show`) |
| Shell | `zsh` on macOS (POSIX-compatible commands only) |
| Local serving | Python 3 `http.server` |
| Checksums | macOS `shasum -a 256` (or `sha256sum`) |
| Evidence | Markdown + PNG screenshots |
| Browser | Current Chromium, Safari, or Firefox for the baseline checklist |
| Accessibility evidence | VoiceOver (macOS Safari), keyboard-only pass, reduced-motion setting |
| Build system / package manager | **None added in RN-M0** |

---

## 5. Commands

All commands run from the repository root. Do not use `sudo`.

### 5.1 Pre-flight

```sh
# Confirm the working tree and baseline commit.
git status --short
git rev-parse HEAD
git log -1 --oneline

# Confirm the web app has no uncommitted content changes.
git diff --exit-code -- index.html

# Confirm no web directory exists yet.
test ! -e web && echo "web/ does not exist"
```

If `git diff --exit-code -- index.html` fails, stop. Commit or restore the web app only after human approval; do not tag a modified baseline.

### 5.2 Create the backup

```sh
# Annotated tag at the pre-move commit.
git tag -a web-baseline-2026-11-11 \
  -m "Web baseline accepted as the RN parity reference (PB-03)"

# Backup branch at the same commit.
git branch backup/web-baseline-2026-11-11

# Record the commit and checksum.
git rev-parse web-baseline-2026-11-11
git rev-parse backup/web-baseline-2026-11-11
shasum -a 256 index.html
```

Both refs must resolve to the same commit. Record the commit SHA and checksum in the evidence file.

> **Date note:** the tag uses the spec date (2026-11-10). If implementation starts on a later date, use `web-baseline-<implementation-date>` and update every reference in this spec and the evidence file consistently. Do not mix tag names.

### 5.3 Capture baseline evidence (before the move)

```sh
python3 -m http.server 8000 --directory .
```

Open `http://localhost:8000/` and run the checklist in §8. Capture screenshots into `docs/specs/react-native/evidence/assets/RN-M0/`.

### 5.4 Move the web app

```sh
mkdir -p web
git mv index.html web/index.html
```

Add `web/README.md` with the web app's run command and its role as the parity reference.

### 5.5 Verify byte identity

```sh
shasum -a 256 web/index.html
git show web-baseline-2026-11-11:index.html | shasum -a 256
```

The two hashes must match exactly.

### 5.6 Verify the moved app

```sh
python3 -m http.server 8000 --directory web
```

Open `http://localhost:8000/` and run the smoke flow in §8.2.

### 5.7 Check for stale operational paths

```sh
# Current-state docs only. Planning/historical docs may legitimately mention the pre-move path.
grep -RIn --exclude-dir=.git \
  -E 'python3 -m http\.server 8000 --directory \.' \
  README.md CLAUDE.md web docs/specs/linear-equation-drag-solver.md

grep -RIn --exclude-dir=.git \
  -E '(^|[^A-Za-z0-9_./-])index\.html' \
  README.md CLAUDE.md web docs/specs/linear-equation-drag-solver.md
```

Expected: no matches. If a match appears in `web/README.md` or the product spec, fix it. Historical files under `docs/specs/milestones/` and `tasks/`, and the RN-M0 planning artifacts, may mention `index.html`; record that as historical/planning drift, do not rewrite them. The ADR may mention the old path as context; review it manually rather than treating it as a current instruction.

### 5.8 Commit

```sh
git status --short
git add web README.md CLAUDE.md docs/adr docs/specs/linear-equation-drag-solver.md \
  docs/specs/react-native tasks/react-native
git commit -m "chore(repo): move web app into web/ and record RN baseline"
```

Use separate commits if practical:

```sh
git commit -m "docs(rn): add RN roadmap and RN-M0 spec"
git commit -m "chore(repo): move web app into web/"
git commit -m "docs(rn): record RN-M0 baseline evidence and layout ADR"
```

Do not mix unrelated content changes into the move commit.

### 5.9 Optional remote backup (ask first)

```sh
git push origin web-baseline-2026-11-11
git push origin backup/web-baseline-2026-11-11
```

Only run after explicit approval. If not approved or the push fails, record "local-only backup" in the evidence file.

### 5.10 Final verification

```sh
git status --short
git tag --list 'web-baseline-*'
git branch --list 'backup/*'
git log --oneline --decorate -5
```

---

## 6. Project structure

### 6.1 Before RN-M0

```text
solve-equation/
├── index.html                       # web app at repository root
├── README.md
├── CLAUDE.md
├── .gitignore
├── docs/
│   ├── specs/
│   │   ├── linear-equation-drag-solver.md
│   │   ├── milestones/00-…04-…      # historical web milestones
│   │   └── react-native/
│   │       └── milestones/
│   │           ├── README.md        # RN roadmap
│   │           └── RN-M0-baseline-and-repo-layout.md
│   ├── react-native-architecture-plan.md
│   ├── react-native-migration-plan.md
│   └── react-native-requirements-open-questions.md
└── tasks/
    ├── 00-…04-…                     # historical web tasks
    └── react-native/
        ├── RN-M0-plan.md
        └── RN-M0-todo.md
```

### 6.2 After RN-M0

```text
solve-equation/
├── web/
│   ├── index.html                   # byte-identical moved web app
│   └── README.md                    # legacy web app run/role note
├── README.md                        # updated current status + path
├── CLAUDE.md                        # updated web/native rules
├── .gitignore                       # unchanged in M0
├── docs/
│   ├── adr/
│   │   └── ADR-011-repository-layout.md
│   ├── specs/
│   │   ├── linear-equation-drag-solver.md
│   │   ├── milestones/00-…04-…      # unchanged historical web milestones
│   │   └── react-native/
│   │       ├── milestones/
│   │       │   ├── README.md        # RN roadmap
│   │       │   └── RN-M0-baseline-and-repo-layout.md
│   │       └── evidence/
│   │           ├── RN-M0-baseline.md
│   │           └── assets/RN-M0/    # screenshots
│   ├── react-native-architecture-plan.md
│   ├── react-native-implementability-review.md
│   ├── react-native-migration-plan.md
│   └── react-native-requirements-open-questions.md
└── tasks/
    ├── 00-…04-…                     # unchanged historical web tasks
    └── react-native/
        ├── README.md                # RN task convention note
        ├── RN-M0-plan.md
        └── RN-M0-todo.md
```

`mobile/` is deliberately absent. RN-M1 creates it.

---

## 7. Code style and documentation conventions

RN-M0 adds no application code. The relevant style is documentation and Git hygiene.

### 7.1 Markdown

- One H1 per document; ATX headings (`#`, `##`, `###`).
- Fenced code blocks with a language tag (`sh`, `text`, `ts`).
- Repository-relative paths in backticks: `web/index.html`, not absolute paths.
- One sentence per line is not required, but keep lines readable and avoid trailing whitespace.
- Relative links must resolve from the file's location.
- Use `y` for the variable in current-state documentation. Historical documents keep their original `x` examples.

### 7.2 Commands in docs

Document the exact command a reader can paste, including the directory:

```sh
# Web app (after RN-M0)
python3 -m http.server 8000 --directory web
# Open http://localhost:8000/
```

Do not write only "serve the app" or omit the directory.

### 7.3 Commit messages

Use imperative, scoped messages:

```text
docs(rn): add RN roadmap and RN-M0 spec
chore(repo): move web app into web/
docs(rn): record RN-M0 baseline evidence and layout ADR
```

Do not rewrite history, force-push, or amend a pushed commit.

### 7.4 Path references in current-state docs

Use:

- `web/index.html` — the web app.
- `mobile/` — the future native app (created in RN-M1).
- `docs/specs/react-native/` — native specifications.
- `tasks/react-native/` — native task plans.
- `docs/adr/` — standalone architecture decision records.

Avoid "the app" without saying which app. Say "the web app" or "the native app".

---

## 8. Testing strategy

RN-M0 has no automated test framework. Verification is checksum, runtime smoke, path inspection, and the existing manual web checklist.

### 8.1 Pre-move baseline checklist

Run every step in `docs/specs/linear-equation-drag-solver.md` §"Final release checklist" (12 steps) against the root `index.html` served with `--directory .`.

Record for each step:

- Result: pass / fail / not applicable.
- Notes: observed behavior, any deviation.
- Evidence: screenshot filename or short recording note.

Also record:

- Date and time.
- macOS version.
- Browser name and exact version.
- Viewports tested: 320 px, 768 px, 1024 px (tablet), 1280 px.
- Console errors (expected: none).
- Network requests (expected: only the local document; no external requests).
- Keyboard-only pass and focus visibility.
- Reduced-motion pass.
- Screen reader pass (VoiceOver) if available; otherwise "not performed" plus the reason.
- Audio behavior if available; otherwise "not performed" plus the reason.
- Any limitation that affects confidence in the baseline.

### 8.2 Post-move smoke flow

Serve from `--directory web` and verify:

1. Landing renders with no console errors and no external requests.
2. Enter `2y - 3 = 8`; Step 1 locks and Step 2 is prefilled.
3. Build `2y = 11`; Check accepts and locks Step 2.
4. Build `y = 11 ÷ 2`; Check completes with the correct answer.
5. Press "Solve another!"; the setup screen returns and focus moves to the input.

If the checksum matches and this smoke flow passes, the move is behavior-preserving.

### 8.3 Integrity checks

- Tag and backup branch resolve to the same commit.
- Pre-move checksum matches `web/index.html`.
- `git diff --find-renames --summary` shows the web app as a rename, not a delete plus add with content changes.
- No `mobile/`, `node_modules/`, `package.json`, `ios/`, or `android/` appears.
- No authoritative doc still instructs `--directory .`.

### 8.4 Evidence file template

Create `docs/specs/react-native/evidence/RN-M0-baseline.md`:

```markdown
# RN-M0 Baseline Evidence

- Date:
- Baseline commit:
- Baseline tag:
- Backup branch:
- SHA-256 of pre-move index.html:
- SHA-256 of web/index.html:
- Tag/branch pushed to origin: yes / no / not approved
- macOS:
- Browser(s):
- Viewports:
- Serving command (pre-move):
- Serving command (post-move):
- Console errors:
- External network requests:
- Keyboard pass:
- Reduced-motion pass:
- Screen reader pass:
- Audio pass:
- Limitations:

## Pre-move checklist results

| Step | Result | Notes | Evidence |
| --- | --- | --- | --- |
| 1 | | | |
| … | | | |
| 12 | | | |

## Post-move smoke flow

| Step | Result | Notes |
| --- | --- | --- |
| 1 | | |
| … | | |
| 5 | | |

## Documentation reconciliation

- README current status corrected: yes / no
- README run command updated: yes / no
- CLAUDE.md web/native split updated: yes / no
- Product spec paths updated (`web/index.html`, `--directory web`): yes / no
- Historical x/y drift recorded, not rewritten: yes / no
- Hosting-root action item recorded: yes / no

## Screenshots

- `assets/RN-M0/landing-320.png`
- `assets/RN-M0/landing-768.png`
- `assets/RN-M0/landing-1280.png`
- `assets/RN-M0/started-step2-768.png`
- `assets/RN-M0/completed-768.png`
- Additional evidence as needed.
```

---

## 9. Acceptance criteria

- [ ] An annotated tag `web-baseline-2026-11-11` and branch `backup/web-baseline-2026-11-11` exist at the same pre-move commit.
- [ ] The baseline commit SHA and the pre-move SHA-256 checksum are recorded in the evidence file.
- [ ] The tag and backup branch are pushed to `origin`, or the evidence file explicitly records "local-only backup; push not approved."
- [ ] `web/index.html` is byte-identical to the tagged baseline (matching SHA-256).
- [ ] `web/README.md` exists and states the web app's role, run command, and relationship to the native app.
- [ ] The web app serves from `python3 -m http.server 8000 --directory web` with no console errors, no failed assets, and no external network requests.
- [ ] The post-move smoke flow in §8.2 passes.
- [ ] `README.md` has the new run command, accurate current status, deployment-root note, and a link to the RN roadmap.
- [ ] `CLAUDE.md` distinguishes web rules (`web/index.html`, no build/dependencies) from native rules (`mobile/`, RN specs), and uses the new serve command.
- [ ] `docs/specs/linear-equation-drag-solver.md` uses `web/index.html` and `--directory web` in its deliverable, markup, command, and checklist references, with no requirement or example changes.
- [ ] `docs/adr/ADR-011-repository-layout.md` records the layout decision, the `web/` + `mobile/` split, the PB-02/PB-10 interpretation, and the hosting follow-up.
- [ ] `docs/specs/react-native/evidence/RN-M0-baseline.md` contains the full pre-move checklist, environment details, limitations, screenshots, and post-move smoke results.
- [ ] `tasks/react-native/README.md` documents the native spec/plan/todo naming convention.
- [ ] Historical web milestone and task files are unchanged, except for the explicitly approved authoritative docs.
- [ ] No React Native project, dependency, lockfile, `mobile/` directory, CI change, or web behavior change is introduced.
- [ ] `git status --short` is clean after the approved commits, or every remaining entry is explained in the evidence file.
- [ ] The human reviewer signs the M0 exit gate.

---

## 10. Execution sequence

```text
Task 1: Pre-flight, commit the approved RN planning docs, confirm the web app is unmodified
  └─ Task 2: Create and verify the baseline tag, backup branch, and checksum
       └─ Task 3: Run the pre-move baseline checklist and capture evidence
            └─ Task 4: Move index.html to web/, add web/README.md
                 └─ Task 5: Update README.md, CLAUDE.md, and add ADR-011
                      └─ Task 6: Update docs/specs/react-native/ evidence and task conventions
                           └─ Task 7: Verify byte identity, smoke flow, paths, and git state
                                └─ Task 8: Commit; optionally push the backup; human gate
```

Tasks 1–7 are sequential because they touch the same repository state. Task 8 is the human gate. The implementation plan `tasks/react-native/RN-M0-plan.md` refines this sequence into T1–T11; use the plan's task IDs for execution and this spec for acceptance.

**Checkpoints:**

- **After Task 2:** tag and branch point to the same commit; checksum recorded.
- **After Task 3:** evidence file has all 12 checklist results and environment details.
- **After Task 4:** `web/index.html` hash matches the baseline.
- **After Task 7:** smoke flow passes; no stale root-serving commands remain.
- **Task 8:** human approval.

---

## 11. Boundaries

### Always

- Tag and checksum the baseline **before** moving anything.
- Keep `web/index.html` byte-identical to the tagged baseline.
- Record the actual commands, browser versions, viewports, and results; never invent evidence.
- Update authoritative current-state docs (`README.md`, `CLAUDE.md`) and the ADR.
- Keep historical web milestone/task docs as historical records.
- Preserve `origin/v1-html` and all existing refs.
- Get human approval before pushing tags/branches to `origin`.
- Run the pre-move checklist and the post-move smoke flow before declaring M0 complete.

### Ask first

- Push the tag or backup branch to `origin`.
- Update historical web milestone/task docs beyond recording drift.
- Add a root-level redirect or modify external hosting/DNS/CDN configuration.
- Store binary screenshots in Git instead of an external evidence store.
- Delete, rename, or force-push any existing branch, tag, or commit.
- Add a dependency, build tool, CI workflow, or `mobile/` directory in M0.
- Change the accepted baseline commit if `index.html` has uncommitted changes.

### Never

- Edit the content, behavior, math, UI, CSS, or accessibility of `web/index.html`.
- Tag a modified or uncommitted web app as the baseline.
- Rewrite Git history, force-push `main`, or delete `origin/v1-html`.
- Create the React Native project, install npm/Expo packages, or add `package.json` in M0.
- Change external hosting or deployment without explicit approval.
- Claim the baseline checklist passed if a step was not run.
- Put learner data, secrets, or environment files in the evidence package.

---

## 12. Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Moving the web app breaks an external deployment root | Medium | High | Record the hosting-root action item in ADR-011 and README; verify locally; owner updates external hosting separately. |
| The backup tag/branch is never pushed and exists only locally | Medium | Medium | Make the push an explicit approval item; record the outcome in evidence. |
| The web app is accidentally edited during the move | Low | High | Checksum before/after; `git diff` review; byte-identity acceptance criterion. |
| Historical `x` examples confuse readers | Medium | Low | Record the drift in the evidence file; keep historical docs unchanged; current docs use `y`. |
| Screenshots bloat the repository | Low | Low | Use a small set of PNGs; ask first before adding large recordings. |
| The evidence checklist is partially run and reported as complete | Medium | High | Require per-step results and limitations; human gate reviews the evidence file. |
| Tagging the wrong commit | Low | High | Record `git rev-parse HEAD` and the tag target; verify both refs resolve to the same SHA. |

---

## 13. Open questions

These must be answered during spec review or before the affected task.

1. **Deployment root:** Is the web app currently deployed from the repository root (for example S3/CloudFront)? If yes, who updates the origin path to `web/` or adds a redirect? *Recommendation:* update the hosting origin to `web/`; do not add a root redirect file unless the host cannot change.*
2. **Remote backup:** Approve pushing `web-baseline-2026-11-11` and `backup/web-baseline-2026-11-11` to `origin`? *Recommendation:* yes; a local-only tag is not a durable backup.*
3. **Evidence storage:** Are small PNG screenshots acceptable in Git under `docs/specs/react-native/evidence/assets/RN-M0/`? *Recommendation:* yes, for a small set; use an external store only if recordings are large.*
4. **Historical docs:** Confirm that `docs/specs/milestones/00`–`04` and `tasks/00`–`04` stay unchanged even though some examples use `x`. The product spec is authoritative and will have its paths updated, but not its requirements or examples. *Recommendation:* leave the historical milestone/task files as records and record the drift.*
5. **Root URL stability:** If the deployed root URL must remain `https://…/` with no `/web/` path, should a CDN redirect or a root `index.html` redirect be added? *Recommendation:* prefer a CDN/hosting redirect; a root redirect file adds a second web entry point and should be a separate, approved change.*
6. **Baseline commit:** Is `5c6adc9` the approved baseline, or should implementation tag the commit that includes the approved RN planning docs? *Recommendation:* commit the planning docs first, then tag the resulting commit before the move; record the SHA.*

---

## 14. Exit gate

**Entry:** This spec and the RN roadmap are approved; the open questions above are answered or explicitly deferred with an owner.

**Done:**

- Every acceptance criterion in §9 passes.
- The evidence file is complete and reviewed.
- A human reviewer approves the repository layout, the web move, and the baseline evidence.
- The reviewer confirms that `web/index.html` is byte-identical and the web app still runs from `web/`.

**Next:** RN-M1 may start only after this gate. RN-M1 creates `mobile/`, scaffolds the Expo project, and runs the feasibility spike. Do not create `mobile/` or install React Native dependencies during RN-M0.