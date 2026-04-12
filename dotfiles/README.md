# Mattermost CLAUDE.md Dotfiles

This repo is the single source of truth for `CLAUDE.md` files used across a set of Mattermost reference repositories. Each subdirectory contains a `CLAUDE.md` that gets symlinked into the corresponding repo in `~/Repositories/Claude-Repos/`.

## What are these files?

`CLAUDE.md` is a project guide that [Claude Code](https://docs.anthropic.com/en/docs/claude-code) reads when it opens a repository. These guides are written with a **support and troubleshooting** focus — they help Claude navigate the codebase to answer questions like:

- Where is a config setting validated?
- Where does an error message originate?
- How is a feature license-gated?
- What API endpoint handles a given request?

The target repos are **read-only reference copies** — they exist for code search, not active development.

## Directory Layout

```
dotfiles/
├── CLAUDE.md                           # Top-level guide (for this repo itself)
├── README.md                           # This file
├── mattermost/CLAUDE.md                # Mattermost server monorepo (Go/React)
├── mattermost-enterprise/CLAUDE.md     # Enterprise features (LDAP, SAML, clustering, etc.)
├── Desktop/CLAUDE.md                   # Electron desktop app
├── Mattemrost-Plugin-Calls/CLAUDE.md   # WebRTC voice/video/screensharing plugin
├── Mattermost-Mobile/CLAUDE.md         # React Native iOS/Android app
├── Mattermost-Plugin-Agents/CLAUDE.md  # AI/LLM integration plugin
├── Mattermost-Plugin-Boards/CLAUDE.md  # Kanban boards plugin (Focalboard)
└── Mattermost-Plugin-Playbooks/CLAUDE.md # Incident management plugin
```

Each subdirectory maps to a repo in `~/Repositories/Claude-Repos/`:

| Dotfiles Subdir | Target Repo | Description |
|---|---|---|
| `mattermost/` | `Claude-Repos/Mattermost/` | Go/React monorepo — server, webapp, mmctl, API |
| `mattermost-enterprise/` | `Claude-Repos/Enterprise/` | Enterprise features — LDAP, SAML, clustering, compliance |
| `Desktop/` | `Claude-Repos/Desktop/` | Electron desktop app — multi-server, notifications, certificates |
| `Mattemrost-Plugin-Calls/` | `Claude-Repos/Mattemrost-Plugin-Calls/` | WebRTC plugin — voice/video calls, recording, transcription |
| `Mattermost-Mobile/` | `Claude-Repos/Mattermost-Mobile/` | React Native app — WatermelonDB, dual database, products |
| `Mattermost-Plugin-Agents/` | `Claude-Repos/Mattermost-Plugin-Agents/` | AI/LLM plugin — multi-provider, MCP, embeddings |
| `Mattermost-Plugin-Boards/` | `Claude-Repos/Mattermost-Plugin-Boards/` | Kanban boards — boards, blocks, cards, templates |
| `Mattermost-Plugin-Playbooks/` | `Claude-Repos/Mattermost-Plugin-Playbooks/` | Incident management — playbooks, runs, checklists |

## How symlinks work

Each `CLAUDE.md` is symlinked from this repo into the target repo root so that Claude Code picks it up automatically:

```bash
ln -sf ~/Repositories/Claude-Stuff/dotfiles/{SubDir}/CLAUDE.md ~/Repositories/Claude-Repos/{RepoDir}/CLAUDE.md
```

To keep the symlinks invisible to git in the target repos, two strategies are used depending on whether the upstream repo already tracks a `CLAUDE.md`:

### Repos where `CLAUDE.md` is NOT tracked upstream

Mattermost, Enterprise, Calls, Boards, Playbooks — the symlink is an untracked file, so it just needs to be excluded:

```
# In each repo's .git/info/exclude:
CLAUDE.md
```

### Repos where `CLAUDE.md` IS tracked upstream

Desktop, Mobile, Agents — these repos have their own `CLAUDE.md` committed. Our symlink replaces it, and `skip-worktree` tells git to ignore the change and preserve the symlink during `git pull`:

```bash
git update-index --skip-worktree CLAUDE.md
```

## Setting up a new repo

1. Create a subdirectory here with a `CLAUDE.md` following the existing pattern
2. Clone the repo into `~/Repositories/Claude-Repos/`
3. Create the symlink:
   ```bash
   ln -sf ~/Repositories/Claude-Stuff/dotfiles/{SubDir}/CLAUDE.md ~/Repositories/Claude-Repos/{RepoDir}/CLAUDE.md
   ```
4. If `CLAUDE.md` is **not** tracked upstream, add it to `.git/info/exclude`:
   ```bash
   echo "CLAUDE.md" >> ~/Repositories/Claude-Repos/{RepoDir}/.git/info/exclude
   ```
5. If `CLAUDE.md` **is** tracked upstream, set skip-worktree:
   ```bash
   cd ~/Repositories/Claude-Repos/{RepoDir} && git update-index --skip-worktree CLAUDE.md
   ```

## Editing guides

Edit the files in **this repo** — the symlinks ensure changes are immediately reflected in the target repos. Do not edit `CLAUDE.md` directly in the target repos.
