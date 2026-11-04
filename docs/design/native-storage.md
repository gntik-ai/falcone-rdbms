# Native storage engine and disk format

**Confirmed product requirement:** Falcone owns the storage engine and its persistent format. First-party storage code is Rust. RocksDB, LevelDB and other database engines may be reference/benchmark tools but are not production storage dependencies. This is a design RFC, not implemented code or a frozen file-format promise.

## Ownership boundary

| Falcone implements and maintains | Eligible qualified building blocks |
| --- | --- |
| Key/row/MVCC encodings, memtables, table readers/writers and index blocks | Rust collections, allocators and memory utilities |
| WAL records, atomic batches, checkpoints, manifests and recovery | OS file I/O, asynchronous runtime and platform bindings |
| Block cache, filters, flush/compaction policy and version GC | Checksum, compression and hash algorithms |
| Persistent Raft log/snapshot adapter and replay fences | A consensus algorithm library such as the qualified Rust Raft core |
| Format compatibility, verification, inspection and conversion tools | Audited cryptography and key-provider clients |

Using a third-party storage engine through a Rust wrapper, retaining an engine as a production fallback, or merely renaming its files does not satisfy this boundary. Reusing ordinary algorithms/libraries does not require inventing cryptography or an operating system. Record exact dependencies and notices; format ownership does not erase obligations for any source reused.

## Initial organization: first-party LSM

Propose an ordered log-structured merge layout: writes enter a first-party WAL and memtable; flush creates immutable sorted tables; background compaction merges tables while retaining required MVCC history. Immutable outputs fit snapshots and shard movement, but their read/write amplification and maintenance latency must be measured. This is our design inference, not a claim that an LSM is universally superior.

M0 compares a small first-party LSM prototype with a first-party B+tree prototype using the same durability, key encodings, hardware and workload. Cover point reads, range scans, random/sequential updates, deletes, snapshots, disk growth and p99 under maintenance. Choose one primary layout before expanding the engine. If evidence favors B+tree, revise the table/cache/checkpoint sections through an ADR; both engine and format remain Falcone-owned. Do not build two production backends by default.

The [book review](documentation-review.md) retains this experiment: B1 chs. 3–6/12, B2 ch. 4 and B3 chs. 2–7 show why layout choice depends on workload and recovery complexity. Include secondary-index writes, retained MVCC versions, recovery time and total WAL/flush/compaction/replication traffic. Compare a clearly specified B+tree variant; do not treat all buffered or copy-on-write trees as an identical baseline.

Use a bounded number of **storage domains** per process. Each domain has a WAL sequence space, manifest and file set and can host many shard prefixes. All shards share budgeted pools; do not allocate a full WAL writer/cache/memtable per tenant. Table outputs are constrained to a shard/key-range boundary, even when input memory is shared. Small shards may create small files; batching and file-count controls require explicit benchmarks rather than unsafe cross-scope mixing.

Keep three identifiers separate:

| Identifier | Meaning |
| --- | --- |
| Engine sequence / batch ID | Ordering and idempotent application inside a storage domain |
| Raft group / term / log index | Consensus history and ownership for a replicated shard |
| SQL transaction ID / timestamp | Relational visibility, intent resolution and cross-shard commit |

An engine atomic batch can contain uncommitted SQL intents. Persisting that batch does not commit the SQL transaction.

Recovery restores those intents and their transaction identity. SQL recovery determines their outcome before visibility/reclamation; an absent coordinator is not an abort decision. Page-oriented undo/redo examples from the literature inform ordering requirements but do not mandate ARIES or an undo-only algorithm for this LSM.

## Storage API contract

The initial API includes `ApplyBatch`, `Get`, `Scan`, `PinReadView`, `Checkpoint`, `ExportShardSnapshot`, `InstallShardSnapshot`, `Verify`, and durable consensus-log operations. These are internal typed Rust interfaces, not SQL/API endpoints. Errors distinguish invalid input, unsupported format, resource exhaustion, I/O failure and corruption.

`ApplyBatch` carries an idempotent batch identity, bounded encoded size, deterministic put/delete/metadata operations, and any applied Raft index. All operations become engine-visible together at a published sequence; readers cannot observe half a batch. Reject over-limit batches before changing state. Larger SQL transactions use multiple bounded intent batches and the SQL commit protocol; never silently split a claimed atomic batch into separately committed pieces.

