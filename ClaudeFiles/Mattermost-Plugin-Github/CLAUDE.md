# Mattermost GitHub Plugin Codebase Guide (Support Focus)

This guide helps navigate the Mattermost GitHub plugin codebase to answer support questions: where OAuth credentials are validated, how channel subscriptions filter events, how the encryption key is rotated, how the org lock (`GithubOrg`) is enforced, how GitHub Enterprise Server is targeted, where webhook events are parsed, and how the review-SLA digest is scheduled.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/ClaudeFiles/Mattermost-Plugin-Github/CLAUDE.md`

## Related Repositories

- **Mattermost Server**: `../Mattermost/` — plugin API (`server/public/plugin/api.go`), `pluginapi` client, KV store, cluster mutex, OAuth proxy (Chimera) config
- **Enterprise**: `../Enterprise/` — not used directly; the plugin is free/open source
- **GitLab plugin**: `../Mattermost-Plugin-Gitlab/` — forked from THIS plugin; many patterns are shared (OAuth, subscriptions, encryption, permalinks). Note GitLab later added a multi-instance model this plugin does NOT have.
- **Docs**: `../Mattermost-Docs/` — user-facing docs at `docs.mattermost.com/integrate/github-interoperability.html`

## Repository Structure

Plugin ID is **`github`**, so all HTTP routes are under `/plugins/github/...`. The Go backend lives under `server/plugin/` (package `plugin`), unlike the GitLab fork which uses `server/` directly.

```
server/
├── main.go                       # Entrypoint: plugin.ClientMain(...)
├── plugin/                       # package plugin — all backend logic
│   ├── plugin.go                 # Plugin struct, lifecycle hooks, KV key constants, OAuth config,
│   │                             # token decrypt, reEncryptUserData (key rotation), GetToDo (~1370 lines, central file)
│   ├── configuration.go          # Configuration struct, validation, secret generation,
│   │                             # OnConfigurationChange → triggers re-encryption on key change
│   ├── api.go                    # HTTP API handlers (OAuth callback, todo, issue create, subscriptions, LHS data, etc.)
│   ├── command.go                # /github slash command (connect, subscriptions, issue, mute, settings, setup, etc.)
│   ├── oauth.go                  # OAuth connect/complete flow, Chimera proxy support
│   ├── flows.go                  # FlowManager — interactive setup wizard (oauth/webhook/announcement steps)
│   ├── subscriptions.go          # Channel subscription CRUD; SubscriptionFlags, Features, repo/org subscribe
│   ├── webhook.go                # /webhook HTTP handler + per-event handlers (postPullRequestEvent, postIssueEvent, ...)
│   ├── permalinks.go             # MessageWillBePosted hook — expands GitHub file permalinks to code previews
│   ├── template.go               # Go templates that render webhook events into Markdown posts
│   ├── review_sla.go             # Per-PR review-SLA start tracking (when a review was requested)
│   ├── sla_digest.go             # Builds + posts the daily "overdue reviews" digest message
│   ├── sla_digest_scheduler.go   # Once-per-day local-midnight scheduler driving the SLA digest
│   ├── cluster.go                # OnPluginClusterEvent — inter-node messages (e.g., subscription updates)
│   ├── audit.go                  # Audit record helpers (key rotation, etc.)
│   ├── support_packet.go         # GenerateSupportData hook — connected-user count + config snapshot
│   ├── mm_34646_token_refresh.go # One-time token-reset migration for MM-34646 (guarded by KV done-key + mutex)
│   ├── utils.go                  # encrypt/decrypt (AES-GCM), URL builders, helpers
│   └── graphql/                  # GraphQL client (github.com/shurcooL/githubv4 style)
│       ├── client.go             # GraphQL client wrapper
│       ├── lhs_query.go          # Left-hand-sidebar query (review requests, assignments, open PRs)
│       ├── lhs_request.go        # LHS request/marshaling
│       └── digest_query.go       # Org-wide open-PR query feeding the SLA digest
├── mocks/                        # Generated mocks
└── testutils/

webapp/                          # React/TypeScript frontend
├── src/
│   ├── index.js                  # Webapp entry — registers components, channel header, LHS, bot icon,
│   │                             # slash command, post types, websocket handlers
│   ├── components/               # UI (sidebar, modals, settings, subscriptions, attach-comment-to-issue)
│   ├── actions/                  # Redux thunks (fetch LHS data, connect, subscriptions CRUD)
│   ├── reducers/ · selectors.ts  # Plugin state (connected, LHS counts, settings)
│   ├── client/                   # API client wrappers for plugin REST endpoints
│   ├── websocket/                # WS handlers (github_connect, github_disconnect, github_refresh, config_update)
│   ├── action_types/ constants/  # Redux + UI constants
│   ├── hooks/ · utils/ · types/  # React hooks, helpers, TS definitions
│   └── manifest.test.ts
├── jest.config.js
└── webpack.config.js

