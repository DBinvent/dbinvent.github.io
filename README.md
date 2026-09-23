# DBinvent

**_Build your own data source, better than on cloud_**

*Chicago based deep tech startup since 2019*

The problem is as simple as swapping a disk — RAID and NAS solved it for
storage decades ago. Nothing that simple exists for the database, which sits
at the heart of almost every IT project. Existing answers work, but they are
expensive, lock you to a vendor, or require downtime and a data transfer — and
they still leave consistency to chance.

---

## What goes wrong today

**Consistency, when a transaction isn't about moving money.**
AWS, on their own outage: *"The second Enactor's clean-up process then deleted
this older plan because it was many generations older than the plan it had just
applied."* A transaction-isolation defect that became a global incident.

**Scaling up, when vertical is no longer an option.**
CAP says you cannot have consistency, availability and partition tolerance at
once — so most products quietly drop one and don't tell you which.
→ [Solving CAP](https://github.com/vkrinitsyn/vkrinitsyn.github.io/blob/main/cap.md)

**Backup, restore, repeat — without compromise.**
Possible today, but it needs a single entry point, manual intervention on
failure, and paying for resources that sit idle.

---

## YaXaHa — the cluster like in cloud, but better, and yours

Vanilla PostgreSQL plus an extension. No forked engine, no dedicated master.
→ [dbinvent.com/cluster](https://dbinvent.com/cluster/)

- **No data transfer.** Your database becomes a cluster node where it already
  sits — nothing dumped, copied or migrated to get started.
- **No vendor lock.** Remove the extension and you are back to a single plain
  PostgreSQL instance, data in place. The way out is documented, not theoretical.
- **No code change.** Same SQL, same drivers, same extensions — every PostgreSQL
  feature is still the PostgreSQL feature, because the engine is unmodified.

And the only one of them that arrives with a **built-in DataFusion MPP engine**
and **Python server functions** — analytics and serverless over the same data,
without moving it anywhere either.

- **Strong consistency on write, eventual on read.** Tuned for the **Ratio
  profile pattern** — the 80% read / 20% write shape most workloads actually
  have: always readable, and writes wait only for what correctness requires.
- **Virtual dynamic partitioning.** Transaction boundaries are derived at
  runtime, so disjoint writes spread across nodes instead of queueing behind
  each other.
- **RAFT consensus, not a single entry point.** The coordinator arbitrates and
  assigns workers; it never becomes the bottleneck every write must pass
  through.
- **Synchronization as configuration.** Per-table rules decide what is
  redundant, what is local, and how strong each commit has to be.

**It is measured, not asserted.** Against PostgreSQL's own synchronous
replication to the same two standbys: **1.7x the throughput (171%)** on a
write-only load and **1.6x (162%)** on the Ratio profile pattern — at equal
replication scope, on a six-node lab with 100 live checks and 11 scenario
tests passing.
→ [What replication actually costs](yaxaha-cluster-performance.md)

---

## What shipped since the first pitch

The 2021 roadmap is now the product.

- **Software-defined topology — AZ, zones, tiers.** Replication scope and row
  placement are *rules*, rewritten while the table is being written to; the
  cluster migrates itself to match. No redeploy, no downtime, no fixed shape —
  which is what makes one cluster hierarchical and unbounded in scale.
- **Self-healing.** A topology that can change has to verify itself:
  continuous placement verification, online partition migration, and recovery
  that repairs rather than reports.
- **MPP — Apache DataFusion.** A best-in-class analytics engine planning
  *against* that declared topology, reachable from server functions and from
  any table already marked for cluster sync.
- **ClickHouse and Postgres, neither asked to be the other.** ClickHouse is not
  transactional; Postgres is not an analytics engine. Both are reached through
  the runtime and the KV interface below.

→ [Where Rows Live](where-rows-live.md) — the full argument, and the testing
that had to pass before any of it could be believed.

---

## The platform around it

- **Serverless functions — [RPPD](https://github.com/vkrinitsyn/rppd).**
  Build your own Lambda or Azure Functions without the vendor: Python triggered
  by Postgres insert, update and delete, deployed as functions rather than
  services. → [overview](https://github.com/vkrinitsyn/vkrinitsyn.github.io/tree/main/rppd)
- **[etcd](https://github.com/vkrinitsyn/etcd) API v3 KV interface**, so
  anything that already speaks etcd needs no bespoke client — plus a
  [queue](https://github.com/vkrinitsyn/etcd/blob/main/queue.md) with order and
  delivery guarantees.
- **Schema management — [Schema Guard](https://github.com/vkrinitsyn/schema_guard)
  and [Rumba RDBM](https://github.com/vkrinitsyn/rdbm).** Declarative *and*
  imperative, Flyway inspired, tracked by checksum per target environment;
  migration as the backbone of CI/CD.
  → [dbinvent.com/rdbm](https://dbinvent.com/rdbm/)
- **[diff-doc](https://github.com/vkrinitsyn/diff-doc-rs)** — commutative patch
  apply, so two users editing the same JSON row get neither a delay nor a lost
  write. → [article](https://medium.com/@v.krinitsyn/concurrent-document-modification-ea1b6e628e2d)

---

**More:** [pitch, 4 pages, PDF](DBinvent%20-%20pitch.pdf) ·
[all articles](https://github.com/vkrinitsyn/vkrinitsyn.github.io?tab=readme-ov-file#articles)
