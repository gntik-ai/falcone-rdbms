# Proposed architecture

Status: proposal; all guarantees below are implementation requirements and release gates, not current behavior. See [research](research.md) for source evidence and [roadmap](quality-and-roadmap.md) for qualification.

## Product boundary

Falcone RDBMS owns SQL evaluation, schema/catalog management, authorization, transactions, distributed indexes, native shard placement, replication, recovery and operational APIs. **Storage-engine and disk-format ownership is a confirmed user requirement.** Falcone implements its own persistence code in Rust: WAL, table files, manifest/checkpoints, cache, compaction, integrity verification and recovery. Third-party database/storage engines are excluded as shipping backends. Qualified general-purpose libraries and a consensus algorithm core remain eligible; Falcone owns their storage integration.

The [native storage design](native-storage.md) specifies this boundary and the proposed first-party LSM layout. Disk-format details remain an RFC until M0 evidence freezes a version; the ownership requirement is already decided.

The initial production target is transactional BaaS workloads in one region across three independent failure domains. Scale connection gateways and query workers independently of storage. S3 initially stores large objects and backups; it is not the synchronous storage path for ordinary row commits.

## Components and deployment

```mermaid
flowchart TD
  C["Applications and Falcone"] --> G["Connection gateways"]
  C --> A["JSON administration API"]
  G --> Q["Query and transaction workers"]
  A --> M["Replicated catalog and placement"]
  Q --> M
  Q --> D["Replicated storage shards"]
  M --> D
  D --> E["Committed change streams"]
  Q --> B["Object coordinator"]
  B --> D
  B --> S["S3 object storage"]
  D --> R["Backup and restore workers"]
  R --> S
```

| Process role | Responsibilities | State and scaling |
| --- | --- | --- |
| `falcone-db-gateway` | PostgreSQL protocol, optional JSON query API, TLS/auth, session state, quotas, cancellation, admission | Ephemeral client/session state; horizontally replicated; restart can disconnect clients |
| `falcone-db-query` | Parse/bind/optimize, execute, coordinate transactions, merge results | Disposable compute; durable transaction outcome is stored elsewhere |
| `falcone-db-meta` | Cluster bootstrap, catalog epochs, shard directory, placement policies, timestamp allocation | Small Raft-replicated metadata service; three voters initially; do not send every row through it |
| `falcone-db-store` | Host many shard Raft groups, versioned rows/indexes, intents, locks, log replay, compaction | Persistent SSD/NVMe; data survives an individual member's loss under replication policy |
| `falcone-db-admin` | REST/JSON and gRPC management, schema jobs, operations/status, API authorization | Replicated front ends; durable jobs/catalog live in replicated metadata |
| `falcone-db-worker` | Blob lifecycle, backup, restore, CDC delivery, index backfill, reconciliation | Idempotent resumable tasks; separately budgeted CPU, disk and network |

Roles can share one binary in local development. Separation into independent deployments is a scaling option, not a requirement for six microservices per small workspace. Never deploy one process, Raft group or database instance per client or automatically per tenant.

The cluster bootstraps from authenticated node membership and its own replicated metadata storage. It must not depend on a tenant database it is supposed to provision. Falcone IAM/Secrets integration has cached verification and explicit failure behavior; no request-loop dependency on this new engine is introduced into its own startup.

## Key and catalog model

Logical identity: `cluster → tenant → workspace → database → schema → table`. Stable numeric/internal IDs prevent rename operations from moving stored data. Security scope is immutable, authenticated context, not a user-controlled SQL session variable.

Conceptual row key: `(tenant_id, workspace_id, database_id, placement_group_id, distribution_value, table_id, primary_key, version)`. Actual binary encoding, field order and version ordering are frozen only after E01/E02 benchmarks. A secondary index uses its own index ID and normalized indexed values, with a row locator. All paths include security scope, including global indexes, blobs, CDC checkpoints, caches and temporary spill files.

Catalog entries include columns/types, constraints, indexes, placement, policy versions, object storage bindings, migration state and schema version. Separate the logical catalog from the physical shard directory. Cache entries are keyed by security scope and epoch, not only SQL text or table name.

## Persistence and consensus

