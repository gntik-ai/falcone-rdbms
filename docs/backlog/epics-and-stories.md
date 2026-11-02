# Epics and user stories

Status: proposed backlog. **18 epics, 72 stories.** IDs are stable planning identifiers, not GitHub issue numbers. Nothing here is implemented or marked complete. Milestones are defined in [quality and roadmap](../design/quality-and-roadmap.md).

P0 = correctness/foundation release blocker; P1 = required product capability; P2 = expansion. Dependencies refer to completion of the relevant slice, not a ban on early interface design. Every story requires observable acceptance evidence and documentation of limits. Security, recovery and compatibility checks apply where relevant, rather than being postponed to a final test phase.

## Epic index

| Epic | Outcome | Priority | Milestone | Depends on |
| --- | --- | --- | --- | --- |
| E01 | Contracts, feasibility and component qualification | P0 | M0 | — |
| E02 | Durable local storage and MVCC | P0 | M1 | E01 |
| E03 | Consensus, membership and catalog | P0 | M2 | E01, E02 |
| E04 | Serializable distributed transactions | P0 | M1–M2 | E02; E03 for distributed paths |
| E05 | SQL semantics, execution and optimizer | P0 | M1–M3 | E02, E04 |
| E06 | Native sharding and distributed integrity | P0 | M2 | E03, E04, E05 |
| E07 | Connection scale and admission | P0 | M1–M4 | E01; E05 for workload certification |
| E08 | Tenant security and policy enforcement | P0 | M1–M4 | E01, E02 |
| E09 | SQL-free administration and online DDL | P1 | M1–M3 | E03, E05, E08 |
| E10 | Geospatial data and queries | P1 | M3 | E05, E06, E09 |
| E11 | Native managed S3 objects | P1 | M3 | E04, E08, E09 |
| E12 | Backup, PITR and recovery | P0 | M3 | E03, E04, E11 |
| E13 | Durable CDC and realtime | P1 | M3 | E04, E06, E08 |
| E14 | Falcone integration | P1 | M3 | E07–E09, E11–E13 |
| E15 | PostgreSQL migration and certification | P0 | M3–M4 | E05, E09, E12–E14 |
| E16 | Kubernetes operations and observability | P0 | M2–M4 | E03, E07; later E11–E13 |
| E17 | Correctness, performance and release evidence | P0 | M0–M4 | Incremental across E01–E16 |
| E18 | Multi-region and advanced workloads | P2 | M5 | M4 evidence |

## E01 — Contracts and feasibility

Goal: make scope, invariants, dependencies and performance assumptions reviewable before implementing a broad engine.

**E01-S01 — Compatibility inventory.** As a Falcone maintainer, I want actual database dependencies and SQL usage inventoried so migration scope is based on evidence.

Acceptance: produce a versioned schema/extension/driver/query inventory and supported/blocked classification; include internal platform data and DocumentDB separately; map every baseline requirement to an epic and milestone. No production-readiness conclusion is inferred from documentation alone.

**E01-S02 — Protocol models.** As an engine engineer, I want transaction, shard and object-GC state machines so dangerous transitions can be checked before code expands.

Acceptance: models cover coordinator/leader failure, prepared state, cutover fencing and backup-pin/GC races; state invariants and liveness assumptions explicitly; retain model-check results and counterexamples. Independent review closes known invariant violations.

**E01-S03 — Component qualification.** As a maintainer, I want pinned embedded components with clear interfaces so reuse does not hide service or licensing obligations.

Acceptance: compare RocksDB and candidate storage alternatives, qualify a Raft core and protocol/crypto/GIS crates, record exact versions/licenses/maintenance risks and SBOM; benchmark FFI overhead; decide source license before adding a license file. No latest-branch dependency is accepted implicitly.

**E01-S04 — Benchmark and simulation foundation.** As a reliability engineer, I want reproducible workload/fault tools so every later result can be independently rerun.

Acceptance: define hardware manifests, seeded data/workloads and deterministic clock/network/disk interfaces; distinguish sockets, sessions and requests; save executable configuration and machine-readable results. Demonstrate replay of an intentionally injected failure.

## E02 — Local persistence and MVCC

Goal: a recoverable single-node engine with stable encodings and bounded storage behavior.

