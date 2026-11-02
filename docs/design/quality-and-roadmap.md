# Quality gates and roadmap

All numeric values here are **proposed engineering targets**, not measured results or capacity promises. M0 must establish hardware, workload and budget before accepting or revising them.

## Scope tiers

| Tier | Required scope |
| --- | --- |
| M0 feasibility | Repository contract; licensed/versioned component candidates; Falcone compatibility inventory; transaction/shard state-machine models; reproducible harness |
| M1 local alpha | Local durability/MVCC; bounded SQL subset; single-node serializable behavior; authenticated tenant isolation; JSON management; driver smoke certification |
| M2 distributed alpha | Three-domain replicated shards; cross-shard transactions/constraints; failover; safe splitting/movement; load admission and connection scale |
| M3 constrained beta | Named spatial subset; managed S3 objects; coordinated backups/PITR; committed CDC; Falcone integration; supported-workspace migration rehearsal |
| M4 production candidate | Independent correctness/security review; fault campaign; sustained workload/failover tests; verified restore/upgrade; published compatibility/SLO limits |
| M5 expansion | Multi-region/locality; object-backed analytical tier; search/ANN; advanced SQL/GIS and development branches |

Do not move a user baseline requirement into an indefinite future: native sharding and high client concurrency are M2, SQL-free administration starts at M1, and geospatial plus S3 large objects are mandatory for M3. Full compatibility with top-tier engines is not a realistic first-release criterion; the supported relational subset must still satisfy integrity and recovery guarantees.

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

## Correctness and reliability gates

- Deterministic simulation from the first storage/transaction milestone: controlled clocks, network, disks and scheduler; save failing seeds. Model-check transaction decisions, topology transitions and GC/pin races separately.
- Property tests/fuzzing for SQL/protocol parsing, key encodings, type conversion, index maintenance and authorization. Differential SQL tests against PostgreSQL cover only the declared common semantics.
- History-based distributed checking: lost updates, write skew, phantoms, concurrent global uniqueness, FK races, fractured reads, commit ambiguity and leader changes. Linearizability checks apply to APIs that actually promise it; do not equate them with every SQL transaction history.
- Crash at every durable boundary: prepare, commit decision, apply, index backfill, shard cutover, snapshot publication, log truncation and object deletion. Verify no half-transaction or acknowledged-data loss.
- Partitions, message reordering/duplication, long pauses, disk full, torn/corrupt data simulation, slow disk, clock steps, stale routing, metadata failover and node replacement. FFI and real-storage behavior also need tests beyond simulation.
- Cross-tenant negative tests across SQL, JSON, gRPC, native protocol, indexes, errors, object URLs, CDC, backups and restored data. Test credential/policy changes during long sessions.
- Backup restoration into a clean cluster, not just successful uploads; verify relational invariants and every protected object. Test KMS/Secrets recovery dependencies and key rotation.
- At least a 7-day mixed-workload soak and repeated restart/upgrade campaigns for M4; longer campaigns as volume and risk justify. No unresolved critical correctness/security issue passes a release gate.

## Operations and observability

Metrics: active/idle sessions, queue time, admission drops, transaction latency/aborts/recovery age, shard hotspots, replication lag, fsync and compaction, snapshot/CDC age, blob lifecycle/GC backlog, backup archive lag, verification and restore durations. Avoid an unbounded Prometheus label per tenant/query; aggregate and expose scoped detail through authenticated APIs/logs.

Use OpenTelemetry traces across gateway/query/shard/object operations with parameter values redacted. Query fingerprints, sampled explain plans, slow-query budgets and index statistics support diagnosis. Separate service availability, data durability, backup freshness and capability health in readiness dashboards.

Kubernetes support includes anti-affinity/topology spread, persistent identities/volumes, PodDisruptionBudgets, bounded drains, storage-aware scale-in and quorum-aware upgrade sequencing. HPA can scale stateless gateways; stateful scale-down requires engine orchestration. Never let a generic CPU autoscaler delete quorum members independently.

Runbooks cover lost quorum, bad disk, metadata restore, unknown transactions, runaway query, replication/CDC backlog, S3 outage, failed upload, accidental logical deletion, tenant export and key loss. Dependency outages must not convert accepted provisioning into a false “active” state.

## Delivery capacity and decision gates

This is a new distributed database product. A credible team needs storage/consensus, transactions/query processing, Rust/networking, SQL compatibility, reliability/testing, and Falcone integration expertise. Some roles can overlap, but correctness review must be independent of the author of the protocol.

A production-grade engine is a multi-stage, potentially multi-year effort; agent-generated code does not replace protocol validation or operating evidence. Do not commit a release date until M0 measures reuse feasibility and compatibility scope. If time-to-production becomes the overriding constraint, explicitly reconsider an engine built on a qualified external transactional store instead of quietly changing the architecture.

Open decisions with recommended defaults:

| Decision | Proposed default | Close by |
| --- | --- | --- |
| First-party language / embedded C++ | Rust engine, qualified RocksDB embedding | M0 |
| Source license | Apache-2.0 proposal; owner acceptance required before adding a license grant | M0 |
| Transaction method | Strict 2PL + MVCC + replicated 2PC; optimize only after correctness evidence | M0 protocol review |
| First SQL/client profile | Falcone Node `pg` workload and defined SQL subset | M0 |
| Initial geography | WGS84 points; planar geometries; exact semantics with conservative index | M0 API freeze |
| Large objects | Explicit `object_ref`, immutable keys, verified seal, database-owned retention | M0 API freeze |
| First production topology | Single region, three failure domains | M0 deployment profile |
| Workspaces requiring extensions | Keep PostgreSQL until individually certified | Migration gate |

These are proposal choices, not a request to delay design work pending approval. Implementation should resolve them with the evidence specified by E01.
