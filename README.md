# Mattermost Claude Code Config

This repo is the single source of truth for `CLAUDE.md` files, slash commands, and skills used across Mattermost projects. Files are symlinked into their target directories.

## What are these files?

`CLAUDE.md` is a project guide that [Claude Code](https://docs.anthropic.com/en/docs/claude-code) reads when it opens a repository. These guides are written with a **support and troubleshooting** focus — they help Claude navigate the codebase to answer questions like:

- Where is a config setting validated?
- Where does an error message originate?
- How is a feature license-gated?
- What API endpoint handles a given request?

The target repos are **read-only reference copies** — they exist for code search, not active development.

## Directory Layout

```
Claude-Stuff/
├── README.md                                   # This file
├── ClaudeFiles/                                # CLAUDE.md files for reference repos
│   ├── CLAUDE.md                               # Guide for this directory itself
│   ├── Mattermost/CLAUDE.md                    # Mattermost server monorepo (Go/React)
│   ├── Mattermost-Enterprise/CLAUDE.md         # Enterprise features (LDAP, SAML, clustering, etc.)
│   ├── Mattermost-Docs/CLAUDE.md               # Sphinx docs site (docs.mattermost.com)
│   ├── Mattermost-Mobile/CLAUDE.md             # React Native iOS/Android app
│   ├── Desktop/CLAUDE.md                       # Electron desktop app
│   ├── Mattermost-Operator/CLAUDE.md           # Kubernetes operator
│   ├── Mattermost-RTCD/CLAUDE.md               # WebRTC SFU offload daemon for Calls
│   ├── Mattermost-Plugin-Agents/CLAUDE.md      # AI/LLM integration plugin
│   ├── Mattermost-Plugin-Boards/CLAUDE.md      # Kanban boards plugin (Focalboard)
│   ├── Mattermost-Plugin-Calls/CLAUDE.md       # WebRTC voice/video/screensharing plugin
│   ├── Mattermost-Plugin-Confluence/CLAUDE.md  # Confluence integration plugin
│   ├── Mattermost-Plugin-Github/CLAUDE.md      # GitHub integration plugin
│   ├── Mattermost-Plugin-Gitlab/CLAUDE.md      # GitLab integration plugin
│   ├── Mattermost-Plugin-LegalHold/CLAUDE.md   # Legal Hold plugin (Enterprise-only)
│   └── Mattermost-Plugin-Playbooks/CLAUDE.md   # Incident management plugin
└── Tickets/                                    # Support ticket analysis workspace
    ├── CLAUDE.md                               # Ticket investigation guide
    ├── commands/                               # Slash commands
    │   ├── mm_ticket_command.md                # /mm_ticket_command — ticket investigation
    │   ├── mm_eir_command.md                   # /mm_eir_command — engineering incident report
    │   ├── mm_rca_command.md                   # /mm_rca_command — customer-facing RCA
    │   └── mm_retro_command.md                 # /mm_retro_command — post-resolution retrospective + KB ingest
    └── skills/
        └── upgrade-advisor/SKILL.md            # /upgrade-advisor — patch upgrade analysis
```

### Code reference repos

Each `ClaudeFiles/` subdirectory maps to a repo in `~/Repositories/Claude-Repos/`:

| ClaudeFiles Subdir | Target Repo | Description |
|---|---|---|
| `Mattermost/` | `Claude-Repos/Mattermost/` | Go/React monorepo — server, webapp, mmctl, API |
| `Mattermost-Enterprise/` | `Claude-Repos/Enterprise/` | Enterprise features — LDAP, SAML, clustering, compliance |
| `Mattermost-Docs/` | `Claude-Repos/Mattermost-Docs/` | Sphinx documentation site — docs.mattermost.com source |
| `Mattermost-Mobile/` | `Claude-Repos/Mattermost-Mobile/` | React Native app — WatermelonDB, dual database, products |
| `Desktop/` | `Claude-Repos/Desktop/` | Electron desktop app — multi-server, notifications, certificates |
| `Mattermost-Operator/` | `Claude-Repos/Mattermost-Operator/` | Kubernetes operator — CRDs, controllers, deployment sizing |
| `Mattermost-RTCD/` | `Claude-Repos/Mattermost-RTCD/` | WebRTC daemon — standalone Go SFU offload service for Calls |
| `Mattermost-Plugin-Agents/` | `Claude-Repos/Mattermost-Plugin-Agents/` | AI/LLM plugin — multi-provider, MCP, embeddings |
| `Mattermost-Plugin-Boards/` | `Claude-Repos/Mattermost-Plugin-Boards/` | Kanban boards — boards, blocks, cards, templates |
| `Mattermost-Plugin-Calls/` | `Claude-Repos/Mattermost-Plugin-Calls/` | WebRTC plugin — voice/video calls, recording, transcription |
| `Mattermost-Plugin-Confluence/` | `Claude-Repos/Mattermost-Plugin-Confluence/` | Confluence integration — OAuth, subscriptions, Server/Cloud webhooks, Forge bridge |
| `Mattermost-Plugin-Github/` | `Claude-Repos/Mattermost-Plugin-Github/` | GitHub integration — OAuth, subscriptions, webhooks, permalinks, review-SLA digest |
| `Mattermost-Plugin-Gitlab/` | `Claude-Repos/Mattermost-Plugin-Gitlab/` | GitLab integration — OAuth, subscriptions, webhooks, permalinks |
| `Mattermost-Plugin-LegalHold/` | `Claude-Repos/Mattermost-Plugin-LegalHold/` | Legal Hold — Enterprise-only, cluster-scheduled exports, HMAC-hashed archives |
| `Mattermost-Plugin-Playbooks/` | `Claude-Repos/Mattermost-Plugin-Playbooks/` | Incident management — playbooks, runs, checklists |

## How symlinks work

Each `CLAUDE.md` is symlinked from this repo into the target repo root so that Claude Code picks it up automatically:

```bash
ln -sf ~/Repositories/Claude-Stuff/ClaudeFiles/{SubDir}/CLAUDE.md ~/Repositories/Claude-Repos/{RepoDir}/CLAUDE.md
```

To keep the symlinks invisible to git in the target repos, two strategies are used depending on whether the upstream repo already tracks a `CLAUDE.md`:

### Repos where `CLAUDE.md` is NOT tracked upstream

Mattermost, Enterprise, Docs, Operator, RTCD, and the Boards, Calls, Confluence, Github, Gitlab, LegalHold, and Playbooks plugins — the symlink is an untracked file, so it just needs to be excluded:

```
# In each repo's .git/info/exclude:
CLAUDE.md
```

### Repos where `CLAUDE.md` IS tracked upstream

Desktop, Mobile, and the Agents plugin — these repos have their own `CLAUDE.md` committed. Our symlink replaces it, and `skip-worktree` tells git to ignore the change and preserve the symlink during `git pull`:

```bash
git update-index --skip-worktree CLAUDE.md
```

## Setting up a new repo

1. Create a subdirectory in `ClaudeFiles/` with a `CLAUDE.md` following the existing pattern
2. Clone the repo into `~/Repositories/Claude-Repos/`
3. Create the symlink:
   ```bash
   ln -sf ~/Repositories/Claude-Stuff/ClaudeFiles/{SubDir}/CLAUDE.md ~/Repositories/Claude-Repos/{RepoDir}/CLAUDE.md
   ```
4. If `CLAUDE.md` is **not** tracked upstream, add it to `.git/info/exclude`:
   ```bash
   echo "CLAUDE.md" >> ~/Repositories/Claude-Repos/{RepoDir}/.git/info/exclude
   ```
5. If `CLAUDE.md` **is** tracked upstream, set skip-worktree:
   ```bash
   cd ~/Repositories/Claude-Repos/{RepoDir} && git update-index --skip-worktree CLAUDE.md
   ```

## Tickets workspace

The `Tickets/` directory manages the Claude Code configuration for the support ticket analysis workspace at `~/Downloads/Tickets/`. Files are symlinked into their expected locations:

| Source | Target |
|---|---|
| `Tickets/CLAUDE.md` | `~/Downloads/Tickets/CLAUDE.md` |
| `Tickets/commands/*.md` | `~/Downloads/Tickets/.claude/commands/*.md` |
| `Tickets/skills/upgrade-advisor/SKILL.md` | `~/Downloads/Tickets/.claude/skills/upgrade-advisor/SKILL.md` |

## Editing guides

Edit the files in **this repo** — the symlinks ensure changes are immediately reflected in the target locations. Do not edit files directly in the target repos or directories.
