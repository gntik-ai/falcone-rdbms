# Distributed core: scalability, sharding and recovery contracts

Status: proposed implementation contracts, revised after the [three-book review](documentation-review.md). These requirements refine the [architecture](architecture.md); no algorithm outline here is an implementation proof. The corresponding M0 models and M1–M4 evidence must pass before a capability is advertised.

## Three requirements, three kinds of proof

| Requirement | Architectural mechanism | Required evidence | Insufficient evidence |
| --- | --- | --- | --- |
| Scalable | Independent gateway/query capacity, bounded execution, partitioned data work and maintenance budgets | Connection and active-work load curves; fixed RF/durability scale-out; measured metadata, memory, I/O and recovery limits | Async Rust, many idle connections, or a larger cache alone |
| Distributed | Replicated data/catalog/status groups, fenced leadership, durable consensus, transaction recovery | Failover and partition histories; acknowledged-data recovery; one-voter loss under load | Several processes that all depend on one unreplicated authoritative server |
| Sharding when necessary | Stable logical routing policy, replicated physical-range directory, split/move/rebalance | Single-range-to-multiple-range growth, large-tenant subdivision, stale-route tests, constraint preservation | Local table partitions, a client-computed node modulo, or read replicas alone |

The completed baseline also includes owned Rust storage/files, SQL-free administration, geospatial support and managed S3 files. Scalability work cannot remove those requirements or weaken correctness silently.

## Deployment and growth profiles

| Profile | Data placement | Intended use | Qualification |
| --- | --- | --- | --- |
| Development | One process/node, local metadata and one/few ranges | Development and persistence tests | Explicitly non-HA; does not satisfy distributed release gate |
| Small production | Three failure domains; metadata quorum and one/few data ranges with three voters each | Low/medium load that does not need many shards | Same wire, file and transaction contracts as scale-out; independent disk/host failures tested |
| Scale-out | Multiple ranges with leaders spread across stores; each range still replicated across domains | Higher data volume, write throughput or tenant isolation needs | Growth and drain tests, correct routing, constrained rebalancing and maintained SLO envelope |

Roles may share a binary/host in small profiles; failure-domain separation remains real. A replica of every row on three machines increases durability and some potential read capacity, but does not divide the row-write load. Adding gateways increases session capacity only until query/store/control capacity becomes limiting.

A table's **logical distribution policy** is separate from the cluster's **physical range allocation**. Compatible small tables/tenants can share a range and its Raft group under the same placement constraints. Large tenants can span ranges. No fixed large cache, memtable, writer thread or Raft group is allocated per client or automatically per tenant. Report the maximum tested ranges per store and the overhead per idle and active range.

## Distribution policy and index placement

Default routing uses a versioned deterministic hash of canonical distribution-column bytes, followed by ranges of that hash space. The authoritative directory maps `(placement group, key interval, generation)` to a replica set. Node addition changes placement and selected range boundaries; it does not change the hash function or compute `hash(key) % current_node_count`.

For locality-sensitive workloads, ordered range policies remain available. The system exposes the physical consequence in a schema/placement preview. Monotone timestamps can concentrate ingestion on a tail range. Hashing spreads distinct keys but cannot split serial updates to one row.

Stable key encoding includes authenticated scope, logical table/index IDs, distribution values and primary-key identity. The normative encoding RFC must prove that colocation keys line up across tables, scans have valid bounds, and tenant isolation does not depend on physical separation alone. Scope prefixes may group data for isolation, but cannot force an entire large tenant to one node.

| Access pattern | Preferred plan | Explicit cost or restriction |
| --- | --- | --- |
| Equality on distribution key | Route to one interval, then local index/row lookup | A global index or FK may still add transaction participants |
| Ordered range with compatible policy | Prune to intersecting intervals | Hot tail risk and topology-aware range iteration |
| Parent/child transaction | Colocate compatible entity keys | An indivisible entity has a published size/load limit; repartitioning changes locality promises |
| Filter on local secondary index only | Query selected/all relevant ranges and merge | Local index lookup does not guarantee low cluster fan-out |
| Filter on global secondary index | Route through its independently sharded index, then fetch rows | Additional writes/participants, network hops and possible skew |
| Aggregate or noncolocated join | Budgeted exchanges and partial work | Cannot promise point-query latency; limits and spill apply |

Global index locators contain stable logical row keys and routing information that can be refreshed by generation. They never contain a durable raw file offset. Unique keys are normalized using the same equality/collation semantics as SQL. Nonunique index keys include a row identity to avoid a single unbounded posting value. One extremely common indexed value still requires a measured query/placement strategy.