**E02-S01 — Key/type encoding.** As an engine developer, I want versioned ordered keys and row encodings so scans and indexes remain correct across upgrades.

Acceptance: golden vectors cover numeric order, UTF-8, NULL, decimals, UUIDs and timestamp boundaries; scope prefixes cannot collide; decode rejects malformed/unknown versions; round-trip and ordering property tests pass.

**E02-S02 — Atomic apply and recovery.** As an operator, I want acknowledged local commits to survive process failure.

Acceptance: atomically persist state changes with applied-log position; replay is idempotent; crash injection at write/fsync/apply boundaries preserves acknowledged data and constraints; document durability configuration and storage assumptions.

**E02-S03 — MVCC and retention.** As a reader/backup worker, I want protected versions so concurrent writes cannot invalidate my allowed snapshot.

Acceptance: version visibility and tombstones are tested; active snapshots/transactions/backup pins prevent premature GC; expired snapshots fail explicitly; retained history and disk-pressure metrics are exposed; long snapshots cannot grow storage without a configured response.

**E02-S04 — Resource-bounded storage.** As a shared-cluster operator, I want controlled compaction/cache use so one tenant cannot exhaust the node.

Acceptance: storage instances/column families and background threads stay within configured bounds as shard count rises; checksums detect injected corruption; compaction pressure triggers backpressure; stress evidence includes noisy-tenant and disk-full cases.

## E03 — Replication and metadata

Goal: replicated data and a catalog that safely survives topology change.

**E03-S01 — Replicated shard log.** As an application owner, I want a shard to survive one voter failure without losing acknowledged writes.

Acceptance: three-voter tests span independent failure domains; minority writes are refused; leader changes preserve committed order; logs are persisted before required acknowledgements; stale leaders cannot serve unauthorized strong reads/writes.

**E03-S02 — Snapshot and membership recovery.** As an operator, I want to replace failed storage members safely.

Acceptance: learners catch up from verified snapshots; membership changes preserve quorum; interrupted snapshot transfer resumes/restarts safely; transaction/lock recovery state is included; old logs are not truncated before recoverability is established.

**E03-S03 — Catalog and routing epochs.** As a query worker, I want versioned metadata so stale routing and schema caches cannot corrupt writes.

Acceptance: bootstrap metadata without an external database; cache invalidation/version checks are tested; stale requests receive retryable epoch errors; catalog quorum loss has documented behavior; restored descriptors cannot reuse an unsafe ownership epoch.

**E03-S04 — Placement and timestamps.** As an operator, I want placement policies and timestamp allocation to survive metadata failover.

Acceptance: replicas honor topology/residency restrictions; monotone allocation survives restart/leader change; batched allocation preserves required ordering; load tests measure allocation contention and metadata availability separately from shard throughput.

## E04 — Serializable transactions

Goal: precise atomicity/isolation and recoverable outcomes across shards.

**E04-S01 — Single-shard serializability.** As a developer, I want transactions to preserve invariants under concurrent reads/writes.

Acceptance: strict key/predicate locking covers existing and absent rows; write-skew/phantom/lost-update histories are rejected or serialize correctly; constraints execute under locks; deadlocks return a documented retryable error; commit/abort releases state safely.

**E04-S02 — Cross-shard atomic commit.** As a developer, I want transfers touching several shards to commit together.

Acceptance: replicate participant prepare state and final transaction decision; fault tests at every phase show no partial visibility or split decision; participant registration is fenced; global index/FK participants are included; one-shard optimization cannot bypass required participants.

**E04-S03 — Recovery and unknown outcomes.** As a client, I want to discover whether a timed-out commit succeeded.

Acceptance: transaction/request IDs resolve outcomes after coordinator failure; a stale coordinator cannot decide again; prepared locks survive/reconstruct on leader loss; unavailable decision quorum causes an explicit wait/error, not guessed abort; retries cannot double-apply a committed request.

**E04-S04 — Safe read paths.** As a reader, I want documented freshness and consistent results during failover.

Acceptance: initial leader reads satisfy the locking protocol; any optimized snapshot/follower mode has a separate safe-timestamp proof and tests for unresolved intents, clock jumps and lag; unsupported freshness modes fail explicitly; no mixed-mode anomaly is accepted.

