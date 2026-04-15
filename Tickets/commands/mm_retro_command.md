Run a post-resolution retrospective on the ticket in the current directory.

The ticket directory is the current working directory. All artifacts (logs,
config, analysis.md, rca.md, eir.md, etc.) are expected to be here or in
subdirectories.

## Prerequisites

This command runs **after** the investigation is complete — the root cause must
be confirmed or the ticket otherwise resolved. If `analysis.md` is missing or
the "Current hypothesis" section is empty/says "unknown", stop and tell the
user the ticket doesn't appear to be resolved yet.

## Your task

Read `analysis.md` and all other artifacts in the ticket directory. If `rca.md`
or `eir.md` exist, read those too. Then produce a retrospective that covers
three areas:

1. **Investigation retrospective** — how did the investigation go?
2. **Follow-up actions** — what needs to happen now?
3. **KB ingest** — does this pattern belong in the knowledge base?

Write the output to `retro.md` in the current directory. After writing, print
the full contents to the screen.

---

## Part 1 — Investigation retrospective

Review the investigation as captured in `analysis.md` and assess honestly:

### Blind triage effectiveness
- Did the blind triage (Step 2) surface the error(s) that turned out to be the
  root cause?
- If not — what was it about the error that made it hard to catch? Was it low
  volume, an unusual format, or buried in noise?
- Were there error patterns that dominated the triage but turned out to be
  irrelevant? Would a known-noise list have helped?

### Hypothesis path
- How many hypotheses were formed before landing on the root cause?
- Were any dead ends avoidable with better tooling or knowledge?
- Did anchoring on the customer's reported symptom delay finding the real cause?
- Was the root cause something the customer reported, something found in triage,
  or something discovered later through a different path?

### Artifact gaps
- What artifacts were available at the start vs. what was ultimately needed?
- Were there artifacts that had to be requested from the customer that could
  have been asked for earlier?
- Were any artifacts present but not useful? (e.g., logs from the wrong time
  window, redacted config)

### Time and effort
- How many sessions did the investigation span? (Count `## Session` headers)
- Was the resolution path efficient, or were there significant detours?
- What single change would have shortened this investigation the most?

### Tooling observations
- Did the `mm_ticket_command` workflow help or hinder for this type of issue?
- Are there specific error signatures or patterns from this ticket that the
  blind triage step should be taught to recognize?
- Are there improvements to the `analysis.md` template that would have helped?
- Should the orient step (Step 1) check for additional artifact types?

Be specific and evidence-based. Don't say "the investigation went well" — say
*what* went well and *why*. Cite sections of the analysis.

---

## Part 2 — Follow-up actions

Identify concrete follow-ups that should happen as a result of this ticket.
For each one, state clearly what the action is and why.

### Jira tickets
- Was a product bug confirmed? If so, does an existing Jira ticket cover it,
  or does one need to be filed?
- Was a misleading error message identified? That's worth a ticket.
- Was there a feature gap (missing config option, missing API, etc.) that
  contributed to the issue?

Use the Jira MCP connector to check for existing tickets before recommending
a new one. Use the GitHub MCP connector to check for existing issues or PRs.

### Documentation
- Was there a config value that behaves differently than documented or expected?
- Was there a deployment pattern (HA, air-gapped, etc.) that lacks guidance?
- Was an error message misleading enough that it should be documented or changed?

### Process improvements
- Should this class of issue be caught by a health check or monitoring?
- Is there a proactive communication we should send to customers on affected
  versions?
- Did the support workflow itself have a gap (e.g., missing escalation path,
  slow artifact collection)?

### Other
- Flag anything else that came up during the investigation that doesn't fit
  the above categories but still needs attention.

If there are no follow-ups in a category, omit that category entirely. Don't
pad with "none identified" sections.

---

## Part 3 — KB ingest

Decide whether this ticket's root cause belongs in the knowledge base at
`~/Downloads/Tickets/kb/`.

### Decision criteria

**Create or update a KB entry when:**
- A bug in Mattermost code was found (will recur until fixed)
- A common misconfiguration was identified (multiple customers will hit it)
- A misleading error message was diagnosed (the diagnostic path is reusable)

**Skip KB ingest when:**
- The issue was purely customer-specific with no reusable diagnostic path
- The root cause is obvious from the error message alone (no diagnostic
  value in documenting the path)

### If creating or updating

1. Read `~/Downloads/Tickets/kb/index.md`
2. Check whether the root cause pattern already has an entry:
   - **Existing entry:** Update it — add the ticket number, any new error
     signature variants, new ruled-out theories, updated version info or
     deployment context. Update the corresponding article if one exists.
   - **New reusable pattern:** Add a new index entry. Create an article in
     `kb/articles/` if the pattern needs depth (ruled-out theories, multi-step
     root cause, code references). Skip the article if the one-line index
     summary is sufficient.
3. Update the index header metadata (entry count, last-updated date).

### index.md entry format

Each entry is one record:

```
`error signature(s)` | root cause summary | article link or "no article" | ticket number(s)
```

### Article template

```markdown
# <Descriptive Title>
<!-- Tickets: XXXXX, YYYYY -->

## Error Signatures
- `exact log string`

## Root Cause
What's happening + code path.

## Fix
Customer action + product fix ref.

## Ruled Out
What was eliminated and why.

## Deployment Context
HA/single/air-gapped. Versions. Triggers.

## Code References
- `server/path/file.go:NN` — function
```

### If skipping

State why the pattern is not reusable. This goes in `retro.md` so the decision
is recorded.

**Do NOT delete or modify the ticket's `analysis.md` — the KB distills it,
does not replace it.**

---

## retro.md format

```markdown
# Ticket <number> — Retrospective

**Date:** <today's date>
**Resolution:** <one-line summary of root cause and fix>

---

## Investigation retrospective

### Blind triage
<!-- Did it catch the root cause? What was missed and why? -->

### Hypothesis path
<!-- How many hypotheses? Dead ends? Was anchoring a factor? -->

### Artifact gaps
<!-- What was missing? What should have been requested sooner? -->

### Efficiency
<!-- Sessions, detours, what would have shortened the investigation -->

### Tooling observations
<!-- What should change in mm_ticket_command or the workflow? -->

---

## Follow-up actions

<!-- Only include categories that have actual items -->

### Jira tickets
- [ ] <action> — <reason>

### Documentation
- [ ] <action> — <reason>

### Process improvements
- [ ] <action> — <reason>

---

## KB ingest

**Decision:** <added / updated / skipped>
<!-- If added/updated: what was changed. If skipped: why. -->
```

---

## Tone and style

- Be honest and specific. This is an internal process improvement tool, not
  a customer-facing document.
- Don't soften findings. If the investigation took a wrong turn, say so and
  say why.
- Focus on systemic improvements, not one-off observations. "We should add
  X pattern to the triage step" is more useful than "this was a tricky ticket."
- Every observation should point toward a concrete action or a decision that
  it's not worth acting on.