Local unique constraints are allowed only when all possible conflicts are in the same placement unit. Otherwise the index participates in the distributed transaction. FK checks protect referenced keys against concurrent delete; cascades are bounded. Unsupported combinations fail before DDL publication. Eventual secondary views may be offered separately later, but are not constraint enforcement.

## Replication and authority

Each data range has a leader-based consensus group, normally three voters. Catalog and transaction-decision data also have replicated authority. A timeout, health score, routing epoch or Kubernetes readiness result can trigger recovery work; none alone authorizes a new writer.

The qualified Raft integration must preserve stable term/vote/log ordering, election restrictions and the current-term commit rule. Leader reads need verified authority and an applied-state barrier; a node's local belief that it is leader is insufficient. Membership transitions and snapshot compaction follow the selected core's verified protocol. These details are qualified against the [Raft specification](https://raft.github.io/raft.pdf) and tests, not inferred from a majority-count shortcut.

Three distinct generations travel in internal requests where relevant: cluster incarnation, shard ownership generation, and coordinator attempt epoch. They solve different stale-actor problems and must not be conflated with file generation or MVCC timestamps. Authenticated receivers validate them at the operation's authoritative boundary.

Metadata watches/caches reduce steady-state directory traffic. Cache misses or invalidations are bounded, coalesced and backpressured. Catalog churn, placement changes, timestamp service traffic and transaction resolution use separately observable queues. Timestamp allocation can share an initial deployment with metadata, but its scaling limit and outage effects are independently measured. No row-by-row catalog transaction or one-global-data-Raft-log design is permitted.

## Initial split and movement state machine

Replica replacement **within the same Raft group** uses membership change. A **range split** creates new ownership identities and requires a handoff. A **distribution-key change** is a schema/data migration, not either of those operations. Expose distinct plan types and status.

The initial split implementation uses precopy plus a fenced drain. It does not promise zero pause on the affected range. The public operation reports measured pause time, estimated temporary space, progress, blocker age and cancellation boundary.

| State | Durable condition and allowed work | Failure/retry behavior |
| --- | --- | --- |
| `PLANNED` | Replicated operation ID, source generation, child intervals, placement and budgets; source remains authoritative | Repeated plan/apply returns same operation; stale generation rejects |
| `COPYING` | Protected source snapshot plus retained ordered delta history; destination files are private | Resume verified chunks or discard unreferenced staging; source still serves |
| `DRAINING` | Source replicates a fence against new affected transactions/lock acquisitions; accepted work resolves | Abort only unprepared work through its decision authority; prepared work must resolve or cutover waits |
| `SEALED` | Source durable final barrier; no transaction can later add an old-generation mutation; final delta/checksum verified; children durably recoverable but inactive | A crash retains the fence and operation; retries cannot reopen source writes by lease expiry |
| `PUBLISHED` | Metadata atomically publishes new directory generation and child coverage, referencing source seal and destination readiness evidence | Children activate only with valid evidence; stale source requests redirect/reject; no second writer |
| `RETIRING` | Old routing/CDC lineage remains resolvable; readers/backups still pin needed files/history | Reclaim only after reachability and retention tests; deadline alone does not authorize deletion |
| `COMPLETE` | New ranges serve; cleanup and lineage retention follow declared policy | Completion reports actual data/replica validation, not job acceptance |

Before `PUBLISHED`, cancellation may restore source admission only through a replicated abort transition that proves children were never activated and invalidates their readiness tokens. After publication, cancellation cannot revert ownership; recovery finishes forward or uses a new, separately fenced movement operation. The model must prove the race between cancellation, delayed activation and controller replacement.

The initial drained cutover avoids transferring a live transaction's lock ownership across new groups. Precopy can contain intents; final activation must establish that relevant decisions are resolved and no old attempt can recreate them. Long prepared transactions may delay the operation beyond its planned window; the system reports the blocker and preserves safety. Later online lock migration needs explicit predicate coverage, participant remapping and recovery proof before replacing this behavior.

Child interval union must equal the original interval with no gaps/overlaps. Copied row/index state, applied source barrier, schema dependencies and deduplication state are validated together. New groups have their own Raft indexes; an imported parent index is recorded as lineage, not used as an arbitrary child commit index. Clients keep the same logical table endpoint. A stale-epoch response permits bounded route refresh; it does not permit blindly replaying a write with unknown outcome.

### Rebalancing safeguards

Use sustained bytes/read/write/CPU/compaction/health signals, minimum residence time and hysteresis. Reserve source and destination temporary space before copying. Cap concurrent transfers per node/domain and aggregate network/I/O. Under overload, stop starting moves and reduce ingress; do not interpret every slow heartbeat as a failed host that needs immediate full relocation.

Support operator preview, pause, pin, drain and cancellation through JSON APIs. Automatic split/balance is enabled only after the same safe operation path is certified manually. Range merging remains a later optimization; publish range-count overhead and lower-bound sizing so deferring merges does not imply unlimited metadata.

## Transaction and retry contract

Ordinary read/write SQL starts with **strong strict 2PL**: acquire compatible key/range locks, read current committed values under those locks, and hold read/write protection until the outcome permits release. Missing keys, index gaps, foreign-key checks and full scans need protection too. Physical cache/memtable latches protect in-memory structures; they are short-lived and are not a transaction protocol.

| Transaction state | Required invariant |
| --- | --- |
| Active | Fenced coordinator; participants registered; all reads/writes follow one isolation method; resource limits enforced |
| Preparing | Participant set frozen; no new participant or SQL operation can bypass it; each prepared vote is durable in that group's quorum |
| Prepared | Intents and sufficient lock/recovery state survive failover; participant cannot decide independently |
| Committed/aborted | One immutable decision stored by the authorized decision quorum; delayed messages cannot reverse it |
| Resolving | Participants apply the same outcome idempotently; reads cannot expose a partial distributed commit |
| Reclaimable | Outcome/dedup history can be removed only under the documented recovery, delayed-message and client-retry horizon |

Before serving conflicts after leader change, recover prepared state and fence or abort unprepared attempts safely. A decision-quorum outage may block affected keys. A process timeout is not an abort decision. Deadlock handling must cover multiple shards and lock upgrades, define victim/retry behavior, and prevent an unbounded retry storm. Cap transaction duration, participants, lock bytes, result bytes and queued waits. A limit reached after prepare invokes outcome recovery rather than unilateral lock release.

For idempotent operations, persist `(scope, client/request key, operation kind, canonical payload hash, transaction outcome/result reference)` under the same commit authority as effects. A duplicate with the same payload returns/resolves the prior outcome; changed payload fails. IDs are not globally readable transaction-status oracles. Publish the retention interval and maximum safe retry horizon. After expiry, reject or require explicit reconciliation; do not infer that an unknown old ID never committed.

Serialization retries create a new attempt and reread data. Do not retry arbitrary interactive transactions after results or external effects have escaped. A transactional outbox can publish events after commit; its consumers still need their own atomic deduplication/sink contract. S3 uploads and external email are not participants in Falcone's internal 2PC.

## Consistent cuts for backup, backfill and streams

An engine `PinReadView` protects local physical files/sequence. A distributed logical snapshot additionally identifies scope, catalog/schema versions, topology generation, transaction frontier and protected external-object versions. Independent local snapshots are not sufficient.

The stable frontier contract requires:

1. A timestamp/order boundary with no future valid transaction decision capable of inserting a commit at or below it. Outstanding timestamp reservations must be closed or accounted for.
2. Every relevant shard has applied/resolved the effects needed for that boundary, including cross-shard decisions. Unresolved earlier work holds the frontier back.
3. Snapshot membership accounts for concurrent splits/moves through lineage; the logical data set is neither duplicated nor omitted.
4. Retention pins are durable before bytes/history can be reclaimed. Object-pin creation serializes with object GC.
5. Snapshot acquisition fails or waits explicitly when authority/history is unavailable. Expired pins invalidate the operation instead of silently returning an incomplete result.

M0 must select and model the exact frontier algorithm, including how timestamp batching and one-shard fast paths interact. M2 must demonstrate it or a documented database-scoped quiescence/barrier implementation with measured unavailability. An HLC reading, maximum observed timestamp or minimum replica log index is not automatically a global safe cut.

Safe internal snapshots are required for backup and online data operations. They do not automatically expose a user-selectable snapshot isolation level or a fresh follower read. Such public modes require a separate anomaly/freshness contract and qualification.

Local recovery/restore is M1, distributed backup/restore is M2, and archived PITR plus managed S3-object-complete restore is M3. Restore creates a new cluster incarnation, validates encryption/codec/schema dependencies and reconciles all constraints before serving. Restore tests include accidental deletion, missing history and missing object bytes, not only node loss. PITR reports the common achievable frontier and gaps; it never claims the requested wall time when some participants cannot reach it.

## Query execution required for useful scale

Logical validation precedes physical optimization. Binding checks scope, names, types, schema version and supported functions. Planning has time, memory and join-enumeration limits. Statistics include row counts, row widths, null/distinct estimates and sampled skew per relevant table/index/range, with age/version and a conservative fallback. Statistics are not transactional truth and may never justify omitting required rows.

The core executor needs point/index/table scans, filters/projections, nested-loop/index/hash/merge joins as selected by supported plan shapes, aggregation, ordering, limit and bounded streaming. Not every algorithm must be initially optimized, but each released operator must have a bounded execution path or reject work before unbounded allocation.

| Concern | Implementation contract | Evidence |
| --- | --- | --- |
| Pipeline | Pull/batch flow control propagates demand; cursors/pins release on end/error/cancel | Slow reader and mid-scan cancellation leave bounded retained resources |
| Sort/materialization | External runs with bounded merge fan-in and temp-space reservation | Correct results with data beyond RAM; disk-full cleanup and restart |
| Hash join/aggregate | Grant memory before building state; spill/partition with skew detection and a terminating fallback | All-equal keys, large duplicate groups and concurrent queries do not recurse/OOM indefinitely |
| Distributed exchange | Bounded per-stream queues and bytes; cancellation/deadlines propagate | Slow/lost worker cannot accumulate unlimited upstream results |
| Pushdown | Preserve SQL types, NULL, collation, volatility and policy filters | Differential results for local/distributed plans, including empty and duplicate inputs |
| Top-k/aggregation | Partial results merge under a specified global order/aggregate state | Ties, nulls, overflow and cross-shard groups match the declared semantics |
| Explain | Show logical/physical plan, estimates, pruning, exchange/fan-out, policy and budgets | Actual counters expose rows/bytes/spill/time and estimate error without value leakage |

Cost includes CPU, I/O, network, row width, memory and skew; an available index does not mandate its use. Prefer colocated work and predicate/projection reduction before exchange. For safety, security-barrier predicates and volatile/error-producing expressions constrain rewrites. Changing a distribution policy invalidates affected cached plans. Continuation tokens bind enough topology/snapshot information to avoid skip/duplicate errors under movement or return an explicit expiry/restart response.

## Unified budgets and admission

Reserve rather than merely observe capacity. A query cannot assume memory remains available because a planner saw free buffers earlier. Charge gateway sessions, execution state, transaction locks, table/cache pins, buffers, spill and transport to appropriate global and scope budgets. Shared resources require documented attribution and bounded accounting overhead.

Reserve a protected lane for consensus, outcome resolution and recovery. Flush, compaction, index backfill, backup, integrity scans and range movement have explicit CPU/I/O/network/temp-space budgets and coordinated concurrency limits. Background debt and low free space trigger staged load shedding before the reserve is consumed. Quota exhaustion cannot delete protected data or cancel a prepared promise.

Every retry chain has an end-to-end deadline, bounded attempts, jitter and a single budget; nesting retries at gateway/query/store layers must not multiply offered work without accounting. Test that overload recedes when input returns to the admitted rate. Include reconnect storms, metadata-watch invalidations, long snapshots, stalled object/CDC workers and a slow storage device.

## Mandatory scenario evidence

| Scenario | Pass condition | Owners |
| --- | --- | --- |
| Grow one replicated range into several across added hosts | Same endpoint/schema; exact keys/indexes; independent-key throughput increases under fixed durability/RF; pause and migration costs reported | E03/E06/E17 |
| One large tenant and many small tenants | Large tenant subdivides; small-tenant resource use remains bounded and scoped | E06/E07/E08 |
| Leader loss or minority partition during commit | No conflicting decisions or lost acknowledged data; unknown results resolvable | E03/E04/E20 |
| Crash at each split state and delayed old requests | One authority, full key coverage, no duplicate effect; cancellation/activation race safe | E06/E17 |
| Deadlock, missing-key predicate and FK/unique race across shards | Serialization or explicit abort preserves invariants | E04/E06 |
| Backup during writes, split, DDL and unresolved prepare | Valid consistent cut or explicit wait/failure; clean restore passes integrity checks | E04/E09/E12 |
| Long snapshot with compaction and low disk | Retained versions remain valid; bounded admission stops exhaustion; no tombstone resurrection | E02/E22 |
| Large sort/join under concurrent memory pressure | Bounded memory/temp data and correct results or documented resource error | E05/E07 |
| Mixed old/new software plus old/new rows and new optional fields | Supported reads and rewrites preserve data; incompatible writes refused | E09/E16/E23 |
| Rebuild replica with a deliberately corrupted range | Divergence localized at a common cut; quarantined member repaired from validated history | E03/E16/E23 |
| Restore then replay delayed old-cluster messages | New incarnation rejects stale authority and dedup/identity confusion | E12/E23 |

Evidence includes exact binary/harness revisions, hardware and data manifests, workload seed, fault schedule, histories, checksums, response histograms, errors, budgets and known limits. A passing model or simulation does not replace real filesystem/device tests; a successful benchmark does not prove serializability.
