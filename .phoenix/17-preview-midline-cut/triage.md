# Triage Report — Issue #17

## Summary
Permalink/link previews render a markdown post body inside a `ShowMore` wrapper that uses **two conflicting clip mechanisms simultaneously**: a pixel `max-height: 105px` on `.post-message__text-container` (`overflow: hidden`) combined with a CSS `-webkit-line-clamp: 4` on `.post-message__text`. When the source post mixes heading levels (`##`/`###`/`####` with different `font-size` and `line-height`), the pixel clip falls inside a heading's glyph row while line-clamp keeps drawing past it — producing the "text cut mid-line" artifact shown in the screenshot. The sample message in the reproduction uses exactly this pattern (h2, h4, h3, h3) and is the minimal repro.

## Phase A — Code map

**Repo radius:** home repo `ShankHarinath/mattermost` only. This is a frontend/UI rendering bug; no contracts or cross-service surface. GitNexus `group_list` returned no group membership for mattermost. Other indexed repos are unrelated Canonix services.

**Searched repos:** `mattermost` (61,203 nodes / 40,789 embeddings, index fresh as of 2026-04-21 10:57).

**Symptom → code mapping:**

| Signal | Symbol / file | Why it matches |
|---|---|---|
| "link preview" (copy-link paste renders a preview) | `PostMessagePreview` — `webapp/channels/src/components/post_view/post_message_preview/post_message_preview.tsx` | Wrapper rendered for a permalink; passes `overflowType='ellipsis'`, `maxHeight={105}` to `PostMessageView` (lines 188–194). |
| "show more button if too long" (expected behaviour) | `ShowMore` — `webapp/channels/src/components/post_view/show_more/show_more.tsx` | Detects overflow (`scrollHeight > maxHeight`, line 85) and renders the ellipsis-variant button (lines 149–161) when `overflowType='ellipsis'`. |
| Mid-line cut (observed) | `.post-message__text-container` + `.post-message-preview--overflow .post-message__text` in `webapp/channels/src/sass/components/_post.scss` lines 1860–1870 | Container has `overflow: hidden` + inline `max-height: 105px`; inner `.post-message__text` additionally uses `-webkit-line-clamp: 4` over a `-webkit-box`. The two clips disagree on rendered heading rows. |
| Markdown rendering | `PostMessageView.render` — `webapp/channels/src/components/post_view/post_message_view/post_message_view.tsx:198-207` | Passes `overflowType`/`maxHeight` through to `ShowMore`. |

**Root-cause location:** the pair at `post_message_preview.tsx:190-191` (the `maxHeight={105}` + `overflowType='ellipsis'` props) and `_post.scss:1860-1870` (the two-mechanism CSS). `show_more.tsx:78-95, 185-193` is the detection/inline-style code that drives the pixel clip.

## Phase B — Runtime evidence (kubectl)

Skipped. This is a client-side rendering bug visible only in the browser (no server logs, no error signature). The reporter supplied a visual repro; the server build number / schema version are informational and don't indicate a runtime signal.

## Phase C — Recent commits in the affected area

`webapp/channels/src/components/post_view/show_more/show_more.tsx`:

| SHA | Date | Note |
|---|---|---|
| 20f9f58e4c | 2025-05-01 | MM-63648 tried to fix "markdown images sometimes do not show the more button" (edited ShowMore.tsx heavily, +45 lines) |
| 332be84efd | 2025-05-28 | **Revert** of 20f9f58e4c — indicates this area has regressions when edited |
| 649b939e6a | MM-63898 | Improve blockquote style (touches nearby CSS) |

The revert is relevant — it tells Architect that prior attempts to tweak overflow detection in `ShowMore` broke other cases. A safe fix should therefore not change `show_more.tsx`'s `checkTextOverflow` logic; instead it should resolve the clip-mechanism conflict in CSS or at the call-site (`post_message_preview.tsx`).

`post_message_preview.tsx` has no recent CSS/layout changes — latest edits are feature additions (autotranslation `1c7246da68`, AI-generated-post indicator `9ea080024b`, burn-on-read `084006c0ea`). None touched `maxHeight={105}` or `overflowType='ellipsis'`.

`_post.scss` — no recent modifications to lines 1860–1870 in the last ~180 days.

**Not a regression.** The defect has been present since the ellipsis variant was introduced; the reporter's specific markdown triggers it because headings mix line-heights.

## Phase D — Similar issues / PRs

`gh issue list` on the fork returned nothing relevant. Per the dispatch note, upstream searches are blocked; I did not probe them.

Local git-log scan for keywords (`line-clamp`, `preview overflow`, `mid-line`, `preview cut`) returned no matching fix commit on this branch — consistent with a latent defect, not a regression.

## Phase E — Hypothesis

