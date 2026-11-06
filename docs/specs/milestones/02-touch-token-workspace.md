# Milestone 2 Spec: Touch-Friendly Token Workspace

**Status:** Complete — implementation and human browser acceptance passed. Detailed browser/version and viewport evidence was not captured in this session. Depends on accepted M0 and M1.

## Objective
Turn the static active equation into a robust, child-friendly token editor usable with mouse, touch, and keyboard/tap-only interaction. The editor produces ordered LHS/RHS token arrays for the validation milestone without permitting changes to locked history.

## Scope
- Render the palette: `0`–`9`, `.`, `+`, `−`, `×`, `÷`, `(`, `)`, and `x`.
- Populate every new active row from copies of the most recently accepted equation's canonical tokens.
- Add/move/reorder/delete tokens within the active LHS or RHS via Pointer Events.
- Use a drag-preview and insertion indicator that work for touch and mouse; disable accidental text selection/scroll only while an intentional horizontal/vertical token drag is active.
- Provide non-drag alternatives: palette click/tap inserts into selected side; clicking/tapping a token selects it; a labelled Delete selected token button removes it; a side-selector/focus rule makes target side unambiguous.
- Support keyboard movement/reordering through clearly documented controls (for example, selected-token Move left, Move right, Move to left side, Move to right side, and Delete), or an equally accessible, discoverable equivalent.
- Implement an explicitly labelled trash drop zone and safe no-op behavior if an invalid drag ends outside a target.
- Serialize tokens deterministically for M3's parser while keeping individual token objects with stable IDs and type/label metadata.

## Interaction rules
- Only `activeRow` tokens are interactive. Locked rows render semantic static math text and cannot be a pointer/keyboard drop target.
- A palette drag creates a new token only on successful drop; it must never remove the palette item.
- An existing-token drag moves that token atomically. If canceled, dropped outside valid targets, or interrupted, it remains in its original array/index.
- Dropping on a side uses the displayed insertion marker. If the side is empty, the token is appended as index `0`.
- Moving a token across sides removes it from the source and inserts it once at the target index.
- Trash deletion has an immediately announced result. No undo is required in this release; the user can reinsert a palette token.
- Palette insertion appends to the currently selected/focused side. A visible selected-side label must be present.

## Approved interaction contract

### State, tokens, and serialization

`activeRow` contains a stable row ID plus independent `leftTokens` and `rightTokens` arrays. Each token is `{ id, symbol, label }`: the ID is generated monotonically for the session and never derived from an array index; `symbol` is the parser-ready canonical glyph; and `label` is the readable spoken name. `selectedSide` is `left` or `right`; `selectedToken` is either null or `{ side, tokenId }`; and `drag` contains only transient Pointer Event state.

The editor tokenizes M1’s normalized `displayEquation`, removes the single equality token, and creates fresh token objects for both sides. `serializeTokens(tokens)` joins symbols with no layout whitespace. Direct-reciprocal sources are copied unchanged; M2 does not evaluate or validate any active-row expression.

### Selection and keyboard controls

The visible two-button side selector controls the palette destination. Activating a palette button appends a new token to that side, selects it, focuses it, and announces the result. Clicking or tapping anywhere inside a token side also selects that side as the palette destination; clicking a token still selects the token and its side. Activating a workspace token selects it; focus alone does not change selection. Switching sides clears the selected token.

The toolbar offers Move left, Move right, Move to left side, Move to right side, and Delete selected token. Invalid actions are disabled. Reordered or transferred tokens stay selected and receive focus. Deletion selects/focuses the token now at the deleted position, otherwise the previous token, otherwise the active-side selector.

### Pointer lifecycle and insertion

`pointerdown` records a potential drag without changing token arrays. A drag activates after 6 CSS pixels for mouse/pen or 10 CSS pixels for touch. Before activation, touch scrolling is unchanged; once active, the persistent active-equation container holds pointer capture and the dragged pointer’s default behavior is prevented. Palette drags create a token only after a valid side drop. Existing-token arrays are changed only at a valid drop commit.

