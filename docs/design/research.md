# Research and design decisions

Research date: **2 November 2026**. The target repository was empty when inspected. This is a proposal, not a code audit of a running Falcone RDBMS.

**Documentation review added 3 November 2026:** [essential features and findings](documentation-review.md) examines the three uploaded books, maps their 43 main chapters to the backlog, and explains changes to the original eight documents and 115 issues. B1 = Sciore, *Database Design and Implementation*, 2nd ed. (2020); B2 = Kleppmann/Riccomini, *Designing Data-Intensive Applications*, 2nd ed. (2026); B3 = Petrov, *Database Internals* (2019). Exact uploaded-file hashes, review coverage and chapter references are recorded there. The books inform design choices; their historical product examples are not new current-product verification.

The review confirms Rust/native storage and prioritizes replicated operation, conditional physical sharding, bounded execution, relational integrity and early restore evidence. It promotes consistent snapshots and management APIs into the core, specifies a fenced-drain split baseline, and defers polished UI/CLI plus advanced workloads. See [distributed contracts](distributed-contracts.md) for the resulting implementation obligations.

## Falcone baseline inspected

Falcone's README describes a Node.js/pnpm/Turbo platform deployed on Kubernetes, with tenant/workspace boundaries, plan capabilities, PostgreSQL, SeaweedFS, Kafka, OpenBao, and FerretDB/DocumentDB. Selected integration files were inspected at commit `9c477e46b20ceb0a28c56d63d0f738e886580813`; this is a source snapshot, not verification of a deployed cluster.

