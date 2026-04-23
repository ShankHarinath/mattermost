# Phoenix Triage — DM List does not scroll when selected row moves off-screen

Source issue: [ShankHarinath/mattermost#21](https://github.com/ShankHarinath/mattermost/issues/21) — "DM List not scrolling with Up/Down arrow movement"
Reporter: ShankHarinath — Created 2026-04-21 — Labels: none
Home repo: `ShankHarinath/mattermost`
Triage rev: 1

## Summary

The Direct Messages modal (opened with `cmd/ctrl+shift+k`) fails to scroll its result list when the selection moves past the visible viewport. The codebase already ships scroll-into-view logic for this case, but in the DM modal that logic is wired through a `selectedItemRef` that never gets a DOM node — so the branch never fires. Every other modal that uses the same multiselect infrastructure (`channel_invite_modal`, `add_users_to_team_modal`, `team_selector_modal`, `channel_selector_modal`) works correctly because they attach `selectedItemRef` directly to a plain DOM `<div>`. The DM modal is the only caller that routes the ref through a `react-redux` `connect()`-wrapped `ListItem` that is missing the `{forwardRef: true}` option. Result: `connect` silently swallows the ref, `selectedItemRef.current` stays `null`, and the scroll branch in `MultiSelectList.componentDidUpdate` is skipped on every arrow press.

## Dataflow diagram

```
User ↓/↑ keypress (document)
        │
        ▼
MultiSelectList.handleArrowPress                            [multiselect_list.tsx:97-130]
  - updates `state.selected` index
  - calls props.onSelect(options[selected])
        │
        ▼  (React re-render)
List (DM modal).renderOptionValue                           [more_direct_channels/list/list.tsx:47-63]
  - renders <ListItem ref={isSelected ? props.selectedItemRef : undefined} …>
        │
        ▼
ListItem (default export = connect-wrapped)                 [more_direct_channels/list_item/index.ts:18]  ← DEFECT SITE
  - connect(mapStateToProps)(ListItem)   // NO {forwardRef: true}
  - ref arrives at HOC, NOT forwarded to the inner React.forwardRef component
        │                                                                   [latent: every arrow press]
        ▼
Inner ListItem(forwardRef)                                  [list_item/list_item.tsx:39,63]
  - never receives the ref; <div ref={ref}> is ref=undefined
        │
        ▼  (commit phase completes; componentDidUpdate fires in MultiSelectList)
MultiSelectList.componentDidUpdate                          [multiselect_list.tsx:77-88]
  - const selectRef = this.selectedItemRef.current          // null (internal ref unused in DM flow)
                    || this.props.selectedItemRef?.current  // null ← SYMPTOM SITE
  - condition `this.listRef.current && selectRef` is FALSE
  - scrollIntoView() never called
        │
        ▼
Result: `.more-modal__options` scroll position unchanged   [CSS: _modal.scss:750-761, overflow: auto]
        → selected row can sit outside the scroll viewport indefinitely
```

Unread hops: none. Every hop is either a concrete file/line above or a React-internal commit.

## Cause-flow / state-per-layer

| Layer | What should happen | What actually happens |
|---|---|---|
| Keyboard (document) | Arrow key fires `handleArrowPress`. | ✓ fires — `multiselect_list.tsx:97`. |
| MultiSelectList state | `state.selected` advances; `onSelect` invoked. | ✓ advances — `multiselect_list.tsx:127-129`. |
| DM `List.renderOptionValue` | Pass `props.selectedItemRef` to the selected row. | ✓ prop passed — `list.tsx:55`. |
| `<ListItem ref=…>` (default export) | Redux-wrapped HOC forwards ref to inner `React.forwardRef` component. | ✗ **← bug:** `connect()` is called without `{forwardRef: true}` at `list_item/index.ts:18`. The ref is attached to the HOC wrapper (a memoized function component) and is silently dropped. |
| Inner `ListItem(forwardRef)` | Attach ref to root `<div>`. | ✗ **← latent:** inner `ref` is `undefined`, so `<div ref={ref}>` at `list_item.tsx:63` never gets a DOM handle. |
| `props.selectedItemRef.current` (held by `more_direct_channels`) | Point at the selected row's DOM div. | ✗ stays `null` every render. |
| `MultiSelectList.componentDidUpdate` scroll branch | `selectRef` truthy → `scrollIntoView(true|false)`. | ✗ **← symptom:** `selectRef = null || null`, scroll branch skipped — `multiselect_list.tsx:77-88`. |
| `.more-modal__options` scroll position | Advances so selected row is in view. | ✗ unchanged; user sees selection move out of the visible area. |

The `← bug:` marker on the connect layer matches the "defect site" marker of the dataflow diagram above.

## Root-cause hypothesis (primary)

- **id:** H1
- **rank:** 1
- **primary_causal_symbol:** `Connect(ListItem)` at `webapp/channels/src/components/more_direct_channels/list_item/index.ts:18`
- **causal narrative:** The DM modal's `List` component attaches `selectedItemRef` via `<ListItem ref={…}>`. `ListItem`'s default export is a `react-redux` `connect()`-wrapped HOC that is missing `{forwardRef: true}`. `connect` by default does not forward refs — the ref binds to an inner wrapper component, and `selectedItemRef.current` never receives the DOM node. The scroll-into-view branch in `MultiSelectList.componentDidUpdate` (`multiselect_list.tsx:77-88`) then short-circuits on a null `selectRef` and silently skips `scrollIntoView()`. This is unique to the DM modal because every other consumer of `selectedItemRef` in the codebase attaches the ref straight to a DOM `<div>` in the optionRenderer.
- **falsifiable trace:** With the DM modal open and `ListItem/index.ts` unmodified, setting a breakpoint on `multiselect_list.tsx:77` or logging `this.props.selectedItemRef.current` after each arrow press will show `null`. Changing line 18 to `connect(mapStateToProps, null, null, {forwardRef: true})(ListItem)` will make the ref resolve to the selected row's `<div>` and the existing scroll branch will fire.
- **expected outcome if true:** Adding `{forwardRef: true}` restores scroll-into-view with no other code changes and no test changes (existing `multiselect_list.test.tsx:43-109` already covers both top and bottom alignment paths and keeps passing).
- **root_cause_confidence:** 0.92
- **fix_direction_confidence:** 0.90

## Hypotheses considered

| id | Claim | Status | Rationale |
|---|---|---|---|
| H1 | `connect()` on `list_item/index.ts:18` drops the ref because `{forwardRef: true}` is absent. | **confirmed** | Direct inspection of the code; contrast against 4 other modals that pass `selectedItemRef` directly to a DOM div; existing tests use an object-literal mock ref and therefore don't exercise the real `connect`+`forwardRef` chain (gap explains why tests pass). |
| H2 | `MultiSelectList`'s `scrollIntoView(bool)` conflicts with nested scroll containers (modal-body vs `more-modal__options`). | refuted | CSS analysis: `.more-modal__list > div { overflow: auto; min-height: 100% }` (`_modal.scss:750-761`) and `.more-modal__list { height: 1px; flex-grow: 500 }` (`_modal.scss:996-999`) bound the options div to a fixed viewport. The modal body itself is `overflow-x: hidden` with default `overflow-y: visible`, so `.more-modal__options` is the nearest scrollable ancestor. `scrollIntoView` would work correctly if it were ever called — but it never is (see H1). |
| H3 | Arrow key handler doesn't fire (key intercepted by ReactSelect). | refuted | Reporter explicitly states the selection row does move up/down; only the scroll doesn't follow. `handleArrowPress` is on `document` and fires via capturing; `e.preventDefault()` at `multiselect_list.tsx:127` confirms the path runs. |
| H4 | The selected class (`more-modal__row--selected`) is applied but `isSelected` never flips in `renderOptionValue` (stale closure from `useCallback`). | refuted | `renderOptionValue`'s `useCallback` deps include `props.selectedItemRef`, whose object identity is stable. The renderer is re-invoked on each state change because `MultiSelectList` re-renders on `setState({selected})`. Visual selection highlight IS working per the reporter. |

Confidence gate for E2: top hypothesis ≥ 0.85 ✓, all signals mapped to code ✓, Phase B not applicable (client-only UI bug — no server log signal expected). E2 skipped; straight to E4/E5.

## Devil's-advocate pass

The reporter did NOT claim any of the following, but they are worth ruling out:

1. **Hidden height regression on `.more-modal__options`.** Could the scroll container have zero/infinite height so that `elemBottom > listBottom` is tautologically false or true? Inspection of `_modal.scss:750-761` + `_modal.scss:979-999` shows bounded height via `.filtered-user-list { height: calc(90vh - 120px) }` and `flex-grow: 500` on the inner list. Non-issue under normal resolutions.
2. **Event handler ordering with ReactSelect.** ReactSelect's native input has its own arrow-key handlers. If `MultiSelectList.handleArrowPress` stops running, selection would freeze. Reporter confirms selection moves — so the document-level listener fires reliably.
3. **A mobile-only manifestation.** `list.tsx` does not pass `isMobileView` to `ListItem` (even though `Props.isMobileView` is required — a type-level gap), meaning the mobile-only `lastPostAt` column is never rendered. This is a separate latent bug with no impact on scroll; call it out only for tracking.

None of these are comparable in confidence to H1.

## Scope limits (cases the proposed fix does NOT handle)

1. **Jump-by-page inside the DM modal.** The `nextPage` / `prevPage` buttons at `multiselect.tsx:140-163` call `this.listRef.current.setSelected(0)` — they reset selection to the first row of the new page, so scroll-into-view is not required there; but if a keyboard "Page Down" equivalent is ever added, the same chain must be audited.
2. **Switch-channel modal (`cmd/ctrl+k`).** That modal uses `SwitchChannelProvider` + `SuggestionList` (a completely different scroll implementation at `suggestion_list.tsx:144-174`). Not addressed by this fix, and the reporter did not mention it.
3. **Hover/mouse-only selection changes.** `select` via `onMouseEnter` sets `state.selected` too; after the fix this will also trigger scroll-into-view. Intended behavior, but worth a visual regression pass because other modals already tolerate it.
4. **The pre-existing `isMobileView` prop wiring gap in `list.tsx:54-62`.** Not part of the reported bug; if not fixed here it remains latent.

## Maintainer-review self-pass

1. *"Why not fix this in `list.tsx` by changing the callsite instead of in `list_item/index.ts`?"* — Because the HOC's missing `{forwardRef: true}` is the root cause and the codebase already uses that option in ~20 other places (grep turned up `user_groups_modal`, `file_upload`, `textbox`, `switch_channel_provider`, etc.). Fixing it at the HOC matches the prevailing convention, keeps the callsite idiomatic (`<ListItem ref={…}>`), and requires no changes in `list.tsx`.
2. *"Does the change break any test?"* — `multiselect_list.test.tsx:20-28` builds a plain object literal as the ref mock and calls `scrollIntoView` on it directly. It does not exercise the real `connect`+`forwardRef` chain, so adding `{forwardRef: true}` does not break it. No snapshot covers this path. Recommend adding a small integration test that mounts `MoreDirectChannels` (not shallow) and asserts that after a ↓ keypress the scroll container's `scrollTop` has changed — this is a real gap the current test suite has.
3. *"Is there any reason NOT to add `{forwardRef: true}` here?"* — None. `connect()`'s only observable difference is that the HOC becomes ref-forwardable. It has no runtime cost beyond a `forwardRef` wrapper and does not affect the inner `mapStateToProps` behavior.

## Pragmatism axis — fix shapes

| Variant | Change | LOC | Files | Notes |
|---|---|---|---|---|
| **Minimal** | `list_item/index.ts:18` → `connect(mapStateToProps, null, null, {forwardRef: true})(ListItem)` | 1 | 1 | Single-line. Restores the ref chain the `MultiSelectList` scroll branch already assumes. No test updates required to keep existing suite green. |
| **Idiomatic (recommended)** | Same as Minimal, plus add a jest test under `more_direct_channels` that mounts the modal, simulates ↓ presses past the visible viewport, and asserts `.more-modal__options` `scrollTop` changed. | ~30 | 2 | Matches how `channel_invite_modal` is exercised; fills the test gap H1's refutation of existing tests exposed. |
| **Architectural** | Refactor `multiselect_list.tsx` scroll logic to stop depending on an external `selectedItemRef` — compute the selected DOM node from `listRef.current.children[this.state.selected]` and scroll that, or use `element.scrollIntoView({block: 'nearest'})`. Removes the `selectedItemRef` prop from ~8 consumer components. | ~80 | 8 | Lower coupling and eliminates the class of bug where a caller forgets to wire `selectedItemRef` (or wires it through an HOC without `{forwardRef: true}`). Higher blast radius; save for a follow-up if it keeps biting. |

`codebase_impact` on `ListItem` reports risk **LOW** (19 indirect imports, 0 processes affected). Minimal is acceptable — no refactor mandate.

**Recommended:** Idiomatic.

## Affected code

| File | Line(s) | Role |
|---|---|---|
| `webapp/channels/src/components/more_direct_channels/list_item/index.ts` | 18 | **Defect site.** Add `{forwardRef: true}` as the 4th arg to `connect`. |
| `webapp/channels/src/components/more_direct_channels/list_item/list_item.tsx` | 39-93 | Consumer of the forwarded ref; no code change needed, already uses `React.forwardRef` correctly. |
| `webapp/channels/src/components/more_direct_channels/list/list.tsx` | 47-63 | Uses `<ListItem ref={isSelected ? props.selectedItemRef : undefined}>`; behavior correct once the HOC forwards refs. |
| `webapp/channels/src/components/more_direct_channels/more_direct_channels.tsx` | 79-85, 267-285 | Creates and threads `selectedItemRef`; no change. |
| `webapp/channels/src/components/multiselect/multiselect.tsx` | 424, 443 | Threads the prop to `MultiSelectList`; no change. |
| `webapp/channels/src/components/multiselect/multiselect_list.tsx` | 67-89 | Scroll-into-view logic; no change — already correct and will fire once `selectRef` becomes non-null. |
| `webapp/channels/src/components/multiselect/multiselect_list.test.tsx` | (new assertions recommended) | Existing tests don't catch this class — see Idiomatic variant. |

## Reference material — recent history

- `webapp/channels/src/components/multiselect/multiselect_list.tsx` — last touched by `d13429aa92` (import-order refactor) and `3f022e728f` (utils split). Scroll logic unchanged since the 2023 monorepo merge (`c943ed6859`).
- `webapp/channels/src/components/more_direct_channels/list_item/list_item.tsx` — last touched by `ea3be1a9f2` ([MM-61578] add-button accessibility). Did not alter the `React.forwardRef` structure.
- `webapp/channels/src/components/more_direct_channels/list/list.tsx` — last touched by `06f59531f5` ([MM-57319] DM modal message improvement). Did not alter the ref flow.
- `webapp/channels/src/components/more_direct_channels/list_item/index.ts` — **never modified after the monorepo merge**. The missing `{forwardRef: true}` has been latent since 2023-03-22 (`c943ed6859`).
- Precedent fixes in the codebase for the same class of bug: `connect(makeMapStateToProps, mapDispatchToProps, null, {forwardRef: true})` in `textbox/index.ts:64`, `file_upload/index.ts:47`, `user_groups_modal/index.ts:65`, `switch_channel_provider.tsx:336`, `suggestion_box/index.js:18`, `search_date_suggestion/index.ts:27`, `search_channel_suggestion/index.ts:28`.

## Runtime signal (Phase B)

Not applicable. This is a pure client-side UI bug in webapp React code. There is no server-side trace and `logs.search` would return no relevant signal. The reporter's deterministic repro (open DM modal, press ↓ past the visible viewport) is sufficient.

## Open questions

- Does Mattermost accept `{forwardRef: true}` as the minimal patch, or does the maintainer prefer the architectural refactor (remove `selectedItemRef` from the `MultiSelect*` API surface entirely)? The codebase uses `{forwardRef: true}` extensively today, so the minimal patch is idiomatic.
- Should the follow-up jest test also cover the hover/mouseenter path (`select` from `onMouseEnter` → `setSelected` → scroll)? Current tests don't cover that either.

## Recommended next step

Apply the Idiomatic fix shape: add `{forwardRef: true}` at `webapp/channels/src/components/more_direct_channels/list_item/index.ts:18`, and add a mount-level jest test under `webapp/channels/src/components/more_direct_channels/` that asserts `.more-modal__options.scrollTop` changes after simulated ↓ keypresses push the selection past the viewport.

---

```yaml
# machine-readable footer
source_issue: 21
home_repo: ShankHarinath/mattermost
affected_repos:
  - ShankHarinath/mattermost
confidence: 0.92
root_cause_confidence: 0.92
fix_direction_confidence: 0.90
primary_causal_symbol: "Connect(ListItem) @ webapp/channels/src/components/more_direct_channels/list_item/index.ts:18"
fix_shape_recommended: idiomatic
hypotheses_count: 4
hypotheses_confirmed: 1
converged: true
recommended_next_step: build
rev: 1
```
