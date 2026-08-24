# Mattermost Confluence Plugin Codebase Guide (Support Focus)

This guide helps navigate the Mattermost Confluence plugin codebase to answer support questions: why channel subscriptions aren't firing, which of the three event-delivery paths a customer is actually on (Server webhook / legacy Cloud Connect webhook / Forge bridge polling), why `/confluence connect` fails, why a v9+ Confluence Server behaves differently from v8, why @-mention DMs don't arrive, and how the Forge shared secret gets out of sync.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/ClaudeFiles/Mattermost-Plugin-Confluence/CLAUDE.md`

Plugin ID: `com.mattermost.confluence`. Bot username: `confluence`. `min_server_version`: **10.7.0**. Default branch is `master` (not `main`).

## Related Repositories

- **Mattermost Server**: `../Mattermost/` — plugin API (`server/public/plugin/api.go`), `pluginapi` client, KV store, `pluginapi/experimental/flow` (the setup wizard framework this plugin builds all its DM wizards on), `pluginapi/experimental/cluster` (`cluster.Schedule` for the Forge poller)
- **Jira plugin** (not cloned here) — `store.AtomicModify` is lifted verbatim from `mattermost-plugin-jira`; the subscription storage model is a direct descendant of Jira's
- **Docs**: `../Mattermost-Docs/` — **there is no Confluence page in the docs site.** The only admin documentation is `docs/admin-guide.md` in this repo, and it is **stale** (see gotchas below). Do not send customers to docs.mattermost.com for Confluence setup.
- **mattermost-for-confluence** (separate repo, not cloned) — the Confluence-side OBR/JAR app used only by Confluence Server **< 9**

## Repository Structure

```
server/                              # Go backend (package main)
├── plugin.go                        # Plugin struct, OnActivate (bot, router, templates, flow manager,
│                                    # command, Forge poller), OnConfigurationChange (auto-generates the
│                                    # three 32-char secrets), ServeHTTP (501s if config invalid),
│                                    # ExecuteCommand, OnDeactivate (stops the poller)
├── controller.go                    # Endpoint struct + Endpoints map, InitAPI (gorilla/mux under
│                                    # /api/v1), checkAuth (Mattermost-User-Id header only),
│                                    # handleStaticFiles, IsAdmin, verifyHTTPSecret (constant-time
│                                    # compare with a repeated URL-unescape loop)
├── http.go                          # loadTemplates, respondTemplate, splitInstancePath
├── command.go                       # ConfluenceCommandHandler — path-joined subcommand map
│                                    # ("settings/notifications/on" etc.), autocomplete data,
│                                    # connect/disconnect/list/unsubscribe/install/help/forge reset
├── flow.go                          # FlowManager — 5 DM wizards: setup, cloud-setup, completion,
│                                    # cloud-completion, announcement. Steps + dialog submit handlers.
│                                    # isForgeWebtriggerURL, postForgeRegister live here
├── forge_poller.go                  # ForgePoller — cluster.Schedule every 30s, HMAC-SHA256-signs the
│                                    # drain request body (X-MM-Signature), dispatches + acks events,
│                                    # alertOnce/dmSysadmins on 401/403/404/503
├── forge_reset.go                   # /confluence forge reset — rotates the shared secret in place,
│                                    # self-heals a missing ForgeResetURL via the register webtrigger
├── forge_event_mapping.go           # forgeToInternalEvent: avi:confluence:* -> internal event names.
│                                    # Page + comment only — NO space events on the Forge path
├── confluence_cloud.go              # POST /api/v1/cloud/{event} — legacy Atlassian Connect webhook
├── confluence_server.go             # POST /api/v1/server/webhook — Server/DC webhook. Branches on
│                                    # ServerVersionGreaterthan9. Admin-API-token fallbacks,
│                                    # respondToTestConnection, dispatchServerMentionDMs*
├── confluence_server_v2.go          # ConfluenceServerEvent accessors + GetNotificationPost message
│                                    # templates for the v9+ enriched path
├── notification.go                  # v9+ notification dispatch (needs a botUserID + triggerer name),
│                                    # SendGenericWHNotification (page events only), eventActions map
├── oauth2.go                        # Route constants + the three OAuth endpoints
├── user.go                          # httpOAuth2Connect/Complete, CompleteOAuth2 -> completeServerOAuth2
│                                    # or completeCloudOAuth2, connectUser/disconnectUser,
│                                    # refreshAndStoreToken, httpGetUserInfo, hasChannelAccess,
│                                    # validateUserConfluenceAccess (live space/page permission probe)
├── auth_token.go                    # AES-GCM encrypt/decrypt of the oauth2.Token using EncryptionKey
├── instance_cloud.go                # Atlassian Cloud 3LO endpoints, scopes, accessible-resources,
│                                    # cloudId -> https://api.atlassian.com/ex/confluence/<cloudId>
├── instance_server.go               # Server OAuth (<instance>/rest/oauth2/latest/{authorize,token}),
│                                    # scopes ADMIN for admins vs READ+WRITE for users
├── client.go / client_cloud.go / client_server.go
│                                    # Client interface + the two impls. Server client uses
│                                    # /rest/api/{user,space,content,user/current}. Cloud client
│                                    # implements GetSelf only — the rest are stubs
├── save_subscription.go             # POST /{channelID}/subscription/{type}
├── edit_subscription.go             # PUT  /{channelID}/subscription/{type}
├── get_subscription.go              # GET  /{channelID}/subscription?alias=
├── get_subscriptions.go             # GET  /autocomplete/GetChannelSubscriptions
├── plugin_config.go                 # GET  /config — returns the version-appropriate event list to the UI
├── config/
│   ├── main.go                      # Configuration struct, atomic Get/SetConfig, IsValid, Sanitize,
│   │                                # package-level `Mattermost plugin.API` and `BotUserID` globals
│   └── manifest.go                  # generated: PluginName = "com.mattermost.confluence"
├── serializer/
│   ├── channel_subscription.go      # Subscription interface, BaseSubscription, Subscriptions (the
│   │                                # 3-index blob), supported-event lists per Confluence version,
│   │                                # FormattedSubscriptionList (the `/confluence list` markdown table)
│   ├── space_subscription.go        # SpaceSubscription Add/Remove/Edit/IsValid/ValidateSubscription
│   ├── page_subscription.go         # PageSubscription — same shape, keyed by pageID
│   ├── confluence_cloud.go          # Legacy Connect webhook payload + message templates
│   ├── confluence_server.go         # ConfluenceServerWebhookPayload (+ the pre-v9 event shape)
│   └── confluence_forge.go          # ForgeEvent/ForgeContent, forgeID (string-or-number), message
│                                    # templates, MentionPageContext. BaseURL injected by the poller
├── service/
│   ├── notification.go              # SendConfluenceNotifications — the pre-v9/Cloud/Forge dispatch path
│   ├── get_subscription_list.go     # GetSubscriptions + the three lookup helpers
│   ├── save_subscription.go         # SaveSubscription -> AtomicModify CAS
│   ├── edit_subscription.go / delete_subscription.go / get_subscription.go
│   ├── interfaces.go + repository_impl.go + store_impl.go + mocks/
│   │                                # SubscriptionRepository / Store seams for tests (mockgen)
│   ├── mentions.go                  # SendMentionDMs — Confluence account -> MM user -> bot DM
│   ├── mention_settings.go          # KV toggle mm_mention_notif_<userID>, DEFAULT ON
│   ├── adf.go                       # ExtractMentionAccountIDsFromADF (Cloud/Forge bodies)
│   ├── storage_xhtml.go             # ExtractMentionAccountIDsFromStorage (Server storage format, regex)
│   └── confluence_service.go        # NormalizeConfluenceURL, CheckConfluenceURL (/status must be
│                                    # "RUNNING"), CallJSON plumbing
├── store/store.go                   # All KV access: subscriptions key, connection + reverse mapping,
│                                    # user record, OAuth2 state, AtomicModify (5 retries, 30ms apart)
└── util/
    ├── util.go                      # GetKeyHash, SplitArgs (quote-aware), GetPluginURL,
    │                                # GetConfluenceServerWebhookURLPath, IsSystemAdmin, Deduplicate,
    │                                # GetBodyForExcerpt (HTML -> text)
    └── types/connection.go          # User, ConfluenceUser, Connection, ConfluenceAccountID()

