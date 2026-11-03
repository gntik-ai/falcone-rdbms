# Epics and user stories

Status: proposed backlog. **23 epics, 92 stories.** Storage-engine and disk-format ownership is confirmed; the implementation designs remain proposals. IDs are stable planning identifiers, not GitHub issue numbers. Nothing here is implemented or marked complete. Milestones are defined in [quality and roadmap](../design/quality-and-roadmap.md).

P0 = correctness/foundation release blocker; P1 = required product capability; P2 = expansion. Dependencies refer to completion of the relevant slice, not a ban on early interface design. Every story requires observable acceptance evidence and documentation of limits. Security, recovery and compatibility checks apply where relevant, rather than being postponed to a final test phase.


## Review priorities and delivery slices

The [3 November documentation review](../design/documentation-review.md) ranks the essential features, records 20 findings and maps 43 chapters across the three uploaded books. [Distributed core contracts](../design/distributed-contracts.md) define the strengthened scalability, replication, shard transition, snapshot and execution obligations. Every story below retains its ID and original acceptance scope except the explicitly revised split, snapshot and backup sequencing contracts.

Core order: native durability/recovery → local transactional SQL → replication/distributed integrity → elastic sharding and bounded execution → required geo/S3/CDC/Falcone integration → production evidence. Security, JSON administration, admission and verification advance with each slice. P1 product capabilities remain required for M3. E09-S04 polished UI/CLI is P2/M5; essential operator APIs/tools are delivered earlier. E12 local restore no longer depends on S3. E13-S01/S02 internal envelopes/lineage are P0/M2 foundations; public delivery is M3.

Each story states its own reviewed priority and gate; these override an epic's broad scheduling range. Review checks supplement its acceptance paragraph. Book references use B1/B2/B3 from the review; these are design sources, not implementation evidence. GitHub parent/checklist links remain the navigational hierarchy.

## Epic index

| Epic | Outcome | Priority | Milestone | Depends on |
| --- | --- | --- | --- | --- |
| E01 | Contracts, feasibility and component qualification | P0 | M0 | — |
| E02 | Native storage integration and MVCC | P0 | M1 | E01, E19–E22, E23 M1 recovery slice |
| E03 | Consensus, membership and catalog | P0 | M2 | E01, E02 |
| E04 | Serializable distributed transactions | P0 | M1–M2 | E02; E03 for distributed paths |
| E05 | SQL semantics, execution and optimizer | P0 | M1–M3 | E02, E04 |
| E06 | Native sharding and distributed integrity | P0 | M2 | E03, E04, E05 |
| E07 | Connection scale and admission | P0 | M1–M4 | E01; E05 for workload certification |
| E08 | Tenant security and policy enforcement | P0 | M1–M4 | E01, E02 |
| E09 | SQL-free administration and online DDL; optional UI/CLI | P0; S04 P2 | M1–M3; S04 M5 | E05, E08; E03 for distributed catalog |
| E10 | Geospatial data and queries | P1 | M3 | E05, E06, E09 |
| E11 | Native managed S3 objects | P1 | M3 | E04, E08, E09 |
| E12 | Early restore, distributed backup and PITR | P0 | M1–M3 | E02/E23 for local; E03/E04 for distributed; E11 for objects |
| E13 | Internal commit envelopes, durable CDC and realtime | P1; S01/S02 P0 | M2–M3 | E04, E06, E08 |
| E14 | Falcone integration | P1 | M3 | E07–E09, E11–E13 |
| E15 | PostgreSQL migration and certification | P0 | M3–M4 | E05, E09, E12–E14 |
| E16 | Kubernetes operations and observability | P0 | M1–M4 | E07; E03 for distributed; later E11–E13 |
| E17 | Correctness, performance and release evidence | P0 | M0–M4 | Incremental across E01–E16 and E19–E23 |
| E18 | Multi-region and advanced workloads | P2 | M5 | M4 evidence |
| E19 | Native disk format and manifest | P0 | M0–M1 | E01 |
| E20 | First-party WAL and durable batches | P0 | M1–M2 | E19 |
| E21 | Native sorted tables, reads and cache | P0 | M1 | E19 |
| E22 | Flush, compaction and history reclamation | P0 | M1–M3 | E20, E21 |
| E23 | Recovery, format evolution and storage tools | P0 | M1–M4 | Relevant E19–E22 slices |

E19–E23 are storage implementation work, not later expansion. E02 is the integration/acceptance epic consuming those components; it does not duplicate their algorithms. M1 slices precede distributed work; M2/M3/M4 hardening extends them as replication, backup and format upgrades become available.

## E01 — Contracts and feasibility

Goal: make scope, invariants, dependencies and performance assumptions reviewable before implementing a broad engine.

**Review basis:** B1 chs. 1, 5, 15; B2 chs. 1, 2, 7–10; B3 chs. 3, 5, 7, 13–14. Findings R01, R05, R10. Freeze workload, failure model and protocol choices before broad implementation.

**E01-S01 — Compatibility inventory.** As a Falcone maintainer, I want actual database dependencies and SQL usage inventoried so migration scope is based on evidence.

**Reviewed priority / gate:** P0 / M0.

Acceptance: produce a versioned schema/extension/driver/query inventory and supported/blocked classification; include internal platform data and DocumentDB separately; map every baseline requirement to an epic and milestone. No production-readiness conclusion is inferred from documentation alone.

Review checks:

- Inventory connection churn, tenant/key skew, transaction participants, row/index/result sizes and growth forecasts.
- Classify three deployment profiles and map each mandatory requirement to a falsifiable gate; prioritize a named Falcone workload over broad feature parity.

**E01-S02 — Protocol models.** As an engine engineer, I want transaction, shard and object-GC state machines so dangerous transitions can be checked before code expands.

**Reviewed priority / gate:** P0 / M0.

Acceptance: models cover coordinator/leader failure, prepared state, cutover fencing and backup-pin/GC races; state invariants and liveness assumptions explicitly; retain model-check results and counterexamples. Independent review closes known invariant violations.

Review checks:

- Model split cancellation versus activation, frozen transaction participants, delayed messages and snapshot-frontier advancement.
- State crash/partition/disk assumptions, distinguish safety from liveness, and record which obligations remain unproven before implementation.

**E01-S03 — Native storage and primitive qualification.** As a maintainer, I want a measured first-party storage design and qualified primitive libraries so Falcone owns persistence without hiding an external engine.

**Reviewed priority / gate:** P0 / M0.

Acceptance: compare first-party LSM/B+tree prototypes under equal durability/workloads and freeze the selected layout in an ADR; external engines may be separate benchmarks only; qualify I/O/checksum/compression/crypto, Raft and protocol/GIS primitives with pinned versions/licenses/SBOM; reject shipping transitive embedded-engine backends; decide source license before adding a license file.

Review checks:

- Measure both native candidates with secondary indexes, retained MVCC, maintenance p99, free-space reserve and recovery time under equivalent fsync/RF settings.
- Select one shipping layout and document why the rejected candidate lost; include comparator/version evolution and primitive persistence responsibilities in the ADR.

**E01-S04 — Benchmark and simulation foundation.** As a reliability engineer, I want reproducible workload/fault tools so every later result can be independently rerun.

**Reviewed priority / gate:** P0 / M0.

Acceptance: define hardware manifests, seeded data/workloads and deterministic clock/network/disk interfaces; distinguish sockets, sessions and requests; save executable configuration and machine-readable results. Demonstrate replay of an intentionally injected failure.

Review checks:

- Add open-loop offered-load sweeps, merged response histograms, error/retry counts and workload/cache-ratio manifests.
- Demonstrate reproducible load reduction after overload and a deterministic storage/network fault; keep capacity targets provisional until measured.

## E02 — Local persistence and MVCC

Goal: integrate the first-party storage components delivered by E19–E23 into a recoverable single-node row/index/MVCC engine with stable encodings and bounded storage behavior. The detailed WAL/table/manifest/compaction implementations belong to those epics.

**Review basis:** B1 chs. 4–6; B2 chs. 4, 5, 8; B3 chs. 1, 3, 5, 7. Findings R11, R12, R15. Integrate physical durability, SQL visibility and bounded shared resources without conflating them.

**E02-S01 — Key/type encoding.** As an engine developer, I want versioned ordered keys and row encodings so scans and indexes remain correct across upgrades.

**Reviewed priority / gate:** P0 / M1.

Acceptance: golden vectors cover numeric order, UTF-8, NULL, decimals, UUIDs and timestamp boundaries; scope prefixes cannot collide; decode rejects malformed/unknown versions; round-trip and ordering property tests pass.

Review checks:

- Specify null versus missing, signed-zero/NaN policy, normalized decimal/composite ordering and comparator/hash/collation versions.
- Primary/secondary row locators remain logical across compaction and movement; a semantic encoding change requires a validated migration.

**E02-S02 — Atomic apply and recovery.** As an operator, I want acknowledged local commits to survive process failure.

**Reviewed priority / gate:** P0 / M1.