The drop index uses rendered workspace-token rectangles in DOM reading order, excluding a dragged same-side token: before a token midpoint inserts before it and after all token midpoints appends. This applies across wrapped lines. A valid side drop inserts exactly once; trash removes an existing token. Pointer cancel, lost capture, outside drop, reset, and session replacement discard transient drag state, release capture, remove the preview/marker, and leave existing arrays unchanged. A completed palette drag suppresses its follow-up pointer-generated click so it cannot insert a second token.

## Accessibility and mobile behavior
- All interactive palette, token, toolbar, and trash controls use semantic buttons or keyboard-operable controls and have at least 44×44px hit areas.
- Text labels and ARIA labels name symbols in words: `×` is “multiply,” `÷` is “divide,” and so on.
- Screen readers receive a concise live announcement for insert, move, and delete operations, including side and position when useful.
- The token arrangement uses wrapping rather than shrinking below legibility or causing document-level horizontal scrolling.
- Pointer capture is released on `pointerup`, `pointercancel`, and component/session reset. Events must not leave the page with stuck dragging state.

## Non-goals
- Checking syntax, equation equivalence, locking a new history row, or win detection (M3/M4).
- Native HTML drag-and-drop, SortableJS, or any other dependency.
- Freehand text editing inside a token side.
- Undo/redo, token grouping, copy/paste, or multi-select.

## Acceptance criteria
- [x] After a valid M1 source equation starts, the active row displays independent LHS/RHS token sequences copied from Step 1 and a static equality symbol.
- [x] Every required palette token can be appended by selecting a side and activating the palette button without any drag gesture.
- [x] Existing tokens can be selected and deleted through a visible button, with the chosen side/selection state announced.
- [x] Mouse and touch/pointer emulation can drag a palette token into either side, reorder an existing token, move one across sides, and drop one into trash.
- [x] Canceled/outside drags do not lose, duplicate, or reorder any existing token.
- [x] A locked Step 1 cannot be edited through pointer, click, keyboard, or programmatic row event handlers exposed by UI controls.
- [x] A keyboard-only user can perform insert, delete, reorder, and cross-side movement without a pointer gesture.
- [x] At 320px, large token sequences wrap inside their side and the page does not horizontally scroll.
- [x] The resulting ordered token arrays serialize to the intended expression text, including adjacency needed for `2x` and `2(x+1)`.

## Human demo script
1. Start `2x - 3 = 8`. Confirm Step 1 is static and Step 2 contains copy tokens on both sides.
2. Select LHS using the visible side control. Activate palette `+`, `1`; confirm both tokens append to LHS, receive meaningful announcements, and do not alter Step 1.
3. Select an inserted token and use Delete selected token. Confirm only that active token is removed.
4. With a mouse, drag `2` from the palette to the start of RHS; reorder it to the end; then drag it to LHS. Confirm insertion location and exactly one moved token after every step.
5. Drag a token to trash. Confirm it is removed. Start a drag of an existing token and release it outside a target; confirm the token returns/retains its original position.
6. Use browser device emulation or a touch device to repeat palette-to-side, reorder, cross-side move, and trash deletion. Confirm scrolling still works when not dragging.
7. Without using a mouse, tab through side/select/edit controls and use the documented keyboard controls to insert `x`, move it between sides, reorder it, and delete it.
8. Narrow the viewport to 320px, create a long token line, and confirm readable wrapping/no horizontal overflow. Reset and verify active editor state clears safely.

## Verification evidence
- Human demo passes using mouse and touch emulation or a physical touch device.
- Keyboard-only pass documents the actual keys/controls used.
- Console remains clear after canceled pointer interactions and reset.

## Exit gate and dependencies
**Entry:** M0 and M1 are accepted; this spec is approved.

**Done:** The editor reliably returns ordered active-side tokens through every supported input path. M3 may consume those tokens for parsing and validation, and must preserve the M2 accessibility paths when it locks/spawns rows.
