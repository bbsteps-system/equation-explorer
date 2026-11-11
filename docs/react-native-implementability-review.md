# React Native Migration — Implementability Review

**Status:** Architect review complete. No React Native project or production code has been created.
**Date:** 2026-11-10
**Scope:** Cross-check of `docs/react-native-migration-plan.md` and `docs/react-native-architecture-plan.md` against the current repository, `index.html`, the product spec, and `docs/react-native-requirements-open-questions.md`.
**Verdict:** **Conditionally implementable.** Milestones M0–M4 are safe to start once M0 and M1 entry decisions are approved. M5 (native interactions) is gated by a feasibility spike and four unresolved inputs. No finding requires a WebView, a backend, or a different architecture.

---

## 1. Summary

The migration plan is technically coherent and appropriately scoped:

- The domain engine is pure logic and can be ported to TypeScript without React Native.
- The UI is modest in surface area (two screens and a token workspace) and fits Expo managed/CNG.
- There is no backend, authentication, persistence, analytics, or third-party runtime dependency.
- Jest + RNTL + Maestro is proportionate to the app.
- Tablet-first with phone support is implementable with `useWindowDimensions`, safe-area insets, and a wrapping token layout.

The plan is not yet **unconditionally** implementable. Four findings can change the architecture or the release scope:

1. **Repository move conflict.** The plan says the web app stays at the repository root (D13), while the requested first step moves it to `web/`. This is a reversible repository change, but it changes the serve/deploy commands and must be recorded as an ADR.
2. **Immediate drag versus scrolling.** Native has no `elementFromPoint`/pointer capture, and the chosen "immediate drag on touch-move" gesture competes with `ScrollView`. This needs the Phase 2 spike before M5 is estimated or promised.
3. **Hardware-keyboard accessibility.** NAT-03 requires external keyboard focus order and activation. React Native does not provide a first-class cross-platform tab order; iOS Full Keyboard Access and Android hardware focus must be proven. If the acceptance bar means an exact web-like Tab order, this is a launch risk.
4. **Required audio assets do not exist.** PB-06 makes sound a first-release requirement, but the repository contains no audio files and ownership/licensing is open (NAT-04). M5 cannot exit without approved assets.

The remaining findings are decision gates or documentation corrections, not architectural blockers.

---

## 2. Findings

Severity: **High** = can invalidate scope, architecture, or release; **Medium** = must be decided before the named milestone; **Low** = documentation, tooling, or planning correction.