Acceptance: atomically persist state changes with applied-log position; replay is idempotent; crash injection at write/fsync/apply boundaries preserves acknowledged data and constraints; document durability configuration and storage assumptions.

Review checks:

- Separate physical batch durability from SQL outcome: recovery preserves unresolved intents without exposing them as committed rows.
- Crash after durable append but before visible apply or response returns a recoverable known/unknown outcome, never a false definitive abort.

**E02-S03 — MVCC and retention.** As a reader/backup worker, I want protected versions so concurrent writes cannot invalidate my allowed snapshot.

**Reviewed priority / gate:** P0 / M1.

Acceptance: version visibility and tombstones are tested; active snapshots/transactions/backup pins prevent premature GC; expired snapshots fail explicitly; retained history and disk-pressure metrics are exposed; long snapshots cannot grow storage without a configured response.

Review checks:

- Differentiate a physical engine view from the distributed transaction-consistent cut required by E04-S04.
- Exercise delete/reinsert, unresolved prepare, snapshot expiry and oldest-pin advancement during compaction; expiry invalidates the reader before reclamation.

**E02-S04 — Resource-bounded storage.** As a shared-cluster operator, I want controlled compaction/cache use so one tenant cannot exhaust the node.

**Reviewed priority / gate:** P0 / M1.

Acceptance: storage domains, shared memtable/cache budgets and background threads stay within configured bounds as shard count rises; checksums detect injected corruption; native compaction pressure triggers backpressure; stress evidence includes noisy-tenant and disk-full cases.

Review checks:

- Measure both idle and active range-count overhead, including files, descriptors, memtables, locks and background queues.
- Enforce aggregate reservations across spill/flush/compaction/backup/movement; overload recovery preserves the consensus/recovery reserve.

## E03 — Replication and metadata

Goal: replicated data and a catalog that safely survives topology change.

**Review basis:** B2 chs. 6, 7, 9, 10; B3 chs. 8–11, 14. Findings R02, R09, R19. Qualify replicated authority, metadata capacity and safe membership at the small and scale-out profiles.

**E03-S01 — Replicated shard log.** As an application owner, I want a shard to survive one voter failure without losing acknowledged writes.

**Reviewed priority / gate:** P0 / M2.

Acceptance: three-voter tests span independent failure domains; minority writes are refused; leader changes preserve committed order; logs are persisted before required acknowledgements; stale leaders cannot serve unauthorized strong reads/writes.

Review checks:

- Qualify the selected core and durable adapter against elections, old-term entries, stale-leader reads and applied-state barriers.
- Run small-production and scale-out topologies; one lost voter under foreground load preserves acknowledged outcomes and reports affected-shard unavailability.

**E03-S02 — Snapshot and membership recovery.** As an operator, I want to replace failed storage members safely.

**Reviewed priority / gate:** P0 / M2.

Acceptance: learners catch up from verified snapshots; membership changes preserve quorum; interrupted snapshot transfer resumes/restarts safely; transaction/lock recovery state is included; old logs are not truncated before recoverability is established.

Review checks:

- Distinguish in-group replica replacement from split ownership transfer; interrupted installs cannot mix snapshots, identities or applied positions.
- Compare canonical logical state at the same cut; quarantine mismatches and repair from verified history, never arbitrary last-write-wins merging.

**E03-S03 — Catalog and routing epochs.** As a query worker, I want versioned metadata so stale routing and schema caches cannot corrupt writes.

**Reviewed priority / gate:** P0 / M1–M2.

Acceptance: bootstrap metadata without an external database; cache invalidation/version checks are tested; stale requests receive retryable epoch errors; catalog quorum loss has documented behavior; restored descriptors cannot reuse an unsafe ownership epoch.

Review checks:

- Show cached row routing avoids per-row catalog transactions; bound watch/cache invalidation and plan-refresh storms.
- Cluster incarnation, schema version and shard generation remain distinct; restored or stale actors cannot revive retired ownership.

**E03-S04 — Placement and timestamps.** As an operator, I want placement policies and timestamp allocation to survive metadata failover.

**Reviewed priority / gate:** P0 / M2.

Acceptance: replicas honor topology/residency restrictions; monotone allocation survives restart/leader change; batched allocation preserves required ordering; load tests measure allocation contention and metadata availability separately from shard throughput.

Review checks:

- Budget catalog/placement/timestamp/resolution queues separately and publish their saturation and failover behavior.
- Timestamp reservations preserve dependencies and are closed/accounted for before stable-frontier advancement; unique allocation alone is not a snapshot proof.

## E04 — Serializable transactions

Goal: precise atomicity/isolation and recoverable outcomes across shards.

**Review basis:** B1 §5.4; B2 chs. 8–10; B3 chs. 5, 13–14. Findings R05, R06, R07, R08. Preserve serializability and durable decisions; make consistent internal cuts a core dependency.

**E04-S01 — Single-shard serializability.** As a developer, I want transactions to preserve invariants under concurrent reads/writes.

**Reviewed priority / gate:** P0 / M1.

Acceptance: strict key/predicate locking covers existing and absent rows; write-skew/phantom/lost-update histories are rejected or serialize correctly; constraints execute under locks; deadlocks return a documented retryable error; commit/abort releases state safely.

Review checks:

- Benchmark lock contention, upgrades, long scans and absent-key predicates; document deadlock victim/prevention rules and lock count/byte limits.
- A resource limit may abort active work through authority but cannot erase a prepared promise; retries discard earlier reads and remain bounded.

**E04-S02 — Cross-shard atomic commit.** As a developer, I want transfers touching several shards to commit together.

**Reviewed priority / gate:** P0 / M2.

Acceptance: replicate participant prepare state and final transaction decision; fault tests at every phase show no partial visibility or split decision; participant registration is fenced; global index/FK participants are included; one-shard optimization cannot bypass required participants.

Review checks:

- Freeze the participant set before the final decision, including independently sharded indexes and referenced-row checks.
- Inject lost prepare/decision replies and delayed old-coordinator messages; every visible result conforms to one outcome and valid lock ownership.

**E04-S03 — Recovery and unknown outcomes.** As a client, I want to discover whether a timed-out commit succeeded.

**Reviewed priority / gate:** P0 / M2.

Acceptance: transaction/request IDs resolve outcomes after coordinator failure; a stale coordinator cannot decide again; prepared locks survive/reconstruct on leader loss; unavailable decision quorum causes an explicit wait/error, not guessed abort; retries cannot double-apply a committed request.

Review checks:

- Persist scope/request key/operation/payload hash and outcome under the same transactional authority as effects; changed-payload key reuse fails.
- Document authorized lookup, dedup retention and client retry horizon; expired history yields reconciliation/explicit expiry rather than blind reexecution.

**E04-S04 — Safe read paths.** As a reader, I want documented freshness and consistent results during failover.

**Reviewed priority / gate:** P0 / M2.

Acceptance: initial leader reads satisfy the locking protocol and applied-state authority checks; required internal consistent snapshots have a reviewed frontier or declared quiesced-barrier implementation with unresolved-intent, clock and lag tests; external follower/snapshot modes need separate freshness/isolation qualification; unsupported modes fail explicitly; no mixed-mode anomaly is accepted.

Review checks:

- Deliver a transaction-consistent internal cut by M2 for backup/backfill/CDC, including unresolved decisions, schema/topology and timestamp reservations.
- Prove the frontier algorithm or a declared quiesced barrier fallback; public follower/snapshot modes remain separately gated and cannot silently replace ordinary locking reads.

## E05 — SQL and query execution

Goal: a useful relational engine with an honest compatibility surface.

**Review basis:** B1 chs. 2, 7–15; B2 chs. 4, 7, 11; B3 ch. 1. Findings R13, R14, R20. Deliver real bounded operators, statistics and useful plans before broad SQL expansion.

**E05-S01 — SQL/type semantics.** As a developer, I want a documented SQL subset with reliable values and errors.

**Reviewed priority / gate:** P0 / M1.

Acceptance: parser/binder/type system supports the M1 profile; differential tests cover NULL logic, numeric overflow, decimal exactness, timezones and collations; unsupported syntax/functions produce stable errors; every released type has wire and JSON encoding tests.

Review checks:

- Bound input size, parse depth, binding work and expression complexity before resource exhaustion.
- Test equality/order agreement between SQL, wire/JSON codecs and persistent comparators, including mixed schema versions and unsupported casts.

**E05-S02 — Relational operations.** As an application developer, I want CRUD, joins and constraints without application-side integrity workarounds.

**Reviewed priority / gate:** P0 / M1–M2.

Acceptance: CRUD/upsert subset, joins, aggregates, ordering, identities and views work inside the transaction layer; SQL and typed API paths share checks; tests include rollback after constraint failure and query cancellation; supported cascade limits are explicit.

Review checks:

- Exercise pipelining, external sorting, grouping and supported join operators on datasets larger than memory, with duplicate/null/empty inputs.
- Memory grants precede allocation; skewed hash partitions have a terminating fallback, and cancel/error paths release cursors, locks where allowed, pins and scoped spill.

