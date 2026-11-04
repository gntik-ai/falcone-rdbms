# Documentation review and essential feature selection

Historical B1–B3 review. The [4 November fourth-book addendum](database-system-concepts-review.md) extends this analysis with B4, 26 additional chapter mappings and four epics. Current totals are 27 epics/108 stories; counts and issue ranges below describe the 3 November baseline.

Review date: **3 November 2026**. Baseline reviewed: proposal commit `ebb625f4b20d54a9b91f1066d67c08196d3af397`, all eight design/backlog documents, and GitHub issues #2–#116. At review time all 115 issues were open, with no comments. This is a design review, not validation of an implemented engine.

## Recommendation

Build a **Rust, shared-nothing, distributed transactional RDBMS with Falcone-owned storage and disk format**. Replicate even a small production database; add physical shards when capacity, throughput or isolation requires them. Keep logical schemas and client endpoints stable as placement changes. Scalable connections, scalable execution, scalable storage and failure tolerance are separate requirements with separate evidence.

The earlier design already selected the right broad components. The main changes are stronger protocol boundaries, narrower first-release scope, earlier recovery gates, and testable resource and topology contracts. Owning storage remains a requirement; this review does not reintroduce RocksDB or another embedded database.

## Sources and review method

| ID | Uploaded work and edition | Coverage and use |
| --- | --- | --- |
| B1 | Edward Sciore, *Database Design and Implementation*, second edition, Springer, 2020; ISBN 978-3-030-33836-7 | All 15 chapters mapped to the backlog. Detailed review of memory/WAL/transactions, records/catalogs, pipelined execution, indexes, materialization, buffer allocation and optimization. JDBC and Java examples supply interface lessons, not an implementation to port. |
| B2 | Martin Kleppmann and Chris Riccomini, *Designing Data-Intensive Applications*, second edition, O'Reilly, February 2026; ISBN 9781098119065 | All 14 chapters mapped. Detailed review of load/response-time measurement, replication, sharding, serializability/atomic commit, partial failures/fencing, consensus, encoding evolution and snapshot-to-CDC handoff. Analytical/streaming/ethical chapters inform boundaries and operations. |
| B3 | Alex Petrov, *Database Internals*, first edition, O'Reilly, 2019; ISBN 978-1-492-04034-7 | All 14 chapters mapped. Detailed review of binary formats, recovery and concurrency, LSM reconciliation/compaction/publication, failure detection, distributed transactions and Raft; B-tree variants inform the layout experiment. |

The supplied Markdown contains approximately **539,277 whitespace-delimited words** and includes OCR artifacts, code, tables, references and indexes. Review used the chapter structures and summaries for whole-book coverage, followed by substantive section reads for the architecture's critical paths. It is not a line-by-line verification of every program listing, exercise, bibliography entry or OCR rendering. Chapter/section references below are more stable than OCR page numbers. No book text or uploaded source files are published with this proposal.

Source identity for reproducing the review:

| Source file | SHA-256 |
| --- | --- |
| `database-design-and-implementation.md` | `1e511d97ec297139038560337d1fdf490683bc95358626442a2c5369eedaf24f` |
| `designing-data-intensive-applications-the-big-ideas-behind-reliable-scalable-and-maintainable-systems-2_compress.md` | `2baead00dbea3b7c3b89e84e88bde3c22e1bbd7e4c9d002a8533212b1751673e` |
| `Petrov_alex_database_internals_a_deep_di.md` | `9ee8e13600ca8975d7744b6514bba5c582fde43ca25084c9eb5dd6c021e3e73a` |

These are educational and architectural sources, not executable specifications. Their product/version examples are historical. B1's pedagogical simplifications are not production requirements. For consensus implementation, the original [Raft paper](https://raft.github.io/raft.pdf), particularly §§5.4, 6–8, supplements B2/B3; its precise rules take precedence over abbreviated descriptions. Existing project research remains dated to its original inspection rather than being represented as newly reverified.

