# React Native Requirements — Open Questions

**Status:** Partially answered. On **2026-11-10** the product confirmed PB-01–PB-10, that the app is a **standalone, fully offline app with no API and no authentication** (SEC-01/02, API-01/02, OFF-01/02), that **tablet is the primary target with phone support** (UX-02), a **single light theme** (UX-03), and **no analytics or crash reporting** (AN-01/AN-02). The implementation stack is fixed as **React Native + TypeScript + Expo modules + Development Builds + Maestro + Claude Code**. All other areas remain open. No other answer below is invented or assumed as a decision.
**Date:** 2026-11-09; answers recorded 2026-11-10
**Purpose:** Collect every unresolved product, UX, security, platform, testing, and delivery question that can change the React Native migration plan, with the impact of leaving it unanswered.
**Companions:** [`react-native-migration-plan.md`](./react-native-migration-plan.md), [`react-native-architecture-plan.md`](./react-native-architecture-plan.md).

---

## 0. How to use this document

### 0.1 Status values

| Status | Meaning |
| --- | --- |
| **Blocks start** | Must be answered before creating the production React Native project or freezing the architecture. |
| **Blocks launch** | Implementation can begin with a documented provisional choice, but the answer must exist before UAT/store submission/release. |
| **Deferred** | Can be decided later without invalidating the architecture; record an owner and revisit date. |
| **N/A today** | Not applicable to the current scope; listed so it is not silently missed if scope changes. |

### 0.2 Owner roles

Use roles, not names, until an owner is assigned: **Product**, **Design**, **Eng**, **QA**, **Privacy/Legal**, **Release/Ops**.

### 0.3 Rules

- An unanswered question does **not** imply a default answer. Where the plans use a provisional assumption to keep moving, it is listed in §2 and labelled as an assumption, not a decision.
- Change a question’s status only by recording an answer in the decision log (§6) and updating the affected documents.
- If an answer changes scope (for example, accounts or persistence), re-run the relevant migration-plan sections before implementation.
- §6 is the authoritative decision log. Where an answer appears inline in §3.1, §6 carries the date, owner, and affected documents.

---

## 1. Facts confirmed by the source (not open questions)

These are settled by inspecting the existing application. They are recorded here so reviewers do not reopen them without new evidence.

| Fact | Evidence |
| --- | --- |
| The product is a single-page, client-only equation-solving game for children. | `index.html:1-6`, `docs/specs/linear-equation-drag-solver.md:5-15` |
| The variable is `y`. | `index.html` workspace/palette; recent commit `e3f3af4 use y` |
| Scope is one-variable linear equations with a unique solution. | `docs/specs/linear-equation-drag-solver.md:17-24`; `index.html:1018-1098` |
| The only variable-denominator exception is `c / y = d` or `d = c / y` with nonzero constants. | `docs/specs/linear-equation-drag-solver.md:17-24`; `index.html:934-969` |
| Exact rational arithmetic is used; no floating-point equation math. | `index.html:629-700` |
| No backend, API, database, or external service exists. | Whole-file inspection of `index.html`; `README.md:19-20` |
| No authentication, accounts, tokens, cookies, or sessions exist. | Whole-file inspection of `index.html`; no auth code |
| No browser storage (`localStorage`, `sessionStorage`, IndexedDB) is used. | Whole-file inspection of `index.html` |
| No third-party JavaScript libraries, CDN, analytics, or fonts are used. | `index.html:1-8`, `:2292` |
| No images are used beyond a favicon data URI; no audio files are used. | `index.html:7`, `:1602-1664` |
| Sound is synthesized at runtime with the Web Audio API. | `index.html:1602-1664` |
| The app is responsive and uses Pointer Events for drag, mouse, and touch. | `index.html:424-461`, `:1884-2005` |
| Accepted notation is integers, terminating decimals, fractions with `÷` or `/`, parentheses, `+`, `-`, `×`/`*`, implicit multiplication, and `y`. | `docs/specs/linear-equation-drag-solver.md:17-24` |
| Each accepted step is locked into history; only the active row is editable. | `docs/specs/linear-equation-drag-solver.md:112-147`; `index.html:2087-2128` |
| The app has no server-state, loading-from-network, or API-failure states. | Whole-file inspection of `index.html` |
| The web project’s human acceptance gates are still recorded as open. | `docs/specs/linear-equation-drag-solver.md:3`; `tasks/04-todo.md` |
| The README is stale and describes an earlier UI-only milestone. | `README.md:5-8` |
| The repository currently contains only the web app plus docs/tasks; there is no native client. | Repository inspection |
| The current web behavior is accepted as the parity baseline for the native migration. | Decision PB-03, 2026-11-10, §6 |
| The native app is a standalone, fully offline app with no API and no authentication. | Product decision, 2026-11-10, §6 (SEC-01, SEC-02, API-01, OFF-01) |
| Tablet is the primary target; phones must also open and work. Orientation is not decided. | Product decision, 2026-11-10, §6 (UX-02) |
| The app uses a single light theme; no dark mode. | Design decision, 2026-11-10, §6 (UX-03) |

---

## 2. Provisional assumptions used by the plans

These are **planning assumptions**, not answers. If any is rejected, the named plan section changes.

**Status column added 2026-11-10:** *Confirmed* = an answer now matches the assumption; *Rejected* = an answer contradicts it; *Needs revision* = an answer changes the assumption but the replacement is not fully specified; *Open* = still an assumption without a product decision.

