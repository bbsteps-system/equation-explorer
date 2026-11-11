# RN-M0 Baseline Evidence

- Date: 2026-11-11 (pre-move checks 23:26–23:35 HKT)
- Baseline commit: `fa97054047d2fe8ae94227ebfd44103f84fee23d`
- Baseline tag: `web-baseline-2026-11-11` (annotated)
- Backup branch: `backup/web-baseline-2026-11-11`
- SHA-256 of pre-move `index.html`: `70b338813133b28c62657b3d42009df9f8e2cc752a3c2d4c8d69ff4c040159a0`
- SHA-256 of `web/index.html`: `70b338813133b28c62657b3d42009df9f8e2cc752a3c2d4c8d69ff4c040159a0` (matches the pre-move baseline)
- Tag/branch pushed to origin: no — push not approved; local-only backup
- macOS: 26.4.1 (25E253)
- Browser(s): Google Chrome 157.0.8103.39 (headless, driven through the Chrome DevTools Protocol)
- Viewports: 320×780, 768×1024, 1024×768, 1280×900
- Serving command (pre-move): `python3 -m http.server 8000 --bind 127.0.0.1 --directory .`
- Serving command (post-move): `python3 -m http.server 8000 --bind 127.0.0.1 --directory web`
- Console errors: none observed
- External network requests: none; the only request was the local document
- Keyboard pass: pass
- Reduced-motion pass: pass
- Screen reader pass: not performed
- Audio pass: chime scheduling exercised, not audibly verified
- Limitations: see "Deviations and limitations" below

## Method

The checklist was executed against the root `index.html` through an automated
Chromium session (Chrome 157 headless, Chrome DevTools Protocol). The temporary
driver scripts lived under `/tmp` and are not committed. The run captured DOM
state after each action, screenshots, console output, and network requests.

This is **not** a human manual acceptance pass. It records observed runtime
behavior so the human reviewer can accept the baseline at Checkpoint A. The
human gate remains open and must be signed separately.

The implementation request covered the whole milestone, so the move followed
evidence capture without a separate human Checkpoint A sign-off. The reviewer
may accept the evidence retrospectively or require follow-up changes; the tag,
checksum, and byte-identity checks preserve the option to redo the move.

The documented serve command omits `--bind`; port 8000 on the wildcard address
was already held by a local Docker process, so the checks used
`--bind 127.0.0.1` with the same port and directory. This is an environment
accommodation, not a project change.

## Pre-move checklist results

