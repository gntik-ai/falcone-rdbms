# Relational objects, planning and data-operation contracts

Status: proposed architecture extension, 4 November 2026. Derived from the [fourth-book review](database-system-concepts-review.md); supplements the [distributed core](distributed-contracts.md) and [API contracts](api-and-data-model.md). Rust ownership of the storage engine and disk format remains mandatory. All features below have explicit capability gates; none is currently implemented.

## Component ownership and dependency direction

| Existing component | Added responsibility | Durable state |
| --- | --- | --- |
| Catalog/meta | Object dependency graph, sequence descriptors, statistics descriptors, schema/policy dependencies | Replicated scoped descriptors; large statistics samples do not belong in one hot metadata row |
| Query/planner | View expansion, physical properties, bounded plan cache, statistics consumption, query fingerprints | Cached plans are disposable; source descriptors and semantic versions are authoritative |
| Store/transactions | Sequence reservation records, validated mutations, lock hierarchy, import batch decisions, installed native index generations | First-party WAL/manifest and replicated transaction authority |
| Admin/worker | Analyze, import/export, bulk index construction and later materialized refresh jobs | Versioned job state, progress, protected cuts and private/public generations |
| CDC/backup | Complete change handoff and retention for jobs/derived state | Existing transaction/topology envelopes and shared pin accounting |

Do not create a new always-running service per feature or tenant. Sequence authorities and durable job progress may use system data shards; the logical catalog stores descriptors and routing, not every per-row allocation or progress update. E24/E25/E26 are implementation owners beneath existing integration epics. E27 consumes those interfaces only after core release evidence.

## Catalog identities, views and constraints

The catalog identifies objects by immutable scoped IDs and versioned descriptors. A name lookup produces an object ID/version; cached definitions never rebind by name alone. Persist dependency kinds for referenced columns/types, constraints, indexes, views, sequences, policies and later materialized definitions. Use bounded traversal and prevent cycles where unsupported.

DDL plans include the affected object IDs, expected descriptor versions, invalidated plans and the authorized dependency closure. `RESTRICT` is the default for drop operations. Explicit `CASCADE` must enumerate the exact affected set in the preview; application revalidates it against concurrent changes. The catalog cannot expose hidden object names through dependency errors. Dropped physical data remains protected by reader, backup and job pins.

Logical views store a bound definition and explicit output schema, including the columns selected by `*` at definition time. The initial profile is **read-only, invoker-authorized views**. The caller needs the view privilege and underlying object rights, and base-table RLS applies. Nested views obey the same rules. Definer execution, updatable views and `WITH CHECK OPTION` are unsupported until separately certified; DDL rejects them rather than accepting ineffective options. These choices are intentionally narrower than many PostgreSQL workloads, which the migration inventory must identify.

Named constraint descriptors define expression/type dependencies, timing, validation state and error identity. Initial immediate semantics:

- `NOT NULL` rejects SQL NULL. A row-local `CHECK` rejects FALSE; TRUE or UNKNOWN passes. Cross-table CHECK subqueries/assertions are unsupported.
- Primary keys reject NULLs. Default UNIQUE permits multiple NULL-containing keys under the declared NULL-distinct policy; a NULLS-NOT-DISTINCT option is not implicitly supported. Non-NULL conflicts still use transactional local/global indexes.
- Initial composite foreign keys use an explicitly declared MATCH SIMPLE profile: a NULL in any referencing column skips the reference check. Other match/timing modes must fail at DDL validation until certified.
- Immediate constraint checks have a documented statement boundary; use the same rule for single-row and multi-row mutations. `DEFERRABLE`/`SET CONSTRAINTS` are rejected initially. A violation puts the transaction into the declared failed state; it cannot leave partial committed effects.

Adding a constraint to existing data is a DDL job. Protect new writes before scanning old data, use a consistent cut, reconcile concurrent changes, and publish VALIDATED only after verification. A pending/failed constraint is observable and cannot be used by the optimizer to eliminate joins or predicates. Existing FK/unique correctness is never temporarily disabled to accelerate import.

