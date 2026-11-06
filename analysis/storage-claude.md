# Storage engine: replacing bbolt with Pebble or RocksDB (grocksdb) — feasibility, risk, performance, effort

- **Method:** static analysis plus desk research; no experiments run.
- **Author:** Claude (Claude Code), 2026-11-01. Written independently. `analysis/storage-agy.md` appeared concurrently and was not read.
- **Versions analyzed:** etcd `v3.7.3` = `0a173d0410a9` (checkout verified at session start; no newer stable tag). bbolt `v1.5.0` `tx.go` fetched read-only from `raw.githubusercontent.com` to verify commit behavior. Upstream etcd `origin/main` consulted only where stated. Candidate engines (from release pages and source, not built): Pebble `v2.1.7` (module `github.com/cockroachdb/pebble/v2`, 2026-08-24); grocksdb `v1.11.1` (2026-09-07), which wraps RocksDB `11.1.2`.
- **Upstream references:**
  - etcd PR [#17119](https://github.com/etcd-io/etcd/pull/17119): deletes force an immediate backend commit.
  - etcd PR [#22425](https://github.com/etcd-io/etcd/pull/22425): `NonBlockingDefrag`, on `main` for v3.8, not in v3.7.x.
  - **etcd issue [#22332](https://github.com/etcd-io/etcd/issues/22332)** (open, opened 2026-08-25 by maintainer serathius): scale storage to 30 GB and prototype alternative engines including Pebble, with prototype numbers. PR [#22339](https://github.com/etcd-io/etcd/pull/22339) (open): the MVCC correctness harness used for those prototypes.
  - Earlier attempts: [#2646](https://github.com/etcd-io/etcd/issues/2646), [#10311](https://github.com/etcd-io/etcd/pull/10311), [#10321](https://github.com/etcd-io/etcd/issues/10321), [#13232](https://github.com/etcd-io/etcd/issues/13232), [#13140](https://github.com/etcd-io/etcd/issues/13140).
  - `Documentation/postmortems/v3.5-data-inconsistency.md`.
  - Industry prior art (AWS EKS, GKE, Alibaba, etcd-compatible stores) is in §8.

## Summary

Terms used throughout. **Backend**: etcd's persistent key-value store under the MVCC layer, today bbolt (a copy-on-write B+tree in one mmapped file), wrapped by `server/storage/backend`. **WAL**: etcd's Raft write-ahead log, a separate set of files, fsynced before an entry counts as persisted. **Apply loop**: the single goroutine that applies committed Raft entries to the MVCC store and backend. **Commit** (backend commit): one bbolt write transaction made durable. etcd batches many applied entries into one commit. **CI** (consistent index): the Raft index of the last entry reflected in the backend, stored inside the backend.

1. **The premise needs checking before any engine work. "Too much write activity" does not by itself mean the storage engine is the bottleneck.** A write passes through five serialized stages: gRPC → Raft replication with WAL fsync on a quorum → single-threaded apply → backend commit → reply. Swapping the engine changes only the last two. Raft, the WAL, and the in-memory key index (`treeIndex`) stay exactly as they are. *(High: code structure verified.)* Which stage binds for your workload can be measured from metrics etcd already exports (§3.4, experiment E0). That measurement should come first.
2. **There is a real, specific mechanism by which an LSM engine could help.** Every bbolt commit runs two `fdatasync` calls (bbolt `tx.go:566-568, 613-615 @ v1.5.0`). It also holds the batch-transaction lock and the read-transaction lock (`batch_tx.go:242-245, 354-359 @ v3.7.3`). That stalls the apply loop and blocks new reads from starting for the length of the commit. Commits happen every 100 ms, and also synchronously:
   - inside the apply loop for **every Raft entry that deletes from the backend**, which includes **every lease revocation** (`batch_tx.go:335`, `lessor.go:359-361`);
   - after every 1,000-key batch of MVCC compaction (`kvstore_compaction.go:90`).

   An LSM run with its own WAL disabled turns a commit into a memtable insert with no fsync. Durability is then deferred to explicit flushes before etcd truncates its WAL (§5.2). *(High that the mechanism exists; how much it costs your workload is unmeasured.)*
3. **Upper bound on the gain:** if the apply loop is the bottleneck, throughput can rise by at most about 1/(1 − f), where f is the fraction of wall time spent in backend commits (`rate(etcd_disk_backend_commit_duration_seconds_sum)`). If Raft/WAL fsync or the network binds instead, an engine swap buys roughly nothing in throughput, though it may cut read tail latency. *(High for the arithmetic; the value of f for your system is unknown.)*

   External evidence points both ways:
   - Steady-state inserts barely improve:
     - an etcd contributor found bbolt was not the bottleneck and Badger "did not change much" (#13232);
     - the upstream Pebble prototype gains about 13% at the storage layer alone (#22332).
   - The large measured gains are elsewhere:
     - writes during MVCC compaction and defrag (#22332);
     - moving bbolt off network block storage, which AWS EKS did by putting bbolt on tmpfs after replacing Raft with an internal journal (§8).

   *(Moderate.)*
4. **An LSM brings new costs on this workload:**
   - Range reads do one point lookup per revision (`kvstore_txn.go:126`), which is slower in an LSM than in a page-cached B+tree. The upstream prototype measured Pebble at about 6.5× lower Get throughput and about 5× lower Range throughput than bbolt (#22332; compression on, "mostly vibed" implementation).
   - Full scans (startup index rebuild, HashKV, and compaction, which rescans from revision 0 every time per `kvstore_compaction.go:44`) must step over tombstones.
   - Background compaction I/O competes with WAL `fdatasync`. The WAL fsync *is* the client-latency critical path, so this risk is the one most likely to cancel the gain.

   *(Moderate: mechanisms verified in code, sizes unknown.)*
5. **Feasibility:** etcd's engine seam is narrower than expected. bbolt types appear in 12 non-test files. MVCC, lease, auth and membership code talk only to the `backend.Backend`/`BatchTx`/`ReadTx` interfaces (§5.1). The hard parts are semantics, not API surface: durability and the CI contract, the snapshot/backup file format, defrag/size/quota meaning, and keeping the robustness-test failpoints. *(High.)*
6. **Pebble is clearly preferable to grocksdb for etcd.**
   - Pebble is pure Go and keeps etcd's `CGO_ENABLED=0` builds (`scripts/build_lib.sh:29`), Go profiling, and Go memory accounting.
   - grocksdb needs cgo and a C++ toolchain, and it has effectively one maintainer doing version bumps.
   - RocksDB-only features (column families, compaction filters, rate limiter) are real advantages, but none is needed for a first prototype.

   *(Moderate-High; see §6.)*
7. **Effort** *(Low confidence; assumes 1–2 engineers strong in Go and storage, not etcd maintainers)*:
   - About **3–5 engineer-weeks** for a benchmarkable Pebble prototype that is not production-safe.
   - About **6–12 engineer-months** to production quality, including crash-consistency verification, snapshot/backup compatibility, migration tooling and soak testing.
   - Then **ongoing fork maintenance**, which gets more expensive at each etcd minor release. Upstream `main` has already rewritten `backend.go` for v3.8 (PR #22425: +564 lines).
8. **Before an engine swap, evaluate a cheaper alternative aimed at the same mechanism:** decouple bbolt commits from the apply loop, or at least add delete support to etcd's write buffer so that lease revocations stop forcing synchronous commits. This keeps the file format, snapshots, backups and tooling intact (§4.4). *(Low-Moderate: my design proposal, not reviewed upstream.)*
9. **Upstream is now actively exploring the same change, and that should reshape the fork plan.**
   - In #22332 (open since 2026-08-25), etcd maintainer serathius proposes storage-interface decoupling and alternative-engine prototypes, and reports MVCC-layer numbers (no Raft/WAL). Against bbolt, Pebble:
     - writes about 1.13× faster at baseline (34k vs 30k QPS; the issue's "~1.7×" label does not match its own figures);
     - keeps write throughput steady during MVCC compaction, where bbolt drops 47%;
     - keeps write throughput up during defrag, where bbolt drops to ~100 QPS;
     - uses much less disk, mostly from compression;
     - reads much more slowly.
   - Another maintainer (ahrtr) is cautious: he asks for an official benchmark tool and a B+tree-vs-LSM analysis first, and says 30 GB is not a problem for bbolt but defrag is.
   - Two earlier attempts found little whole-server gain: Badger in 2018 (#10311) and 2021 (#13232).
   - **The plan:**
     - Fork only as a bridge, and align it with the upstream decoupling.
     - Add Pebble *next to* bbolt behind a flag.
     - Keep the bbolt file as the snapshot and backup interchange format.
     - Pin v3.7.x.
     - Contribute your workload's benchmark data to #22332.

   *(Moderate; §8, §9. Forecast that upstream ships an alternative engine within ~12 months: Low.)*

---

## 1. Premise check

The motivation stated is "performance issues due to amount of write activity". Two implicit premises need testing:

**(a) That the backend engine is the binding constraint.** etcd's write path (verified below, §2) has five serialized stages. The engine is involved only at the end, and its cost reaches clients only through the apply loop. Three non-engine stages commonly bind first:
- WAL `fdatasync` latency, especially on network block storage;
- the leader↔follower round trip;
- apply-loop CPU spent in MVCC, `treeIndex`, protobuf and watch notification.

etcd's performance documentation attributes commit latency mainly to network and disk I/O and puts the bbolt step at "tens of microseconds" per request. An etcd contributor who swapped in Badger reported the result "did not change much" (#13232) (§8). If WAL fsync or the network binds, a new engine will not raise write throughput. *(High.)*

**(b) That more capacity on one etcd cluster is the right fix for a write-heavy workload.** Everything passes through one Raft log and one apply goroutine, whatever the engine. If your write volume can be partitioned by domain across several independent etcd clusters, that scales nearly linearly and needs no fork. Kubernetes uses this pattern to move Events to a separate cluster. Whether it fits your consistency requirements depends on whether you need atomic transactions *across* partitions. I do not know your system.

Neither premise is shown false. Both are unverified, and the first experiment (E0, §10) tests (a) directly using metrics you probably already collect.

## 2. How etcd v3.7.3 writes, and where the backend sits (verified)

### 2.1 The pipeline

| Stage | What happens | Code (`@ v3.7.3`) | Changed by engine swap? |
|---|---|---|---|
| Admission | Reject if applied index lags committed index by > 5,000 (`ErrTooManyRequests`) | `server/etcdserver/v3_server.go:51, :1064` | Indirectly (a faster apply loop lags less) |
| Raft + WAL | Leader sends AppendEntries **in parallel** with its own WAL write; WAL `Save` fsyncs when Raft requires it | `server/etcdserver/raft.go:237-242, :256` | **No** |
| Hand-off to apply | Committed entries go to `applyc`, then a FIFO scheduler, then `applyAll` | `raft.go:146, :232`; `server.go:843-845` | No |
| Apply | MVCC write txn: takes the batch-tx lock, `UnsafeSeqPut` of the revision record, `treeIndex` update, lease attach/detach, watch notify | `server/storage/mvcc/kvstore_txn.go:174-185, :223-262` | Partly: the per-op backend `Put` cost |
| Backend commit | Periodic, or forced (§2.3); runs pre-commit hook (writes CI), bbolt `Commit` (2× fdatasync) | `server/storage/backend/batch_tx.go:261-288, :354-386` | **Yes, this is the main lever** |
| Reply | Client waiter is triggered after apply | — | No |

### 2.2 How etcd uses the backend: data layout and access patterns

**Layout.** Buckets: `key`, `meta`, `lease`, `alarm`, `cluster`, `members`, `members_removed`, `auth`, `authUsers`, `authRoles` (`server/storage/schema/bucket.go:25-56`).

The `key` bucket is keyed by **revision**: an 8-byte main revision, `_`, an 8-byte sub-revision, plus an optional tombstone mark. Its value is a marshaled `mvccpb.KeyValue` containing the user key and value (`kvstore_txn.go:236-259`). Revisions only increase, so **writes to the dominant bucket are appends at the right edge of the B+tree**. etcd sets `FillPercent = 0.9` for them (`batch_tx.go:158-162`). That is the *best* case for a B+tree and removes much of the usual LSM-vs-B-tree write-amplification argument (§3.3). *(High.)*

**User-level deletes are not backend deletes.** An MVCC delete writes a tombstone revision with `UnsafeSeqPut` (`kvstore_txn.go:308-323`). The only backend deletes are:
- MVCC compaction (`kvstore_compaction.go:57`);
- lease deletion (`schema/lease.go:59`), on every revoke, whether from expiry or explicit;
- rare auth, membership and alarm changes (`schema/*.go`).

**Reads.** A range request resolves keys to revisions in the in-memory `treeIndex`. It then does **one backend point lookup per result revision** (`kvstore_txn.go:108-151`, lookup at `:126`). There is no backend range scan on the read path.

**Full scans** happen in three places:
- startup, which rebuilds `treeIndex` from the whole `key` bucket (`kvstore.go:327-372`);
- HashKV, a full-bucket `UnsafeForEach` (`hash.go:33-40`), run for the HashKV RPC and for periodic corruption checks if `--corrupt-check-time` is set;
- MVCC compaction, which scans from revision 0 up to the compaction revision **every time**, because `last` starts zeroed (`kvstore_compaction.go:44, :53`).

**All keys and revision metadata live in RAM** (`treeIndex`, `kvstore.go:100`) regardless of engine. A new engine lets *values* exceed RAM, but not the key set. *(High.)*

### 2.3 Backend commit: triggers, cost, and what it blocks

**Triggers** (all verified):
1. The timer: every `--backend-batch-interval` (default 100 ms) if anything is pending (`backend.go:35-36, :441-457`).
2. The batch limit: `--backend-batch-limit` (default 10,000) pending ops (`batch_tx.go:335`).
3. **Any pending backend delete**, at the end of the transaction that issued it (`batch_tx.go:335`, introduced by PR #17119). The in-code comment explains why: the read buffer cannot represent deletes, so without an immediate commit later reads could see stale data and break linearizability (`batch_tx.go:318-334`). Together with `lessor.go:343-361`, this means **each applied lease revocation runs a full synchronous bbolt commit inside the apply loop.** Revocations are capped at 1,000/s by default (`lessor.go:46`).
4. MVCC compaction: `ForceCommit` after every `--compaction-batch-limit` (default 1,000) deleted keys, then sleeps `--compaction-sleep-interval` (default 10 ms) (`kvstore_compaction.go:85-98`; `kvstore.go:44-45`).
5. Before every disk snapshot (every `--snapshot-count`, default 10,000 entries), to guarantee CI ≥ the snapshot index before WAL segments are released (`server.go:78, :2096-2105, :2124-2130`).
6. Before `Hash`, compaction scheduling, `Snapshot()` and defrag (`kvstore.go:125, :164, :226, :299`; `backend.go:360`).

**Cost.** bbolt `Tx.Commit` does rebalance → spill (serialize dirty nodes into pages, allocate from the freelist) → write pages → `fdatasync` → write meta page → `fdatasync` (bbolt `tx.go:170-220, :520-568, :595-615 @ v1.5.0`). etcd disables freelist persistence (`NoFreelistSync: true`, `config_linux.go:29-32`), so there is no freelist write. Two synchronous flushes per commit is the floor. Exported metrics split the time into `rebalance`/`spill`/`write` (`backend/metrics.go`).

**What it blocks** (verified by reading the lock order; magnitude unmeasured):
- `batchTxBuffered.commit` takes `readTx.Lock()` while holding the batch-tx lock (`batch_tx.go:354-359`). **The apply loop cannot start its next write transaction until the commit finishes.**
- Creating a `ConcurrentReadTx`, which is the main read path, takes `readTx.RLock()` (`backend.go:280`). **New reads cannot start during a commit either.** Reads already in flight continue on their own snapshot.
- Separately, `hashByRev` holds `ReadTx().RLock()` for an entire HashKV full scan (`kvstore.go:198-200`). The write-back at the end of each write transaction needs `readTx.Lock()` (`batch_tx.go:310`), so **HashKV stalls applies for the duration of the scan**. This only matters if periodic corruption checks are enabled (`--corrupt-check-time`, default 0) or HashKV is called. It is engine-independent.

So a backend commit is not in the background. Its cost lands on the apply loop and on read admission, at least 10×/s, plus once per delete-bearing entry and once per compaction batch.

### 2.4 Durability contract between WAL and backend (verified)

- The CI and the Raft ConfState are written **inside the same bbolt transaction as the data**, by the pre-commit hook (`server/storage/hooks.go:45-54`; `batch_tx.go:361-366`). Atomicity of CI with the data is the property whose violation caused the v3.5 data-inconsistency incident (`Documentation/postmortems/v3.5-data-inconsistency.md:13, :31-44`).
- The backend is **allowed to lag the WAL**. Commits are asynchronous (100 ms), and on restart etcd replays WAL entries above the CI.
- The one hard requirement: **before the WAL is truncated past index i, the backend's durable CI must be ≥ i.** etcd enforces this with `KV().Commit()` before `SaveSnap`/`Release` (`server.go:2096-2130`).
- If the backend is behind a received Raft snapshot, `RecoverSnapshotBackend` replaces it with the snapshot DB file (`server/storage/backend.go:102-114`).

**This contract is what makes an LSM with its WAL disabled viable** (§5.2): only the snapshot-time commit has to be durable. Every other commit can be a memtable insert. *(High that the contract is as described. Moderate that no other code path silently relies on per-commit durability; experiment E6 must verify it.)*

## 3. Performance: theoretical model

### 3.1 What an LSM engine (WAL disabled) removes from the critical path
1. **The two fdatasyncs per commit.** The commit becomes "apply the batch to the memtable", which takes microseconds to low milliseconds. This affects every trigger in §2.3. The largest per-event effect is on **delete-forced commits (lease revocations)**. For a rough sense of scale, a revocation costs at least 2 × fdatasync latency + spill on the apply loop:
   - at ~1 ms per fdatasync, typical of network block storage, that caps revocations near ~500/s and stalls all other applies meanwhile;
   - at ~0.05–0.2 ms, local NVMe with power-loss protection, the cap is several thousand per second.

   *(Code path: High. Device latencies: recalled typical values, Low-Moderate; measure in E1.)*
2. **Spill/rebalance CPU while holding locks.** Replaced by memtable inserts. Flush and compaction CPU moves to background goroutines.
3. **Read admission stalls** during commits (§2.3), because a commit no longer holds a lock that new reads need for milliseconds.
4. **Defrag.**
   - In v3.7.x, defrag is stop-the-world for the member: it locks the batch tx, the DB and the read tx for the whole copy (`backend.go:476-614`). Upstream v3.8 adds `NonBlockingDefrag` (Beta, default on, PR #22425), which shrinks this to a short catch-up phase.
   - An LSM reclaims space continuously through compaction, so defrag becomes a no-op or a manual compaction.
5. **File growth during long read transactions** (follower snapshot sends, `backend.go:359-401`). bbolt cannot reuse pages freed after the oldest open read transaction began (recalled bbolt design, High). LSM snapshots also pin data (memtables and SSTs) but do not block the writer.

### 3.2 What it does not change
- Raft replication, WAL fsync on the quorum, network RTT, leader election behavior.
- Apply-loop CPU outside the backend: `treeIndex` operations, lease bookkeeping, protobuf marshaling (`kvstore_txn.go:250`), and watch fan-out (`server/storage/mvcc/watchable_store.go`).
- Memory needed for `treeIndex` (all keys and revisions).
- The single-apply-goroutine design and the 5,000-entry apply-gap admission limit.

### 3.3 What it makes worse, or risks making worse
1. **Point lookups on the read path.**
   - bbolt reads are a B+tree descent over mmapped, usually page-cached pages, with no copying.
   - An LSM lookup checks the memtable(s), then filters and index blocks per level, with block-cache lookups and copies.
   - Every range result is a point lookup (`kvstore_txn.go:126`), so large LIST-style reads could get several times more expensive per item. One mitigation: reuse a single iterator and `SeekGE` in revision order. Results come back in key order, so this needs a sort.
   - Measured upstream (#22332, MVCC layer, 10 KB compressible values, Pebble with compression, prototype not optimized):

     | Read type | bbolt | Pebble |
     |---|---|---|
     | Get | 260k QPS | 40k QPS |
     | Range | 200–300 QPS | 40–60 QPS |

     Some of that gap is block decompression and some is LSM structure. The split is unknown.

   *(Direction: High, given the upstream measurement. Magnitude for your values and with compression off: unknown, measure in E4.)*
2. **Tombstone scans.** MVCC compaction deletes old revisions at the low end of the `key` keyspace. That is the LSM "queue" anti-pattern: deletes at the head, appends at the tail. Until LSM compaction drops the tombstones, three things iterate over them:
   - the next MVCC compaction scan, which starts from zero (`kvstore_compaction.go:44`);
   - startup restore;
   - HashKV.

   With retained history and frequent compaction this can make those scans much slower than on bbolt, where deletes free pages immediately at commit. An LSM fork could fix the compaction case by seeking past the previous compaction point, an etcd-level change. *(Moderate.)*
3. **I/O interference with the WAL.** LSM flush and compaction write in bursts. If the backend and WAL share a device, WAL `fdatasync` tail latency can rise, and that is the client-latency critical path. Mitigation: separate devices, plus I/O throttling. RocksDB has a configurable rate limiter. I have not verified an equivalent in Pebble (Low). This is the risk most likely to cancel the gain. *(Moderate.)*
4. **Write stalls.**
   - bbolt has none, only periodic commit pauses.
   - An LSM stalls writers when L0 or memtables back up.
   - The tail-latency profile changes from "periodic short pauses" to "rare long pauses under compaction debt".
   - Sizing the memtable and L0 thresholds for peak write rates is required.
5. **Write amplification is not a clear win here.**
   - With right-edge appends and 100 ms batching under load, bbolt rewrites each leaf page about once, plus a few branch, meta and root pages per commit. Amplification is low at high batch sizes.
   - It is high only for tiny commits: roughly 6–10 pages for a single small entry. Those are exactly the delete-forced commits.
   - A leveled LSM rewrites data once per level it descends. For non-overlapping append-only keys, many compactions can be "move" compactions, but every flushed SST also contains the hot `meta` keys (CI, overwritten every commit), so files overlap in that range.

   Net write amplification for etcd's pattern has to be measured (E4). It should not be assumed to favor the LSM. *(Low.)*
6. **Space-amplification semantics.**
   - bbolt's file size equals the used high-water mark until defrag.
   - LSM disk usage includes obsolete versions awaiting compaction.
   - The quota check (`quota.go:126-133`) uses `Backend.Size()`, so a NOSPACE alarm could fire on transient compaction debt unless redefined.
7. **Memory.** A block cache and memtables are explicit memory, unlike bbolt's reliance on the OS page cache. Pebble memory is Go-visible (Pebble needs no cgo). RocksDB memory sits outside Go's view, so `GOMEMLIMIT` and Go heap profiles miss it.

### 3.4 Quantitative envelope from existing metrics

These come from metrics etcd v3.7.3 already exports. Names are verified in `server/storage/backend/metrics.go`, `server/storage/mvcc/metrics.go` and `server/etcdserver/metrics.go`, and are prefixed `etcd_` / `etcd_debugging_`.

- f_commit = `rate(etcd_disk_backend_commit_duration_seconds_sum[5m])` is the fraction of wall time spent in backend commits. The apply loop and read admission are blocked during that time.
- commits/s = `rate(etcd_disk_backend_commit_duration_seconds_count[5m])`. A value well above `1 / batch-interval` (10/s) means forced commits from deletes, compaction or the batch limit. That points at lease churn or compaction.
- Breakdown: `etcd_debugging_disk_backend_commit_{rebalance,spill,write}_duration_seconds`.
- WAL: `etcd_disk_wal_fsync_duration_seconds`. An engine swap does not change it.
- Apply saturation: committed − applied index (`etcd_server_proposals_committed_total` − `etcd_server_proposals_applied_total`, both gauges), `etcd_server_proposals_pending`, client `etcdserver: too many requests` errors, and "apply request took too long" log warnings (threshold `--warning-apply-duration`).
- Compaction: `etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds`, `..._total_duration_milliseconds`, `..._keys_total`.
- Size: `etcd_mvcc_db_total_size_in_bytes` and `..._in_use_in_bytes`; defrag `etcd_disk_backend_defrag_duration_seconds`.

**Bound.** If the apply loop is saturated (the gap grows toward 5,000 at peak), max throughput gain ≤ 1/(1 − f_commit + ε). Here ε is the residual commit and stall cost under the LSM, plus any added read and scan cost. For example:
- f_commit = 0.10 → at most about 1.11×;
- f_commit = 0.30 → about 1.43×;
- f_commit = 0.60 → about 2.5×.

If the apply loop is *not* saturated, the throughput gain from the engine is about 0. The only remaining benefit is shorter read and apply stalls, visible as p99 latency. *(High for the arithmetic.)*

### 3.5 Where the case for an LSM is strongest, and where it is weakest

| Workload trait | Effect of LSM engine | Confidence |
|---|---|---|
| High lease churn (grant/revoke), or many `DeleteRange`s of leased keys | Large: removes one synchronous 2×fdatasync commit per revoke from the apply loop | Moderate |
| Slow fdatasync on the backend device (network block storage) | Large: every commit carries it | Moderate |
| Large DB, frequent defrag pain (v3.7.x) | Large operationally; partly addressed by v3.8 `NonBlockingDefrag` | Moderate |
| WAL fsync p99 dominates client latency | ~None | High |
| Apply loop not saturated | ~None in throughput | High |
| Read-heavy with big ranges/LISTs | Possibly negative | Moderate |
| Backend and WAL on the same device | Risk of negative WAL tail latency | Moderate |

## 4. Options

### 4.1 Option P: Pebble backend (pure Go LSM)
Feasible within the existing seam (§5). It needs a new implementation of `Backend`, `BatchTx` and `ReadTx`, plus the semantic work in §5.2–5.7. Keeps `CGO_ENABLED=0`. Details in §6.

### 4.2 Option R: RocksDB via grocksdb (cgo)
Same seam and the same semantic work, plus:
- a cgo build with RocksDB 11.1.2 and four compression libraries;
- cross-compilation via C++ toolchains;
- release-pipeline changes;
- per-call cgo overhead on every `Put`, `Get` and iterator step, which matters for etcd's many small operations (Moderate; not measured);
- a single-maintainer wrapper (§6).

RocksDB-only features could pay off:
- **column families**, to separate append-only revisions from hot `meta`/`lease` keys (atomic across CFs only with WAL or `atomic_flush`);
- **compaction filters**, for lazy MVCC garbage collection. HashKV is logical and skips revisions scheduled for deletion (`hash.go:60-85`), so physically lazy deletion would not break hash comparison. *(Moderate.)* grocksdb's filter is a Go callback per key, i.e. a cgo callback;
- **I/O rate limiter**;
- **universal/FIFO compaction**.

### 4.3 Baseline: tuning without code changes (must be measured first, for a fair comparison)
- Separate the WAL device (`--wal-dir`) from the backend device, and pick devices by fdatasync latency (E1).
- `--backend-batch-interval` and `--backend-batch-limit`: larger batches mean fewer commits, but more read-buffer copying for `ConcurrentReadTx`. Measure.
- `--compaction-batch-limit` and `--compaction-sleep-interval`: fewer, larger compaction commits.
- Auto-compaction retention: less retained history makes compaction scans and DB size smaller.
- Leave `--corrupt-check-time` at 0, or make it rare: each check runs full scans that stall applies (§2.3).
- Client side:
  - batch logically related writes into one `Txn`, so one Raft entry and one apply carry several ops;
  - avoid rewriting unchanged values, since every put creates a revision;
  - **reuse leases** instead of grant/revoke per object;
  - keep values small. Each value is written to the WAL and to the backend. Client-side value compression reduces both, which is the direction Kubernetes is taking (KEP PR kubernetes/enhancements#6321, §8).
- Partition across multiple etcd clusters where cross-partition atomicity is not needed. AWS EKS reports "up to five times the write throughput" from this alone (§8).
- **bbolt page size** (needs a small fork patch, since etcd exposes no flag): an etcd maintainer measured 64 KB pages as best for 1 KB values (#22332 comment, §8). Unvalidated for other value sizes. *(Low.)*

### 4.4 Option B′: keep bbolt, decouple commits from the apply loop (my proposal; not reviewed upstream)
The two largest LSM benefits (§3.1 items 1–3) come from taking the commit off the apply loop. etcd already keeps an in-memory write buffer and read buffer over bbolt (`server/storage/backend/tx_buffer.go`). Two increments:
1. **Delete support in the buffers.** Add tombstone entries to `txWriteBuffer`/`txReadBuffer`, so deletes no longer force an immediate commit (`batch_tx.go:335`). This targets lease-revocation and compaction commit storms. The change is small and possibly upstreamable. Precedent: PR #17119 introduced the forced commit precisely because the buffer lacked delete semantics.
2. **Asynchronous commit.** Commit the previous bbolt write transaction in a background goroutine while the apply loop keeps writing into a new in-memory layer. Reads merge the committed bbolt view with the in-flight layers. This is in effect a memtable in front of bbolt. It is subtle: the read-merge logic has had linearizability bugs before (#17119). It also stays inside `server/storage/backend`, keeps the file format, snapshots, backups, metrics and tools, and needs no data migration.

If E0/E2 show that the commit is the bottleneck, B′ should be evaluated before P or R, because its production surface is far smaller. If the bottleneck is LSM-specific (defrag, DB size beyond what bbolt handles well), B′ does not help.

## 5. Feasibility: what has to change (static analysis at v3.7.3)

### 5.1 The seam
- Non-test files importing `go.etcd.io/bbolt`: 12. These are the backend package (`backend.go`, `batch_tx.go`, `read_tx.go`, `config_*.go`), `server/storage/schema/version.go` (`ReadStorageVersionFromSnapshot(*bbolt.Tx)`), `server/config/config.go` and `server/embed/config.go` (freelist type flag), `etcdutl/snapshot/v3_snapshot.go`, `server/storage/mvcc/testutil/hash.go`, and `tools/etcd-dump-db/backend.go`.
- Non-test packages using the `backend` interfaces include `server/storage/mvcc` (6 files), `server/storage/schema` (14), `server/lease`, `server/etcdserver/{cindex,apply,api,…}`, `etcdutl/etcdutl` (5 commands), `server/verify`, `tools/benchmark`, and `tools/etcd-dump-db`. They use `UnsafePut`/`UnsafeSeqPut`/`UnsafeDelete`/`UnsafeRange`/`UnsafeForEach`, bucket creation, `Lock*`/`Unlock`, `Commit`/`ForceCommit`. All of these map onto any ordered KV store. Map buckets to a 1-byte key prefix in a single keyspace.
- 27 test files import `backend` or `bbolt`.
- The bbolt-specific implementation is about 1,300 lines (`backend.go` 734, `batch_tx.go` 406, `read_tx.go` 151, configs). The buffering layer `tx_buffer.go` (259) is engine-agnostic.

### 5.2 Durability mapping (the correctness-critical part)
- **One write batch per etcd commit.** CI and ConfState go into the same batch through the existing `OnPreCommitUnsafe` hook (`hooks.go:45-54`). That keeps the atomicity required by the v3.5 postmortem.
- **Engine WAL disabled, non-synced commits.** In Pebble, `Options.DisableWAL`. With it, `Sync` writes fail (`"pebble: WAL disabled"`, Pebble `db.go:852 @ v2.1.7`), and durability comes only from `Flush()`. `ForceCommit` therefore needs two modes:
  - a cheap "apply batch" for the periodic and forced commits;
  - a durable "apply + Flush" at the snapshot point (`server.go:2105`) and at `CommitAndStop`/close.

  Otherwise the WAL could be released past a CI that only lived in a lost memtable.
- **Crash-state requirement:** after a crash, the durable state must be a *prefix* of committed batches, so that the CI is consistent with the data.
  - Pebble does not document this for `DisableWAL`. At v2.1.7 the option comment still said "TODO(peter): untested"; `main` has since removed that line.
  - Pebble flushes a prefix of the memtable queue (`compaction.go:1689-1691 @ v2.1.7`). The prefix property should therefore hold if etcd never uses `Ingest`, because ingestion can make a non-overlapping file durable ahead of earlier unflushed batches (`ingest.go:1092-1097`).
  - For RocksDB, a single column family (or `atomic_flush` with several) is needed.

  *(Moderate; must be proven by crash testing, E6.)*
- **Fallback if the prefix property cannot be trusted:** keep the engine WAL on, but unsynced, and sync only at snapshot points. That costs one extra sequential write per commit and no fsync.

### 5.3 Read isolation
- bbolt read transactions give etcd point-in-time views. That matters because MVCC compaction deletes revisions concurrently with reads.
- In Pebble, use `NewSnapshot()` or a per-read iterator (iterators are point-in-time) to replace the per-batch bbolt read transaction.
- The existing read buffer can stay unchanged in a first version. Commits become cheap, so they can be made more frequent and the buffer copying in `ConcurrentReadTx` (`backend.go:279-352`) gets smaller.

### 5.4 Raft snapshots, backups, restore: the file format leaks
These parts assume the backend is **one bbolt file**:
- **Follower snapshots** stream `Backend.Snapshot().WriteTo` (bbolt `Tx.WriteTo`) (`snapshot_merge.go:38-39`; `backend.go:359-401`).
- The receiver saves `<index>.snap.db` (`api/snap/db.go:37, :118`) and **renames that file over `member/snap/db`** (`storage/backend.go:58-68`).
- Bootstrap checks `fileutil.Exist(cfg.BackendPath())` (`bootstrap.go:232`). Snapdir cleanup knows `db.tmp*` and `*.snap.db` (`api/snap/snapshotter.go:244-278`).
- **`etcdctl snapshot save`** streams the same bbolt bytes plus a SHA-256 (`v3rpc/maintenance.go:135-209`). **`etcdutl snapshot restore/status`** opens the file with bbolt, detects the hash by `size % 512 == 32` (`etcdutl/snapshot/v3_snapshot.go:94-98, :123`), and reads the storage version through `schema.ReadStorageVersionFromSnapshot(*bbolt.Tx)`.

Design choices:
- **(a) Keep the bbolt file as the interchange format** *(recommended)*. A Pebble member *exports* to a bbolt-format stream when it sends a snapshot or serves `snapshot save`, and *imports* when it receives one.
  - Pros: mixed bbolt/Pebble clusters work, which enables rolling migration and rollback. Backups stay restorable by stock etcd. Existing tooling keeps working.
  - Cost: building a bbolt file from a Pebble snapshot on each send. This is rare and proportional to DB size.
- **(b) A logical KV stream format.** Engine-neutral, but breaks compatibility with stock `etcdctl`/`etcdutl`.
- **(c) An engine-native checkpoint tarball** (Pebble `Checkpoint` after `Flush`). Fastest, but mixed clusters and stock backups break.

### 5.5 Maintenance semantics
- **`Defrag`**: no-op, or a manual compaction of the `key` range. The upstream gRPC API stays.
- **`Size`/`SizeInUse`**: redefine. For example, `Size` = total live SST + memtable bytes (Pebble metrics), and `SizeInUse` = estimated live data. The quota/NOSPACE alarm (`quota.go:126-133`) must not fire on transient compaction debt.
- **`Hash`** (`backend.go:403-431`) iterates buckets in name order and keys in byte order. A prefix-encoded LSM can reproduce it exactly if the prefix order matches bucket-name order *or* the hash code iterates per bucket.
- **HashKV** is logical and engine-agnostic (`hash.go:33-85`). Cross-member comparison in a mixed cluster still works.
- **Metrics** like `backend_commit_{rebalance,spill,write}` lose their meaning. Add LSM metrics: L0 files, compaction debt, write stalls, flush duration, block-cache hit rate.

### 5.6 Tests and failpoints
- `tests/robustness` (porcupine linearizability checks under faults) uses 20 named gofail failpoints. About 12 sit in backend or compaction code: `beforeCommit`, `afterCommit`, `beforeStartDBTxn`, `afterStartDBTxn`, `beforeWritebackBuf`, `afterWritebackBuf`, `commitBefore/AfterPreCommitHook`, `defragBeforeCopy/Rename`, `compact*` (`tests/robustness/failpoint/gofail.go`). A new engine must provide equivalent markers, or that coverage is lost.
- The robustness suite is black-box at the API level. It is the most valuable correctness asset for the fork.

### 5.7 Build and release
- Pebble needs no build changes.
- grocksdb needs `CGO_ENABLED=1`, which etcd turns off by default (`scripts/build_lib.sh:29`). It also needs static RocksDB and compression libraries for each target architecture, and changes to container images.

## 6. Pebble vs grocksdb

| Criterion | Pebble | grocksdb (RocksDB) | Verified? |
|---|---|---|---|
| Latest | v2.1.7 (2026-08-24), "used in production by CockroachDB 25.3.x"; newer work only on `master` (pseudo-versions) | v1.11.1 (2026-09-07), RocksDB 11.1.2 | V (release pages) |
| Language / build | Pure Go; zstd cgo optional with a pure-Go fallback | cgo + C++ RocksDB + snappy/zlib/lz4/zstd | V |
| License | BSD-3-Clause (not affected by CockroachDB's 2024 relicensing) | grocksdb MIT; RocksDB dual GPLv2/Apache-2.0 | V |
| Maintenance | Cockroach Labs team; README promises no API stability for embedders; format upgrades one-way | Effectively one maintainer; last 12 months of commits are version bumps | V |
| Production use | CockroachDB default since 20.2 (2020); go-ethereum default since v1.12.0 (2023) | Through RocksDB itself: very wide. Through grocksdb: cometbft-db and cosmos-db (optional) | V (per agent research; see §8 sources) |
| Disable WAL, explicit flush | Yes (`DisableWAL`; `Flush`) | Yes (`disableWAL`, `Flush`, `atomic_flush`) | V Pebble; RocksDB recalled (High) |
| Column families | No | Yes | V |
| Compaction filters | No | Yes (Go callback through cgo) | V |
| Checkpoint / SST ingest / range deletes | Yes | Yes | V Pebble; RocksDB recalled (High) |
| Memory visibility to Go runtime | Yes | No (C heap) | Inferred (High) |
| Per-op call overhead | Go function calls | One cgo crossing per Put/Get/iterator step | Inferred (Moderate) |

**Verdict.** Pebble for the prototype and, if it pays off, for production. The asymmetry is deliberate: the RocksDB-only features are optimizations, while cgo plus a single-maintainer wrapper are structural costs for a Go system that ships static binaries. Revisit grocksdb only if a Pebble prototype shows LSM CPU is the remaining bottleneck *and* a RocksDB-specific feature (for example a compaction filter for MVCC GC) addresses it. *(Moderate-High.)*

**Pebble caveat.** The README describes Pebble as "focused on … internal usage by CockroachDB" and promises no API compatibility. Pin a release branch (`pebble-release-2.1` / v2.1.x), budget for re-porting on upgrades, and remember that format-major-version upgrades are irreversible.

## 7. Risks

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | Engine is not the bottleneck; months spent for <10% gain | Unknown until E0; plausible | High (wasted effort) | E0 + E2 gate before any coding |
| 2 | Crash leaves CI inconsistent with data (silent divergence across members) | Low if designed per §5.2; catastrophic if wrong | Critical | Prefix-property proof + crash tests (E6) + robustness suite + `InitialCorruptCheck` and periodic HashKV in soak |
| 3 | WAL fsync tail latency worsens due to compaction I/O | Moderate on shared devices | High | Separate devices; throttling; measure WAL p99 in E5 |
| 4 | Read regressions (per-revision point lookups, tombstone scans) | Moderate | Medium–High for LIST-heavy clients | Iterator reuse; compaction seek fix; measure in E4/E5 |
| 5 | Snapshot/backup format break blocks rolling migration and rollback | High if not designed in | High | Interchange format (a) in §5.4 |
| 6 | Fork drift: costly rebases (v3.8 already rewrote `backend.go`) and delayed security fixes | High over time | Medium–High | Isolated engine seam; pin 3.7.x; scheduled re-ports; or upstream a seam |
| 7 | Pebble API/format churn on upgrade | Moderate | Medium | Pin release branch; upgrade deliberately |
| 8 | Ops ecosystem (dashboards, defrag jobs, quota alerts, backup scripts) silently wrong | High | Medium | Redefine metrics explicitly; runbook review |
| 9 | grocksdb only: build/release complexity; single maintainer; C-heap memory invisible to Go | High (if chosen) | Medium | Prefer Pebble |

## 8. Prior art and upstream position

Sources were read by a research pass on 2026-11-01. Items marked **[V]** were checked against the primary page. I re-read #22332 and its comments, and the status of #22339, directly from the GitHub API. Web pages were read through a summarizing fetch, so quotes from them are close but not checked character by character.

**Upstream etcd**
- **#2646 (2015): why bbolt was chosen.** The requirements were "Pure Go, Reliable, Predictable Performance, Non-blocking Snapshot". bbolt was preferred over leveldb and bitcask, with measured write amplification of about 2–4×. [V]
- **#10311 (2018–2020): Badger backend experiment by xiang90, then a maintainer.** He reported a "large performance improvement for big databases: over 16GB". He closed it after B-tree fixes made bbolt "good enough", saying he would revisit "if we hit another perf issue that LSM might be a better fit". [V]
- **#10321 (2018): "Meta issue: Pluggable backend".** Closed by the stale bot with no decision. One participant argued that reimplementing the etcd API, as CosmosDB did, beats swapping the engine. [V]
- **#13232 (2021): "rocksdb as backend?"** A contributor wrote "we didn't see the boltdb as the bottleneck… mainly… storage latency" and "I actually tried to use badgerdb and the result did not change much." [V]
- **#13140 (2021): in-memory backend.** gyuho noted that a pluggable backend "requires multiple phases, starting with more generic peer-to-peer snapshot contracts". That is the same snapshot-format problem as §5.4. [V]
- **#22332 (2026-08-25, open; serathius, maintainer): "Increase supported storage size to match K8s scalability requirements".** [V, read directly]
  - **Deliverables:** standardized storage benchmarks and correctness tests; storage-interface decoupling; prototypes "including possibility of alternative storage engines".
  - **Motivation:** Kubernetes pods of 10–100 KB, 30 GB targets, stop-the-world defrag, and followers unable to catch up via multi-GB snapshots.
  - **Prototype table** (MVCC layer, 150k keys × 10 KB, written ~10×; the issue notes the Badger and Pebble implementations were "mostly vibed" and checked by the PR #22339 linearizability harness):

    | Metric | bbolt | Pebble |
    |---|---|---|
    | Write QPS, baseline | ~30k | ~34k |
    | Write QPS during compaction | ~15.9k (−47%) | ~35.7k |
    | Write QPS during defrag | ~100 | ~38.4k |
    | Disk, baseline (× logical size) | 1.01× | 0.16× |
    | Disk, after compaction | 2.03× | 0.62× |
    | Get QPS | 260k | 40k |
    | Range QPS | 200–300 | 40–60 |

  - **Comments:**
    - ahrtr (maintainer): "30GB isn't a problem at all for bbolt"; he suggests tuning bbolt page size (in his test 64 KB was best for 1 KB values); "the defragmentation… is indeed a hard problem"; later, "we should have an official & mature benchmark tool first" (#16467) and "LSM-tree may have big write amplification overall."
    - jberkus asked who maintains Pebble long term.
    - serathius: "Will see if I can take it as far as replacing bbolt across whole etcd and passing robustness tests."
    - gyuho suggested leaving value compression to Kubernetes (KEP PR kubernetes/enhancements#6321).
  - **My reading:**
    - The bbolt −47% during compaction matches the mechanism in §2.3: a forced commit every 1,000 deleted keys, holding the locks.
    - The Pebble footprint numbers depend heavily on how compressible the synthetic values are.
    - None of these numbers include Raft or the WAL.
- **PR #22339, "Storage correctness"** (serathius, 2026-08-26): open, not merged. [V]
- In v3.7.3 the bbolt freelist type already defaults to `map`: the Alibaba fix below is on by default and is not a remaining tuning lever (`server/etcdmain/help.go:85`, `server/embed/config.go:1396-1402 @ v3.7.3`). [V-local]

**Hyperscalers**
- **AWS EKS "ultra scale" (blog, 2025-07-16).** [V] https://aws.amazon.com/blogs/containers/under-the-hood-amazon-eks-ultra-scale-clusters/
  - AWS replaced etcd's Raft with an internal journal and moved bbolt "from network-attached Amazon EBS volumes to fully in-memory storage with tmpfs", for "order-of-magnitude" throughput gains. Durability is delegated to the journal.
  - It doubled the supported database size to 20 GB.
  - It split hot resource types into separate etcd clusters for "up to five times the write throughput".
  - Relevance: bbolt I/O on network storage mattered enough for AWS to remove it. The tmpfs move was only safe because AWS also replaced the durability layer. Partitioning was a separate multiplier.
- **GKE.** The 65k-node post (2024-11-13) describes moving from etcd to a Spanner-based store behind the etcd API. The 130k-node post (2025-11-21) describes a "proprietary key-value store based on Google's Spanner". [V] Relevance: the largest operators replaced the whole stack, not just etcd's engine.
- **Alibaba (CNCF blog, 2019-05-09).** Above about 40 GB, bbolt spill took about 8 s. The fix was a hashmap freelist (bbolt PR #141), now the default. The headline "24×" is a narrow benchmark: 1M puts rate-limited to 5k/s, completing in 210 s vs 4,974 s. [V]

**etcd-compatible stores and independent ports**
- **Xline:** CURP consensus protocol, etcd API, RocksDB engine. No comparable numbers found. [V for engine]
- **KubeBrain (ByteDance):** TiKV/Badger. Its own benchmark: inserts ~28.6k vs etcd 3.4 ~10.7k ops/s, deletes ~4.9k vs ~10.8k, on a different and larger node count. The repo is possibly dormant (last push 2024-05). [V]
- **Kine (k3s):** etcd API over SQL or NATS. [V]
- **"etcd-lsm" blog (May 2026, unofficial):** describes an opt-in `--backend=pebble` port behind `backend.Backend` with Pebble's WAL disabled. That is the same design as §5.2. A `kill -9` durability gate is in progress, and no benchmark numbers are published. [V] https://ahmetb.github.io/etcd-lsm/

**etcd documentation**
- Default quota 2 GiB; "8 GiB is a suggested maximum". [V]
- Defrag blocks reads and writes on the member. [V]
- The performance page attributes commit latency mainly to network and disk I/O, with the bbolt step "tens of microseconds" per request. [V] https://etcd.io/docs/v3.6/op-guide/performance/

**What the prior art implies for you**
- An engine swap is not a new idea. A competent Pebble port at the MVCC layer is now cheap to *build*; serathius's was largely AI-generated.
- The open problems are the ones this document flags:
  - durability with the engine WAL off;
  - the snapshot contract;
  - read regressions;
  - whether whole-server write throughput improves when the WAL, not the engine, is the bottleneck.

  Each earlier attempt that measured end-to-end found modest gains for steady-state writes. The large wins are compaction, defrag, footprint, and getting backend I/O off slow devices. *(Moderate.)*

## 9. The fork plan

**Assessment.** In v3.7.x, forking is the only way to change the engine: etcd has no pluggable backend. Upstream, though, is now openly working toward storage-interface decoupling and engine prototypes (#22332). Your fork and upstream's work would overlap in exactly the same files. For **prototyping and experiments E5–E6**, the plan is right. For **production** it turns your team into maintainers of a consensus database's storage layer, with five lasting obligations:
- security and bug-fix backports from etcd 3.7.x and later minors;
- re-porting at every etcd minor release;
- your own release pipeline;
- your own robustness CI;
- your own on-call understanding of a storage engine.

Recommendations if you proceed:
1. **Add, don't replace.**
   - Introduce an engine interface inside `server/storage/backend`, with the bbolt implementation unchanged and the default, and Pebble opt-in through a flag.
   - Benefits: A/B comparison on the same build, upstream tests still exercise bbolt, rebase conflicts stay mostly in one package, and rollback is a configuration change.
2. **Pin the v3.7.x line.** Upstream `main` has already restructured `backend.go` (`d6af8028d`, PR #22425, 2026-09-14: `NonBlockingDefrag`), so a v3.8 re-port will conflict in exactly the files a backend fork rewrites. Treat each minor upgrade as a planned port.
3. **Keep the bbolt file as the interchange format** for Raft snapshots and `snapshot save` (§5.4 option a). This keeps three paths open:
   - **rolling migration:** add a Pebble member as a learner; it receives a bbolt snapshot and imports it; promote it; remove a bbolt member; repeat;
   - **rollback:** the reverse;
   - **disaster recovery** with stock `etcdutl`.
4. **Keep the fork delta small and named.** Put engine code in new files and keep seams minimal in shared files, so `git rebase` conflicts stay readable.
5. **Run `tests/robustness` on every fork change and every upstream rebase**, with Pebble-equivalent failpoints.
6. **Engage with #22332 rather than diverge from it.**
   - Contribute three things:
     - your workload as a benchmark profile (write-heavy, your value sizes, lease use);
     - E0/E2 results;
     - help with the "storage layer decoupling" deliverable.
   - Shape your fork's engine seam after whatever interface upstream converges on. The aim is that the fork shrinks over time instead of growing.
   - Whether upstream ships an alternative engine, and when, is undecided: one maintainer is driving it and another is skeptical. Plan the fork as a bridge that may become permanent. *(Forecast that an alternative engine ships in an etcd release within ~12 months: Low.)*
7. **Consider B′ (§4.4) first.** It may be a much smaller fork, or even upstreamable, if the commit coupling is what hurts. Note that it does not address the problems #22332 is driven by: defrag, footprint, and snapshot size.

## 10. Experiments to decide (to run on hardware representative of production)

General rules: record the etcd version and commit, cluster size and topology (cross-AZ RTT), CPU, RAM, disk model and type, filesystem and mount options, value/key size distributions, concurrency, and exact commands (AGENTS.md §9 rule 4). Use open-loop load at fixed offered rates (`tools/benchmark --rate`) and report p50/p99/p99.9 latency per rate plus max sustainable throughput, not only closed-loop maxima.

**E0 — Attribution on the production cluster or a production replay (no code; 1–3 days). Gate 1.**
- Collect §3.4 metrics at peak, along with leader CPU profiles (`--enable-pprof`, 30–60 s). From the profiles, compute CPU share in bbolt (`Tx.Commit`, `node.spill`, `Cursor.seek`) vs MVCC, `treeIndex`, protobuf, gRPC and watch.
- Compute f_commit, commits/s, the apply gap trend, the WAL fsync p99, and the lease revoke rate (`etcd_server_lease_expired_total` plus explicit revokes).
- Decision:
  - If the apply loop is unsaturated and WAL fsync or network dominates, stop engine work and pursue §4.3.
  - If f_commit ≥ ~0.2 or commits/s ≫ 10, continue.

  The thresholds are a judgment call; agree on them with the team beforehand.

**E1 — Device characterization (½ day).**
- Measure `fdatasync` latency distributions with `fio` for a WAL-like pattern: sequential ~2–8 KiB writes with fdatasync each (the etcd docs suggest `fio --rw=write --ioengine=sync --fdatasync=1 --bs=2300 --size=22m`; recalled, verify against the current docs).
- Also measure a bbolt-commit-like pattern: a few random 4 KiB pwrites, then fdatasync, then one 4 KiB write, then fdatasync.
- This gives the per-commit floor for §3.1 item 1.

**E2 — Ceiling: bbolt on tmpfs, WAL on real disk (no code; 2–3 days). Gate 2.**
- Run three members with `--data-dir` on tmpfs and `--wal-dir` on the production-class disk, against a baseline with both on disk. Replay the workload with `tools/benchmark put`/`txn-mixed`/`watch*`, plus the lease-churn harness from E3.
- This bounds what *any* engine could gain from eliminating backend I/O. It does not remove spill CPU or lock holding, which the profiles cover.
- It is unsafe for production: the backend is lost on reboot.
- Decision: if the ceiling gain on the agreed target metric is below the agreed threshold (for example 30% in sustainable throughput, or in p99 at target load), stop.

**E3 — Commit-trigger microbenchmarks (2–3 days).**
1. `tools/benchmark mvcc put` (backend + MVCC without Raft; `tools/benchmark/cmd/mvcc.go:37-39, :57-58`; `mvcc-put.go:50-54`). Sweep `--batch-interval`/`--batch-limit` and the value size.
2. A small Go-client harness for **lease churn**: grant, attach a key, revoke, at rates of 100–2,000/s, with puts running concurrently. Measure put p99 and commits/s. `tools/benchmark lease-keepalive` only exercises keepalives, not grant/revoke churn, so the harness is needed.
3. Compaction under load: `tools/benchmark put --compact-interval --compact-index-delta` (`put.go:69-70`) while measuring put p99.

**E4 — Engine microbenchmark with etcd's real access pattern (1–2 weeks, including the harness).**
- Capture the backend op stream from a representative etcd: instrument a worktree to log `UnsafePut`/`SeqPut`/`Delete`/`Range` and commit boundaries. Alternatively, synthesize the stream from §2.2.
- Replay it against bbolt (etcd's `backend` package), Pebble (WAL off, NoSync, Flush every N), and optionally grocksdb.
- Measure:
  - commit latency distribution;
  - CPU per op;
  - bytes written per payload byte (`/proc/<pid>/io`);
  - space amplification at steady state;
  - point-lookup p99 with a warm and a cold cache;
  - full-scan throughput after many compaction cycles (the tombstone effect).
- Run for hours, until LSM compaction reaches steady state and the DB is multi-GB.
- Writing a new harness is justified: `tools/benchmark` cannot drive alternative engines.
- Start from upstream's work instead of from scratch: the `BenchmarkStorageWrite` MVCC benchmark shown in #22332 and the correctness harness in PR #22339. Neither is merged; both are reported to support Pebble and Badger variants. Then extend it with your value distribution, lease churn, compression on/off, and Raft-free vs full-cluster runs.

**E5 — Pebble backend prototype in a fork worktree (3–5 weeks). Gate 3.**
- Implement `Backend`/`BatchTx`/`ReadTx` over Pebble, with `Defrag` as a no-op and snapshot transfer done by exporting to bbolt format (or skipped for benchmark clusters).
- Repeat E2/E3 on three-member clusters. Measure WAL fsync p99 alongside, for risk 3.
- In parallel, if E0 implicated delete-forced commits, prototype B′ step 1 (§4.4) and compare.
- Decision: proceed to production engineering only if the improvement holds at p99 across the full workload mix, not just puts.

**E6 — Correctness and crash consistency (before any production use; weeks).**
- Run `tests/robustness` with Pebble-equivalent failpoints.
- Do kill -9 loops under load, and power-loss emulation (dm-log-writes or a fault-injecting FUSE filesystem such as LazyFS).
- Verify after each crash: CI ≤ durable data, members' HashKV agree, and no lost acknowledged writes.
- Soak for weeks with snapshots, member replacement, and the bbolt↔Pebble mixed-cluster migration and rollback.

## 11. Effort estimate (Low confidence)

Assumes 1–2 engineers fluent in Go and storage engines but new to etcd internals.

| Work item | Prototype | Production |
|---|---|---|
| Engine seam inside `backend` + Pebble `Backend`/`BatchTx`/`ReadTx` | 2–3 wk | 4–6 wk |
| Durability mapping (Flush at snapshot, close), crash-prefix verification | 0.5 wk | 4–8 wk (incl. E6) |
| Snapshot send/receive, `snapshot save`/`restore`/`status` via bbolt interchange | skip | 4–6 wk |
| Defrag/Size/quota/metrics semantics, dashboards, alerts | minimal | 2–4 wk |
| etcdutl and tools (`defrag`, `hashkv`, `migrate`, `etcd-dump-db`) | skip | 2–3 wk |
| Tests: port 27 test files' assumptions, failpoints, robustness CI | minimal | 4–8 wk |
| Migration and rollback tooling and rehearsal | skip | 3–6 wk |
| Soak and performance qualification | E5 only | 4–8 wk |
| **Total** | **~3–5 engineer-weeks** | **~6–12 engineer-months** |
| grocksdb instead of Pebble | +1–2 wk (build) | +1–2 months (build/release/multi-arch, memory tooling) |
| Ongoing fork maintenance | — | ~0.25–0.5 FTE, plus a port at each etcd minor |

B′ step 1 (delete support in buffers): about 1–3 weeks to a reviewed patch, including linearizability testing *(Low)*.

Calibration: upstream built MVCC-layer Pebble and Badger prototypes quickly, "mostly vibed" (#22332). That suggests the prototype column could be at the low end. It says nothing about the production column, which is dominated by durability, snapshot compatibility, migration and verification.

## 12. Inputs needed from the team
- Peak and average write rates, and the op mix: put, txn, delete, lease grant/revoke/keepalive, range/LIST, watch counts and fan-out.
- Key count, key and value size distributions, update rate per key, retained history (compaction settings), DB size and in-use size, defrag frequency and impact.
- Cluster size, topology and RTT, disk types (local NVMe vs network block), separate WAL device or not, CPU and RAM, and the etcd version in production.
- Current symptoms: client latency percentiles, `too many requests` errors, leader changes, slow-apply warnings, and the §3.4 metrics at peak.

## 13. What I verified vs inferred, and open questions

**Verified in code** (`@ v3.7.3` unless noted):
- the pipeline and lock structure (§2.1–2.3);
- commit triggers, including delete-forced commits and lease revocation;
- bbolt's two fdatasyncs (`v1.5.0`);
- the CI-in-same-transaction hook and the snapshot-time commit (§2.4);
- the bbolt-file assumptions in snapshot, backup and restore (§5.4);
- the bbolt import surface and test count (§5.1);
- the failpoint list;
- the CGO default;
- upstream `NonBlockingDefrag` on `main`.

**Verified from primary web sources:**
- via a research agent: Pebble and grocksdb versions, licenses and API features, the Pebble `DisableWAL` and `Flush` behavior (§6), and the prior art in §8;
- by me, directly through the GitHub API: #22332's body and comments, and #22339's status.

**Inferred, not tested:**
- that an LSM without its own WAL meets etcd's durability contract (§5.2);
- direction and size of read and scan regressions;
- write-amplification comparison;
- I/O interference with the WAL;
- every effort figure.

**Recalled:** typical fdatasync latencies; RocksDB features; bbolt's page-reuse behavior under long read transactions; the etcd docs' fio recipe.

**Open questions:**
- Is there any code path besides the snapshot-time commit that relies on a backend commit being durable? A first pass found none; E6 should confirm.
- Pebble's guarantee for the crash state with `DisableWAL`: can Cockroach Labs or the source history confirm the prefix property?
- How large is the tombstone-scan penalty for etcd's compaction pattern in Pebble? Does Pebble's tombstone-density compaction mitigate it?
- Where will #22332 land, whether bbolt tuning, an engine seam, or Pebble, and when? Would upstream accept B′ step 1 independently? (§8)
- How much of the #22332 Pebble read slowdown is compression vs LSM structure? Rerun with compression off in E4.
