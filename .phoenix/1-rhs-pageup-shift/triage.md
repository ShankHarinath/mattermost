# Triage Report — ShankHarinath/mattermost#1

**Revision:** 1
**Issue:** ShankHarinath/mattermost#1 — "Bug when you press page up or page down"
**Reporter:** ShankHarinath
**Created:** 2026-04-20T20:16:45Z
**Classification confidence (from dispatcher):** 0.90

## Summary

When the user has typed a long message (causing the message input `<textarea>` to show its own vertical scrollbar) **and** the right-hand side (RHS) panel is open, pressing `PageUp` or `PageDown` causes the entire center channel to shift horizontally to the left. `Ctrl+R` (page reload) is the reporter's workaround, which is consistent with the browser's horizontal `scrollLeft` being reset by a reload.

The repository has **no JavaScript keydown handler for PageUp/PageDown on the message input**. The behavior is native browser default: PageUp/PageDown scrolls the nearest scrollable ancestor of the focused element. That ancestor ends up being `#channel_view.channel-view`, which has 20px of horizontal overflow on the right whenever RHS is open at viewport widths ≥ 1200px, due to a CSS rule introduced in May 2024.

## Phase A — Code mapping

### A.1 Search radius

Chosen repos: `ShankHarinath/mattermost` only.
Reasoning: self-contained webapp UI bug; no indications of a cross-service or cross-repo interaction. The home repo is not part of any indexed group that matters (`group_list` shows backend/frontend/infra groups unrelated to this fork).

### A.2 Queries

| Signal | Hit | File |
|---|---|---|
| PageUp / PageDown symbol | Defined only as a key-code constant; no keydown handler references it | `webapp/channels/src/utils/constants.tsx:1834` (`PAGE_UP`, `PAGE_DOWN`) |
| Message entry keydown handler | `use_key_handler.tsx` — does not handle PageUp/PageDown | `webapp/channels/src/components/advanced_text_editor/use_key_handler.tsx` |
| Channel view container (`#channel_view`) | Rendered by `ChannelController`; `div#channel_view.channel-view` wraps center column | `webapp/channels/src/components/channel_layout/channel_controller.tsx:75-87` |
| RHS-open layout rule | `.rhs-open #channel_view.channel-view { padding-right: 20px; margin-right: -20px }` at `@media (min-width: 1200px)` | `webapp/channels/src/sass/base/_structure.scss:303-317` |

Additional finding: line 312 contains a **selector typo** — `.rhs-open-expanded #channel_view.channel.view` (note `.channel.view` — two separate classes) where the author intended `.channel-view` (one class). This typo means the expanded-RHS branch of the rule never matches any element. It does not directly cause the bug (the non-expanded branch on line 311 still fires), but it is a latent defect in the same commit.

### A.3 Misses

No JS keydown handler anywhere in `webapp/channels/src/**` listens for PageUp/PageDown. Confirmed via:

- `Grep keyCode === 33|keyCode === 34|\.key === 'PageUp'|\.key === 'PageDown'` → no matches in webapp/channels/src
- `Grep PAGE_UP|PAGE_DOWN` → only the key-code constants definition and one Cypress test

So the observed behavior is unambiguously the browser's default scroll response, acting on whichever ancestor it deems scrollable.

### A.4 Affected repos

`ShankHarinath/mattermost` only.

## Phase B — Runtime evidence (kubectl)

**Not applicable.** This is a client-side UI / layout bug. There are no server-side error logs that would corroborate it. Skipped cluster log collection. No runtime signal was recorded.

## Phase C — Recent commits on affected files

`git log webapp/channels/src/sass/base/_structure.scss` (most recent 10):

