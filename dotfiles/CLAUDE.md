# Mattermost CLAUDE.md Dotfiles

This repo manages `CLAUDE.md` files for a set of Mattermost reference repositories in `~/Repositories/Claude-Repos/`. Each subdirectory here corresponds to a repo, and its `CLAUDE.md` is symlinked into the target repo root.

## Purpose

These repos are **read-only reference copies** used for code search and support/troubleshooting. They are NOT for active development. When investigating a support issue:

1. **Refresh the repo first**: `cd ~/Repositories/Claude-Repos/{RepoName} && git fetch origin && git pull`
2. **Search the code** to find where config is validated, where errors originate, how features are gated, etc.
3. **DO NOT** make local code changes, create branches, or commit in the target repos

## Repository Mapping

| Dotfiles Subdir | Target Repo | Description |
|---|---|---|
| `mattermost/` | `~/Repositories/Claude-Repos/Mattermost/` | Go/React monorepo — server, webapp, mmctl, API |
| `mattermost-enterprise/` | `~/Repositories/Claude-Repos/Enterprise/` | Enterprise features — LDAP, SAML, clustering, compliance, data retention |
| `Desktop/` | `~/Repositories/Claude-Repos/Desktop/` | Electron desktop app — multi-server, notifications, certificates |
| `Mattemrost-Plugin-Calls/` | `~/Repositories/Claude-Repos/Mattemrost-Plugin-Calls/` | WebRTC plugin — voice/video calls, screensharing, recording, transcription |
| `Mattermost-Mobile/` | `~/Repositories/Claude-Repos/Mattermost-Mobile/` | React Native iOS/Android app — WatermelonDB, dual database, products |
| `Mattermost-Plugin-Agents/` | `~/Repositories/Claude-Repos/Mattermost-Plugin-Agents/` | AI/LLM plugin — multi-provider, MCP, embeddings, tool calling |
| `Mattermost-Plugin-Boards/` | `~/Repositories/Claude-Repos/Mattermost-Plugin-Boards/` | Kanban boards plugin — boards, blocks, cards, templates |
| `Mattermost-Plugin-Playbooks/` | `~/Repositories/Claude-Repos/Mattermost-Plugin-Playbooks/` | Incident management plugin — playbooks, runs, checklists, automation |

## Symlink Setup

Each `CLAUDE.md` is symlinked from this repo into the target:

```bash
ln -sf ~/Repositories/Claude-Stuff/dotfiles/{SubDir}/CLAUDE.md ~/Repositories/Claude-Repos/{RepoDir}/CLAUDE.md
```

For repos where `CLAUDE.md` is NOT tracked upstream (Mattermost, Enterprise, Calls, Boards, Playbooks), the symlink is hidden from git via `.git/info/exclude`.

For repos where `CLAUDE.md` IS tracked upstream (Desktop, Mobile, Agents), the symlink overrides the upstream file and `git update-index --skip-worktree CLAUDE.md` prevents git from showing it as modified or overwriting it on pull.

## Cross-Repository Investigations

Most support issues span multiple repos. Common patterns:

- **Enterprise feature not working**: Check license in `mattermost/` (`server/public/model/license.go`), config in `mattermost/` (`server/public/model/config.go`), implementation in `mattermost-enterprise/`
- **Plugin issue**: Check plugin code in its repo, then trace the plugin API call back to `mattermost/` (`server/public/plugin/api.go`)
- **Mobile/Desktop issue**: Check client code in the respective repo, then trace the API call or WebSocket event back to `mattermost/`
- **Error message lookup**: Find the translation ID in `mattermost/` (`server/i18n/en.json`), then grep for where it's raised
