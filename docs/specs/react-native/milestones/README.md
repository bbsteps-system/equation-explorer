# React Native Milestone Roadmap — Equation Explorer

**Status:** Proposed. No implementation is authorized until this roadmap and the M0 spec are approved.
**Date:** 2026-11-10
**Companions:** [`../../react-native-migration-plan.md`](../../react-native-migration-plan.md), [`../../react-native-architecture-plan.md`](../../react-native-architecture-plan.md), [`../../react-native-requirements-open-questions.md`](../../react-native-requirements-open-questions.md), [`../../react-native-implementability-review.md`](../../react-native-implementability-review.md)
**Target:** A native iOS and Android app built with React Native + TypeScript + Expo modules, tablet-first with phone support, single light theme, New Architecture, local Xcode/Gradle builds, no WebView.

---

## 0. How this roadmap is used with spec-driven development

Each milestone is a self-contained spec-driven unit. Do not start a milestone until its predecessor's human gate is signed.

For each milestone:

```
SPECIFY ──→ PLAN ──→ TASKS ──→ IMPLEMENT ──→ HUMAN GATE
```

| Artifact | Location |
| --- | --- |
| Milestone spec | `docs/specs/react-native/milestones/RN-M<N>-<slug>.md` |
| Implementation plan | `tasks/react-native/RN-M<N>-plan.md` |
| Task checklist | `tasks/react-native/RN-M<N>-todo.md` |
| Evidence (test results, screenshots, device notes) | `docs/specs/react-native/evidence/RN-M<N>-<slug>.md` |
| ADR (architecture decisions) | `docs/adr/ADR-NNN-<slug>.md` |

Every milestone spec must contain the six spec-driven areas plus gates:

1. **Objective** — user-visible outcome and why it matters.
2. **Scope and non-goals** — explicit boundaries.
3. **Decisions and assumptions** — entry decisions resolved, remaining assumptions surfaced.
4. **Commands** — install, run, test, lint, typecheck, build, E2E.
5. **Project structure** — files and folders added or changed.
6. **Code style** — one real snippet and the conventions that matter.
7. **Testing strategy** — unit, component, integration, E2E, manual.
8. **Acceptance criteria** — testable, observable, with manual checks where automation cannot prove the outcome.
9. **Boundaries** — Always / Ask first / Never.
10. **Open questions** — anything that still needs a human answer.
11. **Exit gate** — who signs and what evidence is recorded.

**Definition of Done for every milestone:**

- [ ] Acceptance criteria pass on the stated devices or simulators.
- [ ] `tsc --noEmit`, lint, and the required tests pass.
- [ ] No network request, secret, analytics event, or persistence was introduced.
- [ ] Accessibility contract is implemented and tested at the milestone's level.
- [ ] Documentation and evidence are updated.
- [ ] A human reviewer signs the milestone gate.

### Numbering conventions

This repository contains two unrelated milestone sequences. Do not mix them.

| Sequence | IDs | Files | Meaning |
| --- | --- | --- | --- |
| Web app milestones | `M0`–`M4` | `docs/specs/milestones/00-…` through `04-…`; `tasks/00-…` through `04-…` | The historical phases of the existing HTML app. They are complete planning units and are not reused for native work. |
| React Native milestones | `RN-M0`–`RN-M8` | `docs/specs/react-native/milestones/RN-M0-…` through `RN-M8-…`; `tasks/react-native/RN-M0-…` through `RN-M8-…` | The new native app sequence defined in this roadmap. |

The bare `00`, `01`, `02` prefixes are only filename sort prefixes; they are **not** a third numbering system. For native work, the `RN-M…` ID is authoritative:

- `RN-M0` = baseline preservation and repository layout
- `RN-M1` = foundation and feasibility spike
- `RN-M2` = native UI architecture
- `RN-M3` = exact math engine
- `RN-M4` = session state and tap/toolbar skeleton
- `RN-M5` = native interactions and celebration
- `RN-M6` = testing, accessibility, and release candidate
- `RN-M7` = UAT and store preparation
- `RN-M8` = production release

Native files always start with the full ID (`RN-M0-baseline-and-repo-layout.md`, not `00-baseline-and-repo-layout.md`) so no one confuses them with the web milestones.

The two sequences overlap in subject matter but not in boundaries. The closest relationships are:

