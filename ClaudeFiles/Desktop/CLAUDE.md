# Mattermost Desktop Codebase Guide (Support Focus)

This guide helps navigate the Mattermost Desktop codebase to answer support questions: finding where config is read, where notifications are handled, how certificate errors occur, how multi-server connections work, etc.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/ClaudeFiles/Desktop/CLAUDE.md`

## Related Repositories

- **Mattermost Server**: `../Mattermost/` — server API, WebSocket events, config model, error translations
- **Enterprise**: `../Enterprise/` — enterprise features (LDAP, SAML, clustering)
- **Mobile**: `../Mattermost-Mobile/` — React Native mobile app (shares server API surface)
- **Plugin-Calls**: `../Mattemrost-Plugin-Calls/` — Calls plugin (desktop has Calls widget integration)

## Repository Structure

```
api-types/              # @mattermost/desktop-api types (DesktopAPI interface exposed to servers)
i18n/                   # Localization language files
src/
├── main/               # Electron main process — OS-level functionality, app lifecycle
│   ├── app/            # App initialization, IPC handler registration
│   │   └── index.ts    # Entry point: initialize()
│   ├── notifications/  # OS notification system
│   ├── diagnostics/    # Diagnostic tools and logging
│   ├── security/       # Certificate store, permissions, pre-auth
│   └── server/         # Server info fetching, server API communication
├── app/                # High-level app modules (runs in main process)
│   ├── mainWindow/     # Main BrowserWindow, modals, dropdowns
│   ├── views/          # MattermostWebContentsView — one per server, loading screen
│   ├── windows/        # BaseWindow, popout manager
│   ├── menus/          # Application menu, tray context menu
│   ├── tabs/           # Tab management
│   ├── preload/        # Preload scripts bridging main ↔ renderer
│   │   ├── internalAPI.js   # window.desktop (full API for internal views)
│   │   └── externalAPI.ts   # window.desktopAPI (restricted API for server views)
│   └── system/         # Badge count, system tray
├── renderer/           # React UI for internal views (top bar, settings, modals)
│   ├── components/     # MainPage, settings panels
│   ├── modals/         # Modal dialogs (settings, certificate, server add, login)
│   ├── hooks/          # React hooks
│   └── css/            # SCSS stylesheets
├── common/             # Shared modules (no Electron-specific imports)
│   ├── config/         # Config read/write, defaults, upgrades, GPO/MDM overrides
│   ├── servers/        # ServerManager singleton — multi-server state
│   ├── views/          # ViewManager, MattermostView
│   ├── communication.ts # All IPC channel name constants
│   └── utils/          # URL utilities, constants, validators
├── assets/             # Platform icons, sounds
└── types/              # Shared TypeScript type definitions
e2e/                    # Playwright E2E tests (separate package.json)
electron-builder.json   # Packaging config
webpack.config.*.js     # Webpack configs (main, renderer, preload, base)
```

## Electron Process Model

The desktop app uses Electron's multi-process architecture:

- **Main process** (`src/main/` + `src/app/` + `src/common/`): Single Node.js process with OS access. Manages windows, notifications, IPC, downloads, auto-updates, security. Entry: `src/main/app/index.ts` → `initialize()`.
- **Renderer processes** (`src/renderer/`): Chromium instances for the app's own UI (top bar, settings, modals). React-based.
- **External views**: Each connected Mattermost server runs in its own `WebContentsView` (via `src/app/views/`), loading the web app directly from that server. Isolated from each other.
- **Preload scripts** (`src/app/preload/`): Secure bridge between main and renderer via `contextBridge`. Two APIs:
  - `internalAPI.js` → `window.desktop` — full API for trusted internal views
  - `externalAPI.ts` → `window.desktopAPI` — restricted API for external server views (the DesktopAPI)

## Configuration System

### Where config is stored

- **User config**: JSON file at the OS-specific user data path
  - macOS: `~/Library/Application Support/Mattermost/config.json`
  - Linux: `~/.config/Mattermost/config.json`
  - Windows: `%APPDATA%\Mattermost\config.json`
- **Config module**: `src/common/config/` — reading, writing, defaults, schema upgrades

### Managed deployment overrides

- **Windows GPO**: Registry-based policy (`src/common/config/`) — admins push config via Group Policy
- **macOS MDM**: Plist-based managed preferences (`src/common/config/`) — admins push config via MDM profiles
- These override user settings and may lock certain options in the UI

### Config defaults and upgrades

- `src/common/config/defaultPreferences.ts` — default values for all settings
- Config schema is versioned; upgrades migrate old config formats on app start

## IPC Communication

All IPC channel names are defined in `src/common/communication.ts`. This is the central registry — search here to find what channels exist.

- **Handlers registered**: `src/main/app/initialize.ts` (global handlers) and individual module constructors (scoped handlers)
- **Handler implementations**: `src/main/app/intercom.ts`
- **Patterns**: `handle`/`invoke` for request/response; `on`/`send` for fire-and-forget

## Server Connection Management

- **ServerManager** (`src/common/servers/`): Singleton managing the list of connected servers
- Each server gets its own `MattermostWebContentsView` (`src/app/views/`)
- Servers are loaded from config on startup; users can add/remove via UI
- Server validation: `src/main/server/` — fetches server info to verify connectivity

## Notification System

- **Notification handling**: `src/main/notifications/`
- Desktop receives push-style notifications via WebSocket events from each connected server
- Notifications flow: Server WebSocket → main process → OS notification API
- Badge count managed in `src/app/system/`

## Security and Certificates

- **Certificate store**: `src/main/security/` — manages trusted certificates
- **Certificate errors**: When a server presents an untrusted cert, the app shows a certificate trust dialog
- **Permissions**: `src/main/security/` — controls what external server views can access
- **Pre-auth**: Handles authentication challenges before loading server content

## Calls Integration

The desktop app has special integration with the Mattermost Calls plugin:

- **Calls widget**: Runs in its own `WebContentsView` (separate from the server view)
- **Calls integration**: `src/app/` contains Calls-specific window and view management
- DevTools for Calls widget: **View → Developer Tools for Calls Widget**

## Build System

### Webpack

Three webpack configs merging from `webpack.config.base.js`:

| Config | Target | Entry |
|---|---|---|
| `webpack.config.main.js` | `electron-main` | `src/main/app/index.ts` |
| `webpack.config.preload.js` | `electron-preload` | `internalAPI.js`, `externalAPI.ts` |
| `webpack.config.renderer.js` | `web` | Multiple entries in `src/renderer/` |

### Compile-time constants

`DefinePlugin` in `webpack.config.base.js` injects build-time globals: `__IS_MAC_APP_STORE__`, `__IS_NIGHTLY_BUILD__`, `__DISABLE_GPU__`, `__SKIP_ONBOARDING_SCREENS__`, `__SENTRY_DSN__`, `__HASH_VERSION__`.

### Packaging

`electron-builder.json` — builds platform packages:
- **Windows**: MSI + ZIP (x64, ARM64), Azure code signing, GPO templates bundled
- **macOS**: DMG + ZIP (x64, ARM64, Universal), hardened runtime, notarization, separate MAS config
- **Linux**: tar.gz, deb, rpm, AppImage, Flatpak

## Debugging and Diagnostics

### Debug logging

Open Settings (`Ctrl/Cmd+,`) → switch logging to **Debug** → reproduce → **View → Show Logs**. Turn off after.

### Developer Tools

| Target | How to open |
|---|---|
| Web App (current server) | **View → Developer Tools for Current Server** |
| Main Window wrapper | **View → Developer Tools for Application Wrapper** |
| Calls Widget (while active) | **View → Developer Tools for Calls Widget** |
| Modals | Run with `MM_DEBUG_MODALS=true` |

### Diagnostics module

`src/main/diagnostics/` — collects diagnostic info for support bundles

## Common Support Investigation Patterns

### "Where is desktop config setting X stored?"
1. Check `src/common/config/` for config handling
2. Look at `defaultPreferences.ts` for the default value
3. For managed deployments: check GPO (Windows registry) or MDM (macOS plist) override paths in `src/common/config/`

### "Why isn't a notification showing on desktop?"
1. Check `src/main/notifications/` for the notification pipeline
2. Verify the WebSocket event is reaching the main process (check server connection in `src/app/views/`)
3. Check OS-level notification permissions and Do Not Disturb state
4. Badge count: `src/app/system/`

### "Why is a certificate being rejected?"
1. Check `src/main/security/` for certificate store and validation
2. Look for certificate error handling in the security module
3. Self-signed certs require manual trust via the certificate dialog

### "How does the desktop Calls widget work?"
1. Calls widget runs in its own `WebContentsView`, separate from server views
2. Integration code in `src/app/` manages the Calls window lifecycle
3. Debug with **View → Developer Tools for Calls Widget**

### "How does auto-update work?"
1. `electron-builder.json` configures the update feed
2. `scripts/generate_latest_version.sh` generates `latest.json` for the auto-updater
3. Update check and download happens in the main process

### "Desktop app not connecting to server?"
1. Check `src/main/server/` for server info fetching and validation
2. Check `src/common/servers/` (ServerManager) for server list state
3. Check `src/main/security/` for certificate/permission issues
4. Look at `src/app/views/` for WebContentsView loading errors

### "User can't reproduce issue in browser — desktop only?"
1. The web app loads in an Electron `WebContentsView`, which differs from a regular browser
2. Check `src/app/preload/externalAPI.ts` for the DesktopAPI injected into the page
3. Check `src/main/app/` for Electron-specific behavior (deep links, protocol handling, file downloads)
4. Reproduction sequence: try in browser first, then Electron DevTools for the server view
