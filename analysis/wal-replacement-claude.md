# Replacing etcd's WAL *and* storage engine: a two-store design for write throughput

- **Method:** static code analysis plus desk research; no experiments run.
- **Author:** Claude (Claude Code), 2026-11-01. This builds on `analysis/storage-claude.md`, the backend-only study by the same author, which it assumes but does not repeat. The first draft was written independently. The same day it was revised after reading the concurrently written `analysis/wal-replacement-codex.md` and `analysis/wal-replacement-agy.md`. §16 lists the ideas taken from them; each was re-checked against source before use. The other `storage-*.md` analyses were not read.
- **Versions analyzed:**
  - etcd `v3.7.3` = `0a173d0410a9`. Upstream `origin/main` @ `1573d5941` (2026-10-28) was consulted only where stated.
  - Dependencies and candidates, shallow-cloned read-only into a scratch directory outside the project:
    - raft `go.etcd.io/raft/v3` `v3.7.0` (`b867cf1`), the version etcd v3.7.3 pins in `server/go.mod:30`;
    - Pebble `v2.1.7` (`66d468e`, 2026-08-24);
    - hashicorp/raft-wal `main` @ `417bc7a` (2026-10-27);
    - lni/dragonboat `master` @ `076c7f6` (2025-07-24).
  - Nothing was built or run.
