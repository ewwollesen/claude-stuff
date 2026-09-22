# Handoff: read-your-own-write bug in app migrations (property fields on read replicas)

**Written:** 2026-08-17
**Source of findings:** code inspection of `mattermost/mattermost` at tag `v11.7.8`
and at `origin/master` commit `338dc6c74d`.
**Origin:** customer report following a 10.11.12 → 11.7.8 upgrade on Aurora PostgreSQL
with a reader endpoint configured.

> **Note for the receiving agent:** this brief was produced in a *read-only reference
> clone*. No code was changed. You are expected to make the change in a writable
> working clone. Every line number below was verified at the ref stated next to it,
> but `origin/master` moves — **locate code by symbol name, not by line number**, and
> re-confirm before editing.

---

## 1. The task

In a writable clone of `mattermost/mattermost`, fix the read-your-own-write race in
the app migrations that create property groups and fields. The migration writes a row
via the master/writer DB handle and then immediately reads that same row back via the
replica/reader handle, with no lag tolerance, no retry, and no master fallback. On a
deployment with a real read replica that is even slightly behind, the read returns
zero rows and the server calls `mlog.Fatal` during startup.

Deliver:

1. A fix for the reported crash (`Managed Category Properties Setup`), targeted for
   backport to `release-11.7`.
2. A fix for the same pattern in the sibling migrations on `master` (there are four —
   see §5).
3. A regression guard (see §8 — a true reproduction test is not achievable; do not
   burn time attempting one before reading that section).

---

## 2. What the customer saw

```
fatal [2026-08-12 23:22:50.102 -04:00] Failed to run app migration
  caller="app/migrations.go:1017"
  migration="Managed Category Properties Setup"
  error="failed to get managed category field: property_field_get_by_name_select: sql: no rows in result set"
```

They confirmed via `psql` that the row *had* durably committed on the writer. Starting
the server a second time succeeded. Their own root-cause analysis was correct; this
brief confirms and extends it.

Their environment: Aurora PostgreSQL, both writer and reader endpoints configured in
`SqlSettings`, Enterprise licensed. The upgrade ran schema migrations 155→195
immediately beforehand, ending with `000195_threadmemberships_cleanup_v2.up.sql` — a
single unbounded `DELETE` across `ThreadMemberships`. That produced the WAL burst that
widened the replication window.

**The WAL burst is the trigger, not the cause.** The code tolerates zero lag, so any
nonzero lag at that instant reproduces it. Do not "fix" this by tuning migration 195.

---

## 3. Verified mechanism (at `v11.7.8` — the version that crashed)

| Step | Location | DB handle |
|---|---|---|
| Create the field | `app/migrations.go:794` → `sqlstore/property_field_store.go:51` | `GetMaster()` |
| Immediately read it back | `app/migrations.go:805` → `:814` → `sqlstore/property_field_store.go:81` | `GetReplica()` |
| Crash | `app/migrations.go:1017` (`mlog.Fatal`) | — |

Error string provenance, confirming the diagnosis:
`errors.Wrap(err, "property_field_get_by_name_select")` at `property_field_store.go:82`,
wrapped again by `failed to get managed category field: %w` at `migrations.go:816`.

**No retry covers this.** The retry layer does wrap the call
(`store/retrylayer/retrylayer.go:9999`), but `isRepeatableError`
(`retrylayer.go:595`) only returns true for Postgres SQLSTATE `40001`
(serialization_failure) and `40P01` (deadlock_detected). `sql: no rows in result set`
falls straight through.

**Blast radius.** `SqlStore.GetReplica()` (`sqlstore/store.go:467` at v11.7.8,
`:474` at master) returns the master handle when *any* of these hold:

```go
len(ss.settings.DataSourceReplicas) == 0 || ss.lockedToMaster || !ss.hasLicense()
```

So this only reaches deployments that are **both licensed and have replicas actually
configured**. That is also why it survived CI — see §8.

