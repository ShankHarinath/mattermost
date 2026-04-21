# Triage Report — Issue #15: [Bug]: Profile Picture Orientation

**Repo:** ShankHarinath/mattermost
**Issue:** https://github.com/ShankHarinath/mattermost/issues/15
**Reporter:** ShankHarinath
**Created:** 2026-04-21T10:03:01Z
**Rev:** 1

## TL;DR

Uploaded profile pictures from iOS rotate 90° after save because `App.AdjustImage` (at `server/channels/app/user.go:1022-1046`) calls `imgDecoder.Decode(file)` — which consumes the `io.ReadSeeker` to EOF — and then calls `imaging.GetImageOrientation(file, format)` on the already-consumed reader **without seeking back to offset 0**. The EXIF read fails, `GetImageOrientation` returns `Upright` (1), `MakeImageUpright` no-ops, and the raw rotated iOS pixel buffer is persisted unrotated. The regression was introduced in commit `f8e16780ef` (PR MM-63436, "Replace Exif parser dependency", 2025-04-01). Confidence 0.90. Fix: insert `file.Seek(0, io.SeekStart)` between the decode and the orientation read (and mirror in `SetTeamIconFromFile`, which has the same defect).

## Phase A — Search radius and symptom-to-code mapping

### A.1 Search radius

- `list_repos`: 11 repos indexed; the issue is on `ShankHarinath/mattermost`, indexed at 2026-04-21T10:33 against `ad03248cd3fb2b2ce4568dbc49f2c6db10f0b520` — fresh.
- `group_list`: groups `backend`, `frontend`, `infra` exist; mattermost is not named in issue routing.
- Decision: **scope limited to the `mattermost` repo**. The bug is a server-side image-processing regression; Mattermost is a monorepo (Go server + React webapp + Go image utilities all in the same repo). No cross-repo links plausible.

### A.2 Queries

- `query "profile picture upload user image"` → pointed at `server/channels/app/user.go`, `server/channels/app/users/profile_picture.go`, `api/v4/source/uploads.yaml`.
- `query "EXIF orientation rotate image"` → surfaced the prime suspects: `server/channels/app/imaging/orientation.go` (`MakeImageUpright`, `GetImageOrientation`, `fwSeeker`), plus the test file.
- `context "MakeImageUpright"` → callers: `App.AdjustImage` (profile), `App.SetTeamIconFromFile` (team icon), `UploadFileTask.postprocessImage` (generic uploads), `prepareImage` (thumbnails/previews).
- `context "AdjustImage"` → outgoing calls include `imgDecoder.Decode`, `GetImageOrientation`, `MakeImageUpright`, `FillCenter`, `EncodePNG`. Only inbound caller is the test `TestAdjustProfileImage` — used by `SetProfileImageFromFile` (resolved via file inspection, not in context edges).
- `impact "AdjustImage" upstream` → LOW risk, 0 indirect callers beyond the test.

### A.3 Interpretation of misses

None — the signal set is narrow (no stack trace), but GitNexus hit the exact suspect symbol on the first vector search.

### A.4 Affected repos

Only `ShankHarinath/mattermost` needs a code change.

## Phase B — Runtime evidence (kubectl)

**Blocker:** `kubectl --context kind-canonix get pods` fails — `connection to 127.0.0.1:59249 refused`. The local kind cluster is not up in this session, so no runtime log/event evidence was collected. The static evidence is conclusive regardless, but if the orchestrator wants runtime confirmation, re-run after bringing up `kind-canonix`.

## Phase C — Recent commits on affected files

- `server/channels/app/imaging/orientation.go` last changed in:
  - **`f8e16780ef` — [MM-63436] Replace Exif parser dependency (2025-04-01, Claudio Costa)** — this is the regression-introducing commit.
  - `316cde2569` — Remove `disintegration/imaging` dependency (earlier).
- `server/channels/app/user.go` — `f8e16780ef` is in its recent history and touches `AdjustImage` exactly.

### What `f8e16780ef` changed in `AdjustImage` (diff excerpt)

