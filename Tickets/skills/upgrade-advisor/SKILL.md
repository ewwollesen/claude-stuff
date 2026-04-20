---
name: upgrade-advisor
description: Analyze fixes between a customer's current Mattermost version and the latest patch to generate a human-readable upgrade recommendation.
argument-hint: "[version]"
allowed-tools: Read, Grep, Glob, Bash, Agent, WebFetch, mcp__github__list_tags, mcp__github__get_tag, mcp__github__list_commits, mcp__github__get_commit, mcp__github__get_file_contents, mcp__claude_ai_Atlassian__getJiraIssue, mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql, mcp__claude_ai_Atlassian__atlassianUserInfo
---

# Mattermost Upgrade Advisor

You are generating an upgrade recommendation report for a Mattermost customer. Your job is to find meaningful bug fixes and security patches between their current version and the latest available patch in the same minor series, then explain each one in plain language.

## Input

`$ARGUMENTS`

This command runs from inside the ticket directory (`pwd` is the ticket dir). It supports two modes:

### Mode 1 — Artifacts in current directory (no argument)
If `$ARGUMENTS` is empty, look for artifacts in `pwd` only. Do not search parent directories or `~/Downloads/Tickets/`. Follow Step 1A.

### Mode 2 — Version only (version passed as argument)
If `$ARGUMENTS` contains a version number (e.g. `10.11.10`, or freeform text like "no support packet, 10.11.10"), extract the version and skip directly to Step 2. You won't have a config, plugin list, or instance URL — that's fine. Generate the report without config-specific relevance tagging and without plugin filtering. Note in the report header that no support packet was available.

**How to decide:** If `$ARGUMENTS` is empty or whitespace-only → Mode 1. If `$ARGUMENTS` contains a semver-looking string (`X.Y.Z`) → Mode 2. Do NOT treat `$ARGUMENTS` as a ticket number or path — that workflow is deprecated.

## Step 1A — Discover the customer's version and config (Mode 1 only)

All lookups in this step run against `pwd`. Do not touch any other directory.

### Find the support packet

Look in `pwd` for a support packet directory matching `mm_support_packet_*`. If multiple are present, use the most recent one:

```bash
ls -d ./mm_support_packet_* 2>/dev/null | sort | tail -1
```

### Get the version

If a support packet exists, read `metadata.yaml` at its root and extract `server_version`. Otherwise, look for a `config.json` directly in `pwd` and check its header/`SqlSettings`/version field — if the config doesn't expose a version, ask the user for it.

### Find the config

Preference order (first match wins):

1. **Support packet node config** — inside the support packet, node subdirectories (e.g. `ip-172-16-*`) each contain a `sanitized_config.json`. Use the first one — in HA the config is typically identical across nodes.
   ```bash
   find ./mm_support_packet_*/ -name "sanitized_config.json" 2>/dev/null | head -1
   ```
2. **Bare `config.json` in `pwd`** — if there's no support packet, fall back to `./config.json`.
3. **Neither** — stop and tell the user no config was found. Suggest they either re-run with a version argument (`/upgrade-advisor 10.11.10`) or drop a `config.json` or support packet in the current directory.

### Get installed plugins

If a support packet exists, read `plugins.json` at its root. It has `enabled` and `disabled` arrays; each entry has `id`, `name`, and `version`:

```bash
cat ./mm_support_packet_*/plugins.json 2>/dev/null | head -1
```

Build a map of installed plugin IDs to their current versions. Use this to:
1. **Skip plugin update commits for plugins that are not installed.** If a commit updates the Calls plugin but Calls is not in the customer's `plugins.json` (neither enabled nor disabled), omit it from the report entirely.
2. **Show the customer's current plugin version** when reporting a plugin update.

If there is no support packet (only a bare `config.json`), you won't have a plugin list. In that case, include all bundled plugin updates in the report without a "currently v…" fragment, and note in the summary that plugin filtering was not possible.

## Step 1B — Pre-flight: detect available data sources

