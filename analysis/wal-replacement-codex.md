# Replacing etcd's Raft WAL and storage engine together

**Method:** static architecture assessment; peer-informed update and source/issue consistency audit completed 2026-11-01.  
**Versions analyzed:** etcd `v3.7.3` (`0a173d0410a96d01091b484daae81365bbcde57e`); etcd Raft `v3.7.0` (`b867cf13f6bc0dae21204302df97bc2355c3af55`); Pebble `v2.1.7` (`66d468e449bbd52ddf9e53c3a7cdec37e885aaee`); grocksdb `v1.11.1` (`7011ee061b9802a0a82dca28385bcda22c9c7566`) with RocksDB `v11.1.2` (`3b446089141659fad25328c5ea3e7ed283df46e4`). Additional library pins are in §8. jetcd is not substantively evaluated here.  
**Upstream references:** etcd [#22332, alternative storage engines](https://github.com/etcd-io/etcd/issues/22332), [#22339, storage correctness tests](https://github.com/etcd-io/etcd/pull/22339), [#22223, proposed asynchronous storage integration](https://github.com/etcd-io/etcd/pull/22223); etcd Raft [#300, commit-index persistence discussion](https://github.com/etcd-io/raft/pull/300), [#551](https://github.com/etcd-io/raft/issues/551) / [#550](https://github.com/etcd-io/raft/pull/550), and [#543](https://github.com/etcd-io/raft/issues/543), asynchronous-storage safety reports and proposed fix. These are context and implementation leads, not completed implementations of the design below.  
**Summary:**

- **High confidence:** Replacing both layers is feasible in principle without changing the client API or Raft's consensus rules. It changes the durable format, bootstrap, recovery, snapshots, and maintenance tooling, not just two constructors.
- **Moderate confidence:** The first reference design remains **one Pebble database containing Raft records and applied etcd state, with Pebble's WAL as the only redo log**. A new comparison candidate, **A2: separate Pebble log and state databases, both with WALs enabled**, tests whether isolating engine stalls and state synchronization is worth the extra coordination (§3.5). This is not a measured throughput ranking.
- **High confidence about the inspected mechanisms; Moderate about untested failure scope:** Raft's asynchronous storage mode needs an explicit safety gate for outstanding term/vote and snapshot-ordering reports. It remains an independent optimization to investigate, after resolving those concerns (§6).
- **High confidence:** A shared WAL does not eliminate the distinction between accepting a Raft entry and applying a committed command. This design still logs the command and its resulting mutations, and may also write both into sorted files. It can be slower than a specialized sequential Raft log.
- **Moderate confidence:** A dedicated journal plus a WAL-disabled, checkpointed Pebble state store has a higher potential to reduce redundant redo traffic. Its checkpoint and recovery protocol is substantially harder; sustained read latency during checkpointing could disqualify it.
- **High confidence:** No library choice by itself guarantees unchanged read latency. Linearizable reads also wait for application of committed entries, so allowing the apply queue to grow is not an acceptable throughput improvement.
- **Low confidence, planning estimates:** Approximately 10–16 engineer-weeks for a decision-quality single-database prototype and 40–80 for a production candidate, including that prototype. The journal/checkpoint design is approximately 16–26 and 60–120 respectively. These are alternatives, not additive commitments; assumptions are in §12.

No experiments, builds, benchmarks, fault injections, or production changes were performed. There is no measured speedup in this report. “Verified” means inspected code or upstream material; “proposal” is an unimplemented design; confidence labels qualify assessments and forecasts. This extends [the backend-only analysis](storage-codex.md), but explains the relevant durability requirements independently.

## 1. Recommendation and what would change it

**Use a unified Pebble database as the reference design for replacing both layers, and compare it with separate Pebble log/state databases before choosing the production layout.** If state-engine stalls or synchronization payload dominate, the separate-engine variant A2 is the next experiment; if Raft-log LSM materialization or applied-state redo dominates, a dedicated journal with a rebuildable state store addresses those costs more directly. Preserve etcd's logical MVCC representation and ordered application initially. MVCC means multi-version concurrency control: etcd retains historical key versions identified by revisions. **Recommendation confidence: Moderate.**

There is an important baseline correction: current etcd does **not** have a Raft WAL plus a bbolt WAL. It has a sequential Raft write-ahead log and bbolt's copy-on-write pages and metadata. A backend-only replacement with a normally configured LSM engine introduces a second WAL. Replacing both layers with one engine removes that *introduced* second WAL, while replacing bbolt's page-update costs with LSM costs. LSM means log-structured merge tree: updates accumulate in memory and are flushed into sorted files, which background compaction merges. [etcd WAL synchronization, `server/storage/wal/wal.go:869 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/wal/wal.go#L869); [bbolt commit, `tx.go:170 @ v1.5.0`](https://github.com/etcd-io/bbolt/blob/v1.5.0/tx.go#L170).

The useful objective is **maximum sustained successful write throughput subject to correctness, bounded resource use, and the read-latency budget**, not maximum append throughput. The team has not yet supplied its workload distribution, production storage characteristics, or a numerical read-latency budget. Consequently, the recommendation is conditional:

| Production bottleneck | Most relevant intervention | Assessment |
|---|---|---|
| WAL synchronization dominates, with many small batches | Improve batching and asynchronous storage integration; compare the existing WAL against the new design | A different file format alone may yield little. **Moderate confidence.** |
| bbolt application, page writes, or maintenance dominates | Pebble state storage; compare separate-WAL and unified designs | Removing the separate WAL is not necessary to establish the backend benefit. **Moderate confidence.** |
| Applied-state redo and Raft-log LSM compaction dominate the unified design | Dedicated journal plus checkpointed state | Stronger reason to accept the additional recovery work. **Moderate confidence.** |
| Transaction evaluation, MVCC indexing, watch delivery, or network dominates | Optimize that work or partition the application workload | These storage designs leave those limits substantially intact. **High confidence about scope.** |
| Large values dominate bytes written | Compression and selective value separation, evaluated separately | Potential benefit is workload dependent; cold reads and ranges can deteriorate. **Moderate confidence.** |

There is presently **no evidence that rewriting etcd's existing WAL implementation, by itself, is the missing throughput improvement**. The architectural opportunities are sharing durability work, improving the persistence pipeline, and reducing redundant data movement. These must be measured separately.

## 2. Verified baseline and the constraints it imposes

### 2.1 Persistence and application are different events

The Raft library supplies accepted log entries, persistent consensus metadata, committed entries, and messages with ordering obligations. An accepted entry may later be overwritten if it never commits. An applied entry has already been committed. A conditional etcd transaction must be evaluated against the ordered applied state; merely accepting its serialized request cannot safely publish its result.

The stable etcd implementation already overlaps parts of this work. Its Ready loop hands work to the application path, permits leader replication to overlap local persistence, calls storage `Save`, updates the in-memory Raft storage, and coordinates follower messages and application notifications. This is not a simple serial “network, fsync, backend, response” pipeline. A replacement must preserve the actual dependencies, not an oversimplified timing diagram. [`server/etcdserver/raft.go:218,237,256,287,297,331 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/raft.go#L218); [Raft `node.go:49 @ v3.7.0`, Ready contract](https://github.com/etcd-io/raft/blob/v3.7.0/node.go#L49).

The current WAL's `Save` groups entries, writes hard state, and synchronizes when `raft.MustSync` requires it. That helper requires synchronization for entries or changed term/vote; **a commit-index-only change does not itself require synchronization**. This describes the ordinary non-rotating `Save` path: rotation, explicit `Sync`, and `SaveSnapshot` can still synchronize a commit-only update. The wrapper interface's broad stable-storage comment must therefore be read alongside its implementation. [`server/storage/storage.go:29 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/storage.go#L29); [`server/storage/wal/wal.go:995,1026,1039 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/wal/wal.go#L995); [Raft `rawnode.go:191 @ v3.7.0`](https://github.com/etcd-io/raft/blob/v3.7.0/rawnode.go#L191).

Ordinary etcd writes also do not each require a new durable bbolt commit. The backend batches, publishes an in-memory read overlay, and commits on thresholds, timer, or explicit boundaries. Defaults are 100 ms and 10,000 pending operations; these are settings, not expected request latency. A proposal that gains throughput by removing a supposed per-request bbolt fsync starts from the wrong baseline. [`server/storage/backend/backend.go:34,445 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/backend.go#L34); [`server/storage/backend/batch_tx.go:308 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/backend/batch_tx.go#L308).

### 2.2 The recovery boundary is part of the database

etcd persists its consistent index, identifying the applied Raft position represented by backend state, together with relevant backend updates. Its precommit hook also saves changed Raft configuration state. Recovery skips **v3 backend mutations already represented by that index**; it still processes replayed entries to reconstruct v2-format membership and related Raft state. Thus “skip already-applied work” must not mean skipping all entry processing. Before creating a disk snapshot and releasing old WAL resources, etcd forces the backend commit boundary. Losing these relationships can either lose acknowledged writes or replay mutations twice. [`server/storage/hooks.go:45 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/hooks.go#L45); [`server/etcdserver/server.go:1907,1922,2096 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/server.go#L1907).

A particularly relevant subtlety: a durable backend consistent index can exceed the recovered `HardState.Commit` because commit-only changes need not have been synchronized. Stable etcd explicitly handles this in force-new-cluster bootstrap. **Do not assume the existing durable commit field always bounds the durable applied index.** Raft PR #300 proposed changing that behavior but was closed **without merging**; the pinned helper retains the optimization. [`server/etcdserver/bootstrap.go:588 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/bootstrap.go#L588); [Raft PR #300](https://github.com/etcd-io/raft/pull/300).

### 2.3 Reads constrain the design beyond engine Get latency

etcd's linearizable read loop obtains a confirmed Raft index and waits until application reaches it. A faster append path with a growing committed-to-applied backlog can therefore make reads slower even if every engine lookup improves. [`server/etcdserver/read/read.go:117,133 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/read/read.go#L117).

The backend's main key space is revision ordered, while an in-memory index maps user keys to revisions. Reads requesting values use that index and retrieve revision records; watch catch-up scans revision history. Consequently, retaining the MVCC layer is useful for compatibility, but an LSM's physical user-key ordering does not automatically accelerate etcd ranges. [`server/storage/mvcc/index.go:39 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/index.go#L39); [`server/storage/mvcc/kvstore_txn.go:108 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore_txn.go#L108); [`server/storage/mvcc/watchable_store.go:426 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/watchable_store.go#L426).

### 2.4 Attribute rotation and encoding costs separately

The ordinary append path is not the entire WAL cost. On its normal successful path, `cut()` explicitly synchronizes the old tail, synchronizes the new tail after its headers, then synchronizes the directory after rename. These are **two file-data sync calls and one directory sync call**, not a measured count of device commands or a cost incurred on every Put. Segment preparation also uses filesystem preallocation. Measure latency around rotations separately from steady appends. [`server/storage/wal/wal.go:785 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/wal/wal.go#L785); [`server/storage/wal/file_pipeline.go:75 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/wal/file_pipeline.go#L75); [`client/pkg/fileutil/preallocate_unix.go:25 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/client/pkg/fileutil/preallocate_unix.go#L25).

Likewise, an entry is serialized and then wrapped in another serialized WAL record. A future journal could encode a batch directly into reusable buffers, but profiling must establish whether this CPU/allocation work matters before introducing a new format. **Potential benefit: Moderate confidence, magnitude unknown.** [`server/storage/wal/wal.go:974 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/wal/wal.go#L974); [`server/storage/wal/encoder.go:66 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/wal/encoder.go#L66).

## 3. Design A: one Pebble database for Raft and applied state

### 3.1 Layout and responsibilities

This design removes `server/storage/wal` as the active storage implementation and removes bbolt as the state backend. It retains etcd Raft, transaction evaluation, MVCC, leases, auth, and watch semantics. Pebble supplies atomic batches, snapshots, sorted-file maintenance, and the sole physical redo log. Its atomic batch API is verified in [`batch.go:83 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/batch.go#L83).

Proposed logical layout within **one** database:

| Namespace | Content | Lifetime |
|---|---|---|
| `raft/entries/<index>` | Term, entry type, serialized command/configuration change | Until safe Raft-log reclamation; includes an uncommitted suffix |
| `raft/local/*` | Current term, vote, hard-state commit, first/last log positions, boundary term | Local consensus state; never copied from a leader as follower identity/state |
| `state/<bucket>/<key>` | All existing logical backend buckets, including revision records and metadata | Existing logical retention rules |
| `state/applied` | Complete applied index and term, associated revision and configuration metadata | Updated atomically with complete applied entry effects |
| `recovery/*` | Format version, snapshot/root descriptors, generation and integrity information | Governs bootstrap, installation, and reclamation |

These strings illustrate namespaces, not a final byte encoding. `state/applied` denotes the existing consistent-index metadata plus required recovery fields, not a second independently advancing source of truth. The encoding must be collision free, preserve byte ordering, and support bounded prefix operations. Raft indexes, etcd MVCC revisions, and Pebble sequence numbers are distinct counters; none substitutes for another. One Raft entry may change several keys under one main MVCC revision, and some entries do not advance the MVCC revision at all.

Initially retain the current in-memory Raft log cache. The stable bootstrap uses `raft.MemoryStorage`; replacing it with disk lookups would add another latency variable to the Raft event loop. The durable adapter and Raft's `Storage` interface are different responsibilities: the latter principally retrieves entries, terms, and snapshots. [`server/etcdserver/bootstrap.go:540,701 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/bootstrap.go#L540); [Raft `storage.go:42 @ v3.7.0`](https://github.com/etcd-io/raft/blob/v3.7.0/storage.go#L42).

### 3.2 Write protocol

**Proposal; correctness feasibility confidence: Moderate until implemented and fault tested.**

1. **Persist accepted Raft work.** Build one engine batch containing any conflicting-tail removal, replacement/appended entries, hard-state changes, and log bounds. A tail overwrite is restricted to an uncommitted suffix. Use synchronized engine writes whenever the Raft storage contract requires durability. Deliver dependent Raft responses only after successful completion. Validate bounds and integer overflow; a namespace range deletion must never reach applied state.
2. **Apply committed work in order.** Evaluate transactions against a stable read view plus the appropriate pending-write overlay. Publish every entry's complete state effects and applied metadata in an atomic engine batch. Group consecutive entries where useful, but preserve per-request results, revisions, and watch order. Empty entries and configuration changes also advance the application boundary correctly.
3. **Keep the engine WAL enabled for applied state.** Initially synchronize these batches explicitly for the simplest reference implementation. Then evaluate WAL-enabled `NoSync` apply batches: they are visible but need not yet survive a power loss, because retained durable Raft entries can rebuild missing application. A later synchronized write can cover batches whose engine `Apply`/`Commit` calls have completed successfully before that barrier. Merely enqueueing a batch on another goroutine is insufficient. No subsequent write is assumed to arrive: explicit barriers remain mandatory for snapshot/reclamation/shutdown boundaries that require durability.
4. **Coordinate append and apply submissions.** A common storage coordinator enforces the dependencies between log availability, application, snapshot installation, and deletion. The existing apply/persistence overlap is not permission to publish a recoverable applied index whose required state or log basis is absent. Two goroutines writing unrelated “latest” metadata without coordination are insufficient.
5. **Fail on storage errors.** An append or sync error must not release a successful Raft acknowledgment. An uncertain apply failure must not trigger a normal client success. Stop participation or enter a defined recovery path; blindly retrying a partially observed logical mutation can duplicate effects.

Pebble exposes an explicit synchronized barrier through `DB.LogData(nil, Sync)`; its checkpoint code uses this for preceding asynchronous writes. This does not require flushing the entire memtable to sorted files. `Sync=false` with the WAL enabled and `DisableWAL=true` have very different recovery properties. [`db.go:736 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/db.go#L736); [`checkpoint.go:35,165 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/checkpoint.go#L35); [`options.go:537 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/options.go#L537).

The commit-index subtlety in §2 needs an explicit policy. Proposed policy: the atomic `state/applied` record is evidence of commitment through that applied position. On restart, reconcile the hard-state commit with the valid applied/snapshot boundary, while checking that the surviving log or snapshot covers it. Alternatively persist an explicit committed watermark with each apply batch. **Never let the apply worker overwrite a newer term/vote with an older copied HardState.** Consensus term/vote ownership stays with the append path. This reconciliation must be specified for ordinary restart, snapshot installation, and force-new-cluster, rather than copied only from one bootstrap branch.

### 3.3 Recovery, snapshots, and log reclamation

Recovery opens the engine using its normal WAL recovery, validates format and metadata, reconstructs the in-memory MVCC index and Raft cache, and resumes from the **recovered complete** applied boundary. Later committed entries are applied once. An uncommitted suffix remains available to Raft but is not applied speculatively. Corruption in required durable data is an error, not authorization to salvage a shorter acknowledged history.

Before deleting a log prefix through index `S`, make durable a complete state boundary covering `S` and the corresponding snapshot metadata. Retain the boundary term required by Raft and an installable snapshot for followers that can no longer receive those entries. Only then publish deletion, accounting for backup/export readers and in-flight transfers. Keeping extra logs is safe; deleting the sole recovery source prematurely is not.

There are **two different snapshot products**:

- A local database checkpoint can contain local Raft metadata and retained log entries, and can serve as a local recovery image.
- A snapshot sent to another member must export the logical replicated state and its index/term/configuration. It must not install the sender's vote, local identity, or uncommitted suffix on the receiver.

Install incoming state into a new generation, validate it, and durably publish the generation selection together with the required local consensus transition before releasing old files. A directory rename alone does not establish cross-file durability. The stock implementation already orders snapshot-file creation before its WAL marker and synchronizes received snapshot files and directories. [`server/storage/storage.go:59 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/storage.go#L59); [`server/etcdserver/api/snap/db.go:37 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/api/snap/db.go#L37).

Do not use Pebble's `WithRestrictToSpans` as an assumed exact state-only export. Its documented behavior can include keys outside the selected spans, including invalid combinations from partially included files. Use an explicitly filtered logical snapshot or a fully specified image format. [`checkpoint.go:50 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/checkpoint.go#L50).

### 3.4 Expected benefit and the main disadvantage

**Potential benefits, Moderate confidence:** one storage engine owns atomicity and physical redo recovery; synchronization can cover earlier apply work; bbolt page rewriting and blocking defrag disappear; LSM compression and write buffering may reduce device traffic. Fewer independently managed persistence domains simplify some crash boundaries.

**Potential regressions, Moderate confidence:** Raft commands and applied values both enter the engine WAL and may enter sorted files. Short-lived Raft records then create flush, deletion, and compaction work that the current sequential WAL does not have. Raft and user state share memory, compaction capacity, and write stalls. Disjoint key prefixes do not fully isolate those resources.

This is **one WAL, not one encoding of each value and not one transaction per consensus operation**. Do not base its justification on eliminating all duplicated payload bytes. It is attractive when engine integration and shared synchronization outweigh log churn. If they do not, Design B addresses that cost more directly.

### 3.5 Comparison candidate A2: separate Pebble log and state databases

**Proposal; recommendation to compare it with A: Moderate confidence.** Use a dedicated Pebble database for Raft entries, hard state, and recovery markers, and another for applied etcd state. The log database synchronizes according to Raft's durability contract. The state database uses WAL-enabled `NoSync` batches, with explicit background durability barriers before log reclamation. This is a middle option between A and B: it reuses normal engine WAL recovery in both stores and does not require B's discard-and-rebuild policy after every unclean shutdown.

The cross-store boundary is explicit:

1. Under application coordination, capture an immutable tuple `(generation, a, term(a), ConfState(a))` after the complete state batch through `a` has successfully completed its engine `Apply`/`Commit` call. Bind the task to that state-database generation, and coordinate generation replacement with the barrier and marker publication.
2. Issue `stateDB.LogData(nil, Sync)` on that same database instance, off the apply worker and ordered after the completed state batch. Successful completion makes those preceding writes durable; application in the same generation may have advanced further, which is harmless.
3. Publish the corresponding generation-qualified durable-state marker in a synchronized log-database batch. A snapshot installation must wait for this task or cancel and fence it before switching generations; a delayed completion must not label a different database generation durable. Only afterward permit the existing snapshot/retention policy to advance log deletion. Preserve the proper term and configuration at each selected snapshot/log boundary; do not substitute the latest configuration for an older boundary's configuration.
4. On restart, recover both engines, reconcile commit/applied metadata as in §3.2, and verify that the state covers the durable reclamation boundary. Extra retained entries are acceptable; a gap is not. Snapshot-generation installation still needs the coordinated protocol in §3.3.

This protocol does not use a `FlushEnd` event as a durable applied index. That callback alone does not establish which complete etcd state boundary is covered. Keeping both WALs enabled also avoids assuming that an arbitrary WAL-disabled working directory can recover as an exact applied prefix. [`checkpoint.go:35 @ Pebble v2.1.7`, synchronization of preceding writes](https://github.com/cockroachdb/pebble/blob/v2.1.7/checkpoint.go#L35).

**Potential benefit, Moderate confidence:** state-engine write stalls do not directly enter the log engine's write-admission queue, and a log sync need not flush the state engine's WAL through a shared engine stream. The stores can have different cache, memtable, and maintenance settings. This separation is useful even before asynchronous Raft integration, although the existing Ready/apply handoff can still transmit application stalls back into consensus.

**Limits:** this still has two redo streams and can materialize Raft entries into sorted files. Separate instances share CPU, memory pressure, and any underlying disk queues or filesystem resources; they are not complete performance isolation. Even on separate devices, a slow apply path still delays client results and linearizable reads. On a shared filesystem, unsynced state writes may also contend with log synchronization. Measure placement instead of assuming that different directories or unsynced writes remove that contention.

Use byte/time thresholds and bounded replay debt to schedule state durability. Do not assume that deleting Raft keys while they remain in a memtable eliminates all later sorted-file work; pinned snapshots and flush/compaction behavior matter. A2 should win a controlled comparison on tail latency or sustained throughput before accepting its cross-database recovery complexity.

## 4. Design B: dedicated authoritative journal, rebuildable Pebble state

This design replaces etcd's WAL with a segmented journal and replaces bbolt with **Pebble configured without its own WAL**. The journal contains the consensus log and required local recovery metadata. Applied state is materialized into Pebble, but its durable recovery source is an immutable checkpoint plus retained journal records. This is a proposed storage protocol, not a configuration change.

Its specific advantage over Design A is that applying an ordinary committed command need not append its full resulting state mutation to another redo stream or put the Raft command itself through LSM compaction. Applied values still enter memtables, sorted files, and subsequent compaction. **Potential reduction in redundant traffic: Moderate confidence; net throughput advantage: Low confidence without workload measurements.**

### 4.1 Journal format and ownership

The journal needs more than an append API:

- Checksummed, versioned batches with unambiguous completion framing; segment identity, ordering, rotation, and directory durability rules.
- Accepted entries, conflicting-suffix replacement, hard-state term/vote/commit, snapshot-install transitions, and checkpoint-root publication.
- A monotonically increasing **physical log sequence number**, identifying journal append order. This differs from a Raft index: an uncommitted Raft index may be replaced by an entry from another term.
- A durable summary of retained consensus metadata when older segments are reclaimed. Current term/vote must survive even if their last original records were in a segment now being deleted.
- Indexed access to retained Raft entries, explicit pinning for recovery and transfers, and bounded reclamation.

The append worker groups ready work, synchronizes, then releases dependent responses. Suffix replacement must recover as a valid old or new state with respect to the advertised durable boundary, never a mixture that claims an absent entry is durable. Persistent commit-only records can be batched as allowed by Raft; the design does not require an extra fsync for every commit notification.

**Storage assumption:** successful synchronization must actually preserve the acknowledged bytes through the specified power-loss model. Checksums detect corruption; they do not make an unsafe disk cache durable. Replication tolerates failures within the consensus fault model, not arbitrary loss of acknowledged stable storage on a quorum.

### 4.2 Why blindly reopening a WAL-disabled store is insufficient

Pebble explicitly describes disabling its WAL as disabling crash recovery. A marker saying “applied through 500” is not, by itself, proof that every logical mutation through 500 survived an interrupted sequence of flushes, maintenance operations, and metadata updates. Relying on a particular flush order requires a separate proof across every relevant operation and future engine upgrade. [`options.go:537 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/options.go#L537).

**Conservative proposed rule: after an unclean shutdown, discard the mutable WAL-disabled working store and rebuild from the last published immutable checkpoint.** This deliberately avoids making recovery depend on the contents of a damaged or partially durable working generation. An optimization that reuses part of that store is deferred until it has its own complete persistence argument and tests.

There is a concrete validation reason for this conservative choice: Pebble's pinned randomized metamorphic configuration forces `DisableWAL=false` when selecting its crashable filesystem. That inspected test path therefore does not validate our WAL-disabled recovery protocol. This is a coverage boundary, not evidence of a specific engine defect or a claim that every Pebble test excludes WAL-disabled operation. [`metamorphic/options.go:851 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/metamorphic/options.go#L851).

The same rule applies if RocksDB replaces Pebble. Atomic flush can be useful, but it does not by itself specify the whole etcd recovery protocol, particularly across column families, external ingestion, metadata, and snapshots. The grocksdb checkpoint API itself warns that skipping a flush can omit recent data when WAL writing is disabled. [`checkpoint.go:21 @ grocksdb v1.11.1`](https://github.com/linxGnu/grocksdb/blob/v1.11.1/checkpoint.go#L21); [RocksDB `include/rocksdb/db.h:1753 @ v11.1.2`](https://github.com/meta/rocksdb/blob/v11.1.2/include/rocksdb/db.h#L1753).

### 4.3 A concrete checkpoint publication protocol

The proposed initial protocol uses only local files; its correctness remains to be validated:

1. Stop state mutation at a complete applied-entry boundary `S`. Include background metadata/compaction changes in the coordination; stopping only foreground Puts is insufficient. Capture its MVCC revision, applied term, membership/configuration, and every state bucket.
2. Flush all relevant state to stable sorted files. With Pebble, `DB.Flush` is the relevant operation. `Checkpoint(WithFlushedWAL())` is **not** a substitute: the WAL-flush option is skipped when the WAL is disabled. [`db.go:2033 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/db.go#L2033); [`checkpoint.go:165 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/checkpoint.go#L165).
3. Create a complete checkpoint in a new directory while that logical state remains fixed. Give it an etcd-format manifest identifying `S`, term, configuration, revision, engine format, and required files. Ensure the checkpoint owns durable local links or copies of every referenced file; synchronize files and directory entries as required. Start with remote/shared object storage disabled, since Pebble's checkpoint API has additional shared-file lifetime caveats. [`checkpoint.go:133,304 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/checkpoint.go#L133).
4. Append and synchronize a journal record selecting the validated checkpoint generation. Only a durable root may become the recovery authority. Never open the sole immutable checkpoint for normal mutation; create a separate working generation from it.
5. Resume application. Reclaim older journal segments only when the new root and required consensus metadata are durable and every retained-entry/snapshot/backup pin permits it. Preserve the previous generation and its necessary journal coverage until the new publication is known durable.

Crash before root publication: recover from the old root and its retained journal; ignore or clean up the orphan checkpoint. Crash after root publication: the new complete checkpoint is available, and replay starts after `S`. This publication ordering is the central safety argument; it still needs filesystem fault testing.

**Read-latency risk, High confidence about the dependency:** application is paused during the simplest flush/checkpoint procedure, so linearizable reads that require later entries wait. **Magnitude unknown.** Hard links reduce copying but do not eliminate memtable flush time, metadata work, pinned disk space, or contention. A bounded pause is a measurement requirement, not a guarantee of this design.

A later background checkpoint builder could export from a pinned engine snapshot while application continues. It would need an exact snapshot boundary and durable independent output, and could hold back reclamation for a long time. It trades the pause for additional I/O, storage, and implementation work. **Expected tradeoff: Moderate confidence.** Do not include this optimization in an initial schedule as though it were already solved.

### 4.4 Recovery and the committed tail

On restart, validate the journal and selected checkpoint, recreate a mutable state store, and replay entries after `S` that are known committed. Suppress client responses and historical watch delivery during internal reconstruction. Rebuild in-memory indexes and lease/auth/membership state using recovery-specific paths. Do not generate new expiry decisions merely because recovery reads an old command.

There may be durable entries whose commit notification did not survive. Preserve them as Raft's uncommitted/unknown tail and let consensus reestablish commitment; **do not replay them solely because they are on disk**. This is not permission to lose a previously acknowledged operation: the operation must remain recoverable through the surviving quorum's durable logs/snapshot state. Service must not declare itself ready for linearizable reads before the recovered state and Raft barrier satisfy the normal read contract.

Every persistent mutation outside ordinary deterministic command application must be classified. Reproducible maintenance can be resumed or repeated; required non-reproducible local information must enter the journal/checkpoint protocol. Lease persistence, auth revisions, alarms, membership, cluster version, compaction progress, and recovery metadata all require review. This report has **not** proved that replaying serialized KV requests alone recreates every durable effect in etcd. That audit is a prerequisite for Design B, not cleanup after a fast prototype.

Journal retention must cover the last durable checkpoint, lagging-follower policy, and any pinned snapshots. If checkpoints cannot keep up, apply backpressure before filling the disk. Recovery time grows with replay debt, meaning the journal after the last checkpoint; checkpoint cadence and storage headroom therefore become operational limits.

### 4.5 Which journal implementation?

For a Go-first implementation, Pebble's `record.LogWriter` is a plausible framing and synchronization building block. It supports asynchronous sync completion, and its contract covers all records written before the sync request. Callers must serialize record submissions and wait for the requested synchronization, checking its completion error; return from `SyncRecord` alone does not establish durability. It is **not a complete Raft journal**: the application still needs segments, indexing, atomic metadata transitions, suffix replacement, reclamation, and recovery. [`record/log_writer.go:924,932,962,974 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/record/log_writer.go#L924).

TiKV's `raft-engine` is the strongest purpose-built alternative examined here for that role: it already manages log batches, indexed entries, synchronization, and reclamation. It introduces a Rust integration boundary and still does not coordinate etcd state checkpoints for us. The library comparison is in §8.

The existing etcd WAL is also an essential control implementation. If it plus a checkpointed Pebble store reaches the same result, replacing the journal itself has not earned its maintenance cost. **Recommendation confidence: High.** The radical proposal should be judged against this less invasive architecture, even though it does not replace both components.

### 4.6 Recycled segments: a specific optimization with a separate safety argument

Pebble's recycler identifies avoiding filesystem metadata updates as the motivation for reusing log files. Its recyclable chunk format carries a log number to distinguish generations, and its WAL-sync chunk format adds a promised synced offset to help distinguish an incomplete tail from corruption. These are useful primitives for a B journal, and normal Pebble engines can already use them in A/A2. [`wal/log_recycler.go:16 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/wal/log_recycler.go#L16); [`record/record.go:71 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/record/record.go#L71); [`format_major_version.go:201 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/format_major_version.go#L201).

**Potential improvement, Moderate confidence:** previously written reusable segments may reduce allocation/metadata overhead and rotation tails on some production filesystems. The benefit on the team's storage is unknown. A custom implementation must still specify durable generation selection, publication of newly created/renamed files, generation-wrap handling, pinning, and the exact recovery policy for invalid chunks. Reuse cannot begin while any required Raft entry, checkpoint basis, or transfer still references the old generation. A generation tag does not justify treating every checksum or generation error as harmless EOF, and recycling does not remove durability requirements for file names and metadata. The pinned reader distinguishes malformed chunks and uses additional recovery logic; it is not simply “any old generation means success.” [`record/record.go:351 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/record/record.go#L351).

## 5. Design C: make the journal also hold long-lived values

A more ambitious design stores value bytes once in a value journal and keeps indexes or pointers in the state engine. Successful application publishes references to those values instead of copying full values into the state LSM. This resembles the key/value separation studied by WiscKey, which puts keys in an LSM and values in a separate log. The paper is relevant to data placement, not proof of a correct etcd integration or a transferable speedup. [Lu et al., WiscKey, FAST 2016](https://www.usenix.org/conference/fast16/technical-sessions/presentation/lu).

This is not achievable by simply pointing an MVCC record at any serialized Raft request:

- A request may fail, take a conditional branch, omit a value through `ignore_value`, delete keys, or create metadata derived during application. Accepted commands are not all visible values.
- Consensus-log retention and value retention differ. A key's latest value can remain live long after its original Raft entry is no longer needed for replication. Retained MVCC history supplies additional versions for historical reads and watch catch-up, but watchers do not prevent logical compaction from overtaking them: lagging watchers receive `CompactRevision`, and reads below the compacted revision fail. This differs from the physical retention required for an active engine iterator or snapshot. [`server/storage/mvcc/watcher_group.go:255 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/watcher_group.go#L255); [`server/storage/mvcc/kvstore_txn.go:81 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore_txn.go#L81).
- Reclaiming a segment requires finding all live references, copying values if necessary, and atomically publishing new pointers before old storage disappears. Readers and snapshots must pin referenced data through reclamation.
- State snapshots become collections of indexes **and all reachable value objects**. Exporting only the index database would produce an incomplete backup.

**Potential benefit, Moderate confidence:** large-value workloads could avoid substantial copying and compaction of payload bytes. **Read risk, Moderate confidence:** a cold point read may need an additional random read, and ranges can lose locality. Compaction savings do not establish that the extra reads fit the user's latency budget.

Badger and engine-managed blob/value separation are useful implementation references, but their value logs do not automatically become etcd Raft logs. The pinned Pebble already has an experimental value-separation policy; this can be evaluated within Design A before inventing a shared Raft/value lifetime protocol. Its default is disabled, and the policy is ignored when `EnableColumnarBlocks()` is false; setting the policy alone is insufficient. [`options.go:747 @ Pebble v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/options.go#L747).

**Do not make Design C the first production route. Recommendation confidence: High.** Consider it only if large-value traffic remains dominant after simpler options and the team accepts a broader storage-system development project. Moving transaction evaluation before consensus to log only final mutations would be another major redesign involving concurrency, retries, revisions, and compatibility; it is outside a local WAL/backend substitution.

## 6. Asynchronous Raft storage: an independent, gated opportunity

The pinned Raft library exposes `AsyncStorageWrites`. Its local append and apply messages can be processed by separate workers. Messages to each worker must be reliable and ordered. Append-response messages are released only after the required writes are durable; apply responses do not inherently require durable state-machine writes. That last point is a Raft interface property, **not** a complete application recovery policy. [Raft `raft.go:153 @ v3.7.0`](https://github.com/etcd-io/raft/blob/v3.7.0/raft.go#L153).

Stable etcd does not enable this option in its Raft configuration; it uses the Ready/Advance integration. Its Ready loop executes the blocking storage save in the same loop that handles ticks. The library capability therefore requires actual integration changes. [`server/etcdserver/bootstrap.go:540 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/bootstrap.go#L540); [`server/etcdserver/raft.go:181,256 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/raft.go#L181).

This does not mean all Raft processing stops during that save: the library's separate `Node.run` loop can still receive proposals and messages while the next Ready waits for `Advance`. That already creates natural batching. The opportunity is to release subsequent storage/network/application work earlier, not to invent batching where none existed. [Raft `node.go:343,435 @ v3.7.0`](https://github.com/etcd-io/raft/blob/v3.7.0/node.go#L343).

**Safety gate added after further source review:** the first version of this report omitted two open asynchronous-storage safety reports. As checked on 2026-11-01, both issues remain open; PR #550, the proposed vote-ordering fix, is also open and unmerged. The cited mechanisms are present in the pinned Raft source, but I have not run the supplied reproductions, which target upstream `main` rather than the pinned release. **Confidence: High for the source observations and status; Moderate for the reported failure scenarios and their complete affected scope.**

| Upstream report | Verified mechanism / reported consequence | Requirement for this proposal |
|---|---|---|
| [Raft #551](https://github.com/etcd-io/raft/issues/551), proposed fix [#550](https://github.com/etcd-io/raft/pull/550) | `send()` defers append/vote responses, but does not put `MsgVote` requests in that deferred class. The reporter's test uses `main@c51f010`, three voters, PreVote disabled, and async storage only on the candidate. It reports that candidate winning without its new term/self-vote durable, crashing, and leading the same term again with conflicting log contents. | Establish durability of the candidate's new term and self-vote before releasing the associated real vote requests. Review the fix and reproduce the failure/fix on the selected version and configuration before enabling async mode; PR existence is not resolution. |
| [Raft #543](https://github.com/etcd-io/raft/issues/543) | The reporter's example uses `main@1931ff5`. Append and apply workers can progress independently: if the append worker installs a newer snapshot in the state machine, entries already dispatched to the apply worker can subsequently be applied again on top of that snapshot. | Stage/persist the snapshot through append storage, but serialize the state-machine replacement through the application coordinator. Fence earlier and later application generations; no queued old entry or completion may mutate the installed image. |

Source anchors: [Raft `raft.go:546,1071 @ v3.7.0`](https://github.com/etcd-io/raft/blob/v3.7.0/raft.go#L546); [`rawnode.go:142 @ v3.7.0`](https://github.com/etcd-io/raft/blob/v3.7.0/rawnode.go#L142); [`log.go:253 @ v3.7.0`](https://github.com/etcd-io/raft/blob/v3.7.0/log.go#L253). The snapshot concern is an application-integration obligation even if upstream ultimately resolves it through documentation rather than a library change. These reports do not establish an enabled async-mode defect in stock etcd v3.7.3, where the mode is off.

There is also useful upstream implementation work: [etcd PR #22223](https://github.com/etcd-io/etcd/pull/22223), head `3a64fae28a34524b0b159d46ae8820f9cc992da2`, proposes an off-by-default alpha gate and ordered append/apply workers. It is open, unmerged, and marked `needs-rebase`. Review it as a starting point for a local prototype; this pass has not audited whether it addresses either report, and its author-reported benchmark gains are not imported into this estimate.

One beneficial contract difference is verified: async mode restricts committed-entry application to entries already stable locally (`applyUnstableEntries()` returns false). This can simplify the log/state availability invariant, but does not eliminate commit-field reconciliation, snapshot fencing, or engine durability barriers. [Raft `rawnode.go:440 @ v3.7.0`](https://github.com/etcd-io/raft/blob/v3.7.0/rawnode.go#L440).

Proposed integration:

1. The Raft event loop sends append work to one ordered persistence worker and committed work to one ordered apply worker, using the library's documented response dependencies.
2. The append worker groups available records by bounded bytes and bounded delay, performs synchronization, checks errors, and returns completion messages in the required order.
3. The apply worker batches compatible work while retaining deterministic entry ordering and transaction visibility. This does not make a single etcd state machine arbitrarily parallel.
4. Admission limits bound outstanding append bytes, committed-to-applied distance, and memory. Reads and election-critical metadata must not starve behind a large batch.
5. Snapshot installation drains or fences both queues at a defined generation boundary. Old worker completions cannot publish into the new generation.

Preserve etcd-specific ordering as well as the library contract. Stable etcd delays candidate/follower messages when pending configuration changes must first be applied, to avoid counting votes under the wrong membership. The asynchronous adapter needs an equivalent dependency; simply forwarding all nonlocal messages immediately is insufficient. Port the corresponding failpoints and test member removal/promotion with outstanding writes. [`server/etcdserver/raft.go:304 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/raft.go#L304).

The library's `Advance` path is not used in asynchronous mode; it explicitly rejects that combination. A detached goroutine around `Save` followed by an early `Advance` is not an implementation of this protocol. [Raft `rawnode.go:475 @ v3.7.0`](https://github.com/etcd-io/raft/blob/v3.7.0/rawnode.go#L475); [`rawnode.go:245 @ v3.7.0`, append-dependent responses](https://github.com/etcd-io/raft/blob/v3.7.0/rawnode.go#L245).

**Expected benefit, Moderate confidence:** more overlap and larger useful batches when persistence blocks progress. **Failure condition:** low concurrency, an already well-batched workload, or apply/CPU/network saturation may leave little benefit; batching delay can worsen low-load latency. A single etcd Raft group also lacks the cross-group batching opportunity of a multi-Raft database. Benchmark asynchronous integration separately from engine choice so that any gain can be attributed correctly.

Keep this branch optional: A, A2, and B can first be evaluated with the existing synchronous Ready/Advance contract. An unresolved asynchronous-mode safety concern is a reason to defer that mode, not to relax the correctness gate or presume every storage layout requires it.

## 7. Read-latency protection and theoretical performance accounting

### 7.1 Count work at the right layers

Let `R` be serialized Raft-command bytes per unit of useful work, `M` the applied mutation bytes, `P` bbolt page traffic, and `F(X)` the actual LSM flush/compaction traffic generated by workload `X`. These are accounting variables, not measured constants. Index writes, compression, retained history, and metadata affect their sizes.

| Architecture | Main redo traffic | Other materialization work |
|---|---|---|
| Current etcd | Raft journal containing `R` | bbolt page/metadata traffic `P` |
| Existing Raft WAL + ordinary Pebble backend | Raft journal `R` plus engine WAL for `M` | `F(M)` |
| A: one engine for both roles | One engine WAL containing both `R` and `M` | `F(R + M)`, including Raft-log deletion effects |
| A2: separate Pebble log/state engines | Log-engine WAL for `R`; state-engine WAL for `M`, with background durability barriers | Separate `F(R)` and `F(M)` workloads; device and filesystem contention may remain |
| B: journal + WAL-disabled checkpointed state | Journal containing `R` and required recovery metadata | `F(M)` plus checkpoint retention/copying costs |
| C: shared value journal | Commands/values plus pointer and publication metadata | Index compaction plus live-value reclamation |

These are structural descriptions, not byte-equality formulas. In particular, `F(R + M)` need not equal `F(R) + F(M)`: layout, grouping, retention, and compaction interact. In Design A, a separate log namespace can help locality but does not avoid its materialization cost.

The number of files is also not the number of synchronizations. Separate synced append and apply calls to one WAL can still cause two waits. Piggybacking requires the writes to be submitted in the right order and a durability barrier that covers them. Conversely, current etcd already amortizes backend commits and batches Raft entries.

For a simplified bottleneck model, sustained throughput is bounded by the minimum of consensus/network capacity, durable-append capacity, ordered-apply CPU capacity, and long-run storage maintenance capacity. Durable-append capacity depends on both bytes and effective entries per sync. Increasing buffering changes short-run throughput but does not raise the long-run limit if compaction or checkpoints fall behind. **Inference confidence: High for the limits; no numerical capacity estimate is supported.**

### 7.2 Preserve read behavior deliberately

For A, A2, or B:

- Keep the existing logical MVCC index and revision semantics initially. Publish index changes, visible state, and revision boundaries together under the required coordination.
- Use point-in-time read views for multi-key transactions and ranges. An engine's atomic write batch alone does not make a sequence of independent reads a consistent snapshot.
- Budget cache memory across the Raft cache, MVCC index, memtables, block/filter caches, Go heap, and any native allocations. Comparing a cache-rich candidate against a memory-starved baseline is not an engine comparison.
- Evaluate Bloom filters, block caching, compression, and compaction parallelism against **mixed** reads and writes. A smaller compressed database can help reads, while decompression and compaction contention can hurt them. **Expected tradeoff: Moderate confidence.**
- Bound apply lag and maintenance debt. Higher admission throughput is not success if linearizable reads wait longer or followers lose the ability to catch up.
- Retain enough free storage for compaction, checkpoint generations, and snapshots. Report pinned bytes and disk-full behavior; logical key count alone is not a usable quota.

An optional decoded-value cache keyed by immutable MVCC revision could protect hot reads, subject to snapshot visibility and memory accounting. A cache keyed only by user key needs additional invalidation and historical-read handling. Neither cache may bypass Raft's linearizable-read barrier.

The user has not defined “meaningfully worse.” For a future test plan, adopt a **provisional, user-adjustable gate**, for example no more than a 10% p99 increase at matched mixed-workload load, plus a separate absolute p99/p99.9 SLO and explicit bounds during maintenance and failover. The 10% is a proposed decision threshold, not a finding or an inferred requirement. Apply it separately to hot/cold point reads, ranges, historical reads, and watch catch-up; averages cannot establish this objective.

### 7.3 Candidate read optimization: fetch selected revisions in physical order

Both independent analyses suggest a useful way to protect range reads without changing the on-disk schema. In its value-fetching path, the pinned `rangeKeys` obtains the logical result's revision list, applies the result limit, then fetches each selected revision individually. Its `CountOnly` and `FastKeysOnly` branches already return from the in-memory index and should retain those shortcuts. Pebble has a forward-seek optimization that can reuse iterator position for increasing targets. These mechanisms are verified; a net performance improvement is **Moderate-confidence, workload-dependent**. [`server/storage/mvcc/kvstore_txn.go:84,90,108 @ etcd v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore_txn.go#L84); [`iterator.go:298 @ Pebble v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/iterator.go#L298).

Proposed algorithm:

1. Select exactly the same revisions as today, including requested historical revision, logical ordering, count, and limit. Retain each result's original position. Sorting must not change which keys a limited request selects.
2. Within one fixed read view, including any required transaction-local overlay, sort a permutation of those targets by their encoded revision keys.
3. Use one bucket-bounded iterator and increasing `SeekGE` targets. Verify an exact key match after each seek, decode/copy values before advancing, and place results back at their original positions. Missing data remains an error; the next greater key is not an acceptable substitute.
4. Retain cancellation checks and memory limits. Leave single-key/small-result reads on the simpler point-lookup path unless measurements favor the iterator.

This preserves the existing logical result under a fixed view; it does not turn an arbitrary set of revision keys into a contiguous scan. Sparse historical targets can still touch many files and blocks. Sorting adds `O(N log N)` work and `O(N)` temporary storage, and blind `Next()` traversal across wide gaps could be much worse. Compare ordinary Gets, sorted Gets, and sorted seeks for dense/sparse revisions and hot/cold caches; select an adaptive threshold only from those results. The optimization may be useful for A, A2, or B independently of WAL replacement.

## 8. Library choices and what they actually supply

The following additional sources were cloned read-only for inspection. Tags were cross-checked with GitHub's latest-release API on 2026-11-01. Release freshness is not a reliability score.

| Library and inspected pin | Useful role | Limitation / decision |
|---|---|---|
| **Pebble `v2.1.7`**, pin in header | Entire engine for A; state engine or journal framing primitive for B | First choice for a Go team. Atomic batches, WAL, snapshots, checkpoints; etcd integration remains substantial. **Recommendation: Moderate confidence.** |
| **grocksdb `v1.11.1` + RocksDB `v11.1.2`**, pins in header | Same basic architectures as Pebble; possible separate column families for log/state | Viable alternative if existing RocksDB operational expertise or measured workload behavior favors it. Requires native build/version and memory-lifetime management. **Recommendation: Moderate confidence.** |
| **TiKV raft-engine `0.4.2`**, `084c5ca0e4f8291ab67afbfe8b9df02747838c86` | Dedicated journal component for B | Purpose-built log storage, but Rust integration and etcd checkpoint coupling remain. Shortlist for B; do not infer benefit from multi-Raft marketing. **Recommendation: Moderate confidence.** |
| **hashicorp/raft-wal `v0.5.0`**, `ebffec3619e9330066b00ebd42135a68ef404421` | Possible journal foundation | Pinned README still labels it experimental. Different Raft interfaces and a bbolt metadata store; not a clear upgrade over etcd's existing WAL. **Recommendation against first choice: Moderate confidence.** |
| **tidwall/wal `v1.2.0`**, `1f6190cf2109dc2fdd653547886f9b8075c9814f` | Small Go append-log primitive | Batch write/sync/truncation APIs, but no complete etcd metadata/checkpoint protocol. Simplicity does not remove the integration proof. **Recommendation against first choice: Moderate confidence.** |
| **Badger `v4.9.6`**, `fbd8d2eefad8be8757249767255faf989945b599` | Alternative Go state engine and value-separation reference | Native transaction/MVCC semantics do not replace etcd's semantics. Has both memtable WAL and value-log machinery; not an automatic single-log solution. **Recommendation to defer: Moderate confidence.** |

**RocksDB details.** The wrapper exposes synchronized writes, WAL disabling, batched writes, checkpoints, and WAL synchronization. One RocksDB database can place Raft and state data in different column families sharing a WAL, with per-family settings where supported. This can improve layout/tuning options but introduces cross-family flush/recovery boundaries and does not guarantee independent write stalls or shared-resource isolation. All required durable batches must keep the WAL enabled in Design A. The wrapper release names native RocksDB `v11.1.2`; do not substitute the newer `v11.12` release without a separate compatibility check. [`options_write.go:22 @ grocksdb v1.11.1`](https://github.com/linxGnu/grocksdb/blob/v1.11.1/options_write.go#L22); [`db.go:1267,1818 @ v1.11.1`](https://github.com/linxGnu/grocksdb/blob/v1.11.1/db.go#L1267); [paired release](https://github.com/linxGnu/grocksdb/releases/tag/v1.11.1); [RocksDB write API, `include/rocksdb/db.h:554 @ v11.1.2`](https://github.com/meta/rocksdb/blob/v11.1.2/include/rocksdb/db.h#L554).

**TiKV raft-engine details.** The pinned implementation stores log-entry locations in memory, appends grouped log batches, supports sync requests and compression, and exposes explicit purging. It also stores small per-group key/value metadata. These are directly relevant primitives for consensus storage. Its garbage collector still needs the embedding system to tell it which logs are safe to discard. A Go/Rust adapter, protobuf translation, error handling, and lifecycle ownership would need to be built or separately verified; this pass did not establish a supported etcd adapter. [`README.md:8,21,27,43 @ 0.4.2`](https://github.com/tikv/raft-engine/blob/0.4.2/README.md#L8); [`src/engine.rs:140,320 @ 0.4.2`](https://github.com/tikv/raft-engine/blob/0.4.2/src/engine.rs#L140).

**HashiCorp details.** Its advertised fsync advantage is against `hashicorp/raft-boltdb`, which stores Raft logs in bbolt, not against etcd's existing sequential Raft WAL. The pinned source supplies separate `StoreLogs`, `DeleteRange`, and stable-metadata operations, with bbolt in `wal-meta.db`. Adapting these methods requires an explicit etcd persistence protocol; matching interface names is not enough. The project's own experimental warning weighs against using it to reduce validation work. [`README.md:10,16,75 @ v0.5.0`](https://github.com/hashicorp/raft-wal/blob/v0.5.0/README.md#L10); [`wal.go:358,464,525 @ v0.5.0`](https://github.com/hashicorp/raft-wal/blob/v0.5.0/wal.go#L358).

**Tidwall details.** The inspected implementation synchronizes writes by default, supports batched writes and explicit sync, and exposes suffix truncation. These are useful low-level operations, but this review has not established the atomic combined metadata/tail-replacement protocol required here. This is a scope limitation, not an asserted library corruption bug. [`wal.go:65,102,441,870,969 @ v1.2.0`](https://github.com/tidwall/wal/blob/v1.2.0/wal.go#L65).

**Badger details.** Its active memtable has its own WAL, and `DB.Sync` discusses synchronization of that WAL and the value log. Therefore “use Badger's value log as the only persistence mechanism” is not a description of its ordinary architecture. Its value separation could still be evaluated for a large-value workload, but that would be a different engine comparison, with extra garbage-collection and read-locality questions. [`memtable.go:31,155 @ v4.9.6`](https://github.com/dgraph-io/badger/blob/v4.9.6/memtable.go#L31); [`db.go:708 @ v4.9.6`](https://github.com/dgraph-io/badger/blob/v4.9.6/db.go#L708).

**A tempting Pebble shortcut to reject:** putting Raft entries only in `Batch.LogData` avoids inserting them into the LSM, but those records are not indexed database keys and do not acquire Raft's retention lifetime. Pebble reclaims its WAL according to engine recovery needs. A correct shared physical journal would require independent retention, indexing, and replay integration; it cannot be obtained by changing `Set` to `LogData`. [`batch.go:1169 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/batch.go#L1169); [`wal/wal.go:104 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/wal/wal.go#L104); [`wal/standalone_manager.go:95 @ v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/wal/standalone_manager.go#L95).

## 9. Correctness obligations and failure schedules

The design review should produce a written invariant set before performance tuning. At minimum:

1. Every released durability-dependent Raft response is backed by stable entries and required term/vote state.
2. Visible applied state is the result of a complete, ordered prefix of committed entries, with etcd revisions and metadata consistent with that prefix.
3. A recovered applied boundary never skips missing effects; replay never duplicates already represented state mutations. Reconstructing in-memory indexes and membership is a separate recovery obligation.
4. There is always a complete recovery basis: durable state plus required log tail for A/A2, or a valid immutable checkpoint plus journal coverage for B.
5. Reclamation cannot invalidate that basis or a pinned snapshot/reader; current consensus term/vote cannot be rolled back to an older checkpoint's values.
6. Snapshot installation preserves the recipient's local identity and the correct consensus transition, while atomically selecting replicated state.
7. Reads, watches, transactions, leases, auth, membership, hashes, and compaction retain their external contracts. Storage improvement is not permission to weaken any of them.

The following are **future test schedules, not reproduced findings**:

| Failure point | Required outcome |
|---|---|
| Append submitted, sync not completed, process/power failure | No successful durability-dependent response was released; recover only valid storage state |
| Append sync succeeds, process dies before sending response | Retain the entry; retries may be ambiguous to the client but cannot corrupt the log |
| Async candidate campaigns with term/vote persistence stalled, then crashes and restarts | No reused election term may produce conflicting accepted/applied contents; exercise the #551 schedule and the selected fix |
| Entry committed, before/within apply batch | Recover old complete applied boundary or complete new boundary; replay only missing committed work |
| Applied state durable, commit-only HardState update lost | Reconcile with valid applied/snapshot evidence; do not discard actually committed history |
| Conflicting uncommitted suffix replaced, crash at every write boundary | Recover a coherent log and bounds; no overwritten committed prefix or falsely durable missing entry |
| B checkpoint files exist, root not durable | Old checkpoint and journal remain sufficient; new files are orphans |
| B new root durable, crash during old-segment deletion | New checkpoint and retained tail remain sufficient, including latest local consensus metadata |
| Snapshot received while append/apply work is pending | One valid generation selected; no old completion writes into new state; sender identity never adopted |
| Recycled segment contains stale generation bytes, or a chunk within a demonstrably durable region is corrupted | Reject stale data and detect required-data corruption; do not silently turn either condition into an acknowledged shorter log |
| Disk full or sync error during append/apply/checkpoint | No false success, no loss of last valid root, bounded failure/backpressure behavior |
| Crash during logical compaction, lease revocation, auth/configuration change | Restore semantically valid state and progress; retained historical/live versions remain correct |
| Repeated leader changes plus slow follower and long snapshot reader | No unsafe reclamation; bounded memory/disk growth or explicit admission failure |

Use etcd's existing robustness framework and linearizability/watch checks as a foundation, extending failpoints to the new storage boundaries. Its model does not automatically cover every lease/auth/membership promise; add targeted coverage for those transitions. [Pinned robustness framework](https://github.com/etcd-io/etcd/tree/v3.7.3/tests/robustness); [pinned integration tests](https://github.com/etcd-io/etcd/tree/v3.7.3/tests/integration).

Process termination alone is insufficient to validate power-loss durability: OS buffers can survive a process kill. Later validation needs filesystem fault injection or a crash simulator that models lost/reordered unsynchronized writes, plus tests on the real storage stack. Repeatable state-machine differential tests against stock etcd can compare transaction outcomes, revisions, metadata, and logical hashes. A passing finite test campaign is evidence, not a mathematical proof of absence of all storage bugs.

**Use the existing LazyFS integration as a concrete starting point.** LazyFS is the filesystem test layer through which etcd's robustness kill/panic failpoints can discard data that has not been synchronized. The pinned kill and panic paths already call `ClearCache`. However, the default scenario builder adds LazyFS cases only for selected low-load, single-member scenarios and removes compaction from those cases. Thus passing the stock LazyFS selection alone would not cover this design's quorum, election, compaction, and snapshot interactions: explicitly extend it to those combinations. [`tests/robustness/failpoint/kill.go:51 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/failpoint/kill.go#L51); [`tests/robustness/failpoint/gofail.go:128 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/failpoint/gofail.go#L128); [`tests/robustness/scenarios/scenarios.go:154 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/scenarios/scenarios.go#L154).

For component-level enumeration, Pebble's `vfs.NewCrashableMem` supplies an existing crashable-filesystem building block; it does not replace device-level validation. Also port the robustness report's WAL reader and retain equivalent ordering failpoints: it currently reads `.wal` files using etcd's decoder, so changing the storage format without updating that tool leaves the test system unable to reconstruct the new log correctly. [`metamorphic/options.go:851 @ Pebble v2.1.7`](https://github.com/cockroachdb/pebble/blob/v2.1.7/metamorphic/options.go#L851); [`tests/robustness/report/wal.go:368 @ etcd v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/report/wal.go#L368).

## 10. Fork scope, compatibility, and operations

A fork is a reasonable implementation vehicle because these changes cut across etcd's storage assumptions. **Feasibility assessment: High confidence; suitability for this team's deployment: Moderate confidence pending maintenance capacity.** Keep the first implementation on the selected stable release and retain a stock backend mode for differential testing.

The seam to introduce is a **durability coordinator** owning append persistence, applied-state publication, recovery, snapshot installation, and reclamation. Separate `RaftLog` and state-backend interfaces can sit beneath it, but their lifetimes cannot be independent. Design A supplies one shared engine instance; A2 supplies separate durable log/state engines; B supplies a journal plus checkpoint manager and working state engine.

Likely changes extend to:

| Area | Why it changes |
|---|---|
| `server/storage/backend`, schema and hooks | Engine transactions, read overlays, consistent-index publication, physical accounting |
| `server/storage/storage.go`, Raft integration and bootstrap | WAL abstraction, hard state, log loading, asynchronous completion if adopted |
| MVCC restore, compaction, watch catch-up | Stable views, replay boundaries, record lifetime, hash equivalence |
| Snapshot sender/receiver, maintenance API, `etcdutl` | New physical format, filtered replicated-state export, installation and offline restore |
| Lease/auth/membership/version/alarm storage | Correct persistence and restart reconstruction for every state category |
| Metrics, quota alarms, defrag, health and support tools | Engine-specific disk/memory accounting, stall/debt visibility, meaningful maintenance behavior |

These are source-backed integration points, not an assertion that all the packages need wholesale rewrites. The current storage wrapper and bootstrap retain concrete `wal.WAL` dependencies; snapshot restore/status code opens bbolt directly. [`server/storage/storage.go:45 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/storage.go#L45); [`server/etcdserver/bootstrap.go:554,628 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/bootstrap.go#L554); [`etcdutl/snapshot/v3_snapshot.go:118,257 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/etcdutl/snapshot/v3_snapshot.go#L118).

Preserving ordinary Raft entry encoding helps wire compatibility, but **does not establish rolling-upgrade compatibility**. A stock member expects the stock snapshot format. Plan one of two explicit routes:

- **First production route:** a coordinated, fenced migration through a validated logical snapshot into a homogeneous forked cluster. Preserve or deliberately transform revisions, compacted revision, leases, auth, and membership using an explicit restore policy. Account for outage/cutover and client watch resumption.
- **Later optional route:** support mixed members by negotiating snapshot formats and implementing both directions of conversion. Test learner addition, catch-up after log truncation, restart, and rollback while members use different engines. This costs materially more. **Effort assessment: Moderate confidence.**

Do not overwrite the only original data directory in place. Produce versioned new directories and maintain immutable pre-migration backups. Keeping an old directory is not rollback after the new cluster accepts writes: rollback then needs a reverse state export or an explicitly accepted loss of those writes. Never allow old and new clusters to accept the same application writes independently during cutover.

Maintain the fork as a small, reviewable series: storage contract and format; engine implementation; recovery and snapshot/tooling; optional async pipeline; tuning. Pin dependencies and storage formats, and continuously merge security/consensus fixes. The project does not authorize pushing or opening PRs against `etcd-io/*`; no fork or implementation worktree was created during this analysis.

## 11. Experiments to run later on representative hardware

### 11.1 Stage gates and controls

All tests below are proposed. Run them on the intended machine/platform, with equivalent durability enabled and comparable memory/CPU limits. Keep cluster size and client behavior fixed for each comparison.

| Stage | Candidate/control | Question answered |
|---|---|---|
| 0 | Stock stable etcd, real workload or representative replay | Which resource limits successful writes? What are the read SLOs and maintenance tails? |
| 1 | Existing WAL + Pebble backend, both WALs enabled | How much benefit comes from the state engine alone? |
| 2 | Design A with synchronous reference apply, then WAL-enabled asynchronous apply | Does sharing the engine/WAL help after accounting for log materialization and durability barriers? |
| 2b | A versus A2 with the same Raft mode, total memory budget, and workload | Is separating engine write admission/state synchronization beneficial enough to justify the extra recovery boundary? |
| 3, safety-gated | Same winning layout with/without Raft `AsyncStorageWrites`, after addressing §6 reports | How much is persistence pipeline batching rather than engine choice? |
| 4, conditional | Design B with existing etcd WAL as control, then replacement journal | Does avoiding applied-state redo help enough to pay for checkpoint and journal changes? |
| 5, conditional | RocksDB or value separation in the relevant winning architecture | Does a specific workload justify native integration or extra value-log machinery? |

Do not build every combination before the preceding evidence justifies it. Correctness checks gate cluster performance testing; operational completeness gates deployment. If Stage 0 finds a client/network/watch bottleneck, the storage branch may stop without a production fork.

### 11.2 Small component experiments

**Durable append test.** Replay realistic Raft batch sizes and payload distributions, including suffix replacement and term/vote changes. Compare existing WAL, Pebble Raft-key writes, and the shortlisted journal using identical synchronization obligations. Measure bytes per useful operation, sync count and latency, CPU, allocations, recovery time, and disk occupancy after sustained truncation. A sequential append-only microbenchmark without reclamation is insufficient to rank A versus B.

**Recycling and placement test.** First isolate the storage mechanism: compare appending into newly preallocated files with overwriting previously written reusable files, using equal batch sizes and durability obligations. Include fresh allocation, steady reuse, and rotation tails. Repeat while a state-like writer runs in another directory on the same filesystem, on a separate filesystem on the same device, and on a separate device where production permits it. This separates potential metadata coupling from physical-device contention. These component numbers guide whether to prototype recycling/A2; they are not equivalent-durability cluster throughput results.

**Combined log/state test.** Reproduce the sequence of accepted commands followed by committed state batches, rather than issuing arbitrary random-key puts. Include revision-ordered records, overwrites of existing user keys, selective MVCC history deletion, metadata, and full-entry atomic boundaries. Measure applied throughput, WAL traffic, sorted-file writes, compaction debt, stalls, and read latency against the same memory budget.

**Checkpoint test for B.** At production-like database and dirty-memtable sizes, measure stop-apply time, checkpoint publication time, pinned bytes, replay debt, and full restart duration. Inject failures at each publication/reclamation boundary. This test can reject B early even if its append numbers look good.

**Range-fetch test.** Compare unchanged point Gets, sorted point Gets, and the §7.3 iterator algorithm over the exact same selected revisions and fixed read view. Sweep result size, revision density, history depth, cache residency, and concurrent writes/compaction. Verify identical results, ordering, limits, counts, and cancellation; measure sorting/allocation cost as well as engine I/O and p99 latency. Keep the optimization independently switchable from the engine and Raft changes.

Prefer existing etcd benchmark and robustness code. A small additional adapter/driver is justified for new journal boundaries or mixed open-loop scheduling that those tools do not express; it should reuse the existing client and result validation, not invent different etcd semantics.

### 11.3 Cluster workload and measurement matrix

Begin with the actual production topology. Add a three-member reference cluster only if useful for isolating differences; a single member does not exercise quorum replication. Run a separate client/load host with headroom.

Vary one dimension at a time around the representative workload:

- Value sizes and compressibility from actual payloads; include small metadata and the production large-value tail. Repeated identical benchmark values can exaggerate compression benefit.
- Hot-key overwrites versus growing key space, retained revision history, transaction sizes, failed compares, and delete/revoke bursts.
- Point reads, ranges of different sizes, historical reads, watches with both caught-up and lagging consumers; distinguish linearizable and explicitly serializable reads.
- Hot working set and data larger than available cache. Include restart/cold-cache behavior.
- Offered write load below saturation and near each candidate's SLO boundary. Use an open-loop schedule or equivalent queue-aware timing so server stalls do not disappear through coordinated omission.
- Active compaction, snapshots, checkpoint publication, slow followers, leader replacement, and storage pressure.

Run long enough that flush/compaction/checkpoint debt reaches a bounded repeating regime and more than one reclamation cycle occurs. A fixed short duration or a run ending while dirty memory is still growing cannot establish sustainable throughput. Randomize run order and repeat sufficiently to distinguish the proposed read-latency budget from noise.

Record successful writes/second; client-observed p50/p95/p99/p99.9 write and read latency; errors/timeouts; commit/apply lag per member; WAL sync count/latency; engine WAL and sorted-file bytes; compaction backlog/stalls; CPU/GC; all memory categories; physical/pinned disk bytes; watch delay; follower recovery; snapshot/checkpoint times; and restart replay time. Keep latency classes separate rather than reporting a single merged percentile.

For reproducibility, each result must include exact etcd/engine commits, build flags, cluster size, CPUs/RAM, disks and filesystem/mount options, virtualization/storage cache settings, network/TLS, client version, concurrency, key/value distributions, retention/maintenance settings, engine options, and the exact commands/configuration. No numbers from this host are performance findings.

**Go/no-go rule:** first require zero detected correctness violations across the agreed fault campaign; then require a pre-agreed material gain in *sustainable successful* writes while every required read class meets its relative and absolute latency limits, resource growth is bounded, and recovery/cutover objectives pass. The amount of gain needed to justify owning the fork remains a team decision. Averages or a write-only peak do not satisfy this gate.

## 12. Effort, risks, and decision inputs

These estimates assume experienced Go/storage engineers, access to a representative test environment, reuse of etcd tests, and a homogeneous first deployment. They are **Low-confidence planning estimates**, not quotes or measured implementation costs. Engineer-weeks measure effort, not calendar time; review, fault-test iteration, and soak periods constrain parallelism.

| Alternative | Decision-quality prototype | Production candidate, including prototype | Main uncertainty |
|---|---|---|---|
| A: unified Pebble | 10–16 engineer-weeks | 40–80 engineer-weeks | Recovery metadata, snapshots/tooling, async integration, read behavior under log churn |
| A: unified grocksdb/RocksDB | 12–20 | 48–96 | Same issues plus native build/ownership and supported platform coverage |
| B: journal + checkpointed state | 16–26 | 60–120 | Full mutation/replay audit, checkpoint pause, publication/GC protocol, journal integration |
| C: shared journal/value storage | No reliable narrow estimate | Larger research/development scope than A or B; not budgeted here | Value lifetime, reclamation, snapshots, and cold-read behavior |

The prototype means enough recovery and snapshot behavior to produce a trustworthy decision, not merely an engine benchmark behind a Put endpoint. Production estimates include format/version handling, basic migration tooling, correctness coverage, metrics, runbooks, and review. They exclude extensive mixed-stock rolling compatibility, broad multi-platform certification, a new filesystem, or a custom value-storage system. A fully new journal implementation rather than adapted primitives could exceed the B range. A2 has not been separately costed: it reuses the engine work but adds cross-store checkpoint/installation cases and placement testing. Likewise, resolving a new consensus-library defect has no reliable schedule here; retain synchronous integration if that optional branch cannot meet its safety gate. **Forecast confidence: Low.**

Implementation order within a selected alternative:

1. Specify recovery invariants, all persistent mutation categories, format versioning, and snapshots. Review the protocol independently of code.
2. Implement the conservative storage path and differential recovery checks.
3. Establish representative mixed-workload baselines; reject designs that fail the read or maintenance gates.
4. Add asynchronous apply durability and/or Raft storage pipelining one at a time, preserving the conservative mode as a diagnostic reference.
5. Complete failure coverage, migration/rollback, support tooling, and extended operational validation.

| Risk | A | B | What resolves it |
|---|---|---|---|
| Incorrect recovery or consensus persistence | High consequence; less new physical recovery machinery | High consequence; substantially more new protocol | Written invariants, fault simulation, differential and robustness tests |
| Read-tail regression | Shared LSM resources, log churn, apply backlog | Checkpoint pauses, state compaction, apply backlog | Mixed-load SLO testing at steady state and during maintenance |
| Storage growth | Log/state compaction, retained snapshots | Checkpoint generations and replay retention | Physical accounting, quotas/headroom, bounded debt and backpressure |
| Upgrade/rollback incompatibility | New engine and durable format | New journal, engine, and checkpoint formats | Versioned images, explicit migration policy and restoration drills |
| Permanent maintenance cost | Fork plus storage dependency | Fork plus checkpoint/journal protocol ownership | Dedicated owners, ongoing upstream integration and release validation |

These are qualitative engineering assessments, **Moderate confidence**, not estimated probabilities.

The remaining decision inputs are the team's actual etcd version, write/read/watch mix, transaction and payload distributions, production disk/network stack, memory limit, acceptable p99/p99.9 read change, recovery-time objective, maintenance/cutover allowance, and expertise with RocksDB/Rust. None has been silently inferred from this analysis host or from unrelated published benchmarks.

## 13. Version checks and limits of this pass

**Revision provenance:** after the user's request, I read [wal-replacement-claude.md](wal-replacement-claude.md) and [wal-replacement-agy.md](wal-replacement-agy.md) as sources of ideas. The update incorporates the split-engine comparison, revision-ordered fetching, recyclable-log/placement experiments, upstream async work and safety gates, and more concrete crash-test coverage. Code-dependent additions were checked against the pinned sources; issue/PR status was checked directly through GitHub. Their performance forecasts and correctness conclusions are not adopted as measurements or proofs. This remains my design assessment, with A as the reference and A2/B as conditional alternatives.

**Consistency audit, 2026-11-01:** rechecked the cited source files and line numbers against their release tags, version/commit pins, and all eight referenced etcd/Raft issues and PRs through GitHub. Corrected imprecise source anchors and qualified recovery replay, watch/history retention, commit-only synchronization, and the existing range-read shortcuts. Tightened the proposed A2 durability barrier to cover completed state writes in the correct database generation, and recorded the asynchronous journal API's completion requirements. The async issue descriptions now distinguish the reporters' `main` examples from mechanisms inspected in the stable Raft release. These corrections leave the architectural recommendation unchanged; they are not runtime validation.

The broader storage investigation [etcd #22332](https://github.com/etcd-io/etcd/issues/22332) remains open. [PR #22339](https://github.com/etcd-io/etcd/pull/22339) remains open and unmerged with a work-in-progress label, so its proposed correctness tests are not treated as shipped validation. The other async issue/PR statuses are recorded in §6; the closed, unmerged commit-persistence proposal is recorded in §2.2.

The existing etcd and jetcd repositories were fetched for tag/release checks without moving their checkouts. The selected latest stable versions remain etcd `v3.7.3` and jetcd `0.8.7`; jetcd is not otherwise part of this design assessment. Candidate library sources were read from temporary clones under `/tmp/ftcd-storage-20261101/`. No source checkout was edited, no Go installation was attempted, and no experiments were run. [etcd release](https://github.com/etcd-io/etcd/releases/tag/v3.7.3); [Pebble release](https://github.com/cockroachdb/pebble/releases/tag/v2.1.7); [grocksdb release](https://github.com/linxGnu/grocksdb/releases/tag/v1.11.1).

Upstream `main` was consulted only for relevant changes; inspected head `1573d59416efa220421c451f5266b24a5d8374ef`. It has snapshot-path changes outside the target release, including [commit `5d1a827c14eb88fdccd231da68259b6b6bf6288b`](https://github.com/etcd-io/etcd/commit/5d1a827c14eb88fdccd231da68259b6b6bf6288b), which stops generating/loading v2 snapshot files. The design here is based on stable code, including its existing snapshot dependencies, not on assuming that simplification has shipped.

An important version-check example: closed [issue #22483](https://github.com/etcd-io/etcd/issues/22483) concerns reversed Raft message ordering in earlier code. The relevant main commit appears in a main-versus-release history comparison, but **the inspected v3.7.3 implementation already preserves the surviving messages' original order**. It is not a current storage-replacement benefit or an unfixed defect established here. Source comparison, not commit ancestry alone, resolves that question. [`server/etcdserver/raft.go:357,406 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/raft.go#L357).

This is an architecture-level static pass, not a formal verification of Pebble, RocksDB, Raft Engine, or an unimplemented adapter. The most consequential unverified items are the complete etcd mutation/replay audit for B, the shared-engine recovery coordinator for A, and mixed-workload latency on the target platform. Those are the first implementation/measurement gates, not grounds for claiming a speedup now.
