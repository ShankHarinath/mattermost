# Phoenix Triage Report — ShankHarinath/mattermost#4

## Summary
Interactive Dialog's dynamic-select field re-issues its options-lookup call every time **any** other field in the modal changes (text, textarea, radio, bool, static select). Expected behavior: the lookup should only run when the dynamic-select itself changes or when a `refresh:true` field is modified and the server explicitly requests updated options. Root cause is on the client in `AppsFormSelectField`, which force-remounts its `react-select/async` component whenever it receives a new `field` prop reference — and `AppsForm.renderElements` produces a new `field` prop object on every render via `createSanitizedField`.

## Classification
- Type: bug (front-end React component behavior)
- Severity: medium–high (each keystroke in a dialog can fan out a server lookup; easy DoS against plugin/integration endpoints, bad UX)
- Area: `webapp/channels/src/components/apps_form` + dialog adapter
- Confidence: 0.88

## Phase A — Code map

Searched repo: `mattermost` (61,147 nodes, indexed at `6878d09547…`, pre-fix SHA). This is a client-only bug — the Go server lookup handler (`server/channels/app/integration_action.go:App.LookupInteractiveDialog`, `server/channels/api4/integration_action.go:lookupDialog`) is a neutral passthrough; nothing server-side decides when to fire. No other indexed repo is relevant (no webhook/BFF involvement).

