# Mattermost Calls Plugin Codebase Guide (Support Focus)

This guide helps navigate the Mattermost Calls plugin codebase to answer support questions: finding where call config is validated, how WebRTC connections are established, where recording/transcription is handled, how TURN credentials work, etc.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/ClaudeFiles/Mattermost-Plugin-Calls/CLAUDE.md`

## Related Repositories

- **Mattermost Server**: `../Mattermost/` — plugin API, WebSocket events, config model, license checks
- **Enterprise**: `../Enterprise/` — enterprise license features
- **Desktop**: `../Desktop/` — desktop Calls widget integration (runs in its own WebContentsView)
- **Mobile**: `../Mattermost-Mobile/` — mobile Calls product module (`app/products/calls/`)

## Repository Structure

```
server/                     # Go backend
├── main.go                 # Plugin entry point
├── plugin.go               # Plugin struct, lifecycle hooks
├── activate.go             # OnActivate/OnDeactivate hooks
├── configuration.go        # Configuration struct and validation
├── api.go                  # HTTP API endpoint handlers
├── api_router.go           # HTTP route registration
├── websocket.go            # WebSocket event handlers (~47KB, largest file)
├── session.go              # Call session management
├── state.go                # Call state tracking
├── rtcd.go                 # RTCD service management (external WebRTC daemon)
├── recording_api.go        # Recording API endpoints
├── transcription_api.go    # Transcription API endpoints
├── job_service.go          # Job service for recording/transcription
├── host_controls.go        # Host controls logic (mute, remove participants)
├── host_controls_api.go    # Host controls API endpoints
├── limits.go               # Participant and feature limits
├── cluster_message.go      # Cluster synchronization for multi-node
├── db/                     # Database models (calls, channels, stats)
├── public/                 # Public interfaces (version, call state, stats)
├── enterprise/             # Enterprise license checking
├── license/                # License validation package
├── cluster/                # Cluster-aware operations
├── batching/               # Session event batching optimization
└── i18n/                   # Server-side translations (26+ languages)
webapp/                     # React/TypeScript frontend
├── src/
│   ├── index.tsx           # Plugin webapp entry point
│   ├── components/         # UI components
│   │   ├── call_widget/    # In-channel call widget
│   │   ├── expanded_view/  # Expanded call view
│   │   ├── incoming_calls/ # Incoming call notification
│   │   ├── modals/         # Call-related modals
│   │   └── screen_sharing/ # Screen sharing UI
│   ├── actions/            # Redux actions
│   ├── types/              # TypeScript definitions
│   └── websocket.ts        # WebSocket client for call events
├── tests/                  # Jest test files
└── i18n/                   # Frontend translations (28+ languages)
standalone/                 # Web-only client for embedded calls (recording widget)
e2e/                        # Playwright E2E tests
lt/                         # Load testing client
plugin.json                 # Plugin manifest with settings schema
Makefile                    # Build system
```

## Configuration

### Plugin settings

All configuration is defined in `server/configuration.go` as the configuration struct. Settings are exposed in the Mattermost System Console via the schema in `plugin.json`.

### Key configuration areas

| Area | Settings | Notes |
|------|----------|-------|
| **ICE/TURN** | ICE host override, TURN credentials, TURN static auth secret, TURN servers | Critical for connectivity |
| **UDP/TCP** | UDP server address, UDP server port, TCP server address, TCP server port | Network binding |
| **RTCD** | RTCD service URL | External WebRTC daemon for scalable deployments |
| **Recording** | Recording quality, max recording duration | Requires calls-offloader |
| **Transcription** | Transcription model size, language, live captions | Requires calls-transcriber |
| **Limits** | Max call participants, default enabled | Per-channel and global |
| **Ringing** | Enable ringing for DM/GM | Push notification integration |
| **Screensharing** | Enable screensharing | Per-deployment toggle |

## WebRTC / RTC System

### How calls are established

1. User initiates a call → WebSocket event to server
2. Server creates call state (`server/state.go`) and session (`server/session.go`)
3. WebRTC signaling happens via WebSocket (`server/websocket.go`)
4. ICE candidates are exchanged for peer-to-peer or relayed connections
5. Media flows via pion WebRTC library (Go-native WebRTC)

### STUN/TURN

- STUN: Used for NAT traversal discovery
- TURN: Relay server for when direct connections fail (firewall, symmetric NAT)
- TURN credentials: Short-lived auth tokens with configurable expiration
- Static auth secret: Shared secret for TURN credential generation

## RTCD Offload Service

For scalable deployments, calls can be offloaded to an external RTCD service:

- **Configuration**: `RTCDServiceURL` in plugin settings
- **Management**: `server/rtcd.go` — connects to and manages the RTCD service
- **How it works**: Instead of handling WebRTC in the plugin process, media is routed through RTCD
- **When to use**: High participant counts, dedicated media servers, reducing server load

## Call State Management

- **Call state**: `server/state.go` — tracks active calls per channel (participants, start time, recording status)
- **Sessions**: `server/session.go` — individual participant sessions within a call
- **Database**: `server/db/` — persistent storage for call records, channel settings, stats
- **Cluster sync**: `server/cluster_message.go` — syncs call state across Mattermost nodes in HA deployments

## Recording and Transcription

### Recording

- **API**: `server/recording_api.go` — start/stop recording endpoints
- **Job service**: `server/job_service.go` — manages recording/transcription jobs
- **External dependency**: Requires `calls-offloader` service
- **Output**: Recordings are stored as file attachments in the channel

### Transcription

- **API**: `server/transcription_api.go` — transcription endpoints
- **Models**: Whisper-based (local) or Azure Speech API
- **Live captions**: Real-time transcription during calls
- **Configuration**: Model size, language, live captions toggle

## License and Limits

- **License checking**: `server/enterprise/` and `server/license/` — validates enterprise license for premium features
- **Participant limits**: `server/limits.go` — enforces max participants per call
- **Feature gating**: Recording, transcription, and advanced features require enterprise license
- **Default limits**: Community edition has participant caps; enterprise unlocks higher limits

## Host Controls

- **Logic**: `server/host_controls.go` — muting participants, removing from call, locking calls
- **API**: `server/host_controls_api.go` — HTTP endpoints for host actions
- **Permissions**: Only the call host (initiator) or system admins can use host controls

## WebSocket Events

`server/websocket.go` is the largest file (~47KB) and handles all real-time call events:

- Call started/ended
- User joined/left
- ICE candidates
- Screen sharing start/stop
- Mute/unmute
- Recording/transcription state changes
- Host control actions
- Reactions during calls

## Common Support Investigation Patterns

### "Users can't connect to a call"
1. Check ICE/TURN config in `server/configuration.go` — are TURN servers configured?
2. Check UDP/TCP port settings — is the server binding to the right address?
3. Check firewall rules — UDP ports need to be open for WebRTC
4. Check `server/websocket.go` for signaling errors
5. If using RTCD: check `server/rtcd.go` for connectivity to the RTCD service

### "RTCD service not working"
1. Check `RTCDServiceURL` in plugin config
2. Look at `server/rtcd.go` for connection and health check logic
3. Verify network connectivity between Mattermost and RTCD
4. Check RTCD service logs separately

### "Recording or transcription not working"
1. Check if enterprise license includes the feature: `server/license/`, `server/enterprise/`
2. Check job service: `server/job_service.go`, `server/job_metadata.go`
3. Check recording config: `server/recording_api.go`
4. Verify calls-offloader/calls-transcriber services are running
5. Check transcription model config: model size, language settings

### "Call participant limit reached"
1. Check `server/limits.go` for limit enforcement
2. License tier determines max participants: `server/license/`
3. Config-level override in plugin settings

### "Calls not working in HA/clustered deployment"
1. Check `server/cluster_message.go` for inter-node state sync
2. Call state must be replicated across nodes: `server/state.go`
3. If using RTCD: all nodes must be able to reach the RTCD service

### "Screen sharing not working"
1. Check if screensharing is enabled in plugin config
2. Check `webapp/src/components/screen_sharing/` for frontend issues
3. Browser permissions for screen capture
4. VP8/AV1 codec support varies by browser

### "What license is required for a specific calls feature?"
1. Check `server/enterprise/` for feature-to-license mapping
2. Check `server/license/` for license validation functions
3. Recording and transcription typically require enterprise license