| Step | Result | Notes | Evidence |
| --- | --- | --- | --- |
| 1 | Pass | Root `index.html` opened through `file://` and through the local server. Both rendered the landing screen with no console errors, no exceptions, and no external requests. The only network request was the document itself. | `landing-1280.png`; console/network capture in the run log |
| 2 | Pass | Started `2y - 3 = 8`. Step 1 locked as `2 × y − 3 = 8`; Step 2 appeared editable with token copies `2 × y − 3` / `8`. | `started-step2-768.png` |
| 3 | Pass | Palette tap inserted `9`; toolbar Move right reordered `1,2` to `2,1`; Move to right side moved a token across; Delete selected token removed it; mouse drag to the trash deleted a token; mouse drag reordered `1,2` to `2,1`; touch-emulated drag moved `5` from the left side to the right side. Row state was asserted after every action. | Run log (step 3a–3g) |
| 4 | Pass | Built the malformed left side `2y −` and pressed Check. The row stayed editable, the left side received error styling (`data-error="true"`), the message was "This side needs a number, y, or parentheses after that operation.", and no history row was created. | Run log (step 4) |
| 5 | Pass | Submitted the inequivalent `2y - 3 = 9`. Rejected with "That changes the answer for y. Try doing the same operation to both sides!", no lock, no history row, tokens preserved. | Run log (step 5) |
| 6 | Pass | Built `2y = 11`; Check locked Step 2, showed the success toast, and created Step 3. Built `y = 11 ÷ 2`; Check completed, showed the completion panel with "The answer is y = 11 ÷ 2.", displayed confetti (no-preference emulation), and constructed an `AudioContext` for the chime. | `completed-768.png`; run log (step 6) |
| 7 | Pass | `y = 5.5` and `y = 11 ÷ 2` both completed exactly. Mixed-fraction setup `2y - 3 = 5 1/2` was accepted and displayed as `2 × y − 3 = 5 1/2`. | Run log (step 7) |
| 8 | Pass | `2/y = 12` and `12 = 2/y` were accepted as direct reciprocal sources without revealing the answer. `y² = 4`, `y = y`, `y = y + 1`, `2y =`, `y/0 = 1`, `2/(y + 1) = 3`, `1/y + 1/(y - 1) = 2`, and `z = 3` were each rejected with a specific message, with no page failure. | Run log (step 8) |
| 9 | Pass (one deviation) | No horizontal overflow at 320, 768, 1024, or 1280 px, on landing or during a session. The toolbox pinned at 8 px below the viewport top while solving, cleared the active row, and returned to `static` flow on completion. A focused token scrolled clear of the toolbox. Enter in the input started the session. Tab reached the palette and toolbar controls; every reached control matched `:focus-visible` and displayed the 4 px focus outline. Enter on a focused palette token inserted it. Deviation: the two clear-side (×) buttons measure 32×44 px at 320 px, below the 44×44 target in the product spec. See "Deviations and limitations". | `landing-320.png`, `landing-768.png`, `landing-1024.png`, `active-320.png` |
| 10 | Pass | With `prefers-reduced-motion: reduce` emulated, completion remained fully understandable (text, toast, answer), confetti was suppressed, and the locked-row animation duration computed to `1e-05s`. With the `AudioContext` constructor throwing, and with `AudioContext` removed entirely, completion still succeeded. | Run log (step 10) |
| 11 | Pass | Before any lock, editing Step 2 and pressing Restore returned both sides to Step 1, disabled Restore, and focused the left side. After locking `2y = 11`, editing the new Step 3 and pressing Restore returned to the locked Step 2 tokens without altering locked history. | Run log (step 11) |
| 12 | Pass | Clear left emptied only the left side, cleared the selection, moved focus to the left side, and disabled only the left clear button. Flip swapped both sides, cleared the selection, kept focus on the Flip button, and left history unchanged. With both sides empty, both clear buttons and Flip were disabled. | Run log (step 12) |

### Console, exceptions, and network

- Console errors: none across all runs.
- Uncaught exceptions: none.
- Failed requests: none.
- Requests observed: the local document URL only (`http://127.0.0.1:8000/` or
  `file:///…/index.html`). No external request was made.

## Post-move smoke flow

Served from `--directory web` at 768×1024 through the same automated Chromium
session.

| Step | Result | Notes |
| --- | --- | --- |
| 1 | Pass | Landing rendered with no console errors, no exceptions, no failed requests, and no external requests; only the local document was requested. |
| 2 | Pass | `2y - 3 = 8` started; Step 1 displayed locked as `2 × y − 3 = 8`; Step 2 was editable and prefilled. |
| 3 | Pass | Built `2y = 11`; Check locked Step 2 and created the next editable row. |
| 4 | Pass | Built `y = 11 ÷ 2`; Check completed with "The answer is y = 11 ÷ 2." and the completion panel visible. |
| 5 | Pass | "Solve another!" returned the setup screen (`mode=landing`, session hidden), cleared the history, showed "Ready for a new equation.", and moved focus to `#equation-input`. |

The moved file's SHA-256 equals the tagged baseline, so together with this smoke
flow the move is behavior-preserving.

## Documentation reconciliation

- README current status corrected: yes — web implemented through M4, accepted
  as parity baseline; native planned; `mobile/` created in RN-M1.
- README run command updated: yes — `--directory web`.
- README deployment and hosting-root note: yes.
- CLAUDE.md web/native split updated: yes — web rules scoped to `web/`, native
  rules scoped to `mobile/`, explicit that the no-build rule does not apply to
  native.
- Product spec paths updated (`web/index.html`, `--directory web`): yes — the
  diff contains path and command strings only; no requirement, notation,
  example, or acceptance behavior changed.
- ADR-011 records the layout, PB-02/PB-10 resolution, hosting follow-up, and
  historical-docs policy: yes.
