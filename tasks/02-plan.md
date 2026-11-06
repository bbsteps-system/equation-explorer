# Milestone 2 Plan — Touch-Friendly Token Workspace

**Spec:** [`docs/specs/milestones/02-touch-token-workspace.md`](../docs/specs/milestones/02-touch-token-workspace.md)  
**Status:** Complete — implementation and human browser acceptance passed; detailed browser/version and viewport evidence was not captured in this session.

## Outcome

Replace M1’s disabled preview workspace with a dependency-free token editor in `index.html`. A learner can create, select, insert, move, reorder, and delete tokens in the one active equation row using mouse, touch, or keyboard-only controls. Locked history remains inert. M2 produces deterministic token arrays for M3 but performs no syntax, equivalence, or win validation.

## Preconditions

1. M0 Task 5 and M1 Task 6 have passed and their human review evidence is recorded.
2. The M2 milestone specification and this plan are approved.
3. The following specification gaps identified during architectural review are resolved in the M2 spec before implementation: state/data contract, canonical source-token conversion, drag activation/scroll policy, wrapped-row insertion-index algorithm, exact keyboard controls/focus rules, and cleanup behavior.

## Approved implementation contract

These decisions resolve the interaction details and are incorporated in the approved M2 specification.

### State and token contract

```js
const state = {
  // Existing M0/M1 fields remain unchanged.
  activeRow: {
    id: string,
    leftTokens: Token[],
    rightTokens: Token[],
  } | null,
  selectedSide: 'left' | 'right',
  selectedToken: { side: 'left' | 'right', tokenId: string } | null,
  drag: DragState | null,
};

const token = {
  id: string,       // unique for the active session; never derived from an index
  symbol: string,   // canonical parser-ready symbol, such as '×' or '÷'
  label: string,    // accessible spoken name, such as 'multiply'
};
```

`serializeTokens(tokens)` returns `tokens.map((token) => token.symbol).join('')`. Layout whitespace is never included. Source equations are tokenized from M1’s normalized display representation, excluding its single equality separator. A direct-reciprocal source is copied unchanged as canonical editable tokens; M2 does not reject or solve it.

### Selection and keyboard controls

The active side is chosen by an explicit labelled two-option side selector. Activating a palette token appends it to `selectedSide`, selects the inserted token, moves focus to it, and announces the outcome. Activating an active token selects it; focus alone does not change selection. Changing sides clears token selection.

The editor exposes these labelled controls, disabled when no selected token makes the action valid:

- **Move left** and **Move right** reorder the selected token in its current side.
- **Move to left side** and **Move to right side** append it to the destination side.
- **Delete selected token** removes it.

After reorder or cross-side movement, the moved token remains selected and receives focus. After deletion, select and focus the token now occupying the deleted index; otherwise select/focus the prior token; otherwise clear selection and focus the active-side selector. Boundary movement is disabled, not a silent no-op.

### Pointer interaction contract

A pointerdown records a potential drag without changing arrays. It becomes a drag only after movement of at least 6 CSS pixels for mouse/pen or 10 CSS pixels for touch. Before activation, normal touch scrolling is preserved; after activation, pointer capture is held and default selection/scrolling is prevented only for the dragged pointer. A palette drag creates a candidate token only after a valid drop. An existing-token drag retains its original `{ side, index }` until commit.

A drop index is computed against the target side’s rendered token midpoints: before the first midpoint inserts at `0`; after all midpoints appends; wrapped visual lines use each token’s on-screen rectangle in reading order. While reordering within a side, exclude the dragged token when calculating the target index. A same-side drop at the original effective index is a no-op that retains selection and focus.

On `pointerup`, commit only a valid side or trash target. On `pointercancel`, lost capture, invalid outside drop, reset, or session replacement, discard all transient drag data; existing arrays stay unchanged and palette drags create nothing. Every termination releases pointer capture, hides the preview/marker, and clears temporary drag styling.

## Dependency graph and delivery slices

```text
Approved M0 + M1 human gates and approved M2 spec/plan
                         │
                         ▼
M2.1 State, token model, canonical source-token conversion, serialization
                         │
                         ▼
M2.2 Active-row rendering, selection, side choice, toolbar controls
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
M2.3 Pointer drag/drop        M2.4 Keyboard-only editing
              └──────────┬──────────┘
                         ▼
M2.5 Accessibility, responsive styling, cleanup integration
                         │
                         ▼
M2.6 Deterministic checks and human acceptance gate
```

M2.3 and M2.4 have independent behavior but both modify the same single-file application, so implement them sequentially after M2.2. Styling, lifecycle cleanup, and verification are integration slices and follow both.

## Implementation order

### 1. Establish token data, state, and pure mutations

Add immutable-shaped token constructors, monotonic session-safe IDs, canonical source display splitting, `serializeTokens`, and pure mutation helpers for append, insert, remove, move, and selection normalization. Source conversion must retain canonical glyphs and parentheses exactly, while excluding equality and formatting spaces. Mutation helpers accept an active row and return new ordered arrays; they never inspect the DOM.

**Checkpoint:** Token sequences derived from `2x − 3 = 8`, `2(x + 1) = 6`, and `2 ÷ x = 12` serialize to parser-ready expressions. Insert/move/delete/cancel cases cannot duplicate or lose token IDs.