**E05-S03 — Distributed optimizer.** As an operator, I want efficient plans and predictable resource limits.

**Reviewed priority / gate:** P0 / M1–M2.

Acceptance: explain reports shard pruning, joins, estimated rows/network and policy filters; colocated joins avoid unnecessary exchange; distributed aggregate/limit semantics match reference results; memory/spill/fan-out caps are enforced and measurable.

Review checks:

- Maintain versioned cardinality/width/null/distinct statistics plus sampled skew; cap planner search and use a conservative stale-statistics fallback.
- Explain estimates and actual rows/network/spill/fan-out; verify pushdown/top-k/joins preserve collation, NULL, volatility and security barriers under concurrent memory pressure.

**E05-S04 — Driver and bulk profile.** As a Falcone maintainer, I want standard clients and import/export tools to work within the declared subset.

**Reviewed priority / gate:** P0 / M1–M3.

Acceptance: certify selected Node `pg` version for simple/extended/prepared queries, transactions, errors/cancel; add named client profiles as tested; selected COPY/CSV/JSON paths handle large streams with bounded memory; introspection never claims nonexistent features.

Review checks:

- Certify cursor/result lifecycle, failed-transaction state, cancellation and reconnection using the named driver and protocol versions.
- Bulk loading respects admission/index/constraint maintenance and transactional batch boundaries; a mid-stream failure reports what committed without unbounded buffering.

## E06 — Sharding and distributed indexes

Goal: native conditional sharding and safe distribution with relational correctness. Small production deployments replicate one/few ranges; growth adds ranges without changing logical endpoints.

**Review basis:** B1 ch. 12; B2 chs. 7–10; B3 chs. 1, 13–14. Findings R02, R03, R04, R14. Prove conditional physical sharding, fenced transitions and distributed index integrity.

**E06-S01 — Distribution policies.** As a workspace administrator, I want hash/range/colocated layouts that fit my workload.

**Reviewed priority / gate:** P0 / M2.

Acceptance: policy preview explains routing and tradeoffs; large tenants can subdivide across nodes; joins use compatible colocation groups; changing a distribution key is rejected or executed through an explicit movement plan; no hidden one-tenant-one-shard limit.

Review checks:

- Prove one/few replicated ranges can grow while logical schemas/endpoints remain stable; compatible small tenants/tables do not require dedicated groups.
- Version the hash-token mapping independently of node count; expose pre-splitting, range-policy hotspots and indivisible colocation limits for oversized entities.

**E06-S02 — Split and move.** As an operator, I want the cluster to redistribute data while clients continue using logical tables.

**Reviewed priority / gate:** P0 / M2.

Acceptance: snapshot/delta/fenced-drain/cutover states survive restart; affected active transactions drain and prepared decisions resolve before child activation; unresolved prepares block cutover rather than losing locks; stale routers cannot duplicate writes; data/index checksums match before retiring old copies; movement respects bandwidth and capacity limits. Live lock migration is deferred pending separate proof.

Review checks:

- Implement PLANNED/COPYING/DRAINING/SEALED/PUBLISHED/RETIRING/COMPLETE with durable operation identity and source/destination authority evidence.
- Initial cutover drains affected transactions under a replicated fence; unresolved prepares block it. Measure admission pause and test cancellation, stale routers and controller crashes at every state.

**E06-S03 — Global constraints.** As a developer, I want unique values and foreign keys enforced regardless of physical shard location.

**Reviewed priority / gate:** P0 / M2.

Acceptance: concurrent cross-shard uniqueness/FK races preserve invariants; global index mutations participate in the row transaction; referenced deletes and cascades cannot leave orphans; unsupported constraint configurations are rejected before table publication.

Review checks:

- Differentiate local and independently sharded global unique/nonunique indexes; locators survive physical rewrites and moves.
- Test changed indexed values, global uniqueness normalization and referenced deletes concurrently with split/backfill; no asynchronous view may enforce a synchronous constraint.

**E06-S04 — Hotspot and balance control.** As an operator, I want placement to respond to actual load without oscillation.

**Reviewed priority / gate:** P0 / M2.

Acceptance: scheduler uses bytes/load/health/locality and respects tenant budgets; hysteresis reduces repeated moves; single-hot-key contention is diagnosed accurately; manual pin/drain and automatic rebalancing are audited; scale-out results include ongoing migration overhead.

Review checks:

- Cap simultaneous transfers and reserve destination/temp capacity; use hysteresis, minimum residence time and safe operator pause/drain.
- Test hot range, hot row, monotone tail and oversized tenant separately, including overload recovery; split only when it can address the measured bottleneck.

## E07 — Connection scale and admission

Goal: serve many clients without confusing connected sessions with executing work.

**Review basis:** B1 chs. 2, 4, 11, 14; B2 chs. 2, 9; B3 ch. 8. Findings R01, R06, R15. Bound sessions, executing work, locks, queues and retries independently.

**E07-S01 — Lightweight gateway sessions.** As an application operator, I want many idle authenticated clients without one backend worker per client.

**Reviewed priority / gate:** P0 / M1–M2.

Acceptance: asynchronous gateway handles the reference session target with measured userspace/kernel/TLS memory; shared shard RPC channels do not grow linearly per client; session cleanup closes all resources; FD/connection-limit errors are observable and safe.

Review checks:

- Account for userspace plus kernel/TLS/socket buffers, auth refresh and per-session prepared metadata at the agreed target.
- Report sockets, authenticated sessions, requests and transactions separately; no per-client store process, fixed memtable or dedicated shard RPC connection.

**E07-S02 — Session semantics.** As a driver user, I want prepared statements and transactions to behave correctly under multiplexing.

**Reviewed priority / gate:** P0 / M1–M2.

Acceptance: bounded session caches preserve parameter types and authorization; active transactions pin the required coordinator; unsupported temp-table/advisory-lock/LISTEN behavior is rejected; no session state crosses tenants; gateway failure yields reconnection and recoverable outcomes.

Review checks:

- Define reset and failed-transaction behavior when sessions/clients reconnect; results and cancellation remain scoped.
- Prepared-plan invalidation observes schema/policy/distribution versions, with bounded refresh and no silent feature emulation.

**E07-S03 — Fair admission.** As a tenant, I want my reserved workload to survive another tenant's overload.

**Reviewed priority / gate:** P0 / M1–M2.

Acceptance: enforce per-scope connections, active requests/transactions, queue deadlines and CPU/memory/scan budgets; protect consensus/recovery capacity; slow readers are cancelled within limits; noisy-neighbor tests meet the agreed fairness objective without unbounded queues.

Review checks:

- Reserve memory, locks, transport buffers and temporary space across foreground and background work before admitting it.
- Test slow readers, long transactions, compaction debt and a retry storm followed by load reduction; normal service resumes without losing prepared state or restarting the cluster.

**E07-S04 — Churn and elastic gateways.** As an operator, I want to add/drain gateways under load.

**Reviewed priority / gate:** P0 / M2–M4.

Acceptance: scale-out increases tested connection capacity; drain stops new work and handles active transactions within policy; reconnect/authentication storms use bounded resources and jitter; distributed load generators demonstrate actual simultaneous clients and LB limits.

Review checks:

- Use end-to-end deadlines and shared retry budgets so nested reconnect/RPC retries do not multiply offered load unboundedly.
- Separate gateway capacity growth from query/store/control bottlenecks and publish drain behavior for unknown outcomes and long-lived sessions.

## E08 — Tenant security

Goal: enforce tenant/workspace isolation in the engine across every access path.

**Review basis:** B2 chs. 5, 13, 14; B1 chs. 7, 10. Findings R12, R20. Preserve immutable authorization scope through every direct, derived and recovery path.

**E08-S01 — Immutable security scope.** As a tenant, I want my data isolated even if an application sends crafted SQL or JSON.

**Reviewed priority / gate:** P0 / M1.

Acceptance: principal/path scope is validated; SQL settings or body fields cannot replace authority; tests cover tables, indexes, catalogs, errors and caches across native protocol/JSON/internal APIs; cross-scope operations require separate audited privilege.

Review checks:

- Exercise direct store/protocol requests, stale plans, global-index lookups and restored data under immutable scope.
- Error timing/detail, stats and transaction-outcome lookup cannot become cross-tenant existence oracles in the declared threat model.

**E08-S02 — RBAC and row policies.** As a workspace administrator, I want roles and row/column policies consistently enforced.

**Reviewed priority / gate:** P0 / M1–M4.

Acceptance: reads, inserts, updates, deletes, bulk paths and policy changes use the same engine checks; forced policies cannot be bypassed by an ordinary table owner; revoked access expires within the specified window; stale privileged policy fails closed.

Review checks:

- Verify policy enforcement survives optimizer pushdown, joins, bulk writes, backfill and schema publication.
- Define revocation behavior for active sessions/transactions and CDC; privileged stale caches fail closed without unilaterally aborting prepared decisions.

**E08-S03 — Transport, secrets and encryption.** As an operator, I want credentials and durable data protected through rotation/recovery.

**Reviewed priority / gate:** P0 / M1–M4.