| Web milestone | Closest native milestone(s) | Note |
| --- | --- | --- |
| Web M0 Foundations/UI shell | RN-M2 Native UI architecture | Native rebuild, not an HTML port |
| Web M1 Exact math engine | RN-M3 Exact math engine | Direct TypeScript port |
| Web M2 Touch token workspace | RN-M2 (UI) + RN-M4 (tap/toolbar) + RN-M5 (drag) | Split across three native milestones |
| Web M3 Guided validation/history | RN-M4 Session state | Reducer, check, lock, history |
| Web M4 Celebration/accessibility hardening | RN-M5 (celebration) + RN-M6 (accessibility/RC) | Split across two native milestones |

---

## 1. Milestone map

| ID | Milestone | Primary outcome | Migration phase | Effort | Gate |
| --- | --- | --- | --- | ---: | --- |
| RN-M0 | Baseline preservation and repository layout | Web baseline is frozen, evidenced, and safely relocated; repository is ready for `mobile/` | 1 | 1–2 pd | M0 exit review |
| RN-M1 | Project foundation and feasibility spike | High-risk native assumptions are proven; production Expo skeleton builds on both platforms with quality gates | 2 + 3 | 6–10 pd | Spike gate A + foundation review |
| RN-M2 | Native UI architecture and static screens | Full tablet-first and phone UI with fixtures, all visual states, and accessibility contracts | 4 | 6–9 pd | Design review |
| RN-M3 | Exact math engine | Pure TypeScript engine with the complete golden corpus | 5a | 4–6 pd | Engine review |
| RN-M4 | Session state and tap/toolbar skeleton | First end-to-end tap-only solve; internal milestone, not releasable | 5b | 4–6 pd | Gate B |
| RN-M5 | Native interaction layer and celebration | Drag, audio, haptics, confetti, accessibility focus, icon/splash, back behavior | 7 | 5–8 pd | Gate C |
| RN-M6 | Testing, accessibility, and release-candidate hardening | Release-candidate quality on the device matrix | 8 | 4–7 pd | Gate D |
| RN-M7 | UAT and store preparation | User validation and submission-ready builds/listings | 9–10 | 6–10 pd | Gate E |
| RN-M8 | Production release | Live on App Store and Google Play with a support runbook | 11 | 2–3 pd | Gate F |

Migration plan Phase 6 (API/auth) is not a milestone. It is an invariant enforced in RN-M1 and RN-M6: no backend, no authentication, no network request, bundled content, fully offline.

**Total:** 38–61 person-days plus 15–20% contingency → **44–73 person-days**, excluding store review waiting time. This is a planning range, not a commitment.

---

## 2. Dependency graph

```
RN-M0  Baseline + repo layout
  │
  ▼
RN-M1  Foundation + feasibility spike
  ├──────────────┬──────────────┐
  ▼              ▼              ▼
RN-M2 UI       RN-M3 Engine   (M5 native spike evidence)
  │              │
  └──────┬───────┘
         ▼
      RN-M4 Tap/toolbar skeleton
         │
         ▼
      RN-M5 Native interactions + celebration
         │
         ▼
      RN-M6 Testing + accessibility + RC
         │
         ▼
      RN-M7 UAT + store preparation
         │
         ▼
      RN-M8 Production release
```

- **RN-M2 and RN-M3 may run in parallel** after RN-M1 if component contracts and domain types are agreed first.
- **RN-M5 must not start** until the RN-M1 spike has answered the drag/scroll and hardware-keyboard questions and approved audio assets exist.
- **RN-M7 store metadata work** may start during RN-M6, but submission depends on UAT sign-off.

---

## 3. Milestone charters

### RN-M0 — Baseline preservation and repository layout

**Objective:** Freeze and evidence the accepted web baseline, then move the web app into `web/` so the native app can live in `mobile/` without path ambiguity.

**Scope:**

- Create an annotated Git tag `web-baseline-2026-11-11` and a backup branch or archive; record checksums.
- Run the web manual acceptance checklist and capture evidence: browser/OS versions, viewport results, screenshots or recordings, console/network results. Record explicitly that the human gates were already accepted by PB-03 but evidence was outstanding (TST-05).
- Reconcile documentation drift: README staleness, `x` versus `y` examples, and the open-gate statements.
- Move `index.html` to `web/index.html` without changing its behavior or content.
- Update the serve command to `python3 -m http.server 8000 --directory web`, README, and `CLAUDE.md`.
- Create `docs/specs/react-native/milestones/`, `tasks/react-native/`, and `docs/specs/react-native/evidence/`.
- Record ADR: repository layout (`web/` + `mobile/`), web lifecycle, and the PB-02/PB-10 interpretation.

