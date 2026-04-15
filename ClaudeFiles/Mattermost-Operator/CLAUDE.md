# Mattermost Operator Codebase Guide (Support Focus)

This guide helps navigate the Mattermost Kubernetes Operator codebase to answer support questions: understanding CRD specs, reconciliation behavior, deployment sizing, database/filestore configuration, and troubleshooting operator issues.

> **READ-ONLY REFERENCE COPY**
> This is a read-only reference for code search and support investigations.
> - DO NOT make local code changes, create branches, or commit to this repo
> - Before searching, refresh from remote: `git fetch origin && git pull`
> - Source of truth for this file: `~/Repositories/Claude-Stuff/ClaudeFiles/Mattermost-Operator/CLAUDE.md`

## Related Repositories

The Mattermost server that this operator deploys lives at `../Mattermost`. Its `CLAUDE.md` covers server config, error messages, API endpoints, and the store layer. When investigating operator issues, you'll often need both repos — the operator manages Kubernetes resources, while the server codebase explains the Mattermost config and behavior those resources produce.

## Repository Structure

Built with **Kubebuilder v2** using `controller-runtime`. Go 1.24, controller-runtime v0.21.

```
main.go                         # Operator entrypoint — registers schemes, creates manager
apis/mattermost/
  v1beta1/                      # Current CRD: Mattermost (installation.mattermost.com)
    mattermost_types.go         # Spec/Status structs, sizing, ingress, DB, filestore
    zz_generated.deepcopy.go    # Generated deep copy methods
  v1alpha1/                     # Legacy CRDs (mattermost.com group)
    clusterinstallation_types.go  # DEPRECATED: ClusterInstallation spec/status
    mattermostrestoredb_types.go  # MattermostRestoreDB spec/status
controllers/mattermost/
  mattermost/                   # v1beta1 Mattermost reconciler
    controller.go               # Main reconcile loop, rate limiter, watches
    mattermost.go               # Core reconciliation logic (service, ingress, deployment)
    database.go                 # Database resource reconciliation
    file_store.go               # Filestore resource reconciliation
    health_check.go             # Mattermost health check integration
    utils.go                    # Controller utilities
  clusterinstallation/          # v1alpha1 ClusterInstallation reconciler (DEPRECATED)
    controller.go               # Legacy reconcile loop
    mattermost.go               # Legacy deployment generation
    mysql.go                    # MySQL operator integration
    postgresql.go               # PostgreSQL support
    minio.go                    # MinIO filestore setup
    bluegreen.go                # BlueGreen deployment strategy
    canary.go                   # Canary deployment strategy
    conversion.go               # v1alpha1 → v1beta1 migration
    migration.go                # Migration helpers
  mattermostrestoredb/          # Database restore reconciler
    controller.go               # Restore reconcile loop
    mysql.go                    # MySQL restore logic
pkg/
  mattermost/                   # Kubernetes resource generation
    mattermost_v1beta.go        # Service, Ingress, Deployment for v1beta1
    mattermost.go               # Generic resource generation
    database_mysql.go           # MySQL connection string building
    database_external.go        # External database support
    file_store.go               # S3/MinIO file storage config
    env_var.go                  # Environment variable management
    healthcheck/health_check.go # Health probe logic
    constants.go                # Sizing presets, limits
    helpers.go                  # Utility functions
  resources/                    # Kubernetes resource CRUD
    create_resources.go         # Create, update, fetch resources
    mysql.go                    # MySQL operator resource management
    minio.go                    # MinIO operator resource management
    update_job.go               # Update job management
  database/                     # Database abstraction
    database.go                 # Database interface
    mysql_operator/             # MySQL operator CRD types
  components/                   # Dependency operator integration
    minio/minio.go              # MinIO operator helpers
    mysql/mysql.go              # MySQL operator helpers
  client/                       # Generated Kubernetes client code
config/
  crd/bases/                    # Generated CRD YAML manifests
  rbac/                         # RBAC roles and bindings
  manager/                      # Operator Deployment manifest
  prometheus/                   # ServiceMonitor for metrics
  samples/                      # Example CR manifests
resources/                      # Dev/test resource manifests (MinIO, Postgres, secrets)
test/                           # Integration and E2E tests
version/version.go              # Semantic version (1.24.0), build-time injection
```

## Custom Resource Definitions

