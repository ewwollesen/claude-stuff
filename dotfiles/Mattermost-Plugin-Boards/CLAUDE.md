# Mattermost Boards Plugin Codebase Guide (Support Focus)

This guide helps navigate the Mattermost Boards (Focalboard) plugin codebase to answer support questions: finding where board permissions are checked, how sharing works, where block/card operations happen, how templates are managed, etc.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/dotfiles/Mattermost-Plugin-Boards/CLAUDE.md`

## Related Repositories

- **Mattermost Server**: `../Mattermost/` — plugin API, config model, license checks
- **Enterprise**: `../Enterprise/` — enterprise license features

## Repository Structure

```
server/                         # Go backend
├── main.go                     # Plugin entry point
├── plugin.go                   # Plugin lifecycle hooks (OnActivate, ServeHTTP)
├── api_adapter.go              # Mattermost plugin API adapter
├── api/                        # REST API handlers
│   ├── api.go                  # Router setup (gorilla/mux)
│   ├── boards.go               # Board CRUD endpoints
│   ├── blocks.go               # Block CRUD endpoints
│   ├── cards.go                # Card endpoints
│   ├── categories.go           # Category management
│   ├── teams.go                # Team endpoints
│   ├── users.go                # User endpoints
│   ├── members.go              # Board member management
│   ├── sharing.go              # Public board sharing
│   ├── search.go               # Search endpoints
│   ├── templates.go            # Template endpoints
│   ├── files.go                # File upload/download
│   ├── auth.go                 # Authentication
│   └── compliance.go           # Compliance export
├── boards/                     # Core boards app
│   └── boardsapp.go            # BoardsApp — central orchestrator
├── app/                        # Business logic (~45 files)
│   ├── boards.go               # Board operations
│   ├── blocks.go               # Block operations
│   ├── cards.go                # Card operations
│   ├── categories.go           # Category operations
│   ├── templates.go            # Template management
│   ├── sharing.go              # Public sharing logic
│   ├── permissions.go          # Permission checking
│   ├── import.go               # Board import
│   ├── export.go               # Board export
│   ├── files.go                # File handling
│   ├── onboarding.go           # New user onboarding
│   └── teams.go                # Team operations
├── model/                      # Data models
│   ├── board.go                # Board struct
│   ├── block.go                # Block struct and block types
│   ├── card.go                 # Card struct
│   ├── category.go             # Category struct
│   ├── properties.go           # Property type definitions
│   ├── sharing.go              # Sharing model
│   └── blocktype.go            # Block type constants
├── services/                   # Service layer
│   ├── store/                  # Database abstraction
│   │   ├── store.go            # Store interface
│   │   └── sqlstore/           # SQL implementation (PostgreSQL + SQLite)
│   ├── auth/                   # Authentication service
│   ├── permissions/            # Permissions engine
│   ├── notify/                 # Notification and mention handling
│   ├── audit/                  # Audit logging
│   ├── webhook/                # Outbound webhooks
│   ├── metrics/                # Prometheus metrics
│   ├── config/                 # Config service
│   ├── scheduler/              # Background task scheduling
│   └── telemetry/              # Usage telemetry
├── ws/                         # WebSocket management (real-time board updates)
├── web/                        # Static file serving
└── integrationtests/           # Integration tests
webapp/                         # React/TypeScript frontend
├── src/
│   ├── index.tsx               # Plugin entry, Redux provider
│   ├── app.tsx                 # Main app component
│   ├── components/             # 113+ React components
│   │   ├── boardSelector/      # Board selection UI
│   │   ├── cardDetail/         # Card editor (24 sub-components)
│   │   ├── cardDialog/         # Modal card view
│   │   ├── blocksEditor/       # Block content editor
│   │   ├── calendar/           # Calendar view (FullCalendar)
│   │   ├── kanban/             # Kanban board view
│   │   ├── table/              # Table view
│   │   └── gallery/            # Gallery view
│   ├── blocks/                 # Block type definitions
│   ├── store/                  # Redux store and slices
│   ├── hooks/                  # Custom React hooks
│   ├── properties/             # Property type renderers
│   ├── widgets/                # Reusable UI widgets
│   ├── i18n/                   # 40+ language translations
│   └── styles/                 # SCSS stylesheets
├── tests/                      # Jest test files
└── webpack.*.js                # Build configs
plugin.json                     # Plugin manifest (ID: focalboard, min server: 10.7.0)
Makefile                        # Build automation
```

## Data Model

### Board / Block / Card hierarchy

- **Board**: Top-level container. Has properties schema, views, members, and permissions.
- **Block**: Generic content unit within a board. Typed (see below). Contains fields and content.
- **Card**: A special type of block representing a work item. Has property values matching the board's property schema.

### Block types

Defined in `server/model/blocktype.go` and `server/model/block.go`:
- `card` — work item with properties
- `view` — board view configuration (kanban, table, calendar, gallery)
- `text` — rich text content
- `image` — image attachment
- `divider` — visual separator
- `comment` — comment on a card
- `checkbox` — checklist item

### Property types

Defined in `server/model/properties.go` — properties are custom fields on cards:
- Text, Number, Email, URL, Phone
- Date, DateTime
- Select, MultiSelect (with color-coded options)
- Person, MultiPerson
- Checkbox
- CreatedBy, CreatedTime, UpdatedBy, UpdatedTime (computed)

### Board views

Each board can have multiple views showing the same cards differently:
- **Kanban**: Drag-and-drop columns grouped by a select property
- **Table**: Spreadsheet-like rows and columns
- **Calendar**: Cards placed on a calendar by date property
- **Gallery**: Card previews in a grid

## Plugin Architecture

### Lifecycle

1. `OnActivate()` in `server/plugin.go` → initializes BoardsApp with API adapter, database, logging
2. `ServeHTTP()` routes all HTTP requests through BoardsApp's API router
3. WebSocket handlers in `server/ws/` push real-time updates (block changes, card limit updates)
4. `OnDeactivate()` — cleanup

### BoardsApp

`server/boards/boardsapp.go` is the central orchestrator:
- Initializes all services (store, auth, permissions, notifications, webhooks, metrics, audit)
- Provides the API router
- Manages the app lifecycle

## Database Layer

### Store interface

- **Interface**: `server/services/store/store.go` — defines all data access methods
- **SQL implementation**: `server/services/store/sqlstore/` — supports PostgreSQL and SQLite
- **Migrations**: Morph-based migration system in the sqlstore

### Key tables

- Boards, Blocks (with block type), Board members
- Categories (user-organized board groups)
- Sharing (public link configuration)
- Sessions, Users (Boards-specific user preferences)

## Permissions

- **Permission checks**: `server/app/permissions.go` — checks board-level and team-level permissions
- **Permissions engine**: `server/services/permissions/` — role-based access control
- **Board roles**: Admin, Editor, Commenter, Viewer
- **Team-level access**: Board visibility within teams

## Sharing

- **Public board sharing**: `server/app/sharing.go` — generates public access links
- **Config gate**: `EnablePublicSharedBoards` setting in plugin configuration
- **Sharing model**: `server/model/sharing.go` — sharing configuration per board
- **API**: `server/api/sharing.go` — endpoints for managing shared boards

## Import / Export

- **Import**: `server/app/import.go` — imports boards from archive files
- **Export**: `server/app/export.go` — exports boards to archive format
- **Archive format**: `.boardarchive` files (used for templates and backup)
- **Templates**: `server/app/templates.go` — built-in board templates stored as boardarchive

## Notifications and Webhooks

- **Mentions**: `server/services/notify/` — @mention handling and notification delivery
- **Webhooks**: `server/services/webhook/` — outbound webhooks triggered by board events
- **Subscriptions**: Users subscribe to boards/cards for update notifications

## Common Support Investigation Patterns

### "Where is a boards API endpoint handled?"
1. Find the route in `server/api/api.go` (gorilla/mux router setup)
2. Follow to the handler in the appropriate `server/api/*.go` file
3. Handler calls business logic in `server/app/*.go`
4. App calls store in `server/services/store/`

### "What block types exist?"
1. Check `server/model/blocktype.go` for type constants
2. Check `server/model/block.go` for the Block struct
3. Frontend renderers: `webapp/src/blocks/`

### "How does board sharing work?"
1. Config: `EnablePublicSharedBoards` must be enabled
2. Logic: `server/app/sharing.go`
3. Model: `server/model/sharing.go`
4. API: `server/api/sharing.go`

### "Board permission denied — why?"
1. Check `server/app/permissions.go` for the specific permission check
2. Check `server/services/permissions/` for the role evaluation
3. Board roles: Admin > Editor > Commenter > Viewer
4. Team-level access may also apply

### "How do board templates work?"
1. Built-in templates: `server/app/templates.go`
2. Template storage: `.boardarchive` format
3. Template API: `server/api/templates.go`
4. Users can create custom templates from existing boards

### "Import/export not working?"
1. Import: `server/app/import.go` — check archive parsing
2. Export: `server/app/export.go` — check archive generation
3. File format: `.boardarchive` (JSON-based archive)

### "Where are board categories managed?"
1. Model: `server/model/category.go`
2. Business logic: `server/app/categories.go`
3. API: `server/api/categories.go`
4. Categories are user-specific — each user organizes their sidebar independently

### "Real-time updates not working?"
1. WebSocket: `server/ws/` — manages connections and broadcasts
2. Events: Block changes, card limit updates
3. Check if WebSocket connection is established (browser DevTools → Network → WS)

### "Where is the store interface?"
1. `server/services/store/store.go` — defines all data access methods
2. Implementation: `server/services/store/sqlstore/`
3. Supports PostgreSQL and SQLite
