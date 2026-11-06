# M0 Implementation Plan: Foundations and UI Shell

**Status:** Approved and implemented — human browser acceptance gate pending.

**Governing specs:**
- `docs/specs/linear-equation-drag-solver.md`
- `docs/specs/milestones/00-foundations-ui-shell.md`
- `CLAUDE.md`

## Outcome
Create a single, directly runnable `index.html` that delivers the Equation Explorer visual shell. It will support only the M0 lifecycle:

```text
landing → non-blank Start / Enter → started mock session → Reset → landing
```

The started state deliberately renders the submitted text without parsing or claiming mathematical validity. Toolbox, trash, build areas, and Check / Next Step communicate forthcoming functionality but remain native-disabled or non-interactive.

## Architecture

### Deliverable
`index.html` contains semantic markup, embedded CSS, and a strict-mode inline JavaScript module. No package, build system, CDN, external asset, network call, parser, or drag code is introduced.

### State and rendering boundary
Maintain application data independently of the DOM:

```js
const state = {
  mode: 'landing',
  draftEquation: '',
  submittedEquation: '',
  feedback: { kind: 'info', message: '' },
};
```

- Input events update `draftEquation` and recompute Start availability.
- Form submission trims the draft. A non-blank value transitions to `started`; a blank value leaves `landing` active and exposes input feedback.
- Reset clears the session fields, replaces any prior feedback with a neutral ready status, and restores input focus.
- Rendering updates controlled regions and uses `textContent` for `submittedEquation`.
- Focus is moved after rendering: to orientation text after start and to the input after reset.

M1 is the first milestone permitted to add parsed source data. M2–M4 may extend state but must not derive source truth from visual markup.

### Page regions and semantics
| Region | M0 responsibility | Future seam |
| --- | --- | --- |
| Header | Product title and short learning context | Stable brand/heading region |
| Setup section | Labelled required form, tip, input feedback, Start | M1 replaces mock acceptance with exact parsing |
| Feedback region | Polite, textual status/error announcements | M3 adds validation feedback states |
| Toolbox section | Visible only in `started`; disabled palette and labelled trash placeholder | M2 enables insertion, dragging, and deletion |
| Solving-path section | Locked literal Step 1 and inactive Step 2 | M3 creates real history and validation |
| Reset action | Clears M0 state and returns to setup | Stable session reset action |

## Implementation sequence and dependencies

```text
Task 1: Document shell and visual tokens
  └─ Task 2: Semantic landing-state markup and responsive CSS
       └─ Task 3: Explicit state, rendering, setup/reset lifecycle
            └─ Task 4: Started mock workspace and accessibility behavior
                 └─ Task 5: Browser acceptance pass and M0 evidence
```

Tasks 1–4 all modify `index.html` and are intentionally sequential to avoid conflicting edits. Task 5 begins only when the lifecycle is complete.

## Detailed implementation approach

### 1. Establish the document shell and design system
Create the HTML document with viewport metadata, a meaningful title, a `header`, a `main`, and semantic sections. Define mobile-first CSS custom properties for the approved warm surface palette, readable text, action colors, focus ring, border, radii, and shadow. Add base typography, sizing, buttons, cards, and reduced-motion overrides.

**Checkpoint:** The page is legible with zero JavaScript behavior required. Its design tokens can represent normal, hover, focus, disabled, error, and info/success-ready states.

### 2. Build the landing UI and responsive layout
Implement the setup form using a visible `<label>`, required equation input, helper/error association, and native Start button. Hide the started-only interface in the landing state. Use compact (320–599px), medium (600–899px), and wide (900px+) layouts, preserving DOM reading order as visual layout changes.

**Checkpoint:** At 320px, 768px, and 1280px, no content overflows horizontally; input and Start stack/align as specified; focus styling is visible.

### 3. Add the stateful setup/reset lifecycle
Add strict-mode JavaScript with DOM references, constants, state, pure state transition helpers where helpful, and rendering/event functions. Keep Start disabled for whitespace-only input. Use form submit to support Enter. Create accessible feedback for blank handling, start/reset status changes, and input validity. Ensure state is reset rather than only visually hidden.

**Checkpoint:** A non-blank input starts a mock session and a reset clears the session state, markup, and input value while replacing prior feedback with a neutral ready status. All user-provided equation display uses `textContent`.

### 4. Render the started-session mock workspace
Add the started-only toolbox, trash zone, Step 1, Step 2 build placeholders, and Check / Next Step control. Make all deferred M2/M3 controls native-disabled where they are controls; otherwise render descriptive non-interactive text. Add explanatory copy and ensure deferred controls cannot receive focus. Move focus to the solving-path orientation after starting.

**Checkpoint:** The application visibly communicates the eventual flow but cannot be mistaken for a validated or editable solver.

### 5. Validate and record M0 completion evidence
Run the exact manual demo script in the M0 spec, both as `file://` and via the prescribed static server. Inspect console and network panels; resize, zoom to 200%, test reduced motion, and tab through every enabled control. Record browser(s), viewport widths, and any limitation in the task completion record or handoff.

**Checkpoint:** Every M0 acceptance criterion and manual demo step passes. No M1–M4 functionality exists.

## Risks and mitigations

| Risk | Effect | Mitigation |
| --- | --- | --- |
| Disabled Start cannot be clicked | A required-message criterion is accidentally impossible to test | Keep disabled state for blank input; use native `required` validation and/or blur-based input feedback, never promise a disabled-button click. |
| Mock state appears mathematically accepted | Learners may believe arbitrary input was validated | Use neutral text such as “Your first equation is ready,” retain literal input, and avoid success/check language for source validity. |
| Desktop styling breaks reading order | Keyboard/screen-reader sequence differs from visual sequence | Keep setup and equation markup in logical reading order; use Grid/Flex only for visual placement. |
| Decorative disabled UI is inaccessible | Users cannot tell why controls are unavailable | Use visible explanatory text, native `disabled` controls, readable contrast, and no disabled controls in tab order. |
| Responsive layout overflows at narrow widths or zoom | M0 fails child/mobile usability requirements | Use intrinsic widths, `minmax(0, 1fr)`, wrapping, minimum 16px gutters, and avoid fixed card heights. |
| Re-rendering loses focus or announcements | Keyboard experience becomes disorienting | Render first, then intentionally focus the orientation/input target; keep feedback as a stable polite live region. |

## Verification plan

### Static/review checks
- Confirm `index.html` contains no external `src`/`href` resource loading, no `eval`, no `Function`, and no `innerHTML` interpolation of submitted input.
- Confirm current markup uses landmarks, headings, real form/button controls, labels, and a polite status region.
- Confirm only M0 behaviors are present.

### Manual browser gate
Run `python3 -m http.server 8000 --directory .`, then perform the M0 spec’s nine-step demo at 320px, approximately 768px, and at least 1280px; also test the directly opened local file. Test Chromium or Safari as required, with Firefox recommended.

## Definition of ready for implementation
- [x] Umbrella product spec approved.
- [x] M0 milestone spec approved by the human reviewer.
- [x] Scope is constrained to UI-only mock sessions.
- [ ] This plan and `tasks/00-todo.md` approved by the human reviewer.

## Definition of done
M0 is complete only when the human demo script and every M0 acceptance criterion pass, the UI state contract is preserved, and the reviewer approves the visual direction. Approval of this plan authorizes implementation; it does not authorize M1–M4 work.