**Non-goals:** No React Native project, no dependency installation, no web feature change, no visual redesign, no deletion of the old history.

**Deliverables:** Git tag and backup; baseline evidence record; moved web app; updated docs/commands; ADR.

**Acceptance criteria:**

- [ ] The annotated tag and backup branch/archive exist and are verified.
- [ ] `web/index.html` is byte-identical to the tagged baseline.
- [ ] The web app runs from `python3 -m http.server 8000 --directory web` with no console errors, failed assets, or network requests.
- [ ] README and `CLAUDE.md` reference the new path and no longer instruct users to serve from the repository root.
- [ ] Baseline evidence is recorded and linked from the milestone evidence file.
- [ ] The PB-02/PB-10 interpretation and repository layout are accepted in the ADR.

**Entry decisions:** Approve the move; resolve PB-02/PB-10 in writing; name the baseline evidence owner (Boris, TST-05).

**Spec must decide:** exact backup mechanism (tag + branch vs archive), whether `docs/` and `tasks/` stay at the root, and whether the web app gets a `web/README.md`.

**Risks:** Broken deployment path or external links; stale documentation; scope creep into web changes.

**Gate:** M0 exit review. If the move is rejected, M0 reduces to backup + baseline evidence and `mobile/` is added alongside the root `index.html`.

---

### RN-M1 — Project foundation and feasibility spike

**Objective:** Prove the assumptions that can invalidate the architecture, then create the production Expo skeleton and quality gates.

This milestone combines migration plan Phase 2 (spike) and Phase 3 (foundation). The spike runs first; production scaffolding begins only after the spike gate.

**Spike scope (time-boxed, throwaway):**

- Hermes `BigInt` and exact-rational arithmetic on iOS and Android.
- A tokenizer/parser slice to confirm the TypeScript port shape.
- Immediate pan on touch-move versus `ScrollView`: measure token drag, drop targeting, and scroll arbitration; test a low-end Android emulator and an iPad simulator.
- `expo-audio` playback with bundled test audio; confirm iOS silent-switch muting and Android media-volume behavior.
- `AccessibilityInfo.announceForAccessibility`, focus behavior, `accessibilityActions`, and duplication with live regions.
- External hardware keyboard: Tab/Shift+Tab, Enter/Space, visible focus on both platforms; define the testable acceptance bar.
- Single-route Android back interception (`BackHandler`) while a session is active.
- New Architecture + Reanimated 4 + Gesture Handler compatibility.

**Foundation scope (production):**

- `mobile/` Expo app created with the current stable SDK; TypeScript strict; Expo Router with `src/app`.
- App identity from DEP-02: name `Equation Explorer`, bundle/package `net.bbsteps.equation`, scheme `equation`.
- New Architecture enabled; single light theme; no committed `ios/`/`android/`; `.gitignore` and prebuild-on-demand workflow documented.
- `app.config.ts`, splash/icon placeholders, status bar, safe-area and gesture root providers, root error boundary.
- ESLint/Prettier, `jest-expo` + RNTL, Maestro skeleton, CI workflow, `expo-doctor`.
- Theme tokens and shared primitives (`AppText`, `Button`, `Card`, `Screen`, `Icon`, `IconButton`).
- Placeholder route that renders a simple accessible screen.
- Run/build/test commands documented; exact dependency versions recorded.
- No-network invariant: no API client, auth, storage, analytics, or crash SDK.

**Non-goals:** Real Setup/Solver screens, engine implementation, production drag, final audio assets, brand assets.

**Deliverables:** Spike results and go/no-go; buildable iOS/Android dev app; CI pipeline; dependency pins; ADRs; run/build docs.

**Acceptance criteria:**

- [ ] Every spike question has recorded evidence: device/OS, observed result, pass/fail, and fallback.
- [ ] Spike gate A is signed. If drag or hardware keyboard fails, the architecture or acceptance bar is revised before foundation tasks continue.
- [ ] The app launches on an iOS simulator and an Android emulator from local development builds.
- [ ] `tsc --noEmit`, lint, and Jest pass; CI is green.
- [ ] `npx expo-doctor` passes; no `ios/` or `android/` directory is tracked.
- [ ] The placeholder screen exposes an accessible heading and action; theme tokens and primitives are documented.
- [ ] The CNG/local-build workflow and signing prerequisites are documented and reproducible.

