# React Native Architecture Plan — Equation Explorer

**Status:** Planning only. No React Native project has been created and no production code has been written. Product and platform decisions from 2026-11-10 (PB, UX, connectivity, analytics, toolchain) are reflected in the decision summary and ADRs.
**Date:** 2026-11-09; decisions incorporated 2026-11-10
**Target:** iOS and Android tablets as the primary target, with phone support (UX-02); TypeScript strict; React Native New Architecture; single light theme (UX-03).
**Companion documents:** [`react-native-migration-plan.md`](./react-native-migration-plan.md) (inventory, feasibility, target-platform decision, delivery, risks) and [`react-native-requirements-open-questions.md`](./react-native-requirements-open-questions.md) (unresolved product and delivery questions).

**Label key**

- **[Observed]** — directly present in the existing source, with a file/line reference.
- **[Inferred]** — a reasonable technical consequence of the source or the chosen platform.
- **[Recommendation]** — a proposed decision for approval; not yet implemented.
- **[Open]** — unresolved; see the open-questions document.

---

## 0. Decision summary

| # | Decision | Choice | Status |
| --- | --- | --- | --- |
| D1 | Delivery platform | React Native + Expo modules, Expo managed / Continuous Native Generation, development build, **local Xcode/Gradle builds** (no EAS Build/Submit) | **Accepted** (2026-11-10; DEP-03) |
| D2 | Navigation | Expo Router, one visible route driven by a session phase; no stack transition in v1 | [Recommendation] |
| D3 | Language | TypeScript strict; no JavaScript source files | **Accepted** (2026-11-10) |
| D4 | UI layer | React Native core components + `StyleSheet`, design tokens, no UI kit | [Recommendation] |
| D5 | State | `useReducer` + React Context; no Redux/Zustand/Jotai/React Query | [Recommendation] |
| D6 | Domain engine | Framework-free pure TypeScript port of the exact-rational engine; `BigInt`; no `eval` | [Recommendation] |
| D7 | Gestures and animation | React Native Gesture Handler + Reanimated; gesture state on the UI thread; drag is a first-release requirement; **immediate drag on touch-move**, no long-press or handle | **Accepted** (PB-04, UX-06) |
| D8 | Feedback | Bundled audio chimes via `expo-audio` (required); `expo-haptics` optional; both failure-tolerant; iOS silent switch mutes, Android follows media volume | **Accepted** (PB-06, NAT-10); mute/haptics controls deferred |
| D9 | Storage | In-memory only in v1; no AsyncStorage, SQLite, or SecureStore | **Accepted** (PB-05) |
| D10 | Testing | Jest for pure logic, RNTL for components/integration, Maestro for E2E | **Accepted** (Maestro, 2026-11-10) |
| D11 | Theming | Single light theme from ported CSS tokens; no dark mode | **Accepted** (UX-03) |
| D12 | WebView/DOM | Not used anywhere; the UI is rebuilt natively | Constraint |
| D13 | Existing web app | Left unchanged in the repository during the migration | [Recommendation]; web continues in parallel (PB-10) |
| D14 | Connectivity | Standalone, fully offline app; bundled content; no backend API, sync, or authentication | **Accepted** (API-01, OFF-01, SEC-01, SEC-02) |
| D15 | App identity | "Equation Explorer"; bundle/package `net.bbsteps.equation`; scheme `equation`; public URL `equation.bbsteps.net` | **Accepted** (DEP-02) |

---

## 1. Architecture principles

1. **The domain is pure and framework-free.** Rational arithmetic, tokenization, parsing, equation analysis, candidate validation, and formatting live in plain TypeScript with no React or React Native imports. They can be tested without a renderer and could later be reused by the web app or a backend.
2. **State is explicit and serializable.** The session reducer owns everything that defines a learner's progress. Rational values are not JSON-serialized; if persistence is added, encode them as decimal strings.
3. **Ephemeral interaction state stays out of the reducer.** Drag translation, confetti animation values, and press feedback live in components/Reanimated shared values, so per-frame updates never cause React renders.
4. **The UI is a projection of state.** Screens render from reducer state and dispatch intent; they do not mutate domain objects or keep duplicate copies of session data.
5. **Native capabilities sit behind narrow façades.** Audio, haptics, accessibility announcements, and gestures are wrapped so features do not import platform modules directly and tests can mock one boundary.
6. **Accessibility is part of every component contract.** Every interactive element has a role, label, state, and an accessible alternative to pointer-only gestures.
7. **No speculative dependencies.** The app is standalone and offline with no server, auth, persistence, analytics, or crash reporting (API-01, SEC-01, PB-05, AN-01, AN-02), so there is no reason for a data-fetching, state-management, storage, monitoring, or UI-kit library.
8. **The web remains the behavioral reference.** Porting mistakes are caught by the golden corpus, not by visual comparison alone.

---

## 2. System context

```
┌─────────────────────────────────────────────────────────────┐
│                        Mobile app                            │
│                                                              │
│  Expo Router shell (src/app)                                 │
│    └── SessionProvider (useReducer + Context)                │
│          ├── SetupScreen                                     │
│          └── SolverScreen                                    │
│                ├── history / active equation                 │
│                ├── token palette + toolbar                   │
│                ├── drag layer (Gesture Handler/Reanimated)   │
│                └── feedback (toast, audio, haptics, confetti)│
│                                                              │
│  Domain engine (pure TS): rationals → tokenizer → parser →   │
│  affine reduction → source/candidate analysis → formatting   │
└─────────────────────────────────────────────────────────────┘
        │                                    │
        │ no network                         │ OS capabilities
        ▼                                    ▼
   (nothing)                    audio playback, haptics,
                                accessibility APIs, gestures
```

- **[Observed]** The existing app has no backend, API client, authentication, account, storage, analytics, or external service (`index.html`, whole file; `README.md:19-20`). The mobile architecture therefore has no server-state layer.
- **[Observed]** The app is a single user solving equations in one browser tab. There is no multi-user, sharing, or sync concept.
- **[Decision]** The mobile app is standalone and fully offline, including first launch (OFF-01); there is no backend API and no authentication (API-01, SEC-01). Any architecture that introduces a network dependency would be adding scope, not migrating it.

### 2.1 Runtime boundaries

| Boundary | Inside | Outside |
| --- | --- | --- |
| App process | Session state, domain engine, UI | Nothing required |
| Device capabilities | Audio playback, haptics, screen reader, gestures | OS / Expo modules |
| Network | Not used | Not used |
| Persistence | Process memory only | Not used in v1 |

---

## 3. Application shell and bootstrap

### 3.1 Entry point

- **[Recommendation]** Use Expo Router with `src/app` as the route root. `mobile/package.json` sets `"main": "expo-router/entry"`.
- **[Recommendation]** `src/app/_layout.tsx` is the only root layout. It composes, from outside in:
  1. `GestureHandlerRootView` (must wrap the app for Gesture Handler).
  2. `SafeAreaProvider`.
  3. `SessionProvider`.
  4. `RootErrorBoundary`.
  5. `StatusBar` with the chosen light/dark style.
  6. A `Stack` navigator with `screenOptions={{ headerShown: false }}`.
- **[Recommendation]** Hold the splash screen until the first layout is ready, then hide it. There is no async bootstrap work beyond loading the JS bundle, so the splash should be brief.
- **[Recommendation]** Set the root view background to the theme page color in app config (`backgroundColor` / `expo-system-ui`) to avoid a white flash.

### 3.2 Bootstrap sequence

```
native launch
  → Expo runtime loads JS bundle
  → root layout mounts providers
  → session reducer initializes to SETUP with an empty draft
  → router renders the single route
  → SetupScreen renders; splash hides
```

There is no token restore, remote config fetch, database open, or font load. **[Recommendation]** Keep it that way; adding any of these must be justified against the migration plan's prerequisites.

### 3.3 Root error boundary

- **[Recommendation]** A class component error boundary at the root catches render errors, displays a simple recovery screen ("Something went wrong" + "Start again"), and resets the session/route.
- **[Recommendation]** Log the error in development only. Do not send it anywhere; crash reporting is out of scope (AN-02).
- **[Inferred]** Domain functions should not throw for expected invalid input; they return discriminated results. Throws are reserved for programmer errors (impossible states), which the boundary catches.

