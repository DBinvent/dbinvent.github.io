# Where Rows Live

The topology is **software-defined**: replication scope and row placement are rules, rewritten at runtime, with the cluster migrating itself to match — no redeploy, no downtime, no fixed shape. That is what makes one cluster hierarchical and unbounded in scale, and it is precisely the structure a **best-in-class MPP engine — DataFusion** — needs in order to plan against it.

Neither store is then asked to be what it isn't. **ClickHouse is not transactional; Postgres is not an analytics engine.** Both are reached instead through a **built-in server-function runtime** — event-triggered, deployed as functions rather than services, in the manner of Lambda or Azure Functions — and through a **standard etcd KV interface**, so anything that already speaks etcd needs no bespoke client.

*Everything below is the argument for that design — what each claim demands in return, and the reliability testing that had to pass before any of it could be believed.*

`YaXaHa cluster · ytserv 0.16 / ytpgc 0.5 · six-node lab · 2026-09-20`

| | |
|---|---|
| **100** | live checks all passing |
| **11** | scenario tests |
| **231** | unit tests ytserv + yxctl |
| **9** | defects found none by unit tests |
| **6** | nodes · 3 zones 2 tiers |

---

### Part 01 The argument

## A deployed shape has a ceiling. A declared one does not.

Most clustered databases have a shape you *deploy*. It lives in a manifest, an operator config, a provisioning script — somewhere outside the running system — and changing it is a project: plan a migration, drain traffic, cut over, hope. The shape is a build artifact.

That approach meets three ceilings, and only the third is usually discussed.

- **Capacity.** If every table lives on every node, the cluster's capacity is one node's capacity. Adding machines adds redundancy, never room. This is the ceiling people hit first and mistake for a hardware problem.
- **Connectivity.** If every node must know every other, connections and health traffic grow quadratically. A thousand-node cluster pings like a thousand-node cluster whether or not any two of those nodes share a single table.
- **Change.** If the shape is deployed, every change to it is a deployment — which means changes are batched, rare, and frightening, which means the shape is always slightly wrong.

**Software-defined topology** answers all three, and the second is answered by the hierarchy rather than by the rules. All-to-all linking is bounded to a *subcluster* — the group of nodes that actually share a table's shard and duplicate it for each other. Nodes inside a subcluster are fully meshed because they must be; nodes in different subclusters have no reason to speak and do not. Levels nest, and the DBA decides how many: the ceiling on scale becomes the subcluster size, not the cluster size, and the subcluster size is a design choice rather than an emergent property.

A flat cluster is simply the degenerate case of that — one subcluster, all-to-all, sharded across `Nodes:0+`. It is the shape this lab runs, which is why the report below exercises full meshing throughout; it is not the shape the architecture is limited to.

What makes all of it *defined* rather than merely configurable is that the shape is *data*: changing it is a write, and the system reconciles itself to the new rule while serving traffic. Configurable systems still make you restart. Defined, in the sense that software-defined networking is defined — the control plane is a running program reading rules, and the rules are editable by anything that can write a row.

That claim is easy to make and expensive to honour. It creates four debts, and this article is the argument that each has been paid — or, in one case, an honest accounting of what has not been.

| What "software-defined" demands | Because otherwise | Where |
|---|---|---|
| **The rule is enforced everywhere a row could leak** | A rule checked only at the entry point is decorative; rows arrive by other paths | Part 02 |
| **Placement is deterministic enough to plan against** | A shape that answers differently each time it is asked cannot be queried, only stored | Part 03 |
| **The rule can change under live writes** | Without this it is configuration with extra steps, not a control plane | Part 04 |
| **The system can prove data followed the rule** | Runtime mutation without verification is a faster way to lose data | Part 05 |

There is a fifth demand that belongs to reliability rather than to shape, and Part 11 returns to it as the thing that actually matters in production: **the replication level must itself be software-defined, per table, across a strict-to-tolerant spectrum** — so that data which matters is committed strictly and, when something goes wrong with it, the system stops rather than continuing to run corrupt.

> **The through-line, stated once**  
> Scoped replication requires scoped verification. Every mechanism that silently assumed *everyone has everything* broke the moment that stopped being true — including the test harness, which reported green while measuring nothing at all. Roughly half of this report is that discovery.
>

---

### Part 02 Demand one — enforcement

## Shape as data: zones, tiers, and three places a row can leak

Placement is expressed with two ideas that compose, and deliberately no more than two. A node carries an **availability zone** — a string, nothing more. Zones carry a **tier** — an integer, where 0 is the top.

Both live in ordinary config rows, module `Z`: `<node_uuid> → <az>` and `<az>.tier → <n>`. They replicate through the config path that already existed, so there is no new gossip protocol, no new consensus, and no second source of truth about where a node sits.

That choice is what makes the topology **software-defined rather than deployed**. There is no cluster manifest, no shape compiled into a build, and nothing to re-provision when the answer changes: zones, tiers, routing and row placement are all rows, and rewriting a row is how you change the shape.

A table then declares its routing rule — which zones hold it, which may originate writes to it, and whether it may cross tiers upward or downward.

> *Diagram: Two-tier topology: tier 0 zone 'top' holds nodes n1 and n2; tier 1 holds zone 'east' with n3, n4 and zone 'west' with n5, n6. An upward arrow from east to top is blocked by the no_up rule.*
**the claim, disproven on a live cluster**

```
recon.job.0 = #1 manual yxtopo.public.rs_t done in 0.0s [repair, chunk 0, whole table]
              0 chunk(s), 0 checked, 0 divergent, 0 repaired, 0 critical
```


*The lab topology, and the rule that gives tiers meaning. `no_up` keeps an edge zone's bulk tables from climbing into the control plane while still letting configuration descend. Before tiers existed, `no_up` was a protobuf field nothing read.*

The direction check and the membership check are asked as a *single* question per peer, rather than two checks at two call sites, precisely so they cannot drift apart. And the whole feature collapses to a no-op for an unconfigured table or an unzoned node: the unrouted path is the same code, not a parallel branch, so a flat cluster cannot be filtered by accident.

