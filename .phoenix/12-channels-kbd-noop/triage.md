# Phoenix Triage Report — ShankHarinath/mattermost#12

**Issue:** Accessibility: "Channels" button in global header is not selectable with keyboard
**Reporter:** ShankHarinath • **Classification confidence (intake):** 0.92
**Triage rev:** 1
**Scout confidence:** 0.88

## Summary

The product-switcher trigger in the global header (the element shown as the "Channels" icon + heading) is rendered as a styled `<div>` with `tabIndex={0}` and the toggle behaviour lives on the parent `MenuWrapper`'s `onClick` only. Because there is no `onKeyDown` / `onKeyUp` on the trigger and `MenuWrapper` itself listens only for mouse clicks (and key events only once the menu is already open — to close it), pressing ENTER or SPACE while the trigger is focused does nothing. The symptom reported matches this code exactly.

## Phase A — Code mapping

**Repo scope decision:**
- GitNexus `list_repos` shows `mattermost` indexed at SHA `d345e92136…` (matches the pre-fix SHA in the dispatch). No group membership relevant for a frontend a11y bug.
- `group_list` / `group_contracts` are not applicable — this is a pure frontend defect, no cross-service contract involvement.
- **Searched set:** `mattermost` only.
- **Affected set:** `mattermost` only.

**GitNexus queries:** `query("product switcher global header Channels button")` and `query("ProductBranding click handler product menu toggle")` both returned empty for this repo (the React/TSX surface appears under-indexed in the process graph for this area). Fell back to Glob + Read over `webapp/channels/src/components/global_header/**`, which resolved the layout cleanly.

**Prime suspects (files are absolute paths):**

1. `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/global_header/left_controls/product_menu/product_menu.tsx`
   - `ProductMenu` composes `MenuWrapper` > `ProductMenuContainer` > (`ProductMenuButton` + `<ProductBranding/>` or `<ProductBrandingTeamEdition/>`).
   - The click-to-toggle wiring is on `ProductMenuContainer` (`onClick={handleClick}`, line 111) and indirectly via `MenuWrapper.toggle` (mouse-only).
   - `handleClick` (line 71) dispatches `setProductMenuSwitcherOpen(!switcherOpen)` — this is the action that needs to fire on keyboard activation.

2. `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/global_header/left_controls/product_menu/product_branding/product_branding.tsx`
   - The licensed variant of the "Channels" branding element users tab to. Rendered as `ProductBrandingContainer` — a styled `<div>` — with `tabIndex={0}` (line 27) but **no** `onClick`, `onKeyDown`, `onKeyUp`, or `role`. Contains the `<h1>` with text `"Channels"` (line 34).

3. `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/global_header/left_controls/product_menu/product_branding_team_edition/product_branding_team_edition.tsx`
   - The free-edition variant of the same element (line 43). Same anti-pattern: styled `<div>` with `tabIndex={0}`, no key handling, no `role`. The reporter's repro would hit this one or the licensed one depending on license state — both exhibit the same bug.

4. `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/widgets/menu/menu_wrapper.tsx`
   - `MenuWrapper` (marked `@deprecated` — line 39) uses `onClick={this.toggle}` on its root `<div>` (line 153). Its `keyboardClose` handler (lines 98–106) only reacts to ESCAPE/TAB to **close** the menu once open. There is no key-based toggle to **open** it.

**Why this is the prime cause (and not the `ProductMenuButton`):** `ProductMenuButton` is a real `IconButton` (compass-components, line 44) and has `aria-label="Product switch menu"` + `aria-expanded`. An IconButton normally is keyboard-activatable, but at the moment the button's own `onClick` is a no-op stub (`onClick: () => {}`, line 52) because the wiring was intentionally moved up to `ProductMenuContainer`. When a keyboard user focuses the button, the IconButton synthesises a click event; that click bubbles up to `ProductMenuContainer`/`MenuWrapper` and fires `handleClick` — **so the small round icon button actually does work via keyboard**. What does NOT work is the second focusable element next to it: the `ProductBranding` / `ProductBrandingTeamEdition` container which the user tabs to after the IconButton. That is the "Channels" text+icon the reporter describes ("TAB to the 'Channels' icon/text button"). Its synthetic activation never fires `click` because it is a `<div>`, not a button, and it has no key handler of its own.

**Blast radius (`mcp__gitnexus__impact` proxy via Grep):** the two branding components are only imported by `product_menu.tsx`. Adding a key handler or promoting the outer element to `<button role="button">` is a local change with no downstream risk.

## Phase B — Runtime evidence (kubectl)

Skipped. This is a purely client-side UX/accessibility defect reproducible only in a browser with a keyboard; it produces no server-side log signature and the kind-canonix cluster would not emit anything relevant. Recorded as an intentional skip rather than a missing signal.

## Phase C — Recent commits on affected files

`git log` on the three prime-suspect files on the current branch shows the last touches are ancient relative to this regression report:

