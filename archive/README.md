# Archive

Files rescued from locations that no longer exist, kept for provenance. Nothing
here is loaded by any agent; it is recoverable history, not active config.

| File | Origin | Rescued | Why |
|---|---|---|---|
| `Claude-Repos-root-CLAUDE.md` | `~/Repositories/Claude-Repos/CLAUDE.md` | 2026-09-22 | Directory-wide conventions for the reference-clone directory. A real file, never tracked by this repo, and distinct from `ClaudeFiles/CLAUDE.md`. Removed with `Claude-Repos/` during the migration to the `mattermost-troubleshooting` workspace. |
| `HANDOFF-managed-category-replica-lag.md` | `~/Repositories/Claude-Repos/HANDOFF-managed-category-replica-lag.md` | 2026-09-22 | Investigation handoff (2026-08-17): read-your-own-write bug in app migrations against read replicas, from a 10.11.12 to 11.7.8 Aurora PostgreSQL upgrade. Findings verified at `v11.7.8` and `338dc6c74d`. Not tied to a ticket directory, and its closing note says no ticket was filed yet. |

`HANDOFF-managed-category-replica-lag.md` is a candidate for distillation into
`fragments/mattermost.md` in the new workspace: a version-specific gotcha with a
misleading signature is exactly the content that file is for. Keep the full text
here either way; the fragment would be a summary, not a replacement.