This skill is designed to run for both engineers (who typically have the Mattermost and Enterprise repos cloned locally) and TAMs (who typically don't, and may have varying levels of MCP access). Before any real analysis work, do a single pre-flight pass to detect what's available. Report the results to the user as a short capability summary before proceeding. If a critical capability is missing, stop and ask the user how to proceed rather than silently degrading.

Run all four checks. They are independent — do them in parallel when possible.

### Check 1 — Community repo: local git

```bash
test -d ~/Repositories/Claude-Repos/Mattermost/.git && echo "mm-local: yes" || echo "mm-local: no"
```

If present, `git fetch --tags` in the same call so data is current:
```bash
test -d ~/Repositories/Claude-Repos/Mattermost/.git && git -C ~/Repositories/Claude-Repos/Mattermost fetch --tags --quiet 2>&1
```

### Check 2 — Community repo: GitHub MCP (only if local missing or as confirmation)

```
mcp__github__list_tags owner=mattermost repo=mattermost perPage=1
```

A successful response with at least one tag means MCP works for the community repo. An auth error means the user's GitHub MCP is not connected.

### Check 3 — Enterprise repo: local git, then MCP fallback

```bash
test -d ~/Repositories/Claude-Repos/Enterprise/.git && echo "ee-local: yes" || echo "ee-local: no"
test -d ~/Repositories/Claude-Repos/Enterprise/.git && git -C ~/Repositories/Claude-Repos/Enterprise fetch --tags --quiet 2>&1
```

If local is absent, test MCP access to the private enterprise repo:
```
mcp__github__list_tags owner=mattermost repo=enterprise perPage=1
```

A 404 or permission error here means the user's GitHub account lacks access to `mattermost/enterprise`. This is common for TAMs who haven't requested repo access. Note it and continue — the skill will proceed with community-only coverage and call this out in the report summary.

### Check 4 — Jira (Atlassian) MCP

```
mcp__claude_ai_Atlassian__atlassianUserInfo
```

A successful response confirms the Atlassian MCP is connected. A failure means Jira priority enrichment won't work — the skill will still produce a report, just without Jira priority labels on commits.

### Pre-flight summary

After the four checks, print a short capability block like this and await no confirmation — just proceed:

```
Pre-flight:
  Community repo: local ✓   (or: MCP ✓, or: unavailable ✗)
  Enterprise repo: local ✓  (or: MCP ✓, or: unavailable — community-only coverage)
  Jira MCP: connected ✓     (or: unavailable — priority labels will be omitted)
```

### Stop conditions

Stop and ask the user if:
- **Both community-repo sources fail.** Without community commits there is nothing to analyze. Tell them to either clone `mattermost/mattermost` to `~/Repositories/Claude-Repos/Mattermost` or connect their GitHub MCP.

Proceed with a noted degradation if:
- Enterprise unavailable → note in report summary that Enterprise-only fixes were not analyzed.
- Jira MCP unavailable → omit `Jira priority: X` fragments entirely.
- WebFetch to `mattermost.com/security-updates/` fails → fall back to the commit-keyword heuristic described in Step 5b.

Record your per-repo source decision (local vs MCP). Every subsequent step that touches git (Step 2, Step 4, Step 5) must use the corresponding source for that repo.

## Step 2 — Determine the target version

### If using local git

```bash
git -C ~/Repositories/Claude-Repos/Mattermost tag --list 'v<major>.<minor>.*' | grep -v rc | sort -V | tail -1
```

### If using GitHub MCP

```
mcp__github__list_tags owner=mattermost repo=mattermost
```

Filter the returned tags to names matching `v<major>.<minor>.*` that are not release candidates, sort by semver, and take the highest.

### Either way

If the customer is already on the latest patch, tell the user and stop.

### Get release dates

**Local:**
```bash
git -C ~/Repositories/Claude-Repos/Mattermost log -1 --format="%ai" v<current_version>
git -C ~/Repositories/Claude-Repos/Mattermost log -1 --format="%ai" v<target_version>
```

**GitHub MCP:** call `mcp__github__get_tag` for each version to get the tagged commit, then `mcp__github__get_commit` on that SHA to read its `commit.committer.date`. Alternatively, `list_tags` returns commit SHAs — skip directly to `get_commit`.

Format as a human-readable date (e.g. "November 21, 2025") for the report header.

## Step 3 — Read and understand the customer's config

Read the config file. Pay attention to which features are enabled or disabled. Key things to note:

- `ServiceSettings`: persistent notifications, collapsed threads, post priority, OAuth, webhooks, scheduled posts, custom groups
- `SamlSettings.Enable` / `LdapSettings.Enable` — authentication method
- `ClusterSettings.Enable` — HA deployment
- `ElasticsearchSettings.EnableSearching` — search backend
- `SqlSettings.DriverName` — postgres vs mysql
- `FileSettings.DriverName` — local vs amazons3
- `PluginSettings.Enable` and `PluginSettings.PluginStates` — active plugins
- `GuestAccountsSettings.Enable`
- `ComplianceSettings.Enable`, `DataRetentionSettings`, `MessageExportSettings`
- `MetricsSettings.Enable`
- `ExperimentalSettings.EnableSharedChannels`
- `EmailSettings.SendPushNotifications` — mobile push

Understand their setup holistically. Don't just check booleans — a customer on Postgres with S3 storage, SAML via Okta, and clustering has a very different profile than one on MySQL with local storage.

## Step 4 — Get the commits, grouped by patch version

Check **both** repos for commits:
- Community: `mattermost/mattermost` (local at `~/Repositories/Claude-Repos/Mattermost` if available)
- Enterprise: `mattermost/enterprise` (local at `~/Repositories/Claude-Repos/Enterprise` if available) — LDAP, SAML, compliance, OpenID, and other licensed features

Both repos use the same version tags. Use the source you selected in Step 1B for each repo independently.

Determine the patch versions in the range. For example, if the customer is on 10.11.8 and the target is 10.11.13, the intermediate versions are 10.11.9, 10.11.10, 10.11.11, 10.11.12, 10.11.13. Use the tag list you already fetched in Step 2, filtered to versions *after* the customer's current version and up to (including) the target.

### Get commits between two tags

For each consecutive pair of versions (e.g. v10.11.8 → v10.11.9), get the list of commits introduced in the later version from each repo:

**Local git:**
```bash
git -C ~/Repositories/Claude-Repos/Mattermost log v10.11.8..v10.11.9 --format="COMMIT:%H%nSUBJECT:%s%nBODY:%b%n---END---"
git -C ~/Repositories/Claude-Repos/Enterprise log v10.11.8..v10.11.9 --format="COMMIT:%H%nSUBJECT:%s%nBODY:%b%n---END---"
```

**GitHub MCP:** the MCP doesn't expose a direct "compare two tags" call, so use either of these patterns:

1. **Preferred — compare endpoint via `get_file_contents` pattern is not applicable here; instead use `list_commits`** with `sha=v<later_tag>` to walk commits backwards in time, stopping when you hit the SHA of `v<earlier_tag>`. Request a reasonable `perPage` (e.g. 100) and paginate until you find the boundary SHA.

   ```
   mcp__github__list_commits owner=mattermost repo=mattermost sha=v10.11.9 perPage=100
   ```

2. **Or** fall back to scraping the compare page via WebFetch:
   ```
   WebFetch: https://github.com/mattermost/mattermost/compare/v10.11.8...v10.11.9
   Prompt: "Return the list of commit SHAs and subjects between these two tags."
   ```
   Use this only if `list_commits` pagination is impractical for the range.

Track which patch version and which repo each commit belongs to.

### Inspecting a commit's diff (for unclear subjects)

**Local git:**
```bash
git -C ~/Repositories/Claude-Repos/Mattermost diff <hash>^..<hash> --stat
git -C ~/Repositories/Claude-Repos/Mattermost show <hash> -- <specific-file>
```

**GitHub MCP:**
```
mcp__github__get_commit owner=mattermost repo=mattermost sha=<hash>
```
This returns the commit with a `files` array (path, status, patch). If you need the full file at that commit, use `mcp__github__get_file_contents` with `ref=<hash>`.

For enterprise commits, substitute `repo=enterprise` in the MCP calls or use the local Enterprise path.

## Step 5 — Analyze each commit

### What to include
- **Bug fixes** — changes that fix broken behavior users would actually experience (UI bugs, incorrect data, crashes, broken workflows)
- **Behavioral changes** — anything that changes how a feature works in a way a user or admin would notice
- **Plugin updates** — bundled plugin version bumps (for plugins the customer has installed only)

### What to exclude
- Test-only changes (adding/modifying test cases with no production code changes)
- CI/CD and workflow file changes
- Linter/formatting fixes
- Build toolchain updates (Go version bumps, base image changes)
- Internal dependency version bumps with no user-facing impact
- Version bump commits ("Update latest patch version to X.Y.Z")
- Automated merge commits
- Code comments or documentation-only changes
- **Individual security-fix commits.** Per Mattermost's responsible disclosure policy, security fixes are NOT described individually. They are summarized aggregately in the Security Fixes table (see Step 5b). If a commit looks like a security patch (sanitization, CSRF, XSS, SSRF, auth bypass, MMSA/CVE reference, constant-time comparison), exclude it from the per-commit analysis — it will already be counted in the table.
- **Plugin updates for plugins the customer does not have installed.** If a commit updates a prepackaged plugin (e.g. Calls, Boards, Playbooks) and that plugin does not appear in the customer's `plugins.json`, omit it entirely. Do not list it with a note that it's not installed — just skip it.

### For each included commit

1. Read the commit subject and body
2. **If the subject is unclear** (e.g. "MM 65084 server-side", "Automated cherry pick of #34247"), inspect the diff using the Step 4 "Inspecting a commit's diff" approach — local git if you have it, `mcp__github__get_commit` otherwise.
3. Write a **1-2 sentence plain-English summary** of what the fix does and why it matters
4. Determine if it's relevant to the customer's enabled features — if so, note the relevance **inline** in the description (e.g. "Your instance has persistent notifications enabled (`AllowPersistentNotifications: true`), making this directly relevant.")
5. **Categorize** the fix into one of two buckets:
   - **Upgrade urgently** — server crashes/panics, data loss, silent config loss, significant functional breakage, anything that could cause an outage or degraded service for a meaningful population of users
   - **Quality of life** — functional improvements, minor bug fixes, UI/UX polish, performance tweaks, fixes for narrow edge cases

### Jira priority lookup (optional but preferred)

If a commit references a Jira ID (e.g. `MM-65575`), fetch the issue via `mcp__claude_ai_Atlassian__getJiraIssue` and include its priority in the report entry, formatted as `(MM-65575, Jira priority: High)`. If no Jira ID is present, the lookup fails, or the TAM running the skill doesn't have Jira access, just omit the Jira fragment — do not invent a priority.

## Step 5b — Build the security fixes summary

Do NOT list individual security commits. Instead, for each patch release in the range (10.11.9, 10.11.10, etc.), determine:

1. **Approximate count** of security fixes in that patch
2. **Severity range** (Low to Medium, Low to High, etc.)

Preferred source: fetch `https://mattermost.com/security-updates/` with WebFetch and count MMSA entries per patch version, noting the highest severity per version.

```
WebFetch: https://mattermost.com/security-updates/
Prompt: "List all MMSA entries affecting Mattermost <major>.<minor>.<N> for each N between <current+1> and <target>. For each patch version, return the count of MMSA entries and the highest severity across them."
```

If the fetch fails or returns nothing, fall back to counting commits in each patch range whose subject or body references MMSA/CVE/security keywords, and mark the counts as "approximate" without severity. Never fabricate severities.

## Rules — CRITICAL

### Honesty above all
- **Do NOT guess or assume what a commit does.** If the commit subject is vague and the diff doesn't make it clear, say: *"Unable to determine the impact of this change from the available information."*
- **Do NOT invent details.** If you can't tell whether a fix is relevant to the customer's config, say so. It is always better to say "I'm not sure if this affects your setup" than to fabricate a connection.
- **Do NOT embellish.** Describe what the fix does factually. Don't add urgency or severity that isn't supported by the code change.
- **Do NOT make up Jira ticket descriptions.** You only know what is in the commit message and the diff. If the Jira ID is `MM-65084` and the commit subject just says "server-side", do not invent what MM-65084 is about.

### Config relevance
When you have the customer's config, note which fixes are relevant to features they have **enabled**. But be honest — if you're not sure whether a fix relates to a config setting, don't force the connection. Just list it as a general fix.

## Step 6 — Generate the report

Output a markdown report following this **exact** template. Do not rename sections, reorder them, change heading levels, or restructure the bullet format. This format is the contract — deviations are regressions.

```
# Upgrade Recommendation: <current> → <target>

**Instance:** [<SiteURL>](<SiteURL>) **Current Version:** <current> (released <date>) **Recommended Version:** <target> (released <date>)

## Security Fixes

Patch releases <current+1> through <target> collectively include **~<N> security fixes** ranging from **<lowest severity> to <highest severity> severity**. Per Mattermost's [responsible disclosure policy](https://mattermost.com/security-updates/), specific vulnerability details are not included here.

| Patch    | Severity Range | Approx. Count |
| -------- | -------------- | ------------- |
| <v1>     | <range>        | <count>       |
| <v2>     | <range>        | <count>       |
| ...      | ...            | ...           |

For full details on disclosed vulnerabilities (including CVE and MMSA identifiers), see: [https://mattermost.com/security-updates/](https://mattermost.com/security-updates/)

## Upgrade urgently — could cause outage or degradation

These fixes address server crashes or significant functional breakage.

- **<Short title>** (<MM-XXXXX, Jira priority: X> if available) — _Introduced in <version>_ <1-2 sentence plain-English description.> <Inline config relevance note if applicable.> PR: [#<number>](https://github.com/mattermost/mattermost/pull/<number>)

## Quality of life — could be annoying for a subset of users

These are functional improvements and minor bug fixes.

- **<Short title>** (<MM-XXXXX, Jira priority: X> if available) — _Introduced in <version>_ <1-2 sentence plain-English description.> <Inline config relevance note if applicable.> PR: [#<number>](https://github.com/mattermost/mattermost/pull/<number>)

## Plugin Updates

- **<Plugin name> updated to v<new>** — _<Plugin name> (currently v<installed>)_ — _Introduced in <version>_ <1-2 sentence description.> PR: [#<number>](https://github.com/mattermost/mattermost/pull/<number>)

> **Note:** Plugin updates listed here only cover pre-packaged plugins that ship with the Mattermost server package. Custom plugins or plugins installed independently from the Mattermost Plugin Marketplace are not included in this analysis.

---

**Summary:** <One paragraph summarizing the upgrade recommendation in plain language. Mention the total number of security fixes and their severity range, call out the most urgent stability issue(s) by name, note plugin update relevance if any, and end with a recommendation. Do NOT oversell.>
```

### Formatting rules — strict

- **Header line** is a single paragraph combining Instance, Current Version, and Recommended Version with bolded labels — NOT a three-line list.
- **Arrow in title** uses the Unicode `→` character, not `->`.
- **Bullet entries** are a single paragraph: bold title, then Jira metadata in parentheses (if any), em-dash `—`, italic `_Introduced in <version>_` (using underscores, not asterisks), then the description sentences, then config relevance inline (if any), then PR link at the end. No line break in the middle of the bullet.
- **Plugin entries** have a second italic segment `_<Plugin name> (currently v<version>)_` between the title and the "Introduced in" fragment.
- **SiteURL in header** is a linked markdown URL, not bare text.
- **Config relevance is inline**, never a separate section. Use language like "Your instance has X enabled (`SettingName: true`), making this directly relevant." when it applies, and omit entirely when it doesn't.

### PR links
Use the **first** PR number from the commit subject (the original PR, not the cherry-pick backport number).

### Empty sections
If a section has no entries, omit that section entirely — but always include **Security Fixes** (even if the count is 0, the table still documents the absence) and always include the final **Summary**.

### Before writing the report — self-check
Compare your output against the reference example below. Your section names, header layout, bullet structure, italic/bold markers, and summary placement must match this pattern exactly. If any differ, fix them before returning.

### Reference example (the canonical format)

````markdown
# Upgrade Recommendation: 10.11.8 → 10.11.13

**Instance:** [https://mattermost.veevadev.com](https://mattermost.veevadev.com/) **Current Version:** 10.11.8 (released November 21, 2025) **Recommended Version:** 10.11.13 (released March 16, 2026)

## Security Fixes

Patch releases 10.11.9 through 10.11.13 collectively include **~30 security fixes** ranging from **low to high severity**. Per Mattermost's [responsible disclosure policy](https://mattermost.com/security-updates/), specific vulnerability details are not included here.

| Patch    | Severity Range | Approx. Count |
| -------- | -------------- | ------------- |
| 10.11.9  | Low to Medium  | 0             |
| 10.11.10 | Low to Medium  | 5             |
| 10.11.11 | Low to High    | 18            |
| 10.11.12 | Low to High    | 6             |
| 10.11.13 | Low to Medium  | 2             |

For full details on disclosed vulnerabilities (including CVE and MMSA identifiers), see: [https://mattermost.com/security-updates/](https://mattermost.com/security-updates/)

## Upgrade urgently — could cause outage or degradation

These fixes address server crashes or significant functional breakage.

- **Server panic when bot posts trigger persistent notifications** (MM-65575, Jira priority: High) — _Introduced in 10.11.10_ Fixed a server crash that occurred when a bot posted in a channel where persistent notifications were configured. The panic causes the server to crash every ~5 minutes until the post is acknowledged. Your instance has persistent notifications enabled (`AllowPersistentNotifications: true`), making this directly relevant. PR: [#34174](https://github.com/mattermost/mattermost/pull/34174)
- **Plugin config wiped on re-enablement** — _Introduced in 10.11.13_ Re-enabling a plugin would lose its custom configuration, reverting to defaults. With your many active plugins (Jira, GitHub, GitLab, Confluence, Zoom, etc.), toggling a plugin off and back on would silently wipe its settings and break integrations. PR: [#35545](https://github.com/mattermost/mattermost/pull/35545)

## Quality of life — could be annoying for a subset of users

These are functional improvements and minor bug fixes.

- **Postgres full-text search performance regression reverted** (MM-66782, Jira priority: Medium) — _Introduced in 10.11.11_ Reverted a prior change that caused Postgres full-text search queries (e.g. "Recent mentions") to time out and return empty results. Your instance uses Elasticsearch with `DisableDatabaseSearch: true`, so this is low relevance unless database search is re-enabled. PR: [#35063](https://github.com/mattermost/mattermost/pull/35063)
- **Login flow fix for SSO-only environments** — _Introduced in 10.11.10_ When email and username sign-in are both disabled (as in your environment), the server no longer attempts an unnecessary username/email lookup before falling through to SAML. Eliminates a needless error path during login. PR: [#34441](https://github.com/mattermost/mattermost/pull/34441)
- **Channel settings modal URL regression** (MM-64725, Jira priority: Medium) — _Introduced in 10.11.11_ After renaming a channel, search using `in:[channel name]` would show the new display name instead of the original channel handle. Regression since v10.8. PR: [#33500](https://github.com/mattermost/mattermost/pull/33500)
- **Keyboard focus wrong when using Shift-Up to reply in thread** (MM-65186, Jira priority: High) — _Introduced in 10.11.9_ When pressing Shift+Up to reply in a thread, keyboard focus stayed in the main compose box instead of moving to the thread reply panel. PR: [#34627](https://github.com/mattermost/mattermost/pull/34627)
- **Group member pagination fix** — _Introduced in 10.11.12_ Group member lists were loaded all at once instead of being paginated, which could cause performance issues for large groups. PR: [#35172](https://github.com/mattermost/mattermost/pull/35172)
- **Reduced unnecessary rerenders on shared channels tooltip** — _Introduced in 10.11.9_ Performance improvement that eliminates unnecessary client-side rerenders. PR: [#34336](https://github.com/mattermost/mattermost/pull/34336)

## Plugin Updates

- **Zoom plugin updated to v1.12.0** — _Zoom (currently v1.8.0)_ — _Introduced in 10.11.10 (v1.11.0), 10.11.12 (v1.12.0)_ The bundled Zoom plugin was updated through two releases. Your instance is on v1.8.0, so this upgrade brings four minor versions of fixes and improvements. PR: [#34734](https://github.com/mattermost/mattermost/pull/34734), [#35167](https://github.com/mattermost/mattermost/pull/35167)
- **Jira plugin updated to v4.5.0** — _Jira (currently v4.4.1)_ — _Introduced in 10.11.10_ The bundled Jira plugin was updated from v4.4.1 to v4.5.0. PR: [#34803](https://github.com/mattermost/mattermost/pull/34803)

> **Note:** Plugin updates listed here only cover pre-packaged plugins that ship with the Mattermost server package. Custom plugins or plugins installed independently from the Mattermost Plugin Marketplace are not included in this analysis.

---

**Summary:** This upgrade from 10.11.8 to 10.11.13 includes approximately 30 security fixes across five patch releases, with severities ranging up to high. The bulk of security fixes landed in 10.11.11. On the stability side, the most urgent fix is a server panic triggered by bots posting in channels with persistent notifications (10.11.10) — which can crash the server every ~5 minutes. The plugin config loss on re-enablement (10.11.13) is also significant given your many active integrations. The remaining bug fixes are minor quality-of-life improvements. Given the volume and severity of the security fixes alone, upgrading is strongly recommended.
````
