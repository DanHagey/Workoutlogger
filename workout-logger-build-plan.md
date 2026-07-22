# Workout Logger — Staged Build Plan (learning build)

One-screen workout entry form for iOS Safari. Single static file (`index.html`),
vanilla HTML/CSS/JS, no backend, no framework, no dependencies. On submit,
assembles a Markdown note client-side and redirects to `obsidian://new`.

**How this build runs:** six stages, one at a time. Claude Code builds a stage,
briefly explains what was added and why, then STOPS and waits for Dan to verify
in a browser before continuing. Do not build ahead. Dan is using this project to
practice programming fundamentals from CS50P — explanations should name the
concept in play, not just the fix.

## Instructions to Claude Code (read first)

1. Build ONE stage per turn. End each turn with the verification steps for Dan.
2. Keep explanations short: 2–4 sentences on what changed and the concept behind
   it. No lectures.
3. When Dan reports something broken, diagnose the root cause plainly before
   fixing. Don't just patch and move on.
4. The note-assembly logic (Stage 3) must be a pure function — takes an object
   in, returns a string out, touches no DOM. This is deliberate: it makes the
   core logic testable from the browser console without submitting the form.
5. Never change the locked decisions below, even if an alternative seems better.

## Locked decisions (do not re-derive or change)

- Vault name: `second-brain`
- Target file: `Inbox/Workout YYYY-MM-DD` (date-only; same-day collisions out of scope)
- URI: `obsidian://new?vault=second-brain&file=<path>&content=<content>`
- Encoding: `encodeURIComponent()` on the `file` and `content` values — never
  `encodeURI()`. A raw `&`, `#`, or newline in typed text corrupts the URL under
  the lighter encoding; that is the original Shortcuts bug this replaces.
- Note body must match the template below exactly — labels, casing, the
  two-space indent on `pre:`/`mid:`, and blank lines.

## Note template (exact output)

```
Date: {date}
Focus: {focus}
Location: {location}
Fuel:
  pre: {fuel_pre}
  mid: {fuel_mid}
Energy: first half {energy1}/10, second half {energy2}/10

{exercise_block}

Total time: {total_time}
Active calories: {active_cal}
Average heart rate: {avg_hr}
Heart range: {hr_range}
```

Rules: empty optional fields keep the label with an empty value (e.g. `  mid: `);
exercise block is verbatim with internal blank lines preserved and trailing
blank lines trimmed; assemble the whole note as one string, then encode it once.

## Fields (single screen, top to bottom)

| # | Field | Widget |
|---|-------|--------|
| 1 | Date | `<input type="date">`, defaulted to today via JS |
| 2 | Focus | Chip group: Lower / Upper Push / Upper Pull / Full Body / Other (reveals text input) |
| 3 | Location | Chip group: Rocklin / Citrus Heights / Other (reveals text input) |
| 4 | Fuel pre | Short text, optional |
| 5 | Fuel mid | Short text, optional |
| 6 | Energy first half | Chips 1–10, two rows of five |
| 7 | Energy second half | Chips 1–10, two rows of five |
| 8 | Exercise block | `<textarea>`, min 10 rows |
| 9 | Total time (min) | Text, `inputmode="numeric"` |
| 10 | Active calories | Text, `inputmode="numeric"` |
| 11 | Average heart rate | Text, `inputmode="numeric"` |
| 12 | Heart range | Text, free (e.g. `105-164`) |

Chips are `<button type="button">` elements with a selected class, minimum
44×44px targets, filled high-contrast selected state, exactly one selected per
group, no deselect-on-retap.

---

## Stage 1 — Static skeleton

**Build:** the full HTML structure: all 12 fields, labels, submit button, dark
mobile-first styling (single column, ~390px, system fonts, no horizontal
scroll). No JavaScript yet except defaulting the date input to today. Chips
render but don't respond to taps.

**CS50P parallel:** this is the "define your data before your logic" step —
same instinct as sketching your variables and input() calls before writing
conditionals.

**Dan verifies:** open the file in a desktop browser at narrow width (or
iPhone). All fields visible in one scroll, date shows today, nothing
interactive yet. Ugly-but-complete is the bar.

## Stage 2 — Chip behavior + "Other" reveals

**Build:** tap-to-select on all four chip groups (one selected per group).
Selecting "Other" on Focus/Location shows a text input below and focuses it;
selecting a named chip hides and clears it.