### Mattermost (v1beta1) — Primary CRD

**Group**: `installation.mattermost.com` | **Kind**: `Mattermost`

Defined in `apis/mattermost/v1beta1/mattermost_types.go`. This is the current, actively maintained CRD.

**Key spec fields**:
- `Image` / `Version` — Mattermost container image and tag
- `Size` — deployment sizing preset: `100users`, `1000users`, `5000users`, `10000users`, `25000users`
- `Replicas` — override replica count (takes precedence over size preset)
- `Scheduling` — resource requests/limits override
- `Ingress` — host(s), TLS, annotations, IngressClass
- `Database` — external DB config (secret name with connection string)
- `FileStore` — external filestore config (S3/MinIO secret)
- `ElasticSearch` — Elasticsearch connection config
- `MattermostEnv` — extra environment variables injected into pods
- `LicenseSecret` — Kubernetes secret containing the Mattermost license
- `Volumes` / `VolumeMounts` — custom pod volumes
- `DNSConfig` — custom DNS settings
- `ServiceAnnotations` — annotations on the generated Service
- `ResourcePatch` — experimental: raw patches applied to generated resources
- `Probes` — liveness/readiness probe configuration
- `PodTemplate` — pod-level customization (init containers, sidecar containers, security context)

**Status fields**: `State`, `Replicas`, `UpdatedReplicas`, `ObservedGeneration`, `Image`, `Version`, `Endpoint`

**States**: `stable`, `reconciling`, `error`

### ClusterInstallation (v1alpha1) — DEPRECATED

**Group**: `mattermost.com` | **Kind**: `ClusterInstallation`

Defined in `apis/mattermost/v1alpha1/clusterinstallation_types.go`. Legacy CRD with migration path to v1beta1.

Additional features not in v1beta1: BlueGreen and Canary deployment strategies, built-in MinIO/MySQL operator management.

### MattermostRestoreDB (v1alpha1)

**Group**: `mattermost.com` | **Kind**: `MattermostRestoreDB`

Database restoration from S3/XtraBackup backups. States: `restoring`, `finished`, `failed`.

## Reconciliation

### v1beta1 Mattermost Controller

Entry point: `controllers/mattermost/mattermost/controller.go`

**Reconcile loop**:
1. Fetch the Mattermost CR (exit if deleted)
2. Set defaults on the spec
3. Validate the spec
4. Check rate limiter — if too many installations are reconciling, requeue with configurable delay
5. Reconcile database resources (secrets, connection config)
6. Reconcile filestore resources (secrets, S3 config)
7. Generate and apply Kubernetes resources: Service, Ingress, Deployment
8. Run health check against Mattermost API
9. Update CR status

**Rate limiting**: Controlled by `MaxReconcilingInstallations` env var (default: 20). When exceeded, reconciliation is dequeued with `RequeueOnLimitDelay` (default: 20s).

**Concurrency**: `MaxReconcileConcurrency` env var (default: 1) controls parallel reconcile goroutines.

**Watches**: Owns Service, Secret, Ingress, Deployment, Job — changes to any of these trigger reconciliation of the parent Mattermost CR.

### Resource Generation

`pkg/mattermost/mattermost_v1beta.go` generates:
- **Service**: ClusterIP by default, configurable annotations
- **Ingress**: Created only if hosts are specified, supports TLS and IngressClass
- **Deployment**: Mattermost container with env vars, probes, volumes, init containers

### Strategic Merge Patching

Resources are updated using `banzaicloud/k8s-objectmatcher` which computes strategic merge patches to avoid conflicts with other controllers (e.g., HPA modifying replica count).

## Deployment Sizing

Defined in `pkg/mattermost/constants.go`:

| Preset | Replicas | CPU Req | CPU Limit | Mem Req | Mem Limit |
|--------|----------|---------|-----------|---------|-----------|
| 100users | 1 | 100m | 2000m | 256Mi | 512Mi |
| 1000users | 2 | 100m | 2000m | 256Mi | 4Gi |
| 5000users | 2 | 100m | 4000m | 256Mi | 8Gi |
| 10000users | 3 | 200m | 4000m | 512Mi | 8Gi |
| 25000users | 4 | 2000m | 8000m | 4Gi | 16Gi |

Custom sizing: set `Replicas` and `Scheduling.Resources` to override presets.

