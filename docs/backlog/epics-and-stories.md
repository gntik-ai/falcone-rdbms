# Epics and user stories

Status: proposed backlog. **23 epics, 92 stories.** Storage-engine and disk-format ownership is confirmed; the implementation designs remain proposals. IDs are stable planning identifiers, not GitHub issue numbers. Nothing here is implemented or marked complete. Milestones are defined in [quality and roadmap](../design/quality-and-roadmap.md).

P0 = correctness/foundation release blocker; P1 = required product capability; P2 = expansion. Dependencies refer to completion of the relevant slice, not a ban on early interface design. Every story requires observable acceptance evidence and documentation of limits. Security, recovery and compatibility checks apply where relevant, rather than being postponed to a final test phase.

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
| E09 | SQL-free administration and online DDL | P1 | M1–M3 | E03, E05, E08 |
| E10 | Geospatial data and queries | P1 | M3 | E05, E06, E09 |
| E11 | Native managed S3 objects | P1 | M3 | E04, E08, E09 |
| E12 | Backup, PITR and recovery | P0 | M3 | E03, E04, E11 |
| E13 | Durable CDC and realtime | P1 | M3 | E04, E06, E08 |
| E14 | Falcone integration | P1 | M3 | E07–E09, E11–E13 |
| E15 | PostgreSQL migration and certification | P0 | M3–M4 | E05, E09, E12–E14 |
| E16 | Kubernetes operations and observability | P0 | M2–M4 | E03, E07; later E11–E13 |
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

**E01-S01 — Compatibility inventory.** As a Falcone maintainer, I want actual database dependencies and SQL usage inventoried so migration scope is based on evidence.

Acceptance: produce a versioned schema/extension/driver/query inventory and supported/blocked classification; include internal platform data and DocumentDB separately; map every baseline requirement to an epic and milestone. No production-readiness conclusion is inferred from documentation alone.

**E01-S02 — Protocol models.** As an engine engineer, I want transaction, shard and object-GC state machines so dangerous transitions can be checked before code expands.

Acceptance: models cover coordinator/leader failure, prepared state, cutover fencing and backup-pin/GC races; state invariants and liveness assumptions explicitly; retain model-check results and counterexamples. Independent review closes known invariant violations.

**E01-S03 — Native storage and primitive qualification.** As a maintainer, I want a measured first-party storage design and qualified primitive libraries so Falcone owns persistence without hiding an external engine.

Acceptance: compare first-party LSM/B+tree prototypes under equal durability/workloads and freeze the selected layout in an ADR; external engines may be separate benchmarks only; qualify I/O/checksum/compression/crypto, Raft and protocol/GIS primitives with pinned versions/licenses/SBOM; reject shipping transitive embedded-engine backends; decide source license before adding a license file.

**E01-S04 — Benchmark and simulation foundation.** As a reliability engineer, I want reproducible workload/fault tools so every later result can be independently rerun.

Acceptance: define hardware manifests, seeded data/workloads and deterministic clock/network/disk interfaces; distinguish sockets, sessions and requests; save executable configuration and machine-readable results. Demonstrate replay of an intentionally injected failure.

## E02 — Local persistence and MVCC

Goal: integrate the first-party storage components delivered by E19–E23 into a recoverable single-node row/index/MVCC engine with stable encodings and bounded storage behavior. The detailed WAL/table/manifest/compaction implementations belong to those epics.

**E02-S01 — Key/type encoding.** As an engine developer, I want versioned ordered keys and row encodings so scans and indexes remain correct across upgrades.

Acceptance: golden vectors cover numeric order, UTF-8, NULL, decimals, UUIDs and timestamp boundaries; scope prefixes cannot collide; decode rejects malformed/unknown versions; round-trip and ordering property tests pass.

**E02-S02 — Atomic apply and recovery.** As an operator, I want acknowledged local commits to survive process failure.

Acceptance: atomically persist state changes with applied-log position; replay is idempotent; crash injection at write/fsync/apply boundaries preserves acknowledged data and constraints; document durability configuration and storage assumptions.

**E02-S03 — MVCC and retention.** As a reader/backup worker, I want protected versions so concurrent writes cannot invalidate my allowed snapshot.

Acceptance: version visibility and tombstones are tested; active snapshots/transactions/backup pins prevent premature GC; expired snapshots fail explicitly; retained history and disk-pressure metrics are exposed; long snapshots cannot grow storage without a configured response.

**E02-S04 — Resource-bounded storage.** As a shared-cluster operator, I want controlled compaction/cache use so one tenant cannot exhaust the node.

Acceptance: storage domains, shared memtable/cache budgets and background threads stay within configured bounds as shard count rises; checksums detect injected corruption; native compaction pressure triggers backpressure; stress evidence includes noisy-tenant and disk-full cases.

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