### Enforcement is three points, not one

Here is the first demand paid. A rule checked only where writes enter is decorative, because rows arrive by other routes — replication, recovery, bulk copy. This one is asked at every place a row could leak:

| Decision point | What it prevents | Zone | Shard |
|---|---|---|---|
| **Party selection** choosing who must acknowledge | A peer in a non-holding zone counted as a candidate, making the threshold unreachable | `checkin.rs:446` | `checkin.rs:449` |
| **Write gate** leader-side admission | A write originating in a read-only zone being accepted | `checkin.rs:750` | `checkin.rs:764` |
| **Apply drop** on receipt | A record that arrived anyway being stored where it must not exist | `cluster.rs:500` | `cluster.rs:514` |

Zone and shard compose as an *intersection*: the zone rule says which nodes may hold the table, the shard rule says which of them holds this particular row, and a node must pass both. There is deliberately no origin-side pre-check on the shard — the origin knows its own identity but not the live ring the leader maintains, and a ring the two disagree about is a row with two owners.

### The bill arrives immediately: quorum

The instructive part is that scoping broke something that had been correct for years, and not through a coding mistake. The commit threshold was computed correctly — against the *cluster*. The party was then filtered by scope. A table routed to `east` (two nodes of six) needed three acknowledgements and could offer at most two.

**Every scoped write aborted.** Not intermittently — categorically, by arithmetic. The first table anyone routed anywhere was instantly unusable.

The fix is one function, `party_size_for(mode, at_least, n, floor_self)`, where `n` is the table's own node set rather than the cluster's. The subtle parameter is the last: party *slots* floor at one when the set is empty, while acknowledgement *counts* floor at zero. Conflating them produced a `0usize - 1` wrap to `18446744073709551615`, surfacing as the cheerfully impossible "Not enough working nodes to satisfy requirements: 18446744073709551615".

> **Two node sets that compose in opposite directions**  
> Who *counts* toward the threshold and who *receives* the write are different sets with different algebra. Across a multi-table transaction the counting set takes the **intersection** — the write is only as committed as its most restrictive table. The fan-out set takes the **union** — every node holding any touched table must receive it. Getting these the same way round is a correctness bug that presents as a performance quirk.
>

---

### Part 03 Demand two — determinism

## Row placement, and the strategy that cannot be planned against

Zone routing decides which nodes may hold a table. Sharding decides which of them holds a given row, from a column value:

- **`AZ_NODE_UUID`** — the column value *is* the owner: a zone name, a full UUID, or an 8-character prefix. Nothing moves unless the data moves.
- **`RING_HASH`** — rendezvous (highest-random-weight) hashing over `(key, live node)`, taking the top `redundancy + 1` nodes.

HRW's defining virtue is that adding or removing a node remaps only the keys that node owned — a property verified by a test named for it. Ideal for storage. And it is exactly here that the second demand bites, because **the ring is built from the *live* peer set.**

The same query planned twice can slice differently. A node joining mid-query changes which rows belong to which executor. That is not a planning input; it is a moving target. So one strategy is plannable and the other is not, and the system must say which it is rather than letting a query engine discover the difference at runtime — a point Part 07 takes up directly, because the partition surface publishes exactly that flag.

> **A hash function is part of your data format**  
> The ring uses a hand-written FNV-1a rather than the standard library's `DefaultHasher`. Rust does not guarantee `DefaultHasher`'s output across releases — and a changed hash score does not produce an error. It silently relocates the owner of every row in the cluster. Any hash whose output determines placement, partitioning, or checksums is a wire format and must be pinned like one.
>

---

### Part 04 Demand three — change under load

## Rewriting the rule while the table is being written to

This is the demand that separates software-defined from merely configurable, and it carries the least forgiving invariant in the system: **never two writers for a shard, never zero.**

The protocol is a dual write with a content-based gate:

- The **old owner keeps committing** for the entire move. It is the sole committer throughout.
- The **incoming nodes join the fan-out** — receiving every write — but are held *out of the quorum denominator*, so a target still filling cannot stall or falsify a commit.
- Ownership **flips only when a frame comparison agrees**: the new owner's checksummed row ranges match the current holder's, across the whole table.
- The request row is **consumed by the flip**, which lets a node restarting mid-move read a still-present request as "resume" rather than "start again".

The gate is content-based on purpose. The obvious alternative — compare applied positions — was not available: the applied high-water mark is node-local and never travels on the wire. But position is the weaker statement anyway. *These rows are present with this content* is what you actually want to know before handing ownership over.

### Where the rows written *before* the move come from

The dual write carries new writes to the incoming nodes and does nothing for rows already there. The original design claimed reconciliation would cover that gap. It cannot: reconciliation converges rows both sides already know about, and its sweep derives its range from *local* bounds, which on an empty table are `(0, 0, 0)`.

**the same operation, working, with the verdict now readable**

```
[A3] yxtopo.public.rs_t backfill: 40 row(s) copied to bdc228b6
[A3] rs_t: 4ef9d487 matched (16 frame(s) agree, 40 row(s))
[A3] rs_t: 9b3a914c matched (16 frame(s) agree, 40 row(s))
[A3] rs_t: bdc228b6 matched (16 frame(s) agree, 40 row(s))
[A3] reshape COMPLETE: yxtopo.public.rs_t now owned by 0 node(s)
     after 14 round(s) - bdc228b6 matched (16 frame(s) agree, 40 row(s))
```

*A repair job over a table holding 200 rows, completing in zero seconds having examined zero of them. A node starting from nothing has nothing to compare, so a comparison-based mechanism has nothing to do. Bulk transfer is a separate capability, not an emergent one.*

So a bulk copy was built: `Copy` over a bidirectional `transfer()` stream. Rows travel as `to_jsonb(t.*)::text` — the identical representation and write path reconciliation already used and had proven — and the copy is **resumable by key, never by offset**, because `OFFSET` skips and repeats rows as the table changes underneath a long transfer.

Building it was the easy part. Three defects stood between `Copy` existing and a single row moving, and each presented identically: *the move simply doesn't happen*.

