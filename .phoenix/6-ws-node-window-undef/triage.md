# Triage Report — ShankHarinath/mattermost#6

## Summary
`@mattermost/client` package crashes immediately with `ReferenceError: window is not defined` whenever `WebSocketClient.initialize()` is invoked inside a Node.js process. The class unconditionally calls `window.addEventListener('online', …)` / `window.addEventListener('offline', …)` in `webapp/platform/client/src/websocket.ts` — a browser-only global that does not exist in Node.js. The package README (`webapp/platform/client/README.md`) still advertises Node.js as a supported runtime (with a user-supplied `globalThis.WebSocket` polyfill), so consumers are steered into a code path that guarantees a crash on every `initialize` call. Regression was introduced by commit `67ab69606a` (PR #30788, "Handle network connectivity changes in websocket", 2025-05-09) which added online/offline reconnect handlers with no environment guard.

## Classification
- **Type:** Regression / environment-compatibility bug in a published client SDK
- **Severity:** High — 100% reproducible, first-call crash, breaks every Node.js consumer of `@mattermost/client` versions > 10.8.0
- **Surface:** Published npm package `@mattermost/client` (currently `11.4.0` per `webapp/platform/client/package.json`)

## Phase A — Repo selection and symptom → code mapping

### A.1 Search radius
- Issue explicitly names `@mattermost/client` and compiled path `node_modules/@mattermost/client/lib/websocket.js` → source lives in the mattermost monorepo at `webapp/platform/client/src/websocket.ts`.
- Only one relevant indexed repo: `mattermost` (`/Users/shashank/Canonix/phoenix-test/mattermost`, 57,846 nodes, 191,892 edges, indexed 2026-04-20).
- No group membership, no cross-repo contracts in play — the bug is self-contained inside the client package of the mattermost monorepo.

### A.2 Symbol resolution (GitNexus)
| Signal | Resolved to | File |
| --- | --- | --- |
| `WebSocketClient` | `Class:webapp/platform/client/src/websocket.ts:WebSocketClient` (lines 37–677) | `webapp/platform/client/src/websocket.ts` |
| `WebSocketClient.initialize` | `Method:webapp/platform/client/src/websocket.ts:WebSocketClient.initialize#3` (lines 124–424) | `webapp/platform/client/src/websocket.ts` |
| `window.addEventListener` / `this.onlineHandler` | Lines 152–156, 202–203, 561–567 in the same file | `webapp/platform/client/src/websocket.ts` |

### A.3 Exact offending code (`webapp/platform/client/src/websocket.ts`)
```ts
// lines 150-157 (inside initialize, before handler setup)
if (this.onlineHandler) {
    window.removeEventListener('online', this.onlineHandler);
}
if (this.offlineHandler) {
    window.removeEventListener('offline', this.offlineHandler);
}
…
// lines 202-203 (registration — crashes on first call in Node)
window.addEventListener('online', this.onlineHandler);
window.addEventListener('offline', this.offlineHandler);
…
// lines 561-567 (inside close(), same vulnerability on teardown)
if (this.onlineHandler) {
    window.removeEventListener('online', this.onlineHandler);
    this.onlineHandler = null;
}
if (this.offlineHandler) {
    window.removeEventListener('offline', this.offlineHandler);
    this.offlineHandler = null;
}
```

Stack trace `node_modules/@mattermost/client/lib/websocket.js:147` (compiled) maps 1:1 onto source line 202 — the first `window.addEventListener('online', …)` — which matches the reporter's trace exactly.

### A.4 Affected repos
Only `ShankHarinath/mattermost` requires a code change. The surface area of the fix is a single source file (`webapp/platform/client/src/websocket.ts`) plus optionally the existing test harness in `webapp/platform/client/src/websocket.test.ts` (which already stubs `window` for jest — it should also gain a Node-runtime test that runs the real, unpatched `initialize`).

## Phase B — Runtime evidence (kubectl)
Intentionally skipped. The defect surfaces in the consumer's Node.js process (the published npm package), not in a Mattermost server container. There is no server log to correlate against; the failure is deterministic at module entry. Recorded explicitly: no kubectl/pod evidence was gathered because none exists for this client-side regression.

## Phase C — Recent commits on affected file
`git log -- webapp/platform/client/src/websocket.ts` (full history on this file, newest first):

| SHA | Date | Subject | Relevance |
| --- | --- | --- | --- |
| `777867dc36` | — | Define types for WebSocket messages and migrate WebSocket actions to TS (#34603) | Unrelated refactor; did not touch `window` usage |
| `761584c040` | — | [MM-64244] Add websocket disconnect reason metric (#31032) | Unrelated |
| **`67ab69606a`** | **2025-05-09** | **Handle network connectivity changes in websocket (#30788)** | **Root-cause commit — introduced `window.addEventListener('online'/'offline', …)` and the `onlineHandler`/`offlineHandler` properties in `initialize()` and `close()`. +199/-63 lines in `websocket.ts`.** |
| `88d5ad06b4` | — | [MM-63583] Send websocket ping immediately after connecting (#30579) | Pre-regression |
| `394ee75889` | — | Add WebSocket client ping implementation (#30293) | Pre-regression |

Author of regression: David Krauser (`david@krauser.org`). Commit message explicitly frames the change as: *"introduces listeners for network changes that will: - Test the websocket if we get an offline event … - Re-connect the websocket immediately if we get an online event"*. No mention of Node.js compatibility; no environment guard was added. The commit also touched `websocket.test.ts` (+207 lines) to add `if (typeof window === 'undefined') { (global as any).window = { addEventListener: jest.fn(...), … } }` (`websocket.test.ts` lines 13–39) — this is direct evidence that the author was aware `window` is not defined outside the browser, but the guard was only placed in the test harness, never in production code.

Reporter's claim — "broken in @mattermost/client versions after 10.8.0" — is consistent: `67ab69606a` is the first commit that introduced a `window` reference on the hot path of `initialize`.

## Phase D — Similar past issues / PRs
- `gh issue list --repo ShankHarinath/mattermost --search "window is not defined" --state all` returns only issue #6 itself. No earlier related reports in the fork.
- Upstream `mattermost/mattermost` access is denied by the counterfactual gate, so no upstream issue/PR search was performed.
- Fix pattern inside the same package (`webapp/platform/client/src/websocket.test.ts:14`) already uses the canonical guard:
  ```ts
  if (typeof window === 'undefined') { … }
  ```
  — this is the idiomatic shape any fix should adopt inside `initialize()` and `close()`.
- `webapp/platform/client/README.md` lines 88–111 document Node.js as an officially supported runtime that requires the user to polyfill `globalThis.WebSocket`. The README makes no mention of needing to polyfill `window`, confirming the contract is that production code must tolerate `window` being absent.

## Phase E — Hypothesis

**Root cause.** `WebSocketClient.initialize()` and `WebSocketClient.close()` in `webapp/platform/client/src/websocket.ts` unconditionally reference the browser-only global `window` to register/unregister `online`/`offline` network-status listeners. In any Node.js host (including users following the package's own README guidance), `window` is `undefined`, so the very first call to `initialize()` throws `ReferenceError: window is not defined` before the WebSocket is ever opened. PR #30788 introduced this behaviour on 2025-05-09 and — even though the same PR had to add a `window` mock to the jest test file — never added an equivalent runtime guard in production code. Published `@mattermost/client` packages from the next release forward ship the defect; `10.8.0` was the last safe version.

**Fix direction (for Architect — do not over-specify here).** Guard every `window`/`addEventListener`/`removeEventListener` touchpoint with a runtime feature check (e.g., `typeof window !== 'undefined' && typeof window.addEventListener === 'function'`) at:
- `webapp/platform/client/src/websocket.ts:152-156` (remove-before-register block in `initialize`)
- `webapp/platform/client/src/websocket.ts:202-203` (register block in `initialize`)
- `webapp/platform/client/src/websocket.ts:561-567` (remove block in `close()`)

When `window` is absent, skip listener registration entirely (reconnect logic still works via the existing ping/onclose paths — the online/offline handlers are purely a browser-only optimization). A targeted Node-runtime unit test that does NOT monkey-patch `window` should be added to prevent regression; `websocket.test.ts` currently hides the defect by pre-installing a `window` stub, which is why the regression escaped CI.

**Confidence:** `0.95`.
Justification: every pre-extracted signal mapped to exact source lines; stack-trace line number (compiled `147`) aligns with TS source line 202; regression commit is explicitly the one named in the issue (PR #30788); the in-repo test-file pattern provides the canonical fix shape; the README documents Node.js as a supported target so a guard is clearly the correct change (not removal of the feature). The only reason this is not 1.00 is that I have not executed the repro against a local install to visually confirm the exception trace — but given the code path is unconditional, a live repro cannot produce a different outcome.

## Open questions (non-blocking)
1. Should the fix also emit a one-time `console.debug` note when `window` is absent, so users debugging in Node can see that the network-event reconnect optimisation is intentionally skipped? (Cosmetic — Architect decides.)
2. Does Mattermost want to additionally add a `typeof addEventListener !== 'undefined'` check on `globalThis` (covers Web Workers / Deno / Bun), or is `window`-only acceptable? The README only mentions Node.js; browser and Node coverage is sufficient for the reported bug.
3. Should `webapp/platform/client/setup_jest.ts` (currently only polyfills `fetch`) take over the `window` mock, and should a second jest project run without any `window` polyfill to lock in Node-safety as a CI invariant?

## Recommended next step
Architect should design a minimal environment-guard patch to `webapp/platform/client/src/websocket.ts` covering the three line-ranges above, plus a companion Node-runtime test that exercises `new WebSocketClient().initialize(url, token)` without any `window` polyfill and asserts it does not throw. Blast radius is contained: GitNexus `context` on `WebSocketClient.initialize` shows only one external caller (`webapp/channels/src/actions/websocket_actions.ts:initialize`, which always runs in the browser and is therefore unaffected) plus internal self-recursion and tests — d=1 upstream risk is LOW.

---

```yaml
# phoenix-triage-footer
issue: ShankHarinath/mattermost#6
rev: 1
affected_repos:
  - ShankHarinath/mattermost
confidence: 0.95
recommended_next_step: architect-plan
primary_file: webapp/platform/client/src/websocket.ts
primary_symbols:
  - WebSocketClient.initialize
  - WebSocketClient.close
regression_commit: 67ab69606a2418fc2da510c2036a24ba942456c2
regression_pr: 30788
```
