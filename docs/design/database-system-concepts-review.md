# Fourth-book review: Database System Concepts

Review date: **4 November 2026**. Baseline: merged `main` commit `1b8784e3682376d07e0b10d6a29d96de87e1572c`, ten repository documents and 115 existing epic/story issues (#2–#116). PR #1 was merged on 3 November. The repository at this baseline contains planning documents, not an implemented database engine.

## Decision

Retain the **first-party Rust storage engine and file format, shared-nothing replicated storage, serializable transactions and native elastic sharding**. The additional book reinforces these choices while exposing underspecified relational and operational interfaces. Extend the architecture with catalog dependency management, explicit view and identity semantics, a statistics lifecycle, placement-aware plan properties, bounded plan reuse, and durable bulk operations. Add managed materialized views as a later capability with its own freshness and security contract.

The update adds **four epics and sixteen stories**, taking the backlog to **27 epics and 108 stories**. It also adds focused checks to **25 existing stories**. E24–E26 decompose earlier broad promises; E27 is a new optional product capability. This is not four independent new services or permission to duplicate E05/E09/E15 implementations.

## Source and review scope

**B4:** Abraham Silberschatz, Henry F. Korth and S. Sudarshan, *Database System Concepts*, seventh edition, McGraw-Hill Education, 2020. Bound-edition ISBN: **978-0-07-802215-9**. The supplied title/copyright/catalog information establishes the edition despite an isolated OCR line saying sixth edition.

- Uploaded filename: `Abraham Silberschatz, Henry Korth and S. Sudarshan - Database System Concepts. 7-McGraw-Hill Education (2020).md`.
- SHA-256: `06be6f47a5e08f298d0c1e01932824146375da25b827ee169ae9ea2ec4775bb1`.
- Approximately 566,231 whitespace-delimited words and 74,434 lines in the supplied rendering, including references, index, examples and OCR artifacts.
- The upload contains the **26 main chapters**, Appendix A and index. Its contents list online chapters 27–32, but their full text is not supplied. They are **not counted as reviewed chapters**.

The review maps every supplied main chapter using its contents, overview and summary, then reads substantive sections for identified gaps: §§4.2/4.4/4.5.6/4.7.3, 7.10, 8.1.2, 14.7, 16.3–16.6, 18.3, 20.4.2, 21.3, 22.3/22.5/22.7/22.9.3, 23.8.4–23.8.5 and 25.1/25.3.1. This is an architecture-focused review, not line-by-line verification of all 566,231 words or every exercise/proof. Section references identify the evidence; the contracts below are Falcone design inferences, not claims that the authors prescribe this exact system.

The [earlier review](documentation-review.md) remains the historical record for B1–B3. Together the two reviews map **69 main chapters across four supplied books**. Product-specific examples in B4 are historical. Its abbreviated consensus explanations do not replace the precise Raft rules already cited in the architecture. No uploaded book text is copied into the repository.

## Essential feature selection

| Priority | Addition or refinement | Why it matters to Falcone | Implementation owner |
| --- | --- | --- | --- |
| P0 | Catalog object identities and dependency graph | Safe DDL, views, plan invalidation and restore need one coherent metadata model | E24-S01; E03/E09 integration |
| P0 | Read-only invoker views and named constraints | Previously promised relational functionality needs a precise security and validation surface | E24-S02/S04; E05/E08 |
| P0 | Durable per-sequence number allocation | Many clients must not share one contended global counter or reuse IDs after failover | E24-S03 |
| P0 | Bounded statistics collection and refresh | A parser and indexes do not prevent poor plans caused by stale, skewed or incomplete estimates | E25-S01 |
| P0 | Physical plan properties, exchanges and parallelism budgets | Native sharding must preserve correct joins and predictable resource use | E25-S02; E05/E07 |
| P0 | Prepared-plan validity and cache control | Connection scale magnifies cache memory, stale authorization and recompile storms | E25-S03; E07-S02 |
| P0/P1 | Streaming imports, durable batch progress, native bulk builds and exports | Migration and large-data administration need failure-safe operations with visible commit boundaries | E26 |
| P0 | Lock hierarchy, deterministic replay and mutation correctness | Fixes gaps in existing correctness contracts without replacing their algorithms | E03/E04/E05/E17/E20 |
| P2 | Runtime Bloom filters | May reduce selective join traffic; correctness and cost need separate qualification | E25-S04 |
| P2 | Managed materialized views | Repeated summaries can become cheaper, but introduce freshness, maintenance and authorization obligations | E27 |

P0 slices are delivered alongside the feature they protect, not as an instruction to finish every epic before M1. M1 remains local, M2 proves distribution and growth, M3 includes the original geospatial/S3/CDC/Falcone baseline and operational data movement, and M4 provides production evidence. Optional materialized views and runtime filters do not block that baseline.

## Findings and architectural changes

### D01 — Make the catalog a dependency model, not just a table registry

**Before:** stable table IDs and schema epochs were present, but dependency traversal, view rebinding and destructive-change impact were implicit in E03/E09.

**Evidence:** B4 §§4.2, 4.5.8 and 13.4 describe views, schemas and the data dictionary. A view is a persistent definition whose meaning depends on other objects; changing or dropping those objects has consequences beyond a name lookup.

**Change:** persist typed edges among tables, columns, types, indexes, constraints, views, sequences and policy versions. Bind definitions to IDs and versions, freeze star expansion, reject cycles, default destructive changes to RESTRICT, and show the exact authorized CASCADE set. Revalidate that set at application time. A stale prepared plan may fail or rebind safely; it cannot silently target a newly created object with the same name. E24-S01 owns implementation, E09 owns orchestration.

### D02 — State view security and update behavior explicitly

**Before:** E05-S02 promised views without a released subset or authority model. General RLS checks alone do not settle whether a view runs with caller or creator privileges.

**Evidence:** B4 §§4.2.4 and 4.7.3 discuss ambiguous updates through views and authorization through stored definitions.

**Change:** first release read-only **invoker** views: the caller must pass view and base-object rights, with base RLS retained. Reject definer-rights and view writes until separately designed. This is a deliberate narrower compatibility profile, not a claim of PostgreSQL view equivalence. Current policy versions participate in cache validation; nested expansion and dependency depth have limits. E24-S02 and E08-S02 certify this boundary.

### D03 — Separate generated identity from transactional commit order

**Before:** identities and sequence migration were mentioned without an allocation/failure contract.

**Evidence:** B4 §4.5.6 introduces identities/sequences; §25.1.7 explains a counter hotspot and why nontransactional allocation can leave gaps after rollback.

**Change:** use replicated per-sequence high-water reservations and fenced allocation generations. A holder may serve only a durably reserved range; unused numbers can be lost. Define cache, overflow, permissions, explicit-value handling, alteration and restore. Generated IDs are not gap-free, timestamps or commit positions. A gap-free business number remains an ordinary serialized application record with its contention cost. No single global identity counter is introduced. E24-S03 is required at local and distributed gates.

### D04 — Give statistics a maintenance lifecycle

**Before:** E05-S03 required row/width/null/distinct estimates and sampled skew, but did not specify who refreshes them or how shard samples combine.

**Evidence:** B4 §16.3 and Note 16.2 cover histograms, frequent values, random sampling and approximate refresh. §25.1.5 connects bad estimates with bad plans.

**Change:** add SQL/JSON analyze jobs, scoped statistics descriptors, weighted collection across ranges, coverage/version tracking and budgeted automatic refresh. Distinct counts are not additive and equal averaging of unequal shards is biased; choose a qualified estimator or resample. Topology changes invalidate coverage assumptions. Missing statistics reduce plan confidence, never row correctness. Samples and histograms can expose values, so access is restricted. E25-S01 owns these operations.

### D05 — Represent distribution as an executable plan property

**Before:** the optimizer included sharding/network costs but exchange requirements and outer-join placement correctness remained broad.

**Evidence:** B4 §§22.3, 22.5.2 and 22.7 show partition/broadcast strategies, exchange operators and the importance of physical distribution properties. An outer row unmatched at one worker may match at another.

**Change:** plans carry ordering, partition keys, colocation, location and placement generation. Insert exchanges for explicit property mismatches. Broadcast only after reserving total receiver memory, and bound the degree of parallelism. A distributed outer join needs a correct owner of unmatched-row decisions. Explain must show transfer/parallelism costs. E25-S02 decomposes the E05 integration gate.

### D06 — Distinguish safe plan reuse from good plan reuse

**Before:** prepared statements, bounded caches and schema invalidation were separate requirements, with no integrated cache key or response to parameter skew.

**Evidence:** B4 §§16.4.3, 16.6.5 and 16.6.6 discuss plan caching, parameter-dependent plans and adaptation. A plan optimized for a rare value may perform poorly for a common one even when logically valid.

**Change:** cache validity includes immutable scope, effective policy, parameter types, object/schema dependencies, relevant semantic settings and placement capabilities. Separately manage cost quality using statistics versions and bounded feedback. Limit variants, compilation work and invalidation traffic; use cooldowns. An adaptive choice never authorizes replaying a partly delivered result or executed write. E25-S03 integrates E05/E07/E08.

### D07 — Treat runtime filtering and query retry as correctness features

**Before:** bounded exchanges existed, but no explicit boundary for dynamic filters or fragment replay.

**Evidence:** B4 §22.9.3.2 describes Bloom-based semijoin reduction; §22.5.4 explains duplicate work/results after worker restart and the cost of materialized retry stages.

**Change:** initial worker failure cancels the query attempt and resolves its transaction; no automatic partial-fragment replay. Safe whole-attempt retry requires retained read context, no escaped effects/results and an explicit replay contract. P2 runtime filters must be complete for their build context, never have false negatives, and preserve join/NULL semantics. Missing/incomplete filters pass through; preserved-side outer joins and NULL-aware anti joins need separate proof. E04-S03, E25-S04 and the execution contract own this distinction.

### D08 — Make bulk import visibility and progress atomicity explicit

**Before:** E05-S04 had bounded COPY/CSV/JSON and E15 handled migration; a general import job had no durable checkpoint definition.

**Evidence:** B4 §25.1.5.3 explains set-oriented bulk work and codec utilities; chs. 17–19 supply transaction/recovery obligations. Splitting a large operation into batches changes its atomicity.

**Change:** keep transaction-scoped COPY atomic within its limits, and introduce a separately named partially committed import job. Its immutable source/parser identity, batch outcome and checkpoint bind to the same commit authority as row effects. A lost acknowledgement is resolved before replay. CSV newlines and compressed streams require real record boundaries. Scope/policy/index/constraint enforcement is shared with ordinary DML. E26-S01/S02 are the implementation owners, E15 remains source-migration correctness owner.

### D09 — Treat bulk index construction and export as durable jobs

**Before:** online index backfill and export existed as high-level outcomes, without an external-sort/file-installation or portable-manifest contract.

**Evidence:** B4 §§14.7, 15.4, 19.3–19.6, 22.2 and 25.1.5.3 connect index costs, sorting, durability and bulk operations.

**Change:** build private native index generations under bounded external sort; capture concurrent writes and validate before publication through the existing DDL state machine. File installation must integrate with native WAL/manifest/consensus durability. Exports bind a consistent cut, logical range coverage and verified chunk manifest; expired pins fail explicitly. Logical exports are not physical backup or PITR. E26-S03/S04 supply the detailed work.

### D10 — Add materialized views with deliberately staged semantics

**Before:** analytics and CDC existed, but there was no managed materialized-view lifecycle or acceptance owner.

**Evidence:** B4 §§4.2.3, 16.5 and 22.7.5 distinguish immediate/deferred maintenance, refresh costs and incremental algebra. Counts and sums need auxiliary state; deleting a current minimum cannot generally be handled by subtracting a value.

**Change:** E27 first provides consistent full refresh into a private generation followed by atomic publication. Then qualify a narrow single-table selection/projection/grouped COUNT/SUM/AVG incremental subset using complete committed changes. Preserve SQL bags, NULL and empty-group rules; unsupported joins/DISTINCT/MIN/MAX maintenance uses full refresh or fails explicitly. Rebuild after lost history. No user trigger language or new storage engine is required.

### D11 — Freshness and authorization are part of derived data

**Before:** RLS and CDC checks did not define who may read an aggregate computed from other users' rows, or whether it can substitute for current data.

**Evidence:** B4 §§4.7.3, 16.5.3 and 23.6.3 establish view authorization, query rewriting and asynchronous derived-state limitations. The concrete Falcone restrictions below are design choices.

**Change:** a published materialized generation identifies a consistent data frontier and schema/policy versions. Initially allow only explicit reads with a declared freshness policy and a common authorized source domain. Principal-dependent RLS requires a proof of equivalent enforcement; otherwise reject materialization. Revocation fences incompatible generations. Automatic substitution into current transactional queries is disabled until separately certified. Asynchronous views never enforce synchronous constraints. E27-S03/S04.

### D12 — Fill correctness gaps without changing the transaction/storage foundations

**Before:** strong strict 2PL, deterministic commands and mutation tests were present, but coarse/fine lock compatibility and particular SQL counterexamples were implicit.

**Evidence:** B4 §18.3 explains intention locks; §23.8.4 requires deterministic state-machine execution; §§16.4.4 and 16.6.3 show bag-preserving subquery rewrites and updates revisiting their own modified index entries.

**Change:** model an explicit lock hierarchy and escalation before using coarse locks, without a central RPC per row. Persist resolved volatile values/identities in commands; test replay under different clocks and binaries. Test NULL-aware anti joins, scalar cardinality, unmatched outer joins and update row revisits. Preserve the existing precise Raft, prepared-state, snapshot-frontier and WAL-prefix contracts. E03/E04/E05/E17/E20 receive targeted additions, rather than competing new protocol epics.

### D13 — Clarify complex data and historical data boundaries

**Before:** SQL JSON, API JSON, MVCC history, backup history and analytical branches could be read as broader compatibility promises.

**Evidence:** B4 §8.1.2 covers JSON representation/path operations; §7.10 distinguishes business-valid intervals and temporal constraints from a current-state table.

**Change:** explicitly distinguish SQL NULL, JSON null and missing paths; freeze duplicate-key, numeric and limit rules for the certified JSON type. REST/JSON administration does not imply full SQL/JSON support. Internal MVCC/PITR does not promise SQL system-time or business-time tables. Temporal overlaps/keys/retention require a later design. Keep graph/RDF/XML, blockchain, unrestricted UDFs and a full analytical engine outside this update.

### D14 — Qualify scale with useful plans and adversarial semantics

**Before:** broad scale-out and failure tests were strong, but the connection between plan quality, parallel resources and semantic counterexamples could be sharper.

**Evidence:** B4 §§20.4.2 and 22.7.2 separate speedup/scale-up and expose startup/skew costs; §25.3.1 motivates small datasets that distinguish correct queries from common wrong variants.

**Change:** measure fixed-work single-query speedup separately from proportional dataset/transaction scale-up, with unchanged durability/RF and reported total resources. Add workload distributions for hot sequences, rare/common parameters, unequal shard sizes and expensive joins. Semantic tests include a targeted counterexample and inspect resulting database state as well as returned rows. E17 owns certification; no new performance claim follows from this review.

## Applicability of every supplied main chapter

| Chapter | Applicable lesson | Falcone destination or deliberate boundary |
| --- | --- | --- |
| 1 Introduction | Engine, users, schemas and operations form one product | Retain relational scope; E01/E05/E09 |
| 2 Introduction to the Relational Model | Keys and relational algebra underpin correctness | E05/E24; JSON transport does not replace relational semantics |
| 3 Introduction to SQL | Bags, NULLs, aggregates, subqueries and updates | E05-S01/S02, E17 semantic counterexamples |
| 4 Intermediate SQL | Views, constraints, identities, schemas and authorization | New E24, supported-subset matrix |
| 5 Advanced SQL | Procedures, triggers, recursion and analytical expressions | E18 demand-driven expansion; no arbitrary code runtime added |
| 6 Database Design Using the E-R Model | Conceptual relationships precede physical placement | E01/E06 schema/colocation preview; no ER editor requirement |
| 7 Relational Database Design | Dependency preservation and temporal meaning | E06 logical/physical distinction, D13 temporal boundary |
| 8 Complex Data Types | JSON and spatial data need explicit operators/encodings | E05 JSON fixtures, E10 geo; graph/XML/RDF deferred |
| 9 Application Development | Prepared access, pooling, security and observability | E07/E08/E14; web framework choice stays outside engine |
| 10 Big Data | Partitioned work and derived pipelines have distinct workloads | E06/E13; no embedded Hadoop/Spark dependency |
| 11 Data Analytics | Precomputation can reduce repeated aggregate work | E27 materialized views; full warehouse/mining engine deferred |
| 12 Physical Storage Systems | Device costs and failure assumptions must be tested | E19–E23 qualification retained; dated hardware constants not adopted |
| 13 Data Storage Structures | Catalog, records and buffers have separate responsibilities | E24 dependencies, existing native format/cache; no second page backend |
| 14 Indexing | Composite/covering indexes, write costs and construction | E05/E26-S03; additional access methods require workload evidence |
| 15 Query Processing | Materialization, sorting and physical algorithms affect budgets | E05/E26 external runs and mutation protection |
| 16 Query Optimization | Statistics, reusable plans, bag-safe rewriting and materialized views | E25/E27 and E05 counterexamples |
| 17 Transactions | Atomicity, recoverability, isolation and failure states | E04/E26 COPY-versus-batch contract |
| 18 Concurrency Control | Intention locks, phantoms, deadlocks and isolation choices | E04 lock hierarchy; retain strong strict 2PL baseline |
| 19 Recovery System | Log ordering and recovery state differ from visibility | Existing E20–E23; do not transplant page-oriented ARIES wholesale |
| 20 Database-System Architectures | Shared-nothing growth, speedup and scale-up limits | E07/E17 measurements and existing three deployment profiles |
| 21 Parallel and Distributed Storage | Attribute skew differs from placement and access skew | E06 range/tenant/hot-key tests; existing native sharding retained |
| 22 Parallel and Distributed Query Processing | Physical properties, exchanges, skew and replay | E25, E04-S03; no promise of general fragment-level retry |
| 23 Parallel and Distributed Transaction Processing | Decisions, replicated state, authority and derived consistency | E03/E04/E20 refinements; quorum loss may still block progress |
| 24 Advanced Indexing Techniques | Bloom/cell/R-tree alternatives have different costs | E25-S04/E10 comparison; no mandatory bitmap/R-tree backend |
| 25 Advanced Application Development | Bulk operations, tuning, benchmarking and migration evidence | E26/E15/E17; safe operational APIs |
| 26 Blockchain Databases | Byzantine/adversarial consensus is a different failure model | Explicitly excluded; no blockchain or smart-contract subsystem |

Appendix A supplies a sample relational schema, not additional product requirements. Online chapters 27–32 are listed by the book but absent from the uploaded body and not included in this coverage claim.

## Ownership, delivery and verification

| New epic | Stories | Relationship to existing work |
| --- | --- | --- |
| E24 | S01 dependencies; S02 logical views; S03 identity; S04 constraints | Implements detail consumed by E03/E05/E08/E09 |
| E25 | S01 statistics; S02 physical properties; S03 plan reuse; S04 runtime filters | Implements detail consumed by E05/E07; S04 is optional M5 |
| E26 | S01 streaming codecs; S02 import jobs; S03 bulk indexes; S04 export manifests | Reuses E04/E09/E12; E05 certifies protocol and E15 migration semantics |
| E27 | S01 full refresh; S02 incremental aggregate; S03 freshness/security; S04 selection/rewrite | Optional M5 capability using E09/E12/E13/E24–E26 |

Required evidence includes dependency and permission races, loss/reuse of sequence reservations, weighted statistics under splitting, parameter skew, globally unmatched outer rows, malformed multiline bulk input, batch checkpoint failures, index-publication crashes, export/reimport equality, and (at M5) view recomputation versus incremental results. See [contracts](relational-operations.md) and the [backlog](../backlog/epics-and-stories.md) for exact acceptance criteria.

The review adds requirements and interfaces, not measured throughput or implemented behavior. Models, actual storage fault tests, semantic/history checks and named Falcone workloads still decide release readiness.
