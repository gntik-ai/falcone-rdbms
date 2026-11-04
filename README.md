# Falcone RDBMS

Design proposal for a first-party, distributed relational database for [Falcone](https://github.com/gntik-ai/falcone).

**Status: architecture and planning only. No database engine is implemented, no performance claims have been validated, and the proposed decisions are open for review.** Initial research: 2 November 2026. Three-book review: 3 November 2026. Fourth-book architecture/backlog extension: 4 November 2026.

The proposed engine is written in **Rust**, with native sharding, distributed transactions, many lightweight client sessions, SQL-free administration, geospatial types, and managed S3-backed large objects. It can run independently or integrate with Falcone's tenant, workspace, identity, storage, event, and quota services.

**Confirmed scope: Falcone owns the storage engine and disk format too.** The engine, WAL, immutable table files, manifest, cache, compaction and recovery code are first-party Rust. RocksDB and other database/storage engines are research and benchmark references only, not runtime backends.

## Read the proposal

| Document | Contents |
| --- | --- |
| [Fourth-book review](docs/design/database-system-concepts-review.md) | Database System Concepts: 26-chapter map, 14 findings, four new epics and targeted revisions |
| [Relational and data-operation contracts](docs/design/relational-operations.md) | Catalog dependencies/views, identity, statistics/plans, bulk jobs and later materialized views |
| [Documentation review and essential features](docs/design/documentation-review.md) | Prioritized core, 20 findings, 43-chapter applicability map, source provenance and reasons for changing the backlog |
| [Distributed core contracts](docs/design/distributed-contracts.md) | Small-to-large deployment, native shard transitions, transaction/snapshot boundaries, bounded execution and failure scenarios |
| [Research](docs/design/research.md) | Comparison of database projects, features to adopt, language and reuse decisions, primary sources |
| [Architecture](docs/design/architecture.md) | Components, storage, consensus, transactions, sharding, connections, isolation, and failure behavior |
| [Native storage and disk format](docs/design/native-storage.md) | First-party storage ownership, proposed LSM design, file layout, WAL/recovery, cache, compaction, and format evolution |
| [API and data model](docs/design/api-and-data-model.md) | JSON administration, gRPC contracts, schema operations, geospatial semantics, S3 object lifecycle |
| [Compatibility and migration](docs/design/compatibility-and-migration.md) | PostgreSQL support boundaries and migration into the existing Falcone platform |
| [Quality and roadmap](docs/design/quality-and-roadmap.md) | Milestone gates, proposed benchmark targets, correctness testing, delivery risks |
| [Epics and user stories](docs/backlog/epics-and-stories.md) | 27 epics and 108 stories with acceptance criteria, dependencies, and milestones |

## Recommended architecture decisions

1. Own the relational engine, transaction protocol, catalog, sharding, policy enforcement, administration APIs, storage engine and persistent disk format.
2. Implement the storage layer in Rust. Start from a first-party LSM design, qualify it against a B+tree prototype in M0, and own persistence/recovery whichever layout is selected. General-purpose I/O, checksum, compression and cryptography libraries remain allowed after qualification. A qualified Rust Raft core may implement consensus; Falcone owns its durable log adapter. No embedded third-party database/storage engine is used as the shipping backend.
3. Support a tested PostgreSQL wire/SQL subset for clients, **not** a claim of full PostgreSQL compatibility. PostgreSQL extensions do not become compatible merely by speaking its protocol.
4. Expose REST/JSON for all administration and gRPC/Protobuf between components. Both feed the same typed catalog operations and authorization checks.
5. Start with one region and three failure domains. Cross-region synchronous writes and disaggregated object-backed table storage are later milestones.
6. Keep file bytes in S3; manage immutable object identities, authorization, references, garbage collection, and backup retention in the database.
7. Make serializability, recovery, tenant isolation, and restore verification release gates, starting with local restore in M1 and distributed restore in M2.
8. Replicate a small production deployment from the start; grow physical ranges when data, throughput or isolation requires sharding. Scale clients, execution, data, metadata and maintenance independently and measure each limit.

The [essential feature ranking](docs/design/documentation-review.md#the-most-important-capabilities) prioritizes durable native storage, replication, elastic sharding, bounded concurrency, relational integrity and practical query execution. JSON administration is core. Geospatial and S3 remain required for M3; a polished admin console, multi-region operation and analytical/search expansion follow the core. The [issue backlog](https://github.com/gntik-ai/falcone-rdbms/issues) retains all previous identities and adds E24–E27. E24–E26 implement detailed catalog, optimizer and bulk-operation contracts beneath the existing integration epics; E27 adds optional materialized views.

## Proposed milestones

| Milestone | Exit condition |
| --- | --- |
| M0 — feasibility | Compatibility inventory, native LSM/B+tree evaluation, disk-format RFC, transaction/shard and persistence models, dependency qualification, benchmark harness |
| M1 — local alpha | First-party persistence and crash/clean-directory restore, durable local SQL, scoped JSON administration and bounded gateway |
| M2 — distributed alpha | Replicated shards, distributed transactions, consistent snapshots/clean-cluster restore, fenced split/move recovery, distributed plans and scale tests |
| M3 — constrained beta | Defined geo subset, managed S3 objects, PITR, CDC, Falcone adapter and migration rehearsal |
| M4 — production candidate | Independent correctness review, sustained fault tests, driver certification, security and restore gates |
| M5 — expansion | Multi-region, analytical tiers, broader SQL, advanced GIS, optional search features and polished administration UI/CLI |

The fourth-book review adds catalog dependency/view/sequence/constraint contracts and statistics/plan validity to M1–M2; distributed physical planning and native index builds to M2–M3; resumable imports/exports to M3; and optional runtime filters/materialized views to M5. See the [delivery map](docs/design/quality-and-roadmap.md#fourth-book-delivery-map) for slices.

Milestones are evidence gates, not release dates. PostgreSQL remains available during migration. Replacing Falcone's relational backend does not automatically replace the PostgreSQL dependency of FerretDB/DocumentDB or other platform components.

See the backlog for proposed implementation work. Story identifiers are planning IDs, not GitHub issue numbers.
