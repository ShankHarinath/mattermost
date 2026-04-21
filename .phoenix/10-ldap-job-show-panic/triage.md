# Triage Report — ShankHarinath/mattermost #10

## Summary

`mmctl ldap job show` (no args) panics with `runtime error: index out of range [0] with length 0` at `server/cmd/mmctl/commands/ldap.go:143`. Root cause: the `LdapJobShowCmd` cobra command is missing an `Args: cobra.ExactArgs(1)` validator, so cobra does not reject arg-less invocations and `ldapJobShowCmdF` blindly dereferences `args[0]`. Every sibling `*JobShowCmd` (extract, import, export) has the validator — this one is a copy-paste omission from the feature's original commit (`3dae305dc7`, PR #25633, April 2024).

## Phase A — Code map

**Search radius considered.** 11 repos indexed. The issue is scoped to a single CLI binary (`mmctl`) that lives inside the `mattermost` monorepo at `server/cmd/mmctl/`. No API contract, no cross-service hop, no group membership matters here — other indexed repos (canonix-engineering/app, cortex, etc., erpnext, twenty) are unrelated products. **Chosen radius: `mattermost` only.**

**Symbol resolution.**

| Signal | Where it resolved |
|---|---|
| `ldapJobShowCmdF` (panic site) | `server/cmd/mmctl/commands/ldap.go:142-151` — `ldapJobShowCmdF(c, command, args)` does `c.GetJob(context.TODO(), args[0])` on line 143 with no length check |
| `LdapJobShowCmd` (cobra registration) | `server/cmd/mmctl/commands/ldap.go:72-78` — **no `Args:` field set** |
| `printPanic` | `server/cmd/mmctl/commands/root.go` (panic formatter, added in commit `213ebc57fb`, PR #27390 — not the bug, just the recovery glue that printed the stack) |
| `withClient.func122` | Generated wrapper in `init.go:112` that invokes the per-command `*CmdF` function |
| Siblings with correct pattern | `ExtractJobShowCmd` (`extract.go:50`), `ImportJobShowCmd` (`import.go:89`), `ExportJobShowCmd` (`export.go:85`) — all three set `Args: cobra.ExactArgs(1)` |

**Impact analysis.** `gitnexus_impact(ldapJobShowCmdF, upstream)` → risk LOW, impactedCount 0 (no production callers — it is wired only through cobra's `RunE`, plus unit and e2e tests). Safe to change.

**Affected repo set.** Only `ShankHarinath/mattermost`.

## Phase B — Runtime evidence

Not applicable: `mmctl` is a local CLI binary, not a cluster workload. The panic stack trace embedded in the issue body is itself the runtime evidence. `kubectl` against `kind-canonix` would not surface this failure mode.

- Runtime signal: stack trace shows recovery at `commands/root.go:85` (`printPanic`) after panic raised from `ldap.go:143` inside `ldapJobShowCmdF`. Cobra `command.execute` at `cobra@v1.8.1/command.go:985` invoked `RunE` without prior arg-count validation — consistent with `LdapJobShowCmd.Args` being unset.
- Frequency: 100% reproducible (`mmctl ldap job show` without an ID panics every invocation).

## Phase C — Recent commits on the affected file

`git log server/cmd/mmctl/commands/ldap.go`:

| SHA | Date | Subject | Relevance |
|---|---|---|---|
| `3dae305dc7` | 2024-04-22 | [MM-56000] Add LDAP job command to mmctl (#25633) | **Root-cause commit.** Diff added `LdapJobShowCmd` and `ldapJobShowCmdF` but omitted `Args: cobra.ExactArgs(1)` that the sibling commands use. High correlation. |
| `9187c772b6` | — | [MM-56074] mmctl job commands (#26855) | Touched unified job-command code; did not retrofit the missing validator on `LdapJobShowCmd`. |
| `c2d08b7540` | — | [MM-63772] Add LDAP setting to re-add removed members (#30787) | Unrelated — changed `ldapSyncCmdF` flag handling. |
| `d316df6d28` | — | Replacing interface{} with any everywhere (#29446) | Unrelated housekeeping. |

## Phase D — Similar past issues / PRs

- **Upstream `mattermost/mattermost#31894` — closed, same title: "[mmctl] [bug] panic on v10.9.1".** Issue-search API surfaced this exact match. It is the upstream counterpart of the filed fork issue and confirms the fix pattern (direct access to the upstream issue body is blocked by the harness allowlist, but the title match plus the fork repo being a vendor copy of `mattermost/mattermost` is enough to be confident the upstream resolution is the same class of fix).
- **`213ebc57fb` — "Print panic message when mmctl panics" (#27390).** Added the `printPanic` recovery path visible in the stack trace. Not the bug; it is why the panic produced a readable stack instead of a bare crash.
- No prior PR on the indexed history has already added `ExactArgs(1)` to `LdapJobShowCmd` — the bug is live at HEAD (`10b67849686e`).

## Phase E — Hypothesis and recommended fix

**Hypothesis.** `LdapJobShowCmd` (declared in `server/cmd/mmctl/commands/ldap.go:72-78`) is missing `Args: cobra.ExactArgs(1)`. Because cobra only enforces arg counts when that field is set, running `mmctl ldap job show` with no positional argument skips validation and dispatches into `ldapJobShowCmdF`, whose first line (`server/cmd/mmctl/commands/ldap.go:143`) indexes `args[0]` unconditionally, producing the reported panic.

**Recommended fix.** Add the validator on the command declaration, mirroring the three sibling commands:

```go
var LdapJobShowCmd = &cobra.Command{
    Use:               "show [ldapJobID]",
    Example:           " import ldap show f3d68qkkm7n8xgsfxwuo498rah",
    Short:             "Show LDAP sync job",
    Args:              cobra.ExactArgs(1),                     // <-- add
    ValidArgsFunction: validateArgsWithClient(ldapJobShowCompletionF),
    RunE:              withClient(ldapJobShowCmdF),
}
```

Additionally, while the example string says "`import ldap show ...`" which is wrong — it should read "`ldap job show ...`" — that is a cosmetic docstring nit worth fixing in the same change but is not load-bearing for the panic.

**Test.** `server/cmd/mmctl/commands/ldap_test.go:180` (`TestLdapJobShowCmdF`) already exists for the happy path; after the fix, cobra will short-circuit the no-arg invocation with `Error: accepts 1 arg(s), received 0` before `RunE` is called, so no panic recovery test is strictly required, but adding a unit assertion that the command declaration sets `ExactArgs(1)` would prevent regression.

**Confidence.** 0.92 — very high. The evidence chain is airtight: the panic line, the missing declaration field, the sibling pattern, the introducing commit, and an upstream closed issue with identical title all corroborate. The only minor uncertainty is whether upstream chose the `ExactArgs(1)` patch or in-function guard — both correct the bug; the declaration-level fix is idiomatic in this codebase.

## Open questions

- Upstream `mattermost/mattermost#31894` is blocked by the `gh-shim` allowlist, so I could not read its resolution PR to confirm the exact patch. Architect or human reviewer can cross-check when lifting the allowlist is acceptable.
- No `kubectl` verification performed — mmctl is a CLI tool, not a cluster workload. The stack trace is the runtime evidence.

---

<!--phoenix:scout-summary
affected_repos:
  - ShankHarinath/mattermost
confidence: 0.92
recommended_next_step: proceed
root_cause_symbol: LdapJobShowCmd
root_cause_file: server/cmd/mmctl/commands/ldap.go
root_cause_line: 72
fix_summary: Add `Args: cobra.ExactArgs(1)` to the LdapJobShowCmd cobra.Command declaration in server/cmd/mmctl/commands/ldap.go so cobra rejects empty-arg invocations before `ldapJobShowCmdF` dereferences args[0].
risk: LOW
blast_radius_direct_callers: 0
related_upstream_issue: mattermost/mattermost#31894
introducing_commit: 3dae305dc7
-->