## Identity and sequence allocation

Separate a sequence descriptor from its allocation authority. The descriptor specifies owner/permissions, integer domain, increment, bounds, cache policy and allocation generation. The initial optimized profile can restrict increments and cycling, but capability discovery must report the exact supported intersection. The simplest baseline is positive increments, no cycle, no backwards restart and explicit overflow errors.

The replicated authority durably reserves disjoint ranges before a holder returns any value from them. A crash may abandon the unused suffix. Allocation holders carry the sequence and cluster generations; sequence alteration, forward restart and restore fence incompatible cached ranges. Changing a generation in metadata alone is insufficient: alteration drains all affected holders or proves fencing on every serving path, including standalone next-value calls. An unreachable holder blocks the change unless that fencing is established; no timeout assumes it stopped. The initial immutable-increment/no-backwards-restart profile avoids unnecessary reset races. A restored allocation authority has a new cluster incarnation and a verified high-water value. Imported explicit IDs do not automatically tell the allocator which values were reserved elsewhere; migration reconciles source sequence state and target bounds.

Sequence allocation is intentionally outside rollback of the consuming row transaction. Gaps are valid, global call/commit order is not promised, and cached ranges may return numerically lower values after another holder has returned higher ones. `GENERATED ALWAYS` versus `BY DEFAULT` controls explicit-value insertion; a table's UNIQUE/PK constraint remains the final collision check. Explicit inserts that collide with future sequence values are reported as constraint errors unless an administrator has safely advanced the sequence. No hidden per-insert table scan is permitted.

Store values chosen for defaults/identities and other nondeterministic expressions in the transaction's deterministic mutation command. Followers replay those results rather than calling a local clock/RNG or reevaluating SQL. An idempotent retry of a committed insert returns the stored outcome/identity. An unknown outcome must be resolved before treating a newly allocated value as a successful replacement.

Avoid one cluster-wide sequence and record reservation latency/cache-waste metrics. Applications needing gap-free business numbering use an explicit transactional counter with its serialization cost; it is not the identity allocation contract. Session-local last-value access requires authenticated session state that survives only the documented session lifetime and never leaks between multiplexed clients.

## Lock hierarchy and deterministic execution

The existing strong strict 2PL baseline requires a conflict model between coarse scans, key/range locks and DDL. Define intention modes and the exact ancestor/descendant coverage within logical lock domains. A table spanning shards requires coordinated protection across its participants; a coarse lock on one shard does not lock the entire logical table.

Do not turn the textbook root into a central metadata RPC for every row. Use local shard authority, stable descriptor/lease dependencies and a modeled protocol for table-wide operations. Acquire covering protection before replacing fine locks. Escalation may wait or cause a safe pre-prepare abort; it cannot drop conflicting protection to save memory. Recover prepared protection before conflicting access after failover. Lock upgrades and escalation participate in deadlock detection/prevention and resource budgets.

Deterministic commands include schema/codec/comparator versions, resolved volatile/default values and the exact ordered logical mutations or a fully specified equivalent. Replay tests vary process clocks, random seeds, scheduling and supported binary versions. Hardware floating-point behavior and unordered hash iteration cannot change replicated logical effects. The selected consensus core's precise commit/read rules still apply; a textbook majority sketch is not the full protocol.

## Statistics lifecycle

`ANALYZE` and typed analyze jobs produce versioned descriptors containing sample provenance, row/byte/null/distinct estimates, heavy hitters, histograms, schema/topology coverage, age and confidence/limitations. Collect a weighted sample across the requested logical relation, or merge a specified estimator that has a justified merge rule. Distinct counts are not summed; histograms from differently sized ranges are not averaged blindly. Splits/moves must not double-count parent and child samples.

