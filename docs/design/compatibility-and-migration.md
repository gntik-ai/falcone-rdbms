# PostgreSQL compatibility and Falcone migration

Status: planned support, not current compatibility. The live Falcone deployment and full dependency graph must be inventoried before migration.

## Compatibility contract

Publish a machine-readable feature matrix and certify specific driver versions. PostgreSQL wire compatibility, SQL syntax, SQL behavior, system catalogs, extensions and replication protocol are distinct contracts. Do not advertise “drop-in PostgreSQL replacement”.

| Area | Target | Explicit limitation |
| --- | --- | --- |
| Wire protocol | Startup/TLS/auth, simple and extended queries, bind/execute, cancellation, errors, transactions | Certify Node `pg` first, then selected Python/JDBC/Go/Rust drivers; no blanket compatibility |
| Types | bool, signed integers, exact decimal, float, UTF-8 text, byte arrays, UUID, date/time/timestamptz, JSON | Freeze overflow, NaN, NULL, timezone, decimal and collation semantics; SQL JSON compatibility does not imply PostgreSQL jsonb binary format |
| Queries | CRUD, upsert subset, joins, aggregates, ordering, paging; common non-recursive CTEs and windows before wider certification | Recursive CTEs, broad function parity, exotic operators and planner equivalence are later |
| Integrity | PK, NOT NULL, UNIQUE, CHECK, FK; scoped global indexes where required | Bound cascades; deferred/exclusion constraints require a separate implementation |
| Transactions | Serializable default, explicit transactions and retry/unknown-outcome errors | No weaker isolation mode until separately validated; serializable is not a strict-serializable marketing claim |
| Schema | Tables, schemas, indexes, defaults, identities, views, online supported changes | Transactional DDL subset; long online jobs have their own lifecycle |
| Introspection | Useful `information_schema` and selected `pg_catalog` views/types | Implement only documented driver/ORM needs; do not fabricate unsupported extensions |
| Session state | Prepared statements, scoped settings, transaction state | Temp tables, session advisory locks and raw LISTEN/NOTIFY are not initial requirements |
| Bulk I/O | CSV/JSON import/export and selected COPY protocol support | No binary PG dump/physical backup compatibility |
| Spatial | Declared Falcone geometry/geography subset | Not binary/function-compatible with all PostGIS |
| Search | Exact vector distance pilot; ANN/full-text in expansion | Existing pgvector-dependent workspaces stay on PostgreSQL until required semantics are certified |
| Extensions/procedural code | Feature inventory and unsupported-feature errors | No PostgreSQL extension ABI, PL/pgSQL, user triggers or arbitrary C extensions initially |
| Replication | Falcone-native durable change stream | No initial PostgreSQL WAL/physical replication protocol compatibility |

The unsupported-feature report is a hard migration gate. A connection that succeeds but misinterprets a type, collation or SQL statement is a failure, not partial success.

## Falcone integration changes

Introduce a backend-neutral database interface around provision, inspect, plan/apply schema, execute data query, manage scoped identity, stream changes, backup/restore and report capabilities. Keep the current PostgreSQL implementation alongside `falcone-rdbms`.

Relevant inspected paths are linked in [research](research.md). Add a Rust-service adapter from the existing Node control-plane/executor; preserve request IDs, idempotency, effective roles, plan capability checks and tenant/workspace binding. Replace SQL-specific plan objects with typed operations at the boundary while retaining a PostgreSQL renderer for the existing backend.

Proposed routing: new backend-neutral `/v1/databases/...` contracts; keep `/v1/postgres/...` for actual PostgreSQL and a documented legacy mapping only where behavior is identical. Avoid silently serving a different database engine under an endpoint whose capability semantics imply PostgreSQL extensions.

Integrations:

- IAM supplies authenticated principals; the engine independently enforces scoped authorization.
- Falcone Secrets provides references/rotation for keys and object credentials; define bootstrap and outage behavior.
- Falcone Storage supplies S3-compatible object bytes; the RDBMS owns row references, snapshot pins and GC coordination.
- Events/realtime consume committed resumable streams with transaction boundaries, not raw storage intents.
- Provisioning reconciles durable operation states; active status means actual usable resources exist.
- Plans/quotas meter sessions, active queries, CPU, stored rows/indexes, blobs, backups and egress without double billing shared physical bytes.
- Logs/metrics correlate gateway, query, transaction, shard and object operations while redacting values.
- Helm/Kubernetes integrates the new service with persistent volumes, topology constraints, secrets, health and disruption budgets.

The engine must also support standalone authentication/configuration for independent tests and deployments; Falcone integrations are providers, not circular startup dependencies.

## Migration stages

1. **Inventory:** catalog actual schemas, volume/churn, extensions, procedures/triggers, advisory locks, collations, sequences, prepared statements, drivers/ORMs, RLS, backup needs and query fingerprints. Classify every workspace as supported, transformable or blocked. Include DocumentDB, internal Falcone persistence and workflow dependencies separately.
2. **Prepare:** create equivalent schema and policies through typed plans; map types and IDs; validate shard choices and object-store bindings; compare query plans and cost. Freeze unsupported DDL during replication or coordinate explicit schema changes.
3. **Snapshot plus CDC:** establish a consistent PostgreSQL snapshot and corresponding logical replication position without a gap. Import scoped data, then apply ordered changes with source transaction boundaries and deduplication. Bound replication-slot WAL growth and stop safely on lag/disk risk.
4. **Reconcile:** compare scoped counts, canonical checksums, constraints and representative query results at the same source position. Differences in floating-point/time/collation handling require documented resolution, not ignored mismatches. Reconcile large-object contents separately.
5. **Shadow reads:** replay sanitized production-shaped reads against the new engine; compare semantics and performance. Do not duplicate arbitrary writes to two systems without a defined consistency protocol.
6. **Cut over one workspace:** fence source writers, drain active transactions, record final LSN, catch up exactly, verify data/sequence values, switch the backend binding with a new generation, then permit target writes. Old connections must not continue writing to the old generation.
7. **Monitor and expand:** canary workspaces first; preserve immutable migration evidence, source backup and explicit rollback window.

PostgreSQL logical replication does not automatically replicate schema DDL or sequence state; handle both explicitly [research S35]. Tables without usable replica identity, partition changes, large objects and extension-defined types need specific migration strategies. A generic CDC connector is not sufficient evidence of correct migration.

## Rollback boundary

Before target writes begin, route back to the fenced source after verification. **After target writes begin, the old PostgreSQL source is stale.** Rollback requires tested reverse CDC for the supported subset or a controlled export/reconciliation/write freeze. Otherwise recovery proceeds forward on Falcone RDBMS. No “instant rollback” claim is allowed merely because the old database still exists.

Retain PostgreSQL for blocked workspaces and dependencies. Replacing the relational service does not remove FerretDB/DocumentDB's PostgreSQL engine or prove compatibility with Temporal/other infrastructure databases. Retire those only through separately scoped migrations.