Acceptance: external TLS and internal mTLS reject invalid peers; credential/key references never appear in query logs; selected at-rest encryption model covers logs/spill/backups; rotation and restore with retained keys are tested; bootstrap does not require a running tenant database.

Review checks:

- Include consensus/engine WALs, temporary runs, diagnostics, snapshots and retained backups in the encryption/key recovery inventory.
- Mixed-version rotation and lost-provider drills preserve access only within valid policy; credentials and signed URLs are redacted.

**E08-S04 — Auditing and scoped resources.** As an auditor, I want reconstructable administrative activity without sensitive-value leakage.

**Reviewed priority / gate:** P0 / M1–M4.

Acceptance: audit mutations with actor/scope/resource/decision/request ID; durable replay handles downstream outage; redaction tests cover query/object credentials; scoped export, backup, temporary files and error messages cannot disclose another tenant's data.

Review checks:

- Document technical export/deletion/retention behavior across rows, indexes, CDC, object versions and backups without promising immediate physical erasure.
- Audit retries and reconciliations by stable operation identity; an unavailable audit sink has bounded buffering and explicit policy.

## E09 — Administration and schema lifecycle

Goal: complete supported administration through typed APIs without requiring SQL.

**Review basis:** B1 chs. 7, 9–11; B2 chs. 5, 8, 9, 12. Findings R07, R08, R12, R20. Prioritize typed APIs, durable plans and schema transitions; polished UI/CLI is P2.

**E09-S01 — Versioned API contracts.** As a platform integrator, I want discoverable typed management operations.

**Reviewed priority / gate:** P0 / M1.

Acceptance: OpenAPI/JSON Schema and Protobuf cover the resource table in the API design; generated clients preserve decimals/int64/errors; auth/limits are equivalent across transports; capability discovery reports actual implementation support.

Review checks:

- Core JSON/gRPC administration is M1; capability/version/limit discovery reflects implementation rather than planned features.
- Test int64/decimal strings, omitted/null/default fields and supported unknown-field preservation across old/new clients and servers.

**E09-S02 — Reviewable idempotent plans.** As an administrator, I want to preview and safely apply a schema or placement change.

**Reviewed priority / gate:** P0 / M1–M2.

Acceptance: plan captures content/catalog/policy versions and estimated impact; apply reauthorizes and rejects stale versions; duplicate identical requests return the same operation; key reuse with changed content fails; progress distinguishes accepted from successfully applied.

Review checks:

- Persist plan/effect/outcome identity and define dedup expiry, authorization and safe retry horizon.
- Schema and placement plans expose affected scope, resource estimates, actual cutover pause and irreversible cancellation boundaries; stale generations cannot apply.

**E09-S03 — Online indexes and columns.** As an application owner, I want supported schema changes without inconsistent reads during backfill.

**Reviewed priority / gate:** P0 / M2–M3.

Acceptance: schema-state transitions are durable; concurrent writes update new structures correctly; uniqueness validation precedes publication; failure/restart/cancel honors the documented commit point; data is reclaimed only after old readers are safe.

Review checks:

- Use E04-S04 consistent cuts and durable pins; schema leases gate old/new writer compatibility and reclamation.
- Test index build/drop concurrently with split/move, leader loss and mixed-version writers, verifying no missing index rows or prematurely reclaimed columns.

**E09-S04 — Administrative UI/CLI.** As a workspace administrator, I want database/role/index/backup/session controls in Falcone's console and a CLI.

**Reviewed priority / gate:** P2 / M5.

Acceptance: both use published APIs, display limits and async status, and support role restrictions; no privileged SQL entry is required for supported operations; destructive actions show their actual scope; cancellation reports whether work already committed.

Review checks:

- This polished console/CLI is P2/M5; JSON APIs and essential inspection/recovery tools ship earlier through E09/E16/E23.
- UI status must distinguish accepted, blocked, committed and complete work and display real limits/cancellation consequences without implying unsupported capabilities.

## E10 — Geospatial

Goal: native location-aware tables and queries with a precisely defined semantic subset.

**Review basis:** B2 chs. 4, 7; B1 chs. 10, 12; B3 chs. 1, 3. Findings R14, R17. Retain the baseline spatial subset and exact results across index and shard paths.

**E10-S01 — Spatial types and formats.** As an application developer, I want to store geometry and WGS84 locations alongside relational data.

**Reviewed priority / gate:** P1 / M3.

Acceptance: supported geometry/Multi types and geography points round-trip via GeoJSON/WKT/WKB; SRID/axis order are explicit; invalid/mixed-SRID inputs follow documented errors; SQL and JSON representations agree; unsupported 3D/raster features are not advertised.

Review checks:

- Version persistent spatial encodings independently from candidate geometry-library versions.
- Declare size/vertex/depth limits and reject malformed or oversized input before uncontrolled allocation; retain exact SRID/axis metadata through backup and migration.

**E10-S02 — Exact predicates and distance.** As a developer, I want spatial answers with known units and edge-case behavior.

**Reviewed priority / gate:** P1 / M3.

Acceptance: declared intersects/contains/within/distance subset matches approved oracle fixtures; geometry uses CRS units and geography uses meters; tests cover holes, boundaries, empty values, poles and antimeridian; precision tolerances are published per operation.

Review checks:

- Freeze per-operation units, tolerances and invalid-input rules in shared SQL/JSON fixtures.
- Compare indexed and full-scan results for degenerate and boundary cases; library upgrades cannot silently change certified semantics.

**E10-S03 — Persistent spatial index.** As an operator, I want spatial filters accelerated without changing answers.

**Reviewed priority / gate:** P1 / M3.

Acceptance: MVCC index updates commit atomically with rows; conservative coverings have no false negatives in generated fixtures; exact recheck removes candidates; oversized geometries use bounded fallback; crashes/backfill/split/move preserve index equivalence to full scans.

Review checks:

- Verify candidate generation is conservative across all persisted versions and topology transitions, not just the in-memory structure.
- Bound cell explosion and row fetch work; large-geometry fallback remains correct under memory pressure and authorization filters.

**E10-S04 — Distributed spatial queries.** As a tenant, I want radius/location queries to work across my shards with predictable cost.

**Reviewed priority / gate:** P1 / M3.

Acceptance: planner shows spatial filtering/pruning and fan-out; all result rows retain scope enforcement; pagination and boundary cases are correct; latency/scan metrics compare indexed and full-scan workloads; query budgets stop expensive unpruned scans safely.

Review checks:

- Explain distinguishes spatial-index selectivity from physical shard pruning; an index does not imply one-shard execution.
- Topology-aware pagination and distributed result merging retain a valid snapshot or return expiry, with no silent missing or duplicate logical rows.

## E11 — Managed S3 large objects

Goal: large files are a first-class database capability with correct publication and retention.

**Review basis:** B1 ch. 6; B2 chs. 5, 8, 9, 11–13; B3 ch. 4. Findings R08, R16, R17. Own immutable references and retention while keeping provider bytes outside internal 2PC.

**E11-S01 — Provider bindings and upload.** As a developer, I want to upload large files directly to qualified object storage.

**Reviewed priority / gate:** P1 / M3.

Acceptance: binding validates required provider capabilities and secret references; multipart uploads resume/reconcile by ID; memory stays bounded with large files; quotas reserve uploads; checksum verification does not treat multipart ETag as a content MD5.

Review checks:

- Test provider timeouts after successful multipart completion and reconcile with stable upload identity.
- Measure bounded database memory and quotas with files larger than node RAM; provider capability qualification is explicit and versioned.

**E11-S02 — Atomic reference publication.** As a developer, I want a row to reference only a verified immutable object.

**Reviewed priority / gate:** P1 / M3.

Acceptance: unsealed/cross-scope objects cannot attach; row and reference records commit atomically; aborts leave reclaimable unreferenced objects; duplicate seal/attach requests are idempotent; object replacement creates a new version; unavailable S3 never becomes silent NULL.

Review checks:

- Expose the exact boundary: verified immutable bytes precede atomic row/reference publication; arbitrary S3 is outside internal 2PC.
- Race seal/attach/abort/retry and changed-content identity reuse; S3 unavailability returns a typed availability error, not missing-row semantics.

**E11-S03 — Retention and garbage collection.** As an operator, I want unneeded bytes reclaimed without breaking reads or restores.

**Reviewed priority / gate:** P1 / M3.

Acceptance: active snapshots, PITR, backup and retained-change references pin required objects; GC claims serialize against attach/pin; fault tests cover delete/retry/restore races; external lifecycle settings incompatible with retention are rejected or explicitly unsupported.

Review checks:

- Connect every pin to a declared snapshot/PITR/backup/CDC horizon and serialize new pins against reclamation claims.
- Test restore and delayed attach during retrying delete; never reclaim solely because a refcount reaches zero or an age threshold elapses.

**E11-S04 — Authorized download and accounting.** As a tenant, I want controlled file access and understandable usage.

**Reviewed priority / gate:** P1 / M3.

