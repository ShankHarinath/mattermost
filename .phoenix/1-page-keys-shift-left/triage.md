# Triage Report

## Summary

**Bug:** Pressing PageUp or PageDown while the message textarea has an internal scrollbar (long draft) and the RHS (right-hand sidebar / thread panel) is open causes the Mattermost UI to "move to the left" — i.e. the center channel visually shifts/scrolls horizontally instead of scrolling the caret within the textarea.

**Root cause hypothesis:** The advanced text editor's keyboard handler (`use_key_handler.tsx`) does **not** intercept `PageUp` / `PageDown`. The `Constants.KeyCodes.PAGE_UP` / `PAGE_DOWN` constants are defined in `webapp/channels/src/utils/constants.tsx` (lines 1834–1835) but are referenced nowhere in the codebase. So PageUp/PageDown fall through to browser default behavior. Because the textarea now has `overflowY: 'auto'` (introduced in commit `2df5ff9d9b`, PR #27369, "MM-54561 Enable scrolling and visible scrollbar in post textbox"), a long draft creates internal scrollable content — but the default PageUp/PageDown action on a `<textarea>` only moves the caret, it does not consume horizontal scroll of the ancestor. When the RHS is open, `.inner-wrap.move--left` is applied (`center_channel.tsx:75`). At narrow-but-not-mobile viewport widths (the media queries in `_tablet.scss:430`, `_mobile.scss:1747`/`1757` and the window-sizing rules in `_desktop.scss:55`–`117`), the combined width of sidebars + RHS + center channel exceeds the viewport, creating horizontal overflow on a parent scroll container. The browser then translates a PageUp/PageDown keystroke (when focus leaves the `<textarea>` caret model momentarily due to the autoresized scrollable content, or via focus handling in `SuggestionBox`) into a horizontal document-scroll — visually shifting the whole app left. The reporter's workaround of `Ctrl+R` resets horizontal scroll by forcing a reload.

**Confidence:** 0.62 — hypothesis is consistent with all evidence (scroll precondition traces to a recent commit; no handler exists; `move--left` geometry + open RHS creates horizontal overflow; Ctrl+R workaround implies document-scroll state reset). Remaining uncertainty: the exact browser-chain DOM ancestor that's receiving and executing the horizontal scroll has not been pinpointed via runtime DevTools — that requires manual reproduction in Chrome/FF with DevTools on a specific window width.

## Evidence

### A. Code mapping (GitNexus + grep)

- `Constants.KeyCodes.PAGE_UP` / `PAGE_DOWN` — declared at `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/utils/constants.tsx:1834-1835`. Grep confirms **zero other references** across `webapp/` — no handler wires them.
- Main message editor key handler — `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/advanced_text_editor/use_key_handler.tsx:138-345`. The `handleKeyDown` checks UP/DOWN, ENTER, ESCAPE, V, B, I, K, C, E, T, P, X, 7/8/9, BACK_SLASH — **no branch handles PAGE_UP or PAGE_DOWN**, so the event propagates with default browser action.
- Textbox composition — `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/textbox/textbox.tsx` wraps an `AutosizeTextarea` inside a `SuggestionBox`. `SuggestionBox` also has no PageUp/PageDown handling (grep in `components/suggestion` returns no matches).
- Textarea scroll — `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/autosize_textarea.tsx:45-47,177` — `style={{overflowY: 'auto'}}` creates the precondition "scroll appears in the message entry window".
- Layout modifier when RHS opens — `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/channel_layout/center_channel/center_channel.tsx:73-77` — applies `move--left` to `.inner-wrap` whenever `rhsOpen` is true. CSS for this class (translate, width changes) lives in `sass/layout/_content.scss`, `sass/responsive/_desktop.scss:26,42,55-117`, `sass/responsive/_tablet.scss:430,470,487`, `sass/responsive/_mobile.scss:1747,1757`.
- Upstream consumers of Textbox (blast radius) — GitNexus `impact` shows both `advanced_create_post.tsx` (center composer) and `advanced_create_comment.tsx` (RHS thread composer) consume Textbox; fix in the shared key handler covers both.

### B. Runtime logs (kubectl)

Intentionally skipped. This is a pure front-end layout/keyboard interaction bug in the React webapp; no Mattermost server pods can observe or emit logs for it. Runtime verification requires browser-side reproduction (Windows 10 / Ubuntu 22.04 per reporter, Mattermost 9.11.1 build 10419911144) with DevTools inspecting `scrollLeft` on `.inner-wrap`, `#channel_view`, and `body`.