forge/                               # Atlassian Forge app (TypeScript) — the current Cloud event path
├── manifest.yml                     # 8 avi:confluence:* triggers -> enqueueFn; webtriggers
│                                    # drain/register/reset; wipeRegistrationFn (break-glass, no trigger)
├── src/index.ts                     # enqueue (enrichWithBody via /wiki/api/v2, 240KB storage cap),
│                                     # drain, register (one-shot per secret, 409 on mismatch),
│                                     # reset, wipeRegistration, onInstalled
└── scripts/validate-manifest.ts     # CI check that manifest handlers match exported functions

webapp/src/                          # React/Redux — JS (not TS), no channel-header icon, no RHS
├── index.js                         # Registers ONLY a reducer, a root component (the subscription
│                                    # modal), and a slashCommandWillBePostedHook
├── hooks/index.js                   # Intercepts `/confluence subscribe` and `/confluence edit` to open
│                                    # the modal client-side after checking /user-connection-info
├── components/subscription_modal/   # The single UI surface
├── client/client.js                 # Calls /plugins/com.mattermost.confluence/api/v1
└── constants/index.js               # CONFLUENCE_EVENTS (8 events — note: no space_updated),
                                     # SUBSCRIPTION_TYPE, error strings

assets/templates/                    # api/v1/oauth2/complete.html, other/message.html + index.css
docs/admin-guide.md                  # STALE — see gotchas
plugin.json                          # Manifest + settings_schema (all secrets declared so support
                                     # packets redact them — MM-69141)