Acceptance: downloads require current row/object authorization; signed URLs expire with documented revocation limits; range downloads work; report logical/physical/upload/retained bytes and egress separately; secrets/URLs are redacted; S3 outage metrics distinguish byte availability from row health.

Review checks:

- State the signed-URL revocation window and scope checks for range downloads and historical references.
- Expose provider request/egress/retention costs without leaking credentials or another tenant's object existence.

## E12 — Backup and recovery

Goal: prove local restore in M1, distributed consistent restore in M2 and object-complete PITR in M3. Replication does not replace historical backups.

**Review basis:** B1 chs. 3, 5; B2 chs. 6, 8, 9, 12; B3 chs. 5, 14. Findings R07, R16. Prove local restore at M1 and distributed restore at M2; add object-complete PITR at M3.

**E12-S01 — Consistent snapshot manifests.** As an operator, I want a backup representing one recoverable database state.

**Reviewed priority / gate:** P0 / M1–M3.

Acceptance: M1 captures a protected local checkpoint; M2 adds a consistent distributed cut with catalog/transaction metadata and shard lineage; M3 includes object identities/pins; concurrent split/move does not omit or duplicate logical data; publication occurs only after manifest/data verification; failed attempts release pins safely.

Review checks:

- Stage acceptance: M1 local checkpoint/export, M2 transaction-consistent distributed snapshot, M3 managed-object-complete manifest.
- Cover schema/topology/transaction frontier and complete range lineage; per-shard files taken at unrelated times do not pass.

**E12-S02 — Archived history and PITR.** As an administrator, I want to restore to a selected supported time.

**Reviewed priority / gate:** P0 / M3.

Acceptance: archive all required history beyond a base snapshot; report a common recoverable watermark across shards and objects; gaps stop restore with explicit errors; retention accounts for unfinished transactions and schema transitions; achieved archive lag is measured.

Review checks:

- Report the common achievable recovery frontier, archival gaps and retained dependencies; wall-clock target selection has explicit mapping/precision.
- Test unresolved prepares, schema changes, split lineage and referenced object versions across base snapshot plus archived history.

**E12-S03 — Verified restore.** As a tenant, I want my restored database and files to be usable and isolated.

**Reviewed priority / gate:** P0 / M1–M3.

Acceptance: restore local row data into a clean directory at M1 and distributed data into a clean cluster at M2; at M3 validate managed object bytes too; verify checksums, constraints and policies; encryption-key dependencies are preflighted; backup credentials cannot expose unrelated scopes; readiness is withheld until verification completes.

Review checks:

- Run clean-directory restore in M1, clean-cluster restore in M2 and object-inclusive restore in M3.
- Assign a new cluster incarnation; delayed old requests, dedup records, file IDs and cache identities cannot be confused with current authority.

**E12-S04 — Disaster recovery drills.** As an operator, I want known recovery limits for node/region and logical data loss.

**Reviewed priority / gate:** P0 / M2–M4.

Acceptance: rehearse lost-cluster recovery with separate backup/object copies where required; publish measured RPO/RTO and data size; simulate missing objects/keys/history; restore failure leaves a clear resumable or terminal operation state and never a false success.

Review checks:

- Separate single-node/zone failover, logical deletion and whole-cluster loss in drills; replicas are not historical backups.
- Publish achieved RPO/RTO with dataset/index/object sizes, encryption dependencies and validation time; failed restoration never advertises a healthy ready database.

## E13 — CDC and realtime

Goal: define internal committed/schema/topology envelopes by M2 and expose resumable delivery for Falcone events and subscriptions in M3.

**Review basis:** B2 chs. 5, 8, 12, 13; B3 chs. 7, 8. Findings R08, R18. Define internal commit/topology envelopes at M2 and expose resumable delivery at M3.

**E13-S01 — Committed change envelope.** As an event consumer, I want committed changes with transaction and schema context.

**Reviewed priority / gate:** P0 / M2–M3.

Acceptance: stream excludes aborted intents; events include scope, table/schema version, transaction ID, key and stable event ID; cross-shard transactions expose completion boundaries; before/after images obey policy and declared availability.

Review checks:

- Design/exercise the internal envelope by M2 so transaction decisions, schema changes and topology lineage are available before public CDC.
- Specify stable identities and complete cross-shard transaction boundaries without exposing intents or claiming a global order for unrelated transactions.

**E13-S02 — Resume and ordering.** As a consumer, I want to restart without silently missing changes.

**Reviewed priority / gate:** P0 / M2–M3.

Acceptance: opaque checkpoints survive shard splits/moves; per-key order is preserved; cross-shard global ordering is not claimed unless implemented; at-least-once duplicates are identifiable; expired checkpoints require an explicit resnapshot instead of silent truncation.

Review checks:

- Tokens bind cluster incarnation, scope, stream version and per-range positions with split/move lineage.
- Retention has finite quotas and expiry; a consumer behind reclaimed history gets a required-resnapshot error, never silent progress.

**E13-S03 — Snapshot-to-stream handoff.** As a realtime client, I want initial rows followed by changes without a gap.

**Reviewed priority / gate:** P1 / M3.

Acceptance: snapshot and stream share a documented watermark; concurrent transaction tests show no lost change; deduplication handles overlap; subscription authorization is rechecked on policy change; slow consumers encounter bounded buffering and clear recovery instructions.

Review checks:

- Acquire snapshot and stream retention through one protected-cut protocol, including concurrent split/DDL/prepare.
- Define overlap deduplication, authorization revocation and bounded slow-consumer behavior; prove no missing change at the handoff.

**E13-S04 — Falcone event adapters.** As a platform integrator, I want Kafka/realtime delivery without affecting database commit correctness.

**Reviewed priority / gate:** P1 / M3.

Acceptance: adapter preserves scope, event IDs and transaction markers; downstream outage spools/retries within retention policy; backlog is visible; no end-to-end exactly-once claim is made without a tested sink protocol; synchronous database commits do not depend on Kafka availability.

Review checks:

- Document a transactional outbox pattern and sink-side atomic deduplication for consumers needing effect-once behavior.
- Kafka/downstream outages remain decoupled from commit until documented storage/retention limits force explicit backpressure or subscription expiry.

## E14 — Falcone integration

Goal: make the new engine an explicitly selectable database backend under existing governance.

**Review basis:** B1 chs. 2, 11; B2 chs. 1, 5, 12, 13. Findings R17, R18, R20. Integrate only certified capabilities and preserve durable operation identity across the platform boundary.

**E14-S01 — Backend-neutral database adapter.** As a Falcone maintainer, I want both PostgreSQL and Falcone RDBMS supported during transition.

**Reviewed priority / gate:** P1 / M3.

Acceptance: typed provision/admin/query/change/backup contracts have capability negotiation; the new adapter uses gRPC with authenticated context; legacy routes retain documented semantics; unsupported features produce actionable errors rather than false emulation.

Review checks:

- Route scoped plans and request IDs through the same typed semantics as direct clients; certify actual backend capabilities.
- Keep PostgreSQL and Falcone routing/bindings distinct so unavailable extensions are never implied by endpoint naming.

**E14-S02 — Provisioning and lifecycle.** As a tenant administrator, I want create/suspend/delete operations to reflect real engine state.

**Reviewed priority / gate:** P1 / M3.

Acceptance: provisioning resumes after partial failures; “active” requires an actual scoped read/write check; suspension covers new and existing sessions according to policy; deletion coordinates retention/backup/object rules; operation audit correlates platform and database IDs.

Review checks:

- Reconcile create/suspend/delete across lost responses with durable operation identity and generation checks.
- Data deletion respects protected rows/files/objects and reports lifecycle progress, rather than treating provisioning acceptance as successful completion.

**E14-S03 — Identity, secrets and storage providers.** As a platform operator, I want first-party services to integrate without circular availability requirements.

**Reviewed priority / gate:** P1 / M3.

Acceptance: IAM/Secrets/Storage contracts are versioned; standalone bootstrap works; credential/policy rotation and provider outages have bounded behavior; database object GC coordinates with storage retention; no tenant database is required to start its own provisioning control path.

Review checks:

- Test identity/secret/storage outages independently from shard quorum and distinguish their readiness impact.
- Define restore/bootstrap dependency order and avoid a control-plane database dependency on the tenant resource being provisioned.

**E14-S04 — Plans, console and telemetry.** As an administrator, I want permissions, quotas and usage visible consistently.

**Reviewed priority / gate:** P1 / M3.

Acceptance: console hides unavailable capabilities based on actual backend discovery; quotas cover connection/query/storage/blob/backup dimensions; usage avoids duplicate physical/logical charging; traces connect platform request to query/transaction/shard; scoped operator roles are tested end to end.

Review checks:

- Expose sessions versus active queries, logical versus physical storage and reserved versus consumed resources separately.
- Platform quotas cannot bypass core locks/pins/durability; capability display follows certified versions and scoped roles.

## E15 — Migration and compatibility certification

Goal: migrate only workloads whose behavior and rollback boundaries have been demonstrated.

**Review basis:** B1 chs. 2, 9, 10; B2 chs. 5, 7–9, 12, 13. Findings R03, R12, R18, R20. Migration acceptance includes workload distribution, semantics, history and one-writer cutover.