Acceptance: saved seeds exercise crashes, partitions, clock steps and disk faults through E02–E13 and E19–E23; histories/checksums expose invariant violations; every fixed fault gains a regression; real filesystem/device and any FFI failure tests complement simulation rather than being assumed covered.

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

## E19 — Native disk format and manifest

Goal: Falcone specifies, implements and maintains its own durable representation; see [native storage](../design/native-storage.md).

**E19-S01 — Format RFC and fixtures.** As an engine maintainer, I want a precise byte-level specification so files survive compiler, platform and software changes.

Acceptance: freeze chosen header/record encodings, file kinds, endian rules, version/feature bits, bounds and checksum coverage after M0 review; golden binaries and a reference decoder agree; malformed/unsupported fields are rejected; no Rust memory layout or foreign-engine file format is the persistence contract.

**E19-S02 — File identity and allocation.** As a storage engineer, I want stable domain/file identities so cache, restore and maintenance cannot confuse old and new data.

Acceptance: file IDs/generations remain unique across crash/restore; metadata binds comparator/codec/domain and shard range; temp/final naming and directory durability are specified; concurrent file creation cannot reuse identities; stale files cannot be mistaken for live files from another domain.

**E19-S03 — Atomic manifest publication.** As an operator, I want flush/compaction outputs published without missing files after power loss.

Acceptance: durable outputs precede durable manifest and root-pointer publication; fault injection at every write/sync/rename/directory boundary yields a recoverable old or new state; partial manifests never publish; missing/corrupt authoritative metadata fails closed; older roots are not silently chosen to discard acknowledged work.

**E19-S04 — Format ownership and tooling boundary.** As a project owner, I want the shipping engine independent of another database's persistent representation.

Acceptance: dependency checks exclude runtime embedded database/storage engines; first-party format code owns tables/WAL/manifests; reference engines run only in isolated benchmarks; disk-layout documentation and golden fixtures are shipped with a declared version; source provenance/notices are recorded for any reused code.

## E20 — First-party WAL and durable batches

Goal: implement reliable acknowledgement, replay and Raft-storage integration in Falcone code.

**E20-S01 — Framed WAL and atomic apply.** As a storage developer, I want bounded atomic batches with unambiguous replay.

Acceptance: framed/fragmented records carry batch identity/count/sequence/checksums and completeness markers; full batches become visible together; malformed/gapped/reordered records are detected; duplicate batch IDs with changed contents fail; limits reject work before mutation; no partial local batch is replayed.

**E20-S02 — Group commit and rotation.** As an application owner, I want a success response only when the required log bytes are durable.

Acceptance: group-commit waiters track individual durable ends; short writes/sync errors/rotation crashes are tested; new segment directory entries are persisted before reliance; write acknowledgement never precedes required sync; disk full fails safely; memory and queue length stay bounded under slow storage.

**E20-S03 — Native consensus-log adapter.** As a distributed-systems engineer, I want Raft entries and hard state persisted without an external storage engine.

Acceptance: implement first-party consensus log/hard-state persistence respecting the selected Raft core's ordering; engine apply stores mutations and shard applied index atomically; restart replays only committed entries above recovered indexes; fault tests cover mismatched log progress, term changes and snapshot installation.

**E20-S04 — Checkpoint and log retention fences.** As an operator, I want old WAL reclaimed only after equivalent recovery state exists.

Acceptance: engine checkpoints cover complete batches across all domain memtables; partial shard flushes cannot skip unrelated mutations; consensus and engine retention have separate safe positions; backup/CDC pins are honored; injected checkpoint/truncation races preserve all acknowledged state.

## E21 — Sorted tables, reads and cache

Goal: implement Falcone's ordered on-disk data access with bounded resource use.

**E21-S01 — Native table builder and decoder.** As an engine developer, I want immutable tables containing sorted scoped MVCC records.

Acceptance: data/index/property/footer blocks implement the frozen Falcone spec; decoder checks offset arithmetic/lengths/checksums before allocation; ordered scans and point reads match an independent reference model; uncompressed baseline works before adding codec options; cross-domain files are rejected.

**E21-S02 — Memtables and atomic visibility.** As a reader, I want concurrent batch application and flush to return complete consistent engine views.

Acceptance: atomic visibility sequence publishes only complete batches; frozen/active memtable transitions have no lost/duplicate logical entries; ordered iteration merges versions correctly; concurrency tests include cancellation and flush; memtable count/bytes are bounded independently of client/shard count.

**E21-S03 — Index blocks and filters.** As an operator, I want fast lookups without filters suppressing real matches.

