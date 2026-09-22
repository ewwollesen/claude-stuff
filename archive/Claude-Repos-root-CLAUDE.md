# Claude-Repos

This directory holds local clones of Mattermost source repositories, used as
**read-only references** for answering support questions, tracing error
messages, understanding code paths, and looking up config behavior.

## What lives here

| Repo | Contents |
|------|----------|
| `Mattermost` | Server, webapp, mmctl, API |
| `Enterprise` | Enterprise features (LDAP, SAML, clustering, etc.) |
| `Mattermost-Plugin-Calls` | Calls plugin |
| `Mattermost-RTCD` | RTCD daemon — standalone Go SFU offload service for Calls |
| `Mattermost-Plugin-Agents` | AI/LLM plugin |
| `Mattermost-Plugin-Boards` | Boards plugin |
| `Mattermost-Plugin-Confluence` | Confluence integration plugin |
| `Mattermost-Plugin-Github` | GitHub integration plugin |
| `Mattermost-Plugin-Gitlab` | GitLab integration plugin |
| `Mattermost-Plugin-LegalHold` | Legal Hold plugin (Enterprise-only) |
| `Mattermost-Plugin-Playbooks` | Playbooks plugin |
| `Mattermost-Mobile` | React Native mobile app |
| `Desktop` | Electron desktop app |
| `Mattermost-Docs` | Sphinx docs site (docs.mattermost.com) |
| `Mattermost-Operator` | Kubernetes operator (CRDs, controllers) |

Each subdirectory has its own `CLAUDE.md` with repo-specific structure and
conventions — check it before deep-diving.

## Conventions

- **Read-only.** Do not commit, push, or modify files here unless the user
  explicitly asks. These are reference clones, not working copies.
- **Refresh before searching.** Run `cd <repo> && git fetch origin && git pull`
  so answers reflect current code, not a stale snapshot.
- **Prefer these over general knowledge.** When a question touches Mattermost
  behavior, errors, config keys, API endpoints, or plugin internals, search
  the relevant repo here first rather than answering from memory.