Statistics can be approximate and stale. Record freshness and coverage rather than pretending an approximate count is transactionally exact. Empty/missing statistics select a bounded fallback. Estimates cannot prove a predicate false or establish uniqueness; only validated semantic constraints may justify such rewrites. Correlation between columns must be represented explicitly when supported, otherwise cost models disclose the independence approximation.

Refresh is an admitted background job triggered manually, after qualified bulk work, or by bounded change/estimate-error thresholds. Use cooldowns, deduplication and cancellation. Pin/release only the necessary read context and never retain history indefinitely. Scope-authorized operators may inspect full statistics; ordinary tenant diagnostics receive redacted summaries. A frequent-value histogram is potentially sensitive data, even if it is called metadata.

## Physical planning and cache validity

Plans track required/provided ordering, distribution keys and equality/hash semantics, colocation, execution site, placement generation, memory and replay properties. Explicit exchanges perform hash/range redistribution, bounded broadcast, gather or ordered merge. Query-network transport does not change persisted shard ownership.

| Plan concern | Required behavior |
| --- | --- |
| Colocation | Avoid exchange only when current descriptors prove compatible placement and key semantics |
| Broadcast | Reserve total build-side memory across receivers and account for network amplification; reject or use a supported bounded fallback if the estimate is exceeded |
| Outer joins | Assign unmatched-row decisions correctly; a local miss is insufficient when matching rows may exist on another worker |
| Aggregation | Preserve bag/NULL semantics and SQL empty-group behavior; AVG combines sum/count, not averages of arbitrary partitions |
| Ordering/limit | Merge under declared collation/null ordering; arbitrary arrival order is not a global ORDER BY |
| Parallelism | Choose a bounded worker/partition count from workload, grants and available resources; more shards do not imply one worker per shard |
| Mutations | Stabilize affected row identities or prove scan safety when updates change scanned index keys; no row revisits from the update's own writes |
| Explain | Show distribution, exchanges, degree of parallelism, estimates, grants and actual rows/bytes/spill with redacted values |

Keep semantic plan validity separate from plan cost quality. Validity includes scope, effective principal/role/policy version, parameter types, object IDs/schema versions and semantics such as collation. Physical plans also validate relevant placement assumptions. A changed statistics estimate can affect quality without changing meaning; policy revocation changes validity even if cost is unchanged.

Bound cache bytes, number of parameter variants and compile work. Test rare versus common values, highly unequal tenants and invalidation storms. Do not share a bound privileged plan across tenants solely because SQL text matches. Feedback uses query/plan fingerprints and estimate error with bounded cardinality; any operator override still obeys authorization and semantics.

### Failure and adaptation boundary

The initial execution protocol aborts/cancels a failed query attempt and its sibling workers, then safely resolves transaction state. Whole-attempt transparent retry is allowed only when declared safe, no effects/results have escaped, the necessary read/transaction context remains valid and an end-to-end budget permits it. A partially streamed query returns an explicit interruption; silently restarting can duplicate delivered rows. Writes with unknown decisions use outcome recovery. Read/write locks and transaction attempts cannot be independently recreated by a restarted fragment.

Fragment-level checkpoints, speculative tasks and adaptive restarts are later capabilities. They require attempt fencing, input/read-context identity, output deduplication and ownership of any side effects. Materializing intermediate results also consumes disk, retention and latency; replication of the underlying table does not make disposable query state durable.

P2 runtime filters initially target qualified inner/semi joins. A complete build-side Bloom filter may reject impossible matches only under identical equality/hash semantics and the same logical read context. Incomplete/missing/expired filters pass through; false positives receive exact recheck. Restrict preserved-side outer joins, NULL-aware anti joins and volatile expressions until proven. Build/distribution overhead and false-positive rate are measured, not assumed beneficial.

## Bulk import, index build and export

### Two distinct ingestion contracts

| Mode | Visibility and failure | Intended use |
| --- | --- | --- |
| Transaction-scoped COPY | Same atomicity and failure state as its user transaction; transaction-size/age limits apply | Bounded atomic imports through the certified client protocol |
| Durable import job | Each validated batch commits atomically; previous batches remain visible after a later failure/cancel | Large resumable loads through explicit administration APIs |

