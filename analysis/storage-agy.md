# Replacing etcd's `bbolt` Storage Backend with `grocksdb` or `pebble`: Static Code Analysis, Theoretical Evaluation, and Experimental Design

- **Versions analyzed:**
  - `etcd`: `v3.7.3` (`0a173d0410a9`, released 2026-10-22)
  - `go.etcd.io/bbolt` (etcd's current embedded B+tree storage engine, declared in `server/go.mod:20 @ v3.7.3`): `v1.5.0`
  - `github.com/cockroachdb/pebble` (CockroachDB's pure-Go Log-Structured Merge-tree [LSM] key-value engine): `v2.1.7` (latest stable release)
  - `github.com/linxGnu/grocksdb` (Cgo bindings for Meta's C++ RocksDB LSM engine, supporting RocksDB 7.x–11.x): `v1.11.1` (released 2026-09-07)
- **Upstream references:**
  - [etcd#10321](https://github.com/etcd-io/etcd/issues/10321) — *Meta issue: Pluggable backend* (closed 2020-04-23)
  - [etcd#10311](https://github.com/etcd-io/etcd/pull/10311) — *WIP: try out badger as an alternative backend* (closed 2019-10-03; maintainer prototype by `xiang90`)
  - [etcd#17119](https://github.com/etcd-io/etcd/pull/17119) — *Commit transaction for each configuration change* (merged 2023-12-21; introduced synchronous `bbolt` commit on any `UnsafeDelete`)
- **Summary:**
  1. **Premise correction — replacing `bbolt` does not remove the synchronous Raft write-ahead log (`WAL`) `fdatasync` on the client write path (High confidence).** Every mutating request in etcd (`Put`, `DeleteRange`, mutating `Txn`, `LeaseGrant`, `LeaseRevoke`, `Compact`) is appended and `fdatasync`-persisted to the Raft WAL (`server/storage/wal/wal.go:881, 995-1037 @ v3.7.3`) before the Raft state machine applies it to the storage backend (`server/etcdserver/raft.go:196-267 @ v3.7.3`). Moreover, normal `Put` and `DeleteRange` operations do **not** synchronously `fdatasync` `bbolt`; they append monotonically increasing 17- or 18-byte revision keys (`server/storage/mvcc/revision.go:34-98 @ v3.7.3`) to an in-memory buffered transaction (`batchTxBuffered`, `server/storage/backend/batch_tx.go:228-386 @ v3.7.3`) that commits asynchronously every `100ms` (`defaultBatchInterval`) or `10,000` operations (`defaultBatchLimit`). If the team's write bottleneck is Raft WAL `fdatasync` latency or single-leader Raft serialization on a small dataset (< 1–2 GiB), replacing `bbolt` with `grocksdb` or `pebble` will yield near-zero write throughput improvement.
  2. **Where `bbolt` *is* the write bottleneck, an LSM backend (`pebble` or `grocksdb`) eliminates five structural pathological stalls in `v3.7.3` (High confidence):**
     - **Synchronous `bbolt` `fdatasync` on every `UnsafeDelete` (`server/storage/backend/batch_tx.go:335 @ v3.7.3`):** Because etcd's in-memory write/read transaction buffer (`txBuffer`, `server/storage/backend/tx_buffer.go:53-259 @ v3.7.3`) cannot represent deletion tombstones, any backend `UnsafeDelete` increments `pendingDeleteOperations` and forces an immediate synchronous `bbolt.Tx.Commit()` (`fdatasync`) inside `Unlock()`. This hits **every single `LeaseRevoke`** (`server/lease/lessor.go:359 @ v3.7.3`) and **every MVCC compaction batch** (`server/storage/mvcc/kvstore_compaction.go:57, 85, 90 @ v3.7.3`), stalling the single-threaded Raft apply loop (`applyEntries`).
     - **Periodic `100ms` / `10,000`-op `bbolt` commit stalls (`server/storage/backend/batch_tx.go:261-288, 354-386 @ v3.7.3`):** During `bolt.Tx.Commit()`, `bbolt` holds both `batchTx.Mutex` (blocking all Raft `applyEntries` writes) and `readTx.mu.Lock()` (blocking all new read transactions while swapping the read transaction buffer) while executing B+tree rebalancing, dirty 4 KiB page spilling, random page writes across the database file, freelist persistence, and two `fdatasync` calls (data pages + meta page).
     - **Copy-on-Write (CoW) B+tree write amplification during MVCC compaction on large datasets (> 2–8 GiB):** While normal `Put` writes append monotonically at the right edge of the `key` bucket's B+tree (`UnsafeSeqPut` with `FillPercent = 0.9`, `server/storage/backend/batch_tx.go:108-115 @ v3.7.3`), MVCC compaction deletes scattered historical revisions across older 4 KiB leaf pages, triggering CoW page copies, tree rebalancing, freelist fragmentation, and severe random write I/O.
     - **Stop-the-world defragmentation (`Defrag()`, `server/storage/backend/backend.go:476-614 @ v3.7.3`):** Reclaiming fragmented space in `bbolt` requires rewriting all live keys into a new file while holding `batchTx.Lock()`, `b.mu.Lock()`, and `readTx.mu.Lock()`, completely freezing reads and writes on that member. LSM background compaction reclaims space continuously without stop-the-world locks.
     - **Kernel `mmap` page-fault stalls when the DB exceeds RAM:** When the `bbolt` file outgrows page cache, B+tree leaf lookups inside `applyEntries` or `rangeKeys` block on synchronous major page faults over `mmap`.
  3. **`pebble` is strictly superior to `grocksdb` as an etcd backend replacement (High confidence).** Because etcd's multiversion concurrency control (`mvcc`) layer keeps all user keys in an in-memory B-tree (`treeIndex`, `server/storage/mvcc/index.go:39-66 @ v3.7.3`) and resolves a range query over $N$ user keys by issuing **$N$ separate backend point lookups by revision** (`server/storage/mvcc/kvstore_txn.go:135-145 @ v3.7.3`), `grocksdb` incurs $O(N)$ Go-to-C (`cgo`) boundary crossings, slice allocations, and manual off-heap memory management per range read. `pebble` is pure Go, eliminates `cgo` overhead, provides `pebble.IndexedBatch` (which natively supports read-your-own-writes *with deletion tombstones*, directly replacing both `bbolt.Tx` and `txBuffer`), supports native range-delete tombstones (`DeleteRange`) for $O(1)$ contiguous revision compaction, and preserves single-binary Go builds.
  4. **Before forking etcd to replace `bbolt`, two low-effort interventions within the existing architecture must be evaluated (High confidence):** (a) adding deletion tombstones to `txBuffer` (`server/storage/backend/tx_buffer.go`) so `LeaseRevoke` stops triggering synchronous `bbolt` `fdatasync` commits (`batch_tx.go:335`), and (b) isolating the Raft WAL (`--wal-dir`) onto low-latency NVMe storage and tuning `--backend-batch-interval` / `--backend-batch-limit`.
  5. **Replacing `bbolt` with `pebble` in a fork requires ~1,800–2,600 LoC across ~15 files (estimated 6–10 engineer-weeks for a production-grade implementation; Moderate confidence)** and carries three non-obvious architectural traps verified in `v3.7.3`:
     - **Bucket prefix collision trap:** Concatenating `append(bucket.Name(), key...)` (as done in upstream prototype PR [#10311](https://github.com/etcd-io/etcd/pull/10311)) corrupts keyspace isolation in `v3.7.3` because bucket name `"auth"` (`schema.Auth`) is a byte prefix of `"authUsers"` (`schema.AuthUsers`) and `"authRoles"` (`schema.AuthRoles`), and `"members"` (`schema.Members`) is a prefix of `"members_removed"` (`schema.MembersRemoved`) (`server/storage/schema/bucket.go:42-58 @ v3.7.3`).
     - **Client-side snapshot checksum modulo-512 trap:** Both the Go client library (`client/v3/snapshot/v3_snapshot.go:35-39, 102 @ v3.7.3`) and `etcdutl` (`etcdutl/snapshot/v3_snapshot.go:95-99, 475 @ v3.7.3`) validate streamed snapshots using `hasChecksum(n int64) bool { return (n % 512) == sha256.Size }`, which assumes the raw database payload is an exact multiple of 512 bytes (true for 4 KiB-paged `bbolt` files, false for arbitrary LSM checkpoint tar streams unless explicitly zero-padded to a 512-byte boundary before the 32-byte SHA-256 trailer).
     - **Double-WAL vs. `consistent_index` crash-recovery invariant:** etcd's crash recovery (`server/etcdserver/bootstrap.go:451-461 @ v3.7.3`) and verification (`server/verify/verify.go:111-131 @ v3.7.3`) require that `consistent_index` and applied MVCC state be persisted atomically (`OnPreCommitUnsafe`, `server/storage/hooks.go:45 @ v3.7.3`) and never exceed `WAL.HardState.Commit`.

---

## 1. Premise Check: Deconstructing etcd's Write Path (`v3.7.3`)

Before investing engineering effort into replacing `bbolt`, we must check the premise that `bbolt` is the primary bottleneck for write-heavy workloads in etcd. In etcd `v3.7.3`, every mutating request passes through **two distinct storage engines** and an **in-memory index**.

```
Client Put / DeleteRange / Txn / LeaseGrant / LeaseRevoke
   │
   ▼
[1] Raft Propose & Leader Append
   │  ├── Leader sends MsgApp to followers in parallel
   │  └── Leader writes entry + HardState to Raft WAL (`server/storage/wal/wal.go:995`)
   │      └── SYNCHRONOUS `fileutil.Fdatasync(w.tail().File)` (`wal.go:881`)  ◄── MANDATORY PER-BATCH FSYNC
   │
   ▼ (Once quorum acknowledges WAL persistence)
[2] Single-Threaded Raft Apply Loop (`applyEntries`, `server/etcdserver/server.go:1175, 1982`)
   │  └── `uberApplier.Apply()` (`server/etcdserver/apply/uber_applier.go:117`)
   │      ├── Acquires `batchTxBuffered.Lock()` (`server/storage/backend/batch_tx.go:304`)
   │      ├── Updates in-memory `treeIndex` (userKey -> Revision) (`server/storage/mvcc/index.go:72`)
   │      ├── Serializes `mvccpb.KeyValue` protobuf (`server/storage/mvcc/kvstore_txn.go:248`)
   │      ├── Calls `UnsafeSeqPut(schema.Key, revBytes, data)` (`kvstore_txn.go:259`)
   │      │   ├── Appends `(revBytes -> data)` into `bbolt.Bucket` (in-memory dirty page, FillPercent=0.9)
   │      │   └── Appends `(revBytes -> data)` into `txWriteBuffer` (`batch_tx.go:350`)
   │      └── Calls `batchTxBuffered.Unlock()` (`batch_tx.go:308`)
   │          ├── If `pending >= 10000 || pendingDeleteOperations > 0`:
   │          │     SYNCHRONOUS `bbolt` Commit (`fdatasync` x2)  ◄── BLOCKING STALL IF TRIGGERED
   │          └── Otherwise: returns immediately to client! (NO `bbolt` disk I/O)
   │
   ▼ (Background Goroutine: `backend.run()`, `server/storage/backend/backend.go:445`)
[3] Asynchronous Periodic Backend Commit (every `defaultBatchInterval = 100ms`)
      └── Calls `batchTxBuffered.Commit()` (`batch_tx.go:261`)
          ├── Acquires `batchTx.Lock()` (blocks [2] `applyEntries`!)
          ├── Calls `OnPreCommitUnsafe()` -> writes `consistent_index` & `term` to `meta` bucket
          ├── Copies `txWriteBuffer` into `txReadBuffer` under `readTx.mu.Lock()` (blocks new reads!)
          └── Calls `bbolt.Tx.Commit()` -> rebalance, spill, random 4KiB writes, 2x `fdatasync`
```

### 1.1 Why Normal `Put` and `DeleteRange` Writes Are Sequential in `bbolt`

A common misconception is that random user keys (`Put("user/9f8a...", val)`) cause random B+tree leaf writes in `bbolt` during normal ingestion. **Verified in `server/storage/mvcc/kvstore_txn.go:196-348 @ v3.7.3`: they do not.**

1. **In-memory user-key index (`treeIndex`):** etcd's MVCC store maintains an in-memory B-tree (`treeIndex`, backed by `github.com/google/btree` with degree 32 in `server/storage/mvcc/index.go:39-66 @ v3.7.3`) that maps each raw user key (`[]byte`) to a `keyIndex` struct holding its generation history and revision numbers. User keys are **never** used as B+tree keys in `bbolt`.
2. **Monotonic revision keys in `bbolt`:** The `key` bucket (`schema.Key`, `server/storage/schema/bucket.go:43 @ v3.7.3`) in `bbolt` is indexed strictly by `Revision` (`struct { Main int64; Sub int64 }`), encoded as big-endian bytes (`server/storage/mvcc/revision.go:34-98 @ v3.7.3`):
   - Normal `Put` key: 17 bytes (`8-byte Main` + `'_'` + `8-byte Sub`).
   - MVCC tombstone (`DeleteRange`) key: 18 bytes (`8-byte Main` + `'_'` + `8-byte Sub` + `'t'`).
3. **Append-only `UnsafeSeqPut`:** Because `(Main, Sub)` increases strictly monotonically with every Raft command applied, both `storeTxnWrite.put` (`kvstore_txn.go:259 @ v3.7.3`) and `storeTxnWrite.delete` (`kvstore_txn.go:323 @ v3.7.3`) call `tw.tx.UnsafeSeqPut(schema.Key, ibytes, d)`. Inside `server/storage/backend/batch_tx.go:108-115 @ v3.7.3`, `UnsafeSeqPut` sets `bucket.FillPercent = 0.9` on the `bbolt` bucket and appends exclusively to the rightmost leaf page of the B+tree.
4. **Client reply before `bbolt` commit:** When `uberApplier.Apply` finishes applying an entry, `s.applyV3.Apply` returns and the server immediately triggers `s.w.Trigger(id, resp)` (`server/etcdserver/server.go:1236, 2048 @ v3.7.3`), unblocking the client's gRPC handler **before** the 100ms background `bbolt` commit runs—*unless* the batch limit (`10,000`) is reached or `pendingDeleteOperations > 0`.

> **Condition under which replacing `bbolt` fails to improve write performance (High confidence):**
> If your workload consists primarily of small `Put`/`DeleteRange` operations, has a total `bbolt` database size under ~1 GiB (fitting comfortably in OS page cache), has low lease revocation churn, and exhibits high `etcd_disk_wal_fsync_duration_seconds` relative to `etcd_disk_backend_commit_duration_seconds`, then your bottleneck is **Raft WAL `fdatasync` latency or single-leader Raft replication**. Replacing `bbolt` with `pebble` or `grocksdb` does not touch `server/storage/wal/` and will not speed up that workload.

---

### 1.2 The Five Mechanisms Where `bbolt` *Does* Bottleneck Write-Heavy Workloads

When does `bbolt` become the dominant bottleneck in etcd `v3.7.3`? Static analysis of `server/storage/` reveals five concrete mechanisms:

#### Mechanism 1: The `pendingDeleteOperations > 0` Synchronous Commit Trap (`batch_tx.go:335 @ v3.7.3`)
In PR [#17119](https://github.com/etcd-io/etcd/pull/17119) (merged Dec 2023), etcd modified `batchTxBuffered.Unlock()` (`server/storage/backend/batch_tx.go:308-340 @ v3.7.3`):

```go
func (t *batchTxBuffered) Unlock() {
	if t.pending != 0 {
		t.backend.readTx.mu.Lock() // blocks concurrent read transactions
		t.buf.writeBack(&t.backend.readTx.buf)
		t.backend.readTx.mu.Unlock()
		if t.pending >= t.backend.batchLimit || t.pendingDeleteOperations > 0 {
			t.commit(false)
		}
	}
	t.batchTx.Unlock()
}
```

Why does `t.pendingDeleteOperations > 0` force an immediate synchronous `t.commit(false)` (which calls `bbolt.Tx.Commit()` and `fdatasync`)? Because `txBuffer` (`server/storage/backend/tx_buffer.go:53-259 @ v3.7.3`) is a flat slice of `(key, val)` pairs (`bucketBuffer.buf []kv`) with **no deletion tombstone representation**. When `batchTx.UnsafeDelete(bucket, key)` deletes a key from `bbolt.Tx`, `txReadBuffer` still holds any older buffered copy of that key, and even if removed from `txReadBuffer`, a concurrent reader falling through `txReadBuffer` to the underlying `bbolt.Tx` read view (`baseReadTx.tx`) would still see the pre-delete value from the last committed `bbolt` transaction until `bbolt.Tx.Commit()` executes and reopens `readTx`!

While PR #17119 was motivated by rare cluster membership changes (`UnsafeDelete` in `members`), **two high-frequency code paths call `UnsafeDelete`:**
1. **Every `LeaseRevoke` (`server/lease/lessor.go:359 @ v3.7.3`):** When a lease expires or is revoked, `le.revoke()` calls `tx.UnsafeDelete(schema.Lease, int64ToBytes( int64(le.id) ))`. Because `LeaseRevoke` goes through Raft and runs on the main `applyEntries` goroutine, **every single `LeaseRevoke` forces a synchronous `bbolt` `fdatasync` commit inside `Unlock()`**, destroying the 100ms batching window! Under heavy lease churn (up to `defaultLeaseRevokeRate = 1000` revocations/sec, `server/lease/lessor.go:46 @ v3.7.3`), `bbolt` is forced to execute up to 1,000 synchronous B+tree commits + `fdatasync`s per second on top of the Raft WAL `fdatasync`.
2. **Every MVCC Compaction Batch (`server/storage/mvcc/kvstore_compaction.go:57, 85, 90 @ v3.7.3`):** `scheduleCompaction` runs in a background goroutine, deletes up to `defaultCompactionBatchLimit = 1000` expired revision keys per batch via `tx.UnsafeDelete(schema.Key, keys[i])`, calls `tx.Unlock()` (which triggers synchronous `t.commit(false)` at line 85 because `pendingDeleteOperations > 0`), and then immediately calls `s.b.ForceCommit()` at line 90 (triggering a second commit lock acquisition), sleeping only `10ms` (`defaultCompactionSleepInterval`) between batches.

**Complete Inventory of `UnsafeDelete` and `UnsafeDeleteBucket` Callers in etcd `v3.7.3`:**

*(Note: `KV.DeleteRange`, `KV.Txn` [with `DeleteRange` ops], `Lock.Unlock`, and `Election.Resign` do **not** call `UnsafeDelete`; they append an 18-byte MVCC tombstone revision `<main>_<sub>t` to the `key` bucket via `UnsafeSeqPut` in `server/storage/mvcc/kvstore_txn.go:323 @ v3.7.3`.)*

| gRPC Service / RPC (or Internal Trigger) | gRPC Entry Point (`@ v3.7.3`) | Backend `UnsafeDelete*` Call Site (`@ v3.7.3`) | Target Bucket | Frequency / Impact |
|---|---|---|---|---|
| **`Lease.LeaseRevoke`** (explicit client call **or** internal leader TTL expiration loop in `lessor.revokeExpiredLeases`, `server/lease/lessor.go:640`) | `server/etcdserver/api/v3rpc/lease.go:53` (`LeaseServer.LeaseRevoke`) | `server/lease/lessor.go:359` $\to$ `server/storage/schema/lease.go:58-59` (`UnsafeDeleteLease` $\to$ `tx.UnsafeDelete(Lease, ...)`) | `lease` | **High (up to 1,000/sec).** Synchronous `bbolt` `fdatasync` per revoked lease inside `applyEntries`. |
| **`KV.Compact`** (explicit client call **or** internal auto-compaction loop in `server/compactor/`) | `server/etcdserver/api/v3rpc/key.go:139` (`kvServer.Compact`) | `server/storage/mvcc/kvstore_compaction.go:57, 85` (`scheduleCompaction` $\to$ `tx.UnsafeDelete(schema.Key, keys[i])`) | `key` | **High (every 10ms / 1,000 keys during compaction).** Synchronous `bbolt` `fdatasync` per batch (`Unlock()` at line 85 + `ForceCommit()` at line 90). |
| **`Cluster.MemberRemove`** | `server/etcdserver/api/v3rpc/member.go:72` (`ClusterServer.MemberRemove`) | `server/etcdserver/api/membership/cluster.go:433` $\to$ `server/storage/schema/membership.go:79` (`MustDeleteMemberFromBackend` $\to$ `tx.UnsafeDelete(Members, mkey)`) | `members` | Rare (cluster reconfiguration; original motivation for PR #17119). |
| **`Auth.UserDelete`** | `server/etcdserver/api/v3rpc/auth.go:123` (`AuthServer.UserDelete`) | `server/auth/store.go:482` $\to$ `server/storage/schema/auth_users.go:49` (`UnsafeDeleteUser` $\to$ `tx.UnsafeDelete(AuthUsers, ...)`) | `authUsers` | Low (admin user management). |
| **`Auth.RoleDelete`** | `server/etcdserver/api/v3rpc/auth.go:75` (`AuthServer.RoleDelete`) | `server/auth/store.go:734` $\to$ `server/storage/schema/auth_roles.go:65` (`UnsafeDeleteRole` $\to$ `tx.UnsafeDelete(AuthRoles, ...)`) | `authRoles` | Low (admin role management). |
| **`Maintenance.Alarm`** (only when `action == DEACTIVATE`) | `server/etcdserver/api/v3rpc/maintenance.go:249` (`maintenanceServer.Alarm`) | `server/etcdserver/api/v3alarm/alarms.go:82` $\to$ `server/storage/schema/alarm.go:81` (`MustDeleteAlarm` $\to$ `tx.UnsafeDelete(Alarm, v)`) | `alarm` | Rare (clearing `NOSPACE` / `CORRUPT` alarms). |
| **`Maintenance.Downgrade`** (when downgrading storage schema from `v3.6` to `v3.5`) | `server/etcdserver/api/v3rpc/maintenance.go:322` (`maintenanceServer.Downgrade`) | `server/etcdserver/adapters.go:84` $\to$ `server/storage/schema/migration.go:82` $\to$ `server/storage/schema/actions.go:48` (`deleteKeyAction.unsafeDo` $\to$ `tx.UnsafeDelete(Meta, MetaStorageVersionName)`) | `meta` | Rare (cluster version downgrade). |
| **Offline CLI Tools** (`etcdutl snapshot restore` / `etcdutl migrate`) | Offline CLI only (not gRPC) | `server/storage/schema/membership.go:69, 130, 139` (`TrimClusterFromBackend` $\to$ `UnsafeDeleteBucket(Cluster)`; `TrimMembershipFromBackend` $\to$ `UnsafeDelete(Members/MembersRemoved, k)`); `server/storage/schema/version.go:66` (`UnsafeClearStorageVersion`) | `cluster`, `members`, `members_removed`, `meta` | Offline only. |

#### Mechanism 2: Mutual Exclusion Between `applyEntries`, `readTx`, and `bbolt.Tx.Commit()`
Unlike LSM engines where writes append to a lock-free/sharded in-memory MemTable while background threads flush and compact immutable SSTables, `bbolt` is a **single-writer CoW B+tree**.
- During `batchTxBuffered.unsafeCommit()` (`server/storage/backend/batch_tx.go:354-386 @ v3.7.3`), `batchTx.Mutex` is held across the entire `bbolt.Tx.Commit()` (`rerr = t.tx.Commit()` at line 378) **and** across `t.backend.readTx.mu.Lock()` (lines 364–380).
- Consequently, while `bbolt` rebalances dirty B+tree nodes, spills dirty 4 KiB pages, writes random pages to the OS, and calls `fdatasync` twice (once for dirty data pages, once for the 4 KiB meta page), **the Raft `applyEntries` loop cannot apply any writes or linearizable read barriers**, and **new `ConcurrentReadTx()` callers cannot copy the read buffer** (`server/storage/backend/backend.go:254-269 @ v3.7.3`).
- On multi-gigabyte databases where a single 100ms window accumulates thousands of dirty 4 KiB pages, `etcd_disk_backend_commit_duration_seconds` spikes from `<2ms` to `50ms–500ms+`, causing sawtooth latency spikes across all writes and reads.

#### Mechanism 3: Random 4 KiB CoW Write Amplification During MVCC Compaction
When `Put` and `DeleteRange` append new revisions via `UnsafeSeqPut`, they write sequentially to the highest revision numbers. However, when MVCC compaction (`scheduleCompaction`) deletes superseded revisions from minutes or hours ago, those 17-byte revision keys live in **old, densely packed (90% full) 4 KiB leaf pages scattered across the entire `bbolt` file**.
- Deleting a single 17-byte revision from an old 4 KiB page requires `bbolt` to copy the entire 4 KiB leaf page PLUS every 4 KiB ancestor branch page up to the root (`O(depth * 4 KiB)` write amplification per touched leaf subtree).
- If compaction deletes 1,000 keys scattered across 600 distinct leaf pages, a single compaction batch of ~100 KiB of logical data can dirty `>2.5 MiB` of scattered 4 KiB pages and force a synchronous `fdatasync` every 10ms.

#### Mechanism 4: Freelist Fragmentation and Stop-the-World `Defrag()`
As `bbolt` allocates new contiguous page extents for large transactions and frees scattered pages from old transactions, the file grows toward `--quota-backend-bytes` (default `2 GiB`, max recommended `8 GiB`, `server/storage/quota.go:32-38 @ v3.7.3`).
- Even with `FreelistMapType` (`bbolt.FreelistMapType`, default in `server/storage/backend/backend.go:128 @ v3.7.3`), `bbolt` never shrinks its file on disk or compacts partially empty 4 KiB leaf pages across non-adjacent branches without an explicit `Defrag()`.
- `backend.Defrag()` (`server/storage/backend/backend.go:476-614 @ v3.7.3`) acquires `b.batchTx.Lock()`, `b.mu.Lock()`, and `b.readTx.mu.Lock()`, opens a brand-new temporary `bbolt` file (`db.tmp`), iterates over every bucket and key to copy live data sequentially, syncs and closes both files, swaps them via `os.Rename`, and reopens the backend. During those seconds or minutes, the etcd member is completely unresponsive to reads and writes.

#### Mechanism 5: `mmap` Page Fault Stalls and OS Dirty-Page Writeback Contention
`bbolt` memory-maps (`mmap`) the entire database file (`InitialMmapSize = 10 GiB` on 64-bit Linux, `server/storage/backend.go:31-47 @ v3.7.3`). If the database size approaches or exceeds available physical memory (or container cgroup memory limits), B+tree traversal during `UnsafeRange` (`rangeKeys`) or `UnsafeDelete` (compaction) triggers major page faults directly on the critical lock path.

---

## 2. Code-Level Coupling Audit in etcd `v3.7.3`

How tightly coupled is etcd `v3.7.3` to `bbolt`? Upstream issue [etcd#10321](https://github.com/etcd-io/etcd/issues/10321) (*Meta issue: Pluggable backend*) and `xiang90`'s prototype PR [etcd#10311](https://github.com/etcd-io/etcd/pull/10311) (*WIP: try out badger as an alternative backend*) show that etcd originally designed `server/storage/backend.Backend` as an abstraction layer, **but several `bbolt`-specific assumptions leak outside `server/storage/backend/`**.

Below is the complete, verified inventory of every `bbolt` touchpoint in `etcd @ v3.7.3` (`0a173d0410a9`):

| Subsystem / File (`@ v3.7.3`) | Nature of `bbolt` Coupling | Impact on `pebble` / `grocksdb` Replacement |
|---|---|---|
| `server/storage/backend/backend.go:63-112` | Defines the `Backend` and `Snapshot` interfaces (`ReadTx`, `BatchTx`, `ConcurrentReadTx`, `Snapshot`, `Hash`, `Size`, `SizeInUse`, `OpenReadTxN`, `Defrag`, `ForceCommit`, `Close`, `SetTxPostLockInsideApplyHook`). | **Clean interface boundary.** None of the method signatures in `Backend` expose `*bbolt.DB` or `*bbolt.Tx`. |
| `server/storage/backend/backend.go:114-734` | Concrete `backend` struct wrapping `*bbolt.DB`, `batchTxBuffered`, `readTx`, `Hash(ignores)`, `Defrag()`, and `bolt.Open(...)`. | **Primary replacement target.** Either replace `backend` internals or implement a parallel `pebbleBackend` / `rocksBackend` satisfying `backend.Backend`. |
| `server/storage/backend/batch_tx.go:30-81` | Defines `Bucket`, `UnsafeReadWriter`, `BatchTx`, `BatchTxzek` interfaces (`UnsafeCreateBucket`, `UnsafeDeleteBucket`, `UnsafePut`, `UnsafeSeqPut`, `UnsafeRange`, `UnsafeDelete`, `UnsafeForEach`, `Lock`, `Unlock`, `Commit`, `CommitAndStop`). | **Clean interface boundary**, but requires mapping `Bucket` namespaces and `UnsafeRange(bucket, key, endKey, limit)` onto a flat LSM keyspace or Column Families (§3.2). |
| `server/storage/backend/read_tx.go:25-151` | Defines `UnsafeReader` and `ReadTx`; implements `readTx` and `concurrentReadTx` using `txReadBuffer` + `*bbolt.Tx`. | **Clean interface boundary.** With `pebble.IndexedBatch` or `pebble.Snapshot`, `txReadBuffer` copying in `ConcurrentReadTx()` can be simplified or replaced by lightweight LSM snapshots (§3.4). |
| `server/storage/backend/tx_buffer.go:1-259` | In-memory sorted slice cache (`txWriteBuffer`, `txReadBuffer`) built specifically to work around `bbolt`'s inability to allow concurrent readers to see uncommitted writes in the active `bbolt.Tx` without lock contention. | Can be **eliminated or replaced** by `pebble.IndexedBatch` + `pebble.Snapshot`, or fixed to support deletion tombstones (§3.3). |
| `server/storage/backend.go:53-113` | `newBackend`, `NewDefaultBackend`, `OpenSnapshotBackend`, `RecoverSnapshotBackend`. Line 65 executes `os.Rename(snapPath, cfg.BackendPath())`, assuming the snapshot (`%016x.snap.db`) and backend (`member/snap/db`) are **single regular files**. | **High coupling.** An LSM database is a **directory** of SSTables, MANIFEST, CURRENT, and WAL files, plus a streamable archive format for snapshots (§3.5). |
| `server/storage/schema/version.go:28-30` | `ReadStorageVersionFromSnapshot(tx *bbolt.Tx) *semver.Version` takes a raw `*bbolt.Tx` parameter. Called only in tests (`version_test.go`). | **Low coupling.** Unused by production server code (production uses `ReadStorageVersion(r backend.ReadTx)` at line 15). |
| `server/etcdserver/api/snap/db.go:39-132` | `SaveDBFrom(r io.Reader, id uint64)` streams a Raft/gRPC snapshot into a single file `filepath.Join(s.dir, fmt.Sprintf("%016x.snap.db", id))` and calls `fileutil.Fsync(f)`. | **Moderate coupling.** Fine if `Snapshot.WriteTo(w io.Writer)` serializes an LSM checkpoint into a single deterministic archive file (`.snap.db`) that `OpenSnapshotBackend` unpacks into the LSM directory (§3.5). |
| `server/etcdserver/api/v3rpc/maintenance.go:135-220` | `Maintenance.Snapshot` streams `snap.WriteTo(w)` over gRPC in 32 KiB chunks and appends a 32-byte `sha256.New()` checksum at the end (`lines 198-206`). | **Critical hidden coupling with `client/v3` and `etcdutl`!** See the `n % 512 == 32` trap below (§2.1). |
| `client/v3/snapshot/v3_snapshot.go:35-39, 102` & `etcdutl/snapshot/v3_snapshot.go:95-99, 475` | `hasChecksum(n int64) bool { return (n % 512) == sha256.Size }` checks whether the downloaded snapshot file size modulo 512 equals 32 (`sha256.Size`)! Furthermore, `etcdutl snapshot restore` (`v3_snapshot.go:258, 345, 505`) calls `bolt.Open` directly on the snapshot file to inspect/modify `members` and `meta` buckets. | **Critical compatibility trap.** `client/v3` requires the raw snapshot stream (`snap.WriteTo`) to be a multiple of 512 bytes. `etcdutl snapshot status/restore` directly opens the snapshot with `bbolt` (§2.1). |
| `server/verify/verify.go:65-138` | `verifyConsistentIndex` calls `backend.NewDefaultBackend` on the data dir and reads `schema.Meta` via `Backend.ReadTx()`. | **Zero `bbolt` coupling** once `NewDefaultBackend` opens the new backend. |
| `tools/etcd-dump-db/backend.go:1-178` | Diagnostic CLI tool importing `go.etcd.io/bbolt` directly to list buckets and iterate keys offline. | Must be updated to open the new backend or use `backend.Backend` interface. |

### 2.1 Two Non-Obvious Traps Discovered in the Code Audit

#### Trap A: The Bucket Prefix Collision Bug in `server/storage/schema/bucket.go:42-58 @ v3.7.3`
In `bbolt`, each bucket is a separate nested B+tree with its own isolated keyspace. When `xiang90` prototyped Badger (an LSM engine without buckets) in PR [#10311](https://github.com/etcd-io/etcd/pull/10311), the prototype mapped `(bucket, key)` into a flat LSM keyspace via naive byte concatenation:
```go
// FROM PR #10311 (BUGGY in etcd v3.7.3!):
rawKey := append(bucket.Name(), key...)
```
Let us check the 10 buckets defined in `server/storage/schema/bucket.go:42-58 @ v3.7.3`:
1. `key` (`"key"`)
2. `meta` (`"meta"`)
3. `lease` (`"lease"`)
4. `alarm` (`"alarm"`)
5. `cluster` (`"cluster"`)
6. `members` (`"members"`)
7. `members_removed` (`"members_removed"`)
8. `auth` (`"auth"`)
9. `authUsers` (`"authUsers"`)
10. `authRoles` (`"authRoles"`)
11. `test` (`"test"`)

Notice two prefix collisions:
- `"auth"` is a strict prefix of `"authUsers"` and `"authRoles"`.
- `"members"` is a strict prefix of `"members_removed"`.

If we concatenate `append(bucket.Name(), key...)` without a delimiter or length prefix:
- `UnsafeForEach(schema.Auth, ...)` or `UnsafeRange(schema.Auth, min, nil, 0)` scanning prefix `"auth"` will accidentally read and return keys from `"authUsers"` and `"authRoles"`!
- `UnsafeDeleteBucket(schema.Members)` (`server/storage/schema/membership.go:69 @ v3.7.3`) deleting prefix `"members"` will wipe out `"members_removed"`!
- `Hash(ignores)` (`server/storage/backend/backend.go:349-400 @ v3.7.3`) will misclassify keys across colliding buckets.

**Required fix:** Encode every flat LSM key with either a single-byte Fixed Bucket ID (`uint8` 1..11, or `uint16`) or a length-prefixed/null-terminated bucket name (`<uint8(len(bucketName))><bucketName><key>`), since none of etcd's bucket names contain `\x00` and all have length $\le 15$ bytes. Using a 1-byte length prefix `<len(bucketName)><bucketName><key>` guarantees prefix-free isolation (by Kraft's inequality, fixed-length-prefixed strings are an instantaneous prefix code) while keeping human-readable bucket names in raw SSTable dumps.

#### Trap B: The `(n % 512) == 32` Client Snapshot Checksum Invariant
When a client calls `etcdctl snapshot save` (or uses `go.etcd.io/etcd/client/v3/snapshot.SaveWithVersion`), the server's `Maintenance.Snapshot` RPC (`server/etcdserver/api/v3rpc/maintenance.go:135-220 @ v3.7.3`) streams `snap.WriteTo(w)` followed by a 32-byte `sha256` checksum. On the receiving side, `client/v3/snapshot/v3_snapshot.go:35-39, 102 @ v3.7.3` checks:
```go
func hasChecksum(n int64) bool {
	// 512 is chosen because it's a minimum disk sector size and
	// boltdb uses page size (commonly 4KB) larger than 512 bytes.
	return (n % 512) == sha256.Size
}
```
If `(fileSize % 512) != 32`, **every existing `clientv3` and `etcdutl` binary rejects the snapshot stream with `fmt.Errorf("sha256 checksum not found")` (`client/v3/snapshot/v3_snapshot.go:103 @ v3.7.3`)!**
- Why did this work for `bbolt`? Because `bbolt` files are always an integer multiple of the OS page size (`4096` bytes, which is $8 \times 512$).
- Why will this break an LSM backend? If `Snapshot.WriteTo(w)` streams a `.tar` archive (whose block size is 512 bytes—which *does* satisfy `size % 512 == 0`!) it works by coincidence, **whereas** if `Snapshot.WriteTo(w)` streams a `.tar.gz` / `.tar.zst` archive or a custom SSTable bundle whose byte length is not a multiple of 512 bytes, `clientv3` snapshot saves will fail immediately.
- **Required fix:** Whatever archive format `Snapshot.WriteTo(w)` emits must be padded to an exact multiple of `4096` bytes (hence `0 mod 512`) before returning, so that when `Maintenance.Snapshot` appends the 32-byte SHA-256 digest, total size satisfies `(n % 512) == 32`.

---

## 3. Deep Technical Comparison: `bbolt` vs. `grocksdb` vs. `pebble`

### 3.1 Architectural Summary Matrix

| Dimension | `bbolt` (`v1.5.0`, current) | `grocksdb` (`v1.11.1` / RocksDB C++) | `pebble` (`v2.1.7`, CockroachDB) |
|---|---|---|---|
| **Data Structure** | Single-file Copy-on-Write (CoW) B+tree (`4 KiB` pages, `mmap`-backed) | Multi-file Log-Structured Merge-tree (LSM) in C++ via `cgo` | Multi-file Log-Structured Merge-tree (LSM) in pure Go |
| **Write Amplification** | Low for sequential `UnsafeSeqPut`; **Very High** ($10\times\text{–}50\times$) for scattered MVCC compaction deletes & CoW branch updates | Low–Moderate ($5\times\text{–}15\times$ leveled compaction; tunable via Universal/FIFO or BlobDB) | Low–Moderate ($5\times\text{–}15\times$ leveled compaction; supports virtual SSTables & value separation) |
| **Space Reclamation** | **Requires stop-the-world `Defrag()`** (`backend.go:476`); freelist pages never shrink file | **Continuous background compaction**; no stop-the-world defrag needed | **Continuous background compaction**; no stop-the-world defrag needed |
| **Delete Tombstone Handling** | Synchronous `fdatasync` commit on **every** `UnsafeDelete` (`batch_tx.go:335`); $O(\log N)$ B+tree page CoW per key | $O(1)$ MemTable tombstone (`Delete`) or range tombstone (`DeleteRange`) | $O(1)$ MemTable/IndexedBatch tombstone (`Delete`) or range tombstone (`DeleteRange`) |
| **Read-Your-Own-Writes Buffer** | Custom `txWriteBuffer` / `txReadBuffer` (`tx_buffer.go`, lacks delete tombstones) | `grocksdb.WriteBatchWithIndex` (via `cgo` per call) | **`pebble.IndexedBatch`** (pure Go, supports `Get`/`NewIter` over uncommitted puts *and* deletes) |
| **Point Read Cost (`rangeKeys`)** | **Fastest when in RAM:** direct `mmap` pointer arithmetic ($0$ syscalls, $0$ allocations, $0$ `cgo`) | **Slowest for small values:** `cgo` boundary crossing + C-to-Go slice copy + bloom/block-cache lookup per revision | **Fast pure-Go:** in-process bloom filter + block cache lookup; zero `cgo` overhead |
| **Range Compaction (`Compact`)** | Must iterate and call `UnsafeDelete` per key (`1000` keys / `10ms` batch) | Supports `DeleteRange([startRev, endRev))` in $O(1)$ when no retained keys exist in range | Supports `DeleteRange([startRev, endRev))` in $O(1)$ when no retained keys exist in range |
| **Snapshots (`Snapshot()`)** | Holds a read transaction (`*bbolt.Tx`) pinning CoW pages; streams single file via `WriteTo` | `grocksdb.Checkpoint` creates hard links of SSTables in `<1ms`; requires archiving directory for `WriteTo` | `pebble.DB.Checkpoint` creates hard links of SSTables in `<1ms`; requires archiving directory for `WriteTo` |
| **Memory & GC Profile** | Low Go heap usage (`mmap` managed by OS kernel page cache); vulnerable to major page-fault stalls | Off-heap C++ Block Cache & MemTables (zero Go GC scan overhead), **but manual `.Destroy()` lifecycle**; leak-prone | Go-managed or `mmap`/off-heap block cache (`pebble` uses manual arena allocation / `MakeArenas` to minimize Go GC scanning) |
| **Build & Deployment** | Pure Go (`CGO_ENABLED=0`) | **Requires `CGO_ENABLED=1`**, C++17 compiler, `librocksdb`, `libzstd`/`liblz4`/`libsnappy`; breaks static Go builds | **Pure Go** (`CGO_ENABLED=0` compatible or minimal OS syscalls); drop-in `go.mod` dependency |

---

### 3.2 Write Path & The Double-WAL / `consistent_index` Durability Contract

A critical theoretical question when placing an LSM engine (`pebble` or `grocksdb`) behind etcd's Raft engine is: **Do we pay for two WALs (the Raft WAL + the LSM WAL), or can we disable the LSM WAL?**

Let us trace etcd's crash-recovery contract in `server/etcdserver/bootstrap.go:451-461 @ v3.7.3` and `server/verify/verify.go:111-131 @ v3.7.3`:
1. Every Raft entry is synced to the Raft WAL (`server/storage/wal/wal.go:881 @ v3.7.3`) before being applied.
2. As entries are applied to `backend`, `consistentIndex` tracks the highest applied Raft `(Index, Term)` in memory (`server/etcdserver/cindex/cindex.go:53-147 @ v3.7.3`).
3. Just before `backend` commits a batch (`batchTx.Commit()`), it invokes `hooks.OnPreCommitUnsafe(tx)` (`server/storage/hooks.go:45 @ v3.7.3`), which writes the current `consistent_index` and `term` into the `meta` bucket **inside the same atomic backend transaction** as the MVCC key-value mutations.
4. On restart after a crash (`bootstrap.go:456 @ v3.7.3`), etcd reads `consistent_index` from the `meta` bucket (`be.ConsistentIndex()`) and tells the applier to **skip all Raft WAL entries with ` entry.Index <= consistent_index`** (`uber_applier.go:120-128 @ v3.7.3`), replaying only `entry.Index > consistent_index` from the Raft WAL!
5. Furthermore, when etcd creates a Raft snapshot (`server/etcdserver/server.go:2092-2145 @ v3.7.3`), it calls `s.kv.Commit()` (which calls `backend.ForceCommit()`) **before** calling `s.r.storage.Release(snap)` / `w.ReleaseLockTo(compactIndex)` to truncate old Raft WAL segments.

**What happens if you set `DisableWAL: true` in `pebble` or `grocksdb`?**
- If you disable the LSM's internal WAL (`pebble.NoSync` with `DisableWAL`), mutations sit in the LSM MemTable. If the MemTable fills up (e.g., 64 MiB) and flushes to an L0 SSTable in the background *mid-batch* or *out of step with `OnPreCommitUnsafe`*, or if a crash occurs when L0 SSTables have flushed partial batch state without the matching `consistent_index`, **the database violates atomic batch recovery (`consistent_index` no longer matches the exact prefix of applied Raft entries)**, causing silent data corruption or a fatal panic in `server/verify/verify.go:118 @ v3.7.3` (`consistent index > HardState.Commit` or missing MVCC keys)!
- **Correct LSM Durability Configuration (High confidence):**
  - **Keep the LSM WAL enabled**, and commit each `batchTx` as a single atomic `pebble.Batch` / `grocksdb.WriteBatch` (which includes the `OnPreCommitUnsafe` write to `meta/consistent_index`).
  - For periodic 100ms commits (`commit(false)`), use **`pebble.NoSync`** (`WriteOptions{Sync: false}`). Why is `NoSync` completely safe here?
    1. Because `pebble.Batch.Commit(pebble.NoSync)` writes the entire atomic batch record to the OS buffer cache in a single `write()` syscall. If the etcd process crashes (SIGKILL/panic), the OS kernel flushes dirty page cache buffers, preserving the atomic batch.
    2. If the **entire machine loses power** before the OS flushes the LSM WAL page cache, the LSM database recovers to an earlier atomic `pebble.Batch` whose `consistent_index` is strictly smaller (`consistent_index_old < consistent_index_latest`), and **etcd's Raft WAL (which *was* `fdatasync`'d before apply!) deterministically replays the missing entries `(consistent_index_old, HardState.Commit]` on boot!**
  - For explicit `ForceCommit()` calls during Raft snapshotting (`server.go:2105 @ v3.7.3`) and clean shutdown (`Close()`), use **`pebble.Sync`** (`WriteOptions{Sync: true}`) (or `db.Flush()`) so that all state up to `snapshot.Metadata.Index` is guaranteed durable on disk before Raft WAL segments prior to `compactIndex` are released/deleted!

This design gives you **the best of both worlds**:
- **Zero `fdatasync` calls in the storage backend during normal 100ms batch commits** (eliminating `bbolt`'s 2x `fdatasync` and random 4 KiB page writes every 100ms!).
- **Strict atomic `(MVCC state, consistent_index)` crash recovery** guaranteed by `pebble.Batch` / `grocksdb.WriteBatch` framing plus Raft WAL replay.

---

### 3.3 Fixing `txBuffer` and `pendingDeleteOperations` with `pebble.IndexedBatch`

In `v3.7.3`, etcd maintains its own custom read-your-own-writes buffer (`txWriteBuffer` and `txReadBuffer` in `server/storage/backend/tx_buffer.go`) because `bbolt` does not allow concurrent read transactions (`readTx`) to read uncommitted writes inside the active write transaction (`batchTx`). As shown in §1.2 (Mechanism 1), `txBuffer`'s lack of deletion tombstones forced PR [#17119](https://github.com/etcd-io/etcd/pull/17119) to add `if t.pendingDeleteOperations > 0 { t.commit(false) }`, turning every `LeaseRevoke` and compaction batch into a synchronous `bbolt` commit.

With **`pebble`**:
- `db.NewIndexedBatch()` creates an in-memory indexed write batch (backed by an internal skiplist over the batch buffer) that implements **both** `pebble.Writer` (`Set`, `Delete`, `DeleteRange`) and `pebble.Reader` (`Get`, `NewIter`).
- When `UnsafeDelete(bucket, key)` is called on `IndexedBatch`, Pebble inserts an in-memory deletion tombstone into the `IndexedBatch`'s skiplist. Any subsequent `Get` or `NewIter` on that `IndexedBatch` automatically masks the deleted key from the underlying LSM levels **without needing to flush or commit the batch to disk**!
- Consequently, `t.pendingDeleteOperations > 0` **no longer needs to force a synchronous commit in `Unlock()`**. Every `LeaseRevoke` and `UnsafeDelete` simply appends a tombstone to the active batch in $<1\ \mu\text{s}$ and commits asynchronously on the normal 100ms `batchInterval`!

*(Note: `grocksdb` offers `NewWriteBatchWithIndex()`, which provides similar semantics in C++, but every read or write against `WriteBatchWithIndex` crosses the `cgo` boundary.)*

---

### 3.4 Read Path: The $N$-Point-Lookup Amplification in `mvcc.rangeKeys`

While LSM engines dominate `bbolt` on write-heavy and compaction-heavy workloads, we must analyze the read path—specifically `storeTxnRead.rangeKeys` in `server/storage/mvcc/kvstore_txn.go:118-155 @ v3.7.3`:

```go
// server/storage/mvcc/kvstore_txn.go:118-146 @ v3.7.3
revs, total := tr.s.kvindex.Revisions(key, end, int64(rev), int(ro.Limit))
// ...
limit := len(revs)
kvs := make([]mvccpb.KeyValue, limit)
revBytes := newRevBytes()
for i, rev := range revs {
	revToBytes(rev, revBytes)
	_, vs := tr.tx.UnsafeRange(schema.Key, revBytes, nil, 0)
	if len(vs) != 1 {
		tr.s.lg.Fatal("range failed to find revision pair", ...)
	}
	if err := kvs[i].Unmarshal(vs[0]); err != nil {
		tr.s.lg.Fatal("failed to unmarshal mvccpb.KeyValue", ...)
	}
}
```

Look closely at what happens when a client (e.g., `kube-apiserver` listing 10,000 pods or your distributed system listing 5,000 state keys) executes a `Range` query over $N$ keys:
1. `tr.s.kvindex.Revisions(...)` scans the in-memory `treeIndex` (ordered by **user key**) and returns $N$ `Revision` structs. Because the user keys were updated at different times, **the $N$ returned `Revision` values are in arbitrary (non-monotonic) order!**
2. The loop then executes **$N$ individual point lookups** (`UnsafeRange(schema.Key, revBytes, nil, 0)` with `endKey == nil`, which is a point lookup for `revBytes`) against the storage backend!

Let us compare the cost of those $N$ point lookups across the three engines:
- **`bbolt` (when dataset fits in RAM):** Each lookup traverses 3–4 B+tree pages in the `mmap`'d address space via pure Go pointer arithmetic and returns a slice pointing directly into the `mmap` page (`zero copy, zero syscall, zero cgo`).
- **`grocksdb`:** For a range scan of $N = 10,000$ keys:
  - If implemented via `UnsafeRange` using a RocksDB iterator (`NewIterator` + `Seek` + `Valid` + `Key` + `Value` + `Close`), that is **$50,000\text{–}60,000$ `cgo` calls per 10k-key range query**! Even if optimized to call `db.GetPinned(readOpts, key)` when `len(endKey) == 0`, that is still $20,000$ `cgo` calls (`GetPinned` + `Destroy`) plus C-to-Go byte copying. At ~60–100 ns per `cgo` round-trip plus RocksDB block-cache mutex/hash lookup, `grocksdb` adds multiple milliseconds of pure CPU overhead to large `Range` queries (High confidence).
- **`pebble`:**
  - When `len(endKey) == 0`, `UnsafeRange` should call **`reader.Get(prefixedKey)`** (returning `(val []byte, closer io.Closer, err error)`) rather than constructing a full `pebble.Iterator`!
  - Furthermore, because `schema.Key` lookups in `rangeKeys` are **always exact 17-byte revision point lookups**, configuring `pebble` with a **10-bits-per-key table bloom filter** (`bloom.FilterPolicy(10)`) ensures that point lookups skip SSTables that do not contain `revBytes` in $O(1)$ without disk or block reads, and configuring an appropriately sized in-memory **Block Cache** (e.g., 512 MiB–2 GiB) serves hot SSTable data blocks in pure Go without `cgo`.
  - **Optional MVCC read optimization enabled by LSM (`Moderate confidence`):** Because `rangeKeys` knows all $N$ `revs` upfront before the loop (`kvstore_txn.go:118`), sorting the index permutation of `revs` in ascending `Revision` order before calling `tr.tx.UnsafeRange` turns $N$ random SSTable lookups into a monotonic forward scan across SSTable blocks (maximizing block-cache locality in `pebble` and `bbolt` alike), or allows using `BatchGet` / a single forward-seeking `pebble.Iterator` (`Iter.SeekGE`) across all $N$ revisions!

---

### 3.5 Startup Restoration (`mvcc.restore`) and Hash Verification

On server startup or snapshot recovery, `store.restore()` (`server/storage/mvcc/kvstore.go:322-432 @ v3.7.3`) rebuilds the in-memory `treeIndex` by scanning **every live revision** in `schema.Key` in chunks of `restoreChunkKeys = 10,000` (`UnsafeRange(schema.Key, min, max, 10000)`).
- In an LSM engine, a full sequential forward scan over `schema.Key` in ascending `Revision` order is the optimal LSM access pattern (sequential merge iteration over SSTable blocks).
- Similarly, `UnsafeHash` / `HashByRev` (`server/storage/backend/backend.go:349-400`, `server/storage/mvcc/hash.go:53-145 @ v3.7.3`) performs a sequential forward scan over all buckets (or over `schema.Key` up to `compactRevision`..`hashRevision`), feeding `(bucketName, key, value)` into `crc32.New(crc32.MakeTable(crc32.Castagnoli))`. Because `crc32` hashes the logical `(bucketName, key, value)` bytes—**not** physical B+tree pages—**a cluster with a `pebble` backend computes the exact same `Hash` and `HashByRev` digests as a `bbolt` backend** (Verified in `backend.go:381-395` and `mvcc/hash.go:116-135 @ v3.7.3`!).
  - *Implication:* You can even run a mixed rolling test where `HashKV` (`HashByRev`) is cross-verified between a standard `bbolt` node and a `pebble` node at the same revision!

---

### 3.6 MVCC Compaction & Eliminating `Defrag()`

In `server/storage/mvcc/kvstore_compaction.go:35-99 @ v3.7.3`, `scheduleCompaction(compactMainRev int64, prevCompactRev int64)` scans `schema.Key` from `0` to `compactMainRev + 1`, checks `s.kvindex.keep(compactMainRev, keep)` to find Which revisions are still the latest generation of a live key, and calls `tx.UnsafeDelete(schema.Key, key)` on every superseded revision.

With `pebble` (or `grocksdb`), compaction improves in three ways:
1. **Batching deletes without synchronous `fdatasync`:** Deleting 1,000 keys in `scheduleCompaction` writes 1,000 tiny 17-byte tombstones into the LSM batch (~20 KiB total) and commits with `pebble.NoSync` in `<100 µs`, instead of rewriting hundreds of scattered 4 KiB B+tree pages and calling `fdatasync`.
2. **Range Tombstones (`DeleteRange`) for contiguous dead revision spans:** Between any two adjacent retained revisions $r_i < r_{i+1}$ in `keep` within `(prevCompactRev, compactMainRev]`, **every single revision in the open interval $(r_i, r_{i+1})$ is dead**. Instead of issuing thousands of individual `Delete(rev)` calls or scanning from `0` on every compaction (`kvstore_compaction.go:42` always starts from revision `0`!), an LSM backend can either start scanning from `prevCompactRev` or issue `DeleteRange` over dead revision spans.
3. **Zero `Defrag()` downtime:** As LSM background compaction merges L0 $\to$ L1 $\to \dots \to$ L$k$, any revision key covered by a point or range tombstone at the bottommost level is physically dropped from the newly written SSTable, and the old SSTable file is unlinked from disk. `backend.Defrag()` becomes a lightweight manual `db.Compact(start, end, true)` (which runs concurrently with reads and writes without holding `batchTx.Lock()`!).
4. **`Size()` and `SizeInUse()` (`server/storage/quota.go:92-125 @ v3.7.3`):** etcd's `backendQuota` checks `s.b.SizeInUse()` on every write (`quota.go:97`). In `bbolt`, `Size()` is total file size and `SizeInUse()` is `Size() - freePages * pageSize`. In `pebble`, `db.Metrics().DiskSpaceUsage()` (or `Total().Size` across LSM levels) reports the exact live SSTable + WAL bytes on disk in $O(1)$ directly from atomic metrics.

---

### 3.7 Snapshots (`backend.Snapshot`) & Multi-File Directory Management

In `bbolt`, `backend.Snapshot()` (`server/storage/backend/backend.go:336-347 @ v3.7.3`) opens a read-only `*bbolt.Tx` and implements `WriteTo(w io.Writer)` via `s.tx.WriteTo(w)`. And `OpenSnapshotBackend` (`server/storage/backend.go:62-69 @ v3.7.3`) simply calls `os.Rename(snapPath, cfg.BackendPath())`.

In `pebble` or `grocksdb`, the database at `cfg.BackendPath()` (`member/snap/db`) is a **directory** of immutable `.sst` files, `MANIFEST-*`, `CURRENT`, `OPTIONS-*`, and `.log` WAL files. How do we implement `backend.Snapshot` and `OpenSnapshotBackend` cleanly?
1. **`backend.Snapshot()`:**
   - First call `ForceCommit()` (or flush the active batch) so `consistent_index` and all committed batches are in the LSM DB.
   - Create a temporary checkpoint directory `member/snap/checkpoint-<timestamp>` using **`pebble.DB.Checkpoint(dir, pebble.WithFlushedWAL())`** (or `grocksdb.Checkpoint.CreateCheckpoint(dir, 0)`).
   - Because SSTables are immutable, `Checkpoint` creates **POSIX hard links (`link(2)`)** to the active `.sst` files in `<1ms` with **zero data copying and zero extra disk space**!
2. **`Snapshot.WriteTo(w io.Writer)`:**
   - Stream the files in the checkpoint directory as a deterministic, uncompressed **POSIX `archive/tar` stream** to `w`, **padded at the end with zero-bytes to a 4,096-byte boundary** (`0 mod 512`).
   - Why `archive/tar`?
     - Standard library (`archive/tar` in Go stdlib, zero external dependencies).
     - `tar` natively pads all blocks and end-of-archive markers to **512-byte multiples**, automatically satisfying the `client/v3` `hasChecksum(n) => (n % 512) == 32` invariant (§2.1 Trap B)!
3. **`Snapshot.Close()`:**
   - `os.RemoveAll(checkpointDir)` unlinks the hard-linked directory.
4. **`OpenSnapshotBackend` (`server/storage/backend.go:62 @ v3.7.3`):**
   - When a follower receives a Raft snapshot (`%016x.snap.db` saved by `snap.SaveDBFrom` in `server/etcdserver/api/snap/db.go:75`), `OpenSnapshotBackend` detects that `snapPath` is a `.tar` archive (or checks magic bytes), extracts it into a staging directory `member/snap/db.tmp`, syncs the directory, removes any old `member/snap/db`, and atomically renames `db.tmp` $\to$ `member/snap/db` before opening `pebble.Open("member/snap/db", opts)`.

---

### 3.8 Head-to-Head Verdict: `pebble` vs. `grocksdb`

**Verdict: Choose `pebble` (`v2.1.7`), not `grocksdb` (`v1.11.1`) (High confidence).**

1. **The `cgo` Tax on etcd's Specific Architecture:** Unlike systems that pass coarse-grained SQL queries across `cgo`, etcd's `mvcc` layer sits *above* `backend.Backend` in Go and calls `UnsafeRange` in a tight Go loop once per returned revision (`kvstore_txn.go:135 @ v3.7.3`) and once per chunk during startup `restore()` (`kvstore.go:360 @ v3.7.3`). Every `cgo` call in Go incurs ~60–100 ns of scheduler/stack switching, prevents Go compiler inlining and escape analysis, and requires copying or pinning C memory (`rocksdb_get_pinned` / `rocksdb_iter_*`). `pebble` executes entirely in Go with zero `cgo` overhead.
2. **`pebble.IndexedBatch` is a Tailor-Made Fit for `batchTxBuffered`:** `pebble.IndexedBatch` natively provides a combined `Reader`/`Writer` in pure Go that handles uncommitted puts *and* delete tombstones, eliminating `txBuffer` (`tx_buffer.go`) and fixing the PR [#17119](https://github.com/etcd-io/etcd/pull/17119) `pendingDeleteOperations > 0` synchronous-commit bottleneck simultaneously.
3. **Operational, Build, and Debugging Risk:** etcd is deployed as a static pure-Go binary (`CGO_ENABLED=0`). Introducing `grocksdb` requires linking C++ `librocksdb` (plus `zstd`, `lz4`, `snappy`), complicates multi-architecture container builds (`amd64`/`arm64`), hides C++ memory allocations from Go's `pprof` heap profiler and `GOMEMLIMIT`, and turns memory-management bugs (missing `.Destroy()` on `grocksdb.Slice` / `ReadOptions` / `WriteBatch` / `Iterator`) into silent off-heap memory leaks or unrecoverable `SIGSEGV` crashes.
4. **Condition under which `grocksdb` would outperform `pebble` (Low–Moderate confidence):** Only if the etcd node is severely constrained by Go Garbage Collection pauses at very high heap sizes (`>32 GiB` in-memory `treeIndex`) and needs SSTable block cache and background compaction threads to live strictly outside the Go runtime (`cgo` off-heap threads). Even there, `pebble` already uses manual off-heap/mmap-backed block cache arenas to minimize Go GC scan overhead.

---

## 4. Effort Estimation & Implementation Blueprint (`pebble` on etcd `v3.7.3`)

Replacing `bbolt` with `pebble` while keeping all `server/storage/mvcc`, Raft, and gRPC layers intact requires modifying or adding approximately **1,800–2,600 lines of Go code** across ~15 files.

| Work Package | Target Files (`@ v3.7.3`) | Key Engineering Tasks | Estimated LoC | Estimated Effort |
|---|---|---|---|---|
| **WP1: Key Encoding & Bucket Mapping** | `server/storage/backend/pebble_key.go` (new) | Implement length-prefixed bucket encoding (`<uint8(len(bucket.Name()))><bucket.Name()><key>`) and upper-bound prefix computation (`prefixEnd`) to prevent the `"auth"`/`"authUsers"` and `"members"`/`"members_removed"` collision bug (§2.1 Trap A). | ~150 LoC | 0.5 wk |
| **WP2: `pebble` Backend, `BatchTx`, and `ReadTx`** | `server/storage/backend/pebble_backend.go`, `pebble_batch_tx.go`, `pebble_read_tx.go` (new) | Implement `Backend`, `BatchTxBuffered`, and `ReadTx` using `*pebble.DB`, `*pebble.IndexedBatch`, and `*pebble.Snapshot`. Use `pebble.NoSync` for 100ms periodic commits and `pebble.Sync` for `ForceCommit()` / `Close()`. Use `reader.Get()` when `endKey == nil` in `UnsafeRange` and `pebble.Iterator` for range scans. | ~900–1,200 LoC | 2.5–3.5 wks |
| **WP3: Checkpoints & `.tar` Snapshot Streaming** | `server/storage/backend/pebble_snapshot.go` (new), `server/storage/backend.go:53-113` | Implement `Snapshot()` via `db.Checkpoint()`, stream via `archive/tar` in `WriteTo(w)` (ensuring `size % 512 == 0` for `client/v3` `hasChecksum`), and unpack `.tar` snapshots into directory in `OpenSnapshotBackend` / `RecoverSnapshotBackend`. | ~350–450 LoC | 1.5–2 wks |
| **WP4: Compaction & Range-Delete Optimization** | `server/storage/mvcc/kvstore_compaction.go:35-99`, `server/storage/backend/backend.go` | Remove redundant `ForceCommit()` per 1,000-key compaction batch; optionally add `UnsafeDeleteRange` or start compaction scan from `prevCompactRev`. Map `Defrag()` to `db.Compact(start, end, true)`. | ~150–250 LoC | 1 wk |
| **WP5: CLI Tooling (`etcdutl`, `etcd-dump-db`) & Verification** | `etcdutl/snapshot/v3_snapshot.go`, `tools/etcd-dump-db/backend.go`, `server/storage/schema/version.go` | Update `etcdutl snapshot restore/status` and `etcd-dump-db` to open the backend via `backend.Backend` (and unpack `.tar` snapshots) instead of calling `bolt.Open` directly. | ~250–450 LoC | 1–1.5 wks |
| **WP6: Robustness, Crash-Recovery & Linearizability Testing** | `server/storage/backend/*_test.go`, `tests/robustness/`, `tests/integration/` | Run etcd's unit tests, integration tests, and Porcupine-based linearizability/failpoint robustness suite (`tests/robustness/`) under crash/restart and snapshot transfer scenarios. | ~200 LoC | 1.5–2 wks |
| **Total** | | | **~2,000–2,700 LoC** | **8–10.5 engineer-weeks** |

*(Note: A minimal proof-of-concept covering only WP1 + WP2 + basic WP3 to run benchmarks on target hardware can be built in **~1.5–2 engineer-weeks** [~800 LoC], before investing in `etcdutl` tooling and full robustness testing.)*

---

## 5. Evaluation of the Fork Strategy & Alternative Mitigations

You proposed that if the analysis and experiments show meaningful improvement, your implementation route would be to **fork `github.com/etcd-io/etcd` and maintain the storage engine replacement on top of the fork**.

### 5.1 Assessment of the Fork Plan

1. **Upstream Will Not Merge an Alternative Storage Engine in the Near Term (High confidence):**
   - Upstream maintainers closed both the Badger prototype ([etcd#10311](https://github.com/etcd-io/etcd/pull/10311)) and the Pluggable Backend meta-issue ([etcd#10321](https://github.com/etcd-io/etcd/issues/10321)), concluding that maintaining multiple storage engines multiplied testing and corruption-postmortem surface area (especially after the v3.5 data inconsistency postmortem in `Documentation/postmortems/v3.5-data-inconsistency.md @ v3.7.3`). Therefore, if you replace `bbolt`, **maintaining a fork is indeed the only realistic delivery vehicle**.
2. **How to Structure the Fork to Minimize Rebase Pain Across Upstream Releases:**
   - **Do NOT delete `bbolt` or edit `server/storage/backend/backend.go` in place.** Keep the existing `bbolt` implementation untouched and introduce `pebble` behind the existing `backend.Backend` interface (`server/storage/backend/pebble_*.go`), toggled by an environment variable or flag (e.g., `--experimental-backend-engine=pebble|bbolt`, or auto-detected by checking whether `member/snap/db` is a directory or a regular file).
   - Why? In `v3.7.3`, `server/storage/backend/` is only ~1,700 lines across 5 files and changes rarely between minor releases. Keeping `bbolt` intact and isolating `pebble` into separate `pebble_*.go` files keeps your touch surface in existing etcd files under **~100 lines** (`server/storage/backend.go`, `server/storage/backend/backend.go`, `etcdutl/snapshot/v3_snapshot.go`), making rebasing from `v3.7.x` $\to$ `v3.8.x` mechanical.
3. **Operational Risks of Running a Storage-Engine Fork in Production:**
   - **Physical snapshot incompatibility across `bbolt` and `pebble` members:** Because Raft snapshots in etcd (`server/etcdserver/snapshot_merge.go:39-84 @ v3.7.3`) stream the raw physical backend snapshot (`backend.Snapshot.WriteTo`) rather than logical KV pairs, a `bbolt` leader cannot send a physical Raft snapshot (`MsgSnap`) directly to a `pebble` follower unless `OpenSnapshotBackend` includes a one-way `bbolt`-file-to-`pebble`-directory converter!
     - **Design recommendation for zero-downtime migration (High confidence):** Write a ~120-line converter inside `OpenSnapshotBackend` and `newBackend`: if `member/snap/db` (or the incoming `%016x.snap.db`) starts with the `bbolt` magic number (`0xED0CDAED` at offset `pageSize + 0`), open it read-only with `bbolt`, iterate all 10 buckets via `ForEach`, write every key-value pair into a new `pebble` directory, and atomically swap it! This single feature gives you **seamless live rolling upgrades from upstream `bbolt` etcd to your `pebble` fork** and **seamless migration back** if needed!
   - **Loss of upstream `bbolt` emergency surgery tools:** Operational playbooks that rely on `bbolt surgery` or raw `bbolt` page inspection will not work on `pebble` SSTables (replace with `pebble` CLI tools / updated `etcdutl`).

### 5.2 Low-Effort Surgical Fixes to Test *Before* Replacing `bbolt`

Before committing 8–10 engineer-weeks to a `pebble` backend replacement, static code analysis of `v3.7.3` reveals **three surgical changes (< 150 LoC total)** that directly attack `bbolt`'s worst write stalls:

1. **Fix `txBuffer` to Support Deletion Tombstones (~80 LoC in `server/storage/backend/tx_buffer.go` and `batch_tx.go`):**
   - If your workload uses leases (`LeaseGrant`/`LeaseRevoke`/TTL expiration) or frequent MVCC compaction, PR [#17119](https://github.com/etcd-io/etcd/pull/17119)'s `if t.pendingDeleteOperations > 0 { t.commit(false) }` (`batch_tx.go:335 @ v3.7.3`) is forcing a synchronous `bbolt` `fdatasync` on **every single `LeaseRevoke` and every compaction batch**.
   - Adding a `deleted bool` tombstone flag to `kv` in `tx_buffer.go` (and filtering tombstones in `bucketBuffer.Range`) eliminates the need to synchronously commit `bbolt` on `UnsafeDelete` during `LeaseRevoke`, restoring 100ms batching with an ~80-line patch!
2. **Decouple MVCC Compaction from Double-Committing (`server/storage/mvcc/kvstore_compaction.go:85, 90 @ v3.7.3`):**
   - In `v3.7.3`, `scheduleCompaction` calls `tx.Unlock()` at line 85 (which synchronously commits `bbolt` because `pendingDeleteOperations > 0`) and then immediately calls `s.b.ForceCommit()` at line 90, while scanning from revision `0` on every compaction run (`line 42`) instead of starting from `prevCompactRev`. Starting the compaction scan at `prevCompactRev` (for keys compacted after `prevCompactRev`) and tuning `--auto-compaction-retention` / compaction batch size immediately reduces B+tree CoW churn.
3. **Hardware & Flag Tuning (`--wal-dir`, `--backend-batch-interval`, `--backend-batch-limit`):**
   - Placing `--wal-dir` on a dedicated physical NVMe namespace prevents `bbolt`'s periodic page-spill writes and `fdatasync`s from queueing behind Raft WAL `fdatasync`s in the disk controller queue.

---

## 6. Experimental Design for Your Target Hardware

To make a strictly data-driven decision on your team's target hardware/platform without guessing, run the following **five small, progressive experiments**. They require zero code changes for Experiments 1–3, an 80-line patch for Experiment 4, and a small standalone ~250-line Go micro-harness for Experiment 5.

### Experiment 1: Production/Staging Bottleneck Attribution via Prometheus Histograms (Zero Code Changes)

Before running synthetic benchmarks, capture the following Prometheus histograms from your existing etcd cluster during peak write load (all verified in `server/etcdserver/metrics.go` and `server/storage/backend/metrics.go @ v3.7.3`):

| Metric Name (`@ v3.7.3`) | Code Location | What It Measures |
|---|---|---|
| `etcd_disk_wal_fsync_duration_seconds` | `server/etcdserver/metrics.go:79` | Raft WAL `fdatasync` latency (`wal.go:881`). **Unaffected by replacing `bbolt`.** |
| `etcd_disk_backend_commit_duration_seconds` | `server/storage/backend/metrics.go:25` | Total `batchTxBuffered.unsafeCommit()` duration (`batch_tx.go:383`). **Eliminated/reduced by `pebble`.** |
| `etcd_debugging_disk_backend_commit_rebalance_duration_seconds` | `server/storage/backend/metrics.go:35` | Time spent rebalancing `bbolt` B+tree nodes during commit. |
| `etcd_debugging_disk_backend_commit_spill_duration_seconds` | `server/storage/backend/metrics.go:45` | Time spent splitting & allocating dirty 4 KiB `bbolt` B+tree pages. |
| `etcd_debugging_disk_backend_commit_write_duration_seconds` | `server/storage/backend/metrics.go:55` | Time spent writing dirty `bbolt` pages + freelist + `2x fdatasync` to disk. |
| `etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds` | `server/storage/mvcc/metrics.go:108` | Duration of each compaction batch holding `batchTx.Lock()`. |
| `etcd_debugging_mvcc_db_total_size_in_bytes` vs. `..._in_use_in_bytes` | `server/storage/mvcc/metrics.go:135, 143` | Total `bbolt` file size vs. live data size (the gap is `bbolt` freelist fragmentation). |

**Decision Rule from Experiment 1:**
- Calculate the **Backend Commit Lock Duty Cycle**:
  $$\text{Backend Duty Cycle} = \frac{\Delta \text{etcd\_disk\_backend\_commit\_duration\_seconds\_sum}}{\Delta t}$$
- Calculate the **Commit Frequency**:
  $$\text{Backend Commits/sec} = \frac{\Delta \text{etcd\_disk\_backend\_commit\_duration\_seconds\_count}}{\Delta t}$$
- **If $\text{Backend Commits/sec} \gg 10\text{/s}$** (the normal `100ms` rate) while write throughput is $< 100,000\text{ ops/s}$: you are hitting the **`pendingDeleteOperations > 0` synchronous commit bug** (`batch_tx.go:335`) from `LeaseRevoke` or compaction!
- **If $\text{Backend Duty Cycle} > 0.20$ (20% of wall-clock time spent inside `bbolt.Commit()`) or `p99(etcd_disk_backend_commit_duration_seconds) > 25ms`**: `bbolt` is blocking `applyEntries` for a major fraction of wall time, and replacing `bbolt` with `pebble` will directly improve write throughput and tail latency.
- **If $\text{Backend Duty Cycle} < 0.05$ (< 5%) and `p99(etcd_disk_wal_fsync_duration_seconds)` dominates request latency**: `bbolt` is **not** your bottleneck; replacing it will not solve your write performance problem.

---

### Experiment 2: Baseline `tools/benchmark` Scaling Curve (Small DB vs. Large Aged DB with Compaction)

Use etcd's built-in `tools/benchmark` (`~/github/etcd/tools/benchmark @ v3.7.3`) on a 3-node cluster (or 1-node to isolate storage from network RTT) on your target hardware across two database states:

1. **State A (Fresh / Small DB, ~200 MiB):**
   ```bash
   benchmark --endpoints=$ENDPOINTS --conns=100 --clients=500 \
     put --key-size=64 --val-size=1024 --total=200000 --key-space-size=100000
   ```
2. **State B (Aged / Large DB, ~6 GiB with active MVCC compaction):**
   Start etcd with `--quota-backend-bytes=8589934592 --auto-compaction-mode=revision --auto-compaction-retention=50000`, pre-populate 4 million updates across 500,000 keys (`~6 GiB`), and then run a mixed workload of `Put` + background compaction + `LeaseKeepAlive`/revocation:
   ```bash
   benchmark --endpoints=$ENDPOINTS --conns=100 --clients=500 \
     put --key-size=64 --val-size=1024 --total=2000000 --key-space-size=500000
   ```

**What to compare:** Measure throughput (ops/sec), `p50`/`p99`/`p99.9` latency, and disk IOPS/MB/s (`iostat -xz 1`) between State A and State B. In `bbolt`, State B typically suffers a **40%–80% throughput drop and $5\times\text{–}20\times$ p99 latency spike** once compaction starts dirtying scattered 4 KiB pages across a 6 GiB file.

---

### Experiment 3: Isolating Raft WAL `fdatasync` vs. `bbolt` Backend Cost (Tmpfs Split Test)

To measure the **exact theoretical upper bound** of what replacing `bbolt` can achieve on your hardware without writing a single line of code:
Run Experiment 2 (State B) under three storage placement configurations on the target machine:
1. **Baseline:** Both `--data-dir` (`member/snap/db`) and `--wal-dir` (`member/wal`) on the target disk.
2. **Simulated Zero-Cost Backend (`member/snap/db` on `tmpfs` RAM disk, `--wal-dir` on target disk):**
   Mount a `10 GiB` `tmpfs` at `/mnt/tmpfs-snap`, bind-mount it to `$DATA_DIR/member/snap`, and keep `$DATA_DIR/member/wal` on the real disk. This removes 100% of `bbolt`'s disk I/O (`write` and `fdatasync`) while keeping `bbolt`'s CPU/lock overhead and the real Raft WAL `fdatasync`.
3. **Simulated Zero-Cost WAL (`--wal-dir` on `tmpfs` RAM disk, `member/snap/db` on target disk):**
   Keep `member/snap/db` on the real disk and put `--wal-dir` on `tmpfs`.

**Decision Rule from Experiment 3 (High confidence):**
- Configuration 2 (`snap` on `tmpfs`, `wal` on real disk) gives you the **hard ceiling on throughput and latency improvement achievable by any backend storage engine replacement** (since no disk-backed LSM can do less disk I/O than `0` disk I/O on `tmpfs`, apart from eliminating `bbolt`'s in-memory B+tree rebalance CPU time).
- If Configuration 2 improves throughput over Configuration 1 by `< 15%`, **do not replace `bbolt`**—your bottleneck is in the Raft WAL or CPU/network path.
- If Configuration 2 improves throughput or p99 latency by `> 40%–300%`, **the backend disk I/O is confirmed as the primary bottleneck**, justifying Experiment 5.

---

### Experiment 4: Testing the 80-Line `txBuffer` Tombstone Fix (If `LeaseRevoke` / Compaction Is Present)

If Experiment 1 shows `etcd_disk_backend_commit_duration_seconds_count` firing much faster than 10 commits/sec:
- Patch `server/storage/backend/batch_tx.go:335 @ v3.7.3` in a test worktree to remove `|| t.pendingDeleteOperations > 0` (along with adding a tombstone marker to `tx_buffer.go` so deleted keys are masked in `txReadBuffer`).
- Re-run your workload. If removing synchronous commits on `UnsafeDelete` resolves your write throughput cliff, you can maintain a **< 100-line patch** instead of a 2,500-line storage engine replacement.

---

### Experiment 5: Standalone Storage-Engine Micro-Harness (`bbolt` vs. `pebble` vs. `grocksdb`)

Before wiring `pebble` into etcd's full codebase, run a standalone ~250-line Go benchmark binary on your target machine that replays etcd's **exact backend access pattern** against `bbolt v1.5.0`, `pebble v2.1.7`, and `grocksdb v1.11.1`:
1. **Phase 1 (Sequential Ingestion + 100ms Batching):**
   - Write 10,000,000 sequential 17-byte `revBytes` keys (`Main: 1..10,000,000, Sub: 0`) with 1 KiB protobuf payloads (`~10 GiB` dataset), committing batches every `100ms` or `10,000` keys (`bbolt.Tx.Commit` vs. `pebble.IndexedBatch.Commit(pebble.NoSync)` + `pebble.Sync` every 10,000 keys).
2. **Phase 2 (Concurrent Mixed Workload — The Real Test):**
   - **Writer Goroutine (`applyEntries` simulator):** Continuously appends new 17-byte `revBytes` keys + updates `meta/consistent_index`.
   - **Compactor Goroutine (`scheduleCompaction` simulator):** Every `10ms`, deletes 1,000 non-retained 17-byte `revBytes` keys from earlier in the revision history (e.g., 70% of revisions in an older window).
   - **Reader Goroutines (16 concurrent `rangeKeys` simulators):** Each reader takes a snapshot/read-tx and performs $N = 1,000$ point lookups of random live 17-byte `revBytes` keys (simulating `mvcc.rangeKeys` resolving 1,000 user keys from `treeIndex`).
   - **Startup Restore (`mvcc.restore` simulator):** Time a full sequential forward scan in chunks of `10,000` keys over all live revisions.

**Success Criteria for Proceeding to Full etcd Fork Implementation:**
Proceed with the `pebble` fork implementation (Section 4) if and only if:
1. Experiment 1 & Experiment 3 confirm that `bbolt` commit/compaction I/O accounts for `> 30%` of write stall time on target hardware; **and**
2. Experiment 4 (`txBuffer` tombstone fix) alone is insufficient because dataset size (`> 2–8 GiB`), compaction CoW write amplification, or `Defrag()` stalls remain; **and**
3. In Experiment 5 (Phase 2), `pebble` delivers `>= 2x` sustained write throughput or `>= 5x` lower `p99` write batch commit latency than `bbolt` while keeping 1,000-key point-lookup `rangeKeys` latency within `1.25x` of `bbolt`.

---

## 7. Open Questions

1. **Dataset Size and Compaction Profile:** What is the steady-state `bbolt` file size (`etcd_mvcc_db_total_size_in_bytes`), live data size (`etcd_debugging_mvcc_db_total_size_in_use_in_bytes`), and average value size in your team's deployment?
2. **Lease Churn vs. KV Churn:** What fraction of your write traffic consists of `Put`/`Txn` versus `LeaseGrant`/`LeaseRevoke` (`LeaseKeepAlive` expiration)?
3. **Read Access Pattern:** Does your application perform large multi-key `Range` queries (e.g., scanning thousands of keys per RPC, where the $N$-point-lookup amplification in `mvcc.rangeKeys` matters), or primarily point reads and watches?
