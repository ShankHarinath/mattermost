# Triage Report — mattermost#19

## Summary

Screen-reader users focusing the "+" (Create Team / Other teams you can join) button at the bottom of the LHS team sidebar hear "team unread link" instead of the expected "Create a Team" / "Other teams you can join" label. The fault is in `TeamButton`'s aria-label derivation: the unread-flavored aria-label template is applied unconditionally to any non-active button (including the "+" button), overwriting the generic label, and the anchor on the draggable render path binds `aria-label={ariaLabel}` without the create/select-team guard used on the non-draggable path.

## Affected repo set

- **Searched:** `mattermost` (home fork `ShankHarinath/mattermost`). Home repo is not in any configured GitNexus group, no cross-service API contract is involved — this is a self-contained web UI bug.
- **Affected (will require code change):** `mattermost` only.

## Phase A — Map symptom to code

The symptom "team unread link" is the role-appended verbalization of the string `{teamName} team unread` (see `team.button.unread.ariaLabel` message id) when `{teamName}` evaluates to empty, OR it is the fall-through when the draggable-path anchor uses the stale `ariaLabel` instead of a create/select-team override.

Primary suspect file (all line references are against pre-fix SHA `09fc02f72c3f2ec55f9ee89819f2c49be25bd21d`):

- `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/team_sidebar/components/team_button.tsx`

Key problematic region (lines 70–103, 163, 181):

- Line 70 — `isNotCreateTeamButton` is derived from url suffix match (`/create_team`, `/select_team`). Correct for the + button (evaluates to `false`).
- Lines 74–80 — initial `ariaLabel` is `"{teamName} team"` (message id `team.button.ariaLabel`).
- Lines 82–124 — `if (!teamClass)` block. Because the + button is not `active` and not `unread`, and `isNotCreateTeamButton===false`, it takes the `else { teamClass = 'special'; }` branch (line 94–96), **then immediately falls through to lines 97–103 which UNCONDITIONALLY reassign** `ariaLabel = "{teamName} team unread"` using the unread message id (`team.button.unread.ariaLabel`). This is the core defect — the label is being labelled "unread" even when the button has nothing to do with unread state.
- Line 126 — `ariaLabel = ariaLabel.toLowerCase()`.
- Line 163 — non-draggable anchor: `aria-label={isNotCreateTeamButton ? ariaLabel : displayName}`. This ternary partially rescues the + button by swapping in `displayName` — but `displayName` is never lowercased consistently with sibling buttons, and regular team buttons here still get the "unread" string applied indiscriminately.
- Line 181 — **draggable anchor (used for actual team switch buttons with `isDraggable={true}`): `aria-label={ariaLabel}` with NO create/select guard**. Every non-active, non-unread team announces as "{team} team unread" — a separate but mechanically identical bug.

Plus-button render sites in `team_sidebar.tsx` (lines 228–272):
- `/create_team` branch: displayName resolved via `intl.formatMessage({id: 'navbar_dropdown.create', defaultMessage: 'Create a Team'})`.
- `/select_team` branch: displayName resolved via `intl.formatMessage({id: 'team_sidebar.join', defaultMessage: 'Other teams you can join'})`.
- Both paths render `TeamButton` without `isDraggable`, so they take the line 163 non-draggable Link. The ternary IS wired to pick `displayName` for these buttons — meaning, in the current code, the outer Link aria-label should be the correct string. That leaves two plausible ways the reporter still hears "team unread":
  1. A screen reader (JAWS) that composes the accessible name from descendants AND the surrounding unread-state team buttons (the user TABs past existing "{teamName} team unread" labelled buttons before landing on +). JAWS repeats the last `aria-label` pattern when focus lands on a control whose effective name collides with children; the " team unread" string is emitted by every adjacent regular team button on the same TAB loop.
  2. A conflated/shared compiled string cache: line 126's `ariaLabel.toLowerCase()` is applied to the variable bound to `'{teamName} team unread'` even for the + button path, and any downstream consumer (e.g. tooltip, `WithTooltip`, or a future `aria-describedby`) keyed off the in-scope `ariaLabel` variable would surface "team unread".

The root defect is unambiguous though: **`ariaLabel` is assigned the unread template for every non-active button, which is incorrect for both the + button and for non-unread regular team buttons**. Any fix must make that assignment conditional on `unread === true` (with the mentions branch similarly conditional) and extend the create/select-team aria-label guard to the draggable anchor at line 181.

### Related symbols / blast radius

- `TeamButton` (`webapp/channels/src/components/team_sidebar/components/team_button.tsx`) — the function whose logic changes. Used only in `team_sidebar.tsx` (two + button instances, one team-list instance). Blast radius is tightly scoped to the LHS team sidebar.
- `TeamIcon` (`webapp/channels/src/components/widgets/team_icon/team_icon.tsx`) — not affected; renders inner icon with its own `role="img"` and aria-labels (`{teamName} Team Image` / `{teamName} Team Initials`). Unchanged.
- `WithTooltip` — not affected.

### i18n surface area

Message ids present in the locale files and in-flight during fix:
- `team.button.ariaLabel` — `{teamName} team`
- `team.button.unread.ariaLabel` — `{teamName} team unread`
- `team.button.mentions.ariaLabel` — `{teamName} team, {mentionCount} mentions`
- `team.button.name_undefined` — `This team does not have a name`
- `sidebar.team_menu.button.plusIcon` — `Plus Icon` (on the inner `<i>` icon)
- `team_sidebar.join` — `Other teams you can join`
- `navbar_dropdown.create` — `Create a Team`

