# What replication actually costs: YaXaHa cluster vs PostgreSQL streaming replication

Everyone quotes the write-throughput drop when you turn on replication. Almost
nobody says where the time goes, or what the number looks like under a workload
that resembles production.

So I profiled a 3-node YaXaHa cluster against PostgreSQL's own synchronous
replication — same hardware, same network, same workload — and instrumented both
sides until every millisecond was accounted for. Every figure below is measured.

## Headline numbers

Write-only, single client. The base for comparison is PostgreSQL built-in
**synchronous** replication to two standbys (`synchronous_commit = on`,
`fsync = on`, both standbys flushing WAL to disk before the primary returns).

| configuration                     | ms | tps | vs PG sync                                            |
|-----------------------------------|---|---|-------------------------------------------------------|
| no replication at all (reference) | 1.42 | 704 | 3.1x faster                                           |
| PostgreSQL **async** replication  | 1.86 | 539 | 2.4x faster                                           |
| **YaXaHa WAL only eventual**      | **2.58** | **388** | **1.7x faster**                                       |
| PostgreSQL sync, 1 standby        | 4.03 | 248 | 1.1x faster                                           |
| **PostgreSQL sync, 2 standbys**   | **4.41** | **227** | **baseline**                                          |
| YaXaHa synchronous 2PC            | 18.26 | 55 | **4.1x slower** <br/> *tl;tr*: needs 3+ nodes to beat |

Now the same systems under an 80/20 read/write mix — the realistic shape:

| configuration | overall tps | read ms | write ms |
|---|---|---|---|
| no replication (reference) | 2360 | 0.153 | 1.471 |
| **YaXaHa WAL only eventual** | **1509** | 0.075 | 2.686 |
| PostgreSQL sync, 2 standbys | 933 | 0.196 | 4.366 |
| YaXaHa synchronous 2PC | 263 | 0.179 | 16.134 |

Reads on a cluster node are effectively free — 0.179 ms with replication against
0.153 ms without. All configurations were verified convergent: identical data on
all three nodes across 14 runs spanning 1, 4 and 8 clients.

## What was measured

Three nodes: one transaction origin running PostgreSQL 16, two peers running
PostgreSQL 18. Workload is `pgbench` at a small scale, one transaction = two
UPDATEs against the two hottest rows — deliberately the worst case for
contention. Every run began from a wiped cluster WAL, a synchronized restart, a
verified leader, and resynchronised tables confirmed identical before the run
started.

### The hardware is modest — and that is the point

I should be straight about the test rig, because the absolute numbers are
constrained by it:

| | |
|---|---|
| Origin node | Intel i5-9400F @ 2.9 GHz, **1 core available**, 7 GB RAM |
| Peer nodes | Intel i7-4790K @ 4.0 GHz (2014-era, 4C/8T), 14 GB RAM |
| Network | 1 GbE, **ICMP round-trip 0.75 ms** (a healthy LAN is 0.1–0.2 ms) |
| Virtualisation | **none — bare metal** |

These are consumer desktop CPUs, one of them a decade old, and the origin node
is running the client, the database and the cluster daemon on a single core. The
network is 4–8× slower than it should be, and that is latency rather than
bandwidth: the messages are 150–200 bytes, so serialization costs 1.6 µs on
1 GbE. The 0.75 ms is NIC, switch, driver and power management.

The upside is that **none of it is virtualised**. There is no hypervisor
scheduling, no noisy neighbour, no burst credits quietly expiring mid-run — the
numbers are stable and repeatable, which is exactly what you need when you are
attributing milliseconds. Better hardware would lift every absolute figure. It
would *not* change the ratios much, with one important exception explained below.

### Why the slow network flatters PostgreSQL

Latency penalises the two systems asymmetrically. PostgreSQL crosses the wire
**once** per commit. The cluster crosses it **four times**. A slow wire therefore
costs the cluster four times as much, and every ratio above is measured on a
network that widens the gap. On a properly provisioned LAN the cluster closes
part of it for free — the wire is roughly half of every hop.

### Conservative by construction

Every table in these runs is configured with sequential consistency **disabled**:

| setting | meaning |
|---|---|
| **off (used here)** | ordering between *different* rows is tolerated; no conflict rejection |
| on | conflicting concurrent writes are rejected outright (SQL 40409) |

This is the conservative floor — none of the cluster's opt-in optimisations are
armed. The figures are a baseline, not a best case.

## Where the time actually goes

