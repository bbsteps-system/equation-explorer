# React Native Migration Plan — Equation Explorer

**Status:** Planning only. No React Native project has been created, no dependencies installed, and the existing web application has not been modified. Product questions PB-01–PB-10 were answered on 2026-11-10 and are reflected below; the remaining questions are in `react-native-requirements-open-questions.md`.

**Date:** 2026-11-09; PB, UX, connectivity, analytics, and toolchain decisions incorporated 2026-11-10

**Target:** A real React Native application for iOS and Android, TypeScript in strict mode, **tablet-first layout with phone support**, single light theme, React Native New Architecture where compatible. A WebView or DOM-component wrapper of the existing HTML is explicitly out of scope.

**Sources inspected**

| Source | What it establishes |
| --- | --- |
| `index.html` (2,295 lines) | Complete application: markup, CSS, math engine, state, DOM rendering, pointer drag, validation, effects |
| `README.md` | Product summary, run/deploy model, partly stale status |
| `docs/specs/linear-equation-drag-solver.md` | Product intent, scope, functional/non-functional requirements, acceptance checklist |
| `docs/specs/milestones/00`–`04` | Milestone contracts and human gates |
| `tasks/00`–`04` | Implementation plans, task status, verification evidence |
| `CLAUDE.md` | Project engineering constraints |
| Installed skills | `assess-react-native-migration`, `expo-web-to-native`, `expo-overview`, `expo-project-structure`, `expo-animation`, `expo-native-ui` (storage), `react-native-best-practices`, `react-navigation`, `planning-and-task-breakdown` |

**Evidence labels used throughout**

- `[Observed]` — directly visible in the source.
- `[Inferred]` — a reasonable technical conclusion from source structure; not yet verified on a device.
- `[Recommendation]` — a proposed decision for this migration.
- `[Open]` — cannot be answered from the source; belongs in `docs/react-native-requirements-open-questions.md`.

No runtime testing, device testing, screen-reader testing, or store validation was performed for this plan. Statements about the current app are source-code findings, not acceptance results.

---

## 0. Executive summary

`[Observed]` Equation Explorer is a dependency-free, fully client-side, offline single-page game that teaches single-variable linear equations in `y`. A learner enters an equation, the app validates it with an exact `BigInt` rational engine, then the learner builds equivalent next steps from draggable math tokens until `y` is isolated. There is no backend, account system, API, analytics, browser storage, third-party library, custom font, or image asset.

`[Observed]` The business logic is unusually portable: the tokenizer, recursive-descent parser, affine reducer, exact rational arithmetic, source analysis, and candidate validation are pure JavaScript functions in a single inline script (`index.html:621-1098`). The UI is the opposite: it is built from 455 lines of CSS and heavy DOM/pointer-event manipulation (`index.html:9-462`, `index.html:1450-2292`), so the visual and interaction layers must be rebuilt natively rather than wrapped.

`[Recommendation]` Use **Expo with managed/Continuous Native Generation (CNG)**, no committed `ios/` or `android/` directories, a development build for realistic testing, and **local Xcode/Gradle builds** for store delivery (DEP-03; no EAS Build/Submit). Use **Expo Router** for a single-route shell, **TypeScript strict**, **React Native New Architecture**, **Gesture Handler + Reanimated** for the immediate-drag token layer (UX-06), **expo-audio + expo-haptics** for the required sound and optional haptics, and **Jest + React Native Testing Library + Maestro** for tests. Expo is sufficient because the app needs no custom native module, camera, file picker, push notification, background task, authentication, or backend.

`[Recommendation]` Rebuild the UI as a native tablet-first experience (UX-02) that still works on phones, rather than reproducing the web page pixel-for-pixel: Setup and Solver become two screens; the token palette becomes a reachable toolbox (bottom-docked on phones, side-docked or bottom-docked on tablets); the history becomes a scrollable native list; completion is an inline panel; hover styles are replaced by press states. The math engine and its 70-case self-check corpus should be ported almost line-for-line into a pure TypeScript module and made the regression backbone of the migration.

**Largest migration risks**

1. `[Inferred]` **Drag-and-drop parity.** The web relies on Pointer Events, `document.elementFromPoint`, `getBoundingClientRect`, pointer capture, a body-level drag preview, and DOM insertion markers (`index.html:1884-2005`). Native has none of these and must hit-test measured token rectangles inside a scrollable screen.
2. `[Inferred]` **Accessibility parity.** The web uses live regions, ARIA pressed/invalid states, explicit focus moves, keyboard alternatives, and reduced-motion CSS. Native equivalents are different APIs and, in the case of external-keyboard token editing, are not guaranteed.
3. `[Inferred]` **Visual-native redesign.** CSS gradients, sticky positioning, `clamp()`, media queries, `box-shadow`, keyframes, and safe-area environment variables have no direct React Native equivalents.
4. `[Decision]` **Baseline accepted, evidence still to capture.** PB-03 (2026-11-10) accepts the current web behavior as the parity baseline, so “parity” is defined. The web spec still says M3's human acceptance and M4's human release gate remain open (`docs/specs/linear-equation-drag-solver.md:3`, `tasks/04-todo.md`), and `README.md:5-8` still describes the app as a UI-only preview. The baseline decision does not exempt the project from recording checklist evidence (TST-05).

**Indicative effort:** 35–58 person-days for the phases below, plus 15–20% contingency (40–70 person-days total), excluding App Store/Google Play review waiting time. This is a planning range, not a commitment. Phase 4 grows because UX-02 requires a tablet-first layout plus a phone layout; Phase 7 sits at its upper end because PB-04 and PB-06 make drag and sound first-release requirements.

---

## 1. Existing application inventory

### 1.1 Purpose and primary users

`[Observed]` The app is a child-friendly practice game for solving a one-variable linear equation step by step. The learner enters a supported equation, sees it locked as Step 1, builds an equivalent next equation from math tokens, checks it, and repeats until the final form `y = value` is accepted and celebrated (`docs/specs/linear-equation-drag-solver.md:5-15`, `index.html:464-579`).

`[Observed]` Primary users:

- **Learner (child):** enters or is given an equation, manipulates tokens, reads feedback, completes the equation.
- **Parent / supporting adult:** helps enter the starting equation and explains why unsupported equations are declined. No account or adult-only area exists.

`[Decision]` PB-01/SEC-03 (2026-11-10): the target is **a child with a parent and a child alone**, ages **10–15**, plus older learners new to equations. Classroom/teacher use is out of scope, so no teacher dashboard, class roster, or account model is planned. The app is not Kids-Category-only; list under Education, with the star rating set by the store questionnaire at submission.

`[Observed]` There is no login, profile, progress saving, leaderboard, hint system, or teacher dashboard.

### 1.2 Feature inventory

| Feature | What it does | Source |
| --- | --- | --- |
| Equation setup | Text input, example placeholder, Start button, tip card, inline feedback | `index.html:476-491` |
| Source validation | Tokenize, parse, reduce to affine form, solve exactly, or reject with a specific message | `index.html:1018-1029` |
| Direct-reciprocal exception | Accepts `c / y = d` or `d = c / y` for nonzero constants, with `y ≠ 0` retained | `index.html:934-969`, spec `:21` |
| Locked Step 1 | Shows the normalized source equation as immutable history | `index.html:514-518`, `index.html:1519-1549` |
| Token palette | 18 token buttons: `0`–`9`, `.`, `+`, `−`, `×`, `÷`, `(`, `)`, `y` | `index.html:597-602`, `index.html:1450-1462` |
| Active editor | Two labeled token sides separated by `=`, initially copied from the latest accepted row | `index.html:528-538`, `index.html:2195-2205` |
| Palette insertion | Tap a palette token to append it to the selected side | `index.html:1828-1842` |
| Token selection | Tap a workspace token; toolbar and focus update | `index.html:1818-1826` |
| Reorder / move | Move selected token left/right; move to the other side | `index.html:1859-1882` |
| Delete | Delete selected token, or drag a token to the trash zone | `index.html:1844-1857`, `index.html:1952-1958` |
| Clear side | Empty only the selected side | `index.html:2044-2056` |
| Flip sides | Swap the two active sides | `index.html:2058-2067` |
| Restore previous step | Replace both active sides with fresh copies of the latest locked row | `index.html:2069-2081` |
| Check | Validate both sides, compare the exact solution with Step 1, reject, lock, or complete | `index.html:2135-2170` |
| Locked history | Immutable accepted rows with step number and locked/solved status | `index.html:1519-1549`, `index.html:2099-2133` |
| Duplicate guard | Ignore rapid repeated Check within 500 ms; reject a row identical to the previous step | `index.html:2011-2027`, `index.html:2141-2148` |
| Completion | Final `y = value` row, praise, answer, “Solve another!” | `index.html:562-576`, `index.html:2112-2123` |
| Celebration | One confetti burst and one synthesized chime; sound is **required** on native (PB-06), haptics optional | `index.html:1558-1664`, `index.html:1724-1729` |
| Toast | Brief visual success confirmation; hidden from assistive technology | `index.html:1666-1702`, `index.html:581` |
| Reset | New equation / Reset and Solve another return to landing | `index.html:2210-2230` |
| Dev self-check | 70 assertions exposed as `window.runMathSelfChecks()` | `index.html:1104-1292` |

### 1.3 User journeys

`[Observed]` Happy path (`docs/specs/linear-equation-drag-solver.md:105-110`):

1. Landing: title, input, tip, disabled Start until non-empty.
2. Enter `2y - 3 = 8`; press Start.
3. Source is normalized and locked as Step 1; Step 2 is prefilled with token copies; palette and editor become active.
4. Choose a side; tap palette tokens, drag tokens, reorder, move across sides, or delete.
5. Press Check. An invalid row stays editable with a specific message and focus on the affected side. An inequivalent row is rejected with “That changes the answer for y.”
6. An equivalent row locks, shows a toast and success chime, and creates a fresh editable next row.
7. Repeat until the row is exactly `y = value`; lock it, show the completion panel (“Good job”, answer, “Solve another!”) and play the celebration chime. Confetti is optional and reduced-motion-safe.
8. Press “Solve another!” or “New equation / Reset” to return to landing.

`[Observed]` Error paths: empty input, whitespace-only input, malformed source, unsupported variable/power/symbol, nonlinear product, unsupported rational, identity/contradiction, too-long input, malformed candidate side, empty candidate side, same-as-previous step, inequivalent candidate. All error messages are child-friendly and mapped from error codes (`index.html:603-619`, `index.html:1043-1049`).

### 1.4 Page sections

| Section | Purpose | Source |
| --- | --- | --- |
| Site header | Brand mark (🧮), title “Equation Explorer”, subtitle “Solve it step by step” | `index.html:465-473` |
| Setup card | Eyebrow, heading, intro, equation form, tip, setup feedback | `index.html:476-491` |
| Session container | Hidden until a session starts; holds reset, toolbox, solving card | `index.html:493-578` |
| Toolbox card | “Token toolbox”, helper text, palette | `index.html:498-507` |
| Solving card | Heading, orientation, step list, active editor, feedback, completion panel | `index.html:509-577` |
| Locked step row | Step number, lock status, normalized equation | `index.html:514-518` |
| Active step row | Step number, side selector, token sides, trash, actions, token toolbar, help text | `index.html:519-559` |
| Completion panel | Celebration mark, heading, praise, verified answer, Solve another | `index.html:562-576` |
| Toast | Fixed success notification | `index.html:581` |

### 1.5 Forms, fields, and validation rules

`[Observed]` There is one form and one field:

- `#equation-form` (`index.html:480`) contains `#equation-input` (`index.html:482`) and `#start-button` (`index.html:484`).
- The input is `type="text"`, `required`, `autocomplete="off"`, `maxlength` is not set in markup, and the placeholder is `e.g. 2y - 3 = 8`.
- Start is disabled while the trimmed draft is empty (`index.html:1422-1424`, `index.html:1743`).
- Validation runs on submit, not on every keystroke (`index.html:2245-2248`, `index.html:2175-2193`). The form's `setCustomValidity` is updated during render but submit is intercepted (`index.html:1741`).
- Blur on a whitespace-only draft shows “Spaces are not an equation. Type something before we begin.” (`index.html:2238-2243`).

`[Observed]` Source grammar and limits (`index.html:586-619`, `index.html:702-772`):

- Maximum raw input length: 512 characters.
- Maximum token count per tokenization: 160.
- Accepted: integers; terminating decimals such as `0.5`; fractions using `/` or `÷`; mixed fractions in setup input such as `5 1/2` (requires whitespace); parentheses; `+`, `-`, Unicode `−`, `*`, `×`, `/`, `÷`; unary signs; variable `y`; implicit multiplication such as `2y` and `2(y + 1)`.
- Rejected: empty side, more than one `=`, unknown symbols, uppercase or other variables, `y²`, nonlinear products such as `y*y`, divisions involving `y` except the narrow direct-reciprocal source form, division by zero, malformed decimals/fractions, unmatched parentheses, missing operands, identities, contradictions, and over-limit input.
- Source solution must be unique. A direct reciprocal source `c / y = d` solves as `y = c / d` with `y ≠ 0`; otherwise linear form solves as `y = (right.constant - left.constant) / (left.coefficient - right.coefficient)` (`index.html:1000-1029`, spec `:134-136`).

`[Observed]` Candidate-row rules (`index.html:1051-1098`, spec `:137-139`):