## The most important capabilities

Priority means delivery order and release gate, not estimated business value. **P0** is core correctness/scale/operability. **P1** is required Falcone product integration for M3. **P2** is later expansion. A P1 baseline cannot be silently omitted from the completed product.

| Rank | Capability | Essential result | Gate / issues |
| --- | --- | --- | --- |
| 1 | Owned durable storage | Falcone can write, recover, inspect and evolve its own files without an external engine; acknowledged data survives the tested fault model | P0, M1 onward; E19–E23, E02 |
| 2 | Replicated distributed operation | Data, catalog and transaction decisions survive the loss of one voter in a three-domain deployment; minority partitions cannot become competing writers | P0, M2; E03, E04, E20 |
| 3 | Native elastic sharding | A small replicated deployment grows into multiple physical shards with correct routing, constrained movement and no application-side shard rewrite | P0, M2; E06 |
| 4 | Scalable client and work admission | Gateways accommodate many clients while execution, memory, lock, I/O and background budgets stay bounded | P0, M1–M4; E07, E02, E21, E22 |
| 5 | Transactional relational integrity | Serializable transactions, atomic cross-shard decisions, durable recovery and correct indexes/constraints preserve application invariants | P0, M1–M2; E04–E06 |
| 6 | Practical SQL execution | Correct types, prepared statements, indexes, selective plans, joins, sorting, aggregation, spill and explain are implemented for a named workload | P0, M1–M2 core; E05 |
| 7 | Recovery and safe operations | Restore works from the first durable milestone; topology, schema and format changes have recovery paths and observable limits | P0, M1–M4; E12, E16, E17, E23 |
| 8 | Security and SQL-free administration | Engine-enforced scope plus typed JSON/gRPC management work through failures and retries; a UI is optional presentation | P0, M1 onward; E08, E09-S01–S03 |
| 9 | Falcone product capabilities | Native geospatial queries, managed S3 files, durable CDC and a tested backend adapter preserve the original requirements | P1, M3; E10, E11, E13, E14 |
| 10 | Safe PostgreSQL transition | Only assessed and reconciled workloads migrate; unsupported extensions remain on PostgreSQL | P0 at migration, M3–M4; E15 |

An early distributed alpha is not the completed product. It still needs the M3 geospatial/S3/integration baseline and M4 release evidence. Conversely, an impressive GIS demo cannot substitute for durable storage or a correct distributed commit.

## Findings and changes to the earlier design

### R01 — Define scalability as several measurable dimensions

**Earlier:** useful connection targets and a scale-out target, but insufficiently explicit load curves and resource ceilings.

**Review:** B2 ch. 2 separates load, throughput, response-time distribution and growth; B1 chs. 4/14 and B3 chs. 5/7 explain finite memory and maintenance costs. A million idle sockets is not a million running queries. A larger cache can also make a scale-out comparison look better without demonstrating distributed execution.

**Change:** certify connection capacity, active work, data capacity, shard count and recovery under load separately. Test fixed-load resource reductions and increased-load resource additions, both steady state and rebalancing. Include dataset-to-cache ratios, open-loop offered load, client-observed queueing, errors, retry traffic and merged latency histograms. Keep all existing numeric targets provisional until M0 records a workload and capacity envelope. E01-S04, E07, E17-S03 own this evidence.

### R02 — Replication is mandatory; many shards are conditional

**Earlier:** every table has a distribution policy, but the physical starting topology was underspecified.

**Review:** B2 chs. 1/6/7 distinguish reliability from write/data scale. B3 chs. 10/14 show that a leader per shard distributes coordination; one global leader for all data would not.

**Change:** specify three profiles: single-node development; small production with replicated metadata and one/few replicated data ranges; scale-out with many replicated ranges across hosts. Small compatible tenants/tables can share physical ranges. A distribution policy does not allocate a dedicated Raft group per table or tenant. Single-node mode never passes the distributed acceptance gate. E03 and E06 prove the small-to-large path.