Per write statement, on the origin:

| stage | eventual | synchronous 2PC |
|---|---|---|
| cluster bookkeeping, total | **0.16 ms** | **3.26 ms** |
| ├ leader check-in | 0.12 | 0.27 |
| ├ local WAL | 0.03 | 0.04 |
| └ fan-out to peers | 0.00 | 2.95 |
| PREPARE (per transaction) | 0.00 | 3.62 |
| COMMIT (per transaction) | 0.09 | 4.10 |

Synchronous commit makes **four sequential remote round trips** per transaction —
two fan-outs, then PREPARE, then COMMIT — accounting for 13.6 ms of the 18.3 ms
measured. Eventual mode makes **none**, which is the entire difference between
2.58 ms and 18.26 ms.

PostgreSQL does the same durability job in **one** round trip, and ships to both
standbys **in parallel**. That is why one standby (4.03 ms) and two standbys
(4.41 ms) cost almost the same: the second replica is nearly free. The cluster's
round trips are sequential, so each one is additive. The 4:1 round-trip ratio is
the 4.1:1 latency ratio — it really is that simple.

A consequence worth internalising: cluster cost scales with **statements, not
transactions**, because the leader is consulted per write statement. Measured at
one, two and three statements per transaction: 22.7 / 39.3 / 44.1 ms, fitting
roughly a fixed commit cost plus a constant per statement.

### Is the overhead network, serialization, or the async runtime?

Isolating a single hop: 2.21 ms of origin wait for 0.75 ms of actual work on the
far side, so **1.46 ms of transport per round trip**. That splits as:

| component | per round trip | share |
|---|---|---|
| wire (measured round-trip time) | 0.76 ms | **52%** |
| gRPC / HTTP2 / async scheduling | ~0.70 ms | **48%** |
| protobuf serialization | 0.0022 ms | **0.15%** |

Roughly **half network, half our own stack — and serialization is a rounding
error**. A 149-byte message costs 0.40 µs to encode and 0.70 µs to decode, so all
four operations in a round trip total 2.2 µs. I had assumed serialization would
be worth optimising. It is not, by three orders of magnitude.

The local path tells a similar story. With propagation disabled entirely, the
write trigger still costs four local socket calls per transaction at ~415 µs
each, and **96% of that sits inside the HTTP/2 client** — about eleven syscalls
per call, a connection-reuse preamble, and a split write for headers then data.
gRPC over HTTP/2 is heavy machinery for talking to a daemon on the same machine.

## Conclusions

**Write-only throughput is the wrong lens.** Turning on replication takes a
standalone PostgreSQL from 704 to 227 tps, and synchronous cluster commit to 55.
Those are write-only numbers, and writes are typically around 20% of a real
workload. Under an 80/20 mix the same systems measure 2360 / 933 / 263 tps.

But the honest version has a second half: 80% of *transactions* being reads is
not 80% of the *time*. With writes 90× more expensive than reads, writes still
consume 96% of the elapsed time. Both halves have to be said together, or you
are selling something.

**Every node accepts reads and writes.** A PostgreSQL standby is read-only and
cannot take a write under any circumstances, so an application cannot simply
read from one host and write to another — that split has to be designed into the
application. In the cluster all nodes are identical and each can hold application
connections for both. So the comparison is not "weaker durability for speed", it
is **three writable nodes at 388 tps against one writable node at 227 tps**. What
eventual mode gives up is read freshness on the peers, not write safety: a peer
serving a stale read still blocks before writing those rows.

**Swapping PostgreSQL replication for cluster WAL is 1.7× faster.** If you want
honest master-to-master 2PC instead, you need three servers to break even — which
happens to be the recommended minimum for a cluster anyway. The remaining 4.1×
gap is structural, not a tuning problem: one parallel round trip versus four
sequential ones. Closing it is a protocol change.

**What the cost buys is configurability, not throughput.** Synchronization is
configured in software per table rather than fixed by the replication topology,
which makes more complex network arrangements expressible. Alongside that sit
[RPPD](https://vkrinitsyn.github.io/rppd) for Python stored procedures and an
[etcd](https://vkrinitsyn.github.io/etcd)-compatible distributed cache. If you
need a single writable primary and nothing else, PostgreSQL streaming
replication is faster and you should use it. The trade only pays when you need
every node writable, or per-table control over how far each write must travel.

---

*Measured on bare-metal hardware described above. Methodology, raw figures and
the instrumentation are available on request.*
