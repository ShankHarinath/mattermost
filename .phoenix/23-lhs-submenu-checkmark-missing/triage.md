## Triage Report — ShankHarinath/mattermost#23

**Classification:** bug  (orchestrator confidence: 0.92)
**Summary:** The LHS category dot-menu Sort submenu (all categories) and Show submenu (DM category) render their `<Menu.Item>` entries without a selected-state `<CheckIcon />`, whereas the visually-analogous channel "Move To" submenu correctly passes the check icon via `trailingElements`.

### Symptom
- Normalized: Submenu items in the LHS sidebar category dot-menu render identically; the user cannot tell which option (sort order / DM show count) is currently active.
- Error signatures: none (UI rendering defect — no exception)
- Reported timeframe: "new LHS menu" rollout — Sort submenu regression; DM Show submenu has been missing the checkmark historically (present pre-7.8.1 per reporter)

### Runtime signal (from kubectl)
- First-seen: n/a — pure client-side UI rendering bug, no server log signal
- Frequency: n/a
- Active now: n/a
- Pod/container state: n/a
- Target(s): n/a (skipped per config; webapp-only defect)
- Affected entities: any user viewing the LHS sidebar in Mattermost Desktop / webapp
- Correlated errors: none

### Affected code (from GitNexus)
- **Repos searched:** mattermost  (rationale: issue explicitly filed against ShankHarinath/mattermost; no cross-repo group membership for the webapp UI component layer)
- **Affected repos (fix required in):** ShankHarinath/mattermost
- `mattermost` · `SidebarCategorySortingMenu` · `webapp/channels/src/components/sidebar/sidebar_category/sidebar_category_sorting_menu.tsx:38-213` — renders both broken submenus (Sort submenu at lines 71-109, Show submenu at lines 132-158); neither branch passes `trailingElements` containing a `<CheckIcon>` on the child `<Menu.Item>` entries and neither compares the item value to the currently-selected preference at item render time.
- `mattermost` · `ChannelMoveToSubMenu` / `createSubmenuItemsForCategoryArray` · `webapp/channels/src/components/channel_move_to_sub_menu/index.tsx:77-118` — reference implementation that works correctly: builds `selectedCategory = <CheckIcon color='var(--button-bg)' size={18}/>` when `currentCategory.display_name === category.display_name` (lines 98-106) and passes it via `trailingElements={selectedCategory}` on each `<Menu.Item>` (line 114).
- Blast radius: LOW — `gitnexus_impact(SidebarCategorySortingMenu, upstream)` reports 0 direct callers, 0 affected processes, 0 modules — the component is rendered declaratively from the sidebar category tree; edits to its internal JSX do not propagate.
- Cross-repo dependencies: none
- Process participation: none detected (GitNexus does not model React render trees as processes)
- Misses: the issue text contains no explicit symbol or file names, so Phase A relied on concept queries ("LHS sidebar category submenu checkmark", "channel menu Move To submenu selected checkmark icon"); both queries converged on the two files above.

### Historical context
**Similar past issues / PRs:**
- No prior issues in `ShankHarinath/mattermost` mention "checkmark" + "submenu" + "sidebar" (searched issues — only 11 open issues exist in the fork, none match).
- `gh search` / upstream lookups are disabled in this environment; cannot scan the wider history.

**Recent commits on affected files:**
- `b244bb621` 2024-07-25 Ben Cooke — "[…] Reduce the number of api requests made to fetch user information for GMs on page load (#27149)"  [flag: touches the sorting menu file but only for data-fetching optimisations; unrelated to menu-item rendering]
- `d13429aa9` — "Enable import order (#24091)"  [flag: mechanical import-ordering change; not a functional edit]
- `e9b3afecc` 2023-08-14 Daniel Espino García — "Mark category as read (#24003)"  [flag: last meaningful functional commit on the sorting menu; predates the `new LHS menu` referenced by reporter. Worth reading to see whether the SubMenu children used to render checkmarks in a previous form.]
- `ChannelMoveToSubMenu/index.tsx` last functional commit `c943ed685` ("Mono repo -> Master (#22553)") — the working reference implementation has been stable.

No single commit in the available fork history cleanly correlates with "new LHS menu shipped" — suggests the LHS redesign landed before the fork was cut or was squashed into a large merge not visible via `git log -- <file>`.