Request flow (top-down):
1. `InteractiveDialogAdapter` (`webapp/channels/src/components/dialog_router/interactive_dialog_adapter.tsx:92`) converts legacy `DialogElement[]` → `AppForm` via `convertDialogToAppForm` and renders `<AppsFormContainer>` with `doAppFetchForm=refreshOnSelect`, `doAppLookup=performLookupCall`.
2. `convertElement` (`webapp/channels/src/utils/dialog_conversion.ts:356`) correctly copies `element.refresh` to `appField.refresh` **only** for SELECT/RADIO (lines 420–440), and sets `appField.lookup = { path: data_source_url }` for dynamic selects (line 429–434). For the reporter's payload this means only `category` has `field.refresh=true`, and only `category` has `field.lookup` → only `category` renders `renderDynamicSelect`.
3. `AppsForm.onChange` (`webapp/channels/src/components/apps_form/apps_form_component.tsx:485`) updates `state.values[name]` via `setState` and fires `refreshOnSelect` **only** when `field.refresh` is true. For text/radio/bool edits the refresh path never runs — so the server call that the reporter sees is **not** the "refresh" call.
4. `renderElements` (`apps_form_component.tsx:659–677`) wraps each field in `createSanitizedField(originalField)` (line 156, introduced by PR #33288) which does `{...field}` and returns a brand-new object on every single render.
5. `AppsFormField` (PureComponent, `apps_form_field.tsx:48`) passes `field` down to `AppsFormSelectField` (PureComponent, `apps_form_select_field.tsx:66`).
6. **`AppsFormSelectField.getDerivedStateFromProps`** (`apps_form_select_field.tsx:75–84`):
   ```ts
   if (nextProps.field !== prevState.field) {
       return {
           field: nextProps.field,
           refreshNonce: Math.random().toString(),
       };
   }
   ```
   The reference comparison fails every render because step 4 minted a new object. `refreshNonce` rotates, and the `<React.Fragment key={this.state.refreshNonce}>` wrapping `AsyncSelect` (lines 238–243) forces React to unmount and remount `AsyncSelect` on every keystroke.
7. `AsyncSelect` is configured with `defaultOptions={true}` (line 124). Per `react-select/async`, this causes `loadOptions` to be invoked on mount. Each remount therefore calls `loadDynamicOptions` → `performLookup` (`apps_form_component.tsx:397`) → `AppsFormContainer.performLookupCall` (`apps_form_container.tsx:168`) → `InteractiveDialogAdapter.performLookupCall` (`interactive_dialog_adapter.tsx:390`) → `actions.lookupInteractiveDialog(...)` → server `POST /api/v4/actions/dialogs/lookup`.

So the "lookup on every field change" is the dynamic-select's options-prefetch firing because the select gets remounted, not because the refresh/lookup pathway thinks it should fire.

**Affected repo set (for edits):** only `ShankHarinath/mattermost`. Fix is purely front-end.

## Phase B — Runtime evidence (kubectl)
Skipped — this is a deterministic React component-lifecycle bug reproducible from the reporter's JSON payload in dev mode. No cluster logs requested or needed. Anyone reproducing per the issue steps will see one `POST /api/v4/actions/dialogs/lookup` per keystroke in every non-lookup field.

## Phase C — Recent commits on the suspect files (`master`, within fork)

Files touched:
- `webapp/channels/src/components/apps_form/apps_form_component.tsx` — `createSanitizedField` added by `5a0dee5fc2` ("Add date and datetime field support for AppsForm", PR #33288). Wraps every field in a fresh object on every render; last touched by `656a0248eb` ("Fix datetime MinDate/MaxDate validation…", PR #35327) with no behavioral change to the spread pattern.
- `webapp/channels/src/components/apps_form/apps_form_field/apps_form_select_field.tsx` — `getDerivedStateFromProps` + `refreshNonce` logic is old (predates the monorepo merge at `c943ed6859`). It originally existed to allow a parent to force a lookup refetch by passing a new field reference; the safety net becomes a liability once something upstream starts minting new references every render.
- `webapp/channels/src/components/dialog_router/interactive_dialog_adapter.tsx` — introduced by `4ee339f43b` ("Update Interactive Dialog to use AppsForm", PR #31821). Dynamic-select support for legacy Interactive Dialog added in `abe8151bad` ("Add Dynamic Select for Interactive Dialog", PR #33586). These two together exposed legacy dialogs (which did NOT previously call any lookup on text changes) to the remount-on-prop-reference behavior.
- `webapp/channels/src/utils/dialog_conversion.ts` — conversion correctness looks OK: `field.refresh` is only copied on SELECT/RADIO. No smoking gun here.

Plausible regressing change (highest correlation): **PR #31821 `4ee339f43b`** "Update Interactive Dialog to use AppsForm" — moving the legacy code path onto AppsForm exposes it to the pre-existing remount anti-pattern. The `createSanitizedField` spread in PR #33288 (`5a0dee5fc2`) converted the anti-pattern from "sometimes triggers" to "triggers on every render", making the bug observable in practice as described.

## Phase D — Similar past issues / PRs (fork only)
Only three other open fork issues (#1, #6, #8) — unrelated (page-up/down, WebSocket/Node client, TXT preview). No prior triage precedent to copy.

## Phase E — Hypothesis & recommended fix

**Hypothesis (confidence 0.88):** `AppsForm.renderElements` calls `createSanitizedField(originalField)` on every render, returning a new `AppField` object each time. Because `AppsFormSelectField.getDerivedStateFromProps` treats any new `field` reference as "something materially changed, reload options", it rotates `refreshNonce`, which changes the Fragment `key` wrapping `AsyncSelect`, forcing an unmount/remount. `AsyncSelect` with `defaultOptions={true}` calls `loadOptions` on mount, producing one lookup request per render — i.e., per keystroke/field change anywhere in the form.

**Recommended fixes (pick one or combine):**

1. **Stop comparing by reference** in `apps_form_select_field.tsx`. The state-derivation should only force remount when something the `AsyncSelect` actually depends on changes — e.g. `field.name`, `field.lookup?.path`, `field.readonly`, `field.multiselect`, `field.hint`. Shallow-compare those keys (or drop the remount-on-prop-change entirely and rely on React's internal reconciliation).
2. **Stabilize the field reference** in `apps_form_component.tsx:renderElements`. Either memoize sanitized fields (e.g. `useMemo`/`shouldUpdate` keyed off the original field) or avoid wrapping non-DATE/DATETIME fields with `createSanitizedField` at all. The TEXT/SELECT/RADIO/BOOL branches don't need sanitation, so the spread is pure churn for them.
3. **Remove `defaultOptions={true}`** on the dynamic-select path and/or debounce `loadDynamicOptions`. Weaker fix — it only reduces request volume, doesn't stop the unnecessary remount.

Fix #1 is the most targeted and keeps existing force-refresh semantics (when the upstream genuinely hands a new field definition, e.g. after a server-driven form refresh sets new options). Fix #2 complements it by eliminating needless work on the hot render path.

## Open questions
- Does the bug also repro outside legacy Interactive Dialogs (e.g. native Apps forms)? The `AppsFormSelectField` pattern is shared, but in native `AppsForm` usage the parent usually doesn't spread fields each render, so it may only be observable via the adapter path. Worth confirming with a quick repro once patched.
- Whether any tests cover "lookup is NOT called on unrelated field changes" — a quick grep shows `apps_form_component.test.tsx` has `refresh: true` scenarios but no "text change does not trigger lookup" assertion. Suggest adding one.

## Proposed next step
Architect drafts fix targeting `webapp/channels/src/components/apps_form/apps_form_field/apps_form_select_field.tsx` (change `getDerivedStateFromProps` to compare relevant field keys, not object identity) and optionally `apps_form_component.tsx:renderElements` (skip or memoize `createSanitizedField`). Add a regression test in `apps_form_component.test.tsx` asserting that a text-field keystroke does not invoke `performLookupCall`. No server changes required.

---

```yaml
# machine-readable triage footer
affected_repos:
  - ShankHarinath/mattermost
confidence: 0.88
recommended_next_step: plan
```