- Each side must tokenize, parse, and reduce to an affine form `a*y + b`.
- The candidate equation must have a unique solution.
- Its exact rational solution must equal the source solution. Floating point is never used.
- A row identical to the previous step is rejected before analysis; a notation-only equivalent is rejected after canonical display comparison.
- A final form requires the left token list to be exactly one `y` token, the right side to contain no `y`, and the right constant to equal the source solution exactly. `1y`, `(y)`, and algebraically equivalent but structurally different left sides are accepted as steps but are not final.
- No candidate can alter locked history.

### 1.6 Buttons and user interactions

| Control | Behavior | Handler / source |
| --- | --- | --- |
| `#start-button` | Submit source; validate; start or show error | `index.html:2245-2248`, `index.html:2172-2208` |
| `#reset-button` | Return to landing and clear session | `index.html:2249`, `index.html:2210-2230` |
| `#solve-another-button` | Same reset path from completion | `index.html:2250` |
| `#check-button` | Validate active row | `index.html:2251`, `index.html:2135-2170` |
| `#restore-button` | Restore previous step tokens | `index.html:2252`, `index.html:2069-2081` |
| `#flip-sides` | Swap sides | `index.html:2253`, `index.html:2058-2067` |
| `#select-left` / `#select-right` | Choose palette insertion side | `index.html:2254-2255`, `index.html:1810-1816` |
| `#clear-left` / `#clear-right` | Clear one side | `index.html:2261-2267`, `index.html:2044-2056` |
| Palette tokens | Tap to append to selected side; drag to insert at a position | `index.html:2256-2260`, `index.html:1987-2005` |
| Workspace token | Tap to select; drag to reorder/move/delete | `index.html:2261-2275`, `index.html:1884-2005` |
| `#move-left` / `#move-right` | Reorder selected token | `index.html:2276-2277`, `index.html:1859-1869` |
| `#move-to-left` / `#move-to-right` | Move selected token to the other side | `index.html:2278-2279`, `index.html:1871-1882` |
| `#delete-token` | Delete selected token | `index.html:2280`, `index.html:1844-1857` |
| Trash zone | Drop an existing token to delete it | `index.html:540-543`, `index.html:1952-1958` |
| Equation input | Update draft, clear/set feedback | `index.html:2232-2243` |
| Pointer drag | Start/update/finish/cancel custom drag | `index.html:2281-2285`, `index.html:1884-2005` |

### 1.7 Dialogs, overlays, and notifications

`[Observed]` There are no modal dialogs, confirmation prompts, sheets, or browser alerts.

- **Completion panel** is an inline `<section role="region">`, not a modal (`index.html:562-576`).
- **Toast** is a fixed success banner with a 2.6 s display and 200 ms exit (`index.html:361-382`, `index.html:1666-1702`). It is `aria-hidden`; the polite live region carries the accessible message.
- **Confetti** is a fixed, pointer-transparent, `aria-hidden` DOM layer with 42 pieces and a 3.2 s cleanup timer (`index.html:384-420`, `index.html:1558-1600`).
- **Feedback** uses `role="status"`, `aria-live="polite"`, `aria-atomic="true"` on setup and session feedback elements (`index.html:490`, `index.html:561`).
- **Shake** on a rejected active row and **pop** on a newly locked row are CSS animations (`index.html:331-332`, `index.html:405-415`).

### 1.8 API endpoints and external services

`[Observed]` None. There is no `fetch`, `XMLHttpRequest`, WebSocket, remote URL, CDN, analytics script, or external asset. The only `href` is an inline `data:image/svg+xml` favicon (`index.html:7`). `grep` for `http://` and `https://` finds only the SVG namespace.

`[Observed]` The README confirms the intended deployment is static client-side hosting with no backend, database, account system, or API (`README.md:19-20`).

### 1.9 Authentication and session handling

`[Observed]` None. There are no tokens, cookies, authorization headers, OAuth flows, identity providers, login screens, or user records. The word “session” in the source means an in-progress equation-solving session (`state.mode`, `#session`), not an authenticated user session.

`[Observed]` Microsoft Entra ID, MSAL, OAuth, and any other identity provider are **not detected**. No mobile redirect URI, app registration, or token storage requirement exists today.

### 1.10 Browser storage usage

`[Observed]` None. There is no `localStorage`, `sessionStorage`, `indexedDB`, `document.cookie`, Cache API, or File System Access usage. A page reload or tab close destroys the entire session, including locked history and the active row.

### 1.11 Images, icons, fonts, and other assets

`[Observed]` The application ships no image, font, audio, or video files.

| Asset type | Current implementation | Source |
| --- | --- | --- |
| Favicon | Inline SVG data URI: rounded indigo square with white `=` | `index.html:7` |
| Brand mark | Emoji `🧮` | `index.html:467` |
| Tip icon | Emoji `💡` | `index.html:487` |
| Trash icon | Emoji `🗑️` | `index.html:541` |
| Completion mark | Emoji `🎉 ✓` | `index.html:571` |
| Lock / check / restore / flip | Unicode/emoji text `🔒`, `✓`, `↺`, `⇄`, `×` | `index.html:515`, `:1536`, `:546-547`, `:531`, `:536` |
| Typography | System font stack: `ui-rounded`, Nunito, Avenir Next, Avenir, Segoe UI, sans-serif | `index.html:46` |
| Background | CSS radial gradients and warm off-white page color | `index.html:42-45` |
| Shadows | CSS `box-shadow` | `index.html:25`, several component rules |
| Sound | Synthesized Web Audio tones; no audio file | `index.html:1602-1664` |

`[Inferred]` Native needs a real app icon, adaptive icon, splash screen, and (if sound parity is required) bundled audio files. Emoji rendering differs between iOS and Android and should be treated as decorative, not as the sole icon.

### 1.12 Third-party packages and libraries

`[Observed]` None. The app is dependency-free vanilla HTML/CSS/JavaScript. There is no package manager, build system, CDN, test runner, UI framework, state library, router, or math library.

`[Observed]` The math parser and rational arithmetic are project code. `eval` and `Function` are not used; `CLAUDE.md` prohibits them (`CLAUDE.md:12`).

### 1.13 Browser-specific APIs

| API / feature | Use | Source |
| --- | --- | --- |
| DOM querying and mutation | `querySelector`, `createElement`, `documentFragment`, `replaceChildren`, `append`, `remove`, `textContent`, `dataset`, `classList`, `hidden` | `index.html:1380-1414`, `index.html:1450-1549`, `index.html:1731-1808` |
| Pointer Events | `pointerdown`, `pointermove`, `pointerup`, `pointercancel`, `lostpointercapture`, `setPointerCapture`, `releasePointerCapture` | `index.html:1884-2005`, `index.html:2281-2285` |
| Hit testing | `document.elementFromPoint` | `index.html:1893-1912` |
| Layout measurement | `getBoundingClientRect`, `offsetWidth`, `offsetHeight` | `index.html:1805-1808`, `index.html:1893-1912`, `index.html:1921-1933` |
| Scroll and focus | `scrollIntoView`, `focus({ preventScroll })`, `tabindex`, `:focus-visible` | `index.html:60-63`, `index.html:1795-1802`, `index.html:2094-2096` |
| Media queries | `window.matchMedia('(prefers-reduced-motion: reduce)')` | `index.html:1565` |
| Resize observation | `ResizeObserver` with `window.resize` fallback | `index.html:2286-2290` |
| Web Audio | `AudioContext` / `webkitAudioContext`, oscillators, gain ramps | `index.html:1602-1664` |
| CSS animation | `@keyframes`, transitions, CSS custom properties | `index.html:361-420`, `index.html:459-461` |
| Safe-area environment variables | `env(safe-area-inset-top/right)` | `index.html:190`, `index.html:363-364` |
| Reflow hack | `void element.offsetWidth` to restart animation | `index.html:1699`, `index.html:2086` |
| Dev global | `window.runMathSelfChecks`, `console.table` | `index.html:1287-1292` |

`[Observed]` No `history.pushState`, `location`, cookies, `navigator`, `FileReader`, `Blob`, `URL.createObjectURL`, `crypto`, `Notification`, `serviceWorker`, or `SpeechSynthesis` usage.

### 1.14 Responsive behavior

`[Observed]` Mobile-first CSS with a minimum body width of 320px and breakpoints at 600px and 900px (`index.html:38-40`, `index.html:424-457`).

- Below 600px: compact toolbox padding; hide toolbox eyebrow/helper text; smaller toolbox heading and palette gaps.
- At 600px+: setup form becomes two columns; locked rows become a four-column grid.
- At 900px+: wider spacing; active equation becomes a three-column grid with the equals sign in the middle; larger type and token areas.
- The toolbox is `position: sticky` below the top safe area while solving, becomes static on completion, and uses a measured CSS custom property for scroll padding (`index.html:187-197`, `index.html:1805-1808`).
- The page uses `clamp()` for fluid heading/input sizing, flex-wrap for tokens, and `overflow-wrap: anywhere` for long equations.
- `body` intentionally avoids horizontal scrolling; token sides wrap rather than forcing page overflow.

`[Decision]` UX-02 (2026-11-10): **tablet is the primary target and phones must also work.** The 900px desktop-like composition is therefore in scope as the tablet layout, and the 600px phone composition is the minimum-width fallback. Orientation is still unstated. Native has no media queries; breakpoints become `useWindowDimensions()` branches or separate layouts.

### 1.15 Accessibility behavior

`[Observed]` Accessibility is a first-class requirement in the spec (`docs/specs/linear-equation-drag-solver.md:148-153`) and is implemented in markup and behavior.

| Behavior | Implementation | Source |
| --- | --- | --- |
| Landmarks | `header`, `main`, `section`, `article`, `aside` | `index.html:465-579` |
| Headings | `h1`, `h2`, labeled sections | `index.html:469`, `:478`, `:502`, `:511`, `:572` |
| Form labeling | `<label for>`, `aria-describedby`, placeholder example | `index.html:481-483` |
| Invalid state | `aria-invalid` toggled from feedback | `index.html:1742` |
| Live feedback | `role="status"`, `aria-live="polite"`, `aria-atomic="true"`, focusable feedback | `index.html:490`, `:561` |
| Toggle buttons | `aria-pressed` on side selectors | `index.html:524-525`, `:1764-1765` |
| Token names | `aria-label` includes readable token name and side | `index.html:1458`, `:1483` |
| Selected/dragging states | `data-selected`, `data-dragging` plus visible styles | `index.html:1480-1481`, `:266-269` |
| Keyboard alternatives | Tap insertion, token selection, move/reorder/delete buttons; no drag-only path | `index.html:551-557`, `:2256-2280` |
| Focus visibility | 4px focus outline with offset | `index.html:60-63`, `:334-337` |
| Focus management | Focus moves to feedback, affected side, restored side, locked side, completion panel | `index.html:1795-1802`, `:2094-2096`, `:2119-2120`, `:2131-2132` |
| Reduced motion | All transitions/animations reduced to near-zero; confetti skipped entirely | `index.html:459-461`, `:1565` |
| Touch targets | Minimum 44px buttons/tokens | `index.html:54`, `:213`, `:266` |
| Color not sole signal | Text, icons, labels, and shapes accompany color | spec `:100`, `:145` |
| Decorative hiding | Icons, emoji, confetti, toast hidden from assistive technology | `index.html:467`, `:487`, `:541`, `:569-571`, `:581` |
| Screen-reader duplication avoided | Toast is `aria-hidden`; live region carries the message | spec `:145` |

`[Observed]` Several contrast questions deserve an audit rather than an assumption: the focus ring `#f59e0b` on white and the default border `#cbd5e1` on white are below the 3:1 non-text contrast threshold used by WCAG 1.4.11. The existing spec requires high contrast but does not record a contrast audit. This is a migration opportunity to fix or explicitly accept.

### 1.16 Error, loading, empty, and success states

| State | Current behavior | Source |
| --- | --- | --- |
| Loading | None. All work is synchronous; no spinners or skeletons. | whole script |
| Empty input | Start disabled; whitespace-only blur shows an error. | `index.html:1422-1424`, `:2238-2243` |
| Invalid source | Inline setup error with a specific child-friendly message; input retained; focus returns to input. | `index.html:2185-2193` |
| Empty token side | “Drop or add tokens here” placeholder; Check rejects with a friendly empty-side message. | `index.html:1468-1473`, `:594`, `:1043-1049` |
| Malformed candidate | Active row remains editable; error feedback, red side/row styling, shake, focus on affected side. | `index.html:2090-2097`, `:2157-2160` |
| Inequivalent candidate | “That changes the answer for y. Try doing the same operation to both sides!” | `index.html:595`, `:2161-2164` |
| Same as previous | Friendly rejection; no history change. | `index.html:596`, `:2145-2148`, `:2165-2168` |
| Accepted non-final | Row locks, confirmation toast, success chime, new prefilled active row. | `index.html:2125-2133` |
| Completed | Final row locks, completion panel (“Good job”, answer, “Solve another!”), celebration chime; optional confetti. | `index.html:2112-2123` |
| Audio unavailable | Silent failure; visual completion still works. | `index.html:1597-1645` |
| Reduced motion | Confetti skipped; animations near-zero. | `index.html:1565`, `:459-461` |

### 1.17 State model

`[Observed]` A single plain object holds all application state (`index.html:1362-1378`):

