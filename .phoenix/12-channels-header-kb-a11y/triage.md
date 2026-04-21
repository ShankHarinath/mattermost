## Triage Report — ShankHarinath/mattermost#12

**Classification:** bug  (orchestrator confidence: 0.9)
**Summary:** The "Channels" product-branding button in the global header is focusable with TAB but has no keyboard (ENTER/SPACE) activation handler, so pressing those keys is a silent no-op — the product switcher menu only opens on mouse click.

### Symptom
- Normalized: Keyboard user TABs to the global-header "Channels" button, presses ENTER or SPACE, nothing happens; mouse click works.
- Error signatures: none (silent no-op — no console error, no stack trace)
- Reported timeframe: unspecified (the upstream bug was filed 2024, same symptom persists at this fork's HEAD)

### Runtime signal (from kubectl)
- First-seen: not applicable — this is a client-side React/DOM UI bug, no server-side symptom
- Frequency: not applicable (no log lines are emitted for a failed keyboard interaction)
- Active now: yes (bug is present at HEAD — fix absent in source)
- Pod/container state: not applicable
- Target(s): none queried — no server component is in the causal path
- Affected entities: any keyboard-only user, screen-reader user, or a11y-tester
- Correlated errors: none

Note: I intentionally did not run `kubectl logs`; a browser-side accessibility defect produces no telemetry server-side. Recording this explicitly instead of fabricating a runtime signal.

### Affected code (from GitNexus)
- **Repos searched:** mattermost  (rationale: issue explicitly filed against `ShankHarinath/mattermost`; no cross-service / contract boundary — this is a pure webapp UI bug)
- **Affected repos (fix required in):** mattermost
- `mattermost` · `ProductMenuButton` (styled `IconButton`) · `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/global_header/left_controls/product_menu/product_menu.tsx:44-60` — defines the visible icon-button; its `onClick: () => {}` is an intentional no-op (the comment at lines 50-51 calls this a "known issue" with the IconButton component), so ENTER/SPACE on the button fires the no-op and swallows the activation.
- `mattermost` · `ProductMenuContainer` (styled `<nav>`) · same file:34-42, rendered at 111 with `onClick={handleClick}` — the actual `dispatch(setProductMenuSwitcherOpen(!switcherOpen))` lives here (file:71). Because the handler is on the container, only bubbled mouse-clicks toggle the menu; a keyboard ENTER/SPACE fired on the inner `<button>` hits the no-op and doesn't reach the container, producing the silent failure.
- `mattermost` · `ProductBranding` · `webapp/channels/src/components/global_header/left_controls/product_menu/product_branding/product_branding.tsx:21-38` — has `tabIndex={0}` on the container `<div>` (line 27), so it is in the tab order, but no `role="button"`, `onKeyDown`, or activation handler. This is the "Channels" text/icon the reporter tabs to.
- `mattermost` · `ProductBrandingTeamEdition` · `webapp/channels/src/components/global_header/left_controls/product_menu/product_branding_team_edition/product_branding_team_edition.tsx:41-52` — identical shape (`tabIndex={0}` only, no key handler); same bug on free-edition instances.
- Blast radius: scope is confined to the product-menu subtree under `webapp/channels/src/components/global_header/left_controls/product_menu/` (6 files total per the reference PR; one snapshot file and one trivial test assertion). No shared utility or Redux action changes required — `setProductMenuSwitcherOpen` already exists and works correctly; the fix is at the DOM/event-handler layer only.
- Cross-repo dependencies: none (pure webapp change)
- Process participation: none detected (GitNexus `query` returned zero processes for these keywords — the interaction is short, synchronous dispatch, not a multi-step indexed flow; also the `impact` tool hit a corrupted-WAL error on this index, so blast-radius numbers were not computable from the graph. Code reading is authoritative here.)
- Misses: GitNexus `query` for "Channels button global header product switcher keyboard accessibility" and "product switcher menu toggle open" both returned zero processes — interpretation: index either lacks embeddings for UI/DOM-level React components or the natural-language queries didn't match. `grep` + `Read` recovered the code definitively. `impact` tool errored (`Corrupted wal file`), unrelated to the bug.

### Historical context
**Similar past issues / PRs:**
- #27068 — upstream `mattermost/mattermost` issue, **identical title and body** ("Accessibility: 'Channels' button in global header is not selectable with keyboard", same JIRA MM-55270). CLOSED 2025-01-23. This is the same bug being reported against the fork.
- #29224 — upstream PR **`[MM-55270] fix(accessibility): "channels" button in global header is not selectable with keyboard`**, MERGED 2025-01-23 17:27 UTC. Touched exactly these files: `product_menu.tsx` (+36/-18), `product_branding.tsx` (+21/-5), `product_branding_team_edition.tsx` (+3/-5), plus `product_menu.test.tsx` (+1/-1) and two snapshots. PR release note: *"Added the ability to toggle the switcher menu in the global header using the SPACE and ENTER keys while the product branding is in focus."*
- #27069 — sibling a11y issue (focus restoration on settings modal close). Same category but unrelated fix.

**Recent commits on affected files:**
- `d13429aa92` 2023 — "Enable import order" (formatting only) — not causal.
- `c3e69e97e4` 2023 — compass-components → compass-icons icon migration — not causal.
- `b7175560da` 2022 — ESLint rule for compass-components — not causal.
- `c943ed6859` 2022 — "Mono repo -> Master" — initial import.
- **No commit at this fork's HEAD (`d345e92136`, 2025-01-23 18:01:18 +0100) contains the MM-55270 fix.** `git log --grep="MM-55270"` returns nothing. HEAD is timestamped ~30 min *after* upstream PR #29224 merged (17:27 UTC), so the fork branched from a commit that did not yet contain the fix. The bug is therefore present and un-fixed at HEAD.

### Root-cause hypothesis
`ProductMenuButton` (styled `IconButton` in `product_menu.tsx:44-60`) receives a no-op `onClick: () => {}` (a known limitation called out in the source comment — the third-party IconButton treats a missing onClick as "disabled" and so a placeholder is required). The real toggle handler lives on the ancestor `ProductMenuContainer` (line 71, `handleClick` dispatches `setProductMenuSwitcherOpen`). Mouse clicks bubble from the inner `<button>` up to the `<nav>` and invoke `handleClick`. But when a keyboard user activates the button with ENTER/SPACE, the browser dispatches a `click` event on the inner button element where the no-op handler runs and nothing else is wired — there is no `onKeyDown` on either `ProductMenuButton` or on the `ProductBranding`/`ProductBrandingTeamEdition` siblings which also carry `tabIndex={0}` but lack keyboard handlers. The fix pattern is established by upstream PR #29224: make the branding container a proper interactive element (add `role="button"`, `onKeyDown` for ENTER/SPACE, call `handleClick`/toggle dispatch; and/or move the click+key handling off the no-op IconButton onto a real `<button>`), mirroring the six-file change set in that PR.

**Confidence:** 0.92

### Evidence chain
1. Issue title is verbatim-identical to upstream `mattermost/mattermost#27068`, and the body reproduces the same reproduction steps and JIRA ticket (MM-55270) → this is the same bug, already diagnosed and fixed upstream by the community.
2. Upstream PR #29224 (MERGED 2025-01-23 17:27 UTC) is titled "[MM-55270] fix(accessibility): 'channels' button in global header is not selectable with keyboard" and modifies `product_branding.tsx`, `product_branding_team_edition.tsx`, and `product_menu.tsx` → confirms the causal files.
3. Fork HEAD (`d345e92136ef`, 2025-01-23 18:01 local / after 17:27 UTC merge) does NOT include the MM-55270 fix — `git log --grep="MM-55270"` returns empty, and inspection of `product_branding.tsx` shows only `tabIndex={0}` with no key handler, while `product_menu.tsx` still has the `onClick: () => {}` no-op on the IconButton → bug is present at HEAD.
4. Reading `product_menu.tsx` shows `onClick={handleClick}` is on `ProductMenuContainer` (line 111), not on the button; the button's own `onClick` (line 52) is an explicit no-op. Reading `product_branding.tsx` shows `tabIndex={0}` without `role`, `onKeyDown`, or handler → explains why the container's click fires for mouse but not for keyboard-synthesized clicks and why the sibling focusable "Channels" text node also doesn't respond to keys.
5. Test file `product_menu.test.tsx:161` uses `wrapper.find(ProductMenuContainer).simulate('click')` (not the button) → corroborates that production code put the handler on the container by design.

### Recommended next step
proceed to fix

### Open questions
- Should the fix mirror upstream PR #29224 exactly (preferred: minimizes drift and matches the QA steps published there), or should the fork take a different a11y approach (e.g., lift click+keydown onto a real `<button role="button">` wrapping both icon and text, eliminating the split-handler pattern entirely)?
- Upstream PR mentions a linked ticket requiring focus to move to the first menu item after open — the current fork issue body references "(see linked ticket)" but the linked ticket is not supplied. If it is #27069 (the sibling a11y focus issue), that is a second, separate change; if it must land together with this one, clarify scope.
- GitNexus `impact` tool returned `Corrupted wal file. Read out invalid WAL record type.` for `ProductMenuButton` — the graph index appears partially corrupted for this repo. Not a blocker for this triage (code reading was sufficient), but it should be reported for reindex.

<!-- MACHINE-READABLE FOOTER — DO NOT REMOVE; downstream skills parse this block -->
<!--phoenix:scout-summary
affected_repos: [mattermost]
confidence: 0.92
recommended_next_step: proceed
-->
