# Triage Report — Issue #21: DM List not scrolling with Up/Down arrow movement

**Repo:** ShankHarinath/mattermost
**Filed:** 2026-04-21 by ShankHarinath
**Investigator:** Scout (rev 1)
**Classification confidence:** 0.92

## Symptom

When the user opens the **Direct Messages** modal (the `+` next to "Direct Messages" in the channel list, or `Cmd/Ctrl+Shift+K`) and presses Up/Down arrow keys to move the highlighted row, the list does **not** scroll the selected row back into view once it leaves the visible viewport. The arrow keys keep advancing the internal selection, but the visible scroll position stays put, so the user effectively loses track of the cursor. Expected behavior is that the list auto-scrolls to keep the selected row on-screen, like the at-mention autocomplete and emoji picker do.

## Phase A — Code map

The DM modal uses a chain of generic components, all of which live under `webapp/channels/`:

| Layer | File | Role |
|---|---|---|
| Modal shell | `webapp/channels/src/components/more_direct_channels/more_direct_channels.tsx` | Owns the modal, creates `selectedItemRef` (line 85), passes it down. |
| List wrapper | `webapp/channels/src/components/more_direct_channels/list/list.tsx` | Wraps `MultiSelect`, supplies a custom `optionRenderer` that attaches `selectedItemRef` to the currently-selected `ListItem` (lines 47–63). |
| Row | `webapp/channels/src/components/more_direct_channels/list_item/list_item.tsx` | Forwards the ref to the row's outer `<div className='more-modal__row …'>`. |
| Generic multiselect | `webapp/channels/src/components/multiselect/multiselect.tsx` | Threads `selectedItemRef` through to `MultiSelectList` (lines 424, 443). |
| Arrow handler + scroll logic | `webapp/channels/src/components/multiselect/multiselect_list.tsx` | Listens for ArrowUp / ArrowDown on `document` (lines 60, 97–130), updates `state.selected`, and in `componentDidUpdate` (lines 67–89) tries to scroll the selected row into view. |

The relevant scroll block:

```ts
// multiselect_list.tsx, lines 73-89
if (prevState.selected === this.state.selected) {
    return;
}
const selectRef = this.selectedItemRef.current || this.props.selectedItemRef?.current;
if (this.listRef.current && selectRef) {
    const elemTop = selectRef.getBoundingClientRect().top;
    const elemBottom = selectRef.getBoundingClientRect().bottom;
    const listTop = this.listRef.current.getBoundingClientRect().top;
    const listBottom = this.listRef.current.getBoundingClientRect().bottom;
    if (elemBottom > listBottom) {
        selectRef.scrollIntoView(false);
    } else if (elemTop < listTop) {
        selectRef.scrollIntoView(true);
    }
}
```

`listRef` is attached to the inner `<div id='multiSelectList' className='more-modal__options'>` (line 224).

The relevant CSS (`webapp/channels/src/sass/components/_modal.scss`):

```scss
// lines 750-761
.more-modal__list {
    display: flex;
    flex-direction: column;
    > div {
        overflow: auto;
        min-height: 100%;
    }
}

// lines 979-999 (the .filtered-user-list scope used by the DM modal)
.filtered-user-list {
    display: flex;
    height: calc(90vh - 120px);
    flex-direction: column;
    .multi-select__wrapper { display: flex; height: 1px; flex-direction: column; flex-grow: 500; }
    .more-modal__list      { height: 1px; flex-grow: 500; }
}
```

## Phase B — Runtime evidence

This is a pure client-side React/CSS bug. Server logs are not relevant; `kubectl` checks on the cluster would not surface it. Skipped intentionally.

## Phase C — Recent commits on suspect files

`git log` on `multiselect_list.tsx` and `multiselect.tsx`:

- `c943ed6859b` (2023-03-22, "Mono repo -> Master") introduced the current scroll-into-view block (lines 77–88) — `git blame -L 77,90` attributes every line of that block to that single import commit. No subsequent commit has touched the scroll logic itself.
- All subsequent edits in the directory (`b2fbee1608`, `e080f9f5ed`, `b4ee90afca`, `8f0d2b05ea`, …) are i18n / styling refactors, not behavioural.

So the bug has been latent since the monorepo import; nothing recent regressed it. It is a long-standing defect in the original logic.

`git log` on `more_direct_channels/`:
- `ea3be1a9f2` (`MM-61578` — keyboard a11y for Add button), `8b8d0a0a09` (duplicated role dialog fix), `a65ba84697` (filter remote users), `06f59531f5` (DM modal copy improvement) — none touch arrow-scroll logic.

## Phase D — Similar past issues / PRs