---

## 4. Navigation

### 4.1 Observed navigation behavior

- **[Observed]** The web app has no routes, URLs, history entries, or navigation UI. It renders a setup card and a solving session in one page and toggles them via state (`state.mode`; `index.html:1362-1378`, `:2172-2230`).
- **[Observed]** Reset returns the learner to the landing/setup state and clears the session (`resetSession`, `index.html:2210-2230`).

### 4.2 Recommendation: one visible route, phase-driven screens

- **[Recommendation]** Register one route, `src/app/index.tsx`, which renders `SetupScreen` when `session.phase === 'setup'` and `SolverScreen` when the session is active or complete.
- Keep `src/app/+not-found.tsx` for defensive deep-link handling.
- Do **not** push a second route for the solver in v1.
- Do **not** add tabs, drawers, modals, or a header.

Why:

- The source has no navigation model; introducing a stack creates new back-stack semantics that the product has not specified.
- Back behavior is decided (UX-08): a back action must preserve the in-memory session, never discard it.
- A single route preserves the exact "setup ↔ session" transition and keeps session state in one provider without route-lifecycle questions.
- A simple cross-fade or no transition is appropriate; a stack animation would imply a navigable place to go back to.

Consequences:

- **[Decision]** While a session is active, intercept Android hardware back and keep the learner on the solver; the session is unchanged. On setup, back exits normally. There is no in-app iOS back gesture with a single route. Starting over is an explicit Reset, not a back action.
- **[Open]** If the product later wants a shareable/deep-linked equation or a resume screen, add a second route then; the domain and reducer are unaffected.

### 4.3 Alternative considered

| Option | Why not chosen for v1 |
| --- | --- |
| Two routes (`/` and `/solve`) with router push/back | Adds a back-stack and session-lifecycle surface the product does not need; back must preserve the session (UX-08), which is simpler with no route to pop |
| Tabs/drawer | No multiple sections in the product |
| Modal for completion | The completion panel is part of the solving surface in the source (`index.html:562-576`); a modal would change the flow |

### 4.4 Route map

| Route | File | Renders |
| --- | --- | --- |
| `/` | `src/app/index.tsx` | `SetupScreen` or `SolverScreen` based on session phase |
| unmatched | `src/app/+not-found.tsx` | Simple "page not found" with a link back to `/` |

No route params, query strings, or deep-link data carry learner content.

---

## 5. Feature boundaries

| Feature area | Responsibility | May import | Must not import |
| --- | --- | --- | --- |
| `features/equation/domain` | Rationals, tokenizer, parser, affine reduction, source analysis, candidate analysis, formatting, shared domain types | Standard TS only (`bigint`, no DOM, no RN) | React, React Native, Expo, session state, theme |
| `features/session` | Workspace token model, row model, session reducer, actions, selectors, signatures/duplicate rules, engine bridge | `features/equation/domain`, shared pure utilities | React Native components, Expo modules, theme |
| `features/setup` | Setup screen, equation input, validation display, start action | `features/session`, `shared/ui`, `shared/theme`, `shared/a11y` | Domain internals beyond public API |
| `features/solver` | Solving screen composition, history, active equation, palette, toolbar, drag, completion panel | `features/session`, `features/equation/domain` selectors, `shared/ui`, `shared/theme`, `shared/native`, `shared/a11y` | Direct Expo audio/haptics calls; direct domain mutations |
| `shared/ui` | Presentational primitives (`AppText`, `Button`, `Card`, `Screen`, `Icon`, `IconButton`) | React Native, theme | Feature state, domain |
| `shared/theme` | Colors, spacing, radii, typography, shadows, motion durations | React Native types only | Feature state |
| `shared/native` | Audio, haptics, feedback orchestration façades | Expo modules | Domain, feature state (accepts plain callbacks/values) |
| `shared/a11y` | Announcements, focus helpers, reduced-motion access | React Native, Reanimated | Feature state |
| `shared/config` | Constants (limits, durations, palette definition) | Nothing | Feature state |
| `app` | Routing, providers, error boundary | All of the above | Domain internals beyond public API |

**[Recommendation]** Enforce boundaries with ESLint `import/no-restricted-paths` or a simple review rule. The most important invariant is that `features/equation/domain` imports no React Native code; this keeps the engine testable and portable.

---

## 6. Screens and component boundaries

### 6.1 Setup screen

| Component | Responsibility | Props / state |
| --- | --- | --- |
| `SetupScreen` | Compose the landing page; submit to session | reads session, dispatches start |
| `BrandHeader` | App name, variable badge, short subtitle | static |
| `SetupCard` | Card container, form semantics | children |
| `EquationField` | Controlled `TextInput`; validation message | `value`, `onChangeText`, `onSubmit`, `error`, `inputRef` |
| `StartButton` | Primary action; disabled when empty | `onPress`, `disabled` |
| `TipCard` | Notation help | static |
| `ValidationMessage` | Error text with accessible announcement | `message` |

### 6.2 Solver screen

| Component | Responsibility | Props / state |
| --- | --- | --- |
| `SolverScreen` | Compose history/active/toolbox; orchestrate check | reads session, dispatches |
| `SolverHeader` | Title and reset action | `onReset` |
| `LockedHistory` | Renders immutable accepted rows | `rows` |
| `LockedRow` | One accepted equation with step number | `row` |
| `ActiveEquationCard` | Active row, step label, side selector, token sides, toolbar | session slice |
| `SideSelector` | Choose left/right side | `selectedSide`, `onSelect` |
| `TokenSide` | Wrap tokens for one side; drop target | `side`, `tokens`, `selectedId`, handlers |
| `TokenChip` | Pressable token; selected/drag/drop states | `token`, `selected`, handlers |
| `TokenPalette` | 18 insertion buttons | `onInsert` |
| `WorkspaceToolbar` | Move left/right, move to other side, delete, clear, restore, flip | state flags + handlers |
| `CheckButton` | Submit active equation | `onPress`, `disabled` |
| `FeedbackToast` | Transient rejection message | `message`, visibility |
| `CompletionPanel` | Verified answer, "Solve another!" | `answer`, `onReset` |
| `ConfettiOverlay` | Celebration; reduced-motion aware | `active` |

### 6.3 Shared primitives

| Component | Contract |
| --- | --- |
| `Screen` | Safe-area + scroll container + background + content width |
| `AppText` | Theme typography; maps `variant` to style; never a bare string outside `Text` |
| `Button` | Primary/secondary/quiet variants; accessibility role/state; min 44/48 dp target |
| `IconButton` | Accessible label required; `hitSlop` when visual is small |
| `Card` | Surface, border, radius, shadow/elevation tokens |
| `Icon` | Wraps the chosen vector icon set behind a project-owned name map |
| `Divider` | Token-based separator |

### 6.4 Component rules

- **[Recommendation]** Presentational components receive data and callbacks; they do not read the session context. Container screens read context and pass slices down. This keeps RNTL tests simple and avoids hidden coupling.
- **[Recommendation]** Token chips are the only components that combine gesture, animation, and accessibility state. That complexity is isolated in `features/solver/components/TokenChip`.
- **[Recommendation]** No component imports `expo-audio` or `expo-haptics` directly; they call the `shared/native/feedback` façade.

---

## 7. State ownership

### 7.1 State catalogue

