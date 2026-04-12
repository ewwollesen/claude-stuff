# Mattermost Mobile Codebase Guide (Support Focus)

This guide helps navigate the Mattermost Mobile codebase to answer support questions: finding where data is stored, how the app syncs with the server, where push notifications are handled, how the dual database system works, etc.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/ClaudeFiles/Mattermost-Mobile/CLAUDE.md`

## Related Repositories

- **Mattermost Server**: `../Mattermost/` — server API, WebSocket events, config model, error translations
- **Enterprise**: `../Enterprise/` — enterprise features (LDAP, SAML, Intune MAM)
- **Plugin-Calls**: `../Mattemrost-Plugin-Calls/` — Calls plugin (mobile has product module: `app/products/calls/`)
- **Plugin-Agents**: `../Mattermost-Plugin-Agents/` — AI plugin (mobile has product module: `app/products/agents/`)
- **Plugin-Playbooks**: `../Mattermost-Plugin-Playbooks/` — Playbooks plugin (mobile has product module: `app/products/playbooks/`)

## Repository Structure

```
app/                              # Main application source
├── actions/                      # Data operations
│   ├── local/                    # Database-only operations
│   └── remote/                   # API fetch → persist to DB
├── client/                       # Server communication
│   ├── rest/                     # REST API client mixins (45+ endpoint groups)
│   └── websocket/                # Real-time event handling
├── components/                   # 93+ reusable UI components
├── database/                     # WatermelonDB management (CRITICAL — see below)
│   ├── manager/                  # DatabaseManager singleton
│   ├── models/app/               # App database models (server list, app state)
│   ├── models/server/            # Per-server database models (channels, users, posts)
│   ├── operator/                 # Data sync operators (AppDataOperator, ServerDataOperator)
│   └── schema/                   # Database schemas and migrations
├── hooks/                        # 80+ custom React hooks
├── managers/                     # System managers
│   ├── network_manager.ts        # One REST Client per server
│   ├── websocket_manager.ts      # Real-time connection management
│   ├── session_manager.ts        # Session/auth lifecycle
│   └── security_manager.ts       # Security features
├── products/                     # Modular feature products
│   ├── agents/                   # AI agents (own models, actions, components, screens)
│   ├── calls/                    # Voice/video calls
│   └── playbooks/                # Playbooks with dedicated DB models
├── queries/                      # Database query layer (query/observe/get/prepare patterns)
├── screens/                      # 68 screen components
├── store/                        # Ephemeral in-memory UI state (NOT persisted)
├── context/                      # React contexts (device, theme, locale)
└── utils/                        # 100+ helper functions
android/                          # Android native (Gradle, SDK 35, Kotlin 2.0)
ios/                              # iOS native (Xcode, CocoaPods, iOS 16.0+)
detox/                            # E2E test framework (separate package)
libraries/@mattermost/            # Custom native modules
├── rnutils/                      # Native utilities (orientation, notifications)
├── rnshare/                      # Share extension integration
├── hardware-keyboard/            # External keyboard detection
└── secure-pdf-viewer/            # Secure PDF viewing
share_extension/                  # Android share extension (separate bundle)
```

## Dual Database System (WatermelonDB)

This is the most important architectural concept for mobile support investigations.

### Two separate SQLite databases

1. **App Database**: One global database — stores server list, global app state. Models in `app/database/models/app/`. Never deleted.
2. **Server Databases**: One database per connected Mattermost server — stores channels, users, posts, threads, teams, etc. Models in `app/database/models/server/`. Created on login, deleted on logout.

### Key components

- **DatabaseManager** (`app/database/manager/index.ts`): Singleton managing all database instances
- **Operators** (`app/database/operator/`): AppDataOperator and ServerDataOperator handle syncing data from API responses into the database (create/update/delete)
- **Schema** (`app/database/schema/`): Database schemas with migration system (version-based)

### Database locations

- **iOS**: App Group directory (shared with share extension and notification service extension)
- **Android**: `${documentDirectory}/databases/`

### Query patterns

`app/queries/` provides four patterns:
- `query*()` — returns WatermelonDB Query object
- `observe*()` — returns RxJS Observable (reactive, auto-updates UI)
- `get*()` — returns Promise (one-shot async fetch)
- `prepare*()` — returns prepared records for batch operations

## Network Layer

- **NetworkManager** (`app/managers/network_manager.ts`): Creates one REST Client per server
- Uses `@mattermost/react-native-network-client` (custom native HTTP library)
- Exponential retry policy (3 attempts)
- Session-based authentication (Bearer token)
- Certificate pinning and SSL validation support

### REST API client

The REST client uses a **ClientMix** pattern — endpoint groups are organized as mixins in `app/client/rest/`:
- Each mixin covers a domain (users, channels, posts, teams, etc.)
- Product-specific API calls live in `app/products/{product}/client/rest.ts`

## Actions Pattern

- **Local Actions** (`app/actions/local/`): Database-only operations (no API calls)
- **Remote Actions** (`app/actions/remote/`): Fetch from server API → use operators to persist to local DB → return `{error}` on failure
- Data flow: Remote Action → REST Client → API response → ServerDataOperator → WatermelonDB

## State Management

Hybrid approach:
- **WatermelonDB**: Primary data store (persisted to SQLite). All server data, user data, channel data.
- **Ephemeral Stores** (`app/store/`): In-memory transient UI state (typing indicators, draft state, navigation state). NOT persisted — lost on app restart.

## Navigation

- Uses **react-native-navigation** v7 (NOT react-navigation)
- Each screen is a separate registered component
- Bottom tab navigation (channels, threads, search, etc.) with nested stack navigators
- Every independently shown/hidden component must be registered as a screen

## Products Architecture

Modular features in `app/products/` — each has its own:
- Database models and schema
- Actions (local/remote)
- Components and screens
- Client API mixins
- Type definitions

### Agents product (`app/products/agents/`)

- AI agents integration
- Streaming via `custom_mattermost-ai_postupdate` WebSocket events
- Event types: start, message updates, reasoning, tool calls, annotations, end

### Calls product (`app/products/calls/`)

- Voice/video calling
- Integrates with the Calls plugin's WebRTC stack

### Playbooks product (`app/products/playbooks/`)

- Playbook runs with dedicated DB models
- Status updates and checklists

## WebSocket Events

- **WebsocketManager** (`app/managers/websocket_manager.ts`): Manages real-time connections
- One WebSocket per connected server
- Events trigger database updates via operators
- Key events: `posted`, `post_edited`, `post_deleted`, `channel_updated`, `user_updated`, `typing`, etc.

## Push Notifications

- Native push notification handling in both iOS and Android native layers
- iOS: Notification Service Extension (in App Group for database access)
- Android: Firebase Cloud Messaging
- Self-compiled apps require their own Mattermost Push Notification Service (MPNS)

## TypeScript Path Aliases

Important for searching the codebase — imports use aliases:
```
@actions/*     → app/actions/*
@agents/*      → app/products/agents/*
@calls/*       → app/products/calls/*
@database/*    → app/database/*
@queries/*     → app/queries/*
@store         → app/store/index
@share/*       → share_extension/*
@typings/*     → types/*
```
Full list in `tsconfig.json`.

## Common Support Investigation Patterns

### "Where is mobile data for X stored?"
1. Determine if it's app-level or server-level data
2. App-level: `app/database/models/app/` (server list, global preferences)
3. Server-level: `app/database/models/server/` (channels, users, posts, threads)
4. Check the WatermelonDB schema in `app/database/schema/`

### "How does mobile sync data from the server?"
1. WebSocket events trigger database updates: `app/managers/websocket_manager.ts`
2. Remote actions fetch data: `app/actions/remote/`
3. Operators persist to DB: `app/database/operator/`
4. Queries/Observables update the UI reactively: `app/queries/`

### "Where does the mobile app connect to the server?"
1. NetworkManager creates clients: `app/managers/network_manager.ts`
2. REST API mixins: `app/client/rest/`
3. WebSocket: `app/client/websocket/`
4. Session management: `app/managers/session_manager.ts`

### "Push notifications not working on mobile?"
1. Check if self-compiled (requires own MPNS)
2. iOS: Notification Service Extension in the native layer
3. Android: Firebase Cloud Messaging setup
4. Server-side: check push notification settings in Mattermost config

### "Where is the Calls/Agents/Playbooks UI on mobile?"
1. Each lives in `app/products/{product}/`
2. Has its own screens, components, actions, and DB models
3. Agents streaming: `custom_mattermost-ai_postupdate` WebSocket events

### "Mobile app losing data or showing stale data?"
1. Check the database lifecycle: `app/database/manager/`
2. Server databases are per-server — created on login, destroyed on logout
3. Check sync handlers in `app/database/operator/` — they must handle create/update/delete
4. Check if the observable in `app/queries/` is properly subscribed

### "SSL/certificate issues on mobile?"
1. Self-signed certificates are NOT supported
2. Certificate pinning via `@mattermost/react-native-network-client`
3. Security manager: `app/managers/security_manager.ts`

### "Mobile app crash or performance issue?"
1. Check if it's a database issue (WatermelonDB memory usage)
2. Ephemeral store state: `app/store/` — look for memory leaks in subscribers
3. Large channels/threads: query performance in `app/queries/`
4. Native module issues: `libraries/@mattermost/`