| Evidence | Finding relevant to this design |
| --- | --- |
| [README](https://github.com/gntik-ai/falcone/blob/main/README.md) | Multitenant BaaS; document service also depends on a PostgreSQL-based engine |
| [Data API reference](https://github.com/gntik-ai/falcone/blob/9c477e46b20ceb0a28c56d63d0f738e886580813/docs-site/api/postgresql.md) | Workspace-addressed CRUD, structural management, pagination, and vector search |
| [DDL executor](https://github.com/gntik-ai/falcone/blob/9c477e46b20ceb0a28c56d63d0f738e886580813/apps/control-plane-executor/src/runtime/postgres-ddl-executor.mjs) | Structural/governance plans, preview execution, grants and forced tenant RLS |
| [Realtime executor](https://github.com/gntik-ai/falcone/blob/9c477e46b20ceb0a28c56d63d0f738e886580813/apps/control-plane-executor/src/runtime/postgres-realtime-executor.mjs) | This particular runtime path uses triggers and dedicated LISTEN connections; it is not a durable CDC log |
| [Data adapter](https://github.com/gntik-ai/falcone/blob/9c477e46b20ceb0a28c56d63d0f738e886580813/packages/adapters/src/postgresql-data-api.mjs) | Existing capability names, query operations, filter operators, and pgvector distance operators |

Consequences: build a new backend adapter while preserving scoped contracts; keep engine policy checks even when the gateway authenticates; plan durable change streams explicitly; treat vector compatibility and DocumentDB migration as separate work. This inspection does not establish that every documented capability is live.

## Database comparison

The “adopt” column is a Falcone recommendation inferred from primary documentation. Inclusion does not mean source code will be copied or the product will run underneath Falcone.

| Project | Relevant documented strengths | Adopt for Falcone | Boundary or tradeoff |
| --- | --- | --- | --- |
| **PostgreSQL** | Broad relational semantics, SQL conformance documentation, policies, declarative partitioning | Behavioral reference; rich types, constraints, explain plans, policies and migration expectations | Local table partitioning is not a complete distributed sharding/consensus system [S1–S3] |
| **MySQL / InnoDB** | MVCC, row locking, crash recovery, primary-key clustering, foreign keys, spatial indexes | Explicit recovery invariants, concurrency behavior, relational integrity | InnoDB alone is not a distributed cluster engine; do not confuse it with NDB [S4] |
| **CockroachDB** | SQL over replicated ranges; distributed transaction records/intents; layered architecture | Small replicated shards, online movement, transaction recovery, explicit consistency modes | Use as behavioral reference; current repository licensing is CSL, not a permissive source base [S5–S7] |
| **YugabyteDB** | DocDB tablets, hash/range distribution, automatic splitting, cross-table transactions | Colocation, split/move orchestration, replicated transaction decisions | PostgreSQL-facing and distributed storage layers remain complex; core and management licenses differ [S8–S11] |
| **TiDB / TiKV** | Stateless SQL compute, placement/timestamp service, transactional KV regions | Separate compute from storage; load-aware placement; scalable region management | TiKV is not an SQL RDBMS by itself; its default snapshot isolation is not evidence of serializability [S12–S14] |
| **Citus** | PostgreSQL extension with distributed/reference tables and distributed execution | Distribution keys, colocated joins, scoped reference data | Retains PostgreSQL; distribution restrictions must be visible. Treat as reference, not presumed permissive reuse [S15–S16] |
| **Vitess** | Routing gateway, connection pooling, sharding-aware execution | Lightweight connection front end; clear session-state limits; topology-aware routing | Built around MySQL; its connection architecture is more relevant here than engine reuse [S17–S18] |
| **Neon** | Separate compute, replicated WAL, page service and object-backed history | Long-term inspiration for independent storage/compute and branching | Does not itself establish horizontally sharded OLTP writes; too much additional storage complexity for the first release [S19] |
| **FoundationDB** | Transactional distributed KV and deterministic simulation | Simulation, failure reproduction and small explicit correctness contracts | Not an SQL engine; adopting the service would change the goal to an SQL layer on an external database [S20] |
| **DuckDB** | Analytical SQL and S3 file access through httpfs | Later Parquet/CSV scans and analytical export lane | S3 query access does not solve distributed transactional storage [S21] |
| **SQLite** | Embedded SQL, simplicity, reliable local persistence | Local developer experience and test-oracle opportunities for shared semantics | Its intended local concurrency model does not meet the distributed service requirement [S22] |
| **PostGIS** | Geometry/geography distinction and spatial indexing/functions | GIS semantics and correctness oracle for a named supported subset | Not a standalone RDBMS; do not claim compatibility with its entire extension surface [S23–S24] |
| **Oracle distributed database** | System-managed, user-defined and composite distribution; shard-aware routing | Placement policy plus automatic subdivision within a residency boundary | Commercial reference only; do not infer edition availability or costs [S25] |
| **SQL Server FILESTREAM** | Relational association with externally stored file bytes | Treat large objects as a managed database capability | FILESTREAM uses filesystem integration, not an atomic transaction protocol for arbitrary S3 [S26] |

## Language decision

**Recommend Rust for all first-party engine services.** Rust's ownership model and absence of mandatory garbage collection suit a memory-sensitive database engine [S27]. This is an engineering choice, not proof that Rust will outperform C or Go for a particular workload.

| Language | Fit | Main cost | Decision |
| --- | --- | --- | --- |
| Rust | Storage, protocol parsing, query execution, concurrent services | More demanding implementation and FFI discipline | Primary language |
| C | Low-level control; PostgreSQL heritage | Manual memory management and a larger audit burden for new network-facing code | Limited interoperability only when justified |
| Go | Strong service/tooling ergonomics and concurrency model | Allocation/GC behavior must be measured under tail-latency pressure | Valid alternative, but do not add a second first-party language without a concrete benefit |

Existing Falcone Node.js services remain outside the database engine; gRPC is the integration boundary. An operator can also be Rust to keep the service uniform.

## Build/reuse decision

**Confirmed user decision: own both the storage engine and its disk format.** This supersedes the earlier RocksDB embedding recommendation. External engines remain behavioral/performance references; implementing storage ownership is in the initial delivery scope, not a future replacement project.

| Option | Assessment |
| --- | --- |
| Wrap PostgreSQL/Citus | Fastest route to a custom managed service, but does not achieve an independent RDBMS engine |
| Build SQL on externally deployed TiKV/FoundationDB | Reduces persistence/transaction work; still introduces a separately operated database dependency |
| Fork an entire distributed RDBMS | Substantial inherited behavior, language and upgrade obligations; licensing and architecture constrain autonomy |
| Own relational engine on an embedded third-party storage engine | Does not meet the confirmed storage ownership requirement; excluded as the shipping design |
| **Own relational engine, storage engine and disk format** | Selected boundary: first-party Rust persistence, recovery, table organization and maintenance; reuse qualified general-purpose primitives |
| Implement every consensus/crypto/compression primitive from scratch | Not required by storage ownership. Qualified primitives remain eligible; Falcone implements their persistence integration and format contracts |

Storage-layout recommendation: start with a first-party LSM because immutable files and ordered versioned keys fit the proposed shard/MVCC design. This is an architectural hypothesis; M0 compares it with a first-party B+tree prototype, including read latency, updates, compaction/write amplification and crash behavior. The decision selects a layout, not an external engine. See [native storage and disk format](native-storage.md).

Reference systems and candidate components, **not approved pinned dependencies**:

- **RocksDB / LevelDB** as references for ordered storage, maintenance tradeoffs and benchmark baselines [S28, S36–S38]. Their code, WAL/SST formats and engine binaries are not the storage implementation or compatibility target. Falcone owns a separately specified format and implementation.
- **raft-rs** as a candidate Rust consensus core [S29]. Falcone still implements transport, durable logs, snapshots, membership changes, scheduling and recovery. Qualification includes maintenance, API suitability and fault tests at a pinned revision.
- **Apache Arrow / DataFusion** for batch representation and possibly analytical operators [S30]. Do not assume its SQL support supplies an OLTP transaction engine, PostgreSQL semantics or tenant authorization. Keep a dedicated OLTP execution path initially.
- **GeoRust** for candidate geometry primitives and algorithms [S31]. Qualify each supported algorithm; projection or geodesic dependencies require separate review. Whole-PostGIS compatibility is not implied.
- Candidate Tokio, tonic, rustls, checksum/compression/cryptography primitives, PostgreSQL protocol, S3 client and SQL-parser crates require exact-version, security, license, behavior and maintenance qualification in E01 before selection. An embedded database disguised behind a Rust wrapper does not meet the ownership boundary. Storage-format parsing and persistence state machines remain first-party code.

The research adds explicit persistence references: LevelDB documents log framing and block-indexed table structures; PostgreSQL documents write-ahead ordering and grouped log flushes; SQLite's atomic-commit discussion identifies filesystem/power-loss assumptions. These inform Falcone's independent durability contracts and fault tests; they do not prove the proposed implementation correct [S36–S40].

The first-party source-license recommendation is Apache-2.0, subject to repository-owner acceptance. No license grant is created by this document. Preserve upstream notices and record exact source/version/file licenses. A rewrite into another language does not automatically remove upstream license obligations.

## Primary sources

Links were consulted on 2 November 2026. “stable/current” pages are moving references; implementation work must pin versions. Research does not certify advertised performance or every edition's feature availability.

| ID | Source |
| --- | --- |
| S1 | [PostgreSQL SQL conformance](https://www.postgresql.org/docs/current/features.html) |
| S2 | [PostgreSQL partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html) |
| S3 | [PostgreSQL row security](https://www.postgresql.org/docs/17/ddl-rowsecurity.html) |
| S4 | [MySQL InnoDB introduction and feature table](https://dev.mysql.com/doc/refman/8.4/en/innodb-introduction.html) |
| S5 | [CockroachDB transaction layer](https://docs.cockroachlabs.com/docs/stable/architecture/transaction-layer) |
| S6 | [CockroachDB evolving architecture](https://www.cockroachlabs.com/blog/continuum-architecture/) |
| S7 | [CockroachDB license](https://github.com/cockroachdb/cockroach/blob/master/LICENSE) |
| S8 | [YugabyteDB DocDB](https://docs.yugabyte.com/stable/architecture/docdb/) |
| S9 | [YugabyteDB sharding](https://docs.yugabyte.com/stable/architecture/docdb-sharding/) |
| S10 | [YugabyteDB distributed transactions](https://docs.yugabyte.com/stable/architecture/transactions/distributed-txns/) |
| S11 | [YugabyteDB component licensing](https://github.com/yugabyte/yugabyte-db/blob/master/LICENSE.md) |
| S12 | [TiDB architecture](https://docs.pingcap.com/tidb/stable/tidb-architecture/) |
| S13 | [TiDB architecture FAQ](https://docs.pingcap.com/tidb/stable/tidb-faq/) |
| S14 | [TiKV repository](https://github.com/tikv/tikv) |
| S15 | [Citus overview](https://docs.citusdata.com/en/stable/get_started/what_is_citus.html) |
| S16 | [Citus repository](https://github.com/citusdata/citus) |
| S17 | [Vitess architecture](https://vitess.io/docs/24.0/overview/architecture/) |
| S18 | [Vitess VTGate](https://vitess.io/docs/24.0/concepts/vtgate/) |
| S19 | [Neon WAL, safekeepers and object storage](https://neon.com/blog/wal-s3-lakebase-storage-for-the-era-of-agents) |
| S20 | [FoundationDB paper](https://www.foundationdb.org/files/fdb-paper.pdf) |
| S21 | [DuckDB S3 API support](https://duckdb.org/docs/current/core_extensions/httpfs/s3api) |
| S22 | [Appropriate uses for SQLite](https://www.sqlite.org/whentouse.html) |
| S23 | [PostGIS geometry versus geography](https://postgis.net/documentation/faq/geometry-or-geography/) |
| S24 | [PostGIS manual](https://www.postgis.net/docs/manual-dev/postgis-es.html) |
| S25 | [Oracle distributed database overview](https://docs.oracle.com/en/database/oracle/oracle-database/26/shard/overview1.html) |
| S26 | [SQL Server BLOB storage options](https://learn.microsoft.com/en-us/sql/relational-databases/blob/binary-large-object-blob-data-sql-server?view=sql-server-ver17) |
| S27 | [Rust language](https://rust-lang.org/) |
| S28 | [RocksDB README and licensing](https://github.com/meta/rocksdb/blob/main/README.md) |
| S29 | [raft-rs](https://github.com/tikv/raft-rs) |
| S30 | [DataFusion introduction](https://datafusion.apache.org/user-guide/introduction.html) |
| S31 | [GeoRust](https://github.com/georust/geo) and [API documentation](https://docs.rs/geo/latest/geo/) |
| S32 | [PgBouncer pooling compatibility](https://www.pgbouncer.org/features.html) |
| S33 | [S3 multipart upload and checksum behavior](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html) |
| S34 | [S3 conditional writes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html) |
| S35 | [PostgreSQL logical replication restrictions](https://www.postgresql.org/docs/current/logical-replication-restrictions.html) |
| S36 | [LevelDB implementation organization](https://github.com/google/leveldb/blob/main/doc/impl.md) |
| S37 | [LevelDB log framing](https://github.com/google/leveldb/blob/main/doc/log_format.md) |
| S38 | [LevelDB table format](https://github.com/google/leveldb/blob/main/doc/table_format.md) |
| S39 | [SQLite atomic commit and hardware assumptions](https://www.sqlite.org/atomiccommit.html) |
| S40 | [PostgreSQL write-ahead logging](https://www.postgresql.org/docs/current/wal-intro.html) |
| S41 | [Ongaro and Ousterhout, In Search of an Understandable Consensus Algorithm, extended version](https://raft.github.io/raft.pdf), §§5.4, 6–8; consulted 3 November 2026 to qualify consensus rules |

The architecture, targets, roadmap and backlog in the accompanying documents are proposed Falcone designs derived from these observations, not claims that upstream systems implement the exact combination.
