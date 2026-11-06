# Milestone 0 Spec: Foundations and UI Shell

**Status:** Revised draft — requires human approval before implementation.

## Objective
Deliver a polished, responsive, accessible visual shell for **Equation Explorer**, a child-friendly linear-equation learning game. M0 proves the information architecture, responsive layout, setup/reset lifecycle, semantic foundations, and visual language before mathematical parsing or token manipulation is introduced.

A reviewer should be able to open one local HTML file, understand the intended learning journey immediately, enter a sample equation, inspect a believable started-session shell, and return safely to the landing state. M0 must not imply that entered equations have been mathematically checked.

## Assumptions and decisions
- The application targets current evergreen Chromium, Safari, and Firefox browsers, including mobile Safari and Chrome on touch devices. Internet Explorer is out of scope.
- The product remains one dependency-free `index.html` with embedded CSS and JavaScript; it must work when opened as a local file and when served as static content.
- M0 intentionally accepts any non-blank text for its mock session. M1 replaces this temporary rule with supported-equation parsing and exact validation.
- The responsive design is mobile-first. The layout must be usable from 320px wide upward, not merely visually compressed.
- Deferred controls are visible to explain the eventual workflow but are programmatically disabled and labelled as unavailable until their owning milestone. They must not look actionable or claim a validation result.
- The visual design will use system fonts and CSS/Unicode icons only; no externally loaded font, image, CDN, audio, or analytics resource is allowed.

## Scope
- Create the standalone `index.html` document shell with inline CSS and vanilla JavaScript.
- Build the application header, equation-setup form, learning tip, toolbox card, trash-zone placeholder, solving-path/history card, feedback/live-region area, and reset control.
- Implement an explicit UI-only session model with two states: `landing` and `started`.
- In the started state, render the submitted text as an immutable mock Step 1 and render a clearly inactive mock Step 2 with left/right build areas, equality separator, disabled Check / Next Step control, disabled palette, and disabled trash zone.
- Implement Start submission through click and Enter; prevent a blank or whitespace-only start; provide accessible feedback; manage focus after start and reset.
- Implement responsive styles for compact (320–599px), medium (600–899px), and wide (900px+) layouts.
- Establish reusable visual tokens and base behavior for focus, disabled, error, success-ready, and reduced-motion states that later milestones can extend without redesigning the page.

## Non-goals
- Parsing, normalizing, or mathematically checking a non-empty equation.
- Functional token insertion, drag/reorder/delete, or editable expression construction.
- Enabling Check / Next Step, growing history, success/error validation animations, sound, or confetti.
- Persisting a session, network communication, external dependencies/assets, a build system, package configuration, or automated-test framework.

## Tech stack
- **Markup:** semantic HTML5 in `index.html`.
- **Styling:** CSS custom properties, Grid/Flexbox, media queries, `:focus-visible`, and `prefers-reduced-motion`.
- **Behavior:** strict-mode, vanilla ES2020+ JavaScript; DOM APIs only.
- **Future compatibility:** section IDs and named render functions may be retained by M1–M4, but M0 must not introduce speculative parser or drag-and-drop code.

## Commands
```sh
# Serve the standalone app locally (recommended during review)
python3 -m http.server 8000 --directory .

# Open it on macOS
open http://localhost:8000/

# Inspect the local files used by the milestone
find . -maxdepth 3 -type f | sort
```

There is no install, build, lint, or automated-test command in M0. The required verification is the manual browser demo in this document.

## Project structure
```text
index.html                                      # M0 shell; eventual complete application
CLAUDE.md                                       # Project-wide technical conventions
docs/
  specs/
    linear-equation-drag-solver.md              # Approved umbrella product specification
    milestones/
      00-foundations-ui-shell.md                # This milestone specification
      01-exact-linear-math-engine.md            # M1, not implemented in M0
      02-touch-token-workspace.md               # M2, not implemented in M0
      03-guided-validation-history.md           # M3, not implemented in M0
      04-celebration-accessibility-hardening.md # M4, not implemented in M0
```

## Draft UI and state flow

### Landing state
```text
┌────────────────────────────────────────────────────────────────────┐
│ 🧮 Equation Explorer                         Solve it step by step  │
│                                                                    │
│ Start an equation                                                  │
│ Enter an equation                                                  │
│ [ e.g. 2x - 3 = 8                                      ] [Start]  │
│ Tip: Whatever you do to one side, do it to the other side!        │
│ [Helpful feedback appears here when needed.]                       │
└────────────────────────────────────────────────────────────────────┘
```

The setup card is the first and primary task. Before a session starts, the toolbox and solving path are either not rendered or are visually and semantically hidden. The Start button is disabled whenever `draftEquation.trim()` is empty. Because a disabled button cannot be activated, blank submission feedback is supplied when the input loses focus while blank after an attempted edit, or via the input's native required validation on form submission; implementation must not promise a click on a disabled button.