**Entry decisions:** CI provider (F07); SDK/version policy (F12); Xcode/Android SDK/signing availability (F18); device/simulator access; spike acceptance criteria; drag and keyboard fallback policy (F02, F03).

**Spec must decide:** whether the spike lives in a separate throwaway app or a `spikes/` branch; the exact acceptance wording for hardware keyboard; the minimum Android test device; and the version-pinning policy.

**Risks:** Drag/scroll conflict unsolved; hardware keyboard gap; toolchain/signing delays; Expo/New Architecture version mismatch; spike scope creep.

**Gate:** Spike gate A, then foundation review. Checkpoint A from the migration plan is satisfied here.

---

### RN-M2 — Native UI architecture and static screens

**Objective:** Build the complete native UI with realistic fixture data and every visual state, tablet-first with a phone fallback, before real logic is connected.

**Scope:**

- Setup screen: brand header, equation input, Start, tip card, validation message.
- Solver screen shell: header/reset, locked history, active equation card, side selector, token sides, token chips, palette, toolbar, Check, feedback/toast, completion panel.
- Tablet-first composition with a phone layout; safe-area handling; keyboard avoidance.
- Design tokens from the web CSS intent: colors, spacing, radii, typography, shadows/elevation, motion durations.
- Accessibility contracts: roles, labels, states, hints, focus order skeleton, 44/48 dp targets, dynamic type, reduced motion.
- Static fixtures covering landing, active, rejected, accepted, completed, empty side, disabled toolbar, and long-equation states.
- Component inventory and RNTL component tests for labels, roles, states, and disabled logic.

**Non-goals:** Real parsing or validation, reducer state transitions, drag gestures, audio, haptics, confetti, persistence.

**Deliverables:** Tappable static UI on both platforms; component inventory; screenshot review; RNTL component tests; design sign-off.

**Acceptance criteria:**

- [ ] Every screen and state renders at 320–430 dp phone widths and 768–1024+ dp tablet widths.
- [ ] No horizontal overflow, clipping, or overlap; equations wrap inside their side.
- [ ] 200% text (or the approved threshold) remains usable; token chips and completion content wrap.
- [ ] All interactive targets meet 44×44 pt / 48×48 dp; focus/pressed/disabled/selected/error states are visible and not color-only.
- [ ] The palette is reachable and does not cover the active row or feedback in either layout.
- [ ] Keyboard avoidance works on Setup; Return submits.
- [ ] RNTL asserts accessible names, roles, states, and disabled logic.
- [ ] Design review approves the native redesign against the web's visual intent (UX-01), not pixel parity.

**Entry decisions:** Orientation (F05); dynamic-type threshold (F06); icon strategy and the `expo-font` question (F14); design approval owner.

**Spec must decide:** tablet and phone breakpoints; palette docking (side vs bottom); whether the solver scrolls as a whole; the component prop contracts that RN-M3 and RN-M4 will use.

**Risks:** Two form-factor layouts; dynamic type; palette ergonomics; gradient/shadow platform differences; design churn.

**Gate:** Design review. Freeze the component contracts before RN-M4 integration.

---

### RN-M3 — Exact math engine

**Objective:** Port the web's exact-rational equation engine into a pure TypeScript module with the complete regression corpus.

**Scope:**

- `features/equation/domain/`: rational arithmetic, tokenizer, parser/AST, affine reduction, direct-reciprocal source analysis, source analysis/solve, candidate analysis/final-form predicate, formatting, result types.
- Port `MATH_MESSAGES` verbatim into one message catalog.
- Port the self-check corpus into table-driven Jest tests. The current corpus has **62 check invocations**; port every one and record the actual count. Also add the product spec's release examples.
- Preserve limits: `MAX_INPUT_LENGTH = 512`, `MAX_TOKEN_COUNT = 160`.
- Preserve the narrow direct-reciprocal exception; do not generalize rational equations.
- Add property-style tests for rational normalization and inverse operations if inexpensive.
- No React, React Native, Expo, DOM, clock, randomness, locale, or I/O imports.

**Non-goals:** Session reducer, workspace helpers, UI components, validation messages in the UI, persistence.

**Deliverables:** Pure engine module; golden test suite; coverage report; engine API documentation.

**Acceptance criteria:**

