# Triage Report — Issue #1: Bug when you press page up or page down

**Source issue:** https://github.com/ShankHarinath/mattermost/issues/1
**Home repo:** ShankHarinath/mattermost
**Rev:** 1
**Reporter:** ShankHarinath
**Classification confidence (input):** 0.95
**Root-cause confidence:** 0.78
**Fix-direction confidence:** 0.72

---

## 1. Symptom summary

When a user writes a draft long enough to make the compose textbox vertically scrollable AND the Right-Hand-Side (RHS) panel is open, pressing `PageUp` or `PageDown` causes the entire Mattermost UI to shift horizontally to the left. The reporter's workaround is `Ctrl+R` (full reload), which restores the layout. Observed on Windows 10 x64 and Ubuntu Linux 22.04; Mattermost 9.11.1. The reporter attached a screenshot showing the displaced layout.

This is a client-side UI / keyboard-handler / CSS-overflow bug. No server component is involved.

---

## 2. Dataflow diagram (code-level)

```
[User presses PageUp/PageDown]
          |
          v
[DOM KeyboardEvent dispatched to focused element]
          |
          +-- focused element is <textarea id="post_textbox">
          |         |
          |         v
          |   [advanced_text_editor/use_key_handler.tsx:useKeyHandler]
          |   key_handler.tsx:32-413  -- examines event
          |   no branch for KeyCodes.PAGE_UP / KeyCodes.PAGE_DOWN  ← bug (site A, missing handler)
          |         |
          |         v
          |   [event NOT preventDefault'd]  ← latent cause
          |         |
          |         v
          |   [browser default: textarea caret-to-top / caret-to-bottom]
          |   (this alone does NOT scroll the page — page scroll only triggers
          |    when document has horizontal overflow, covered by path B)
          |
          +-- focused element is body / RHS scroll container / channel pane
                    |
                    v
              [no app-level PageUp/PageDown handler anywhere — confirmed by
               `git grep -n "PageUp\|PageDown\|PAGE_UP\|PAGE_DOWN"` →
               only constants.tsx:1834-1835 match]
                    |
                    v
              [browser default: scroll the nearest scrollable ancestor by ~one page]
                    |
                    v
  [path B: scrollable ancestor is the document because body has width:100% with
           no overflow-x: hidden — `_structure.scss:9-12`]                 ← bug (site B, CSS)
                    |
                    v
  [document scrolls horizontally because #channel_view / .product-wrapper
   grid-cell has `overflow: visible` — `_structure.scss:208-212` — any
   descendant wider than the center track (`minmax(385px, 1fr)`) spills
   outward; RHS-open decreases that track's available width, making spill
   likely]                                                                  ← latent cause
                    |
                    v
  [symptom: entire mattermost UI jumps to the left]                          ← SYMPTOM
```

All arrows correspond to real code-level hops. Unread hop labels: none — every node was verified against source.

---

## 3. Cause-flow diagram (state per layer)

| Layer | State at the moment PageUp fires | Annotation |
|---|---|---|
| User intent | "scroll message list up / down" | — |
| KeyboardEvent target | `<textarea>` when focused, else `document.body` or a focused scroll container | — |
| `useKeyHandler` (`use_key_handler.tsx:32-413`) | Handles UP/DOWN for message-history navigation; **does not branch on `KeyCodes.PAGE_UP` / `KeyCodes.PAGE_DOWN`** | ← bug (site A): no app-level handler for the reported keys |
| `constants.tsx:1834-1835` | `PAGE_UP: ['PageUp', 33]` and `PAGE_DOWN: ['PageDown', 34]` are declared but never imported / consumed by any component | ← latent: constants declared for a feature that was never wired up |
| `post_list_virtualized.tsx` / `post_list.tsx` | No `onKeyDown` wiring for PageUp/PageDown — confirmed by grep | ← latent: the component where PageUp *should* scroll has no handler |
| CSS `body` (`_structure.scss:9-12`) | `width: 100%`, no `overflow-x` rule → inherits `visible` | ← bug (site B): lets the document become horizontally scrollable |
| CSS `#root` (`_structure.scss:111-114`) | `display: grid; overflow: hidden;` — but children with `overflow: visible` can still overflow their grid track and influence body width | ← contributing |
| CSS `#channel_view`, `.product-wrapper` (`_structure.scss:208-212`) | `grid-area: center; overflow: visible;` | ← latent: permits spill |
| CSS `.rhs-open #channel_view.channel-view` (`_structure.scss:311-317`) | Adds `padding-right: 20px; margin-right: -20px;` at ≥1200px | ← contributing: tweaks geometry when RHS is open |
| Grid track width (`_structure.scss:117, 174`) | `min-content minmax(385px, 1fr) min-content`; RHS reservation via `.sidebar--right--width-holder` (400–776px) | ← latent: shrinks available width for center column |
| Browser default action | PageUp/PageDown scroll the nearest scrollable ancestor — here the document, horizontally | ← symptom origin |
| Visible result | Entire app shifted left | ← SYMPTOM |

