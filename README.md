# Falcone RDBMS

Design proposal for a first-party, distributed relational database for [Falcone](https://github.com/gntik-ai/falcone).

**Status: architecture and planning only. No database engine is implemented, no performance claims have been validated, and the proposed decisions are open for review.** Research date: 2 November 2026.

The proposed engine is written in **Rust**, with native sharding, distributed transactions, many lightweight client sessions, SQL-free administration, geospatial types, and managed S3-backed large objects. It can run independently or integrate with Falcone's tenant, workspace, identity, storage, event, and quota services.

## Read the proposal

| Document | Contents |
| --- | --- |
| [Research](docs/design/research.md) | Comparison of database projects, features to adopt, language and reuse decisions, primary sources |
| [Architecture](docs/design/architecture.md) | Components, storage, consensus, transactions, sharding, connections, isolation, and failure behavior |
| [API and data model](docs/design/api-and-data-model.md) | JSON administration, gRPC contracts, schema operations, geospatial semantics, S3 object lifecycle |
| [Compatibility and migration](docs/design/compatibility-and-migration.md) | PostgreSQL support boundaries and migration into the existing Falcone platform |
| [Quality and roadmap](docs/design/quality-and-roadmap.md) | Milestone gates, proposed benchmark targets, correctness testing, delivery risks |
| [Epics and user stories](docs/backlog/epics-and-stories.md) | 18 epics and 72 stories with acceptance criteria, dependencies, and milestones |

## Recommended architecture decisions

1. Own the relational engine, transaction protocol, catalog, sharding, policy enforcement, and administration APIs.
2. Reuse qualified embedded libraries: initially RocksDB for local persistence and a Rust Raft implementation for consensus. RocksDB is C++; the first-party engine remains Rust. This avoids a separately operated PostgreSQL, TiKV, or FoundationDB dependency.
3. Support a tested PostgreSQL wire/SQL subset for clients, **not** a claim of full PostgreSQL compatibility. PostgreSQL extensions do not become compatible merely by speaking its protocol.
4. Expose REST/JSON for all administration and gRPC/Protobuf between components. Both feed the same typed catalog operations and authorization checks.
5. Start with one region and three failure domains. Cross-region synchronous writes and disaggregated object-backed table storage are later milestones.
6. Keep file bytes in S3; manage immutable object identities, authorization, references, garbage collection, and backup retention in the database.
7. Make serializability, recovery, tenant isolation, and restore verification release gates.

## Proposed milestones

| Milestone | Exit condition |
| --- | --- |
| M0 — feasibility | Compatibility inventory, transaction/shard models, dependency qualification, benchmark harness |
| M1 — local alpha | Durable local SQL, scoped JSON administration, gateway, single-node recovery |
| M2 — distributed alpha | Replicated shards, distributed transactions, split/move recovery, gateway scale tests |
| M3 — constrained beta | Defined geo subset, managed S3 objects, PITR, CDC, Falcone adapter and migration rehearsal |
| M4 — production candidate | Independent correctness review, sustained fault tests, driver certification, security and restore gates |
| M5 — expansion | Multi-region, analytical tiers, broader SQL, advanced GIS, optional search features |

Milestones are evidence gates, not release dates. PostgreSQL remains available during migration. Replacing Falcone's relational backend does not automatically replace the PostgreSQL dependency of FerretDB/DocumentDB or other platform components.

See the backlog for proposed implementation work. Story identifiers are planning IDs, not GitHub issue numbers.