### R03 — Make partitioning and routing a complete contract

**Earlier:** hash/range/colocation choices were listed, with limited detail about stable mappings and indivisible placement units.

**Review:** B2 ch. 7 covers hash ranges, rebalancing, hot keys, multitenancy, routing and indexes; B3 ch. 13 discusses partitioning tradeoffs. Hashing by the current node count remaps excessive data. Hashing only tenant identity cannot scale a large tenant. Hashing does not remove a single-key bottleneck.

**Change:** default to versioned stable hash-token ranges over chosen distribution columns, with explicit ordered-range placement when needed. Physical ranges map through a replicated directory to replica sets. Colocation is a contract with known split restrictions; a giant colocated entity cannot be invisibly scattered while retaining local-transaction claims. Pre-splitting is available for known bulk loads. Movement, distribution-key changes and replica replacement have distinct operations. E06-S01/S04.

### R04 — Replace an informal split workflow with fenced transitions

**Earlier:** snapshot/delta/cutover was an outline and implied transferring prepared locks across arbitrary boundaries.

**Review:** B2 chs. 7/9 and B3 chs. 8/13/14 make delayed requests, ownership and prepared decisions critical. A routing-cache update is not authority to create a new writer.

**Change:** the initial split protocol precopies data, then drains affected transactions under a replicated fence before activating children. It permits a measured, bounded admission pause on affected ranges. Unresolved prepared transactions block cutover; a timeout cannot erase them. Moving live locks without draining is a later optimization requiring its own proof. Both source and destination persist an operation identity, descriptor generation and activation evidence. Unaffected ranges remain available under their own budgets. See [distributed contracts](distributed-contracts.md). E01-S02, E06-S02, E17-S01.

### R05 — Keep consensus, isolation and atomic commit distinct

**Earlier:** the architecture correctly separated them, but stories did not name enough failure transitions.

**Review:** B1 §5.4, B2 chs. 8/10 and B3 chs. 5/13/14 explain three different obligations: ordering a shard's replicated commands, scheduling transactions serializably, and deciding all participants together.

**Change:** retain strong strict two-phase locking as the initial baseline: read/write/range locks survive until the transaction's durable outcome permits release. MVCC is a version mechanism, not a substitute for conflict checking. Replicated 2PC freezes participants before a decision; a one-shard fast path must include every index/constraint participant. Failover must fence unprepared attempts and recover prepared locks before conflicting reads/writes. E04 remains a correctness-first choice subject to explicit contention gates, not a claim that locking is the fastest design.

### R06 — Qualify the contention cost rather than silently weaken isolation

**Earlier:** serializable locking was proposed with broad deadlock handling, and optimized reads were left open.

**Review:** B2 ch. 8's 2PL/SSI comparison and B3 ch. 5 show different failure and workload costs. Long scans can block writers under 2PL; optimistic validation can amplify aborts under contention. Neither algorithm name establishes scale.

**Change:** benchmark point transactions, read/write overlap, missing-key predicates, upgrades from read to write locks, long scans and distributed deadlocks. M0 chooses a documented deadlock prevention/detection policy. Transaction age, lock count/bytes and wait queues have limits. Changes to OCC/SSI require a new concurrency protocol ADR and history checking; weakened isolation is not an automatic overload response. E04-S01/S03, E07-S03.

### R07 — Promote consistent snapshot machinery into the core

**Earlier:** safe snapshots were described as an optimization, while backups, backfills and CDC already depended on them.

**Review:** B1 §5.4.6, B2 chs. 8/12 and B3 ch. 5 show that physical read views and transaction-consistent views are different. Independent shard checkpoints can include only half of a distributed transaction.

**Change:** by M2, define a protected cluster/database cut including transaction decisions, schema versions and topology. No unresolved transaction may later appear at/below its stable frontier. Timestamp allocation alone does not establish that frontier. An explicitly quiesced consistent checkpoint is a valid initial fallback with declared availability impact; independently timed shard copies are not. Ordinary SQL still uses the serializable locking path; externally exposed stale/follower modes are deferred. E04-S04, E12-S01, E13-S03.