- **Upstream references:**
  - etcd issue [#16291](https://github.com/etcd-io/etcd/issues/16291) and **PR [#22223](https://github.com/etcd-io/etcd/pull/22223)** (open; adopts Raft `AsyncStorageWrites` behind an alpha gate).
  - etcd-io/raft PR [#8](https://github.com/etcd-io/raft/pull/8), which added `AsyncStorageWrites`, and the open safety reports raft [#543](https://github.com/etcd-io/raft/issues/543) and [#551](https://github.com/etcd-io/raft/issues/551).
  - etcd issue [#22332](https://github.com/etcd-io/etcd/issues/22332), in which upstream prototypes alternative storage engines, and draft PR [#22339](https://github.com/etcd-io/etcd/pull/22339) "Storage correctness", an MVCC-layer correctness test for those engines.
  - etcd-io/raft PR [#300](https://github.com/etcd-io/raft/pull/300) (closed unmerged) and etcd PR [#20055](https://github.com/etcd-io/etcd/pull/20055) (merged 2025-05-30): the applied index can exceed the durable commit index.
  - Upstream `main` commit `5d1a827c1` (2026-07-22), which stops generating v2 snapshot files.
  - Industry and academic prior art is in §11, with each source marked as read directly or second-hand.
- **Citation convention:** `path:line` refers to etcd `v3.7.3` unless prefixed `raft:` (raft v3.7.0), `pebble:` (Pebble v2.1.7), `raft-wal:` or `dragonboat:`.

## Summary

Terms used throughout:
- **WAL**: etcd's Raft write-ahead log (`server/storage/wal`).
- **Backend**: the persistent key-value store under MVCC, bbolt today.
- **Ready**: one batch of work that the Raft library hands to etcd. It contains entries to persist, messages to send, and committed entries to apply.
- **Apply loop**: the single goroutine that applies committed entries to MVCC and the backend.
- **CI** (consistent index): the Raft index of the last entry reflected in the backend, stored inside it.
- **Group commit**: many writes made durable by one `fdatasync`.
- **Log store** and **state store**: the two components of the design proposed here, replacing the WAL and the backend respectively.

1. **Premise correction: etcd's WAL is not where most of the write cost lives, so a new log *format* by itself will not raise throughput much.**
   - The WAL is already an append-only, group-committed log. All entries in one Ready are written and made durable by a single `fdatasync` (`server/storage/wal/wal.go:995-1036`).
   - While that `fdatasync` runs, Raft keeps accepting proposals, and the next Ready carries all of them together (`raft:node.go:354, :443`). Under load the per-entry fsync cost is therefore already shared across many entries. *(High: code verified.)*
   - Touching the log can buy three things, none of which needs a new log format except the second:
     - **(a) Pipelining.** etcd drives Raft in synchronous Ready/Advance mode. While a member's fsync is in flight, its Raft goroutine cannot emit the next Ready. On the leader, an entry proposed during that window reaches followers only after the leader's own fsync finishes (`server/etcdserver/raft.go:242-256, :331`). Raft's `AsyncStorageWrites` mode removes this stall (`raft:raft.go:153-187`). etcd v3.7.3 does not enable it (`server/etcdserver/bootstrap.go:540-552`).
       - **Measured elsewhere:** the Raft PR that added it reported 20–25% lower proposal latency and 73% higher maximum throughput on AWS gp3 (raft PR #8). CockroachDB measured −31% average and −44% p99 latency at moderate load, and slightly *worse* latency at saturation (cockroach PR #94165).
       - **Upstream etcd already has an open PR adopting it:** #22223, behind an alpha feature gate, needs rebase. Its author reports +25% QPS on a laptop Docker cluster.
       - **Two unreviewed safety-bug reports against the async mode were filed in the last two weeks.** raft #551: a vote request can leave before the candidate's own term and vote are durable; this applies to v3.7.0 (`raft:raft.go:546`). raft #543: a snapshot can be installed ahead of entries already handed to the apply thread. Both must be fixed or worked around before enabling the mode (§4.3, §11.1).

       *(Mechanism Moderate-High; size on your hardware Low.)*
     - **(b) Cheaper `fdatasync`.** etcd appends into freshly `fallocate`d 64 MB segments (`file_pipeline.go`, `client/pkg/fileutil/preallocate_unix.go:27`). On ext4 and XFS, the first write into such space converts an "unwritten" extent, and `fdatasync` must then commit the filesystem journal.
       - The ext4 and XFS source code confirms the mechanism (§11.3).
       - A 2025 kernel-patch measurement on ext4/NVMe found 4 KiB write+`fdatasync` ran at 20.1k IOPS into unwritten extents versus 98.8k into already-written blocks.
       - Pebble and RocksDB recycle log files to avoid exactly this (`pebble:wal/log_recycler.go`: "faster than to a new log file on some common filesystems (xfs, and ext3/4) due to avoiding metadata updates").
       - etcd's format **cannot** recycle segments without a format change. Its torn-write detection assumes unwritten space reads as zeros (`server/storage/wal/decoder.go:146-149, :168-200`), so stale bytes from a reused segment would read as corruption.

       *(Mechanism High; gain for etcd on your devices Low until experiment X1.)*
     - **(c) Less disk interference.** Removing bbolt's random page writes and fsyncs from the device that holds the log.
2. **The largest single lever is still the backend.** That is the mechanism in `storage-claude.md`: every bbolt commit runs two `fdatasync`s while holding the locks the apply loop and new readers need. Commits are forced on every lease revocation and every MVCC compaction batch. The design below includes that fix and adds (a) to (c). *(High that the mechanism exists; how much it costs your workload is unmeasured.)*
3. **Proposed design: "two stores, one synchronous write" (§4).**
   - **Log store** (replaces the WAL): the Raft log, HardState and recovery markers, in a **dedicated Pebble instance** whose write-ahead log is synced once per batch. Pebble already provides log-file recycling, chunk formats that tell real corruption apart from a torn tail, and atomic multi-key batches.
   - **State store** (replaces bbolt): a second Pebble instance written with **no synchronous writes at all**. The CI goes atomically into every batch.
   - **Coupling:** a **durable-applied checkpoint** makes the state store durable with one fsync, off the apply path, before any log prefix is discarded.
   - **Raft mode:** `AsyncStorageWrites`, so log fsyncs overlap Raft processing and network sends.
   - **Per member and per Raft batch:** one `fdatasync` (log store). Zero synchronous writes on the apply path. One state-store fsync per checkpoint interval.
   - **Commit-index updates alone are written but not synced**, as etcd's WAL does today. Recovery then raises the durable commit index to the applied index, which is safe because only committed entries are applied (§4.2, §4.7).
   - **Deployment:** put the two stores on separate filesystems, ideally separate devices. On ext4, a journal commit triggered by one file's `fdatasync` can force out other files' unsynced data. CockroachDB attributed the lack of gain in its own two-engine prototype to this (§4.2, §11.2).
   - A custom segmented log built on Pebble's `record` package is an optional later optimization (§4.2, §7). Use it only if measurement shows the second Pebble instance costs too much CPU or memory. *(Design proposal; Moderate that it is correct as specified, pending the crash testing in §5.4.)*
4. **Three verified facts shape the correctness design (§2, §5):**
   - **(i) etcd already lets the backend run ahead of durable Raft state, in two ways.**
     - It applies committed entries before they are locally durable: it sends them to the apply loop before the WAL save (`raft.go:232` precedes `:256`).
     - It does not sync a HardState change that only moves the commit index, so after a crash CI can exceed the durable commit index. Bootstrap corrects this only under `--force-new-cluster`, by resetting the commit index to max(commit, CI) (`server/etcdserver/bootstrap.go:588-600`, etcd PR #20055, after raft PR #300 was closed).

     Raft's async mode forbids applying entries that are not locally durable (`raft:rawnode.go:440-445`), which removes the first case. The design keeps the second case, for throughput, and applies the commit-index correction on every restart.
   - **(ii) Pebble's `DisableWAL` is excluded from Pebble's own crash tests.** The metamorphic suite turns it on 50% of the time but forces it off whenever crash simulation is active (`pebble:metamorphic/options.go:683, :852-853`). The state store should therefore keep Pebble's WAL on, unsynced. `DisableWAL` is an optimization to adopt only after your own power-loss testing.
   - **(iii) etcd's robustness suite already has a power-loss scenario.** LazyFS, a filesystem that drops unsynced data on demand, is cleared on every kill or panic failpoint (`tests/robustness/failpoint/kill.go:51-53`, `gofail.go:128-130`). This is the right primary gate for a design that relies on unsynced writes plus explicit sync points. Today, though, LazyFS variants are generated only for single-member, low-load scenarios, with compaction turned off (`scenarios/scenarios.go:154-169`). The multi-member and compaction paths this design changes need new LazyFS scenarios (§5.4).

   *(High for the facts.)*
5. **Read latency, the stated constraint (§4.8, §6.3):**
   - Small linearizable reads should be **unchanged or better** under write load. Their cost is dominated by the ReadIndex round, a leader-confirmation round trip, plus waiting for apply to catch up. Apply no longer stalls on backend commits, and readers are no longer blocked by the commit lock.
   - **Large ranges, serializable reads and cache-cold reads get slower.** Each returned revision is a point lookup (`server/storage/mvcc/kvstore_txn.go:108, :126`), which costs more in an LSM than in an mmapped B+tree. The upstream prototype measured Pebble at roughly 5–6.5× lower read throughput at the MVCC layer (#22332, in `storage-claude.md` §8).
   - Mitigations:
     - a block cache sized to the working set;
     - one iterator per request, fetching revisions in ascending order so Pebble can use its forward-seek optimization (`pebble:iterator.go:308`), then reordering by key;
     - optionally, a cache of decoded records keyed by revision, which needs no invalidation because revisions are immutable;
     - acceptance gates in X5.

   This is the risk most likely to violate "without making read latency meaningfully worse". *(Moderate.)*
6. **Rejected alternatives (§3):**
   - **A single Pebble instance for both log and state.** State-machine compaction stalls would block Raft log appends.
   - **grocksdb with column families and per-write WAL disable.** Needs cgo; see `storage-claude.md` §6.
   - **Using a Raft-log library from another project.**
     - hashicorp/raft-wal is MPL-2.0 and built for the hashicorp/raft interface.
     - dragonboat's `tan` sits under `internal/`, so it cannot be imported; its last commit was 2025-07.
   - **Key-value separation with the Raft log as the value log.** It halves write bytes for large values, at high complexity and read cost.
   - **An in-memory state machine with checkpoints,** as in Chubby §2.10 (`doc/chubby-osdi06.md:323`). Bounded by RAM and slow to start.
   - **Replacing Raft with an external journal,** as AWS EKS did. That is outside this scope.
7. **Recommendation: get there in stages, and expect most of the gain before the WAL format changes (§8):**
   - **S1:** Pebble state store, etcd WAL unchanged. This is option P in `storage-claude.md`.
   - **S2:** `AsyncStorageWrites`, etcd WAL format still unchanged. Start from upstream PR #22223 rather than writing it from scratch, and fix raft #551 and #543 first.
   - **S3:** log store moved to Pebble. Only if X1 shows the `fdatasync` metadata cost is material on your hardware, or you want WAL failover or stronger corruption detection.
   - **S4:** custom log. Probably never.

   Each stage ships and is measured on its own. *(Moderate for the ordering; Low for magnitudes.)*
8. **Effort** *(Low confidence; assumes 1–2 engineers strong in Go and storage, not etcd maintainers)*:
   - Prototype: S1 3–5, S2 2–4 (starting from PR #22223), S3 3–5 engineer-weeks.
   - S1–S3 to production quality: about **12–24 engineer-months** in total.
   - Ongoing fork maintenance: about **0.5–1 FTE**.

   S2 and S3 rewrite the Raft loop, bootstrap, snapshot installation and the 13 non-test files outside the `wal` package that import it (§9).
9. **Fork plan:** the log-side changes are more fork-hostile than the backend change. These files are changing on both branches. `main` alone has removed v2 snapshot files (`5d1a827c1` and follow-ups) and the `verifySnapshotIndex` assertion (`b02869083`), and changed WAL error handling (`702fd61ff`). Other recent changes to the same files were backported into v3.7.3: Ready message ordering (`e71275f78`, from `01a80f67b`) and `MinimalEtcdVersion`'s WAL read (`23b61e6d9`, from `2e2963af6`).
   - Pin to `release-3.7`.
   - Get S2 from upstream: review, benchmark and help land PR #22223 (issue #16291) instead of maintaining your own.
   - S1 aligns with upstream issue #22332.
   - Treat S3 as fork-only (§14).

---

## 1. Premise check: what the WAL costs today, and what replacing it can buy

### 1.1 The WAL as implemented (verified at v3.7.3)

| Aspect | Behavior | Evidence |
|---|---|---|
| Segments | 64 MB (64×10⁶ bytes) files. A background goroutine pre-creates the next file as `0.tmp`/`1.tmp` and preallocates it with `fallocate(mode 0)`, which extends the file with unwritten extents on ext4 and XFS. The file is renamed into place at cut. | `wal.go:55`; `file_pipeline.go` (`alloc`, `run`); `fileutil/preallocate_unix.go:25-33` |
| Framing | An 8-byte little-endian length, whose top byte flags padding, then a protobuf `walpb.Record{type, crc, data}` padded to 8 bytes "so length never gets a torn write". | `encoder.go` (`encode`, `encodeFrameSize`) |
| Checksums | CRC-32C chained across all records and across segments. Each new segment starts with a CRC record carrying the previous value. | `wal.go:65, :808-818, :1063-1065` |
| Buffering | A 128 KiB page-aligned write buffer with 4 KiB pages. | `pkg/ioutil/pagewriter.go:23`; `encoder.go:36` (`walPageBytes = 8 * minSectorSize`) |
| Sync policy | One `fdatasync` per `Save` if there are entries or the term or vote changed. A HardState change that only moves the commit index is written but **not** synced. | `wal.go:1010, :1026-1033`; `raft:rawnode.go:189-198` |
| Cut | At ≥64 MB: truncate, `fdatasync`, take the preallocated file, write CRC, metadata and HardState, `fdatasync`, rename, fsync the directory. | `wal.go:785-866` |
| Snapshot marker | `SaveSnap` writes the snapshot file first, then a WAL snapshot record, then `fdatasync`. | `storage.go:59-76`; `wal.go:1039-1061` |
| Truncation | `ReleaseLockTo(snapshot index)` drops locks on older segments. A purge goroutine deletes unlocked segments beyond the newest 5 every 30 s. | `wal.go:904-946`; `server.go:606`; `embed/config.go:63` |
| Recovery | `ReadAll` reads from the segment holding the snapshot record. A later entry with a lower index overwrites the earlier suffix. A zeroed record means end of log. A CRC failure counts as a torn write only if some sector of the record is all zeros, and only in the last file; any other CRC failure is fatal. Bootstrap then repairs a torn write once by truncating the last segment at the last valid record. The tail is zeroed before appending resumes. | `wal.go:472-595`; `decoder.go:88-150, :168-200`; `bootstrap.go:645-649`; `repair.go:67-98` |
| I/O errors | Any `Save`, `SaveSnap`, `Sync` or `Release` failure calls `lg.Fatal`. | `raft.go:249-283` |
| Seam | `storage.Storage`: `Save`, `SaveSnap`, `Close`, `Release`, `Sync`, `MinimalEtcdVersion`. The last one re-reads the whole WAL. | `storage.go:29-42, :109-125` |

### 1.2 The Raft loop around it (verified)

Per Ready, `server/etcdserver/raft.go` runs these steps in order:
1. Hand committed entries (and any snapshot) to the apply loop: `applyc <- ap` at `:232`.
2. **Leader only:** send messages to followers at `:242`, before its own disk write. This is the Raft thesis §10.2.1 optimization.
3. Save an incoming snapshot (`:249`), then **`storage.Save(HardState, Entries)`, i.e. WAL write plus `fdatasync`** (`:256`).
4. For an incoming snapshot: `Sync`, signal the apply loop, `ApplySnapshot` into `MemoryStorage`, then `Release` the WAL (`:269-283`).
5. Append the entries to Raft's in-memory log, `MemoryStorage` (`:287`).
6. **Follower only:** send messages after the save, so acknowledgements always follow persistence. If a committed configuration change is pending, first wait for it to be applied (`:289-324`).
7. `Advance()` (`:331`).

The Raft library offers the next Ready only after `Advance` (`raft:node.go:354` arms `readyc` only when `advancec == nil`). Proposals and incoming messages are still stepped into Raft's in-memory state meanwhile, which is what produces the natural batching.

The leader counts itself toward a quorum only after its own entries are durable ("self-ack ... once they have been durably persisted", `raft:raft.go:835-842`).

Entries are applied concurrently with the WAL write of the same Ready. Each apply job then waits for that write to finish before checking whether to snapshot (`server.go:980-985`). Apply jobs run one at a time on a FIFO scheduler (`server.go:843-845`; `pkg/schedule/schedule.go`), so the next Ready's entries start applying only after the previous Ready's WAL write completes.

### 1.3 Where write cost goes, per member

| Cost | Today | Notes |
|---|---|---|
| Synchronous device flushes on the Raft critical path | 1 WAL `fdatasync` per Ready. On ext4/XFS, because segments are fresh `fallocate` space, it also commits a journal transaction for the unwritten-to-written extent conversion. | Journal mechanism: *High* (ext4 and XFS source, RocksDB and Pebble rationale; §11.3). Size on your device: unknown (X1). |
| Synchronous device flushes on the apply path | 2 bbolt `fdatasync`s per backend commit: every 100 ms, **plus one per entry that deletes** (lease revoke) and per 1,000-key compaction batch. Apply and new read transactions are blocked meanwhile. | `storage-claude.md` §2.3 |
| Raft goroutine blocked on disk | For the whole `fdatasync`, on every member. | §1.2 |
| Segment rotation | Every 64 MB of log, the cut runs inside `Save` on the Raft goroutine: 2 `fdatasync`s, a rename and a directory fsync. A periodic tail-latency spike rather than a throughput limit. | `wal.go:785-866`, `:1036` |
| Bytes written per value byte | WAL ≈1× plus framing, and bbolt copy-on-write pages: leaf pages plus the branch path plus meta per commit. Total probably 2–3×, much of it random 4 KiB writes. | Low confidence; depends on value size and commit size. |
| Interference | bbolt's random writes and fsyncs share a device with the WAL unless `--wal-dir` points elsewhere. | |

### 1.4 Consequence for the design

Rewriting the WAL alone attacks the smallest of these costs. A design that **replaces both** is worthwhile only because it can make the Raft log the *only* synchronous write, and because the integration point (`storage.Storage`, the Raft loop, snapshot install, recovery) has to be reopened anyway to do that properly. Throughput gains will come from four places, in decreasing expected size:
1. no synchronous backend commits on the apply path;
2. pipelined log writes;
3. no bbolt I/O interfering with log fsyncs;
4. cheaper `fdatasync` on recycled files.

*(Ordering: Moderate. It depends on workload, and E0/X0 decide it.)*

## 2. The correctness contract any replacement must keep

These invariants are written so that each can be checked against code and tests. The "today" column shows how etcd v3.7.3 satisfies each one.

| # | Invariant | How v3.7.3 satisfies it |
|---|---|---|
| I1 | **Persist before acknowledging.** Term, vote and log entries are durable before any message that depends on them leaves the node: vote grants, `MsgAppResp`, the leader's self-ack. | Synchronous `Save` before follower sends (`raft.go:256` before `:324`); `MustSync` (`raft:rawnode.go:189-198`); leader self-ack after persistence (`raft:raft.go:835-842`) |
| I2 | **Log integrity.** After a crash the durable log is the sequence of written batches with suffix replacement. A torn tail is discarded. **Corruption inside the synced region is detected and is fatal, never silently treated as end-of-log.** Silently dropping an entry that was acknowledged breaks Raft safety. | `ReadAll` suffix replacement (`wal.go:487-505`); torn-tail versus CRC distinction (`decoder.go:146-149`); fatal `ErrCRCMismatch` |
| I3 | **CI is atomic with state.** The applied index stored in the state store describes exactly the state stored with it. | CI and ConfState written inside the bbolt transaction by the pre-commit hook (`storage/hooks.go:45-54`); see `Documentation/postmortems/v3.5-data-inconsistency.md` |
| I4 | **State durable before log truncation.** Entries ≤ i are discarded only after the state store is durable through ≥ i **and** a durable marker records (i, term, ConfState), so recovery knows where the log starts. | `KV().Commit()` before `SaveSnap` and `Release` (`server.go:2096-2130`) |
| I5 | **No gap at recovery, and a consistent commit index.** Durable CI ≥ (first retained log index − 1). Today CI may also *exceed* the last durable log index, because entries are applied before the local WAL write, and may exceed the durable commit index, because commit-only HardState changes are not synced. Both must be tolerated. | No bootstrap check of CI against the last log index (apply-before-save, `raft.go:232`, `:256`); commit reset to max(commit, CI) only under `--force-new-cluster` (`bootstrap.go:588-600`) |
| I6 | **Idempotent replay.** Entries ≤ CI are not re-applied to the state store. | `server.go:1907-1912` (`shouldApplyV3` only when `index > CI`) |
| I7 | **Atomic snapshot install.** A received snapshot's state and its log marker become durable as one unit with respect to a crash. A partial snapshot is never used. | Snapshot file before WAL record (`storage.go:63-76`); `RecoverSnapshotBackend` redoes the database swap on restart (`storage/backend.go:102-114`) |
| I8 | **Membership consistency.** ConfState is stored atomically with state and in log markers. | `hooks.go:45-54`; `walpb.Snapshot.ConfState` (`storage.go:61-65`) |
| I9 | **Fail-stop on I/O error.** A failed write or fsync on the log is fatal, and fsync is never retried, because after a failed fsync the page-cache state is undefined. | `lg.Fatal` (`raft.go:249-283`) |
| I10 | **Deterministic logical state.** State-machine results and `HashKV` do not depend on the storage engine or physical layout, so cross-member corruption checks keep working. | `HashKV` hashes logical key-value pairs (`storage-claude.md` §5.5) |
| I11 | **Holds under power loss, not just process crash.** Every claim above must survive losing all unsynced writes, in any order, at any instant. | LazyFS robustness scenario (`scenarios.go:154-169`; `kill.go:51-53`) |

## 3. Design space

### 3.1 Options considered

| Id | Log | State | Sync writes per Raft batch (per member) | Notes |
|---|---|---|---|---|
| **A** | etcd WAL (sync Ready loop) | Pebble, unsynced (= `storage-claude.md` option P) | 1 WAL fsync, plus 0 on the apply path | Leaves the WAL alone; captures the backend lever. |
| **A+** | etcd WAL driven by `AsyncStorageWrites` | Pebble, unsynced | 1, overlapped with Raft processing | No log format change. |
| **B** (proposed) | **Dedicated Pebble instance** (WAL synced per batch, recycled files, sync-offset chunks) | **Pebble, WAL on but unsynced** | 1 (data-only `fdatasync` on recycled files) | Two instances: log-side stalls and state-side stalls are isolated. |
| B-L | Custom segmented "ledger" on Pebble's `record` package | Pebble, unsynced | 1 | Saves B's memtable copy and CPU; new crash-sensitive code. Optimization only. |
| C | One Pebble instance holds both log and state | (same instance) | 1, but each fsync also carries the state bytes written since the last one | CockroachDB's current production layout; its separate-engine design is unshipped (§11.2). State compaction stalls block log appends. |
| D | One RocksDB instance via grocksdb: log column family synced, state column family with per-write WAL disable | (same) | 1 | Needs cgo (`storage-claude.md` §6). RocksDB-only per-write `disableWAL`. |
| E | Ledger as the **value log** (key-value separation) | Pebble holds key → (segment, offset) | 1 | Values written once. Needs value-log GC tied to MVCC compaction; snapshots must materialize values. |
| F | Ledger | **In memory**, checkpointed periodically | 1 | Chubby's rewrite was a simple write-ahead-log-plus-snapshot database (Chubby §2.10, `doc/chubby-osdi06.md:323`). RAM-bound; full-state checkpoints; slow restart. |
| G | External replicated journal replacing Raft | Engine on tmpfs | 0 local | AWS EKS's approach (`storage-claude.md` §8). A different system, out of scope. |

### 3.2 Assessment

| | Write throughput upside | Read latency risk | Correctness risk | Effort | Compatibility |
|---|---|---|---|---|---|
| A | Removes apply-path fsyncs (largest lever when apply-bound) | Ranges and cold reads slower (LSM) | Moderate (prefix property of unsynced state; snapshot format) | `storage-claude.md` estimate | WAL tools unchanged |
| A+ | A plus up to ~1 fsync of commit latency removed (§6.2) | Same as A | Moderate-high (Raft loop rewrite: snapshot and conf-change ordering; open raft #551/#543) | A + 2–4 weeks prototype (from PR #22223) | Unchanged on disk |
| **B** | A+ plus cheaper fsync (device-dependent), WAL failover option, log/state isolation | Same as A | Moderate (all crash-recovery code is Pebble's; etcd protocol is new) | A+ + 3–5 weeks prototype | New on-disk log; tools must be ported |
| B-L | ≈B; saves CPU and memory | Same | Higher (new log code) | B + 2–3 months | Same as B |
| C | ≈B on fsync count; worse fsync size; stalls couple | Same | Moderate | ≈B | Same as B |
| D | ≈C | Same | Moderate | ≈B + cgo/build work | Same as B |
| E | Saves ~1× value bytes; only matters if bandwidth-bound | **Worse**: every value read is a ledger `pread` | High | 6–12 months more | Snapshot format changes again |
| F | Highest (no engine on apply path) | Best for in-memory reads | Moderate | High | Restart time and RAM bound the dataset |

### 3.3 Why separate log and state stores (B over C)

- **Failure isolation.** In C, a state-machine write stall blocks Raft log appends. Causes include too many L0 files, a full memtable queue, or compaction debt after a large MVCC compaction. A blocked append blocks follower acknowledgements, the leader's self-ack and vote persistence. In B, a state-store stall blocks only the apply loop. etcd already turns apply lag into client backpressure (`ErrTooManyRequests` when apply trails commit by >5,000 entries, `server/etcdserver/v3_server.go:51-53`, enforced in `util.go:60-69`; lease revocations may get 10,000 when prioritization is enabled), and the Raft log, heartbeats and elections continue. *(Moderate: the Pebble stall triggers are verified as options, `pebble:options.go:845, :910`; whether they fire on your workload is unmeasured.)*
- **fsync payload.** In C, every synced log batch also carries the state-machine bytes written since the previous sync. This roughly doubles the bytes per critical-path fsync. In B, the state store writes its WAL without syncing, off the critical path.
- **Tuning.** The two stores have opposite needs. The log is a FIFO that is mostly deleted before it is flushed. The state is a long-lived, read-heavy index. Separate instances get separate memtable sizes, caches and compaction settings.
- **Snapshot contents.** A snapshot sent to another member must carry replicated state only, never the sender's HardState, vote or uncommitted log. In B the state instance holds none of those, so any whole-instance view or hard-link checkpoint is a valid snapshot payload. In C the export must be filtered by key. Pebble's span-restricted checkpoint does not do that filtering: it "can still surface keys outside of these spans", and those keys may be stale (`pebble:checkpoint.go:50-56`). *(Point from the codex analysis; Pebble comment verified.)*
- **Cost of separation:** two Pebble instances, so two sets of background goroutines and file handles. Also, no atomic batch can span log and state, which is why I4 needs an explicit protocol (§4.5).

## 4. Proposed design (option B)

### 4.1 Overview and on-disk layout

```
<data-dir>/member/
  log/          Pebble instance #1: Raft log + HardState + markers   (synced WAL)
  state/        Pebble instance #2: MVCC + meta/lease/auth/members    (unsynced WAL)
  snap/         staging area for incoming Raft snapshots
  FORMAT        layout version; etcd refuses to start on a mismatch
```

Components:
- **Raft node** in `AsyncStorageWrites` mode.
- **Append thread:** consumes `MsgStorageAppend` in order and writes to the log store.
- **Apply thread:** the existing apply loop. It consumes `MsgStorageApply` in order and writes to the state store.
- **Checkpointer:** advances the durable-applied index and truncates the log.
- **Snapshot receiver and sender.**

`MemoryStorage` stays as Raft's read-side `Storage` for now. It already holds the catch-up window (`DefaultSnapshotCatchUpEntries = 5000`, `server.go:85`). That keeps the change small; serving Raft reads directly from the log store is a later optimization.

### 4.2 Log store

**Keys** (instance #1):

| Key | Value |
|---|---|
| `l/<index, 8-byte big-endian>` | marshaled `raftpb.Entry` |
| `m/hardstate` | `raftpb.HardState` |
| `m/truncated` | {index, term}: the log starts after this |
| `m/checkpoint` | {index, term, ConfState}: the state store is known durable through this index |
| `m/snapshot` | {index, term, ConfState, staging id}: written only at snapshot install |
| `m/meta` | cluster ID, member ID, format version: what WAL metadata holds today |

**Append** (one Pebble batch per drain of the queue):
- If the first entry's index ≤ the current last index: `DeleteRange(l/first, l/∞)`. This is suffix replacement, the same semantics as `ReadAll` (`wal.go:487-505`), but explicit and atomic.
- `Set(l/i, entry)` for each entry.
- `Set(m/hardstate)` if the HardState is non-empty.
- **Sync rule.** Commit with `Sync` if any drained message carries entries or a snapshot, or changes the term or vote. This is the rule `raft.MustSync` encodes (`raft:rawnode.go:189-198`). Otherwise commit with `NoSync`.
  - Why it matters: in async mode, a HardState change that only moves the commit index still produces a `MsgStorageAppend` (`raft:rawnode.go:200-208, :230-241`). Syncing every batch would add an `fdatasync` per commit-index advance, which on the leader is roughly one per replication round. etcd's WAL skips that sync today (`wal.go:1010`). My first draft synced every batch; both other analyses specify the `MustSync` rule.
  - Safety: an unsynced batch carries no new entries. Any `MsgAppResp` it releases acknowledges entries from earlier messages, and the append thread committed those with `Sync` before it started this batch.
  - Consequence: the durable commit index can trail the applied index after a crash. Recovery corrects it (§4.7 step 7).
- After the commit returns: append the entries to `MemoryStorage`, then deliver the message's responses: the self-ack (`MsgStorageAppendResp`), `MsgAppResp`, vote responses. The order matters for I1: Raft may read stabilized entries from `Storage` after the self-ack.
- Group commit: the append thread drains every queued `MsgStorageAppend` into one batch before committing.
- **Single writer.** Only the append thread writes to the log store. The checkpointer's marker and truncation (§4.5) go through the append thread's FIFO, and only the append thread writes `m/hardstate`, so no other path can overwrite a newer term or vote with an older one.

**Pebble configuration** for this instance:
- format major version ≥ `FormatWALSyncChunks`, so the WAL reader can tell corruption from logical end of log (`pebble:format_major_version.go:201-206`; chunk format in `pebble:record/record.go:87-100`). That is I2.
- WAL recycling is on by default (`MaxNumRecyclableLogs = MemTableStopWritesThreshold + 1`, `pebble:open.go:355`).
- memtable sized so that the log is truncated before it is flushed (see below).
- optional `WALFailover` to a second device for disk-stall tolerance (`pebble:options.go:1036-1041, :1286-1290`).

**Why a Pebble instance and not a custom log first.**
- Every crash-recovery path is Pebble's existing, crash-tested code. CockroachDB runs it in production as its WAL (§11.2, §11.4).
- etcd adds no file-format code, only key layout and protocol. For "correctness is a must", that is the deciding factor.
- Costs:
  - each entry is copied into a memtable, which takes CPU and duplicates `MemoryStorage`;
  - entries still live when a memtable flushes are written again to an SST file.
- **Design rule to keep the second cost near zero:** checkpoint and truncate by log bytes, at intervals well below the memtable size. For example, a 64 MiB memtable with a checkpoint every ≤16 MiB of log. Most entries are then deleted while still in the memtable, and Pebble's flush drops keys covered by a range deletion in the same memtable. *(Moderate: inferred from Pebble's flush semantics, not tested.)*
- The dragonboat `tan` and TiKV raft-engine critiques of LSM-based Raft logs (§11.2: redundant memtables, keys and serialization) are aimed at multi-Raft systems with thousands of groups. etcd has one group. *(Low-Moderate that the overhead is negligible for etcd; X2 measures it.)*
- **The closest precedent is CockroachDB's unshipped separate-engine design** (§11.2, cockroach#16624, which is still open). It too keeps a WAL in *both* engines. The state-machine WAL is "very rarely synced", and the log is truncated only after the state engine reports a flush. That is the same coupling as §4.5.
- One lesson from that thread changes deployment advice. Their two-engine prototype showed **no throughput gain over a single engine**, which they attributed to "the behavior of ext4, where an fdatasync flushes everything in the buffer cache". My inference: under ext4's default `data=ordered` mode, a journal commit triggered by the log's `fdatasync` also forces out other files' dirty data that is tied to the same transaction. The "unsynced" state-store writes are then not free if both stores share an ext4 filesystem. *(Mechanism: Moderate.)*
  - Two things cut this coupling: putting the log store and state store on **separate filesystems**, and recycled log files, whose `fdatasync` needs no journal commit.
  - X1(d) measures it.

**Custom ledger (B-L), only if X2 shows instance #1 costs matter.**
- Segments made of Pebble `record.LogWriter` chunks (`pebble:record/log_writer.go:414-427, :538-541`) using the WAL-sync chunk format. One record per append batch, carrying {HardState, entries, optional marker}. The current HardState goes into every segment header, as etcd does at cut (`wal.go:820-826`).
- Recycling: a segment becomes reusable once its last index ≤ the truncation point and it is older than the segment holding the newest durable marker. Stale chunks are rejected by log number (`pebble:record/log_writer.go:414-417`).
  - **Keep a fixed ring of segment files and never rename them.** The log number in every chunk identifies the generation, so recovery can order segments by the log number in each file's first chunk. Rotation then changes no directory entry and no file size, so it needs no directory fsync and no journal commit. Pebble's own rotation does both (§6.1), and so does etcd's cut. *(Idea from the agy analysis. Moderate: a torn or unsynced first chunk in the newest segment must be read as "generation never started", which needs its own crash tests.)*
- New, never-used segments are zero-filled once with real writes rather than `fallocate`, so even first use avoids extent conversion.
- This is 2–3 engineer-months of new crash-sensitive code. hashicorp/raft-wal's ALICE crash-exploration setup (`raft-wal:alice/README.md`) is a model for testing it.

### 4.3 Raft integration (`AsyncStorageWrites`)

Raft's contract, from `raft:raft.go:153-187`:
- `MsgStorageAppend` carries entries, HardState and snapshot. Its writes must be durable before its responses are delivered.
- `MsgStorageApply` carries committed entries. Durability is not required before responding.
- Messages to the same thread must be processed in order.

In async mode, committed entries are released for applying only once they are locally durable (`applyUnstableEntries() = !asyncStorageWrites`, `raft:rawnode.go:440-445`).

New Raft goroutine loop:
1. Receive a Ready.
2. Send network messages immediately. Raft has already attached any message that depends on persistence as a response inside `MsgStorageAppend`.
3. Forward `MsgStorageAppend` to the append thread and `MsgStorageApply` to the apply thread.
4. Step responses back into Raft as they arrive.
5. Never call `Advance` (`raft:rawnode.go:475-482` panics if it is called in async mode).

**Two open Raft-library safety reports against this mode must be closed first** (§11.1). Neither has a maintainer response yet. I checked both reports and the relevant v3.7.0 code.
- **raft #551: a vote request leaves before the candidate's term and vote are durable.**
  - `send()` holds back only `MsgAppResp`, `MsgVoteResp` and `MsgPreVoteResp` until the storage write completes (`raft:raft.go:546`). `MsgVote` therefore goes out in `Ready.Messages` immediately.
  - Suppose a candidate wins, crashes before its HardState is durable, restarts at the old term, and wins the same term again with a different log. One term then has two leaders' entries.
  - Fix in Raft (proposed PR #550), or work around it in etcd: hold any outgoing `MsgVote` until the `MsgStorageAppend` from the same Ready has been made durable.
- **raft #543: a snapshot can overtake entries already handed out.**
  - With the snapshot installed by the append thread, entries already handed to the apply thread can be applied *on top of* the newer snapshot.
  - The design avoids this structurally:
    - the append thread only commits the snapshot marker;
    - the state swap is queued into the **apply thread's FIFO**, behind every `MsgStorageApply` handed out earlier (§4.6);
    - etcd's CI skip (I6) is a second line of defense.
- Whether upstream PR #22223 handles either issue: not checked.

Existing etcd orderings that must be re-expressed rather than lost:
- **Wait for configuration changes on followers.** Before sending messages, a follower waits for pending configuration changes to be applied, "otherwise we might incorrectly count votes" (`raft.go:296-319`). In async mode the equivalent is to hold back outgoing *follower* messages released after an append that contained a committed configuration change, until the apply thread reports it applied. *(High that this needs explicit handling; the exact mechanism is a design task.)*
- **Snapshot persisted before it is applied.** The apply loop waits on `notifyc` before replacing the backend (`server.go:1031`). See §4.6.
- **Leader send before persist.** Preserved automatically: Raft emits `MsgApp` to followers in the Ready, not as storage responses.
- **Every robustness failpoint at these steps** (`raftBeforeSave`, `raftAfterSave`, `raftBeforeLeaderSend`, `raftBeforeFollowerSend`, `raftBeforeApplySnap`, `raftAfterWALRelease`, … in `tests/robustness/failpoint/gofail.go:38-60`) needs an equivalent in the new loop. Otherwise the robustness suite silently loses coverage.

### 4.4 State store

- **Keys** (instance #2): one prefix byte per current bbolt bucket (`key`, `meta`, `lease`, `alarm`, `cluster`, `members`, `members_removed`, `auth`, `authUsers`, `authRoles`; `schema/bucket.go:25-56`). Within a prefix, the same key bytes as today, which keeps revision order.
- **Write path** (apply thread):
  - The entries of one `MsgStorageApply` are applied into one **indexed** Pebble batch, so later operations in the same etcd transaction, and later entries in the group, read earlier writes.
  - CI, term and ConfState are `Set` in the same batch (I3).
  - Commit with `NoSync`.
  - Pebble publishes the batch to readers once it is in the memtable (`pebble:commit.go:144-205`).
  - MVCC's `currentRev` is advanced only after the batch commit returns. Readers bounded by `currentRev` therefore never look up a revision that Pebble does not yet show. This replaces the read-buffer writeback that bbolt mode needs under the read lock (`batch_tx.go:310`).
- **Durability mode:**
  - Default: **Pebble WAL on, unsynced.** On crash, Pebble replays its WAL up to the first invalid record. The recovered state is therefore a prefix of committed batches, and this is the code path Pebble crash-tests.
  - Cost: state bytes are written twice, once to Pebble's WAL and once at flush. This matters on bandwidth-capped volumes (§6.1).
  - `DisableWAL` saves the WAL bytes, but its crash behavior is not covered by Pebble's crash tests (`pebble:metamorphic/options.go:852-853`). With it, `Sync` writes are rejected (`pebble:db.go:852-854`), so the durability point becomes `Flush()`. Adopt it only after X6 passes with it.
- **Deletes:** MVCC compaction and lease revocation become ordinary unsynced deletes. No forced commit (removes `batch_tx.go:335`'s effect).
  - Every revision key is written exactly once, so Pebble `SingleDelete` would let a tombstone and its key cancel out early. Pebble's semantics are undefined if a key was set twice, so treat it as a later optimization with an invariant check, not a default.
- **Writes outside the apply path, and today's forced commits.**
  - etcd forces backend commits outside the 100 ms cycle at a few call sites. Map each one by its purpose:
    - Where the forced commit exists only because bbolt's read buffer cannot hold deletes, use an ordinary unsynced commit. Example: `mvcc/kvstore_compaction.go:90`, "instead of letting them accumulate in the write buffer".
    - Where it promises durability to a caller, use `state.LogData(nil, pebble.Sync)`. Examples: `v3_server.go:413`, which ensures a finished physical compaction does not resume after a crash; `mvcc/kvstore.go:226`, which persists the scheduled compaction.
    - The remaining call sites are initialization paths in lease, auth and alarm.
  - These calls are rare, so the conservative mapping costs nothing measurable.
  - Other writes outside apply need no sync. Examples: background compaction batches, and the storage-version monitor's schema update (`schema/schema.go:53-57`, `LockOutsideApply`).
    - Pebble's WAL replay is prefix-preserving, so a crash yields the state at one earlier instant, just as a bbolt commit boundary does.
    - Those tasks resume from persisted progress, as they do today.
  - *(Moderate. The codex analysis raised the general obligation to classify every write outside apply. For this design it reduces to stock etcd's obligations plus this mapping.)*
- **Tuning starting points** *(Low confidence; to be set by X4/X5)*:
  - memtable 64 MiB (Pebble's default is 4 MiB, `pebble:options.go:1477`);
  - block cache ≥ the hot working set (default 8 MiB, `pebble:options.go:40`);
  - compression on;
  - value separation off by default. It exists in v2.1.7 (`pebble:options.go:1162-1200`, `FormatValueSeparation`) but adds a second read for separated values.

### 4.5 Durable-applied checkpoints and log truncation (I4)

The checkpointer runs periodically: by bytes of log appended (rule in §4.2), and at least every T seconds.
1. **Capture.** In the apply thread, after a batch commit, capture `(a, term(a), ConfState(a))` for the last applied index `a`.
2. **Sync the state store.** Off the apply thread, make the state store durable through `a` with `state.LogData(nil, pebble.Sync)`, the same barrier Pebble's own checkpoint uses (`pebble:checkpoint.go:43, :167`). Pebble writes batches to its WAL in sequence order under one mutex (`pebble:commit.go:150-205`), so a synced record makes every earlier batch durable. In `DisableWAL` mode, use `state.Flush()` instead.
3. **Mark.** After step 2 completes, enqueue `Set(m/checkpoint, {a, term, ConfState})` into the append thread's FIFO; it rides the next append batch. It needs no sync of its own. Pebble replays its WAL in order, so the marker is durable whenever the later truncation is, and losing both leaves an older marker with more log.
4. **Truncate.** `DeleteRange(l/0, l/a+1)` plus `Set(m/truncated, {a, term(a)})`, through the same FIFO. This does not need its own sync either. If it is lost, recovery simply finds extra log, which is harmless.
5. **Memory.** Compact `MemoryStorage` as today (`compactRaftLog`, `server.go:2139-2170`), still retaining the catch-up window. Memory retention and disk retention are now independent.

Why this order is safe: a log prefix is deleted only after a durable marker says the state store covers it, and only after the state store was synced through that point. A crash between any two steps leaves either an older marker with more log, or a newer marker whose state is already durable.

**Optional: a disk catch-up window longer than the in-memory one.**
- Today a follower more than 5,000 entries behind must receive a full snapshot (`server.go:80-85`). At the ~10k writes/s that comment assumes, that is half a second of lag. A snapshot transfer streams the whole database and costs leader CPU, disk and network while writes continue.
- Alternative: delay step 4 to retain a configured amount of log, say a few hundred MiB. Then serve `raft.Storage.Entries` below `MemoryStorage`'s first index from the log store. Lagging followers then catch up from the log.
- Costs:
  - retained entries outlive memtable flushes and are written to SSTs, so the §4.2 design rule no longer holds for them;
  - a new `raft.Storage` implementation is needed.
- *(Idea from the agy analysis. Value: Moderate for clusters where followers restart or partition often. Not part of the S3 baseline.)*

This replaces etcd's current disk snapshot (`snapshot()` → `KV().Commit()` → `SaveSnap` → `Release`, `server.go:2083-2130`). Upstream `main` has already stopped writing v2 `.snap` files (`5d1a827c1`), so "snapshot" there already means backend plus a WAL record. The design matches that direction.

### 4.6 Raft snapshots (send and install; I7)

**Send** (leader to a lagging follower):
- Open a Pebble `Snapshot` (`pebble:db.go:1629`) in the apply thread at applied index `a`, so the view and its index match.
- Stream it in one of two formats:
  - **(1) bbolt file.** Interchange format: compatible with old-format members and with `etcdctl snapshot save` / `etcdutl snapshot restore`, which expect a bbolt file with an optional SHA-256 trailer (`server/etcdserver/api/v3rpc/maintenance.go:135-209`; `etcdutl/snapshot/v3_snapshot.go:94-98`). Costs CPU and temporary space to build.
  - **(2) SST stream.** Between new-format members: sorted key-value pairs written as SSTs and ingested by the receiver.
- Use (1) during migration and for backups, and (2) once all members are new (`storage-claude.md` §5.4).

**Install** (follower):
1. The rafthttp receiver writes the incoming state into a fresh Pebble directory under `snap/<id>`, ingests and syncs it, and fsyncs the directory. This happens before Raft sees `MsgSnap`, as `SaveDBFrom` does today.
2. The append thread, handling `MsgStorageAppend{Snapshot}`, commits one synced log-store batch: `DeleteRange(l/0, l/∞)`, `Set(m/snapshot, {index, term, ConfState, id})`, `Set(m/truncated, {index, term})`, `Set(m/hardstate)`. **This batch is the commit point.**
3. After the marker commit, the append thread enqueues a "swap" item into the apply thread's FIFO. It lands behind all earlier `MsgStorageApply` items, which avoids raft #543. When the apply thread reaches it:
   - close the old state store;
   - rename `state` → `state.old` and `snap/<id>` → `state`, then fsync `member/`;
   - reopen, and rebuild `treeIndex` and MVCC as `applySnapshot` does today;
   - delete `state.old`.
4. **Recovery redo:** if `m/snapshot.index` > the state store's CI and `snap/<id>` exists, redo step 3. If `snap/<id>` is missing, the store is corrupt, which is fatal; it cannot happen if step 1 synced. This mirrors `RecoverSnapshotBackend` (`storage/backend.go:102-114`).

### 4.7 Recovery algorithm

1. Read `FORMAT`; refuse to start on a mismatch.
2. Open the state store. Pebble replays its unsynced WAL to the first invalid record. Read CI `s`, its term and ConfState.
3. Open the log store. Read `m/meta`, `m/hardstate`, `m/truncated` (= `t`), `m/checkpoint`, `m/snapshot`, and the entries `(t, last]`.
4. Apply the snapshot redo rule (§4.6).
5. Check **`s ≥ t`**. A state store behind the log's start means lost committed data, which is fatal; I4 guarantees it cannot happen without storage corruption.
6. Check **`s ≤ last`**. In async mode the state store never runs ahead of the local log, for three reasons:
   - Raft hands over only locally durable entries (`raft:rawnode.go:440-445`);
   - applied entries are committed, so suffix replacement never removes them;
   - truncation removes only entries ≤ the checkpoint ≤ `s`.

   Stock etcd can violate this check, because it applies before the local WAL save and has no bootstrap check. Data converted from the old format may therefore need an exemption.
7. **Reconcile the commit index:** set `HardState.Commit = max(HardState.Commit, min(s, last))`.
   - Why it is needed: commit-only HardState writes are unsynced (§4.2), so the durable commit index can trail entries already applied. Any logic that treats entries above the commit index as uncommitted would then discard committed entries. raft PR #300 shows exactly this under `--force-new-cluster`.
   - Defense in depth: Raft panics at start if the commit index is below its snapshot index or above its last index (`raft:raft.go:2037-2040`). FIFO routing of the markers (§4.5) already keeps the commit index ≥ `t`; the max makes that independent of routing.
   - Why it is safe: an applied entry was committed.
   - Precedent: pav-kv proposed this rule in raft PR #300. etcd applies it today, but only under `--force-new-cluster` (`bootstrap.go:588-600`, etcd PR #20055). *(The codex analysis flagged the commit-index subtlety; the fix and its precedent are verified.)*
8. Build `MemoryStorage`: `ApplySnapshot({t, term(t), ConfState})`, `SetHardState`, `Append(entries)` (same as `bootstrap.go:702-710`). Restart Raft. Entries ≤ `s` are skipped by apply (I6).
9. Rebuild `treeIndex` by scanning the state store, as today (`kvstore.go:327-372`).

### 4.8 Read path (the "no meaningful read regression" constraint)

- **Linearizable read** = ReadIndex round + wait until applied ≥ read index + engine lookups.
  - The middle term shrinks under write load, because apply never blocks on a commit.
  - Readers are no longer excluded during a commit, which today holds `readTx.Lock` (`batch_tx.go:354-359`).
  - The first term is unchanged.
- **Engine lookups.** A Range at revision `rev` gets `(key, revision)` pairs in key order from `treeIndex` (`kvstore_txn.go:108`), then does one point lookup per revision (`:126`). Design rules:
  - **one Pebble iterator per request**, not one `Get` per revision;
  - **sort the revisions ascending** and fetch with successive `SeekGE`, so Pebble can use its forward-seek optimization (`TrySeekUsingNext`, `pebble:iterator.go:308`) instead of descending the tree for every key;
  - reorder into key order before returning.
- **Block cache** sized so that steady-state reads hit memory: Pebble caches uncompressed blocks. **Bloom filters** help point lookups that miss in a level.
- **Optional revision-keyed record cache.**
  - What: a cache of decoded key-value records keyed by revision.
  - Why it needs no invalidation: revisions are immutable, so writes never invalidate an entry. Compacted revisions are simply no longer requested and age out.
  - Benefit: hot keys that are read more often than written skip the LSM lookup entirely.
  - Costs and limits: its memory must be budgeted against the block cache, and it does nothing for cold reads or keys that change often. It sits below MVCC, so it does not bypass the linearizable-read wait.
  - *(Idea from the codex analysis. Benefit: Low-Moderate; X5 decides.)*
- **Snapshots and long reads** no longer block anything. bbolt's long read transactions can block remapping when the file grows (`storage-claude.md` §3.3). Pebble iterators pin files and memtables instead, which costs space, not latency.
- **Where reads get worse:**
  - large ranges (n point lookups);
  - serializable reads (no ReadIndex to hide engine time);
  - cache-cold reads after restart or compaction (read amplification across levels);
  - full scans: startup index rebuild, `HashKV`, and MVCC compaction rescanning from revision 0 (`kvstore_compaction.go:44`). These step over tombstones until compaction removes them.

  The #22332 prototype numbers (MVCC layer, compression on, "mostly vibed") are the only data: Get about 6.5× and Range about 5× lower throughput than bbolt. *(Moderate that a regression exists for these cases; its size after the rules above is unknown, X5.)*

### 4.9 Maintenance operations

| Operation | New meaning |
|---|---|
| Defrag | No-op, or a manual `Compact(start, end)` over the `key` prefix after a large MVCC compaction (`pebble:db.go:1825`). It no longer stops the world (`backend.go:476-614` today). |
| DB size and quota | From Pebble disk usage (`pebble:db.go:2465`). The meaning changes: space amplification from tombstones and compaction temporaries. Keep a headroom margin (`storage-claude.md` §3.3). |
| `HashKV` / corruption check | Unchanged logically (I10): it iterates revisions. |
| Backup (`Maintenance.Snapshot`) | Build a bbolt file from a Pebble snapshot (§4.6 format 1). Costs CPU and temporary space per backup. |
| `--unsafe-no-fsync` | Maps to unsynced log-store commits. Testing only, as today. |

### 4.10 Deployment and tuning

- **Device placement**, best first:
  - log store and state store on separate devices;
  - log store on a device with power-loss protection (PLP: a capacitor-backed cache that makes flushes cheap) if the fleet has any;
  - same device, if forced.

  The state store's flushes and compactions write large sequential files, but they still take queue slots from the log's `fdatasync`. Pebble limits compaction concurrency and smooths writeback through `BytesPerSync`/`WALBytesPerSync` (`pebble:options.go:495-498, :1020-1029`).
- **Metrics:**
  - keep the dashboard-compatible names: `etcd_disk_wal_fsync_duration_seconds` for log-store sync latency (from Pebble's WAL sync-latency metric), `etcd_disk_backend_commit_duration_seconds` for state-store batch commits;
  - add: checkpoint lag (`applied − checkpoint`), log bytes not yet truncated, Pebble L0 sublevels and write stalls for both instances, compaction debt.

## 5. Correctness argument

### 5.1 Invariants mapped to mechanisms and tests

| Invariant | Mechanism in B | Primary test |
|---|---|---|
| I1 | Responses are delivered only after the synced log-store commit (§4.2). Raft attaches dependent messages as responses (`raft:raft.go:153-187`). | Robustness with `raftBeforeSave`/`raftAfterSave`-equivalent failpoints under LazyFS. Porcupine linearizability checking (`tests/robustness`). |
| I2 | Pebble WAL replay with WAL-sync chunks (`pebble:format_major_version.go:201-206`); explicit `DeleteRange` for suffix replacement. | Bit-flip injection into log-store WAL files; expect fail-stop, not truncation. |
| I3 | CI, term and ConfState are in every state-store batch (§4.4). | Crash at every `MsgStorageApply` boundary under LazyFS; compare CI against content with `HashKV`. |
| I4 | Checkpoint protocol order: capture, sync state, mark, truncate (§4.5). | Failpoints between each step; LazyFS clear-cache at each point. |
| I5 | Recovery checks `t ≤ s ≤ last` and raises the commit index to `min(s, last)` (§4.7). | Unit tests over a crash-state enumeration (below), including a lost commit-only HardState. |
| I6 | Existing CI skip (`server.go:1907-1912`). | Existing. |
| I7 | Staged directory plus a synced log-store marker as commit point, with recovery redo (§4.6). | Existing `raftBeforeApplySnap`/`raftAfterApplySnap`/`raftAfterSaveSnap`-style failpoints (`gofail.go:55-60`) ported to the new steps, under LazyFS. |
| I8 | ConfState in state batches and in the `m/checkpoint` and `m/snapshot` markers. | Membership-change robustness scenarios. |
| I9 | Any log-store commit error is fatal. Pebble returns write and sync errors to the committer. | Error injection: Pebble's `vfs` error-injecting FS in unit tests; dm-flakey on hardware. |
| I10 | Logical `HashKV`. | Mixed bbolt/Pebble members in one cluster, with corruption checks enabled. |
| I11 | All of the above under LazyFS. | `scenarios.go:154-169`, extended. |

### 5.2 Crash-point enumeration (state after power loss, and what recovery does)

| Crash point | Durable state afterwards | Recovery result |
|---|---|---|
| During a log-store append batch, before sync completes | Pebble WAL replays the batch fully or not at all. No responses were sent. | Raft resends or re-proposes. Safe (I1). |
| After the log sync, before responses are delivered | Entries durable, nobody told | Safe: durable without acknowledgement is allowed. |
| After an unsynced commit-only HardState batch | Durable commit index older than entries already applied | Recovery raises it to `min(s, last)` (§4.7 step 7). It cannot fall below `t`, because markers go through the append FIFO after the HardState that covers them. |
| During apply (unsynced state batches) | State = a prefix of applied batches (Pebble WAL replay), CI matches (I3) | Replay from CI + 1 using the log, which still holds those entries because no checkpoint covered them. |
| Checkpoint: after capture, before state sync | Older `m/checkpoint`, log untruncated | Same as the previous row. |
| After the state sync, before the `m/checkpoint` commit | State durable through `a`; marker older | Extra log retained; harmless. |
| After `m/checkpoint`, before truncation | Marker `a`, log still holds ≤ a | Harmless; truncation is redone later. |
| Snapshot install, before the marker batch | Staging directory maybe complete; old state and log intact | Staging directory deleted at startup; Raft re-requests the snapshot. |
| After the marker batch, before the directory swap | Marker says snapshot `i` > CI; staging complete | Redo the swap (§4.6). |
| During the directory swap | Either name state is possible; `member/` fsync decides | Redo is idempotent: look for `state.old`, `state`, `snap/<id>`. *(Needs explicit unit tests. Renames and directory fsyncs are a common source of crash-consistency bugs; raft-wal's ALICE harness (§11.2) exists to find exactly these.)* |

### 5.3 Fault model beyond crashes

- **Torn and reordered writes before a sync.** Covered by Pebble's chunk CRCs and sync-offset chunks for both instances. Unsynced state-store WAL pages may reach disk out of order; replay stops at the first gap, so the prefix property holds.
- **Silent corruption of synced data.**
  - Today's WAL has a narrow gap here. Within the last segment, a record that fails its CRC is classified as a *torn write* if any sector of it is all zeros (`decoder.go:168-200`). Bootstrap then truncates there automatically (`bootstrap.go:645-649`, `repair.go:67-98`). A device error that zeroes a sector inside already-synced, already-acknowledged entries would therefore be silently truncated, not reported.
    - A second path needs no CRC failure at all. A zero length field anywhere in the last segment is read as end of log (`decoder.go:95-100`), and `ReadAll` then zeroes the rest of the file before appending resumes (`wal.go:563-568`).
    - hashicorp/raft-wal's README makes a related criticism (§11.2): a record whose payload legitimately contains an all-zero sector "will be falsely detected as a torn-write" if it is later corrupted in any way.
    - This is exactly the "conflating crash with corruption" class from the protocol-aware-recovery paper (§11.6). A follower that silently loses acknowledged entries can help elect a leader that lacks a committed entry.
    - *(Mechanism verified in code; likelihood Low; no etcd incident known to me.)*
  - Log store: Pebble's WAL-sync chunks record a promised synced offset, so replay can tell corruption *before* that offset (fatal) from an unsynced tail (discard) (`pebble:record/record.go:87-100`). RocksDB, by contrast, turns recycling off in its strictest recovery modes because it "cannot differentiate" the two (§11.3).
    - X6 must confirm that Pebble's replay actually fails on corruption below the synced offset with recycling on.
    - This is fatal (I2).
  - State store: Pebble block checksums.
  - Policy, as today: fail-stop, then replace the member and re-seed it from a snapshot. Automatic repair from peers, the "protocol-aware recovery" idea in §11.6, is attractive but out of scope.
- **fsync failure:** fatal, never retried (I9).
- **Disk full:** Pebble returns `ENOSPC`. Log-store errors are fatal, as `Save` errors are today. State-store errors must also be fatal, because a partial apply cannot be undone. The quota alarm must leave enough headroom for compaction temporaries.

### 5.4 Verification plan (before any production use)

1. **Unit level.**
   - Log store and recovery against Pebble's `vfs.NewCrashableMem()`, the FS its own crash tests use (`pebble:metamorphic/options.go:853`). Enumerate the crash states in §5.2.
   - Property-test the recovery checks.
2. **Robustness suite:**
   - LazyFS on every scenario, including 3-member clusters and compaction; today LazyFS variants exist only for single-member, low-load scenarios without compaction (`scenarios.go:154-169`);
   - new failpoints at each step of §4.3–§4.6;
   - Porcupine linearizability plus watch-guarantee checks;
   - mixed-format clusters.
3. **Power-cycle on real hardware** with the team's devices. Write-cache behavior and firmware flush handling are not simulated by LazyFS.
4. **Antithesis** (deterministic simulation), already set up in `tests/antithesis/`.
5. **MVCC-layer differential test for S1.** Upstream draft PR #22339 (serathius, 2026-08, work in progress) adds `server/storage/mvcc/storage_correctness_test.go` (1,130 lines). It uses Porcupine and validates watch events and historical reads. It was written for the #22332 engine prototypes. Reuse it rather than writing a new one. *(Its file list and test-function names were checked via the GitHub API; its coverage was not read.)*
6. **`DisableWAL`** only after (1)–(3) pass with it.

## 6. Performance model

### 6.1 Per-member I/O per Raft batch of n entries totaling b bytes

| | Today (WAL + bbolt) | B (default: state WAL on) | B with `DisableWAL` |
|---|---|---|---|
| Critical-path flushes | 1 `fdatasync` on fresh `fallocate` space (probably plus a filesystem journal commit) | 1 `fdatasync` on a recycled file (data only) | same |
| Apply-path flushes | 2 per backend commit (≥ every 100 ms, plus per delete entry and per compaction batch) | 0 | 0 |
| Background flushes | — | state WAL sync per checkpoint; SST syncs at flush and compaction | SST syncs only |
| Log rotation (every ~64 MiB of log) | Cut inside `Save` on the Raft goroutine: 2 `fdatasync`s, a rename and a directory fsync (`wal.go:785-866`) | Pebble WAL rotation at memtable rotation, with the commit pipeline held: close and sync the old WAL, rename a recycled file, sync the WAL directory (`pebble:db.go:2805-2840`; `pebble:wal/standalone_manager.go:150, :194`). About one `fdatasync` fewer than today, not zero; the fixed-ring ledger (B-L, §4.2) would avoid the rename and directory sync. | same |
| Bytes, log | b + framing | b + framing (+ small if flushed before truncation, §4.2) | same |
| Bytes, state | copy-on-write pages, random 4 KiB | ≈b (Pebble WAL) + ≈b compressed (flush) + compaction write amplification | flush + compaction |
| Rough total per value byte | ~2–3× | ~3–5× (sequential) | ~2–4× (sequential) |

*(Rough totals: Low confidence, my estimate.)* Revision keys are append-only, so Pebble compactions over them should often be cheap. MVCC-compaction tombstones and the frequently overwritten meta and lease keys force real rewrites.

**Implication:** B trades *synchronous, random, critical-path* I/O for *more, sequential, background* bytes. This is a net win on local NVMe. On **bandwidth-capped cloud volumes**, background bytes can become the new limit at high write rates. Check this in X4 against the volume's MB/s cap, and consider `DisableWAL` and compression there.

### 6.2 Throughput regimes

Let T_f be `fdatasync` latency, RTT the leader↔follower round trip, and C the number of in-flight client requests.

- **R1: closed-loop, latency-bound.** Throughput ≈ C / latency.
  - In synchronous mode, a proposal arriving while the leader's `Save` is in flight is not sent to followers until that save completes (§1.2). Its commit is then delivered to apply only in the Ready after the next save. Commit latency is therefore roughly T_f (wait) + max(T_f, RTT + T_f) + up to T_f (Ready cycle). All of that is exposed to the client.
  - In async mode: roughly max(T_f, RTT + T_f) plus small Ready overheads.
  - Example (illustrative, not measured): T_f = 2 ms, RTT = 0.5 ms. About 4.5–5 ms sync versus about 2.5–3 ms async, so up to ~1.5–1.8× closed-loop throughput. With PLP NVMe (T_f ≈ tens of µs) the difference vanishes. *(Model: Moderate. Numbers: Low.)*
  - Measured elsewhere (§11.1; none of it on etcd's real server or your hardware):
    - Raft PR #8 (a Raft pipeline harness using etcd's WAL package, on AWS gp3): −20–25% latency, +73% maximum throughput.
    - CockroachDB: −31% average and −44% p99 latency at 16k QPS; slightly worse at saturation; +17.6% on YCSB-A.
    - etcd PR #22223 (Docker on a laptop): +25% QPS, −24% WAL fsyncs.

    The direction is consistent. The sizes are not transferable.
  - Where T_f sits (§11.8):
    - enterprise NVMe with PLP: microseconds to tens of microseconds, so the async gain is small;
    - consumer NVMe without PLP: hundreds of µs to ~3 ms;
    - general-purpose cloud volumes: vendors quote "single-digit millisecond" latency, which is where async and recycling matter most.
  - A cheaper `fdatasync` (recycling) lowers T_f in every term.
  - Removing backend commit stalls lowers the apply wait (storage-claude bound: ≤ 1/(1 − f_commit)).
- **R2: open-loop or high concurrency.** Batches grow until something saturates.
  - With synchronous Ready/Advance, the Raft goroutine's cycle is T_cpu + T_io.
  - With async it is max(T_cpu, T_io), so the gain is at most 2×, reached when they are equal.
  - Beyond that, the bound is CPU in the Raft goroutine (raft v3.7.0 clones every proposed entry, `raft:raft.go:814-820`), in the apply goroutine, or in gRPC.
- **R3: apply-bound.** Gain ≤ 1/(1 − f_commit) from removing backend commits (`storage-claude.md` §3.4). Unchanged by the log-side work.
- **R4: network-bound.** The leader sends every entry to N − 1 followers. Leader egress ≈ (N − 1) × write bytes/s. With 10 KB values at 10k writes/s, a 3-member cluster needs ~200 MB/s (~1.6 Gbit/s) of leader egress. **No storage design changes this.**

### 6.3 Read latency model

L_read = T_ReadIndex + T_apply_wait + T_engine(n).

| Term | Effect of B |
|---|---|
| T_ReadIndex | Unchanged: one heartbeat-quorum round. |
| T_apply_wait | Lower under write load: no 2× `fdatasync` commit stalls, no read-lock exclusion. |
| T_engine(n), small n, warm cache | Slightly higher per lookup. Probably negligible next to RTT. *(Low-Moderate.)* |
| T_engine(n), large n or cold cache | Higher. The main regression risk (§4.8). |

### 6.4 What bounds throughput after B

1. The single apply goroutine, which must stay serial for revision assignment and transaction compare semantics.
2. The Raft goroutine's CPU.
3. Network fan-out (R4).
4. `treeIndex` and watch notification.

Pipelining the apply stage is the next frontier: decode, auth and lease lookups in parallel; serial compare-and-assign; state-batch commit and watch fan-out off the serial path. It is not part of this proposal. Beyond that, partitioning across clusters (`storage-claude.md` §1(b)) is the only linear-scaling option.

## 7. Library choices

| Library | Role in B | Verdict | Evidence |
|---|---|---|---|
| **Pebble v2** (BSD-3) | Log store (#1) and state store (#2) | **Use.** Pure Go, keeps `CGO_ENABLED=0`; WAL recycling, sync-offset chunks, WAL failover, checkpoints, value separation; production use at CockroachDB. | `pebble:` citations above; §11.2–§11.4 |
| Pebble `record` package | Custom ledger (B-L) | Optional, later. Exported API used by Pebble's own WAL; may change across Pebble majors. | `pebble:record/record.go`, `log_writer.go` |
| grocksdb / RocksDB | Option D | Not recommended: cgo, single maintainer (`storage-claude.md` §6). Per-write `disableWAL` is its one structural advantage here. | `storage-claude.md` |
| hashicorp/raft-wal (MPL-2.0) | Log store alternative | Do not depend on it. It implements hashicorp/raft's `LogStore`, not etcd raft's `Storage`; MPL-2.0 is file-level copyleft and outside CNCF's default license allowlist. Its design and its ALICE test harness are worth studying. | `raft-wal:LICENSE`, `raft-wal:alice/README.md`; §11.2 |
| dragonboat `tan` (Apache-2.0 + Pebble BSD) | Log store alternative | Do not depend on it. It lives under `internal/` (not importable); its last commit was 2025-07-24; it is built for multi-group Raft. A useful reference design. | `dragonboat:internal/tan/README.md` |
| TiKV raft-engine (Apache-2.0, Rust) | Log store alternative | Do not depend on it. It is purpose-built for multi-Raft logs, but using it from Go needs a C FFI layer and a Rust toolchain, which gives up `CGO_ENABLED=0`. | §11.2 |
| etcd WAL (current) | Log in S1/S2 | Keep for S1–S2. Its format blocks recycling (§1.1), but it is proven. | §1.1 |
| io_uring / O_DIRECT | Log write path | Not worth it at this stage: low expected gain over recycled-file `fdatasync`, and immature Go libraries. *(Low.)* | — |

## 8. Staging plan

| Stage | Change | On-disk format | Expected benefit | Gate to proceed |
|---|---|---|---|---|
| S0 | Tuning and measurement (`storage-claude.md` §4.3, E0) | none | Baseline; may be enough | E0/X0 say whether apply, `fdatasync` or network binds |
| S1 | Pebble state store (option A) | backend only | Removes apply-path fsyncs | E5 in `storage-claude.md` shows gain and acceptable reads (X5) |
| S2 | `AsyncStorageWrites` with the existing WAL (option A+), starting from upstream PR #22223 | **none** | Up to ~1 fsync less commit latency (R1), better batching (R2) | raft #551 and #543 fixed or worked around; X3 shows gain at your T_f/RTT; LazyFS robustness passes |
| S3 | Log store on Pebble (option B) | new log format | Cheaper `fdatasync` (X1), WAL failover, stronger corruption detection, log/state isolation | X1 shows ≥ ~20–30% lower `fdatasync` p99 on recycled files on your devices, or WAL failover is needed for disk-stall tails |
| S4 | Custom ledger (B-L) | another log format | CPU and memory | X2 shows log-store #1 overhead > ~5% of a core at target load, or memory pressure |

S2 does not depend on S1. It could go first if X0 shows `fdatasync` latency dominating. S3 depends on S2, because the new log store is designed around the async append thread.

## 9. Compatibility, migration and tooling

- **Non-test importers of the `wal` package** (13 files, excluding the package's own `wal/testing`). Each needs a log-store equivalent or a format switch:
  - `server/storage/storage.go`, `server/etcdserver/bootstrap.go`, `server/embed/util.go`, `server/storage/schema/schema.go`, `server/verify/verify.go`;
  - `etcdutl/etcdutl/common.go`, `etcdutl/etcdutl/migrate_command.go`, `etcdutl/snapshot/v3_snapshot.go` (restore *creates* a WAL);
  - `tools/etcd-dump-logs/{main,raw}.go`, `tools/proto-annotations/cmd/etcd_version.go`;
  - `tests/robustness/report/wal.go` (robustness reports read WALs);
  - `contrib/raftexample/raft.go` (example, can stay).
- **`MinimalEtcdVersion`** closes, re-reads and reopens the WAL (`storage.go:109-128`). It runs only when the storage schema is migrated to a lower version (`schema/schema.go:69-70`), so it is not a steady-state cost. The log store must support a full scan, which it does; tracking the version incrementally at append time would be cheaper still.
- **Mixed-version clusters.** The Raft wire protocol is unchanged; only the `MsgSnap` payload is format-specific. New-format members must send and accept bbolt-format snapshots while any old member exists (§4.6 format 1).
- **Migration paths:**
  - **(a) Member replacement:** remove a member, add a new-format member that receives a snapshot. Simplest and uses only Raft.
  - **(b) Offline converter:** clean shutdown, then bbolt to Pebble with CI, and WAL entries > CI into the log store. Faster for big data, but new code.

  Recommend (a) first.
- **Downgrade:** member replacement in reverse, or a reverse converter. The `FORMAT` file prevents an old binary from opening a new layout.
- **Backups:** keep producing bbolt files (§4.9) so `etcdutl snapshot restore` and existing runbooks keep working.

## 10. Risks

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | The binding constraint is network, apply CPU or client concurrency, not storage, so gains are small | Moderate | High (wasted effort) | E0/X0 first; stage gates |
| 2 | The async Raft loop loses an etcd-specific ordering (configuration-change wait, snapshot notify) | Moderate | Critical (split vote, divergence) | Port every failpoint (§4.3); targeted robustness scenarios |
| 2b | Raft-library bugs in `AsyncStorageWrites` itself: #551 (vote sent before durable), #543 (snapshot overtakes applies), both open and unreviewed. The mode has far less production mileage than synchronous mode outside CockroachDB. | **Known** | Critical (two leaders in one term; state divergence) | Fix or work around both (§4.3) before any use; add the reporters' reproduction tests; Antithesis runs with the gate on |
| 2c | Making persistence asynchronous breaks log/state consistency after restart. Xline hit exactly this with its initial all-async persistence design (§11.6). | Moderate | Critical | Every async step is bounded by an explicit durable marker (§4.5, §4.6); the §5.2 enumeration; LazyFS |
| 3 | Crash bug in the checkpoint, truncation or snapshot-swap protocol | Moderate | Critical (lost committed data) | §5.2 enumeration; LazyFS; CrashableMem unit tests; power-cycle |
| 4 | Read-latency regression on large ranges or cold cache | Moderate-High | High (violates the stated constraint) | §4.8 rules; X5 gates; fall back to bbolt for read-heavy clusters |
| 5 | Background write amplification exceeds the cloud volume's bandwidth cap | Moderate on capped volumes | High | X4 on the real volume type; compression; `DisableWAL` after testing |
| 6 | Pebble state-store write stalls under compaction debt | Moderate | Medium (apply backpressure → `ErrTooManyRequests`) | Tuning; stall metrics; the B design isolates the log |
| 7 | Format change breaks tools and runbooks (backups, `etcdutl`, dump-logs) | High if unaddressed | Medium | §9 list; keep bbolt as interchange |
| 8 | Fork drift: upstream changes the Raft loop, bootstrap and snapshots | High | Medium-High | Pin to `release-3.7`; keep S2 format-free; propose S2 upstream |
| 9 | `fdatasync` savings from recycling are negligible on your devices | Moderate | Low (S3 skipped) | X1 before S3 |
| 10 | Pebble behaviors relied on but undocumented (flush drops keys deleted by a range deletion in the same memtable; ordering of the unsynced WAL) | Low-Moderate | Medium | Tests in X2 and X6; read the Pebble code paths before S3 |

## 11. Prior art

Markers:
- **[V]**: I read the primary source this session.
- **[Va]**: a research subagent read the primary source this session; I did not re-read it.
- **[Vt]**: read by a subagent through a text-extraction tool, so check quotes before reusing them.
- **[U]**: snippet or memory only.

All dates are as stated by the source.

### 11.1 Asynchronous Raft storage writes

- **etcd-io/raft PR [#8](https://github.com/etcd-io/raft/pull/8)** "raft: support asynchronous storage writes". Nathan VanBenschoten, merged 2022-12-21, milestone v3.6.0. [Va]
  - Benchmark: rafttoy (a Raft pipeline harness) using etcd's WAL package plus Pebble plus gRPC, on 3× AWS m5.4xlarge with gp3 volumes.
  - "reduces end-to-end latency of proposals by 20-25%". At 16 MB/s, average latency was 13.2 ms for stock Raft and 5.2 ms with async writes.
  - "an increase in maximum throughput of 73%", from 30 to 52 MB/s.
- **CockroachDB PR [#94165](https://github.com/cockroachdb/cockroach/pull/94165)** "kv: integrate raft async storage writes", merged 2023-02-03 and shipped in v23.1. [Va]
  - kv0 workload on 3× n2-standard-32: at 16k QPS, average latency −30.8% and p99 −44.0%. At saturation (64k QPS), average +5.9% and p99 +9.4%.
  - YCSB-A: +17.57% ops/s.
  - The PR's own caveat: no top-end throughput gain until log append and state-machine application are separated from the Raft loop.
- **etcd issue [#16291](https://github.com/etcd-io/etcd/issues/16291)** "Integrate raft's new feature (async write) into etcd". Open, opened 2023-07-24 by ahrtr, stale-closed twice and reopened 2026-01-09. [Va]
  - Earlier attempt: PR [#18027](https://github.com/etcd-io/etcd/pull/18027) (serathius, 2024). Draft with failing robustness tests; stale-closed 2025-08-24. [Va]
- **etcd PR [#22223](https://github.com/etcd-io/etcd/pull/22223)** "Raft async storage writes" (mjlshen). Opened 2026-07-30, open, labels `needs-rebase` and `size/XXL`. [V]
  - Alpha gate `RaftAsyncStorageWrites`, off by default.
  - Its description: "processes append and apply work in FIFO order; batches compatible contiguous WAL appends; responds only after the corresponding write or application completes; preserves existing snapshot persistence and crash-recovery ordering".
  - Preliminary numbers from a 3-member Docker Desktop cluster on a MacBook (64 clients, 1 KiB values): +25.0% requests/s, −19.9% average latency, −23.8% WAL fsync count.
  - The author says the tests and the benchmark design were written partly with generative AI.
- **raft [#543](https://github.com/etcd-io/raft/issues/543)** "With AsyncStorageWrites, entries applied after a snapshot that includes them diverge the state machine". Opened 2026-10-23, open, 0 comments. [V]
- **raft [#551](https://github.com/etcd-io/raft/issues/551)** "with AsyncStorageWrites, MsgVote is sent before the candidate's term and vote are durable". Opened 2026-10-31, open, 0 comments; the author account is "perfloop-agent"; proposed fix PR #550. [V]
  - I confirmed the cited deferral logic in raft v3.7.0: `send()` defers only `MsgAppResp`/`MsgVoteResp`/`MsgPreVoteResp` (`raft:raft.go:546`).

### 11.2 Separating the Raft log from the state machine

- **CockroachDB, separate Raft-log engine (cockroach#16624).** Open, milestone "Later", reassigned 2026-03. A prerequisite, "loosely coupled log truncations" (#93248), closed 2026-05-01. **Not shipped** in any release notes through v26.3. [Va/Vt]
  - Design comment (sumeerbhola, 2022-05-25), quoting: "The design mentioned in the previous comment has a WAL in both storage engines. The WAL for the state machine engine is very rarely synced." Under loose coupling, "the next flushed callback from the state machine engine causes the truncation." [Va]
  - Write amplification measured in that comment (compaction bytes / WAL bytes, 1,000-byte values) [Va]:
    - 4.06 with one engine;
    - 3.70 with two engines, truncating after a state-machine flush;
    - 2.82 with two engines, truncating almost immediately.
  - Throughput was equal between one and two engines, attributed to "the behavior of ext4, where an fdatasync flushes everything in the buffer cache". [Va]
  - What CockroachDB shipped instead, v25.4.0 (2025): "reduce write amplification by storing Raft log values in separate blob files", using Pebble value separation, on by default. [Va]
- **TiKV raft-engine.** A log-structured, Bitcask-like store for multi-Raft logs; Apache-2.0. [Vt]
  - InfoQ (2022-05-12), TPC-C with 5,000 warehouses on AWS r5.4xlarge + gp3: "improves QPS by up to 4%, and reduces tail latency by 20%"; "Write I/O is reduced by 25% ~ 40%"; CPU −12%.
  - The default since TiKV 6.1.
  - TiKV's experimental "partitioned Raft KV" (raftstore v2) "disables WAL for all tablets and store all states to raft engine" (tikv PR #13939). That is the TiKV analogue of a state store with no WAL. The docs still label it experimental, not for production.
- **hashicorp/raft-wal.** MPL-2.0 (`raft-wal:LICENSE` [V]). [Va]
  - The README still says "This library is still considered experimental!"
  - It cites "only one fsync per append vs two in BoltDB" and "Efficient truncations" against raft-boltdb.
  - It criticizes etcd's WAL torn-write heuristic: "if a legitimate record contains more than 1kb of zero bytes and happens to ever be corrupted after writing, that record will be falsely detected as a torn-write" (`raft-wal:README.md` [V]; see §5.3).
  - Crash testing with ALICE (`raft-wal:alice/README.md` [V]).
  - Adoption [Va/Vt]: experimental in Consul 1.15 (2023-02). Consul's current config reference says WAL is the default, while another Consul page says it is not yet; the docs disagree. Opt-in in Vault 1.16+, with "proceed with caution".
- **dragonboat `tan`.** "a high performance database for storing Raft log and metadata". It contains Pebble-derived code; its motivation lists redundant memtables, keys and serialization in LSM-based logs (`dragonboat:internal/tan/README.md` [V]).
  - The changelog calls it experimental under an unreleased v4. The default log store is still Pebble-based. [Va]

### 11.3 WAL recycling and `fdatasync` metadata cost

- **RocksDB** `options.h` (`recycle_log_file_num`): "more efficient because the blocks are already allocated and fdatasync does not need to update the inode after each write." [Va]
  - Correctness caveat (`db_impl_open.cc:107-123`): recycling is forced off in the `kTolerateCorruptedTailRecords` and `kAbsoluteConsistency` recovery modes, because recovery "cannot differentiate between this and a real corruption." [Va]
- **Pebble:**
  - "Writing to a recycled log file is faster than to a new log file on some common filesystems (xfs, and ext3/4) due to avoiding metadata updates" (`pebble:wal/log_recycler.go`) [V];
  - recycling is on by default (`pebble:open.go:355`) [V];
  - WAL-sync chunks "disambiguate between corruption and logical EOF" (`pebble:format_major_version.go:201-206`) [V].
- **Linux:**
  - ext4 commits the journal on `fdatasync` after unwritten-extent conversion (`fs/ext4/extents.c` conversion path and `fs/ext4/fsync.c` datasync transaction ID);
  - XFS sets the inode's datasync sequence on non-timestamp inode changes (`xfs_inode_item.c`). [Va]
  - Kernel patch series "fallocate: introduce FALLOC_FL_WRITE_ZEROES" (Zhang Yi, 2025-06-19): "The extent state must be converted to a written state … which can trigger numerous metadata changes and consequent journal I/O." Measured with fio (4 KiB, O_DIRECT, fdatasync, ext4 on NVMe): **20.1k IOPS into unwritten extents, 98.8k into written blocks.** [Va]

### 11.4 WAL failover

- **CockroachDB/Pebble WAL failover.** Preview in v24.1, GA in v24.3. [Va]
  - The docs: transient disk stalls "often lasting on the order of several seconds". Failover "bounds Raft log latencies to about 100ms" and lets a node "survive stalls of up to 40s". It relocates only the WAL; reads still stall on cache misses. [Vt]
  - A later fix (v25.2.26 / v25.4.19 notes): "a write could lose its sync notification, leaving a Raft log sync blocked forever". The feature carries its own risk. [Va]
- In the log store (§4.2) this is a configuration option (`pebble:options.go:1036-1041`), not part of the baseline design.

### 11.5 Removing the double write (log + state)

- **Nezha: "A Key-Value Separated Distributed Store with Optimized Raft Integration"** (Wang, Dong, Cheng, Xu). arXiv 2603.09122, ICDE 2026. [Va]
  - Values live only in the Raft log; the engine stores offsets.
  - Results: +460% put, +12.5% get, +72.6% scan, against the authors' own Go + RocksDB baseline, with values of 1–256 KB.
  - Without garbage collection, get is −21% and scan −40%.
  - Gains concentrate on large values. This is option E in §3.
- **PASV** (FAST '22) removes the storage-engine WAL in a Raft-replicated LSM database. **PALF** (PVLDB 2024, OceanBase) makes the Paxos log *the* database WAL. **LSM-Raft** (PACMMOD 2025) ships compacted SSTs to followers. [Va for citations; content U]

### 11.6 Crash consistency and storage faults

- **Protocol-aware recovery** (Alagappan et al., FAST '18). It studied etcd with log fault injection: "LogCabin, ZooKeeper, and etcd crash sometimes when their logs are faulty." It also states: "Most log-based systems conflate these two cases" (crash versus corruption). The silent-truncation safety violation it found was in ZooKeeper and LogCabin. [Va]
- **"When Amnesia Strikes"** (LazyFS; Ramos, Azevedo, Kingsbury, Pereira; PVLDB 2024). It studied etcd. [Va]
  - Bug: a lost write after the first WAL write made etcd fail to restart with "snapshot not found".
  - Torn writes in the bootstrap database write were filed as etcd #16596 and closed as not a code-level problem.
  - A split-brain found with Jepsen plus LazyFS, etcd #14143, was closed as not planned on 2025-07-28 with no root cause. Forcing `MustSync` to true reportedly made it much harder to reproduce.
  - **The baseline you would be replacing is itself not fully crash-verified.** The new design should be held to I1–I11, not merely to parity.
- **"Can Applications Recover from fsync Failures?"** (ATC '20). It did not study etcd. Its finding that "all three file systems mark pages clean after fsync fails" is why I9 forbids retrying. [Va]
- **Xline** (an etcd-API-compatible store). Its Jepsen write-up says "the initial design of Xline was to make all storage persistence operations asynchronous", which led to log/KV inconsistency after restart (CNCF blog, 2024-02-19). This is a direct warning for any design with unsynced state writes. [Vt]
- **etcd #14370** "Durability API guarantee broken in single node cluster", fixed in v3.5.5 and v3.4.21: committed entries were applied before persistence in the single-node case. It is a precedent for how subtle the apply-versus-persist ordering is. [Va]
- **Jepsen, etcd 3.4.3** (2020): no storage faults tested. [Va]

### 11.7 etcd's own guidance and numbers

- **Performance page** (v3.7 docs, copied from v3.6). The only official write numbers are from **etcd 3.2.0** on GCE with 8 vCPUs: 44,341 QPS at 22 ms average with 1,000 clients against the leader, and 50,104 QPS at 20 ms against all members, with 256-byte values. "For SSDs, the latency is often lower than 1ms." [Va]
- **FAQ:** "wal_fsync_duration_seconds (p99 duration should be less than 10ms)", "backend_commit_duration_seconds (p99 duration should be less than 25ms)". [Va]
- An etcd source comment assumes "The max throughput is around 10K" (`server.go:80-85`). [V]

### 11.8 `fdatasync` latency by device class

| Device class | Figure | Source (quality) |
|---|---|---|
| Enterprise NVMe with PLP (Samsung PM9A3; Solidigm D7-P5520) | fdatasync ~0.7–10 µs | Small Datum blog, 2026-01 (reputable individual; measures sync after an O_DIRECT write, not etcd's buffered pattern) [Va] |
| Consumer NVMe without PLP (Samsung 990 Pro) | ~2.8 ms | same [Va] |
| GCP local NVMe / Hyperdisk Balanced | ~28 µs / ~47 µs (fsync ~740 µs on ext4) | same [Va] |
| AWS gp3 / io2 Block Express | "single-digit millisecond" / "under 500 microseconds" (per I/O) | AWS docs (vendor targets, not fsync measurements) [Va] |
| GCP pd-ssd; Azure Premium SSD | "single-digit millisecond" | vendor docs [Va] |

*Implication:* the `fdatasync` cost that S2 and S3 attack ranges over three orders of magnitude across plausible production devices. X1 on your actual devices decides whether S2 and S3 are worth it.

## 12. Experiments (on hardware representative of production)

All experiments record the context required by AGENTS.md §9 (etcd commit, cluster size, hardware, storage stack, client, key and value sizes, concurrency, exact command). None may be run on the analysis host (AGENTS.md §7).

Two measurement rules apply throughout (both from the codex analysis):
- **Measure latency open-loop, or correct for coordinated omission.** The design's main effect is removing stalls: bbolt commits, WAL cuts. A closed-loop client stops issuing requests during a stall, which hides the stall from its percentiles. `tools/benchmark put --rate` caps the rate but stays closed-loop: each client waits on the limiter and starts its latency clock afterwards (`tools/benchmark/cmd/put.go:64, :97-99`).
- **Read numbers only at steady state.** Wait until Pebble compaction debt and the checkpoint cadence have settled into a repeating pattern, and include more than one truncation cycle.

**X0: regime identification (no code; 1–3 days). Gate for everything.**
- From production metrics:
  - `etcd_disk_wal_fsync_duration_seconds` (p50/p99);
  - `etcd_disk_backend_commit_duration_seconds` rate (f_commit);
  - apply duration;
  - `etcd_server_proposals_pending`;
  - `etcd_network_peer_round_trip_time_seconds`;
  - leader egress bytes;
  - CPU profiles of the leader (Raft goroutine, apply goroutine, gRPC).
- Compare against etcd's own FAQ thresholds: WAL fsync p99 < 10 ms, backend commit p99 < 25 ms (§11.7).
- **Ceiling matrix** (stock binary and flags only, on a test cluster with production-class devices; idea from the agy analysis). Run the same load four ways:
  1. WAL and backend both on disk;
  2. backend on tmpfs, WAL on disk via `--wal-dir` (`embed/config.go:606`); this cell is `storage-claude.md` E2;
  3. WAL on tmpfs, backend on disk;
  4. both on tmpfs.
  - (2) − (1) bounds what S1 can gain.
  - (3) − (1) bounds S2 and S3 together: async writes and recycling can only hide or shrink `fdatasync` cost, and tmpfs removes it.
  - (4) is the CPU and network ceiling (R2–R4).
  - These are upper bounds, because tmpfs removes the write work as well as the flush, and Pebble is not free. If (3) − (1) is small on production devices, skip S2 and S3.
  - tmpfs is volatile: the matrix says nothing about durability.
- Classify the workload into R1–R4 (§6.2).
- Includes `storage-claude.md` E0.

**X1: `fdatasync` cost on target devices (no etcd; 1–2 days). Gate for S3.**
- A small Go or `fio` program on each candidate volume type, with block size 4–256 KiB per sync:
  - (a) append into an `fallocate`d file plus `fdatasync`, which is etcd's pattern;
  - (b) overwrite a previously written file plus `fdatasync`, which is the recycling pattern;
  - (c) (a) and (b) with `O_DSYNC`;
  - (d) (a) and (b) while another process does random 4 KiB writes plus `fdatasync` on the same device (bbolt interference);
  - (e) separate devices.
- Report p50/p99/p99.9.
- Also run (a), (b) and (d) on ext4 with `data=ordered` (the default) and on XFS. Include a case where an unsynced writer on the same filesystem dirties data while the log syncs; this tests the ext4 coupling in §4.2.
- Prior expectation: the kernel-patch measurement in §11.3 found ~5× IOPS between unwritten and written blocks on fast NVMe. On millisecond-class cloud volumes, the device flush may dominate and shrink the ratio.
- Decision: proceed with S3 only if (b) beats (a) by ≥ ~20–30% at p99, or if (d) shows interference that separation fixes.

**X2: log write path microbenchmark (2–4 days).**
- Compare, with identical batch sizes:
  - etcd's `wal` benchmarks (`server/storage/wal/wal_bench_test.go:27-37`);
  - a Pebble log-store prototype (entries as keys, `Sync` batches, `DeleteRange` truncation);
  - optionally Pebble `record.LogWriter` alone.
- Measure throughput, CPU per entry, allocations, `fdatasync` count and memory.

**X3: `AsyncStorageWrites` prototype with the existing WAL (S2; 2–4 weeks).** Start from upstream PR #22223, rebased onto `release-3.7`, with the raft #551/#543 fixes applied.
- Three members on target hardware.
- `tools/benchmark put`/`txn-put` concurrency sweep (1 → 1,000 clients), value sizes 256 B / 4 KiB / 32 KiB.
- Plot throughput and p50/p99 latency curves against the baseline.
- Repeat with injected RTT (tc netem) at 0.2/1/5 ms to locate the R1 crossover.

**X4: B end-to-end (S1+S2+S3 prototype).**
- As X3, plus a lease-churn harness (revocations at 100–5,000/s), MVCC compaction during load, snapshot send to a lagging member, and a 24 h run.
- Record device MB/s against the volume cap (risk 5) and Pebble write-stall counters.

**X5: read-latency guardrails. Gate for S1 and later.**
- Under the X4 write load, measure linearizable and serializable Get p50/p99, Range with 1k/10k/100k keys, and watch event latency.
- Repeat with block cache at 25/50/100% of the dataset, and after a cold restart.
- **Suggested acceptance thresholds** (the team should set the real ones):
  - p99 linearizable Get within +10% of baseline;
  - p99 Range(10k) within +25%;
  - no regression in watch latency.

**X6: crash consistency (continuous during S2–S3; weeks).**
- §5.4 items 1–4.
- At least N thousand LazyFS-enabled robustness runs per release candidate, with zero linearizability violations.
- Power-cycle tests on target hardware.

**X7: soak and operations (2+ weeks).**
- 7-day mixed load.
- Member replacement in both directions (migration path (a)).
- Backup and restore with stock `etcdutl`.
- Rolling upgrade and downgrade.

## 13. Effort estimate (Low confidence)

| Stage | Prototype (benchmarkable, not safe) | To production quality |
|---|---|---|
| S1 Pebble state store | 3–5 eng-weeks (`storage-claude.md` §11) | 6–12 eng-months (same) |
| S2 `AsyncStorageWrites` | 2–4 eng-weeks (backport and finish PR #22223; fix raft #551/#543) | +3–5 eng-months (ordering audit, failpoints, LazyFS campaign). Less if upstream lands it. |
| S3 Pebble log store | 3–5 eng-weeks | +4–7 eng-months (format, recovery, snapshot install, tools in §9, migration) |
| S4 custom ledger | 6–10 eng-weeks | +2–3 eng-months |
| **S1–S3 total** | ~10–16 eng-weeks | **~12–24 eng-months** |
| Fork maintenance | — | ~0.5–1 FTE ongoing (Raft loop and bootstrap churn upstream) |

Assumes 1–2 engineers strong in Go and storage and new to etcd internals. Crash-consistency verification is the long pole and should not be compressed.

## 14. The fork plan, revisited

- **The log side is more fork-hostile than the backend side.** S2 and S3 rewrite `server/etcdserver/raft.go` (the Ready loop), `bootstrap.go`, `storage.go`, snapshot install and receipt, and the tools in §9. These areas change often, on `main` and on `release-3.7`:
  - main only (not in v3.7.3): `5d1a827c1` "Stop generating v2 snapshot files, nor load v2 snapshot files" (2026-07-22) and its follow-ups (`ce48442d1`, `f5dc18a29`, `37729af01`); `b02869083` "server: remove verifySnapshotIndex assertion"; `702fd61ff` "wal: close files on Open and Create error paths";
  - backported into v3.7.3: `e71275f78` "etcdserver: keep Ready message order in processMessages" (main `01a80f67b`, 2026-10-17); `23b61e6d9` "Update MinimalEtcdVersion to read latest snapshot entry from WAL" (main `2e2963af6`).

  Rebasing S2/S3 across minor releases will be expensive.
- **Pin to `release-3.7`.** Cherry-pick security and correctness fixes only.
- **S2 is already in flight upstream.**
  - Issue #16291 was opened by maintainer ahrtr. Maintainer serathius agreed in 2026-07 that it belongs behind a feature gate.
  - PR #22223 implements it as the alpha gate `RaftAsyncStorageWrites`. It is open and needs a rebase; robustness CI failed on 2026-07-31.
  - Contributing reviews, the raft #551/#543 fixes, and target-hardware benchmarks there costs less than carrying it in a fork.
- S1 aligns with #22332. S3 and S4 should be assumed fork-only.
- **Keep the bbolt file as the interchange format** for snapshots and backups throughout. That is what keeps rollback to stock etcd possible: replace members back with stock binaries.

## 15. What I verified vs inferred, and open questions

**Verified in code (this session):**
- the WAL format, sync policy, cut, recovery and torn-write logic (§1.1);
- the Ready-loop ordering, and apply running concurrently with the WAL save (§1.2);
- raft v3.7.0 `AsyncStorageWrites` semantics, leader self-ack after persistence, and Ready gated on `Advance` (`raft:raft.go`, `rawnode.go`, `node.go`);
- that bootstrap tolerates `CI > commit`;
- the LazyFS robustness scenario;
- Pebble's `DisableWAL` comment and its exclusion from crash tests;
- Pebble's WAL recycling rationale and default, sync-offset chunks, commit-pipeline ordering, `LogData` and `Sync` behavior, value separation, iterator seek optimization;
- the raft-wal license and ALICE harness;
- that dragonboat `tan` is an internal package, and its README motivation;
- upstream `main` commits touching the WAL, snapshots and the Raft loop;
- etcd PR #22223 and raft issues #543 and #551 (via the GitHub API), including that #551's cited deferral logic is present in raft v3.7.0 (`raft:raft.go:546`);
- that bootstrap auto-repairs torn tails by truncation (`bootstrap.go:645-649`).
- Added in the revision (§16):
  - that a commit-only HardState still produces a `MsgStorageAppend` in async mode (`raft:rawnode.go:200-208, :230-241`);
  - Raft's start-up range check on the commit index (`raft:raft.go:2037-2040`);
  - raft PR #300 and its comments, and etcd PR #20055 with the code it added (`bootstrap.go:588-600`);
  - Pebble's WAL rotation sequence (`pebble:db.go:2805-2840`, `pebble:wal/standalone_manager.go:150, :194`);
  - Pebble's span-restricted checkpoint caveat (`pebble:checkpoint.go:50-56`);
  - etcd's `ForceCommit` call sites and their comments;
  - that `MinimalEtcdVersion` runs only on a schema downgrade (`schema/schema.go:69-70`);
  - that `tools/benchmark put` is closed-loop;
  - PR #22339's file list.

**Errors in the first draft, corrected in the revision:**
- I5 and §4.7 cited `bootstrap.go:589-597` as tolerating CI above the last durable log index. That code actually handles CI above the durable *commit* index, and only under `--force-new-cluster`. Stock etcd has no bootstrap check of CI against the last log index.
- §4.2 synced every log-store batch, which would add an `fdatasync` per commit-index advance.

**Corrected in a later consistency pass (same day; every `path:line` citation re-read at the pinned versions, and every issue and PR re-fetched from the GitHub API):**
- Summary item 9 and §14 cited `01a80f67b` (Ready message order) and `2e2963af6` (`MinimalEtcdVersion`) as main-only changes. Both were backported and are in v3.7.3 as `e71275f78` and `23b61e6d9`. The main-only examples are now the v2-snapshot removal series, `b02869083` and `702fd61ff`.
- §1.2 said the apply loop waits for the WAL write "only before taking a snapshot". The wait is in every apply job, and jobs run one at a time, so the next Ready's apply starts only after the previous WAL write.
- Summary item 4(iii) and §5.4 implied LazyFS covers the robustness suite generally. LazyFS variants exist only for single-member, low-load scenarios with compaction off.
- §5.3 and §11.2 misdescribed raft-wal's criticism: it concerns records whose payload contains an all-zero sector, not device-zeroed sectors. §5.3 now also records a second silent-truncation path: a zero length field in the last segment.
- Line numbers fixed: `wal.go:65` (crcTable), `wal.go:820-826` (HardState at cut), `pagewriter.go:23`, `encoder.go:36`, `server.go:2139-2170`, `bootstrap.go:588-600`, `scenarios.go:154-169`.
- Citations added: `bootstrap.go:540-552` (etcd does not enable async writes), `util.go:60-69` (where the apply-gap limit is enforced), `pebble:checkpoint.go:43, :167` (Pebble's own `LogData(nil, Sync)` barrier).

**Read by research subagents, not re-read by me** (marked [Va]/[Vt] in §11):
- the CockroachDB, TiKV, HashiCorp, RocksDB and kernel sources;
- the papers;
- the etcd docs numbers;
- the `fdatasync` latency figures.

**Inferred or estimated (not verified):**
- that `fdatasync` into `fallocate`d space costs a journal commit on your filesystems, and how much (X1);
- every throughput and latency number in §6, all illustrative;
- write-amplification totals (§6.1);
- that range deletions elide log entries at flush (§4.2);
- that a second Pebble instance's overhead is negligible for one Raft group (X2);
- the effort figures.

**Open questions:**
1. Device and filesystem in production: local NVMe with or without PLP, cloud volume type, ext4 or XFS. This decides X1 and risk 5.
2. Write profile: value-size distribution, lease-revocation rate, transaction mix, client concurrency. This decides R1–R4.
3. Read profile: share of large ranges and serializable reads. This decides how binding risk 4 is.
4. Whether cross-partition atomicity is really required; if not, partitioning beats all of this (`storage-claude.md` §1).
5. Tolerance for an on-disk format that stock etcd cannot read. This decides whether S3 is acceptable at all.

## 16. Revision: ideas taken from the codex and agy analyses

After the first draft, I read `wal-replacement-codex.md` and `wal-replacement-agy.md` as a source of ideas, not to evaluate them. I re-checked each adopted item against source before using it. The design is unchanged in structure: two Pebble stores, async Raft, checkpoint-gated truncation.

| Change | Where | Source of the idea | Why adopted |
|---|---|---|---|
| Log-store batches sync only when `raft.MustSync` requires it; commit-only HardState is unsynced | §4.2, summary 3 | both | My first draft synced every batch: one extra `fdatasync` per commit-index advance |
| Recovery raises the commit index to `min(s, last)`, with raft PR #300 and etcd PR #20055 as precedent | §4.7 step 7, §5.1, §5.2, I5 | codex (the commit-index subtlety) | Required by the item above; also corrected my mis-citation of `bootstrap.go:589-597` |
| Checkpoint marker and truncation go through the append FIFO and need no sync; single writer for the log store and `m/hardstate` | §4.2, §4.5 | codex (never let another path overwrite term or vote) | Removes a race and a sync |
| Mapping of etcd's `ForceCommit` call sites; why writes outside apply need no sync | §4.4 | codex (classify every write outside apply) | Closes a gap in the state-store durability argument |
| Snapshot payload must exclude Raft-local state; Pebble span-restricted checkpoints are not exact | §3.3 | codex | Another concrete reason for B over C |
| Optional revision-keyed record cache | §4.8, summary 5 | codex | Cheap read-latency mitigation for hot keys |
| Open-loop latency measurement and steady-state rule | §12 | codex | The design's gains are stall removal, which closed-loop clients hide |
| Ceiling matrix with tmpfs and `--wal-dir` | §12 X0 | agy (extends my own E2 in `storage-claude.md`, which covered only the backend cell) | Zero-code upper bounds for S1 and for S2+S3 before building anything |
| Rotation cost row: Pebble's WAL rotation still costs a sync, a rename and a directory sync | §1.3, §6.1 | agy (the cut stall) | Corrects an implicit overstatement of B's per-rotation savings |
| Fixed-ring, rename-free segment reuse for the custom ledger | §4.2 (B-L) | agy | Removes rotation metadata work entirely |
| Optional disk catch-up window longer than memory's | §4.5 | agy | Fewer full snapshot transfers to lagging followers |
| PR #22339 MVCC correctness test as the S1 differential test | §5.4 | codex (upstream reference) | Reuse instead of writing new tests |
| TiKV raft-engine library row | §7 | both | Completes the library table |

**Considered and not adopted**, with reasons about this design only:
- Pebble with `DisableWAL` as the state store's default. Already deferred until power-loss testing passes (§4.4), because Pebble's crash tests exclude it.
- A synchronous dependence of the Raft loop on the apply loop through the unbuffered `applyc`. The receiving side hands each item straight to an unbounded FIFO scheduler (`server.go:843-845`), so I found no steady-state stall to remove.
- Runtime WAL reopening by `MinimalEtcdVersion`. It runs only on a schema downgrade (§9).
- A unified single-engine design (codex Design A) and a journal with a rebuildable, WAL-disabled state store (codex Design B, agy Option C). These are alternative architectures rather than ideas that fit inside B; §3 already covers the trade-offs that matter here.