### Restarting works, but nothing in the code guarantees it

Important, and easy to miss. At v11.7.8:

- `migrations.go:801` writes the `managedCategorySetupDoneKey` flag via `SaveOrUpdate`.
- `migrations.go:814` is the read that then fails.

The flag is written **before** the failing read, so the migration marks itself complete
and *then* dies. On restart, `System().GetByName` is master-routed by design
(`sqlstore/system_store.go:88-89` wraps the read in
`store.RequestContextWithMaster(...)`), so the flag is *guaranteed* visible and the
`data != nil` branch at `:770` always wins. That branch short-circuits to the same
`cacheManagedCategoryIDs()` — performing the same two **replica** reads, with the same
absence of lag tolerance.

Be precise about why the retry nevertheless succeeds, because it is not random:

1. **The WAL burst does not recur.** Schema migrations 155→195 are already applied, so
   morph no-ops through them. The second boot reaches the app migrations with no write
   storm in front of it — the condition that produced the lag is structurally absent.
2. **The timing gap inverts.** On the first boot the write-to-read gap was microseconds.
   On the second, the rows have been committed since the previous boot — seconds to
   minutes of replication headroom.

So a restart is a **reliable-in-practice workaround carrying no guarantee**, not a lucky
escape. The residual risk requires lag from an *independent* source: production traffic
on the writer, another HA node running its own migrations concurrently, or a reader that
just failed over and is catching up cold. That risk is real and is higher during a
rolling HA upgrade than a single-node restart — but do not describe it as a coin flip.

Note also that `system_store.go` already applying `RequestContextWithMaster` is further
evidence the codebase understands this bug class and has fixed it selectively. The
property store simply was not covered.

**Consequence for the fix:** repairing only the create path is insufficient. The
short-circuit path at `:770-772` must be safe too.

---

## 4. Current state on `origin/master` (`338dc6c74d`)

The property store was refactored since 11.7. Relevant differences:

- Store reads now take a `context.Context` and route via `DBXFromContext`:
  `sqlstore/property_field_store.go:87` (`GetFieldByName`), `:95`
  (`GetFieldByNameForObjectType`), `:108` (`getFieldByName`).
- `GetFieldByNameForObjectType` is new; the managed-category migration uses it
  instead of `GetFieldByName`.
- `getFieldByName` now converts `sql.ErrNoRows` into `store.NewErrNotFound(...)`,
  so the customer's exact error string will differ on master.
- `RegisterPropertyGroup` now takes `*model.PropertyGroup` rather than a name string.
- `PropertyGroup` gained `Version` / `SchemaVersion`, and
  `PropertyGroupStore.IncrementVersion` exists.

**The bug is still present.** `getPropertyFieldByNameForObjectType`
(`app/properties/property_field.go:199-201`) passes a bare `context.Background()`,
so `DBXFromContext` resolves to `GetReplica()`. `cacheManagedCategoryIDs`
(`app/migrations.go:1152`) still calls it, and `GetPropertyGroup` (`:1153`) still
resolves to a replica read.

**The primitive you need already exists and is already used elsewhere in this
package.** In `sqlstore/context.go`:

```go
func WithMaster(ctx context.Context) context.Context          // deprecated alias
func RequestContextWithMaster(rctx request.CTX) request.CTX
func HasMaster(ctx context.Context) bool
func (ss *SqlStore) DBXFromContext(ctx context.Context) *sqlxDBWrapper
```

Existing precedent to copy — `app/properties/property_field.go:179-181`:

```go
func (ps *PropertyService) getPropertyFieldFromMaster(groupID, id string) (*model.PropertyField, error) {
	return ps.fieldStore.Get(store.WithMaster(context.Background()), groupID, id)
}
```

It is called from `app/properties/access_control.go:144` and
`app/properties/session_attributes.go:76`. `store.WithMaster` is likewise used for the
pre-update read at `property_field.go:257` and the linked-source read at `:65`.