### R08 — Make idempotency a durable data contract

**Earlier:** request IDs and unknown outcomes were required, without a retention or replay-window contract.

**Review:** B2 chs. 8/9/13 and B3 ch. 8 distinguish delivery, execution and acknowledgement. A transaction can commit despite a lost response. TCP, Raft log indexes and random IDs alone do not make a retried application request harmless.

**Change:** requests eligible for idempotency bind scope, operation, key and canonical payload hash to a durable outcome in the same transactional decision. Changed-payload reuse fails. Define lookup authorization, retention, expiry and the client retry horizon; expired outcomes do not authorize blind replay. A retried transaction attempt discards prior reads. External effects use a transactional outbox/consumer deduplication protocol, not a universal exactly-once claim. E04-S03, E09-S02, E13-S04.

### R09 — Treat metadata and timestamps as capacity-limited services

**Earlier:** the metadata service hosted catalog, placement and timestamp allocation and was acknowledged as a possible bottleneck.

**Review:** B2 chs. 7/10 and B3 chs. 10/14 highlight coordination concentration and failure impact.

**Change:** cache versioned immutable descriptors; row operations contact their data groups directly after resolution. Separate logical queues and budgets for catalog mutations, placement, timestamp allocation and transaction resolution. Distribute transaction-status records across system shards. Timestamp batching must preserve dependencies and close unused reservations safely before snapshot advancement; unique IDs are not necessarily commit order. Measure these services under catalog churn and leader failure before claiming data-plane scale. E03-S03/S04, E16-S02.

### R10 — Keep the LSM choice empirical and measure total amplification

**Earlier:** a first-party LSM with a B+tree comparison was already proposed.

**Review:** B2 ch. 4 and B3 chs. 2–7 support workload-based selection. B-tree variants include buffering and copy-on-write; an LSM's sequential ingestion does not eliminate compaction, cache misses or space costs.

**Change:** keep one production layout after M0. Compare first-party candidates with the same safety settings and dataset, including secondary indexes, MVCC history, deletes, bounded recovery and maintenance p99. Measure WAL, replication, flush and compaction bytes separately; include free-space reserve, SSD/filesystem effects and long snapshots. Do not add a columnar engine, value-log subsystem or multiple production backends merely because a book describes them. E01-S03, E17-S03, E22-S02.

### R11 — Preserve the distinction between local batch durability and SQL commit

**Earlier:** separate engine/consensus logs and atomic application were correctly chosen.

**Review:** B1 §5.3 and B3 chs. 5/7 explain why logging, checkpoint publication and reclamation must agree. Their page-based undo/redo examples cannot simply be transplanted into a versioned LSM.

**Change:** retain separate first-party engine and Raft logs initially. An engine batch may durably store an uncommitted SQL intent. Replay restores it and transaction recovery resolves visibility; compaction cannot convert it into a committed row or discard it merely because its coordinator is absent. Shared-domain WAL deletion uses a complete covered prefix, not the largest sequence flushed by one shard. E20-S03/S04, E22-S01/S03, E23-S01.

### R12 — Add explicit comparator and schema evolution invariants

**Earlier:** versioned keys, rows and file headers were specified, but mixed-version rewrite behavior was sparse.

**Review:** B1 ch. 6, B2 ch. 5 and B3 ch. 3 show that persistent identity and encoding outlive the writing process. Loss of an unknown field during read-modify-write is possible even when both reads decode successfully.

**Change:** pin comparator/hash/collation/row-schema versions. Preserve unknown optional fields where supported, or reject writes that would discard them. Never reinterpret existing sorted keys under a changed comparator. Feature activation checks every writer, reader, replica, backup and restore tool in the support window. Decode errors and oversized lengths fail before allocation. E02-S01, E19, E23-S03, E09-S01.

