Generate a customer-facing Root Cause Analysis (RCA) report for the ticket in the current directory.

The ticket directory is the current working directory. All artifacts (logs,
config, analysis.md, etc.) are expected to be here or in subdirectories.

## Your task

1. Read `analysis.md` in the current directory as your primary source of
   truth. Do not speculate or add information that is not grounded in the
   analysis or artifacts from this ticket directory.

2. If `analysis.md` is missing or incomplete, say so and ask whether to run
   the ticket investigation skill first before proceeding.

3. If an `eir.md` exists in the current directory, use it as an additional
   source — but remember the EIR is internal and must be translated
   appropriately for a customer audience.

4. Write the RCA in Markdown to `rca.md` in the current directory.

5. After writing the file, print the full contents of `rca.md` to the screen
   so it is ready to copy and paste into a Mattermost channel or ticket
   response.

---

## RCA format

Use exactly this structure:

```markdown
# Root Cause Analysis — <one line description>

**Ticket:** <number>
**Date:** <date of incident>
**Severity:** <P1 / P2 / P3>
**Status:** <Resolved / Monitoring>
**Mattermost version:** <version>

---

## Summary

Two to three sentences. What happened, what was the impact, and what was
done to resolve it. Clear and direct — the customer should understand this
without reading further if they choose not to.

---

## Timeline

| Time (UTC) | Event |
|------------|-------|
| YYYY-MM-DD HH:MM | <event> |

Include: when the issue began, when it was reported, when it was resolved.
Use confirmed timestamps only.

---

## Root cause

A clear, factual explanation of what caused the issue. Explain it in terms
of system behavior rather than fault or blame. Technical detail is
appropriate but should be accessible to a technical administrator, not
necessarily a developer.

---

## Impact

- **Scope:** <what was affected>
- **Duration:** <start to resolution>
- **Data loss:** <yes / no / none detected>

---

## Resolution

What was done to resolve the issue. Focus on the steps taken and the
outcome.

---

## Prevention

What is being done or recommended to prevent recurrence. Include any
configuration recommendations, upgrade guidance, or monitoring suggestions
that apply to the customer's environment.

---

*If you have questions about this report please reply to your support ticket.*
```

---

## Tone and style guidelines

**Write with confidence and professionalism.** This report represents
Mattermost to the customer. It should feel like it came from a team that
knows its product deeply and handled the situation competently.

**Specific tone guidance:**
- Factual and measured — describe what happened without dramatizing it
- Resolution-focused — emphasize what was done and what was learned, not
  what went wrong
- Avoid language that implies negligence, carelessness, or surprise —
  prefer "a condition was identified" over "we missed" or "we failed to"
- Avoid overly defensive or apologetic language — one brief acknowledgment
  of impact is sufficient, do not repeat it
- Do not editorialize — stick to facts, avoid phrases like "unfortunately"
  or "regrettably" repeated throughout
- Use passive or system-focused framing for the root cause where appropriate
  — "a misconfiguration in the TLS settings caused..." rather than
  "Mattermost incorrectly configured..."
- Active and confident voice for the resolution — "The support team
  identified and resolved..." not "it was eventually determined..."

**What to leave out:**
- Internal Jira or GitHub references
- Internal team names or individual names
- Any finding that is embarrassing without being relevant to the customer
- Speculation about causes that were ruled out
- Details about other customers or tickets

**What to never do:**
- Do not make commitments on behalf of Mattermost engineering (e.g. "this
  will be fixed in the next release") unless that is confirmed in the
  analysis
- Do not assign blame to the customer's environment or configuration without
  being tactful — "the observed behavior was consistent with a
  configuration where X is set to Y" rather than "the customer
  misconfigured..."
- Do not include customer-identifying information
