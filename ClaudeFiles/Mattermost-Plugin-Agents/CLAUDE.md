# Mattermost Agents Plugin Codebase Guide (Support Focus)

This guide helps navigate the Mattermost Agents (AI) plugin codebase to answer support questions: finding where LLM providers are configured, how conversations flow, where MCP tools are managed, how embeddings and search work, etc.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/ClaudeFiles/Mattermost-Plugin-Agents/CLAUDE.md`
> - **Verified as of:** upstream `78248541` (2026-08-26). Paths drift fast — confirm a path
>   still exists before quoting it to a customer.
> - Upstream tracks its own `CLAUDE.md` (it imports `AGENTS.md`), so this guide lives at
>   `CLAUDE.local.md` and is git-ignored via `.git/info/exclude`. Never restore it over
>   `CLAUDE.md` with `git update-index --skip-worktree` — that silently blocks `git pull`.

## Related Repositories

- **Mattermost Server**: `../Mattermost/` — plugin API, WebSocket events, bot management, config model
- **Enterprise**: `../Enterprise/` — enterprise license features
- **Mobile**: `../Mattermost-Mobile/` — mobile Agents product module (`app/products/agents/`)

## Repository Structure

```
server/                     # Go plugin entry point
├── main.go                 # Plugin struct, Mattermost lifecycle hooks
api/                        # HTTP API handlers (Gin framework)
├── api.go                  # API struct, route registration
llm/                        # LLM abstraction layer
├── service_types.go        # Provider type constants (openai, anthropic, azure, etc.)
├── providers.go            # Provider registry and factory
├── configuration.go        # ServiceConfig, BotConfig, model settings
├── language_model.go       # Core LanguageModel interface
├── stream.go               # LLM response streaming
├── stream_generator.go     # Stream generation utilities
├── tools.go                # Tool/function calling definitions
conversations/              # Conversation management
├── conversations.go        # Conversation lifecycle, auto-execute policy
├── handle_messages.go      # Message handling and routing
├── tool_approval.go        # Tool approval prompts and follow-up completions
toolrunner/                 # Tool-calling loop and completion orchestration
├── toolrunner.go           # Run loop, tool execution, max-round limits
mcp/                        # Model Context Protocol client
├── client.go               # MCP client implementation
├── client_manager.go       # MCP session and OAuth management
mcpserver/                  # Embedded MCP server
├── plugin_handlers.go      # In-plugin MCP server wiring
├── http_server.go          # Standalone HTTP MCP server + bearer auth
├── stdio_server.go         # Standalone stdio MCP server
├── proxy_tools.go          # Proxying to external MCP servers
├── tools/                  # MCP tool definitions (one file per resource)
bots/                       # Bot management
├── bots.go                 # Bot user creation and permissions
config/                     # Plugin configuration management
store/                      # Database layer
├── store.go                # Database operations
├── migrate.go              # Schema migrations
embeddings/                 # Vector embeddings for RAG
├── embeddings.go           # Embedding provider abstraction
indexer/                    # Document indexing for search
search/                     # Semantic search implementation
postgres/                   # pgvector storage for embeddings
prompts/                    # Prompt templates (*.tmpl files)
streaming/                  # Response streaming utilities
websearch/                  # Web search integration
meetings/                   # Meeting transcription/summary integration
bifrost/                    # LLM gateway client — ALL providers route through here
enterprise/                 # Enterprise license checking
metrics/                    # Prometheus metrics collection
evals/                      # Prompt evaluation framework
webapp/                     # React/TypeScript frontend
├── src/
│   ├── components/         # UI components (chat, settings, admin)
│   ├── actions/            # Redux actions
│   └── websocket.ts        # WebSocket client for streaming events
e2e/                        # Playwright E2E tests
plugin.json                 # Plugin manifest with settings schema
Makefile                    # Build system
```

## LLM Provider System

### Supported providers

| Provider | Service Type | Auth | Notes |
|----------|-------------|------|-------|
| OpenAI | `openai` | API key | Native streaming, Responses API |
| Anthropic | `anthropic` | API key | Thinking/reasoning, structured output |
| Azure OpenAI | `azure` | API URL + API key | Azure-hosted OpenAI models |
| AWS Bedrock | `bedrock` | IAM or credentials | AWS-hosted models |
| OpenAI-compatible | `openaicompatible` | API URL + key | Ollama, vLLM, local models |
| Cohere | `cohere` | API key | Fixed endpoint |
| Mistral | `mistral` | API key | Lite model support |
| Scale | `scale` | Custom headers | Custom auth |

### Key files

- **Provider types**: `llm/service_types.go` — string constants for each provider
- **Provider registry**: `llm/providers.go` — table of OpenAI-compatible quirks (fixed base
  URLs, custom auth transports). Note: `GetOpenAICompatibleProvider` has no non-test callers
  since the Bifrost migration — do not assume this file affects runtime behavior.
- **Configuration**: `llm/configuration.go` — `ServiceConfig` (provider connection) and `BotConfig` (bot behavior)
- **Core interface**: `llm/language_model.go` — `LanguageModel` interface all providers implement
- **Streaming**: `llm/stream.go`, `llm/stream_generator.go` — streaming response handling
- **Gateway client**: `bifrost/` — wraps the `maximhq/bifrost` library and implements
  `llm.LanguageModel` for every provider above. Not enterprise-gated, not agent-specific:
  `bots.getLLM()` (`bots/bots.go`) builds *all* bot clients via `bifrost.NewFromServiceConfig`.
  - `bifrost/config.go` — maps service type → provider constant, applies fixed base URLs
  - `bifrost/bifrost.go` — completions, streaming, tool calls, reasoning/thinking config
  - `bifrost/embeddings.go` — embedding provider used by `search/embeddings.go`
  - `bifrost/models.go` — model list fetching for the admin UI (`api/api.go`)
  - `bifrost/transcription.go` — audio transcription for `meetings/`

**Start here for any provider bug** (auth, base URL, streaming, token limits, tool calls) —
`bifrost/`, not `llm/providers.go`.

> **Gotcha — Scale AI:** the System Console still offers `scale`
> (`webapp/src/components/system_console/service.tsx`) and `llm/configuration.go` accepts it,
> but `bifrost.MapServiceTypeToProvider` has no `scale` case, so bot creation fails with
> `unsupported service type: scale`. Scale support (#517, Mar 2026) added the registry entry
> in `llm/providers.go` but was never wired into `bifrost/config.go` after the Bifrost
> migration (#484, Feb 2026). Re-check `bifrost/config.go` before repeating this to a customer.

## Conversation Flow

### How a message becomes an AI response

1. User posts a message mentioning the bot or in a DM with the bot
2. Plugin hook `MessageHasBeenPosted` fires → `server/main.go`
3. Message routed to `conversations/handle_messages.go`
4. Context assembled (thread history, system prompt, tools)
5. Completion requested: `toolrunner/toolrunner.go` (`Run`) → `llm/` → `bifrost/` → provider API
6. Response streamed back via SSE → posted as bot message
7. If tools are called: tool execution → results fed back → follow-up completion

### Tool calling

- **Tool definitions**: `llm/tools.go` — defines available tools
- **Auto-execution policy**: `conversations/conversations.go` — `shouldAutoExecuteTool()` decides what runs without user approval; `allToolsAutoRunEverywhere()` handles the all-auto case
- **Approval flow**: `conversations/tool_approval.go` — `HandleToolCall()` / `HandleToolResult()` for tools needing a click
- **Execution**: `toolrunner/toolrunner.go` — `executeTools()` runs the approved calls
- **Mattermost tools**: `mmtools/` — Mattermost-specific tools (search, channel info, etc.)
- **MCP tools**: `mcp/` — tools provided via Model Context Protocol

## Bot Management

- **Bot creation/config**: `bots/bots.go` — multiple AI assistants can be configured
- **Per-bot settings**: Each bot can have custom instructions, model overrides, tool access
- **Permissions**: Role-based access control (channel/user allow/block lists)
- **System Console UI**: Plugin settings in `plugin.json` define the admin interface

## MCP (Model Context Protocol)

### Client side

- **MCP client**: `mcp/client.go` — connects to external MCP servers
- **Session management**: `mcp/client_manager.go` — manages MCP sessions and OAuth flows
- **Auto-approval**: Conversations can auto-approve certain MCP tools

### Server side

The plugin also *exposes* MCP tools. `mcpserver/` has its own `AGENTS.md`
(imported by its `CLAUDE.md`) — read it before deep dives.

#### Four server variants

| Variant | File | Access mode | Semantic search |
|---|---|---|---|
| In-memory (embedded) | `inmemory_server.go` | `remote` | `*search.Search` passed in directly |
| Plugin handlers | `plugin_handlers.go` | `remote` | HTTP callback |
| HTTP (standalone) | `http_server.go` | `remote` | HTTP callback |
| Stdio (standalone) | `stdio_server.go` | **`local`** | HTTP callback |

All four funnel into `registerTools(accessMode, searchService, fileContentService)`
in `mcpserver/server.go`.

- **Stdio is the only `local` variant.** File-access tools check
  `accessMode != AccessModeLocal` (`tools/file_utils.go`, `tools/files.go:240`) and
  refuse otherwise, so file tools work over stdio and not over HTTP. Args fields
  tagged `access:"local"` are stripped from the schema in remote mode
  (`NewJSONSchemaForAccessMode`). Modes are defined in `tools/access_mode.go`.
- **Search takes a network round-trip in every variant except in-memory.**
  `NewHTTPSemanticSearchService(pluginURL)` POSTs to
  `<pluginURL>/api/v1/search/raw` (`tools/search_http.go:77`), served by
  `api/api.go:353` → `api/api_search.go:153` `handleRawSearch`. If `pluginURL` is
  wrong or unreachable, **search tools fail while every other tool works** — a
  useful discriminator when only search is broken.

#### Tool definitions and visibility

- **Definitions**: `mcpserver/tools/`, one file per resource (`posts.go`,
  `channels.go`, `files.go`, …). Registered via per-area `getXTools()` functions
  aggregated by `mcpTools()` in `tools/provider.go:122`.
- **Proxying**: `mcpserver/proxy_tools.go` forwards to external MCP servers.
- **Tools can disappear from `tools/list`.** An `Available func() bool` predicate
  (`tools/provider.go:73`) is re-evaluated on *every* `tools/list` call, so the
  tool list is not static across calls. Automation tools are always registered but
  gated on `isAutomationPluginInstalled()` (`tools/automations.go:24`). A missing
  automation tool therefore means the gate returned false, not that the tool was
  never registered.
- **The automation gate is 404-based, not success-based.** It GETs the automation
  plugin's `/automations` route and treats *any* status other than 404 as
  installed — 401 and 403 both count as installed. Only a 404, a malformed
  request, or a transport error yield false; the latter two log
  `Automation plugin check failed: bad request` / `: connection error`. So "tool
  missing" points at a 404 (plugin genuinely absent) or a connection failure,
  never at an auth problem.
- **Dev-only tools** (`getDevUserTools`, `getDevPostTools`, `getDevTeamTools`) are
  appended only when `p.devMode` is set (`tools/provider.go:147`), so they should
  never appear on a production server.

## Embeddings and Semantic Search

### Vector storage

- **Embeddings**: `embeddings/embeddings.go` — abstraction over embedding providers
- **pgvector**: `postgres/` — PostgreSQL pgvector extension for vector storage and similarity search
- **Indexing**: `indexer/` — indexes documents (posts, channels) into vector embeddings

### Search flow

1. User asks a natural language question
2. Question is embedded into a vector via `embeddings/`
3. Vector similarity search against `postgres/` pgvector store
4. Results returned with relevant context for the LLM

### Requirements

- PostgreSQL with pgvector extension enabled
- Embedding model configured (usually same provider as LLM)

## Streaming

- **Stream handling**: `streaming/` — manages streaming responses to clients
- **LLM streaming**: `llm/stream.go` — streams tokens from LLM providers
- **Frontend**: `webapp/src/websocket.ts` — WebSocket client receives streaming events
- **Mobile**: Uses `custom_mattermost-ai_postupdate` WebSocket event for streaming updates

## Configuration

### Plugin settings

Settings defined in `plugin.json` and managed via `config/`:
- LLM service connections (provider, API key, model, URL)
- Bot configurations (name, instructions, model overrides)
- Embedding settings
- MCP server connections
- Access control (allow/block lists per channel/user/team)

### Database

- **Store**: `store/store.go` — database operations
- **Migrations**: `store/migrate.go` — schema migrations run on plugin activation

## Enterprise Features

- **License checking**: `enterprise/` — validates enterprise license for premium features
- **Access control**: Role-based restrictions on bot access
- **Team restrictions**: Limit AI bots to specific teams

> `bifrost/` is **not** an enterprise feature — it is the LLM gateway client used by all
> installations. See [LLM Provider System](#llm-provider-system).

## Prompt Templates

- **Location**: `prompts/*.tmpl` — Go template files
- **System prompts**: Template-based system instructions for each bot
- **Summarization**: Channel and thread summarization templates
- **Action items**: Task extraction templates

## Common Support Investigation Patterns

### "Which LLM providers are supported?"
1. Check `llm/service_types.go` for provider type constants
2. Check `bifrost/config.go` (`MapServiceTypeToProvider` / `IsSupported`) for what is
   *actually* constructible at runtime — this is the authoritative list, not `llm/providers.go`
3. Check `plugin.json` and `webapp/src/components/system_console/service.tsx` for what the
   admin UI *offers* — a type can be offered here and still fail at step 2 (see Scale gotcha)

### "AI bot not responding to messages"
1. Trace the flow: `server/main.go` (MessageHasBeenPosted hook) → `conversations/handle_messages.go` → `toolrunner/toolrunner.go`
2. Check bot config: `bots/bots.go` — is the bot enabled and configured?
3. Check LLM connection: `llm/configuration.go` — API key, URL, model settings
4. Check permissions: `enterprise/` — is the user/channel allowed?
5. Check streaming: `streaming/` — is the response getting back to the client?

### "MCP tools not working"
1. Check MCP client config: `mcp/client_manager.go`
2. Check OAuth flow: `mcp/client_manager.go` — OAuth session management
3. Check tool registration: `llm/tools.go` and `mcp/client.go`
4. Check auto-approval settings in conversation config

### "Embeddings/vector search not returning results"
1. Check pgvector setup: `postgres/` — is PostgreSQL pgvector extension enabled?
2. Check indexing: `indexer/` — are documents being indexed?
3. Check embedding config: `embeddings/embeddings.go` — is the embedding model configured?
4. Check search: `search/` — search query and result handling

### "Streaming responses not working"
1. Check `streaming/` for server-side stream handling
2. Check `llm/stream.go` for LLM provider streaming
3. Check `webapp/src/websocket.ts` for frontend WebSocket handling
4. Mobile: check `custom_mattermost-ai_postupdate` WebSocket event handling

### "Where are API endpoints defined?"
1. All HTTP routes: `api/api.go` — Gin framework router
2. LLM Bridge API for external integrations
3. MCP endpoints for OAuth and tool management
4. Admin endpoints for configuration

### "What license is required for AI features?"
1. Check `enterprise/` for feature-to-license mapping
2. Access control features require enterprise license
3. Basic bot functionality may work without enterprise license depending on configuration