**E15-S01 — Migration assessor.** As a workspace owner, I want to know blockers before moving data.

**Reviewed priority / gate:** P0 / M3–M4.

Acceptance: inventory SQL/types/extensions/policies/drivers and dataset characteristics; produce supported/transformable/blocked findings and a schema plan; pgvector/PL/pgSQL/DocumentDB dependencies are explicit; no blocked workspace can be auto-approved for cutover.

Review checks:

- Assess shard/distribution keys, index placement, participant count, hot entities and query memory alongside SQL/extension compatibility.
- Unsupported advanced SQL/vector workloads remain blocked until named certification evidence exists; priority does not equal implemented support.

**E15-S02 — Snapshot plus PostgreSQL CDC.** As an operator, I want a repeatable import without lost source writes.

**Reviewed priority / gate:** P0 / M3–M4.

Acceptance: snapshot/LSN handoff is gap-free; preserve source transaction boundaries; deduplicate replay; coordinate DDL and sequence values explicitly; handle replica identity and large objects; bound source WAL retention and stop safely when limits are exceeded.

Review checks:

- Exercise source snapshot/CDC handoff through source reconnect, replay duplication and target split/failover.
- Checkpoint source position and target transactional effect coherently; bound source slot/history growth and report unresolved source transactions.

**E15-S03 — Semantic reconciliation.** As an application owner, I want proof that my supported workload still works.

**Reviewed priority / gate:** P0 / M3–M4.

Acceptance: compare counts/checksums at matched positions; verify constraints/policies/objects and query fixtures; shadow reads are side-effect-free and scoped; driver/ORM versions are named; unsupported semantics block promotion rather than being silently normalized away.

Review checks:

- Compare canonical logical results at matched cuts, including comparator/collation, index entries, policy and blob checksums.
- Publish target resource/plan differences and representative skew/large-join results; matching row counts alone cannot pass.

**E15-S04 — Fenced cutover and recovery.** As an operator, I want one authoritative writer and a truthful rollback plan.

**Reviewed priority / gate:** P0 / M3–M4.

Acceptance: fence/drain source writes, catch up to final LSN, verify and atomically change binding generation; old connections cannot keep writing; before-target-write rollback is rehearsed; after-target-write rollback requires validated reverse replication or explicit reconciliation/write freeze.

Review checks:

- Inject delayed source requests and stale binding generations during cutover; exactly one writer authority remains.
- Rehearse failure before and after first target write, naming reverse-CDC/reconciliation requirements and immutable recovery evidence.

## E16 — Operations and observability

Goal: deploy, diagnose and upgrade the engine safely on Kubernetes.

**Review basis:** B2 chs. 2, 5, 7, 9, 13; B3 chs. 9–12, 14. Findings R09, R15, R19. Operate the authority and recovery protocols under overload, corruption and version change.

**E16-S01 — Kubernetes deployment.** As an operator, I want reproducible dev and three-domain cluster profiles.

**Reviewed priority / gate:** P0 / M1–M2.

Acceptance: Helm/operator handles persistent identities/volumes, topology, secrets and disruption budgets; readiness reflects quorum/dependencies; stateless scale-out is independent; stateful scale-in drains/moves replicas safely; namespace deletion cannot silently bypass retention policy.

Review checks:

- Offer explicit dev, small replicated and scale-out profiles with measured range/role overhead and tested growth.
- Drain and upgrade use engine authority/membership operations; generic CPU autoscaling cannot independently remove stateful voters.

**E16-S02 — Metrics and diagnosis.** As an SRE, I want to identify connection, transaction, shard and object bottlenecks.

**Reviewed priority / gate:** P0 / M1–M4.

Acceptance: publish the architecture's metrics and trace fields; sampled query fingerprints omit secrets/values; dashboards distinguish availability/durability/archive freshness; labels stay bounded; alerts identify actionable conditions with runbook links.

Review checks:

- Expose allocation/metadata queues, lock bytes/waits, prepared age, safe-frontier blockers, split pause, spill/exchange and retry amplification.
- Correlate these with compaction/backup/movement reservations and bounded labels; dashboards separate safety, availability and recovery lag.

**E16-S03 — Upgrade and downgrade boundaries.** As an operator, I want version changes without accidental unreadable data.

**Reviewed priority / gate:** P0 / M2–M4.

Acceptance: mixed-version matrix and feature negotiation are tested; rolling sequence preserves quorum; golden disk/wire fixtures remain readable as promised; irreversible format changes require explicit preflight and restore path; downgrade refusal explains the actual boundary.

Review checks:

- Test old/new binaries against old/new schema, wire and file data, including unknown-field read-modify-write preservation.
- Enable new persistent semantics only after required replicas/writers/readers/restore tools are capable; comparator changes require an explicit migration.

**E16-S04 — Runbooks and repair.** As an on-call engineer, I want tested procedures for failures.

**Reviewed priority / gate:** P0 / M1–M4.

Acceptance: rehearse replica replacement, lost quorum, corrupt disk, unknown commits, disk full, S3 outage and backup recovery; repair cannot promote stale unverified data; diagnostic bundles redact tenant data; exercises record achieved recovery times.

Review checks:

- Drill slow/partitioned nodes, prepared-transaction blockers, stalled splits, overload recovery and same-cut replica divergence.
- Repair uses verified history and a new incarnation where required; never guess transaction outcomes or merge corrupt values by timestamp.

## E17 — Release evidence

Goal: release decisions depend on reproducible evidence instead of feature checklists.

**Review basis:** B1 chs. 5, 14, 15; B2 chs. 2, 8–10, 13; B3 chs. 5, 7–14. Findings R01, R04, R06, R19. Require reproducible scenario evidence and publish tested limits rather than feature-count claims.

**E17-S01 — Deterministic fault campaign.** As a reliability engineer, I want reproducible failures throughout development.

**Reviewed priority / gate:** P0 / M0–M4.

Acceptance: saved seeds exercise crashes, partitions, clock steps and disk faults through E02–E13 and E19–E23; histories/checksums expose invariant violations; every fixed fault gains a regression; real filesystem/device and any FFI failure tests complement simulation rather than being assumed covered.

Review checks:

- Add cutover cancellation/activation, metadata-watch storms, stable-frontier reservations, dedup expiry and delayed pre-restore requests to the campaign.
- Separate safety assertions from liveness under stated quorum/resource assumptions; save minimal counterexamples and rerun them after fixes.

**E17-S02 — SQL and security differential suite.** As a reviewer, I want independent checks of semantics and isolation.

**Reviewed priority / gate:** P0 / M1–M4.

Acceptance: common SQL behavior is compared with PostgreSQL; geo uses approved oracle fixtures; histories check promised transaction properties; cross-tenant attacks span every data/management path; all expected deviations are documented and no unexplained failure is waived.

Review checks:

- Cover skewed spill joins, local/global indexes, NULL/collation equality, volatile expressions and security-preserving optimizer rewrites.
- Use transaction histories for claimed isolation and point-operation histories for claimed linearizability; neither substitutes for the other.

**E17-S03 — Capacity and fairness evidence.** As a product owner, I want defensible scalability claims.

**Reviewed priority / gate:** P0 / M0–M4.

Acceptance: publish hardware/data/TLS/RF/durability settings with full latency/error/resource results; run the target connection/churn/skew/scale-out matrix; separate offered from completed load; revise unmet goals transparently; results are reproducible from checked-in harness versions.

Review checks:

- Publish load curves for gateways, execution, storage and metadata, plus idle/active range-count envelopes and cache-size controls.
- Separate steady state from rebalancing/compaction/failure periods; report client queueing, errors, retries and repeated-run variation without averaging percentiles.

**E17-S04 — Production-candidate gate.** As a release owner, I want clear acceptance criteria before supporting durable customer workloads.

**Reviewed priority / gate:** P0 / M4.

Acceptance: complete independent protocol/security review, seven-day mixed-load soak, restore and rolling-upgrade drills; no open critical correctness/security defects; support matrix/SLO/runbooks/versioned artifacts are published; candidate approval cites evidence and remaining known limits.

Review checks:

- Require explicit pass/fail evidence for all three minimum requirements plus the retained geo/S3/admin/native-storage baseline.
- Gate production on clean-cluster/object restore, format evolution and scope-specific SLOs; no unchecked critical protocol obligation is waived as a documentation task.

## E18 — Expansion after the core release

Goal: add advanced capabilities without diluting the initial correctness contract. All four stories are P2 and require separate detailed design before implementation.

**Review basis:** B2 chs. 1, 3, 4, 6, 11–13; B3 chs. 6, 11, 13. Findings R10, R17, R20. Defer alternative workload/consistency models until the core and original baseline are qualified.

**E18-S01 — Multi-region policies.** As a global tenant, I want locality and residency controls with explicit consistency/latency choices.

**Reviewed priority / gate:** P2 / M5.

