# Mattermost Enterprise Codebase Guide (Support Focus)

This guide helps navigate the Mattermost Enterprise repo to answer support questions about enterprise-only features like LDAP, SAML, clustering, compliance, data retention, etc.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/dotfiles/Mattermost-Enterprise/CLAUDE.md`

## Related Repository

The open-source Mattermost server lives at `../Mattermost`. That repo defines the interfaces this code implements, all config structs, error translation strings, API endpoints, and the store layer. Its `CLAUDE.md` has a full guide to navigating that codebase. When investigating an enterprise feature, you'll often need both repos:

- **Interfaces**: `../Mattermost/server/einterfaces/` defines the contracts
- **Registration hooks**: `../Mattermost/server/channels/app/enterprise.go` and `../Mattermost/server/channels/app/platform/enterprise.go`
- **License model**: `../Mattermost/server/public/model/license.go`
- **Config structs**: `../Mattermost/server/public/model/config.go`
- **Error translations**: `../Mattermost/server/i18n/en.json`
- **API endpoints that gate features**: `../Mattermost/server/channels/api4/`

## How This Repo Integrates

This repo implements the enterprise feature interfaces defined in the open-source server. Each package registers its implementation at startup, and the server's einterfaces layer dispatches to these implementations at runtime. Community edition builds exclude this code entirely.

## Enterprise Feature Packages

### LDAP (`ldap/`)
- **Implements**: `einterfaces.LdapInterface`, `einterfaces.LdapDiagnosticInterface`, `ejobs.LdapSyncInterface`
- **Key files**: `ldap.go` (auth + sync), `ldap_sync_job.go` (worker), `ldap_sync_scheduler.go`, `ldap_session.go`, `ldap_membership.go`, `diagnostic.go`
- **What it does**: LDAP/AD authentication, user/group sync with scheduling, custom profile attribute mapping, diagnostic tests
- **License**: `Features.LDAP`, `Features.LDAPGroups`

### SAML (`saml/`)
- **Implements**: `einterfaces.SamlInterface`
- **Key files**: `saml.go`, `authnrequest.go`, `authnresponse.go`, `service_provider.go`, `certificate.go`, `xmlsec.go`
- **What it does**: SAML 2.0 SSO, SP metadata generation, assertion validation, attribute mapping, certificate management
- **License**: `Features.SAML`
- **Note**: Requires `xmlsec1` for testing

### Clustering (`cluster/`)
- **Implements**: `einterfaces.ClusterInterface`
- **Key files**: `cluster.go`, `gossip_client.go`, `gossip_client_*.go` (per-feature handlers), `redis.go`, `log_parser.go`
- **What it does**: Inter-node gossip protocol (memberlist), Redis broadcast, leader election, config/log/plugin-status sync across nodes, support packet collection from all nodes
- **License**: `Features.Cluster`

### Compliance (`compliance/`)
- **Implements**: `einterfaces.ComplianceInterface`
- **Key files**: `compliance.go`
- **What it does**: Compliance report generation, CSV/ZIP export for audit trails
- **License**: `Features.Compliance`

### Data Retention (`data_retention/`)
- **Implements**: `einterfaces.DataRetentionInterface`, `ejobs.DataRetentionJobInterface`
- **Key files**: `data_retention.go`, `worker.go`, `scheduler.go`, `orphaned_rows.go`
- **What it does**: Global and per-policy retention, automatic message/file deletion, Elasticsearch index cleanup, orphaned row cleanup
- **License**: `Features.DataRetention`

### Message Export (`message_export/`)
- **Implements**: `einterfaces.MessageExportInterface`, `ejobs.MessageExportJobInterface`
- **Key files**: `message_export.go`, `worker.go`, `scheduler.go`, `membership_map.go`
- **Export formats**: `csv_export/`, `actiance_export/`, `global_relay_export/` (with SFTP delivery and SMTP notification)
- **What it does**: Batch message export in multiple compliance formats, incremental export with timestamp tracking
- **License**: `Features.MessageExport`, `Features.Compliance`

### Cloud (`cloud/`)
- **Implements**: `einterfaces.CloudInterface`
- **Key files**: `cloud.go`, `audit_logging.go`, `ip_filtering.go`, `cloud_mock.go`
- **What it does**: Cloud Workspace Service (CWS) integration, subscription/customer management, cloud limits, invoice management, audit logging, IP filtering
- **License**: Cloud-specific

### Access Control (`access_control/`)
- **Implements**: `einterfaces.AccessControlServiceInterface`, `ejobs.AccessControlSyncJobInterface`
- **Key files**: `service.go` (PAP + PDP), `decision.go`, `administration.go`, `sync.go`, `worker.go`
- **CEL utilities**: `cel_utils/` — `attributes.go`, `linter.go`, `normalizer.go`, `sqlizer.go`, `visual_format.go`
- **What it does**: Attribute-Based Access Control (ABAC) using CEL expressions, policy CRUD, policy evaluation, SQL generation from CEL, policy sync jobs
- **License**: Enterprise Advanced, Feature flag: `AttributeBasedAccessControl`

### Auto-Translation (`autotranslation/`)
- **Implements**: `einterfaces.AutoTranslationInterface`
- **Key files**: `autotranslation.go`, `translation.go`, `worker.go`, `recovery_job.go`, `language.go`, `cache.go`, `mask.go`
- **Providers**: `provider/libretranslate/`, `provider/agents/` (Mattermost Agents AI)
- **What it does**: Channel/user-level translation, language detection, caching (10-min TTL), worker pool, sensitive data masking
- **License**: Enterprise Advanced (`MinimumEnterpriseAdvancedLicense`)

### License Management (`license/`)
- **Implements**: `einterfaces.LicenseInterface`
- **Key files**: `license.go`
- **What it does**: Mattermost Entry license generation, trial eligibility checking

### Account Migration (`account_migration/`)
- **Implements**: `einterfaces.AccountMigrationInterface`
- **Key files**: `account_migration.go`
- **What it does**: Migrate users between auth providers (to LDAP or SAML), dry-run mode, duplicate handling

### Notification (`notification/`)
- **Implements**: `einterfaces.NotificationInterface`
- **Key files**: `notification.go`
- **What it does**: ID-loaded push notifications (content fetched server-side rather than embedded in push)

### OAuth Providers (`oauth/`)
- **Subdirectories**: `google/`, `office365/`, `openid/`
- **What it does**: Google, Office 365/Entra ID, and OpenID Connect authentication

### Push Proxy (`push_proxy/`)
- **Implements**: `einterfaces.PushProxyInterface`
- **Key files**: `push_proxy.go`, `push_proxy_worker.go`, `push_proxy_scheduler.go`
- **What it does**: Push proxy auth token management, RSA key validation, periodic token refresh

### Intune (`intune/`)
- **Implements**: `einterfaces.IntuneInterface`
- **Key files**: `intune.go`, `validator.go`
- **What it does**: Microsoft Entra ID MSAL token validation, Intune MAM policy enforcement, user matching to O365/SAML
- **License**: Enterprise Advanced, requires Office 365 or SAML enabled

### IP Filtering (`ip_filtering/`)
- **Implements**: `einterfaces.IPFilteringInterface`
- **Key files**: `cloud.go`
- **What it does**: IP range filtering for cloud deployments (cloud-only)

### Outgoing OAuth (`outgoing_oauth_connections/`)
- **Implements**: `einterfaces.OutgoingOAuthConnectionInterface`
- **Key files**: `outgoing_oauth_connection.go`, `oauth_client.go`
- **What it does**: OAuth 2.0 client connections to external services, token refresh, audience matching

## Error Messages

Enterprise errors use translation IDs prefixed with `"ent."`:
- `ent.saml.license_disable.app_error`
- `ent.data_retention.generic.license.error`
- `ent.autotranslation.feature_unavailable`

These IDs are translated using the same i18n system as the core server (`../Mattermost/server/i18n/en.json`).

Common error helpers in `data_retention/data_retention.go`:
- `newLicenseError()` — returns error when license doesn't include the feature
- `newInternalError()` — returns internal server error with runtime caller info

## License Checking Pattern

Every enterprise feature validates its license before operating:

```go
func (impl *FeatureImpl) hasValidLicense() bool {
    license := impl.Server.License()
    return license != nil && *license.Features.FeatureName
}
```

License tiers (from `model`): E10, E20, Professional, Enterprise, Enterprise Advanced, Mattermost Entry.

## Common Support Investigation Patterns

### "Why is enterprise feature X not working?"
1. Check if the license includes the feature — look at `Features` struct in `../Mattermost/server/public/model/license.go`
2. Check if the feature is enabled in config — look at the relevant settings struct in `../Mattermost/server/public/model/config.go`
3. Check the enterprise implementation here for additional validation (e.g., `checkConfigAndLicense()` in `saml/saml.go`)

### "Where is the LDAP/SAML/etc. implementation?"
1. Find the interface in `../Mattermost/server/einterfaces/{feature}.go`
2. Find the implementation in `{feature}/` in this repo
3. Find the API endpoint in `../Mattermost/server/channels/api4/{feature}.go`

### "What does enterprise error 'ent.X.Y' mean?"
1. Search `../Mattermost/server/i18n/en.json` for the error ID
2. Grep this repo for the error ID to find where it's raised
3. Read surrounding code for the triggering condition

### "How does clustering work?"
1. `cluster/cluster.go` — main coordination
2. `cluster/gossip_client.go` — inter-node messaging via memberlist
3. `cluster/redis.go` — Redis-based broadcast alternative
4. Config: `ClusterSettings` in the open-source config model

### "How does data retention / message export work?"
1. Check the implementation in `data_retention/` or `message_export/`
2. Both use a scheduler + worker pattern with the job system
3. Jobs are defined in `../Mattermost/server/public/model/job.go`
4. Job infrastructure in `../Mattermost/server/channels/jobs/`
