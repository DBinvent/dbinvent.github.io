# yt-pgxt

**A Postgres extension that turns ordinary Postgres nodes into a
distributed cluster — no fork, no proprietary storage engine, no
coordinator.**

Status per claim: **Available today** (working, in active use) vs. **In
development** (designed, not yet functional) — flagged throughout so
nothing here promises more than what's real today.

## 0. Competitors

<!-- verified 2026-08-05: names/owners current as of this pass; see per-section
     tables below for the specific claims backed by each link. -->

| Product | Category | Links |
|---|---|---|
| **Greenplum** | Postgres *fork*, now closed-source under Broadcom (May 2024) — MPP data warehouse, dedicated coordinator + segment nodes | [Greenplum architecture overview](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/admin_guide-intro-arch_overview.html) |
| **ClickHouse** | Purpose-built OLAP columnar engine, own SQL dialect and wire protocol; ClickHouse Inc., independent | [ClickHouse architecture overview](https://clickhouse.com/docs/academic_overview) |
| **CockroachDB** | Postgres-*wire-compatible* full rewrite — distributed KV store (Pebble storage engine) with Raft consensus; Cockroach Labs, still independent (private) as of 2026 | [CockroachDB architecture overview](https://www.cockroachlabs.com/docs/stable/architecture/overview) |
| **Citus** (Microsoft, acquired 2019) | Postgres *extension* — closest architectural cousin, coordinator + worker shards; current docs live on Microsoft Learn (citus-14) | [Citus overview — Microsoft Learn](https://learn.microsoft.com/en-us/postgresql/citus/what-is-citus?view=citus-14) |
| **EDB Postgres Distributed** (PGD, formerly BDR) | Postgres extension — logical-replication-based multi-master; latest is PGD v6.4 | [PGD architecture overview](https://www.enterprisedb.com/docs/pgd/latest/overview/basic-architecture/) |

**yt-pgxt:** Postgres **extension**, stock Postgres core untouched, no
coordinator node, no custom storage engine — rows live in ordinary
Postgres tables on every node.

---

## 1. No data transfer on cluster migration

*Status: **Available today** (topology changes) · **In development** (partition rebalancing)*

Row synchronization rides on Postgres's own trigger mechanism, writing
through the database's native storage — not a proprietary distributed
file format. Cluster topology and per-table sync policy live in ordinary
config tables that reload live. Adding or removing a node is a
configuration change, not a data-shuffling operation.

<!-- verified 2026-08-05 -->
| Competitor | Cluster expansion / rebalancing reality |
|---|---|
| **Greenplum** | Expanding the cluster (`gpexpand`) still physically redistributes data in a two-phase process (segment init, then table redistribution). Docs direct operators to run redistribution "during low-use hours" and allow splitting it "into batches over an extended period" — meaningful load impact, though current docs no longer use the words "read-only". [gpexpand reference](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/utility_guide-ref-gpexpand.html) |
| **ClickHouse** | `clickhouse-copier`, the tool operators used to script re-insert/ETL resharding, is obsolete and removed from core packages (moved to a standalone, unmaintained repo). No built-in automatic rebalancer has replaced it. [copier repo](https://github.com/ClickHouse/copier) |
| **CockroachDB** | Automatic range rebalancing via Raft/gossip; CockroachDB's own engineers document rebalancing "crowd[ing] out other, more salient work," and the product ships cluster settings (`kv.snapshot_rebalance_max_rate` etc.) specifically to throttle rebalance impact on live traffic. "Rebalance storms" is informal terminology, but the tradeoff is vendor-acknowledged. [replication layer](https://www.cockroachlabs.com/docs/stable/architecture/replication-layer) |
| **Citus** | `citus_rebalance_start()` is still current (citus-14) and still real data transfer via logical replication, with throttle/parallelism controls (`citus.max_background_task_executors_per_node`, `parallel_transfer_colocated_shards`). Citus 13.2 added snapshot-based node addition — promoting a streaming-replica clone instead of a full network copy for the *add-a-node* case — narrowing but not eliminating the gap. [cluster management](https://learn.microsoft.com/en-us/postgresql/citus/cluster-management?view=citus-14) |

**Why it matters:** the failure modes competitors carry during scaling —
rebalance-induced hot spots, degraded-cluster windows, coordinator
downtime mid-resharding — don't apply the same way here. Lower risk to
schedule a scaling event during business hours, and lower risk of an
expansion turning into an incident. Caveat: this is strongest for
*topology* changes today (adding/removing nodes); selective row placement
across shard/az groups for true partition-level rebalancing is still in
development.

**No hassle on reverse — no vendor lock-in.** Because rows are stored in
native Postgres tables rather than a proprietary engine, the adoption
decision is reversible with the same low friction as adopting it: remove
the extension, drop the sync triggers/config tables, and every node is
left holding a plain, standalone Postgres database — no export step, no
proprietary-format conversion, no stranded data.

<!-- verified 2026-08-05 -->
| Competitor | What leaving actually takes |
|---|---|
| **Greenplum** | Leaving requires `gpbackup`/unload (`COPY ... ON SEGMENT` to per-segment CSV, or GPSS) to get data out of segment storage into a portable format — not just an uninstall. [gpbackup docs](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum-backup-and-restore/1-28/greenplum-backup-and-restore/admin_guide-managing-backup-gpbackup.html) |
| **ClickHouse** | MergeTree's on-disk part format isn't plain/portable rows; getting data out means an explicit export (SQL `FORMAT` clause, `INTO OUTFILE`, or `clickhouse-client`) to Parquet/CSV/etc — a real ETL step, not a file copy. [export formats](https://clickhouse.com/docs/integrations/data-formats/parquet) |
| **CockroachDB** | Pebble (a Go, RocksDB-inspired engine) has been the default storage engine since v20.2 (2020); RocksDB is fully retired, not just an alternative. Leaving still means a dump/export step, not an uninstall. [storage layer](https://www.cockroachlabs.com/docs/stable/architecture/storage-layer) |
| **Citus** | `undistribute_table()` is still the current mechanism, and it's a real operation with real constraints (data must fit locally on the coordinator; fails on foreign-key-linked tables unless cascaded) — not a no-op. [undistributing a table](https://techcommunity.microsoft.com/t5/azure-database-for-postgresql/citus-tips-how-to-undistribute-a-distributed-postgres-table/ba-p/2114362) |

---

## 2. No code base change in a DB platform

*Status: **Available today***

Ships as a Postgres extension, loaded into **unmodified** community
Postgres. Operators run vanilla Postgres binaries — nothing in the core
engine is patched or forked.

<!-- verified 2026-08-05 -->
| Competitor | Postgres compatibility reality                                                                                                                                                                                                                                                                                                                                                                                                                                  |
|---|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Greenplum** | Greenplum 7 (released Sept 2023) is based on PostgreSQL 12; current community Postgres is 18 (Postgres 19 due ~Sept/Oct 2026) — a 6-major-version lag, on Greenplum's own release cadence, not community's. [feature summary](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/ref_guide-feature_summary.html) · [PostgreSQL 18 release](https://www.postgresql.org/about/news/postgresql-18-released-3142/) |
| **CockroachDB** | `pg_trgm` remains unsupported (open tracking issue), PostGIS support is still tracked/in-progress not shipped, and pgvector support is partial: storage and distance-comparison operators work, but vector *indexing* isn't supported. Wire-compatible, not internally Postgres. [pg_trgm issue](https://github.com/cockroachdb/cockroach/issues/71580) · [pgvector status](https://www.cockroachlabs.com/glossary/distributed-db/pgvector/)                    |
| **ClickHouse** | Not a Postgres-compatibility contender by design; included as the analytics-alternative comparison point.                                                                                                                                                                                                                                                                                                                                                       |
| **Citus / EDB PGD** | Current Citus 13 docs still list cross-shard foreign-key restrictions (a foreign key between distributed tables must include the distribution column) and sequence caveats (sub-BIGINT sequences can fail on worker-node inserts) as active limitations. [SQL workarounds](https://docs.citusdata.com/en/stable/develop/reference_workarounds.html)                                                                                                             |

**Why it matters:** full compatibility with the Postgres extension
ecosystem side by side (PostGIS, pg_cron, pgvector, ...), upgrade cadence
that isn't gated by a vendor fork, and existing DBA tooling (`pg_dump`,
`pg_stat_*`, standard monitoring) keeps working unmodified. No
data-consistency risk from a divergent query engine, and customers keep
every stock Postgres feature they already rely on.

---

## 3. No single entry point

*Status: **Available today***

A built-in DNS service publishes live-node records under the cluster's own
zone, continuously refreshed from actual cluster membership. Resolving the
cluster's name returns every currently-live node's address — connections
distribute naturally without a coordinator or proxy being a required,
cluster-critical component. Writes still go through a dynamically elected
leader for ordering, but that leadership is elected with automatic
backup/reserve failover, not a statically pinned endpoint clients must
hardcode.

<!-- verified 2026-08-05 -->
| Competitor | Entry-point architecture |
|---|---|
| **Citus** | A single coordinator remains the default, explicitly documented as "the single point of failure and the bottleneck of the system." HA still means the customer bolts on Postgres streaming replication + an orchestrator (Patroni, `pg_auto_failover`) themselves — not out of the box. [Patroni 3.0 & Citus](https://www.citusdata.com/blog/2023/03/06/patroni-3-0-and-citus-scalable-ha-postgres/) |
| **ClickHouse** | The `Distributed` table engine (query fan-out/merge from an entry node) and `chproxy` (ClickHouse-aware HTTP proxy for routing/caching/failover) remain the standard pattern; still no built-in service discovery. [Distributed engine](https://clickhouse.com/docs/engines/table-engines/special/distributed) · [chproxy](https://www.chproxy.org/getting_started/) |
| **CockroachDB** | **Genuine peer on this specific axis** — any node can serve, but it ships no DNS layer; customers bring their own LB/DNS/service-mesh. |
| **Greenplum** | A hard single entry point: clients connect through the coordinator (or standby coordinator) and segments aren't directly reachable. Current docs just use "coordinator" rather than the older "master" label, which doesn't change the claim either way. [architecture overview](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/admin_guide-intro-arch_overview.html) |
| **EDB PGD** | PGD Proxy remains the recommended routing layer, reading write-leader identity from PGD's own Raft consensus and routing writes there. The closest architectural parallel among competitors on this axis — though PGD still fronts with a dedicated proxy tier rather than DNS. [PGD Proxy](https://www.enterprisedb.com/docs/pgd/5.9/routing/proxy/) |

**Why it matters:** one fewer HA-critical tier to design and operate — no
separate load-balancer/proxy layer that itself needs a failover story —
and cluster membership changes are visible to clients "for free" via DNS
instead of requiring load-balancer reconfiguration. Better baseline
resiliency, and no extra network hop on the read path.

---

## 4. Programmatic synchronization configuration

*Status: **Available today***

Every table's replication policy is a first-class, live-reloadable
setting — no restart, no migration:

- **None** — no sync at all, purely local data.
- **N nodes** — an explicit "+N node" quorum requirement.
- **Half the cluster.**
- **The whole cluster.**

The mechanism behind the dial matters as much as the dial itself: this
isn't a storage-layer replication factor applied asynchronously in the
background (the way a fixed replica count is). Each write commits
transactionally (XA/two-phase-commit style) to exactly the node set the
table's policy names at that moment — durability is a write-time
transactional guarantee to a *dynamic, programmatically configured*
participant set, and changing that set is a config-table update, not a
schema or DDL change. Reads, by contrast, can be served by any live node
via the DNS mechanism (§3) without going through the write quorum. The
resulting model is **eventual consistency on read, strong consistency on
write**.

None of the competitors below combine these two properties — dynamic,
schema-free participant count *and* per-write transactional commit to
that set. Each does at most one.

<!-- verified 2026-08-05 -->
| Competitor | Closest mechanism, and where it falls short |
|---|---|
| **Greenplum / ClickHouse** | Replication factor and consistency are generally a *cluster- or table-creation-time* topology decision, not a live per-write tunable. |
| **CockroachDB** | **Closest analogue for "dynamic"** — `ALTER TABLE ... CONFIGURE ZONE USING num_replicas` is live and per-table, but it governs a fixed Raft replica count reached via background consensus, not a per-write transactional acknowledgment to a variable set. The tradeoff it buys is strong consistency on *both* read and write (any replica is Raft-consistent), not this product's eventual-read/strong-write split. [replication zones](https://www.cockroachlabs.com/docs/stable/configure-replication-zones) |
| **Citus** | **Notice:** `citus.shard_replication_factor` has been deprecated since Citus 5 (2016) and removed from docs entirely as of Citus 11.x (~2023) — it's gone, not just stale terminology. Current Citus durability comes from two fixed shapes, neither a runtime per-table dial: Postgres streaming replication of an entire worker node (all its shards at once, async, not transactional), or reference tables — always fully copied to every worker with true 2PC: strong on both read and write, but a fixed "whole cluster" shape with no dial down. [concepts](https://docs.citusdata.com/en/stable/get_started/concepts.html) · [docs issue](https://github.com/citusdata/citus_docs/issues/1032) |

**Why it matters:** durability and latency become a per-table dial instead
of a whole-cluster policy — hot, ephemeral tables cost nothing to
replicate, critical tables get transactional durability across exactly
the nodes that matter to them, and that dial can move live as traffic
patterns change, with zero migration downtime and no schema touch. Reads
stay cheap and eventually consistent from any node; writes stay
transactionally strong against a participant set that scales with the
table's actual durability needs, not a cluster-wide default.

---

## 5. Built-in framework for serverless + schema migration

*Status: **Available today** (serverless) · **In development** (schema migration)*

**Serverless — available today.** A Postgres row-change trigger can
directly invoke a python function across the cluster, backed by its own
lightweight leader election and cron-style scheduling for recurring jobs.
Commit-triggered work can route to a python function, a distributed cache
update, or a distributed queue — all configuration-driven, with no
separate CDC pipeline, event bus, or FaaS platform to stand up and wire
together.

**Schema migration — in development.** A YAML-defined schema+data model
merges declaratively into Postgres, and the cluster config already
reserves a slot to track schema-version state cluster-wide. The piece
still being built is the wire path that would broadcast a schema change
through the cluster's own leader/consensus mechanism the same way data
changes propagate — today that still requires an out-of-band tool run by
each node's operator. Real design, not yet functional — flagged so it
isn't pitched as shipped.

None of Greenplum, ClickHouse, or CockroachDB ship a built-in
"data-change triggers *a function*" layer.

<!-- verified 2026-08-05 -->
| Competitor | What it ships instead |
|---|---|
| **CockroachDB** | Native `CREATE CHANGEFEED` CDC (Kafka/webhook/cloud-storage/sinkless sinks) — but a changefeed has no code-execution capability at all, at any point; it emits an event to a sink and stops. That's a different problem (event delivery) from the one this section claims (direct function invocation on commit) — not a partial or weaker version of the same capability. [CREATE CHANGEFEED](https://www.cockroachlabs.com/docs/stable/create-changefeed) |
| **Greenplum** | GPSS (Greenplum Streaming Server) is ingestion-only (Kafka/RabbitMQ/S3 → Greenplum), not outbound CDC. [GPSS overview](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum-streaming-server/2-0/gp-streaming-server/overview.html) |
| **ClickHouse** | Its newer ClickPipes connectors are inbound CDC *into* ClickHouse from Postgres/MySQL/Mongo, not outbound triggers. |

<!-- verified 2026-08-05 -->
| Competitor | Online DDL reality |
|---|---|
| **CockroachDB** | `ALTER TABLE` schema changes run in the background with "no table locking or read/write impact" per current docs. [online schema changes](https://www.cockroachlabs.com/docs/v26.2/online-schema-changes) |
| **Citus** | Automatic DDL propagation covers a defined subset (adding columns, `CREATE INDEX CONCURRENTLY`, most constraints), but changing a distribution column's type is prohibited outright, and several other statement classes still require manual propagation or are excluded. The differentiator framing holds — "no separate migration tool/pipeline to keep in sync with cluster topology," not "competitors can't do online schema changes" — but Citus's online DDL is narrower than a bare "genuine online, non-blocking schema changes" claim implies. [DDL reference](https://learn.microsoft.com/en-us/postgresql/citus/reference-ddl?view=citus-14) |

---

## 6. Composable with proven infrastructure, not a replacement for it

*Status: **Available today***

Nothing here forces operators to abandon standard, battle-tested Postgres
operational patterns — they're optional layers that sit alongside the
cluster, not tooling this product replaces:

- **HAProxy** — a generic TCP load balancer can front the cluster exactly
  as it would any Postgres fleet, using the built-in DNS service (§3) or a
  static backend list as its target set — no custom health-check protocol
  required.
- **PgBouncer** — connection pooling works as normal; a cluster-aware
  variant goes further, hooking the pooler's own transaction boundary
  (COMMIT/ROLLBACK) to synchronize with the cluster's own commit
  protocol before acknowledging the client — pooling and cluster
  consistency compose instead of fighting each other.
- **Native Postgres streaming replication** — because core Postgres is
  untouched (§2), a node can still run standard streaming replication for
  a local read-replica or a zero-RPO standby, entirely independent of
  cross-node cluster sync.

<!-- corrected 2026-08-05: scope of this correction is the "vs. competitors"
     comparison only — the three first-party bullets above (HAProxy,
     PgBouncer, native Postgres streaming replication) are this product's
     own capabilities, not claims about competitors, and none of them
     needed correction. Within the comparison itself, only the
     HAProxy/PgBouncer framing for 2 of the 5 competitors needed fixing
     (Greenplum, CockroachDB); ClickHouse, Citus, and EDB PGD held up as
     originally stated. Not a wholesale miss — see per-product notes. -->

| Competitor | Standard-tooling story |
|---|---|
| **Greenplum** | **Notice:** standard community PgBouncer "is included in your Greenplum Database installation" per current docs, not a fork; a real, first-party-supported plain-Postgres pooling story. (Docs describe pooler-level failover via config reload rather than officially prescribing HAProxy in front, so the HAProxy half of the claim is weaker than the PgBouncer half — but PgBouncer itself is not bespoke.) [PgBouncer in Greenplum](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/admin_guide-access_db-topics-pgbouncer.html) |
| **CockroachDB** | **Notice:** HAProxy is *better* supported than stated: a first-party `cockroach gen haproxy` command generates a ready-to-use config for the running cluster. PgBouncer is the opposite: real but community-tested only, "not officially supported by Cockroach Labs," with genuine gaps (`scram-sha-256`/`hba`/`pam` auth methods unsupported as of this pass). Net: still not a forced bespoke-proxy story. [cockroach gen](https://www.cockroachlabs.com/docs/stable/cockroach-gen) · [community tooling](https://www.cockroachlabs.com/docs/stable/community-tooling) |
| **ClickHouse** | Not Postgres wire protocol, so PgBouncer doesn't apply; `chproxy` remains the ClickHouse-specific routing layer (§3). |
| **Citus** | Can pool to its coordinator, but that's pooling to a single point (§3), not cluster-wide. |
| **EDB PGD** | `pgd-proxy` remains the recommended routing layer (§3), not generic HAProxy/PgBouncer. |

**Why it matters:** lower adoption cost and lower operational risk —
existing HA/pooling runbooks, monitoring, and staff expertise around
HAProxy, PgBouncer, and Postgres streaming replication carry over
directly, instead of requiring a new bespoke ops stack learned from
scratch.

---

## 7. Built-in logical sharding — mirrored, overlapping, or disjoint

*Status: **Available today** (mirrored) · **In development** (overlap & disjoint partitioning)*

One data-placement configuration surface spans three distinct
architectures, instead of forcing a single one per table:

- **Mirrored** — full replication: every node holds a complete copy (the
  default "symmetric" sync mode, see §4's "whole cluster" commit setting)
  — maximum read locality and failure tolerance, at full storage cost per
  node.
- **Overlap (redundancy)** — rows partitioned across owner nodes, but each
  partition is also mirrored to N-1 extra nodes for redundancy — a
  tunable middle ground between full replication and strict partitioning.
- **Disjoint** — strict non-overlapping partitions: each row has exactly
  one primary-owner node, no redundancy copy — minimum storage cost,
  maximum horizontal capacity.

All three are the *same* per-table configuration surface (a partition-key
strategy plus a redundancy count), not three different products — a table
moves between them by changing configuration, not by a schema rebuild or
a migration to a different table type.

The write-consistency guarantee tracks the same dial as the storage
shape, not a separate setting. Full-cluster mirroring is deliberately
*not* committed synchronously to every node in one transaction — at
whole-cluster scale that would trade away availability for no real
benefit, so it favors eventual convergence (§4). Overlap and disjoint
shapes commit each write transactionally (XA-style, §4) to exactly the
small, explicit owner-plus-redundancy node set that partition names —
partial replication, but a real transactional guarantee across that
partial set. Smaller, explicit node sets get transactional strength;
full-cluster mirroring trades that for availability at scale. Same
overall model as §4: eventual consistency on read, strong consistency on
write, scoped to whichever nodes actually hold the data.

<!-- corrected 2026-08-05: Greenplum and Citus sub-claims were overstated
     ("not one dial adjusted", "no live path") — both actually expose a
     live command to move between placement models, just not a free one.
     Fixed in place; see notes. -->

| Competitor | Placement-model flexibility |
|---|---|
| **Greenplum** | **Notice:** distribution policy is *not* frozen at creation time: `ALTER TABLE ... SET DISTRIBUTED BY / RANDOMLY / REPLICATED` is a live, documented command — it *is* one dial, adjustable later, without dropping/recreating the table. The real distinction from this product's model is cost, not DDL rigidity: Greenplum's dial-turn triggers a full physical data redistribution (the same `reorganize=true` machinery as §1's `gpexpand`), not a lightweight config flip. [ALTER TABLE](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/ref_guide-sql_commands-ALTER_TABLE.html) |
| **ClickHouse** | Mirrored means `ReplicatedMergeTree` (full copy per replica, via Keeper/ZooKeeper); sharded means the `Distributed` table engine with a separate sharding key, fanning out to per-shard `ReplicatedMergeTree` tables — two different table setups combined together, not a single live-adjustable spectrum. [replication handbook](https://posthog.com/handbook/engineering/clickhouse/replication) |
| **CockroachDB** | Per-table/index/partition `CONFIGURE ZONE USING num_replicas` (see §4) *is* a real, first-class per-table dial for replica count. The actual gap from this product's three-way model is different: CockroachDB has no distinct "disjoint, zero-redundancy" mode — ranges are always replicated (default 3x) as the same mechanism that provides sharding, so there's no first-class way to say "this partition, no redundancy copy" the way `num_replicas` can say "this partition, N copies." |
| **Citus** | **Notice:** a live path to move a table between placement models *does* exist and doesn't require redefining the table's schema: `SELECT undistribute_table('t');` then `SELECT create_reference_table('t');`. Worth noting: it's a real operation, not a free config flip — data physically moves to the coordinator and back out, and it's two separate function calls rather than one dial, closer in spirit to Greenplum's reorganize than to a config toggle. [DDL reference](https://learn.microsoft.com/en-us/postgresql/citus/reference-ddl?view=citus-14) |

**Why it matters:** one mental model and one configuration surface for
the full spectrum, from maximum availability (full copies everywhere) to
maximum horizontal scale (no redundancy) — and a specific table can move
along that spectrum as its access pattern changes, without a schema
rebuild or adopting a different table type. The consistency guarantee
moves with it automatically: dial toward disjoint/overlap and writes get
transactional strength against a small, explicit node set; dial toward
full mirroring and the table trades that for maximum read availability —
one config surface driving both placement and consistency together,
instead of the two being set independently (or not being configurable at
all) as they are in every competitor's model above.

---

## 8. Concurrent, path-level safe editing of the same document

*Status: **In development***

Transaction boundaries aren't limited to whole-row granularity. For JSON/
document columns, a boundary can be scoped down to a specific path (leaf)
within the document — so two transactions touching *different* fields of
the *same* JSON document can commit concurrently instead of one blocking
or conflicting with the other, the way whole-row locking/versioning
otherwise forces.

<!-- verified 2026-08-05 -->
| Competitor | Document-write granularity |
|---|---|
| **Postgres-based** (Greenplum, Citus, EDB PGD) | Postgres itself resolves write conflicts on a JSONB column at the *row* level: it rewrites/locks the whole row on any update, so two transactions editing different keys in the same document still contend for the same row lock and serialize under MVCC, unless the application hand-rolls its own merge logic. [Postgres concurrency](https://devcenter.heroku.com/articles/postgresql-concurrency) |
| **CockroachDB** | Conflict detection is by KV key (row/range-scoped), not JSON-path-aware; no path-level exception exists there either. |
| **ClickHouse** | **A different axis, not a narrowing gap** — standard SQL `UPDATE` shipped in v25.7 (2025), applied via non-blocking, snapshot-based "patch parts." But the claim here isn't about `UPDATE` syntax — it's about transactional safety: no dirty reads of a partial/in-flight write, and repeatable apply (the same conflict-resolution outcome regardless of retry or replay order). Patch parts give neither: no isolation guarantee against reading a part mid-application, and no multi-statement ACID transaction wrapping the update. Mutation syntax and transactional isolation are two different things. [updating data](https://clickhouse.com/docs/updating-data/overview) |

**Why it matters:** for workloads with wide, semi-structured documents —
user profiles, configuration blobs, nested settings — this removes a
common hidden bottleneck where unrelated concurrent edits to the same
document serialize against each other purely because they share a row,
not because they actually conflict. Fewer retries, less contention, more
usable concurrency without redesigning the schema into many narrow
tables to get parallelism.

---

Together, these are the properties of a distributed system — without
asking anyone to leave Postgres.