- `mode`: `landing` | `started` | `completed`
- `draftEquation`, `submittedEquation`
- `sourceEquation` (analysis result, source tokens, solution)
- `lockedRows` (immutable accepted rows with `displayEquation`, `isFinal`, token copies)
- `activeRow` (mutable token sides)
- `selectedSide`, `selectedToken`
- `drag` (pointer drag session)
- `feedback` (kind, message, side, scope)
- `lastAcceptedCheck` (duplicate guard signature and timestamp)
- `sourceStepSignature`, `sourceTokens` (restore baseline)
- `animateLockedRowId`, `celebrationPlayed`

`[Observed]` Rendering is a full state-to-DOM projection (`render()`, `index.html:1731-1803`). Business logic never reads equation text back from the DOM; the source equation and exact solution remain in state (spec `:117`). The only DOM-derived business values are token IDs and symbols, which are stored in state as token objects.

`[Observed]` Module layout inside the script:

| Area | Lines |
| --- | --- |
| Constants, palette, messages | `index.html:586-619` |
| Exact rational arithmetic | `index.html:621-700` |
| Tokenizer | `index.html:702-772` |
| Recursive-descent parser | `index.html:774-892` |
| Affine reduction and formatting | `index.html:894-998` |
| Source analysis and solve | `index.html:1000-1029` |
| Candidate analysis and final-form predicate | `index.html:1031-1098` |
| Self-check corpus | `index.html:1104-1292` |
| Workspace token/row helpers | `index.html:1294-1360` |
| State and DOM references | `index.html:1362-1414` |
| Rendering | `index.html:1450-1808` |
| Editing commands | `index.html:1810-1882` |
| Pointer drag | `index.html:1884-2005` |
| Check / lock / start / reset | `index.html:2007-2230` |
| Event wiring and init | `index.html:2232-2292` |

### 1.18 Documentation and source inconsistencies

`[Observed]` These are material to requirements confirmation:

1. **README is stale.** It says “Milestone 0 … is implemented” and “The current app is a UI-only preview” and “Milestone 1 … is specified but not implemented” (`README.md:5-8`). The spec header says the app is implemented through M4 with automated verification complete (`docs/specs/linear-equation-drag-solver.md:3`), and `index.html` contains the full engine, workspace, history, and celebration. The README should not be used as a feature inventory without this correction.
2. **Variable naming drift.** The app uses `y` (`index.html:597-602`, `:757-760`, `:512`), while several spec and task examples use `x` (for example `docs/specs/linear-equation-drag-solver.md:184-196` and `tasks/04-todo.md`). The recent commit “use y” suggests the code is the intended product but the documentation has not been fully updated.
3. **Human acceptance is still open, but no longer blocks parity.** `docs/specs/linear-equation-drag-solver.md:3` says M3's human acceptance gate and M4's human release gate remain open; `tasks/04-todo.md` marks Task 7 (human release gate) unchecked and Task 0 (M3 acceptance prerequisite) unchecked. PB-03 (2026-11-10) accepts the current behavior as the parity baseline anyway; a human still has not signed off the visible flow, screen-reader behavior, or browser matrix, and that evidence must be captured for UAT (TST-05).
4. **No automated test suite exists in the repository.** The 70-case `runMathSelfChecks()` corpus is executed manually in a browser DevTools console (`index.html:1104-1292`, `tasks/04-todo.md`). It is excellent raw material for Jest but is not currently a CI gate.
5. **Potential contrast gaps.** As noted in §1.15, the focus ring and default borders may not meet WCAG non-text contrast. This is a confirmed source value, not a confirmed failure; it needs an audit.
6. **“No persistence” is now a product decision.** The web app loses everything on reload by design. PB-05 (2026-11-10) confirms no persistence for v1, so process death losing the session is accepted behavior, not a migration bug. Back navigation preserves the in-memory session (UX-08); Reset is the explicit discard path.

---

## 2. Reusability assessment

Classification key:

- **Directly reusable:** can move with little or no change.
- **Reusable after adaptation:** logic or content is sound but must be transcribed, typed, or re-hosted.
- **Must be rewritten:** depends on the browser DOM/CSS/event model.
- **Obsolete or unnecessary on mobile:** browser-only or not part of the mobile product.
- **Requires further investigation:** feasibility or product intent is unresolved.

### 2.1 Business rules — reusable after adaptation

`[Observed]` The math rules are pure functions over strings, token arrays, and exact rationals. They do not read the DOM, use timers, or depend on browser APIs (`index.html:621-1098`). They can be transcribed into a TypeScript module with the same function boundaries and behavior.

Reusable rules include:

- Exact rational normalization, arithmetic, comparison, and zero checks (`index.html:629-684`).
- Number/decimal/mixed-fraction parsing (`index.html:690-700`, `:718-754`).
- Operator precedence, unary signs, implicit multiplication, parentheses (`index.html:774-892`).
- Affine reduction and nonlinear/unsupported-rational detection (`index.html:898-932`).
- Direct-reciprocal source analysis (`index.html:934-969`).
- Canonical display formatting (`index.html:971-998`).
- Source solve and unique-solution checks (`index.html:1000-1029`).
- Candidate-side analysis, solution equivalence, and final-form predicate (`index.html:1031-1098`).
- Same-as-previous and duplicate-check rules (`index.html:2011-2027`, `:2135-2170`).
- Workspace token/row operations: insert, remove, move, copy, serialize (`index.html:1294-1360`).

Why adaptation is required: the code is JavaScript inside an HTML file; the project requires TypeScript strict mode and a native module boundary. `BigInt` remains the correct representation. The 70-case self-check corpus should become Jest tests, and the browser-only `window.runMathSelfChecks` export should disappear.

### 2.2 Data models — reusable after adaptation

`[Observed]` The implicit models are simple:

- `Rational = { numerator: bigint, denominator: bigint }`, normalized with a positive denominator and GCD reduction.
- AST nodes: `number`, `variable`, `group`, `unary`, `binary`.
- Affine value: `{ x: Rational, constant: Rational }`.
- Workspace token: `{ id, symbol, label }`.
- Active row: `{ id, leftTokens, rightTokens }`.
- Locked row: active row plus `displayEquation` and `isFinal`.
- Session state as listed in §1.17.

Why adaptation: no formal TypeScript types exist; only JSDoc and conventions. Add discriminated unions for parse results, errors, verdicts, and reducer actions. IDs should remain session-local and deterministic for tests.

### 2.3 Type definitions — must be created

`[Observed]` The source has JSDoc for a few functions (`index.html:640`, `:1051-1061`, `:1088-1091`) but no TypeScript. The migration needs explicit types for:

- Token symbols and AST nodes.
- `MathError` codes and messages.
- `Result<T, MathError>`-style success/error unions.
- `AnalyzedSource` (`linear` or `direct-reciprocal`).
- `CandidateVerdict` (`error`, `inequivalent`, `accepted`).
- Session state, actions, and events.
- Component props.

Why this is a rewrite rather than reuse: there is nothing to copy; the types are a new safety layer around the ported logic.

### 2.4 Validation logic — reusable after adaptation

`[Observed]` Validation is deterministic and side-effect-free. It produces error codes that map to child-friendly messages (`index.html:603-619`). It rejects unsupported or unsafe input without `eval`. This is the most valuable reusable asset.

Adaptation work:

- Port the tokenizer/parser/reducer exactly.
- Preserve error codes, messages, and source offsets where useful.
- Keep the 512-character and 160-token limits.
- Keep the direct-reciprocal exception narrow; do not generalize rational equations.
- Add tests for every current self-check case and for the final release checklist examples.
- Do not duplicate rules in UI components; the reducer/engine owns them.

### 2.5 API clients — not applicable

`[Observed]` There are no API clients, endpoints, or network calls. Nothing to reuse. Do not introduce React Query, Axios, or a generated client in v1.

### 2.6 Authentication logic — not applicable

`[Observed]` There is no authentication, session token, identity provider, or user model. Nothing to reuse. Do not introduce SecureStore, OAuth, or Entra ID unless a future requirement explicitly adds accounts.

### 2.7 State management — reusable after adaptation

`[Observed]` The current state machine is sound: `landing → started → completed`, with immutable locked rows and a mutable active row. The render function is a pure projection of state. The business rules for transitions are clear.

Adaptation work:

- Replace the mutable `const state = { ... }` object and imperative handlers with a `useReducer` reducer and typed actions.
- Keep locked rows immutable by copying tokens when locking.
- Keep the source equation and exact solution in state; never reconstruct them from rendered text.
- Move drag state out of the reducer into gesture shared values/local refs, because it updates per frame.
- Keep effect triggers (toast, chime, confetti, haptics, announcements) outside the reducer.
- No Redux, Zustand, Jotai, MobX, or context-per-value. One session reducer plus screen-local state is sufficient.

### 2.8 Utility functions — mostly reusable after adaptation

| Utility | Reuse | Notes |
| --- | --- | --- |
| `tokenLabel`, `createWorkspaceToken` | Reusable after adaptation | Replace incrementing module counters with a session-scoped ID source; keep labels. |
| `tokensFromDisplay`, `serializeTokens` | Reusable after adaptation | Pure; depends on the tokenizer. |
| `insertTokenInRow`, `removeTokenFromRow`, `moveTokenInRow` | Reusable after adaptation | Immutable array operations; ideal reducer helpers. |
| `rowSignature` | Directly reusable | Simple string join; useful for duplicate detection. |
| `appendVariableHighlightedText` | Must be rewritten | DOM text-node manipulation; becomes nested `<Text>` rendering. |
| `updateStickyToolboxHeight` | Obsolete | No sticky CSS; replace with layout design. |
| `launchConfetti`, `clearConfettiLayer` | Must be rewritten | DOM/CSS particles; use Reanimated overlay. |
| `scheduleToneSequence`, chime functions | Must be rewritten | Web Audio synthesis; use bundled audio assets via `expo-audio`. Sound is required (PB-06), so haptics alone are not a substitute. |
| `showToast`, `hideToast`, `clearToast` | Must be rewritten | DOM fixed banner; use a native animated overlay. |
| `runMathSelfChecks` | Reusable after adaptation | Convert to Jest tests; remove the browser global. |

### 2.9 HTML structure — must be rewritten

`[Observed]` The semantic shell is useful as an information architecture, not as code. `header`, `main`, cards, sections, articles, fieldset/legend, and the step rows map to native `View`/`Text`/`Pressable` composition, but every element and attribute must be recreated.

### 2.10 CSS and responsive layout — reusable after adaptation (design intent), must be rewritten (implementation)

`[Observed]` The visual language is a strong asset:

- Warm off-white page, indigo primary, teal/coral accents, green success, red error, red variable.
- Large rounded cards, 44px+ touch targets, chunky buttons with bottom shadow.
- Clear hierarchy, generous spacing, child-friendly typography.
- Explicit selected, error, drop-target, disabled, and focus states.
- Reduced-motion support.

Reusable after adaptation: color values, radii, spacing intent, type hierarchy, state styling, motion intent.

Must be rewritten: CSS rules, cascade, `clamp()`, media queries, gradients, sticky/fixed positioning, `box-shadow`, `outline`, keyframes, transitions, `::before`, `env(safe-area-inset-*)`, `overflow-wrap`, and `touch-action`.

### 2.11 Browser event handling — must be rewritten

`[Observed]` All interaction is wired through DOM events: `submit`, `click`, `input`, `blur`, and document-level pointer events (`index.html:2232-2292`). Drag uses a hand-rolled pointer state machine with capture, a fixed preview, `elementFromPoint`, and drop-index calculation (`index.html:1884-2005`).

Native replacement:

- Buttons become `Pressable` with `onPress`.
- Input becomes `TextInput` with `onChangeText`/`onSubmitEditing`.
- Drag becomes Gesture Handler `Gesture.Pan()` plus Reanimated shared values, measured rects, and a native overlay.
- Hover is removed; pressed/focused states replace it.
- No `preventDefault`, pointer capture, or event bubbling assumptions.

### 2.12 DOM manipulation — must be rewritten

`[Observed]` Rendering is imperative: query selectors, create elements, replace children, set datasets, toggle hidden, set text content, append fragments, manage focus, and scroll into view (`index.html:1380-1414`, `index.html:1450-1808`, `index.html:2090-2132`). React rendering replaces all of it.

Key mappings:

| DOM operation | Native replacement |
| --- | --- |
| `querySelector` / refs | React refs, props, state |
| `createElement` / `replaceChildren` | JSX and declarative lists |
| `textContent` | `<Text>` children |
| `dataset` / `classList` | props and style arrays |
| `hidden` | conditional render or `display: 'none'` |
| `append` / `remove` | React tree |
| `focus()` / `scrollIntoView()` | refs, `ScrollView.scrollTo`, accessibility focus |
| `offsetWidth` reflow hack | Reanimated animation restart |
| `elementFromPoint` | measured rects and hit testing |

### 2.13 Browser storage — not applicable

`[Observed]` No storage is used. Nothing to migrate. If session resume is added later, it is a new feature, not a port.

### 2.14 Third-party libraries — not applicable

`[Observed]` None. There are no browser SDKs to swap. The migration adds React Native libraries for gestures, animation, navigation, audio, and haptics, but no legacy web library blocks the move.

### 2.15 Assets — must be recreated