- [ ] Every existing self-check case passes; the test report records the actual count and explains the 62-versus-70 planning correction.
- [ ] All product-spec release examples pass.
- [ ] Engine branch coverage is at least 90%; the golden corpus is a required CI gate.
- [ ] `y = 11 ÷ 2` formatting, `1y`/`(y)` non-final behavior, and the reciprocal exception match the web.
- [ ] Invalid, nonlinear, unsupported-rational, identity, contradiction, and over-limit inputs return typed reasons, not exceptions.
- [ ] No `eval`, `Function`, floating-point equation math, React, or React Native import appears in the domain.
- [ ] The engine can be tested without a renderer.

**Entry decisions:** Name an engine reviewer; accept the corpus-count correction (F13).

**Spec must decide:** the exact public function signatures and result unions; whether the tokenizer exposes source offsets; how display formatting and token glyphs are represented.

**Risks:** Subtle parser/formatting differences; BigInt serialization mistakes; over-generalizing the reciprocal exception.

**Gate:** Engine review with the golden suite green. This is the highest-value regression asset; do not proceed on a partially green corpus.

---

### RN-M4 — Session state and tap/toolbar walking skeleton

**Objective:** Deliver the first end-to-end tap-only solving experience: setup → solve → complete. This is internal milestone RN-1 from the migration plan, not a releasable build.

**Scope:**

- Session types, workspace token/row helpers, reducer, actions, selectors, signature/duplicate rules, engine bridge, session context.
- Setup validation and source acceptance.
- Active row editing: palette append, token selection, move left/right, move to the other side, delete, clear side, flip sides, restore previous step.
- Check → reject, lock, or complete; immutable locked history; duplicate 500 ms guard; same-as-previous and source-step rejection.
- Completion panel with verified answer and "Solve another!"; reset.
- Android back preserves the in-memory session; no persistence.
- Feedback messages and accessible announcements.
- Reducer unit tests, integration tests, and a Maestro tap-only flow.

**Non-goals:** Drag, trash drop, audio, haptics, confetti, final icon/splash, store metadata.

**Deliverables:** Tap-only working app; reducer/selector tests; integration tests; Maestro flow; demo evidence.

**Acceptance criteria:**

- [ ] `2y - 3 = 8` → `2y = 11` → `y = 11 ÷ 2` completes with the correct answer.
- [ ] Malformed, inequivalent, same-as-previous, source-return, and unsupported rows are rejected with the correct message and affected side.
- [ ] Locked history is append-only and immutable; only the active row is editable.
- [ ] Palette taps append to the selected side; toolbar disabled states match selection bounds.
- [ ] Reset clears the session; Android back does not discard it.
- [ ] Duplicate rapid Check produces one lock.
- [ ] Reducer branch coverage is at least 85%; integration tests exercise setup → accepted → completed and the rejection paths.
- [ ] No drag, audio, or persistence code is required for this milestone.
- [ ] The app is explicitly labelled internally as not releasable because drag is required (PB-04).

**Entry decisions:** RN-M2 component contracts frozen; RN-M3 engine green.

**Spec must decide:** reducer action set and payloads; whether drag will reuse the same workspace helpers; focus/announcement behavior on lock and completion.

**Risks:** Reducer transition bugs; history mutation; signature mismatches; focus behavior differences.

**Gate:** Gate B. Tap-only solving works end-to-end and the golden engine suite is green.

---

### RN-M5 — Native interaction layer and celebration

**Objective:** Add the native-only capabilities that make the app releasable: drag, audio, haptics, confetti, accessibility focus, icon/splash, and back behavior.

**Scope:**

- Token drag with immediate pan on touch-move (UX-06): Gesture Handler + Reanimated shared values, measured token/side rects, drag overlay, insertion index, side-level highlight, trash drop.
- Drag/scroll arbitration chosen in the RN-M1 spike; no per-frame React renders.
- Tap/toolbar path remains equally complete; `accessibilityActions` on selected tokens; hardware-keyboard path verified.
- Bundled success/error chimes via `expo-audio`; silent-switch behavior per NAT-10; failure-tolerant playback.
- Optional haptics via `expo-haptics`; failure-tolerant.
- Confetti overlay with reduced-motion skip; shakin/pop feedback.
- Accessibility announcements and focus moves for invalid setup, rejected check, accepted row, completion, and reset.
- App icon, adaptive icon, splash; final or approved placeholder assets.
- Android back interception preserving the session; no persistence.

**Non-goals:** New screens, new math rules, analytics, crash reporting, OTA updates, mute/haptics settings unless NAT-04 is approved.