## E05 — SQL and query execution

Goal: a useful relational engine with an honest compatibility surface.

**E05-S01 — SQL/type semantics.** As a developer, I want a documented SQL subset with reliable values and errors.

Acceptance: parser/binder/type system supports the M1 profile; differential tests cover NULL logic, numeric overflow, decimal exactness, timezones and collations; unsupported syntax/functions produce stable errors; every released type has wire and JSON encoding tests.

**E05-S02 — Relational operations.** As an application developer, I want CRUD, joins and constraints without application-side integrity workarounds.

Acceptance: CRUD/upsert subset, joins, aggregates, ordering, identities and views work inside the transaction layer; SQL and typed API paths share checks; tests include rollback after constraint failure and query cancellation; supported cascade limits are explicit.

**E05-S03 — Distributed optimizer.** As an operator, I want efficient plans and predictable resource limits.

Acceptance: explain reports shard pruning, joins, estimated rows/network and policy filters; colocated joins avoid unnecessary exchange; distributed aggregate/limit semantics match reference results; memory/spill/fan-out caps are enforced and measurable.

**E05-S04 — Driver and bulk profile.** As a Falcone maintainer, I want standard clients and import/export tools to work within the declared subset.

Acceptance: certify selected Node `pg` version for simple/extended/prepared queries, transactions, errors/cancel; add named client profiles as tested; selected COPY/CSV/JSON paths handle large streams with bounded memory; introspection never claims nonexistent features.

## E06 — Sharding and distributed indexes

Goal: native online distribution with relational correctness.

**E06-S01 — Distribution policies.** As a workspace administrator, I want hash/range/colocated layouts that fit my workload.

Acceptance: policy preview explains routing and tradeoffs; large tenants can subdivide across nodes; joins use compatible colocation groups; changing a distribution key is rejected or executed through an explicit movement plan; no hidden one-tenant-one-shard limit.

**E06-S02 — Split and move.** As an operator, I want the cluster to redistribute data while clients continue using logical tables.

Acceptance: snapshot/delta/cutover states survive restart; prepared transactions and predicate ranges cross boundaries safely; stale routers cannot duplicate writes; data/index checksums match before retiring old copies; movement respects bandwidth and capacity limits.

**E06-S03 — Global constraints.** As a developer, I want unique values and foreign keys enforced regardless of physical shard location.

Acceptance: concurrent cross-shard uniqueness/FK races preserve invariants; global index mutations participate in the row transaction; referenced deletes and cascades cannot leave orphans; unsupported constraint configurations are rejected before table publication.

**E06-S04 — Hotspot and balance control.** As an operator, I want placement to respond to actual load without oscillation.

Acceptance: scheduler uses bytes/load/health/locality and respects tenant budgets; hysteresis reduces repeated moves; single-hot-key contention is diagnosed accurately; manual pin/drain and automatic rebalancing are audited; scale-out results include ongoing migration overhead.

## E07 — Connection scale and admission

Goal: serve many clients without confusing connected sessions with executing work.

**E07-S01 — Lightweight gateway sessions.** As an application operator, I want many idle authenticated clients without one backend worker per client.

Acceptance: asynchronous gateway handles the reference session target with measured userspace/kernel/TLS memory; shared shard RPC channels do not grow linearly per client; session cleanup closes all resources; FD/connection-limit errors are observable and safe.

**E07-S02 — Session semantics.** As a driver user, I want prepared statements and transactions to behave correctly under multiplexing.

Acceptance: bounded session caches preserve parameter types and authorization; active transactions pin the required coordinator; unsupported temp-table/advisory-lock/LISTEN behavior is rejected; no session state crosses tenants; gateway failure yields reconnection and recoverable outcomes.

**E07-S03 — Fair admission.** As a tenant, I want my reserved workload to survive another tenant's overload.

Acceptance: enforce per-scope connections, active requests/transactions, queue deadlines and CPU/memory/scan budgets; protect consensus/recovery capacity; slow readers are cancelled within limits; noisy-neighbor tests meet the agreed fairness objective without unbounded queues.

**E07-S04 — Churn and elastic gateways.** As an operator, I want to add/drain gateways under load.

