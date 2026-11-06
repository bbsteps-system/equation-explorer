# M0 Task List: Foundations and UI Shell

**Status:** Complete — human browser acceptance was confirmed by the reviewer; detailed environment evidence was not captured in this session.

**References:** `tasks/00-plan.md`, `docs/specs/milestones/00-foundations-ui-shell.md`, `CLAUDE.md`

- [x] **Task 1: Create the standalone semantic shell and visual foundation**
  - **Implementation status:** Complete; static semantics/safety inspection passed. Consolidated browser review remains Task 5.
  - **Depends on:** Approved M0 plan and task list.
  - **Acceptance:** `index.html` is a standalone HTML5 document with viewport/title metadata, `header`, `main`, logical section headings, and no remote resources. It provides mobile-first base styles and CSS custom properties for the specified surfaces, action/feedback colors, focus ring, border, radii, and shadow. System fonts, high-contrast text, 44px enabled controls, visible `:focus-visible` styling, and a `prefers-reduced-motion` override are present.
  - **Verify:** Open the file directly. Inspect the Network panel for zero external requests. Enable reduced motion and confirm no essential state depends on transitions. Inspect an enabled button/input at 200% browser zoom.
  - **Files:** `index.html`

- [x] **Task 2: Implement the responsive landing setup experience**
  - **Implementation status:** Complete; consolidated viewport and keyboard review remains Task 5.
  - **Depends on:** Task 1.
  - **Acceptance:** The landing UI has a prominent title, labelled required equation input, `e.g. 2x - 3 = 8` placeholder, helper text, Start button, equivalence tip, and an associated polite feedback area. Started-only content is not actionable or exposed in landing. The layout follows compact (320–599px), medium (600–899px), and wide (900px+) requirements without changing DOM reading order or causing horizontal scroll.
  - **Verify:** At 320px, ~768px, and ≥1280px, inspect setup layout, tab order, focus visibility, and overflow. Confirm whitespace-only input leaves Start disabled.
  - **Files:** `index.html`

- [x] **Task 3: Implement explicit M0 state and setup/reset transitions**
  - **Implementation status:** Complete; static lifecycle and safety inspection passed; browser confirmation remains Task 5.
  - **Depends on:** Task 2.
  - **Acceptance:** Strict-mode inline JavaScript maintains `mode`, `draftEquation`, `submittedEquation`, and `feedback` separately from markup. Input updates Start availability; clicking Start or pressing Enter with trimmed non-blank text starts the mock session; blank submission produces accessible required feedback without starting. Reset clears session data, replaces any prior feedback with a neutral ready status, returns to landing, and restores focus to the input. Submitted input is rendered only with `textContent`.
  - **Verify:** Test blank/whitespace, `2x - 3 = 8`, arbitrary non-math text, Enter submission, reset, focus placement, and live announcements. Inspect source to confirm no `innerHTML`, `eval`, or parser behavior.
  - **Files:** `index.html`

- [x] **Task 4: Render and protect the started mock workspace**
  - **Implementation status:** Complete; static disabled-control and mock-state inspection passed; browser confirmation remains Task 5.
  - **Depends on:** Task 3.
  - **Acceptance:** A started session shows literal submitted text in visually locked Step 1; toolbox tokens, labelled trash, Step 2 LHS/RHS build areas, and Check / Next Step affordance appear with orientation copy. Deferred controls are native-disabled or plain non-interactive descriptions, have explanatory unavailable text, are skipped by keyboard focus, and make no assertion that Step 1 is mathematically valid. Focus moves after starting to a meaningful solving-path orientation target.
  - **Verify:** Start with `2x - 3 = 8` and arbitrary non-math input. Confirm the same neutral mock status, literal rendering, disabled behavior, focus position, and absence of functional token/check interaction.
  - **Files:** `index.html`

- [x] **Task 5: Execute the M0 human acceptance gate and record results**
  - **Implementation status:** Complete; the reviewer confirmed the browser acceptance gate in this session. Browser/version and viewport details were not retained in the handoff.
  - **Depends on:** Tasks 1–4.
  - **Acceptance:** Every acceptance criterion and all nine steps in M0’s manual demo script pass in a current Chromium or Safari browser, both from `file://` and through `python3 -m http.server 8000 --directory .`. No console exceptions or failed resource requests occur. The reviewer can verify 320px, ~768px, ≥1280px, keyboard behavior, 200% zoom, and reduced motion.
  - **Verify:** Follow `docs/specs/milestones/00-foundations-ui-shell.md` lines 195–203 (or its current manual demo section) exactly. Record browser engine/version and widths in the implementation handoff or PR.
  - **Files:** No production-file change expected; update only approved implementation evidence if the project later adopts a location for it.

## Delivery order
Tasks 1–4 are sequential changes to one file. Task 5 is a hard gate: do not mark M0 complete or begin M1 until it passes and the human reviewer approves the result.