| Asset | Reuse | Action |
| --- | --- | --- |
| Inline SVG favicon | Concept only | Create app icon, adaptive icon, splash icon. |
| Emoji/Unicode icons | Directly usable but platform-dependent | Keep as decorative `Text` with accessible labels, or replace with vector icons. |
| System font stack | Adapted | Use platform system fonts; no custom font required. |
| Web Audio tones | Must be recreated | Bundle short original audio files (required by PB-06); haptics are an additional cue, not a replacement. |
| Confetti colors | Directly reusable | Use the six-color palette. |
| CSS gradients | Adapted | Simplify to solid background plus decorative shapes, or add a gradient library. |

### 2.16 Documentation and test corpus

- **Directly reusable:** product spec, milestone specs, manual release checklist, error messages, palette labels, 70 self-check cases.
- **Reusable after adaptation:** README (correct staleness first), task lists (milestone structure), acceptance criteria (translate browser-specific steps to device steps).
- **Must be rewritten:** browser-specific verification commands, DOM-oriented demo scripts, the `window.runMathSelfChecks` invocation.

### 2.17 Reusability summary

| Area | Classification | Primary reason |
| --- | --- | --- |
| Exact math engine | Reusable after adaptation | Pure logic, no DOM; needs TypeScript port and Jest |
| Validation rules and messages | Reusable after adaptation | Deterministic, child-friendly, well-scoped |
| Workspace token helpers | Reusable after adaptation | Immutable array logic; depends on engine |
| State machine semantics | Reusable after adaptation | Clear phases and invariants; needs reducer |
| HTML structure | Must be rewritten | DOM-specific |
| CSS implementation | Must be rewritten | No CSS in React Native |
| Responsive layout | Reusable after adaptation | Breakpoint intent, new native layout |
| Pointer drag | Must be rewritten | Pointer Events and DOM hit testing |
| DOM rendering | Must be rewritten | Imperative DOM vs React |
| Browser storage / API / auth | Not applicable | Not present |
| Third-party libraries | Not applicable | None present |
| Assets | Must be recreated | App icon, splash, audio; emoji usable decoratively |
| Manual checklist | Reusable after adaptation | Needs device/screen-reader translation |

---

## 3. Web-to-native mapping

Complexity: **Low** = straightforward API swap; **Medium** = design and implementation work with known patterns; **High** = novel interaction, platform risk, or parity uncertainty.

| Existing HTML section or behavior | Source | Proposed React Native screen or component | Proposed React Native API or package | Complexity | Risks / unresolved questions |
| --- | --- | --- | --- | --- | --- |
| Page shell: `header`, `main`, cards, sections | `index.html:464-579` | `SetupScreen`, `SolverScreen`; `Screen`, `Card` shared components | `View`, `ScrollView`, `SafeAreaView` from `react-native-safe-area-context` | Low | Native has no landmarks; use heading roles and labels. |
| Headings and paragraph text | `index.html:469-479`, `:500-512` | `Heading`, `BodyText`, `Eyebrow` shared components | `Text` with `accessibilityRole="header"` | Low | Font scaling at 200% must wrap; no `clamp()`. |
| Brand mark and decorative icons | `index.html:467`, `:487`, `:541`, `:571` | `BrandMark`, `IconText` | `Text` emoji or `@expo/vector-icons` | Low | Emoji rendering differs; icon must not be the only label. |
| Equation input | `index.html:480-485` | `EquationForm` / `EquationInput` | `TextInput`, `KeyboardAvoidingView` | Low–Medium | Mobile keyboard covers content; smart punctuation and auto-correct must be disabled; return key starts. |
| Start button, reset, Solve another | `index.html:484`, `:495`, `:575` | `PrimaryButton`, `SecondaryButton` | `Pressable`, `Text` | Low | 44pt/48dp targets; disabled and pressed states. |
| Hover styles on buttons and tokens | `index.html:138`, `:183`, `:226`, `:264`, `:316` | Press/focus state styling | `Pressable` `style={({pressed}) => ...}`, `onFocus`/`onBlur` | Low | No hover on touch; do not port hover-only affordances. |
| Disabled states | `index.html:139`, `:265`, `:317`, `:324` | Disabled visual + accessibility state | `Pressable disabled`, `accessibilityState={{ disabled }}` | Low | Ensure disabled text remains readable; web uses opacity 1 intentionally. |
| Token palette (18 wrapped buttons) | `index.html:506`, `:597-602`, `:1450-1462` | `TokenPalette` in a toolbox: bottom-docked on phones, side- or bottom-docked on tablets | `View` `flexWrap`, `Pressable`, optionally horizontal `ScrollView` | Medium | Tablet-first ergonomics; palette must not cover active content; 18 targets need clear grouping. |
| Side selector (`Left side` / `Right side`) | `index.html:522-526` | `SideSelector` segmented control | `Pressable` + `accessibilityState={{ selected }}` | Low | Use `accessibilityRole="button"`; selected state must be announced. |
| Token side drop zone | `index.html:528-538` | `TokenSide` | `View` with `onLayout`, measured rect, `flexWrap` | Medium | Drop target must work inside a `ScrollView`; empty placeholder. |
| Workspace token | `index.html:1474-1485` | `WorkspaceToken` | `Pressable` + `GestureDetector` | Medium–High | Tap selects; immediate drag on touch-move; selected/dragging states; accessible label includes token name and side. |
| Pointer drag with preview | `index.html:1884-1936` | `useTokenDrag` + `DragOverlay` | `react-native-gesture-handler` `Gesture.Pan()` + `react-native-reanimated` shared values | High | No `elementFromPoint`; must measure token rects; scroll gesture conflict; immediate-pan versus scroll threshold (UX-06). |
| Drop-index calculation | `index.html:1893-1912` | Pure `computeInsertionIndex` + drop-target state | Reanimated worklet or JS on gesture end | High | Wrapped layout makes index calculation non-trivial; simplify to side-level highlight, compute index on drop. |
| Trash zone | `index.html:539-543` | `TrashDropZone` | `View` with measured rect; drop detection | Medium | Must be reachable by thumb; tap deletion remains the accessible path. |
| Clear side (×) | `index.html:531`, `:536` | `ClearSideButton` | `Pressable` | Low | Overlay positioning; 44px target; disabled when empty. |
| Move/reorder/delete toolbar | `index.html:551-556` | `TokenToolbar` | `Pressable` buttons | Low | This is the primary accessible alternative to drag; must be prominent. |
| Restore previous step / Flip sides / Check | `index.html:545-548` | `EquationActions` | `Pressable` | Low | Disabled logic must match reducer state. |
| Locked history rows | `index.html:514-518`, `:1519-1549` | `LockedHistory` | `View` mapped inside `ScrollView` | Low–Medium | History is short; `FlatList` is unnecessary. Variable-height wrapped equations favor `ScrollView`. |
| Active equation display with `y` highlighted | `index.html:528-538`, `:1506-1517` | `EquationText` | Nested `Text` with color span | Low | Parse display string into segments or render token spans; keep accessible label. |
| Tables / long lists | none | Not applicable | — | — | No tables; history and palette are small. Do not add virtualization prematurely. |
| Modals / dialogs | none | Completion is inline; optional `Modal` only if product wants a full-screen celebration | RN `Modal` or Expo Router `presentation: 'modal'` | Low | Current app has no modal; inline panel preserves history visibility. |
| Navigation | none; state-driven `landing` / `started` / `completed` | Single Expo Router route switching Setup/Solver by phase; completion inline | `expo-router` | Medium | Back during a session is intercepted to preserve it (UX-08); no back-stack. |
| File selection / camera | none | Not applicable | — | — | No permissions or pickers needed. |
| `localStorage` / `sessionStorage` | none | In-memory state; optional future non-sensitive preferences | `AsyncStorage` or `expo-sqlite/localStorage`; `expo-secure-store` only for secrets | Low | Do not store the session in plain storage by default; no secrets exist. |
| Cookies | none | Not applicable | — | — | No cookie jar or auth headers. |
| Browser history / back button | none | Android back from Solver returns to Setup; optional confirmation | Expo Router `useRouter`, `usePreventRemove` | Medium | Losing an in-progress session on back is poor UX; decide confirm vs discard. |
| Responsive CSS / media queries | `index.html:424-457` | `useWindowDimensions()` breakpoint branches; tablet-first layout with a phone layout | `useWindowDimensions`, `Platform` | Medium | Tablet is primary and phones must work (UX-02); orientation still unresolved. |
| `clamp()`, `rem`, `vh` | `index.html:97`, `:109`, `:111`, `:355` | Fixed dp scale with dynamic type support | `StyleSheet`, `PixelRatio`, `useWindowDimensions` | Medium | Avoid hard-coded heights; text must scale. |
| Sticky toolbox | `index.html:187-197`, `:1805-1808` | Bottom toolbox on phones; side panel or bottom dock on tablets | `View` layout, `ScrollView` content padding, safe-area insets | Medium | Web sticky has no native equivalent; choose per form factor, not thumb reach alone. |
| CSS radial gradients | `index.html:42-45` | Solid background + decorative absolute circles | `View` with `borderRadius`; optional `expo-linear-gradient` | Low–Medium | Exact radial gradient needs SVG/Skia or a dependency; simplify first. |
| CSS box shadows | `index.html:25`, `:94`, `:135`, `:372` | `Card` shadow tokens | iOS shadow props; Android `elevation` | Low | Shadows differ by platform; test contrast/elevation. |
| CSS transitions and keyframes | `index.html:331-332`, `:361-420`, `:459-461` | Reanimated entering/exiting and timing | `react-native-reanimated` | Medium | No CSS; reduced motion must be explicit. |
| Shake on rejection | `index.html:331`, `:2083-2088` | `useAnimatedStyle` translateX sequence | Reanimated | Low–Medium | Respect reduced motion; do not block feedback. |
| Confetti | `index.html:384-420`, `:1558-1600` | `ConfettiOverlay` | Reanimated absolutely positioned pieces | Medium | 42 pieces, cleanup, reduced motion, no layout overflow. |
| Web Audio chimes | `index.html:1602-1664` | `useCelebrationAudio` with bundled assets | `expo-audio` | Medium | Required by PB-06, so audio files are in scope; Web Audio synthesis cannot port; the iOS silent switch mutes and Android follows media volume (NAT-10); asset licensing remains open. |
| `prefers-reduced-motion` | `index.html:1565`, `:459-461` | `useReducedMotion` | Reanimated `useReducedMotion` / `AccessibilityInfo` | Low | Ensure confetti and shake are skipped/reduced, not just shortened. |
| `aria-live` / `role="status"` | `index.html:490`, `:561` | `FeedbackText` + announcements | `accessibilityLiveRegion` (Android), `AccessibilityInfo.announceForAccessibility` | Medium | Avoid duplicate announcements from toast; iOS/Android behavior differs. |
| `aria-pressed`, `aria-invalid`, `aria-describedby` | `index.html:524-525`, `:1742`, `:482` | Accessibility props | `accessibilityState`, `accessibilityHint`, `accessibilityLabel` | Medium | RN has no direct `aria-invalid`; use error text/hint. |
| Focus moves (`focus()`, `scrollIntoView`) | `index.html:1795-1802`, `:2090-2132` | Accessibility focus + scroll refs | refs, `findNodeHandle`, `AccessibilityInfo.setAccessibilityFocus`, `ScrollView.scrollTo` | Medium–High | Focus APIs differ by platform and New Architecture; verify on device. |
| Keyboard interaction / tab order | `index.html:2256-2280` | Toolbar buttons + Pressable focus; external keyboard | RN focus system, `accessibilityActions` | High | No reliable tab order on mobile; hardware keyboard support must be investigated. |
| Downloads / uploads / share | none | Not applicable | — | — | No file or share requirements. |
| Toast | `index.html:361-382`, `:1666-1702` | `FeedbackToast` | Reanimated overlay; `accessibilityElementsHidden` | Low–Medium | Must not double-announce; auto-dismiss and cleanup on reset. |
| Favicon | `index.html:7` | App icon, adaptive icon, splash | `app.json` / `app.config.ts`, asset files | Low | New artwork required; 1024px PNG, Android adaptive icon. |
| `window.runMathSelfChecks` | `index.html:1104-1292` | Jest suite | `jest-expo` | Low | Convert 70 assertions; keep as regression gate. |
| `ResizeObserver` / `offsetHeight` | `index.html:1805-1808`, `:2286-2290` | `onLayout` if measurement is needed | `onLayout`, `measureInWindow` | Low | Layout redesign should remove the need. |
| `document` / `window` globals | throughout | RN runtime | none | Medium | Any remaining web-only code must be removed; no DOM polyfill. |

---

## 4. Target-platform recommendation

### 4.1 Options compared

