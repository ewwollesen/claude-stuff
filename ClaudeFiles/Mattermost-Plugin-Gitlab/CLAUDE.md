# Mattermost GitLab Plugin Codebase Guide (Support Focus)

This guide helps navigate the Mattermost GitLab plugin codebase to answer support questions: where OAuth credentials are validated, how channel subscriptions filter events, how the encryption key is rotated, how the group lock (`GitlabGroup`) is enforced, and where webhook events are parsed.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/ClaudeFiles/Mattermost-Plugin-Gitlab/CLAUDE.md`

## Related Repositories

- **Mattermost Server**: `../Mattermost/` — plugin API (`server/public/plugin/api.go`), `pluginapi` client, KV store, cluster mutex, OAuth proxy (Chimera) config
- **Enterprise**: `../Enterprise/` — not used directly; the plugin is free/open source
- **Docs**: `../Mattermost-Docs/` — user-facing docs at `docs.mattermost.com/integrate/gitlab-interoperability.html`

This plugin was forked from `mattermost-plugin-github`; structure and patterns mirror that plugin closely.

## Repository Structure

```
server/                          # Go backend (package main)
├── plugin.go                    # Plugin struct, lifecycle hooks (OnActivate/OnInstall/etc.), KV helpers,
│                                # token refresh, re-encryption, group-lock enforcement (~1270 lines, central file)
├── configuration.go             # Plugin settings struct, validation, encryption key generation, key rotation hook
├── instance.go                  # NEW multi-instance model — InstanceConfiguration stored in KV
│                                # (`Gitlab_Instance_Configuration_Map`); preferred over legacy plugin settings
├── api.go                       # HTTP API endpoint handlers (OAuth callback, todo, issue create, subscriptions, etc.)
├── api_test.go
├── command.go                   # /gitlab slash command (connect, subscriptions, webhook, pipelines, etc.)
├── oauth.go                     # OAuth connect/complete flow, Chimera proxy support
├── flow.go                      # FlowManager — interactive setup wizard via plugin flows
├── subscriptions.go             # Channel subscription CRUD; namespace allow-listing
├── subscription/                # Subscription model (features bitfield, project/group path)
│   └── subscription.go          # Subscription struct, feature parsing ("merges,issues,pipeline,label:foo")
├── webhook.go                   # /webhook HTTP handler — receives GitLab webhook payloads
├── webhook/                     # Per-event-type handlers (parses GitLab payloads, emits Mattermost posts)
│   ├── issue.go                 # Issue events
│   ├── merge_request.go         # MR events
│   ├── note.go                  # Comment events
│   ├── push.go                  # Push events
│   ├── pipeline.go              # Pipeline events (incl. child pipelines)
│   ├── jobs.go                  # Job events
│   ├── deployment.go            # Deployment events
│   ├── release.go               # Release events
│   ├── tag.go                   # Tag push events
│   ├── constants.go             # Feature flag constants matching subscription features
│   └── webhook.go               # Webhook interface and entry dispatch
├── gitlab/                      # Thin wrapper around go-gitlab client
│   ├── gitlab.go                # Gitlab interface (mockable), UserInfo, settings, LHS data
│   ├── api.go                   # GitLab REST calls (projects, MRs, issues, hooks, etc.)
│   ├── user.go                  # GetLHSData (sidebar reviews/assignments/todos)
│   ├── pipeline.go              # Pipeline-related calls
│   └── webhook.go               # WebhookInfo type
├── permalinks.go                # MessageWillBePosted hook — expands GitLab file permalinks to code previews
├── reencrypt_test.go            # Token re-encryption tests (key rotation flow)
├── cluster.go                   # OnPluginClusterEvent — inter-node messages (e.g., subscription updates)
├── audit.go                     # Audit record helpers (key rotation, instance changes)
├── support_packet.go            # GenerateSupportData hook — what shows up in mm_support_packet
└── main.go                      # Entrypoint: plugin.ClientMain(NewPlugin())

webapp/                          # React/TypeScript frontend
├── src/
│   ├── index.ts                 # Plugin webapp entry — registers components, channel header,
│   │                            # sidebar (LHS), bot icon, slash command, post types
│   ├── components/              # UI components (sidebar, modals, settings, subscriptions modal)
│   ├── actions/                 # Redux thunks (fetch LHS data, connect, subscriptions CRUD)
│   ├── reducers/                # Plugin state (connected, LHS counts, settings)
│   ├── selectors/               # Selectors over plugin state
│   ├── client/                  # API client wrappers for plugin REST endpoints
│   ├── websocket/               # WebSocket event handlers (gitlab_connect, gitlab_refresh,
│   │                            # gitlab_disconnect, gitlab_channel_subscriptions_updated, create_issue)
│   ├── action_types/            # Redux action type constants
│   ├── constants/               # UI-side constants
│   ├── types/                   # TypeScript definitions
│   └── hooks/                   # React hooks
├── jest.config.json
└── webpack.config.js