| State | Owner | Shape | Persisted? | Notes |
| --- | --- | --- | --- | --- |
| Session phase | Session reducer | `'setup' \| 'solving' \| 'complete'` | No | Derived from rows is possible but explicit is clearer |
| Draft equation input | Setup screen local state or reducer | `string` | No | Local while typing; sent on start |
| Submitted/source equation | Session reducer | `string \| null` | No | Immutable after start |
| Solution | Session reducer | `Rational \| null` | No | Exact value from source analysis |
| Locked rows | Session reducer | `LockedRow[]` | No | Append-only |
| Active row | Session reducer | `ActiveRow \| null` | No | Only editable row |
| Selected side | Session reducer | `'left' \| 'right'` | No | Defaults left |
| Selected token id | Session reducer | `string \| null` | No | Cleared on lock/reset |
| Feedback | Session reducer | `Feedback \| null` | No | Message + tone + optional side |
| Last accepted check | Session reducer | signature record | No | Duplicate/same-step guard |
| Source step signature | Session reducer | `string \| null` | No | Detects returning to source |
| Celebration played | Session reducer | `boolean` | No | Ensures one celebration |
| Drag translation / target | `TokenChip` + Reanimated shared values | UI-thread values | No | Never in reducer |
| Confetti progress | `ConfettiOverlay` | Reanimated values | No | Not in reducer |
| Toast timer | `FeedbackToast` effect | timer id | No | Auto-dismiss 2600 ms, fade 200 ms |
| Keyboard visibility | Screen local state if needed | `boolean` | No | Platform behavior |

### 7.2 Why `useReducer` + Context and not a state library

**[Recommendation]** The complete session is one cohesive state machine with a small, well-defined action set. `useReducer` provides:

- atomic transitions, which match the "lock a row" and "reject a candidate" semantics;
- pure, testable transition functions;
- no extra dependency, matching the project's minimal-dependency principle.

A state library would add indirection and bundle size without solving a demonstrated problem. If profiling later shows broad context re-renders, split `SessionStateContext` and `SessionDispatchContext`, then memoize slices; do not add a library first.

### 7.3 Reducer actions

| Action | Payload | Effect |
| --- | --- | --- |
| `SET_DRAFT` | `string` | Update setup input |
| `START_SESSION` | `{ equation, sourceAnalysis }` | Validate again, create source locked row + prefilled active row, phase `solving` |
| `SET_SIDE` | `'left' \| 'right'` | Change selected side; clear or retain selection per product decision |
| `SELECT_TOKEN` | `tokenId \| null` | Select/deselect a token |
| `APPEND_TOKEN` | `paletteToken` | Append to selected side (faithful to the web behavior) |
| `DELETE_TOKEN` | `tokenId` | Remove from its side; no-op if missing |
| `MOVE_TOKEN` | `{ tokenId, direction: -1 \| 1 }` | Reorder within side; clamped |
| `MOVE_TOKEN_TO_SIDE` | `tokenId` | Move to the opposite side |
| `CLEAR_ACTIVE` | — | Empty both active sides |
| `RESTORE_ACTIVE` | — | Restore active row from the last locked row |
| `FLIP_ACTIVE` | — | Swap left and right sides |
| `APPLY_CHECK` | `CheckOutcome` | Accept/lock/complete or reject with feedback |
| `DISMISS_FEEDBACK` | — | Clear feedback |
| `CELEBRATION_DONE` | — | Mark celebration played |
| `RESET_SESSION` | — | Return to setup and clear all session state |

**[Recommendation]** The screen's event handler calls the domain engine first, then dispatches `APPLY_CHECK` with a discriminated outcome. The reducer remains pure and does no parsing.

### 7.4 Session state machine

```
                 START_SESSION (valid source)
   SETUP ───────────────────────────────────────► SOLVING
     ▲                                              │  │
     │                                              │  │ APPLY_CHECK(accepted, final)
     │ RESET_SESSION                                │  ▼
     └──────────────────────────────────────────── COMPLETE
                                                    │
                                                    │ RESET_SESSION
                                                    └──► SETUP

   SOLVING:
     edit actions  → active row changes, feedback cleared appropriately
     APPLY_CHECK(rejected)  → feedback set; rows unchanged
     APPLY_CHECK(accepted, non-final) → active row locked; new active row prefilled
     APPLY_CHECK(accepted, final)     → final row locked; phase COMPLETE; celebration
```

Invariants:

1. Locked rows are append-only and never edited.
2. The active row always belongs to the current step and never mutates a locked row.
3. Acceptance requires the active equation to have the same exact solution as the source.
4. Only the final form — one `y` token on the left, no `y` on the right, right value equal to the source solution — completes the session.
5. A reset clears every session field, including signatures, feedback, and celebration.
6. A rejected check never changes history.
7. The source row is always the first locked row.

### 7.5 Derived state and selectors

Compute, do not store:

- `activeLeftTokens` / `activeRightTokens`;
- `selectedToken` and its side/index;
- `canMoveLeft`, `canMoveRight` (selection exists and index bounds);
- `canMoveToOpposite`, `canDelete`;
- `stepNumber = lockedRows.length + 1`;
- `isComplete`;
- `finalAnswerText` (from the source solution, formatted);
- `feedbackVisible` (feedback not null).

Use small pure selector functions in `features/session/selectors.ts`; memoize at the screen only if profiling shows a cost.

### 7.6 Signatures and duplicate guards

**[Observed]** The web app tracks a source-step signature and a last-accepted signature, and enforces a 500 ms duplicate-action window (`index.html:587`, `:2007-2055`).

**[Recommendation]** Model these as explicit pure helpers:

- `tokenSignature(tokens)` — stable serialization of glyphs/ids for display equality.
- `equationSignature(left, right)` — canonical string used for same-as-previous checks.
- `isSameAsPrevious(candidate, lockedRows)` — token-identical row compared with the last locked row.
- `isSourceStep(signature, sourceSignature)` — rejects returning to the initial equation.
- `lastAcceptedCheck` stores the signature and timestamp; Check ignores a second press within 500 ms.

Keep the exact rules and comparison semantics ported from source; do not "simplify" them without golden tests.

---

## 8. Domain engine design

### 8.1 Module layout

```
features/equation/domain/
  rational.ts            # Rational type, GCD, normalize, add/sub/mul/div, negate, zero, equality
  tokenize.ts            # string → source tokens; limits; Unicode operators; implicit multiplication markers
  parser.ts              # recursive-descent parser → AST; precedence; unary; parentheses
  ast.ts                 # AST types and helpers
  affine.ts              # reduce an AST to (coefficient, constant) over Rational
  reciprocal.ts          # narrow direct-reciprocal source exception
  source-analysis.ts     # classify source equation; solve; identity/contradiction/unsupported
  candidate-analysis.ts  # validate an active equation against the source solution; final-form predicate
  format.ts              # canonical expression/answer formatting (÷ for fractions, etc.)
  result.ts              # discriminated result types and error/rejection categories
  types.ts               # shared domain types (Rational, SourceToken, EquationStatus, ...)
```

No barrel `index.ts` files; import from the specific module (keeps bundles and intent clear, per the React Native best-practices skill).

### 8.2 Rational representation

```ts
/** Canonical: denominator > 0, gcd(|numerator|, denominator) = 1, zero is 0/1. */
export type Rational = Readonly<{
  numerator: bigint;
  denominator: bigint;
}>;

export type BinaryOp = 'add' | 'subtract' | 'multiply' | 'divide';
```

Rules:

- Never use binary floating point for equation values.
- Normalize on every construction; expose factory functions rather than public mutable objects.
- Division by zero returns a discriminated error, never `Infinity`/`NaN`.
- Do not serialize `Rational` with `JSON.stringify` (BigInt throws). If persistence is added, use `numerator.toString()` / `denominator.toString()`.
- **PoC required:** verify `BigInt` runtime behavior on Hermes on both platforms, including large numerators from repeated multiplication, before building on it.

### 8.3 Parsing pipeline

```
input string
  → reject over MAX_INPUT_LENGTH
  → tokenize (whitespace, integer, decimal, fraction, operators, parentheses, y)
  → reject unsupported symbols / malformed numbers / token overflow
  → split at a single '=' into left and right expression token lists
  → recursive-descent parse each side
  → affine reduction (a·y + b) on each side
  → source classification
```

**[Recommendation]** Preserve the source's acceptance rules exactly:

- integers, terminating decimals, `÷` or `/` fractions, parentheses, `+`, `-`, `×`/`*`, implicit multiplication (`2y`), and `y`;
- one variable, unique solution;
- the narrow direct-reciprocal exception (`c / y = d` or `d = c / y`, nonzero constants) only;
- identity, contradiction, nonlinear, unsupported rational, and malformed input are invalid with a specific reason.

