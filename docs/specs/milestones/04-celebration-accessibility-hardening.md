# Milestone 4 Spec: Celebration, Accessibility, and Release Hardening

**Status:** Implemented — automated verification is complete; the human demo and release checklist remain the acceptance gate. Depends on M0–M3.

## Objective
Finish Equation Explorer as a joyful and reliable standalone learning experience. Add an exact isolated-variable win condition, resilient celebration effects, and the final accessibility/responsive/compatibility pass.

## Scope
- Recognize final form only when the accepted candidate is structurally `x = constant`, where LHS represents exactly the canonical token form `x`, RHS is a rational constant, and it matches the original exact solution.
- Support final answers expressed through the palette as an integer, terminating decimal, or improper fraction division expression. The setup parser remains the only route for whitespace-based mixed fractions; the UI may display a mixed fraction only if a canonical formatting policy is separately approved.
- Replace M3's neutral final-form placeholder with textual praise, an accessible completion panel, “Solve another!” reset path, and one-time confetti.
- Play a short synthesized Web Audio chime after a user action when possible; failure to construct/resume AudioContext must be caught and must not affect completion.
- Honor `prefers-reduced-motion`: minimize/remove particle and shake motion while retaining clear success text and icon.
- Perform final browser, responsive, keyboard, and offline/direct-file validation.

## Approved implementation contract
Approved with the M4 plan in `tasks/04-plan.md` and implemented on 2026-11-07.

- M3's `<p id="final-status">` placeholder and its focus target are removed. A semantic `#completion-panel` (`role="region"`, labelled by its heading, described by its praise and answer, `tabindex="-1"`) is the only completed-state outcome and receives focus. Its answer text comes from the final locked row's verified `displayEquation`.
- “Solve another!” calls the existing reset path. The top reset control and starting a new equation from setup remain available alternatives.
- Confetti is one fixed, pointer-transparent, `aria-hidden` DOM/CSS layer with 42 deterministic pieces and a 3.2-second cleanup timer. No layer or pieces are created when `prefers-reduced-motion: reduce` matches.
- The chime is a low-volume, three-note Web Audio signal, lazily created from the final Check gesture. No visible sound toggle is added. Constructor, `resume()`, scheduling, and `close()` failures are caught silently.
- `state.celebrationPlayed` permits one effect launch per completed session. Effects run only after the completed DOM renders; render, load, reset, and self-checks never launch them.
- `startSession()` and `resetSession()` clear the confetti timer/layer, close the audio context, and reset the one-time flag. Reset during animation removes the layer immediately.
- No dependency, network request, build step, font, image, persistence, or analytics is added. The only new asset is an inline data-URI favicon that prevents a local-server 404.

Automated evidence is recorded in `tasks/04-todo.md`. The human demo and final release checklist below have not yet been completed, and M3's human acceptance gate also remains open.

## Win detection contract
- The candidate must already have passed all M3 equivalence checks.
- Left-side tokens must be semantically canonical `x` after display-normalization. Do not celebrate `1x`, `(x)`, `x + 0`, or `2x ÷ 2` unless a future approved canonicalization policy deliberately rewrites the active row before validation.
- The right side must reduce to a constant rational equal to the retained source solution; it may be tokenized as `5.5`, `11 ÷ 2`, or appropriate integer digits/operators.
- Celebration must fire once per completed session. Re-activating controls after completion cannot replay it or append new rows.
- The completion panel must state the verified answer in accessible text and provide a focusable “Solve another!” button that resets all source/history/token/celebration state.

## Effect design and resilience
- Prefer locally generated DOM/CSS or canvas particles—no images, fonts, or network downloads. Effects clean up their nodes/timers after completion.
- The sound uses Web Audio API oscillator/gain nodes with low volume and short duration; it must be initiated only from the user’s Check action flow.
- A visible sound toggle is optional. If included, it must be keyboard accessible and default to a safe, non-disruptive behavior. Audio permission/autoplay limitations are normal, not errors shown to children.
- Reduce-motion mode uses static celebratory decoration or a single non-moving check icon, not repeated moving particles.
- Do not convey winning only through color, sound, or animation.

