Begin a Mattermost support ticket investigation for the ticket in the current directory.

The ticket directory is the current working directory. All artifacts (logs,
config, analysis.md, etc.) are expected to be here or in subdirectories.

**First**, rename the session using `/rename` to include the ticket number and
customer name (derived from the directory name or artifacts). For example:
`/rename Ticket 12345 — Acme Corp`

**Keep analysis.md sections current, not just appended to.** The CLAUDE.md
workflow says to append session logs rather than overwrite the file — that
applies to *session headers* and *chronological findings*. But the following
sections must always reflect the **latest** understanding, not accumulate stale
entries:

- **Current hypothesis** — Replace with the active theory. If a prior
  hypothesis was superseded, move it to "Ruled out" with a brief reason.
- **Ruled out** — Add superseded hypotheses here as you go. Never delete
  ruled-out entries.
- **Correlation** — Update to reflect the current mapping between symptoms
  and findings. Remove correlations that turned out to be wrong.
- **Open questions** — Remove questions that have been answered. Add new ones.
- **Next steps** — Replace with whatever is actually next. Don't leave stale
  action items.

Every time you update `analysis.md`, review these sections and revise them
so that someone reading the file cold gets an accurate picture of where the
investigation stands *right now*.

**Anchor errors to the incident window.** Volume is not relevance. Before
forming any hypothesis from triage:

- Record the **first-seen timestamp** for each error class you flagged, not
  just the count. An error appearing 200 times since last Tuesday is a
  different signal than the same error first appearing at the incident time.
- Compare those timestamps against when the customer says the issue started.
  Errors that predate the reported start by hours or days are likely
  background noise unless you can draw a specific causal link (slow resource
  leak, accumulating replication lag, certificate nearing expiry, etc.).
  State the causal link explicitly — don't assume one.
- Prioritize errors that **first appear** in or near the incident window over
  loud, long-running errors that were already firing before the issue began.
- If the timeline doesn't line up — e.g., the customer reports a 2pm outage
  but the only errors in the log are at 10am — say so explicitly in
  `analysis.md`. Either the log doesn't cover the actual incident window
  (request fresh logs scoped to the right time range) or the loud errors are
  unrelated (don't anchor on them). Both are findings, not gaps to paper
  over.

Capture this in the `Timeline` section of `analysis.md` with concrete
timestamps, and reference it from the `Correlation` section when explaining
how triage findings map (or don't) to the reported symptom.

Then follow the workflow defined in CLAUDE.md exactly:

0. **Before any analysis**, list every file in the current directory (recursively). Attempt to read or access each artifact. If ANY file cannot be read or parsed (PDFs, images, compressed archives, unknown formats, etc.), STOP IMMEDIATELY and report which files are inaccessible and why. Do not proceed with analysis until all artifacts can be accessed — incomplete input leads to wasted work.
1. Orient yourself on the ticket directory
2. Check for an existing `analysis.md` and summarize it if present
3. Perform blind triage on the log(s) before asking anything about the reported symptom
4. Present your triage findings, then ask what the customer reported