| ID | Assumption | Used in | Status | Impact if rejected |
| --- | --- | --- | --- | --- |
| A1 | The web behavior can be accepted as the parity baseline via the manual checklist, even though the human gates are still recorded open. | Migration plan §6, §10; architecture §1, §8 | **Confirmed** by PB-03 | Parity target undefined; add a baseline-acceptance phase before implementation |
| A2 | Tap/toolbar editing can ship before custom drag; drag is confirmed separately. | Migration plan RN-1; architecture ADR-005 | **Rejected** by PB-04 — drag is required in the first release | Drag enters the critical path immediately; Phase 2 PoC and Phase 7 estimates increase |
| A3 | In-memory sessions are acceptable for v1. | Architecture §11, ADR-006 | **Confirmed** by PB-05 | Persistence design, privacy review, and storage dependency required |
| A4 | The primary target is a phone in portrait; tablet/landscape are not required at launch. | Architecture §14, migration plan §6.8 | **Rejected** by UX-02 — tablet is the primary target and phones must also work | Additional tablet layouts and breakpoints; orientation still unstated |
| A5 | A single light theme is acceptable at launch. | Architecture §14, ADR-009 | **Confirmed** by UX-03 — single light theme, no dark mode | Dark-mode tokens and system-theme handling required |
| A6 | English-only launch is acceptable. | Architecture §15 | **Confirmed** by PB-08 | Localization architecture and copy assets required |
| A7 | No analytics or crash reporting is required at launch. | Migration plan §5.5; architecture §17 | **Confirmed** by AN-01/AN-02 — no analytics or reporting | Privacy disclosure, SDK selection, consent, and event design required |
| A8 | Expo managed/CNG is acceptable; no custom native code is expected. | Migration plan §4; architecture ADR-001 | **Confirmed** — React Native + TypeScript + Expo modules + development builds | Bare/prebuild workflow and native ownership required |
| A9 | The existing web app remains in the repository and is not modified by the migration. | Whole plan | **Needs revision** — PB-02 makes the native app the replacement; PB-10 keeps the web running in parallel | Define the eventual web retirement path and the parallel-maintenance window |
| A10 | A package manager/build toolchain is permitted for the React Native app (the web app’s no-build constraint does not apply to `mobile/`). | Architecture §22 | **Confirmed** — the chosen React Native + Expo modules + development-build toolchain requires npm/Expo tooling in `mobile/` | The project cannot be scaffolded as specified |

---

## 3. Open questions by area

### 3.1 Product and business

| ID | Question | Answer (2026-11-10) | Why it matters / impact | Status | Owner |
| --- | --- | --- | --- | --- | --- |
| PB-01 | What is the target age band and primary user: child alone, child with parent, or classroom/teacher? | **Child with parent, and child alone; ages 10–15 plus older learners new to equations.** Classroom/teacher is out of scope. | Drives copy, supervision model, accessibility level, and store positioning. Age band is set (SEC-03); the store rating is finalized at submission (DEP-07). | **Answered** (2026-11-10) | Product |
| PB-02 | Is the native app intended to replace the web app, complement it, or become a separate product line? | **The native app replaces the web app.** | Determines whether the web stays the reference, whether features must stay in parity, and whether the engine should be shared later. Replacement raises a retirement/migration path for existing web users and a parity obligation until retirement. | **Answered** | Product |
| PB-03 | Is the current web behavior formally accepted as the parity baseline? If not, which checklist must be run and by whom? | **Yes — current web behavior is accepted as the parity baseline.** | The spec and task list still record human acceptance gates as open. UAT now has a reference, but evidence still needs to be captured (TST-05). | **Answered** | Product + QA |
| PB-04 | Is drag-and-drop required for the first mobile release, or can tap/toolbar editing ship first? | **Both are required: drag-and-drop and tap/toolbar editing.** | Gesture Handler/Reanimated drag is on the critical path and Phase 2/7 grow; drag is no longer deferrable. | **Answered** | Product |
| PB-05 | Must an in-progress session survive backgrounding, app termination, or device restart? | **No — session persistence is not required.** | Persistence, rehydration, and a storage dependency stay out of scope; process death may lose the session. | **Answered** | Product |
| PB-06 | Is sound required, optional, or should it be removed on mobile? | **Sound is required.** | Bundled audio assets, licensing, mute behavior, and silent-switch policy enter scope. | **Answered** | Product + Design |
| PB-07 | Is there any monetization, advertising, subscription, or account plan? | **No monetization at this time.** | No IAP, ads, subscriptions, or accounts; store policy and privacy disclosure stay simple. Revisit if this changes. | **Answered** | Product |
| PB-08 | Is English-only acceptable for launch, and is localization expected later? | **English only.** | No localization architecture required; still centralize copy so it is addable later. | **Answered** | Product |
| PB-09 | What does success look like for the native app (completion rate, session length, retention, classroom adoption)? | **No measurable goal at this time.** | UAT has no quantitative target and analytics is not justified by a goal. Launch quality is judged by the PB-03 baseline and the a11y/UAT checklists. | **Answered** | Product |
| PB-10 | Should the web app be retired, frozen, or maintained in parallel after launch? | **The web app is maintained in parallel.** | The web stays available after the native launch; parity/product decisions must account for two live clients until a retirement decision is made. | **Answered** | Product |

### 3.1.1 Consequences of the PB answers

What these answers settle, and what they make newly critical:

| Answer | Settles | Makes critical next |
| --- | --- | --- |
| PB-01 | No classroom/teacher features, no accounts, no progress dashboard. Copy and supervision target ages 10–15, with a parent available. | Store rating finalized at submission (DEP-07); copy review with parent present (UX-09, TST-01). |
| PB-02 | The native app is the product’s successor, so full behavioral parity is mandatory until the web is retired. | Retirement path for the web and how existing web users are told; whether the engine is ever shared (architecture §25.8). |
| PB-03 | “Parity” means the current web behavior, so the PB-03 baseline governs UAT and every porting decision. | Baseline evidence capture and an owner (TST-05); the open human gates no longer block scope, only documentation. |
| PB-04 | Drag and tap/toolbar both ship in the first release. The tap path remains the accessible alternative, not a fallback for a missing drag. | Gesture affordance (UX-06), tap insertion behavior (UX-05), and PoC scope; Phase 2/7 estimates and risk R1 change. |
| PB-05 | No storage dependency; in-memory session is the accepted v1 posture. | Confirm the user-visible behavior on process death/back navigation (UX-08) so a child is not surprised by data loss. |
| PB-06 | Bundled celebration/success audio is required. | Silent-switch behavior is set (NAT-10); audio asset ownership/licensing and optional mute control (NAT-04) remain. |
| PB-07 | No ads/IAP/subscriptions/accounts; privacy answers stay “no data collected” unless analytics is later approved. | Nothing new; AN-01/AN-02 remain open independently. |
| PB-08 | English-only; no localization work. | Keep user-facing strings centralized for a future localization change. |
| PB-09 | No success metric; no analytics justification from goals. | AN-01/AN-02 remain open as engineering/product choices, not measurement targets. |
| PB-10 | The web app stays live in parallel after native launch. | Ownership/staffing for two clients (DEP-10) and a defined review point for retirement. |

