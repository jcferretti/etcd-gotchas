# Replacing etcd's bbolt backend with grocksdb or Pebble

**Method:** first static and theoretical assessment, 2026-11-01.  
**Versions analyzed:** etcd `v3.7.3` (`0a173d0410a96d01091b484daae81365bbcde57e`), its bbolt dependency `v1.5.0` (`e7a8b2dd498494a3766ba24dd94d3509e5588485`); Pebble `v2.1.7` (`66d468e449bbd52ddf9e53c3a7cdec37e885aaee`); grocksdb `v1.11.1` (`7011ee061b9802a0a82dca28385bcda22c9c7566`) with RocksDB `v11.1.2` (`3b446089141659fad25328c5ea3e7ed283df46e4`). jetcd is not substantively evaluated in this report.  
**Upstream references:** etcd [#22332, storage capacity and alternative engines](https://github.com/etcd-io/etcd/issues/22332), [#22339, proposed storage correctness tests](https://github.com/etcd-io/etcd/pull/22339), [#22425, non-blocking defragmentation](https://github.com/etcd-io/etcd/pull/22425), historical [#13232](https://github.com/etcd-io/etcd/issues/13232), [#13293](https://github.com/etcd-io/etcd/issues/13293), and [#2646](https://github.com/etcd-io/etcd/issues/2646).  
**Summary:**

- **High confidence:** Both engines expose the main primitives needed to implement an etcd backend. Neither is a drop-in replacement. The work includes transaction visibility, recovery metadata, snapshots, restore tools, quotas, and maintenance semantics.
- **Moderate confidence:** Pebble is the better first prototype for a Go team without an existing RocksDB integration advantage. This is a judgment about implementation and operational burden, not a measured performance ranking.
- **Moderate confidence:** Meaningful improvement is plausible if bbolt application, history deletion, fragmentation, or storage traffic is the limiting resource. A backend replacement alone does not remove quorum communication, the Raft WAL, ordered application, the in-memory MVCC index, or watch fan-out. The team's actual bottleneck remains unknown.
- **High confidence:** etcd's physical write workload is already largely revision-ordered and batched. Comparing arbitrary random-key writes in bbolt with an LSM engine would misrepresent the proposed change.
- **Moderate confidence:** Recovery and snapshot integration are the largest correctness risks. Existing etcd failure history makes these concrete engineering concerns rather than hypothetical objections.
- **Low confidence, planning estimate:** A decision-quality prototype is approximately 6–10 engineer-weeks for Pebble or 8–14 for grocksdb/RocksDB; a production candidate is approximately 24–48 or 32–64 engineer-weeks respectively, including the prototype. Assumptions and exclusions are in §9.

No builds, tests, benchmarks, fault injections, or production changes were performed. This report contains no measurements of the team's workload and makes no numerical speedup prediction.

## 1. Recommendation and decision boundary

**Proceed to bottleneck measurement and, if it supports the hypothesis, a Pebble prototype in an etcd fork. Do not yet commit to deploying a replacement backend.** There is enough code-level feasibility and upstream interest to justify that next step; there is insufficient evidence to say the replacement will solve this project's write problem. **Confidence: Moderate.**

The distinction matters because “lots of writes” describes demand, not the resource exhausted by that demand. A workload can be limited by backend application, Raft log synchronization, network latency, serialized transaction evaluation, watches, client concurrency, or combinations of these.

| Observation on the real workload | Implication for this proposal |
|---|---|
| Backend commit/spill/rebalance or its locks consume much of application time; the committed-to-applied backlog grows | Strong reason to prototype an alternative backend. **Moderate confidence.** |
| History compaction or defragmentation causes the unacceptable pauses | Strong reason to investigate an LSM backend, alongside upstream bbolt maintenance improvements. **Moderate confidence.** |
| Raft WAL synchronization is slow while backend work is light | Limited direct benefit; first investigate storage latency and contention. A backend can still help indirectly if it reduces shared-device traffic. **Moderate confidence.** |
| CPU profiles are dominated by protobuf processing, the MVCC index, transaction evaluation, TLS, or watch delivery | The backend swap leaves the dominant work largely intact. **High confidence about scope; Moderate about the resulting benefit.** |
| Large, compressible values dominate storage and snapshot traffic | Compression makes either LSM candidate more interesting, but does not automatically compress etcd's separate Raft log or client/peer request traffic. **Moderate confidence.** |
| Values are already compressed/encrypted and the current working set fits in cache | The compression opportunity shrinks; added LSM and binding overhead may outweigh benefits. **Moderate confidence.** |

“grocksdb” is a Go binding to the C++ RocksDB engine. The operational choice is therefore **grocksdb plus an explicitly selected RocksDB build** versus **Pebble**, a Go engine. Evaluating the wrapper alone is insufficient.

## 2. Scope, version checks, and evidence quality

The local etcd and jetcd tags were fetched and checked against GitHub releases; jetcd was also checked against Maven Central. The existing pins remain the latest stable releases under this project's definition: etcd `v3.7.3` and jetcd `0.8.7`. Neither checkout was moved or edited. Pebble and grocksdb stable tags were checked against remote tags and GitHub's latest-release API. The API returned [Pebble v2.1.7](https://github.com/cockroachdb/pebble/releases/tag/v2.1.7) and [grocksdb v1.11.1](https://github.com/linxGnu/grocksdb/releases/tag/v1.11.1); indexed search results were older.

RocksDB `v11.1.2` is the native version named by grocksdb's release and `build.sh:40 @ v1.11.1`. It is **not** the newest native RocksDB release: GitHub's latest-release API returned `v11.12`. Compatibility of that newer native release with this wrapper was not established. Pinning a wrapper and assuming it supports every later native release would leave an unexamined dependency. [Release pairing](https://github.com/linxGnu/grocksdb/releases/tag/v1.11.1), [build script](https://github.com/linxGnu/grocksdb/blob/v1.11.1/build.sh#L40), [newer native release](https://github.com/meta/rocksdb/releases/tag/v11.12).

Candidate engine and bbolt source clones were downloaded to `/tmp/ftcd-storage-20261101/` for reading. No implementation worktrees were created. Code citations below use the pinned tags; the evidence table in §12 gives full paths and starting lines. **Verified** means inspected source or upstream metadata, not executed behavior. **Inference** and **proposal** identify conclusions or designs that still require validation.

### What upstream work establishes

[Issue #22332](https://github.com/etcd-io/etcd/issues/22332) is open and proposes storage decoupling, standardized tests, and alternative-engine prototypes. Its author reports promising Pebble write/maintenance and compression results, alongside worse reads in some comparisons. The implementations are described as preliminary. This is relevant directional evidence, **Moderate confidence**, not an audited full-cluster performance result or evidence that crash recovery and upgrades are complete. I do not transfer its throughput or compression ratios to this project: versions, complete engine configuration, durability equivalence, and the team's payload distribution have not been established here.

[PR #22339](https://github.com/etcd-io/etcd/pull/22339), still open and unmerged when checked, contains MVCC correctness test changes. Its head is `c80ad786f30a4d7a57ec726780b9470a9863e8d9`. The inspected file list contains two test files, not a complete Pebble backend. It is a useful candidate for later test reuse, not an implementation ready to adopt.

Historical [#13232](https://github.com/etcd-io/etcd/issues/13232) and [#13293](https://github.com/etcd-io/etcd/issues/13293) discuss RocksDB and other alternatives; their closed status does not mean such a backend shipped. The current stable implementation opens bbolt directly. Historical discussion is context, not a current benchmark.

**A relevant change exists on upstream `main`, not the target release:** [#22425](https://github.com/etcd-io/etcd/pull/22425), commit `d6af8028d9cef8363eef323e10af36751c6e60d4`, adds mostly non-blocking defragmentation. The inspected `main` was `1573d59416efa220421c451f5266b24a5d8374ef`; `v3.7.3` still has the blocking implementation. No additional storage/cindex changes appeared in the inspected `v3.7.3..origin/release-3.7` log. If defrag is the primary problem, an upstream solution deserves comparison before accepting a permanent engine fork. That does not establish that the `main` feature is production-ready for this team.

## 3. What the replacement would actually change

### 3.1 The write path has two persistence layers

The following is a dependency diagram, not a literal timing diagram; replication, persistence, and application have pipelining in the implementation.

```text
Client Put / Txn
       |
       v
Raft proposal -> replication + Raft WAL persistence -> committed entry
                                                        |
                                                        v
                                               ordered application
                                                        |
                             transaction evaluation + MVCC index update
                                                        |
                                                        v
                                      backend writes + read visibility
                                                        |
                                             response / watch events

Current backend durability: batched bbolt commits
Candidate backend durability: LSM batches, engine WAL, SST flush/compaction

Raft snapshots / log reclamation require a durable backend boundary.
```

The proposal and waiter are in `server/etcdserver/v3_server.go:1059,1114`; Raft persistence and message ordering are in `server/etcdserver/raft.go:237,255`; the ordered apply loop is in `server/etcdserver/server.go:1892`. The Raft WAL synchronizes in `server/storage/wal/wal.go:869,995`. These paths remain with either engine. [E1–E3](#12-code-evidence-map)

**Verified:** etcd does not require a new bbolt durable commit for every ordinary Put response. Its backend publishes buffered writes to readers and commits on a timer, operation threshold, or a forced boundary. Defaults are **100 ms** and **10,000 pending operations**; these are implementation settings, not predicted request latency or throughput. Backend deletes and some metadata operations cause earlier commits. [E4–E6](#12-code-evidence-map)

This is why replacing “one synchronous B-tree write per request” with “one asynchronous LSM write per request” would be the wrong comparison. The baseline already amortizes backend synchronization and uses the Raft log for recovery of state not yet committed to bbolt. **Confidence: High.**

### 3.2 The on-disk key order is revision order

etcd's MVCC layer—multi-version concurrency control, retaining key history by revision—keeps an in-memory B-tree mapping user keys to revisions. The backend `key` bucket maps an encoded **main revision/subrevision** to a serialized `mvccpb.KeyValue`. Revisions are encoded in big-endian order; a tombstone marker distinguishes deletion records. [E7–E9](#12-code-evidence-map)

Consequences verified from the code:

- Updating random application keys still appends new revision keys to the backend. It does not overwrite the same physical backend key each time.
- `UnsafeSeqPut` sets bbolt's bucket fill fraction to `0.9`, explicitly exploiting append-heavy access.
- A user `DeleteRange` writes MVCC tombstone records. It is **not** directly a backend range deletion.
- A user range read normally resolves revisions through the in-memory index, then fetches individual values by revision. Moving to an LSM does not automatically turn that into a contiguous user-key scan.
- The in-memory index, its locking and memory consumption, and the cost of restoring it remain in a backend-only replacement.

Evidence: `kvstore_txn.go:108,125,223,259,308,323`; `revision.go:23,83`; `index.go:38`; `batch_tx.go:149`; `kvstore.go:327`. [E5, E7–E10](#12-code-evidence-map)

**Inference — Moderate confidence:** etcd's ordered append pattern reduces the usual advantage attributed to LSMs on random overwrites. The more promising opportunities are cheaper history deletion, reduced foreground B-tree work, compression, and background space reclamation. This does not imply bbolt is optimal: revision appends still dirty tree pages, and sparse retained history makes deletion and space reuse substantial work.

### 3.3 Three different operations are called “compaction”

| Operation | What it removes | Effect of changing the backend |
|---|---|---|
| etcd MVCC compaction | Historical revisions no longer required by etcd's retention policy | Still required; the MVCC index decides what must survive. |
| Raft log compaction/reclamation | Log history recoverable from a later durable state/snapshot | Still required; its durability dependency must remain correct. |
| LSM compaction | Reorganizes sorted files and eventually discards superseded engine records and engine tombstones | Newly introduced background work; it does not know etcd's retention policy by itself. |

bbolt **defragmentation** is different again: it rewrites retained data to reclaim file space. In `v3.7.3` it takes the backend write and read locks. [E12–E13](#12-code-evidence-map)

**Do not implement MVCC compaction as `DeleteRange(all revisions <= R)`.** A key last written long before `R` may still be the current live value. `kvstore_compaction.go:30,53` obtains a set of revisions to retain and selectively deletes the rest. Because application revisions are separate physical LSM keys, the engine cannot infer that two such records are versions of the same application key. A native TTL or compaction filter would likewise be unsafe without an independently proved mapping of etcd semantics. **Confidence: High.** [E11](#12-code-evidence-map)

### 3.4 Current costs worth separating

bbolt's ordinary durable transaction path writes dirty pages, synchronizes them, writes a metadata page, and synchronizes again (`tx.go:170,244,267,566,595 @ bbolt v1.5.0`). These calls happen per backend transaction, not per API write. On Linux, etcd sets `NoFreelistSync: true`, so a diagnosis assuming that every commit serializes the freelist would be wrong for this configuration. In-memory freelist work and recovery costs still deserve profiling. [E14–E15](#12-code-evidence-map)

Backend `UnsafeDelete` currently forces a commit at unlock because the read overlay does not buffer deletions. MVCC history compaction uses such deletes and explicitly forces commits between chunks. Ordinary user deletion is usually a tombstone **put**, so it is important to distinguish these paths in a profile. Auth writes also explicitly commit. [E5, E11, E16](#12-code-evidence-map)

**Inference — Moderate confidence:** an LSM adapter with correct delete visibility can remove some bbolt-specific synchronization pressure. A replacement that preserves every forced commit mechanically may leave much of this opportunity unrealized; one that removes them without re-establishing their visibility and durability purpose can corrupt state.

## 4. Feasibility and a conservative implementation design

### 4.1 There is a usable interface, but its contract extends beyond its signatures

`backend.Backend`, `BatchTx`, and `ReadTx` provide a reasonable boundary. Most MVCC and metadata code already uses these interfaces. However, the implementation and configuration are bbolt-specific, and snapshot consumers assume a bbolt database file. [E4–E6, E17–E20](#12-code-evidence-map)

| Required behavior | Candidate implementation | Principal trap |
|---|---|---|
| Buckets and bytewise ordering | Stable bucket-name encoding followed by raw key bytes in one engine database | Existing `Bucket.ID()` values explicitly must not be persisted; prefixes must be unambiguous. |
| Atomic related updates | One engine batch containing the data and associated metadata | Separate metadata writes can leave a recoverable database claiming data that is absent. |
| Reads of preceding writes within a transaction | Pebble indexed batch; RocksDB indexed write batch, or a deliberately implemented overlay | Plain write batches do not provide the needed read view. |
| Concurrent read transaction | A stable engine snapshot acquired in coordination with MVCC revision/index publication | Repeated live `Get` calls can observe different points; a snapshot acquired too early misses a published revision. |
| `ForceCommit` | Publish pending writes and establish a durable engine boundary | Merely inserting into a memtable is insufficient. |
| Snapshot/export | A complete logical export or a packaged, consistent engine checkpoint | An engine snapshot handle is not a durable transferable backup. |
| Hashes, size, maintenance | Explicitly defined adapters and tool support | File checksums, SST size, and manual compaction are not equivalent to existing etcd meanings. |

For the first implementation, use **one database/keyspace**, with all bucket namespaces in it, for either candidate. This simplifies atomicity and lets the same logical encoding exercise both engines. Use an explicitly versioned encoding based on bucket names or a newly defined durable namespace registry—not the current in-memory numeric IDs. Preserve bucket existence and empty-bucket behavior for metadata and full-state hashing. **Proposal; Moderate confidence in this as the simplest starting point.**

RocksDB column families are feasible later, but separate flush/recovery behavior adds proof obligations. Its API distinguishes flushing one family from an atomic multi-family flush. Single-keyspace correctness should be established before optimizing bucket placement. Pebble does not require column families to implement etcd buckets. [E21, E25](#12-code-evidence-map)

### 4.2 Preserve the durable applied-prefix invariant

The **consistent index**, etcd's backend metadata recording the Raft entry already incorporated into the database, determines which log entries are skipped during recovery. It is distinct from the MVCC revision. Several changes may share an MVCC revision, and Raft contains entries that do not advance that revision.

For the recovered consistent index `I`, the recovered database must contain the corresponding complete state, including its metadata, rather than an arbitrary mixture of updates from different application boundaries. At each durable backend boundary, data and consistent-index advancement must agree atomically. The same requirement covers membership/configuration state and mutations outside ordinary KV writes. **Confidence: High.**

Existing code deliberately coordinates the apply lock, in-memory consistent-index update, and precommit hook. The hook saves both the index and dirty Raft configuration state in the backend transaction. `server.go:2105` commits the backend before saving a disk Raft snapshot and releasing older WAL data. [E22–E24, E12](#12-code-evidence-map)

This boundary has failed before: etcd's [v3.5 data-inconsistency postmortem](https://github.com/etcd-io/etcd/blob/v3.7.3/Documentation/postmortems/v3.5-data-inconsistency.md#L18) describes a race that persisted the index before its data, causing recovery to skip changes. It is historical evidence of the failure mechanism, not a claim that `v3.7.3` still has that defect.

A concrete failure a new adapter must prevent:

1. Raft entry `I` updates several records.
2. A periodic commit or checkpoint persists `consistent_index = I` before all of those records are covered by the same recoverable engine state.
3. The process or machine crashes.
4. Restart reads `I`, skips the already “applied” entry, and silently loses its missing updates.

Persisting data without the matching index can also cause incorrect replay, especially for conditional transactions and revision allocation. Replication does not fix an incorrect local replay rule.

### 4.3 Keep the engine WAL for the first prototype

**Recommended starting design, not a proven implementation:** retain etcd's Raft WAL and enable the LSM engine's own WAL. Preserve serialized application. Publish logically complete batches with their corresponding metadata; synchronize the engine at the existing durability boundaries. There are two reasonable implementation paths:

- Keep a correct read overlay over pending backend batches, committing a synchronous engine batch at existing commit boundaries. This stays closer to current batching but retains overlay complexity.
- Apply complete, atomic batches to the engine with its WAL enabled but without synchronizing each batch; use engine snapshots for readers and a synchronized barrier at `ForceCommit` and other required durability boundaries. This can simplify publication but changes how the hooks and consistent index are associated with each engine batch.

The second path needs particular care: every recoverable published batch must carry the appropriate metadata, and a background commit must not read an index for an unfinished application. Preserve the lock ordering and the distinction between application work and background compaction; do not move hooks by analogy alone.

Pebble offers synchronous batch application and a WAL synchronization operation through `LogData(nil, Sync)`; its checkpoint options explicitly use that barrier. RocksDB supports atomic `Write` and synchronized writes; an empty synchronized write is documented to synchronize the WAL. The grocksdb binding also exposes `FlushWAL(true)`. The exact adapter barrier must serialize against writers and be verified against rollover, flush, close, and recovery. [E26–E29](#12-code-evidence-map)

**Why not disable the engine WAL immediately?** Raft can theoretically supply redo, but that does not automatically make arbitrary partially flushed engine state a valid Raft application prefix. The adapter would need to prove durable-prefix recovery across memtable flushes, metadata, engine manifests, checkpoint creation, and Raft log reclamation. A crash after the Raft log is discarded and before the replacement state is durable can lose acknowledged writes. Pebble's `DisableWAL` documentation explicitly warns about crash recovery. **Confidence: High about the hazard; Moderate that eliminating this log could eventually be safe under a separately designed protocol.** [E12, E27, E29](#12-code-evidence-map)

Conversely, synchronizing the engine on **every API write** adds a second foreground durability operation that the current backend often amortizes. That is a useful conservative correctness reference, but an unfair sole performance comparison. Report both the durability policy and its batching in every result.

### 4.4 Preserve visibility, lifetimes, and failure handling

These are required implementation properties, not optional optimizations:

- Coordinate engine snapshot creation with the MVCC `revMu` publication boundary in `kvstore_txn.go:209`. An engine snapshot is not itself an etcd revision.
- Preserve reads within write transactions, ordered ranges, limits, empty-value versus missing-key distinction, metadata overwrites, bucket deletion, and iterator error propagation.
- Copy returned bytes or retain their owners for the promised transaction lifetime. Pebble `Get` returns a closer; closing it invalidates the borrowed value. grocksdb values and iterators carry native resource lifetimes. Both can produce use-after-release errors through a careless adapter. [E30–E32](#12-code-evidence-map)
- Treat failure while applying a committed Raft mutation as a state-machine failure. Do not skip the mutation and continue applying later entries because an engine write stalled or returned an I/O error. Existing bbolt mutation/commit failures are fatal. Design engine background-error handling with the same integrity objective.
- Keep advanced ingestion/excision and native TTL/filter behavior out of the first adapter. Pebble explicitly documents an excision exception to ordinary snapshot isolation; these features are unnecessary for the baseline Set/Delete/range implementation. [E30](#12-code-evidence-map)

Pebble's lack of a general SQL-style transaction facility is not a blocker. etcd already evaluates and orders its transactions; the backend needs atomic batches, stable reads, and the correct surrounding synchronization. The same reasoning means RocksDB `TransactionDB` is not required for the initial design. **Confidence: High about the required primitives; actual adapter correctness remains untested.**

## 5. Pebble versus grocksdb/RocksDB

| Dimension | Pebble `v2.1.7` | grocksdb `v1.11.1` + RocksDB `v11.1.2` |
|---|---|---|
| Basic fit | Atomic batches, indexed batches, snapshots, ordered iterators, checkpoints, compression | Equivalent core capabilities exposed through Go bindings |
| Language/build | Go engine with a supported non-cgo path | cgo plus a native RocksDB/C++ and compression-library build |
| Lifetime management | Closers and explicit snapshot/iterator lifetime; still requires discipline | Native options, batches, slices, iterators, snapshots, and database handles require explicit cleanup |
| Instrumentation | Go-side metrics/events and more direct integration with Go profiling | Must combine Go, wrapper, native engine, allocator, and OS measurements |
| Tuning scope | Subset focused on CockroachDB's needs | Broader native feature set; wrapper coverage must be checked per feature |
| Deployment burden | Smaller native dependency burden; non-cgo build can simplify cross-compilation | Reproducible native build, compiler/ABI and linking choices, native security updates, platform testing |
| Reuse advantage | Strong default for an etcd/Go team | Stronger if the team already operates, tunes, packages, and debugs RocksDB |
| Expected overall integration risk | Lower of these two, **Moderate confidence** | Higher unless native expertise already exists, **Moderate confidence** |
| Expected throughput winner | **Unknown** | **Unknown** |

Pebble's release notes describe production use in CockroachDB; this supports the maturity of the engine, not of a hypothetical etcd adapter. Its scope and APIs are documented in the pinned source. **Confidence: High that upstream makes this production-use statement; Moderate when transferring that experience to this integration.** [Pebble release](https://github.com/cockroachdb/pebble/releases/tag/v2.1.7), [E25–E27](#12-code-evidence-map)

Calling Pebble “pure Go” needs qualification: the inspected version has both cgo and non-cgo implementations for allocation and Zstandard compression. It can avoid a native RocksDB dependency, but a build must explicitly record `CGO_ENABLED` and relevant build tags. Comparing native-compression and Go-compression builds without recording that difference would confound the result. [E33](#12-code-evidence-map)

grocksdb's `Put` into a write batch itself crosses cgo; one final batch write does not reduce the entire operation to one language-boundary crossing. Its indexed batches provide merged reads and iterators, so functionality is available. **Inference — Moderate confidence:** per-operation wrapper overhead is more consequential for tiny, cache-resident records than for large writes dominated by storage. No overhead figure or end-to-end penalty is established here. [E31–E32, E34](#12-code-evidence-map)

The wrapper README lists missing portions of the native C API, including several event/background-error callbacks. That does not establish an unusable engine, but observability and error handling may require wrapper additions. Treat RocksDB and grocksdb as two maintained dependencies, and budget accordingly. [grocksdb README:68 @ v1.11.1](https://github.com/linxGnu/grocksdb/blob/v1.11.1/README.md#L68)

Pebble v2 does not directly open modern RocksDB databases; nor does either engine open a bbolt database. Do not plan an engine switch or rollback around assumed on-disk compatibility. Pebble format-major upgrades can also be irreversible. [Pebble README:87,123 @ v2.1.7](https://github.com/cockroachdb/pebble/blob/v2.1.7/README.md#L87)

## 6. Theoretical performance impact

### 6.1 Where an LSM can help

An LSM engine accumulates changes in memory, writes sorted-table files, and merges them in background compaction. This moves some structural work away from the foreground path; it does not eliminate that work.

| Potential improvement | Mechanism and limiting condition | Confidence |
|---|---|---|
| Lower foreground backend write cost | Memtable/batch insertion replaces some bbolt page manipulation. Benefit shrinks when Raft, index updates, or serialization dominate. | Moderate |
| Less disruptive MVCC history deletion | Engine delete tombstones can defer physical cleanup. Benefit fails if accumulated cleanup cannot keep pace or makes reads too expensive. | Moderate |
| Lower physical footprint and snapshot bytes | SST compression can reduce compressible payload storage. Native checkpoint transfer can preserve that compression. Already compressed values and bbolt-format export reduce the opportunity. | Moderate |
| Less scheduled stop-the-world space reclamation | Automatic file reclamation can replace the normal bbolt defrag workflow. Long-lived snapshots, compaction debt, and disk pressure can still delay reclamation. | Moderate |
| Lower indirect Raft WAL latency | Lower competing device traffic may reduce WAL queueing. Higher LSM compaction traffic can reverse the effect. | Moderate |

### 6.2 Costs and regressions to expect as possibilities

- **More write paths:** the conservative design writes etcd's Raft WAL, an engine WAL, flushed SSTs, and compaction outputs. bbolt already writes the Raft WAL plus backend pages; count total bytes and synchronization cost rather than merely counting files. **High confidence about the paths; net amplification is unknown.**
- **Write stalls:** memtable or level-0 backlog can stop foreground writes. Pebble exposes explicit stop thresholds; RocksDB documents stalls for flush/compaction backlog. More background threads can also steal CPU/I/O from Raft. **High confidence about the mechanism; Moderate about production impact.** [E35](#12-code-evidence-map), [RocksDB write stalls](https://github.com/meta/rocksdb/wiki/Write-Stalls).
- **Point/range read regression:** searching engine levels, decoding blocks, and decompressing values may cost more than a hot bbolt read. etcd's many value fetches by revision make mixed read/write and range tests essential. Bloom filters do not eliminate decoding of successfully found values. **Moderate confidence.**
- **Watch recovery regression:** live watch fan-out mostly remains above the backend, while lagging watchers read revision history from it. Faster Puts do not establish faster watch catch-up. **High confidence about the paths; direction unknown.** `watchable_store.go:346,426` and §3.2. [E36](#12-code-evidence-map)
- **Memory pressure:** etcd's index remains while engine memtables, caches, filters, pending batches, and pinned snapshots consume additional memory. Cache choices and cgo allocation determine which costs appear in Go heap profiles. Compare process RSS and cgroup accounting as well as heap size. **Moderate confidence about increased pressure; magnitude unknown.**
- **Burst illusion:** a short run can finish before flush/compaction debt is paid. The relevant capacity is the sustainable arrival rate at the required latency, with bounded memory, disk use, and backlog. **High confidence as an experimental requirement.**

**An additional layout hypothesis — Moderate confidence:** a benchmark containing only increasing revision keys can also make the LSM candidate look too favorable. Real backend batches mix those appends with updates to fixed metadata keys, such as `consistent_index`, and compaction deletes older revision keys. In a single keyspace, that mixture can broaden sorted-file key ranges and increase overlap and later compaction work. The direction and magnitude depend on flush/file boundaries and engine policy; they must be measured. Include the metadata and deletion stream in the adapter benchmark. Do not split metadata into an independently committed database merely to obtain cleaner append-only measurements: that would sacrifice the atomicity requirement in §4.2. [E9, E11, E22–E23](#12-code-evidence-map), [RocksDB tuning guide](https://github.com/meta/rocksdb/wiki/RocksDB-Tuning-Guide).

For an LSM member, account for approximately:

```text
device writes = Raft-WAL writes + engine-WAL writes
              + SST flush writes + compaction writes + snapshot/export writes
```

Measure these components separately where possible and reconcile them with device counters. Physical space amplification, bytes-written amplification, and compression ratio are different quantities. A small database file does not prove low device wear or low write latency. [RocksDB tuning guide](https://github.com/meta/rocksdb/wiki/RocksDB-Tuning-Guide)

### 6.3 A useful upper-bound model, not a forecast

If the backend accounts for fraction `f` of serial service time and its service time improves by factor `s`, with all other work unchanged:

```text
overall service-time speedup = 1 / ((1 - f) + f/s)
```

For illustration, `f = 0.20, s = 4` gives about `1.18×`; `f = 0.70, s = 4` gives about `2.11×`. These are arithmetic examples, **not measurements or forecasts**. The model is only an approximation: stages overlap, shared-device contention changes, and queueing near saturation can amplify tail-latency effects. **Confidence: High in the arithmetic; Moderate in its use for initial reasoning.**

A more useful capacity model is the minimum sustainable rate across quorum/WAL, ordered apply, backend maintenance, network, and client resources. A faster backend only moves the limit if it is the bottleneck or materially frees resources used by that bottleneck. Adding LSM write concurrency cannot independently parallelize etcd's ordered application loop. [E1–E3, E7](#12-code-evidence-map)

### 6.4 Lower-cost comparisons worth retaining

Before attributing all gain to a new engine, compare against an appropriately configured bbolt baseline:

- Suitable production storage and, where relevant, separation of Raft WAL and backend I/O contention.
- Backend batch interval/limit tradeoffs, measured with recovery and tail latency; do not disable fsync for the production comparison.
- History-retention and maintenance schedules consistent with the application's watch/replay requirements.
- Avoiding redundant writes and batching operations where application semantics permit it. A transaction changes revision grouping and failure semantics; it is not a free transport optimization.
- Compression at the application's value boundary if payloads allow it. Unlike SST-only compression, this can also reduce client and Raft payloads, but requires a compatible value encoding and coordinated readers.
- The upstream non-blocking defrag work if maintenance pauses, rather than sustained write cost, are the main problem.

These are candidate comparisons, not changes authorized or performed in this pass. Sharding into multiple etcd clusters could increase aggregate capacity, but changes cross-shard transaction and revision semantics; it belongs to a separate architectural decision.

## 7. Operational risks that make this more than an adapter

### 7.1 Snapshots, restore, and member catch-up

There are three distinct concepts: an engine read snapshot, an etcd/Raft recovery boundary, and a portable backup or peer snapshot stream. They must refer to compatible state, but they are not interchangeable.

Current code exports a bbolt read transaction, receives snapshot data as a synchronized single file, and renames that file into the backend location. `etcdutl snapshot status` opens bbolt directly; restore also manipulates logical metadata and revisions. The maintenance stream adds an integrity digest. [E4, E17–E20](#12-code-evidence-map)

Both candidates have checkpoints, but an engine checkpoint is a collection of files, including manifests and potentially WALs. It is not automatically a stream in etcd's expected format. For either engine the implementation must bind the checkpoint to the correct applied state, preserve all metadata, package it, verify it, durably install it, and recover safely from interruption. Directory and manifest synchronization matter as well as file synchronization. [E28, E37](#12-code-evidence-map)

Two designs are credible:

| Design | Advantage | Cost / limitation |
|---|---|---|
| Export a consistent **bbolt-format** snapshot from the LSM's logical records; import bbolt into the selected local engine | Can preserve existing snapshot tooling and provide a bridge to stock peers | Requires export/import code, temporary space, full-data processing, and identity/metadata fidelity; gives up native compressed-transfer advantages. |
| Versioned **native checkpoint archive** with engine and format identifiers | Preserves engine files and their compression; potentially less conversion work | Requires fork-aware peers, restore/status tooling, format negotiation or explicit rejection, and an upgrade/rollback plan. |

**Recommendation — Moderate confidence:** choose this format policy before implementing the full backend. An offline-conversion prototype with fork-only snapshots has less compatibility scope; a rolling production rollout may justify the bbolt bridge. Native checkpoint creation and a later stream must also have a defined byte count and cleanup lifecycle to satisfy the current snapshot interface.

### 7.2 Mixed-engine clusters are not automatically safe

Raft entries and the KV API need not change, so bbolt and LSM members can in principle apply the same logical commands. This does **not** establish rolling compatibility. A temporarily healthy mixed cluster might fail only when a lagging stock member needs a snapshot that it cannot open, or when hash/metadata semantics differ. **Confidence: High about the snapshot incompatibility; Moderate about feasible mixed-engine operation after explicit bridging.** [E17–E20](#12-code-evidence-map)

Start validation with homogeneous clusters. Only claim mixed-engine support after testing peer snapshot transfer in both directions, restart, membership changes, hashes at a common revision, maintenance, and downgrade rejection. Do not rely on “our follower will never fall far enough behind to need a snapshot.”

### 7.3 Migration and rollback

A safe offline converter must preserve retained MVCC records, tombstones, compaction metadata, leases, auth, alarms, membership, storage version, and the consistent-index/term relationship. A client `Get` followed by `Put` into a fresh cluster is **not** an equivalent conversion: it loses revision/history and other state. All bucket types are enumerated in `schema/bucket.go:58`. [E9, E22–E24](#12-code-evidence-map)

Suggested initial migration policy:

1. Use a stopped member or a validated consistent snapshot as the source. Decide explicitly whether restoring a new cluster or converting an existing member; their identity rules differ.
2. Convert to a separate, version-marked engine directory. Validate decoded records and canonical hashes, metadata, and recovery behavior before cutover.
3. Publish the new storage selection with crash-safe staging; leave the original source intact until validation and the rollback policy permit removal.
4. Rehearse recovery using the fork's exact binaries and tools. Include interrupted conversion, snapshot installation, and version mismatch.

An old bbolt file becomes stale as soon as the new backend accepts writes. Returning to it is not lossless rollback. Define rollback as validated reverse conversion or an explicitly accepted restore/data-loss boundary. Lease lifetime during a maintenance outage also requires an explicit policy; byte-for-byte conversion does not freeze wall-clock time.

### 7.4 Quotas, hashes, and maintenance contracts

The current quota checks `Backend.Size()` plus an estimated request cost. The default is 2 GiB; 8 GiB is a suggested maximum that produces a warning when exceeded, not an absolute engine-addressing limit. [E38](#12-code-evidence-map)

An LSM adapter needs separate definitions and metrics for logical retained bytes, live SST bytes, WAL bytes, obsolete/pinned files, checkpoint space, and temporary compaction headroom. An estimate of logical bytes alone cannot prevent disk-full during compaction. Preserve the configured quota's documented meaning or version the behavior explicitly; compression is not a reason to silently relax protection.

Preserve etcd's canonical hash inputs and filtering. Hash decoded logical records in the required order, not SST file bytes: replicas can have different compaction layouts while storing the same state. `HashKV` at a common revision is useful but does not validate all auth/lease/membership state, and does not replace operation-history checking. [E4, E39](#12-code-evidence-map)

Define `Defragment` for each engine. It might request bounded manual compaction/reclamation or be explicitly unnecessary, but silently mapping it to an unlimited full compaction could create a new I/O incident. Snapshot pinning can prevent immediate space recovery even after a maintenance call succeeds.

### 7.5 Risk register

Severity describes consequences; confidence describes how strongly this pass supports the assessment. Occurrence probabilities have not been measured.

| Risk | Consequence | Severity / confidence | Required evidence before production |
|---|---|---|---|
| Consistent index and data disagree after crash | Lost or replayed mutations; replica divergence | Critical / High | Crash-prefix tests across apply, commit, flush, snapshot, and log reclamation |
| Read snapshot and MVCC publication disagree | Stale/missing reads or broken transaction behavior | Critical / High | Concurrent read/write and history validation |
| MVCC compaction translated incorrectly | Live data or required watch history deleted | Critical / High | Differential tests with sparse old live revisions and delete/recreate histories |
| Snapshot format/install mismatch | Member cannot rejoin or backups cannot restore | Critical / High | Bidirectional transfer where supported, disaster recovery, interrupted installs |
| LSM maintenance falls behind | Tail spikes, disk growth, write stalls, possible availability loss | High / Moderate | Sustained aged workloads and resource-limited fault tests |
| Native lifetime/build error with grocksdb | Leaks, process crash, corruption, deployment inconsistency | High / Moderate | Native memory/error tooling and platform matrix |
| Fork drifts from etcd fixes | Security/reliability patches arrive late or conflict | High / Moderate | Named owners, continuous rebases, independent release qualification |

## 8. Experiments to run later on representative hardware

These are designs only. Small experiments should first eliminate bad hypotheses; expensive fault and soak testing follows only for a candidate that survives those gates.

### 8.1 Stage A: locate the bottleneck before modifying etcd

Collect synchronized metrics and profiles during a representative write-pressure window, including one maintenance cycle if maintenance is involved. Use the team's current server version as an additional baseline if it differs from `v3.7.3`.

Record per member:

- `etcd_disk_wal_fsync_duration_seconds`, `etcd_disk_backend_commit_duration_seconds`, and the backend rebalance/spill/write histograms when exposed.
- Committed/applied progress, proposals pending/failed, leader changes, peer RTT and traffic, application RPC latency, watch lag, compaction and snapshot durations.
- CPU and allocation profiles, mutex/block profiles where appropriate, process RSS, cgroup pressure, and disk throughput/latency/utilization.
- Database allocated/in-use size, live keys, retained history, churn, payload sizes and compressibility, watch counts, transactions per second **and mutations/bytes per second**.

Metric definitions are in `backend/metrics.go:20` and `etcdserver/metrics.go:75`. A slow commit histogram alone does not establish its fraction of request time; correlate counts, time series, locks, and profiles. A slow WAL can be the victim of backend I/O, so inspect device contention before classifying it as unrelated. [E40](#12-code-evidence-map)

**Decision:** if the critical path and constrained resources have little backend involvement, stop the engine project and address the observed bottleneck. If backend costs are material, use these observations to set the candidate's target.

### 8.2 Stage B: isolate backend/MVCC behavior

Extend existing etcd benchmarks before creating a separate harness. The pinned tree contains `BenchmarkBackendPut`, `BenchmarkStorePut`, `BenchmarkStorePutUpdate`, `BenchmarkStoreTxnPut`, store restore benchmarks, and watchable-store benchmarks. `BenchmarkBackendPut` uses random physical keys; it is diagnostic but not representative of revision appends. Several existing benchmarks also reuse values or omit long-term maintenance. [E41](#12-code-evidence-map)

A small adapter comparison should retain the same MVCC layer, serialized writer, bucket encoding, durability boundaries, and total memory budget. Add only the selector and workload coverage the existing tools lack. Run:

1. Revision-ordered appends at the observed payload sizes, both new application keys and overwrites.
2. A bounded active key set with retained historical revisions and repeated MVCC compaction, including sparse old live values that must survive.
3. Concurrent point/range readers and long-lived snapshots, plus slow watch catch-up.
4. Restart/index reconstruction and snapshot/export/import after an aged workload.

Compare compression off first, then explicitly selected compression settings with representative payloads. Measure engine foreground cost, total device writes, retained bytes, peak space, compaction debt, memory, and completion time after outstanding durable work is drained. Include the engine's WAL and the same persistence guarantees in every compared mode.

The upstream correctness PR in §2 may supply useful model checking, but its unmerged status and test-only scope must be recorded. Passing in-memory/MVCC correctness alone is not evidence of power-loss safety.

### 8.3 Stage C: full-cluster comparison

Use separate homogeneous bbolt, Pebble, and—if still justified—RocksDB clusters with identical machines, network, resource limits, TLS/auth settings, Raft settings, retention, and client placement. A three-voter cluster is a reasonable initial test if production has three voters; otherwise match production. A single-member cluster is only an isolation experiment.

Start with the real workload and a few contrasts rather than a huge synthetic matrix:

| Variable | Initial cases |
|---|---|
| Key behavior | Mostly new keys; bounded hot-key overwrites; actual production mix |
| Payload | Actual distribution; small records; larger records; compressible and distinct incompressible values |
| Reads | Write-only isolation; production mix; expensive range/list operations |
| Transactions | Single Put; representative conditional/multi-operation transactions |
| Watches | None; production fan-out; lagging/reconnecting consumers |
| History | Production retention; a controlled longer-retention contrast |
| Maintenance | Normal operation; compaction; snapshot; bbolt defrag / documented LSM maintenance |
| Load | Below target; target; capacity sweep to SLO failure |

Do not interpret more Put QPS as success if reads, watch lag, follower catch-up, memory, or recovery violate their requirements. All replicas perform application work; a healthy leader alone is not a sufficient result.

**Proposed command template, not executed.** These flags exist in `tools/benchmark/cmd/put.go:60 @ v3.7.3`. The concrete values below are starting test parameters, not assumptions about production. Set the binary, endpoints, offered rate, and output path on the test platform; include its actual hardware/configuration manifest with the result.

```bash
"$BENCHMARK_BIN" \
  --endpoints="$ETCD_ENDPOINTS" --clients=64 --conns=4 \
  --precise --sample \
  put --key-size=32 --val-size=1024 --key-space-size=100000 \
  --total=1000000 --rate="$OFFERED_PUTS_PER_SECOND" \
  > "$RESULT_FILE"
```

Use either the production auto-compaction configuration or the tool's compaction flags, not both accidentally. Increase duration/repeat count until the system has traversed representative retention and engine-compaction cycles and resource/backlog trends stabilize; a fixed operation count is not proof of steady state. Repeat sufficiently to expose variability, report all runs and failure cases, and randomize candidate order where feasible.

**Two verified tool limitations affect this decision:**

1. `put.go:86` generates one value and reuses it for every operation; `txn_put.go:76` does likewise. Even if that value was initially random, repetition across records can be highly compressible. Such a test cannot establish the production compression benefit. Add seeded, distinct per-record data and a representative payload corpus, or replay the application's actual value-generation distribution.
2. `put.go:97–101` waits on the rate limiter and then starts the latency timer around the RPC. Its fixed worker pool does not measure delay from an independent intended arrival time when overloaded. For SLO validation, record intended-arrival-to-completion latency or use a suitably instrumented load driver, along with errors and achieved rate. Otherwise queueing outside the measured RPC can be hidden.

These limitations justify a small extension to existing tooling, not an unrelated benchmark framework. If the application uses jetcd, repeat the decisive scenario with its actual client behavior after establishing server limits with the Go tool. [E42](#12-code-evidence-map)

### 8.4 Stage D: correctness and failure experiments

Before trusting performance results as deployable, extend etcd's existing `tests/robustness` and relevant integration/e2e tests. Add engine-boundary failpoints where the existing bbolt points no longer reach the dangerous code. Reuse operation-history and watch validators; hash equality alone is too weak. [E43](#12-code-evidence-map)

Required targeted cases:

- Crash before/after batch publication and durable synchronization; restart with asynchronous engine writes pending.
- Crash while MVCC compaction deletes history or advances compaction metadata; include the latest record being a tombstone.
- Crash during native flush, manifest update, checkpoint creation, snapshot receive/install, and just before/after Raft log reclamation.
- Restart all members, not only one member that can hide damage by rejoining healthy peers. Verify successful acknowledged mutations, transaction outcomes, and monotonic revision/history behavior.
- Slow/failing storage, disk full, memory pressure, delayed compaction, long snapshot readers, and follower catch-up under continued load.
- Auth changes, leases and expiry, alarm recovery, membership changes, backup restore, engine-version rejection, and the proposed migration/rollback path.
- If rolling compatibility is claimed, both directions of snapshot exchange and catch-up across engine types.

`SIGKILL` tests process failure but leaves the host page cache alive. A durability claim also needs a harness capable of losing unsynchronized writes or representative VM/machine crash behavior; use an isolated test environment. Separate these failure models in the results.

### 8.5 Reproducibility and acceptance gates

Every run must record: etcd and adapter commits; engine versions; grocksdb/native RocksDB pairing; compiler/toolchain, cgo/build tags and linked libraries; member count; CPU/RAM and limits; kernel/filesystem/storage stack and disk durability behavior; WAL/data placement; network topology; all server/engine options; client and exact command; key/value distributions and compression properties; concurrency/arrival model; retention, watches, maintenance, and run duration. No result from this analysis host is proposed as a performance finding.

Choose “meaningful improvement” before comparing engines. A possible investment gate is **at least 50% more sustained successful mutations/second at the same agreed p99 latency and resource budget**, or elimination of a demonstrated maintenance/recovery failure that matters more than raw throughput. This is a proposed business threshold, not a forecast. A smaller gain may still be worthwhile if its operating cost is lower; a larger gain fails if correctness, read/watch SLOs, recovery time, or space/memory limits fail.

At minimum require:

- No correctness violations in the agreed fault/history suite; failures block release regardless of throughput.
- Stable memory, disk use, compaction debt, and follower backlog under sustained target load.
- Read, watch, snapshot, restart, and restore results within explicit SLOs.
- A rehearsed migration and recovery/rollback policy.
- Benefit reproduced with the team's actual payloads and client behavior, not only repeated-value synthetic Puts.

## 9. Effort and staffing

These are judgment-based planning ranges, **Low confidence**, not estimates derived from an implemented patch. One engineer-week is one experienced engineer's focused full-time work for a week. Ranges are cumulative from the start of implementation and exclude waiting for representative hardware or organizational approval.

| Milestone | Pebble | grocksdb/RocksDB | What the milestone means |
|---|---:|---:|---|
| Functional prototype | 3–6 engineer-weeks | 4–8 engineer-weeks | Basic backend adapter and small homogeneous-cluster operation; no production claim |
| Decision-quality prototype | 6–10 engineer-weeks | 8–14 engineer-weeks | Representative performance, initial crash/recovery tests, and explicit remaining gaps |
| Production candidate | 24–48 engineer-weeks | 32–64 engineer-weeks | Complete recovery/backup/maintenance/tooling, migration policy, fault coverage, platform CI, and release preparation |

Assumptions: retain etcd's MVCC schema and Raft WAL; implement one alternative engine first; begin with one Linux deployment platform and homogeneous clusters; permit a planned maintenance window for initial migration; have experienced Go/storage engineers, access to etcd internals, and part-time SRE plus independent correctness review.

For a team of two such engineers, allow roughly **6–12 elapsed months for Pebble** or **8–15 months for RocksDB**, including integration, review, and soak time. **Confidence: Low.** Work is not perfectly parallelizable, and passing a crash/recovery milestone cannot be guaranteed on a calendar. These are budget envelopes, not promised release dates.

The production work divides into:

- Backend contract/encoding and snapshot-consistent read/write implementation.
- Consistent-index, commit, restart, and Raft reclamation correctness.
- Snapshot formats, status/restore/export/import, hashes, quotas, maintenance, and metrics.
- Differential, robustness, crash, sustained-load, and recovery testing.
- Packaging, rollout, rollback, documentation, and ongoing upstream integration.

grocksdb adds native dependency packaging, explicit resource/error auditing, mixed Go/native diagnostics, and possibly wrapper work. Existing RocksDB expertise could substantially narrow the difference. Conversely, a zero-downtime stock-to-fork rollout, many supported platforms, changing the MVCC key layout, or eliminating the engine WAL can push either project beyond these ranges.

No numerical estimate for continuing maintenance is defensible from this pass. The fork needs named owners after launch: etcd security/backport intake, engine format upgrades, regression qualification, incident diagnosis, and release artifacts remain permanent responsibilities.

## 10. Assessment of the fork plan

**A fork is a practical vehicle, provided it is treated as a maintained storage product rather than a one-time library substitution. Confidence: Moderate.** The public client API can largely remain unchanged under the proposed scope; on-disk and maintenance compatibility require deliberate work.

Recommended implementation sequence if later measurements justify it:

1. Fork from the selected stable etcd release and preserve the bbolt implementation as an explicit baseline/fallback **for compatible data directories**. A fallback flag cannot make an LSM directory readable by bbolt.
2. Make engine selection, format identification, factory construction, and error/metric boundaries explicit. Refactor the shared contract before scattering engine conditionals through MVCC.
3. Implement Pebble behind that boundary, keeping revision encoding, metadata semantics, and Raft storage unchanged. Keep optimization patches separate from the initial behavior-preserving integration.
4. Choose and implement snapshot and migration compatibility before claiming operational completeness. Reject unsupported formats at startup.
5. Establish continuous upstream rebase and test jobs; retain regression coverage for bbolt as well as the new engine. Qualify every etcd/engine upgrade together.
6. Consider a RocksDB implementation if Pebble fails a relevant requirement or measurements justify the extra native integration cost. Do not incur two production backend maintenance burdens merely to obtain the first comparison.

Upstream #22332 means some decoupling/testing work may become reusable; re-check it before building competing infrastructure. Upstream adoption is not assured. No repository was forked, pushed, or proposed for upstream contribution in this pass.

## 11. Unresolved inputs and conditions that could change the recommendation

The following remain unknown; the report does not substitute invented workload assumptions:

- Current production etcd version, member count, topology, hardware/storage stack, and resource limits.
- Required throughput and p99/p99.9 latency; whether the problem is steady load, bursts, or maintenance pauses.
- Key/value sizes and compressibility; active data and retained history; hot-key churn, deletes, lease activity, transaction shape, and watch fan-out/catch-up.
- Profiles and metrics identifying the constrained resource; whether client concurrency/retries contribute.
- Backup/recovery objectives, permissible maintenance window, stock-tool and mixed-version compatibility requirements.
- Team expertise and ownership capacity for Go storage internals versus native RocksDB.

The Pebble-first recommendation weakens if the team already has a well-supported RocksDB platform or needs a proven native feature absent from Pebble. The entire engine-replacement recommendation weakens if Raft/network/client/application work dominates, or if a narrow bbolt/upstream fix meets the SLO. It strengthens if representative measurements show costly backend maintenance and compression opportunities with adequate CPU/I/O headroom, and the team can fund recovery correctness and long-term ownership. **Confidence: Moderate in these conditional judgments.**

## 12. Code evidence map

Each row identifies inspected code at a pinned tag. Links point to the starting line; neighboring lines implement the rest of the described behavior. `etcd` means `v3.7.3 @ 0a173d0410a9`; other tags/commits are stated in the header. This map also makes claims independently reviewable without the scratch clones.

| ID | Verified source and purpose |
|---|---|
| E1 | etcd [`server/etcdserver/v3_server.go:1059`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/v3_server.go#L1059), proposal/waiter; [`server/etcdserver/server.go:1892`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/server.go#L1892), ordered application and replay filtering |
| E2 | etcd [`server/etcdserver/raft.go:237`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/raft.go#L237), replication/persistence ordering |
| E3 | etcd [`server/storage/wal/wal.go:869`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/wal/wal.go#L869), synchronization; [`:995`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/wal/wal.go#L995), Save/MustSync |
| E4 | etcd [`server/storage/backend/backend.go:34`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/backend.go#L34), defaults/interfaces; [`:195`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/backend.go#L195), bbolt open; [`:354`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/backend.go#L354), ForceCommit/snapshot/hash; [`:445`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/backend.go#L445), timer commits |
| E5 | etcd [`server/storage/backend/batch_tx.go:31`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/batch_tx.go#L31), buckets/interfaces; [`:149`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/batch_tx.go#L149), sequential fill; [`:308`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/batch_tx.go#L308), publication and delete commit; [`:361`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/batch_tx.go#L361), precommit hook |
| E6 | etcd [`server/storage/backend/read_tx.go:28`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/read_tx.go#L28), read interface and overlay; [`server/storage/backend/backend.go:280`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/backend.go#L280), concurrent read view |
| E7 | etcd [`server/storage/mvcc/kvstore_txn.go:46`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore_txn.go#L46), read/revision coordination; [`:108`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore_txn.go#L108), revision fetches; [`:174`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore_txn.go#L174), write locking; [`:209`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore_txn.go#L209), publication; [`:223`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore_txn.go#L223), put; [`:308`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore_txn.go#L308), delete |
| E8 | etcd [`server/storage/mvcc/revision.go:23`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/revision.go#L23), encoding; [`server/storage/mvcc/index.go:38`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/index.go#L38), in-memory B-tree |
| E9 | etcd [`server/storage/schema/bucket.go:24`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/schema/bucket.go#L24), namespaces and metadata |
| E10 | etcd [`server/storage/mvcc/kvstore.go:327`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore.go#L327), restore/index reconstruction and compaction recovery |
| E11 | etcd [`server/storage/mvcc/kvstore_compaction.go:28`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore_compaction.go#L28), keep set, selective deletion, forced commits |
| E12 | etcd [`server/etcdserver/server.go:2083`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/server.go#L2083), durable backend before disk snapshot/release; [`:2139`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/server.go#L2139), Raft log compaction |
| E13 | etcd [`server/storage/backend/backend.go:476`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/backend.go#L476), blocking defrag |
| E14 | bbolt `v1.5.0` [`tx.go:170`](https://github.com/etcd-io/bbolt/blob/v1.5.0/tx.go#L170), commit; [`:566`](https://github.com/etcd-io/bbolt/blob/v1.5.0/tx.go#L566), data sync; [`:595`](https://github.com/etcd-io/bbolt/blob/v1.5.0/tx.go#L595), metadata write/sync |
| E15 | etcd [`server/storage/backend/config_linux.go:29`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/config_linux.go#L29), NoFreelistSync |
| E16 | etcd [`server/storage/schema/auth.go:112`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/schema/auth.go#L112), explicit auth commit |
| E17 | etcd [`server/storage/backend.go:31`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend.go#L31), factory; [`:58`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend.go#L58), snapshot rename/open/recovery |
| E18 | etcd [`etcdutl/snapshot/v3_snapshot.go:118`](https://github.com/etcd-io/etcd/blob/v3.7.3/etcdutl/snapshot/v3_snapshot.go#L118), bbolt status; [`:257`](https://github.com/etcd-io/etcd/blob/v3.7.3/etcdutl/snapshot/v3_snapshot.go#L257), restore; [`:374`](https://github.com/etcd-io/etcd/blob/v3.7.3/etcdutl/snapshot/v3_snapshot.go#L374), revision bumping |
| E19 | etcd [`server/etcdserver/api/v3rpc/maintenance.go:135`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/api/v3rpc/maintenance.go#L135), snapshot stream/digest; [`server/storage/schema/version.go:48`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/schema/version.go#L48), bbolt snapshot-version reader |
| E20 | etcd [`server/etcdserver/api/snap/db.go:37`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/api/snap/db.go#L37), received file synchronization/rename/directory synchronization |
| E21 | RocksDB `v11.1.2` [`include/rocksdb/db.h:1744`](https://github.com/facebook/rocksdb/blob/v11.1.2/include/rocksdb/db.h#L1744), column-family flush boundaries |
| E22 | etcd [`server/etcdserver/cindex/cindex.go:30`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/cindex/cindex.go#L30), applied/consistent indices; [`server/storage/schema/cindex.go:69`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/schema/cindex.go#L69), persistence |
| E23 | etcd [`server/storage/hooks.go:45`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/hooks.go#L45), consistent-index and conf-state precommit |
| E24 | etcd [`server/etcdserver/server.go:2455`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/server.go#L2455), post-lock application hook |
| E25 | Pebble `v2.1.7` [`README.md:11`](https://github.com/cockroachdb/pebble/blob/v2.1.7/README.md#L11), supported feature scope |
| E26 | Pebble `v2.1.7` [`batch.go:83`](https://github.com/cockroachdb/pebble/blob/v2.1.7/batch.go#L83), indexed reads and atomic batch semantics |
| E27 | Pebble `v2.1.7` [`db.go:736`](https://github.com/cockroachdb/pebble/blob/v2.1.7/db.go#L736), LogData; [`:810`](https://github.com/cockroachdb/pebble/blob/v2.1.7/db.go#L810), Apply; [`options.go:537`](https://github.com/cockroachdb/pebble/blob/v2.1.7/options.go#L537), DisableWAL |
| E28 | Pebble `v2.1.7` [`checkpoint.go:35`](https://github.com/cockroachdb/pebble/blob/v2.1.7/checkpoint.go#L35), synchronization option; [`:133`](https://github.com/cockroachdb/pebble/blob/v2.1.7/checkpoint.go#L133), checkpoint files |
| E29 | RocksDB `v11.1.2` [`include/rocksdb/db.h:554`](https://github.com/facebook/rocksdb/blob/v11.1.2/include/rocksdb/db.h#L554), atomic/synchronized Write; [`:1765`](https://github.com/facebook/rocksdb/blob/v11.1.2/include/rocksdb/db.h#L1765), WAL flush/sync; grocksdb `v1.11.1` [`options_write.go:22`](https://github.com/linxGnu/grocksdb/blob/v1.11.1/options_write.go#L22), sync/WAL options; [`db.go:1818`](https://github.com/linxGnu/grocksdb/blob/v1.11.1/db.go#L1818), FlushWAL |
| E30 | Pebble `v2.1.7` [`db.go:540`](https://github.com/cockroachdb/pebble/blob/v2.1.7/db.go#L540), borrowed values; [`:1582`](https://github.com/cockroachdb/pebble/blob/v2.1.7/db.go#L1582), indexed batch; [`:1615`](https://github.com/cockroachdb/pebble/blob/v2.1.7/db.go#L1615), snapshot/excision semantics |
| E31 | grocksdb `v1.11.1` [`slice.go:35`](https://github.com/linxGnu/grocksdb/blob/v1.11.1/slice.go#L35), borrowed native memory/free; [`db.go:530`](https://github.com/linxGnu/grocksdb/blob/v1.11.1/db.go#L530), Get; [`:1351`](https://github.com/linxGnu/grocksdb/blob/v1.11.1/db.go#L1351), snapshot lifecycle |
| E32 | grocksdb `v1.11.1` [`write_batch_wi.go:196`](https://github.com/linxGnu/grocksdb/blob/v1.11.1/write_batch_wi.go#L196), merged reads; [`:268`](https://github.com/linxGnu/grocksdb/blob/v1.11.1/write_batch_wi.go#L268), iterator ownership/invalidation |
| E33 | Pebble `v2.1.7` [`internal/compression/zstd_nocgo.go:5`](https://github.com/cockroachdb/pebble/blob/v2.1.7/internal/compression/zstd_nocgo.go#L5), Go compression path; [`internal/manual/manual_nocgo.go:5`](https://github.com/cockroachdb/pebble/blob/v2.1.7/internal/manual/manual_nocgo.go#L5), Go allocation path; corresponding `_cgo.go` files provide native paths |
| E34 | grocksdb `v1.11.1` [`write_batch.go:40`](https://github.com/linxGnu/grocksdb/blob/v1.11.1/write_batch.go#L40), per-Put cgo call; [`db.go:1267`](https://github.com/linxGnu/grocksdb/blob/v1.11.1/db.go#L1267), batch write |
| E35 | Pebble `v2.1.7` [`options.go:837`](https://github.com/cockroachdb/pebble/blob/v2.1.7/options.go#L837), L0 limits; [`:890`](https://github.com/cockroachdb/pebble/blob/v2.1.7/options.go#L890), memtable limits |
| E36 | etcd [`server/storage/mvcc/watchable_store.go:346`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/watchable_store.go#L346), catch-up; [`:426`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/watchable_store.go#L426), backend revision scan |
| E37 | RocksDB `v11.1.2` [`include/rocksdb/utilities/checkpoint.h:27`](https://github.com/facebook/rocksdb/blob/v11.1.2/include/rocksdb/utilities/checkpoint.h#L27), checkpoint files/flush; grocksdb `v1.11.1` [`checkpoint.go:34`](https://github.com/linxGnu/grocksdb/blob/v1.11.1/checkpoint.go#L34), binding |
| E38 | etcd [`server/storage/quota.go:27`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/quota.go#L27), defaults/suggested maximum; [`:126`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/quota.go#L126), admission calculation |
| E39 | etcd [`server/storage/mvcc/hash.go:33`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/hash.go#L33), logical record hashing; [`:102`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/hash.go#L102), Hash versus HashByRev scope |
| E40 | etcd [`server/storage/backend/metrics.go:20`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/metrics.go#L20), commit histograms; [`server/etcdserver/metrics.go:75`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/metrics.go#L75), progress/proposal metrics |
| E41 | etcd [`server/storage/backend/backend_bench_test.go:28`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/backend_bench_test.go#L28), physical-key benchmark; [`server/storage/mvcc/kvstore_bench_test.go:30`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore_bench_test.go#L30), store benchmarks; [`server/storage/mvcc/watchable_store_bench_test.go:28`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/watchable_store_bench_test.go#L28), watch benchmarks |
| E42 | etcd [`tools/benchmark/cmd/put.go:60`](https://github.com/etcd-io/etcd/blob/v3.7.3/tools/benchmark/cmd/put.go#L60), flags; [`:86`](https://github.com/etcd-io/etcd/blob/v3.7.3/tools/benchmark/cmd/put.go#L86), reused value and RPC timing; [`tools/benchmark/cmd/txn_put.go:76`](https://github.com/etcd-io/etcd/blob/v3.7.3/tools/benchmark/cmd/txn_put.go#L76), reused transaction value |
| E43 | etcd [`tests/robustness/README.md:19`](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/README.md#L19), failure history; [`:77`](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/README.md#L77), history/fault validation approach |