**Deliverables:** Native interaction layer; bundled audio; final/placeholder assets; device recordings; capability tests.

**Acceptance criteria:**

- [ ] Drag reorders, moves across sides, and deletes via trash on iOS and a low-end Android device/emulator.
- [ ] Scrolling remains usable wherever the design requires it; the chosen arbitration is documented and tested.
- [ ] Tap/toolbar editing performs every drag operation; drag is never the only path.
- [ ] Required success/error sound plays; the iOS silent switch mutes it; Android follows media volume; audio failure never blocks completion.
- [ ] Completion shows "Good job", keeps the answer visible, and offers "Solve another!" (UX-07).
- [ ] Confetti and shake are skipped or reduced under reduced motion; success/error meaning remains.
- [ ] VoiceOver/TalkBack announcements and focus moves match the contract; no duplicate announcements from the toast.
- [ ] Hardware keyboard focus/activation works on the approved platform bar.
- [ ] No permission prompt appears; all assets load offline.
- [ ] Haptics are no-ops on unsupported devices and never block.

**Entry decisions:** Approved audio assets and licensing (F04); final or approved brand placeholders (F08); drag fallback and keyboard acceptance (F02, F03); NAT-04 mute/haptics control decision.

**Spec must decide:** drag activation/arbitration details; drop-index rule; audio session configuration; confetti piece count and cleanup; whether haptics ship enabled.

**Risks:** Drag/scroll conflict; gesture feel; audio silent switch; accessibility focus reliability; emoji/icon differences; low-end performance.

**Gate:** Gate C. Drag, audio, haptics, confetti, and accessibility focus are integrated and usable.

---

### RN-M6 — Testing, accessibility, and release-candidate hardening

**Objective:** Make the app release-candidate quality on both platforms.

**Scope:**

- Complete unit, component, and integration coverage; full Maestro suite on iOS simulator and Android emulator.
- VoiceOver and TalkBack manual passes; dynamic type; reduced motion; contrast audit; touch-target audit.
- Device matrix: iPad, Android tablet, small phone, current phone, low-end Android; orientation and safe areas.
- Performance/startup checks; no per-frame React renders; cleanup of timers/animations.
- Airplane-mode/offline verification; no-network invariant test; no persistence confirmation.
- Bug fixes; CI hardening; known-issues list; release-candidate builds.

**Non-goals:** New features, store submission, UAT sessions.

**Deliverables:** Test report; accessibility report; device matrix evidence; known-issues list; RC builds.

**Acceptance criteria:**

- [ ] All critical flows pass on iOS and Android.
- [ ] No blocker accessibility defects; VoiceOver and TalkBack traversal is recorded.
- [ ] Golden engine and reducer suites are green in CI.
- [ ] No console errors; no startup network request; app works in airplane mode after install.
- [ ] Performance is acceptable on the low-end Android target.
- [ ] The RC passes the approved dynamic-type and reduced-motion checks.
- [ ] Known issues are triaged with severity and owners.

**Entry decisions:** Accessibility standard (NAT-01); platforms/OS versions for screen-reader sign-off (NAT-02); sign-off owner (TST-02); physical-device availability (TST-04); device matrix availability (TST-03).

**Spec must decide:** the exact a11y checklist and pass/fail bar; which Maestro flows are PR versus release gates; the performance baseline.

**Risks:** Accessibility gaps found late; gesture feel; platform differences; device availability.

**Gate:** Gate D. Release candidate passes the device and accessibility matrix.

---

### RN-M7 — UAT and store preparation

**Objective:** Validate the app with real learners/adults and prepare compliant store submissions.

**Scope:**

- Scripted UAT sessions covering setup, tap solving, drag solving, error comprehension, completion, and reset.
- Observe children with and without a parent; collect comprehension feedback on messages.
- Triage and fix release blockers; classify all other feedback.
- Store metadata: descriptions, keywords, screenshots, category (Education, not Kids-only), age rating from the questionnaire, privacy policy URL, support URL, data-safety answers ("Data Not Collected").
- Final icon/splash; signing credentials; local Xcode archive and Play Console upload; TestFlight/Internal App Sharing builds; reviewer notes.
- Verify identity `net.bbsteps.equation` is used consistently.

**Non-goals:** Production rollout; feature expansion from UAT unless it fixes a release blocker.

**Deliverables:** UAT sign-off; prioritized defect list; store listings; signed builds; beta builds; privacy documentation.

