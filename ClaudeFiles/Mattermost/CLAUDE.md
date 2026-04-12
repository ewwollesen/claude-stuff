# Mattermost Codebase Guide (Support Focus)

This guide helps navigate the Mattermost codebase to answer support questions: finding where config is validated, where error messages originate, how features are gated, etc.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/ClaudeFiles/Mattermost/CLAUDE.md`

## Related Repository

The enterprise/commercial Mattermost code lives at `../Enterprise`. It implements the interfaces defined in `server/einterfaces/` and is compiled in via build tags. Its `CLAUDE.md` has a full guide to each enterprise feature package (LDAP, SAML, clustering, compliance, data retention, message export, etc.). When investigating enterprise features, you'll need both repos.

## Repository Structure

This is a full-stack monorepo:

```
server/                     # Go backend (PostgreSQL only, min v14)
  cmd/mattermost/           # Main server binary
  cmd/mmctl/                # CLI management tool
  channels/                 # Core server code
    api4/                   # REST API v4 handlers
    app/                    # Business logic
    store/                  # Database layer (interfaces + SQL impl)
    web/                    # HTTP handler framework, context, error handling
    db/migrations/postgres/ # Database migrations (morph-based, numbered .up/.down.sql)
  public/                   # Public Go module (imported by plugins & mmctl)
    model/                  # All data models, config structs, permissions
    plugin/                 # Plugin interfaces
    pluginapi/              # Plugin API helpers
    shared/i18n/            # i18n initialization and translation loading
  config/                   # Config store (file, database, memory backends)
  enterprise/               # Enterprise feature stubs & source-available code
  einterfaces/              # Enterprise feature interfaces (contracts)
  platform/                 # Shared platform services
  i18n/                     # Server translation JSON files (en.json, es.json, etc.)
  templates/                # Email and notification templates
  build/                    # Docker configs and build assets
webapp/                     # React/TypeScript frontend
  channels/src/i18n/        # Webapp translation JSON files
  platform/                 # Shared packages (Redux, types, components, API client)
