# Mattermost Playbooks Plugin Codebase Guide (Support Focus)

This guide helps navigate the Mattermost Playbooks plugin codebase to answer support questions: finding where playbook runs are created, how automation triggers work, where permissions are checked, how the GraphQL API works, etc.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/ClaudeFiles/Mattermost-Plugin-Playbooks/CLAUDE.md`

## Related Repositories

- **Mattermost Server**: `../Mattermost/` — plugin API, WebSocket events, config model, license checks
- **Enterprise**: `../Enterprise/` — enterprise license features
- **Mobile**: `../Mattermost-Mobile/` — mobile Playbooks product module (`app/products/playbooks/`)

## Repository Structure

```
server/                         # Go backend
├── plugin.go                   # Plugin lifecycle (OnActivate, ServeHTTP)
├── manifest.go                 # Generated manifest
├── api/                        # REST API handlers
│   ├── api.go                  # Router setup (gorilla/mux), handler registration
│   ├── playbooks.go            # Playbook CRUD endpoints
│   ├── playbook_runs.go        # Run CRUD and lifecycle endpoints
│   ├── actions.go              # Automation action endpoints
│   ├── signal.go               # Signal/keyword trigger endpoints
│   └── telemetry.go            # Telemetry endpoints
├── app/                        # Business logic services
│   ├── playbook_service.go     # PlaybookService — playbook CRUD
│   ├── playbook_run_service.go # PlaybookRunService — run lifecycle, status updates
│   ├── permissions_service.go  # PermissionsService — RBAC checks
│   ├── action.go               # Action definitions
│   ├── actions_service.go      # ActionsService — automation execution
│   ├── task_actions.go         # Task-level automation
│   ├── condition.go            # Condition definitions
│   ├── condition_service.go    # ConditionService — event triggers
│   ├── category_service.go     # CategoryService — run categorization
│   ├── property_service.go     # PropertyService — custom fields
│   ├── reminder.go             # Status update reminders
│   ├── regular_digest_service.go # Periodic digest notifications
│   └── export.go               # Run export
├── graphql/                    # GraphQL API
│   ├── models.go               # GraphQL type definitions
│   ├── resolvers.go            # GraphQL resolvers
│   └── schema.graphqls         # GraphQL schema (407 lines)
├── sqlstore/                   # PostgreSQL data layer
│   ├── sqlstore.go             # SQLStore interface and implementation
│   ├── migrations.go           # Database migrations
│   ├── playbook.go             # Playbook queries
│   ├── playbook_run.go         # Run queries
│   └── stats.go                # Statistics queries
├── command/                    # Slash command handlers
├── bot/                        # Bot implementation (status posts, reminders)
├── scheduler/                  # Job scheduling
├── config/                     # Configuration management
├── enterprise/                 # Enterprise-only features
├── metrics/                    # Prometheus metrics (port :9093)
└── telemetry/                  # Usage telemetry
webapp/                         # React/TypeScript frontend
├── src/
│   ├── components/
│   │   ├── backstage/          # Playbook/run management (main list views)
│   │   ├── rhs/                # Right-hand sidebar (run details panel)
│   │   ├── checklist/          # Checklist UI components
│   │   └── modals/             # Dialogs and forms
│   ├── graphql/                # GraphQL queries and mutations (Apollo Client)
│   ├── hooks/                  # Custom React hooks
│   ├── types/                  # TypeScript type definitions
│   ├── actions.ts              # Redux actions
│   ├── reducer.ts              # Redux state management
│   ├── client.ts               # HTTP client wrapper
│   └── websocket_events.ts     # WebSocket event handlers
├── i18n/                       # Internationalization files
client/                         # Go SDK for Playbooks API
e2e-tests/                      # Cypress E2E tests
loadtest/                       # Load testing (browser + backend)
plugin.json                     # Plugin manifest
Makefile                        # Build automation
```

## Data Model

### Core entities

- **Playbook**: A template defining the process — contains stages, checklists, actions, metrics definitions, default permissions, and automation configuration. Think of it as the "recipe."
- **Playbook Run**: An active instance of a playbook — has status (InProgress/Finished), participants, current checklist state, metrics values, timeline events. Think of it as the "execution."
- **Checklist**: An ordered list of tasks within a playbook/run stage. Each item has a title, assignee, due date, and completion state.
- **Action**: An automated trigger — fires on specific events (status update, member join, task completion) to execute webhooks, post messages, or run integrations.
- **Condition**: Event triggers for automation — defines when actions should fire.
- **Property**: Custom fields on runs (text, number, etc.) — defined in the playbook, filled in the run.
- **Category**: User-organized grouping of runs.

### Run lifecycle

1. Run is created from a playbook (manually or via keyword trigger)
2. A dedicated channel is created for the run
3. Participants are added; checklists are populated from the playbook
4. Status updates are posted periodically (with optional reminders)
5. Tasks are checked off as work progresses
6. Run is finished → retrospective can be filed
7. Run can be exported for audit

## API Layer

### Dual API

The plugin exposes both REST and GraphQL APIs:

- **REST API**: `server/api/` — gorilla/mux router, prefix `/api/v0`. Handles CRUD operations, run lifecycle, actions, telemetry.
- **GraphQL API**: `server/graphql/` — endpoint at `/query`. Schema in `schema.graphqls` (407 lines). Used primarily by the webapp for complex queries.
- **Middleware**: `MattermostAuthorizationRequired` — all endpoints require Mattermost session auth

### Slash commands

`server/command/` — Playbooks registers slash commands:
- `/playbook run` — start a new run
- `/playbook update` — post a status update
- `/playbook finish` — finish a run
- And more (list, info, checklist management)

## Automation and Actions

### Actions

- **Definitions**: `server/app/action.go` — action types and configuration
- **Execution**: `server/app/actions_service.go` — runs actions when triggers fire
- **Task actions**: `server/app/task_actions.go` — automation tied to specific checklist items

### Conditions and triggers

- **Conditions**: `server/app/condition.go`, `server/app/condition_service.go` — event-based trigger definitions
- **Signal keywords**: `server/api/signal.go` — keyword triggers that auto-create runs when specific words appear in channels
- **Broadcast channels**: Runs can broadcast status updates to configured channels

### Reminders

- **Status update reminders**: `server/app/reminder.go` — reminds the run owner to post status updates
- **Digest notifications**: `server/app/regular_digest_service.go` — periodic digest of run activity

## Permissions

- **Permission service**: `server/app/permissions_service.go` — RBAC checks for all operations
- **Roles**: Playbook member, Playbook admin, Run member, Run admin
- **Enterprise**: `server/enterprise/` — additional permission features gated by license
- **Public/Private**: Playbooks and runs can be public (team-visible) or private (members-only)

## Database Layer

- **PostgreSQL only**: The plugin only supports PostgreSQL (errors if MySQL/SQLite)
- **SQLStore**: `server/sqlstore/sqlstore.go` — store interface and implementation
- **Migrations**: `server/sqlstore/migrations.go` — numbered migrations
- **Query builder**: Masterminds Squirrel for SQL construction
- **Max JSON field size**: 256KB

## Bot

- **Bot implementation**: `server/bot/` — posts status updates, reminders, and notifications into run channels
- **Bot account**: Created on plugin activation with the Playbooks icon

## Metrics

- **Prometheus**: `server/metrics/` — exposes metrics on port `:9093`
- **Run metrics**: Custom metrics defined in playbooks (duration, currency, integer types)
- **Plugin metrics**: Request latency, active runs, error rates

## Common Support Investigation Patterns

### "How are playbook runs created?"
1. Manual: `server/api/playbook_runs.go` — create run endpoint
2. Keyword trigger: `server/api/signal.go` — signal keywords in channel messages
3. Slash command: `server/command/` — `/playbook run`
4. Business logic: `server/app/playbook_run_service.go` — run creation and channel setup

### "Automation trigger not firing"
1. Check action definitions: `server/app/action.go`
2. Check condition evaluation: `server/app/condition.go`, `server/app/condition_service.go`
3. Check action execution: `server/app/actions_service.go`
4. Check task-level actions: `server/app/task_actions.go`
5. Verify the trigger event type matches the condition configuration

### "Status update reminders not working"
1. Check reminder logic: `server/app/reminder.go`
2. Check digest service: `server/app/regular_digest_service.go`
3. Check scheduler: `server/scheduler/` — job scheduling
4. Check bot: `server/bot/` — is the bot posting?

### "Permission denied on playbook/run operation"
1. Check `server/app/permissions_service.go` for the specific permission check
2. Roles: member vs admin at both playbook and run level
3. Public vs private visibility
4. Enterprise features: `server/enterprise/` — additional permission gating

### "Where is the GraphQL schema?"
1. Schema definition: `server/graphql/schema.graphqls` (407 lines)
2. Type definitions: `server/graphql/models.go`
3. Resolvers: `server/graphql/resolvers.go`
4. Frontend queries: `webapp/src/graphql/`

### "Where is the SQL schema / migrations?"
1. Migrations: `server/sqlstore/migrations.go`
2. Store interface: `server/sqlstore/sqlstore.go`
3. Query implementations: `server/sqlstore/playbook.go`, `server/sqlstore/playbook_run.go`
4. PostgreSQL only — no SQLite/MySQL support

### "Slash command not working"
1. Command handlers: `server/command/`
2. Check if the command is registered in `plugin.json`
3. Verify the user has permission to use the command in that channel

### "Run export not working"
1. Export logic: `server/app/export.go`
2. Check permissions: export may require run member/admin role
3. Check if the run has the data expected (status updates, checklists)

### "Where does the right-hand sidebar panel come from?"
1. Webapp RHS components: `webapp/src/components/rhs/`
2. WebSocket events: `webapp/src/websocket_events.ts` — real-time updates
3. Redux state: `webapp/src/reducer.ts`