### Started mock state
```text
┌────────────────────────────────────────────────────────────────────┐
│ [New equation / Reset]                                             │
│                                                                    │
│ TOKEN TOOLBOX                                      🗑 Trash         │
│ [0][1][2][3][4][5][6][7][8][9][.]  [+][-][×][÷][(][)][x]          │
│ Available when building begins                                     │
│                                                                    │
│ YOUR SOLVING PATH                                                  │
│ Step 1  🔒  2x − 3                     =  8                       │
│ Step 2      [Build left side]            = [Build right side]     │
│             [Check / Next Step — coming soon]                      │
│                                                                    │
│ You'll build the next step here.                                   │
└────────────────────────────────────────────────────────────────────┘
```

Step 1 displays `submittedEquation` as text, without parsing or normalization. It must use `textContent`, never `innerHTML`. The mock workspace labels communicate the future interaction but all not-yet-implemented controls use their native `disabled` state (or are plain non-interactive text), include an explanation such as “Available in the next learning step,” and are omitted from the tab order.

### Responsive behavior
- **Compact: 320–599px.** One column. Input and Start stack vertically; palette wraps into rows; Step 2 displays LHS, `=`, RHS, then the disabled Check control in reading order. A 16px minimum inline page gutter and wrapping token labels prevent horizontal scrolling.
- **Medium: 600–899px.** Setup may place input and Start on one line when enough space exists. Toolbox and solving path remain vertically ordered. Equation rows may keep an inline `=` when both sides have enough room.
- **Wide: 900px+.** Center a readable content column (maximum 1,120px); setup input and Start are inline; toolbox and solving path are distinct cards. The Step 2 LHS, equality sign, RHS, and disabled check control appear in one visual row while preserving logical DOM/tab order.

### State transition contract
```text
landing
  ├─ update input ───────────────► landing (draftEquation changes)
  ├─ valid non-blank submit ─────► started (submittedEquation is set)
  └─ blank attempted submit ─────► landing (input feedback is shown)

started
  └─ Reset / New equation ───────► landing (all session-only state cleared)
```

M0's script must retain state separate from markup, at minimum:

```js
const state = {
  mode: 'landing', // 'landing' | 'started'
  draftEquation: '',
  submittedEquation: '',
  feedback: { kind: 'info', message: '' },
};
```

M1 may add parsed source data only after it accepts the source expression. M2–M4 may extend the object; they must not infer source-equation truth from rendered DOM text.

## Accessibility and interaction requirements
- Use a `header`, a `main` landmark, and separately labelled setup, toolbox, and solving-path sections. Use an `h1` followed by a logical heading hierarchy.
- The setup input has a visible `<label>`, `placeholder="e.g. 2x - 3 = 8"`, `required`, and associated helper/error text through `aria-describedby`.
- Start and Reset/New equation are native button elements. The setup uses a `<form>` so Enter behaves consistently.
- Feedback is rendered into a `role="status"` or `aria-live="polite"` region. Input errors use an appropriate invalid state (`aria-invalid="true"`) only while the error is active.
- On successful mock start, focus moves to the solving-path heading or its short orientation text, not to a disabled future control. On reset, focus returns to the input.
- Every enabled interactive control has a visible `:focus-visible` treatment with sufficient contrast. Minimum target size is 44×44 CSS pixels for enabled touch controls.
- Textual labels and icon names convey status; color, lock icons, opacity, or motion are supplementary only.
- Respect `prefers-reduced-motion: reduce` by removing nonessential transitions and animations. M0 must not require animation to communicate any state.

## Visual-system requirements
- Define a small set of semantic CSS custom properties for page/background, surface, primary action, accent, success-ready, error, text, muted text, border, focus ring, radii, and shadow.
- Use a warm light background, dark readable text, bright but sufficiently contrasting indigo/teal/coral accents, rounded cards, and friendly system typography. Do not make color contrast depend on a decorative palette alone.
- Provide distinct visuals for normal, hover (where supported), focus-visible, disabled, and feedback states. Disabled M0 placeholders must remain readable and explain why they are unavailable.
- Avoid fixed heights for content cards and equation areas. Wrapping text and enlarged browser text must remain usable.

## Code style
Keep all M0 code in a strict-mode inline script, organized into immutable constants, state, DOM references, render functions, and event handlers. Rendering reads state and replaces only controlled content; state-changing events update state, then render. Use two-space indentation, `const` by default, camelCase for variables/functions, and UPPER_SNAKE_CASE for fixed constants.