### R13 — Strengthen the actual relational execution plan

**Earlier:** CRUD/joins/aggregates and a distributed optimizer were grouped into four broad stories.

**Review:** B1 chs. 7–15 explain why a parser and a storage engine do not make a useful RDBMS. Index statistics, physical operators, materialization and buffer allocation drive usable performance. B2 chs. 7/11 add network exchanges and skew.

**Change:** require cardinality/width/null/distinct statistics and sampled skew information; budget planning itself; select index, nested-loop, hash and merge paths by capabilities and estimated cost. Reserve memory before operator execution; spill sort/hash/aggregate state with bounded fan-in and safe cleanup. Duplicate-heavy hash joins must terminate through a fallback, not recursive repartitioning forever. Projection/filter/partial aggregation pushdown must respect NULL, volatility, collation and RLS semantics. E05-S02/S03, E07-S03, E21-S04.

### R14 — Make indexes distributed data with explicit semantics

**Earlier:** transactional global constraints were present, but local/global nonunique index access and cost needed clarification.

**Review:** B1 ch. 12, B2 ch. 7 and B3 ch. 1 distinguish primary lookup, secondary locators and write maintenance. Local indexes may require scatter/gather. Global indexes require their own placement and transactional updates.

**Change:** first implement local indexes and distribution-aware local uniqueness. A global unique/nonunique index is independently sharded and updated in the row transaction when certified. No asynchronous index may enforce a synchronous constraint. Explain shows fan-out, row fetches and transaction participant expansion. Row locators use stable logical keys, not file offsets that compaction invalidates. E05-S03, E06-S03, E02-S01.

### R15 — Make background work part of the admission system

**Earlier:** compaction and foreground queries had budgets, but interactions were not a complete admission contract.

**Review:** B1 chs. 4/14, B2 chs. 2/7 and B3 chs. 5/7/8 connect memory pressure, maintenance, rebalancing and cascading overload.

**Change:** coordinated reservations cover memory, CPU, disk bytes/IOPS, transfer bandwidth, file descriptors and temporary space. Protect consensus, transaction resolution and recovery from both foreground load and background jobs. Stop scheduling new transfers when recovery or compaction debt exceeds the admitted envelope. Test overload followed by load reduction; the system must recover without a full restart. E02-S04, E07-S03, E16, E22-S02.

### R16 — Move backup evidence earlier and separate backup from replication

**Earlier:** E12 was entirely M3 and depended on managed S3 objects, even for ordinary row backups.

**Review:** B1 chs. 3/5, B2 chs. 6/8/12 and B3 chs. 5/14 distinguish replica failover, crash recovery, historical recovery and disaster recovery. Replication also replicates accidental deletion.

**Change:** E12 spans M1–M3. M1 proves local export/checkpoint restore into an empty directory; M2 proves distributed transactional snapshots and restore into a clean cluster; M3 adds archival PITR and S3-object-complete restore. Object support is a dependency only for object-inclusive backup slices. A restored cluster receives a new incarnation to reject delayed requests and avoid identity reuse. E12-S01/S03/S04, E23.

### R17 — Keep geospatial and S3 useful but bounded

**Earlier:** geospatial and managed objects were well scoped and remain required.

**Review:** B2 ch. 4's multidimensional indexing and B1 ch. 6/B3 ch. 4's oversized records support indexed candidate selection and separating large payloads. The books do not establish arbitrary S3 transaction atomicity or full PostGIS behavior.

**Change:** retain exact spatial recheck after conservative candidate retrieval, with shard/byte budgets. Retain sealed immutable object identities and atomic database reference publication. Pin object bytes for every protected historical view, independently of file GC. File size does not imply routing bytes through SQL workers. Advanced GIS and object-backed analytical tables remain P2. E10/E11 retain the baseline.

### R18 — Treat CDC as an evolving public contract

**Earlier:** resumable committed changes were specified; data contracts and topology retention needed more precision.