**Tension to confirm:** PB-02 says the native app *replaces* the web app, while PB-10 says the web app is *maintained in parallel*. These are compatible only if read as “native is the intended successor, but the web stays live and maintained for now.” Under that reading, the web remains the parity reference during the overlap, and a retirement decision is deferred. If that is not the intent, PB-02 and PB-10 need to be reconciled before Phase 1 exit.

### 3.2 UI and UX

**Answered 2026-11-10:** UX-01 — **native redesign** preserving the flow (not pixel parity). UX-02 — **tablet is the primary target; phones must also open and work**; orientation (portrait vs landscape) remains open. UX-03 — **single light theme only**. UX-05 — palette taps **append to the end** of the selected side. UX-06 — **immediate drag on touch-move** (no long-press or handle). UX-08 — back/reset **preserves the in-memory session**; process death still loses it (PB-05).

| ID | Question | Why it matters / impact if unanswered | Status | Owner |
| --- | --- | --- | --- | --- |
| UX-01 | Should the native app be a faithful visual translation of the web page or a native redesign that preserves the flow? | **Native redesign that preserves the flow.** Pixel parity is explicitly not the goal. | Design review is judged against the web’s visual intent, not pixel comparison. | **Answered** (2026-11-10) | Design + Product |
| UX-02 | Which devices and orientations must be supported at launch: small phones, large phones, tablets, landscape? | **Tablet is the primary target; phones must also work.** Orientation is unstated. | Tablet-first layout plus a phone layout; test matrix covers both. Orientation still open. | **Answered** (2026-11-10) | Product + Design |
| UX-03 | Is dark mode required, and does it need to follow the system setting? | **No dark mode; single light theme only.** | No dark tokens, no system-theme switching; lock `userInterfaceStyle` to light. | **Answered** (2026-11-10) | Design |
| UX-04 | Should validation run on submit only, or also while typing? | The source shows a clear submit-time error flow. Unanswered: error timing and screen-reader noise differ. | Deferred | Design + Product |
| UX-05 | After tapping a palette token, should it append to the end of the side or insert after the selected token? | **Append to the end of the selected side.** This matches the current web behavior. | No insertion-position logic for palette taps; drag remains the way to insert mid-side. | **Answered** (2026-11-10) | Design + Product |
| UX-06 | What is the drag affordance: immediate drag on touch-move, long-press then drag, or a dedicated drag handle? | **Immediate drag on touch-move**, matching the web. | Must solve drag-vs-scroll conflict inside the scrolling screen; prototype in the Phase 2 PoC. Long-press is not the chosen fallback. | **Answered** (2026-11-10) | Design |
| UX-07 | What should completion show and do: confetti, sound, answer panel, “Solve another!”, auto-focus, replay? | **Completion plays the success sound, shows “Good job”, keeps the answer visible, and offers “Solve another!”.** Confetti is optional, not required; if kept, reduced motion disables it. | Defines the completion panel and Phase 4/7 celebration scope. | **Answered** (2026-11-10) | Design + Product |
| UX-08 | What should back/reset do mid-session: exit, confirm discard, preserve session, or return to setup? | **Preserve the in-memory session.** Back does not discard work; there is no persistence, so process death still loses it (PB-05). | Single-route shell can ignore Android back while solving; an explicit reset remains the only way to discard. | **Answered** (2026-11-10) | Product + Design |
| UX-09 | Should error copy be rewritten for mobile, or kept verbatim from the web? | The copy is part of the learning design and the spec’s message catalog. Unanswered: copy changes invalidate parity tests and may reduce clarity. | Deferred | Product + Design |
| UX-10 | How large must text be allowed to grow before layout changes (Dynamic Type / font scale)? | Affects token chips, equations, and completion layout. Unanswered: accessibility testing has no threshold. | Blocks launch | Design + QA |
| UX-11 | Should the app show a custom header/brand, or rely on system status bar and safe areas? | Affects the shell and the amount of vertical space on small phones. Unanswered: Phase 4 layout is unsettled. | Deferred | Design |
| UX-12 | Is haptic feedback wanted for taps, errors, and success? | Adds a sensory channel but can be unwanted in classrooms. Unanswered: needs a setting/decision. | Deferred | Product + Design |

### 3.3 Authentication and security

**2026-11-10:** the app is standalone with no authentication (SEC-01, SEC-02), so SEC-05 and SEC-06 are N/A. Security scope is limited to bundling no secrets and collecting no data.

| ID | Question | Why it matters / impact if unanswered | Status | Owner |
| --- | --- | --- | --- | --- |
| SEC-01 | Are user accounts, sign-in, or profiles planned at any point in the next 12 months? | **No authentication is needed.** The app is standalone; no accounts, sign-in, or profiles. | No auth provider, token storage, refresh, or route guard enters scope. | **Answered** (2026-11-10) | Product |
| SEC-02 | Will any learner data leave the device (equation history, progress, identifiers)? | **No.** The app is fully offline with no API, so no learner data is transmitted. | No data-leaves-device privacy surface; a privacy policy is still required for store submission. | **Answered** (2026-11-10) | Product + Privacy/Legal |
| SEC-03 | Is the app targeting Apple’s Kids Category and/or Google Play Families policy? What age rating? | **Audience: ages 10–15, plus older learners new to equations.** Because the audience extends beyond 12, do not target the Kids Category exclusively; list under Education. Final rating comes from the store content questionnaire. | Kids/Families restrictions still apply only if the store listing is placed in those programs; no ads, accounts, or data collection keeps the listing simple. | **Answered — store category/rating at submission** (2026-11-10) | Product + Privacy/Legal |
| SEC-04 | Who owns the privacy policy and support URL, and when will they be published? | Both stores require them; the app collects no data today but still needs a policy. Unanswered: submission cannot complete. | Blocks launch | Privacy/Legal + Release/Ops |
| SEC-05 | Is there a Microsoft/Entra ID or school-identity requirement? (No identity provider is detected today.) | **No.** No identity provider, school identity, or SSO is needed. | MSAL redirect URIs, broker behavior, and secure token storage stay out of scope. | **N/A** (2026-11-10) | Product |
| SEC-06 | If a backend or third-party SDK is added, who owns secrets and API keys? | **N/A.** There is no backend or API. | No secrets or API keys exist to own or ship. | **N/A** (2026-11-10) | Eng + Security |
| SEC-07 | Is storing the equation/session on the device acceptable, and for how long? | **N/A:** PB-05 rules out session persistence. | No session data is stored, so retention/deletion does not arise. Reopen only if persistence is added later. | **N/A (PB-05)** | Product + Privacy/Legal |
| SEC-08 | If crash reporting is added, may it include app state/input, or must it be scrubbed? | **N/A.** No crash-reporting SDK is used (AN-02). | No data leaves the device; revisit only if crash reporting is added. | **N/A** (2026-11-10) | Privacy/Legal + Eng |