### 8.4 Result types

Target discriminated unions (names illustrative; the ported implementation should derive them from the source's fields):

```ts
export type RejectionReason =
  | 'empty'
  | 'too-long'
  | 'malformed'
  | 'unsupported-symbol'
  | 'not-linear'
  | 'unsupported-rational'
  | 'divide-by-zero'
  | 'identity'
  | 'contradiction'
  | 'not-equivalent'
  | 'same-as-previous'
  | 'source-step'
  | 'not-final-form';

export type SourceAnalysis =
  | { kind: 'solvable'; solution: Rational; display: { left: string; right: string } }
  | { kind: 'invalid'; reason: RejectionReason };

export type CandidateAnalysis =
  | { kind: 'accepted'; isFinal: boolean; display: { left: string; right: string } }
  | { kind: 'rejected'; reason: RejectionReason; side?: 'left' | 'right' };
```

**[Recommendation]** Map `RejectionReason` to message copy in one place (`features/equation/messages.ts` or `shared/config/messages.ts`), ported verbatim from the source `MATH_MESSAGES` catalog (`index.html:603-619`). Never construct user-facing copy inside the parser.

### 8.5 Domain invariants and tests

- Pure functions; no I/O, no clock, no randomness, no locale.
- Every exported function has a unit test for its success and failure paths.
- The 70-case `runMathSelfChecks()` corpus (`index.html:1104-1292`) becomes a Jest table test, unchanged except for the variable name where the source uses `y`.
- The spec's release examples are additional cases.
- Add property-style tests for rational arithmetic (normalization, inverse operations) if cheap; optional.

### 8.6 Formatting

- **[Observed]** The source has a dedicated formatter (`formatAst`, `index.html:971-998`) and displays fractions with `÷`.
- **[Recommendation]** Keep display formatting in the domain so history, active rows, final answer, and tests share one canonical representation.
- Do not use `Intl.NumberFormat` or locale-dependent formatting; the math notation is fixed.
- Unicode minus (`−`) vs ASCII hyphen: preserve the source's glyph choices consistently and use them as test fixtures.

---

## 9. Data models

### 9.1 Workspace model

```ts
export type Side = 'left' | 'right';

export interface WorkspaceToken {
  readonly id: string;              // unique per row, not per glyph
  readonly kind: 'number' | 'variable' | 'operator' | 'paren';
  readonly label: string;           // display glyph(s), e.g. '2', '.', 'y', '+', '−'
  readonly insertText: string;      // text contributed to the serialized equation
}

export interface Row {
  readonly id: string;
  readonly tokens: Readonly<Record<Side, readonly WorkspaceToken[]>>;
}

export interface LockedRow extends Row {
  readonly stepNumber: number;
  readonly display: string;         // canonical formatted equation, if useful
}

export interface ActiveRow extends Row {
  // Editable; same shape, different lifecycle
}
```

**[Recommendation]** Two token representations are intentional:

- `SourceToken` (domain): produced by the tokenizer from a string; used for parsing.
- `WorkspaceToken` (session/UI): produced when the source is displayed and when palette items are inserted; carries a stable `id` for selection/drag.

Bridge them with an adapter that serializes a side's tokens to text (respecting implicit multiplication), then call the domain tokenizer. Golden tests must confirm that serialize→tokenize preserves the source's acceptance and limit behavior. Do not let a workspace token bypass domain validation.

### 9.2 Row and selection model

- A row has exactly two sides; an empty side is valid while editing.
- Token order is array order within a side.
- Selection is `{ side, tokenId }` or `null`; store only the id in state and derive side/index.
- Moving between sides appends to the destination (or inserts at a chosen index during drag). Define and test the index rule.
- Locking a row freezes its tokens; the new active row starts as a copy with fresh ids.

### 9.3 Palette model

**[Observed]** `PALETTE_TOKENS` has 18 entries (`index.html:597-602`).

```ts
export interface PaletteToken {
  readonly key: string;
  readonly label: string;
  readonly insertText: string;
  readonly accessibilityLabel: string; // e.g. "Insert 2", "Insert variable y"
}
```

**[Recommendation]** Generate accessibility labels from the token metadata, not handwritten per button; keep special cases (minus, multiply, divide, parentheses) in a small map.

---

## 10. Server state, API client, and authentication

| Concern | Status | Architecture consequence |
| --- | --- | --- |
| REST/GraphQL API | None [Observed] | No API client, no base URL, no interceptors, no loading/error network states |
| Server state cache | None | No React Query/SWR/axios |
| Authentication | None [Observed] | No auth provider, token storage, refresh, or route guard |
| Session/account | None | One local learner; no identity |
| Payments/subscriptions | None | No IAP/RevenueCat |
| Push notifications | None | No notification permission or token handling |

**[Decision]** The app is standalone and fully offline with no backend API and no authentication (API-01, OFF-01, SEC-01, SEC-02, 2026-11-10). Content is bundled; nothing is fetched or synced.

**[Recommendation]** Do not create placeholder `api/` or `auth/` folders. Empty abstraction layers invite accidental scope. Document the "none" decision in the architecture and revisit only if the product adds accounts or sync.

---

## 11. Storage

- **[Decision]** v1: process memory only (PB-05). No storage dependency, no migration, no privacy surface.
- The session survives backgrounding only as long as the OS keeps the process; process death loses it. This is accepted behavior, not a defect.
- Back preserves the in-memory session (UX-08); Reset is the only explicit discard path. Do not add an automatic resume prompt. Process death still loses the session (PB-05).
- If resume is ever required, persist `{ sourceEquation, lockedRows }` as plain strings/tokens — never `BigInt` — and rehydrate by re-running the engine. Validate the restored equation on load and fall back to setup on failure.
- If preferences are added (sound/haptics), use one documented non-sensitive key-value store (AsyncStorage or `expo-sqlite/localStorage`). Do not introduce both.
- Never store tokens, secrets, or personal data in plain storage. No secrets exist today.

---

## 12. Error handling

### 12.1 Expected domain outcomes

- Expected invalid input is data, not an exception: discriminated `SourceAnalysis` / `CandidateAnalysis` results.
- UI maps each rejection reason to a message and, where known, the affected side.
- Rejected checks leave history untouched and keep the active row editable.

### 12.2 Unexpected errors

- Root error boundary catches render/lifecycle errors and offers "Start again".
- Developer errors (impossible reducer state, missing active row while solving) fail loudly in development via an `assertNever`/invariant helper; in production the boundary catches them.
- Never show a raw stack trace or internal error message to a child.

### 12.3 Native capability errors

| Capability | Failure behavior |
| --- | --- |
| Audio playback | Swallow the error; log in development; never block completion |
| Haptics | Swallow; not supported on all devices |
| Accessibility announcement | Swallow; visual feedback remains |
| Drag gesture cancellation | Return the token to its origin; state unchanged |
| Splash hide | Hide in a `finally` so a failure cannot trap the learner on the splash |

**[Recommendation]** Wrap native calls in the `shared/native` façade with `try/catch` and a single dev-only log. No alert dialogs for non-critical failure.

### 12.4 Error surfaces

- Setup validation: inline message below the field plus screen-reader announcement.
- Active-row rejection: inline feedback near the active card plus toast; never modal.
- Completion success: panel + announcement + required sound (PB-06) + optional confetti/haptics; no blocking alert.
- Fatal error: full-screen boundary, one recovery action.

---

## 13. Forms and validation

### 13.1 Equation field

| Concern | Recommendation |
| --- | --- |
| Control | Controlled `TextInput` with local draft state; dispatch only on submit |
| Keyboard | `autoCapitalize="none"`, `autoCorrect={false}`, `spellCheck={false}`, `returnKeyType="go"` |
| Limit | Enforce `MAX_INPUT_LENGTH = 512` at the field and again in the engine |
| Submit | Start button and keyboard Return both call the same handler |
| Empty state | Start disabled when trimmed input is empty |
| Invalid state | Keep input, show the specific reason, focus the field, announce the error |
| Valid state | Clear feedback, start the session |
| Paste | Accepted; parsed exactly like typed input |
| Validation timing | On submit, not on every keystroke (matches the source's clear error flow) [Inferred] |

- **[Inferred]** Do not use a form library. One field and one submit action do not justify it.
- **[Recommendation]** The equation input should not be a `Text` display; learners type on a normal keyboard. The token workspace remains the structured editor after start.

### 13.2 Active-row validation

- Check is disabled only when it cannot possibly help (for example, a busy state); malformed rows still produce the specific message when checked.
- Validation runs in the domain, not in components.
- The affected side is highlighted when the domain can identify it.
- The same error message is used for a tap and a drag/drop attempt that produces an invalid row.

### 13.3 Accessibility of validation

- Error text is not color-only; it includes an icon and text.
- iOS: call `announceForAccessibility` with the error.
- Android: `accessibilityLiveRegion="polite"` on the message container (or announcement).
- Mark the field with `accessibilityHint` describing the accepted notation; native has no `aria-invalid`.

---

## 14. Theme and styling

### 14.1 Token translation

Port the CSS custom properties (`index.html:10-29`) into a typed token module:

```ts
export const colors = {
  page: '#fff9ef',
  primary: '#4338ca',
  teal: '#0f766e',
  coral: '#c2410c',
  success: '#15803d',
  error: '#b91c1c',
  variable: '#b91c1c',
  focus: '#f59e0b',
  // surface, text, muted, border, disabled, ... ported from source
} as const;
```

**[Recommendation]** Preserve the confirmed palette and semantic roles; extract every remaining value from the CSS rather than approximating. Add `colors.overlay`, `colors.disabled`, and `colors.surfaceAlt` only if the source needs them.

| Web construct | Native approach |
| --- | --- |
| CSS custom properties | Typed `as const` token objects |
| `clamp()` type sizes | Numeric sizes with `useWindowDimensions` only if needed; prefer fixed dp + font scaling |
| `px` values | Unitless dp numbers |
| `rem`/`em` | Not available; use a type scale |
| Media queries (600/900) | `useWindowDimensions` width checks; the tablet layout is the primary target and the compact layout is the phone fallback |
| Gradients | Avoid or use `expo-linear-gradient` only if design requires; not a v1 dependency |
| Box shadows | iOS `shadowColor/Offset/Opacity/Radius`; Android `elevation`; `Platform.select` |
| `position: sticky` toolbox | Fixed layout region or `stickyHeaderIndices` in a ScrollView; decide in UI implementation |
| `@keyframes` | Reanimated |
| `prefers-reduced-motion` | Reanimated `useReducedMotion` |
| Focus ring | Explicit pressed/focused style on `Pressable`; no CSS `:focus` |
| `env(safe-area-inset-*)` | `useSafeAreaInsets` |

### 14.2 Theme structure

```
shared/theme/
  colors.ts
  spacing.ts
  radii.ts
  typography.ts
  shadows.ts
  motion.ts
  tokens.ts      # composed export; no React
  useTheme.ts    # optional hook; returns tokens + insets/dimensions as needed
```

- **[Decision]** Single light theme only (UX-03). Set the app's `userInterfaceStyle` to `light` so the OS does not invert or tint native surfaces unexpectedly. No dark-mode tokens or system-theme handling.
- Do not add a theming library. A static token object plus `StyleSheet.create` is sufficient.
- **Contrast:** verify text/background, muted text, borders, disabled states, focus, and token colors against WCAG AA. The web's focus/border contrast concerns carry over; fix them in the token design, not per screen.
- **Typography:** system fonts; no font assets. Support Dynamic Type/font scaling. Use `maxFontSizeMultiplier` only for tiny, non-content labels if layout breaks; never cap body content globally.
- **Touch targets:** minimum 44×44 pt / 48×48 dp; use `hitSlop` without shrinking the visual.
- **Shadows:** subtle; Android needs `elevation` for stacking and shadow.

---

## 15. Assets

| Asset | Source today | Mobile plan |
| --- | --- | --- |
| App icon | Favicon data URI (`index.html:7`) | Create a 1024×1024 PNG; adaptive Android foreground/background; no text baked in |
| Splash | None | Simple splash with page color and mark; `expo-splash-screen` |
| Brand mark | Emoji + text (`index.html:467`) | Prefer a simple vector/PNG mark or styled text; emoji only if intentional |
| Icons | Emoji/Unicode (`index.html:541`, `:571`) | Use `@expo/vector-icons` for essential actions; keep emoji decorative only |
| Audio | Web Audio synthesis (`index.html:1602-1664`) | Bundle short chimes (m4a/wav) created/owned by the project; no runtime synthesis. Sound is required (PB-06), so this is not optional scope |
| Confetti | CSS/HTML pieces | Reanimated views; no image asset required |
| Fonts | System stack | System fonts; no font assets |
| Images | None besides favicon | None |
| Localization | English strings inline | Keep English in v1; centralize strings for future localization |

- **[Recommendation]** Keep `assets/` at the project root (`mobile/assets/`) for app-config assets and runtime `require()` imports. Do not mix generated build artifacts into source control.
- **[Decision]** Sound is required (PB-06). The iOS silent switch mutes and Android follows media volume (NAT-10). **[Open]** Audio licensing/ownership; mute/haptics controls are deferred (NAT-04).
- **[Open]** Final icon/splash design and brand approval.

### 15.1 App identity (DEP-02)

| Setting | Supplied value | Configuration note |
| --- | --- | --- |
| Display name | Equation Explorer | `expo.name` / store listing |
| Bundle identifier (iOS) | `net.bbsteps.equation` | Reverse-DNS for `bbsteps.net` |
| Package name (Android) | `net.bbsteps.equation` | Same identifier; unique and immutable once published |
| Public URL | `equation.bbsteps.net` | Hostname/universal-link host; not a URL scheme |
| URL scheme | `equation` | Confirmed; `equation.bbsteps.net` would be an invalid scheme |

**Identity values are permanent after the first store upload.** Create the store records with `net.bbsteps.equation`.

---

## 16. Native module boundaries

| Capability | Expo module | Project façade | Used by | Fallback / note |
| --- | --- | --- | --- | --- |
| Drag on touch-move | `react-native-gesture-handler` | `TokenChip` gesture composition | Solver only | Tap/toolbar editing is always available |
| Animation | `react-native-reanimated` | Component-local shared values | Token drag, confetti, transitions | Reduced motion disables/shortens |
| Audio | `expo-audio` | `shared/native/audio.ts` | `feedback` façade | Silent if unavailable or muted |
| Haptics | `expo-haptics` | `shared/native/haptics.ts` | `feedback` façade | Silent no-op on unsupported devices |
| Screen reader | React Native accessibility APIs | `shared/a11y/announce.ts`, focus helpers | Feedback, lock, completion | Visual feedback remains |
| Safe area | `react-native-safe-area-context` | `Screen` | All screens | Insets from provider |
| Screens/navigation | `expo-router`, `react-native-screens` | `src/app` | Shell | Single route |
| Splash | `expo-splash-screen` | Root layout | Bootstrap | Hide defensively |
| Status bar | `expo-status-bar` | Root layout | Shell | Style from theme |
| Icons | `@expo/vector-icons` | `shared/ui/Icon` | Buttons, feedback | Text/emoji fallback if not used |

**[Recommendation]** No custom native modules, config plugins, or prebuild-only changes are expected. If one becomes necessary, it changes the delivery model; escalate before implementation.

**[Recommendation]** `shared/native/feedback.ts` exposes intent-level functions (`playSuccess`, `playError`, `playTap`, `flashSuccess`, `flashError`) and internally decides audio/haptics. Features never know which module fired.

---

## 17. Logging and diagnostics

- **[Decision]** No analytics and no crash reporting in v1 (AN-01, AN-02, 2026-11-10). No telemetry SDK, event schema, or reporting endpoint.
- **[Recommendation]** Use `__DEV__` guards. Production logs are silent by default.
- Never log raw learner input, full session state, or anything that could identify a child.
- The math self-check stays a Jest suite. Do not ship a callable global self-check in production; if a developer menu is useful, expose it only in development builds.
- React Native DevTools/Metro logs and development builds are the diagnostic path; field issues come from user reports.
- If crash reporting is ever approved, reverse this decision explicitly, integrate a native-compatible SDK behind a small façade, scrub input, and update the privacy disclosure.
- Track the checklist results (device matrix, accessibility passes) in the repo, as the web project did in `tasks/04-todo.md`.

---

## 18. Accessibility architecture

### 18.1 Component contract

Every interactive component must define:

- `accessibilityRole` (`button`, `header`, `text`, `summary`? — use the closest native role);
- `accessibilityLabel` that includes the operation and the token, not just the glyph;
- `accessibilityState` (`selected`, `disabled`);
- `accessibilityHint` where the outcome is not obvious;
- a minimum 44/48 dp target;
- a visible pressed/focus style that is not color-only.

### 18.2 Tokens and equations

- Each token is a `Pressable` with a label such as "2, left side, position 1 of 3, selected".
- The side container has a label ("Left side of the equation, 3 tokens").
- The active equation has a summary label built from the tokens ("2 y minus 3 equals 8") so a screen-reader user can hear the whole equation without traversing every glyph.
- The history rows are read-only text; each row includes its step number.
- Avoid `accessible={true}` on a container that hides interactive children; group only non-interactive text.

### 18.3 Alternatives to gesture

- `accessibilityActions` on a selected token: "Move left", "Move right", "Move to other side", "Delete".
- The toolbar provides the same operations visually and is fully focusable.
- Drag is never the only path. This is required for screen readers and external keyboards.
- **Both** paths are required (NAT-03): screen-reader users get `accessibilityActions`; hardware-keyboard users get a logical focus order through palette, tokens, and toolbar with Tab/Shift+Tab, activation with Enter/Space, and a visible focus ring. External keyboard support does not rely on `accessibilityActions`, which screen readers expose but hardware Tab does not.

### 18.4 Announcements and focus

| Event | Announcement | Focus |
| --- | --- | --- |
| Invalid setup | Specific reason | Move focus to the field |
| Rejected check | Specific reason + side | Keep focus on the active row/Check |
| Row accepted | "Step accepted. Step N is ready." | Move to the new active row |
| Completion | "Solved. y equals …" | Move focus to the completion heading |
| Reset | "New equation. Enter an equation to start." | Move focus to the input |

- iOS: `AccessibilityInfo.announceForAccessibility`.
- Android: `accessibilityLiveRegion` on a persistent text node, or the same announcement API.
- Test with VoiceOver and TalkBack; assertion in RNTL that the façade was called.

### 18.5 Motion, type, and color

- `useReducedMotion` from Reanimated; skip confetti and shake, keep a static success/error treatment.
- Dynamic Type up to 200%/maximum font scale; equations and tokens wrap; no clipped glyphs.
- Contrast audited in tokens.
- Errors and selection use text/icon/shape as well as color.
- No hover-dependent affordances.

---

## 19. Performance architecture

The app is small; the main risks are drag smoothness and accidental re-render cascades, not data volume.

| Concern | Approach |
| --- | --- |
| Drag frames | Reanimated shared values + `useAnimatedStyle`; no `setState` per move |
| Drop targets | Measure side/token layouts on layout change (`onLayout` / `measureInWindow`), store rects in refs/shared values |
| Validation | Run only on Check, not on every token change; engine calls are cheap but unnecessary |
| Rendering | Render from state directly; `React.memo` only after profiling shows a problem |
| Lists | No virtualized list needed (max 160 tokens); use wrapping views/ScrollView |
| Startup | No network, storage, fonts, or heavy providers |
| Bundle | No barrel imports, no UI kit, no data/state libraries |
| Animations | Reanimated on the UI thread; confetti capped at the source's 42 pieces |
| Cleanup | Cancel timers/animations on unmount/reset; no leaked listeners |
| Measurement | Use React Native DevTools profiler on the drag/check flow; measure before optimizing |

**[Recommendation]** Do not introduce the React Compiler, atomic state, or memoization in v1 unless a measured problem appears. The React Native best-practices skill explicitly warns against speculative memoization.

---

## 20. Testing boundaries

| Layer | Tool | What is real | What is mocked |
| --- | --- | --- | --- |
| Domain | Jest | Everything | Nothing |
| Session reducer/selectors | Jest | Everything | Nothing |
| Components | RNTL | Components + theme | Native modules via façades |
| Screen integration | RNTL | Provider, reducer, domain, components | Audio/haptics/announcements; Reanimated per its Jest setup |
| E2E | Maestro | Whole app on simulator/emulator | Nothing (device capabilities real or ignored) |
| Manual | Device/simulator | Whole app | Nothing |

Rules:

- Mock at the façade (`shared/native/feedback`, `shared/a11y/announce`), not deep inside Expo modules.
- Do not snapshot entire screens; assert accessible names, roles, states, and messages.
- Keep the 70-case engine corpus and the reducer tests as required CI gates.
- E2E covers critical journeys only; do not duplicate every unit case in Maestro.

---

## 21. Dependency register

Versions are not pinned here. **[Recommendation]** Create the project with the current stable Expo SDK and install Expo-managed packages with `npx expo install`, then pin exact versions in `package.json`/lockfile. Verify with `npx expo-doctor` and the Expo upgrade skill.

### 21.1 Runtime dependencies

| Package | Purpose | Why needed | Optional | Native config | New Architecture | Expo compatibility |
| --- | --- | --- | --- | --- | --- | --- |
| `expo` | Expo runtime and SDK | Required by the managed workflow | No | App config, plugins | Supported | Official |
| `react` | UI runtime | Required by React Native | No | None | Supported | Official |
| `react-native` | Native runtime/components | Required | No | None | Supported; target enabled | Official |
| `expo-router` | File-based routing/shell | Single-route shell, typed routes, future deep links | Yes, but recommended | `scheme`, router plugin | Supported | Official |
| `react-native-safe-area-context` | Safe-area insets | Notches, home indicator | No (router peer) | None | Supported | Official |
| `react-native-screens` | Native screen containers | Router dependency | No (router peer) | None | Supported | Official |
| `react-native-gesture-handler` | Drag gestures | Token drag (immediate pan) | Required if drag ships; router dependency | None (root view) | Supported | Official |
| `react-native-reanimated` | UI-thread animation | Drag, confetti, layout transitions | Required for drag/celebration | Babel plugin; worklets | Reanimated 4 requires New Arch | Official |
| `react-native-worklets` | Worklet runtime | Reanimated 4 peer | No if Reanimated 4 used | Babel plugin | Supported | Official |
| `expo-haptics` | Tactile feedback | Optional success/error/tap cues | Yes | None; no permission | Supported | Official |
| `expo-audio` | Audio playback | Success/error chimes from bundled assets | Yes | None; no permission | Supported | Official |
| `@expo/vector-icons` | Icon set | Essential action icons | Yes | Bundled fonts | Supported | Official |
| `expo-splash-screen` | Splash control | Avoid white flash; hide when ready | No (template) | App config | Supported | Official |
| `expo-status-bar` | Status-bar style | Match theme | Yes | None | Supported | Official |
| `expo-system-ui` | Root background color | Avoid flash on Android | Yes | App config | Supported | Official |

### 21.2 Development dependencies

| Package | Purpose | Optional |
| --- | --- | --- |
| `typescript` | Strict typing | No |
| `@types/react` | React types | No |
| `jest` | Test runner | No |
| `jest-expo` | Expo Jest preset | No |
| `@testing-library/react-native` | Component/integration tests | No |
| `@types/jest` | Test types | No |
| `eslint` | Linting | No |
| `eslint-config-expo` | Expo lint rules | No |
| `prettier` | Formatting | Yes |
| `babel-preset-expo` | Babel preset (template) | No |
| `@callstack/eslint-plugin-react-native-a11y` or equivalent | Accessibility lint | Yes |
| Maestro CLI | E2E | External tool, not an npm dependency |
| Xcode / Android SDK + JDK | Local build and submission toolchain (DEP-03) | External tools, not npm dependencies |

### 21.3 Explicitly excluded

| Package/category | Reason |
| --- | --- |
| Redux, Zustand, Jotai, MobX | `useReducer` + Context is sufficient |
| React Query, SWR, axios | No server state |
| React Hook Form, Zod/Yup | One field; validation is the domain engine |
| UI kits (Paper, Elements, NativeBase) | No need; design tokens + primitives |
| NativeWind, styled-components | `StyleSheet` + tokens is simpler and dependency-free |
| AsyncStorage, SecureStore, expo-sqlite | No persistence in v1 (PB-05); revisit only if that decision is reversed |
| expo-notifications | No notifications |
| Analytics/crash SDKs | Excluded by decision (AN-01, AN-02) |
| expo-updates | OTA not required for v1; local builds only (DEP-03) |
| expo-image | No remote images |
| expo-font | System fonts |
| expo-av | Deprecated relative to `expo-audio` |
| Linear gradient | Only if design requires a gradient; not needed for tokens |
| React Native Web / DOM components / WebView | Forbidden by the migration brief |

---

## 22. Proposed folder structure

```
mobile/
  app.json
  package.json
  tsconfig.json                 # strict: true, "@/*": ["./src/*"]
  babel.config.js               # expo preset + reanimated/worklets plugin
  jest.config.js
  jest-setup.ts
  .eslintrc.cjs
  .prettierrc
  assets/
    icon.png
    adaptive-icon.png
    splash.png
    audio/
      success.m4a
      error.m4a
      tap.m4a
  maestro/
    setup-validation.yaml
    solve-tap.yaml
    rejections.yaml
    reset.yaml
  src/
    app/
      _layout.tsx               # providers, error boundary, Stack, status bar
      index.tsx                 # phase switch: Setup | Solver
      +not-found.tsx
    features/
      equation/
        domain/
          rational.ts
          tokenize.ts
          parser.ts
          ast.ts
          affine.ts
          reciprocal.ts
          source-analysis.ts
          candidate-analysis.ts
          format.ts
          result.ts
          types.ts
        messages.ts             # ported MATH_MESSAGES catalog
      session/
        types.ts                # SessionState, Row, WorkspaceToken, Feedback
        workspace.ts            # pure token/row helpers (insert/remove/move/copy/serialize)
        signatures.ts           # duplicate, same-as-previous, source-step guards
        reducer.ts              # pure transitions
        actions.ts              # action creators (plain objects)
        selectors.ts
        engine-bridge.ts        # state + domain → CheckOutcome
        session-context.tsx     # provider + hooks
      setup/
        SetupScreen.tsx
        components/
          BrandHeader.tsx
          EquationField.tsx
          StartButton.tsx
          TipCard.tsx
          ValidationMessage.tsx
      solver/
        SolverScreen.tsx
        components/
          SolverHeader.tsx
          LockedHistory.tsx
          LockedRow.tsx
          ActiveEquationCard.tsx
          SideSelector.tsx
          TokenSide.tsx
          TokenChip.tsx
          TokenPalette.tsx
          WorkspaceToolbar.tsx
          CheckButton.tsx
          FeedbackToast.tsx
          CompletionPanel.tsx
          ConfettiOverlay.tsx
        hooks/
          useTokenDrag.ts
          useCheckActiveRow.ts
    shared/
      ui/
        AppText.tsx
        Button.tsx
        IconButton.tsx
        Card.tsx
        Screen.tsx
        Icon.tsx
        Divider.tsx
      theme/
        colors.ts
        spacing.ts
        radii.ts
        typography.ts
        shadows.ts
        motion.ts
        tokens.ts
        useTheme.ts
      native/
        audio.ts
        haptics.ts
        feedback.ts
      a11y/
        announce.ts
        focus.ts
        useAnnouncer.ts
      config/
        limits.ts
        timing.ts
        palette.ts
        icons.ts
      test/
        renderWithSession.tsx
        mocks.ts
```

Notes:

- `src/app` is the Expo Router root; screens live in features, not routes, so routing stays thin.
- No `index.ts` barrels; imports name the module.
- `shared/test` contains test-only helpers; it is not imported by production code.
- The existing web `index.html`, `docs/`, and `tasks/` remain at the repository root and are not modified by this plan. The RN app lives under `mobile/`.
- **[Open]** Whether to use a monorepo/shared package for the engine later; not needed for v1.

---

## 23. Source-to-module mapping

| Existing source (function / section) | New location | Port type |
| --- | --- | --- |
| `greatestCommonDivisor`, `normalizeRational`, add/subtract/multiply/divide, negate, zero, equality (`index.html:629-684`) | `features/equation/domain/rational.ts` | Direct port to TypeScript, BigInt preserved |
| `tokenizeEquation` (`:702-772`) | `domain/tokenize.ts` | Direct port; preserve limits and Unicode operators |
| `EquationParser` (`:780-876`) | `domain/parser.ts` + `domain/ast.ts` | Direct port; class → functions/closure allowed, behavior identical |
| `parseEquationTokens`, `parseExpressionTokens` (`:878-892`) | `domain/parser.ts` public functions | Direct port |
| `reduceAffine` (`:898-932`) | `domain/affine.ts` | Direct port |
| `analyzeDirectReciprocalSource` (`:934-969`) | `domain/reciprocal.ts` | Direct port; keep the exception narrow |
| `formatAst` (`:971-998`) | `domain/format.ts` | Direct port; canonical display |
| `reduceAndSolveEquation`, `analyzeSourceEquation` (`:1000-1029`) | `domain/source-analysis.ts` | Direct port returning typed results |
| `analyzeCandidateSide`, `analyzeCandidateEquation`, `isFinalFormCandidate` (`:1032-1098`) | `domain/candidate-analysis.ts` | Direct port returning typed results |
| `runMathSelfChecks` (`:1104-1292`) | `features/equation/domain/__tests__/golden.test.ts` | Convert to Jest cases; do not ship the console check |
| `MATH_MESSAGES` (`:603-619`) | `features/equation/messages.ts` | Copy verbatim; map reason → message |
| `PALETTE_TOKENS` (`:597-602`) | `shared/config/palette.ts` | Recreate as typed metadata with accessibility labels |
| `nextTokenId`, `tokenLabel`, `createWorkspaceToken`, `copyTokensForNewRow`, `createActiveRow`, `tokensFromDisplay`, `serializeTokens`, `removeTokenFromRow`, `insertTokenInRow`, `moveTokenInRow` (`:1294-1360`) | `features/session/workspace.ts` | Port as pure helpers over the new row types |
| `state` object (`:1362-1378`) | `features/session/types.ts` + `reducer.ts` | Redesign into a typed state machine |
| `renderPalette` (`:1450-1472`) | `features/solver/components/TokenPalette.tsx` | Rewrite as components |
| `renderTokenSide` (`:1474-1543`) | `TokenSide.tsx` + `TokenChip.tsx` | Rewrite |
| `renderToolbar` (`:1545-1557`) | `WorkspaceToolbar.tsx` | Rewrite |
| `appendVariableHighlightedText` (`:1559-1573`) | `AppText`/equation display helper | Rewrite using nested `Text` |
| `renderLockedRows` (`:1575-1598`) | `LockedHistory.tsx` + `LockedRow.tsx` | Rewrite |
| `render` (`:1600-1729`) | Screen composition; celebration moved to `ConfettiOverlay` | Rewrite |
| `updateStickyToolboxHeight` (`:1731-1808`) | Layout measurement in the solver screen | Rewrite; no sticky height needed if layout is fixed |
| Set side/select/append/delete/reorder/move (`:1810-1882`) | Reducer actions + screen handlers | Port semantics |
| `cleanupDrag`, `dragTargetAt`, `updateDrag`, `finishDrag`, `beginDrag` (`:1884-2005`) | `useTokenDrag.ts` + `TokenChip` + measured drop targets | Rewrite with Gesture Handler/Reanimated |
| Signature/duplicate guard (`:2007-2030`) | `features/session/signatures.ts` | Port rules, typed |
| Clear/flip/restore/reject (`:2032-2085`) | Reducer actions | Port semantics |
| `lockActiveRow`, `checkActiveEquation` (`:2087-2128`) | `engine-bridge.ts` + `APPLY_CHECK` | Port orchestration; domain call stays pure |
| Completion/celebration/sound (`:1558-1729`, `:2112-2123`) | `CompletionPanel`, `ConfettiOverlay`, `shared/native/feedback` | Rewrite; bundled audio |
| `startSession`, `resetSession` (`:2130-2230`) | `START_SESSION`, `RESET_SESSION` | Port semantics |
| Event listeners, ResizeObserver, init (`:2232-2292`) | Provider/root layout; dimension hooks | Obsolete DOM wiring; replace with React lifecycle |
| CSS tokens and layout (`:9-462`) | `shared/theme/*` + component styles | Design intent ported; implementation rewritten |
| Accessibility markup and live regions (`:480-576`, `:1731-1803`) | Component accessibility props + `shared/a11y` | Rewritten with native APIs |

---

## 24. Architecture decisions (ADR summaries)

### ADR-001 — Expo managed / CNG

- **Context:** No custom native code, modules, permissions, or background work. Needs gestures, animation, audio, haptics, safe area, navigation, and store delivery.
- **Decision:** React Native + TypeScript + Expo modules, Expo managed/CNG, and development builds. Build and submit locally with Xcode and Gradle; do not use EAS Build or EAS Submit (DEP-03). Confirmed by the product owner on 2026-11-10.
- **Consequences:** Fast setup, official modules, native projects generated on demand. Development builds, not Expo Go, are the test runtime. Local machines must carry the Xcode/Gradle toolchain and signing credentials; no cloud build farm or EAS-hosted OTA is assumed. Boris owns the Apple Developer, Google Play, and Expo accounts (DEP-01). If local capacity becomes a release bottleneck, EAS can be revisited as an additive change.
- **Alternatives:** Bare RN CLI — more control, more maintenance, no benefit today. Expo prebuild with committed native projects — unnecessary unless custom native code appears.

### ADR-002 — Single route, phase-driven screens

- **Context:** The source has no navigation; one session at a time.
- **Decision:** One Expo Router route; switch Setup/Solver by session phase. Intercept Android hardware back during an active session so the session is preserved (UX-08); back on setup exits normally.
- **Consequences:** No back-stack ambiguity and no accidental session loss. A future resume/deep-link feature can add routes without changing the domain.

### ADR-003 — Pure TypeScript domain engine

- **Context:** The math engine is the product's core and the highest regression risk.
- **Decision:** No React/RN imports in `features/equation/domain`; typed discriminated results; BigInt rationals; no `eval`.
- **Consequences:** Fast tests, portability, clear error mapping. Requires a strict port and golden tests.

### ADR-004 — `useReducer` + Context, no state library

- **Context:** One cohesive session machine; no server cache; no cross-screen shared atoms.
- **Decision:** Reducer with typed actions, provider at the root, pure selectors.
- **Consequences:** Small dependency surface and testable transitions. If re-render profiling later shows a problem, split contexts/selectors before adding a library.

### ADR-005 — Native gestures and UI-thread animation

- **Context:** Drag/reorder/delete is the signature interaction; CSS Pointer Events and `elementFromPoint` do not exist.
- **Decision:** Gesture Handler immediate pan on touch-move (no long-press or handle, UX-06); Reanimated shared values; measured drop rects; toolbar/`accessibilityActions` alternative. Drag is required in the first release (PB-04), not a later enhancement.
- **Consequences:** Smooth drag without React renders and no discoverability cost from a hidden long-press. Requires a PoC for scroll conflict (decide whether the solver scrolls at all, or use a pan threshold/direction lock), drop targeting, and low-end Android. Drag cannot be the only path; tap/toolbar must be equally complete.

### ADR-006 — In-memory session in v1

- **Context:** No storage in the source; no accounts; child privacy favors minimal data; the product owner confirmed persistence is not required (PB-05).
- **Decision:** No persistence dependency in v1.
- **Consequences:** Process death loses the session; accepted. No retention/deletion policy is needed while nothing is stored. If resume is ever required, add a narrowly scoped rehydration path; never persist BigInt directly.

### ADR-007 — Bundled audio, not runtime synthesis

- **Context:** The web synthesizes tones with Web Audio; native has no equivalent built-in synthesizer exposed to RN; sound is required (PB-06).
- **Decision:** Bundle short chimes and play them with `expo-audio`; configure ambient playback so the iOS silent switch mutes and Android follows media volume (NAT-10). No in-app mute control in v1.
- **Consequences:** Requires asset creation/licensing and a mute control decision (NAT-04). Haptics provide an additional non-audio cue, not a substitute for required sound.

### ADR-008 — Maestro for E2E

- **Context:** Small app, two platforms, simple critical journeys, no complex native synchronization.
- **Decision:** Maestro YAML flows; manual device/a11y testing for capabilities it cannot assert. Confirmed on 2026-11-10.
- **Consequences:** Low setup and maintenance. Detox is unwarranted unless the app later gains complex native state.

### ADR-009 — Static light theme from tokens

- **Context:** The web is a single warm light theme; no dark mode; the product confirmed a single light theme only (UX-03).
- **Decision:** Typed tokens + `StyleSheet`; `userInterfaceStyle: light`.
- **Consequences:** Fast, consistent UI with no theme-switching code. Dark mode is out of scope; tokens keep it addable if that decision ever changes.

### ADR-010 — No WebView or DOM components

- **Context:** Explicit migration constraint; the goal is a real native app.
- **Decision:** Rebuild all UI with native components; never wrap `index.html`; never use Expo DOM components for app UI.
- **Consequences:** Full rewrite of markup/CSS/DOM logic, but native accessibility, gestures, performance, and store compliance.

---

## 25. Open architecture questions

These are architecture-relevant; the full product list is in the open-questions document.

**Resolved by the 2026-11-10 answers:** drag is required (PB-04); no session resume (PB-05); sound is required (PB-06); no accounts/monetization (PB-07); English only (PB-08); web maintained in parallel (PB-10); standalone offline app with no backend API and no authentication (API-01, OFF-01, SEC-01, SEC-02); **tablet is the primary target with phone support** (UX-02); **single light theme** (UX-03); **no analytics or crash reporting** (AN-01, AN-02); **native redesign rather than pixel parity** (UX-01); **palette inserts append to the selected side** (UX-05); **immediate drag on touch-move** (UX-06); **completion shows “Good job”, plays the success sound, and offers “Solve another!”** (UX-07); **back preserves the in-memory session** (UX-08); **audience ages 10–15 plus new equation learners; Education listing, not Kids-only** (SEC-03); **both screen-reader actions and external hardware keyboard** (NAT-03); **silent switch mutes on iOS, media volume on Android** (NAT-10); **Boris owns the Apple/Google/Expo accounts** (DEP-01); **identity `net.bbsteps.equation`, scheme `equation`** (DEP-02); **local Xcode/Gradle builds, no EAS Build/Submit** (DEP-03); **iOS/iPadOS 16+, Android 9 (API 28)+** (TST-03, DEP-09); **keep all 70 engine cases** (TST-07).

1. **Audio assets and optional controls:** Who supplies/owns the chimes; is there a mute toggle and a haptics toggle (NAT-04)? Silent-switch behavior is decided (NAT-10).
2. **Device orientation:** Tablet is primary and phones must work (UX-02), but portrait vs landscape is unstated. This affects layout breakpoints and testing.
3. **Engine sharing:** The native app replaces the web app (PB-02) while the web runs in parallel (PB-10). Should the ported engine be shared with the web now, or stay a separate port until the web is retired? This affects repository structure and the "no build system" web constraint.
4. **Web retirement:** PB-02 and PB-10 together imply native is the successor, not an immediate cutover. What triggers retirement, and who maintains parity until then?
5. **Release cadence:** Local build path and account ownership are decided (DEP-01, DEP-03); who cuts and ships a release, and on what schedule, is not.
6. **Brand assets:** Icon/splash design and brand approval remain open; identity values are settled (DEP-02).