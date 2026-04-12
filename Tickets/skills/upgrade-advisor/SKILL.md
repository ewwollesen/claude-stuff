---
name: upgrade-advisor
description: Analyze fixes between a customer's current Mattermost version and the latest patch to generate a human-readable upgrade recommendation.
argument-hint: "<ticket-number>"
allowed-tools: Read, Grep, Glob, Bash, Agent
---

# Mattermost Upgrade Advisor

You are generating an upgrade recommendation report for a Mattermost customer. Your job is to find meaningful bug fixes and security patches between their current version and the latest available patch in the same minor series, then explain each one in plain language.

## Input

`$ARGUMENTS`

This command supports two modes:

### Mode 1 — Ticket number (with support packet)
If the argument looks like a ticket number (e.g. `50987-testing`, `12345`), treat it as a ticket directory at `~/Downloads/Tickets/<ticket-number>/` and follow Step 1A below.

### Mode 2 — Version only (no support packet)
If the argument contains a version number but no ticket directory (e.g. the user says something like "no support packet, version is 10.11.10" or just passes a bare version like `10.11.10`), extract the version and skip directly to Step 2. You won't have a config, plugin list, or instance URL — that's fine. Generate the report without config-specific relevance tagging and without plugin filtering. Note in the report header that no support packet was available.

**How to decide:** Look at `$ARGUMENTS`. If it's a path to a ticket directory that exists, use Mode 1. If it contains freeform text mentioning a version, or is just a version number, use Mode 2. When in doubt, check if the directory exists first.

## Step 1A — Discover the customer's version and config (ticket mode only)

### Find the support packet

Look inside the ticket directory for a support packet directory. These follow the naming pattern `mm_support_packet_*`. There may be multiple — use the most recent one (latest timestamp in the directory name).

```bash
ls -d ~/Downloads/Tickets/<ticket-number>/mm_support_packet_* 2>/dev/null | sort | tail -1
```

### Get the version

The support packet contains a `metadata.yaml` at its root. Read it and extract the `server_version` field. This is the customer's current version.

### Find the config

Inside the support packet, look for node subdirectories (e.g. `ip-172-16-*` or hostname directories). Each node has a `sanitized_config.json`. Read the config from the first node — in HA deployments the config is typically identical across nodes.

```bash
find <support_packet_dir> -name "sanitized_config.json" | head -1
```

If no support packet exists, check for a `config.json` directly in the ticket directory. If neither exists, proceed without config (you can still generate the report, just without config-specific relevance tagging).

If you cannot determine the version from any available artifact, ask the user.

### Get installed plugins

The support packet contains a `plugins.json` file at its root. Read it to get the list of installed plugins and their versions. The file has two arrays: `enabled` and `disabled`. Each entry has `id`, `name`, and `version` fields.

```bash
cat <support_packet_dir>/plugins.json
```

Build a map of installed plugin IDs to their current versions. You will use this to:
1. **Skip plugin update commits for plugins that are not installed.** If a commit updates the Calls plugin but Calls is not in the customer's `plugins.json` (neither enabled nor disabled), omit it from the report entirely.
2. **Show the customer's current plugin version** when reporting a plugin update, so they can see what version they're on vs. what version the update brings.

## Step 2 — Determine the target version

First, fetch the latest tags so we have up-to-date release information:

```bash
git -C ~/Repositories/Claude-Repos/Mattermost fetch --tags
```

Then find the latest patch tag in the same minor series:

```bash
git -C ~/Repositories/Claude-Repos/Mattermost tag --list 'v<major>.<minor>.*' | grep -v rc | sort -V | tail -1
```

If the customer is already on the latest patch, tell the user and stop.

### Get release dates

Get the release date for both the customer's current version and the target version using the tag commit date:

```bash
git -C ~/Repositories/Claude-Repos/Mattermost log -1 --format="%ai" v<current_version>
git -C ~/Repositories/Claude-Repos/Mattermost log -1 --format="%ai" v<target_version>
```

Format these as human-readable dates (e.g. "November 21, 2025"). You'll use them in the report header.

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
- `~/Repositories/Claude-Repos/Mattermost` — community/open-source server
- `~/Repositories/Claude-Repos/Enterprise` — enterprise-only features (LDAP, SAML, compliance, OpenID, etc.)

The enterprise repo contains fixes that only apply to licensed enterprise features but are critical for customers who use them. Both repos use the same version tags.

First, fetch tags from both repos:

```bash
git -C ~/Repositories/Claude-Repos/Mattermost fetch --tags
git -C ~/Repositories/Claude-Repos/Enterprise fetch --tags
```

Then get all the patch versions in the range. For example, if the customer is on 10.11.8 and the target is 10.11.13, the intermediate versions are 10.11.9, 10.11.10, 10.11.11, 10.11.12, 10.11.13.

```bash
git -C ~/Repositories/Claude-Repos/Mattermost tag --list 'v<major>.<minor>.*' | grep -v rc | sort -V
```

Filter this list to only versions *after* the customer's current version and up to (including) the target.

Then for each consecutive pair of versions, get commits from **both** repos:

```bash
# Community repo — commits introduced in 10.11.9
git -C ~/Repositories/Claude-Repos/Mattermost log v10.11.8..v10.11.9 --format="COMMIT:%H%nSUBJECT:%s%nBODY:%b%n---END---"

# Enterprise repo — commits introduced in 10.11.9
git -C ~/Repositories/Claude-Repos/Enterprise log v10.11.8..v10.11.9 --format="COMMIT:%H%nSUBJECT:%s%nBODY:%b%n---END---"
```

Track which patch version and which repo each commit belongs to. When analyzing diffs for enterprise commits, use the enterprise repo path:

```bash
git -C ~/Repositories/Claude-Repos/Enterprise diff <hash>^..<hash> --stat
```

## Step 5 — Analyze each commit

### What to include
- **Security fixes** — sanitization, input validation, auth issues, redirect validation, constant-time comparisons, XSS, CSRF, injection, CVEs
- **Bug fixes** — changes that fix broken behavior users would actually experience (UI bugs, incorrect data, crashes, broken workflows)
- **Behavioral changes** — anything that changes how a feature works in a way a user or admin would notice

### What to exclude
- Test-only changes (adding/modifying test cases with no production code changes)
- CI/CD and workflow file changes
- Linter/formatting fixes
- Build toolchain updates (Go version bumps, base image changes)
- Internal dependency version bumps with no user-facing impact
- Version bump commits ("Update latest patch version to X.Y.Z")
- Automated merge commits
- Code comments or documentation-only changes
- **Plugin updates for plugins the customer does not have installed.** If a commit updates a prepackaged plugin (e.g. Calls, Boards, Playbooks) and that plugin does not appear in the customer's `plugins.json`, omit it entirely. Do not list it with a note that it's not installed — just skip it.

### For each included commit

1. Read the commit subject and body
2. **If the subject is unclear** (e.g. "MM 65084 server-side", "Automated cherry pick of #34247"), look at the actual diff:
   ```bash
   git -C ~/Repositories/Claude-Repos/Mattermost diff <hash>^..<hash> --stat
   ```
   And if needed, read the changed files:
   ```bash
   git -C ~/Repositories/Claude-Repos/Mattermost show <hash> -- <specific-file>
   ```
3. Write a **1-2 sentence plain-English summary** of what the fix does and why it matters
4. Determine if it's relevant to the customer's enabled features

## Rules — CRITICAL

### Honesty above all
- **Do NOT guess or assume what a commit does.** If the commit subject is vague and the diff doesn't make it clear, say: *"Unable to determine the impact of this change from the available information."*
- **Do NOT invent details.** If you can't tell whether a fix is relevant to the customer's config, say so. It is always better to say "I'm not sure if this affects your setup" than to fabricate a connection.
- **Do NOT embellish.** Describe what the fix does factually. Don't add urgency or severity that isn't supported by the code change.
- **Do NOT make up Jira ticket descriptions.** You only know what is in the commit message and the diff. If the Jira ID is `MM-65084` and the commit subject just says "server-side", do not invent what MM-65084 is about.

### Config relevance
When you have the customer's config, note which fixes are relevant to features they have **enabled**. But be honest — if you're not sure whether a fix relates to a config setting, don't force the connection. Just list it as a general fix.

## Step 6 — Generate the report

Output a markdown report:

```
# Upgrade Recommendation: <current> -> <target>

**Instance:** <SiteURL from config>
**Current Version:** <current> (released <date>)
**Recommended Version:** <target> (released <date>)

## Security Fixes

- **<Short title>** (<Jira ID if available>) — *Introduced in <version>*
  <1-2 sentence plain-English description of the fix and why it matters>
  PR: [#<number>](https://github.com/mattermost/mattermost/pull/<number>)

## Fixes Relevant to Your Environment

- **<Short title>** (<Jira ID if available>) — *<feature name>* — *Introduced in <version>*
  <1-2 sentence plain-English description>
  PR: [#<number>](https://github.com/mattermost/mattermost/pull/<number>)

For plugin updates, include the customer's installed version:

- **Jira plugin updated to v4.5.0** — *Jira (currently v4.4.1)* — *Introduced in 10.11.10*
  The bundled Jira plugin was updated from the version shipped with the server to v4.5.0.
  PR: [#<number>](https://github.com/mattermost/mattermost/pull/<number>)

When there are plugin updates in this section, add the following note at the end of the section:

> **Note:** Plugin updates listed here only cover pre-packaged plugins that ship with the Mattermost server package. Custom plugins or plugins installed independently from the Mattermost Plugin Marketplace are not included in this analysis.

## Other Notable Fixes

- **<Short title>** (<Jira ID if available>) — *Introduced in <version>*
  <1-2 sentence plain-English description>
  PR: [#<number>](https://github.com/mattermost/mattermost/pull/<number>)

---

**Summary:** <One paragraph summarizing the upgrade recommendation in plain language>
```

### PR links
Use the **first** PR number from the commit subject (the original PR, not the cherry-pick backport number).

### Summary paragraph
Write a brief, honest summary. If there are security fixes, emphasize those. If there's only one minor fix, say that. If there's nothing urgent, say that too. Do NOT oversell.

### Empty sections
If a section has no entries (e.g. no security fixes), omit that section entirely.