`PinReadView` protects a manifest/table version and engine sequence for physical consistency. SQL MVCC timestamps and distributed snapshot protection are additional requirements; an engine read view alone is not a cluster-wide SQL snapshot. Iterators check scope and bounds and release pins on cancellation.

## Disk format v0: proposed contract

Publish a standalone normative format specification and golden binary fixtures before claiming compatibility. No file depends on Rust struct layout, pointer size, host endianness or unversioned serializer defaults. Specify integer widths, length bounds, canonical key ordering, checksum coverage and decoder behavior. Header integer fields are little-endian; ordered key encodings separately use a canonical sort-preserving representation. String/collation versioning belongs in key metadata.

The codec contract also covers null-versus-missing fields, signed-zero/NaN policy, decimal normalization, composite-prefix escaping and logical primary-key locators. Comparator, hash and collation versions are durable semantics: changing them requires a validated rewrite/reindex transition, not only a library upgrade. Mixed-version read-modify-write must preserve supported unknown optional fields or refuse the write. New format features activate only when all required readers/writers/replicas/restore tools support them.

Proposed common file envelope (64 bytes; illustrative until M0 freezes it):

| Byte offset | Bytes | Field |
| --- | --- | --- |
| 0 | 8 | Magic `FLCNSTOR` |
| 8 | 2 | Format major |
| 10 | 2 | Format minor |
| 12 | 2 | File kind |
| 14 | 2 | Header byte length |
| 16 | 8 | Required feature bits |
| 24 | 16 | Unique file ID |
| 40 | 8 | File generation |
| 48 | 8 | Payload offset |
| 56 | 4 | Reserved, zero in v0 |
| 60 | 4 | CRC32C covering bytes 0–59 |

The file-specific metadata binds storage-domain UUID, shard/range identity where applicable, comparator/encoding versions and codec IDs. Readers validate header length/offsets before allocation. Unknown required features, unsupported major versions and invalid checksums fail closed. A checksum detects accidental corruption; it is not authentication. The initial deployment may use qualified volume encryption; future block-level encryption needs explicit AEAD/nonce/key-version contracts using audited primitives.

| File class | Proposed suffix/name | Required contents |
| --- | --- | --- |
| Engine WAL segment | `*.fwal` | Domain/segment identity, previous-segment link, framed atomic batches, sequence bounds, checksums and seal metadata |
| Consensus log segment | `*.frlog` | Raft group/term/index records and durable hard-state updates under a separate protocol |
| Sorted table | `*.fst` | Scoped ordered MVCC entries, data/index/filter/property blocks, validated footer with block handles |
| Immutable manifest | `MANIFEST-<generation>.fmf` | Live files and hashes, ranges/levels, durable sequence coverage, schema/format dependencies, retention references |
| Current-root descriptor | `CURRENT` | Checksummed generation and exact manifest identity/digest; a separately specified small record |
| Shard snapshot manifest | `*.fsnap` | Exact Raft applied index, descriptors, immutable file subset and required transaction/catalog state |
| Temporary output | Private staging names | Never visible until referenced by a durably published manifest |

Suffixes are descriptive; file kind/magic/version and authenticated administrative context control interpretation. Allocate file identities so crashes, restore or rebuild never reuse a cache/encryption identity accidentally. Lock/process-control files are not durable user data. The manifest, root descriptor and file parsers are first-party code.

## WAL and acknowledgement rules

Falcone implements segmented append records with bounded lengths, batch ID, sequence range, operation count and checksums. A fragmented batch has an explicit completeness trailer; replay accepts all operations or none. Fragmentation/record-size limits and cross-segment behavior are part of the versioned spec. Detect record gaps, segment reordering, wrong-domain files and duplicate IDs with conflicting payloads.

For standalone storage, under the documented reliable-fsync filesystem/device contract:

1. Validate/encode the complete batch and reserve bounded resources.
2. Append its framed bytes to the active WAL. Handle short writes and interrupted calls; a successful write call is not a durability acknowledgement.
3. Synchronize through the batch's end position. Group commit can share a flush across several callers; each waiter is released only when its required bytes are durable.
4. Install the complete batch into the memtable and publish its visible sequence atomically. On an unrecoverable post-durability in-memory failure, restart/replay instead of returning a false definitive abort.
5. Return success. If the response is lost, the batch may already be durable and must resolve by identity.

WAL rotation persists the new segment's identity and directory entry before relying on it for acknowledged work. I/O errors fail/stop the writer and preserve evidence; do not continue past an unknown durability boundary. `fsync` disabled is not an acceptable production benchmark mode.