```

## Event Delivery: three separate paths

This is the single most important thing to establish on any Confluence ticket. **Ask which path the customer is on before anything else.**

| Path | Trigger | Entry point | Direction | Notes |
|------|---------|-------------|-----------|-------|
| **Confluence Server / DC < 9** | Confluence-side OBR app posts a webhook | `POST /api/v1/server/webhook?secret=` → `serializer.ConfluenceServerEventFromJSON` → `service.SendConfluenceNotifications` | inbound | Payload is self-describing; no user connection needed |
| **Confluence Server / DC ≥ 9** | Native Confluence webhook | same route, but `ServerVersionGreaterthan9` branch → enrich via REST → `notification.SendConfluenceNotifications` | inbound + outbound REST | Needs either the event-triggering user connected via OAuth **or** `AdminAPIToken` set |
| **Confluence Cloud (current)** | Forge app enqueues into Forge storage; **Mattermost polls** | `ForgePoller.drainOnce` every 30s → `forgeToInternalEvent` → `service.SendConfluenceNotifications` | **outbound only** | No inbound webhook. Firewall/proxy must allow egress to `*.webtrigger.atlassian.app` |
| **Confluence Cloud (legacy Connect)** | Atlassian Connect descriptor webhook | `POST /api/v1/cloud/{event}?secret=` | inbound | Route still exists and still works, but **Atlassian killed the Connect-descriptor install path on 2026-03-31** (see the comment in `command.go`). New installs cannot use it. |

`ServerVersionGreaterthan9` is a **bool stored in plugin config**, set by the setup wizard from the admin's answer — not detected. If an admin answers wrong, or upgrades Confluence past 9 without re-running `/confluence install server`, the plugin uses the wrong parsing and enrichment path.

## Configuration

Set in **System Console → Plugins → Confluence**, but most values are written by the setup wizards, not typed by hand.

| Setting | Type | Purpose |
|---------|------|---------|
| `Secret` | generated, 32 chars | Webhook shared secret. Appended as `?secret=` to both the Server and legacy Cloud webhook URLs. Regenerating it **invalidates every configured Confluence-side webhook**. |
| `EncryptionKey` | generated, 32 chars | AES-GCM key for the per-user stored OAuth token (`auth_token.go`). **Changing it makes every existing connection undecryptable** — all users must `/confluence connect` again. |
| `AdminAPIToken` | text, secret | Confluence Data Center PAT. Used on the v9+ path when the event-triggering user isn't connected, so a detailed (rather than generic) notification can still be sent. Validated as ≥32 chars if non-empty. Must belong to a Confluence **admin** or it can't read restricted spaces/pages. |
| `ConfluenceURL` | text | The instance base URL. Written by the wizard, trailing slash trimmed by `Sanitize()`. Doubles as the **instance ID** in every KV key. |
| `ConfluenceOAuthClientID` / `Secret` | text | OAuth app credentials. Cloud = Atlassian developer console 3LO app; Server = an incoming Application Link. |
| `ForgeSharedSecret` | generated, 32 chars | HMAC key for drain/reset requests to the Forge bridge. Auto-regenerated whenever it isn't exactly 32 chars. |
| `ForgeDrainURL` / `ForgeRegisterURL` / `ForgeResetURL` | text, secret | Forge webtrigger URLs. Drain + register are pasted by the admin in the wizard dialog; reset comes back from the register response. Validated against `*.webtrigger.atlassian.app`. |
| `ForgeInstallURL` | text, secret | Optional. If set, the wizard renders a click-through install link instead of pointing at the self-host runbook. |
| `IsCloud` | bool (not in settings_schema) | Set by the cloud wizard. Decides whether `/confluence connect` targets Atlassian 3LO or the Server's local OAuth endpoints. |

### Validation and the 501 wall

`Configuration.IsValid()` requires `Secret` **exactly** 32 chars and `EncryptionKey` **exactly** 32 chars. `ServeHTTP` calls `IsValid()` on **every request** and returns **HTTP 501 "This plugin is not configured."** if it fails — which takes down the webhook endpoints *and* the OAuth endpoints *and* the flow wizard's own HTTP handlers. Log line: `This plugin is not configured.`

`OnConfigurationChange` auto-generates any of `Secret` / `EncryptionKey` / `ForgeSharedSecret` that isn't exactly 32 chars and calls `SavePluginConfig`, so a truncated or hand-edited value silently becomes a *new* value. Watch for `Auto-generated missing <name>.` in the logs — that is the fingerprint of a broken integration that "worked yesterday."

## Storage Model (KV)

Everything lives in plugin KV. There is no database table and no filestore usage.

| Key | Value | Written by |
|-----|-------|-----------|
| `base64(sha256("confluence_subs"))` | **All** subscriptions for the whole server, as one JSON blob | `store.GetSubscriptionKey()` + `AtomicModify` |
| `<ConfluenceURL>_<mattermostUserID>` | `types.Connection` (encrypted OAuth token, accountId, isAdmin) | `store.StoreConnection` |
| `<ConfluenceURL>_<confluenceAccountID>` | the Mattermost user ID (reverse mapping) | `store.StoreConnection` — **skipped** when the MM user is the `"admin"` sentinel |
| `user_<mattermostUserID>` | `types.User` (which instance URL they're connected to) | `store.StoreUser` |
| `ots_<state>` | OAuth2 state, 15-minute expiry | `store.StoreOAuth2State` |
| `mm_mention_notif_<mattermostUserID>` | `"1"` / `"0"` — absent means **enabled** | `service.SetMentionNotificationEnabled` |
| `forge_alert_key` | last Forge alert category, for de-duplicating sysadmin DMs | `ForgePoller.storeAlertKey` |

### The subscriptions blob

`serializer.Subscriptions` holds three indexes of the same data, all in the one blob:

- `ByChannelID[channelID][alias]` → the Subscription (drives `/confluence list` and edit)
- `ByURLSpaceKey["confluence_subs/<hostname>/<spaceKey>"][channelID]` → `[]event`
- `ByURLPageID["confluence_subs/<hostname>/<pageID>"][channelID]` → `[]event`

Note the lookup keys use **only the hostname** of the base URL (`url.Parse(...).Hostname()`), so path-based Confluence installs (`https://host/confluence`) collapse to the same key — but the subscription's stored `BaseURL` is the full URL, and event dispatch matches on `event.GetURL()` which is `ConfluenceURL`. A mismatch between the subscription's `BaseURL` and the configured `ConfluenceURL` is a common silent-failure cause.