api/v4/source/              # OpenAPI YAML specs for API documentation
e2e-tests/                  # Cypress & Playwright E2E tests
```

## Configuration System

### Where config is defined

- **Config struct**: `server/public/model/config.go` — ~30 nested sub-structs (ServiceSettings, SqlSettings, LdapSettings, etc.)
- **Default values**: Constants at top of same file, applied via `Config.SetDefaults()`
- **Validation**: `Config.IsValid()` in same file — delegates to each sub-struct's `isValid()` method

### How config is loaded

`server/config/store.go` — `Store.Load()` orchestrates:
1. Read from backing store (file JSON or database)
2. `SetDefaults()` fills nil pointers
3. `fixConfig()` normalizes paths/locales
4. `applyEnvironmentMap()` applies `MM_*` env var overrides
5. `IsValid()` validates everything
6. Emits change listener events

### Config backing stores

- **FileStore** (`server/config/file.go`) — reads/writes `config.json`
- **DatabaseStore** (`server/config/database.go`) — stores config in PostgreSQL
- **MemoryStore** (`server/config/memory.go`) — testing only

### Environment variable overrides

`server/config/environment.go` — any `MM_*` env var overrides the matching config field.
Pattern: `MM_SERVICESETTINGS_SITEURL` overrides `ServiceSettings.SiteURL`.
Env overrides are stripped before persisting config to storage.

### Sensitive data

`server/config/utils.go` — `desanitize()` restores real passwords/secrets when saving partial config from the UI. `Config.Sanitize()` redacts secrets before sending to clients.

## Error Messages and i18n

### How errors work

- **AppError struct**: `server/public/model/utils.go` — contains `Id` (translation key), `Message` (translated text), `DetailedError`, `StatusCode`
- **Creating errors**: `model.NewAppError(where, translationId, params, details, statusCode)`
- **Translation IDs** follow the pattern: `api.<domain>.<operation>.app_error`

### Where translations live

- **Server**: `server/i18n/en.json` — thousands of entries, array of `{id, translation}` objects
- **Webapp**: `webapp/channels/src/i18n/en.json` — flat key-value JSON
- **i18n init**: `server/public/shared/i18n/i18n.go` — loads translations, manages locales

### Common error helpers

`server/channels/web/context.go`:
- `NewInvalidParamError()` — bad request body param
- `NewInvalidURLParamError()` — bad URL param
- `SetPermissionError()` — permission denied
- `NewServerBusyError()` — service unavailable

### Finding an error message by its ID

1. Search `server/i18n/en.json` for the error ID to find the English text
2. Grep for the error ID in `server/channels/` to find where it's raised
3. The `Where` field in the error tells you the originating function

## API Layer

### Route definitions

- **Router setup**: `server/channels/api4/api.go` — `Routes` struct defines all route groups, `Init()` registers everything
- **Handler files**: `server/channels/api4/{resource}.go` (e.g., `channel.go`, `user.go`, `post.go`, `system.go`)
- **Each resource** has an `Init{Resource}()` function that registers routes on the appropriate subrouter

### Handler types (server/channels/api4/handlers.go)

| Wrapper | Auth Required | Notes |
|---------|--------------|-------|
| `APIHandler` | No | Public endpoints |
| `APISessionRequired` | Yes | Most endpoints |
| `APISessionRequiredMfa` | Yes (MFA optional) | Login flow |
| `APILocal` | No | Local mode only (Unix socket) |
| `CloudAPIKeyRequired` | Cloud key | Cloud-specific |
| `RemoteClusterTokenRequired` | Remote token | Shared channels |

### Permission checks

- **Per-endpoint**: Handlers call `c.App.SessionHasPermissionTo(session, permission)` and variants
- **Permission constants**: `server/public/model/permission.go` — 350+ permission definitions
- **Authorization helpers**: `server/channels/app/authorization.go`

### Finding an API endpoint

1. Identify the resource (users, channels, teams, etc.)
2. Open `server/channels/api4/{resource}.go`
3. Look at the `Init{Resource}()` function for route registration
4. Find the handler function referenced in the route

## Feature Flags

- **Definition**: `server/public/model/feature_flags.go` — `FeatureFlags` struct with bool/string fields
- **Defaults**: `SetDefaults()` method in the same file
- **Runtime access**: `config.FeatureFlags.FlagName` via the config store
- **Sync with Split.io**: `server/channels/app/featureflag/feature_flags_sync.go` — polls external service on cluster leader
- **Client exposure**: `server/config/client.go` — flags sent as `FeatureFlag{Name}` in client config
- **Platform setup**: `server/channels/app/platform/feature_flags.go` — starts/stops sync based on cluster role

## Enterprise / License System

### How enterprise code integrates

The open-source repo defines **interfaces** that enterprise code implements:

1. **Interfaces**: `server/einterfaces/` — contracts for LDAP, SAML, compliance, clustering, etc.
2. **Registration**: `server/channels/app/enterprise.go` — `Register{Feature}Interface()` functions
3. **Enterprise stubs**: `server/enterprise/` — build-tag-gated imports:
   - `//go:build enterprise` → imports from private `github.com/mattermost/enterprise` repo
   - `//go:build enterprise || sourceavailable` → local source-available implementations
   - No tag → community edition only
4. **Wiring**: Enterprise packages call `Register*()` in their `init()` functions

### License model

- **Struct**: `server/public/model/license.go` — `License` with `Features` sub-struct (booleans for each feature)
- **SKUs**: E10, E20, Professional, Enterprise, Enterprise Advanced, Mattermost Entry
- **Loading**: `server/channels/app/platform/license.go` — checks `MM_LICENSE` env var, then database, then disk file
- **Validation**: `server/channels/utils/license.go` — RSA-256 signature verification

### License checks in code