Acceptance: scale-out increases tested connection capacity; drain stops new work and handles active transactions within policy; reconnect/authentication storms use bounded resources and jitter; distributed load generators demonstrate actual simultaneous clients and LB limits.

## E08 — Tenant security

Goal: enforce tenant/workspace isolation in the engine across every access path.

**E08-S01 — Immutable security scope.** As a tenant, I want my data isolated even if an application sends crafted SQL or JSON.

Acceptance: principal/path scope is validated; SQL settings or body fields cannot replace authority; tests cover tables, indexes, catalogs, errors and caches across native protocol/JSON/internal APIs; cross-scope operations require separate audited privilege.

**E08-S02 — RBAC and row policies.** As a workspace administrator, I want roles and row/column policies consistently enforced.

Acceptance: reads, inserts, updates, deletes, bulk paths and policy changes use the same engine checks; forced policies cannot be bypassed by an ordinary table owner; revoked access expires within the specified window; stale privileged policy fails closed.

**E08-S03 — Transport, secrets and encryption.** As an operator, I want credentials and durable data protected through rotation/recovery.

Acceptance: external TLS and internal mTLS reject invalid peers; credential/key references never appear in query logs; selected at-rest encryption model covers logs/spill/backups; rotation and restore with retained keys are tested; bootstrap does not require a running tenant database.

**E08-S04 — Auditing and scoped resources.** As an auditor, I want reconstructable administrative activity without sensitive-value leakage.

Acceptance: audit mutations with actor/scope/resource/decision/request ID; durable replay handles downstream outage; redaction tests cover query/object credentials; scoped export, backup, temporary files and error messages cannot disclose another tenant's data.

## E09 — Administration and schema lifecycle

Goal: complete supported administration through typed APIs without requiring SQL.

**E09-S01 — Versioned API contracts.** As a platform integrator, I want discoverable typed management operations.

Acceptance: OpenAPI/JSON Schema and Protobuf cover the resource table in the API design; generated clients preserve decimals/int64/errors; auth/limits are equivalent across transports; capability discovery reports actual implementation support.

**E09-S02 — Reviewable idempotent plans.** As an administrator, I want to preview and safely apply a schema or placement change.

Acceptance: plan captures content/catalog/policy versions and estimated impact; apply reauthorizes and rejects stale versions; duplicate identical requests return the same operation; key reuse with changed content fails; progress distinguishes accepted from successfully applied.

**E09-S03 — Online indexes and columns.** As an application owner, I want supported schema changes without inconsistent reads during backfill.

Acceptance: schema-state transitions are durable; concurrent writes update new structures correctly; uniqueness validation precedes publication; failure/restart/cancel honors the documented commit point; data is reclaimed only after old readers are safe.

**E09-S04 — Administrative UI/CLI.** As a workspace administrator, I want database/role/index/backup/session controls in Falcone's console and a CLI.

Acceptance: both use published APIs, display limits and async status, and support role restrictions; no privileged SQL entry is required for supported operations; destructive actions show their actual scope; cancellation reports whether work already committed.

## E10 — Geospatial

Goal: native location-aware tables and queries with a precisely defined semantic subset.

**E10-S01 — Spatial types and formats.** As an application developer, I want to store geometry and WGS84 locations alongside relational data.

Acceptance: supported geometry/Multi types and geography points round-trip via GeoJSON/WKT/WKB; SRID/axis order are explicit; invalid/mixed-SRID inputs follow documented errors; SQL and JSON representations agree; unsupported 3D/raster features are not advertised.

**E10-S02 — Exact predicates and distance.** As a developer, I want spatial answers with known units and edge-case behavior.

Acceptance: declared intersects/contains/within/distance subset matches approved oracle fixtures; geometry uses CRS units and geography uses meters; tests cover holes, boundaries, empty values, poles and antimeridian; precision tolerances are published per operation.

**E10-S03 — Persistent spatial index.** As an operator, I want spatial filters accelerated without changing answers.

Acceptance: MVCC index updates commit atomically with rows; conservative coverings have no false negatives in generated fixtures; exact recheck removes candidates; oversized geometries use bounded fallback; crashes/backfill/split/move preserve index equivalence to full scans.

**E10-S04 — Distributed spatial queries.** As a tenant, I want radius/location queries to work across my shards with predictable cost.