plugin.json                      # Plugin manifest with settings schema (min_server_version, executables, settings UI)
Makefile
```

## Configuration

### Plugin settings (legacy + new instance model)

The plugin has **two parallel ways** to configure OAuth credentials, and this matters for support:

1. **Legacy plugin settings** (`configuration.go`): `GitlabURL`, `GitlabOAuthClientID`, `GitlabOAuthClientSecret`. Set via System Console under Plugins → GitLab.
2. **Instance configuration** (`instance.go`): KV-stored `InstanceConfiguration` keyed by name. The active default is `DefaultInstanceName` in plugin settings. This is the **new** model — `resolveOAuthCredentials` in `plugin.go` tries instance config first, then falls back to legacy fields for upgrades from v1.11 and earlier.

Key plugin settings (`server/configuration.go`):

| Setting | Purpose | Notes |
|---------|---------|-------|
| `GitlabURL` | Base URL for the GitLab installation | `https://gitlab.com` for SaaS, custom for self-hosted |
| `GitlabOAuthClientID` / `GitlabOAuthClientSecret` | OAuth app credentials | Required unless using Chimera; encrypted at rest is NOT applied to these (they live in plugin config) |
| `WebhookSecret` | Verifies inbound GitLab webhooks | Auto-generated; must match `X-Gitlab-Token` header |
| `EncryptionKey` | AES key used to encrypt stored user OAuth tokens | Auto-generated; rotated triggers full re-encryption |
| `PreviousEncryptionKey` | Internal-only fallback during key rotation | Not persisted; held in memory while `reEncryptUserData` runs |
| `GitlabGroup` | (Optional) Restrict the plugin to a single GitLab group | Enforced by `isNamespaceAllowed` in `plugin.go` |
| `EnablePrivateRepo` | Allow subscriptions to private repos | Per-deployment toggle |
| `EnableChildPipelineNotifications` | Post events for child pipelines too | Defaults true |
| `EnableCodePreview` | Permalink expansion: `disable` / `public` / `privateAndPublic` | Public is the default |
| `UsePreregisteredApplication` | Use Chimera-hosted OAuth (Cloud only) | Requires `PluginSettings.ChimeraOAuthProxyURL` on the server |

### Chimera proxy

When `UsePreregisteredApplication=true`, the plugin proxies OAuth through Chimera (`getOAuthConfigForChimeraApp` in `plugin.go`). Used on Mattermost Cloud so customers don't have to register their own GitLab OAuth app. The server-side Chimera URL is read from `PluginSettings.ChimeraOAuthProxyURL` or the `MM_PLUGINSETTINGS_CHIMERAOAUTHPROXYURL` env var.

## Storage Model (KV)

All persistent state lives in plugin KV. Keys (constants in `plugin.go`):

| Key suffix | Value | Notes |
|------------|-------|-------|
| `<userID>_userinfo` | `gitlab.UserInfo` JSON | NOT encrypted; holds GitLab username, ID, settings, last todo post time |
| `<userID>_usertoken` | Encrypted OAuth `*oauth2.Token` | AES-encrypted with `EncryptionKey` |
| `<userID>_gitlabtoken` | Legacy bundled token+info | Migrated to split keys above by `migrateGitlabToken` on first read |
| `<gitlabUsername>_gitlabusername` | Mattermost userID | Reverse lookup: GitLab → Mattermost |
| `<gitlabID>_gitlabidusername` | GitLab username | Reverse lookup: GitLab numeric ID → username |
| `subscriptions` | Channel subscriptions map (project path → []Subscription) | Single global key, mutated under cluster mutex |
| `Gitlab_Instance_Configuration_Map` | Map of instance name → `InstanceConfiguration` | New multi-instance model |
| `Gitlab_Instance_Configuration_Name_List` | Ordered list of instance names | Used for default-instance fallback |
| `<userID>-oauth-token` (mutex) | Cluster mutex lock for token refresh | Prevents concurrent refresh from multiple nodes |
| `gitlab-reencrypt-lock` (mutex) | Cluster mutex for key rotation | Only one node re-encrypts at a time |

## OAuth Flow

1. User runs `/gitlab connect` → redirected to `/oauth/connect` (in `oauth.go`)
2. Plugin builds OAuth state, redirects to GitLab authorize URL
3. GitLab redirects back to `/oauth/complete` with code
4. Plugin exchanges code for token, stores encrypted token + user info in KV
5. WebSocket `gitlab_connect` event broadcast to user