Acceptance: define synchronous versus async replication profiles, leader placement, safe regional failover and achieved RPO; quantify WAN write latency; verify residency for replicas/backups/objects; no simultaneous partition-tolerant local-write promise is made for strong transactions.

Review checks:

- Treat new regional failure and latency assumptions as a separate protocol profile with its own tests.
- Do not inherit a single-region zero-loss claim across async replication or split-brain regional promotion.

**E18-S02 — S3 analytical tier and branches.** As a developer/analyst, I want inexpensive historical scans and isolated test copies.

**Reviewed priority / gate:** P2 / M5.

Acceptance: separately specify Parquet manifests/snapshot visibility, schema evolution, retention and branch divergence; qualify DataFusion/Arrow integration; analytical CPU/I/O cannot starve OLTP; branch/object GC is tested; no arbitrary S3 file is treated as a transactional table.

Review checks:

- Keep analytical/branch manifests, garbage collection and resource isolation separate from transactional S3 object references.
- Require named workload/cost evidence before adding a second execution/storage tier; branch semantics and retention are independently designed.

**E18-S03 — Search and advanced SQL.** As a Falcone developer, I want richer search and query features when core semantics are stable.

**Reviewed priority / gate:** P2 / M5.

Acceptance: prioritize exact-vector/ANN/full-text and broader SQL by actual migration demand; define recall/freshness and policy-filter behavior; measure distributed top-k correctness; add functions/triggers/UDFs only with a resource and security model; preserve honest PostgreSQL compatibility claims.

Review checks:

- Tie any priority increase to a specific blocked migration and certified semantic/security requirements.
- Search index freshness/recall/top-k and resource behavior are explicit; broader syntax does not imply full PostgreSQL compatibility.

**E18-S04 — Advanced GIS.** As a GIS user, I want more coordinate systems and complex operations.

**Reviewed priority / gate:** P2 / M5.

Acceptance: qualify projection libraries and grid-data licensing/versioning; add tested geography polygons, overlay/buffer/KNN/spatial joins by explicit subset; compare edge-case results and index plans; raster/topology/3D require separate scope and are not implied by a `geometry` column.

Review checks:

- Use separate feature gates and oracle fixtures for each added CRS/geometry/geography operation.
- Persist transformation/grid/algorithm version dependencies and preserve conservative indexing, sharding and budget contracts.

## E19 — Native disk format and manifest

Goal: Falcone specifies, implements and maintains its own durable representation; see [native storage](../design/native-storage.md).

**Review basis:** B1 chs. 3, 6; B2 ch. 5; B3 chs. 3, 4, 6. Findings R10, R12. Make persistent encoding and publication independently specified and version-safe.

**E19-S01 — Format RFC and fixtures.** As an engine maintainer, I want a precise byte-level specification so files survive compiler, platform and software changes.

**Reviewed priority / gate:** P0 / M0–M1.

Acceptance: freeze chosen header/record encodings, file kinds, endian rules, version/feature bits, bounds and checksum coverage after M0 review; golden binaries and a reference decoder agree; malformed/unsupported fields are rejected; no Rust memory layout or foreign-engine file format is the persistence contract.

Review checks:

- Specify comparator, schema, hash and optional-field evolution alongside the common envelope and every file-specific format.
- Fixtures cover cross-platform byte order, numeric edge cases, malformed lengths and conflicting feature bits; source books do not freeze Falcone bytes.

**E19-S02 — File identity and allocation.** As a storage engineer, I want stable domain/file identities so cache, restore and maintenance cannot confuse old and new data.

**Reviewed priority / gate:** P0 / M1.

Acceptance: file IDs/generations remain unique across crash/restore; metadata binds comparator/codec/domain and shard range; temp/final naming and directory durability are specified; concurrent file creation cannot reuse identities; stale files cannot be mistaken for live files from another domain.

Review checks:

- Define allocation across restore/new cluster incarnations and old staging artifacts; identity includes ownership context.
- Concurrent jobs cannot reuse file/cache identities or import a shard file under an incompatible comparator/range descriptor.

**E19-S03 — Atomic manifest publication.** As an operator, I want flush/compaction outputs published without missing files after power loss.

**Reviewed priority / gate:** P0 / M1.

Acceptance: durable outputs precede durable manifest and root-pointer publication; fault injection at every write/sync/rename/directory boundary yields a recoverable old or new state; partial manifests never publish; missing/corrupt authoritative metadata fails closed; older roots are not silently chosen to discard acknowledged work.

Review checks:

- Model old/new manifest visibility separately from reader reachability and log retention.
- Test power loss after each file/root/directory durability boundary and concurrent compaction precondition failure; no half-published file set is accepted.

**E19-S04 — Format ownership and tooling boundary.** As a project owner, I want the shipping engine independent of another database's persistent representation.

**Reviewed priority / gate:** P0 / M1.

Acceptance: dependency checks exclude runtime embedded database/storage engines; first-party format code owns tables/WAL/manifests; reference engines run only in isolated benchmarks; disk-layout documentation and golden fixtures are shipped with a declared version; source provenance/notices are recorded for any reused code.

Review checks:

- Publish a standalone format specification and independent inspection/fixture decoder with the release.
- Check shipping dependency graphs at pinned revisions; benchmark reference engines and qualified general primitives remain explicitly separated.

## E20 — First-party WAL and durable batches

Goal: implement reliable acknowledgement, replay and Raft-storage integration in Falcone code.

**Review basis:** B1 §§4.2, 5.3; B3 chs. 5, 7, 14; B2 chs. 8, 10. Findings R05, R11. Prove durability acknowledgements, complete-prefix retention and Raft/application replay boundaries.

**E20-S01 — Framed WAL and atomic apply.** As a storage developer, I want bounded atomic batches with unambiguous replay.

**Reviewed priority / gate:** P0 / M1.

Acceptance: framed/fragmented records carry batch identity/count/sequence/checksums and completeness markers; full batches become visible together; malformed/gapped/reordered records are detected; duplicate batch IDs with changed contents fail; limits reject work before mutation; no partial local batch is replayed.

Review checks:

- Specify complete-batch visibility, replay identity retention and same-ID/different-content rejection across restart.
- A multi-batch SQL transaction remains intents until its durable SQL outcome; no fragmented WAL batch can expose a partial mutation.

**E20-S02 — Group commit and rotation.** As an application owner, I want a success response only when the required log bytes are durable.

**Reviewed priority / gate:** P0 / M1.

Acceptance: group-commit waiters track individual durable ends; short writes/sync errors/rotation crashes are tested; new segment directory entries are persisted before reliance; write acknowledgement never precedes required sync; disk full fails safely; memory and queue length stay bounded under slow storage.

Review checks:

- Measure group-commit queueing and sync latency under slow disks, many ranges and foreground overload.
- Only waiters whose end positions are durably covered succeed; a failed sync poisons/halts the affected writer until safe recovery.

**E20-S03 — Native consensus-log adapter.** As a distributed-systems engineer, I want Raft entries and hard state persisted without an external storage engine.

**Reviewed priority / gate:** P0 / M2.

Acceptance: implement first-party consensus log/hard-state persistence respecting the selected Raft core's ordering; engine apply stores mutations and shard applied index atomically; restart replays only committed entries above recovered indexes; fault tests cover mismatched log progress, term changes and snapshot installation.

Review checks:

- Test persisted term/vote state, old-term log entries, leader changes, snapshot boundaries and committed-versus-applied lag.
- Do not remove either WAL without a separately modeled replacement durability contract; application state and applied index recover atomically.

**E20-S04 — Checkpoint and log retention fences.** As an operator, I want old WAL reclaimed only after equivalent recovery state exists.

**Reviewed priority / gate:** P0 / M1–M3.

Acceptance: engine checkpoints cover complete batches across all domain memtables; partial shard flushes cannot skip unrelated mutations; consensus and engine retention have separate safe positions; backup/CDC pins are honored; injected checkpoint/truncation races preserve all acknowledged state.

Review checks:

- Shared-domain retention advances through a fully covered prefix, not a maximum flushed sequence from one range.
- Test hot/cold range partial flush, incomplete checkpoint and backup/CDC pin races while consensus and engine positions remain distinct.

## E21 — Sorted tables, reads and cache

Goal: implement Falcone's ordered on-disk data access with bounded resource use.

**Review basis:** B1 chs. 4, 6, 12, 14; B2 ch. 4; B3 chs. 3, 5, 7. Findings R12, R13, R15. Own bounded lookup/iteration/cache behavior and prevent filters or races from changing results.

**E21-S01 — Native table builder and decoder.** As an engine developer, I want immutable tables containing sorted scoped MVCC records.

**Reviewed priority / gate:** P0 / M1.

Acceptance: data/index/property/footer blocks implement the frozen Falcone spec; decoder checks offset arithmetic/lengths/checksums before allocation; ordered scans and point reads match an independent reference model; uncompressed baseline works before adding codec options; cross-domain files are rejected.

Review checks:

- Validate comparator/schema/codec identities before reads; row locators are logical and survive rewrites.
- Fuzz nested length/offset overflow and decompression limits; independent full scans match point/range results at every supported MVCC view.