```js
function startMockSession() {
  const equation = state.draftEquation.trim();

  if (!equation) {
    state.feedback = {
      kind: 'error',
      message: 'Type an equation before we begin.',
    };
    render();
    equationInput.focus();
    return;
  }

  state.mode = 'started';
  state.submittedEquation = equation;
  state.feedback = {
    kind: 'info',
    message: 'Your first equation is ready. You will build the next step here.',
  };
  render();
  solvingPathHeading.focus();
}
```

Do not use `innerHTML` with user-provided text, derive state from CSS classes, attach click handlers to non-semantic elements, or add parser/drag behavior ahead of its milestone.

## Acceptance criteria
- [ ] Opening `index.html` directly presents the landing state with title, labelled required input, example placeholder, Start control, equivalence tip, and clear visual hierarchy.
- [ ] The Start button is disabled when the trimmed input is empty and becomes enabled when non-blank text is entered; the form prevents a blank session and supplies an accessible required-input message if blank submission is attempted.
- [ ] Clicking Start or pressing Enter with non-blank text enters the started mock state. Step 1 visibly locks and renders the submitted text literally; the shell makes no claim that it is valid mathematics.
- [ ] Started state renders toolbox, trash, left/right build areas, and Check / Next Step as clearly disabled future controls with explanatory text; none can be focused or activated.
- [ ] Reset/New equation clears `draftEquation` and `submittedEquation`, restores landing state, replaces any prior feedback with the neutral status “Ready for a new equation.”, and returns focus to the equation input.
- [ ] Compact layout at 320px has no horizontal page scrolling, clipped controls, or overlap; the mock equation row follows the prescribed vertical reading order.
- [ ] Medium and wide layouts have distinct setup, toolbox, and workspace cards; at 900px+ the mock equation row is readable inline without changing DOM reading order.
- [ ] Keyboard users can reach every enabled action in a logical order, always see focus, and receive useful labels and live feedback. Disabled future controls are not focusable.
- [ ] A reduced-motion preference leaves both states and all feedback understandable with no essential animation.
- [ ] The app makes no remote request and has no console errors when opened as `file://` or served locally.

## Testing strategy and human demo script
M0 has no approved automated test runner. The implementation owner runs and records this manual pass in current Chromium or Safari; Firefox is recommended as a second engine.

1. Open `index.html` directly, then run `python3 -m http.server 8000 --directory .` and open `http://localhost:8000/`. In both cases, verify no console errors or network/resource failures.
2. On landing, confirm the title, prominent setup task, input label/placeholder, disabled blank Start control, learning tip, and absence of an actionable workspace.
3. Type spaces only. Confirm Start remains disabled. Clear/refocus the required input or attempt native form submission and verify a readable required-input message without a session starting.
4. Enter `2x - 3 = 8`, press Enter, and verify the started mock state: literal locked Step 1, visible but disabled future workspace controls, visible orientation text, and a polite status announcement. Confirm no message says the input was mathematically accepted.
5. Confirm keyboard focus lands on the solving-path orientation after starting. Tab through enabled controls and confirm disabled palette, trash, and Check are skipped.
6. Select Reset/New equation. Confirm landing returns; input and session state clear; the prior feedback is replaced by a neutral ready status; focus is in the equation input.
7. Repeat steps 3–6 at 320px, approximately 768px, and at least 1280px viewport width. Check for no horizontal scrolling, clipping, or overlap; inspect the prescribed stacked and inline layouts.
8. Enable a reduced-motion preference, repeat start/reset, zoom browser text to 200%, and verify readability, non-motion state cues, and usable controls.
9. If available, use a screen reader to confirm the form label, helper/error association, setup status, landmark/section names, and Reset label are meaningful.

## Boundaries
- **Always:** follow `CLAUDE.md`; preserve the one-file/no-dependency architecture; retain an explicit UI state object; use semantic controls; render user input as text; validate blank input; maintain responsive, keyboard, and reduced-motion behavior; run the full manual demo before declaring M0 done.
- **Ask first:** add a dependency/CDN/font/image; split the single file; introduce a build or test framework; change the breakpoints or child-facing visual direction; begin parsing math or enabling drag/check interactions; commit or publish changes.
- **Never:** use `innerHTML` for user input; use `eval()`/`Function()`; send entered equations off-device; misrepresent mock input as mathematically verified; make a non-semantic element the only way to perform an action; leave enabled interactive controls below 44×44px; remove accessibility behavior to simplify the shell.

## Exit gate and dependencies
**Entry:** The umbrella product spec and this revised M0 specification are approved.

**Done:** Every acceptance criterion and the human demo script pass; a reviewer approves the visual direction, responsive behavior, and UI state contract. Implementation produces only the M0 shell—no hidden M1–M4 functionality. M1 may start after this gate, at which time it replaces the mock non-blank start rule with real parsing and source-equation validation.