1. **An `Option` that meant three different things data loss**
   The function choosing which node to compare against returned `Option<Peer>`. `None` meant both *"this node holds it, read locally"* and *"no reference exists"* — and the caller read a missing reference as *local*. The migration monitor runs on the leader, and a leader very often holds no zoned table at all. So it framed its own empty copy, saw a count of zero, and reported:

   …on a cluster where the node it had just handed the table to held **zero rows**. Verbatim the failure the gate exists to prevent, reached through a different door.

   **Fix:** a four-state reference — `Local`, `Peer`, `Empty`, `Unreachable`. The last split is the one that matters: *every holder answered and every one holds nothing* is a statement about the cluster, and the flip is free; *nobody could be reached* is an absence of evidence, and the move must stay open. Both look like "no rows found". Only one is proof.

   ```
   [A3] reshape COMPLETE: yxtopo.public.rs_t now owned by 0 node(s)
        after 2 round(s) - bdc228b6 matched (16 frame(s) agree, 40 row(s))
   ```
2. **The backfill raced its own peer list, then gave up silently invisible**
   The monitor's first tick fires five seconds after start-up, so a move resumed from a still-present request fires while the peer list is empty. Every target resolved to nothing and each spawned copy simply returned — on the one code path whose two other exits both log. A move that skipped its *entire* backfill and a move with nothing to copy were indistinguishable.

   **Fix:** wait for the peer, and say so if it never arrives. Self is skipped before the wait, since a node cannot pull from itself.

3. **The copy's own writes were replicated back at the cluster hard stop**
   The one that actually stopped rows moving, and worth stating plainly: **the insert that lands a copied row fires that table's replication trigger, which turns the row into a fresh cluster write and sends it to the party — which is precisely the set of nodes that already have it, because the copy exists to fetch what they already hold.**

   The peer answers with a duplicate key, the trigger's cluster write fails, and the local insert fails with it. `ON CONFLICT DO NOTHING` does not save it — the conflict is on the *peer*. A 40-row backfill applied exactly one row and died.

   **Fix:** `SET LOCAL session_replication_role = replica` inside the apply transaction. `LOCAL`, not session-wide, because the connection comes from a pool and a session left with triggers disabled would silently stop replicating whatever ran on it next. The same pass made the apply one transaction per *chunk*; it had been opening a connection per **row**.

   ```
   Fail to modify peer [bd..0deac7] of party: 1/2: DB: DB error:
     ERROR: duplicate key value violates unique constraint "rs_t_pkey"
     DETAIL: Key (id)=(1) already exists.            ×40
   ```

Two findings fell out of the same hunt, both about observability rather than logic. The success line was logged at `info` while nodes run at `warning` — so "N rows copied" was invisible, and its absence looked identical to a copy that never started. And the Postgres driver's error `Display` is the literal string `db error`, so `copy apply public.rs_t: db error` was the *entire* diagnostic; learning that the real message was `yt_synchronize: Network: Timeout expired` meant reading the database log on the right node at the right second.

***Fourteen rounds.** The comparison examines one target per five-second tick, so the flip lands roughly seventy seconds *after* the data has arrived. "The rows are there" and "ownership moved" are much further apart than either a test or an operator would assume — unknowable while the verdict was logged below the level the nodes run at.*

With that, demand three is paid: a table's placement rule can be rewritten while the table is under write load, and the cluster moves the data, dual-writes throughout, and transfers ownership only once the new holders can *prove* they have it. That proof is the subject of the next part, and it is not a coincidence that it comes next — the gate in this section is the verification layer, borrowed.

---

### Part 05 Demand four — proof

## Why a mutable topology must verify itself

Verification is usually sold as a nice-to-have: a background scrubber, a consistency checker you run on Sundays. Under a software-defined topology it is structural. Once rows are scoped and mobile, "replication reported success" stops being evidence of anything, because the set of nodes that *should* have a row is itself a moving quantity. The migration gate in Part 04 is not a separate mechanism — it *is* this one, pointed at a question about ownership.

### Frames

A table is summarised as *frames*: ranges of the primary key, each carrying `{lo, hi, count, xor32, sum64, max_ts}`. The per-row basis is `to_jsonb(t.*)` minus ignored columns — chosen over a text cast because the text form is column-order dependent, and two nodes whose table grew columns in a different order would diverge on every row while being byte-identical in content.

The folds are XOR and SUM, both commutative, because two nodes never scan in the same physical order. And the split uses `ntile()` — **equal row count, not equal key width**:

> **Why equal-width ranges are the wrong primitive**  
> Real primary keys are gappy: deletes, sequence caches, per-node allocation ranges. Splitting a key *range* into equal parts gives wildly unequal row counts — one frame with everything, several with nothing. `ntile()` over the ordered key gives equal cardinality and exact boundaries in one index pass. Remember this decision: Part 07 turns out to depend on it entirely.
>

Comparison descends — frames that disagree are split until the differing rows are identified — then a vote is taken across the holders. A clear majority is repaired from. A tie is *not* guessed at: the row is blocked for writes and a critical alert is filed.

### The skip rule, or: don't call a moving target divergent