```go
-func (a *App) AdjustImage(file io.Reader) (*bytes.Buffer, *model.AppError) {
-    img, _, err := a.ch.imgDecoder.Decode(file)
+func (a *App) AdjustImage(rctx request.CTX, file io.ReadSeeker) (*bytes.Buffer, *model.AppError) {
+    img, format, err := a.ch.imgDecoder.Decode(file)
     ...
-    orientation, _ := imaging.GetImageOrientation(file)
+    orientation, err := imaging.GetImageOrientation(file, format)
+    if err != nil {
+        rctx.Logger().Warn("Failed to get image orientation", mlog.Err(err))
+    }
```

The parameter type changed from `io.Reader` to `io.ReadSeeker` specifically so the stream could be rewound — **but the `file.Seek(0, io.SeekStart)` call was omitted**. Compare with the correct pattern in `prepareImage` (`server/channels/app/file.go:1164-1182`), added in the same PR, which does:

```go
img, imgType, release, err = imgDecoder.DecodeMemBounded(imgData)
if err != nil { ... }
if _, err = imgData.Seek(0, io.SeekStart); err != nil { ... }
orientation, err := imaging.GetImageOrientation(imgData, imgType)
```

`SetTeamIconFromFile` in `server/channels/app/team.go:1980-1992` also ships the identical defect — no seek between `image.Decode(file)` and `GetImageOrientation(file, format)` — so team icons from iOS will exhibit the same visual bug.

Prior behavior: the pre-`f8e16780ef` implementation used the `goexif` parser which scanned the raw `io.Reader` (buffering internally) and did not require a seekable input — hence the missing `Seek(0)` was latent. After switching to `bep/imagemeta`, the reader is consumed by `imgDecoder.Decode` first, exposing the defect.

## Phase D — Similar past issues and PRs

`gh issue list` and `gh pr list` against `mattermost/mattermost` (upstream) are blocked by the counterfactual harness allowlist (only `ShankHarinath`, `canonix-engineering`, `canonix` owners permitted). No prior matching issue exists in the home repo. No remediation precedent available. The fix pattern is nonetheless prescribed by the in-repo `prepareImage` function — seek back to start between decode and EXIF read.

## Phase E — Root cause hypothesis

**Root cause.** `App.AdjustImage` in `server/channels/app/user.go` reads the uploaded file twice but doesn't rewind between reads. `a.ch.imgDecoder.Decode(file)` on line 1024 advances `file` past the EXIF header (for JPEGs) / to EOF. `imaging.GetImageOrientation(file, format)` on line 1029 then feeds an already-consumed `io.ReadSeeker` into `imagemeta.Decode`, which fails to locate the Orientation tag. `GetImageOrientation` returns `Upright` (constant value `1`) alongside an error (downgraded to a `Warn` log). `MakeImageUpright(img, 1)` hits the `default` arm and returns the decoded pixel matrix unchanged. For iOS photos (which encode the sensor-native landscape pixel grid plus an `Orientation=6 / RotatedCW` EXIF tag), the saved PNG ends up rotated 90° counter-clockwise relative to how the user sees it in the picker — exactly the reporter's symptom.

The same bug is present in `App.SetTeamIconFromFile` (`server/channels/app/team.go:1980-1992`); it was mechanically ported in the same commit.

**Why "only when choosing a profile picture" (per reporter).** Other image flows use different code paths that *do* rewind or buffer correctly:
- Generic attachments → `UploadFileTask.preprocessImage` / `postprocessImage` use `io.MultiReader(bytes.NewReader(t.buf.Bytes()), t.teeInput)` which supplies fresh bytes.
- Thumbnails/previews → `prepareImage` explicitly seeks to 0 between decode and orientation read.
- Profile picture → `AdjustImage` (broken). Team icon → `SetTeamIconFromFile` (broken, but the reporter only tested profile).

**Why the test didn't catch it.** `TestAdjustProfileImage` (user_test.go:125-148) only asserts `adjusted.Len() > 0` and `NotEqual(testjpg, adjusted)`. It does not assert orientation correctness on a JPEG with a non-upright EXIF tag. The 18 new fixtures in `server/tests/exif_samples/` added by the same PR are only used by `orientation_test.go` (unit-level `GetImageOrientation`), not by `AdjustImage`'s test.