Acceptance: planner shows spatial filtering/pruning and fan-out; all result rows retain scope enforcement; pagination and boundary cases are correct; latency/scan metrics compare indexed and full-scan workloads; query budgets stop expensive unpruned scans safely.

## E11 — Managed S3 large objects

Goal: large files are a first-class database capability with correct publication and retention.

**E11-S01 — Provider bindings and upload.** As a developer, I want to upload large files directly to qualified object storage.

Acceptance: binding validates required provider capabilities and secret references; multipart uploads resume/reconcile by ID; memory stays bounded with large files; quotas reserve uploads; checksum verification does not treat multipart ETag as a content MD5.

**E11-S02 — Atomic reference publication.** As a developer, I want a row to reference only a verified immutable object.

Acceptance: unsealed/cross-scope objects cannot attach; row and reference records commit atomically; aborts leave reclaimable unreferenced objects; duplicate seal/attach requests are idempotent; object replacement creates a new version; unavailable S3 never becomes silent NULL.

**E11-S03 — Retention and garbage collection.** As an operator, I want unneeded bytes reclaimed without breaking reads or restores.

Acceptance: active snapshots, PITR, backup and retained-change references pin required objects; GC claims serialize against attach/pin; fault tests cover delete/retry/restore races; external lifecycle settings incompatible with retention are rejected or explicitly unsupported.

**E11-S04 — Authorized download and accounting.** As a tenant, I want controlled file access and understandable usage.

Acceptance: downloads require current row/object authorization; signed URLs expire with documented revocation limits; range downloads work; report logical/physical/upload/retained bytes and egress separately; secrets/URLs are redacted; S3 outage metrics distinguish byte availability from row health.

## E12 — Backup and recovery

Goal: a backup is proven restorable, including distributed transactions and external objects.

**E12-S01 — Consistent snapshot manifests.** As an operator, I want a backup representing one recoverable database state.

Acceptance: capture a protected timestamp, catalog/transaction metadata, shard boundaries and object identities; concurrent split/move does not omit or duplicate logical data; publication occurs only after manifest/data verification; failed attempts release pins safely.

**E12-S02 — Archived history and PITR.** As an administrator, I want to restore to a selected supported time.

Acceptance: archive all required history beyond a base snapshot; report a common recoverable watermark across shards and objects; gaps stop restore with explicit errors; retention accounts for unfinished transactions and schema transitions; achieved archive lag is measured.

**E12-S03 — Verified restore.** As a tenant, I want my restored database and files to be usable and isolated.

Acceptance: restore into a clean namespace/cluster; validate checksums, constraints, policies and object bytes; dependencies on encryption keys are preflighted; backup credentials cannot expose unrelated scopes; readiness is withheld until verification completes.

**E12-S04 — Disaster recovery drills.** As an operator, I want known recovery limits for node/region and logical data loss.

Acceptance: rehearse lost-cluster recovery with separate backup/object copies where required; publish measured RPO/RTO and data size; simulate missing objects/keys/history; restore failure leaves a clear resumable or terminal operation state and never a false success.

## E13 — CDC and realtime

Goal: expose committed resumable changes for Falcone events and subscriptions.

**E13-S01 — Committed change envelope.** As an event consumer, I want committed changes with transaction and schema context.

Acceptance: stream excludes aborted intents; events include scope, table/schema version, transaction ID, key and stable event ID; cross-shard transactions expose completion boundaries; before/after images obey policy and declared availability.

**E13-S02 — Resume and ordering.** As a consumer, I want to restart without silently missing changes.

Acceptance: opaque checkpoints survive shard splits/moves; per-key order is preserved; cross-shard global ordering is not claimed unless implemented; at-least-once duplicates are identifiable; expired checkpoints require an explicit resnapshot instead of silent truncation.

**E13-S03 — Snapshot-to-stream handoff.** As a realtime client, I want initial rows followed by changes without a gap.

Acceptance: snapshot and stream share a documented watermark; concurrent transaction tests show no lost change; deduplication handles overlap; subscription authorization is rechecked on policy change; slow consumers encounter bounded buffering and clear recovery instructions.