| ID | Severity | Finding | Impact if unresolved | Resolution / milestone |
| --- | --- | --- | --- | --- |
| F01 | High | **Repository layout conflict.** Migration plan D13 says the web app is left unchanged at the root; the requested M0 moves it to `web/`. PB-10 also requires the web app to stay live and maintained in parallel. | Moving the file breaks `python3 -m http.server 8000 --directory .`, static-hosting root assumptions, README/CLAUDE commands, and any external link to `index.html`. | M0: tag and back up the baseline, move to `web/index.html`, update commands/docs, and record an ADR. Keep the web app behavior byte-identical. |
| F02 | High | **Immediate drag versus scroll.** UX-06 requires drag on touch-move, but the token sides wrap and the solver may need vertical scrolling. Gesture Handler `Pan` will compete with `ScrollView`. | Drag may feel unreliable or scrolling may be impossible on phones; PB-04 makes drag release-gating. | M1 spike: prove on iOS and low-end Android; decide the scroll/drag arbitration (token-only pan, direction lock, non-scrolling sides, or a scroll surface outside token areas). M5 implements the chosen approach. |
| F03 | High | **Hardware keyboard on mobile.** NAT-03 requires both screen-reader actions and external hardware-keyboard focus/activation. RN has no guaranteed cross-platform Tab order; iOS relies on Full Keyboard Access and Android on focusable views. | An exact web-like Tab order may be unachievable, which would fail the stated accessibility bar. | M1 spike: define the acceptance bar in testable platform terms; verify Tab/Shift+Tab, Enter/Space, and visible focus on both platforms. If exact order is impossible, record the platform limitation and agreed fallback in the M5 spec. |
| F04 | High | **Required audio assets are missing.** The web synthesizes tones; native must bundle chimes (ADR-007), and sound is required (PB-06). No audio files or license terms exist in the repo. | M5 cannot meet acceptance criteria; a last-minute asset problem blocks the release. | M5 entry gate: approved, project-owned `success` and `error` (and optional tap) audio files. M1 spike proves playback and silent-switch behavior only. |
| F05 | Medium | **Orientation is undecided.** UX-02 decided tablet-primary and phone support, not portrait versus landscape. | The M2 layout spec cannot define breakpoints, available height, or test orientations. | M2 entry decision. Recommendation: support both orientations on tablets; support phone portrait and add landscape if the layout passes. Record any excluded orientation. |
| F06 | Medium | **Dynamic-type threshold is undecided.** UX-10 has no maximum font scale. | M2/M6 cannot pass or fail "large text" consistently; token chips may clip. | M2 entry decision. Recommendation: iOS 200% and Android maximum font scale, with wrapping and no clipped glyphs; cap only tiny non-content labels if necessary. |
| F07 | Medium | **CI provider is undecided.** DEP-04 is open, while the migration plan requires CI gates. | M1 acceptance ("CI passing") has no target. | M1 entry decision. Recommendation: GitHub Actions for typecheck/lint/Jest; keep Maestro and local native builds on a release schedule if macOS-runner cost is a concern. |
| F08 | Medium | **Brand assets are undecided.** App icon, adaptive icon, splash, and store screenshots (NAT-06) do not exist. | M5 uses placeholders; M7/store submission blocks. | M2 uses approved placeholders; M5 and M7 require final assets. Identity values themselves are settled (DEP-02). |
| F09 | Medium | **Store, privacy, and UAT ownership is undecided.** SEC-04, DEP-07, TST-01/02/06 remain open. | M6–M7 gates cannot be signed; submission cannot complete. | M6 entry: name the accessibility sign-off owner and UAT participants. M7 entry: privacy policy, data-safety answers, category/rating, and support URL. |
| F10 | Medium | **PB-02 versus PB-10 tension.** PB-02 says native replaces the web; PB-10 says the web is maintained in parallel. | The move, parity obligation, and maintenance ownership are ambiguous. | M0 exit: written resolution, e.g. "native is the intended successor; the web stays live and maintained until a separate retirement decision." Record it in the ADR. |
| F11 | Medium | **CNG plus local builds needs an explicit workflow.** "No committed `ios/`/`android/`" and "local Xcode/Gradle builds" are compatible, but only via `npx expo prebuild`/`npx expo run:*` and a documented gitignore policy. | Contributors may commit generated native projects or produce non-reproducible builds. | M1: document `expo run:ios` / `expo run:android`, prebuild-on-demand, lockfile, Node version, and ignored native directories. Verify with `npx expo-doctor`. |
| F12 | Medium | **Dependency versions are unspecified.** The plans say "current stable Expo SDK" without pinning RN, React, Reanimated, Gesture Handler, or expo-audio. | New Architecture/Reanimated 4 compatibility surprises; different developers get different trees. | M1: create with the current stable Expo SDK, install Expo-managed packages with `npx expo install`, commit the lockfile, and record exact versions in the M1 spec/evidence. |
| F13 | Low | **The self-check corpus is 62 cases, not 70.** `runMathSelfChecks()` (`index.html:1105-1292`) contains 62 `check`/helper invocation sites; each helper records exactly one case. The migration plan and TST-07 say 70. | Acceptance would cite a number that cannot be reproduced; a reviewer may think cases were dropped. | M3: port every existing case, record the actual count in the test report, and note the planning correction. The TST-07 intent (keep all cases) is unchanged. |
| F14 | Low | **`@expo/vector-icons` pulls `expo-font`.** The dependency register excludes `expo-font` ("system fonts"), but the recommended icon package depends on it transitively. | Contradictory dependency policy; avoidable review friction. | M2: either accept `expo-font` as a transitive dependency of `@expo/vector-icons` and document it, or use text/Unicode icons for essential actions. Decide in the theme/icon strategy. |
| F15 | Low | **Back interception mechanism is underspecified.** The migration plan mentions Expo Router `usePreventRemove`; with one visible route there may be no navigation action to prevent. | Android back could exit or discard the session instead of preserving it (UX-08). | M1 spike/M5: use `BackHandler` (`hardwareBackPress`) for the single-route case, returning `true` while a session is active; verify on Android. |
| F16 | Low | **Phase 6 (API/auth) has no milestone home.** It is N/A but must not silently disappear. | A future contributor might add a network layer, or the "no network" invariant may go untested. | M1 and M6: add a no-network invariant test (no `fetch`/`axios` imports, no startup request) and an airplane-mode check. |
| F17 | Low | **Web baseline evidence is not captured, and `x`/`y` drift remains.** PB-03 accepts the baseline but TST-05 evidence is outstanding; README/spec examples still use `x`. | Parity disputes have no reference; docs contradict the app. | M0: run the web manual checklist, record screenshots/browser details, and correct README/spec examples to `y`. |
| F18 | Low | **Local build toolchain and signing provisioning are unconfirmed.** DEP-01 assigns accounts to Boris, but the machine's Xcode/Android SDK/signing state is unknown. | M1/M8 can stall at the first native build or archive. | M1 entry: verify Xcode, Android SDK/JDK, simulators/emulators, and signing credentials. M7 repeats for release signing. |
| F19 | Medium | **Native capability behavior needs device evidence.** `expo-audio` silent-switch behavior, accessibility focus under the New Architecture, and announcement duplication differ by OS version. | Unit tests and simulators do not prove PB-06, NAT-03, or NAT-10. | M1 spike on physical/simulated devices; M5 integrates; M6 manual matrix. |
| F20 | Low | **No OTA and no crash reporting imply a slow-fix runbook.** AN-01/AN-02 and DEP-03 are accepted, but no support process exists. | Production defects are diagnosed only from user reports and fixed through store review. | M8: define the support/feedback, hotfix, and release runbook; record it in handover docs. |