**Review:** B2 chs. 5/12/13 show snapshot handoff, schema evolution, retained offsets, derived-state rebuilding and outboxes.

**Change:** reserve internal commit envelopes and topology lineage by M2, then expose delivery at M3. Tokens bind cluster incarnation, scope, schema/stream version and per-shard progress, including split lineage. Consumer lag cannot retain history forever. Record transaction completion boundaries without claiming total order across unrelated shards; define resnapshot and outbox patterns. Index maintenance remains synchronous where integrity requires it. E13, E20-S04, E22-S03.

### R19 — Audit corruption without inventing a second conflict-resolution model

**Earlier:** checksum audits were mentioned but not clearly separated from eventually consistent repair.

**Review:** B3 chs. 11/12 distinguish convergence tools from consensus; B2 ch. 13 recommends end-to-end checks.

**Change:** compare canonical logical ranges at the same applied/snapshot position. A mismatch triggers quarantine and repair from validated consensus/snapshot history. It does not invoke last-write-wins, hinted handoff or arbitrary value merging. Merkle-style summaries are an optional optimization for locating mismatches. E03-S02, E16-S04, E23-S02.

### R20 — Reduce presentation and expansion scope, preserve migration truth

**Earlier:** a full console and CLI shared the required administration milestone; broad SQL/search language risked looking like an initial promise.

**Review:** B1 chs. 2/11 separate an interface from engine functionality; B2 chs. 1/3/13 argue for fit-for-purpose components and explicit derived systems.

**Change:** typed administration APIs are core; E09-S04's polished UI/CLI is P2 after the API contract, with minimal operator tooling delivered by E16/E23. Core SQL and plans precede advanced windows/CTEs/functions. Vector-dependent migrations remain blocked until individually certified. Retain all existing issue identities, change scope/priorities in place, and publish a critical path rather than create a replacement backlog.

## Whole-book applicability map

The tables cover **43 main chapters**. “Use” is a Falcone design inference. Chapters explaining alternatives do not create a requirement to implement every algorithm.

### B1 — Database Design and Implementation

| Chapter | Lesson used | Destination / boundary |
| --- | --- | --- |
| 1 Database Systems | Storage, concurrency, recovery and query interpretation form one system | E01/E02/E05; reject a KV-only definition of completion |
| 2 JDBC | Explicit transaction, result and prepared-statement lifetimes | E05-S04/E07-S02; Node `pg` first, no custom JDBC driver required |
| 3 Disk and File Management | Device/file abstraction and durability assumptions | E19/E20/E23; qualify actual storage, do not transplant disk-era timing constants |
| 4 Memory Management | Bounded buffers, pins, eviction and log forcing | E02-S04/E21-S04; no unbounded cache per client/shard |
| 5 Transaction Management | Recovery ordering, serializable schedules, phantoms, deadlocks, versions | E04/E20/E22/E23; adapt to replicated intents, not SimpleDB undo-only recovery |
| 6 Record Management | Variable fields, stable locations and large records | E02-S01/E19/E11; logical locators and explicit object references |
| 7 Metadata Management | Catalogs, index metadata and useful statistics | E03/E05-S03/E09; bootstrap and replication beyond the local example |
| 8 Query Processing | Operator composition and pipelining | E05-S02; bounded batch/point paths, release resources on cancel |
| 9 Parsing | Syntax is distinct from semantic validation | E05-S01; bounded parser, binder and stable unsupported-feature errors |
| 10 Planning | Logical plans, physical costs, verified names/types | E05-S03; add CPU/network/memory and planning budgets |
| 11 JDBC Interfaces | Embedded/server separation and remote calls | E07/E09; avoid a thread per remote object/client |
| 12 Indexing | Selectivity, index joins and mutation maintenance | E05/E06/E21; indexed lookup is not always cheaper |
| 13 Materialization and Sorting | Reuse versus streaming, external sort, groups and merge joins | E05-S02/S03; explicit temp-space and cleanup contracts |
| 14 Effective Buffer Utilization | Memory-sensitive joins and allocation | E05/E07; real reservations and skew fallback, not a stale free-buffer count |
| 15 Query Optimization | Equivalent trees, pushdown, join order and physical alternatives | E05-S03; bound search and preserve security/SQL semantics |