**Root cause.** Link/permalink previews double-clip markdown content:

1. `PostMessagePreview` (`post_message_preview.tsx:188-194`) renders `PostMessageView` with `overflowType='ellipsis'` and `maxHeight={105}`.
2. `ShowMore.render` (`show_more.tsx:186-193`) applies inline `style={{maxHeight: 105}}` to `.post-message__text-container`, whose CSS (`_post.scss:1860-1862`) has `overflow: hidden`. This clips content at exactly 105 pixels, ignoring line boundaries.
3. `_post.scss:1864-1870` additionally clips the inner `.post-message__text` with `display:-webkit-box; -webkit-line-clamp: 4; text-overflow: ellipsis;`, which clips at 4 visual lines.

When the message contains mixed heading levels (the reporter's repro is `##`/`####`/`###`/`###`, giving ~28–40px per line depending on heading size), the 105px pixel clip lands **inside** a heading row that `-webkit-line-clamp: 4` still considers part of the visible line. The outer `overflow: hidden` then chops the heading horizontally, producing the mid-line cut. With plain paragraph text (uniform line-height ~20px) both mechanisms agree and the cut is clean — which is why the bug only "sometimes" manifests.

**Expected fix direction** (for Architect, not decided here):
- Prefer a single clip mechanism. Either (a) drop the inline `maxHeight` for the `ellipsis` variant and rely purely on `-webkit-line-clamp` (with the clamp value aligned to rendered content), or (b) drop `-webkit-line-clamp` and let the pixel clip be the sole mechanism with a line-height–aligned `maxHeight` (e.g. round `105` down to a multiple of the actual line-height, or compute it from measured content).
- The minimal-risk change is option (a): remove the inline `style={{maxHeight: collapsedMaxHeightStyle}}` when `overflowType === 'ellipsis'`, since the CSS line-clamp already handles the clip visually, and `ShowMore` only needs the pixel measurement to *detect* overflow (it can still compare `scrollHeight > maxHeight` via the ref without forcing the clip height onto the DOM).
- Keep the revert history in mind — changes to `show_more.tsx`'s overflow detection have caused regressions before. Prefer changing the CSS or the single call-site `post_message_preview.tsx`.

**Confidence:** 0.78 — root cause is identified with high specificity (exact files and lines), and the mechanism explains both "sometimes" frequency and the specific repro. Confidence held back from ≥0.85 because the fix has two reasonable shapes and there is no runtime evidence or prior matching fix PR to corroborate.

## Open questions (for Architect)

- Should the fix also address `restore_post_modal.tsx` (`maxHeight={100}`) and `message_attachment.tsx` / `embedded_binding.tsx` (`maxHeight={200}`) which use the same `ShowMore` + CSS pair? They don't have `overflowType='ellipsis'` (so the `-webkit-line-clamp` rule at `_post.scss:1864` doesn't apply), but the pixel-clip half is still present. Likely out of scope — preview is the only site with the double-clip.
- The 105px constant at `post_message_preview.tsx:191` is magic. Should it be derived from CSS (line-height × line-clamp)? Separately worth tidying.
- Should snapshot tests `post_message_preview.test.tsx.snap` be updated as part of the fix? They assert the rendered DOM including the inline `max-height`.

## Files the fix will touch

- `webapp/channels/src/components/post_view/show_more/show_more.tsx` — conditional on overflowType, gate the inline `maxHeight` style.

  *or*
- `webapp/channels/src/components/post_view/post_message_preview/post_message_preview.tsx` — adjust `maxHeight={105}` / stop passing it.

  *or*
- `webapp/channels/src/sass/components/_post.scss:1860-1870` — reconcile the two clip mechanisms (e.g. use only `-webkit-line-clamp` on the preview path).

Plus associated tests and snapshots:
- `webapp/channels/src/components/post_view/show_more/show_more.test.tsx` and its snapshot.
- `webapp/channels/src/components/post_view/post_message_preview/__snapshots__/post_message_preview.test.tsx.snap`.

---

```yaml
# machine-readable footer
affected_repos:
  - ShankHarinath/mattermost
confidence: 0.78
recommended_next_step: plan
rev: 1
risk: LOW
leak_signals: []
notes:
  - "Frontend-only bug. No runtime/log signal; Phase B skipped intentionally."
  - "Root cause is a double-clip between inline max-height (show_more.tsx) and -webkit-line-clamp (_post.scss). Fix is narrow; ShowMore blast radius is LOW (d=1 direct importers only)."
  - "Prior revert 332be84efd of MM-63648 indicates overflow-detection changes in show_more.tsx have regressed before — prefer CSS/call-site fix over editing show_more.tsx's detection logic."
  - "Issue breadcrumb NOT posted (disable_issue_comments=true per dispatch instruction)."
```