For Chimera/Cloud, steps 2–3 go through the Chimera proxy URL instead of GitLab directly.

## Token Refresh & Encryption

- **Refresh**: `getOrRefreshTokenWithMutex` in `plugin.go`. Acquires a per-user cluster mutex, refreshes if expiring within `tokenExpiryBuffer` (5 min). Uses `oauth2.ReuseTokenSourceWithExpiry` to align oauth2 lib's buffer with ours.
- **Invalid grant**: If refresh returns `invalid_grant`, `handleRevokedToken` disconnects the user and DMs them to reconnect.
- **Encryption**: `encrypt`/`decrypt` in `utils.go` (AES-GCM with `EncryptionKey`).
- **Key rotation**: Changing `EncryptionKey` in System Console triggers `reEncryptUserData` (in `plugin.go`). It:
  1. Acquires `gitlab-reencrypt-lock` cluster mutex
  2. Lists all `*_usertoken` keys via `pluginapi.WithChecker`
  3. For each key, decrypt with previous key → encrypt with new key → `KVCompareAndSet`
  4. On failure, `forceDisconnectUser` cleans up KV and DMs the user
  5. Idempotency check: if already decryptable with the new key, skip
  6. Writes an audit record (`reEncryptUserData` event) with migrated/disconnected counts

## Webhook Pipeline

1. GitLab sends POST to `/plugins/com.github.manland.mattermost-plugin-gitlab/webhook`
2. `handleWebhook` in `webhook.go` validates `X-Gitlab-Token` against `WebhookSecret`
3. Dispatches by event type to `server/webhook/*.go` handlers
4. Handler parses payload, matches against subscriptions (`subscriptions.go`)
5. For each matching subscription, builds a Mattermost post via `poster.Poster`
6. Group lock (`isNamespaceAllowed`) filters events outside the configured group

Subscription features are a comma-separated string (`merges,issues,pipeline,label:bug`). Constants in `server/webhook/constants.go` and the help text in `command.go`:

- `issues`, `merges` (MRs), `pushes`, `issue_comments`, `merge_request_comments`, `pipeline`, `tag`, `pull_reviews`, `merge_request_assigns`, `label:"<name>"`
- Default when none specified: `merges,issues,tag`

## Slash Commands

`/gitlab` subcommands (see `command.go`):

| Subcommand | Purpose |
|------------|---------|
| `connect` / `disconnect` | OAuth user link |
| `me` | Show connected GitLab account |
| `todo` | Post a DM with todos, reviews, assigned issues/MRs |
| `subscriptions list/add/delete` | Manage channel subscriptions |
| `webhook list/add` | Manage GitLab project/group webhooks via API |
| `pipelines run <repo> <ref>` | Trigger a GitLab pipeline |
| `settings` | Per-user notification settings (notifications on/off, reminders) |
| `setup` | Re-run the interactive setup wizard (FlowManager) |
| `about` | Build info |

## Group Lock (`GitlabGroup`)

When set, restricts the plugin to a single GitLab group. Enforced in `isNamespaceAllowed` (`plugin.go`):

```go
if namespace != allowedNamespace && !strings.HasPrefix(namespace, allowedNamespace+"/") { return ErrNamespaceNotAllowed }
```

Changing this setting triggers `notifyUsersOfDisallowedSubscriptions` — sweeps existing subscriptions, finds those outside the new allowed namespace, DMs the creator(s) listing the now-orphaned subscriptions per channel. The subscriptions themselves remain in KV but stop receiving events.

## Permalink Expansion

`MessageWillBePosted` hook in `plugin.go` matches `gitlabPermalinkRegex` (file blob URLs with line anchors), then `permalinks.go` fetches the file content via the poster's OAuth token and replaces the link with a Markdown code block. Gated by `EnableCodePreview`:
- `disable` — no expansion
- `public` (default) — only public projects
- `privateAndPublic` — all projects (warned: leaks code into channels)

## Support Packet

`generateSupportData` in `support_packet.go` is registered via the plugin API; output appears in `mm_support_packet/plugins/com.github.manland.mattermost-plugin-gitlab/` when an admin runs **Generate Support Packet**. Includes configuration snapshot, subscription counts, instance config (without secrets).

## Common Support Investigation Patterns

### "/gitlab connect doesn't work / OAuth redirect fails"
1. Check `canConnect` in `plugin.go` — requires either a valid instance config OR all of `GitlabOAuthClientID/Secret/URL` set
2. Verify `SiteURL` is set (plugin refuses to activate without it)
3. Check `oauth.go` `connectUserToGitlab` for state/cookie handling
4. If Chimera: verify `PluginSettings.ChimeraOAuthProxyURL` or the env var is set
5. Check that the GitLab OAuth app's redirect URI matches `<SiteURL>/plugins/com.github.manland.mattermost-plugin-gitlab/oauth/complete`