- RN task conventions added: yes — `tasks/react-native/README.md`.
- Historical `x`/root-path drift recorded, not rewritten: yes — see below.
- Hosting-root action item recorded: yes — ADR-011 and README.

## Historical drift

The historical web milestone specs (`docs/specs/milestones/00`–`04`) and task
files (`tasks/00`–`04`) still contain `x` examples and root-path serve commands.
They are records of the work as planned and delivered and are intentionally not
rewritten. The authoritative current-state documents use `y` and the `web/`
paths. The product spec already used `y`; only its paths changed.

## Remote backup status

The annotated tag `web-baseline-2026-11-11` and the backup branch
`backup/web-baseline-2026-11-11` exist locally. **Local-only backup; push not
approved.** No remote push was attempted. The owner may approve pushing both refs
to `origin` as a follow-up.

## Deviations and limitations

1. **Automation, not a human pass.** All checks were driven programmatically.
   Visual judgment (layout aesthetics, animation smoothness) was not assessed by
   a human reviewer.
2. **No screen reader pass.** VoiceOver was not driven or observed. The
   automated run verified semantic roles, labels, live regions, and focus
   behavior in the DOM but did not confirm screen-reader speech output.
3. **Audio not audibly verified.** The chime path was exercised and an
   `AudioContext` was constructed, but no one listened to the output. Blocked
   and missing audio were both shown not to block completion.
4. **No physical touch device.** Touch behavior was exercised through
   DevTools touch-event emulation, not on hardware.
5. **Clear-side buttons below the target size.** At 320 px the clear-side
   buttons measure 32×44 px. The product spec's non-functional requirements ask
   for at least 44 px touch controls. This is a pre-existing web-app deviation;
   RN-M0 must not change web UI, so it is recorded here rather than fixed.
6. **Stale landing help text.** The setup helper still reads "You can use any
   example for now. Equation checking comes next." while equation checking is
   implemented. This is pre-existing web copy; RN-M0 records it and does not
   change web content.
7. **Reduced motion at OS level.** The implementation machine reports
   `prefers-reduced-motion: reduce` by default. Both reduce and no-preference
   were emulated; the confetti path was verified only under emulated
   no-preference.
8. **Port binding.** Port 8000 was occupied on the wildcard address by Docker,
   so the documented command was run with `--bind 127.0.0.1`.

## Screenshots

- `assets/RN-M0/landing-320.png` (320×780)
- `assets/RN-M0/landing-768.png` (768×1024)
- `assets/RN-M0/landing-1024.png` (1024×768)
- `assets/RN-M0/landing-1280.png` (1280×900)
- `assets/RN-M0/active-320.png` (320×780, Step 2 active)
- `assets/RN-M0/started-step2-768.png` (768×1024, Step 2 active)
- `assets/RN-M0/completed-768.png` (768×1024, completed with celebration)

## Verification (RN-M0 Task 10)

| Check | Result |
| --- | --- |
| Tag and backup branch resolve to the same commit | Pass — both `fa97054047d2fe8ae94227ebfd44103f84fee23d` |
| `web/index.html` hash equals the tagged baseline | Pass — both `70b3388…59a0` |
| Web app appears as a rename, not a rewrite | Pass — `git diff --find-renames --summary` reports `rename index.html => web/index.html (100%)` |
| Post-move smoke flow | Pass — all 5 steps |
| Current-state docs contain no `--directory .` | Pass — no matches |
| Current-state docs contain no bare root `index.html` | Pass — no matches |
| `mobile/`, `package.json`, `node_modules/`, `ios/`, `android/` absent | Pass — all absent |
| Relative links in current docs resolve | Pass — all resolve (two RN milestone documents had `../../react-native-*.md` links corrected to `../../../`; those files are current RN docs, not historical web records) |
| Historical web milestone/task files unchanged | Pass — no modifications under `docs/specs/milestones/` or `tasks/00`–`04` |

## Git refs

```sh
git rev-parse web-baseline-2026-11-11^{commit}    # fa97054047d2fe8ae94227ebfd44103f84fee23d
git rev-parse backup/web-baseline-2026-11-11      # fa97054047d2fe8ae94227ebfd44103f84fee23d
git rev-parse HEAD                                # fa97054047d2fe8ae94227ebfd44103f84fee23d
```