- `c519bee236` — Fixathon: Web app dependency updates part 2 (Sass) (#29037)
- `9f8bc4b171` — [MM-60215] Webapp resize with announcement banner enabled will cause screen to blank out (#28065)
- `244b7e565b` — MM-59855 Fix layout issue on smaller screens when rhs is expanded (#28025)
- `6316606289` — MM-59944 FIxing playbooks scroll issue (#27861)
- ...
- **`8fa6757949` — MM-56975 UI Incremental Refinements (#26407)** (2024-05-08, Matthew Birtch)

`git blame` on the suspect block (lines 311-316) attributes both the `padding-right: 20px; margin-right: -20px;` rule **and** the `.channel.view` typo to commit `8fa6757949`. This commit predates Mattermost 9.11.1 (the reporter's build), so the defect is present in the reported build.

## Phase D — Similar past issues and PRs

- `gh issue list --repo ShankHarinath/mattermost --search "page up page down"` → only this issue (#1); no prior history in the fork.
- Cross-fork search on upstream `mattermost/mattermost` denied by local `gh` shim (`gh-shim: DENIED — arg 'mattermost/mattermost' references owner 'mattermost'`). Recorded as a blocker; see Open Questions.

## Phase E — Hypothesis

**Root cause (proposed, not verified against a live browser):**

The CSS block at `webapp/channels/src/sass/base/_structure.scss:311-317`

```scss
.rhs-open #channel_view.channel-view,
.rhs-open-expanded #channel_view.channel.view {   // <-- typo: should be .channel-view
    @media screen and (min-width: 1200px) {
        padding-right: 20px;
        margin-right: -20px;
    }
}
```

combined with `overflow: hidden` on `#channel_view.channel-view` (line 304) creates a scroll container whose content box is 20px wider than its padding/margin box on the right. Browsers (notably Chromium on Windows / Ubuntu, matching the reporter's OS list) treat `overflow: hidden` elements as keyboard-scrollable. When the focused `<textarea>` has its own vertical scrollbar, PageUp/PageDown can bubble out to the nearest scrollable ancestor — `#channel_view` — and scroll it horizontally by 20px, visually pushing the entire center channel to the left. Pressing `Ctrl+R` reloads the page, resetting `scrollLeft` to 0, which matches the reporter's "possible fix."

The typo on line 312 is a separate latent bug in the same commit: the intended selector `.channel-view` (hyphenated, one class) was written as `.channel.view` (two classes), so the `rhs-open-expanded` branch never applies. If the bug reproduces only with non-expanded RHS, the typo is circumstantial; if it reproduces with expanded RHS as well, there may be additional scroll containers at play.

**Confidence:** 0.65

- Strong static evidence: exact file+lines identified, introducing commit identified, reporter's symptom (shift-left) and workaround (`Ctrl+R` = reload) are both consistent with an uncleared horizontal `scrollLeft` on `#channel_view`.
- Not verified in a live browser (cannot reproduce without a running desktop build), so confidence is below the "strong hit" 0.85 threshold.
- Not verified whether `#channel_view` is actually the container the browser picks for PageUp/PageDown (could also be `html`/`body`). If the scroll lands on a different ancestor, the same class of fix still applies but at a different element.

## Recommended fix direction (for Architect — do not implement in this phase)

1. **Primary:** eliminate the 20px horizontal overflow on `#channel_view.channel-view` when RHS is open. Options, in order of preference:
   - Replace `padding-right: 20px; margin-right: -20px;` with a gap approach that does not introduce overflow (e.g., let the RHS gap come from the grid column, or move the spacing onto `.sidebar--right` / a separate spacer element).
   - If the negative margin must stay, add `overflow-x: clip` (stronger than `hidden` for scroll-prevention) on `#channel_view.channel-view` so PageUp/PageDown cannot scroll it horizontally.
2. **Secondary (latent defect cleanup):** fix the selector typo on `webapp/channels/src/sass/base/_structure.scss:312` — `.channel.view` → `.channel-view` — so the `rhs-open-expanded` branch matches its intended element.
3. **Test:** add a Cypress regression case in `e2e-tests/cypress/tests/integration/channels/keyboard_shortcuts/` that (a) opens RHS, (b) types a multi-line message to force a textarea scrollbar, (c) presses PageUp/PageDown, and asserts `#channel_view.scrollLeft === 0` afterwards.

## Open Questions / Blockers

- **Live-browser reproduction not performed.** Triage is static-analysis only. An Architect or human reviewer should confirm in a running instance which element receives `scrollLeft` on PageUp/PageDown at viewport width ≥ 1200 with `.rhs-open` set on body. A DevTools check of `document.getElementById('channel_view').scrollLeft` before and after PageUp would prove or refute the hypothesis in seconds.
- **Upstream history blocked.** `gh` shim denied searches on `mattermost/mattermost`, so I could not check whether this bug (or a near-duplicate) is tracked upstream, nor whether PR #26407's diff included other layout changes that may have overflow implications. Recommend the human reviewer check upstream before authoring a fix, to avoid duplicate work.
- **Typo scope uncertain.** The typo on line 312 may or may not be part of the visual symptom depending on whether the RHS in the reporter's screenshot was in expanded mode. Screenshot provided in the issue is hosted on GitHub user-attachments and was not fetched.

## Files of interest (absolute paths)

- `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/sass/base/_structure.scss` — lines 303-317 (root cause), line 312 (typo)
- `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/channel_layout/channel_controller.tsx` — lines 75-87 (container markup)
- `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/advanced_text_editor/use_key_handler.tsx` — confirms absence of a PageUp/PageDown handler
- `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/utils/constants.tsx` — line 1834-1835 (`PAGE_UP`/`PAGE_DOWN` constants only; unused in event handlers)

---

```yaml
# machine-readable footer
affected_repos:
  - ShankHarinath/mattermost
confidence: 0.65
recommended_next_step: architect_plan
```