### 3.4 API and backend

**2026-11-10:** no backend is planned (API-01), content is bundled (API-02), and API-03/API-04 are N/A. There is no API client, server state, or network layer in the architecture.

| ID | Question | Why it matters / impact if unanswered | Status | Owner |
| --- | --- | --- | --- | --- |
| API-01 | Is any backend planned: progress sync, accounts, teacher dashboard, content delivery? | **No.** The app is standalone and offline; no backend, API, sync, or server state. | No API client, base URL, loading/error network states, or server-state library. | **Answered** (2026-11-10) | Product |
| API-02 | Will equation content remain hard-coded, or be fetched/updated remotely? | **Content is bundled with the app**; nothing is fetched or updated remotely. | No data layer, caching, or content validation from a server; content ships in the binary. | **Answered** (2026-11-10) | Product |
| API-03 | If a backend exists, what are the base URLs/environments, and who owns them? | **N/A.** No backend exists. | No environments, endpoints, or secrets to configure. | **N/A** (2026-11-10) | Eng + Product |
| API-04 | Is offline-first behavior a requirement if a backend is added? | **N/A.** There is no backend; offline is the only mode. | No caching, sync, or conflict handling. | **N/A** (2026-11-10) | Product + Eng |

### 3.5 Native capabilities

| ID | Question | Why it matters / impact if unanswered | Status | Owner |
| --- | --- | --- | --- | --- |
| NAT-01 | Which accessibility standard is the acceptance bar (WCAG 2.2 AA equivalent, platform HIG, internal checklist)? | Without a bar, VoiceOver/TalkBack, contrast, and dynamic-type work have no pass/fail line. Unanswered: a11y sign-off is subjective. | Blocks launch | QA + Product |
| NAT-02 | Is VoiceOver, TalkBack, or both required at launch, and on which OS versions? | Determines the manual test matrix and any platform-specific announcements/focus work. Unanswered: one platform may ship untested. | Blocks launch | QA |
| NAT-03 | What does “keyboard accessible” mean on mobile: screen-reader actions only, external hardware keyboard, or both? | **Both:** full screen-reader actions and external hardware-keyboard focus/activation. | Requires explicit focus order and activation for hardware keyboards, plus `accessibilityActions`; test with VoiceOver/TalkBack and a Bluetooth keyboard. | **Answered** (2026-11-10) | Product + QA |
| NAT-04 | Are haptics and audio expected to have user-facing toggles? Where should those live? | The web has no settings screen. Unanswered: adds UI and persistence scope. | Deferred | Product + Design |
| NAT-05 | Is share/export/print of an equation or progress needed? | Adds file/share modules and permissions. The source has none. Unanswered: scope discovery late. | N/A today | Product |
| NAT-06 | Are app icon, splash, and store screenshots designed and approved? Who owns the brand assets? | Required for submission. Unanswered: release blocked at the last step. | Blocks launch | Design + Release/Ops |
| NAT-07 | Are there any device permissions expected (camera, microphone, files, notifications)? | The current app needs none. Unanswered: privacy disclosures and review risk. | N/A today | Product |
| NAT-08 | Should the app support deep links (for example a shared equation or a classroom link)? | The source has no routes or links. Unanswered: router/scheme design may need to change. | Deferred | Product |
| NAT-09 | Are app clips, widgets, watch, or TV targets in scope? | Each is a separate native target and would change the delivery model. Unanswered: architecture and effort are wrong. | N/A today | Product |
| NAT-10 | Is the system silent switch expected to mute game sounds? | **Audio is expected to play.** Use the platform-default behavior: the iOS silent switch mutes, and Android follows media volume. No in-app mute control. | Configures the audio session for ambient/respect-silent playback; sound remains part of completion and feedback. | **Answered** (2026-11-10) | Product + Design |

### 3.6 Offline and connectivity

**2026-11-10:** fully offline including first launch (OFF-01), no online-only features (OFF-02), and OFF-04 is N/A. The app must never depend on a network call to launch or solve.

| ID | Question | Why it matters / impact if unanswered | Status | Owner |
| --- | --- | --- | --- | --- |
| OFF-01 | Must the app work fully offline, including first launch after install? | **Yes.** The app is standalone and fully offline, including first launch. | No hidden network dependency; all JS, audio, and assets are bundled; airplane mode is the test baseline. | **Answered** (2026-11-10) | Product |
| OFF-02 | Are there any online-only features planned (content updates, leaderboards, sync)? | **No.** No online-only features. | No data layer or network error states. | **Answered** (2026-11-10) | Product |
| OFF-03 | If the app is offline and an OTA update check fails, should launch continue silently? | Still an engineering/release question, but only relevant if EAS Update is adopted (DEP-05). | If OTA is adopted, a failed check must never block launch. | Deferred | Eng + Release/Ops |
| OFF-04 | How should the app behave on slow or unstable networks if a backend is added later? | **N/A.** No backend, so no slow-network behavior exists. | No retries, timeouts, or offline states to design. | **N/A** (2026-11-10) | Eng |

### 3.7 Notifications

| ID | Question | Why it matters / impact if unanswered | Status | Owner |
| --- | --- | --- | --- | --- |
| NOT-01 | Are push or local notifications planned (practice reminders, streaks, teacher assignments)? | Adds permissions, a notification service, deep links, and privacy disclosure. The source has none. Unanswered: a likely engagement feature may be missing from scope. | Deferred | Product |
| NOT-02 | If notifications are added, who owns the messaging, scheduling, and opt-in flow? | Permission prompts and notification content need product ownership. Unanswered: cannot implement responsibly. | N/A today | Product |
| NOT-03 | Are notifications allowed under the chosen kids/families store policy? | Some notification/ad patterns are restricted for children. Unanswered: store rejection risk. | N/A today | Privacy/Legal |