Writes go through `store.AtomicModify` (KVCompareAndSet, 5 attempts, 30ms apart). Under contention it fails with `reached write attempt limit`. Because the entire server's subscriptions are one key, high-churn environments can genuinely contend here.

## Permissions Model

Stricter than most plugins, and it changes with the Confluence version.

| Action | Requirement |
|--------|-------------|
| `/confluence install *`, `/confluence forge reset` | System Admin |
| Save / edit / get subscription, autocomplete list | **System Admin** (hard-coded `util.IsSystemAdmin`) **and** channel membership (`hasChannelAccess` → `GetChannelMember`) |
| Any of the above on **Confluence ≥ 9** | Additionally: the acting admin must be OAuth-connected, **and** `validateUserConfluenceAccess` does a live `GetSpaceData`/`GetPageData` call — a 403 there is surfaced as "User does not have an access to this Confluence space/page" |
| `/confluence list` | System Admin on < 9; **any connected user** on ≥ 9 |
| `/confluence unsubscribe` | System Admin, plus connected on ≥ 9 |
| `/confluence connect`, `settings notifications` | Any user |

`checkAuth` only checks that `Mattermost-User-Id` is present — every per-endpoint authorization is inside the handler.

## Supported Events

Internal names (`serializer/channel_subscription.go`): `page_created`, `page_updated`, `page_trashed`, `page_restored`, `page_removed`, `comment_created`, `comment_updated`, `comment_removed`, `space_updated`.