In a replicated shard, the **consensus log** establishes durable replicated commands. The **engine WAL** establishes local deterministic application. Both are Falcone-owned, initially separate streams. Persist required Raft entries/hard state before protocol responses that require durability; apply only committed entries; store row/index/intent mutations and `applied_index` in the same engine batch. Log acceptance does not make an uncommitted Raft command visible.

After restart, replay engine WAL to recover application state, then replay committed consensus entries above each recovered applied index. Do not use a global engine sequence as a shard's Raft index. Durable SQL outcome/prepare requirements from the [transaction design](architecture.md) still determine client acknowledgement. Duplicate log writes are initially acceptable; a unified log needs a separate proof covering consensus, local apply, checkpoint and replay before optimization.

## Tables, filters and cache

Implement deterministic ordered key encoding, point lookups, bounded range iterators and a first-party table builder/reader. Data blocks contain sorted versioned entries with explicit operation/tombstone tags. Index blocks locate key ranges; footer handles include bounds/lengths and checksums. Optional prefix compression and codecs are identified on disk. Establish a simple uncompressed correctness baseline before adding codec complexity.

Bloom filters are an optimization: hash/comparator/prefix versions must match, and a missing/unsupported optional filter falls back to scanning rather than dropping results. Required corrupt structures fail; all lookups recheck keys. Sorted keys include enough version metadata to distinguish committed values, intents and tombstones according to the upper-layer encoding.

The first cache is a bounded read/block cache with quotas and admission/eviction accounting. Keys include domain/file identity, generation, block offset and encoding/codec version. Flush/compaction outputs are immutable and never overwrite a cached file in place. Hold file-version pins during reads; deletion waits for pins. Large scans should not evict the entire point-read working set. Cap memtable count and bytes separately from cache bytes.

Start with buffered I/O on explicitly qualified Linux filesystem/device profiles. Direct I/O or io_uring is a later measured option. The I/O abstraction must support short reads/writes, flush and directory-sync failures, disk full and simulated power loss; changing a kernel API does not establish correctness.

## Flush, publication and checkpoint ordering

A flush converts a frozen memtable range into table files. A checkpoint must represent complete sequence coverage, including all operations of every covered batch and required apply metadata. Partial shard flushes cannot advance a domain's global recovery watermark past still-unflushed mutations.

Publication protocol for a new file set:

1. Write each output into a unique staging file; validate keys/blocks/checksums and synchronize file data.
2. Publish immutable final file names and synchronize their parent directories according to the qualified filesystem contract.
3. Write and synchronize a new immutable manifest naming the exact complete live file set and recovery watermark; synchronize its directory entry.
4. Write/synchronize a temporary root descriptor, atomically replace `CURRENT`, then synchronize its parent directory. Only after that boundary may code rely on the new root surviving a crash.
5. Retire superseded WAL/files only when the new root covers their data and replica/snapshot/backup/CDC/read-view pins permit reclamation. Reclaim orphans after recovery establishes they are unreachable; do not infer liveness from file age.

Before publication, the old manifest and replay logs remain sufficient. After publication, the new manifest references only durable files. A crash must yield a valid recoverable old or new state; it must never produce a reference to an incomplete output. Keep obsolete manifests temporarily for diagnosis, but do not silently choose an older generation when the authoritative manifest is corrupt or missing: that could discard acknowledged data.

Engine-WAL retention is based on complete local checkpoint coverage plus external pins, not on the largest sequence seen in any table. Consensus-log retention additionally depends on the Raft snapshot/replication protocol. Backup and change-stream archives have separately proven coverage. Recycling one log never implies the others are safe to delete.

## Compaction and version reclamation

Begin with a bounded leveled-compaction design and explicit L0 overlap/pressure limits; exact levels/sizes are benchmark decisions. Compaction selects an input version, writes new immutable outputs and publishes them through the same manifest protocol. Concurrent jobs must reserve non-conflicting ranges or validate their input-version preconditions before installation.

MVCC GC needs the oldest protected SQL snapshot and transaction/backup/CDC requirements, not just file timestamps. Preserve the newest required version visible at each relevant horizon, unresolved intents and required tombstones. Drop a tombstone only when no protected reader needs it and no older live value can reappear from any overlapping lower-level file. SQL row deletion, object-reference deletion and S3 byte reclamation remain separate coordinated lifecycles.