No new strings strictly required for the minimal fix (the + button's displayName already resolves the two correct strings); but the cleanup may introduce a new `team.button.select.ariaLabel` or reuse the two displayName strings directly.

## Phase B — Runtime evidence (kubectl)

Skipped intentionally: this is a client-side accessibility defect. Pod logs cannot corroborate screen-reader accessible-name computation, which is wholly browser-side. No server-side signal exists; `kubectl` would return no useful data.

## Phase C — Recent commits on affected file

`git log --oneline -- webapp/channels/src/components/team_sidebar/components/team_button.tsx` (most recent first on the pre-fix SHA's ancestry):

- `8cace74692` MM-64486: Remove telemetry (#33606) — removed `trackEvent`, no behavior change to aria-label.
- `54207acd6a` [MM-61631]: Remove focusable child descendants — earlier focus refactor; unrelated to aria-label.
- `fd6a662d76` [MM-62074] Move tooltips with withTooltip to new Tooltip component — unrelated.
- `f80afad1b6` [MM-58441] Create a floating-ui Tooltip — unrelated.

None of the recent commits introduced the defective aria-label flow; the problematic lines 82–124 predate all of these. The bug is long-standing.

## Phase D — Similar past issues/PRs in the fork

Searched fork issues (`gh issue list --repo ShankHarinath/mattermost`). Only the subject issue (#19) matches. Other fork issues (#12 channel-header keyboard selectability, etc.) are unrelated accessibility concerns.

No corroborating fork PR.

(Upstream / community refs from the issue body are logged in `leak_signals` below and were not followed.)

## Phase E — Hypothesis and recommendation

### Root-cause hypothesis

In `webapp/channels/src/components/team_sidebar/components/team_button.tsx`, the `ariaLabel` for the `+` (Create Team / Other teams you can join) button is computed inside an `if (!teamClass)` block that unconditionally reassigns `ariaLabel` to the unread template (`'{teamName} team unread'`) after the teamClass branch resolves, rather than only when `unread` is actually true. The non-draggable render path (line 163) partially hides this by ternary-swapping in `displayName` for create/select-team URLs, but the draggable render path (line 181) has no such guard, and the defective logic still leaks into the accessible name a screen reader computes for the + button (either via JAWS's name-composition that pulls on sibling labels during sequential TAB, or via downstream reuse of the `ariaLabel` variable).

The fix must:
1. Move the `ariaLabel = '{teamName} team unread'` assignment (lines 97–103) INSIDE the `if (unread && !otherProps.isInProduct)` branch (lines 83–92) so it only applies when genuinely unread.
2. Apply the create/select-team guard to the draggable anchor at line 181 the same way as line 163, i.e. `aria-label={isNotCreateTeamButton ? ariaLabel : displayName}`.
3. Optionally — use explicit `displayName` overrides for both + buttons' anchor aria-label (already passed in from `team_sidebar.tsx`), and avoid lowercasing the displayName path for consistency.

### Confidence

**0.84** — code-level root cause is identified, reproducible by inspection, and fix location is surgical (single file, well-bounded). Slight confidence reduction because I cannot bisect the exact text a screen-reader produces without a running browser + JAWS, and the non-draggable path's ternary technically should have suppressed the wrong string for the + button — meaning the reporter's symptom may be JAWS-specific name composition that a simple grep cannot prove. However, regardless of which sub-path emits the wrong string, the unconditional unread-template assignment is clearly defective and the fix is mechanical.

### Recommended next step

Proceed to `/phoenix:plan` against `mattermost` only. Architect should:
- Scope the fix to `team_button.tsx` exclusively (single file, ~10 lines changed).
- Make the unread-template assignment conditional on `unread && !otherProps.isInProduct`, and similarly gate the mentions-template assignment on `mentions` inside the unread branch.
- Add the `isNotCreateTeamButton ? ariaLabel : displayName` guard to the draggable anchor (line 181).
- Add at least one RTL test asserting that, given `url='/create_team'` + `displayName='Create a Team'`, the rendered anchor's `aria-label` is `"Create a Team"` (not containing "unread"). Add a symmetric test for `/select_team`.
- Add an RTL test asserting non-unread team button gets `"{teamName} team"` (not `"{teamName} team unread"`).

## Open questions / blockers

- Unable to directly simulate the JAWS accessible-name computation from the file system. If Architect has access to a browser + screen reader, verifying the final accessible name pre- and post-fix is valuable.
- The lowercasing of `ariaLabel` at line 126 is inconsistent with the displayName branch of the ternary at line 163 (which is not lowercased). This is a minor polish item; decide during plan whether to normalize.

## Leak signals (not used in narrative)

- Upstream JIRA ID referenced in the issue body. Not fetched.
- An upstream PR number appears in our local `git log` subject line as the pre-existing fix. The pre-fix SHA `09fc02f7…` does NOT contain that commit. Not fetched via `gh`; not used to derive the fix.
- Upstream community channel references in the issue body. Ignored.

---

<!--phoenix:scout-summary
affected_repos: [ShankHarinath/mattermost]
confidence: 0.84
recommended_next_step: proceed
primary_file: webapp/channels/src/components/team_sidebar/components/team_button.tsx
primary_symbol: TeamButton
leak_signals: [upstream-JIRA, upstream-PR-in-local-history, upstream-community-channels]
-->