**Acceptance criteria:**

- [ ] UAT scenarios pass; no severity-1/2 defects remain; product owner signs off.
- [ ] Metadata is complete and consistent with actual behavior (no data collection, no ads, no accounts).
- [ ] Builds are accepted by TestFlight and Play Console internal testing.
- [ ] No policy blockers remain; reviewer test notes are prepared.
- [ ] The privacy policy and support URL are live.

**Entry decisions:** UAT participants and recruitment (TST-01); beta testers (TST-06); privacy policy owner (SEC-04); category/rating/data-safety (DEP-07); support owner (DEP-08); final brand assets (NAT-06).

**Spec must decide:** UAT script and success criteria; defect severity definitions; store screenshot set and messaging.

**Risks:** Child usability issues not visible in QA; scheduling; scope changes from feedback; store review rejection; signing/account delays.

**Gate:** Gate E. UAT sign-off and store-ready builds.

---

### RN-M8 — Production release

**Objective:** Release safely and support the first production cohort.

**Scope:**

- Phased rollout on both stores.
- Store review response and any required metadata changes.
- Hotfix plan through store review (no OTA service; DEP-03).
- Post-release regression; support/feedback process; handover documentation.
- No analytics or crash-reporting SDK; field issues are collected from user reports and reproduced locally.

**Non-goals:** New features, OTA infrastructure, analytics, accounts, monetization.

**Deliverables:** Production release; release notes; support/feedback runbook; handover docs.

**Acceptance criteria:**

- [ ] Release is live on both stores.
- [ ] No P1 defects are reported during phased rollout.
- [ ] The support and hotfix process is defined and owned.
- [ ] The maintenance owner and upgrade cadence are recorded.

**Entry decisions:** Release cadence and rollout percentage (DEP-06); maintenance/upgrade owner (DEP-10); maintenance budget (DEP-11).

**Risks:** Store rejection/delay; platform-specific production bugs; slow diagnosis without crash reporting; slow fixes without OTA.

**Gate:** Gate F. Production release.

---

## 4. Cross-cutting requirements

These apply to every milestone and must appear in each spec's Boundaries section.

**Always:**

- Follow the architecture boundaries: `features/equation/domain` imports no React/React Native/Expo; no feature imports Expo modules directly; use `shared/native/*` façades.
- Use exact rational arithmetic for equation values; no floating-point equation math.
- Use project code for parsing/validation; never `eval` or `Function`.
- Keep the web app unchanged in behavior; native work stays in `mobile/`.
- Maintain 44/48 dp targets, visible focus/pressed states, dynamic type, reduced motion, and accessible names/roles/states.
- Run typecheck, lint, and tests before declaring a task complete.
- Record evidence for manual and device checks.

**Ask first:**

- Adding a runtime dependency beyond the approved dependency register.
- Committing `ios/` or `android/`, or changing the CNG/local-build model.
- Changing app identity, bundle/package, scheme, or store category.
- Adding persistence, network access, analytics, crash reporting, accounts, or monetization.
- Changing the public math scope or adding a new notation rule.
- Editing the web app or moving files outside `mobile/` after RN-M0.

**Never:**

- Use WebView, DOM components, React Native Web, or embedded web content for app UI.
- Persist `BigInt` with `JSON.stringify` or store learner data off-device.
- Log raw equation input, tokens, or session state in production.
- Make drag the only way to perform an operation.
- Use color alone to convey error, selection, or success.
- Block completion when audio, haptics, or an announcement fails.
- Remove or weaken a failing golden test to make a build pass.

---

## 5. Decision schedule

