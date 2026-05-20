# Mattermost Legal Hold Plugin Codebase Guide (Support Focus)

This guide helps navigate the Mattermost Legal Hold plugin codebase to answer support questions: how holds are scheduled and executed, where the data is written and hashed, why a hold isn't picking up messages, how the S3-override bucket works, why the plugin won't activate without a valid Enterprise license, and how to run the offline processor against a downloaded export.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/ClaudeFiles/Mattermost-Plugin-LegalHold/CLAUDE.md`

## Related Repositories

- **Mattermost Server**: `../Mattermost/` — plugin API (`server/public/plugin/api.go`), `pluginapi` client, KV store, cluster mutex / scheduled jobs, filestore backend (`server/shared/filestore`), `ChannelMemberHistory` table queried directly via `SQLStore`
- **Enterprise**: `../Enterprise/` — required at runtime: `OnActivate` refuses to start without `IsEnterpriseLicensedOrDevelopment` (Entry SKU is rejected)
- **Docs**: `../Mattermost-Docs/` — user-facing docs at `docs.mattermost.com/comply/legal-hold.html` (linked from the System Console settings header via `mattermost.com/pl/legal-hold-documentation`)

The plugin ID is `com.mattermost.plugin-legal-hold` (NOT the older `com.mattermost.plugin-legal-hold` variants used in some forks — this is the canonical Mattermost one).

## Repository Structure

```
server/                              # Go backend (package main)
├── plugin.go                        # Plugin struct, OnActivate (Enterprise license gate), OnConfigurationChange,
│                                    # Reconfigure (rebuilds filestore backend, schedules the job, repairs
│                                    # HasMessages on activation). FixedFileSettingsToFileBackendSettings copies
│                                    # mattermost-server's settings conversion with nil-safe defaults
├── api.go                           # ServeHTTP — gorilla/mux router. All endpoints require ManageSystem.
│                                    # /api/v1/legalholds (GET list / POST create), .../release, PUT update,
│                                    # /download (streamed ZIP), /run (single hold runOnce), /legalhold/run
│                                    # (RunAll), /test_amazon_s3_connection, /groups/search
├── api_test.go
├── plugin_test.go
├── main.go                          # plugin.ClientMain(&Plugin{})
├── config/
│   └── configuration.go             # Configuration struct: TimeOfDay, EnableFilestoreConnectionTest,
│                                    # AmazonS3BucketSettings{Enable, Settings model.FileSettings}
├── jobs/
│   ├── job_manager.go               # JobManager (sync.Map of Job interface) — generic add/remove/list/stop
│   ├── jobs_util.go                 # loggerIface
│   ├── legal_hold_job_interface.go  # LegalHoldJobInterface (RunAll / RunSingleLegalHold / GetRunningLegalHolds)
│   ├── legal_hold_job_settings.go   # parseLegaHoldJobSettings (note the typo), CalcNext (always next day at TimeOfDay)
│   └── legal_hold_job.go            # LegalHoldJob — cluster.Schedule for the daily run, cluster.JobOnceScheduler
│                                    # for ad-hoc per-hold runs via /api/v1/legalholds/{id}/run
├── legalhold/
│   ├── legal_hold.go                # Execution struct — orchestrates one execution window: GetChannels,
│   │                                # ExportData (CSV in 10000-post batches), ExportFiles, UpdateIndexes,
│   │                                # WriteFileHashes. cluster.NewMutex("legal_hold_execution") with 5s wait
│   ├── legal_hold_test.go
│   └── hash.go                      # In-memory Hash/HashList helpers used during execution
├── model/
│   ├── legal_hold.go                # LegalHold struct, CreateLegalHold/UpdateLegalHold DTOs, validators,
│   │                                # NextExecutionStartTime/EndTime, IsFinished, BasePath/IndexPath,
│   │                                # default ExecutionLength = 86400000 (24h)
│   ├── index.go                     # LegalHoldIndex / Users / Teams / Channels / Memberships
│   ├── export.go                    # LegalHoldCursor, LegalHoldPost (CSV-tagged for gocsv)
│   ├── channel.go                   # ChannelMetadata (SQL query result type)
│   ├── file_info.go                 # Abridged FileInfo (ID/Path/Name/Size/MimeType)
│   └── hash.go                      # HashList type alias (map[string]string) for hashes.json
├── store/
│   ├── kvstore/
│   │   ├── kvstore.go               # KVStore interface (Create/GetAll/GetByID/Update/Delete)
│   │   └── legal_hold.go            # KV-backed impl. Key prefix: "kvstore_legal_hold_<id>".
│   │                                # CreateLegalHold rejects name dupes AND functional dupes
│   │                                # (same Start/End/Users/Groups/IncludePublicChannels).
│   │                                # Updates use SetAtomic(oldValue) — concurrent updates return
│   │                                # "already been updated by someone else"
│   └── sqlstore/
│       ├── store.go                 # SQLStore with master + replica sqlx handles built from
│       │                            # pluginapi.Client.Store (Postgres or MySQL)
│       ├── legal_hold.go            # GetPostsBatch (cursor over Posts table, joins Users/Channels/Teams/Bots,
│       │                            # DM display names via split_part/substring_index),
│       │                            # GetChannelIDsForUserDuring (queries ChannelMemberHistory),
│       │                            # GetFileInfosByIDs, GetChannelMetadataForIDs
│       └── group.go                 # SearchLDAPGroupsByPrefix — UserGroups where Source='ldap'
└── utils/
    ├── utils.go                     # Min/Max/DeduplicateStringSlice
    ├── backend.go                   # MinIO test backend settings
    └── testcontainers.go            # testcontainers-go helpers for Postgres + MySQL + MinIO in tests