So the team already understands this class of bug and applied the mechanism in the
runtime paths. **The migration paths were simply missed.**

---

## 5. Full inventory of affected call sites

### 5a. `master` — every property read in `app/migrations.go`

All of these run after a master write in the same function, and all resolve to the
replica. Verified line numbers at `338dc6c74d`:

| Line | Call | Migration |
|---|---|---|
| 644 | `RegisterPropertyGroup(...)` | Content Flagging |
| 651 | `SearchPropertyFields(...)` | Content Flagging |
| 749 | `GetPropertyFieldByNameForObjectType(...)` (create-failed guard) | Content Flagging |
| 786 | `RegisterPropertyGroup(...)` | Boards |
| 791 | `SearchPropertyFields(...)` | Boards |
| 871 | `GetPropertyFieldByNameForObjectType(...)` (create-failed guard) | Boards |
| 987 | `SearchPropertyFields(...)` | (shared property-setup helper) |
| 1027 | `GetPropertyFieldByNameForObjectType(...)` (create-failed guard) | (shared helper) |
| 1081 | `RegisterPropertyGroup(...)` | Managed Category |
| 1086 | `GetPropertyFieldByNameForObjectType(...)` (existence check) | Managed Category |
| 1102 | `GetPropertyFieldByNameForObjectType(...)` (create-failed guard) | Managed Category |
| **1153** | `GetPropertyGroup(...)` | **Managed Category — `cacheManagedCategoryIDs`** |
| **1158** | `GetPropertyFieldByNameForObjectType(...)` | **Managed Category — the reported crash** |

**Note the `SearchPropertyFields` variant is a distinct failure mode worth
understanding.** `SearchPropertyFields` reads the replica
(`property_field_store.go:335`) to decide which fields already exist. A stale replica
makes the migration believe a field is missing, so it attempts a create that violates
the typed unique index; the create-failed guard then re-reads the *same stale replica*,
also gets nothing, and returns the error → `mlog.Fatal`. Same crash, different route.
Fixing only the `cacheManagedCategoryIDs` read leaves this live.

### 5b. Store-layer read that cannot currently opt in

`SqlPropertyGroupStore.Get(name string)` (`sqlstore/property_group_store.go:75`,
reading via `GetReplica()` at `:86`) takes **no context**, so it has no way to request
master routing. This matters in two places:

- `cacheManagedCategoryIDs` → `GetPropertyGroup` → `groupStore.Get` → replica.
- `SqlPropertyGroupStore.Register` (`:24`) inserts with `ON CONFLICT (Name) DO NOTHING`
  on master, then on `rowsAffected == 0` falls back to `s.Get(group.Name)` — a replica
  read of a row another cluster node may have just written. Verified still present on
  master at `:55-57`.

---

## 6. Fix options

### Option A — remove the read-back entirely (smallest; fixes the reported crash)

`SqlPropertyFieldStore.Create` calls `field.PreSave()`, which populates `field.ID`
(`model/property_field.go:147-155`), and returns that same pointer. So by the time the
migration reaches `cacheManagedCategoryIDs()`, **it already holds `group.ID` and the
created field's ID in memory.** The read-back is pure redundancy on the create path.

```go
createdField, err := s.propertyService.CreatePropertyField(nil, field)
// ... existing create-failed guard ...
s.Channels().managedCategoryGroupID = group.ID
s.Channels().managedCategoryFieldID = createdField.ID
return nil
```

- ~8 lines, no interface change, no codegen.
- **Correct by inspection** — no read means no race, nothing to simulate. This is a
  meaningful advantage given §8.
- Backports to `release-11.7` cleanly.
- **Insufficient alone:** the `data != nil` short-circuit and the create-failed guard
  still read the replica.

### Option B — route migration reads to master (the correct fix)

Field side, mirroring the existing `getPropertyFieldFromMaster` precedent:

```go
func (ps *PropertyService) getPropertyFieldByNameForObjectTypeFromMaster(groupID, targetID, objectType, name string) (*model.PropertyField, error) {
	return ps.fieldStore.GetFieldByNameForObjectType(store.WithMaster(context.Background()), groupID, targetID, objectType, name)
}
```

plus a public wrapper alongside `GetPropertyFieldByNameForObjectType`
(`property_field.go:516`) that preserves the existing `runPostGetPropertyField(rctx, field)`
call. ~15 lines.

Group side — pick one:

- **Add a `ctx` param to `PropertyGroupStore.Get`.** Touches `store/store.go`,
  `sqlstore/property_group_store.go`, and the generated `retrylayer`, `timerlayer`,
  and `storetest/mocks`. The generated layers are produced by `make store-layers`
  (`server/Makefile:370`) — regenerate, do not hand-edit. Large-looking diff, small
  real effort. The field store already went through exactly this refactor; copy its
  shape.
- **Or use the existing group cache.** `RegisterPropertyGroup` already populates
  `ps.groupCache`, and `ps.Group(name)` checks that cache before hitting the store
  (`app/properties/property_group.go`). Swapping `GetPropertyGroup` → `Group` in
  `cacheManagedCategoryIDs` makes the create path a cache hit with zero query. Does
  not help the short-circuit path, but that path has no same-process write to race.

### Option C — bracket the whole app-migration block

`LockToMaster()` / `UnlockFromMaster()` already exist
(`sqlstore/store.go:737` and `:741`) and `GetReplica()` honors the flag at `:474`.
Wrapping the `m1` loop in `runAppMigrations` fixes every read-your-own-write in every
current *and future* app migration in ~4 lines.

Genuinely attractive as the systemic answer, with two caveats a reviewer will raise:
`lockedToMaster` is an unsynchronized `bool` (fine at startup, before request serving
begins, but expect the question), and it masks individual bugs rather than fixing them.

### Recommendation

- **`release-11.7` backport:** Option A plus the field half of Option B. Small,
  obviously safe, no codegen, no interface churn.
- **`master`:** Option C *or* the group `ctx` param, applied across all four migrations
  in §5a — this pattern is replicating with each new property-backed feature, so a
  systemic fix has compounding value here.

---

## 7. Do not do these

1. **Do not change the shared `getPropertyFieldByNameForObjectType` /
   `getPropertyFieldByName` to use master.** They are on hot runtime paths
   (`api4/custom_profile_attributes.go`, `app/access_control.go`, `app/board.go`,
   plugin API, and others). Forcing those to the writer is a real performance
   regression and defeats the point of configuring replicas. The master-routed variant
   must be a **separate entry point**, as `getPropertyFieldFromMaster` already is.
2. **Do not "fix" migration `000195_threadmemberships_cleanup_v2`.** It is the trigger,
   not the cause. See §2.
3. **Do not add a sleep or a bounded retry loop as the primary fix.** That trades a
   fast crash for a slow crash and leaves the race intact. (A retry may be reasonable
   as defense-in-depth *after* master routing, but not instead of it.)
4. **Do not hand-edit `retrylayer.go`, `timerlayer.go`, or `storetest/mocks/`.**
   They are generated — `make store-layers`.
5. **Do not advise clearing replicas via `MM_SQLSETTINGS_DATASOURCEREPLICAS=""`** in
   any docs or release note you write. `applyEnvKey` does `strings.Split(value, " ")`
   for slice fields (`config/environment.go:80`), so an empty value yields a
   one-element slice containing an empty string. `len() == 1`, so `GetReplica()` will
   *not* fall back to master and will instead try to use an empty DSN. Point the
   replica DSN at the writer endpoint, or remove the entry from config, instead.

---

## 8. Testing reality — read before writing tests

**A red-to-green reproduction test is not achievable with the current harness.
Budget accordingly.**