**CS50P parallel:** event handlers are functions passed as arguments — the JS
equivalent of passing a function name around in Python. The selected-chip
tracking is state, like a variable you update inside a loop.

**Dan verifies:** tap through every chip group; confirm single-select, confirm
both "Other" reveals show/hide/clear correctly.

## Stage 3 — Note assembly (the core)

**Build:** a pure function `buildNote(data)` — takes a plain object of field
values, returns the finished note string per the template. Also a
`collectForm()` function that reads the DOM into that object. Wire nothing to
submit yet.

**CS50P parallel:** this is exactly a CS50P exercise: f-string-style assembly
(JS template literals use `${}` where Python uses `{}`), a dict → an object,
and separating pure logic from I/O so it's testable — same reason CS50P has you
write testable functions for `check50`.

**Dan verifies:** in the browser console, run
`buildNote(collectForm())` after filling the form, and eyeball the string
against the template — including the two-space indents and blank lines. This
console check is the habit worth keeping: prove the core logic before wiring
the button.

## Stage 4 — Encoding + redirect

**Build:** submit handler that calls the Stage 3 functions, builds the URI with
`encodeURIComponent()` on the file path and content, and sets
`window.location.href`.

**CS50P parallel:** encoding is the same problem as escaping quotes in strings
or sanitizing input — the data is fine, the *transport* has reserved
characters. `encodeURIComponent` vs `encodeURI` is a scope difference: the
former escapes `&`, `#`, `=` too, which is why the lighter one broke Shortcuts.

**Dan verifies (the regression test — do this one carefully):** on desktop,
log the URI to console instead of redirecting isn't needed — just check on
iPhone: fill the form with an exercise block containing `&`, `#`, `%`, quotes,
and blank lines. Submit. Note must land in Obsidian as
`Inbox/Workout {date}.md` with every character intact.

## Stage 5 — Validation + copy fallback

**Build:**
- Validation on submit: Focus, Location (non-empty text if "Other"), both
  Energy values, and exercise block are required. On failure, scroll to and
  highlight the first bad field. No `alert()`.
- After a successful submit: reveal a persistent strip with a **Copy note**
  button (`navigator.clipboard.writeText` of the plain note text, with a
  "Copied" state) and hint text: "If Obsidian didn't open, copy the note and
  paste it manually." Custom-scheme handoffs fail *silently* on iOS — there is
  no error to catch, so the fallback is always shown, never gated on detection.
- Do not clear the form after submit; if the redirect fails, the data must
  survive.

**CS50P parallel:** validation is guard clauses — the `if not x: return` pattern
from CS50P's input-checking exercises. The always-on fallback is defensive
programming when the failure mode is undetectable.

**Dan verifies:** submit with missing required fields (should highlight, not
navigate); submit valid (Obsidian opens, copy button works, form data intact).

## Stage 6 — Polish + acceptance pass

**Build:** spacing, selected-state contrast in bright light, full-width ≥52px
submit button, and a pass through the acceptance checklist. No animations
beyond simple state changes, no new features.

**Dan verifies:** full acceptance checklist below, on the iPhone, launched from
the Home Screen web clip specifically (web clips run in a slightly different
Safari context than a tab — this is where scheme-handoff quirks would surface).

---

## Delivery

Host `index.html` on GitHub Pages, then Safari → Share → Add to Home Screen.
A purely local file doesn't work: iOS Files opens HTML in Quick Look, which
does not execute JavaScript, and `file://` URLs can't be web-clipped.

## Acceptance checklist

- [ ] Loads with today's date pre-filled
- [ ] All 12 fields on one scrollable screen; no multi-step flow
- [ ] Chip groups single-select; "Other" reveals show/hide/clear correctly
- [ ] Special-character paste (`&`, `#`, `%`, quotes, blank lines) arrives intact
- [ ] Note matches template character-for-character, including `  pre: ` /
      `  mid: ` with empty values
- [ ] Copy-note button appears post-submit and copies plain note text
- [ ] Form data survives submit
- [ ] Zero network requests beyond the page itself
- [ ] Works launched from the Home Screen web clip

## Out of scope (v1)

Same-day collision handling, multi-vault support, offline/service worker,
read-back of notes, any backend. Known quirk to expect, not a bug: logging
twice in one day opens the existing note instead of appending.

## Optional follow-up exercise (deferred, actual Python)

A CLI version of Stage 3 in Python — `build_note(data: dict) -> str` with a
couple of tests — is a clean CS50P-native rep of the same logic. Worth doing
after the app ships, not before.