### C. Recent commits on affected files

- `2df5ff9d9b` — **MM-54561 Enable scrolling and visible scrollbar in post textbox (#27369)**, 2024-07-25. Adds `overflowY: 'auto'` to the textarea style. This is the change that makes the bug's precondition ("scroll appears in the message entry window") reachable. The bug is a side-effect of that feature because no PageUp/PageDown handler was added alongside.
- `12f8ff0e8b` disable preview formattingBar disabled, `5030e2cdb2` Fix Ctrl-up when changing the value on different inputs (#28062), `422df51d77` Fix unfocussed react to last message (#28091) — recent key-handler work, none touching PageUp/PageDown.

### D. Similar past issues / PRs

No prior issue or PR referencing PageUp/PageDown was found in the fork (`ShankHarinath/mattermost`) or surfaced in the local git log. Upstream `mattermost/mattermost` search returned no matches via `gh` in this environment — may require repo-scoped search with better keywords.

## Proposed fix direction (for Architect — not implementation)

Intercept `PageUp` and `PageDown` inside `use_key_handler.tsx` `handleKeyDown`, similar to how `UP` / `DOWN` / `ENTER` / `ESCAPE` are handled. On key press:

1. Let the native textarea default (caret move within the textarea) happen.
2. Call `e.stopPropagation()` so the event cannot bubble to an ancestor document-scroll handler.
3. Optionally, explicitly scroll the textarea via `el.scrollTop += ±el.clientHeight` and `e.preventDefault()` for cross-platform determinism.

Alternative (or complementary) fix — eliminate the horizontal scroll root. Audit the `.inner-wrap.move--left` and `#root` CSS geometry in `sass/base/_structure.scss:111-124`, `sass/layout/_content.scss`, and the responsive files to ensure no ancestor of the textarea is horizontally scrollable at the viewport widths the reporter hits (likely `1025px <= width <= ~1280px` when the RHS narrows the main column). Adding `overflow-x: hidden` to `.inner-wrap` or the outer grid container is the canonical cure for this class of "browser scrolls the page horizontally" bugs.

Architect should pick one of these approaches — the key-handler interception is lower-risk (localized change in one file, MEDIUM blast radius per `impact` — 9 direct importers, no processes affected), and it fixes the symptom for all causes (regardless of which ancestor introduces the horizontal scroll).

## Files most likely to change

- `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/advanced_text_editor/use_key_handler.tsx` — primary fix location.
- Possibly `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/sass/layout/_content.scss` or `_structure.scss` — if Architect chooses the CSS `overflow-x: hidden` approach.
- Tests — `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/advanced_text_editor/advanced_text_editor.test.tsx` should gain a case covering PageUp/PageDown while a draft is long enough to scroll.

## Open questions

- **Viewport-width band for reproduction** — the reporter did not state window size. The CSS mediaqueries suggest this only manifests between ~1025px and ~1680px when RHS is open. Architect / QA should reproduce at 1280×800 and 1440×900 with RHS open and a ~2000-char draft.
- **Browser(s)** — reporter mentions Windows 10 and Ubuntu 22.04 but not Chrome/FF/Edge. Worth confirming whether all Chromium-based browsers are affected equally.
- **SuggestionBox focus state** — whether an active @-mention or slash-command suggestion popover changes PageUp/PageDown behavior needs confirmation; the suggestion handler currently does not bind these keys, but the popover can steal focus.
- **Horizontal-scroll root** — not pinpointed in code-only investigation; DevTools repro required to confirm whether it's `body`, `#root`, `.inner-wrap`, or `#channel_view` that gains non-zero `scrollLeft`.
- **Possible complication — scheduled_posts / `scheduled_post_indicator`** — recent work around scheduled posts (`e281b3f37e Feature scheduled messages (#28932)`) touched the key handler; unlikely to be a direct cause but worth a quick review when editing `use_key_handler.tsx`.

## Recommended next step

Implement the PageUp/PageDown interception in `use_key_handler.tsx` and add `e.stopPropagation()` (and explicit textarea scroll nudge + `e.preventDefault()` for determinism). Verify with a unit test that fires PageUp/PageDown on the textbox and asserts the event does not bubble, plus a manual repro at 1280×800 with RHS open and a long draft.

---

```yaml
# phoenix-triage-footer
affected_repos:
  - mattermost
confidence: 0.62
recommended_next_step: plan
```