## Final hardening requirements
- Verify no horizontal scrolling at 320px-wide portrait view and common desktop widths.
- Verify no console errors when opened from `file://` and when served through Python’s local static server.
- Test current Chromium and at least one of Safari or Firefox when available. Record unavailable-browser limitations rather than claiming unperformed checks.
- Use native HTML/CSS/JS compatible with current evergreen browsers; do not add a polyfill or dependency without approval.
- Ensure reset during/after confetti clears timers, pointer state, live-region state, locked rows, active token selection, and completion state safely.

## Non-goals
- Audio recordings, external music, user accounts, leaderboard, saved achievements, analytics, printing, automatic solution reveal, or advanced lesson content.
- A new test framework or package-based build pipeline.
- Supporting non-unique/nonlinear equations beyond M1's friendly rejection.

## Acceptance criteria
Automated evidence for these criteria is recorded in `tasks/04-todo.md`; the boxes remain unchecked until the human demo and reviewer sign-off.

- [ ] For `2x - 3 = 8`, valid steps `2x = 11` then `x = 11 ÷ 2` result in a locked final row, one completion panel, textual praise, and one celebration only.
- [ ] Fresh sessions ending in `x = 5.5` and `x = 11/2` also complete; exact rational comparison recognizes both as `11/2`.
- [ ] A valid equivalent but non-isolated row such as `2x = 11` never celebrates.
- [ ] An equivalent but noncanonical left side such as `1x = 11/2` does not falsely complete; it remains a valid next step according to M3 rules.
- [ ] Audio failure/blocking does not cause an error UI, prevent visual celebration, or prevent reset.
- [ ] With reduced motion enabled, win/error meaning remains clear without moving particles or shake animation.
- [ ] “Solve another!” completely resets the session, focuses the setup input, and permits a new complete game without stale effects or duplicated handlers.
- [ ] The final release checklist in the umbrella spec passes, including responsive, pointer, keyboard, invalid-input, and direct-file checks.

## Human demo script
1. Serve the app locally, start `2x - 3 = 8`, and complete `2x = 11` then `x = 11 ÷ 2`. Confirm the final row locks and celebration panel/confetti appears once with readable praise.
2. Activate Check again or interact with any residual row control if available. Confirm no second celebration, no new history, and no error.
3. Select Solve another!, confirm return to initial setup and focus, then complete fresh sessions through `x = 5.5` and `x = 11/2`.
4. In another session, submit valid `1x = 11/2`. Confirm it is an accepted non-final step—not a win—then make canonical `x = 11/2` and confirm completion.
5. Block or disable audio in browser settings/devtools where possible and complete a solution. Confirm visual completion works and console remains clear. Repeat with reduced motion enabled; confirm effects are non-moving but success remains explicit.
6. Open `index.html` directly, then run the same primary success case using `python3 -m http.server 8000 --directory .`. Confirm both work without network requests.
7. At 320px, tablet, and desktop widths, execute a minimal setup/edit/check/win path. Tab through setup, palette, active row, feedback, reset, and completion controls; confirm visible focus, labelled controls, and no horizontal overflow.
8. Run invalid-source and invalid-step checks from the umbrella final release checklist. Record browser/version and any Safari/Firefox result available.

## Verification evidence
- A completed final release checklist with browser/version and viewport results.
- Human confirms direct-file and local-server operation, one successful integer/fraction/decimal completion each, and reduced-motion/audio-failure behavior.
- Console inspection shows no uncaught exceptions or failed remote resource requests.

## Exit gate and dependencies
**Entry:** M0–M3 are accepted; this spec is approved.

**Done:** All acceptance criteria and the complete human demo script pass. The reviewer approves the final experience and browser evidence. The application is then ready for a separate explicit request to commit, publish, or deploy.