API handlers gate features like:
```go
if c.App.Channels().License() == nil || !*c.App.Channels().License().Features.LDAP {
    c.Err = model.NewAppError(...)
    return
}
```

Search for `License().Features.` to find all license-gated code paths.

### Plugin license helpers

`server/public/pluginapi/license.go`:
- `IsEnterpriseLicensedOrDevelopment()`
- `IsE10LicensedOrDevelopment()`
- `IsE20LicensedOrDevelopment()`

## Database / Store Layer

### Architecture

```
App Layer (server/channels/app/)
  -> Store Interface (server/channels/store/store.go) — 60+ sub-interfaces
    -> Timer Layer (metrics)
    -> Retry Layer (deadlock handling)
    -> Local Cache Layer (in-memory cache)
    -> Search Layer (Elasticsearch/Bleve)
    -> SQL Store (server/channels/store/sqlstore/) — 136 implementation files
```

### Key locations

- **Interfaces**: `server/channels/store/store.go` — Store, UserStore, ChannelStore, PostStore, etc.
- **SQL implementations**: `server/channels/store/sqlstore/{entity}_store.go`
- **Migrations**: `server/channels/db/migrations/postgres/` — numbered up/down SQL files
- **Store errors**: `server/channels/store/errors.go` — ErrInvalidInput, ErrNotFound, ErrConflict, ErrLimitExceeded, ErrUniqueConstraint

### Query building

Uses Squirrel library (`github.com/mattermost/squirrel`) for SQL query construction.

## mmctl CLI

### Location and structure

`server/cmd/mmctl/` — built with Cobra CLI framework

- **Entry point**: `mmctl.go` → `commands.Run()`
- **Commands**: `commands/` — ~46 command files (user.go, channel.go, config.go, system.go, etc.)
- **Client interface**: `client/client.go` — 180+ API methods wrapping `model.Client4`
- **Errors**: `commands/errors.go` — ErrEntityNotFound, NotFoundError, BadRequestError
- **Output**: `printer/` — supports plain text and JSON output formats

### Connection modes

- **Remote**: Token-based auth to server API (default)
- **Local**: Unix socket connection (`--local` flag), bypasses auth

### Building mmctl

```bash
cd server && make mmctl-build
# Cross-compile: GOOS=linux GOARCH=amd64 go build -trimpath -o bin/mmctl-linux-amd64 ./cmd/mmctl
```

## Session / Authentication System

### Token parsing

`server/channels/app/authentication.go` — `ParseAuthTokenFromRequest()` extracts tokens from incoming requests in this priority order:
1. Session cookie (`MMAUTHTOKEN`)
2. `Authorization: Bearer <token>` header (session tokens)
3. `Authorization: Token <token>` header (OAuth tokens)
4. `access_token` query parameter
5. Cloud token header → routed to `GetCloudSession()` (separate path)
6. Remote cluster token header → routed to `GetRemoteClusterSession()` (separate path)

### Session resolution

`server/channels/app/session.go` — `GetSession()`:
1. Checks session cache/DB via `platform.GetSession()`
2. If not found, **falls through** to `createSessionForUserAccessToken()` as a catch-all
3. `createSessionForUserAccessToken()` looks up the token in the `UserAccessTokens` table
4. If that also fails, logs a warning mentioning "user access token" — **this is misleading** because any token type (session cookie, OAuth, WebSocket) that fails step 1 ends up here

### Token types

All token types use the same `model.NewId()` format (26-char alphanumeric). You cannot distinguish token types by format alone. Session tokens, personal access tokens, and OAuth tokens all look identical.

### HTTP handler routing

`server/channels/web/handlers.go` — routes tokens to the appropriate session handler:
- Cloud tokens → `GetCloudSession()`
- Remote cluster tokens → `GetRemoteClusterSession()`
- Everything else → `GetSession()` (the fallback chain above)

## WebSocket Layer

### Connection and auth