processor/                           # Separate go binary (own go.mod) — offline tool, NOT part of the plugin
├── main.go → cmd/root.go            # Cobra command: --legal-hold-data <zip> --output-path <dir>
│                                    # --legal-hold-secret <secret>
├── parse/                           # ZIP traversal: index.json, channels, posts CSV, files, hash verification
└── view/templates/                  # html/template files for human-readable HTML output

webapp/                              # React/TypeScript admin-console-only UI (no channel header, no commands)
├── src/
│   ├── index.tsx                    # Registers ONLY two admin console custom settings:
│   │                                # LegalHoldsSettings and AmazonS3BucketSettings
│   ├── client.ts                    # APIClient — wraps /plugins/com.mattermost.plugin-legal-hold/api/v1
│   ├── manifest.ts
│   └── components/
│       ├── legal_holds_setting.tsx  # Top-level legal holds table + create button
│       ├── legal_hold_table/        # List view
│       ├── create_legal_hold_form.tsx
│       ├── update_legal_hold_form/
│       ├── amazon_s3_bucket_settings.tsx + admin_console_settings/  # S3 override UI + "Test connection"
│       ├── users_input/             # User picker (async)
│       ├── groups_input/            # LDAP group picker — calls /api/v1/groups/search
│       ├── show_secret_modal.tsx    # Displays the per-LegalHold HMAC secret after creation
│       └── confirm_release.tsx

plugin.json                          # Plugin manifest. Settings: TimeOfDay, EnableFilestoreConnectionTest,
                                     # LegalHoldsSettings (custom), AmazonS3BucketSettings (custom)