### B2 — Designing Data-Intensive Applications, second edition

| Chapter | Lesson used | Destination / boundary |
| --- | --- | --- |
| 1 Trade-Offs in Data Systems Architecture | OLTP/OLAP, distributed versus local, source versus derived data | Core OLTP, three profiles; analytics E18 |
| 2 Defining Nonfunctional Requirements | Load, tail response time, fault tolerance, scalability, operability | E01-S04/E07/E16/E17; targets need measurements |
| 3 Data Models and Query Languages | Relational/document/graph/event model tradeoffs | E05; JSON columns do not imply graph/CQRS engines |
| 4 Storage and Retrieval | LSM/B-tree tradeoffs, columnar and multidimensional indexes | E01-S03/E19–E22/E10; one initial row storage layout |
| 5 Encoding and Evolution | Mixed-version readers/writers and unknown data preservation | E02/E09/E13/E23; versioned API/disk/CDC contracts |
| 6 Replication | Failover, lag and client-visible consistency | E03/E04-S04; synchronous quorum baseline, no unfenced promotion |
| 7 Sharding | Tenant subdivision, hash/range mapping, skew, routing and local/global indexes | E06; small deployment and elastic transitions are both tested |
| 8 Transactions | Isolation anomalies, 2PL/SSI, internal distributed commit and retries | E04/E06; no silent weakening under contention |
| 9 The Trouble with Distributed Systems | Partial faults, pauses, unreliable time and fencing | E01-S02/E03/E04/E06/E17; timeouts are suspicion |
| 10 Consistency and Consensus | Linearizability, ordering and coordination services | E03/E04; distinguish Raft order, transaction isolation and timestamp IDs |
| 11 Batch Processing | Partitioned operators, shuffle, locality and bounded jobs | E05/E09/E12; use for execution/backfill, defer a general analytics platform |
| 12 Stream Processing | CDC, snapshot handoff, ordering, replay and state | E13/E15; at-least-once delivery plus declared deduplication |
| 13 A Philosophy of Streaming Systems | Derived views, end-to-end correctness, audits and idempotency | E13/E17; asynchronous views cannot replace synchronous SQL constraints |
| 14 Doing the Right Thing | Data minimization, sensitive-data exposure and lifecycle consequences | E08/E11/E12; technical retention/export/redaction controls, no blanket legal-compliance claim |

### B3 — Database Internals

| Chapter | Lesson used | Destination / boundary |
| --- | --- | --- |
| 1 Introduction and Overview | DBMS layers, row/column layout, primary and secondary indexes | E02/E05/E06; logical primary keys survive physical rewrites |
| 2 B-Tree Basics | High fanout, search, splits and merges | E01-S03 comparison; no mandatory second shipping engine |
| 3 File Formats | Explicit encoding, variable sizes, versions, checksums and bounds | E19/E21/E23; CRC detects accidents, not hostile authenticity |
| 4 Implementing B-Trees | Headers, overflow, compression, bulk load and maintenance | E01-S03/E19/E21; LSM needs equivalent format rigor, not B-tree page mutation |
| 5 Transaction Processing and Recovery | WAL/cache coordination, ARIES, concurrency, locks versus latches | E04/E20/E21/E23; local latches do not survive leader failover as transaction locks |
| 6 B-Tree Variants | Copy-on-write/buffering/indirection tradeoffs | M0 comparison; avoid selecting by algorithm name alone |
| 7 Log-Structured Storage | Reconciliation, tombstones, amplification, flush publication and log stacking | E20–E22; preserve visibility and recovery before reclamation |
| 8 Introduction and Overview | Asynchrony, delivery ambiguity, fault models and cascading failures | E04/E07/E17; bounded retries and explicit safety/liveness assumptions |
| 9 Failure Detection | Suspicion versus proof, timeout tradeoffs | E03/E16; detector cannot grant write authority |
| 10 Leader Election | Coordination bottlenecks and split-brain risks | E03; use consensus-backed leadership, not a separate ad hoc election |
| 11 Replication and Consistency | Operation/session guarantees and weaker consistency models | E03/E04; no inference that quorum arithmetic alone provides serializability |
| 12 Anti-Entropy and Dissemination | Summaries for divergence detection and gossip | E16/E23 integrity audits; no gossip authority or LWW merge in the SQL core |
| 13 Distributed Transactions | Participant promises, uncertain decisions and coordination costs | E04/E06; 2PC over replicated groups, no timeout-based heuristic commit |
| 14 Consensus | Ordered replicated state, elections, membership and recovery | E03/E20/E17; qualify primitive and first-party durable integration together |