### "Webhook events not appearing in channel"
1. Verify `WebhookSecret` matches the `Secret token` configured in GitLab project/group webhooks
2. Check webhook URL: `<SiteURL>/plugins/com.github.manland.mattermost-plugin-gitlab/webhook`
3. Look in `webhook.go` `handleWebhook` for early-return paths (secret mismatch, unknown event type)
4. Check the per-event handler in `server/webhook/*.go` — feature filter logic lives there
5. Verify the channel actually has a subscription with the right feature: `/gitlab subscriptions list`
6. If group-locked: check `GitlabGroup` and `isNamespaceAllowed`

### "Token revoked / users keep getting disconnected"
1. Check server logs for `Revoking OAuth token` or `Failed to refresh OAuth token`
2. Look at `handleRevokedToken` callers — typically `invalid_grant` from GitLab or `invalidTokenError` (401) from API calls
3. Check whether the OAuth app was rotated on the GitLab side
4. Check encryption key rotation logs: `forceDisconnectUser` is called on re-encryption failure (`reEncryptUserData` audit record will say how many)

### "Encryption key rotation issues"
1. Logs: `Encryption key changed, re-encrypting user tokens`
2. Audit record `reEncryptUserData` shows `TotalUsers`, `MigratedCount`, `ForceDisconnectCount`
3. If users were force-disconnected: the old key was likely already wrong before rotation, or the KV entries were corrupt
4. `PreviousEncryptionKey` is held in memory only — restarting mid-rotation strands tokens encrypted with the old key

### "Permalink previews leak private code"
1. Check `EnableCodePreview` setting — `privateAndPublic` is the only mode that exposes private repos
2. `MessageWillBePosted` runs as the post author's OAuth token, so visibility matches that user's GitLab access

### "Subscriptions list is empty after upgrade"
1. KV key is the global `subscriptions` key — if missing, no subscriptions exist
2. Check `AddSubscription` for the path-normalization logic (`fullPathFromNamespaceAndProject`)
3. Group lock might be filtering them out at notification time — check `GitlabGroup` setting

### "Multi-instance configuration"
1. Default instance is `DefaultInstanceName` in plugin settings
2. Instance config is in KV at `Gitlab_Instance_Configuration_Map`
3. Falls back to legacy plugin settings (`GitlabOAuthClientID/Secret/URL`) if no instance is found — see `resolveOAuthCredentials`
4. Plugin currently uses one active instance at a time; the map exists for future multi-instance UX

### "Chimera proxy on Cloud"
1. Set `UsePreregisteredApplication=true` in plugin settings
2. Verify `chimeraURL` was registered: `registerChimeraURL` in `plugin.go`
3. OAuth flows through `getOAuthConfigForChimeraApp` instead of direct GitLab
4. Customer's GitLab app credentials are **not** needed — the Chimera proxy holds the centrally managed credentials

### "/gitlab webhook add fails"
1. Requires the user's OAuth token to have permission to create webhooks on the project/group
2. `command.go` `webhookCommand` → calls `GitlabClient.NewProjectHook` / `NewGroupHook` (in `server/gitlab/api.go`)
3. Check that the user is a maintainer/owner on the GitLab side

### "Permalink expansion timing out / slow posts"
1. `MessageWillBePosted` is **synchronous** — slow GitLab API calls block post creation
2. Network connectivity between Mattermost and GitLab is the usual culprit
3. Self-hosted GitLab behind firewall: check egress rules

## Where things live (cheat sheet)

| You're asking about... | Look at... |
|------------------------|------------|
| Why a webhook didn't post | `server/webhook/<event>.go` + `subscriptions.go` |
| OAuth setup failures | `oauth.go`, `flow.go`, `plugin.go:resolveOAuthCredentials` |
| Token decryption errors | `plugin.go:getGitlabUserTokenByMattermostID`, `utils.go:decrypt` |
| Slash command behavior | `command.go` |
| Settings storage / validation | `configuration.go`, `instance.go` |
| LHS sidebar counts (reviews, assignments, todos) | `server/gitlab/user.go:GetLHSData`, `plugin.go:GetToDo` |
| Permalink preview behavior | `permalinks.go`, `plugin.go:MessageWillBePosted` |
| Cluster sync (HA) | `cluster.go`, `plugin.go:OnPluginClusterEvent` |
| Audit records | `audit.go`, `plugin.go:reEncryptUserData` |
| What's in the support packet | `support_packet.go` |
