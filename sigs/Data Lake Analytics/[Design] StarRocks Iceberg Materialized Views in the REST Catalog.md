# [Design] StarRocks Iceberg Materialized Views in the REST Catalog

- SIG: Data Lake Analytics
- Status: Draft (prototype) — seeking SIG / maintainer input, especially on the incremental-refresh direction
- Tracking issue: StarRocks/starrocks#79956
- Depends on: the Apache Iceberg materialized-view spec (apache/iceberg#11041, impl #9830) — currently unreleased

## 1. Motivation

Apache Iceberg's materialized-view spec extends the view spec so a view can carry a `storage-table` identifier plus a `RefreshState` (the source-snapshot lineage of the last refresh). This makes a materialized view a first-class, engine-agnostic object in an Iceberg catalog: any spec-aware engine can discover it, read its precomputed data from the storage table, and judge its freshness from the `RefreshState`.

This proposal adds StarRocks support for creating and maintaining such materialized views **in an Iceberg REST catalog (IRC)**, so that:
- a StarRocks-created MV is readable by other engines (Trino, Spark) through standard Iceberg catalog APIs, and
- StarRocks can (in future) consume externally-produced Iceberg MVs.

## 2. Goals / Non-goals

**Goals**
- `CREATE / DROP / RENAME` and refresh of an Iceberg materialized view that lives in the IRC.
- Reuse StarRocks' task scheduler to drive refresh and record run-history.
- Interoperable on-disk representation: native Iceberg `View` + storage table + `RefreshState`, no StarRocks-only encoding.

**Non-goals (initial scope)**
- Query rewrite over the IRC-resident MV (future).
- Incremental / partition-level refresh (future — see §6, the main open question).
- MVs over non-Iceberg base tables (initial scope is all-Iceberg bases; see §6).

## 3. Proposed design: the IRC owns the MV

`CREATE MATERIALIZED VIEW <irc_catalog>.<db>.<mv> AS <query>` creates, **in the IRC**:
- an Iceberg `View` whose `currentVersion().storageTable()` points at a `<mv>__storage` Iceberg table, and
- the `<mv>__storage` table itself (initially empty).

StarRocks persists **no native materialized view** in its internal catalog for these MVs — the IRC is the single source of truth for the definition and freshness. StarRocks keeps only a refresh **task** (a schedule) and task-run history locally.

On refresh, StarRocks runs `INSERT OVERWRITE <catalog>.<db>.<mv>__storage SELECT <query>` and stamps an Iceberg `RefreshState` (the base tables' snapshot ids read during the refresh) into the overwrite snapshot's summary — a single atomic Iceberg commit. Freshness is therefore recorded in the catalog in the spec-standard form.

### 3.1 Flow

```
CREATE   analyzer (validate; feature-flagged; all-Iceberg bases; REST catalog)
         -> connector (IcebergMetadata): create <mv>__storage + spec View (storageTable linkage)   -> IRC
         -> register refresh Task keyed by the IRC identifier (catalog.db.mv)                        -> local (task only)

DROP /   connector: drop / rename View + <mv>__storage   -> IRC
RENAME   update / remove the refresh Task                -> local

REFRESH  task scheduler (or manual REFRESH) -> dedicated Iceberg-MV refresh processor:
         - build RefreshState from the base tables' current snapshots
         - INSERT OVERWRITE <mv>__storage SELECT <query>  (Iceberg sink)
         - commit storage snapshot + RefreshState in its summary (one atomic commit)   -> IRC
         - record run status                                                            -> local task-run history

READ     query <mv>__storage through the IRC like any Iceberg table (transparent rewrite is future work)
```

### 3.2 Why a dedicated refresh processor
StarRocks' native MV refresh pipeline is coupled to a locally-registered, lockable `MaterializedView` object (it re-resolves and locks the MV from the local catalog in several places). Since an IRC MV has no local object, refresh uses a small **dedicated processor** on the existing task framework: it reads the view definition, runs the `INSERT OVERWRITE`, and writes the `RefreshState`. It needs none of the native per-partition version-map machinery for full refresh.

### 3.3 Concurrency
No StarRocks object lock is needed: same-MV refreshes are serialized by the task scheduler (one running task-run per task); commit atomicity comes from Iceberg's own commit protocol; base-table read consistency comes from Iceberg snapshot isolation (bases are all Iceberg). Only the leader FE schedules refreshes.

## 4. Alternatives considered

- **Native MV mirrored to the catalog** (cf. StarRocks/starrocks#78193): keep a native StarRocks MV and mirror a spec view into the IRC. Reuses the full native MV engine (including incremental), but the catalog becomes a copy, not the owner — two sources of truth. This proposal instead makes the catalog authoritative.
- **Reuse the native refresh pipeline for the IRC MV**: not viable without a locally-registered MV (the pipeline's lock/resolve assumptions), hence the dedicated processor.

## 5. Initial scope

Full refresh, non-partitioned storage, all-Iceberg base tables; manual and scheduled refresh; freshness = the standard `RefreshState`. REST catalog only; gated by a feature flag (default off).

## 6. Open question for the SIG: incremental refresh

Incremental (partition-change-tracking) refresh is one of StarRocks' most valuable MV features. In the current codebase it relies on per-partition version maps that live on the local `MaterializedView`. With the MV owned by the IRC, the spec's `RefreshState` is **per source table** (one snapshot id per source), not per partition — so it can express a coarse fresh/stale verdict but not "which partitions changed."

Two directions, and we'd like the SIG's guidance:
1. **Extend what we write to the catalog** — carry per-partition / snapshot-delta state as a StarRocks extension in the storage-table snapshot summary (alongside the standard `RefreshState`, which other engines keep reading), and compute staleness from it. Keeps the catalog authoritative; requires reimplementing the partition-change logic against catalog-resident state.
2. **Keep a local MV object** to reuse the native incremental pipeline — simpler, but reintroduces the catalog-vs-engine duality this design avoids.

Questions:
- Does the IRC-owned direction match where StarRocks wants external/Iceberg MVs to go?
- Is there appetite for a StarRocks extension to the snapshot summary to carry per-partition freshness?
- Any guidance on decoupling the partition-change-tracking logic from the local `MaterializedView`?

## 7. Phasing
1. **This proposal** — CREATE/DROP/RENAME + full refresh writing `RefreshState`.
2. Freshness-aware **query rewrite** (reading `RefreshState`).
3. **Incremental / partition-level** refresh (per §6).

## 8. Compatibility & limitations
- Requires a REST catalog whose server understands the MV spec (persists the view `storage-table` field + snapshot-summary); a stock server would drop the linkage.
- Not mergeable until the Iceberg MV spec is released; this is a prototype to be ready when it lands.
- Behavior is unit-tested; end-to-end validation against a live MV-aware REST catalog is pending.