| Criterion | Expo managed / CNG (recommended) | Expo prebuild with committed native projects | Bare React Native CLI |
| --- | --- | --- | --- |
| Existing browser dependencies | None that need a browser; all app APIs have Expo/RN equivalents | Same | Same |
| Required native capabilities | Gestures, animation, audio, haptics, safe area, icons — all provided by Expo SDK | Same, plus hand-edited native config if needed | Same, but assembled manually |
| Authentication | None today; future `expo-auth-session` / MSAL-compatible dev build | Same, with native project control | Same, with most native control |
| Push notifications | Not needed; `expo-notifications` available later | Same | Manual APNs/FCM setup |
| Camera / file access | Not needed | Not needed | Not needed |
| Background processing | None | None | None |
| Offline support | Bundled JS and assets work offline; no OTA service assumed | Same | Same |
| Custom native modules | Not needed; Expo Modules API/config plugins if ever needed | Easier to add native code directly | Full control, highest maintenance |
| App Store / Google Play | Local Xcode archive + Play Console upload (DEP-03) | Same, with native projects in repo | Manual Xcode/Gradle + fastlane |
| Over-the-air updates | Not used in v1 (DEP-03) | Not used in v1 | Requires a third-party/self-hosted solution |
| CI/CD | GitHub Actions for tests; local builds/signing | Same | Custom pipelines |
| Long-term maintenance | SDK upgrades; native projects generated on demand | Native project drift; merge conflicts after prebuild | Highest: RN upgrades, native toolchains, signing |
| Team learning effort | Lowest | Medium | Highest |
| New Architecture | Default on current Expo SDKs | Configurable | Configurable |

### 4.2 Recommendation

`[Decision]` **React Native + TypeScript + Expo modules + Development Builds + Maestro + Claude Code** (2026-11-10). The delivery approach remains **Expo managed / Continuous Native Generation (CNG), no committed `ios/` or `android/` directories.**
`[Decision]` Development builds are the testing runtime (not Expo Go). Maestro is the E2E tool. Claude Code is the implementation workflow; it does not add a runtime dependency.
`[Decision]` **Build and submit locally with Xcode and Gradle; do not use EAS Build or EAS Submit** (DEP-03). Boris owns the Apple Developer, Google Play, and Expo accounts (DEP-01).

Reasons:

1. **No custom native requirement.** The app needs no camera, file picker, push, background task, Bluetooth, payments, or bespoke native SDK. Every required capability is covered by Expo SDK modules: Gesture Handler, Reanimated, safe-area-context, screens, expo-audio, expo-haptics, expo-router.
2. **The app is small and offline.** Expo's managed workflow keeps the project to TypeScript, app config, and assets, which matches the product's dependency-free spirit.
3. **Store delivery is local and already owned.** Xcode archives/signing and Gradle/Play Console uploads are the delivery path (DEP-03); Boris owns the accounts (DEP-01). No cloud build service or budget is required.
4. **New Architecture is the default.** Reanimated 4 requires it; Expo's current SDK supports it. This aligns with the target requirement.
5. **CNG is not a cage.** If a native config or module later requires it, `npx expo prebuild` generates native projects without rewriting the JavaScript/TypeScript app. The recommendation can change without a rewrite.
6. **Testability.** Expo dev builds, Jest, RNTL, and Maestro work normally; tests run locally and in CI without a build service.

### 4.3 Trade-offs

- You rely on Expo's SDK release cadence. Mitigation: pin versions, keep an upgrade budget, and use GitHub Actions for tests.
- Debugging a native issue is less direct without committed native projects. Mitigation: use a development build, `npx expo prebuild --clean` when needed, and keep native config in `app.config.ts`/plugins.
- Local builds require the Xcode/Android SDK toolchain and signing credentials on the build machine, and they occupy that machine for the build. `[Open]` Release cadence and who cuts releases.
- Some custom native SDKs may require a config plugin or prebuild. `[Open]` This is not expected for v1.
- OTA updates would require EAS Update or a self-hosted service, which the local-build decision does not assume. `[Recommendation]` No OTA in v1; ship through store review.

### 4.4 Rejected alternatives

| Alternative | Why rejected |
| --- | --- |
| **Expo prebuild with committed native projects** | No custom native code exists; committed native projects add maintenance, upgrade conflicts, and signing overhead with no current benefit. Use only if a native config/module forces it. |
| **Bare React Native CLI** | More setup and long-term maintenance; no capability advantage for this app; loses Expo's managed modules and config plugins. |
| **WebView or DOM-component wrapper** | Explicitly excluded by the target requirements and by the goal of a real native app. A 2 MB web runtime and web interaction model would not satisfy the native accessibility/performance goals. |
| **React Native Web / PWA** | Not the requested target; a PWA would not deliver App Store/Google Play presence or native gestures/audio. |
| **Native Swift/Kotlin rewrite** | Two codebases for a small app; React Native is appropriate because the business logic is pure and the UI is simple. |

### 4.5 Conditions that would change the recommendation

`[Open]` Switch to Expo prebuild (with native projects) or bare React Native if any of these become true:

- A required native SDK has no Expo config plugin and cannot be integrated through Expo Modules.
- The product needs a custom app extension, widget, watch app, App Clip, or share extension.
- The product needs enterprise/brownfield embedding inside an existing native app.
- Conditional-access authentication requires a broker integration that cannot be configured in a dev build.
- A specific RN version or native toolchain is mandated by an external constraint.
- Local builds are blocked by a policy that requires a managed cloud build service (then revisit EAS Build).

Until then, managed/CNG is the lower-risk, lower-maintenance choice.

---

## 5. Security and storage

### 5.1 Sensitive information inventory

| Data type | Present? | Evidence | Sensitivity | Mobile recommendation |
| --- | --- | --- | --- | --- |
| Access tokens | No | No auth code or token handling in `index.html` | High if introduced | `expo-secure-store` (Keychain / Android Keystore) only; never AsyncStorage |
| Refresh tokens | No | No auth code | High if introduced | `expo-secure-store`; never log or persist in plain storage |
| User identifiers | No | No accounts, profiles, or IDs | High if introduced | Do not collect unless required; document retention and consent |
| Personal information | No | Only free-text equation input | Potential (a child could type anything) | Keep on device; do not log or transmit; treat as learner content |
| API keys | No | No network clients or config | High if introduced | Never embed long-lived secrets in the app bundle; use a backend/BFF |
| Cookies | No | No `document.cookie` | N/A | No native cookie jar by default; use bearer tokens if auth is added |
| `localStorage` / `sessionStorage` | No | No storage API calls | N/A | In-memory only in v1; non-sensitive preferences may use AsyncStorage or `expo-sqlite` localStorage |
| URL query parameters | No | No routing or URL state | N/A | Do not put learner data or tokens in deep-link params |
| Log output | Minimal | `console.table` in the dev self-check only | Low | Remove/guard in production; never log raw input, tokens, or state |
| Equation history | In memory | `lockedRows` in state | Low–Medium (learner work) | In-memory v1, confirmed by PB-05; not persisted, so no retention or deletion policy |
| Audio / microphone | No | Web Audio synthesis only; no recording | N/A | Playback needs no permission; do not add microphone permission |
| Analytics events | No | No analytics | N/A | No analytics by default; if added, use a privacy-reviewed native SDK and disclose it |
| Crash reports | No | No crash reporting | Potential | If added, scrub input/state; disclose in privacy policy |

### 5.2 Current storage posture

`[Observed]` The app is entirely in-memory. There is no persistence, no cookie, no token, no keychain, no database, and no file storage. Closing or reloading loses the session. This is the simplest and safest posture and should be the default for the React Native v1.

`[Decision]` **v1 storage decision: in-memory only.** Confirmed by PB-05 (2026-11-10): session persistence is not required.

- Keep the session in a React reducer/context for the lifetime of the app process.
- Do not add AsyncStorage, SQLite, or SecureStore to v1.
- If the app is backgrounded and resumed, React state normally survives unless the OS kills the process; process death loses the session and that is accepted.
- Back preserves the in-memory session (UX-08); Reset is the explicit discard path. Do not add a resume prompt.
- If session resume is added later, persist only the raw source input and token symbols. Rehydrate by re-running the engine; never persist `BigInt` directly. Encode rationals as decimal strings if needed.

### 5.3 Storage recommendations for future work

| Use case | Recommended storage | Why |
| --- | --- | --- |
| Non-sensitive preferences (sound on/off, haptics, last-used equation) | `AsyncStorage` or `expo-sqlite` localStorage polyfill | Simple key-value; unencrypted is acceptable for non-sensitive data |
| Larger local history / multiple sessions | `expo-sqlite` | Relational queries, transactions, size |
| Authentication tokens / secrets | `expo-secure-store` | iOS Keychain, Android Keystore-backed encryption |
| One-shot secrets / session-only data | In-memory | No persistence, no cleanup burden |
| Files (exports, reports) | `expo-file-system` + share sheet | Only if the product adds export/share; not present today |

`[Recommendation]` Do **not** store authentication tokens, refresh tokens, personal information, or API keys in plain AsyncStorage. AsyncStorage is unencrypted, included in device backups in some configurations, and readable on rooted/jailbroken devices. Use SecureStore for secrets. The current app has no secrets, so SecureStore is not needed yet.

`[Inferred]` The Expo skill's storage guidance prefers the `expo-sqlite/localStorage` polyfill over AsyncStorage for simple key-value data. Either is acceptable for non-sensitive preferences; choose one during implementation and document it. The deciding factor should be whether the project already uses SQLite or wants a minimal dependency. For v1, neither is needed.

### 5.4 Authentication and identity providers

`[Observed]` Microsoft Entra ID, MSAL, OAuth, OpenID Connect, and any other identity provider are **not detected**. There is no application registration, client ID, tenant ID, redirect URI, scope, token endpoint, or logout flow.

`[Decision]` No authentication in v1 (SEC-01, 2026-11-10): the app is standalone and needs no accounts or sign-in. If a future product requirement adds accounts, document and decide the following before implementation:

- Identity provider and tenant configuration.
- Application registration type (public client/native app).
- Redirect URI scheme: typically `msauth.<bundle-id>://auth` for MSAL, registered in the app config and the identity provider.
- PKCE authorization-code flow; no client secret in the app.
- Token storage in `expo-secure-store`; no tokens in AsyncStorage, logs, URLs, or crash reports.
- Access-token lifetime, refresh behavior, revocation, logout, and account switching.
- Conditional Access / MFA / broker requirements; these may force a development build and native configuration.
- iOS associated domains / Android package and signature hashes.
- Privacy policy, consent, and children's data requirements.
- Whether learner progress remains local or is synced to a backend.

This is a conditional future section, not a current dependency.

### 5.5 Logging, analytics, and privacy

`[Observed]` The only logging is `console.table(cases)` inside the developer self-check (`index.html:1288`). There is no analytics, crash reporting, session replay, or telemetry.

`[Decision]` AN-01/AN-02 (2026-11-10): **no analytics and no crash reporting.** No analytics SDK, event schema, crash reporter, consent prompt, or telemetry leaves the device.

`[Recommendation]`

- In development, use `__DEV__`-guarded logs and React Native DevTools.
- In production, do not log raw equation input, token state, or session history. Log only non-identifying error categories if needed.
- Do not add analytics or crash reporting without reversing the product decision and completing a privacy review; both are out of scope for v1.
- App Store privacy answers are “Data Not Collected.” Confirm with the final dependency list.
- A privacy policy URL is still likely required for store submission; create one even though no data is collected.

### 5.6 Children's privacy and store compliance

`[Inferred]` The product is explicitly child-friendly (`index.html:6`, `docs/specs/...:5-8`). PB-01 confirms the users are a child alone or a child with a parent; SEC-03 sets the audience at **ages 10–15 plus older learners new to equations**, which spans the under-13 and teen bands. PB-07 confirms no monetization, ads, subscriptions, or accounts. The no-data, no-ads, no-account design is favorable under either store program. Because the audience is not under-13-only, the app should **not** be listed exclusively in Apple's Kids Category or Google Play Families; use the Education category and let the store questionnaire set the rating. Store category confirmation, privacy policy, and data-safety answers remain before submission (SEC-04, DEP-07).

---

## 6. Testing plan

### 6.1 Strategy

Use a small test pyramid tuned to this app:

1. **Unit tests for the pure engine** — highest value, fastest, most deterministic. The 70 existing self-checks and the spec's release examples are the seed corpus.
2. **Unit tests for the session reducer** — state transitions, immutability, duplicate guards, restore/flip/clear behavior.
3. **Component tests with React Native Testing Library** — rendering, accessible names/roles/states, disabled logic, feedback, completion.
4. **Integration tests** — render the app with the provider, drive tap/toolbar flows, assert engine/UI integration without a device.
5. **End-to-end tests with Maestro** — critical journeys on iOS simulator and Android emulator.
6. **Manual accessibility and device testing** — VoiceOver, TalkBack, dynamic type, reduced motion, external keyboard, low-end Android, orientation/safe areas.
7. **Release regression checklist** — adapt the web's final release checklist to native.

### 6.2 Unit tests for business rules

Tooling: `jest-expo`, TypeScript, pure functions, no React Native renderer for engine tests.

Must cover:

- Rational normalization, GCD, signs, zero canonicalization, add/subtract/multiply/divide, exact equality.
- Integer, terminating decimal, fraction, mixed-fraction parsing.
- Tokenizer: whitespace, operators, Unicode `− × ÷`, implicit multiplication, parentheses, malformed numbers, unsupported symbols, length/token limits.
- Parser precedence: parentheses, unary, multiplication/division, addition/subtraction, adjacent operands, unmatched parentheses.
- Affine reduction: constants, variable, groups, unary, addition/subtraction, scalar multiplication, nonlinear products, division by variable, division by zero.
- Source analysis: unique solution, identity, contradiction, nonlinear, unsupported rational, direct reciprocal (`c / y = d` and `d = c / y`), nonzero-domain restriction, canonical display formatting.
- Candidate analysis: valid equivalent, inequivalent, malformed side, empty side, nonlinear, divide-by-zero, unsupported rational, final-form predicate, exact fraction comparison, `1y` / `(y)` not final.
- Workspace helpers: insert/remove/move/copy/serialize, missing-token no-op, immutability.
- Duplicate/same-as-previous rules: 500 ms guard, token-identical row, notation-only equivalent.
- Regression corpus: all 70 `runMathSelfChecks()` cases (`index.html:1104-1292`) and the spec's final release examples (`docs/specs/linear-equation-drag-solver.md:184-196`), translated from `x` to `y` where the app uses `y`. **TST-07 decision: keep all 70.** The corpus executes in milliseconds; the cost is the one-time table-driven port (about a day), not runtime. Halving it would drop coverage of error codes and workspace rules for negligible savings.