The bug requires the read to land on a physically separate database that is behind the
writer. In the test harness that never happens:

- `storetest.databaseSettings()` (`store/storetest/settings.go`) hardcodes
  `DataSourceReplicas: []string{}` with no env override.
- `GetReplica()` short-circuits to `GetMaster()` when that list is empty.

So in every test the write and the read-back go through the *same handle to the same
database*. The row is always present. **A test of these migrations is green before the
fix and green after** — it exercises the code and proves nothing about what changed.

Note also that `MainHelper.ToggleReplicasOff()` / `ToggleReplicasOn()`
(`channels/testlib/helper.go:215` and `:229`) look like they help, but the list they
save and restore is itself empty, so "replicas on" and "replicas off" are the same
state under CI defaults.

Real coverage would need a live primary/replica pair in CI, a proxy that withholds
replication, or an injected fake replica handle serving deliberately stale reads. None
exists; building any of it is a much larger project than this fix.

### What you *can* and should deliver

A **regression guard**, not a reproduction. Using the store mocks
(`store/storetest/mocks/PropertyFieldStore.go`), assert that the migration's read is
issued with a master-routed context:

```go
// pseudocode
propertyFieldStore.On("GetFieldByNameForObjectType",
    mock.MatchedBy(func(ctx context.Context) bool { return store.HasMaster(ctx) }),
    ...,
).Return(field, nil)
```

This pins the intent so a later refactor cannot silently revert it. Be explicit in the
PR description that it is a guard, not a reproduction — do not let it be described as
proof the race is closed.

**Because no test can carry the argument, the PR description has to.** State the
mechanism, cite the file:line pairs above, and explain why the fix is correct by
inspection. This is also the strongest argument for Option A on the backport: "there is
no read, therefore there is no race" is verifiable by a reviewer without trusting any
claim about replication behavior.

---

## 9. Acceptance criteria

- [ ] `cacheManagedCategoryIDs` no longer performs a replica read of a row written by
      the same process — via Option A, Option B, or Option C.
- [ ] The `data != nil` short-circuit path is also safe (see §3, "Restarting works,
      but nothing in the code guarantees it").
- [ ] The `RegisterPropertyGroup` → `Get` conflict fallback
      (`property_group_store.go:55-57`) no longer reads a stale replica.
- [ ] On `master`, the `SearchPropertyFields`-based existence checks in the Content
      Flagging, Boards, and shared-helper migrations (§5a) are covered too.
- [ ] No hot runtime read path was switched to master routing (§7.1).
- [ ] Generated store layers regenerated via `make store-layers`, not hand-edited.
- [ ] A regression guard exists per §8, correctly labelled.
- [ ] PR description carries the correctness argument explicitly (§8).
- [ ] Backport to `release-11.7` prepared, or a reason recorded for not doing so.
- [ ] A release note is drafted — the customer specifically flagged its absence. It
      should name the affected configuration (licensed + read replicas configured) so
      other operators can assess exposure before upgrading.

---

## 10. Reproduction, if you want one manually

Not required for the fix, but the most direct path:

1. Postgres primary + streaming replica; configure both in `SqlSettings.DataSource`
   and `SqlSettings.DataSourceReplicas`. Apply an Enterprise license — without one,
   `GetReplica()` returns master and nothing reproduces.
2. Start from a 10.11.x schema with enough data that migration 195 is expensive.
3. Artificially delay the replica (`pg_wal_replay_pause()` on the standby, or a proxy
   holding back the stream).
4. Start the 11.7.x server and let app migrations run.

Expected: `mlog.Fatal` with `failed to get managed category field: ... sql: no rows in
result set`. Confirm on the primary via `psql` that the `PropertyFields` row exists.

---

## 11. Upstream tracking

Searched `mattermost/mattermost` issues and PRs for this — **found nothing tracking
it**. Assume no existing ticket; file one, or link the internal ticket here when it
exists.
