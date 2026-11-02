# Falcone RDBMS

Design proposal for a first-party, distributed relational database for [Falcone](https://github.com/gntik-ai/falcone).

**Status: architecture and planning only. No database engine is implemented, no performance claims have been validated, and the proposed decisions are open for review.** Research date: 2 November 2026.

The proposed engine is written in **Rust**, with native sharding, distributed transactions, many lightweight client sessions, SQL-free administration, geospatial types, and managed S3-backed large objects. It can run independently or integrate with Falcone's tenant, workspace, identity, storage, event, and quota services.

**Confirmed scope: Falcone owns the storage engine and disk format too.** The engine, WAL, immutable table files, manifest, cache, compaction and recovery code are first-party Rust. RocksDB and other database/storage engines are research and benchmark references only, not runtime backends.

## Read the proposal

| Document | Contents |
| --- | --- |
| [Research](docs/design/research.md) | Comparison of database projects, features to adopt, language and reuse decisions, primary sources |
| [Architecture](docs/design/architecture.md) | Components, storage, consensus, transactions, sharding, connections, isolation, and failure behavior |
| [Native storage and disk format](docs/design/native-storage.md) | First-party storage ownership, proposed LSM design, file layout, WAL/recovery, cache, compaction, and format evolution |
| [API and data model](docs/design/api-and-data-model.md) | JSON administration, gRPC contracts, schema operations, geospatial semantics, S3 object lifecycle |
| [Compatibility and migration](docs/design/compatibility-and-migration.md) | PostgreSQL support boundaries and migration into the existing Falcone platform |
| [Quality and roadmap](docs/design/quality-and-roadmap.md) | Milestone gates, proposed benchmark targets, correctness testing, delivery risks |
| [Epics and user stories](docs/backlog/epics-and-stories.md) | 23 epics and 92 stories with acceptance criteria, dependencies, and milestones |

## Recommended architecture decisions

1. Own the relational engine, transaction protocol, catalog, sharding, policy enforcement, administration APIs, storage engine and persistent disk format.
2. Implement the storage layer in Rust. Start from a first-party LSM design, qualify it against a B+tree prototype in M0, and own persistence/recovery whichever layout is selected. General-purpose I/O, checksum, compression and cryptography libraries remain allowed after qualification. A qualified Rust Raft core may implement consensus; Falcone owns its durable log adapter. No embedded third-party database/storage engine is used as the shipping backend.
3. Support a tested PostgreSQL wire/SQL subset for clients, **not** a claim of full PostgreSQL compatibility. PostgreSQL extensions do not become compatible merely by speaking its protocol.
4. Expose REST/JSON for all administration and gRPC/Protobuf between components. Both feed the same typed catalog operations and authorization checks.
5. Start with one region and three failure domains. Cross-region synchronous writes and disaggregated object-backed table storage are later milestones.
6. Keep file bytes in S3; manage immutable object identities, authorization, references, garbage collection, and backup retention in the database.
7. Make serializability, recovery, tenant isolation, and restore verification release gates.

## Proposed milestones

| Milestone | Exit condition |
| --- | --- |
| M0 — feasibility | Compatibility inventory, native LSM/B+tree evaluation, disk-format RFC, transaction/shard and persistence models, dependency qualification, benchmark harness |
| M1 — local alpha | First-party WAL/tables/manifest/cache/compaction and verified crash recovery, then durable local SQL, scoped JSON administration and gateway |
| M2 — distributed alpha | Replicated shards, distributed transactions, split/move recovery, gateway scale tests |
| M3 — constrained beta | Defined geo subset, managed S3 objects, PITR, CDC, Falcone adapter and migration rehearsal |
| M4 — production candidate | Independent correctness review, sustained fault tests, driver certification, security and restore gates |
| M5 — expansion | Multi-region, analytical tiers, broader SQL, advanced GIS, optional search features |

Milestones are evidence gates, not release dates. PostgreSQL remains available during migration. Replacing Falcone's relational backend does not automatically replace the PostgreSQL dependency of FerretDB/DocumentDB or other platform components.

See the backlog for proposed implementation work. Story identifiers are planning IDs, not GitHub issue numbers.