- **`comment_removed` is not supported on Confluence Server ≥ 9** (`SupportedEventsV9AndAbove` omits it). Saving a subscription with it returns `event 'comment_removed' is not supported by the current Confluence Server version`.
- **`space_updated` is effectively dead.** It's in neither supported-events list, so the UI never offers it, and both dispatchers bail when `pageID == ""` — which is always true for a space event. Do not promise space-level notifications.
- The **Forge path carries no space events at all** — `forgeToInternalEvent` maps page and comment triggers only.
- `GET /api/v1/config` is what the webapp uses to render the correct event checkboxes, so a wrong `ServerVersionGreaterthan9` shows the wrong list.

## @-Mention DMs

Separate feature from channel subscriptions; a user can get one and not the other.

| Path | Mention source | Extractor |
|------|----------------|-----------|
| Forge / Cloud | ADF body attached by the Forge app's `enrichWithBody` | `service.ExtractMentionAccountIDsFromADF` |
| Server ≥ 9, user connected | `/rest/api/content/<id>?expand=body.storage` via the user's token | `service.ExtractMentionAccountIDsFromStorage` (regex on `ri:account-id` / `ri:userkey`) |
| Server ≥ 9, admin token | same, via `AdminAPIToken` | same |
| Server < 9 / legacy Cloud webhook | **not implemented** | — |

Delivery requires: recipient's Confluence account has a **reverse KV mapping** (i.e. they ran `/confluence connect`), notifications not toggled off, and the mention isn't the actor's own. Eligible events are page/comment created/updated only. Bot mentions (`userType: "APP"`) are skipped.

Forge caveat: `enqueue` drops the body entirely if the enriched event would exceed ~224KB of Forge storage — logged as `body too large for Forge storage ... mentions will be skipped for this event`. Big pages silently lose mention DMs while still delivering the channel notification.

## Forge Bridge Lifecycle

1. Customer self-hosts the app from `forge/` under their own Atlassian developer account (Mattermost does not currently publish it).
2. `forge deploy` + `forge install`; `onInstalled` logs the drain and register webtrigger URLs.
3. Admin runs `/confluence install cloud`, pastes drain + register URLs into the wizard dialog.
4. Plugin POSTs `{"secret": "<32 chars>"}` to the register webtrigger. The bridge stores it via `kvs.setSecret` and returns all three webtrigger URLs.
5. `ForgePoller` polls the drain URL every 30s, HMAC-SHA256 over the raw body in `X-MM-Signature`, acks by key, deletes acked keys on the bridge side.

**`register` is one-shot per secret.** A second registration with a *different* secret returns **HTTP 409**. Recovery order:
1. `/confluence forge reset` — rotates in place using the current secret (and self-heals a missing `ForgeResetURL` from the register webtrigger).
2. If the two sides have already drifted so the current secret is wrong, a Forge admin must run `forge invoke -f wipeRegistrationFn -e <env>`, then re-run `/confluence install cloud`.

The poller DMs **all** system admins (page 0, 50 per page) on `alertOnce`, de-duplicated via the `forge_alert_key` KV value and cleared on the next successful drain. Alert categories: `secret_empty`, `hmac_mismatch` (401/403), `not_registered` (404/503).

The HTTP client refuses redirects (`http.ErrUseLastResponse`) so the signed body never leaves the validated host.

## Slash Commands

`/confluence` — subcommands resolve by longest-prefix path join (`settings/notifications/on`), so `help` is the fallback when args are empty and `executeConfluenceDefault` handles anything unmatched.