Use Falcone's first-party storage engine with a bounded number of storage domains per process, rather than allocating a complete engine per tiny tenant or shard. The initial design is an LSM with ordered memtables, immutable sorted table blocks, a versioned manifest, WAL, Bloom filters and a block cache. Prefixes identify logical shards; immutable output files remain within a shard/key-range boundary to simplify movement and snapshots. Domains share bounded pools for memtables/cache/compaction without giving every shard a fixed large allocation. Measure the fairness cost of shared storage and offer dedicated node pools for large tenants.

Falcone defines and maintains the file format and logical key encoding. It does not reuse RocksDB SST/WAL formats or require its libraries. M0 compares the first-party LSM prototype with a small first-party B+tree prototype on representative workloads; changing the layout does not relax ownership.

Each shard is a Raft replication group, normally three voters on three distinct hosts/failure domains. A majority is required for writes. Membership changes use a qualified joint-consensus/learner procedure; a routing epoch alone is not a substitute for consensus fencing. Replicas also serve checksum audits and snapshot recovery.

Durability contract:

1. A shard leader proposes deterministic commands, never reevaluating nondeterministic SQL independently on replicas.
2. Required log entries are durably persisted on a quorum before commit acknowledgement.
3. Applying a command changes row/index/intent state and its applied-log index atomically through Falcone's own bounded `ApplyBatch` and engine WAL. Local engine sequence numbers, Raft indexes and SQL transaction timestamps are distinct coordinates.
4. Replay after a crash is idempotent. A snapshot contains its exact shard descriptor, schema dependencies, transaction state and applied index.
5. Log truncation happens only after a durable recoverable checkpoint and all applicable replication/backup/CDC obligations permit it. The initial design keeps separate first-party consensus and engine WAL streams with explicit replay fences. Removing duplicate persistence requires a later protocol design and crash-test evidence, not disabling a WAL by configuration.

Use MVCC versions for consistent historical reads, backups and recovery. Track active transactions, protected snapshot timestamps, backups and CDC retention before collecting old versions. Long-lived snapshots have explicit limits; storage exhaustion must throttle/reject work before corrupting durability.

## Distributed transaction protocol

**Serializable is the proposed default.** Consensus, MVCC and two-phase commit alone do not prove serializability. The initial correctness-first protocol is strict two-phase locking of read/write keys and predicates, layered over MVCC, with replicated transaction status and two-phase commit for multiple shards. Read-committed/snapshot modes are deferred until separately specified and tested.

Transactions are identified by a stable ID and attempt/ownership epoch. A hashed transaction-status key distributes decision records across system shards. The status record includes participants, deadline, state, coordinator fencing epoch, and the immutable final decision. An HA timestamp allocator in the metadata service allocates monotone epochs/timestamps for MVCC; batching and failover must preserve the protocol's ordering. It is a benchmarked potential bottleneck, not an assumed infinitely scalable service.

The initial read/write transaction algorithm locks the accessed keys and index ranges, reads the latest committed version under those locks, and holds locks through final decision. Predicate protection includes absent keys, secondary-index gaps, full scans, and constraint checks. Fall back to a coarse table/index lock when a precise range cannot be derived. Retrying a transaction must not reuse stale reads. Do not mix arbitrary snapshot reads into this algorithm and assume serializability remains intact.

Single-shard transactions use a one-phase path where all writes, constraints and the decision can be replicated atomically in that group. If an operation adds a participant, use the distributed path before committing.

For multiple shards:

1. Record participant membership under a fenced coordinator epoch. Obtain shared/exclusive key and predicate locks, then validate constraints under the same transaction.
2. Prepare each participant: replicate write intents and enough lock/transaction recovery state to survive leader loss. Participants promise not to commit or release prepared state independently.
3. After every participant has prepared, allocate a commit timestamp consistent with observed dependencies and persist `COMMITTED` in the transaction-status quorum. Otherwise persist `ABORTED` through the valid decision authority.
4. Apply the decision to participants idempotently. Reads encountering unresolved intents consult the authoritative decision or wait; they never expose a partially committed multi-shard transaction.
5. Acknowledge success after the durable decision and required participant prepare durability establish recoverability. Asynchronous cleanup must not alter the outcome.