- **Router**: `server/channels/app/platform/websocket_router.go` — handles incoming WebSocket messages
- **Auth**: The `authentication_challenge` action passes the token to `GetSession()` (same as REST API)
- WebSockets reuse the session token from the user's original login — no separate WebSocket token
- On connection drop, clients reconnect with their existing token without re-authenticating

### Hub system

- **Web hub**: `server/channels/app/platform/web_hub.go` — manages WebSocket connections and message broadcasting
- Connections register with the hub after successful auth (`HubRegister`)
- WebSocket events are broadcast types like `WebsocketEventPostEdited`, `WebsocketEventPostPosted`, etc.

### Common log patterns after server restart

After restart, all WebSocket clients try to reconnect with stale tokens, generating:
1. `"Error while creating session for user access token"` (session.go) — misleading, see above
2. `"Error while getting session token"` (websocket_router.go) — confirms it's a WebSocket reconnection
3. These are harmless — clients eventually get 401'd and re-authenticate

## Notification System

### Post creation vs update

- **CreatePost** (`server/channels/app/post.go`): Calls `handlePostEvents()` → `SendNotifications()` — triggers email, push, desktop notifications
- **UpdatePost** (`server/channels/app/post.go`): Only broadcasts a `WebsocketEventPostEdited` for real-time UI refresh. **Does not call SendNotifications** — no email/push/desktop notifications on edits
- **SendNotifications**: `server/channels/app/notification.go` — the main notification dispatch function

### Plugin hooks for posts

Defined in `server/public/plugin/hooks.go`:
- `MessageHasBeenPosted` — fires after CreatePost (new messages)
- `MessageHasBeenUpdated` — fires after UpdatePost, receives both `newPost` and `oldPost`
- `MessageWillBeUpdated` — fires before UpdatePost, can modify/reject the edit

### Thread following

- **API**: `PUT /api/v4/users/{user_id}/teams/{team_id}/threads/{thread_id}/following` (follow) / `DELETE` (unfollow)
- **Handler**: `server/channels/api4/user.go` — `followThreadByUser` / `unfollowThreadByUser`
- **Permission**: Requires `SessionHasPermissionToUser` — a user can only follow/unfollow threads for themselves, bots cannot follow threads on behalf of other users
- **Not in plugin API**: No `UpdateThreadFollowForUser` or thread follower query methods available to plugins

## Plugin API

### Available methods

`server/public/plugin/api.go` — the full interface plugins can call via RPC. Key methods:
- `CreatePost` — creates a post (triggers full notification pipeline)
- `SendEphemeralPost` — visible to one user only
- `GetPostThread` — gets all posts in a thread (but not who follows it)
- `SendMail` — send email to a specific address
- `SendPushNotification` — push notification to a specific user (v9.0+)

### Convenience wrappers

`server/public/pluginapi/` — higher-level wrappers around the plugin API (post.go, email.go, user.go, etc.)

### Limitations