### 3.8 Analytics and monitoring

**2026-11-10:** no analytics and no crash reporting (AN-01, AN-02). AN-04, AN-05, and SEC-08 are N/A as a consequence. Store privacy answers remain “Data Not Collected.”

| ID | Question | Why it matters / impact if unanswered | Status | Owner |
| --- | --- | --- | --- | --- |
| AN-01 | Is product analytics allowed at launch? If so, which events and which vendor? | **No analytics or analytics reporting.** | No analytics SDK, event schema, consent prompt, or disclosure beyond “no data collected.” | **Answered** (2026-11-10) | Product |
| AN-02 | Is crash reporting required? Which vendor, and what data may be collected? | **No crash reporting.** Treated as part of “no analytic report” and the standalone offline posture; no reporting SDK transmits data. | Production crashes are diagnosed from local/dev builds and user reports, not a vendor dashboard. | **Answered** (2026-11-10) | Eng + Product |
| AN-03 | Is performance monitoring required (startup, frame drops)? | Not covered by the “no analytics” answer. Unanswered: whether any performance SDK is wanted on top of manual DevTools checks. | Useful for validating drag on low-end Android. Unanswered: performance regressions go unnoticed. | Deferred | Eng |
| AN-04 | Is any consent/ATT prompt required for tracking? | **N/A.** No tracking or analytics. | No ATT/consent prompt. | **N/A** (2026-11-10) | Privacy/Legal |
| AN-05 | Who owns the event schema and dashboards if analytics is added? | **N/A.** No event schema or dashboards. | Nothing to own. | **N/A** (2026-11-10) | Product |

### 3.9 Testing and UAT

| ID | Question | Why it matters / impact if unanswered | Status | Owner |
| --- | --- | --- | --- | --- |
| TST-01 | Who participates in UAT, how are they recruited, and what scenarios must pass? | UAT is a release gate. Unanswered: no objective acceptance. | Blocks launch | Product + QA |
| TST-02 | Who signs off accessibility, and against which checklist? | Required for launch quality. Unanswered: cannot declare a11y done. | Blocks launch | QA + Product |
| TST-03 | Which iOS and Android minimum versions and physical devices are in the test matrix? | **iPadOS/iOS 16+, Android 9 (API 28)+**, raised to the chosen Expo SDK’s floor if it is higher. Test devices: iPad 9th gen or newer + iPad Air; Galaxy Tab A9+ or S9 FE; iPhone SE 3rd gen + a current iPhone; Pixel 8a + one 3 GB low-end Android. | Sets emulator images, dependency versions, and the manual matrix. | **Answered** (2026-11-10) | Eng + QA |
| TST-04 | Is a physical-device pass required for every release, or are simulators acceptable? | Audio, haptics, gestures, and performance need real devices. Unanswered: release quality is unverifiable. | Blocks launch | QA + Release/Ops |
| TST-05 | Who owns the web baseline evidence (manual checklist results, screenshots, recorded sessions)? | **Boris (product owner) owns the baseline evidence.** | Evidence is captured before parity disputes arise; migration acceptance has an owner. | **Answered** (2026-11-10) | Boris |
| TST-06 | Are beta testers available via TestFlight and Play Internal App Sharing? | Enables real-user validation before public release. Unanswered: no staged rollout. | Blocks launch | Product + Release/Ops |
| TST-07 | Must the 70-case math self-check corpus remain the authoritative engine test set? | **Yes — keep all 70 cases; do not halve.** They execute in milliseconds; the real cost is the one-time port to Jest, which is table-driven and low. Halving saves little and drops error-code/branch coverage. | It is the strongest regression asset for the engine; every rejection code and the candidate final/inequivalent paths are represented. | **Answered** (2026-11-10) | Eng |
| TST-08 | Are there regulated accessibility or education-sector procurement requirements? | May impose VPAT/accessibility statements and data terms. Unanswered: launch could fail procurement. | Deferred | Product + Privacy/Legal |

### 3.10 Deployment and operations

| ID | Question | Why it matters / impact if unanswered | Status | Owner |
| --- | --- | --- | --- | --- |
| DEP-01 | Who owns the Apple Developer, Google Play, and Expo/EAS accounts, and are they already provisioned? | **Boris owns the Apple Developer, Google Play, and Expo/EAS accounts.** Provisioning status still to be confirmed. | Ownership is settled; only the provisioning checklist remains. | **Answered** (2026-11-10) | Boris |
| DEP-02 | What are the final app name, bundle identifiers (`com.*`), Android package, and URL scheme? | **App name: Equation Explorer. Bundle/package: `net.bbsteps.equation`. URL scheme: `equation`. Public URL: `equation.bbsteps.net`.** | Identity is frozen; injecting a different bundle/package after the first store upload requires a new app record. | **Answered** (2026-11-10) | Boris + Eng |
| DEP-03 | Is EAS Build/Submit cloud delivery approved, or are local builds required? | **Local builds** (Xcode/Gradle with locally generated native projects). No EAS Build/Submit. | Signing, CI, and release steps are local; requires macOS/Xcode and Android SDK maintenance on the build machine. | **Answered** (2026-11-10) | Boris |
| DEP-04 | Which CI provider and branch/release process are used? | Gates tests and builds. Unanswered: quality checks are manual. | Blocks launch | Eng |
| DEP-05 | Is OTA update (EAS Update) allowed? Under what rollout/rollback policy? | Adds a runtime-update lane and audit requirements. Unanswered: hotfix capability unknown. | Deferred | Eng + Release/Ops |
| DEP-06 | What release cadence and phased-rollout percentage are expected? | Affects monitoring and support. Unanswered: cannot plan post-launch response. | Blocks launch | Product + Release/Ops |
| DEP-07 | What is the store category, age rating, and privacy/data-safety answer set? | Required for submission. Unanswered: review rejection risk, especially for kids. | Blocks launch | Product + Privacy/Legal |
| DEP-08 | Who provides support and responds to store reviews/user feedback? | Required operationally. Unanswered: no post-launch owner. | Blocks launch | Product |
| DEP-09 | What is the minimum supported iOS/Android OS, and does the current Expo SDK support it? | **iOS/iPadOS 16+ and Android 9 (API 28)+**, raised to the chosen Expo SDK’s floor if it is higher. | Determines device coverage and dependency versions; pair with TST-03. | **Answered** (2026-11-10) | Eng + Product |
| DEP-10 | Who owns ongoing dependency upgrades, store compliance, and two-platform releases after launch? | The web needed no store maintenance; native does. Unanswered: the app may decay after launch. | Blocks launch | Eng + Product |
| DEP-11 | Is there a maintenance/app-update budget for the first 12 months? | Covers SDK upgrades, OS changes, and store requirements. Unanswered: sustainability is unknown. | Deferred | Product |

