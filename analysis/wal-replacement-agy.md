# Replacing Both the Raft WAL and Storage Engine in etcd: End-to-End Persistence Redesign

- **Versions analyzed:**
  - `etcd`: `v3.7.3` (`0a173d0410a9`, released 2026-10-22)
  - `go.etcd.io/raft/v3` (declared in `server/go.mod:30 @ v3.7.3`): `v3.7.0`
  - `go.etcd.io/bbolt` (declared in `server/go.mod:25 @ v3.7.3`): `v1.5.0`
  - `github.com/cockroachdb/pebble` (including `github.com/cockroachdb/pebble/record`): `v2.1.7`
  - `github.com/linxGnu/grocksdb`: `v1.11.1`
  - Alternative WAL/log libraries evaluated: `github.com/lni/dragonboat/v4` (`tan` LogDB), `github.com/tidwall/wal`, `github.com/hashicorp/raft-wal`, `github.com/tikv/raft-engine`
- **Upstream references:**
  - [etcd#10321](https://github.com/etcd-io/etcd/issues/10321) — *Meta issue: Pluggable backend*
  - [etcd#22332](https://github.com/etcd-io/etcd/issues/22332) — *Increase supported storage size to match K8s scalability requirements* (includes naive Pebble MVCC prototype read/write numbers)
  - [etcd#17119](https://github.com/etcd-io/etcd/pull/17119) — *Commit transaction for each configuration change*
  - [etcd#16291](https://github.com/etcd-io/etcd/issues/16291), [etcd#18027](https://github.com/etcd-io/etcd/pull/18027), [etcd#22223](https://github.com/etcd-io/etcd/pull/22223) — *Support Raft `AsyncStorageWrites` (`RaftAsyncStorageWrites` alpha feature gate)*
  - [etcd-io/raft#8](https://github.com/etcd-io/raft/pull/8), [etcd-io/raft#300](https://github.com/etcd-io/raft/pull/300) — *`AsyncStorageWrites` (`MsgStorageAppend` / `MsgStorageApply`) design and implementation*
  - [etcd-io/raft#551](https://github.com/etcd-io/raft/issues/551), [etcd-io/raft#550](https://github.com/etcd-io/raft/pull/550) — *Open `AsyncStorageWrites` election-safety bug: `MsgVote` sent before `MsgStorageAppend` persists new `Term`/`Vote`*
  - [etcd-io/raft#543](https://github.com/etcd-io/raft/issues/543) — *Open `AsyncStorageWrites` ordering bug: `MsgStorageAppend` snapshot application racing queued `MsgStorageApply` entries*
  - [cockroachdb/cockroach#16624](https://github.com/cockroachdb/cockroach/issues/16624) — *Cross-file `ext4` (`data=ordered`) journal flush coupling between synced Raft log and unsynced state store*
  - Relevant `etcd` commits: upstream `main`-only commit `5d1a827c1` (2026-07-22, stops generating/loading legacy v2 `.snap` files); `2e2963af6` (2026-07-23 on `main`, backported to `v3.7.3` as `23b61e6d9`, updating `MinimalEtcdVersion` to scan WAL from `LatestSnapshotEntry`); `01a80f67b` (2026-10-17 on `main`, backported to `v3.7.3` as `e71275f78`, preserving `Ready` message order in `processMessages`)
- **Summary:**
  1. **Replacing only `bbolt` leaves half of etcd's write-path bottlenecks untouched (High confidence).** Even with an ideal state-machine backend, etcd `v3.7.3` is bottlenecked by: (a) a synchronous, non-pipelined Raft loop (`server/etcdserver/raft.go:185-336 @ v3.7.3`) that fails to enable `go.etcd.io/raft/v3`'s `AsyncStorageWrites` and blocks on an unbuffered `r.applyc <- ap` channel (`raft.go:146, 232`) *before* sending leader `MsgApp` network messages or starting WAL disk writes; (b) a legacy WAL implementation (`server/storage/wal/wal.go @ v3.7.3`) that executes **3–4 synchronous `fdatasync`/`fsync` calls plus file rename/re-open/re-lock** on every 64 MB segment rotation (`w.cut()`, `wal.go:785-867`), triggers `ext4`/`XFS` unwritten-extent journal commits on every `fdatasync` into `fallocate`d blocks (`file_pipeline.go:81`), and double-marshals/triple-copies every entry (`wal.go:974-983`, `encoder.go:66-140`, `pkg/ioutil/pagewriter.go:56-99`); (c) a synchronous 3-`fdatasync` checkpoint stall on the `applyAll` goroutine every `DefaultSnapshotCount = 10,000` entries (`server/etcdserver/server.go:1215-1228, 2083-2137 @ v3.7.3`); and (d) **double-WAL `fdatasync` amplification**, where every key-value mutation is synced twice if both the Raft log and backend sync their logs.
  2. **Do *not* store both Raft log entries and MVCC state inside a single unified LSM instance (High confidence).** While placing `raft_log/<index>` and MVCC buckets into one `pebble` or `grocksdb` database eliminates `server/storage/wal/`, it forces short-lived, append-only Raft log entries through MemTable flushes and multi-level SSTable compactions alongside long-lived MVCC data, doubling LSM compaction I/O and allowing LSM L0 compaction write stalls to block Raft `Save(HardState, Entries)`—triggering leader heartbeat timeouts and cluster elections. Both TiKV (which replaced its RocksDB `raftdb` instance with the BitCask-like `raft-engine`) and Dragonboat (which built `tan` to replace Pebble for Raft logs) proved that consensus logs require a dedicated append/truncate log structure.
  3. **Optimal Target Architecture — Co-designed Recycled Raft Log (`pebble/record.LogWriter` + `AsyncStorageWrites`) + Unsynced-Apply `pebble` State Machine with Background `LogData(Sync)` Checkpoint Barrier (High confidence):**
     - **Consensus & WAL layer:** Enable `Config.AsyncStorageWrites = true` (`go.etcd.io/raft/v3` `v3.7.0`, `server/go.mod:30 @ v3.7.3`) to decouple the Raft state machine goroutine from disk I/O via `LocalAppendThread` (`MsgStorageAppend`) and `LocalApplyThread` (`MsgStorageApply`), while adding explicit `raftNode` guards for open upstream bugs **[etcd-io/raft#551](https://github.com/etcd-io/raft/issues/551)** (holding back outgoing `MsgVote` until `MsgStorageAppend` completes) and **[etcd-io/raft#543](https://github.com/etcd-io/raft/issues/543)** (serializing snapshot swaps through `LocalApplyThread`'s FIFO queue). Replace `server/storage/wal` and `server/etcdserver/api/snap` (`Snapshotter`) with **`RaftLogEngine`** built on **`github.com/cockroachdb/pebble/record.LogWriter`** (`v2.1.7`) with **`WriteWALSyncOffsets`** (`pebble/record/record.go:87-100, 144`, `log_writer.go:512-515 @ v2.1.7`). This provides a built-in `SyncRecord` group-commit queue, **`logNum`-tagged pre-written WAL file recycling** (`ErrInvalidLogNum`) that eliminates both `w.cut()`'s 4-fsync rotation stall and `ext4`/`XFS` unwritten-extent journal commits, and 8-byte promised synced-offset chunk headers that fix etcd `v3.7.3`'s `isTornEntry` silent-truncation bug (`decoder.go:126-148, 170-203 @ v3.7.3`).
     - **Co-designed State Machine layer (`pebble v2.1.7`):** Because `RaftLogEngine` retains recycled WAL segments until Pebble's state is durable up to `safeRecycleIndex`, **Pebble never performs an `fdatasync` on the `applyEntries` or 100ms batch-commit path**. Our recommended default mode (**Mode C2**) runs Pebble with its recycled WAL enabled in **`pebble.NoSync`** mode—preserving 100% crash-test coverage (`pebble/metamorphic/options.go:852-853 @ v2.1.7`), atomic prefix recovery, and native `DeleteRange`—and advances `safeRecycleIndex` asynchronously via a single background **`db.LogData(nil, pebble.Sync)`** call (`pebble/db.go:736-745 @ v2.1.7`) without even requiring a 64 MiB MemTable flush. An optional WAL-less mode (**Mode C1: `DisableWAL: true`**) eliminates the buffered `write(2)` syscall as well, fenced on `db.Flush()` + immutable `db.Checkpoint()` rollover.
  4. **Read latency can be kept equal to `bbolt` on point reads and made *faster* than `bbolt` on multi-key `Range` reads via Revision-Order Permutation (`TrySeekUsingNext`) (High confidence).** Whereas a naive Pebble MVCC prototype ([etcd#22332](https://github.com/etcd-io/etcd/issues/22332)) regressed read throughput by 5–6.5× due to uncached point-iterator churn, our design combines: (a) advancing `mvcc.currentRev` after `IndexedBatch.Commit(pebble.NoSync)` so readers query Pebble's lock-free MemTable/Block Cache directly without `readTx.mu.Lock()` (`batch_tx.go:310, 356 @ v3.7.3`), (b) direct `db.Get()` + 10-bit Bloom filters + 1–4 GiB Block Cache for single-key lookups, (c) sorting the $N$ revisions (`revpairs`) returned by `treeIndex.Revisions()` in ascending `Revision` order and scanning them with a **single forward-seeking `pebble.Iterator`** in `storeTxnRead.rangeKeys` (`server/storage/mvcc/kvstore_txn.go:108-151 @ v3.7.3`), which automatically activates Pebble's internal `flags.EnableTrySeekUsingNext()` optimization (`pebble/iterator.go:308-311, 1331-1333 @ v2.1.7`), and (d) bounding `commitIndex - appliedIndex` lag so linearizable reads waiting on `r.server.ApplyWait(confirmedIndex)` (`server/etcdserver/read/read.go:133-138 @ v3.7.3`) never queue behind unbounded async apply batches.

---

## 1. Diagnosis: Why etcd `v3.7.3`'s Current WAL and Consensus Pipeline Bottleneck Write Throughput (**High confidence**)

In [storage-agy.md](storage-agy.md), we analyzed the right half of etcd's write path (`server/storage/backend` and `bbolt`). When we inspect the left half—`server/etcdserver/raft.go`, `server/storage/wal/`, `server/etcdserver/api/snap/`, and `server/storage/storage.go` at `v3.7.3` (`0a173d0410a9`)—we find **six structural bottlenecks** that cap write throughput and inflate tail latency even if `bbolt` were infinitely fast (**High confidence**).

### 1.1 Bottleneck 1: Synchronous, Non-Pipelined `raftNode` Loop & Unbuffered `applyc` Coupling (`server/etcdserver/raft.go:146, 185-336 @ v3.7.3` — **High confidence**)

Look at the exact order of operations inside `raftNode.start()` (`server/etcdserver/raft.go:185-336 @ v3.7.3`):

```go
// server/etcdserver/raft.go:185-332 @ v3.7.3
case rd := <-r.Ready():
    // ...
    ap := toApply{
        entries:       committedEntries,
        snapshot:      proto.Clone(rd.Snapshot).(*raftpb.Snapshot),
        notifyc:       notifyc,
        raftAdvancedC: raftAdvancedC,
    }
    updateCommittedIndex(&ap, rh)

    // [STALL POINT A]: r.applyc is UNBUFFERED (`make(chan toApply)` at line 146)!
    select {
    case r.applyc <- ap:
    case <-r.stopped:
        return
    }

    // Leader only sends MsgApp network packets AFTER `r.applyc <- ap` unblocks!
    if islead {
        r.transport.Send(r.processMessages(rd.Messages))
    }

    // [STALL POINT B]: Synchronous WAL write + fdatasync on the raftNode goroutine!
    if err := r.storage.Save(rd.HardState, rd.Entries); err != nil {
        r.lg.Fatal("failed to save Raft hard state and entries", zap.Error(err))
    }
    // ...
    r.raftStorage.Append(rd.Entries)

    if !islead {
        // [STALL POINT C]: Follower does not send MsgAppResp to leader until AFTER WAL fdatasync!
        msgs := r.processMessages(rd.Messages)
        notifyc <- struct{}{}
        // ...
        r.transport.Send(msgs)
    } else {
        notifyc <- struct{}{}
    }
    r.Advance()
```

And now compare this with `EtcdServer.applyAll()` on the main server goroutine (`server/etcdserver/server.go:972-993 @ v3.7.3`):

```go
func (s *EtcdServer) applyAll(ep *etcdProgress, apply *toApply) {
    s.applySnapshot(ep, apply)
    s.applyEntries(ep, apply)
    // ...
    s.applyWait.Trigger(ep.appliedi)

    // [STALL POINT D]: Blocks applyAll until raftNode finishes r.storage.Save()!
    <-apply.notifyc

    s.snapshotIfNeededAndCompactRaftLog(ep)
    // ...
}
```

This creates a **tightly coupled lockstep barrier** between consensus, network replication, WAL disk I/O, and state-machine application:
1. **State-machine stalls block consensus and network replication (Stall Point A):** Because `r.applyc` is an **unbuffered channel** (`raft.go:146`) and `r.applyc <- ap` (`raft.go:232`) executes **before** `r.transport.Send` (`raft.go:242`) and `r.storage.Save` (`raft.go:256`), if `applyAll` takes 20ms on batch $k$ (e.g., due to a backend commit or large transaction), `raftNode` cannot even send batch $k+1$'s `MsgApp` packets to followers or start batch $k+1$'s WAL `fdatasync`!
2. **WAL `fdatasync` blocks Raft state progression (Stall Point B):** While `r.storage.Save` is blocked inside `fileutil.Fdatasync` (`wal.go:881`), `raftNode` cannot call `r.Advance()`. Until `r.Advance()` is called, `go.etcd.io/raft/v3` suppresses emitting the next `Ready` struct—meaning follower `MsgAppResp` acknowledgments that arrive during the leader's `fdatasync` sit idle in `recvc` instead of advancing `Commit` and dispatching committed entries to `applyEntries`.
3. **Unused upstream capability (`AsyncStorageWrites`):** `go.etcd.io/raft/v3` `v3.7.0` (already imported by etcd `v3.7.3` in `server/go.mod:30`) explicitly added `Config.AsyncStorageWrites` (`MsgStorageAppend` / `MsgStorageApply`) to eliminate this exact synchronous `Ready`/`Advance` bottleneck, yet `raftConfig()` in `server/etcdserver/bootstrap.go:540-552 @ v3.7.3` leaves it disabled (upstream tracking issue [etcd#16291](https://github.com/etcd-io/etcd/issues/16291) and open PR [etcd#22223](https://github.com/etcd-io/etcd/pull/22223) have begun wiring it behind an alpha gate on `main`, though two upstream `etcd-io/raft` bugs—[#551](https://github.com/etcd-io/raft/issues/551) and [#543](https://github.com/etcd-io/raft/issues/543)—must be guarded against, as detailed in §3.1).

---

### 1.2 Bottleneck 2: The 4-Sync Segment Rotation Stall (`w.cut()`), `ext4`/`XFS` Unwritten-Extent Journal Commits, Startup `ZeroToEnd`, and `isTornEntry` Silent Truncation (`server/storage/wal/wal.go:566, 785-867`, `file_pipeline.go:81`, `decoder.go:126-148, 170-203 @ v3.7.3` — **High confidence**)

Every time the active `.wal` file reaches `SegmentSizeBytes` (`64 MB`, `wal.go:55`), `w.Save()` calls `w.cut()` **synchronously on the `raftNode` goroutine while holding `w.mu.Lock()`** (`wal.go:1036`). Inside `w.cut()` (`wal.go:785-867 @ v3.7.3`), etcd executes:
1. `w.tail().Seek(0, io.SeekCurrent)` and `w.tail().Truncate(off)` (`wal.go:787-794`) — modifying filesystem inode size metadata.
2. **Sync #1:** `w.sync()` $\to$ `fileutil.Fdatasync(w.tail().File)` (`wal.go:796`).
3. `newTail, err := w.fp.Open()` (`wal.go:803`) — waits on `filePipeline` (`file_pipeline.go:62-88`) if background `fallocate(64 MB)` on `0.tmp`/`1.tmp` is still running.
4. Writes `CrcType`, `MetadataType`, and `StateType` headers into `newTail` (`wal.go:816-826`).
5. **Sync #2:** `w.sync()` $\to$ `fileutil.Fdatasync(newTail.File)` (`wal.go:829`).
6. `os.Rename(newTail.Name(), fpath)` (`wal.go:838`) — renames `%d.tmp` to `%016x-%016x.wal`.
7. **Sync #3:** `fileutil.Fsync(w.dirFile)` (`wal.go:842`) — full directory `fsync` to persist the rename!
8. `newTail.Close()` (`wal.go:848`), followed by `fileutil.LockFile(fpath, os.O_WRONLY, ...)` (`wal.go:850` — `open(2)` + `flock(2)`) and `newTail.Seek(off, io.SeekStart)`.
9. Immediately wakes `filePipeline.run()` (`file_pipeline.go:90-105`), which creates the next `.tmp` file and calls `fileutil.Preallocate(64 MB)` on the same directory, contending with ongoing WAL `fdatasync`s for the filesystem journal (e.g., ext4 `jbd2` or XFS log).

Why does `wal.WAL` perform this expensive file-creation/rename/directory-fsync dance every 64 MB? Because it encodes the starting Raft index in the **filename** (`%016x-%016x.wal`, `util.go:110 @ v3.7.3`) and uses zero-bytes (`l == 0`, `decoder.go:95 @ v3.7.3`) as its EOF sentinel! This design causes **three additional severe penalties** (**High confidence**):
- **Filesystem Journal Transaction Penalty on *Every* `fdatasync` (`fallocate` Unwritten-Extent Conversion — **High confidence** on mechanism; **Moderate confidence** on exact IOPS ratio on target NVMe):** `filePipeline.alloc()` (`server/storage/wal/file_pipeline.go:81 @ v3.7.3`) pre-allocates each new 64 MB `.tmp` file using `fileutil.Preallocate(f.File, fp.size, true)` (`client/pkg/fileutil/preallocate.go:27-36 @ v3.7.3`) $\to$ `preallocExtend` $\to$ `syscall.Fallocate(int(f.Fd()), 0, 0, sizeInBytes)` (`client/pkg/fileutil/preallocate_unix.go:25-27 @ v3.7.3`). On Linux `ext4` and `XFS`, `fallocate(mode 0)` allocates **unwritten extents** (flagged as uninitialized in the extent tree so reads return zeroes without disk I/O). Consequently, **every time `WAL.Save()` writes into a fresh 4 KiB block of an `fallocate`d WAL file and calls `fdatasync`**, the filesystem must convert that block's extent metadata from *unwritten* to *written*, which modifies filesystem metadata and forces `fdatasync` (`fs/ext4/fsync.c`, `xfs_inode_item.c`) to commit a full **filesystem journal transaction** (`jbd2` / XFS log) rather than a pure data flush! In 2025 Linux kernel benchmarks for `FALLOC_FL_WRITE_ZEROES`, 4 KiB synchronous writes + `fdatasync` into `fallocate(mode 0)` unwritten extents on NVMe achieved only **20.1k IOPS (167 MB/s)**, whereas overwriting already-written recycled blocks achieved **98.8k IOPS (772 MB/s — $4.9\times$ higher throughput)** because `fdatasync` skips the filesystem journal when extent state does not change. Because `v3.7.3` never recycles `.wal` files and instead `fallocate`s a fresh file every 64 MB, **every single WAL `fdatasync` in `v3.7.3` pays the unwritten-extent journal commit tax!**
- **Startup `ZeroToEnd` Stall (`wal.go:563-568 @ v3.7.3` — **High confidence**):** Using `l == 0` as the EOF sentinel forces `WAL.ReadAll()` on startup to call `fileutil.ZeroToEnd(w.tail().File)`, synchronously writing up to **64 MB of zeroes** before the server can finish booting.
- **Silent Committed-History Truncation Bug on Zeroed Sectors (`decoder.go:126-148, 170-203 @ v3.7.3` — **High confidence**):** When `WAL.ReadAll()` encounters either a protobuf unmarshal failure (`decoder.go:125-130 @ v3.7.3`) or `crc.ErrCRCMismatch` (`decoder.go:138-150 @ v3.7.3`) anywhere in the last `.wal` segment (`len(d.brs) == 1`), it calls `d.isTornEntry(data)` (`decoder.go:170-203`). `isTornEntry` splits `data` (`recBytes + padBytes`) across 512-byte sector boundaries and returns `true` if **any** 512-byte chunk is all zeroes—converting the error into `io.ErrUnexpectedEOF`! On startup (`openWALFromSnapshot` in `server/etcdserver/bootstrap.go:642-656 @ v3.7.3`), `io.ErrUnexpectedEOF` triggers `wal.Repair` (`server/storage/wal/repair.go:67-99 @ v3.7.3`), which **silently truncates the WAL at `lastOffset`** and boots cleanly. If a storage or firmware fault zeroes a 512-byte sector inside an *already-synced, already-committed* entry earlier in the last 64 MB segment, etcd `v3.7.3` misclassifies mid-log corruption as a torn tail write and silently truncates committed Raft entries (violating Raft's persistence invariant).

---

### 1.3 Bottleneck 3: Double Protobuf Serialization & Triple Memory Copy Per Entry (`wal.go:974-983`, `encoder.go:66-140`, `pagewriter.go:56-99 @ v3.7.3` — **High confidence**)

In `w.Save(st, ents)` (`wal.go:995-1037 @ v3.7.3`), for every single `raftpb.Entry`:
1. `w.saveEntry(ents[i])` (`wal.go:976`) calls `b := pbutil.MustMarshalMessage(e)`, allocating a new heap `[]byte` per entry (even though a `TODO: add MustMarshalTo to reduce one allocation` has sat at line 975 since 2015).
2. `w.encoder.encode(rec)` (`encoder.go:74-93`) computes CRC32C over `b`, wraps `b` inside a second protobuf message (`walpb.Record`), and marshals `walpb.Record` into `e.buf` (copying the payload bytes a second time).
3. `write()` (`encoder.go:124-140`) calls `e.bw.Write(uint64buf)` and `e.bw.Write(data)` on `ioutil.PageWriter` (`pkg/ioutil/pagewriter.go:56-99`), which copies the bytes a **third time** into a 128 KiB user-space buffer (`pw.buf`) before issuing `os.File.Write`.

---

### 1.4 Bottleneck 4: Every-10,000-Entry Triple-`fdatasync` Stall on `applyAll` (`server/etcdserver/server.go:1215-1228, 2083-2137 @ v3.7.3` — **High confidence**)

Despite a comment at `server/etcdserver/server.go:1216 @ v3.7.3` reading `// TODO: Remove disk snapshot in v3.7`, every `DefaultSnapshotCount = 10,000` applied entries (`server.go:78`), the `applyAll` goroutine synchronously executes `s.snapshot(ep, true)` (`server.go:2083-2137`):
1. Marshals cluster membership into legacy v2 JSON format (`GetMembershipInfoInV2Format`, `server.go:2085`).
2. Calls `s.KV().Commit()` (`server.go:2105`) $\to$ **synchronous backend commit (`2x fdatasync` in `bbolt`)**.
3. Calls `s.r.storage.SaveSnap(snap)` (`server.go:2124`, `server/storage/storage.go:59-77`), which:
   - Calls `st.s.SaveSnap(snap)` (`server/etcdserver/api/snap/snapshotter.go:79-109`) $\to$ creates `%016x-%016x.snap`, writes it, and calls **`WriteAndSyncFile` (`fdatasync` + directory `fsync`)**—even though on normal v3 restart (`bootstrap.go:112, 413-438 @ v3.7.3`), etcd reconstructs `ConfState` from the v3 backend (`buildConfStateFromV3store`) and `Index`/`Term` from `wal.ValidSnapshotEntries`, never even reading the `.snap` file! *(Note: Upstream `main` commit `5d1a827c1` on 2026-07-22 finally stopped writing the legacy v2 `.snap` file in `SaveSnap`, though `v3.7.3` still writes it and `main` still executes the synchronous `s.KV().Commit()` + `st.w.SaveSnapshot` on `applyAll`.)*
   - Calls `st.w.SaveSnapshot(&walsnap)` (`wal.go:1039-1061`) $\to$ **acquires `w.mu.Lock()` (blocking `raftNode`'s `Save()`!)** and calls **`w.sync()` (`fdatasync`)**!
4. Calls `s.r.storage.Release(snap)` (`storage.go:82-89`) $\to$ acquires `w.mu.Lock()` again (`wal.go:905`) to unlock old `.wal` files.

---

### 1.5 Bottleneck 5: O(N) Full-WAL Scans (`ValidSnapshotEntries`) and Runtime WAL Close/Reopen (`MinimalEtcdVersion` — **High confidence**)

- `ValidSnapshotEntries` (`wal.go:608-695 @ v3.7.3`) opens **every `.wal` file in the directory starting from file 0** (`openWALFilesFn(lg, walDir, names, 0, false)` at line 648) and decodes every record from the beginning of the earliest file just to locate `SnapshotType` and `StateType` records.
- Worse, `storage.MinimalEtcdVersion()` (`server/storage/storage.go:109-128 @ v3.7.3`, introduced by commit `2e2963af6` on `main` / backported to `v3.7.3` as `23b61e6d9`, and invoked by the cluster version monitor in `server.go:2213`) calls `st.w.LatestSnapshotEntry()` (`wal.go:597`, scanning all WAL files from index 0) and then calls **`st.w.Reopen()` (`wal.go:263-269`), which calls `w.Close()` on the live WAL, stops `filePipeline`, releases all `flock` locks, re-reads all WAL entries since the last snapshot via `w.ReadAll()`, zeroes the tail to 64 MB, and spawns a new `filePipeline` at runtime!**

---

### 1.6 Bottleneck 6: Double-WAL Write Amplification Across the Node (**High confidence**)

If we merely replace `bbolt` with an LSM engine (`pebble` or `grocksdb`) while keeping `server/storage/wal`, every single key-value mutation is written sequentially to a write-ahead log **twice**:
1. First as a `raftpb.Entry` in the Raft WAL (`server/storage/wal/`).
2. Second as an `mvccpb.KeyValue` in the LSM engine's WAL (`member/snap/db/*.log`), before being flushed a third time to L0 SSTables and compacted across L1..L$k$.

Replacing **both** the WAL and the storage engine allows us to co-design their durability contract so that **every mutation is written to a WAL only once**.

---

## 2. Exploring the Design Space: Three Radical Architectural Options

When replacing both the Raft WAL and `bbolt`, there are three distinct architectural ways to organize consensus persistence and state-machine storage:

| Architecture | Consensus Log (`RaftLog`) | State Machine (`MVCC + Meta`) | Total `fdatasync`s per `Put` | Vulnerability to LSM Compaction Stalls | Verdict |
|---|---|---|---|---|---|
| **Option A: Unified Single LSM** | Stored as KV pairs (`raft_log/<Index>`) inside the **same** `pebble` / `grocksdb` instance | Stored in the **same** `pebble` / `grocksdb` instance | **$1\times$** (at Raft `Save` with `Sync`; MVCC `Apply` uses `NoSync`, but writes `2x` to the same WAL/LSM) | **High (Fatal flaw):** L0/memtable stalls in the unified LSM block `r.storage.Save()`, causing Raft leader heartbeat timeouts and elections. | **Reject** (High confidence) |
| **Option B (Mode C2 Default): Co-Designed Recycled Raft WAL + `pebble.NoSync` State Store with `LogData(Sync)` Barrier** | Dedicated recycled log engine (`pebble/record.LogWriter` + `FormatWALSyncChunks`) + `AsyncStorageWrites` | `pebble` with its recycled WAL enabled in **`pebble.NoSync`** mode; periodic background **`db.LogData(nil, pebble.Sync)`** advances `safeRecycleIndex` | **$1\times$** (1 group-committed Raft WAL `fdatasync`; **0 `fdatasync`s** on MVCC `applyEntries` or 100ms batch commits) | **Zero:** Raft log appends are 100% isolated from Pebble LSM compaction. 100% covered by Pebble's metamorphic crash tests (`options.go:852`) and supports native `DeleteRange`. | **Recommended Default Target Architecture** (High confidence) |
| **Option C (Mode C1 Opt-In): Co-Designed Recycled Raft WAL + WAL-Less Checkpoint-Fenced `pebble`** | Dedicated recycled log engine (`pebble/record.LogWriter` + `FormatWALSyncChunks`) + `AsyncStorageWrites` | `pebble` with **`DisableWAL: true`**; background `db.Flush()` + immutable hard-link **`db.Checkpoint()`** rollover advances `safeRecycleIndex` | **$1\times$** (written to a log **only once** in the Raft WAL; `applyEntries` is 100% in-memory with `0 write()` and `0 fdatasync()` syscalls) | **Zero:** Raft log appends are isolated; requires restoring from the last immutable `db.Checkpoint()` directory after an unclean crash because Pebble does not crash-test `DisableWAL: true` across mid-compaction crashes. | **High-Throughput Opt-In Mode** (Moderate–High confidence) |

### 2.1 Why Option A (Unified Single LSM for Both Raft Log and MVCC) Is an Anti-Pattern

It is tempting to ask: *"Since Pebble already has a fast WAL and key-value store, why not delete `server/storage/wal/` and store Raft log entries as keys (`\x00raft/<8-byte big-endian Index> -> raftpb.Entry`) inside the exact same Pebble instance that stores MVCC keys?"*

**This architecture has been tested at scale in production by both TiKV (PingCAP) and Dragonboat, and both projects built dedicated log engines (`tikv/raft-engine` and Dragonboat's `tan`) specifically to escape it (Verified — High confidence):**
1. **Compaction Write Amplification on Ephemeral Data:** Raft log entries have a completely different lifecycle from MVCC keys. Raft entries are written strictly sequentially by `Index`, read almost exclusively from the recent tail (which is already in memory), and **100% deleted in bulk** every `10,000` entries via prefix truncation (`Compact`). If inserted into a unified Pebble LSM, those `raftpb.Entry` records fill the 64 MiB MemTable, flush to L0 SSTables alongside MVCC keys, and get compacted into L1/L2 SSTables *before* `DeleteRange([0, compactIndex))` can physically reclaim them at the bottommost LSM level. Every client payload is therefore flushed and compacted **twice** by the LSM (once as a `raftpb.Entry`, once as an `mvccpb.KeyValue`).
2. **Failure Domain Violation (LSM Write Stalls Kill Raft Consensus):** Every LSM engine (`pebble` and `grocksdb`) enforces L0 sublevel and memtable backpressure (`L0StopWritesThreshold`): when background compaction temporarily falls behind burst writes, the LSM intentionally delays or blocks incoming `Batch.Commit()` calls. In a unified Pebble DB, a spike in MVCC compaction or SSTable flushing delays `r.storage.Save(rd.HardState, rd.Entries)`. Blocking `Save()` delays Raft heartbeats and `MsgAppResp` replies beyond `ElectionTimeout`, triggering leader step-downs (`CheckQuorum`) and cascading elections across the cluster.

**Conclusion:** Consensus log persistence (`Save`) and state-machine storage (`MVCC`) **must** have independent write paths and independent I/O backpressure domains.

---

## 3. Detailed Specification of the Proposed Target Architecture (Option B / C Co-Design)

Our proposed replacement architecture redesigns etcd's persistence layer around **four tightly integrated components**:
1. **Asynchronous Consensus Pipeline (`AsyncStorageWrites` in `server/etcdserver/raft.go`, with safety guards for `raft#551` and `raft#543`)**
2. **`RaftLogEngine` (Recycled, Group-Committed Segmented WAL built on `github.com/cockroachdb/pebble/record` with `FormatWALSyncChunks` + Bounded Tail Cache)**
3. **Checkpoint-Fenced `pebble` MVCC Engine (`Mode C2 Default: pebble.NoSync` + background `db.LogData(nil, pebble.Sync)` barrier, or `Mode C1 Opt-In: DisableWAL: true` + `db.Checkpoint()` rollover)**
4. **Read-Path Accelerator (`treeIndex` + Revision-Sorted Single-Iterator `rangeKeys` with `TrySeekUsingNext` in `server/storage/mvcc/kvstore_txn.go`)**

```
                            ┌──────────────────────────────────────────────────────────┐
                            │                 1. Raft State Machine                    │
                            │         (`go.etcd.io/raft/v3` v3.7.0 in memory)          │
                            │     Config.AsyncStorageWrites = true (NON-BLOCKING)      │
                            │     + Guards for `raft#551` (MsgVote) & `raft#543`       │
                            └─────────────┬──────────────────────────────┬─────────────┘
                                          │                              │
                        MsgStorageAppend  │ (async chan)                 │ MsgStorageApply (async chan,
                        (Entries, State)  ▼                              ▼  dispatched as soon as quorum ACKs)
┌─────────────────────────────────────────────────────┐      ┌─────────────────────────────────────────────────────┐
│        2. LocalAppendThread (`RaftLogEngine`)       │      │       3. LocalApplyThread (`pebble` MVCC Engine)    │
│  • Single-pass flat batch encode (`MarshalAppend`)  │      │  • Applies entries to `treeIndex` + `IndexedBatch`  │
│  • `pebble/record.LogWriter` (`SyncRecord` queue)   │      │  • `Unlock()` NEVER fdatasyncs (even on deletes!)   │
│  • `FormatWALSyncChunks` (synced-offset header)     │      │  • Commits batch via `pebble.NoSync` (0 `fdatasync` │
│  • Pre-written ring of 64MiB `.log` files           │      │    syscalls!) and THEN advances `mvcc.currentRev`   │
│    recycled via `logNum` header (0-fsync rotation!) │      │  • Background checkpoint calls `db.LogData(Sync)`   │
│  • Replies `MsgStorageAppendResp` -> Raft/Network   │      │    (Mode C2) or `Flush`+`Checkpoint` (Mode C1)      │
└─────────────────────────┬───────────────────────────┘      └─────────────────────────┬───────────────────────────┘
                          │                                                            │
                          └────────────◄─── Advances `safeRecycleIndex` ───────────────┘
                                       (Recycles `.log` segments <= durablePebbleIndex)
```

---

### 3.1 Component 1: Asynchronous Consensus Pipeline (`AsyncStorageWrites`) and Mandatory Safety Guards

Instead of `raftNode.start()` (`server/etcdserver/raft.go:185-336 @ v3.7.3`) blocking synchronously on `r.applyc <- ap` and `r.storage.Save()`, we enable **`Config.AsyncStorageWrites = true`** in `raftConfig()` (`server/etcdserver/bootstrap.go:540 @ v3.7.3`; see upstream tracking issue [etcd#16291](https://github.com/etcd-io/etcd/issues/16291) and open PR [etcd#22223](https://github.com/etcd-io/etcd/pull/22223)).

In `go.etcd.io/raft/v3` `v3.7.0`, `AsyncStorageWrites` replaces the blocking `Ready()`/`Advance()` contract with two ordered local message streams inside `rd.Messages`:
1. **`MsgStorageAppend` (targeted at `raft.LocalAppendThread = math.MaxUint64`):**
   - Carries `Entries` to append, `HardState` (`Term`, `Vote`, `Commit`) to persist, and attached `Responses` (`MsgAppResp`, `MsgVoteResp`, or internal local notifications) to send once the append is durable on disk.
   - Our **`LocalAppendThread`** goroutine (or pool of batch-coalescing workers feeding `RaftLogEngine`) reads `MsgStorageAppend` from a buffered channel (capacity `64`), appends the batch to `RaftLogEngine`, waits for the group-committed `fdatasync` to complete, immediately dispatches the attached `Responses` to `r.transport.Send()` and `r.Step()`, and never blocks the `raftNode` loop!
   - **Why this transforms write throughput:** While `fdatasync` #1 is in flight on the NVMe drive (taking e.g. `200 µs–1 ms`), `raftNode` continues running! It accepts new client proposals, steps incoming follower `MsgAppResp` messages, immediately sends new `MsgApp` replication batches over the network to followers, and queues `MsgStorageAppend` #2, #3, #4 into `LocalAppendThread`'s channel. The instant `fdatasync` #1 finishes, `LocalAppendThread` drains #2, #3, and #4 in a single batch and syncs them in **one** `fdatasync`!
2. **`MsgStorageApply` (targeted at `raft.LocalApplyThread = math.MaxUint64 - 1`):**
   - Carries `CommittedEntries` to apply to the MVCC state machine.
   - Our **`LocalApplyThread`** goroutine reads from a buffered channel, applies the entries to `treeIndex` and `pebble.IndexedBatch`, immediately triggers client response waiters (`s.w.Trigger(id, resp)`), and steps `MsgStorageApplyResp` back to `raftNode`.
   - Notice what is gone: **`<-apply.notifyc` (`server.go:983 @ v3.7.3`) is completely eliminated** because `go.etcd.io/raft/v3` itself guarantees that `MsgStorageApply` is only emitted for entries whose `Index <= Commit` (and on the leader/follower, after local storage append if required by the Raft state machine)!

#### Mandatory Safety Guards in `raftNode` for Open `etcd-io/raft` `v3.7.0` Bugs (`raft#551`, `raft#543`, and ConfChange Barrier)

Static inspection of `go.etcd.io/raft/v3` `v3.7.0` and `server/etcdserver/raft.go @ v3.7.3` reveals **four mandatory correctness guards** that our `raftNode` loop must enforce when `AsyncStorageWrites = true` (Verified — High confidence):
1. **Deferring Outgoing `MsgVote` Until `MsgStorageAppend` Is Durable ([etcd-io/raft#551](https://github.com/etcd-io/raft/issues/551), fix PR [#550](https://github.com/etcd-io/raft/pull/550)):**
   In `go.etcd.io/raft/v3` `v3.7.0` (`raft.go:546`), `send()` routes `MsgAppResp`, `MsgVoteResp`, and `MsgPreVoteResp` into `MsgStorageAppend.Responses` so they are transmitted only after the corresponding `HardState` and `Entries` are synced to disk—**but omits outgoing `MsgVote`!** When a node campaigns for election (`campaign()`), it increments `Term`, sets `Vote = r.id`, and immediately emits `MsgVote` in `rd.Messages` *before* `MsgStorageAppend` has persisted the new `Term` and `Vote`. If the candidate transmits `MsgVote` to peers, loses power before `LocalAppendThread` finishes `fdatasync`, and reboots at its prior `Term`/`Vote`, it can grant a vote to a different candidate in the same term, violating Raft election safety. Until `etcd-io/raft#550` is merged and tagged, our `raftNode` loop must intercept any outgoing `MsgVote` in `rd.Messages` and append it to the same `Ready` batch's `MsgStorageAppend.Responses` slice so `MsgVote` is only sent over `r.transport` after `RaftLogEngine` syncs `HardState`.
2. **Serializing Snapshot Backend Swaps Through `LocalApplyThread`'s FIFO Queue ([etcd-io/raft#543](https://github.com/etcd-io/raft/issues/543)):**
   In `AsyncStorageWrites` mode, an incoming Raft snapshot (`MsgSnap`) is delivered inside `MsgStorageAppend.Snapshot` (`LocalAppendThread`), whereas previously committed entries are delivered inside `MsgStorageApply` (`LocalApplyThread`). If `LocalAppendThread` replaces the live MVCC `pebble.DB` instance immediately upon persisting `MsgStorageAppend.Snapshot` while older `MsgStorageApply` batches are still queued in `LocalApplyThread`'s channel, `LocalApplyThread` can subsequently apply stale log entries *on top of* the newer snapshot! To prevent this race, `LocalAppendThread` must only write and `fsync` the incoming snapshot directory (`member/snap/db.tmp`) and then enqueue a ` snapshotSwapTask` into `LocalApplyThread`'s FIFO channel before stepping `MsgStorageAppendResp`, ensuring all prior `MsgStorageApply` batches drain before the Pebble database handle is swapped.
3. **Preserving the Follower `EntryConfChange` Transport Barrier (`server/etcdserver/raft.go:289-336` & `server/etcdserver/server.go:983, 1951 @ v3.7.3`):**
   On a follower, when `CommittedEntries` contains an `EntryConfChange`, `raft.go:289-325 @ v3.7.3` performs a two-send handshake on the capacity-1 `notifyc` channel (`notifyc <- struct{}{}` at line 302 and a second blocking `notifyc <- struct{}{}` at line 317) so `raftNode` blocks until `applyAll` (`server.go:983`) has applied all preceding entries and triggered `applyWait` before `raftNode` calls `r.transport.Send(msgs)` (`raft.go:324`), while `s.apply()` (`server.go:1951`) waits on `<-raftAdvancedC` (`raft.go:335`) before applying the `ConfChange`. In `AsyncStorageWrites` mode, whenever `LocalAppendThread` processes a batch where a prior or current `MsgStorageApply` contains an unapplied `EntryConfChange` (or `EntryConfChangeV2`), it must await the conf-change apply completion barrier before dispatching peer responses.
4. **Updating `tests/robustness` GoFail Failpoints and WAL Report Decoder (`tests/robustness/failpoint/gofail.go:38-39, 53-59` & `tests/robustness/report/wal.go:35, 368 @ v3.7.3`):**
   Existing robustness failpoints (`raftBeforeSave`, `raftAfterSave` at `gofail.go:38-39`, and `raftBeforeLeaderSend`, `raftBeforeFollowerSend`, `raftBeforeApplySnap`, `raftAfterApplySnap`, `raftAfterWALRelease`, `raftBeforeSaveSnap`, `raftAfterSaveSnap` at `gofail.go:53-59`, corresponding to `server/etcdserver/raft.go:241-284 @ v3.7.3`) assume synchronous execution on `raftNode`. Equivalent failpoints must be placed inside `LocalAppendThread` and the background checkpoint coordinator, and `ReadAllWALEntries` in `tests/robustness/report/wal.go:368 @ v3.7.3` must be updated to read `RaftLogEngine` segments so etcd's Porcupine linearizability tests can reconstruct persisted operations after crash failpoints.

---

### 3.2 Component 2: `RaftLogEngine` — Recycled, Group-Committed WAL Built on `pebble/record.LogWriter` (**High confidence**)

We replace `server/storage/wal` (`wal.WAL`, `encoder`, `decoder`, `filePipeline`) and `server/etcdserver/api/snap` (`Snapshotter` `.snap` files) with a purpose-built **`RaftLogEngine`** package (`server/storage/raftlog/`) built on **`github.com/cockroachdb/pebble/record`** (**High confidence**).

#### 1. Zero-Fsync Segment Rotation & Zero Unwritten-Extent Journal Commits via Pre-Written File Recycling (`logNum`)
Why write a custom framing format from scratch when `cockroachdb/pebble/record` (`v2.1.7`) already provides CockroachDB's battle-tested WAL record layer?
- **How `pebble/record` with `WriteWALSyncOffsets` works (`pebble/record/record.go:87-100, 139-145`, `log_writer.go:477-516 @ v2.1.7`):**
  `record.LogWriter` formats variable-length records into fixed **32 KiB physical blocks**. When configured with `LogWriterConfig{WriteWALSyncOffsets: func() bool { return true }}` (`log_writer.go:512-515`), `LogWriter` emits WAL-sync chunks (`walSyncFullChunkEncoding..walSyncLastChunkEncoding`, `record.go:133-136`) whose header inside each 32 KiB block is 19 bytes (`walSyncHeaderSize = recyclableHeaderSize + 8`, `record.go:96-98, 144`):
  $$\underbrace{\text{CRC32C}}_{\text{4 bytes}} \;\Vert\; \underbrace{\text{Size}}_{\text{2 bytes}} \;\Vert\; \underbrace{\text{Type}}_{\text{1 byte}} \;\Vert\; \underbrace{\text{LogNum}}_{\text{4 bytes (uint32 file generation)}} \;\Vert\; \underbrace{\text{SyncOffset}}_{\text{8 bytes (uint64 promised sync offset)}}$$
- **Pre-Written Segment Recycling Pool (Eliminating `ext4`/`XFS` Unwritten Extents):**
  - At cluster bootstrap (or when growing the pool), `RaftLogEngine` maintains a ring of `64 MiB` segment files in `member/wal/` named by logical `logNum` (`%06d.log`), plus a small `MANIFEST` recording the active `minLogNum..maxLogNum` range.
  - **Crucial kernel optimization:** Whenever a new `.log` segment file is allocated to expand the pool, `RaftLogEngine` **writes real zero blocks once** (or uses `FALLOC_FL_WRITE_ZEROES` where supported by the kernel) rather than `fallocate(mode 0)` alone. This converts all 64 MiB of filesystem extents to **written extents** up front. From that point forward, every `fdatasync` during normal operation and across `logNum` segment rotations overwrites already-allocated, already-written blocks—completely avoiding `ext4`/`XFS` unwritten-extent conversion and its **~5× journal commit throughput penalty** (§1.2)!
  - When the active `LogWriter` reaches `64 MiB`, rotation does **not** truncate the old file, does **not** rename `.tmp` files, does **not** `fsync` the parent directory, and does **not** call `fallocate` if a recyclable segment is available (`logNum < safeRecycleLogNum`).
  - Instead, it pops the oldest obsolete segment file from the recycle queue, reuses its pre-written file blocks with `newLogNum = prevLogNum + 1`, seeks to offset `0` (`Seek(0, io.SeekStart)`), and instantiates `record.NewLogWriter(file, newLogNum, cfg)`!
- **Deterministic Crash Recovery & Fixing `v3.7.3`'s `isTornEntry` Silent-Truncation Bug:**
  - **End-of-log detection without `ZeroToEnd`:** Because the recycled file contains old 32 KiB blocks from generation `oldLogNum < newLogNum` beyond the current write offset, on startup `record.NewReader(file, newLogNum)` checks the 4-byte `LogNum` in every chunk header. As soon as it reads past the last written block of `newLogNum`, it encounters `oldLogNum != newLogNum`, returns `record.ErrInvalidLogNum` (treated as clean `io.EOF`), and stops—eliminating `wal.go:566`'s 64 MB `ZeroToEnd` startup stall.
  - **Distinguishing mid-log storage corruption from a torn tail (`SyncOffset`):** Unlike `v3.7.3`'s `isTornEntry()` (`decoder.go:126-148, 170-203`), which misclassifies any zeroed 512-byte sector in the last WAL file as a benign torn tail and silently truncates committed entries, `WriteWALSyncOffsets` records the 8-byte `SyncOffset` (`log_writer.go:480`) promised by `LogWriter.SyncRecord`. During recovery:
    1. If a chunk at file offset `< maxPromisedSyncOffset` fails CRC32C or has an invalid `LogNum`, `RaftLogEngine` raises a **fatal storage corruption error (`ErrCorruptWAL`)** and refuses to silently truncate committed Raft history.
    2. If a chunk at file offset $\ge \text{maxPromisedSyncOffset}$ fails CRC32C or returns `ErrInvalidLogNum`, it is provably an unsynced torn write that was never acknowledged to Raft or any client, and `RaftLogEngine` truncates cleanly at the last valid record boundary.

#### 2. Single-Pass Flat Batch Encoding (Eliminating Double Protobuf & Triple Copy)
Instead of `wal.WAL`'s per-entry `MustMarshalMessage(e)` $\to$ `walpb.Record` marshal $\to$ `PageWriter` copy (`wal.go:974-983`, `encoder.go:66-140`), `RaftLogEngine.Save(st, ents)` encodes an entire batch of entries + `HardState` into a **single `LogWriter` record** using a reusable pre-allocated buffer (`batchBuf []byte`):

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ Single `LogWriter` Batch Record Payload (written in one call to `SyncRecord`)          │
├──────────────┬────────────────────────┬──────────────────┬─────────────────────────────┤
│ BatchHeader  │ HardState (optional)   │ EntryCount       │ Packed Entries              │
│ 1B flags     │ 8B Term | 8B Vote |    │ uvarint N        │ For i = 0..N-1:             │
│ (hasState,   │ 8B Commit (fixed 24B,  │                  │   [4B LittleEndian Len_i]   │
│  hasSnap,    │ zero protobuf overhead)│                  │   [raftpb.Entry MarshalTo]  │
│  hasMeta)    │                        │                  │   (marshaled directly into  │
│              │                        │                  │    batchBuf[off : off+Len]) │
└──────────────┴────────────────────────┴──────────────────┴─────────────────────────────┘
```

- **Memory copies eliminated:**
  - `HardState` is 3 `uint64`s (`Term`, `Vote`, `Commit`)—written as 24 raw little-endian bytes instead of a heap-allocated protobuf `walpb.Record`.
  - Each `raftpb.Entry` is marshaled **once**, directly into `batchBuf` using `proto.MarshalOptions{UseCachedSize: true}.MarshalAppend(batchBuf, ent)`.
  - `LogWriter.SyncRecord(batchBuf, wg, &err)` frames `batchBuf` into 32 KiB blocks and computes a single hardware-accelerated Castagnoli CRC32C per chunk.

#### 3. Built-In Group Commit via `LogWriter.SyncRecord`
`pebble/record.LogWriter` runs an internal flush loop with a lock-free/condvar-coordinated sync queue (`flusher.syncQueue`).
- If `mustSync == false` (`!raft.MustSync(st, prevSt, len(ents))`), `LogWriter.WriteRecord(batchBuf)` queues the bytes in the 32 KiB block buffer without waking the disk syncer.
- If `mustSync == true`, `LogWriter.SyncRecord(batchBuf, wg, &syncErr)` appends the record and coalesces all concurrent/pipelined batches that arrived during the previous `fdatasync` into a **single `write(2)` + `fdatasync(2)`**.

#### 4. Bounded In-Memory Tail Ring + Sparse Index (Replacing `raft.MemoryStorage` & `ValidSnapshotEntries`)
In `v3.7.3`, etcd keeps **all uncompacted Raft entries in a contiguous Go slice in RAM** (`raft.MemoryStorage.ents` in `bootstrap.go:701-713 @ v3.7.3`) AND scans all `.wal` files from index 0 on `ValidSnapshotEntries` (`wal.go:608-695`) and `MinimalEtcdVersion` (`storage.go:109`).
- In `RaftLogEngine`:
  - We maintain a **bounded in-memory ring buffer** of the most recent $W$ entries (e.g., `16,384` entries or `64 MiB` max) to serve `raft.Storage.Entries(lo, hi, maxSize)` and `Term(i)` in $O(1)$ without disk I/O for active followers.
  - For older entries still in the retained `.log` segments (e.g., allowing up to `512 MiB` of retained WAL segments so a temporarily disconnected follower can catch up via log replay instead of triggering an expensive full-DB `MsgSnap` snapshot transfer!), `RaftLogEngine` keeps a compact in-memory sparse index (`[]struct{ FirstIndex uint64; LogNum uint32; Offset uint32 }` — only 16 bytes per batch!) and reads cold entries on demand via `pread(2)` (`os.File.ReadAt`) without touching the write offset!
  - Furthermore, `RaftLogEngine` tracks `maxEtcdVersion *semver.Version` incrementally as entries are appended (`wal.MinimalEtcdVersion(ents)`), so `storage.MinimalEtcdVersion()` (`storage.go:109 @ v3.7.3`) returns in **$O(1)$ from an atomic pointer** instead of closing, scanning, and reopening the WAL at runtime!

---

### 3.3 Component 3: Co-Designed `pebble` State Machine (Zero-`fdatasync` Apply Path & Background `LogData(Sync)` Checkpoint Barrier)

When we pair `RaftLogEngine` (Component 2) with **`pebble` (`v2.1.7`)** as the `backend.Backend` implementation, we unlock a major architectural simplification over replacing `bbolt` in isolation:

#### 1. Eliminating Backend `fdatasync`s (`Mode C2 Default: Unsynced Recycled WAL + LogData(Sync)` vs. `Mode C1 Opt-In: DisableWAL + Checkpoint Rollover`)
Because `RaftLogEngine` already durably `fdatasync`s every committed `raftpb.Entry` before it is applied, **any state in Pebble that has not yet been synced to disk is 100% reconstructible by replaying `RaftLogEngine` from `pebble.ConsistentIndex() + 1` to `HardState.Commit`!**

During normal execution (`applyEntries`), `batchTxBuffered` accumulates MVCC key-value writes (`UnsafeSeqPut`, `UnsafePut`) and deletes (`UnsafeDelete`) into a **`pebble.IndexedBatch`**. Because `pebble.IndexedBatch` natively indexes deletion tombstones in memory, **`Unlock()` (`batch_tx.go:335 @ v3.7.3`) NEVER forces a synchronous commit on `UnsafeDelete` (`LeaseRevoke` or MVCC compaction)**. Every `100ms` (or `10,000` operations), `batchTxBuffered.Commit()` invokes `OnPreCommitUnsafe` (writing `meta/consistent_index` and `meta/term` into the same `IndexedBatch`), calls `indexedBatch.Commit(pebble.NoSync)` (**zero `fdatasync` syscalls**), and **only after `indexedBatch.Commit(pebble.NoSync)` returns** publishes the updated `mvcc.currentRev` so concurrent readers can read directly from Pebble's MemTable.

We support two durability modes for the Pebble state store, with **Mode C2 as the recommended default**:

1. **Mode C2 (Recommended Default — `pebble.Options{DisableWAL: false}` with `pebble.NoSync` + Periodic `db.LogData(nil, pebble.Sync)` Barrier):**
   - **How it works on the apply path:** `indexedBatch.Commit(pebble.NoSync)` appends the batch to Pebble's own recycled `.log` file in the OS page cache (`1 buffered write(2)`, **`0 fdatasync(2)`**) and inserts into Pebble's 64 MiB MemTable.
   - **Why Mode C2 is the safest default (Verified in Pebble `v2.1.7`):**
     - **100% covered by Pebble's metamorphic crash test suite:** Inspection of `pebble/options.go:537-542 @ v2.1.7` (`// TODO(peter): untested`) and `pebble/metamorphic/options.go:683, 851-853 @ v2.1.7` shows that Pebble's own crash-consistency test suite **forcibly sets `DisableWAL = false` whenever crash testing (`strictFS` / `vfs.NewCrashableMem()`) is enabled**, because an unclean crash during a background SSTable flush or compaction with `DisableWAL: true` cannot recover the in-flight working directory from a WAL.
     - **Strict atomic-prefix recovery (`pebble/commit.go:150-205 @ v2.1.7`):** Pebble's `commitPipeline` serializes WAL appends under a mutex and writes each `IndexedBatch` (including `OnPreCommitUnsafe`'s `consistent_index`) as an atomic batch record. After any process crash or OS power loss, Pebble recovers an exact atomic prefix of batches up to some `consistent_index` $I_{\text{pebble}}$, and `RaftLogEngine` replays $(I_{\text{pebble}}, I_{\text{wal}}]$.
     - **Lightweight checkpoint barrier without forcing 64 MiB SSTable flushes (`db.LogData(nil, pebble.Sync)` at `pebble/db.go:736-745 @ v2.1.7`):** To advance `RaftLogEngine`'s `safeRecycleIndex` in the background (e.g., every `10,000` applied entries or `5s`), the background checkpoint goroutine does **not** even need to flush Pebble's 64 MiB MemTable to an L0 SSTable! It simply reads the latest committed `consistent_index` $I_{\text{mem}}$, calls **`db.LogData(nil, pebble.Sync)`** (which issues **one** `fdatasync` on Pebble's WAL, making all prior `pebble.NoSync` batches $\le I_{\text{mem}}$ durable on disk), and calls `raftLogEngine.AdvanceSafeRecycleIndex(I_mem - cfg.SnapshotCatchUpEntries)`.
     - **Instant `SIGKILL` recovery & native `DeleteRange`:** If the `etcd` process restarts without an OS reboot, the kernel page cache already holds the unsynced Pebble WAL bytes, so replay takes `0` entries; and Pebble's native `DeleteRange` tombstones work out of the box.

2. **Mode C1 (High-Throughput Opt-In — `pebble.Options{DisableWAL: true}` + `db.Flush()` & Immutable `db.Checkpoint()` Rollover):**
   - **How it works on the apply path:** `indexedBatch.Commit(pebble.NoSync)` inserts directly into Pebble's in-memory MemTable skiplist with **zero disk writes and zero syscalls (`0 write()`, `0 fdatasync()`)**.
   - **Mandatory crash-consistency guard for `DisableWAL: true`:** Because Pebble does not guarantee that a working directory with `DisableWAL: true` can open cleanly if power is lost mid-flush/mid-compaction (`pebble/metamorphic/options.go:851-853`), Mode C1's background checkpoint coordinator must: (a) call `db.Flush()` to sync the MemTable to an L0 SSTable, (b) create an immutable hard-link checkpoint directory via **`db.Checkpoint(checkpointDir)`** (`pebble/checkpoint.go:110-125 @ v2.1.7`, which takes `<1ms` via `link(2)` and `fsync`s the checkpoint directory), and (c) only then advance `safeRecycleIndex` in `RaftLogEngine`. On startup after an unclean shutdown, Mode C1 replaces the working directory with a hard-linked copy of the last verified `checkpointDir` before replaying `RaftLogEngine` from $I_{\text{ckpt}} + 1$.

3. **Filesystem Journal Isolation Between `RaftLogEngine` and `pebble` ([cockroach#16624](https://github.com/cockroachdb/cockroach/issues/16624)):**
   - In CockroachDB's two-engine investigation ([cockroach#16624](https://github.com/cockroachdb/cockroach/issues/16624)), sharing a single `ext4` (`data=ordered`) filesystem between a synced Raft-log store and an unsynced state store caused `ext4` journal commits (`jbd2`) triggered by the Raft log's `fdatasync` to flush dirty page-cache buffers belonging to the unsynced state store!
   - Our design prevents this cross-store coupling in two ways: first, pre-writing `RaftLogEngine`'s recycled 64 MiB segment files (§3.2.1) ensures that `RaftLogEngine`'s `fdatasync` calls do not modify block-allocation metadata and therefore do not trigger `ext4` journal commits; second, in production deployments on `ext4`, placing `--wal-dir` (`RaftLogEngine`) and `--data-dir` (Pebble state store) on separate filesystems/mounts (or using `XFS`) guarantees zero kernel writeback coupling between the synced Raft log and unsynced Pebble state store.

- **What both modes eliminate in `server/etcdserver/server.go:1215-1228, 2083-2137 @ v3.7.3`:**
  The synchronous `s.snapshot(ep, true)` call every `10,000` entries on the `applyAll` goroutine (`s.KV().Commit()` + `Snapshotter.SaveSnap` + `WAL.SaveSnapshot` = 3 synchronous `fdatasync`s) is **completely removed from the `applyAll` critical path**!

#### 2. Proof of Crash-Recovery Correctness Across All Failure Scenarios (High Confidence)

Let $I_{\text{durable}}$ be the `consistent_index` at the last completed background checkpoint barrier (`db.LogData(nil, pebble.Sync)` in Mode C2, or `db.Flush()` + `db.Checkpoint()` in Mode C1), $I_{\text{pebble}}$ be the `consistent_index` recovered by Pebble on reboot ($I_{\text{durable}} \le I_{\text{pebble}} \le I_{\text{mem}}$), and $I_{\text{wal}}$ be `HardState.Commit` in `RaftLogEngine` ($I_{\text{mem}} \le I_{\text{wal}}$).

| Crash / Failure Scenario | State on Disk Upon Reboot | Recovery Behavior & Why It Is 100% Linearizable |
|---|---|---|
| **Case 1: Power loss or kernel panic during normal writes** | In Mode C2, Pebble replays its WAL up to the last intact batch record $I_{\text{pebble}} \ge I_{\text{durable}}$ (each `IndexedBatch` commits atomically with its `OnPreCommitUnsafe` `consistent_index`). In Mode C1, Pebble restores the last immutable `checkpointDir` at $I_{\text{pebble}} = I_{\text{durable}}$. `RaftLogEngine` holds all entries $(I_{\text{durable}} - \text{catchup}, I_{\text{wal}}]$ because `.log` segments $\ge I_{\text{durable}}$ were fenced against recycling until the checkpoint barrier completed. | On boot (`bootstrap.go`), Pebble opens at $I_{\text{pebble}}$. `RaftLogEngine` replays committed entries $(I_{\text{pebble}}, I_{\text{wal}}]$. `EtcdServer.apply` and `uberApplier` skip v3 backend application for entries $\le I_{\text{pebble}}$ (`server/etcdserver/server.go:1900-1914` & `server/etcdserver/apply/uber_applier.go:126-128 @ v3.7.3`) and deterministically re-apply $(I_{\text{pebble}}, I_{\text{wal}}]$ into Pebble, restoring exact state at $I_{\text{wal}}$. |
| **Case 2: Crash mid-`SyncRecord` in `RaftLogEngine` (torn WAL write vs. mid-file corruption)** | Last 32 KiB block in active `.log` segment has an invalid CRC32C or trailing `oldLogNum != activeLogNum` at offset $\ge \text{maxPromisedSyncOffset}$. | `pebble/record.Reader` with `WriteWALSyncOffsets` checks the chunk offset against `maxPromisedSyncOffset`: a mismatch at/above the synced offset is truncated cleanly at the last valid batch boundary ($\ge I_{\text{wal}}$), whereas a mismatch below the synced offset halts with `ErrCorruptWAL`. Zero acknowledged writes are lost. |
| **Case 3: Crash during Pebble background MemTable flush or compaction** | In Mode C2, Pebble's synced/unsynced WAL files remain intact until the new `MANIFEST` syncs, recovering cleanly to $I_{\text{pebble}} \ge I_{\text{durable}}$. In Mode C1, the immutable `checkpointDir` is untouched by in-flight compactions in the working directory. | In both modes, $I_{\text{pebble}} \le I_{\text{wal}}$ and `RaftLogEngine` still holds all entries $> I_{\text{durable}}$. Replay from $I_{\text{pebble}} + 1$ is exact and idempotent. |
| **Case 4: Follower receives Raft `MsgSnap` (`applySnapshot`)** | Leader creates a hard-link `pebble.Checkpoint` in `<1ms`, streams it as a 512-byte-aligned `.tar` archive (`WriteTo`), and follower `LocalAppendThread` saves `member/snap/db.tmp`. | Follower `LocalAppendThread` enqueues a snapshot-swap barrier on `LocalApplyThread` (`raft#543` guard), atomically swaps `db.tmp` $\to$ `db`, records `SnapshotMetadata(snap.Index, snap.Term, snap.ConfState)` in `RaftLogEngine`, and advances `safeRecycleIndex`. |

---

### 3.4 Component 4: Protecting and Improving Read Latency (`rangeKeys` Revision-Order Scan & `applyWait` Bound)

The second part of your objective is: **"...without making read latency meaningfully worse."**

In upstream issue [etcd#22332](https://github.com/etcd-io/etcd/issues/22332), an early unoptimized prototype of Pebble under etcd's MVCC layer (using Pebble's default 8 MiB block cache, Snappy compression on all levels, and $N$ separate iterator allocations in `rangeKeys`) suffered a **5–6.5× read throughput regression** compared to `mmap`-backed `bbolt`. Why did that happen? Because etcd's read path (`storeTxnRead.rangeKeys`, `server/storage/mvcc/kvstore_txn.go:108-151 @ v3.7.3`) does **not** scan user keys in the storage backend:
1. It queries the in-memory B-tree (`revpairs, total := tr.s.kvindex.Revisions(key, end, rev, int(ro.Limit), ro.WithTotalCount)` at `kvstore_txn.go:108 @ v3.7.3`) to get a slice of $N$ `revision` structs (`revpairs []revision`), ordered by **user key**.
2. It then loops `for i, revpair := range revpairs[:len(kvs)]` (`kvstore_txn.go:118-149 @ v3.7.3`) and performs **$N$ separate backend point lookups** by 17-byte `revBytes`:
   ```go
   // server/storage/mvcc/kvstore_txn.go:125-126 @ v3.7.3
   revBytes = RevToBytes(revpair, revBytes)
   _, vs := tr.tx.UnsafeRange(schema.Key, revBytes, nil, 0)
   ```
Because `revpairs` is ordered by user key—not by revision—if `revpairs` contains 2,000 keys updated at different times in the past, `rangeKeys` issues **2,000 random-order point probes** across the storage engine!

With `pebble`, we combine **four concrete read-path optimizations** that eliminate every root cause of the #22332 regression so that both single-key point reads and multi-key range scans match or beat `bbolt`:

1. **Advancing `mvcc.currentRev` Only After `IndexedBatch.Commit(pebble.NoSync)` (Lock-Free MemTable Reads):**
   - In `v3.7.3`, every write `Unlock()` and every 100ms commit acquires `t.backend.readTx.Lock()` (`batch_tx.go:310-314, 356-385 @ v3.7.3`) to copy `txWriteBuffer` into `txReadBuffer` and reopen `bbolt.Tx`, blocking concurrent `ConcurrentReadTx()` calls (`backend.go:279-297 @ v3.7.3`).
   - In our Pebble integration, committing `indexedBatch.Commit(pebble.NoSync)` into Pebble's lock-free MemTable skiplist *before* advancing MVCC's visible `currentRev` allows concurrent `ReadTx` readers to read directly from the Pebble `DB` (`db.Get` / `db.NewIter` or a `pebble.Snapshot`) with **zero `readTx.mu.Lock()` contention and zero write-buffer copying**!
2. **Direct `db.Get()` + 10-Bit Bloom Filters + 1–4 GiB Block Cache for Point Reads (`N = 1`):**
   - When `endKey == nil` in `UnsafeRange` (`kvstore_txn.go:126`, `read_tx.go:79-82 @ v3.7.3`), call `db.Get(prefixedKey)` directly instead of allocating a `pebble.Iterator`.
   - Configure Pebble with `bloom.FilterPolicy(10)` (10 bits per key $\implies <1\%$ false-positive rate) so any SSTable level that does not contain `revBytes` is skipped in $O(1)$ CPU cache hits.
   - Configure a **1–4 GiB Pebble Block Cache** (instead of Pebble's 8 MiB default) with **no compression on L0/L1** (`pebble.NoCompression`) and LZ4 compression on L2+ (`pebble.LZ4Compression`).
3. **Revision-Order Permutation + Single Forward Iterator (`TrySeekUsingNext`) for Multi-Key Range Reads (`N > 1` in `kvstore_txn.go:118-149`):**
   Look at `rangeKeys` (`server/storage/mvcc/kvstore_txn.go:118-149 @ v3.7.3`): the output `kvs[i]` must be in user-key order (`i = 0..len(kvs)-1`), **but the order in which we read `revpairs[i]` from the immutable `ReadTx` view does not matter!**
   When `len(kvs) > 1`:
   - Construct an index permutation `perm := [0, 1, ..., len(kvs)-1]` and sort `perm` so that `revpairs[perm[0]] < revpairs[perm[1]] < ... < revpairs[perm[N-1]]` (monotonically increasing `Revision` order!).
   - Open **one single `pebble.Iterator`** bounded to the `key` bucket prefix and step strictly forward via `iter.SeekGE(revBytes_k)`. Because consecutive `SeekGE` calls are now issued in **strictly ascending key order** (`cmp < 0`), Pebble's `Iterator.SeekGE` **automatically enables `flags.EnableTrySeekUsingNext()`** (`pebble/iterator.go:298-316, 1331-1333 @ v2.1.7`), which advances via fast in-block `Next()` steps whenever `revBytes_k` falls inside the current 32 KiB SSTable data block! Each unmarshaled value is written directly into `kvs[perm[k]]`.
   - **Why this is a massive read-latency win (High confidence):**
     1. **One iterator allocation instead of $N$ lookups.**
     2. **Monotonic block-cache locality + automatic `TrySeekUsingNext`:** Because `schema.Key` in Pebble is sorted by 17-byte `revBytes`, sorting the probes in ascending `Revision` order visits SSTable data blocks in a **single strictly increasing pass**. Whenever multiple live keys were written in the same burst (sharing a 32 KiB SSTable block), Pebble's `TrySeekUsingNext` path serves them from the **already-pinned data block in the iterator** with zero additional index-block searches, zero block-cache lookups, zero bloom-filter checks, and zero block decompressions!
     3. **Zero client API or ordering change:** Because the unmarshaled `KeyValue` for `revpairs[perm[k]]` is written into `kvs[perm[k]]`, the returned `kvs` slice is still in 100% identical user-key order!
4. **Bounding `commitIndex - appliedIndex` Lag to Protect Linearizable Reads (`server/etcdserver/read/read.go:117, 133-138 @ v3.7.3`):**
   - Every linearizable `Range` request confirms `confirmedIndex` via `ReadIndex` (`read.go:117`) and then waits on `<-r.server.ApplyWait(confirmedIndex)` (`read.go:133-138 @ v3.7.3`) when `appliedIndex < confirmedIndex`. If `AsyncStorageWrites` allowed `commitIndex` to run tens of thousands of entries ahead of `appliedIndex`, linearizable reads would stall waiting for `LocalApplyThread` to catch up.
   - Because our `LocalApplyThread` performs **zero `fdatasync`s** (committing batches into Pebble's MemTable with `pebble.NoSync`), apply latency is sub-millisecond; in addition, we bound `LocalApplyThread`'s unapplied entry backlog (e.g., max `4,096` entries in flight) so `ApplyWait` queuing latency remains $< 1\text{ ms}$.

---

## 4. Evaluation of Candidate Libraries for Each Subsystem

You asked whether we should use `pebble`, `grocksdb`, or other alternative libraries for relevant parts of the WAL + storage engine replacement. Below is our assessment of every major library option in the Go/C++/Rust ecosystem:

### 4.1 For the Consensus WAL (`RaftLogEngine`)

| Candidate Library | Language / License | Strengths | Weaknesses / Fit for etcd | Verdict |
|---|---|---|---|---|
| **`github.com/cockroachdb/pebble/record`** (`LogWriter` / `Reader`, inside `pebble v2.1.7`) | Pure Go (BSD/Apache-2.0) | **Already included in `pebble`** (zero extra dependencies); built-in `SyncRecord` group-commit queue; 32 KiB block framing with hardware CRC32C; native **`logNum` file recycling** (`ErrInvalidLogNum`) that eliminates `w.cut()`'s 4-fsync stall and startup `ZeroToEnd`. | Low-level record stream (requires ~450 LoC wrapper to manage the ring of segment files and sparse `Index -> (logNum, offset)` slice). | **Selected (#1 Choice)** (High confidence) |
| **Dragonboat `tan`** (`github.com/lni/dragonboat/v4/internal/tan`) | Pure Go (Apache-2.0) | Purpose-built Go Raft `LogDB` created to replace Pebble for Raft logs; maintains in-memory `index` mapping `RaftIndex -> fileOffset` with segment compaction/GC; actually uses `pebble/record`-derived log primitives internally! | Internal package in `dragonboat/v4` designed around multi-group `(ShardID, ReplicaID)` keys rather than etcd's `wal.WAL` / `serverstorage.Storage` interface; requires extracting/adapting ~2,000 LoC. | **Strong Reference / #2 Choice** (High confidence) |
| **`github.com/tidwall/wal`** (`v1.1.8`) | Pure Go (MIT) | Very small (~600 LoC), clean segmented log indexed by monotonic `uint64` (`WriteBatch`, `Read`, `TruncateFront`, `TruncateBack`); zero-allocation hot path. | Does **not** recycle pre-allocated segment files with `logNum` tags (deletes/creates segment files on truncation) and lacks `LogWriter`'s built-in concurrent `SyncRecord` group-commit queue. | **Good for quick prototyping; inferior to `pebble/record` on segment rotation** (High confidence) |
| **`github.com/hashicorp/raft-wal`** (`v0.4.x`) | Pure Go (MPL-2.0) | Supports segment rotation and tail caching. | Designed for `hashicorp/raft`'s `LogStore`/`StableStore` interfaces; MPL-2.0 license differs from etcd's Apache-2.0; heavier abstraction layer. | **Do not use** |
| **TiKV `raft-engine`** (`github.com/tikv/raft-engine`) | Rust (Apache-2.0, requires C FFI / `cgo`) | Production-proven BitCask-style Raft log engine in TiKV; LZ4 batch compression; recycled log files. | Written in Rust; requires maintaining a custom C FFI + `cgo` binding layer, Rust toolchain in build pipeline, and cross-language memory management. | **Overkill / High `cgo` maintenance cost** |
| **Linux `io_uring` / `O_DIRECT`** (`github.com/iceber/iouring-go` or `O_DIRECT \| O_DSYNC` VFS) | Go + Linux kernel (`7.0.14` on host) | Submitting 4 KiB-aligned `O_DIRECT \| O_DSYNC` writes to pre-allocated recycled segment files bypasses the Linux kernel page cache (`inode->i_rwsem`) and completes durability in **one** NVMe write instead of `write()` + `fdatasync()`. | Requires 4 KiB sector padding on every synced batch; Linux-specific (`O_DIRECT` alignment rules). | **Best pluggable VFS enhancement on top of `pebble/record.LogWriter`** (Moderate confidence) |

### 4.2 For the State Machine Backend (`MVCC + Leases + Auth + Membership`)

| Candidate Library | Language / License | Why It Wins or Loses in the Full WAL + Backend Redesign | Verdict |
|---|---|---|---|
| **`github.com/cockroachdb/pebble`** (`v2.1.7`) | Pure Go (BSD/Apache-2.0) | Pure Go (zero `cgo` tax on $N$ point lookups in `rangeKeys`); `pebble.IndexedBatch` replaces both `bbolt.Tx` and `txBuffer` (fixing PR #17119's `UnsafeDelete` `fdatasync` stall); supports `DisableWAL: true` + `EventListener.FlushEnd` for true single-WAL execution; includes `pebble/record.LogWriter` for the Raft WAL in the same module; hard-link `Checkpoint` in `<1ms`. | **Selected (#1 Choice)** (High confidence) |
| **`github.com/linxGnu/grocksdb`** (`v1.11.1` / RocksDB C++) | C++17 via `cgo` (Apache-2.0 / GPLv2) | Supports `WriteOptions.DisableWAL(true)` per batch while keeping `DeleteRange` working, and C++ compaction threads run outside Go GC. **However:** incurs $O(N)$ `cgo` calls per `Range` query, requires manual `.Destroy()` memory management across every slice/iterator/batch, and breaks pure-Go static builds. | **#2 Choice (Only if off-heap C++ compaction is strictly required)** (High confidence) |

---

## 5. Side-by-Side Quantitative & Architectural Comparison

Let us compare the exact I/O and CPU operations performed on the critical write path across **four progression levels**:
1. **Baseline (`etcd v3.7.3`):** `wal.WAL` + `bbolt v1.5.0`
2. **Stage 1 (`storage-agy.md`):** `wal.WAL` + `pebble v2.1.7` (Backend-only replacement)
3. **Stage 2 (Proposed Option B / Mode C2 Default):** `AsyncStorageWrites` + `RaftLogEngine` (`pebble/record` + `FormatWALSyncChunks`) + `pebble` (`pebble.NoSync` WAL + background `db.LogData(nil, pebble.Sync)` barrier)
4. **Stage 3 (Proposed Option C / Mode C1 Opt-In):** `AsyncStorageWrites` + `RaftLogEngine` (`pebble/record` + `FormatWALSyncChunks`) + `pebble` (`DisableWAL: true` + `db.Flush()` & `db.Checkpoint()` rollover)

| Metric / Operation | Baseline (`etcd v3.7.3`) | Stage 1: Replace `bbolt` Only (`pebble`) | Stage 2 (Mode C2 Default): `RaftLogEngine` + `pebble.NoSync` | Stage 3 (Mode C1 Opt-In): `RaftLogEngine` + `DisableWAL` Pebble |
|---|---|---|---|---|
| **Consensus / Disk Pipelining** | **None (Synchronous):** `raftNode` blocks on `r.applyc` and `WAL.Save` `fdatasync` | **None (Synchronous):** `raftNode` still blocks on `r.applyc` and `WAL.Save` `fdatasync` | **Full Pipelining (`AsyncStorageWrites`):** `raftNode` never blocks on disk or apply; automatic multi-batch group commit | **Full Pipelining (`AsyncStorageWrites`):** `raftNode` never blocks on disk or apply; automatic multi-batch group commit |
| **WAL Segment Rotation (`64 MiB`) & `fdatasync` Extent Mode** | **3–4 synchronous `fdatasync`/`fsync`s** (`w.cut()`) + `ext4`/`XFS` unwritten-extent journal commit on every `fdatasync` into `fallocate`d blocks | **3–4 synchronous `fdatasync`/`fsync`s** (`w.cut()`) + `ext4`/`XFS` unwritten-extent journal commit on every `fdatasync` | **Zero extra `fsync`s & zero unwritten extents:** `Seek(0)` on pre-written recycled `.log` file with `logNum + 1` | **Zero extra `fsync`s & zero unwritten extents:** `Seek(0)` on pre-written recycled `.log` file with `logNum + 1` |
| **Protobuf Marshals & Copies per Entry in Raft WAL** | **2 marshals + 3 copies** (`raftpb.Entry` $\to$ `walpb.Record` $\to$ `PageWriter` $\to$ OS) | **2 marshals + 3 copies** | **1 marshal + 1 copy** (`MarshalAppend` into flat batch $\to$ `LogWriter` 32 KiB block) | **1 marshal + 1 copy** (`MarshalAppend` into flat batch $\to$ `LogWriter` 32 KiB block) |
| **Number of Logs Written per `Put`** | **2** (Raft WAL + `bbolt` B+tree pages) | **2** (Raft WAL + Pebble WAL) | **2** (Synced Raft WAL + Unsynced Pebble WAL; **1 `fdatasync` total**) | **1 (Raft WAL only!)** |
| **Syscalls on `LeaseRevoke` (`UnsafeDelete`) in `applyEntries`** | **2x `fdatasync` + random 4 KiB writes** on **every single `LeaseRevoke`** (`batch_tx.go:335`) | **0** (appended to `IndexedBatch` in RAM) | **0** (appended to `IndexedBatch` in RAM) | **0** (appended to `IndexedBatch` in RAM) |
| **Syscalls on 100ms Backend Commit (`batchTx.Commit`)** | **2x `fdatasync`** + random 4 KiB page writes while holding `batchTx.Lock` & `readTx.mu.Lock` | **1 buffered `write()`** (`pebble.NoSync`, **0 `fdatasync`**) | **1 buffered `write()`** (`pebble.NoSync`, **0 `fdatasync`**) | **0 syscalls (`0 write()`, `0 fdatasync()`)** — pure in-memory MemTable skiplist insert! |
| **Every-10,000-Entry Checkpoint (`EtcdServer.snapshot`)** | **3 synchronous `fdatasync`s** on `applyAll` goroutine (`KV.Commit` + `.snap` + `WAL.SaveSnapshot`) | **3 synchronous `fdatasync`s** on `applyAll` goroutine | **0 synchronous stalls on `applyAll`** (driven asynchronously by background `db.LogData(nil, pebble.Sync)`) | **0 synchronous stalls on `applyAll`** (driven asynchronously by background `db.Flush()` + `db.Checkpoint()`) |
| **Space Reclamation (`Defrag`)** | **Stop-the-world `Defrag()`** (freezes all reads/writes) | Continuous background LSM compaction | Continuous background LSM compaction | Continuous background LSM compaction |
| **Multi-Key `Range` Read (`rangeKeys` over $N$ keys)** | $N$ random `mmap` B+tree probes (stalls if DB $>$ RAM or during 100ms `readTx.mu` lock) | $N$ random Pebble point probes | **1 forward-seeking `pebble.Iterator` pass** (`TrySeekUsingNext`) over sorted revision permutation (zero `readTx.mu` lock) | **1 forward-seeking `pebble.Iterator` pass** (`TrySeekUsingNext`) over sorted revision permutation (zero `readTx.mu` lock) |

### Expected Performance Impact (Moderate–High Confidence)

1. **Sustained Write Throughput (`Put` / `DeleteRange` / `Txn`):**
   - **Small-to-medium datasets (< 2 GiB) under high concurrency:** Replacing the synchronous `raftNode` loop + `wal.WAL` with `AsyncStorageWrites` + `RaftLogEngine` (`pebble/record.LogWriter` group commit + pre-written recycled segments that eliminate `ext4`/`XFS` unwritten-extent journal commits and `w.cut()` 4-fsync rotation stalls) + eliminating the 10,000-entry triple-`fdatasync` stall in `applyAll` is projected to improve sustained write throughput by **$1.5\times\text{–}2.5\times$** and reduce `p99` write latency by **$3\times\text{–}5\times$** (Moderate confidence; fails if network RTT between geo-distributed nodes rather than local disk/pipeline serialization dominates).
   - **Large aged datasets (2–16+ GiB) with active MVCC compaction or high lease churn (`LeaseRevoke`):** Combining `RaftLogEngine` + `AsyncStorageWrites` with `pebble` (`pebble.NoSync` or `DisableWAL`) eliminates both the per-`LeaseRevoke` synchronous `bbolt` `fdatasync` (`batch_tx.go:335 @ v3.7.3`) and `bbolt`'s random 4 KiB CoW page-spill stalls, yielding an expected **$3\times\text{–}10\times$ improvement in sustained write throughput** and **$10\times\text{–}20\times$ reduction in `p99` latency** compared to `etcd v3.7.3` (High confidence).
2. **Read Latency:**
   - **Single-key point reads:** Within **$\pm 10\%$** of in-memory `bbolt` at `p50`, and **significantly better at `p99`/`p99.9`** because `ConcurrentReadTx()` no longer blocks behind `bbolt`'s 100ms `readTx.mu.Lock()` commit window or major `mmap` page faults (High confidence).
   - **Multi-key `Range` queries ($N = 100..10,000$ keys):** With the Revision-Order Permutation + single forward `pebble.Iterator` (automatic `TrySeekUsingNext`) optimization (§3.4), `Range` latency on Pebble is projected to be **equal to or $1.2\times\text{–}2\times$ faster than `bbolt`** on aged databases where revisions from the same write burst share 32 KiB SSTable data blocks (Moderate confidence).

---

## 6. Touch Surface, Effort Estimation, and Phased Implementation Plan (**High confidence** on touch surface; **Moderate confidence** on LoC and schedule estimates)

Because `server/storage/wal` is only imported by **5 non-test production files** in `server/` (`server/storage/storage.go`, `server/storage/schema/schema.go`, `server/verify/verify.go`, `server/etcdserver/bootstrap.go`, and `server/embed/util.go`, plus test helper `server/storage/wal/testing/waltesting.go`), the code touch surface for replacing the WAL is surprisingly well-contained (**High confidence**).

| Phase / Subsystem | Key Files Modified or Added (`@ v3.7.3`) | Estimated LoC | Estimated Effort | Can Be Shipped & Tested Independently? |
|---|---|---|---|---|
| **Phase 1: `pebble` State-Machine Backend + Sorted-Revision `rangeKeys`** (from `storage-agy.md` + §3.4) | `server/storage/backend/pebble_*.go`, `server/storage/backend.go`, `server/storage/mvcc/kvstore_txn.go:118-149` | ~2,000 LoC | 6–8 wks | **Yes.** Drops cleanly behind `backend.Backend`. |
| **Phase 2: `AsyncStorageWrites` (with `raft#551`/`#543` guards) + Asynchronous Checkpoint Fencing** (§3.1 & §3.3) | `server/etcdserver/raft.go:174-342`, `server/etcdserver/bootstrap.go:540`, `server/etcdserver/server.go:972-993, 1215-1228, 2083-2137` | ~500 LoC | 2–3 wks | **Yes.** Can be enabled even before replacing `wal.WAL` to immediately pipeline Raft network replication, WAL writes, and MVCC apply! |
| **Phase 3: `RaftLogEngine` (`pebble/record.LogWriter` Recycled WAL + `WriteWALSyncOffsets` + Tail Ring)** (§3.2 & §3.3) | `server/storage/raftlog/*.go` (new), `server/storage/storage.go`, `server/etcdserver/bootstrap.go:413-749`, `server/verify/verify.go` | ~1,150 LoC | 3–4 wks | **Yes.** Implements `serverstorage.Storage` (`Save`, `SaveSnap`, `Release`, `Sync`, `MinimalEtcdVersion`). |
| **Phase 4: Offline CLI (`etcdutl`, `etcd-dump-logs`) & LazyFS / Porcupine Robustness Suite** | `etcdutl/`, `tools/etcd-dump-logs/`, `tests/robustness/report/wal.go:35, 368`, `tests/robustness/failpoint/gofail.go:38-39, 53-59` | ~500 LoC | 1.5–2 wks | **Yes.** |
| **Total (Full WAL + Storage Engine Replacement)** | | **~4,150 LoC** | **13–17 engineer-weeks** | |

### Recommended Strategic Sequence & Verification Plan (High Confidence)

Do **not** attempt a 4,000-line big-bang rewrite before running three targeted validation steps on your target hardware:

1. **Step 1 — Validate the Ceiling with the `tmpfs` Matrix & Filesystem `fdatasync` Microbenchmark (Zero etcd Code Changes, 1 Day on Target Hardware):**
   - **1A (`2 x 2 tmpfs` Ceiling Matrix):** Extend **Experiment 3** from [storage-agy.md](storage-agy.md#L422-L441) into a $2 \times 2$ matrix using `tools/benchmark` on an aged 6 GiB dataset:
     - **Run 1 (Baseline):** `member/wal` on Disk, `member/snap` on Disk.
     - **Run 2 (Backend Ceiling):** `member/wal` on Disk, `member/snap` on `tmpfs`.
     - **Run 3 (WAL Ceiling):** `member/wal` on `tmpfs`, `member/snap` on Disk.
     - **Run 4 (Combined Ceiling — Pure Raft + MVCC CPU/Network Bound):** Both `member/wal` and `member/snap` on `tmpfs` (or with `--unsafe-no-fsync`).
     - *Why this 4-run matrix is decisive:* Run 2 tells you the exact gain from replacing `bbolt` alone; Run 3 tells you the exact gain from optimizing the WAL alone; Run 4 tells you the **combined ceiling** of replacing both the WAL and `bbolt` (while keeping the current synchronous `raftNode` loop).
   - **1B (`fallocate` Unwritten-Extent vs. Pre-Written Recycled Segment Microbenchmark):** Run a standalone 50-line Go/fio microbenchmark on your target host's filesystem (`ext4` vs. `XFS`) comparing 4 KiB–16 KiB `write + fdatasync` latency into (i) a freshly `fallocate(mode 0)`d 64 MiB file (`v3.7.3` `filePipeline` behavior) vs. (ii) a pre-written recycled 64 MiB file (`RaftLogEngine` `logNum` behavior), both alone and while a second goroutine writes unsynced 64 KiB batches to a separate file on the same filesystem (testing `ext4 data=ordered` cross-file journal coupling per [cockroach#16624](https://github.com/cockroachdb/cockroach/issues/16624)).

2. **Step 2 — Prototype Phase 2 (`AsyncStorageWrites` + Removing the 10,000-Entry `applyAll` Snapshot Stall) + Phase 1 PoC (~1,000 LoC, 2–3 Weeks):**
   - Notice that **Phase 2** (enabling `Config.AsyncStorageWrites = true` in `bootstrap.go:540` with the `raft#551`/`#543` guards, decoupling `r.applyc` in `raft.go:232`, and replacing the synchronous 3-`fdatasync` `s.snapshot(ep, true)` stall every 10,000 entries in `server.go:1215` with background `db.LogData(nil, pebble.Sync)`) is only **~500 lines of code** and works with **both** the existing WAL and the new `RaftLogEngine`!
   - Combined with the minimal `pebble` backend PoC (`pebble.NoSync`) and `pebble/record.LogWriter`, you can benchmark the full end-to-end architecture on your target hardware in ~3 weeks of prototyping before investing in offline CLI tooling (`etcdutl`) and migration utilities.

3. **Step 3 — Power-Loss & Crash-Consistency Verification Using Existing `LazyFS` Robustness Harness and `vfs.NewCrashableMem()`:**
   - **Unit/Component level:** Test `RaftLogEngine` (`WriteWALSyncOffsets`) and the `pebble.NoSync` + `db.LogData(nil, pebble.Sync)` checkpoint coordinator under Pebble's deterministic crash simulator **`vfs.NewCrashableMem()`**, which drops arbitrary unsynced writes and directory operations at random I/O boundaries and verifies that reopening `pebble` + replaying `RaftLogEngine` always recovers the exact committed prefix.
   - **Cluster/E2E level:** Run etcd's existing **LazyFS** power-loss robustness suite (`tests/robustness/scenarios/scenarios.go:154-166 @ v3.7.3` and `tests/robustness/failpoint/kill.go:51-57 @ v3.7.3`, which calls `lazyfs.ClearCache(ctx)` to discard unsynced kernel page-cache writes upon simulated node crash) together with Porcupine linearizability verification.

---

## 7. Open Questions

1. **Kernel & Storage Stack on Target Hardware:** Does your production environment run Linux $\ge 5.15$ on local NVMe SSDs (where `O_DIRECT | O_DSYNC` or `io_uring` aligned writes on pre-allocated recycled `pebble/record` segments can cut WAL sync latency in half), or on network-attached block storage (EBS / Ceph / SAN, where group-commit batch depth via `AsyncStorageWrites` + `LogWriter.SyncRecord` is the primary lever)?
2. **Follower Catch-Up Window:** When a follower restarts or experiences a brief network partition in your environment, how many writes typically occur during the outage? Sizing `RaftLogEngine`'s pre-allocated recycled segment ring to e.g. `8 x 64 MiB = 512 MiB` allows followers to catch up across hundreds of thousands of writes via fast log streaming without ever triggering a full multi-gigabyte `MsgSnap` snapshot transfer.
