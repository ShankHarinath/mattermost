# Triage Report — ShankHarinath/mattermost#8

## Summary
- **Issue:** [Bug]: No preview for TXT files — Mattermost Server 11.0.2, Windows 10. Clicking to preview any `.txt` file in chat renders an endless loading spinner; download works fine.
- **Root cause (hypothesis):** The August 2025 refactor that migrated `CodePreview` from a class component to a function component (`5eaa22525c`, PR #33302) introduced a useEffect race. The fetch-gate effect captures `shouldNotGetCode=true` from the initial render (because `codeInfo.lang` is `''` on mount), so the fetch is short-circuited on the only render where the `fileUrl !== prevFileUrl` gate is open. On the subsequent render the language is set (`'text'`), but by then `prevFileUrl === fileUrl` so the fetch is never attempted. `status` stays `'loading'` forever — permanent spinner.
- **Confidence:** 0.82

## Affected repos
- `ShankHarinath/mattermost` — webapp frontend.

## Evidence

### A. Code path mapped (GitNexus + Read)
The click-to-preview path for a TXT file in the post list is:

1. `FilePreviewModal.render` — `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/file_preview_modal/file_preview_modal.tsx:339-398`. Dispatches on file type; for TXT, fileType is `FileTypes.TEXT` (via `TEXT_TYPES: ['txt','rtf','vtt']` at `webapp/channels/src/utils/constants.tsx:1554`). TEXT falls through Image/Video/Audio/PDF checks and hits `hasSupportedLanguage(fileInfo)` on line 380.
2. `hasSupportedLanguage` — `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/code_preview.tsx:23-25`. For `.txt`, it calls `SyntaxHighlighting.getLanguageFromFileExtension('txt')`, which finds `text: {name:'Text', extensions:['txt','log'], aliases:['txt']}` at `webapp/channels/src/utils/constants.tsx:2003` and returns `'text'` — truthy, so `CodePreview` is chosen.
3. `CodePreview` — `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/code_preview.tsx:27-141`. This is where the render hangs.

### B. The race condition (the bug)
`CodePreview` has two `useEffect`s:

- **Effect A (lines 41-61)** — resolves language from extension, sets `codeInfo.lang = 'text'`, sets `status='loading'`, and sets `prevFileUrl=fileUrl`. Guard: `fileUrl !== prevFileUrl`.
- **Effect B (lines 65-105)** — fetches the file and calls `setStatus('success')` on completion. Guard: `fileUrl !== prevFileUrl`. Body contains an early-return: `if (shouldNotGetCode) return;` where `shouldNotGetCode = !codeInfo.lang || fileInfo.size > MAX` (line 63).

**On first mount, both effects run after the same initial render and therefore share the same captured closure values:** `codeInfo.lang === ''`, `prevFileUrl === undefined`.

- In Effect B's closure: `shouldNotGetCode = !'' = true` — `getCode()` returns immediately without calling `fetch`.
- State updates from Effect A commit (`lang='text'`, `prevFileUrl=fileUrl`, `status='loading'`). React re-renders.
- Effect B re-runs because `codeInfo` changed. New closure: `shouldNotGetCode=false` (good), but now `fileUrl === prevFileUrl`, so the `if (fileUrl !== prevFileUrl)` gate blocks `getCode()`.
- Net result: `getCode()` is never called with both gates open. `status` stays `'loading'` → infinite `<LoadingSpinner/>`.

This matches the reporter's "endless loop" symptom exactly and explains why Download still works (download uses a plain `<a href>` bypassing CodePreview).

### C. Why the previous class version worked (regression origin)
In the original class component (pre-commit `5eaa22525c`), ordering was deterministic:
- `getDerivedStateFromProps` ran before render and set `state.lang = 'text'`.
- `componentDidMount` fired `getCode()` with `this.state.lang` already populated, so the `if (!this.state.lang...)` gate was open.

The function-component rewrite moved the lang-derivation into a post-render effect, creating a 1-frame window where the fetch effect sees the empty lang and bails out. The PR description — "refactor(code_preview): move state updates into useEffect" and "refactor(code_preview): remove dependency from useEffect" — corresponds directly to where the race was introduced.

### D. Git correlation
- `5eaa22525c feat(code_preview): migrate CodePreview to a function component (#33302)` (Aug 27, 2025) is the only substantive change to `code_preview.tsx` in recent history — prior commits are only lint/import rearrangements.
- Mattermost Server 11.0.2 (per issue body) ships with this commit in its webapp bundle; Windows 10 / OS is irrelevant — the bug is in the React render graph.

### E. Runtime confirmation
Skipped per triage directive (counterfactual/offline evaluation — no running cluster for this repo). The code-level trace is fully sufficient to explain the reported symptom.

### F. Similar past issues / PRs
- `gh search` and upstream `mattermost/mattermost` access are shimmed (denied) for this counterfactual run — noted, did not retry.
- Fork `ShankHarinath/mattermost`: no existing PR or issue addresses code_preview (only issues #1, #4, #6, #8; no prior fix branch for #8).

## Prime suspect file(s)
- `/Users/shashank/Canonix/phoenix-test/mattermost/webapp/channels/src/components/code_preview.tsx` — lines 41-105. The fix needs to ensure the fetch runs after the language-derivation effect has committed (e.g., collapse the two effects into one, or fetch inside the first effect, or gate fetch on `codeInfo.lang` rather than a stale `shouldNotGetCode`).

## Suggested fix direction (for Architect)
Options, in increasing order of invasiveness:

1. **Drop the `shouldNotGetCode` early-return race** — Inside `getCode`, re-read the gate from fresh state, or pass `usedLanguage` directly. Simplest: move `getCode()` into Effect A so it runs inside the same tick that derives `lang`, eliminating the cross-effect closure hazard.
2. **Collapse both effects into one** — A single effect keyed on `[fileUrl, fileInfo.extension, fileInfo.size]` that both resolves `lang` and fetches. Mirrors the original class-component ordering. This is almost certainly what the PR author intended.
3. **Use a ref for `prevFileUrl`** — not sufficient on its own, because the dual-effect race still leaks.

Architect should verify the snapshot test `webapp/channels/src/components/file_preview_modal/__snapshots__/file_preview_modal.test.tsx.snap` (touched in the same migration PR) still passes and add a regression test that asserts `status` transitions `'loading' → 'success'` for a `.txt` fileInfo.

## Open questions
- Does the bug reproduce for every language CodePreview handles (`.js`, `.py`, `.json`, ...), or only for `text`-language files? Root-cause analysis says all, but only TXT is reported. An Architect could verify with a reproduction.
- Is any additional call site besides `FilePreviewModal` rendering `CodePreview`? Grep shows `code_preview` imported only from `file_preview_modal.tsx` in the component tree (plus its own test file), so blast radius is limited to the file-preview modal.
- The `gh` shim denial for upstream mattermost/mattermost blocked any check for an existing upstream fix; noted, not blocking.

## Confidence rationale
- Static analysis pinpoints a concrete, reproducible race in `code_preview.tsx` that matches the exact symptom ("endless loop" = stuck spinner).
- Single recent commit (`5eaa22525c`) cleanly accounts for the regression — pre-bug class version did not have this problem.
- Downgraded from 0.9 to 0.82 only because runtime repro/log confirmation was skipped per directive, and I couldn't cross-check an upstream fix PR.

---

```yaml
issue: 8
source_repo: ShankHarinath/mattermost
affected_repos:
  - ShankHarinath/mattermost
confidence: 0.82
recommended_next_step: plan
prime_suspect_files:
  - webapp/channels/src/components/code_preview.tsx
prime_suspect_symbols:
  - CodePreview
suspect_commits:
  - 5eaa22525c
runtime_evidence: skipped_counterfactual_offline
```