| Command | Handler | Notes |
|---------|---------|-------|
| `connect` / `disconnect` | `executeConnect` / `executeDisconnect` | `connect` refuses if `ConfluenceURL` is empty or OAuth isn't configured |
| `subscribe` / `edit "<name>"` | **client-side only** | Intercepted by `slashCommandWillBePostedHook` in the webapp; never reaches the server as a command. If the webapp bundle fails to load, these appear to do nothing. |
| `list` | `listChannelSubscription` | |
| `unsubscribe "<name>"` | `deleteSubscription` | Alias matching for delete is exact; `GetInsensitiveCase` exists but is used elsewhere |
| `install` / `install cloud` / `install server` | flow wizards | System Admin only |
| `settings notifications [on\|off]` | mention DM toggle | |
| `forge reset` | `executeForgeReset` | System Admin only, Cloud only |
| `help` | version-aware; admins additionally get `sysAdminHelpText` | |

## HTTP API (`/plugins/com.mattermost.confluence/api/v1`)

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| POST | `/server/webhook?secret=` | secret only | Confluence Server/DC webhook. Also answers `{"test": true}` connection probes with 200 OK. |
| POST | `/cloud/{event}?secret=` | secret only | Legacy Atlassian Connect webhook |
| POST | `/{channelID}/subscription/{type}` | user + sysadmin + channel | Create subscription |
| PUT | `/{channelID}/subscription/{type}` | user + sysadmin + channel | Edit subscription |
| GET | `/{channelID}/subscription?alias=` | user + sysadmin + channel | Fetch one, for the edit modal |
| GET | `/autocomplete/GetChannelSubscriptions` | user + sysadmin | Autocomplete for `edit`/`unsubscribe` |
| GET | `/oauth2/connect` | user | Start OAuth |
| GET | `/oauth2/complete.html` | user | OAuth callback (rendered from `assets/templates`) |
| GET | `/user-connection-info` | user | `{can_run_subscribe_command, server_version_greater_than_9}` — drives the webapp hook |
| GET | `/config` | user | Version-appropriate event list for the modal |
| GET | `/static/*` | none | Bundle assets |

Plus the flow framework's own routes, registered by `flow.InitHTTP(fm.router)` inside `newFlow`.

`{type}` must be `space_subscription` or `page_subscription`; anything else is a 400.

## Common Support Investigation Patterns

### "Notifications aren't arriving in the channel"
1. **Establish the path first** (see the table above). Cloud on the Forge bridge means the plugin polls *outbound* — there's nothing to look for in an inbound-request log.
2. Check for `This plugin is not configured.` — a 501 wall kills the webhook route entirely (`Secret`/`EncryptionKey` must be exactly 32 chars).
3. Secret mismatch: `request URL: secret did not match` from `verifyHTTPSecret`. The Confluence-side webhook URL embeds the secret; regenerating `Secret` in the System Console silently breaks it.
4. `BaseURL` drift: the subscription stores the URL it was created with. If `ConfluenceURL` was later changed (added/removed trailing slash is handled by `Sanitize`, but scheme or host changes are not), lookups miss. Compare the stored `BaseURL` against current config.
5. Event not in the subscription's `Events` list, or not supported for the configured version — `getNotificationChannelIDs` filters by exact event string.
6. Space-level event? See the `space_updated` note — it never fires.
7. v9+ specifically: the *event-triggering* Confluence user must be connected, or `AdminAPIToken` must be set. Otherwise you get the degraded `SendGenericWHNotification` — and that only handles **page** events (`eventActions` has no comment entries), logging `Unsupported Confluence action. Generic notification will not be sent` for comments.

### "Notifications say 'Someone published a page on Confluence with the id 12345'"
That's `SendGenericWHNotification` — the v9+ fallback when the triggering user isn't connected and no `AdminAPIToken` is set. Log line: `Error getting client for the user who triggered webhook event. Sending generic notification`. Fix: set an admin API token, or have users connect.

### "`/confluence subscribe` does nothing"
It's a client-side interception (`webapp/src/hooks/index.js`), not a server command. Check: the webapp bundle loaded, and `/user-connection-info` returned `can_run_subscribe_command: true`. On v9+ a false value means *not connected*; on < 9 it means *not a system admin*. The corresponding ephemeral messages are `User not connected. Please use /confluence connect.` vs the admin-only message.