Suggested coverage target: engine branch coverage ≥ 90%; reducer branch coverage ≥ 85%. Do not use coverage as a substitute for the golden corpus.

### 6.3 Component tests with React Native Testing Library

Test the user-visible contract, not implementation details:

- SetupScreen renders label, input, Start, tip; Start disabled when empty; invalid source shows the exact message; valid source navigates.
- SolverScreen renders locked history, active sides, palette, toolbar; selected/drop-target/error states; disabled buttons match state.
- TokenPalette exposes 18 buttons with labels such as “Insert zero”, “Insert variable y”.
- WorkspaceToken has an accessible name containing the token label and side.
- SideSelector exposes selected state.
- Toolbar disabled states: move left at index 0, move right at end, move-to-opposite disabled on current side, delete disabled with no selection.
- Completion panel renders the verified answer and “Solve another!”; focus/announcement behavior is invoked.
- FeedbackToast is hidden from accessibility and auto-dismisses.
- ConfettiOverlay renders zero pieces when reduced motion is enabled.
- Error messages are associated with the affected side and announced once.

Use RNTL queries by role/label; avoid `testID`-only assertions except for animation/measurement boundaries.

### 6.4 Integration tests

Render the root with the session provider and simulate:

- Enter `2y - 3 = 8` → Start → Step 1 locked → Step 2 prefilled.
- Tap palette `2`, `y` → Check → accepted → Step 2 locked → Step 3 prefilled.
- Build `y = 11 ÷ 2` → Check → completed → answer shown.
- Invalid source (`2x = 3`, `y² = 4`, `y = y`, `2/(y + 1) = 3`) → specific rejection, no session.
- Malformed active side (`2y −`) → rejection, no history row, still editable.
- Inequivalent active row (`2y - 3 = 9`) → rejection.
- Same-as-previous row → rejection.
- Restore/flip/clear/delete/move → correct state, locked history unchanged.
- Reset → landing, effects cleaned up.
- Duplicate rapid Check → one lock.

Mock only native boundaries: `expo-audio`, `expo-haptics`, Reanimated timing (or use its Jest setup), and accessibility announcement calls. Keep the engine real.

### 6.5 End-to-end tests

`[Decision]` Use **Maestro** (confirmed 2026-11-10).

Why Maestro over Detox:

- The app has no complex native synchronization, background work, or custom native modules; Detox's synchronization advantages are not needed.
- Maestro flows are simple YAML, run on iOS simulators and Android emulators, and are easy for a small team to maintain.
- Maestro can exercise visible text, taps, long presses, swipes, and keyboard input, which fits the setup → tap → check → complete journey.
- Detox requires more native test-build configuration, device setup, and maintenance; it is justified for large apps with complex native state, not this one.

Maestro flows:

1. **Setup validation:** invalid source shows the expected message; valid source starts.
2. **Tap-only completion:** enter `2y - 3 = 8`; insert tokens via palette; use move/delete toolbar; check equivalent steps; reach `y = 11 ÷ 2`; verify completion; reset.
3. **Rejection paths:** malformed side, inequivalent row, same-as-previous, unsupported source.
4. **Restore/flip/clear:** verify disabled states and that locked history is unchanged.
5. **Drag smoke test:** press and immediately drag a token, cross sides, drop in trash. Drag is required (PB-04), so this is a release-gating flow, not optional.
6. **Reset and repeat:** two back-to-back sessions; no stale completion/effect.
7. **Accessibility labels smoke test:** assert visible labels/roles are exposed through Maestro's text matching; supplement with manual screen-reader testing.

Maestro cannot reliably assert audio, haptics, reduced-motion behavior, dynamic type, or VoiceOver/TalkBack output. Those are manual checks.

### 6.6 Accessibility testing

- **Automated:** RNTL role/label/state assertions; optional `eslint-plugin-react-native-a11y` for common mistakes; `AccessibilityInfo` announcement spies in tests.
- **Manual iOS:** VoiceOver traversal of Setup and Solver; confirm headings, labels, selected/disabled states, error announcements, completion focus, and that decorative confetti/toast are silent.
- **Manual Android:** TalkBack traversal; confirm live-region announcements and focus behavior.
- **Dynamic type:** 200% text size on iOS and maximum font scale on Android; verify no clipping, token buttons remain usable, and equations wrap.
- **Reduced motion:** enable the system setting; verify confetti and shake are skipped/reduced while success/error meaning remains.
- **Contrast:** audit text, focus ring, borders, disabled states, and token colors; fix the focus/border concerns noted in §1.15 or record an accepted exception.
- **Touch targets:** 44×44 pt iOS / 48×48 dp Android; use `hitSlop` where visuals are smaller.
- **Hardware keyboard:** required in addition to screen-reader actions (NAT-03). Verify Tab/Shift+Tab traversal through palette, tokens, and toolbar, Enter/Space activation, visible focus, and that every pointer operation has a keyboard path. Screen-reader `accessibilityActions` and hardware Tab are tested separately.

### 6.7 iOS and Android testing

- **Minimum OS (TST-03/DEP-09):** iPadOS/iOS 16+, Android 9 (API 28)+.
- **Tablets first (UX-02):** iPad (standard and a smaller iPad class) and a standard Android tablet; these are the primary acceptance devices.
- iOS simulators: at least one iPad, plus a small phone (iPhone SE class) and a current standard phone to prove the phone fallback.
- Android emulators: a tablet configuration, a small 360dp phone, a current Pixel-class phone, and a low-end device/API level for performance.
- Test safe areas, status bar, keyboard avoidance, Android back button, rotation if supported, and app background/resume.
- Physical device pass before release, including at least one physical tablet; simulators/emulators do not prove audio, haptics, gesture feel, or performance.

### 6.8 Screen sizes and layout

- **Primary target: tablet.** Verify the tablet layout at standard and large tablet widths (roughly 768–1024dp and above).
- **Phone fallback:** minimum width 320–360dp; target 360–430dp. The app must open and complete a solve on a phone.
- Test with large text, bold text, and display-size scaling on both form factors.
- Verify no horizontal overflow; equations wrap inside their side.
- Verify the palette/toolbox remains reachable and does not cover the active row or feedback in both layouts.
- Verify completion panel and answer wrap.
- Orientation remains open (UX-02 answered device size, not orientation); test whichever orientations ship and record any excluded one.

### 6.9 Keyboard behavior

- Setup: keyboard appears, content scrolls/avoids the keyboard, Return starts the equation, invalid input keeps focus and shows feedback.
- Auto-correct, smart punctuation, and capitalization are disabled for the equation field.
- Paste of supported and unsupported text behaves predictably.
- External keyboard: focus traversal, activation, and token editing are required, not optional (NAT-03); test with a hardware keyboard attached on a tablet.

### 6.10 Offline and slow-network conditions

`[Decision]` The app is standalone and fully offline, including first launch (OFF-01, 2026-11-10). It makes no network requests, so “offline” is the normal state. Verify:

- Airplane mode: app launches and all features work.
- No blocking network call at startup.
- No OTA update service is assumed (DEP-03); the shipped bundle and assets are self-contained.
- Audio assets are bundled, not fetched.

There is no slow-network test for app data because there is no server data. If a backend is added later, add latency/failure tests then.

### 6.11 API failure and authentication expiry

`[Decision]` Not applicable in v1: the app is standalone with no API, no auth, no tokens, and no expiry (SEC-01, SEC-02, API-01, 2026-11-10). Record this explicitly in the test plan so these cases are not silently omitted if requirements change. If auth or a backend is added later, add token-refresh, expired-session, offline-auth, and logout tests.

### 6.12 Upgrade and regression testing

- Keep the engine golden tests and reducer tests as a non-negotiable CI gate.
- Run the native release checklist for every Expo SDK upgrade, React Native upgrade, Reanimated/Gesture Handler upgrade, and audio-library upgrade.
- Use the `expo-upgrade` skill/process; do not hand-bump versions.
- Re-run Maestro flows on both platforms after upgrades.
- Keep a device matrix record with OS versions and any untested platforms, as the web project did in `tasks/04-todo.md`.
- Add regression tests for every bug found in UAT.

### 6.13 CI gates

Minimum pull-request gates:

- `tsc --noEmit` with strict mode.
- ESLint (Expo config) and formatting check.
- Jest unit/component tests.
- Maestro smoke flow on at least one iOS simulator and one Android emulator (nightly or release branch if too slow per PR).
- Local Expo prebuild/native build check for both platforms (DEP-03).

Release gates:

- Full Maestro suite on both platforms.
- Manual accessibility/device checklist.
- Store metadata/privacy review.
- Crash-free smoke run on a release build.

---

## 7. Delivery plan

Effort is in person-days (pd) for one experienced React Native engineer with product/design/QA support. Ranges are planning estimates, not commitments. They exclude store review waiting time and major scope changes.

### Phase 1 — Discovery and requirements confirmation

- **Objective:** Confirm scope parity, target users, platforms, and the meaning of “done” before code.
- **Main activities:** Reconcile the stale README/spec; translate `x` examples to `y`; record the PB/UX/NAT/DEP decisions in the scope matrix; capture the web baseline evidence that PB-03 assumes (checklist results, screenshots, recordings) under TST-05 (Boris); confirm what remains out of v1 (orientation, store rating at submission); confirm store credentials and the local build path (DEP-01, DEP-03).
- **Deliverables:** Approved scope matrix incorporating the PB answers; corrected baseline documentation; confirmed device/OS matrix; resolved remaining blocking open questions; recorded web baseline evidence.
- **Dependencies:** Product owner, designer, QA, and store/legal input.
- **Acceptance criteria:** No unresolved blocker in the requirements document; PB-03 baseline evidence recorded even though the baseline itself is accepted; v1 scope signed off.
- **Risks:** The PB-03 decision is accepted but unrecorded behavior may diverge from the spec; the PB-02/PB-10 replacement-versus-parallel tension is unresolved. Documentation drift wastes time.
- **Effort:** 2–4 pd.

### Phase 2 — Architecture and proof of concept

- **Objective:** Prove the riskiest technical assumptions before building the app.
- **Main activities:** Scaffold a throwaway Expo app (not the production app); prove Hermes `BigInt` and the exact-rational engine; port a small slice of the tokenizer/parser; spike an immediate-pan token drag (UX-06) with measured drop targets and a scroll-conflict decision; spike reduced-motion confetti; spike `expo-audio` playback and confirm iOS silent-switch muting (NAT-10); spike VoiceOver/TalkBack announcements, `accessibilityActions`, and hardware-keyboard focus (NAT-03); prove the single-route shell with a session provider and Android back interception (UX-08).
- **Deliverables:** PoC branch or notes; measured results; updated risk register; final dependency versions; go/no-go recommendation.
- **Dependencies:** Phase 1 decisions; device/simulator access; Expo account.
- **Acceptance criteria:** BigInt works on device; drag is usable on iOS and a low-end Android emulator; audio/confetti/a11y behave acceptably; no fatal New Architecture incompatibility.
- **Risks:** Drag gesture conflicts with scrolling; audio silent-switch behavior; accessibility focus APIs differ; PoC scope creep.
- **Effort:** 4–7 pd.

### Phase 3 — Project foundation

- **Objective:** Create the production Expo project and its quality gates.
- **Main activities:** `create-expo-app` with TypeScript; strict tsconfig; Expo Router; folder structure; app config (identity per DEP-02, orientation, icon/splash placeholders); ESLint/Prettier; Jest/RNTL; Maestro skeleton; local Xcode/Gradle build configuration (DEP-03); CI; error boundary; theme tokens; shared components; safe-area/gesture root providers.
- **Deliverables:** Buildable iOS/Android development app; CI passing; documented run/build commands; initial ADRs.
- **Dependencies:** Phase 2 decisions.
- **Acceptance criteria:** App launches on iOS simulator and Android emulator; typecheck/lint/unit test commands pass; a dev build can be produced.
- **Risks:** Expo SDK/dependency version mismatches; local signing/Xcode/Gradle setup delays.
- **Effort:** 2–3 pd.

### Phase 4 — Static UI migration

- **Objective:** Build the native Setup and Solver screens with realistic static data and full visual states.
- **Main activities:** Setup screen; brand/header; equation input; primary buttons; tip card; Solver screen; locked history; active equation; token sides; palette; toolbar; completion panel; feedback/toast; theme tokens; responsive/safe-area layout; loading/empty/error/success visual states.
- **Deliverables:** Tappable static UI on both platforms; component inventory; screenshot review against the web design intent.
- **Dependencies:** Phase 3.
- **Acceptance criteria:** All screens and states render in the tablet-first layout and the phone layout; no clipping at 768–1024dp+ tablet widths, 320–430dp phone widths, or 200% text; 44/48dp targets; keyboard avoidance works; design review approved.
- **Risks:** Visual redesign decisions; two form-factor layouts; dynamic type; palette ergonomics; gradient/shadow differences.
- **Effort:** 6–9 pd (tablet-first layout plus the phone fallback, UX-02).