### 2. Render active editor and accessible non-drag controls

Replace M1’s disabled Step 2 preview with active-side controls, semantic token buttons, a selected-side indicator, selected-token styling, a movement toolbar, and a labelled trash target. Render Step 1 as locked static math and expose mutation handlers only for the active row. Use text labels and accessible symbol names; all enabled targets meet the 44px size requirement.

**Checkpoint:** A keyboard-only learner can choose a side, append palette tokens, select a token, reorder it, move it between sides, and delete it. Locked rows expose neither mutation controls nor drop targets.

### 3. Implement safe Pointer Events drag lifecycle

Add event delegation for palette and active-token pointer interactions. Use the approved drag threshold, pointer capture, preview, insertion marker, target calculation, and atomic commit/rollback rules. Reject invalid target drops without state mutation. Prevent native scrolling/selection only after an intentional drag starts; release all drag state on cancellation, lost capture, reset, and session change.

**Checkpoint:** Mouse and touch emulation can insert, reorder, cross-side move, and trash-delete; canceled and outside drags retain active arrays exactly.

### 4. Integrate keyboard focus, feedback, and responsive layout

Provide concise polite announcements for completed insert, move, and delete actions; do not announce every provisional marker movement. Enforce the stated focus transitions after toolbar and palette actions. Add active/selected/drag/target visual states distinct from focus and hover, a reduced-motion-safe presentation, and wrapping side layouts that avoid document-level horizontal overflow.

**Checkpoint:** At 320px, long token sequences wrap; selection and focus are visible; touch scrolling works before a drag activates; reset clears active editing, selection, indicators, capture, and feedback correctly.

### 5. Verify deterministic behavior and complete the M2 gate

Extend the browser-callable self-check infrastructure with pure token-model cases only. Run inline-script syntax validation, browser self-checks, and the M2 human script through direct-file and static-server paths. Record browser/version, widths, mouse/touch/keyboard evidence, and console/network results in the M2 task list.

**Checkpoint:** All automated checks pass; every M2 acceptance criterion and manual scenario passes with no uncaught console error or failed resource.

## Risks and mitigations

| Risk | Consequence | Mitigation |
| --- | --- | --- |
| Array indices become implicit token identities | Reorder/delete targets the wrong token after a mutation | Use stable generated IDs only; never use an index as a key or selection ID. |
| Touch drag blocks normal page scrolling | Mobile editor becomes frustrating or inaccessible | Do not prevent defaults until the pointer has crossed the explicit touch threshold. |
| Wrapped lines produce incorrect insertion positions | Tokens move to unexpected indexes | Use target token bounding-rect midpoints in DOM reading order and test multi-line before/between/after cases. |
| Canceled pointer paths mutate arrays | Tokens disappear, duplicate, or reorder unexpectedly | Retain source location and commit changes only on valid pointerup; centralize cleanup. |
| Selection and focus diverge | Keyboard controls affect an unexpected token | Store selection separately from focus; normalize it after every mutation and render distinct visual states. |
| Source display is reparsed from rendered DOM | Spacing or DOM changes alter learner math | Derive editable token arrays from private analyzed source/display data and serialize symbols only. |
| M2 validates source or active tokens | Direct-reciprocal sources cannot be edited and M3 seams blur | Keep all mathematics and validation out of M2; it only owns ordered token editing. |
| Locked rows receive delegated mutations | History integrity breaks | Require every UI mutation handler to resolve the current `activeRow.id`; locked rows render inert text only. |

## File-impact plan

| File | Planned change |
| --- | --- |
| `index.html` | Add token state/model, active-row rendering, non-drag controls, Pointer Events interactions, responsive/accessibility styling, and pure token self-checks. |
| `tasks/02-todo.md` | Mark implementation tasks and record validation evidence. |
| `docs/specs/milestones/02-touch-token-workspace.md` | Resolve the listed interaction decisions before code; do not alter it to excuse implementation behavior. |

## Verification checkpoints

1. **Token-model checkpoint:** Pure self-checks prove source conversion, IDs, serialization, insert/reorder/cross-side/delete, boundary no-ops, and cancellation rollback without DOM state.
2. **Keyboard/accessibility checkpoint:** Keyboard-only interactions prove all mutation paths, deterministic focus, announcements, selected/active side semantics, and locked-row protection.
3. **Pointer/mobile checkpoint:** Mouse and touch emulation prove activation threshold, scrolling before drag, preview/marker behavior, all valid destinations, cancellation, lost capture, and reset cleanup at 320px and desktop width.
4. **Release checkpoint:** Syntax check, self-checks, direct-file/static-server browser checks, console/network inspection, and the entire M2 human demo are documented by a reviewer.

## Definition of ready for implementation

- [ ] M0 Task 5 and M1 Task 6 human gates are complete.
- [ ] The M2 spec has incorporated or explicitly approved the proposed interaction contract above.
- [ ] This plan and `tasks/02-todo.md` are approved by a human reviewer.
- [ ] The one-file, dependency-free and exact-math boundaries remain unchanged.

## Completion rule

M2 is complete only after the reviewer confirms every supported interaction path returns a deterministic ordered active token array without mutating locked history, all keyboard alternatives work, and the M2 human browser gate is recorded in `tasks/02-todo.md`. M3 may then consume the serialized arrays for mathematical validation.