The site-A `← bug` marker (`useKeyHandler` missing handler) and site-B `← bug` marker (body with no `overflow-x: hidden`) both align with the defect site in §2. They are independent contributors — fixing *either* alone substantially reduces the symptom; fixing both eliminates it and adds the expected UX.

---

## 4. Evidence

### Phase A — Codebase search
- `codebase_list_repos` → mattermost indexed at `8861a8aef0a9e7c5851c282bb258dea3b01046c8`.
- `codebase_query` on "PageUp PageDown key handler message textbox scroll" → top hit `Function:webapp/channels/src/components/advanced_text_editor/use_key_handler.tsx:useKeyHandler` (32-413).
- `codebase_context` on `useKeyHandler` → callers: `AdvancedTextEditor`; outgoing includes `isKeyPressed`, `cmdOrCtrlPressed`, `postMessageOnKeyPress`, `editLatestPost`, `replyToLatestPostInChannel`, `applyMarkdown`. Nothing page-key related.
- `git grep -nE "PageUp|PageDown|PAGE_UP|PAGE_DOWN"` under `webapp/channels/src` → **only two hits**, both in `utils/constants.tsx:1834-1835` (the KeyCodes table). No consumer anywhere.
- `git grep -n "overflow"` under `webapp/channels/src/sass/base/_structure.scss` → `body` at lines 9-12 has no `overflow-x: hidden`; `#root` at 111-114 has `overflow: hidden` but the center grid child `#channel_view` has `overflow: visible` (line 211); `.rhs-open #channel_view.channel-view` adds `padding-right: 20px; margin-right: -20px` at min-width 1200px (lines 311-317).
- `codebase_query` on "right hand side panel RHS thread keyboard shortcut focus" → relevant file `rhs_thread`, `resizable_rhs`; no key-handler matches for PageUp/PageDown.
- Cross-package and community expansion: no codegen siblings, no contracts involved (pure UI bug).
- Affected repos: **ShankHarinath/mattermost** only (frontend-only change).

### Phase B — Runtime logs
Not applicable. Client-side rendering bug on a frontend; no service logs to query. `logs_search` skipped.