| Decision | Needed by | Owner | Recommendation / note |
| --- | --- | --- | --- |
| Move web to `web/`; repository layout | RN-M0 entry | Product + Eng | Approve the move; record an ADR. If rejected, keep root and add `mobile/` alongside. |
| PB-02 / PB-10 interpretation | RN-M0 exit | Product | "Native is the intended successor; web stays live and maintained until a separate retirement decision." |
| Web baseline evidence | RN-M0 exit | Boris (TST-05) | Run the checklist; capture browser/OS, viewports, screenshots, console/network. |
| CI provider | RN-M1 entry | Eng | GitHub Actions for typecheck/lint/Jest; schedule heavier Maestro/native runs. |
| SDK/version policy | RN-M1 entry | Eng | Current stable Expo SDK; `npx expo install`; commit lockfile; `expo-doctor`; record exact versions. |
| Xcode/Android SDK/signing | RN-M1 entry | Boris | Verify toolchain and credentials before scaffolding. |
| Spike acceptance criteria | RN-M1 entry | Eng + QA | Evidence per assumption; explicit fallback for drag and keyboard. |
| Drag/scroll arbitration | RN-M1 exit | Eng + Design | Token-only pan, direction lock, or non-scrolling token surface; prove on low-end Android. |
| Hardware-keyboard acceptance bar | RN-M1 exit | Product + QA | Platform-specific: iOS Full Keyboard Access and Android hardware focus; exact Tab order only if proven. |
| Orientation | RN-M2 entry | Product + Design | Recommend tablet both orientations; phone portrait plus landscape if the layout passes; record exclusions. |
| Dynamic-type threshold | RN-M2 entry | Design + QA | Recommend iOS 200% / Android max scale with wrapping. |
| Icon strategy / `expo-font` | RN-M2 entry | Design + Eng | Accept `expo-font` transitively for `@expo/vector-icons`, or use text/Unicode icons. |
| Engine reviewer | RN-M3 entry | Eng | Assign before the port starts. |
| Audio assets and licensing | RN-M5 entry | Product + Design | Hard gate: approved, project-owned chimes. |
| Brand icon/splash | RN-M5 / RN-M7 | Design | Placeholders are acceptable in M2; final assets before store submission. |
| NAT-01/NAT-02 accessibility bar | RN-M6 entry | QA + Product | WCAG 2.2 AA equivalent; VoiceOver and TalkBack on the supported OS floor. |
| A11y sign-off and UAT owners | RN-M6 / RN-M7 | Product + QA | Name owners before the gate, not after. |
| Privacy policy, store category/rating, data-safety | RN-M7 entry | Privacy/Legal + Product | Education listing; "Data Not Collected"; rating from the questionnaire. |
| Release cadence, support, maintenance | RN-M8 entry | Product + Release/Ops | Define rollout percentage, hotfix path, and post-launch owner. |

---

## 6. Sequencing and effort

| Milestone | Effort (pd) | Cumulative (pd) |
| --- | ---: | ---: |
| RN-M0 Baseline + repo layout | 1–2 | 1–2 |
| RN-M1 Foundation + spike | 6–10 | 7–12 |
| RN-M2 UI architecture | 6–9 | 13–21 |
| RN-M3 Math engine | 4–6 | 17–27 |
| RN-M4 Tap/toolbar skeleton | 4–6 | 21–33 |
| RN-M5 Native interactions | 5–8 | 26–41 |
| RN-M6 Testing + a11y + RC | 4–7 | 30–48 |
| RN-M7 UAT + store prep | 6–10 | 36–58 |
| RN-M8 Production release | 2–3 | 38–61 |
| **Contingency (15–20%)** | 6–12 | **44–73** |

**Critical path:** RN-M1 → RN-M5 → RN-M6 → RN-M7. RN-M2 and RN-M3 can run in parallel after RN-M1; RN-M4 needs both.

**Checkpoints (migration plan mapping):**

- **A — after RN-M1 spike:** BigInt, drag feasibility, audio, accessibility, both-platform builds; go/no-go.
- **B — after RN-M4:** Tap/toolbar solving works end-to-end with the full golden corpus green. Not releasable.
- **C — after RN-M5:** Drag, audio, haptics, confetti, and accessibility focus are integrated.
- **D — after RN-M6:** RC passes the device and accessibility matrix.
- **E — after RN-M7:** UAT sign-off and store-ready builds.
- **F — after RN-M8:** Production release.

---

## 7. Recommended first actions

1. Approve this roadmap and the [implementability review](../../react-native-implementability-review.md).
2. Run spec-driven development for **RN-M0**: produce `docs/specs/react-native/milestones/RN-M0-baseline-and-repo-layout.md`, then `tasks/react-native/RN-M0-plan.md` and `tasks/react-native/RN-M0-todo.md`.
3. Execute RN-M0: tag, back up, capture baseline evidence, move the web app, update commands/docs, and record the layout ADR.
4. Only after the M0 gate, run spec-driven development for **RN-M1** (foundation + spike). Do not scaffold the production app before the M0 layout is approved.
5. Treat the spike gate in RN-M1 as a hard stop: if drag or hardware keyboard fails, revise the M5 acceptance bar and the architecture before writing UI or gesture code.