### Root-cause hypothesis
`SidebarCategorySortingMenu` renders two `<Menu.SubMenu>` blocks — the "Sort" submenu (lines 71-109) and the "Show" submenu (lines 132-158). The parent `<Menu.SubMenu>` elements correctly show the current selection on the row that opens the submenu (via `trailingElements={<>{sortDirectMessagesSelectedValue}<ChevronRightIcon/></>}` and the analogous `showMessagesCountSelectedValue`). However, each inner `<Menu.Item>` — one per `CategorySorting` enum value, and one per `Constants.DM_AND_GM_SHOW_COUNTS` entry — is constructed with only `id`, `labels`, and `onClick`. There is no `trailingElements` prop and therefore no `<CheckIcon>` is rendered against the active choice, which is exactly what the user sees ("All options in the submenu appear the same without a checkmark").

The working `ChannelMoveToSubMenu` demonstrates the correct pattern: compute a `selectedCategory` node (`<CheckIcon color='var(--button-bg)' size={18}/>` when the item matches the active value, otherwise `null`) and pass it as `trailingElements` on each `<Menu.Item>`. The fix is to mirror that pattern in both submenus: compare `category.sorting` to `CategorySorting.Alphabetical` / `CategorySorting.Recency` for the Sort items, and compare `dmGmShowCount` to `selectedDmNumber` for the Show items, rendering a `<CheckIcon>` in `trailingElements` when equal.

This also matches the reporter's observation that both problems surface in the same component — a single file-level fix resolves both the "Sort" regression and the historically-broken "Show" submenu.

**Confidence:** 0.90

### Evidence chain
1. GitNexus `query("sidebar category menu Sort submenu checkmark")` → top-ranked file is `sidebar_category_sorting_menu.tsx` containing both `SidebarCategorySortingMenu` and its `handleSortDirectMessages` — confirming the Sort submenu lives in that single file.
2. Read of `sidebar_category_sorting_menu.tsx` (lines 89-108, 150-157) → inner `<Menu.Item>` elements for both Sort and Show submenus have exactly `id`, `labels`, `onClick`, and (for Show) `key` — no `trailingElements`, no `leadingElement`, no comparison of item value to selected state → matches the visual symptom exactly.
3. GitNexus `query("channel menu Move To submenu selected checkmark icon")` → top-ranked render function `ChannelMoveToSubMenu` in `channel_move_to_sub_menu/index.tsx`.
4. Read of `channel_move_to_sub_menu/index.tsx` (lines 98-106, 114) → explicit `<CheckIcon color='var(--button-bg)' size={18}/>` rendered into `trailingElements` when `currentCategory.display_name === category.display_name` → confirms the positive reference pattern the reporter compared against.
5. `gitnexus_impact(SidebarCategorySortingMenu, upstream)` → LOW risk, 0 callers, 0 processes → editing the file is safe and isolated.
6. Reporter's note "at least some of this ticket is an existing issue and not a regression" aligns with the fact that the Show-submenu checkmark path was never implemented in this file — only the parent row's `showMessagesCountSelectedValue` was wired up.

### Recommended next step
proceed to fix

### Open questions
- Was the `gh issue comment` breadcrumb deliberately disabled for this run? The harness denied the single `gh issue comment 23 …` call; the rendered body is at `/tmp/phoenix-issue-comment-23-triage-rev1.md` should the orchestrator wish to post it.
- Styling decision for the implementer: `ChannelMoveToSubMenu` uses `<CheckIcon color='var(--button-bg)' size={18}/>` inside `trailingElements`. The Sort submenu's parent row currently places `{sortDirectMessagesSelectedValue}` (text) + `<ChevronRightIcon/>` in `trailingElements` on the _opener_ row; for the inner items a check icon alone (matching Move To) is the most consistent choice, but confirm with the design owner if a different icon/color is preferred.
- The Sort submenu renders regardless of category type, but `handleSortDirectMessages` is named for DMs and fires telemetry `ui_sidebar_sort_dm_*`. If the same menu shows on non-DM categories, the fix should still be correct (it reads `category.sorting`), but reviewer should confirm the intended categories.

<!-- MACHINE-READABLE FOOTER — DO NOT REMOVE; downstream skills parse this block -->
<!--phoenix:scout-summary
affected_repos: [ShankHarinath/mattermost]
confidence: 0.90
recommended_next_step: proceed
leak_signals: [MM-51091]
-->