If a response is lost, return/resolve an **unknown outcome** using the transaction/request ID. Do not report failure as proof that a transaction did not commit. Never retry arbitrary user transactions silently: nondeterministic values and external side effects matter. Allow bounded transparent retries only for explicitly safe operations before returning visible results.

Leader failover reconstructs prepared locks before serving conflicting work; unprepared attempts must be safely fenced/aborted or recovered. Resolve deadlocks by a documented global order/victim rule and return a retryable error. Coordinator lease expiry initiates recovery; it is not permission to discard prepared locks. If the decision quorum is unavailable, affected transactions may block. This is an explicit availability tradeoff.

Read-only optimized snapshots are a separate protocol: publish a stable timestamp below every unresolved transaction that could commit at or below it, account for lagging participants, and check intents. Initially ordinary serializable reads use the locking path. Follower reads require a documented safe timestamp and freshness contract; route fresh reads through the leader/ReadIndex path. HLC timestamps alone do not establish serializability or external consistency. No strict-serializable/external-consistency claim is made until independently checked.

## Native sharding and placement

Every logical table has a distribution policy from creation. Start with auto-selected defaults but show them in introspection and plan previews.

| Strategy | Appropriate workload | Cost |
| --- | --- | --- |
| Hash primary/entity key | General OLTP and even write spread | Ordered scans touch several shards |
| Range key | Time/range scans, ordered workloads | Monotonic writes can create a hot tail |
| Tenant/workspace placement plus row subdivision | Shared BaaS with residency/plan isolation | Needs automatic subdivision for a large tenant |
| Colocated table group | Parent/child joins and transactions by the same entity key | Layout and key encoding must match across tables |
| Scoped reference tables | Small shared lookup data | Replication/update overhead; explicit size limits |

Do not make `hash(tenant_id)` the only partitioning rule: one tenant can exceed a node. Distinguish tenant placement from row sharding. A single very hot row is still serialized; splitting a shard cannot remove that contention.

Split/move workflow: select a boundary → establish a snapshot and ordered delta stream → catch up a new replica/range → validate completeness → commit a fenced descriptor transition → drain old routing → retire obsolete copies after protected-reader/checkpoint requirements. A split must transfer outstanding intents and predicate-lock coverage; routers may retry stale-epoch requests but must not duplicate applied writes. The detailed split state machine is an M0/M2 proof obligation.

Schedule by bytes, CPU, read/write load, compaction pressure, replica health, tenant budgets and locality. Use hysteresis and transfer budgets to avoid constant movement. Support safe manual pinning/drain in addition to automation. Initial system supports splitting and replica movement; merging ranges is a later optimization.

Unique constraints and foreign keys must remain correct across shards. Local unique indexes suffice only when the distribution key makes uniqueness local. Otherwise use a transactional global index keyed by `(scope, index_id, normalized_value)` and include it as a participant. FK existence/deletion checks lock referenced keys; cascades need bounded size and all-or-nothing behavior. Unsupported distributed constraints are rejected at DDL time, never silently weakened. Initial beta may restrict very large cascades and deferred constraints, but ordinary cross-shard uniqueness/FKs are required for certified workloads.

## Query execution and relational features

Pipeline: parse SQL or typed JSON query → authenticate/bind names → inject immutable scope and RLS rules → type check → optimize → construct distributed plan → execute with budgets → stream results. JSON management is a separate typed catalog path; it is not string concatenation disguised as an API.

Implement a low-overhead row/point-operation path for OLTP. Use batches for scans/joins/aggregates where worthwhile. Costing includes shard count, network bytes, row estimates, join distribution and memory/spill pressure. Prefer shard pruning, colocated joins, predicate/aggregate pushdown, top-k pushdown and keyset pagination. Unprunable queries have explicit fan-out, byte and time budgets. Spill is tenant-scoped, encrypted and cleaned after failure.

Core SQL includes exact numeric values, NULL/three-valued logic, Unicode/collation policy, timestamps, JSON, PK/FK/UNIQUE/CHECK/NOT NULL, defaults, generated identities, CRUD/upsert, joins, aggregation, ordering, prepared statements, transactions and explain plans. See the [compatibility matrix](compatibility-and-migration.md) for staged scope. A parser or PostgreSQL wire library is not a complete SQL engine.