plugin.json                       # Manifest: settings schema, min_server_version (10.7.0), executables
Makefile
```

## Configuration

Unlike the GitLab fork, this plugin has a **single** configuration model (no KV-stored multi-instance map). Settings live in `Configuration` (`server/plugin/configuration.go`), set via System Console → Plugins → GitHub.

| Setting | Purpose | Notes |
|---------|---------|-------|
| `GitHubOAuthClientID` / `GitHubOAuthClientSecret` | OAuth app credentials | Required unless using Chimera (`UsePreregisteredApplication`) |
| `GitHubOrg` | (Optional) Restrict the plugin to one or more GitHub orgs | Comma-separated; enforced when connecting and subscribing |
| `EnterpriseBaseURL` / `EnterpriseUploadURL` | GitHub Enterprise Server base/upload URLs | Empty = github.com SaaS. `getBaseURL()` in configuration.go |
| `WebhookSecret` | Verifies inbound GitHub webhooks | Auto-generated; must match the webhook's Secret + `X-Hub-Signature-256` |
| `EncryptionKey` | AES key used to encrypt stored user OAuth tokens | Auto-generated; changing it triggers full re-encryption |
| `EnablePrivateRepo` | Allow OAuth scope + subscriptions for private repos | Changes OAuth scope from `public_repo` to `repo` |
| `ConnectToPrivateByDefault` | `/github connect` requests private scope without `connect private` | Per-deployment toggle |
| `EnableLeftSidebar` | Show the GitHub LHS (reviews/assignments/open PRs/unread) | |
| `EnableCodePreview` | Permalink expansion: `disable` / `public` / `privateAndPublic` | Public is default |
| `EnableWebhookEventLogging` | Log full inbound webhook payloads | Debugging aid; verbose |
| `ShowAuthorInCommitNotification` | Include commit author in push notifications | |
| `GetNotificationForDraftPRs` | Notify on draft PR review requests | |
| `UsePreregisteredApplication` | Use Chimera-hosted OAuth (Cloud only) | Requires `PluginSettings.ChimeraOAuthProxyURL` on the server |
| `ReviewTargetDays` | Days from PR open until a review is "due" (`0` = SLA disabled) | Drives review_sla.go + the digest |
| `OverdueReviewsChannelID` | Channel for the daily overdue-review digest | Empty = digest disabled even if `ReviewTargetDays>0` |

### OAuth scopes

`getOAuthConfig` (`plugin.go`) requests: `public_repo` (or `repo` when `EnablePrivateRepo` + private connect), `notifications`, `read:org`, `admin:org_hook`. Granted scopes are validated against the `X-OAuth-Scopes` response header (`validateOAuthScopes`).

### Chimera proxy

When `UsePreregisteredApplication=true`, OAuth is proxied through Chimera (`getOAuthConfigForChimeraApp`). Used on Mattermost Cloud so customers don't register their own GitHub OAuth app. The server-side Chimera URL comes from `PluginSettings.ChimeraOAuthProxyURL` (or `MM_PLUGINSETTINGS_CHIMERAOAUTHPROXYURL`).

## Storage Model (KV)

Persistent state lives in plugin KV. Key constants/suffixes in `plugin.go`:

| Key | Value | Notes |
|-----|-------|-------|
| `<userID>_githubtoken` | `GitHubUserInfo` JSON (token encrypted) | `Token.AccessToken` is AES-encrypted with `EncryptionKey`; rest of struct is not |
| `githuboauthkey_<state>` | OAuth state | Short-lived; verifies the OAuth callback |
| `<githubUsername>_githubusername` | Mattermost userID | Reverse lookup: GitHub → Mattermost |
| `<userID>_githubprivate` | Private-repo connection flag | Whether the user granted private scope |
| `subscriptions` | Channel subscriptions (repo/org → []*Subscription) | Single global key, mutated under cluster mutex |
| review-SLA start keys | PR review-request start times | See `reviewSLAStartKey(owner, repo, prNumber, login)` in review_sla.go |
| `reencrypt_user_data_mutex` | Cluster mutex for key rotation | Only one node re-encrypts at a time |
| `mm34646_token_reset_mutex` / `mm34646_token_reset_done` | One-time MM-34646 token-reset guard | Runs once cluster-wide, then sets the done-key |

## OAuth Flow

1. `/github connect` → `/oauth/connect` (`oauth.go`)
2. Plugin stores OAuth state (`githuboauthkey_<state>`), redirects to GitHub authorize URL (github.com or `EnterpriseBaseURL`)
3. GitHub redirects to `/oauth/complete` with code + state
4. Plugin verifies state, exchanges code for token, stores encrypted token + `GitHubUserInfo` in KV
5. WebSocket `github_connect` event broadcast to the user

For Chimera/Cloud, steps 2–3 go through the Chimera proxy URL.

## Token Encryption & Key Rotation

- **Encryption**: `encrypt`/`decrypt` in `utils.go` (AES-GCM with `EncryptionKey`). Only `GitHubUserInfo.Token.AccessToken` is encrypted.
- **Key rotation**: changing `EncryptionKey` in System Console is detected in `OnConfigurationChange` (`configuration.go` compares `previousConfig.EncryptionKey` to the new value), which calls `reEncryptUserData(new, previous)` in `plugin.go`. It:
  1. Acquires the `reencrypt_user_data_mutex` cluster mutex
  2. Lists all `*_githubtoken` keys
  3. For each: idempotency check (if already decryptable with the new key, skip) → decrypt with previous key → re-store with new key
  4. Writes a `reEncryptUserData` audit record
- The previous key is passed in-memory for the duration of the rotation; restarting mid-rotation can strand tokens still encrypted with the old key.

## Webhook Pipeline

1. GitHub sends POST to `<SiteURL>/plugins/github/webhook`
2. `handleWebhook` (`webhook.go`) validates the `X-Hub-Signature-256` HMAC against `WebhookSecret`
3. Parses the event (`github.ParseWebHook`), dispatches by type to a `post<Type>Event` handler in `webhook.go`
4. Handler matches against subscriptions (`subscriptions.go`), applies feature + flag filters
5. Renders the post via Go templates (`template.go`) and posts as the bot

Event handlers present (`webhook.go`): pull request, issue, push, create, delete, issue comment, PR review, PR review comment, star, workflow job, workflow run, release, discussion, discussion comment.

Subscription **features** (comma-separated, validated in `command.go`):
`issues`, `pulls`, `pulls_merged`, `pulls_created`, `pushes`, `creates`, `deletes`, `issue_comments`, `pull_reviews`, `stars`, `releases`, `workflow_failure`, `workflow_success`, `discussions`, `label:"<name>"`. Default when none specified: `pulls,issues,creates,deletes`.

Subscription **flags** (`SubscriptionFlags` in `subscriptions.go`): `--exclude-org-member`, `--include-only-org-members`, `--render-style`, `--exclude` (repos when subscribing to an org). `EnableWebhookEventLogging` logs raw payloads for debugging.

## Slash Commands

`/github` subcommands (see `command.go`):

| Subcommand | Purpose |
|------------|---------|
| `connect` / `connect private` / `disconnect` | OAuth user link (private requests `repo` scope) |
| `me` | Show connected GitHub account |
| `todo` | DM with unread notifications + PRs awaiting your review |
| `subscriptions list/add/delete` | Manage channel subscriptions (repo or org) |
| `issue create` | Create a GitHub issue from Mattermost |
| `mute list/add/delete/delete-all` | Mute notifications from specific GitHub users |
| `settings` | Per-user notification settings (notifications on/off) |
| `setup [oauth\|webhook\|announcement]` | Re-run the interactive setup wizard (FlowManager) |
| `about` | Build info |
| `help` | Slash command help |

## Org Lock (`GithubOrg`)

When set, restricts the plugin to the configured org(s). Enforced when a user connects and when subscribing to a repo/org — subscriptions/connections outside the allowed org(s) are rejected. Search `command.go`/`subscriptions.go` for the org-membership and allow-list checks.

## GitHub Enterprise Server

Set `EnterpriseBaseURL` (and `EnterpriseUploadURL`) to point the plugin at a self-hosted GitHub Enterprise Server instead of github.com. `getBaseURL()` in `configuration.go` selects the base; the GitHub REST/GraphQL clients and OAuth URLs are all built from it. Empty values = github.com SaaS.

## Permalink Expansion

`MessageWillBePosted` hook (`plugin.go` → `permalinks.go`) matches GitHub blob URLs with line anchors and replaces them with a Markdown code block fetched via the post author's OAuth token. Gated by `EnableCodePreview`: `disable`, `public` (default; public repos only), `privateAndPublic` (also private — warns it leaks code). It runs **synchronously**, so slow GitHub API calls delay post creation.

## Review SLA & Daily Digest (unique to this fork)

- `ReviewTargetDays > 0` enables review-SLA tracking. `review_sla.go` records when a review was requested per PR/reviewer (KV start keys), using PR timeline data to find the most recent surviving request (`findMostRecentReviewRequestTime`, `findEarliestSurvivingTeamRequestTime`).
- `sla_digest_scheduler.go` fires once per local midnight (`durationUntilNextLocalMidnight`); `maybePostDailyOverdueSLADigest` (`sla_digest.go`) is a no-op unless both `OverdueReviewsChannelID` and `ReviewTargetDays` are set.
- The digest queries org-wide open PRs via GraphQL (`graphql/digest_query.go`), resolves requested reviewers (including team expansion), computes overdue items past the target, and posts a summary to `OverdueReviewsChannelID`. It impersonates a "service" connected GitHub user (`pickServiceGitHubUser`) to make the API calls.

## Support Packet

`GenerateSupportData` (`support_packet.go`) emits a file under `mm_support_packet/plugins/github/` when an admin runs **Generate Support Packet**. Includes the connected-user count and a configuration snapshot (without secrets).

## Common Support Investigation Patterns

### "/github connect doesn't work / OAuth redirect fails"
1. Verify `GitHubOAuthClientID/Secret` are set (or `UsePreregisteredApplication` for Cloud/Chimera)
2. Verify `SiteURL` is set on the server
3. GitHub OAuth app callback URL must be `<SiteURL>/plugins/github/oauth/complete`
4. For GitHub Enterprise: `EnterpriseBaseURL` must match the GHES host
5. Org-locked: connecting user must belong to an org in `GithubOrg`

### "Webhook events not appearing in channel"
1. Webhook URL must be `<SiteURL>/plugins/github/webhook`, content type `application/json`
2. Webhook Secret must equal `WebhookSecret` (validated as `X-Hub-Signature-256` HMAC in `handleWebhook`)
3. Confirm the channel has a subscription with the right feature: `/github subscriptions list`
4. Check the relevant `post<Type>Event` handler in `webhook.go` for filter/early-return paths
5. Turn on `EnableWebhookEventLogging` to see raw inbound payloads in the logs
6. Org-locked: events for repos outside `GithubOrg` are filtered

### "Token decryption errors / users disconnected after key change"
1. Logs around `reEncryptUserData`; audit record `reEncryptUserData`
2. `EncryptionKey` change is what triggers re-encryption (detected in `OnConfigurationChange`)
3. Restarting mid-rotation can strand tokens encrypted with the old key
4. The MM-34646 one-time token reset (`mm_34646_token_refresh.go`) runs once cluster-wide; check the `mm34646_token_reset_done` KV key

### "Private repo events/permalinks not working"
1. `EnablePrivateRepo` must be on (changes OAuth scope to `repo`)
2. The user must reconnect with `/github connect private` (or set `ConnectToPrivateByDefault`)
3. Permalink previews of private code require `EnableCodePreview=privateAndPublic`

### "Overdue review digest not posting"
1. Both `ReviewTargetDays>0` and `OverdueReviewsChannelID` must be set
2. Scheduler runs at local midnight — check server timezone (`time.Local`)
3. Needs a connected "service" GitHub user with org access (`pickServiceGitHubUser`)
4. GraphQL/org access errors are logged in `collectAllOverdueSLAItems` / `fetchAllOrgOpenPRs`

### "GitHub Enterprise Server not working"
1. `EnterpriseBaseURL` (API) and `EnterpriseUploadURL` must both be set correctly
2. Network egress from Mattermost to the GHES host must be open
3. The OAuth app must be registered on the GHES instance, not github.com

### "Permalink expansion slow / delays posts"
1. `MessageWillBePosted` is synchronous — slow GitHub API blocks posting
2. Usual culprit is connectivity between Mattermost and GitHub/GHES

## Where things live (cheat sheet)

| You're asking about... | Look at... |
|------------------------|------------|
| Why a webhook didn't post | `webhook.go` (`post<Type>Event`) + `subscriptions.go` + `template.go` |
| OAuth setup failures | `oauth.go`, `flows.go`, `plugin.go:getOAuthConfig` |
| Token decryption / key rotation | `plugin.go:reEncryptUserData`, `utils.go:decrypt`, `configuration.go:OnConfigurationChange` |
| Slash command behavior | `command.go` |
| Settings storage / validation | `configuration.go` |
| LHS sidebar counts (reviews/assignments/PRs) | `graphql/lhs_query.go`, `api.go`, `plugin.go:GetToDo` |
| Permalink preview behavior | `permalinks.go`, `plugin.go:MessageWillBePosted` |
| Review SLA / overdue digest | `review_sla.go`, `sla_digest.go`, `sla_digest_scheduler.go`, `graphql/digest_query.go` |
| Cluster sync (HA) | `cluster.go`, `plugin.go:OnPluginClusterEvent` |
| GitHub Enterprise targeting | `configuration.go:getBaseURL`, `EnterpriseBaseURL`/`EnterpriseUploadURL` |
| What's in the support packet | `support_packet.go` |
