# Mattermost RTCD Codebase Guide (Support Focus)

This guide helps navigate the `rtcd` (real-time communication daemon) codebase to answer support questions: how the WebRTC SFU is configured, how clients authenticate, how ICE/TURN candidates are generated, where media routing lives, why a deployment can't reach the service, etc.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/ClaudeFiles/Mattermost-RTCD/CLAUDE.md`

## What rtcd is

`rtcd` is a standalone Go service that offloads WebRTC and media-processing work from the [Mattermost Calls plugin](https://github.com/mattermost/mattermost-plugin-calls). It runs as a separate daemon (Docker, systemd, or bare binary) so calls can scale independently of Mattermost server processes. It's only meant to be interfaced by the Calls plugin — it is not a general-purpose SFU.

## Related Repositories

- **Calls plugin**: `../Mattermost-Plugin-Calls/` — the only supported client of `rtcd`. The plugin's `server/rtcd.go` connects to and manages this service.
- **Mattermost server**: `../Mattermost/` — hosts the Calls plugin; users connect via WebSocket through the plugin
- **Enterprise**: `../Enterprise/` — RTCD usage typically requires an enterprise license (gated by Calls plugin license checks)

## Repository Structure

```
cmd/rtcd/                   # Main entry point — CLI flag parsing, config load, signal handling
├── main.go                 # Service bootstrap (load config → New → Start → wait for SIGINT/SIGTERM → Stop)
└── config.go               # CLI config loading (TOML file + env overrides)
config/
└── config.sample.toml      # Documented sample config — primary reference for all settings
service/                    # Main service package (singleton orchestration)
├── service.go              # Service struct: wires api + ws + rtc + auth + store
├── session.go              # Per-client session state
├── client.go               # Internal client tracking
├── client_msg.go           # Message types exchanged with the Calls plugin
├── auth.go                 # Auth glue between API/WS handlers and auth service
├── audit.go                # Audit logging
├── system.go               # System info / version endpoints
├── version.go              # Build version metadata
├── option.go               # Service options
├── api/                    # HTTP API server (decoupled from business logic via handlers)
│   ├── server.go           # HTTP server, TLS, routing
│   └── mux.go              # Route definitions
├── ws/                     # WebSocket server + client (client is exported for plugin use)
│   ├── server.go           # WS server (used by service)
│   ├── client.go           # WS client (imported by Calls plugin)
│   ├── conn.go             # Connection wrapper
│   └── msg.go              # WS message types
├── rtc/                    # WebRTC SFU — the heart of media routing
│   ├── server.go           # RTC server lifecycle (start/stop, listeners)
│   ├── call.go             # Per-call state and orchestration
│   ├── group.go            # Call group / channel grouping
│   ├── session.go          # Per-participant RTC session
│   ├── sfu.go              # Selective Forwarding Unit logic (forward tracks between sessions)
│   ├── track.go            # Audio/video/screen track handling
│   ├── simulcast.go        # Simulcast layer selection
│   ├── stun.go             # STUN handling for public IP discovery
│   ├── turn.go             # TURN credential generation (HMAC w/ static auth secret)
│   ├── net.go              # UDP/TCP listener setup, multi-socket binding
│   ├── multi_conn.go       # Multi-socket UDP for performance scaling
│   ├── rate.go             # Rate limiting / pacing
│   ├── msg.go              # RTC signaling messages (SDP, ICE)
│   ├── metrics.go          # Prometheus metrics for RTC
│   ├── logger.go           # RTC-specific logger wiring (pion log adapter)
│   ├── dc/                 # Data channel handling (lock + msg)
│   ├── stat/               # Stats collection (per-stream/per-session)
│   └── vad/                # Voice activity detection
├── auth/                   # Authentication service
│   ├── service.go          # Register / unregister / authenticate clients
│   ├── crypto.go           # bcrypt hashing of authKey, bearer token generation
│   └── session_cache.go    # Cache of authenticated sessions (TTL configurable)
├── store/                  # Persistent K/V store (registered clients + hashed credentials)
│   ├── store.go            # Store interface
│   └── bitcask.go          # Bitcask implementation (embedded, on-disk)
├── perf/
│   └── metrics.go          # Prometheus metrics registration
└── random/                 # Crypto-safe random IDs
client/                     # Public Go client library — imported by Calls plugin
├── client.go               # Client connect/auth lifecycle
├── api.go                  # HTTP API calls
├── call.go                 # Call session client
├── rtc.go                  # RTC client
├── rtc_monitor.go          # RTC connection monitoring
├── websocket.go            # WS client wrapper
└── config.go               # Client config
logger/                     # Mattermost-style logger config (console + file, JSON option)
docs/                       # User-facing docs
├── getting_started.md      # Deployment guide (systemd / Docker)
├── implementation.md       # Architecture overview with mermaid diagrams
├── project_structure.md    # Top-level package layout
├── env_config.md           # Generated list of RTCD_* env vars (mirrors config.sample.toml)
└── security.md             # Auth flow + WebRTC signaling security
build/                      # Dockerfile + build artifacts
scripts/                    # Build/release helpers
testfiles/                  # Test fixtures
go.mod                      # Module: github.com/mattermost/rtcd
Makefile                    # `make go-run`, `make test`, build/release targets
```

## Configuration

### Where settings live

- **Sample config + inline docs**: `config/config.sample.toml` — the primary reference for any config question
- **Env override format**: All settings can be overridden with `RTCD_<SECTION>_<KEY>` (uppercase, underscores). Full list in `docs/env_config.md`.
- **Config loading**: `cmd/rtcd/config.go` (CLI loader) and per-package `config.go` files for validation
- **Default config path**: `config/config.toml` (the service starts with built-in defaults if none exists)

### Sections

| Section | Purpose | Key fields |
|---------|---------|------------|
| `[api]` | HTTP/WS listener, TLS, auth | `http.listen_address` (default `:8045`), `http.tls.*`, `security.allow_self_registration`, `security.enable_admin`, `security.admin_secret_key`, `security.session_cache.expiration_minutes` |
| `[rtc]` | WebRTC SFU networking | `ice_address_udp`/`ice_port_udp` (default `:8443`), `ice_address_tcp`/`ice_port_tcp`, `ice_host_override`, `ice_host_port_override`, `ice_servers`, `turn.static_auth_secret`, `turn.credentials_expiration_minutes`, `enable_ipv6`, `udp_sockets_count`, `nack_buffer_size`, `nack_disable_copy` |
| `[store]` | Persistent K/V store path | `data_source` (default `/tmp/rtcd_db`) |
| `[logger]` | Logging | `enable_console`, `console_json`, `console_level`, `enable_file`, `file_json`, `file_level`, `file_location`, `enable_color` |

### Default ports

- `8045/tcp` — HTTP/WebSocket API (Calls plugin connects here)
- `8443/udp` — primary media transport (audio/video/screen)
- `8443/tcp` — fallback media transport when UDP is blocked

## Authentication Flow

The Calls plugin (the only client) authenticates over HTTP/WS using either basic auth or bearer tokens. See `service/auth/`.

### Registration (`POST /register`)
1. Client sends `{clientID, authKey}` JSON
2. Server bcrypts the `authKey` and stores it in the bitcask K/V store under `clientID`
3. Returns 201 on success

Two registration modes (`config/config.sample.toml [api]`):
- **Self-registration** (`security.allow_self_registration = true`): any client can call `/register`. Fine on private networks.
- **Admin-mediated** (`security.enable_admin = true` + `security.admin_secret_key`): only the superuser can register clients via basic auth with the admin secret.

By default the Calls plugin uses the Mattermost diagnostic ID as `clientID`.

### Authentication

- **Basic auth**: client sends `clientID:authKey`, server bcrypt-compares against stored hash
- **Bearer auth**: client posts credentials to get a token, then uses `Authorization: Bearer <token>`. Tokens cached in `service/auth/session_cache.go` with TTL `security.session_cache.expiration_minutes` (default 1440).

## WebRTC / SFU Architecture

`rtcd` is a Selective Forwarding Unit — it accepts media tracks from each participant and forwards them to other participants without transcoding. Built on [pion](https://github.com/pion) (Go-native WebRTC).

### Signaling path

```
User → Calls plugin (Mattermost server) → rtcd
       (WebSocket)                        (WebSocket: relayed SDP/ICE)
```

The user's browser never talks to `rtcd` over WebSocket — only over WebRTC for media. SDP offers/answers and ICE candidates are relayed through the Calls plugin (see `docs/security.md` mermaid diagram).

### Connection establishment

1. Plugin opens WS to `rtcd` (per-Mattermost-instance connection)
2. Plugin relays SDP offer from user → `rtcd` returns SDP answer
3. ICE candidates flow both directions (relayed via plugin)
4. Once ICE succeeds, media flows User ↔ rtcd directly via UDP (or TCP fallback)
5. `rtcd` forwards each user's tracks to every other user in the same call

### Public IP / NAT handling

Critical for connectivity — most "users can't connect" issues live here.

- `ice_host_override` — explicit public IP/hostname to advertise. **Required behind NAT/load balancer.**
  - Single value: `8.8.8.8`
  - Multiple NAT mappings: `8.8.8.8/10.0.2.2,8.8.4.4/10.0.2.1` (external/internal pairs)
- `ice_host_port_override` — advertise a different port than the listener (e.g., when an NLB rewrites ports)
  - Per-IP mapping: `localIPA/8443,localIPB/8444` for K8s deployments where one config serves multiple pods
- `ice_servers` — STUN/TURN servers for users behind restrictive NATs. Format:
  ```toml
  ice_servers = [
    {urls = ["stun:..."], username = "...", credential = "..."},
    {urls = ["turn:..."], username = "...", credential = "..."}
  ]
  ```
- `turn.static_auth_secret` — shared secret for generating short-lived TURN credentials (HMAC-based, RFC 7635 style). Implementation in `service/rtc/turn.go`.

### Multi-socket UDP

`udp_sockets_count` opens N UDP sockets per local address to reduce contention (default scales with CPU count × 100). Implementation in `service/rtc/multi_conn.go`. Trade-off: more sockets = more file descriptors.

### NACK / packet loss recovery

- `nack_buffer_size` — packets buffered per video SSRC for retransmission. Powers of 2, default 256 (~8.5s @ 30fps).
- `nack_disable_copy` — perf vs. stability trade-off. **Default `true` for back-compat, but should be `false` for production** (avoids memory corruption when ring buffer wraps). Source comment lives in `config/config.sample.toml`.

## Persistent Store

`service/store/bitcask.go` — embedded K/V store. Stores:
- Registered client IDs → bcrypt-hashed authKeys
- Configured via `[store] data_source` (default `/tmp/rtcd_db` — **change for production**)

If the data source is wiped, all registered clients are lost and must re-register.

## Common Support Investigation Patterns

### "Calls plugin can't connect to rtcd"
1. Check `RTCDServiceURL` in Calls plugin config (in Mattermost) — format `http://clientID:authKey@host:8045` or just `http://host:8045`
2. Verify rtcd HTTP API is reachable: `curl http://<host>:8045/version`
3. Check rtcd auth config:
   - If plugin is self-registering, confirm `RTCD_API_SECURITY_ALLOWSELFREGISTRATION=true` (or `[api] security.allow_self_registration = true`)
   - Look for registration entries in the bitcask store path (`[store] data_source`)
4. Check rtcd logs for auth failures (`service/auth/service.go`)
5. Plugin-side: `Mattermost-Plugin-Calls/server/rtcd.go` for client connection logic

### "Users can't establish media (WebRTC) connection"
1. Check `[rtc] ice_host_override` — is the public IP/hostname advertised correctly?
2. Check firewall: `8443/udp` (primary) and `8443/tcp` (fallback) must be open inbound to rtcd
3. If behind NAT/LB: confirm `ice_host_override` matches public IP and `ice_host_port_override` matches LB port mapping
4. Check `ice_servers` — for users behind restrictive NATs, TURN must be configured
5. RTC server lifecycle: `service/rtc/server.go`
6. Listener binding: `service/rtc/net.go`, `service/rtc/multi_conn.go`

### "Calls work but disconnect under load"
1. Check `udp_sockets_count` — too low causes contention. Defaults scale with CPU.
2. Check `nack_buffer_size` — too small means lost packets aren't retransmitted; too large = memory pressure
3. Check `nack_disable_copy` — if `true`, can cause crashes when ring buffer wraps. Recommend `false` in prod.
4. File descriptor limits on the host (multi-socket UDP can open hundreds per address)
5. Metrics: `service/perf/metrics.go`, `service/rtc/metrics.go` — Prometheus scraping

### "TURN credentials not working"
1. `turn.static_auth_secret` must match between rtcd and the TURN server (TURNServer-side coturn, etc.)
2. Credential generation: `service/rtc/turn.go`
3. Expiration: `turn.credentials_expiration_minutes` (default 1440)

### "rtcd lost all client registrations"
1. Check if `[store] data_source` was on ephemeral storage (`/tmp/`, container without volume mount)
2. Bitcask data should be on a persistent volume in production
3. Re-registration: clients with `allow_self_registration` will re-register automatically; otherwise admin must re-create

### "Multiple Mattermost instances sharing one rtcd"
- Each Mattermost installation registers as a separate client (different `clientID`s)
- Calls from different installations are isolated (separate call groups in `service/rtc/group.go`)
- Single rtcd can serve many installations — see `docs/implementation.md` "Sample deployment" diagram

### "What version is running?"
- HTTP: `GET /version` returns build metadata
- Source: `service/version.go`

## Build / Run

- `make go-run` — build and run with default config
- `make test` — run test suite
- Docker image: `mattermost/rtcd:latest` (Dockerfile in `build/Dockerfile`)
- Module path: `github.com/mattermost/rtcd`

## Key External Dependencies

- **pion/webrtc** — WebRTC implementation (Go-native, no CGO)
- **bitcask** — embedded persistent K/V store (`git.mills.io/prologic/bitcask`)
- **mattermost/logr** — structured logging (shared with Mattermost server)
