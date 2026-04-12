# Mattermost Agents Plugin Codebase Guide (Support Focus)

This guide helps navigate the Mattermost Agents (AI) plugin codebase to answer support questions: finding where LLM providers are configured, how conversations flow, where MCP tools are managed, how embeddings and search work, etc.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/ClaudeFiles/Mattermost-Plugin-Agents/CLAUDE.md`

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
├── auto_run_tools.go       # Automatic tool execution
conversations/              # Conversation management
├── conversations.go        # Conversation lifecycle
├── handle_messages.go      # Message handling and routing
├── completion.go           # LLM completion orchestration
mcp/                        # Model Context Protocol client
├── client.go               # MCP client implementation
├── client_manager.go       # MCP session and OAuth management
mcpserver/                  # Embedded MCP server
├── handlers.go             # MCP server request handlers
├── tools.go                # MCP tool definitions
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
bifrost/                    # Enterprise agentic features (Bifrost integration)
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
- **Provider registry**: `llm/providers.go` — factory that creates the right client per type
- **Configuration**: `llm/configuration.go` — `ServiceConfig` (provider connection) and `BotConfig` (bot behavior)
- **Core interface**: `llm/language_model.go` — `LanguageModel` interface all providers implement
- **Streaming**: `llm/stream.go`, `llm/stream_generator.go` — streaming response handling

## Conversation Flow

### How a message becomes an AI response

1. User posts a message mentioning the bot or in a DM with the bot
2. Plugin hook `MessageHasBeenPosted` fires → `server/main.go`
3. Message routed to `conversations/handle_messages.go`
4. Context assembled (thread history, system prompt, tools)
5. Completion requested: `conversations/completion.go` → `llm/` → provider API
6. Response streamed back via SSE → posted as bot message
7. If tools are called: tool execution → results fed back → follow-up completion

### Tool calling

- **Tool definitions**: `llm/tools.go` — defines available tools
- **Auto-execution**: `llm/auto_run_tools.go` — tools that run without user approval
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

- **Embedded MCP server**: `mcpserver/` — the plugin itself exposes MCP tools
- **Handlers**: `mcpserver/handlers.go` — processes incoming MCP requests
- **Tool definitions**: `mcpserver/tools.go` — tools exposed to MCP clients

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
- **Bifrost**: `bifrost/` — enterprise agentic orchestration features

## Prompt Templates

- **Location**: `prompts/*.tmpl` — Go template files
- **System prompts**: Template-based system instructions for each bot
- **Summarization**: Channel and thread summarization templates
- **Action items**: Task extraction templates

## Common Support Investigation Patterns

### "Which LLM providers are supported?"
1. Check `llm/service_types.go` for provider type constants
2. Check `llm/providers.go` for the provider factory
3. Check `plugin.json` for the admin UI configuration schema

### "AI bot not responding to messages"
1. Trace the flow: `server/main.go` (MessageHasBeenPosted hook) → `conversations/handle_messages.go` → `conversations/completion.go`
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