## Decisions deliberately not adopted

| Alternative from the literature | Why it is outside the initial core |
| --- | --- |
| Eventually consistent multi-leader/leaderless row writes | Conflict resolution would change uniqueness, FK and transaction promises; a separate future product profile would be needed |
| Gossip as the authoritative shard catalog | Dissemination is useful, but concurrent ownership requires stronger agreement |
| Full serial execution on one global thread | Simple scheduling would impose a cluster-wide bottleneck and require restricted transactions |
| Immediate distributed SSI/OCC implementation | Potentially valuable; does not remove the need for conflict tracking, predicate coverage, recovery and workload evidence |
| ARIES copied unchanged into the LSM | Valuable recovery concepts, but page mutation, versioned intents and immutable-file publication differ |
| Both a B+tree and LSM in production | Doubles recovery/format qualification without an established workload need |
| Full OLAP, graph, streaming framework and ANN engine | These dilute the initial transactional contract; CDC/export can integrate specialized systems |
| Full PostgreSQL extension compatibility | Protocol similarity does not provide the extension ABI or semantics |
| S3 in the synchronous row-write path | Object-managed files meet the baseline without requiring a new remote page-store protocol |
| A full admin console before core APIs and recovery | The API satisfies administration without SQL; diagnostics and safe recovery are more urgent than presentation |

## Delivery order and retained backlog

All **23 epics and 92 stories retain their IDs and GitHub issue numbers**. The review strengthens their acceptance criteria and adds source/priority traceability; it does not mark design work implemented.

1. **M0:** E01 and E17 foundations; freeze the fault model, storage experiment, comparator, transaction and shard transition ADRs. Measure workload shape before freezing capacity targets.
2. **M1:** E19–E23 local slices → E02 integration → E04 local isolation → E05 core SQL; E07/E08/E09 API slices alongside them. E12 proves local restore here.
3. **M2:** E03/E20 consensus integration → E04 distributed transactions and consistent cuts → E06 safe split/move/global integrity. E05 distributed plans, E07 admission, E12 distributed restore, E13 internal envelopes and E16 operations are required evidence.
4. **M3:** E10 geo, E11 managed S3, E12 object-inclusive PITR, E13 delivery, E14 integration and E15 migration rehearsals. Close every retained baseline requirement.
5. **M4:** independent reviews, sustained capacity/fault tests, clean-cluster restore, mixed-version upgrades and release-specific compatibility evidence.
6. **M5:** E18 expansion and E09-S04 polished console/CLI. Prioritize any advanced feature only when a named workload justifies moving it earlier.

Cross-epic dependencies apply to milestone slices. In particular, local backup does not wait for S3 objects, local administration does not wait for a distributed catalog, and safe distributed snapshots cannot wait until after PITR implementation begins. The [roadmap](quality-and-roadmap.md), [distributed contracts](distributed-contracts.md) and [backlog](../backlog/epics-and-stories.md) carry the resulting acceptance details.