A row written seconds ago will legitimately differ between nodes. Two independent conditions defer a verdict rather than producing a false positive — **temporal** (the row was seen within a stabilisation window, configured as a multiple of the statement timeout) and **structural** (the last writer's subterm is incomplete, so the cluster itself has not finished with it).

After five deferrals a row is reported as *unverifiable under load* — a distinct state from *verified identical* and from *known divergent*. Collapsing those three into two is how a checker starts lying. One subtle bug lived here: re-queuing a skipped row reset its settle clock, so a row under steady write pressure could be deferred forever without ever reaching the threshold. Re-queue and first-queue are now separate operations.

### Verification that had never once run

Then the finding that reframes this entire section. Two parameter-binding bugs, one layer apart, meant **the repair path had never executed successfully in the history of the codebase.**

| Site | Written | Effect | Correct form |
|---|---|---|---|
| **Primary-key lookup** on the path of every round | `$1::regclass` | Driver infers the *parameter* type as `regclass`; every call fails with "error serializing parameter 0". No round ever ran. | `($1::text)::regclass` |
| **The repair write** reachable only once the first was fixed | `$1::jsonb` | The same trap one layer down. Detect, vote, descend and block all worked; every repair died at the final write. | `($1::text)::jsonb` |

> **Why years of green tests never caught it**  
> `0 rounds, 0 rows checked` is indistinguishable from *nothing needed repair*. Every unit test passed and always had — none of them can bind a parameter to a real PostgreSQL. And the blocking behaviour concealed the second bug *correctly*: after three failed repairs a row is blocked and a critical alert filed, which is the right response — but "blocked because repair failed" and "blocked because unrepairable" render identically. A system whose repair was **entirely non-functional** presented as one doing its job.
>
> The second bug was only reachable once the first was fixed. No amount of code reading produces that sequence; only running it against a real cluster does.
>

With both fixed, self-healing is demonstrated end to end: a row corrupted out of band on one node is detected, voted on, and rewritten from the majority, with no other node disturbed. A table nobody writes to heals on an age trigger alone. And the unrepairable case does the right thing:

**live: 50 rows, four partitions, ids deliberately gappy**

```
{"table":"yxtopo.public.pt_t","key":"id","partitions":4,"rows":50,
 "filter":"","ignore":[],"db":"yxtopo","stable_window_sec":10,"sharded":false}

{"part":0,"of":4,"lo":null, "hi":1372, "where":"id < 1372"}                 → 13
{"part":1,"of":4,"lo":1372, "hi":5103, "where":"id >= 1372 and id < 5103"}  → 13
{"part":2,"of":4,"lo":5103, "hi":10647,"where":"id >= 5103 and id < 10647"} → 12
{"part":3,"of":4,"lo":10647,"hi":null, "where":"id >= 10647"}               → 12
```

**a deliberate three-versus-three split vote**

```
2 health cerr  pk=   cerr pre-warn cur 2 +4.00/min eta 45s (warn@5)
3 recon  yxtopo.public.al_t  pk=3
    row is critical and now BLOCKED for writes
    - 2 distinct value(s) across 6 node(s), no majority
```

*No majority, so no guess: the row is blocked for writes and filed as critical to a node-local alert table, deduplicated, and resolved rather than deleted. The `health` row above it is incidental — and, as Part 10 records, was itself the cause of a false test failure.*

### The gate in front of the gate

Before any checksum is computed, a schema gate proves both sides have the same columns and primary key. Without it a single added column reports as *N divergent rows*, burying one structural problem under thousands of false data problems. The gate reports the difference, suggests the ignore-column set that would unblock comparison, and — importantly — an *unknown* verdict never blocks. A checker that cannot reach a peer must not thereby refuse writes.

---

### Part 06 The bill

## Nine defects, and what each looked like from outside

Four demands, paid. This is what paying them cost. Every entry was found by running against a live six-node cluster; **none** was found by the 231 unit tests, all of which passed throughout. They live in parameter binding, start-up ordering, transaction scope, and log levels — the seams between components, which is exactly where a test that stubs its dependencies cannot look.

| Defect | Presented as | Root cause | Class |
|---|---|---|---|
| Commit threshold vs. scope | Every scoped write aborts, citing a 20-digit node requirement | Threshold counted against the cluster, party filtered by scope; unsigned underflow at zero | correctness |
| Repair key lookup | `0 rounds, 0 rows checked` — reads as "nothing to do" | `$1::regclass` parameter-type inference | never ran |
| Repair write | `3 divergent, 0 repaired, 1 blocked` | `$1::jsonb`, the same trap one layer down | never ran |
| Reference collapse | Ownership flips onto a node holding zero rows | `Option` encoding three distinct facts | data loss |
| Backfill peer race | Move stalls; no log line at all | Monitor ticks before the peer list is populated; silent early return | silent |
| Copy fires replication | `copy apply: db error` after exactly one row | Apply insert re-enters the cluster write path; conflict is on the peer | hard stop |
| Registration rollback | `yt_complete: not connected`, naming no table | Trigger DDL and config write share a transaction; refusal on a non-leader rolls back both | half-install |
| Idleness read as lag | 6,613 demotions; every peer marked spare; all writes refused | Absence of recent writes indistinguishable from falling behind | outage |
| Dead suppression flag | Nothing — it silently did nothing at all | `SET SESSION yt.sync = 1` reads no defined setting; Postgres accepts any dotted name | decoy |

### The registration deadlock, in detail

One deserves expansion, because it is a genuine chicken-and-egg that shipped, and because it is the sharpest illustration of what a software-defined control plane costs when the control plane is stored in the thing it controls.

Registering a table for replication does two things: it creates a trigger (local DDL) and writes a config row. Once the config table is *itself* registered for replication — the normal end state — that second write **is a cluster write**, and on a node that is not the leader it is refused:

```
ERROR:  yt_synchronize: Config: This configuration update require call from cluster leader
CONTEXT: SQL statement "INSERT INTO public.yt_config(name, module, value) values ($1, $2, $3)"
         PL/pgSQL function yt_setup(...) line 131 at EXECUTE
```

The refusal aborts the statement, and the rollback takes the `CREATE TRIGGER` with it. The node ends up with **neither** — while the leader's copy of the config says the table is registered. Every subsequent write on that node fails with `yt_complete: not connected`, which names neither the table nor the reason. Which nodes were affected depended on *which node won the last election*.

Measured on a fresh six-node cluster: the bootstrap registration of the config table itself succeeded on five nodes and failed on one — leaving that one as the only node *without* a config trigger, and therefore the only node on which the *next* registration could succeed. One node in six could originate a write. The other five were silently inert.

> **Local is the correct scope, not merely the convenient one**  
> The fix suppresses the trigger for the config write and puts it in its own subtransaction, so it can never roll back the DDL. It also makes registration purely node-local — and that is the *right* semantics, not a shortcut. `CREATE TRIGGER` is local DDL and never replicates. Letting only the config row travel produced the worst of both worlds: peers holding a config row for a table with no trigger, half-registered and silently so. **A cluster that cannot elect a leader must still be installable.**
>

---

### Part 07 The payoff — query

## The structure that verifies is the structure that plans

Here the argument turns. Everything so far was cost: rules enforced at three points, a determinism constraint, a migration protocol, a verification layer. The return on all of it is that the cluster now *knows* something no ordinary database knows about itself, in a form another engine can consume.

Start with the intuition most people bring, because it is backwards.

On a **sharded** table a partition is a *placement fact*: row X lives only on node N, and the planner has no freedom. With `RING_HASH` it is worse than constrained — the ring is built from the live peer set, so the same query planned twice can slice differently, and a node joining mid-query changes which rows belong to which executor.

On a **non-sharded** table every holder holds every row, so a partition is a *scheduling decision*. Any node can take any slice. The partition count is independent of the node count. A node joining or leaving does not invalidate a plan. There is no ring, so there is nothing to be unstable.

> **Redundant copies are not an obstacle to parallelism — they are what makes it free**  
> An earlier analysis in this project concluded that full redundancy offered "no parallel-scan speedup" and that real MPP was gated on implementing sharding first. That reasoning was exactly inverted, and the inversion cost months of misdirected planning. Full redundancy *removes* the placement constraint from the scheduler. The hard case is the sharded one.
>

### The auto-partitioner was already there

Recall the `ntile()` decision from Part 05 — equal row count, exact boundaries, one index pass. That *is* an auto-partitioner. It had simply never had a way out of the process: everything a planner needed to slice a table already existed inside the cluster and was private to it.

It is now published. `yt_info('p:<table>[/<n>]')` returns a plan header plus one row per partition, each carrying an executable predicate:

**an incremental file that adds one column, read as "remove the primary key"**

```
Primary key on test_schema.test_table has changed from ["id"] to []
but with_index_drop is disabled. SQL would be:
  ALTER TABLE test_schema.test_table DROP CONSTRAINT test_table_pkey;
```

*The counts on the right are not from the plan — they are the result of *executing* each predicate. 13+13+12+12 = 50 by sum and by set-union. Those ids jump 7, 28, 63 … 17 500; an equal-width slice would have put nearly everything in the last partition.*

> *Diagram: Partition range diagram: four half-open ranges tiling the whole integer line, unbounded below the first boundary and above the last.*

*Two rules that look like pedantry and are not. The first and last ranges are **unbounded**, so a row inserted after planning still lands in exactly one partition; slicing `[min..max]` would silently drop it. And a boundary is the *next* bucket's first key, because the keys strictly between one bucket's max and the next bucket's min are real, addressable, and — with gappy ids — numerous.*

The header carries what a planner must apply *on top of* the range predicate, so it never needs a second surface to stay correct: the table's own filter, ignored columns, the source database, the stability window, and the `sharded` flag that answers Part 03's question — *is this dimension plannable at all?*

A companion surface answers the other half: *where* the table lives, and whether that answer is trustworthy. Its `complete` flag is the field a planner must actually check, because false means this node's view of placement is not authoritative and a fan-out built on it may miss data. That matters more than it sounds: **a planner that scans a node not holding the table reads the empty result as "no matching rows" rather than "not here", and under-reports without a single error.** A catalogue that cannot say "I am not sure" is worse than no catalogue.

> **Analytics must never be able to block transactions**  
> The layout handler *spawns* — the only one of its kind that does. The state-machine consumer is strictly serial by default, so an `ntile` full index scan run inline would stall every cluster commit behind it for its duration. The caller still waits on its channel; the state machine does not. The same instinct governs the sink in Part 08, pointed the other way.
>

What this buys, concretely: an engine can ask the cluster how to cut a table, get equal-cardinality ranges with executable predicates, learn which nodes can answer and whether that list is complete, and schedule the scan across them — using structure the cluster maintains anyway for its own verification. No separate statistics pipeline, no second catalogue to drift.

---

### Part 08 The payoff — reach

## Two engines, neither asked to be the other

The second half of the header's claim, and the one most often got wrong in practice: the temptation with a transactional cluster is to make it do analytics too, and with an analytical store to bolt transactions onto it. Both produce a system that is mediocre at both.

**ClickHouse is not transactional. Postgres is not an analytics engine.** The design takes that as a premise rather than a problem, and invests instead in *reach* — one way to address both, so neither has to grow the other's capabilities.

### Commit is an event

The analytical half runs on a different principle from replication. A table declares what should happen when a row commits, and the cluster dispatches it — no polling, no external change-data-capture process, no second copy of the schema to keep in step. Two sinks exist, and they compose.

The first is a **server function** in the Lambda / Azure Functions sense — not merely code that happens to run on the server. Functions are *stored*, versioned by checksum, bound to a table and a topic, given a queue and a priority, and invoked by the event rather than by a caller. You deploy a function, not a service: nothing to provision, nothing to scale, and no separate deployment that can drift from the schema it reads. The runtime is embedded in the cluster and speaks Python.

This is also where the `DF` binding earns its place. A function that receives a committed row can turn around and query the *whole* cluster through the MPP engine, using the partition layout of Part 07 — transactional event in, analytical answer out, without either engine leaving its lane.

The second is the **ClickHouse sidecar**: committed rows are converted to JSON and posted to ClickHouse's HTTP interface as `INSERT INTO <db>.<table> FORMAT JSONEachRow`, gated by a node-level switch so a single node can be designated the analytics feeder rather than all of them duplicating the write.

Two implementation choices worth noting. The HTTP client is hyper used directly rather than a higher-level crate — hyper is already compiled in through the gRPC stack, so the sink adds no dependency. And the queue is bounded at ten thousand rows, *dropping* rather than growing: an analytics sink must never become a source of back-pressure on the transactional path. The mirror image of spawning the layout handler in Part 07.

### A standard interface instead of a bespoke one

Both are reachable over a **standard etcd KV interface** the cluster exposes on its own RPC port — a deliberately unglamorous choice with a large payoff. etcd's protocol is already spoken by an enormous amount of existing software, so cluster state and queues are addressable without inventing, or asking anyone to adopt, a bespoke client. A queue prefix maps onto a ClickHouse table, so a writer that knows only how to put a key can feed the analytical store.

This is the same instinct as storing topology in config rows rather than a manifest: prefer the interface that already exists over the one you would have to teach people. A software-defined system whose control plane needs a custom SDK has moved the deployment problem rather than solved it.

### What the pairing actually yields

Two projections of one commit log, with different shapes answering different questions. The OLTP side stays transactional and is queried in place and in parallel, with the cluster telling the planner exactly how to slice it and which nodes can answer. The OLAP side receives the same committed rows as an append stream into a columnar store built for scans that would be unreasonable against the transactional cluster. Neither is a replica of the other in the replication sense.

What unifies them is not a shared storage format but that shared reach: the embedded function runtime can address both, and the etcd interface means a great deal of existing software can too — without either engine giving up what it is good at.

> **And the constraint that governs all of it**  
> MPP is designed for stabilised data only. The stability window is published in the plan header for exactly this reason — a planner slicing a table that is still settling will read in-flight divergence as fact. The cluster can say when a cut is safe; it cannot stop a caller from ignoring that.
>

---

### Part 09 The payoff — change

## Migration as the backbone of CI/CD

Part 02 made the cluster's *shape* software-defined. This is the other half, and it is the half that decides whether anyone can actually develop against the thing: **a schema change has to be verifiable in a pipeline and deployable to a running cluster, with no operator visiting each node.**

A cluster you cannot change safely is a cluster development stops at. Teams batch schema changes into a quarterly window precisely because each one is frightening, and batching makes the next one more frightening still. The way out is not more caution per change; it is making a change small, routine, reversible and — above all — *refused* when it is not safe.

Which splits the problem in two, and both halves have to hold:

- **CI** — a schema change is validated against a real database on every commit, and an unsafe one fails the build rather than reaching an environment.
- **CD** — a validated change rolls to a live cluster on its own, node by node, with the cluster refusing anything it cannot do safely.

The second gets all the attention. The first is what makes the second boring, which is the only state in which anyone deploys schema changes willingly.

### CI: what runs before a cluster ever sees it

Three mechanisms, none of them new infrastructure:

- **`prepare` is `migrate` without the writes.** The same code path, the same schema comparison, the same guards — it simply does not apply. That is what makes it a usable gate: a dry run down a different path proves only that the different path works. The same switch exists per-migration as a `dry_run` column, safe to set on a request just to see the expansion.
- **A version's content is frozen by checksum.** A script that has run successfully carries a content hash, and a changed file is not a new state of the same version — it is a different thing wearing a taken name, and it is refused. That is immutability enforced at the tool, not by convention in a review.
- **Destructive changes fail the build.** The seven migration options default to off, so an index drop, a trigger drop, a revoke, a type narrowing or an absent primary key raises an *error*, naming the table and the exact SQL it declined to run. In a pipeline that is a red build on the commit that introduced it — the cheapest possible place to find it.

The last one is not hypothetical: it is how the primary-key defect below was found, and it was found as a failing test rather than as missing constraints in production.

### CD: what the cluster does with it

### The table is the API

One config row turns it on: module `C`, `schema_guard_schema`, naming the schema that holds `sg_history`. Empty means the cluster watches nothing — migration scheduling is opt-in, for the same reason the ClickHouse sink is: a cluster that silently begins executing rows a tool wrote into a table is the worst possible default.

After that the entire operator interface is two SQL statements:

- **insert** a row into `sg_history` describing the migration
- **select** it back: child rows appear with `parent` set, or `error` is filled in

There is no RPC to call, no function to install, no CLI step and no agent to deploy. That matters more than it sounds: it makes the same interface usable by a human at a psql prompt, by CI, and by an automated agent that can only run SQL. The row *is* the invocation — every rdbm parameter that makes sense cluster-wide is a column, so reading the table shows the migration rather than a job id to go and resolve.

> **destination, and the fan-out**  
> `NULL` or `'cluster'` means *this* cluster — what an operator writes. A cluster uuid targets one cluster where a single table serves several. A **node** uuid is what the scheduler *writes*, never what you write: the leader expands a request into one child row per node, carrying the calculated per-node parameters and a `parent` pointer back at the request.
>
> Leader-only, because the request row replicates to every node — if each expanded it the cluster would get N copies of the plan. And a row that already has a parent is never re-expanded, which is the guard that stops an expansion expanding.
>

### Where a migration comes from

Scripts are sourced from a git repository, an http(s) archive, or a signed package. Git is the interesting one, because of what it lets the history answer: the resolved commit is recorded in the migration row, so the database can be asked *which revision produced this schema* — a question a directory path cannot answer and a deployment ticket answers only by convention.

Credentials never travel in the row. `sg_history` replicates to every node, so a password written there is a password published cluster-wide; the `*_secret` columns name a KV entry, resolved on the node that runs the migration. SSL certificate paths are excluded for a related reason — a filename is node-local, and `/etc/ssl/node3.key` means something different, or nothing, on node 5.

### The rule that makes continuous migration survivable

Continuous deployment of schema only works if the tool refuses to do damage by default. Two rules carry that weight, and one of them was found the hard way during this work.

**Absent means unchanged. SchemaGuard generates no DROP.** A YAML table definition is incremental: it names what should exist, not the complete desired state. Columns already worked that way — nothing generates `DROP COLUMN` for a column in the database and absent from the YAML. Primary keys did not, and the asymmetry was live:

*The schema file adds a single `tr int` column to an existing table. Because it does not restate the primary key, the desired PK computed as empty and the difference read as a removal. The fix is one clause — `!desired_pk.is_empty() && desired_pk != existing_pk` — so an absent declaration means *unchanged*, consistent with columns.*

Note *why* this surfaced as an error rather than as silent damage: the change was gated behind `with_index_drop`, which defaults to **false**. That is the second rule — **seven migration options, all defaulting to off, every one of them guarding a destructive class**: index drops, trigger drops, revokes, type narrowing. The engine raises rather than proceeding. Turning one on is a deliberate, per-migration decision, never a setting left enabled.

> **What fail-fast bought here, concretely**  
> Without it, every incremental schema file that did not restate a primary key would have silently dropped it, on every table, on every node. The guard turned a data-loss class into a build failure — and the failure message named the table, the old PK, the new one, and the exact SQL it declined to run.
>

### One loader, three consumers

Continuous migration also requires that everything agrees on what a table currently *looks like*. Three components were reading the information schema — the cluster's own consistency gate, the installer, and the migration tool — and each had grown its own query. Two of those were removed: the migration tool carried a strict-subset fork of the loader, and the installer carried a hand-written table query.

All three now call the same loader. The gate's opinion is what decides whether replication proceeds, so the installer and the migration tool had better be reading exactly the same thing — two components that disagree about what a table looks like will eventually disagree about whether a migration is needed.

### The window, and the rule that governs it

A rolling migration means the cluster runs a **mixed schema** for its duration: some nodes migrated, some not, application traffic served by both. There is no window in which this is untrue, and shortening it does not make it safe. Every migration rolled this way must be backward compatible — adding a nullable column, a table, an index, widening a type — or the application must already handle both shapes, deployed *before* the migration rather than with it.

Expand / contract is the standard way through, and it is three rolls each individually safe: expand (compatible), deploy code, contract (compatible). A rename is expand/contract with a copy in the middle. It is never a rename.

The tool cannot check this for you, and says so rather than pretending: whether a column is still read is a fact about application code, not about the database.

### Verified, and what is not yet wired

The migration tool was moved onto the cluster's own async loader and run against a live PostgreSQL 18, where the decisive evidence was a single line — `ADD CONSTRAINT test_table_pkey PRIMARY KEY (id);` present in both the expected and actual schema dumps after an incremental migration. The primary key survived. Every remaining difference in that run was a stale golden file: PostgreSQL 14 renamed the `public` schema's owner, and the schema engine improved its generated foreign-key constraint names.

Honestly stated: the planner and its guards are built and tested, the per-node execution loop is not yet wired to the write path, and the secret store the `*_secret` columns point at does not exist yet — which is why a signed package remains the route for a migration needing credentials. The interface is settled; part of the plumbing behind it is not.

---

### Part 10 Method

## The tests were green because they measured nothing

This section is the most portable part of the report, because none of it is specific to this codebase — and because it explains how a system could be this thoroughly broken while reporting health.

Every defect in Part 06 was invisible for the same underlying reason: **some signal was absent, mis-scoped, or actively lying.** The harness built to catch them had the same disease. Four instances, each costing a full diagnostic cycle:

1. **A digest comparison of two empty tables passes**
   `md5(NULL) = md5(NULL)`. The migration test printed **"east and west agree exactly"** for an entire day while both sides held zero rows. Not a weak assertion — an assertion that *could not fail*. Every digest comparison now refuses to run below a required row count.

2. **A killed run poisons the next one through two-phase commit**
   Clearing the cluster's own write-ahead state is not a full reset. A cluster write that dies between `PREPARE` and `COMMIT` leaves a prepared transaction behind — those survive a database restart *by design* and hold their table locks indefinitely. The next run's opening `DROP TABLE IF EXISTS` blocks, and the run produces **no output whatsoever**, which reads as a slow cluster rather than a wedged one.

3. **Restarting a container does not pick up a rebuilt image**
   Several rounds of "the fix didn't work" were the old binary. Confirmed by grepping the deployed executable for a string from the change and getting zero. That verification is one command and belongs in the loop, not in the postmortem.

4. **`set -euo pipefail` plus a non-matching `grep` in an assignment**
   Kills the script mid-run with no summary line — so a test that *found nothing* looks identical to a test that *never finished*. Harmless inside an argument list, fatal in a command substitution. This is how the migration test lost its final two checks on the very run that first went green.

Two further traps were about the cluster rather than the harness, and both made a test report the wrong failure. **Seed with single-row transactions** — a multi-row transaction replicates intermittently here, and the seed is *setup*, never the thing under test. And **never verify a seed on one node and assert on another**: the alert test checked one node (11 rows) then asserted against a different one (0 rows), reporting `expected [SPLIT-B] got []` — which reads as "the out-of-band update failed" when the truth was "the seed never arrived".

The pattern held to the very end. In the final round two failures appeared that were *correct about the cluster but imprecise about what they measured*: one raced a flip that legitimately lands seventy seconds after the data, and the other counted *all* unresolved alerts on a shared sink, so an incidental health warning was reported as a defect in the reconciliation alert. The product behaved correctly in both.

> **Two questions worth asking of any suite**  
> Before trusting a *passing* test: what would it take for this to fail? If the answer is "nothing, given the current data", it is not a test — it is a counted no-op. Before trusting a *failing* test: is it measuring the thing it names? Roughly half the failures in this effort were assertions, not defects.
>

### Observability as a first-class fix

Three separate times the correct response to a defect was not to change the logic but to make the system able to say what it was doing. The backfill result, the comparison verdict, and the refusal to start a requested move were all logged *below* the level production nodes run at. A move stuck for three minutes produced a config row, a hundred lines of activity, and **not one word about the blockage**.

The general form: a once-per-operation result that a human or a test needs to read must be logged at the level the system actually runs at, and state-transition lines should log *on change* — so a long healthy operation stays quiet while a stuck one says what it is waiting for, once.

### Three rules this left behind

Every defect in Part 06 was a failure to observe one of these, and they generalise past this codebase.

- **Enforce at every leak point, not at the entry point.** Four of the nine were a rule that held on one path and not another. A scoped system has more paths than an unscoped one, and they do not announce themselves.
- **Demand proof, not reports.** "Replication succeeded" is a hypothesis about two byte sequences. The frame comparison exists because the migration gate needed something stronger than a position — and that same machinery turned out to be what an MPP planner wanted. Verification paid for itself twice.
- **Log at the level you actually run at.** A subsystem that cannot explain why it is waiting will eventually wait forever in production, and you will hear about it from a customer rather than from a dashboard.

---

### Part 11 Conclusion

## What a software-defined topology is worth, and what it charges

The claim at the top of this article was that replication scope and row placement can be rules rather than deployment, rewritten at runtime, with the cluster reconciling itself — and that a cluster shaped this way is both scalable beyond one node's capacity and, unexpectedly, *easier* to query in parallel than a conventionally sharded one.

That claim now rests on a hundred live checks rather than on design intent. The four demands are paid: the rule is enforced at every point a row could leak, placement is deterministic where it must be and honestly flagged where it is not, a table's rule can be rewritten under write load with ownership transferring only on proof, and the system verifies itself rather than trusting its own success reports.

| Scenario | What it proves | Checks |
|---|---|---|
| `15_setup` | Registration succeeds on every node, leader or not | 14 |
| `20_flat` | An unrouted table still replicates to all six — the regression gate | 6 |
| `30_az` | A minority-zone write is accepted, lands in the zone, reaches nobody else | 7 |
| `40_tier` | `no_up` keeps a tier-1 table out of tier 0 | 6 |
| `50_shard` | Each row lands only in its owner's zone | 18 |
| `60_write_az` | A zone may hold a table and still be refused as a write origin | 5 |
| `70_reshape` | A table moves zones losing no row and duplicating none; bulk copy moves the pre-existing rows | 9 |
| `80_recon` | An out-of-band corruption is detected and repaired from the majority | 6 |
| `85_alert` | An unrepairable split vote is blocked and filed as critical | 13 |
| `88_quiescent` | A table nobody writes to still heals, on the age trigger alone | 4 |
| `95_partition` | Partition predicates, executed, tile the table exactly once | 12 |
| `rdbm` suite | Schema migration end to end against live PostgreSQL 18; the primary key survives an incremental migration | live |

### What actually matters in production

Placement and partitioning are the interesting engineering. They are not the reason anyone would run this. The reason is a triad, and it only works as a triad:

- **A software-defined replication *level*, per table, on a strict-to-tolerant spectrum.** One table demands every holder acknowledge before the transaction returns. Another is content with a majority. A third wants a floor of *n* nodes. A fourth is a cache and may replicate lazily or not at all. This is a config row, not a deployment, and it is set per table because importance is a property of data, not of clusters. The same cluster carries both the ledger and the scratch table without compromising for either.
- **Real two-phase commit underneath it.** A strictness level is a promise about a boundary, and it is worth nothing without an atomic boundary to promise about. The write prepares across the party and commits only once the level is satisfied — so "committed" means committed everywhere it was required to be, not "sent and probably fine". The prepared-transaction machinery in Part 09 is visible in this report precisely because it is real.
- **Schema change as a continuous, refusable operation.** A migration is a row an operator inserts, expanded by the leader into per-node work, sourced from a git revision the history records. Every destructive class — index drop, trigger drop, revoke, type narrowing, and now an absent primary key — is off by default and raises rather than proceeding. Continuous *integration* of schema — validated on every commit against a real database — is what makes continuous deployment of it boring. Both halves only work because the tool's instinct is to refuse. See Part 09.
- **Configurable monitoring, with self-healing as an option rather than an assumption.** Per table: verify or don't; repair from the majority or only report; how many attempts before giving up; how long data must settle before disagreement counts as divergence. Automatic repair is right for some tables and wrong for others, and the system should not decide that on the operator's behalf.

The reason those three belong together is what happens when they meet a real fault. For data marked important — strict level, verification on, repair enabled — a divergence that cannot be resolved by majority does not get papered over and does not get guessed at. **The row is blocked for writes and a critical alert is filed, and the system declines to keep running on data it cannot vouch for.** That is the guarantee, and it is the opposite of the usual failure mode, in which a cluster stays cheerfully available while quietly accumulating divergence that surfaces months later as an unexplainable balance.

The strict-to-tolerant spectrum is what makes that affordable. Blocking on every table would make the cluster brittle; blocking on none would make it untrustworthy. Letting the DBA say which tables are which — in a row, changeable at runtime, like everything else here — is the whole point.

### Scope of this report

Three things are deliberately outside it, and none is a debt against the architecture.

- **Distributed execution is DataFusion's job, not the cluster's.** The cluster publishes placement, partition boundaries and stability; it never forwards data for someone else's query. That division is deliberate — even a fully meshed sharded cluster should not become a query router, and DF exists precisely to do this. What remains is packaging: the client is not yet in the distribution, so the Python binding stays inert until an engine is stood up beside it.
- **Fault injection and sustained concurrency are a separate programme.** Every scenario here is a happy path — no node killed mid-write, no partition, no leader loss under load, all writes serial. Hardening against that class is what the architecture in this article is *for*, and it is watched continuously rather than proven in a single pass; measuring it properly deserves its own report rather than a footnote in this one.
- **One migration stall remains under observation.** A move once refused to flip, survived a three-minute wait, and has not recurred across repeated runs since. The instrumentation added in Part 09 means a recurrence will name its own cause, which is the useful state for something that happens rarely and cannot be summoned on demand.

> **The finding worth carrying elsewhere**  
> Of nine defects, *none* was found by a unit test, and all 231 passed throughout. They lived in parameter binding, start-up ordering, transaction scope, and log levels — the seams between components, which is exactly where a test that stubs its dependencies cannot look. The six-container harness paid for itself with the first bug it found; its own four defects were worth finding too. **A test that cannot fail is more dangerous than no test, because it is counted.**
>

The cluster can now answer one question about itself that it could not answer before: *where do these rows live, and can you prove it?* Every hard problem in this report — quorum arithmetic, migration ownership, divergence repair, query planning, the analytics handoff — turned out to be a variant of that single question. Getting a system to the point where it can answer it honestly is most of the work. Getting it to admit when it cannot is the rest.

---

`Six-node Docker lab · PostgreSQL 18 · ytserv 0.16.0 · ytpgc 0.5  100 scenario checks · 173 + 58 unit tests · 2026-09-20`