- `product_branding.tsx` / `product_branding_team_edition.tsx`: last non-trivial change `708d046c1b "Adding more visibility to the free edition"` (PR #26698) and `4136343476 "Fixathon: Web app dependency updates part 1"` (PR #29036). Neither altered the `<div tabIndex={0}>` trigger pattern.
- `menu_wrapper.tsx`: last change `241001f969 "MM-50384 : Mark all other menus deprecated"` — only added the `@deprecated` JSDoc; behaviour unchanged.

There is no recent commit that plausibly *introduced* this bug — it is a long-standing accessibility defect in a deprecated menu primitive plus an originally-mouse-only trigger element.

## Phase D — Similar past issues / PRs

`gh issue list` on the fork (`ShankHarinath/mattermost`) returns only issue #12 itself — the fork has no prior triage history. Working purely from local code + the fork issue, there is no "fix pattern" from a prior merged PR to lift; the fix must be derived from the symptom.

## Phase E — Root-cause hypothesis & recommended fix direction

**Hypothesis (confidence 0.88):** The "Channels" element the issue refers to is the `ProductBranding` (or `ProductBrandingTeamEdition`) container next to the small products IconButton in the global header's left controls. It is a `<div tabIndex={0}>` with no `role="button"` and no key handlers, and the surrounding `MenuWrapper` toggles only on `onClick` (mouse). Therefore focusing it with TAB and pressing ENTER/SPACE produces no event that reaches `handleClick` in `product_menu.tsx`, so `setProductMenuSwitcherOpen` is never dispatched. Mouse click works because the click bubbles to `ProductMenuContainer` / `MenuWrapper`.

**Recommended fix direction (Architect to decide exact shape):**
- Preferred: promote `ProductBrandingContainer` and `ProductBrandingTeamEditionContainer` from styled `<div>` to a real `<button>` (or `role="button"` with `onKeyDown` that activates on `Enter`/`Space`), wire the same `handleClick` as the container, and ensure focus then moves to the first item in the opened menu (per the issue's "Expected" bullet). Add the corresponding ARIA (`aria-haspopup="menu"`, `aria-expanded`, `aria-controls='product-switcher-menu'` already exists on the sibling IconButton — reuse the pattern).
- Alternative (narrower): keep the `<div>` but attach an `onKeyDown` that calls the toggle on `Enter` / `Space` using the existing `isKeyPressed` helper at `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/utils/keyboard.ts` and `Constants.KeyCodes.ENTER` / `Constants.KeyCodes.SPACE`. This is minimally invasive but does not fully satisfy WAI-ARIA semantics.
- Focus management: after the menu opens, move focus to the first `ProductMenuItem` (the `'/'` Channels entry rendered at `product_menu.tsx:127`). This matches the issue's "Expected" behaviour.

Test coverage to add/update:
- `product_branding.test.tsx` and `product_menu.test.tsx` should assert keyboard activation (`fireEvent.keyDown` with `Enter` and ` ` / `Space`) dispatches `setProductMenuSwitcherOpen(true)`.

## Files to touch

- `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/global_header/left_controls/product_menu/product_branding/product_branding.tsx`
- `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/global_header/left_controls/product_menu/product_branding_team_edition/product_branding_team_edition.tsx`
- Likely: `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/global_header/left_controls/product_menu/product_menu.tsx` (if keyboard handler is lifted to `ProductMenuContainer` rather than added per-branding, or if focus-on-open is wired here).
- Tests adjacent to the above.

## Open questions for Architect

1. Should the fix replace the `<div tabIndex={0}>` pattern with a real `<button>` in both the licensed and team-edition variants, or keep the DOM structure and bolt on `onKeyDown`? The former is the "right" a11y answer but requires a style audit (focus ring, default button styles, heading inside button is acceptable but unusual).
2. Focus-into-first-menu-item on open: the `Menu` / `MenuWrapper` pair is marked `@deprecated` ("Use the 'webapp/channels/src/components/menu' instead"). Is Architect in scope to swap to the non-deprecated menu here, or should the fix stay inside the deprecated primitive?
3. The small `ProductMenuButton` IconButton next to the branding element is *also* nominally keyboard-activatable but its `onClick` is a documented no-op (`// TODO@UI: remove the onClick`) and it depends on click-bubbling through `ProductMenuContainer`. Worth verifying (with manual a11y testing) that TAB lands first on the IconButton and that it still opens the menu via keyboard — if not, both focusable elements need the fix.

## Blockers encountered

- GitNexus `query()` returned zero processes/symbols for the two search terms tried on the `mattermost` repo, despite the index being fresh (last indexed 2026-04-21). Front-end TSX coverage in the process graph looks thin. Investigation proceeded via Glob + Read without issue. Not a blocker for the fix — just lower-confidence "automated" evidence; the manual reading of the three files is conclusive.
- No `## Issue comment` block in the dispatch payload → issue-comment breadcrumb skipped per orchestrator policy.

---

```yaml
# machine-readable footer
affected_repos:
  - ShankHarinath/mattermost
confidence: 0.88
recommended_next_step: plan
rev: 1
```