- No access to `SendNotifications()` (the app's internal notification dispatch)
- No thread follower/subscriber queries
- No way to follow/unfollow threads on behalf of users
- Cannot call app-layer functions directly — all interaction is via the RPC plugin API interface

## Email System

- **Mail package**: `server/platform/shared/mail/mail.go` — constructs and sends emails via SMTP
- **Message-ID**: Always included. Uses provided `messageID` or auto-generates `<{random16}@{hostname}>`
- **In-Reply-To**: Supported for threading notification emails in mail clients
- **Templates**: `server/templates/` — Go HTML templates for email content
- **Email service**: `server/channels/app/email/` — higher-level email functions (invites, notifications, password resets)

## Cloud / CWS System

### Config

`server/public/model/config.go` — `CloudSettings` struct:
- `CWSURL` — Customer Web Server URL (default: `https://customers.mattermost.com`)
- `CWSAPIURL` — CWS API URL
- `CWSMock` — mock mode for testing
- `Disable` — disables all CWS interactions (`MM_CLOUDSETTINGS_DISABLE=true`)

### CWS availability check

- **Endpoint**: `GET /api/v4/cloud/check-cws-connection` → `handleCheckCWSConnection` in `server/channels/api4/cloud.go`
- **Gate**: `ensureCloudInterface()` checks two things before the actual CWS call:
  1. Cloud interface is registered (enterprise code compiled in)
  2. `CloudSettings.Disable` is not `true`
- **Enterprise impl**: `CheckCWSConnection` in `server/einterfaces/cloud.go` — actual connectivity check
- **Webapp**: `webapp/channels/src/components/common/hooks/useCWSAvailabilityCheck.ts` — checks CWS status, shows air-gapped modal if unavailable

## Team Domain Restrictions

### How domain filtering works

- **Team-level**: `Team.AllowedDomains` field — space/comma-delimited domain list
- **Server-level**: `TeamSettings.RestrictCreationToDomains` config setting
- **Check function**: `server/channels/app/teams/utils.go` — `IsTeamEmailAllowed()`
- **Bot exemption**: Bots (`user.IsBot == true`) bypass domain checks entirely — first check in `IsTeamEmailAllowed()`
- **Guest handling**: Guests check against `GuestAccountsSettings.RestrictCreationToDomains` instead

### Where domain checks happen

- `JoinUserToTeam` → `IsTeamEmailAllowed()` → error `api.team.join_user_to_team.allowed_domains.app_error`
- `CreateTeamWithUser` → `IsTeamEmailAllowed()` → error `api.team.is_team_creation_allowed.domain.app_error`
- `CreateUserWithInviteId` in `user.go` → `CheckUserDomain()` directly — **no bot exemption** in this path
- Both error IDs produce the same user-facing message: "The user cannot be added as the domain associated with the account is not permitted"

## Common Support Investigation Patterns

### "Where is config setting X validated?"
1. Find the field in `server/public/model/config.go` (search for the setting name)
2. Look for the parent struct's `isValid()` method in the same file
3. Check `SetDefaults()` for the default value

### "What does error message Y mean?"
1. Search `server/i18n/en.json` for the error ID
2. Grep for the error ID in `server/channels/` to find where it's raised
3. Read the surrounding code for context on what triggers it

### "Is feature Z enterprise-only?"
1. Check `server/public/model/license.go` for the feature in the `Features` struct
2. Grep for `License().Features.{FeatureName}` to find where it's checked
3. Check `server/einterfaces/` for the enterprise interface definition

### "Where is API endpoint W handled?"
1. Check `server/channels/api4/{resource}.go`
2. Look at the `Init{Resource}()` function for route registration
3. Follow the handler function for business logic

### "What does mmctl command V do?"
1. Find `server/cmd/mmctl/commands/{resource}.go`
2. Look at the Cobra command definition for usage/description
3. Follow the `RunE` function for implementation

### "Why are there 'user access token' warnings in the logs?"
1. Check if they coincide with a server restart (search for "Starting Server" or "Current version")
2. Look for the companion `websocket_router.go` "Error while getting session token" line — confirms WebSocket reconnection
3. If the same token repeats on a schedule (e.g., every 5 min), it's an integration with a stale token
4. If many different tokens appear in a burst after restart, it's the normal reconnection storm — harmless

### "Why didn't editing a post send notifications?"
1. `UpdatePost` in `server/channels/app/post.go` does NOT call `SendNotifications`
2. Only `CreatePost` triggers the full notification pipeline
3. Workaround: have the bot post a reply in the thread when done — `CreatePost` with `root_id` triggers notifications to thread followers

### "Can a bot join a team with domain restrictions?"
1. Check `IsTeamEmailAllowed()` in `server/channels/app/teams/utils.go` — bots are exempted
2. But `CreateUserWithInviteId` in `user.go` calls `CheckUserDomain()` directly with NO bot exemption
3. Verify the account is actually a bot (`is_bot: true`) vs a regular user account used like a bot