Bound compaction CPU, read/write bandwidth and output reservations; reserve free space for WAL/recovery. Throttle ingestion before disk exhaustion. Do not auto-delete live history to satisfy a quota. Metrics include write/read/space amplification, L0 backlog, pending flush bytes, oldest pin, stalls and file count. Exact crash-safe range deletion is deferred until specified; no unsafe fast-path is required for the first release.

Reservations compose with query spill, backups, backfills and shard precopy at node level. Enforce the aggregate budget, not just separate per-job caps. Long pins have an explicit admission/expiry contract; a pin cannot disappear merely because the GC worker needs space. Log truncation across a shared domain advances only through a completely covered prefix, even when a later shard sequence has already flushed. Validate hot/cold shard flush combinations and interrupted maintenance against a reference history.

## Recovery, verification and upgrades

Recovery validates `CURRENT`, manifest identity/checksums and every required file/version, rebuilds in-memory metadata, then replays complete batches beyond the recoverable checkpoint. Distinguish an incomplete final append under the stated crash-only storage model from a checksum failure in a fully framed record or interior log. Never skip a corrupt record and continue as though data were complete. Replicated recovery checks required consensus indexes and can fetch a verified snapshot/log from a healthy peer. Corruption that invalidates promised durable history requires quarantine/recovery, not automatic tail truncation.

Inspection tools must read the format without starting the SQL server: header/manifest/WAL dump with redaction, checksum scan, dry-run recovery and scoped logical export. Normal repair restores from a verified replica/backup. Any best-effort salvage is a separately authorized offline export into a new directory with explicit missing-data reporting; never mutate the only copy or advertise salvage as verified recovery.

Each disk-format change ships fixtures and a reader/writer compatibility matrix. Unknown mandatory formats refuse startup. Upgrade gates check the whole cluster before writers enable a new format; old files can remain readable while compaction rewrites them. Downgrade stops at an explicit irreversible boundary. Backups retain format/codec/key metadata and a supported restore/conversion route. Fail closed on unsupported/corrupt data instead of silently dropping fields or rows.

## Storage-specific release gates

- M0: ownership-boundary check; first-party LSM/B+tree experiment; format/durability RFC; model of manifest/WAL publication and deletion; byte-level fixture plan.
- M1: first-party WAL, table, manifest, cache and bounded compaction paths pass model/reference tests; crash injection at every write/flush/rename/directory-sync boundary preserves acknowledged batches; cold restart and clean-directory restore recover without a third-party engine.
- M2: integrate durable Raft hard state/logs, snapshot installation, fenced shard splitting and prepared transactions; prove replay fences and a transaction-consistent clean-cluster restore in failure tests.
- M3: snapshot/backup/CDC/object retention all constrain file/history GC; restore into a fresh process/cluster validates bytes and relational invariants.
- M4: malformed-file fuzzing, bit-flip/truncation/reordered-write campaigns, real filesystem/device recovery, seven-day mixed-load compaction/space stress, format upgrade/restore drills, and independent persistence-protocol review.

Source references are in [research](research.md), S28 and S36–S40. These references explain established persistence mechanisms; the layout and contracts in this document are Falcone design proposals. No performance or durability guarantee is established until implementation evidence satisfies the gates. Implementation tasks are E19–E23, with E02 covering their integration into relational MVCC storage.


## Fourth-book integration additions — 4 November 2026

B4 does not change first-party engine/file ownership or the M0 LSM-versus-B+tree evaluation. Its page/ARIES and alternative index examples are references, not instructions to add a second persistence backend. E19–E23 remain the storage implementation owners.

The [relational contracts](relational-operations.md) add native durable record families for catalog dependencies, sequence reservations/generations, job progress and (later) materialized-generation manifests. Each needs version/bounds/checksum/replay fixtures under the existing format policy; do not serialize host/Rust structure layouts. Resolved identity/default values and typed mutations replay deterministically across supported versions.

E26-S03's private sorted index runs use owned codecs and comparator versions. Installing files must use a qualified manifest/WAL/consensus protocol, with checksum/uniqueness verification and recovery evidence before publication. This does not allow foreign SST ingestion or disabled logging. Bulk construction, ordinary DML and replicated recovery must agree on one visible index generation.

Import/export jobs and later materialized views join shared retention accounting. Old source cuts, staging files and view generations are reclaimed only after all readers/transactions/backups/CDC/object obligations release them. A canceled worker is not by itself evidence that bytes are unreachable. Exported logical chunks do not replace a physical native backup or its format/codec/key dependencies.