## Many connected clients

Gateways use asynchronous I/O with bounded CPU pools for parsing/crypto and bounded worker queues. There is no dedicated backend thread/process/storage connection for every connected user. Query and shard RPCs multiplex over shared channels.

Track four quantities separately: open sockets, authenticated sessions, in-flight requests, and active transactions. An idle connection is cheap; an active transaction can pin locks, snapshots and execution resources. HTTP/2 streams are not separate TCP clients in benchmarks.

Keep prepared-statement/session metadata at the gateway with bounded caches. Transactions pin their coordinator/session until completion in the first implementation. Unsupported session features fail explicitly; do not silently multiplex session-level locks, temporary tables or LISTEN semantics across unrelated sessions. Gateway failure can drop sessions, but durable transaction outcome is recoverable. Client reconnection with jitter and request deduplication are part of the contract.

Admission operates per tenant/workspace and globally: connection limits, authentication-rate caps, active-transaction limits, query CPU/memory/scan budgets, queue deadlines, fair scheduling, slow-reader cancellation and result-size limits. Reserve resources for consensus, recovery and control traffic so user overload cannot prevent failover. Distinguish retryable overload from authorization denial.

## Isolation and security

All entry points, including native PostgreSQL connections, authenticate to a scoped principal. Engine checks enforce tenant/workspace binding, table/column rights and RLS. The SQL-visible `tenant_id` is not the authority; privileged SQL cannot change scope without a separate audited administrative capability. Validate uniqueness/errors/catalog listings so they cannot become cross-tenant existence oracles.

Use TLS externally and mTLS between nodes; short-lived service identity and key rotation; encryption of local durable/spill/backup storage under the chosen deployment model; secret references rather than keys in schema definitions. Integrate with Falcone IAM and Secrets through versioned provider interfaces. Database emergency bootstrap identity must not depend on the database being already online.

Policies/capabilities are cached with a version and expiry. Define maximum revocation delay and fail closed when stale policy cannot be refreshed for privileged mutations. Plan quotas cannot bypass correctness or silently cancel a prepared transaction. Privileged cross-tenant operations require separate credentials, scope declaration and audit. Audit records redact values by default and have durable delivery/replay.

## Failure contract

| Failure | Required behavior |
| --- | --- |
| One of three replicas lost | Continue only where quorum and placement guarantees remain; repair under budget |
| Minority network partition | Minority rejects strong writes/reads; majority proceeds when protocol permits |
| Metadata quorum unavailable | Reject catalog/placement/timestamp-dependent operations; only explicitly safe cached/snapshot reads may continue |
| Coordinator fails mid-commit | Recover final decision and intents; never guess from a timeout |
| Disk full/corruption | Throttle/fail safely, preserve recoverable log, isolate unhealthy replicas and alert |
| S3 unavailable | Ordinary row-only operations can continue; blob access and backup progress degrade explicitly |
| IAM/Secrets temporarily unavailable | Use only valid cached material within its policy; fail closed after expiry; report dependency health |
| Gateway restart | Clients reconnect; committed writes remain committed; unknown outcomes resolve by request ID |
| Region loss | Use tested backups/DR profile; single-region replication does not promise region-loss RPO zero |

The engine prioritizes consistent acknowledged data under partition. It does not promise uninterrupted writes everywhere, unlimited connections, zero-latency cross-shard transactions, or universal SQL compatibility.

## Proposed module boundaries

Rust workspace crates: `types`, `catalog`, `sql`, `planner`, `executor`, `transactions`, `storage`, `replication`, `sharding`, `protocol-pg`, `api-admin`, `auth-policy`, `geo`, `objects`, `cdc`, `backup`, `telemetry`, `simulation`, and `testkit`. Within first-party `storage`, define `format`, `io`, `wal`, `memtable`, `table`, `manifest`, `cache`, `compaction`, `recovery` and `inspector` modules. Start with coarse modules and split crates where compilation and interfaces justify it; this is not an instruction to create empty scaffolding for every module.

Maintain versioned disk/wire/catalog formats, capability negotiation, golden encoding fixtures and a mixed-version support matrix. A destructive format upgrade requires a tested forward plan and defined downgrade boundary.