**Confidence.** 0.90 — static evidence is conclusive (exact function identified, exact line identified, regression commit identified, correct-pattern reference exists in the same PR). The 0.10 gap reflects no live runtime repro (kubectl unavailable) and no upstream precedent lookup (harness-blocked). A proposed fix would pass existing tests and should be accompanied by a new unit test that feeds a `right.jpg` / `left.jpg` EXIF fixture through `AdjustImage` and asserts the output pixels correspond to an upright image.

## Recommended fix (for Architect)

1. In `server/channels/app/user.go:1022-1046` (`AdjustImage`), insert a seek-to-start between `Decode` and `GetImageOrientation`:
   ```go
   img, format, err := a.ch.imgDecoder.Decode(file)
   if err != nil { ... }

   if _, seekErr := file.Seek(0, io.SeekStart); seekErr != nil {
       return nil, model.NewAppError("SetProfileImage", "api.user.upload_profile_user.seek.app_error", nil, "", http.StatusInternalServerError).Wrap(seekErr)
   }

   orientation, err := imaging.GetImageOrientation(file, format)
   ```
2. Apply the same fix in `server/channels/app/team.go:1980-1992` (`SetTeamIconFromFile`).
3. Extend `TestAdjustProfileImage` to assert orientation correctness against one of the existing `server/tests/exif_samples/right.jpg` fixtures — decode the returned buffer and compare pixel dimensions/corner samples against `up.jpg` / an upright reference.

## Open questions / blockers

- Runtime verification pending: `kind-canonix` cluster was unreachable during triage (connection refused on 127.0.0.1:59249). Fix can still proceed on static evidence.
- Upstream `mattermost/mattermost` issue/PR search was blocked by the counterfactual-harness owner allowlist; cannot confirm or rule out an existing upstream fix PR.

## Files of interest (absolute paths)

- `/Users/shashank/Canonix/phoenix-test/mattermost/server/channels/app/user.go` (lines 1005-1070, esp. 1022-1046 `AdjustImage`)
- `/Users/shashank/Canonix/phoenix-test/mattermost/server/channels/app/team.go` (lines ~1980-1992 `SetTeamIconFromFile` — same defect)
- `/Users/shashank/Canonix/phoenix-test/mattermost/server/channels/app/file.go` (lines 1164-1182 `prepareImage` — correct-pattern reference)
- `/Users/shashank/Canonix/phoenix-test/mattermost/server/channels/app/imaging/orientation.go` (entire file)
- `/Users/shashank/Canonix/phoenix-test/mattermost/server/channels/app/imaging/decode.go` (entire file)
- `/Users/shashank/Canonix/phoenix-test/mattermost/server/channels/app/user_test.go` (lines 125-148 `TestAdjustProfileImage`)
- `/Users/shashank/Canonix/phoenix-test/mattermost/server/channels/app/users/profile_picture.go` (unrelated to bug — only the read/default-image path)
- `/Users/shashank/Canonix/phoenix-test/mattermost/server/tests/exif_samples/` (EXIF fixtures for test extension)

## Machine-readable footer

<!--phoenix:scout-summary
affected_repos:
  - ShankHarinath/mattermost
confidence: 0.90
recommended_next_step: proceed
primary_suspect_symbol: App.AdjustImage
primary_suspect_file: server/channels/app/user.go
primary_suspect_lines: "1022-1046"
secondary_suspect_symbol: App.SetTeamIconFromFile
secondary_suspect_file: server/channels/app/team.go
regression_commit: f8e16780ef03d0cfb84f873aa626f31193f87cf6
regression_pr: "MM-63436"
risk_of_fix: LOW
leak_signals:
  - scout_mentioned_internal_jira_id_MM-63436
  - scout_mentioned_internal_jira_id_MM-67872
blockers:
  - kubectl_kind_canonix_unreachable
  - upstream_gh_search_denied_by_harness
-->