Publish codecs for CSV and JSON Lines: encoding, delimiters, escaping, column order, NULL/missing/default values, exact numerics/times, maximum record/nesting size and error policy. A transport chunk is not a record; quoted CSV may contain newlines. Compressed inputs require deterministic replay or a qualified seek index before resumability is claimed. Native PostgreSQL binary COPY is supported only if independently certified; CSV/JSON support does not imply it.

Import jobs bind scope, target versions, immutable source checksum/identity, parser/options version, job generation and record/batch progress. Object sources use authorized storage bindings and short-lived credentials. Arbitrary caller URLs and unscoped server filesystem paths are not import permissions.

Commit each batch's effects and progress through one atomic decision, with a job/batch dedup key and payload hash. If progress lives on another system shard, it becomes a transaction participant or uses an equivalently proven decision binding. A response timeout does not authorize replay before resolving the batch. Retries cannot pair new source bytes with an old checkpoint. Rejected-row storage is bounded and protected; rejection policy and committed counts are explicit. Cancellation stops new batches and resolves in-flight decisions; it is not a rollback of previously committed batches.

All paths enforce the same RLS, constraints, identity, object-reference and index rules as ordinary writes. Pre-sort or batching optimizations do not turn off WAL, replication or global uniqueness. After load, publish verified completion counts and schedule bounded statistics refresh. Bulk import into an existing table is not an invisible staging-table swap or a whole-file transaction.

### Native bulk index construction

Generate first-party sorted runs under the index comparator/schema version, merge with bounded fan-in and verify checksums/uniqueness. Reserve temporary/live/replicated bytes and keep source snapshot plus concurrent changes protected. Use the existing `VALIDATING → WRITE_ONLY → BACKFILLING → VALIDATING_DATA → PUBLIC` DDL lifecycle rather than inventing a competing publisher.

Private files become public only after the manifest and replication installation protocol prove durable recoverability and catch up the admitted write frontier. Exactly how file installation is represented in Raft and native WAL is an M0/M2 design obligation; importing an external SST file or disabling logging is not an implementation shortcut. Crash tests cover installation, publication and cleanup, including unique keys that span sorted runs and shards.

### Logical export manifests

Export from one protected authorized consistent cut. Bind schema/codec/policy/topology versions, logical key-range coverage, chunk IDs, checksums and counts. Snapshot lineage prevents duplication/omission during splits. Resume uses the same cut; an expired pin requires failure/restart, never a fresh shard snapshot spliced into old output.

Initial resumable exports may use a qualified base-table/key-ordered scope; arbitrary query resumption requires a separate stable ordering/read-context contract. Authorization is checked against the declared revocation window during long jobs. Generated files and manifests are protected by quota, encryption and lifecycle policy.

Declare whether objects are exported as references only or as a verified object-complete package. Importing references into another scope requires an authorized object mapping; a provider URL is not a durable object identity. Logical export verifies portability and data equality; physical backup/PITR additionally carries native recovery history and remains E12's responsibility.

## Materialized views — P2, M5

The first materialized implementation is a managed derived relation with explicit full refresh. A job reads a protected consistent cut, builds a private native generation, validates it and atomically publishes a manifest. A reader pins exactly one generation. An interrupted refresh preserves the previous generation with truthful freshness/error status. A dropped view still retains bytes needed by active readers and backups.

Then qualify incremental maintenance for deterministic single-table selection/projection and grouped COUNT/SUM/AVG. Retain sufficient row/non-NULL counts and sums to distinguish empty/all-NULL/zero groups. Preserve bag multiplicities and SQL's global-empty aggregate behavior. Apply updates as old/new effects under complete source transaction boundaries. Persist changes and consumed progress atomically, including topology lineage. Bootstrap uses the existing gap-free snapshot-to-CDC contract. Expired change history triggers rebuild; it never skips missing transactions.