Makefile
e2e-tests/                           # Playwright suite
```

## Configuration

The plugin has a deceptively small `Configuration` struct — most of the complexity lives in the **per-LegalHold** settings stored in KV, not in plugin config.

| Plugin setting | Type | Purpose |
|----------------|------|---------|
| `TimeOfDay` | string `"3:00am -0700"` | Time-of-day for the daily cluster scheduled job. Parsed with layout `"3:04pm -0700"`. Use `+0000` for UTC. Bad value = job fails to start (`parseLegaHoldJobSettings` returns error). |
| `EnableFilestoreConnectionTest` | bool | If true, every `Reconfigure` calls `filesBackend.TestConnection()`. On failure, **the plugin auto-disables `AmazonS3BucketSettings.Enable`** via `SavePluginConfig` to prevent activation failures from a misconfigured override bucket. |
| `LegalHoldsSettings` | custom | Webapp-rendered list/CRUD UI; no server-side setting, the React component just renders `LegalHoldsSetting`. |
| `AmazonS3BucketSettings` | custom (`{Enable bool, Settings model.FileSettings}`) | Optional override of the server's `FileSettings` for legal hold storage. When `Enable=true`, `Reconfigure` builds a *separate* `FileBackend` and writes legal hold data there instead of the server's main filestore. |

### License gating

`OnActivate` (`server/plugin.go`):

```go
if !pluginapi.IsEnterpriseLicensedOrDevelopment(config, license) ||
   (!pluginapi.IsConfiguredForDevelopment(config) && license.SkuShortName == MattermostEntrySkuShortName) {
    return fmt.Errorf("this plugin requires an Enterprise license")
}
```

So:
- No license / Team Edition without `EnableDeveloper` + `EnableTesting` → won't activate.
- Entry SKU is **explicitly rejected** even though it's a paid SKU.
- Professional / Enterprise / E20 / E10 / dev mode → activates.

## Data Model & Storage

### KV-stored LegalHold (one record per hold)

Key: `kvstore_legal_hold_<26-char ID>` (`legalHoldPrefix` in `store/kvstore/legal_hold.go`).

Fields (`server/model/legal_hold.go`):

| Field | Notes |
|-------|-------|
| `ID` | 26-char Mattermost ID generated via `mattermostModel.NewId()` |
| `Name` | URL-safe (`IsValidAlphaNumHyphenUnderscore`), 2–64 chars, used in file storage path |
| `DisplayName` | 2–64 chars, free-form |
| `UserIDs` / `GroupIDs` | At least one of these must be non-empty. Groups are resolved to users at execution time via `GetGroupMemberUsers` (paged 50/page, capped at 100 pages = 5000 members per group) |
| `StartsAt` / `EndsAt` | Unix millis. `EndsAt=0` means open-ended. |
| `IncludePublicChannels` | If false, public channels are excluded from `GetChannelIDsForUserDuring` (but deleted channels are still included — see `LEFT JOIN` in `sqlstore/legal_hold.go`) |
| `LastExecutionEndedAt` | Watermark — the upper bound of the last executed window |
| `ExecutionLength` | Default 86400000 ms (24h). Hardcoded in `NewLegalHoldFromCreate`; not user-configurable through the API. |
| `Secret` | Random 26-char ID set at creation time. Used as the HMAC-SHA512 key for all file hashes in this hold. Shown to the admin once via `show_secret_modal.tsx`. |
| `HasMessages` | Denormalized: true once any batch has been written. Used to gate the Download button. Repaired on plugin activation by checking `IndexPath()` existence. |
| `Status` | DTO-only (`executing`). Computed from `GetRunningLegalHolds()` at list time. Never persisted. |

### File backend layout

Base path: `legal_hold/<Name>_<ID>/` (`LegalHold.BasePath()`).

Within a hold:
- `index.json` — `LegalHoldIndex` JSON, merged across executions (`UpdateIndexes` merges with any pre-existing file)
- `hashes.json` — `HashList` (`map[path]hex`) of HMAC-SHA512 hashes keyed by the hold's `Secret`, written by `WriteFileHashes`
- `<channelID>/messages/messages-<batchCreateAt>-<batchPostID>.csv` — `LegalHoldPost` rows (gocsv)
- `<channelID>/files/files-<batchCreateAt>-<batchPostID>/<fileID>/<cleanName>` — file attachment, with the original `FileInfo.Name` reduced to `filepath.Base` and path-traversal-checked (see `Execution.filePath`, hardened in MM-68439)

### Cluster coordination

| Mutex / scheduler | Where | Purpose |
|-------------------|-------|---------|
| `cluster.NewMutex("legal_hold_execution")` | `legalhold/legal_hold.go` `Execute` | Only one node may execute *any* legal hold at a time. 5s `LockWithContext` timeout — others bail with `failed to lock cluster mutex`. |
| `cluster.Schedule(papi, "legal_hold_job", ...)` | `jobs/legal_hold_job.go` `start` | Daily run across the cluster (one node fires per `TimeOfDay`). |
| `cluster.JobOnceScheduler` | `jobs/legal_hold_job.go` | Ad-hoc per-hold runs initiated by `POST /api/v1/legalholds/{id}/run`. Key prefix: `legal_hold_run_<id>`. |

`OnActivate` deletes a stale legacy KV entry `cron_legal_hold_job` (left over from older versions that used a different scheduler).

## Execution Flow (one daily window)

1. Daily cluster job fires (`run`) → `runWith(legalHolds, forceRun=false)`.
2. For each LegalHold, while `!IsFinished() && (forceRun || NeedsExecuting(now)) && LastExecutionEndedAt < now`:
   1. Build `Execution` for the next window `[NextExecutionStartTime, NextExecutionEndTime]` (default 24h-wide, capped at `EndsAt`).
   2. Acquire `legal_hold_execution` cluster mutex.
   3. `GetChannels` — resolve `GroupIDs` → users (paged), de-dupe with `UserIDs`, query `ChannelMemberHistory` for channels each user belonged to during the window.
   4. `ExportData` — for each channel, page `Posts` in batches of `PostExportBatchLimit=10000`. Write each batch as a CSV file. Set `HasMessages=true` as soon as any batch exists.
   5. `ExportFiles` — for each post's `FileIds`, `CopyFile` from the source filestore path to the legal hold path. Compute HMAC-SHA512 of file contents using the hold's `Secret`.
   6. `UpdateIndexes` — read existing `index.json`, merge new users/teams/channels in, write back.
   7. `WriteFileHashes` — merge accumulated `hashes` map with any existing `hashes.json`, write back.
   8. Update KV `LegalHold` with new `LastExecutionEndedAt` (clamped to `now`) and `HasMessages`. Loop.

> **Note** (the `forceRun` path): when an admin clicks "Run now" on a single hold, it goes through `JobOnceScheduler.ScheduleOnce` → `runOnce` → `runWith([hold], forceRun=true)`. `NeedsExecuting` is bypassed but `IsFinished` and the `LastExecutionEndedAt >= now` check still apply, so a hold that's already caught up to "now" won't do anything new.

## HTTP API (`server/api.go`)

All endpoints require `Mattermost-User-ID` header AND `PermissionManageSystem` (System Admin). Routes are rebuilt on every request (the router is reassigned in `ServeHTTP`).

| Method | Path | Handler | Notes |
|--------|------|---------|-------|
| GET | `/api/v1/legalholds` | `listLegalHolds` | Annotates `Status=executing` for any hold present in `GetRunningLegalHolds()` |
| POST | `/api/v1/legalholds` | `createLegalHold` | Validates via `IsValidForCreate`, generates ID + Secret in `kvstore` |
| PUT | `/api/v1/legalholds/{id}` | `updateLegalHold` | ID in URL must match body. `ApplyUpdates` only writes DisplayName/UserIDs/GroupIDs/IncludePublic/EndsAt — Name and StartsAt are immutable post-creation |
| POST | `/api/v1/legalholds/{id}/release` | `releaseLegalHold` | Refuses with 409 if status is `executing`. Removes file directory THEN deletes KV entry |
| POST | `/api/v1/legalholds/{id}/run` | `runSingleLegalHold` | Schedules a one-shot via `JobOnceScheduler` |
| GET | `/api/v1/legalholds/{id}/download` | `downloadLegalHold` | Streams a ZIP of `legal_hold/<Name>_<ID>/` directly to the response with on-the-fly Deflate compression |
| POST | `/api/v1/legalhold/run` | `runJobFromAPI` | Async `RunAll` (fires-and-forgets) |
| POST | `/api/v1/test_amazon_s3_connection` | `testAmazonS3Connection` | Tests the S3 override bucket — fails 400 if S3 override is not enabled |
| GET | `/api/v1/groups/search?prefix=` | `searchLDAPGroups` | Returns up to 10 LDAP `UserGroups` whose Name or DisplayName starts with `prefix` |

## Processor (offline post-processor)

The `processor/` directory is a separate Go binary with its own `go.mod`. It does NOT ship with the plugin — admins build it locally and run it against a downloaded `legalholddata.zip`.

```
./processor --legal-hold-data ./legalholddata.zip --output-path ./out --legal-hold-secret "<secret>"
```

- `--legal-hold-secret` is the per-hold `Secret` shown when the hold was created. The processor verifies every file's HMAC-SHA512 against `hashes.json`.
- Outputs an HTML site rooted at `out/index.html`, traversable in a browser (no JS required, Ctrl+F search).
- Recent fixes (`44800ab`, `5eff41c`) deal with deleted users in DMs/GMs and missing files in already-processed exports.

## Common Support Investigation Patterns

### "Plugin won't activate / 'this plugin requires an Enterprise license'"
1. Check `OnActivate` in `server/plugin.go` — exact rejection conditions above.
2. **Entry SKU is rejected** — customers on Entry tier cannot use Legal Hold, period. Confirm SKU with `GET /api/v4/license` (`SkuShortName`).
3. Trial license should work. Dev mode (`EnableDeveloper=true` AND `EnableTesting=true`) bypasses the license check.

### "Legal hold isn't capturing any messages / `HasMessages=false`"
1. Has the daily job ever run for it? Check `LastExecutionEndedAt`. If it equals `StartsAt`, the job has never written anything.
2. Is `EnableLegalHoldJobs` actually true? `parseLegaHoldJobSettings` returns `EnableLegalHoldJobs=false` only if `cfg==nil`; a non-nil config makes it always true. Bad `TimeOfDay` makes `OnConfigurationChange` return an error and the job won't be added.
3. Were the target users members of any channels during the window? Check `ChannelMemberHistory` for the window in question.
4. If `IncludePublicChannels=false`, public channels are excluded — but `GetChannelIDsForUserDuring` still includes channels deleted before the window (`LEFT JOIN` with `Channels.id IS NULL`). This was fix `07d000c`.
5. Group hold? Look for "exceeds the maximum number of members" — `getUsersForGroups` caps each group at 5000 members and aborts if exceeded.

### "Run-once doesn't seem to do anything"
1. `JobOnceScheduler` swallows scheduling errors at run time. Check server logs for `Creating legal hold ad-hoc job`.
2. `runWith(...forceRun=true)` skips `NeedsExecuting` but still respects `IsFinished` and `LastExecutionEndedAt >= now`. A hold whose `LastExecutionEndedAt` is already at `now` (because the daily job ran recently) won't re-process.
3. Cluster mutex contention: `legal_hold_execution` has a **5-second** lock timeout. If another node is executing, the run fails with `failed to lock cluster mutex` and is **NOT retried** automatically.

### "Download endpoint produces a truncated/corrupted ZIP"
1. `downloadLegalHold` streams directly — there is no temp file, no size header. A 5xx mid-stream produces a partial ZIP.
2. `FileBackend.ListDirectoryRecursively` failures or transient S3 errors during `CopyFile`/`Reader` will produce a truncated archive.
3. The archive is built with `zip.Deflate` and flushed per entry; client must read to EOF.

### "S3 bucket override breaks the plugin / plugin won't reactivate"
1. `EnableFilestoreConnectionTest=true` (default) runs `TestConnection()` on every config change. On failure, the plugin **auto-sets `AmazonS3BucketSettings.Enable=false`** via `SavePluginConfig` and returns the error. The S3 override silently turns itself off — the admin sees "connection test for filestore failed" once but the toggle flips, which is intentional self-healing.
2. If `EnableFilestoreConnectionTest=false`, a bad S3 config will make subsequent reads/writes during execution fail instead.
3. `Reconfigure` calls `FixedFileSettingsToFileBackendSettings`, which itself calls `fileSettings.SetDefaults(true)` to nil-safe the pointers. Older Mattermost versions where `FileSettings.ToFileBackendSettings` existed are no longer relied on.

### "Updating a hold returns 'already been updated by someone else'"
1. KV uses `SetAtomic(oldValue)` for compare-and-swap. Concurrent writes (rapid UI clicks across tabs / nodes) lose.
2. The hold may have been just-now updated by the running job (`HasMessages`, `LastExecutionEndedAt`). Re-fetch and retry on the client.

### "Created hold but UI shows 'duplicate' error"
Two kinds of duplicates are blocked in `CreateLegalHold` (`store/kvstore/legal_hold.go`):
1. Name conflict: any other hold with the same `Name`.
2. **Functional duplicate**: same `StartsAt`, `EndsAt`, `IncludePublicChannels`, AND the same set (unordered) of `UserIDs` and `GroupIDs`. Even a renamed but otherwise-identical hold is rejected.

### "LDAP group picker returns nothing"
1. `searchLDAPGroups` requires the group to be `Source='ldap'` and `DeleteAt=0`. Manually created groups won't appear.
2. Search is case-insensitive prefix on Name OR DisplayName, capped at 10 results. Typing a substring further down the name will fail.
3. Wildcards `%`/`_` in the prefix are escaped by `sanitizeSearchTerm`.

### "Processor fails to verify hash / 'invalid hash'"
1. `--legal-hold-secret` must exactly match the hold's `Secret` from the time of creation. If it was rotated or lost, hashes cannot be verified. The secret is shown **once** (`show_secret_modal.tsx`) at creation and never again — admins are expected to store it.
2. HMAC-SHA512 with the secret as key, hex-encoded. Compare `hashes.json` entries to fresh hashes of the files.
3. Hash key for messages is the CSV content as written (post-`gocsv.MarshalString`), not the raw posts.

### "Files don't end up in the legal hold export"
1. `ExportFiles` swallows missing-file errors: `Failed to find file attachment to copy` is logged, then it `return nil` to avoid stalling the whole job. So files referenced by `Posts.FileIds` but not present in the filestore are silently skipped.
2. Files with hostile names (`""`, `.`, `..`, `/`) are logged as `Skipping file with unsafe name in legal hold export` and skipped — see `Execution.filePath`, hardened in MM-68439.
3. The export uses `FileInfo.Path`, not a regenerated path — if `FileInfo` rows are missing for a `FileID`, that file is simply absent.

### "Hold status stuck in 'executing'"
1. `Status=executing` is purely derived from `JobOnceScheduler.ListScheduledJobs()` finding a key matching `legal_hold_run_<id>`. If a `runOnce` job crashed or the node died mid-run, the scheduler entry may persist.
2. Status is recomputed on every `listLegalHolds` call; restart of the plugin or completion of the job clears it.

## Where things live (cheat sheet)

| You're asking about... | Look at... |
|------------------------|------------|
| Why a hold won't activate | `server/plugin.go` `OnActivate` (license check + Entry SKU rejection) |
| Why no messages were captured | `server/sqlstore/legal_hold.go` `GetChannelIDsForUserDuring` + `GetPostsBatch` |
| Why files are missing from export | `server/legalhold/legal_hold.go` `ExportFiles`, `Execution.filePath` |
| Daily schedule behavior | `server/jobs/legal_hold_job.go` + `legal_hold_job_settings.go` (CalcNext) |
| Ad-hoc "Run now" | `server/jobs/legal_hold_job.go` `RunSingleLegalHold`, `runOnce` |
| Cluster locking failures | `server/legalhold/legal_hold.go` (5s mutex timeout) + `server/jobs/legal_hold_job.go` (cluster.Schedule) |
| S3 override behavior | `server/plugin.go` `Reconfigure`, `server/api.go` `testAmazonS3Connection` |
| Duplicate-hold rejection | `server/store/kvstore/legal_hold.go` `CreateLegalHold` |
| Index merging across executions | `server/legalhold/legal_hold.go` `UpdateIndexes` |
| File hashing | `server/legalhold/legal_hold.go` `hashFromReader`, `WriteFileHashes`; per-hold `Secret` |
| LDAP group resolution | `server/legalhold/legal_hold.go` `getUsersForGroups`; `server/sqlstore/group.go` |
| HTTP routes / authz | `server/api.go` `ServeHTTP` (ManageSystem required) |
| What the admin UI registers | `webapp/src/index.tsx` (just two custom admin console settings — no LHS/header) |
| Offline processing of a downloaded ZIP | `processor/` (separate go.mod, own README) |
| Path-traversal hardening | `server/legalhold/legal_hold.go` `Execution.filePath` (MM-68439) |
