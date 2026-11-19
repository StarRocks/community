# [Design] StarRocks: MV Registration in Iceberg Catalog

**Author**: Manish Malhotra
**Last Updated**: 2026-08-23

---

## Context: Iceberg Materialized View Spec

The Apache Iceberg community has been formalizing a [Materialized View specification](https://github.com/apache/iceberg/pull/9830) that extends the existing Iceberg View spec with two additions:

- **`storageTable`** in `ViewVersion` — a pointer to a real Iceberg table holding precomputed MV rows. Non-null = materialized view. Engines can read data directly.
- **`RefreshState`** in the storage table's snapshot summary — records which source table snapshot IDs were current at refresh time, enabling external freshness checks.

Trino has already begun implementing this spec ([trinodb/trino#28866](https://github.com/trinodb/trino/issues/28866)). Spark support is expected to follow.

**This work is designed to be fully aligned with that spec** and structured in phases that track its maturity:

- **Phase 1 (this PR)** — Metadata mirror: MV visible in IRC/HMS, discoverable, AuthZ via Ranger. Iceberg lib: current version. BE change: No.
- **Phase 2a** — Spec compliance: `storageTable` pointer, `RefreshState` tracking. Iceberg lib: #9830 in a release. BE change: No.
- **Phase 2b** — Data interoperability: export MV data to `__storage` Iceberg table post-refresh. Iceberg lib: #9830 in a release. BE change: No.
- **Phase 3** — Native dual-write: SR refresh writes Iceberg format directly. Iceberg lib: #9830 + BE work. BE change: Yes.

**Phase 1 is the right first step regardless of Iceberg MV spec maturity.** Central metadata registration in IRC/HMS is a prerequisite for all subsequent phases — the view entry, lifecycle hooks, and `IcebergMVRegistrar` infrastructure built here carry forward unchanged into Phase 2a and 2b. Meanwhile, the Iceberg MV spec will continue to mature and become available in the Iceberg client library. Phases 2a/2b can be picked up as a follow-on PR with no rework of Phase 1.

**Phase 3 (native dual-write) is deferred** — it requires SR's BE refresh pipeline to emit Iceberg-format (Parquet + manifests) files in addition to native tablets, which is a significant BE change. Phases 2a/2b achieve full spec compliance and cross-engine data interoperability via a post-refresh export without touching the BE.

---

## Overview

StarRocks Materialized Views (MVs) are currently registered only in StarRocks' internal native catalog (`default_catalog`). This makes them invisible to external systems — data registries, governance tools, and other query engines — that discover datasets through an Iceberg-compatible metastore (HMS or Iceberg REST Catalog / IRC).

This document proposes mirroring MV registrations into the configured Iceberg catalog so that the MV's SQL definition and metadata are discoverable by any Iceberg-aware system, while StarRocks continues to own and serve the MV natively.

---

## Problem Statement

In deployments where Iceberg is used as the central metadata standard, all datasets are expected to be registered in the central Iceberg metastore (HMS or IRC) so that the data registry can discover them, build lineage graphs, and surface them to data consumers.

StarRocks MVs are a first-class asset — they represent curated, pre-aggregated views of the data that teams depend on for reporting and analytics. However, because they live only in `default_catalog`, they are invisible to the data registry. Teams discover them only by querying StarRocks directly, and there is no central record of what an MV computes, how it is refreshed, or what base tables it depends on.

---

## Goals

- Register StarRocks MVs in the Iceberg catalog (HMS or IRC) at creation time so they are discoverable by external systems.
- Store the complete MV SQL definition (including REFRESH, PARTITION BY, DISTRIBUTED BY, ORDER BY, and PROPERTIES clauses) in the registration so consumers understand what the MV computes.
- **Use session context to determine the registration target** — the catalog and namespace the user is working in when issuing `CREATE MATERIALIZED VIEW` determines where the view is registered in IRC/HMS. No SQL scanning or per-MV config needed. Optional global override via `iceberg_mv_sync_catalog` for admin/migration scenarios.
- Support both HMS-backed and REST Catalog (IRC) backed Iceberg catalogs with a single implementation.
- Support **BEST_EFFORT** mode (default): IRC failure logs a warning, `CREATE MV` still succeeds. And **STRICT** mode: IRC failure on CREATE rolls back the SR DDL.
- Make the feature opt-in via an `iceberg_mv_sync_enabled` config flag.
- **Restrict to shared-data (S3) mode only** — no-op in shared-nothing mode.

## Non-Goals (Phase 1)

- **Shared-nothing mode** — MV data lives on local BE disks and is not accessible to external engines. IRC/HMS registration provides no value in this mode.
- **Making MV data directly readable by Spark or Trino** — SR data is in native table format. Cross-engine data interoperability is addressed in Phase 2b via post-refresh export to an Iceberg storage table.
- **Setting the `storageTable` pointer in the Iceberg View entry** — this requires the #9830 Iceberg library and is Phase 2a work.
- Extending `SHOW MATERIALIZED VIEWS` to show IRC-registered MVs.
- Automatic MV recreation from IRC/HMS on SR metadata loss (Phase 1.5 follow-on).
- Staleness tracking or freshness signals (`last_refresh_time` property) — deferred; no external consumer depends on it yet and it adds an IRC round-trip on every refresh.
- IRC namespace auto-creation — if the target namespace does not exist in IRC/HMS, `CREATE MV` fails with the same error as any other IRC operation. Operator pre-creates namespaces in IRC as needed.

---

## Background

### Iceberg View Specification

The Iceberg spec defines a **View** object that stores:

- One or more SQL representations, each with a `sql` string and a `dialect` tag (e.g. `"starrocks"`, `"spark"`, `"trino"`).
- A schema (output columns).
- A properties map for arbitrary key-value metadata.
- An optional `storageTable` pointer — when non-null, the view is a Materialized View with a backing table.

Iceberg Views are supported in both HMS and IRC today, and StarRocks' Iceberg connector already implements view create/drop/load operations against both backends via the `IcebergCatalog` interface.

### How Iceberg Views Are Stored in HMS

When a view is created via the Iceberg HMS catalog, Iceberg:

1. Writes a **metadata JSON file** to S3 or HDFS containing the full view definition.
2. Creates a `VIRTUAL_VIEW` entry in HMS with `parameters["table_type"] = "ICEBERG-VIEW"` and `parameters["metadata_location"]` pointing to the metadata file on S3/HDFS.

This means the full MV DDL (which can be several KB) lives in the metadata file, not in HMS table properties — so there are no HMS field size constraints.

### SR MV DDL Reconstruction

StarRocks already has a method `MaterializedView.getMaterializedViewDdlStmt()` that generates the complete, replay-ready `CREATE MATERIALIZED VIEW` DDL string from the in-memory MV object. This is the same string returned by `SHOW CREATE MATERIALIZED VIEW`. It includes all clauses: REFRESH, PARTITION BY, DISTRIBUTED BY, ORDER BY, and PROPERTIES.

### How Trino Does It

Trino's Iceberg connector stores MV definitions in HMS as a `VIRTUAL_VIEW` entry with `viewExpandedText = "/* Presto Materialized View */"` as the type discriminator, and encodes the full MV definition as a Base64 JSON string in `viewOriginalText`. The JSON includes the original SQL, output columns, grace period, and staleness behavior. The backing data lives in a separate Iceberg table.

This doc proposes a simpler version of the same pattern for StarRocks.

### Why the Iceberg Entry Is a Mirror, Not a Replacement

A natural question is: why not store the MV definition only in the Iceberg catalog and eliminate the SR-internal BDB entry? This is not feasible because SR's BDB metadata contains two constructs that have no Iceberg equivalent.

> **Note on BDB persistence**: In Kubernetes deployments, the FE's BDB journal is mounted on a persistent volume (e.g., cloud block storage) at `/opt/starrocks/fe/meta`, so it survives FE pod restarts. It is not ephemeral local storage, but it is SR-proprietary and not readable by any external system.

1. **`shardGroupId` per partition (shared-data mode)**

   In shared-data mode, MV data lives on S3/HDFS. However, the path to that data is not a fixed S3 prefix. It is mediated by StarOS **shard groups** (identified by `shardGroupId`), and StarOS resolves the shard group to the actual S3 object locations at query time. Without the `shardGroupId` stored in BDB, SR cannot locate the MV data even though it sits on S3. The Iceberg spec has no concept of shard groups — there is no place to store this pointer in an Iceberg View entry.

2. **`AsyncRefreshContext` — incremental refresh state**

   `AsyncRefreshContext` tracks the base table partition versions that were current at the last successful refresh. This is what allows SR to do incremental (rather than full) refreshes: on each refresh cycle, SR compares the current base table version map against the stored baseline to determine which partitions have changed and need recomputation. This state is SR-internal; the Iceberg spec has no staleness or version-tracking model at the view level.

Because neither `shardGroupId` nor `AsyncRefreshContext` can be expressed in an Iceberg View entry, SR's BDB metadata must be retained as the source of truth. The Iceberg registration is therefore **additive** — a discoverability mirror — rather than a replacement of SR's internal metadata store.

---

## Design

### Storage Format

When a StarRocks MV is created, an Iceberg View entry is created in the configured catalog with the following content:

- **`sql`** — Full `CREATE MATERIALIZED VIEW` DDL. Source: `mv.getMaterializedViewDdlStmt(true)`.
- **`dialect`** — `"starrocks"`. Marks this as SR-specific SQL.
- **`schema`** — MV output column schema. Converted to Iceberg Schema.
- **`properties["starrocks.mv"]`** — `"true"`. Discriminates MV from regular view.
- **`properties["comment"]`** — MV comment, if present.

> **Open design question**: The current implementation stores the full DDL in `sql()`. The original design intent was to store SELECT-only in `sql()` (for view expansion compatibility) and the full DDL in `properties["starrocks.mv.ddl"]`. The transparent rewrite path (`SELECT FROM sync_catalog.db.mv_name` → native MV) has not been validated end-to-end with full DDL in `sql()`. See Open Question #5.

**`sql` field** stores the complete replay-ready DDL:

```sql
CREATE MATERIALIZED VIEW `mv_sales_by_region` (
  `region` VARCHAR(64),
  `total_sales` DECIMAL(18, 2)
)
COMMENT "Daily sales aggregated by region"
PARTITION BY RANGE(order_date) ()
DISTRIBUTED BY HASH(region) BUCKETS 8
ORDER BY (region)
REFRESH DEFERRED ASYNC START("2026-01-01 00:00:00") EVERY(INTERVAL 1 DAY)
PROPERTIES (
  "replication_num" = "3",
  "partition_ttl" = "30 DAY"
)
AS SELECT region, sum(amount) AS total_sales
   FROM orders o JOIN dates d ON o.date_key = d.date_key
   GROUP BY region;
```

#### Why Full DDL in `sql()`, Not Select-Only?

The current implementation stores the full `CREATE MATERIALIZED VIEW` DDL in `sql()` because it is the single source of truth for BDB recovery (Phase 1.5 / Phase 2) — no separate property needed. It also simplifies the write path.

**Known risk**: SR's view expander expects a `SELECT` in the view body. If a user queries `SELECT * FROM sync_catalog.db.mv_name`, SR loads the view from IRC and uses `sqlFor("starrocks")` as the view body. If that body starts with `CREATE MATERIALIZED VIEW`, view expansion will fail. Whether `MaterializedViewRewriter` intercepts before expansion (making this safe) is the key open question — see Open Question #5 below. If expansion fails, the fix is straightforward: store SELECT-only in `sql()` and move the full DDL to `properties["starrocks.mv.ddl"]`.

#### Why Not Decompose Into Individual Properties?

One alternative is to store each MV attribute as a separate Iceberg View property (e.g. `starrocks.mv.refresh.type`, `starrocks.mv.partition.by`, etc.). This is rejected for the following reasons:

- It requires maintaining a mapping between SR internal objects and property strings.
- It requires a custom reader to reconstruct the MV from properties.
- It creates a fragile coupling — any new SR MV clause requires a new property key.
- The existing `getMaterializedViewDdlStmt()` already produces the correct, canonical DDL string. Storing it whole in a single field is simpler and more robust.

#### MV vs Regular View Disambiguation

Any system loading views from the Iceberg catalog can distinguish SR MVs from regular Iceberg Views using two signals:

- **`properties["starrocks.mv"]`** — absent on regular Iceberg views; `"true"` on SR Materialized Views.
- **`dialect`** — `"spark"`, `"trino"`, etc. on regular views; `"starrocks"` on SR MVs.
- **`storageTable()`** — null on regular views; non-null on SR MVs (future: Phase 2a).

For now, `properties["starrocks.mv"] = "true"` is the primary discriminator.

#### What External Systems Can Do With It

- **Data Registry** — Sees: MV exists, SQL definition, base tables via SQL. Can do: lineage, discoverability, impact analysis.
- **Trino / Spark / Flink** — Sees: view with dialect `"starrocks"`. Can do: read SQL for reference, cannot execute directly.
- **StarRocks** — Sees: full MV in `default_catalog`. Can do: serve queries natively, transparent query rewrite.
- **Governance tools** — Sees: MV metadata in metastore. Can do: classify, tag, audit.

#### Behavior of Other Query Engines (Phase 1)

Since the view has no `spark`, `trino`, or `flink` dialect SQL representation, other engines will fail gracefully if a user attempts to query the view directly. The failure is scoped to that single statement — it does not crash the session or corrupt the catalog.

- **Spark** — Catalog listing (`SHOW VIEWS`): works, lists view name and properties. Direct query (`SELECT * FROM mv`): `AnalysisException: View written in unsupported dialect`.
- **Trino** — Catalog listing: works, view appears in `information_schema.views`. Direct query: `TrinoException: View is not compatible`.
- **Flink** — Catalog listing: works, catalog metadata load succeeds. Direct query: `ValidationException` at query planning.

**Mitigation**: `properties["starrocks.mv"] = "true"` is the signal for well-behaved consumers (data registry, governance tools) to treat this entry as an SR-native asset rather than a directly queryable view. Tools should check this property before attempting SQL expansion.

In Phase 2, once `storageTable` is set (apache/iceberg#9830), Spark and Trino bypass SQL expansion entirely and read from the backing Iceberg table — the dialect mismatch becomes irrelevant for query execution.

### Authorization Model

#### How AuthZ Works for SR MVs via IRC

When SR receives `SELECT * FROM my_iceberg_catalog.sales_db.mv_sales`, the analysis phase resolves the view from IRC — this is the enforcement point:

```
SELECT * FROM my_iceberg_catalog.sales_db.mv_sales
    ↓
SR analysis: resolves view from IRC
    ↓
IRC native ACL checks user access to mv_sales view → allow/deny
    ↓ (if allowed)
IRC returns view sql() = "SELECT region, sum(amount) FROM ... GROUP BY region"
    ↓
SR expands view → MV rewriter matches → serves precomputed result
```

Enforcement happens at the IRC layer — the same point as for regular Iceberg tables. MVs registered as Iceberg views are governed identically to tables with no special-casing.

#### Identity Model

Users authenticate via a JWT issued by the organization's identity provider (e.g. OIDC), which maps to LDAP/directory groups. SR role grants are made to LDAP groups — no per-user SQL management needed. The only deployment-time setup required:

```sql
-- One-time at cluster init: grant IRC catalog access to the analyst LDAP group
CREATE ROLE analyst;
GRANT USAGE ON CATALOG <irc_catalog_name> TO ROLE analyst;
GRANT ROLE analyst TO GROUP '<ldap-analyst-group>';
```

**Internal operations** (MV refresh, IRC sync retries, startup reconciliation) run under the SR root identity and have full `default_catalog` access.

#### `default_catalog` Access Control

`default_catalog` is open by default in SR — catalog-level USAGE is defined in the privilege model but not enforced at the query path. In Phase 1, `default_catalog` remains accessible to users who know to query it directly. Closing this bypass is addressed in **Phase 1b** (see Future Work) via code-level enforcement in SR's query analysis and `ShowExecutor` paths, with an internal access flag to allow the MV rewriter and refresh paths to continue working.

### HMS Compatibility

HMS is Thrift-protocol compatible with the standard Hive Metastore protocol, which means:

- SR's `IcebergHiveCatalog` connects to HMS via the existing Thrift client — no new integration code needed.
- HMS treats table properties as opaque key-value pairs. The `parameters["metadata_location"]` pointer that Iceberg stores is passed through without inspection — HMS does not need to understand Iceberg view format.
- Iceberg reads/writes the actual view metadata as a JSON file on S3/HDFS directly. HMS only stores the path pointer, never the JSON content.
- If SR is already using HMS as an Iceberg catalog for reading tables, the Thrift connection and auth are already configured. Phase 1 view CRUD (create, replace, drop) uses the same connection — no HMS-side changes required.

StarRocks' `IcebergCatalog` interface abstracts both HMS and REST Catalog backends:

```
IcebergCatalog (interface)
├── IcebergHiveCatalog   ← HMS backend (Thrift)
│   └── delegates to Apache Iceberg HiveCatalog
├── IcebergRESTCatalog   ← IRC backend (REST)
│   └── delegates to Apache Iceberg RESTCatalog
```

Both implement `getViewBuilder()`, `dropView()`, and `getView()`. Registration code written against the `IcebergCatalog` interface works for both HMS and IRC with no special-casing.

### Lifecycle Events

Three MV lifecycle events require synchronization with the Iceberg catalog:

**CREATE MATERIALIZED VIEW**
- Trigger: After SR successfully registers the MV in `default_catalog`.
- Action: Call `catalog.getViewBuilder().withQuery(...).withProperty(...).createOrReplace()`.
- Hook point: `LocalMetastore.createMaterializedView()`, after `onCreate()` succeeds.

**DROP MATERIALIZED VIEW**
- Trigger: After SR successfully drops the MV from `default_catalog`.
- Action: Call `catalog.dropView()`.
- Hook point: `LocalMetastore.dropMaterializedView()`, after `db.dropTable()` succeeds.

**ALTER MATERIALIZED VIEW** (refresh scheme, properties change, or rename)
- Trigger: After SR successfully alters the MV and `AlterMVJobExecutor.process()` returns.
- Hook point: `LocalMetastore.alterMaterializedView()`, after `AlterMVJobExecutor.process()`.
- **RENAME**: `IcebergMVRegistrar.compensateDrop(oldName)` then `registerOrReplace(newName)`.
- **All other ALTER types** (refresh scheme, properties): `IcebergMVRegistrar.registerOrReplace()`.
- **ALTER MV ACTIVE/INACTIVE**: skipped — SR-internal state, no IRC representation.
- IRC sync is always best-effort for ALTER — the SR change is already committed to BDB.
- **Concurrency note**: SR serializes ALTER and async refresh per MV via a task lock (`TaskManager` / `MvTaskRunContext`). If a refresh is running when ALTER is issued, the ALTER waits until the refresh task completes before committing. The `IcebergMVRegistrar` hook fires after the ALTER commits — HMS/IRC always receives the post-alter definition, never a partial or in-flight state. No special handling needed in `IcebergMVRegistrar` for this case.

In all three cases, failure behavior depends on `iceberg_mv_sync_mode`:

- **BEST_EFFORT** (default): IRC failure logs a warning; the SR DDL operation succeeds. For ALTER and DROP, the failed op is enqueued for async retry (Phase 1.5a). Any divergence that survives all retries is fixed by startup reconciliation (Phase 1.5).
- **STRICT**: IRC failure on CREATE triggers a compensating `db.dropTable()` at the `LocalMetastore` call site, then `DdlException` is returned to the client. For DROP and ALTER, STRICT is treated as BEST_EFFORT (the SR operation cannot be undone).

### Flow Diagrams

#### CREATE MATERIALIZED VIEW Flow

```
SQL Client
    |
    |  CREATE MATERIALIZED VIEW mv_sales_by_region
    |  REFRESH ASYNC EVERY(INTERVAL 1 DAY)
    |  AS SELECT region, sum(amount) FROM orders GROUP BY region;
    |
    v
StarRocks FE  (LocalMetastore.createMaterializedView)
    |
    |-- 1. Parse + analyze DDL
    |-- 2. Build MaterializedView object
    |-- 3. onCreate() → persist MV in BDB (default_catalog)
    |       |
    |       |  MV now live in SR. Queries can be served.
    |
    |-- 4. IcebergMVRegistrar.registerOrReplace(ctx, mv, dbName)
            |
            |-- isActive() guard (iceberg_mv_sync_enabled + catalog configured)
            |-- resolveIcebergMetadata(syncCatalog)
            |-- buildViewDefinition:
            |     inlineViewDef = mv.getMaterializedViewDdlStmt(true)  → full DDL
            |     properties["starrocks.mv"] = "true"
            |
            v
        HMS / IRC  (via IcebergHiveCatalog or IcebergRESTCatalog)
            |
            |-- IcebergMetadata.syncMVView() → icebergCatalog.createView(replace=true)
            |-- write metadata JSON → S3/HDFS
            |-- store metadata_location in HMS/IRC
            |
            v
        [BEST_EFFORT] failure → log warning, continue
        [STRICT]      failure → db.dropTable(mvName) at LocalMetastore call site + DdlException
    |
    v
SQL Client  ← CREATE MV succeeded (or DdlException in STRICT mode)
```

#### SELECT Flow — MV Transparent Rewrite

This is the query path when a general user queries the MV via the external catalog. The MV rewrite happens transparently — the user gets precomputed results from SR's native MV without knowing `default_catalog` exists.

```
SQL Client (analyst role — no access to default_catalog)
    |
    |  SELECT * FROM my_catalog.sales_db.mv_sales_by_region
    |
    v
StarRocks FE  (query analysis)
    |
    |-- 1. Resolve catalog: my_catalog → external (HMS/IRC)
    |
    |-- 2. Load view from HMS/IRC
    |       |
    |       |  IRC native ACL: does analyst have access to mv_sales_by_region?
    |       |  → ALLOW / DENY
    |       |
    |       v
    |   HMS / IRC returns Iceberg View:
    |       sqlFor("starrocks") = "SELECT region, sum(amount)
    |                              FROM my_catalog.sales_db.orders
    |                              GROUP BY region"
    |
    |-- 3. IcebergApiConverter.toView()
    |       extracts selectSql via icebergView.sqlFor("starrocks")
    |       → SR sees it as a regular view with a SELECT definition
    |
    |-- 4. View expansion
    |       query becomes:
    |         SELECT * FROM (
    |           SELECT region, sum(amount)
    |           FROM my_catalog.sales_db.orders
    |           GROUP BY region
    |         )
    |
    |-- 5. MaterializedViewRewriter runs
    |       scans default_catalog MVs for a matching viewDefineSql
    |       |
    |       |  match found:
    |       |  default_catalog.sales_db.mv_sales_by_region
    |       |  viewDefineSql = "SELECT region, sum(amount)
    |       |                   FROM my_catalog.sales_db.orders
    |       |                   GROUP BY region"
    |       |    ↑ same fully-qualified SELECT → MATCH
    |       |
    |       v
    |   rewrite: scan default_catalog.sales_db.mv_sales_by_region
    |             (native tablets, precomputed data)
    |
    |-- 6. Execute rewritten plan
    |       SR BEs read precomputed MV data from S3 (StarOS shard groups)
    |
    v
SQL Client  ← results (from precomputed MV, not re-aggregated from orders)
```

Key points:

- The analyst never touches `default_catalog` directly — authz is enforced at step 2 (IRC native ACL). Direct `default_catalog` access will be blocked in Phase 1b.
- The SELECT stored in `sql("starrocks")` is fully qualified with catalog prefix (e.g. `my_catalog.sales_db.orders`) — this is what makes the MV rewriter match.
- The MV rewriter is existing SR infrastructure — no changes needed for this flow. `ConnectorTblMetaInfoMgr` injects `relatedMaterializedViews` onto the external catalog `Table` objects at load time, so `MvRewritePreprocessor` finds the native MV candidate automatically via `table.getRelatedMaterializedViews()`.
- `properties["starrocks.mv.ddl"]` (the full CREATE DDL) is never read in this path — it exists only for BDB recovery (Phase 1.5 / Phase 2).

#### REFRESH Flow — Incremental Refresh + Phase 2b Export (future)

This diagram shows what happens during a scheduled or manual MV refresh. In Phase 1, HMS/IRC is not touched during refresh — the catalog entry is static (set at CREATE/ALTER time). The Phase 2b export step is shown as a future addition.

```
Refresh Trigger
    |
    |  scheduled:  REFRESH ASYNC EVERY(INTERVAL 1 DAY)  [fires automatically]
    |  manual:     REFRESH MATERIALIZED VIEW mv_sales_by_region
    |
    v
SR FE  —  RefreshMaterializedViewTask
    |
    |-- 1. Load AsyncRefreshContext  (from BDB)
    |       base table version map from last refresh:
    |       my_catalog.sales_db.orders → iceberg_snapshot_id: 12345
    |
    |-- 2. Incremental refresh check
    |       compare current snapshot IDs vs stored baseline
    |       my_catalog.sales_db.orders → current snapshot: 12350 (changed)
    |       → partitions [2026-03-20 .. 2026-03-25] changed → refresh these only
    |       → unaffected partitions skipped (no re-computation)
    |
    |-- 3. Build refresh query plan
    |       INSERT OVERWRITE INTO mv_sales_by_region PARTITION(order_date=...)
    |       SELECT region, sum(amount)
    |       FROM my_catalog.sales_db.orders
    |       WHERE order_date BETWEEN '2026-03-20' AND '2026-03-25'
    |       GROUP BY region
    |
    v
SR BEs  —  execute refresh query
    |
    |-- 4. Read base table data
    |       scan my_catalog.sales_db.orders via HMS (Iceberg scan over Thrift)
    |       HMS resolves metadata_location → reads Parquet files from S3
    |
    |-- 5. Write to MV native tablets
    |       output rows written to StarOS shard groups on S3
    |       (SR native format — not Iceberg)
    |
    v
SR FE  —  post-refresh bookkeeping
    |
    |-- 6. Update AsyncRefreshContext  (write to EditLog / BDB)
    |       record new base table snapshot IDs:
    |       my_catalog.sales_db.orders → iceberg_snapshot_id: 12350
    |       enables next refresh to be incremental from this point
    |
    |-- 7. [Phase 1] HMS/IRC not touched
    |       catalog entry (sql, properties) was set at CREATE time and is unchanged
    |       the MV is fresh; external systems see it via the existing view entry
    |
    |-- 8. [Phase 2b — future, no BE change]
    |       IcebergMVRegistrar.exportToStorageTable()
    |         INSERT OVERWRITE INTO my_catalog.sales_db.mv_sales_by_region__storage
    |         SELECT * FROM default_catalog.sales_db.mv_sales_by_region
    |         (uses SR's existing Iceberg write path — IcebergSinkOperator)
    |       update RefreshState in storage table snapshot summary:
    |         view_version_id, source snapshot IDs, refresh timestamp
    |       → Spark / Trino can now read precomputed data from __storage
    |       → RefreshState lets external engines assess freshness without querying SR
    |
    v
Refresh complete
```

Key points:

- **Phase 1**: HMS/IRC is not called during refresh. The view entry in HMS/IRC is static, updated only on ALTER. Refresh only updates SR's native tablets and `AsyncRefreshContext`.
- **`AsyncRefreshContext`** is SR-internal — it stores Iceberg snapshot IDs of base tables at last refresh time, enabling incremental (partition-level) refresh. There is no equivalent in the Iceberg spec today; it is why SR's BDB metadata cannot be replaced by IRC alone.
- **Phase 2b export** is a plain SR query (`INSERT INTO ... SELECT FROM`) — no new BE code. It runs after the native refresh completes and uses the existing `IcebergSinkOperator` write path.
- **`RefreshState`** (Phase 2b) records the same snapshot IDs that `AsyncRefreshContext` already tracks — it is a projection of SR-internal state into the Iceberg spec format for external consumption.

---

## Configuration

Three configuration fields control the feature:

```
iceberg_mv_sync_enabled  = false          (default: disabled)
iceberg_mv_sync_mode     = "BEST_EFFORT"  (default)
iceberg_mv_sync_catalog  = ""             (default: use session catalog)
```

`iceberg_mv_sync_enabled`: Master switch. Set to `true` to enable MV mirroring.

`iceberg_mv_sync_mode`: Controls behavior when catalog registration fails on CREATE MV.

- **BEST_EFFORT** (default) — CREATE MV: failure logs a warning; `CREATE MV` succeeds in SR. DROP/ALTER: always best-effort; SR operation succeeds regardless.
- **STRICT** — CREATE MV: failure rolls back SR DDL (SR DROP + `DdlException` returned to client). DROP/ALTER: always best-effort; SR DROP is already committed and cannot be undone.

**Why STRICT mode for DROP is still best-effort**: Once SR has committed the `DROP MATERIALIZED VIEW`, the MV is gone from `default_catalog`. There is no way to un-drop it if the subsequent catalog `dropView()` fails. The operator must resolve the ghost entry via startup reconciliation (Phase 1.5) or manually.

`iceberg_mv_sync_catalog`: The name of the target Iceberg catalog registered in StarRocks. Must be set for sync to be active. When set, all MVs register in this catalog regardless of session context. Session-based catalog resolution (using `ConnectContext.getCurrentCatalog()` when the config is empty) is planned as a follow-on enhancement.

All configs are mutable — changeable at runtime without restarting the FE:

```sql
ADMIN SET FRONTEND CONFIG ("iceberg_mv_sync_enabled" = "true");
ADMIN SET FRONTEND CONFIG ("iceberg_mv_sync_mode" = "STRICT");
ADMIN SET FRONTEND CONFIG ("iceberg_mv_sync_catalog" = "irc_catalog");
```

### Catalog Resolution

Target catalog is read from `iceberg_mv_sync_catalog`. `IcebergMVRegistrar.isActive()` guards all sync operations:

```java
private static boolean isActive() {
    return Config.iceberg_mv_sync_enabled
            && !Config.iceberg_mv_sync_catalog.isEmpty();
}
```

Target namespace = `mv.getDbName()` (the database the MV was created in). The namespace must pre-exist in the target catalog.

**Session-based resolution** (using `ConnectContext.getCurrentCatalog()` as the target when `iceberg_mv_sync_catalog` is empty) is a planned follow-on — see Open Question #6.

---

## Implementation

### New Files

**`IcebergMVRegistrar.java`** (`fe/fe-core/src/main/java/com/starrocks/connector/iceberg/`)

A utility class with static methods:

- `registerOrReplace(ConnectContext ctx, MaterializedView mv, String dbName)` — called on CREATE and ALTER
- `compensateDrop(ConnectContext ctx, String dbName, String mvName)` — called on DROP and ALTER RENAME (old name)
- `dbExistsInSyncCatalog(ConnectContext ctx, String dbName)` — checks namespace existence for db auto-create logic

Access path: `ConnectorMgr.getConnector(syncCatalog)` → `CatalogConnector.getNativeMetadata()` → `IcebergMetadata.syncMVView()` / `dropMVView()`.

Internally, `registerOrReplace` does:

1. Guard: `isActive()` check (`iceberg_mv_sync_enabled` + non-empty `iceberg_mv_sync_catalog`).
2. Build `ConnectorViewDefinition`: `inlineViewDef = mv.getMaterializedViewDdlStmt(true)`, `properties["starrocks.mv"] = "true"`.
3. Call `IcebergMetadata.syncMVView(ctx, viewDef)` → `icebergCatalog.createView(replace=true)`.
4. On exception:
   - **BEST_EFFORT**: logs warning, returns normally.
   - **STRICT**: throws `DdlException`. Compensating `db.dropTable()` is handled at the `LocalMetastore` call site.

**`IcebergMVRegistrarTest.java`** (`fe/fe-core/src/test/java/com/starrocks/connector/iceberg/`)

9 unit tests covering: disabled config, empty catalog (no-op), catalog not found (BEST_EFFORT swallows), non-Iceberg catalog (BEST_EFFORT swallows), happy path (`syncMVView()` called), `compensateDrop` path (`dropMVView()` called), STRICT mode throws `DdlException`.

**`LocalMetaStoreWithMaterializedViewTest.java`** (`fe/fe-core/src/test/java/com/starrocks/server/`)

5 wiring tests: DROP compensation, ALTER RENAME (drop old + register new), ALTER properties (`registerOrReplace` called), ALTER ACTIVE/INACTIVE (IRC sync skipped).

### Modified Files

- **`common/Config.java`** — Add `iceberg_mv_sync_enabled`, `iceberg_mv_sync_mode`, and `iceberg_mv_sync_catalog` fields.
- **`connector/CatalogConnector.java`** — Add `getNativeMetadata()` — returns `normalConnector.getMetadata()` directly, bypassing `CatalogConnectorMetadata` wrapper.
- **`connector/iceberg/IcebergMetadata.java`** — Add package-private `syncMVView()` and `dropMVView()`.
- **`server/LocalMetastore.java`** — Hook CREATE MV (after `onCreate()`), DROP MV (after `db.dropTable()`), and ALTER MV (after `AlterMVJobExecutor.process()`); auto-create `default_catalog` db on demand.
- **`sql/analyzer/MaterializedViewAnalyzer.java`** — Use `dbExistsInSyncCatalog()` to skip spurious db-not-found errors when sync catalog is active.

No changes to grammar, AST, parser, or `SHOW MATERIALIZED VIEWS`.

---

## Testing Plan

### Unit Tests

- `IcebergMVRegistrarTest` covers all lifecycle events and failure scenarios (see above).

### Integration Tests

**CREATE MV → verify in HMS/IRC:**

```sql
ADMIN SET FRONTEND CONFIG ("iceberg_mv_sync_enabled" = "true");

CREATE MATERIALIZED VIEW mv_sales_by_region
REFRESH ASYNC EVERY(INTERVAL 1 DAY)
AS SELECT region, sum(amount) FROM prod_catalog.sales.orders GROUP BY region;
```

Expected: target catalog auto-detected as `prod_catalog`. IRC returns the view with `dialect="starrocks"`, `starrocks.mv="true"`, and the full DDL in `sql`.

**ALTER MV → verify IRC updated:**

```sql
ALTER MATERIALIZED VIEW mv_sales_by_region REFRESH ASYNC EVERY(INTERVAL 6 HOUR);
```

Expected: IRC view `sql` field contains updated REFRESH clause.

**DROP MV → verify removed from IRC:**

```sql
DROP MATERIALIZED VIEW mv_sales_by_region;
```

Expected: IRC returns 404 for the view.

**Failure resilience**: Shut down the Iceberg catalog server, then execute `CREATE MATERIALIZED VIEW`. The DDL must succeed in SR, and a warning must appear in the FE log.

**HMS backend**: Same tests with an HMS-backed Iceberg catalog. Verify via `DESCRIBE FORMATTED` that the HMS entry has `table_type = VIRTUAL_VIEW` and `metadata_location` on S3/HDFS.

---

## Open Questions

1. **Namespace mapping** — **Resolved**: target namespace = MV's `dbName`. Target catalog = `iceberg_mv_sync_catalog` config (must be set explicitly). The `default_catalog` database is auto-created if missing; the IRC namespace must pre-exist.
2. **Multi-catalog** — **Resolved**: registration target is driven by config, not base table catalogs. Cross-catalog MVs are supported.
3. **MV rename** — **Resolved**: implemented as `compensateDrop(old)` + `registerOrReplace(new)`. Atomic `renameView()` is not used (not standardized across catalog implementations).
4. **Staleness tracking** — Deferred. Updating `last_refresh_time` on each refresh adds an IRC round-trip per refresh cycle. Deferred to Phase 2a follow-up once there is a concrete consumer.
5. **`sql()` content: SELECT-only vs full DDL** — **Open (integration test pending)**. Current implementation stores the full `CREATE MATERIALIZED VIEW` DDL in `sql()`. The original design called for SELECT-only in `sql()` so SR's view expander works correctly when a user queries `SELECT * FROM sync_catalog.db.mv_name`. Whether `MaterializedViewRewriter` intercepts before view expansion (making full DDL safe) must be validated end-to-end. If expansion fails, the fix is to store SELECT-only in `sql()` and move full DDL to `properties["starrocks.mv.ddl"]`.
6. **Session-based catalog resolution** — **Planned follow-on**. Currently `iceberg_mv_sync_catalog` must be set explicitly. The planned enhancement: when the config is empty and `iceberg_mv_sync_enabled=true`, fall back to `ConnectContext.getCurrentCatalog()` and skip if it is `default_catalog` or not Iceberg-backed.
7. **Recovery from divergence** — If an IRC entry is deleted externally, SR detects and re-registers on next leader startup (Phase 1.5 reconciliation).

---

## Future Work

### Phase 1b — `default_catalog` Access Control (hard prerequisite for IRC AuthZ)

Phase 1 registers MVs in IRC and establishes it as the discovery and authz boundary. But IRC authz only holds if users cannot bypass it by querying `default_catalog` directly. Without Phase 1b, registering MVs in IRC does not actually solve the MV authz problem — a user who knows `default_catalog.db.mv_name` can still access the MV without going through IRC. Phase 1b is therefore a hard prerequisite for IRC to be the authoritative authz boundary, not optional hardening.

**Code-level enforcement is confirmed implementable** — all three enforcement hooks exist in the codebase with clean insertion points.

**The Core Tension**

The same `default_catalog` MV access must behave differently depending on the caller: internal SR paths (MV rewriter, refresh tasks) need unrestricted access, while direct user queries against `default_catalog` must be blocked once `iceberg_mv_sync_enabled=true`. The enforcement design distinguishes these callers via an internal bypass flag rather than a blanket catalog-level check.

**Implementation**

1. **Query analysis** (`QueryAnalyzer.java`, `resolveTable()`): `TableName.getCatalog()` preserves the user-supplied catalog before `normalization()`. When `iceberg_mv_sync_enabled=true` and the user-supplied catalog is `default_catalog` and the resolved object is an MV, throw `AccessDeniedException` before normalization. The rewriter sets the catalog programmatically after parse — user-supplied vs. internally-set catalog is distinguishable at this point.
2. **`ShowExecutor`** (`visitShowMaterializedViewStatement()`): already has an authz filter loop using `Authorizer.checkAnyActionOnMaterializedView()`. Add the `default_catalog` MV filter in the same loop. `SHOW CREATE MATERIALIZED VIEW` handler follows the same pattern.
3. **Internal bypass flag** (`ConnectContext.bypassAuthorizerCheck`): a boolean already exists for this exact pattern. The MV rewriter and refresh paths set this before accessing `default_catalog` internally — the check at points 1 and 2 skips when the flag is set. Root/admin identity always bypasses regardless.

### Staleness Tracking

On each successful MV refresh, update the Iceberg view properties with:

- `starrocks.mv.last_refresh_time` — timestamp of last successful refresh
- `starrocks.mv.last_refresh_state` — SUCCESS / FAILED

This enables external freshness checks without querying SR.

### Full Iceberg MV Spec Compliance

**Phase 2a — Spec Metadata (no BE change, requires #9830 in lib)**

Once SR upgrades to an Iceberg library version that includes [apache/iceberg#9830](https://github.com/apache/iceberg/pull/9830):

1. Create `<mv_name>__storage` as a real Iceberg table in IRC (schema matches MV output columns).
2. Set `storageTable` in the Iceberg View entry to point to that table.
3. After each MV refresh, write `RefreshState` into the storage table's snapshot summary, recording the Iceberg snapshot IDs of base tables that were current at refresh time.

This makes SR MVs fully spec-compliant — external engines can assess freshness without querying SR. The `storageTable` pointer exists; data population follows in Phase 2b.

All Phase 1 metadata carries forward unchanged — `starrocks.mv=true`, the SELECT in `sql()`, the DDL in `properties["starrocks.mv.ddl"]`. Phase 2a is purely additive.

**Phase 2b — Data Interoperability via Post-Refresh Export (no BE change)**

After each SR refresh completes, FE triggers an export query:

```sql
INSERT OVERWRITE INTO my_catalog.db.mv_sales_by_region__storage
SELECT * FROM default_catalog.db.mv_sales_by_region
```

SR already has Iceberg write support (`IcebergSinkOperator`) and can query its own native MVs. This export runs as a regular SR query — no new BE code required.

Result: Spark and Trino can read precomputed MV data directly from the `__storage` Iceberg table, with `RefreshState` telling them how fresh the data is.

**Trade-offs vs Phase 3:**

- **BE changes** — Phase 2b (export): None. Phase 3 (native dual-write): Major.
- **Freshness lag** — Phase 2b: refresh time + export time. Phase 3: near-zero.
- **S3 cost** — Both: 2x (native + Iceberg).
- **Consistency** — Phase 2b: best-effort (export may lag). Phase 3: atomic per refresh.
- **Timeline** — Phase 2b: follow-on PR. Phase 3: long-term.

Phase 2b achieves cross-engine data interoperability at a fraction of the engineering cost of Phase 3. For batch analytics use cases, the export lag is acceptable.

### Phase 3 — Native Dual-Write (requires BE change, long-term)

The BE refresh pipeline writes MV output as both SR native tablets (for SR query serving) and Iceberg-format Parquet files (for the `__storage` table). Eliminates the export lag and does both S3 writes atomically within the refresh transaction.

This is significant BE work — the tablet sink must be extended to emit Iceberg manifests and commit to the Iceberg storage table's snapshot log. Deferred until Phase 2b proves the value and the Iceberg MV spec stabilizes across engines.

### SHOW MATERIALIZED VIEWS Extension

Extend `SHOW MATERIALIZED VIEWS` to optionally include MVs registered in the configured Iceberg catalog, with a `source` column indicating `"native"` or `"iceberg"`.

### Startup Reconciliation and Automatic MV Recovery

**Why This Is Needed**

In shared-data mode, Iceberg table schemas are stored in IRC and survive FE metadata loss. SR MVs are not — they live only in BDB (on a persistent volume at `/opt/starrocks/fe/meta/bdb/`). If BDB is wiped (volume deleted, volume corrupted, or accidental `meta_dir` wipe), all MV definitions, `AsyncRefreshContext`, and `shardGroupId` pointers are lost. The Iceberg base tables are intact in IRC; only SR's MV layer is gone.

With the IRC mirror in place, the full `CREATE MATERIALIZED VIEW` DDL is stored in IRC. This makes automated recovery possible without operator intervention.

**Phase 1.5 — Divergence Reconciliation (follow-on to this PR)**

On FE leader startup, `IcebergMVRegistrar.reconcile()` runs once to fix drift between SR and IRC caused by transient IRC failures during normal operations:

```
For each view in IRC with starrocks.mv=true:
  exists in default_catalog?
    YES, DDL matches   → no-op
    YES, DDL differs    → createOrReplace() in IRC  (stale DDL from failed ALTER)
    NO                  → registerOrReplace()        (missed CREATE, fill the gap)

For each MV in default_catalog:
  exists in IRC?
    NO  → registerOrReplace()             (missed CREATE, fill the gap)
    YES, but DDL differs → createOrReplace()

Cleanup: IRC entries with starrocks.mv=true but no matching MV in default_catalog
  → dropView() from IRC  (ghost from failed DROP cleanup)
```

This is cheap (one IRC list call + diff) and handles all divergence cases.

**Phase 1.5a — Async Retry for Failed ALTER/DROP Sync (follow-on to this PR)**

For CREATE, STRICT mode provides a rollback guarantee. For ALTER and DROP, the SR operation is already committed to BDB when IRC sync runs — there is nothing to roll back. Instead, failed IRC operations are retried asynchronously so transient failures (network blip, IRC temporarily unavailable) resolve without waiting for FE restart.

**Mechanism**: `IcebergMVRegistrar` owns a single-threaded `ScheduledExecutorService` (1 background thread, FE leader only). On IRC sync failure for ALTER or DROP, `IcebergMVRegistrar.scheduleRetry()` submits a delayed task:

```java
void scheduleRetry(PendingSyncOp op) {
    long delayMs = Math.min(op.retryCount * 30_000L, 300_000L); // 30s, 60s, 90s ... capped
    retryExecutor.schedule(() -> {
        try {
            op.execute();
        } catch (Exception e) {
            if (op.retryCount < MAX_RETRIES) {
                op.incrementRetryCount();
                scheduleRetry(op);
            } else {
                LOG.warn("Giving up IRC sync for {} after {} retries — Phase 1.5 reconciliation will repair", op, op.retryCount);
            }
        }
    }, delayMs, TimeUnit.MILLISECONDS);
}
```

- **1 background thread** shared across all pending retries — consistent with the `RoutineLoadTaskScheduler` pattern used elsewhere in SR FE.
- **Retry cap**: `MAX_RETRIES = 5` (configurable). After exhaustion, a warning is logged and Phase 1.5 startup reconciliation repairs the divergence on next leader startup.
- **In-memory only**: Pending retries are not persisted to BDB. If the FE restarts before retries complete, Phase 1.5 handles the remainder. This is acceptable — Phase 1.5 is cheap (one IRC list call + diff).
- **Not applicable to CREATE**: CREATE failures in BEST_EFFORT mode also log a warning, but the MV is immediately visible in SR so it will be picked up by Phase 1.5 reconciliation on the next startup gap-fill pass.

**Phase 2 — Full BDB Recovery (future)**

If BDB is wiped entirely, SR can reconstruct all MVs from IRC on leader startup:

1. Detect: IRC has `starrocks.mv=true` entries that do not exist in `default_catalog`.
2. For each missing MV (topological order — dependencies first):
   a. Pull DDL string from IRC `sql()` field.
   b. Re-execute via SR DDL executor (equivalent to operator running CREATE MV).
   c. SR creates the MV in `default_catalog` with fresh `shardGroupId`s.
   d. Enqueue for full refresh (old S3 data is orphaned; auto-vacuum reclaims it).
3. Throttle: recover N MVs at a time to avoid flooding the refresh scheduler.

**Complications to handle:**

- **MV dependency order** — Topological sort on base table references before re-executing.
- **IRC catalog not yet warmed up** — Delay reconciliation until external catalog is ready.
- **Full refresh storm** — Throttle: recover N MVs per batch with configurable interval.
- **`AsyncRefreshContext` lost** — All recovered MVs fall back to full refresh mode.
- **Old S3 data orphaned** — StarOS auto-vacuum reclaims orphaned shard groups.
- **Partial recovery failure** — Log warning per MV, continue with others — operator fixes remainder.

**Phasing summary:**

- **Phase 1 (this PR)** — Trigger: CREATE/ALTER/DROP. Builds the IRC record; foundation for all subsequent phases.
- **Phase 1b (hard prerequisite for IRC authz)** — Trigger: follow-on PR. Code-level `default_catalog` access control (`QueryAnalyzer`, `ShowExecutor`, `bypassAuthorizerCheck` flag). Without this, registering MVs in IRC does not enforce authz — users can bypass IRC by querying `default_catalog` directly.
- **Phase 1.5a** — Trigger: ALTER/DROP IRC sync failure. In-memory async retry (single background thread, up to `MAX_RETRIES`). Covers transient failures without waiting for restart.
- **Phase 1.5** — Trigger: FE leader startup. Divergence from transient IRC failures; ghost entry cleanup. Backstop for anything Phase 1.5a didn't repair.
- **Phase 2a** — Trigger: Iceberg lib upgrade (#9830). `storageTable` pointer, `RefreshState`, spec compliance.
- **Phase 2b** — Trigger: follow-on PR after 2a. Post-refresh export; Spark/Trino can read MV data.
- **Phase 2 (BDB recovery)** — Trigger: FE leader startup + BDB wipe detected. Full MV reconstruction from IRC DDL.
- **Phase 3** — Trigger: long-term BE work. Native dual-write; eliminates export lag.

---

## References

- Apache Iceberg View Spec: `iceberg/api/src/main/java/org/apache/iceberg/view/`
- Iceberg HMS View Storage: `iceberg/hive-metastore/src/main/java/org/apache/iceberg/hive/HiveViewOperations.java`
- Iceberg MV Spec PR: [apache/iceberg#9830](https://github.com/apache/iceberg/pull/9830) — adds `storageTable`, `RefreshState`, Spark `CREATE MATERIALIZED VIEW` (Phase 2 reference)
