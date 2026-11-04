# Quality gates and roadmap

All numeric values here are **proposed engineering targets**, not measured results or capacity promises. M0 must establish hardware, workload and budget before accepting or revising them.

Revision: 4 November 2026, following the [earlier review](documentation-review.md) and [Database System Concepts review](database-system-concepts-review.md). The minimum distributed core is defined by [scenario evidence](distributed-contracts.md#mandatory-scenario-evidence), not by the number of checked feature boxes.

## Scope tiers

| Tier | Required scope |
| --- | --- |
| M0 feasibility | Confirmed native-storage ownership; first-party LSM/B+tree prototype comparison; disk-format and durability RFC; qualified primitive libraries; Falcone compatibility inventory; transaction/shard/persistence models; reproducible harness |
| M1 local alpha | First-party WAL/table/manifest/cache/compaction implementation with crash and clean-directory restore; integrated local MVCC; bounded SQL subset and execution; single-node serializable behavior; authenticated tenant isolation; JSON management; driver smoke certification |
| M2 distributed alpha | Three-domain replicated shards; cross-shard transactions/constraints; failover; fenced split/move and small-to-large growth; distributed plans; consistent snapshot and clean-cluster restore; internal change envelopes; load admission and connection scale |
| M3 constrained beta | Named spatial subset; managed S3 objects; coordinated backups/PITR; committed CDC; Falcone integration; supported-workspace migration rehearsal |
| M4 production candidate | Independent correctness/security review; fault campaign; sustained workload/failover tests; verified restore/upgrade; published compatibility/SLO limits |
| M5 expansion | Multi-region/locality; object-backed analytical tier; search/ANN; advanced SQL/GIS, development branches and polished administration UI/CLI |

Do not move a user baseline requirement into an indefinite future: native sharding and high client concurrency are M2, SQL-free administration starts at M1, and geospatial plus S3 large objects are mandatory for M3. Full compatibility with top-tier engines is not a realistic first-release criterion; the supported relational subset must still satisfy integrity and recovery guarantees.

Native storage and format ownership is also a baseline requirement, delivered from M1 onward. The [storage-specific gates](native-storage.md) and E19–E23 cover implementation work formerly delegated to an embedded engine. E02 verifies integration of those components with row/index/MVCC semantics; it is not a second storage implementation.

P0 covers the core protocol, execution, security, API, recovery and evidence requirements. P1 covers required product capabilities for M3. P2 covers expansion. E09-S01–S03 are P0 API/schema foundations; E09-S04 is P2 presentation work. E12 now spans M1–M3: S3 objects are a dependency only for object-inclusive backup, not for local/row-only restore. E13's commit/schema/topology envelope is designed and exercised internally by M2; external delivery remains M3. Public follower reads remain deferred, but consistent internal cuts are mandatory.

## Critical path and gate ownership

| Gate | Work that must already be real | Failure prevents |
| --- | --- | --- |
| M0 protocol/layout decision | E01-S02/S03 plus E17 harness; recovery, lock/commit, snapshot frontier and ownership transition models; pinned workload and primitive decisions | Freezing file format, choosing concurrency optimizations or promising capacity |
| M1 durable vertical slice | E19–E23 local slices, E02, E04-S01, E05 core, E07/E08/E09 API slices, E12 local restore | Claiming a durable local relational engine |
| M2 distributed vertical slice | E03/E20, E04 distributed/snapshot slices, E06, E05 distributed execution, E07 admission, E12 distributed restore, E13 envelope and E16 operations | Calling the database scalable/distributed/sharded based only on multiple processes |
| M3 product completeness | Geo, S3 reference lifecycle and object-complete backup, CDC delivery, Falcone integration and supported migration rehearsal | Claiming the original Falcone baseline is complete |
| M4 production evidence | Independent reviews, mixed-version/recovery/fault/soak campaigns, certified workload and limits | Supporting production durability/SLO promises |

Dependency edges refer to these slices. E09's local API does not wait for E03's distributed catalog; E12's local restore does not wait for E11. No milestone assumes implementation completeness merely from an earlier design document.

## Benchmark plan

Proposed reference profile: one region, three failure domains; 3 metadata nodes (4 vCPU/8 GiB each), 6 storage nodes (16 vCPU/64 GiB, SSD/NVMe, at least 1 TB each), separately scaled query nodes (16 vCPU/32 GiB), and gateways (8 vCPU/16 GiB). Record real disk durability, CPU models, TLS termination, load balancer, kernel limits, network RTT/bandwidth and virtualization. This is a test profile, not a minimum supported installation or a production sizing quote.

| Dimension | Initial target / evaluation | Mandatory reporting |
| --- | --- | --- |
| Idle clients | 50,000 authenticated TLS PostgreSQL sessions per gateway; 100,000 is a stretch | Process RSS plus kernel/TLS/socket memory, FDs, accept latency, keepalive traffic, credential refresh |
| Aggregate connections | 500,000 across 10 gateways; 1 million a later stretch | Simultaneous open clients, number of load generators, reconnect churn and LB limits; no HTTP stream/socket conflation |
| Admission | 1,000 active requests per gateway offered load, execution capped by budgets | Queue time, actual running concurrency, rejection rate, memory, tenant fairness |
| Latency | Hypothesis: p99 under 50 ms for indexed single-shard point reads and under 100 ms for small cross-shard transfers | Same workload/data/RF/TLS/durability settings; not asserted for joins, remote regions or arbitrary contention |
| Scale-out | At least 1.6x sustained throughput from 3 to 6 storage nodes on evenly distributed independent-key work | Same per-node resources and RF; include rebalancing/compaction; report hot-key workloads separately |
| Failover | Hypothesis: affected-shard p99 recovery under 30 s after one voter loss | Detection, election, recovery, client retry; report duration of actual write unavailability |
| Durability | Zero lost acknowledged transactions in tested single-failure/quorum-preserving scenarios | Histories, checksums, seed/config and injected fault schedule; excludes untested correlated loss |
| Backup/DR | Proposed archival-lag RPO goal at most 60 s; RTO goal under 60 min for a defined 100 GiB dataset | Record backup bytes, object bytes, bandwidth, verification duration and achieved restore point; not a universal SLA |
| Fairness | At least 90% of an admitted reference tenant's baseline throughput during a noisy-tenant test | Resource reservations, offered loads, queue limits and overload behavior |

Connection capacity math must include all layers. Even 32 KiB of incremental userspace state per session is about 1.53 GiB for 50,000 sessions, before socket buffers and shared caches. Measure it rather than assuming the language makes connections free. Connected users, transactions per second and simultaneous queries are different metrics.

Run: read-heavy and balanced point workloads; insert-heavy sequential/random keys; bank-transfer invariants; skewed hot rows; tenant skew; joins with/without colocation; range scans; large result/slow-reader cases; repeated connect/disconnect; small and large blobs; mixed analytical/OLTP contention. Data must exceed cache in at least one scenario. Benchmark with synchronous durability, replication and TLS enabled. Publish p50/p95/p99, throughput, error/abort rates, queue times, resource use, compaction and cost per workload; do not publish a standalone flattering TPS number.

Add storage-only baselines before SQL benchmarks: equivalent durable batches and ordered lookups against a simple reference model and separate benchmark installations of established engines. Record flush/fsync latency, WAL bytes per logical byte, compaction write amplification, live/retained/temp space, cache hit rate, file count and recovery throughput. Compare the first-party LSM/B+tree prototypes on equal correctness/settings; a competitor engine must not become a hidden fallback in release binaries.

### Measurement protocol added by the book review

1. Publish a workload manifest: row/index sizes, key/tenant skew, read/write ratio, transaction participants, result sizes, dataset/cache ratio, connection churn, history retention and background jobs. Keep RF, fsync, TLS and per-node resources fixed for comparisons.
2. Sweep offered arrival rate independently from completion rate. Include client queueing, timeout/error counts and retry traffic; a stalled closed-loop generator must not hide waiting requests. Record both open-loop saturation and realistic bounded-client runs.
3. Measure end-to-end response distributions with enough observations and duration to support each reported percentile. Merge histograms across workers; do not average p99 values. Report repeated-run variation, not just the best run.
4. Separate gateway scaling, query-compute scaling, storage capacity/throughput scaling and metadata/timestamp scaling. Run fixed total data and constant data-per-node variants to expose cache effects. Record logical/physical bytes and replication overhead.
5. Measure stable pre-growth, active rebalancing and stable post-growth periods separately. Include affected-range admission pause, backlog, temporary disk use and repair bandwidth. A 1.6x target is evaluated for its named independent-key workload, not for one hot row or arbitrary cross-shard joins.
6. Increase idle and active shard counts independently. Measure heartbeat traffic, per-group memory, file count, descriptor/watch load, restart/election behavior and leader distribution. Set a supported range-count envelope from results.
7. Combine compaction, backup, DDL backfill and movement with foreground queries. Verify reservations protect consensus and outcome recovery. Test sustained overload and return to normal load without requiring a cluster restart.
8. Record client-visible availability under node/zone loss, coordinator failure and metadata outage separately. Distinguish safety (no invalid outcome) from liveness (progress under stated quorum/resource assumptions).

The books motivate these dimensions (B1 chs. 4/14/15; B2 chs. 2/7/9; B3 chs. 5/7/8). The specific thresholds are Falcone hypotheses. Before a public scale claim, E17-S03 must link every claim to a reproducible run and state the largest tested topology and workload.

## Correctness and reliability gates

- Deterministic simulation from the first storage/transaction milestone: controlled clocks, network, disks and scheduler; save failing seeds. Model-check transaction decisions, topology transitions and GC/pin races separately.
- Property tests/fuzzing for SQL/protocol parsing, key encodings, type conversion, index maintenance and authorization. Differential SQL tests against PostgreSQL cover only the declared common semantics.
- History-based distributed checking: lost updates, write skew, phantoms, concurrent global uniqueness, FK races, fractured reads, commit ambiguity and leader changes. Linearizability checks apply to APIs that actually promise it; do not equate them with every SQL transaction history.
- Crash at every durable boundary: prepare, commit decision, apply, index backfill, shard cutover, snapshot publication, log truncation and object deletion. Verify no half-transaction or acknowledged-data loss.
- Partitions, message reordering/duplication, long pauses, disk full, torn/corrupt data simulation, slow disk, clock steps, stale routing, metadata failover and node replacement. FFI and real-storage behavior also need tests beyond simulation.
- Cross-tenant negative tests across SQL, JSON, gRPC, native protocol, indexes, errors, object URLs, CDC, backups and restored data. Test credential/policy changes during long sessions.
- Backup restoration into a clean cluster, not just successful uploads; verify relational invariants and every protected object. Test KMS/Secrets recovery dependencies and key rotation.
- At least a 7-day mixed-workload soak and repeated restart/upgrade campaigns for M4; longer campaigns as volume and risk justify. No unresolved critical correctness/security issue passes a release gate.
- Native-format golden fixtures and fuzzing cover every file kind, feature bit, length/offset and checksum; an independent reference decoder/model checks canonical encodings. Power-loss simulation includes dirty OS buffers, short writes, failed syncs and rename/directory durability. `kill -9` alone is not power-loss coverage.
- Verify manifest publication/reclamation, partial flush watermarks, WAL segment gaps, torn tails, corrupt committed data, compaction tombstone resurrection, cache identity reuse and double-log replay fences. Unsupported versions/corruption fail explicitly; no fallback to an older root may silently lose acknowledged writes.
- Check mixed-version read-modify-write preservation of unknown optional fields, comparator/collation activation, delayed pre-restore messages, request-dedup expiry, and split cancellation versus activation. Feature negotiation must include restore/inspection tools as well as online servers.
- Test external sort and hash join/aggregation beyond memory, concurrent memory grants, all-equal join keys, empty/null inputs, early cancellation and spill disk exhaustion. A query that stays within its admitted budget must not be killed by another query's unaccounted allocation.
- Compare replicas at a common logical/applied cut; quarantine divergence and repair from validated history. Do not resolve corruption by last-write-wins or a majority of arbitrary current values.

## Operations and observability

Metrics: active/idle sessions, queue time, admission drops, transaction latency/aborts/recovery age, shard hotspots, replication lag, fsync and compaction, snapshot/CDC age, blob lifecycle/GC backlog, backup archive lag, verification and restore durations. Avoid an unbounded Prometheus label per tenant/query; aggregate and expose scoped detail through authenticated APIs/logs.

Add lock bytes/waits, oldest prepared transaction, safe-frontier lag/blockers, dedup retention/expiry, metadata/timestamp queueing, directory-watch backlog, range-count overhead, split state/pause, reserved versus used memory/temp space, operator spill/exchange bytes and retry amplification. Readiness must distinguish a process running from its authority and ability to serve the requested operation.

Use OpenTelemetry traces across gateway/query/shard/object operations with parameter values redacted. Query fingerprints, sampled explain plans, slow-query budgets and index statistics support diagnosis. Separate service availability, data durability, backup freshness and capability health in readiness dashboards.

Kubernetes support includes anti-affinity/topology spread, persistent identities/volumes, PodDisruptionBudgets, bounded drains, storage-aware scale-in and quorum-aware upgrade sequencing. HPA can scale stateless gateways; stateful scale-down requires engine orchestration. Never let a generic CPU autoscaler delete quorum members independently.

Runbooks cover lost quorum, bad disk, metadata restore, unknown transactions, runaway query, replication/CDC backlog, S3 outage, failed upload, accidental logical deletion, tenant export and key loss. Dependency outages must not convert accepted provisioning into a false “active” state.

## Delivery capacity and decision gates

This is a new distributed database product with a first-party storage engine and file format. A credible team needs persistent-format/WAL/recovery, cache/compaction/I/O, consensus, transactions/query processing, Rust/networking, SQL compatibility, reliability/testing, and Falcone integration expertise. Some roles can overlap, but correctness review must be independent of the author of the protocol.

A production-grade engine is a multi-stage, potentially multi-year effort; agent-generated code does not replace protocol validation or operating evidence. Do not commit a release date until M0 measures native-storage feasibility, recovery costs and compatibility scope. Owning persistence adds a critical path for format, WAL, files, cache, compaction, recovery and upgrades. If delivery constraints tighten, reduce SQL/expansion scope or stage milestones; do not silently replace the confirmed ownership requirement with a third-party engine.

Open decisions with recommended defaults:

| Decision | Proposed default | Close by |
| --- | --- | --- |
| Storage ownership | Confirmed: first-party Rust engine and Falcone disk format; external engines are reference/benchmark tools only | Decided by user |
| Physical layout and format | First-party LSM proposed; compare first-party B+tree; freeze a versioned disk-format RFC and recovery model | M0 |
| Primitive libraries | Qualify I/O, checksum/compression/crypto, codecs and consensus core; Falcone owns persistence integration | M0 |
| Source license | Apache-2.0 proposal; owner acceptance required before adding a license grant | M0 |
| Transaction method | Strict 2PL + MVCC + replicated 2PC; optimize only after correctness evidence | M0 protocol review |
| First SQL/client profile | Falcone Node `pg` workload and defined SQL subset | M0 |
| Initial geography | WGS84 points; planar geometries; exact semantics with conservative index | M0 API freeze |
| Large objects | Explicit `object_ref`, immutable keys, verified seal, database-owned retention | M0 API freeze |
| First production topology | Single region, three failure domains | M0 deployment profile |
| Workspaces requiring extensions | Keep PostgreSQL until individually certified | Migration gate |

These are proposal choices, not a request to delay design work pending approval. Implementation should resolve them with the evidence specified by E01.


## Fourth-book delivery map

The backlog now has 27 epics/108 stories. E24–E26 refine the implementation under existing integration gates; E27 and E25-S04 are optional M5 capabilities. Dependencies are interface/milestone slices, not circular requirements to finish all of E05 before any E24/E25 work.

| Gate | Required additions | Evidence |
| --- | --- | --- |
| M0 | Freeze catalog/identity/constraint/JSON profiles; model sequence fencing, lock hierarchy and durable job progress | Protocol assumptions, feature matrix and counterexamples; E01/E24/E26 design slices |
| M1 | E24 local descriptors/views/identity/constraint behavior; E25-S01/S03 local statistics and plan validity; E26-S01 initial streaming subset | Restart, failed-statement, NULL/bag/JSON and revoked-plan tests within local limits |
| M2 | E24 distributed authority/dependencies; E25 distributed collection/physical plans/cache validity; E26-S03 online native index build foundations | Leader loss, unequal shards/splits, outer-join ownership, covering lock conflicts, private/public index recovery |
| M3 | E26 operational import/export and full certified codec/build integration; original geo/S3/CDC/Falcone requirements | Crash-resume batch evidence, verified export/reimport, object completeness, migration and source-sequence reconciliation |
| M4 | Production qualification includes all required new slices | No unresolved critical semantic/security/recovery defects; bounded plan/cache/job resource use under faults |
| M5 | E25-S04 runtime filters and E27 materialized generations/qualified incremental subset | Exact-query/full-refresh comparison, false-negative tests, freshness/policy enforcement and total workload cost |

Add targeted semantic datasets that distinguish a correct plan from common wrong rewrites: duplicate-valued EXISTS, NULL-aware NOT IN, empty/global aggregates, outer rows matching only on another worker, scalar subqueries with multiple rows, and UPDATEs that change their own scanned key. Validate post-transaction database state, not only returned rows. Every comparison uses the declared common semantics rather than assuming all PostgreSQL syntax is supported.

Measure **speedup** of a fixed query/work amount separately from **scale-up** with proportionally larger datasets/transaction rates and resources. Report absolute throughput/latency as well as ratios. Keep durability/RF/TLS/cache assumptions visible; report startup, scheduling, skew and total CPU/network/memory, including broadcast amplification. Faster one-query elapsed time can consume more total resources and harm concurrent OLTP. No new numeric SLO follows from B4.

Add plan compilation/cache bytes and invalidation reasons, statistics age/coverage/sample size, parallel grants, sequence reservation/waste, bulk job committed-batch progress and (M5) materialized generation/frontier/lag. Metrics and logs must have bounded cardinality and protect sampled values/parameters. Test overload recovery with analyze/import/index/export jobs active; derived work never consumes reserved consensus/outcome-recovery capacity.

Implementation evidence must link each new story to an exact binary/harness version, workload and fault schedule. Static document validation in this PR does not demonstrate runtime behavior.
