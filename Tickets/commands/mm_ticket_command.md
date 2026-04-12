Begin a Mattermost support ticket investigation for the ticket in the current directory.

The ticket directory is the current working directory. All artifacts (logs,
config, analysis.md, etc.) are expected to be here or in subdirectories.

**Do not add comments to inline code.** When you write bash one-liners,
inline Python scripts, or any throwaway code in tool calls, omit all comments.
These snippets are ephemeral and only executed by you — comments add noise and
trigger unnecessary permission prompts.

**First**, rename the session using `/rename` to include the ticket number and
customer name (derived from the directory name or artifacts). For example:
`/rename Ticket 12345 — Acme Corp`

Then follow the workflow defined in CLAUDE.md exactly:

0. **Before any analysis**, list every file in the current directory (recursively). Attempt to read or access each artifact. If ANY file cannot be read or parsed (PDFs, images, compressed archives, unknown formats, etc.), STOP IMMEDIATELY and report which files are inaccessible and why. Do not proceed with analysis until all artifacts can be accessed — incomplete input leads to wasted work.
1. Orient yourself on the ticket directory
2. Check for an existing `analysis.md` and summarize it if present
3. Perform blind triage on the log(s) before asking anything about the reported symptom
4. Present your triage findings, then ask what the customer reported