Acceptance: index seeks match full scans for generated key/version distributions; Bloom/prefix filters use versioned matching hash/comparator semantics with zero false negatives in fixtures; missing optional filters fall back safely; corruption is detected; prefix-compressed and uncompressed results agree.

**E21-S04 — Block cache and I/O budgets.** As a tenant, I want predictable memory use and read latency during scans and file replacement.

Acceptance: cache keys bind immutable file identity/generation/offset; pins protect live readers from deletion; scan admission avoids uncontrolled eviction; tenant/global memory budgets hold; failed/short I/O yields precise errors; repeated flush/compaction cannot return stale cached blocks.

## E22 — Flush, compaction and history reclamation

Goal: reclaim and organize disk space without breaking snapshots or acknowledged data.

**E22-S01 — Flush/checkpoint pipeline.** As a storage engineer, I want frozen memtables materialized into verifiable files.

Acceptance: all mutations and apply metadata through a published checkpoint are covered; files stay within declared shard boundaries; snapshot/read pins survive transitions; partial output failures retain WAL recovery; manifest installation follows E19 ordering; incomplete temporary outputs are never readable as live tables.

**E22-S02 — Native compaction scheduler.** As an operator, I want maintenance to bound read/space amplification without starving queries or recovery.

Acceptance: first-party selection/merge logic respects input versions and non-conflicting ranges; concurrent jobs validate publication preconditions; CPU/I/O/temp-space budgets and disk reserve are enforced; ingestion backpressure precedes exhaustion; report amplification/backlog/stalls and benchmark p99 during compaction.

**E22-S03 — MVCC and tombstone safety.** As an application owner, I want compaction to preserve every supported historical view.

Acceptance: protected snapshots, unresolved intents and backup/CDC pins constrain version deletion; tombstones cannot be dropped while hidden older values could reappear; generated histories compare before/after compaction; cross-level overlap and delete/reinsert cases are covered; S3 byte GC stays separately coordinated.

**E22-S04 — Obsolete files and orphan recovery.** As an operator, I want reclaimable bytes removed without deleting a live file.

Acceptance: deletion requires a published reachability decision plus reader/snapshot/backup safety; crashes between publish/unlink/directory sync leave recoverable state; aborted jobs' outputs are reclaimed by reachability rather than age; file-count/retained-space metrics identify leaks; low space never triggers deletion of protected data.

## E23 — Recovery, upgrades and storage diagnostics

Goal: make the owned format independently inspectable, recoverable and evolvable.

**E23-S01 — Cold-start recovery and verification.** As an operator, I want native recovery to either reconstruct valid state or clearly refuse unsafe service.

Acceptance: validate authoritative root/manifest/files then replay complete WAL batches; distinguish permitted incomplete tail from corruption; never skip corrupt committed history; reconcile Raft/application fences; recovery fixtures cover missing segments/files, wrong domains and interrupted publication; no third-party engine is needed to open data.

**E23-S02 — Format inspection and safe export.** As an on-call engineer, I want to inspect damaged data without modifying the only copy.

Acceptance: offline tools inspect headers/manifests/WAL, validate checksums and dry-run recovery with redaction; verified restore uses replica/backup evidence; any salvage writes a new output and enumerates missing/corrupt data; tools reject unsupported versions and never report best-effort salvage as full recovery.

**E23-S03 — Format upgrades and downgrade limits.** As an operator, I want format changes with explicit compatibility guarantees.

Acceptance: reader/writer/version matrix and golden fixtures cover old/current files; cluster capability checks gate new writes; conversion/rewrite can restart safely; backups record codec/key/format dependencies; irreversible boundaries block unsafe downgrade with an actionable restore route.

**E23-S04 — Persistence fault qualification.** As a release reviewer, I want independent evidence for power-loss and corruption behavior.

Acceptance: deterministically inject short/reordered writes, failed file/directory syncs, torn records, bit flips, disk full and manifest/compaction races; verify acknowledged batches and no tombstone resurrection; test qualified real filesystems/devices and a reference decoder/model; retain seeds and review results before M1/M4 promotion.

## Suggested first implementation slice

Start with E01-S01 through E01-S04 and E17's harness foundations. Implement E19–E23's M1 slice first: encode a native file → append/fsync an atomic batch → publish a memtable view → flush/compact through a manifest → crash/restart and verify the same state. Then integrate E02 and the small SQL path: scoped authenticated connection → create a table through JSON → insert/read → crash/restart → verify row and isolation. Add native consensus-log integration and a two-shard bank transfer before broadening SQL/GIS features. Preserve existing planning IDs; the new storage epics are prerequisites, not work postponed beyond the RDBMS release.

Issue creation can preserve these IDs in titles, with one parent epic and four child stories each. Keep the acceptance text and dependencies when converting to project issues; do not mark an item complete solely because a stub API or plan builder exists.