Within `ShankHarinath/mattermost`, `gh issue list --search "scroll arrow"` returns only this same issue (#21). `gh pr list --search "multiselect scroll"` returns nothing. Per counterfactual constraints, no upstream cross-repo lookups were attempted.

## Phase E — Root-cause hypothesis

**The scroll never runs because `listRef` is attached to the wrong element.**

`listRef` points to `<div id='multiSelectList' className='more-modal__options'>` — the inner div whose CSS is `overflow: auto; min-height: 100%`. Because `min-height: 100%` (with no `max-height` or `height`) lets the element grow to fit ALL its rendered children, the `more-modal__options` element's box always wraps every row. The actual viewport-clipping is performed by the constrained-height ancestor `.more-modal__list` (`height: 1px; flex-grow: 500;` inside `.filtered-user-list { height: calc(90vh - 120px) }`).

Consequently:

- `this.listRef.current.getBoundingClientRect().bottom` returns the bottom of `.more-modal__options`, which contains every rendered row, so `elemBottom > listBottom` is **always false** for any selected row (the row is by definition inside the list element).
- `elemTop < listTop` is likewise **always false**.
- Both branches at lines 83–87 are dead code; `scrollIntoView()` is never called.

The selected row index updates correctly (the `more-modal__row--selected` class moves), but no scrolling is requested, so the highlight disappears off-screen as the user keeps pressing Down.

**Suggested fix direction** (for Architect — not prescriptive):

Option A — minimal: in `multiselect_list.tsx::componentDidUpdate`, change `listRef` so it is attached to the actual scroll container (one DOM level up — e.g. `.more-modal__list > div` is the wrong layer; the scroller is `.more-modal__list` itself once the constrained-height layout is applied). Concretely, either move the `ref={this.listRef}` from `more-modal__options` (line 224) onto `more-modal__list` (line 215), or compare against `selectRef.offsetParent` / nearest scrollable ancestor.

Option B — robust: drop the manual bounding-rect comparison and call `selectRef.scrollIntoView({block: 'nearest'})` unconditionally on selection change. `block: 'nearest'` is a no-op when the row is already in view, and it correctly walks the scrollable-ancestor chain regardless of which element owns the scrollbar. This matches what `webapp/channels/src/components/admin_console/permission_schemes_settings/permission_system_scheme_settings.tsx:130` already does for similar selection-driven scrolling.

Either approach must be regression-tested against the 14 other consumers of `MultiSelect` (channel selector, channel invite, add users to team/group/role, team selector, etc.) — see Phase blast-radius below. Existing snapshot/unit tests `multiselect_list.test.tsx` only assert `scrollIntoView` is called with `false` / `true`; they pass today because the tests inject a mock `listRef.current.getBoundingClientRect` that returns the visible viewport, masking the production bug. Tests will need updating if Option B is taken.

## Affected repos / blast radius

`mcp__gitnexus__impact MultiSelectList upstream` reports **risk: LOW** at the symbol level (1 direct importer — `multiselect.tsx`), but **14 indirect consumers at d=2**, all in `ShankHarinath/mattermost`:

- `more_direct_channels/list/list.tsx` (the bug-reported entry point)
- `more_direct_channels/more_direct_channels.tsx`
- `team_selector_modal/team_selector_modal.tsx`
- `channel_selector_modal/channel_selector_modal.tsx`
- `channel_invite_modal/channel_invite_modal.tsx`
- `add_users_to_team_modal/add_users_to_team_modal.tsx`
- `add_user_to_group_multiselect/add_user_to_group_multiselect.tsx`
- `add_groups_to_team_modal/add_groups_to_team_modal.tsx`
- `add_groups_to_channel_modal/add_groups_to_channel_modal.tsx`
- `admin_console/system_roles/system_role/add_users_to_role_modal/add_users_to_role_modal.tsx`
- (plus a handful of helper files)

The fix is purely UI behaviour, so all of these benefit (they almost certainly have the same latent bug). Only one repo needs a code change.

## Open questions

1. Should the fix preserve the explicit "align bottom" vs "align top" distinction (current `scrollIntoView(false)` / `scrollIntoView(true)` semantics) or move to `block: 'nearest'`? The visual outcome differs slightly when a row enters view from above vs below.
2. Should arrow-key handling be scoped to the modal/component (e.g. `keydown` on the list root) instead of `document` (current behaviour at line 60)? Out of scope for this triage but worth flagging.
3. The existing tests in `multiselect_list.test.tsx` mock `listRef.current.getBoundingClientRect` such that the assertions pass even with the buggy ref target. Any fix should also fix the test fixtures so they reflect realistic DOM dimensions.

## Confidence

**0.88** — Strong static evidence: the suspect block is small (12 lines), the dead-code condition is mechanically derivable from the CSS layout, the bug is consistent with the user's verbatim description (selection moves but scroll does not), and there are no recent regressions to consider (logic is unchanged since import). Slightly below 0.95 because I did not run the modal in a browser to confirm the bounding-rect values empirically.

## Recommended next step

Hand off to Architect with the two fix options above. Architect should pick one, draft the patch in `multiselect_list.tsx`, update `multiselect_list.test.tsx`, and add an e2e/Playwright check (the `e2e-tests/playwright/support/ui/components/channels/find_channels_modal.ts` helper exists and could be extended to assert visible-row tracking under arrow-key navigation).

---

```yaml
# machine-readable
affected_repos:
  - ShankHarinath/mattermost
confidence: 0.88
recommended_next_step: architect
leak_signals:
  - "Issue body references upstream JIRA MM-22872 and links to mattermost.atlassian.net (mentioned in user-supplied body, not surfaced in triage)."
  - "Issue body links to community.mattermost.com upstream community channels (boilerplate footer, not used in investigation)."
  - "Investigation stayed within the ShankHarinath/mattermost fork; no upstream mattermost/* PR numbers, JIRA IDs, or repos were looked up or cited."
```