### Phase 5 — Business logic and validation migration

- **Objective:** Port the exact math engine and session state machine with regression tests.
- **Main activities:** Port rational/tokenizer/parser/affine/source/candidate/formatting into pure TS; port workspace helpers; port the 70-case corpus to Jest; implement the session reducer and actions; wire Setup validation; wire Check/lock/history/restore/flip/clear/delete/move; preserve duplicate/same-as-previous rules; add error mapping.
- **Deliverables:** Pure engine module with tests; reducer with tests; end-to-end tap-only solving flow.
- **Dependencies:** Phase 3; Phase 4 components for integration.
- **Acceptance criteria:** 70/70 golden tests pass; all spec examples pass; `2y - 3 = 8 → 2y = 11 → y = 11 ÷ 2` completes; invalid/inequivalent/same rows rejected; locked history immutable.
- **Risks:** Subtle parser/formatting differences; BigInt serialization mistakes; reducer transition bugs.
- **Effort:** 4–6 pd.

### Phase 6 — API and authentication integration

- **Objective:** Confirm and document the product decision that the app is standalone and offline with no API or authentication in v1 (SEC-01, SEC-02, API-01, OFF-01).
- **Main activities:** Verify no residual network/auth assumptions; document the “none” decision; add a guard/test that no startup network request exists; record future integration points.
- **Deliverables:** Explicit N/A record; architecture note for future backend/auth.
- **Dependencies:** None remaining — the decision is made.
- **Acceptance criteria:** App works fully offline including first launch; no tokens/secrets/endpoints; privacy answers remain “no data collected.”
- **Risks:** Scope creep from “while we're here” account features; a future feature reintroducing a network dependency.
- **Effort:** 0–1 pd.

### Phase 7 — Storage and native capability integration

- **Objective:** Implement the native-specific capabilities: required gestures/drag, required audio, optional haptics, confetti, accessibility focus, and back-navigation behavior. No persistence (PB-05).
- **Main activities:** Custom token drag with immediate pan (UX-06), measured drop targets, trash drop, insertion index; tap/toolbar and hardware-keyboard accessible alternatives (NAT-03); `expo-audio` chimes with bundled assets; `expo-haptics`; Reanimated confetti with reduced motion; accessibility announcements/focus; app icon/splash; Android back interception that preserves the session (UX-08). No session persistence; only non-sensitive preference storage if a mute/haptics control is approved (NAT-04).
- **Deliverables:** Native interaction layer; assets; capability tests; device recordings.
- **Dependencies:** Phase 2 PoC; Phase 4/5.
- **Acceptance criteria:** Drag reorders/moves/deletes correctly on both platforms; tap/toolbar path is equally complete; required sound plays, is muted by the iOS silent switch, and follows media volume on Android (NAT-10); completion shows “Good job” and “Solve another!” (UX-07); confetti, if kept, is reduced-motion-safe; no permission prompts; assets load offline.
- **Risks:** Drag/scroll conflict; audio silent switch; emoji/icon differences; accessibility focus reliability.
- **Effort:** 5–8 pd. Drag and sound are both first-release requirements (PB-04, PB-06), so the upper half of the earlier 4–7 pd range no longer applies as a contingency; treat Phase 7 as a critical-path phase.

### Phase 8 — Testing and accessibility

- **Objective:** Make the app release-candidate quality on both platforms.
- **Main activities:** Complete unit/component/integration tests; Maestro flows; VoiceOver/TalkBack passes; dynamic type; reduced motion; contrast audit; device matrix; performance/startup checks; bug fixes; CI hardening.
- **Deliverables:** Test report; accessibility report; known-issues list; release candidate builds.
- **Dependencies:** Phases 4–7.
- **Acceptance criteria:** All critical flows pass on iOS and Android; no blocker accessibility defects; golden engine tests green; no console errors; performance acceptable on low-end Android.
- **Risks:** Accessibility gaps discovered late; gesture feel issues; platform differences.
- **Effort:** 4–7 pd.

### Phase 9 — UAT

- **Objective:** Validate the app with the actual learner/adult users and the product owner.
- **Main activities:** Scripted UAT sessions; observe children using drag and tap paths; collect comprehension feedback on error messages; verify parent/teacher expectations; triage issues; fix release blockers.
- **Deliverables:** UAT sign-off; prioritized defect list; copy/UX adjustments.
- **Dependencies:** Phase 8; participant scheduling.
- **Acceptance criteria:** UAT scenarios pass; no severity-1/2 defects; product owner signs off.
- **Risks:** Child usability issues not visible in QA; scheduling; scope changes from feedback.
- **Effort:** 3–5 pd.

### Phase 10 — App Store and Google Play preparation

- **Objective:** Prepare compliant, complete store submissions.
- **Main activities:** Final app icon/splash; screenshots; store descriptions; age rating; kids/families category; privacy policy; data-safety answers; final bundle/package identity (DEP-02); signing credentials; local Xcode archive and Play Console upload (DEP-03); TestFlight/internal testing; review guidelines check.
- **Deliverables:** Store listings; signed builds; TestFlight/Internal App Sharing builds; privacy documentation.
- **Dependencies:** Phase 9; Apple/Google accounts; legal/privacy approval.
- **Acceptance criteria:** Builds accepted by TestFlight/Play Console; metadata complete; no policy blockers; reviewer test notes prepared.
- **Risks:** Kids-category review scrutiny; privacy policy gaps; signing/account delays; rejection for metadata or external links.
- **Effort:** 3–5 pd.

### Phase 11 — Production release and monitoring

- **Objective:** Release safely and observe the first production cohort.
- **Main activities:** Phased rollout; store review response; hotfix plan; post-release regression; documentation handover. No OTA service (DEP-03); fixes ship through store review. No analytics or crash-reporting SDK (AN-01/AN-02); field issues are collected from user reports and reproduced locally.
- **Deliverables:** Production release; support/feedback plan instead of a monitoring dashboard; release notes; handover docs.
- **Dependencies:** Phase 10.
- **Acceptance criteria:** Release live on both stores; no P1 defects reported during phased rollout; support process defined.
- **Risks:** Store rejection/delay; platform-specific production bugs; no crash reporting means slower diagnosis; no OTA means a slow store-review cycle for urgent fixes.
- **Effort:** 2–3 pd.

### 7.1 Total and sequencing

| Phase | Effort (pd) |
| --- | ---: |
| 1. Discovery and requirements | 2–4 |
| 2. Architecture and PoC | 4–7 |
| 3. Project foundation | 2–3 |
| 4. Static UI | 6–9 |
| 5. Logic and validation | 4–6 |
| 6. API/auth (N/A) | 0–1 |
| 7. Storage/native capabilities | 5–8 |
| 8. Testing and accessibility | 4–7 |
| 9. UAT | 3–5 |
| 10. Store preparation | 3–5 |
| 11. Release and monitoring | 2–3 |
| **Subtotal** | **35–58** |
| Contingency (15–20%) | 5–12 |
| **Planning range** | **40–70 pd** |

`[Recommendation]` The critical path is Phase 2 → 5 → 7 → 8. Phase 4 can run partly in parallel with Phase 5 once the component contracts are agreed. Phase 10 can start store metadata work while Phase 8 is running, but submissions depend on UAT.

**Milestone checkpoints**

- **Checkpoint A (after Phase 2):** PoC proves BigInt, drag, audio, confetti, and accessibility announcements; go/no-go on the Expo approach.
- **Checkpoint B (after Phase 5):** Tap/toolbar solving works end-to-end with the 70-case engine suite green. Not releasable: drag is required (PB-04).
- **Checkpoint C (after Phase 7):** Drag, audio, haptics, confetti, and accessibility focus are integrated and usable.
- **Checkpoint D (after Phase 8):** Release candidate passes the device/a11y matrix.
- **Checkpoint E (after Phase 9):** UAT sign-off.
- **Checkpoint F (after Phase 11):** Production release.

---

## 8. Risk register

Likelihood/impact: High, Medium, Low. “Evidence” cites the source or the technical cause.

| ID | Risk | Likelihood | Impact | Evidence | Mitigation | Decision / investigation required |
| --- | --- | --- | --- | --- | --- | --- |
| R1 | Drag-and-drop rewrite does not match web behavior or feels unreliable | High | High | Web uses Pointer Events, pointer capture, `elementFromPoint`, `getBoundingClientRect`, body preview, insertion markers (`index.html:1884-2005`); PB-04 requires drag in the first release | PoC in Phase 2; immediate pan on touch-move (UX-06); measured rects; side-level highlight; tap/toolbar path remains the accessible alternative; test on low-end Android | **Resolved:** drag is required (PB-04) and the affordance is immediate drag (UX-06). Residual risk: scroll-conflict behavior and fallback if the PoC fails on both platforms |
| R2 | Accessibility parity is weaker than web | Medium | High | Web has live regions, `aria-pressed`, focus moves, keyboard alternatives, reduced motion (`index.html:490`, `:524-525`, `:1795-1802`, `:459-461`) | A11y-first component contracts; VoiceOver/TalkBack passes; `accessibilityActions` plus hardware-keyboard focus order (NAT-03); announcements/focus; manual checklist; fix contrast | **Resolved:** both screen-reader actions and external hardware keyboard are required (NAT-03). Acceptance owner still needed (TST-02) |
| R3 | Visual redesign is judged as a regression | Medium | Medium | CSS gradients, sticky toolbox, clamp, shadows, keyframes (`index.html:9-462`); tablet-first layout (UX-02) plus phone fallback | Design review; native redesign principles; screenshot comparisons per form factor; agree “native, not pixel-identical” | **Resolved:** native redesign, not pixel parity (UX-01). Brand assets remain open (DEP-02 follow-up) |
| R4 | Math engine port introduces subtle regressions | Medium | High | 600+ lines of parser/rational logic; 70 self-checks; exact formatting (`index.html:621-1098`) | Port line-for-line; convert 70 cases to Jest; add spec examples; golden tests in CI; differential testing against web output | Approve the golden corpus; assign engine reviewer |
| R5 | `BigInt` behaves differently on Hermes/devices | Low | Medium | Web relies on BigInt literals and `BigInt()` (`index.html:629-700`) | PoC on iOS/Android; verify RN/Hermes version; avoid JSON serialization; encode strings if persisted | Confirm minimum RN/Expo SDK; add device test |
| R6 | Audio parity is lost or problematic | Medium | Medium | Web synthesizes tones with Web Audio (`index.html:1602-1664`); no audio assets; sound is required (PB-06) | Bundle original short chimes; `expo-audio`; iOS silent switch mutes, Android follows media volume (NAT-10); haptics as an additional cue | **Resolved:** sound required and silent-switch policy set. Remaining: asset ownership/licensing; mute control deferred (NAT-04) |
| R7 | Confetti/animation causes jank or ignores reduced motion | Low | Low | 42 CSS pieces, keyframes (`index.html:384-420`) | Reanimated UI thread; 42 pieces max; `useReducedMotion`; cleanup; device test | Confirm reduced-motion acceptance criteria |
| R8 | External keyboard / tab-order requirement cannot be met | Medium | Medium | Web provides keyboard/tap alternatives (`index.html:551-557`); RN has no automatic tab order | Explicit focus order and focusable controls for hardware keyboards; `accessibilityActions` for screen readers; toolbar alternative; device test with a keyboard attached | **Resolved:** both paths are required (NAT-03). Residual risk is iOS/Android focus-order differences |
| R9 | Session lost on background/termination | High | Low | No persistence; reload clears state (`CLAUDE.md:9-10`, source) | Accept process-death loss (PB-05); back preserves the session and Reset is explicit (UX-08) | **Resolved:** no persistence required (PB-05); back preserves the session (UX-08). Residual risk is user surprise after process death |
| R10 | App Store/Google Play review rejection | Medium | High | Child-friendly product (`index.html:6`, spec `:5-8`); no privacy docs in repo; audience is ages 10–15 plus new learners (SEC-03); no ads/accounts (PB-07) | Education listing rather than Kids-only; privacy policy; rating from the store questionnaire; no ads/analytics/external links; reviewer notes; TestFlight/internal testing | Remaining: privacy policy owner, store category confirmation, data-safety answers (SEC-04, DEP-07) |
| R11 | Emoji/icon/font rendering differs across platforms | Medium | Low | Emoji and Unicode icons (`index.html:467`, `:541`, `:571`); system font stack | Use vector icons for essential actions; emoji only decorative; test both platforms; app icon as PNG | Decide icon strategy and app icon design |
| R12 | Performance on low-end Android / many tokens | Low–Medium | Medium | Max 160 tokens per side; full re-render on every state change (`index.html:587`, `:1731-1803`) | Small tree; avoid per-frame setState; Reanimated drag; measure startup/interaction; memoize only if measured | Set performance baseline/target |
| R13 | New Architecture or dependency incompatibility | Low | Medium | Target requires New Arch; Reanimated 4 requires it | Pin Expo-compatible versions; PoC; avoid unmaintained libraries; `expo-doctor` | Confirm final dependency versions at implementation start |
| R14 | Team learning curve for Expo/native release | Medium | Medium | Team currently web-only | Phase 2 PoC; pair with RN-experienced reviewer; use skills/docs; budget training | Confirm team capacity and reviewer |
| R15 | Stale documentation drives wrong requirements | Medium | Medium | README says UI-only; spec uses `x`; human gates open (`README.md:5-8`, spec `:3`, tasks `04-todo.md`) | Reconcile docs in Phase 1; single source of truth; review with product owner | Approve corrected scope baseline |
| R16 | No automated web baseline; parity relies on manual checklist | Medium | Medium | Only `runMathSelfChecks` in browser console (`index.html:1104-1292`); no CI; baseline accepted but evidence not captured (PB-03) | Convert self-checks to Jest; run web manual checklist; record evidence; differential tests | Baseline accepted; evidence owner is Boris (TST-05). Format still to confirm |
| R17 | Learner input/privacy leaks through logs or a later analytics SDK | Low | Low | Free-text input; no logging/analytics; no analytics or crash reporting by decision (AN-01/AN-02) | No raw-input logs; privacy review; keep the no-telemetry decision | Privacy policy still required; no monitoring plan |
| R18 | Store accounts/signing/local build setup delays release | Medium | Medium | No store configuration in repo; local Xcode/Gradle path (DEP-03) | Start credentials in Phase 1; verify local archive/upload early; prepare TestFlight/internal testing early | **Resolved:** Boris owns the accounts (DEP-01); local build path decided (DEP-03). Residual risk is signing/toolchain setup time |
| R19 | OTA update misuse or runtime incompatibility | Low | Medium | No OTA service is in scope (DEP-03) | Ship through store review; no OTA configuration | **Resolved:** OTA not in scope |
| R20 | Platform differences in safe area, keyboard, back button, audio | Medium | Medium | Web uses `env(safe-area-inset-*)`, browser history absent, Web Audio (`index.html:190`, `:363-364`); tablet and phone layouts differ (UX-02) | Safe-area insets; KeyboardAvoidingView; back interception preserving the session (UX-08); two-form-factor device matrix | Orientation remains open; back behavior decided |