Incremental joins, DISTINCT aggregates, MIN/MAX deletions, recursive views, volatile functions and external sources are outside the first incremental subset. Full refresh is the explicit fallback where supported. No synchronous maintenance is added to every base transaction by default, and no unrestricted user trigger/UDF engine is required.

A published generation binds data frontier, source coverage, schema and policy versions, refresh mode and generation ID. The frontier is a consistent cut under the existing protocol, not a minimum/maximum of unrelated timestamps. Explicit reads may request a freshness bound; wait or return a clear error when it cannot be met. Asynchronous views cannot enforce base-table constraints or silently replace fresh serializable reads.

Materialized aggregates can reveal information that cannot be removed by filtering the final aggregate row. Initially require a common versioned authorization domain with no principal-dependent RLS, unless an equivalent enforcement design is proven. Otherwise reject creation. Tenant separation alone is insufficient. Role/policy changes fence affected generations and plans; readers must not continue using an invalid authorization snapshot past the stated window.

Automatic optimizer rewrites are disabled initially. A later rewrite must prove equivalence of expression, bags/NULLs, security, schema and snapshot/freshness. Explain identifies its chosen generation/frontier. Recommendations to create/drop views are reviewable plans based on total read, write, storage, refresh and retention costs. Background maintenance stays subordinate to protected OLTP/consensus/recovery budgets.

## Complex data and time boundaries

SQL JSON is a typed value with separate SQL NULL, JSON null and missing-path behavior. The certified profile must specify duplicate object keys, numeric range/precision, equality/order, path casts, limits and wire encoding. Version its native encoding; do not claim PostgreSQL jsonb or general SQL/JSON compatibility from REST/JSON administration. Query operators and indexes on JSON paths require named capability tests.

MVCC versions, transaction-consistent snapshots, PITR, branches, system-versioned SQL tables and business-valid intervals solve different problems. The baseline exposes none of the latter temporal-table semantics merely by retaining old versions. A later temporal capability must specify interval endpoints/overlap constraints, temporal keys/FKs, historical schema/policy and retention. Normalization and entity relationships inform schema design; colocation is a physical optimization, not permission to denormalize authoritative data automatically.

## Qualification scenarios

| Scenario | Pass condition | Owner |
| --- | --- | --- |
| Rename/drop and recreate a table while views/plans exist | Same-ID safe evolution or explicit invalidation; no accidental rebinding | E24-S01/S02, E25-S03 |
| Revoke a role while a cached/nested view executes | Access follows the published revocation window; no cached privilege bypass | E08, E24-S02 |
| Sequence holder crashes or survives a reset/restore | No reuse from stale reservations; gaps/order behavior documented | E24-S03 |
| Add a constraint while conflicting writes and splits run | No VALIDATED publication with violating data or missed changes | E24-S04, E09 |
| Analyze unequal shards during a split | Declared weighted coverage or explicit retry; no parent/child double count | E25-S01 |
| Outer-join row matches only on another worker | Exactly the correct unmatched/matched multiplicities | E25-S02, E17 |
| Change indexed keys while scanning an UPDATE | Each intended row modified as specified, with no update-induced revisit | E05-S02 |
| Lose a worker after streaming results | Explicit failed stream; no invisible duplicate delivery from replay | E04-S03, E05 |
| Crash between row batch commit and import acknowledgement | Outcome/progress recovery determines exactly which batch committed | E26-S02 |
| Disk full during index merge/publication | Old public state survives; staging cleanup respects pins | E26-S03 |
| Resume export after split or pin expiry | Same complete cut, or explicit expiry; never mixed snapshots | E26-S04 |
| Delete last/non-NULL aggregate row and replay a CDC batch | Incremental result matches full recomputation at the same frontier | E27-S02 |
| Change RLS while refreshing a materialized aggregate | Invalid generation cannot expose formerly authorized source data | E27-S03 |

The roadmap adds these tests at their feature gates. They do not replace existing crash, power-loss, serializability, partition, sharding, geospatial or S3 tests.