### 3.10.1 App identity format check (DEP-02)

Confirmed values (2026-11-10). The reverse-DNS form was adopted so the identifiers match the `bbsteps.net` domain the project controls:

| Item | Confirmed value | Note |
| --- | --- | --- |
| App name | Equation Explorer | Appears in store listings and `app.json`. |
| iOS bundle ID / Android package | `net.bbsteps.equation` | Reverse-DNS for `bbsteps.net`; permanent after the first store upload. |
| URL scheme | `equation` | Used for deep links if they are ever added; `equation.bbsteps.net` is not a valid scheme. |
| Public URL | `equation.bbsteps.net` | Store support/marketing URL and, later, the universal-link host with `applinks:`/`intentFilters`. |

---

## 4. Highest-impact questions

Ordered by how much of the plan changes if the answer is different.

**Resolved on 2026-11-10** are marked with their answer; the rest remain open.

1. **PB-03 — Is the web baseline accepted?** ✅ **Answered: yes, current web behavior is the baseline.** Evidence capture (TST-05) is still required.
2. **PB-04 / UX-05 / UX-06 — Is drag required at launch, and how does tap editing work?** ✅ **All answered:** both drag and tap/toolbar are required (PB-04), palette taps append (UX-05), and drag starts immediately on touch-move (UX-06). The remaining risk is technical: making immediate drag coexist with scrolling.
3. **PB-05 / UX-08 — Must sessions survive termination, and what does back do?** ✅ **Answered: no persistence; back preserves the in-memory session.** Only process death loses work.
4. **NAT-03 — What does keyboard accessibility mean on mobile?** ✅ **Answered: both screen-reader actions and external hardware keyboard.** Custom focus order and activation work are required, not just `accessibilityActions`.
5. **PB-01 / SEC-03 — Target age and store category.** ✅ **Answered:** ages 10–15 plus older learners new to equations; child alone or with a parent. Not a Kids-Category-only app; list under Education. The exact star rating comes from the store questionnaire and is confirmed at submission (DEP-07).
6. **PB-06 / NAT-10 — Is sound required, and does the silent switch mute it?** ✅ **Answered:** sound is required and expected to play; the iOS silent switch mutes and Android follows media volume. NAT-04 (mute/haptics controls) is deferred.
7. **UX-02 / TST-03 / DEP-09 — Device, orientation, and OS matrix.** ✅ **Answered:** tablet primary, phones supported; iOS/iPadOS 16+ and Android 9 (API 28)+. Orientation remains open. Tablet-first layouts increase Phase 4 scope.
8. **SEC-04 / DEP-07 — Privacy policy and data-safety answers.** Still open. Required for submission; unresolved late, they block release.
9. **DEP-01 / DEP-02 / DEP-03 / DEP-09 — Accounts, identifiers, build path, OS floor.** ✅ **Answered:** Boris owns the accounts; identity is `net.bbsteps.equation` / scheme `equation` / `equation.bbsteps.net`; local Xcode/Gradle builds; iOS/iPadOS 16+ and Android 9 (API 28)+. Release cadence and provisioning check remain.
10. **AN-01 / AN-02 — Analytics and crash reporting.** ✅ **Answered: neither.** No analytics SDK, event schema, crash reporter, or consent prompt; production issues are diagnosed locally and from user reports.

---

## 5. Decisions needed before each delivery phase

Rows struck through are resolved by the 2026-11-10 answers. Remaining IDs must still be resolved.

| Phase | Must be resolved before starting |
| --- | --- |
| 1. Discovery | ~~PB-01~~, ~~PB-02~~, ~~PB-03~~, ~~PB-04~~, ~~PB-07~~, ~~SEC-01~~, ~~SEC-02~~, ~~API-01~~, ~~OFF-01~~, ~~TST-03~~, ~~TST-05~~, ~~TST-07~~, ~~DEP-01~~, ~~DEP-02~~, ~~DEP-03~~, ~~DEP-09~~ |
| 2. Architecture/PoC | ~~PB-04~~, ~~PB-06~~, ~~UX-06~~, ~~NAT-03~~, ~~NAT-10~~, A1–A10 confirmed or revised (A1/A3/A5/A6/A7/A8/A10 confirmed; A2/A4 revised; A9 needs revision) |
| 3. Foundation | ~~DEP-01~~, ~~DEP-02~~, ~~DEP-03~~, ~~DEP-09~~, ~~PB-05 (persistence)~~ |
| 4. Static UI | ~~UX-01~~, UX-02 (orientation only — tablet/phone decided), ~~UX-03~~, ~~UX-05~~, ~~UX-07~~, ~~UX-08~~, UX-10, UX-11, NAT-06 |
| 5. Logic/validation | ~~PB-03~~, ~~TST-07~~, UX-04, UX-09 |
| 6. API/auth | ~~SEC-01~~, ~~SEC-05~~, ~~API-01–API-04~~ — phase is N/A |
| 7. Storage/native | ~~PB-05~~, ~~PB-06~~, ~~SEC-07~~, NAT-04, ~~NAT-10~~, UX-12 |
| 8. Testing/a11y | NAT-01, NAT-02, ~~NAT-03~~, UX-10, TST-02, TST-04 |
| 9. UAT | TST-01, TST-06, ~~PB-09~~ |
| 10. Store prep | ~~SEC-03~~ (rating finalized at submission), SEC-04, DEP-06, DEP-07, DEP-08, NAT-06 |
| 11. Release | DEP-04, DEP-05, ~~AN-01~~, ~~AN-02~~, DEP-10, DEP-11 |