### "`/confluence connect` fails or the OAuth callback errors"
1. `OAuth config not set for Confluence plugin` → `ConfluenceURL` empty or `IsOAuthConfigured()` false.
2. Redirect URI must match exactly: `<SiteURL>/plugins/com.mattermost.confluence/api/v1/oauth2/complete.html` (`util.GetPluginURL()` + `routeUserComplete`). A wrong or trailing-slash `ServiceSettings.SiteURL` breaks this.
3. `invalid oauth state, please try again` → the `ots_` KV entry expired (15 min) or the browser session changed mid-flow.
4. Cloud: the token must grant access to a site whose URL matches `ConfluenceURL` — `matchCloudResource` compares the configured URL against `accessible-resources`. A user with grants only on a different site fails here.
5. Server vs Cloud scopes differ (`ADMIN` vs `READ`/`WRITE`; Cloud adds `write:confluence-content` for admins). `IsCloud` decides which endpoints are used, and it's set by the *cloud* wizard only — an instance set up with `install server` will never use the 3LO endpoints.

### "Everyone got disconnected after a config change"
`EncryptionKey` changed (or was auto-regenerated because it wasn't exactly 32 chars). Stored tokens are AES-GCM-encrypted with it and cannot be recovered. Look for `Auto-generated missing Encryption Key.` in the logs. Every user must reconnect.

### "Forge events stopped / sysadmins got a bot DM about the bridge"
Match the DM text to the alert category:
- *"shared secret is empty but a drain URL is configured"* (`secret_empty`) — the secret was regenerated while the bridge kept the old one. Re-run `/confluence install cloud`.
- *"rejected drain request (HTTP 401/403)"* (`hmac_mismatch`) — secrets drifted. Try `/confluence forge reset`; if that fails, `forge invoke -f wipeRegistrationFn -e <env>` then re-install.
- *"reports it is not registered (HTTP 404/503)"* (`not_registered`) — bridge was reinstalled/wiped, or the webtrigger URL rotated (redeploying Forge can change webtrigger URLs).
Also check egress to `*.webtrigger.atlassian.app` — the plugin is the client here. Poller logs are at Debug level (`forge drain: skipped, missing config`, `drain: invoked`), so raise the log level before concluding it isn't running.

### "Only one node polls / cluster behavior"
`ForgePoller` uses `cluster.Schedule(..., forge_drain_poller, MakeWaitForInterval(30s))`, so exactly one node polls at a time. There is no per-instance state; a failover just resumes on the next tick. Nothing else in this plugin is cluster-coordinated.

### "Duplicate notifications in a channel"
A channel can hold both a space subscription and a page subscription that match the same event. `getNotificationChannelIDs` concatenates both result sets and then `util.Deduplicate`s **channel IDs** — so a channel is only posted to once per event. Duplicates across *different* channels are expected. True duplicates in one channel usually mean two delivery paths are live at once (e.g. the legacy Cloud Connect webhook still installed *and* the Forge bridge registered) — check whether both `Secret`-based cloud webhooks and `ForgeDrainURL` are configured.

### "Can't create a subscription — 'a subscription with the same name already exists'"
`ValidateSubscription` rejects on two grounds per channel: duplicate alias, and duplicate URL+spaceKey (or URL+pageID). The second one is the surprising one — a channel can't have two subscriptions to the same space even under different names.

### "Subscription save intermittently fails"
`reached write attempt limit` from `AtomicModify` — all subscriptions server-wide are one KV key with a 5-attempt/30ms CAS loop. Retry; if chronic, the customer has heavy concurrent subscription editing.

### "Support packet doesn't show the Confluence URL / credentials"
Intentional. MM-69141 declared `ConfluenceURL`, the OAuth credentials, and all Forge URLs in `plugin.json`'s `settings_schema` (several with `"secret": true`) specifically so support packets redact them. Ask the customer directly for these values.

## Gotchas

- **`docs/admin-guide.md` is stale.** It still describes the Cloud setup as "upload the Connect descriptor from this URL," which Atlassian disabled on 2026-03-31. Cloud setup is now OAuth 2.0 (3LO) + the self-hosted Forge bridge. It also says "Mattermost server v5.19+" while `plugin.json` requires **10.7.0**, and mislabels the v9+ wizard step (says select **No** for the "≥ 9" question in the ≥ 9 section). Trust the code and `flow.go` wizard text over this file.
- **There is no Confluence page on docs.mattermost.com.** Don't link customers there.
- `ServerVersionGreaterthan9` is admin-declared, not detected. Wrong answer → wrong parsing path, wrong event list, wrong permission checks.
- `config.Mattermost` and `config.BotUserID` are package-level globals set in `OnActivate`. Anything invoked before activation completes will nil-panic; `OnConfigurationChange` guards on `config.Mattermost == nil`.
- `store.AdminMattermostUserID` is the literal string `"admin"`, used as a sentinel connection owner. `StoreConnection`/`DeleteConnectionFromKVStore` deliberately skip the reverse mapping for it so mention DMs don't resolve to `"admin"`.
- `util.Deduplicate` iterates a map, so output order is non-deterministic. Notification channel ordering is not stable.
- `Endpoints` is a map keyed by a hash of path+method, and `InitAPI` iterates it — route registration order is non-deterministic. It works because the paths don't overlap ambiguously, but don't assume mux precedence.
- `verifyHTTPSecret` loops URL-unescaping the provided secret until it stops changing, to tolerate double-encoded query strings from Confluence-side configs.
- The Cloud client (`client_cloud.go`) implements only `GetSelf` — `GetSpaceData`, `GetPageData`, `GetSpaceKeyFromSpaceID`, and the mention lookups are stubs. Anything that needs a real Cloud API call goes through the Forge app instead.
- Watch for on-the-wire typos when grepping: the flow step constant is `stepServerVersionQuestion flow.Name = "server-verstion-question"` ("verstion" is the actual persisted step name).
- Go 1.25, Node 24.13.1 (`.nvmrc`). The `forge/` app has its own `package.json` and CI workflow (`.github/workflows/forge-ci.yml`) separate from the plugin build.

## Where things live (cheat sheet)

| You're asking about... | Look at... |
|------------------------|------------|
| Which delivery path is active | `config.Configuration` (`IsCloud`, `ServerVersionGreaterthan9`, `ForgeDrainURL`) |
| Webhook secret rejection | `server/controller.go` `verifyHTTPSecret` |
| "This plugin is not configured" 501 | `server/plugin.go` `ServeHTTP` + `server/config/main.go` `IsValid` |
| Auto-regenerated secrets | `server/plugin.go` `OnConfigurationChange` |
| Which events a version supports | `server/serializer/channel_subscription.go` `SupportedEventsV*`, `ValidateEventsForServerVersion` |
| Subscription → channel resolution | `server/service/notification.go` + `server/notification.go` `getNotificationChannelIDs` |
| Subscription storage / KV keys | `server/store/store.go`, `server/serializer/{space,page}_subscription.go` |
| Duplicate-subscription rejection | `ValidateSubscription` in the two subscription serializers |
| Subscription write contention | `server/store/store.go` `AtomicModify` |
| Who may subscribe | `server/save_subscription.go`, `server/user.go` `validateUserConfluenceAccess` |
| OAuth connect/callback | `server/user.go`, `server/oauth2.go`, `server/instance_{cloud,server}.go` |
| Token encryption | `server/auth_token.go` (AES-GCM, `EncryptionKey`) |
| Setup wizard text and steps | `server/flow.go` |
| Forge polling / alerts | `server/forge_poller.go` |
| Forge secret rotation | `server/forge_reset.go`, `forge/src/index.ts` `register`/`reset`/`wipeRegistration` |
| Forge event type mapping | `server/forge_event_mapping.go` |
| Forge body enrichment / size cap | `forge/src/index.ts` `enrichWithBody`, `enforceStorageLimit` |
| @-mention DM dispatch | `server/service/mentions.go`, `mention_settings.go`, `adf.go`, `storage_xhtml.go` |
| Notification message wording | `server/confluence_server_v2.go`, `server/serializer/confluence_{cloud,forge}.go` |
| Generic/degraded notifications | `server/notification.go` `SendGenericWHNotification` |
| Admin API token usage | `server/confluence_server.go` `*WithAPIToken` functions |
| Slash command routing | `server/command.go` `ConfluenceCommandHandler` |
| Why `/confluence subscribe` is client-side | `webapp/src/hooks/index.js` |
| What the webapp registers | `webapp/src/index.js` (reducer + root modal + command hook only) |
| Support-packet redaction | `plugin.json` `settings_schema` (`"secret": true` fields) |