**E21-S02 — Memtables and atomic visibility.** As a reader, I want concurrent batch application and flush to return complete consistent engine views.

**Reviewed priority / gate:** P0 / M1.

Acceptance: atomic visibility sequence publishes only complete batches; frozen/active memtable transitions have no lost/duplicate logical entries; ordered iteration merges versions correctly; concurrency tests include cancellation and flush; memtable count/bytes are bounded independently of client/shard count.

Review checks:

- Test writes racing freeze/publication and iterators spanning old/new table views with complete-batch visibility.
- Old memory remains readable until durable table installation and pin release; cancelling readers releases resources without changing committed state.

**E21-S03 — Index blocks and filters.** As an operator, I want fast lookups without filters suppressing real matches.

**Reviewed priority / gate:** P0 / M1.

Acceptance: index seeks match full scans for generated key/version distributions; Bloom/prefix filters use versioned matching hash/comparator semantics with zero false negatives in fixtures; missing optional filters fall back safely; corruption is detected; prefix-compressed and uncompressed results agree.

Review checks:

- Cover negative lookups, prefix boundaries, normalized equal values and versioned hash/comparator changes.
- A missing optional filter can fall back; a corrupt required index/block cannot be treated as a clean not-found result.

**E21-S04 — Block cache and I/O budgets.** As a tenant, I want predictable memory use and read latency during scans and file replacement.

**Reviewed priority / gate:** P0 / M1.

Acceptance: cache keys bind immutable file identity/generation/offset; pins protect live readers from deletion; scan admission avoids uncontrolled eviction; tenant/global memory budgets hold; failed/short I/O yields precise errors; repeated flush/compaction cannot return stale cached blocks.

Review checks:

- Budget pinned and decoded blocks, prefetch, OS-cache effects and concurrent query buffers, not just nominal cache capacity.
- Run scan pollution plus point workload and failed file replacement; admission cannot evict pinned state or let one tenant consume the recovery reserve.

## E22 — Flush, compaction and history reclamation

Goal: reclaim and organize disk space without breaking snapshots or acknowledged data.

**Review basis:** B1 chs. 4, 5; B2 chs. 4, 8, 12; B3 ch. 7. Findings R10, R11, R15. Coordinate publication, complete-prefix recovery, amplification and protected history.

**E22-S01 — Flush/checkpoint pipeline.** As a storage engineer, I want frozen memtables materialized into verifiable files.

**Reviewed priority / gate:** P0 / M1–M2.

Acceptance: all mutations and apply metadata through a published checkpoint are covered; files stay within declared shard boundaries; snapshot/read pins survive transitions; partial output failures retain WAL recovery; manifest installation follows E19 ordering; incomplete temporary outputs are never readable as live tables.

Review checks:

- Coordinate memtable freeze, durable file publication and complete-prefix WAL retention under a single reviewed ordering.
- Exercise many tiny ranges and shared batches, including a later range flushed before earlier pending mutations; snapshot coverage remains complete.

**E22-S02 — Native compaction scheduler.** As an operator, I want maintenance to bound read/space amplification without starving queries or recovery.

**Reviewed priority / gate:** P0 / M1–M2.

Acceptance: first-party selection/merge logic respects input versions and non-conflicting ranges; concurrent jobs validate publication preconditions; CPU/I/O/temp-space budgets and disk reserve are enforced; ingestion backpressure precedes exhaustion; report amplification/backlog/stalls and benchmark p99 during compaction.

Review checks:

- Report steady-state and maintenance read/write/space amplification including pending flush, protected history and output reservations.
- Coordinate background budgets with backup/backfill/movement and foreground spill; low space sheds new work before protected/recovery bytes are threatened.

**E22-S03 — MVCC and tombstone safety.** As an application owner, I want compaction to preserve every supported historical view.

**Reviewed priority / gate:** P0 / M1–M3.

Acceptance: protected snapshots, unresolved intents and backup/CDC pins constrain version deletion; tombstones cannot be dropped while hidden older values could reappear; generated histories compare before/after compaction; cross-level overlap and delete/reinsert cases are covered; S3 byte GC stays separately coordinated.

Review checks:

- Keep the latest version required at each retained frontier and every unresolved decision dependency, not just versions newer than one timestamp.
- Generate cross-level delete/reinsert histories with active backups/CDC and topology changes; neither stale rows nor deleted objects reappear.

**E22-S04 — Obsolete files and orphan recovery.** As an operator, I want reclaimable bytes removed without deleting a live file.

**Reviewed priority / gate:** P0 / M1–M3.

Acceptance: deletion requires a published reachability decision plus reader/snapshot/backup safety; crashes between publish/unlink/directory sync leave recoverable state; aborted jobs' outputs are reclaimed by reachability rather than age; file-count/retained-space metrics identify leaks; low space never triggers deletion of protected data.

Review checks:

- Delete by durable reachability plus reader/history pins; age is only an operational signal.
- Race cancellation, publication, unlink, restore and a reused filename; file identity prevents deleting a newly live file.

## E23 — Recovery, upgrades and storage diagnostics

Goal: make the owned format independently inspectable, recoverable and evolvable.

**Review basis:** B1 ch. 5; B2 chs. 5, 9, 13; B3 chs. 3, 5, 7, 14. Findings R11, R12, R16, R19. Recover, inspect and evolve owned files without hiding lost state or discarding unknown data.

**E23-S01 — Cold-start recovery and verification.** As an operator, I want native recovery to either reconstruct valid state or clearly refuse unsafe service.

**Reviewed priority / gate:** P0 / M1–M2.

Acceptance: validate authoritative root/manifest/files then replay complete WAL batches; distinguish permitted incomplete tail from corruption; never skip corrupt committed history; reconcile Raft/application fences; recovery fixtures cover missing segments/files, wrong domains and interrupted publication; no third-party engine is needed to open data.

Review checks:

- Separate incomplete unacknowledged tail handling from corrupt committed/interior history and report the recovery evidence.
- Restore unresolved intents, decision references and all replay fences before serving conflicts; no silent fallback to an older root or guessed abort.

**E23-S02 — Format inspection and safe export.** As an on-call engineer, I want to inspect damaged data without modifying the only copy.

**Reviewed priority / gate:** P0 / M1–M2.

Acceptance: offline tools inspect headers/manifests/WAL, validate checksums and dry-run recovery with redaction; verified restore uses replica/backup evidence; any salvage writes a new output and enumerates missing/corrupt data; tools reject unsupported versions and never report best-effort salvage as full recovery.

Review checks:

- Validate canonical range checksums at common cuts and export source provenance plus missing-data reports for salvage.
- Inspect without SQL startup or source mutation; normal repair accepts only validated replica/backup history and respects version/scope limits.

**E23-S03 — Format upgrades and downgrade limits.** As an operator, I want format changes with explicit compatibility guarantees.

**Reviewed priority / gate:** P0 / M2–M4.

Acceptance: reader/writer/version matrix and golden fixtures cover old/current files; cluster capability checks gate new writes; conversion/rewrite can restart safely; backups record codec/key/format dependencies; irreversible boundaries block unsafe downgrade with an actionable restore route.

Review checks:

- Test mixed-version rewrite preserves supported unknown fields or refuses mutation; comparator/collation changes cannot reinterpret old sorted files.
- Include offline tools, backup codec/key dependencies and new-cluster identity in forward/rollback validation.

**E23-S04 — Persistence fault qualification.** As a release reviewer, I want independent evidence for power-loss and corruption behavior.

**Reviewed priority / gate:** P0 / M1–M4.

Acceptance: deterministically inject short/reordered writes, failed file/directory syncs, torn records, bit flips, disk full and manifest/compaction races; verify acknowledged batches and no tombstone resurrection; test qualified real filesystems/devices and a reference decoder/model; retain seeds and review results before M1/M4 promotion.

Review checks:

- Add manifest/WAL partial-prefix races, retained intents, overlapping tombstones, split staging and mixed-version decoding faults.
- Separate process crash from simulated power loss and actual device/filesystem qualification; persist reproducible seeds and independent review evidence.

## Suggested first implementation slice

Start with E01-S01 through E01-S04 and E17's harness foundations. Implement E19–E23's M1 slice first: encode a native file → append/fsync an atomic batch → publish a memtable view → flush/compact through a manifest → crash/restart and verify the same state. Then integrate E02 and the small SQL path: scoped authenticated connection → create a table through JSON → insert/read → crash/restart → verify row and isolation → restore into an empty directory through E12. Add native consensus-log integration, a two-shard bank transfer, a fenced split and a transaction-consistent clean-cluster restore before broadening SQL/GIS features. Preserve existing planning IDs; storage and recovery epics are prerequisites, not work postponed beyond the RDBMS release.

GitHub issues already exist: epics E01–E23 are #2–#24; stories E01-S01–E23-S04 are #25–#116 in epic/story order. Preserve these IDs and numbers when updating scope. Each epic links its four stories; each story links its parent. Native sub-issue metadata is not assumed. A stub API, plan builder or documentation update does not complete an implementation story.