**E13-S04 — Falcone event adapters.** As a platform integrator, I want Kafka/realtime delivery without affecting database commit correctness.

Acceptance: adapter preserves scope, event IDs and transaction markers; downstream outage spools/retries within retention policy; backlog is visible; no end-to-end exactly-once claim is made without a tested sink protocol; synchronous database commits do not depend on Kafka availability.

## E14 — Falcone integration

Goal: make the new engine an explicitly selectable database backend under existing governance.

**E14-S01 — Backend-neutral database adapter.** As a Falcone maintainer, I want both PostgreSQL and Falcone RDBMS supported during transition.

Acceptance: typed provision/admin/query/change/backup contracts have capability negotiation; the new adapter uses gRPC with authenticated context; legacy routes retain documented semantics; unsupported features produce actionable errors rather than false emulation.

**E14-S02 — Provisioning and lifecycle.** As a tenant administrator, I want create/suspend/delete operations to reflect real engine state.

Acceptance: provisioning resumes after partial failures; “active” requires an actual scoped read/write check; suspension covers new and existing sessions according to policy; deletion coordinates retention/backup/object rules; operation audit correlates platform and database IDs.

**E14-S03 — Identity, secrets and storage providers.** As a platform operator, I want first-party services to integrate without circular availability requirements.

Acceptance: IAM/Secrets/Storage contracts are versioned; standalone bootstrap works; credential/policy rotation and provider outages have bounded behavior; database object GC coordinates with storage retention; no tenant database is required to start its own provisioning control path.

**E14-S04 — Plans, console and telemetry.** As an administrator, I want permissions, quotas and usage visible consistently.

Acceptance: console hides unavailable capabilities based on actual backend discovery; quotas cover connection/query/storage/blob/backup dimensions; usage avoids duplicate physical/logical charging; traces connect platform request to query/transaction/shard; scoped operator roles are tested end to end.

## E15 — Migration and compatibility certification

Goal: migrate only workloads whose behavior and rollback boundaries have been demonstrated.

**E15-S01 — Migration assessor.** As a workspace owner, I want to know blockers before moving data.

Acceptance: inventory SQL/types/extensions/policies/drivers and dataset characteristics; produce supported/transformable/blocked findings and a schema plan; pgvector/PL/pgSQL/DocumentDB dependencies are explicit; no blocked workspace can be auto-approved for cutover.

**E15-S02 — Snapshot plus PostgreSQL CDC.** As an operator, I want a repeatable import without lost source writes.

Acceptance: snapshot/LSN handoff is gap-free; preserve source transaction boundaries; deduplicate replay; coordinate DDL and sequence values explicitly; handle replica identity and large objects; bound source WAL retention and stop safely when limits are exceeded.

**E15-S03 — Semantic reconciliation.** As an application owner, I want proof that my supported workload still works.

Acceptance: compare counts/checksums at matched positions; verify constraints/policies/objects and query fixtures; shadow reads are side-effect-free and scoped; driver/ORM versions are named; unsupported semantics block promotion rather than being silently normalized away.

**E15-S04 — Fenced cutover and recovery.** As an operator, I want one authoritative writer and a truthful rollback plan.

Acceptance: fence/drain source writes, catch up to final LSN, verify and atomically change binding generation; old connections cannot keep writing; before-target-write rollback is rehearsed; after-target-write rollback requires validated reverse replication or explicit reconciliation/write freeze.

## E16 — Operations and observability

Goal: deploy, diagnose and upgrade the engine safely on Kubernetes.

**E16-S01 — Kubernetes deployment.** As an operator, I want reproducible dev and three-domain cluster profiles.

Acceptance: Helm/operator handles persistent identities/volumes, topology, secrets and disruption budgets; readiness reflects quorum/dependencies; stateless scale-out is independent; stateful scale-in drains/moves replicas safely; namespace deletion cannot silently bypass retention policy.

**E16-S02 — Metrics and diagnosis.** As an SRE, I want to identify connection, transaction, shard and object bottlenecks.

Acceptance: publish the architecture's metrics and trace fields; sampled query fingerprints omit secrets/values; dashboards distinguish availability/durability/archive freshness; labels stay bounded; alerts identify actionable conditions with runbook links.

**E16-S03 — Upgrade and downgrade boundaries.** As an operator, I want version changes without accidental unreadable data.