Open follow-up before Phase 1 exits: reconcile PB-02 (replace) with PB-10 (maintain in parallel) as described in §3.1.1, and record the web retirement approach.

---

## 6. Decision log

Copy a row per decision. Do not delete the original question ID. Rows below were supplied by the product owner on 2026-11-10; no rationale beyond the stated answer was provided, so the rationale column records the consequence rather than an invented reason.

| ID | Question (short) | Decision | Decided by | Date | Rationale / evidence | Docs updated |
| --- | --- | --- | --- | --- | --- | --- |
| PB-01 | Primary user | Child with parent, and child alone; no classroom/teacher scope | Product | 2026-11-10 | Stated answer | This doc §3.1, §3.1.1; migration plan §1.1 |
| PB-02 | Web relationship | Native app replaces the web app | Product | 2026-11-10 | Stated answer | This doc §2 A9, §3.1.1; migration plan §10 |
| PB-03 | Web baseline acceptance | Current web behavior is the parity baseline | Product | 2026-11-10 | Stated answer; evidence capture still required (TST-05) | This doc §1, §3.1, §3.1.1; migration plan §1.18, §10.3 |
| PB-04 | Drag at launch | Both drag-and-drop and tap/toolbar editing required | Product | 2026-11-10 | Stated answer; drag no longer deferrable | This doc §2 A2, §4; migration plan §10.2, R1; architecture ADR-005 |
| PB-05 | Session persistence | No persistence required | Product | 2026-11-10 | Stated answer; in-memory v1 confirmed | This doc §2 A3; migration plan §5.2; architecture §11, ADR-006 |
| PB-06 | Sound | Sound is required | Product + Design | 2026-11-10 | Stated answer; bundled audio in scope | This doc §3.1.1; migration plan §3, §5.1, R6; architecture §15, ADR-007 |
| PB-07 | Monetization | No monetization at this time | Product | 2026-11-10 | Stated answer; no IAP/ads/accounts planned | This doc §3.1; migration plan §5.4–5.6; architecture §10 |
| PB-08 | Localization | English only | Product | 2026-11-10 | Stated answer | This doc §2 A6; architecture §15 |
| PB-09 | Success metric | No measurable goal at this time | Product | 2026-11-10 | Stated answer; UAT judged by baseline + checklists | This doc §3.1, §4; migration plan §9 |
| PB-10 | Web lifecycle | Web app maintained in parallel | Product | 2026-11-10 | Stated answer; conflicts with PB-02 unless read as “successor, not retired yet” (see §3.1.1) | This doc §2 A9, §3.1.1; migration plan §10 |
| SEC-01 | Accounts/sign-in | No authentication; standalone app | Product | 2026-11-10 | Stated answer “no auth needed” | This doc §3.3; migration plan §5.4, Phase 6; architecture §10 |
| SEC-02 | Learner data leaves device | No; fully offline, no API | Product | 2026-11-10 | Stated answer “no API needed” | This doc §3.3; migration plan §5.1; architecture §10 |
| API-01 | Backend planned | No backend/API/sync | Product | 2026-11-10 | Stated answer “no API needed” | This doc §3.4; migration plan §2.5, Phase 6; architecture §10 |
| OFF-01 | Offline including first launch | Yes; standalone offline app | Product | 2026-11-10 | Stated answer | This doc §3.6; migration plan §6.10; architecture §2 |
| SEC-05, SEC-06, API-02–API-04, OFF-02, OFF-04 | Consequences of the above | N/A or bundled content; no backend, no remote content, no online features | Product | 2026-11-10 | Direct consequence of the standalone/offline decision | This doc §3.3–3.4, §3.6 |
| UX-02 | Device scope | Tablet is the primary target; phones must also open and work | Product + Design | 2026-11-10 | Stated answer; orientation not stated | This doc §2 A4, §3.2, §4; migration plan §1.14, §6.7–6.8, Phase 4; architecture §14, §19 |
| UX-03 | Dark mode | Single light theme only; no dark mode | Design | 2026-11-10 | Stated answer | This doc §2 A5, §3.2; architecture §14, ADR-009 |
| UX-01 | Visual direction | Native redesign preserving the flow; not pixel parity | Product + Design | 2026-11-10 | Stated answer | This doc §3.2, §4; migration plan §10.2, Phase 4; architecture §6 |
| UX-05 | Palette tap insertion | Append to the end of the selected side | Design + Product | 2026-11-10 | Stated answer; matches web behavior | This doc §3.2; migration plan §3; architecture §7 |
| UX-06 | Drag affordance | Immediate drag on touch-move; no long-press/handle | Design | 2026-11-10 | Stated answer; matches web behavior | This doc §3.2, §4; migration plan R1, Phase 2; architecture ADR-005 |
| UX-07 | Completion behavior | Success sound, “Good job”, answer visible, “Solve another!”; confetti optional and reduced-motion-safe | Product + Design | 2026-11-10 | Stated answer | This doc §3.2, §4, §5; migration plan §7 Phase 4/7; architecture §6 |
| UX-08 | Back/reset mid-session | Preserve the in-memory session; back does not discard | Product + Design | 2026-11-10 | Stated answer; consistent with no persistence (PB-05) | This doc §3.2, §4; migration plan §3; architecture ADR-002, §25 |
| SEC-03 | Age band and store program | Ages 10–15 plus older learners new to equations; not Kids-Category-only; Education category; rating from the store questionnaire | Product | 2026-11-10 | Stated answer | This doc §3.3, §4, §5; migration plan §5.6, Phase 10 |
| NAT-03 | Mobile keyboard accessibility | Both screen-reader actions and external hardware keyboard | Product + QA | 2026-11-10 | Stated answer | This doc §3.5, §4, §5; architecture §18; risk R8 |
| NAT-10 | Silent-switch policy | Audio expected; iOS silent switch mutes, Android follows media volume; no in-app mute | Product + Design | 2026-11-10 | Stated answer | This doc §3.5, §4, §5; migration plan Phase 7, R6; architecture ADR-007 |
| TST-03 | OS floor and device matrix | iPadOS/iOS 16+, Android 9 (API 28)+; tablet-first device matrix as listed in §3.9 | Boris | 2026-11-10 | Recommendation approved | This doc §3.9, §4, §5; migration plan §6.7; architecture §25 |
| TST-05 | Baseline evidence owner | Boris | Boris | 2026-11-10 | Stated answer | This doc §3.9; migration plan Phase 1 |
| TST-07 | Engine corpus size | Keep all 70 cases; do not halve | Boris | 2026-11-10 | Recommendation approved; corpus runs in milliseconds | This doc §3.9, §5; migration plan §6.2, §10.1; architecture §20 |
| DEP-01 | Store/Expo accounts | Boris owns Apple Developer, Google Play, Expo/EAS accounts | Boris | 2026-11-10 | Stated answer; provisioning status still to confirm | This doc §3.10; migration plan Phase 1/10 |
| DEP-02 | App identity | Equation Explorer; bundle/package `net.bbsteps.equation`; scheme `equation`; public URL `equation.bbsteps.net` | Boris | 2026-11-10 | Stated answer; reverse-DNS format accepted (§3.10.1) | This doc §3.10.1; migration plan §4; architecture §15 |
| DEP-03 | Build path | Local builds with Xcode/Gradle; no EAS Build/Submit | Boris | 2026-11-10 | Stated answer | This doc §3.10; migration plan §4, Phase 3/10; architecture ADR-001 |
| DEP-09 | Minimum OS versions | iOS/iPadOS 16+, Android 9 (API 28)+, raised to the Expo SDK floor if higher | Boris | 2026-11-10 | Recommendation approved; paired with TST-03 | This doc §3.10, §4, §5; migration plan §6.7; architecture §25 |
| AN-01 | Analytics | No analytics or analytics reporting | Product | 2026-11-10 | Stated answer | This doc §2 A7, §3.8, §4; migration plan §5.5; architecture §17 |
| AN-02 | Crash reporting | No crash reporting | Product + Eng | 2026-11-10 | Read from “no analytic report” plus the standalone offline posture; confirms no reporting SDK transmits data | This doc §2 A7, §3.8; migration plan §5.5, Phase 11; architecture §17 |
| Toolchain | Implementation stack | React Native + TypeScript + Expo modules + Development Builds + Maestro + Claude Code | Eng | 2026-11-10 | Stated preference | This doc §2 A8; migration plan §4, §6.5, §10.1; architecture ADR-001, ADR-008 |