---

## 3. Implementability by migration phase

| Migration phase | Implementable now? | Condition |
| --- | --- | --- |
| 1. Discovery and requirements | Yes | M0 closes the documentation and evidence gaps. |
| 2. Architecture and PoC | Yes | M1 runs the spike; drag and hardware keyboard are the go/no-go items. |
| 3. Project foundation | Yes | M1 also records the CNG/local-build workflow and pins versions. |
| 4. Static UI | Yes | M2 needs orientation, dynamic-type, and icon-strategy decisions. |
| 5. Logic and validation | Yes | M3/M4 are the lowest-risk, highest-value work; correct the corpus count. |
| 6. API/auth (N/A) | Yes | Fold into M1/M6 as a no-network invariant. |
| 7. Storage/native capabilities | Conditionally | M5 is gated by F02, F03, F04, and F19. |
| 8. Testing and accessibility | Conditionally | M6 needs F09 and F19 resolved. |
| 9. UAT | Conditionally | M7 needs participants and a sign-off owner. |
| 10. Store preparation | Conditionally | M7 needs privacy/store decisions and final assets. |
| 11. Release | Yes, after M7 | M8 needs a release cadence and support owner. |

---

## 4. Recommended corrections to the source plans

Apply these when the owning milestone spec is written; do not silently rewrite a decision document.

1. **Corpus count:** replace "70 cases" with "all cases in the current `runMathSelfChecks()`; 62 invocation sites at the time of this review." Preserve the TST-07 intent.
2. **Repository layout:** add an ADR for `web/` + `mobile/`, update CLAUDE.md, README, and the serve/deploy commands, and record the PB-02/PB-10 resolution.
3. **Back navigation:** specify `BackHandler` for the single-route Android case; keep `usePreventRemove` only if a second route is added later.
4. **Dependency policy:** document that `@expo/vector-icons` brings `expo-font` transitively, or choose text icons.
5. **Local build workflow:** add the exact `expo run:ios`, `expo run:android`, prebuild, `expo-doctor`, lockfile, and gitignore commands.
6. **Milestone gates:** carry the M0–M8 decisions from this review into the roadmap and each milestone spec.
7. **No-network invariant:** add an explicit test and an airplane-mode acceptance check rather than leaving Phase 6 as a prose N/A.

---

## 5. Go/no-go conditions

**Go for M1** when:

- M0 has preserved the web baseline and the repository layout is approved.
- The web app still runs unchanged from its new location.
- The PB-02/PB-10 interpretation is recorded.
- The M1 spike acceptance criteria and the device/simulator access are agreed.

**Go for M2–M4** when:

- The M1 spike proves BigInt, immediate drag's feasibility, audio playback, accessibility announcements/focus, and both-platform dev builds.
- A drag/scroll fallback is chosen even if drag is not yet fully polished.
- The hardware-keyboard acceptance bar is stated in platform terms.

**Go for M5** when:

- Approved audio assets exist.
- Orientation, dynamic type, and brand placeholders are decided.
- The drag approach and keyboard fallback are documented.

**Go for M6–M8** when:

- The accessibility standard and sign-off owner are named.
- UAT participants and the physical-device matrix are available.
- Privacy policy, store category/rating, and data-safety answers are approved.

---

## 6. Evidence inspected

| Source | What was checked |
| --- | --- |
| `index.html` | Engine limits (`MAX_INPUT_LENGTH = 512`, `MAX_TOKEN_COUNT = 160`), 62 self-check invocations, palette, source/candidate rules, drag implementation, messages |
| `docs/react-native-migration-plan.md` | Phases, effort, risks, prerequisites, checkpoints, open questions |
| `docs/react-native-architecture-plan.md` | Folder structure, state model, domain boundaries, dependency register, ADRs |
| `docs/react-native-requirements-open-questions.md` | Answered and open product/platform/testing/delivery decisions |
| `docs/specs/linear-equation-drag-solver.md` | Product scope, interaction flow, accessibility requirements, release checklist |
| `docs/specs/milestones/00-04`, `tasks/00-04` | Existing milestone and task conventions, open human gates, stale-document evidence |
| Repository root | Current layout: `index.html` at root, no `mobile/`, no native projects, no package manager |