---

## 9. Open questions

The full, grouped list with impact statements is in **`docs/react-native-requirements-open-questions.md`**. PB-01–PB-10 were answered on 2026-11-10:

| Answered | Decision |
| --- | --- |
| PB-01 | Child with parent and child alone; no classroom/teacher scope |
| PB-02 | Native app replaces the web app |
| PB-03 | Current web behavior is the parity baseline |
| PB-04 | Drag-and-drop **and** tap/toolbar editing are both required |
| PB-05 | No session persistence |
| PB-06 | Sound is required |
| PB-07 | No monetization at this time |
| PB-08 | English only |
| PB-09 | No measurable success goal at this time |
| PB-10 | Web app maintained in parallel |

Also resolved on 2026-11-10: **no authentication** (SEC-01), **no learner data leaves the device** (SEC-02), **no backend/API and bundled content** (API-01, API-02), and **fully offline including first launch** (OFF-01). SEC-05/06, API-03/04, and OFF-02/04 are N/A as a consequence.

Also resolved in the latest batch: **native redesign, not pixel parity** (UX-01); **palette inserts append to the selected side** (UX-05); **immediate drag on touch-move** (UX-06); **back preserves the session** (UX-08); **both screen-reader actions and external hardware keyboard** (NAT-03); **Boris owns the store/Expo accounts** (DEP-01); **identity `net.bbsteps.equation` / scheme `equation` / `equation.bbsteps.net`** (DEP-02); **local Xcode/Gradle builds, no EAS Build/Submit** (DEP-03); **OS floor iOS/iPadOS 16+, Android 9/API 28+** (TST-03, DEP-09); **keep all 70 engine cases** (TST-07); **baseline evidence owner is Boris** (TST-05).

The highest-impact questions that remain:

1. **NAT-04** — Are there user-facing mute/haptics controls? Silent-switch behavior is decided (NAT-10).
2. **UX-02 (orientation)** — Tablet primary and phones supported are decided, but portrait vs landscape is unstated.
3. **SEC-04 / DEP-07** — Privacy policy owner, final store category, and data-safety answers. Age band and audience are decided (SEC-03).
4. **TST-01 / TST-02 / TST-06** — UAT participants, accessibility sign-off owner, beta testers.
5. **Brand assets** — App icon and splash design (identity values are settled).
6. **Release cadence** — Local builds and account ownership are decided; who cuts releases and how often is not.
7. **PB-02 / PB-10** — The native app replaces the web while the web is maintained in parallel; the retirement trigger is unresolved.

---

## 10. Final decision material

### 10.1 Recommended target approach

**React Native + TypeScript + Expo modules + Development Builds + Maestro + Claude Code; Expo managed / Continuous Native Generation, no committed native projects, with local Xcode/Gradle builds (DEP-03).** Use Expo Router as a single-route shell, TypeScript strict, React Native New Architecture, Gesture Handler + Reanimated with immediate drag (PB-04, UX-06), expo-audio (sound required by PB-06) plus optional expo-haptics, and Jest + RNTL + Maestro. Rebuild the UI as a native tablet-first experience with a native redesign rather than pixel parity (UX-01, UX-02); port the exact math engine to pure TypeScript; keep all 70 self-check cases as the regression gate (TST-07 decision). Do not use a WebView, DOM component, or Expo DOM.

### 10.2 Recommended first implementation milestone

**RN-1: Tap/toolbar walking skeleton (internal milestone, not the release scope).**

Scope:

- Expo project with TypeScript strict, New Architecture, Expo Router, app config placeholders.
- Port the exact math engine and convert the 70 self-checks to Jest; all green.
- Setup screen with input validation and error messages.
- Solver screen with locked Step 1, active Step 2, palette tap insertion, token selection, move/reorder/delete via toolbar, clear/restore/flip, Check, lock, completion panel, reset.
- No custom drag, no confetti, no audio; those follow in Phase 7.
- Runs on iOS simulator and Android emulator; one Maestro smoke flow; basic accessibility labels.

Acceptance: `2y - 3 = 8` → `2y = 11` → `y = 11 ÷ 2` completes; malformed, inequivalent, and same-as-previous rows are rejected; locked history is immutable; reset works; tests and typecheck pass.

`[Decision]` PB-04 makes drag a **first-release requirement**, so RN-1 is a sequencing milestone only. The first store release (after Phase 7) must include both drag and the tap/toolbar path; a tap-only build is not releasable on its own.

Why this milestone first: it proves the engine port, domain state machine, navigation, input, and native rendering — the seams most likely to invalidate the plan — before investing in the high-risk drag layer and celebration.

### 10.3 Prerequisites before coding begins

- [x] Product owner confirms no persistence (PB-05), no monetization/accounts (PB-07), and English-only launch (PB-08).
- [x] Product owner confirms no auth and no backend; the app is standalone and fully offline (SEC-01, SEC-02, API-01, API-02, OFF-01).
- [x] No analytics and no crash reporting (AN-01, AN-02); store privacy answers remain “Data Not Collected.”
- [ ] Web baseline evidence is recorded (PB-03 accepts the baseline; TST-05 owner is Boris); README/spec `x` vs `y` drift is resolved.
- [x] Drag and tap/toolbar are both required in the first release (PB-04).
- [x] Drag affordance (UX-06), tap insertion position (UX-05), and keyboard-accessibility definition (NAT-03) are agreed.
- [x] Engine regression corpus: keep all 70 cases (TST-07).
- [x] Sound is required (PB-06).
- [x] Silent-switch policy: iOS mutes, Android follows media volume (NAT-10).
- [ ] Audio asset ownership/licensing and optional mute/haptics controls (NAT-04) are agreed.
- [x] Theming: single light theme, no dark mode (UX-03).
- [x] Device class: tablet primary, phones supported (UX-02).
- [x] Visual direction: native redesign, not pixel parity (UX-01).
- [ ] Orientation (portrait/landscape), app icon, and splash are agreed.
- [x] Audience: ages 10–15 plus older learners new to equations; Education listing, not Kids-only (SEC-03).
- [ ] Privacy policy owner and final store rating/data-safety answers are agreed (SEC-04, DEP-07).
- [x] Supported iOS/Android versions and device matrix are agreed: iOS/iPadOS 16+, Android 9 (API 28)+ (TST-03, DEP-09).
- [x] Account ownership: Boris owns Apple Developer, Google Play, and Expo (DEP-01).
- [ ] Store credentials/signing are available on the local build machine (DEP-03).
- [x] Toolchain confirmed: React Native + TypeScript + Expo modules + development builds + Maestro + Claude Code.
- [ ] RN/Expo dependency policy and reviewer capacity are confirmed.
- [ ] Phase 2 PoC acceptance criteria are agreed.
- [ ] UAT and accessibility sign-off owners are named.
- [ ] The PB-02/PB-10 replacement-versus-parallel tension is resolved in writing.

### 10.4 Go or no-go criteria

**Go** when all are true:

- The app’s scope is confirmed as the existing equation solver, with no hidden backend/auth/analytics requirements.
- Expo managed/CNG is acceptable to engineering and operations.
- The Phase 2 PoC proves BigInt, drag (required by PB-04, no tap-only fallback), accessibility announcements/focus, audio/confetti, and both-platform builds.
- The web baseline is accepted (PB-03) and its evidence has been recorded.
- Store accounts and privacy ownership exist (DEP-01); local build tooling is available (DEP-03).
- The team can support iOS and Android releases after launch.

**No-go or defer** when any are true:

- The target requires WebView/DOM wrapping rather than a native UI.
- A required native SDK/feature forces bare/prebuild and the team cannot support it.
- Drag/accessibility acceptance cannot be met on the supported device matrix and no fallback is approved.
- The web baseline cannot be accepted or reconstructed, so “parity” is undefined.
- Store/legal/kids-privacy requirements cannot be resolved.
- There is no budget/owner for two-platform maintenance and store releases.
- The product would be equally served by the existing web app and native delivery has no measurable benefit.

### 10.5 Plan review and approval checklist

- [ ] Inventory matches the source; no feature was invented or omitted.
- [ ] Reusability classifications are accepted, especially “engine reusable after adaptation” and “UI must be rewritten.”
- [ ] Web-to-native mapping covers every required interaction, including drag, storage, navigation, keyboard, and responsive behavior.
- [ ] Expo managed/CNG recommendation and rejected alternatives are accepted.
- [ ] No WebView/DOM-component approach is approved.
- [ ] Architecture plan and folder structure are approved.
- [ ] Dependency list is approved; no state-management, API, auth, or UI-kit library is added without justification.
- [ ] Security/storage posture is approved: in-memory v1, no secrets, no plaintext token storage.
- [ ] Testing plan, Maestro choice, and accessibility criteria are approved.
- [ ] Delivery phases, effort range, dependencies, and checkpoints are approved.
- [ ] Risk register owners and mitigations are assigned.
- [ ] All blocking open questions are resolved or explicitly deferred with an owner and date.
- [ ] Prerequisites and go/no-go criteria are accepted.
- [ ] Approval to create the React Native project is given separately; this plan does not authorize implementation.

---

## Appendix A — Evidence index

| Topic | Source |
| --- | --- |
| Product purpose and users | `docs/specs/linear-equation-drag-solver.md:5-15`; `index.html:465-491` |
| Scope, notation, exceptions | `docs/specs/linear-equation-drag-solver.md:17-24`; `index.html:586-619`, `:702-772` |
| Interaction flow | `docs/specs/linear-equation-drag-solver.md:105-110`; `index.html:2172-2230` |
| Functional requirements | `docs/specs/linear-equation-drag-solver.md:112-147` |
| Non-functional requirements | `docs/specs/linear-equation-drag-solver.md:148-153` |
| Final release checklist | `docs/specs/linear-equation-drag-solver.md:184-196` |
| Human gates still open | `docs/specs/linear-equation-drag-solver.md:3`; `tasks/04-todo.md` Task 0/7 |
| Stale README | `README.md:5-8` |
| Math engine | `index.html:621-1098` |
| Self-check corpus | `index.html:1104-1292` |
| Workspace helpers | `index.html:1294-1360` |
| State | `index.html:1362-1378` |
| DOM rendering | `index.html:1450-1808` |
| Pointer drag | `index.html:1884-2005` |
| Check/lock/start/reset | `index.html:2007-2230` |
| CSS and responsive rules | `index.html:9-462` |
| Accessibility markup | `index.html:480-576`, `:1450-1517`, `:1731-1803` |
| Celebration | `index.html:1558-1729`, `:2112-2123` |
| No API/auth/storage | Whole-file search of `index.html`; `README.md:19-20` |
| Assets | `index.html:7`, `:46`, `:467`, `:487`, `:541`, `:571` |

## Appendix B — Glossary

- **CNG:** Continuous Native Generation; Expo generates native projects on demand from app config and plugins instead of committing them.
- **EAS:** Expo Application Services; cloud build, submit, update, and workflow services.
- **New Architecture:** React Native's Fabric renderer/Turbo Modules architecture; required by Reanimated 4.
- **RNTL:** React Native Testing Library.
- **Maestro:** YAML-based mobile UI testing tool.
- **Direct-reciprocal source:** The narrow accepted equation form `c / y = d` or `d = c / y` with nonzero constants.
- **Final form:** Left side exactly one `y` token; right side has no `y`; right constant equals the source solution.
- **Locked row:** An accepted, immutable history row; only the active row is editable.