**Recommendation batch confirmed by the product owner on 2026-11-10:** TST-03 (device/OS matrix), TST-07 (keep all 70 engine cases), DEP-09 (OS floor), and the DEP-02 reverse-DNS identity `net.bbsteps.equation`. No recommendation from that batch remains pending.

---

## 7. Effect of answers on the plans

| Answer area | Documents/sections to revise |
| --- | --- |
| Primary user (PB-01) | Migration plan §1.1, §7 Phase 9; audience set at 10–15 (SEC-03); store rating at submission (DEP-07); copy review (UX-09) |
| Web relationship/lifecycle (PB-02, PB-10) | Migration plan §10; architecture §1; parallel-maintenance staffing (DEP-10); future web retirement plan |
| Parity baseline (PB-03) | Migration plan §1.18, §6, §10; TST-05 evidence owner |
| Drag at launch (PB-04) | Migration plan §7 Phase 2/7 and RN-1; architecture ADR-005; risk R1 |
| English only (PB-08) | Architecture §15; keep strings centralized for future localization |
| Monitoring/success (PB-07, PB-09) | Architecture §17; migration plan §5.5 — analytics still an open engineering decision, not goal-driven |
| Accounts/auth (SEC-01, SEC-05) | **Resolved N/A.** Migration plan §4, §5.4, §7 Phase 6, §8; architecture §10–11; no SecureStore/auth SDK |
| Backend/API (API-01–API-04) | **Resolved N/A.** Migration plan §2.5, §7 Phase 6; architecture §10; no data layer or network dependency |
| Offline (OFF-01, OFF-02) | **Resolved: fully offline.** Migration plan §6.10; architecture §2, §10 |
| Persistence | Architecture §11, §7.1; migration plan §5.2–5.3, Phase 7; privacy section |
| Drag at launch | Migration plan §7 Phase 2/7 and RN-1; architecture ADR-005; risk R1 |
| Audio/haptics | Architecture §15–16; dependency register; migration plan Phase 7; risk R6 |
| Tablet/landscape | **Tablet resolved as primary (UX-02); orientation still open.** Architecture §14, §19; migration plan §6.7–6.8; Phase 4 and effort |
| Dark mode | **Resolved: light only (UX-03).** Architecture §14, ADR-009; no system-theme testing needed |
| Analytics/crash reporting | **Resolved: none (AN-01, AN-02).** Migration plan §5.5, §6.13; architecture §17; store privacy answers remain “Data Not Collected” |
| Notifications | Architecture §10; dependency register; privacy; store category |
| Keyboard accessibility | Architecture §18; migration plan §6.9; risk R8 |
| OS/device matrix | Migration plan §6.7; architecture dependency versions; effort/risk |
| Store/account decisions | Migration plan §7 Phase 10–11; architecture §15; risk R10/R18 |

---

## 8. Questions deliberately out of scope unless scope changes

These were checked and are **not** part of the current application. They are listed so future scope decisions do not silently inherit a “no” answer. The 2026-11-10 answers confirm several of these for the native v1: no accounts or authentication (PB-01, PB-07, SEC-01), no backend/API (API-01), no payments/ads (PB-07), English only (PB-08), and no session persistence (PB-05). The app is standalone and fully offline (OFF-01), with no analytics or crash reporting (AN-01, AN-02).

- Accounts, sign-in, profiles, or identity providers.
- Backend APIs, sync, teacher dashboards, or content management.
- Payments, subscriptions, advertising, or in-app purchases.
- Push/local notifications.
- Camera, microphone, photo library, files, contacts, or location.
- Offline content downloads beyond the bundled app.
- Sharing, export, print, or social features.
- Tablet-specific, landscape, watch, TV, widget, or app-clip layouts.
- Localization beyond English.
- Analytics, crash reporting, session replay, or experimentation.
- WebView, embedded web content, or DOM components.
- A public web version of the React Native app.