## Database Configuration

### External Database (v1beta1)

The operator expects a Kubernetes Secret containing the DB connection string. Specified via `spec.database.external.secret`.

The secret must contain a `DB_CONNECTION_STRING` key with a DSN like:
```
postgres://user:pass@host:5432/mattermost?sslmode=disable
mysql://user:pass@tcp(host:3306)/mattermost?charset=utf8mb4
```

### MySQL Operator Integration (v1alpha1 only)

The legacy ClusterInstallation CRD can manage MySQL via the Oracle MySQL Operator. CRD types are bundled in `pkg/database/mysql_operator/`.

## File Store Configuration

### External Filestore (v1beta1)

Specified via `spec.fileStore.external.secret`. The secret must contain:
- `accesskey` — S3 access key
- `secretkey` — S3 secret key

Additional spec fields: `url`, `bucket`, `useS3SSL`

### MinIO Integration (v1alpha1 only)

The legacy CRD can deploy and manage MinIO via the MinIO Operator.

## Operator Bootstrap

`main.go` startup sequence:
1. Initialize structured logger (logrus via blubr)
2. Read env config: `MaxReconcilingInstallations`, `RequeueOnLimitDelay`, `MaxReconcileConcurrency`
3. Register API schemes: core K8s, v1alpha1, v1beta1, MinIO operator, MySQL operator
4. Create controller-runtime manager with metrics on `:8383`
5. Register all three reconcilers
6. Start manager with signal handler

**Leader election**: Optional via `--enable-leader-election` flag, required for HA operator deployments.

## RBAC

Defined in `config/rbac/role.yaml`. The operator needs cluster-wide permissions for:
- Mattermost CRDs (all verbs)
- Core resources: Deployments, Services, Ingresses, Secrets, Pods, Jobs, PVCs
- MinIO and MySQL operator CRDs (if using built-in database/filestore)
- Events (create, patch)

## Build System

Key Makefile targets:
- `make build` — compile operator binary
- `make build-image` — build Docker image
- `make build-fips` — FIPS-compliant build
- `make generate` — run code generators (deepcopy, clients, CRD manifests)
- `make manifests` — regenerate CRD YAML from Go types
- `make unittest` — run unit tests
- `make run` — run operator locally against current kubeconfig
- `make install` / `make uninstall` — install/uninstall CRDs
- `make deploy` — deploy operator to cluster
- `make kind-start` / `make kind-load-image` — local Kind cluster dev

## Common Support Investigation Patterns

### "Why is the Mattermost CR stuck in 'reconciling' state?"
1. Check operator logs for errors during reconciliation
2. Check if `MaxReconcilingInstallations` limit is being hit (look for requeue messages)
3. Verify the database secret exists and contains a valid `DB_CONNECTION_STRING`
4. Verify the filestore secret exists with valid `accesskey`/`secretkey`
5. Check if the generated Deployment pods are crashing (kubectl get pods)

### "Why aren't my spec changes taking effect?"
1. The operator uses strategic merge patching — check if another controller (HPA, VPA) is overriding fields
2. Check the CR status for errors: `kubectl get mattermost -o yaml`
3. Check operator logs for validation failures during reconciliation

### "How do I migrate from ClusterInstallation to Mattermost?"
1. See `docs/migration.md` for the migration guide
2. The `controllers/mattermost/clusterinstallation/conversion.go` file handles v1alpha1 → v1beta1 conversion
3. Key differences: v1beta1 uses external DB/filestore secrets instead of managing operators directly

### "What sizing should I use?"
1. Check `pkg/mattermost/constants.go` for the preset definitions
2. Presets can be overridden with explicit `replicas` and `scheduling.resources`
3. The preset names (100users, 1000users, etc.) are rough guidelines — actual sizing depends on usage patterns

### "Why is the health check failing?"
1. Health check logic is in `controllers/mattermost/mattermost/health_check.go`
2. The operator hits the Mattermost API health endpoint
3. Common causes: database connectivity, license issues, missing config
4. Health check failures cause a 6-second requeue delay

### "How do I add custom environment variables?"
1. Use `spec.mattermostEnv` in the Mattermost CR — these are injected into the pod spec
2. For sensitive values, reference a Secret via `valueFrom.secretKeyRef`
3. The env var list is built in `pkg/mattermost/env_var.go`