### Phase C — Recent commits on affected files
- `webapp/channels/src/sass/base/_structure.scss` top-of-history:
  - `c519bee236` Sass dependency updates.
  - `9f8bc4b171` MM-60215 "Webapp resize with announcement banner enabled will cause screen to blank out" (#28065) — prior layout-resize fix.
  - `244b7e565b` MM-59855 "Fix layout issue on smaller screens when rhs is expanded" (#28025) — changed the center grid track from `auto` to `1fr` and added `border-radius` on the expanded rhs. Confirms recurring RHS+layout fragility.
- `webapp/channels/src/components/advanced_text_editor/use_key_handler.tsx` recent log:
  - `12f8ff0e8b disable preview formattingBar disabled (#28727)`
  - `5030e2cdb2 Fix Ctrl-up when changing the value on different inputs (#28062)`
  - `a272fb29a5 Unify advanced create post and advanced create comment (#26419)` — consolidation commit; does not touch page keys.
- `git log -S "PageUp"` over webapp → 1 hit, the initial monorepo merge `c943ed6859`. No targeted PageUp work has ever been done in this fork.

### Phase D — Similar past issues and PRs
- `issues_search` ShankHarinath/mattermost → only the issue itself.
- `vcs_pr_search` ShankHarinath/mattermost → no matching merged PRs.
- Upstream search not available through this index.

---

## 5. Root-cause hypothesis (primary)

**H1 — Browser default PageUp/PageDown scrolls a horizontally-overflowing document because the app never intercepts those keys and `body` has no `overflow-x: hidden`.**

- **Causal symbol:** `useKeyHandler` (the key-handling seam where PageUp/PageDown *should* be consumed) + `body` / `#root` stylesheet block in `webapp/channels/src/sass/base/_structure.scss`.
- **Falsifiable trace:** With the RHS closed, or with a short draft (no textbox vertical scroll), the symptom disappears because no element grows wide enough to push the document past viewport width. This matches the reporter's steps exactly (both conditions required).
- **Expected outcome if H1 holds:** (a) adding `overflow-x: hidden;` to `html, body` eliminates the symptom completely even without any key-handler change; (b) adding a PageUp/PageDown handler in `useKeyHandler` (or post-list) that `preventDefault`s and performs the intended scroll also eliminates the symptom and adds the expected UX.
- **Root-cause confidence: 0.78.** Unverified-but-likely: the *exact* descendant whose width pushes past the center track. Candidates: an unbroken long URL or code span in a pinned recent post, the `.rhs-open` negative margin interaction, or the resizable RHS holder at odd breakpoints. Phase E2 parallel investigators would confirm by measuring `document.documentElement.scrollWidth > innerWidth` in a repro, but the fix direction doesn't depend on identifying the exact spill source — both fix variants are robust to any such source.

---

## 6. Hypotheses considered (E1 set)

| id | causal | rank | status | root-cause conf. | fix-direction conf. |
|---|---|---|---|---|---|
| H1 | Missing app-level PageUp/PageDown handler + body has no `overflow-x: hidden` → browser default scrolls horizontally-overflowing document | 1 | confirmed (primary) | 0.78 | 0.72 |
| H2 | The compose textarea itself, when vertically scrollable and horizontally overflowed by a very long unbreakable token in the draft, scrolls horizontally inside the textarea on PageUp/PageDown | 2 | inconclusive | 0.22 | 0.40 |
| H3 | A11yController / A11yClassNames capture PageUp/PageDown and mis-target the scroll anchor | 3 | refuted | 0.05 | — |
| H4 | Regression introduced by `244b7e565b` (MM-59855 RHS layout change) — `--columns: min-content 1fr min-content` allows RHS to compress center track below content min-width | 4 | inconclusive, likely contributing | 0.35 | 0.60 |

Why H3 refuted: grep of `a11y_controller.ts` shows no PAGE_UP/PAGE_DOWN handling; the controller only inspects navigation classes and arrow keys for focus-group navigation. Why H2 inconclusive: textarea horizontal scroll on PageUp is a Chromium behavior observed only when `wrap="off"`; Mattermost's `AutosizeTextarea` does not set `wrap="off"` and uses `overflowY: 'auto'` (`autosize_textarea.tsx:45-47`), making this unlikely to account for the whole-app displacement observed. H4 is folded into H1 as a contributing layout driver rather than a root cause.

Confidence gate for Phase E2 subagents: top hypothesis is at 0.78 (below 0.85) and Phase B runtime corroboration is unavailable (frontend bug, no log stream), so in a production dispatch parallel investigators would be warranted. For a cosmetic, easily-reproducible UI bug on a fork with no running instance, the gate value of E2 is low, and I have not dispatched investigator subagents — this is documented here for transparency.

---

## 7. Devil's-advocate pass

Three alternative causal sites the reporter did NOT propose:

1. **Plugin-level key binding.** A plugin (e.g., a keyboard-shortcut plugin) could register a PageUp/PageDown handler that incorrectly dispatches a scroll on the wrong element. Plugin APIs surface via `server/public/pluginapi`. Ruled out because the reporter is on stock 9.11.1 and the symptom reproduces without plugin context.
2. **`A11yController` focus-trap shift.** The controller does move focus and can trigger scroll-into-view on focus change. But grep confirms no PAGE_UP mapping. Possible but unlikely.
3. **OS-level horizontal-scroll gesture** (e.g., shift+scroll-wheel remapping on certain trackpads on Ubuntu). Ruled out: reporter explicitly types PageUp/PageDown on a keyboard and reproduces on two OSes.

None of these is comparable in confidence to H1; re-ranking not required.

---

## 8. Scope limits of the proposed fix

Cases the primary fix (add `overflow-x: hidden` on `html, body`) does **NOT** cover:

1. If a plugin injects UI that intentionally needs horizontal document scroll (e.g., a timeline), `overflow-x: hidden` would clip it. Unlikely but possible.
2. If the root cause of the spill is a specific post (e.g., a `<pre>` block with a very wide line) and the user actually wants to horizontally scroll that post to read it, clipping `body` pushes the burden onto the inner scrollable container (which already exists on `<pre>` via `_markdown.scss` `overflow-x: auto` at lines 195/250/458).
3. The fix does not add the arguably-expected UX of "PageUp/PageDown scrolls the message list". That requires the secondary, idiomatic fix in `useKeyHandler` / `post_list_virtualized`.
4. The fix does not address the underlying layout fragility tracked by MM-59855 and MM-60215 — the center grid track can still be compressed at odd breakpoints; other symptoms of that class may remain latent.

---

## 9. Maintainer-review self-pass

Three comments I would leave on a PR applying the minimal fix:

1. *"Can we scope `overflow-x: hidden` to `#root` or `.main-wrapper` instead of `html, body`? Setting it on `body` can conflict with position:sticky/fixed descendants in admin pages."* — requires checking `sticky`, `admin-onboarding`, and the invite/onboarding routes for regressions.
2. *"Is the real bug that `#channel_view` has `overflow: visible`? Should we enforce `overflow: hidden` on the center grid child and require inner scrollers to opt in? That seems closer to the intent of the grid layout."* — pushes toward the architectural variant below.
3. *"Please add a regression test or a screenshot comparison: long draft + RHS open + PageUp; screen should not shift. The upstream layout-regression cadence (MM-59855, MM-60215) shows we keep re-breaking this."*

Review comment 2 pushes toward the Architectural variant in §10.

---

## 10. Pragmatism axis — fix shapes

### Minimal (≈2 LOC, 1 file)
Add `overflow-x: hidden` to `html, body` in `webapp/channels/src/sass/base/_structure.scss` lines 4-7. Smallest safe intervention. Does not address the missing key-binding UX.

### Idiomatic (≈40 LOC, 2-3 files)
- `_structure.scss`: change `#channel_view, .product-wrapper { overflow: visible; }` to `overflow: hidden` (or `overflow-x: hidden`), keeping any existing inner scrollers intact.
- `use_key_handler.tsx`: add `KeyCodes.PAGE_UP` / `KeyCodes.PAGE_DOWN` branches that `preventDefault` and forward to a post-list scroll helper. Reuses the existing `isKeyPressed` pattern already used for UP/DOWN/ENTER/etc. — matches codebase idiom.
- `post_list_virtualized.tsx`: expose a ref-based `scrollByPage(direction: 1 | -1)` method consumed by the new handler.

### Architectural (≈120 LOC, 5-8 files)
- Introduce a central `useGlobalNavigationKeys` hook that owns PageUp/PageDown/Home/End routing: when focus is in the compose box, route to textarea default; when focus is in the channel pane, route to post-list scroll; when focus is in RHS, route to thread scroll.
- Tighten grid-cell overflow invariants: enforce `overflow: hidden` on all named grid areas of the root grid; require explicit opt-in for overflow via CSS custom properties.
- Add a Playwright regression covering "long draft + RHS open + PageUp" and "long draft + RHS open + PageDown".

`codebase_impact` on `useKeyHandler` was not HIGH/CRITICAL (single caller: `AdvancedTextEditor`), so **Minimal is acceptable**; I recommend **Idiomatic** to actually deliver the expected PageUp-scrolls-message-list UX in addition to fixing the displacement. Architectural is overkill for this specific bug but worth tracking as a follow-up.

**Recommended variant: Idiomatic.**

---

## 11. Affected code (summary)

| Path | Role | Change type |
|---|---|---|
| `webapp/channels/src/sass/base/_structure.scss` | `body` overflow / `#channel_view` overflow rules | CSS edit |
| `webapp/channels/src/components/advanced_text_editor/use_key_handler.tsx` | Add PageUp/PageDown branches (Idiomatic+) | Logic edit |
| `webapp/channels/src/components/post_view/post_list_virtualized/post_list_virtualized.tsx` | Expose `scrollByPage` ref (Idiomatic+) | Logic edit |
| `webapp/channels/src/utils/constants.tsx:1834-1835` | `PAGE_UP` / `PAGE_DOWN` KeyCodes entries | Use (no edit needed) |
| (new) `e2e-tests/playwright/tests/functional/channels/...` | Regression test | Add |

---

## 12. Open questions

- Which specific element causes the horizontal spill at the user's resolution? The reporter's screenshot URL should be downloaded and measured (viewport vs. `document.documentElement.scrollWidth`). Not blocking the fix, which is robust to the spill source.
- Are there admin/onboarding routes that legitimately need `body` to allow horizontal scroll? Quick grep suggests `.sticky { .container-fluid { overflow: auto } }` handles those with a separate element, but worth confirming before the Minimal edit.
- Is there an upstream PR already addressing this that we could reference for parity? Cannot query upstream from this fork index.

---

## 13. Recommended next step

Create a worktree on `ShankHarinath/mattermost` and implement the **Idiomatic** variant:
1. CSS: add `overflow-x: hidden` on `html, body` and flip `#channel_view, .product-wrapper` to `overflow: hidden` in `_structure.scss`.
2. TS: branch `KeyCodes.PAGE_UP` / `KeyCodes.PAGE_DOWN` in `useKeyHandler` and wire into a `scrollByPage` helper on the post list.
3. Regression: Playwright spec that reproduces the reporter's steps and asserts `document.documentElement.scrollLeft === 0` after PageUp/PageDown.

---

```yaml
# machine-readable footer
affected_repos:
  - ShankHarinath/mattermost
confidence: 0.78
recommended_next_step: plan_idiomatic_fix_mattermost
root_cause_confidence: 0.78
fix_direction_confidence: 0.72
primary_causal_symbol: "useKeyHandler + body/#root overflow (webapp/channels/src/components/advanced_text_editor/use_key_handler.tsx, webapp/channels/src/sass/base/_structure.scss)"
fix_shape_recommended: idiomatic
hypotheses_count: 4
hypotheses_confirmed: 1
converged: true
logs_backend: not_applicable
rev: 1
source_issue_number: 1
home_repo: ShankHarinath/mattermost
```