Acceptance: mixed-version matrix and feature negotiation are tested; rolling sequence preserves quorum; golden disk/wire fixtures remain readable as promised; irreversible format changes require explicit preflight and restore path; downgrade refusal explains the actual boundary.

**E16-S04 — Runbooks and repair.** As an on-call engineer, I want tested procedures for failures.

Acceptance: rehearse replica replacement, lost quorum, corrupt disk, unknown commits, disk full, S3 outage and backup recovery; repair cannot promote stale unverified data; diagnostic bundles redact tenant data; exercises record achieved recovery times.

## E17 — Release evidence

Goal: release decisions depend on reproducible evidence instead of feature checklists.

**E17-S01 — Deterministic fault campaign.** As a reliability engineer, I want reproducible failures throughout development.

Acceptance: saved seeds exercise crashes, partitions, clock steps and disk faults through E02–E13; histories/checksums expose invariant violations; every fixed fault gains a regression; real disk/FFI failure tests complement simulation rather than being assumed covered.

**E17-S02 — SQL and security differential suite.** As a reviewer, I want independent checks of semantics and isolation.

Acceptance: common SQL behavior is compared with PostgreSQL; geo uses approved oracle fixtures; histories check promised transaction properties; cross-tenant attacks span every data/management path; all expected deviations are documented and no unexplained failure is waived.

**E17-S03 — Capacity and fairness evidence.** As a product owner, I want defensible scalability claims.

Acceptance: publish hardware/data/TLS/RF/durability settings with full latency/error/resource results; run the target connection/churn/skew/scale-out matrix; separate offered from completed load; revise unmet goals transparently; results are reproducible from checked-in harness versions.

**E17-S04 — Production-candidate gate.** As a release owner, I want clear acceptance criteria before supporting durable customer workloads.

Acceptance: complete independent protocol/security review, seven-day mixed-load soak, restore and rolling-upgrade drills; no open critical correctness/security defects; support matrix/SLO/runbooks/versioned artifacts are published; candidate approval cites evidence and remaining known limits.

## E18 — Expansion after the core release

Goal: add advanced capabilities without diluting the initial correctness contract. All four stories are P2 and require separate detailed design before implementation.

**E18-S01 — Multi-region policies.** As a global tenant, I want locality and residency controls with explicit consistency/latency choices.

Acceptance: define synchronous versus async replication profiles, leader placement, safe regional failover and achieved RPO; quantify WAN write latency; verify residency for replicas/backups/objects; no simultaneous partition-tolerant local-write promise is made for strong transactions.

**E18-S02 — S3 analytical tier and branches.** As a developer/analyst, I want inexpensive historical scans and isolated test copies.

Acceptance: separately specify Parquet manifests/snapshot visibility, schema evolution, retention and branch divergence; qualify DataFusion/Arrow integration; analytical CPU/I/O cannot starve OLTP; branch/object GC is tested; no arbitrary S3 file is treated as a transactional table.

**E18-S03 — Search and advanced SQL.** As a Falcone developer, I want richer search and query features when core semantics are stable.

Acceptance: prioritize exact-vector/ANN/full-text and broader SQL by actual migration demand; define recall/freshness and policy-filter behavior; measure distributed top-k correctness; add functions/triggers/UDFs only with a resource and security model; preserve honest PostgreSQL compatibility claims.

**E18-S04 — Advanced GIS.** As a GIS user, I want more coordinate systems and complex operations.

Acceptance: qualify projection libraries and grid-data licensing/versioning; add tested geography polygons, overlay/buffer/KNN/spatial joins by explicit subset; compare edge-case results and index plans; raster/topology/3D require separate scope and are not implied by a `geometry` column.

## Suggested first implementation slice

Start with E01-S01 through E01-S04 and E17's harness foundations. Then implement a small vertical path: scoped authenticated connection → create a table through JSON → insert/read a row → crash/restart → verify the committed row and isolation. Add replicated storage and a two-shard bank transfer before broadening SQL/GIS features.

Issue creation can preserve these IDs in titles, with one parent epic and four child stories each. Keep the acceptance text and dependencies when converting to project issues; do not mark an item complete solely because a stub API or plan builder exists.
