# Watchers: Raft Bypass, Leader Failover, API Promises, and Client/Server Failure States

- **History:** revised 2026-11-01 for exposition clarity and cross-validated against `etcd v3.7.3` and `jetcd 0.8.7`
- **Versions analyzed:** etcd `v3.7.3` (`0a173d0410a9`); jetcd `jetcd-0.8.7` (`23b61f1499a5`); raft `go.etcd.io/raft/v3` `v3.7.0`; selected Kubernetes `v1.37.2` storage/cacher files; selected upstream `main` commits cited explicitly where relevant
- **Upstream references:**
  - **etcd issues & PRs (Watch correctness, failover, progress, proxy, and Raft/ReadIndex bypass):**
    [#5324](https://github.com/etcd-io/etcd/pull/5324),
    [#5328](https://github.com/etcd-io/etcd/pull/5328),
    [#5332](https://github.com/etcd-io/etcd/pull/5332),
    [#5354](https://github.com/etcd-io/etcd/issues/5354),
    [#7247](https://github.com/etcd-io/etcd/issues/7247),
    [#7896](https://github.com/etcd-io/etcd/pull/7896),
    [#8411](https://github.com/etcd-io/etcd/issues/8411),
    [#8673](https://github.com/etcd-io/etcd/issues/8673),
    [#8980](https://github.com/etcd-io/etcd/issues/8980),
    [#9872](https://github.com/etcd-io/etcd/pull/9872),
    [#10289](https://github.com/etcd-io/etcd/issues/10289),
    [#10422](https://github.com/etcd-io/etcd/issues/10422),
    [#10681](https://github.com/etcd-io/etcd/issues/10681),
    [#10819](https://github.com/etcd-io/etcd/issues/10819),
    [#12022](https://github.com/etcd-io/etcd/issues/12022),
    [#12254](https://github.com/etcd-io/etcd/issues/12254),
    [#12534](https://github.com/etcd-io/etcd/issues/12534),
    [#12991](https://github.com/etcd-io/etcd/issues/12991),
    [#13822](https://github.com/etcd-io/etcd/issues/13822),
    [#14303](https://github.com/etcd-io/etcd/issues/14303),
    [#14699](https://github.com/etcd-io/etcd/issues/14699),
    [#15220](https://github.com/etcd-io/etcd/issues/15220),
    [#15237](https://github.com/etcd-io/etcd/pull/15237),
    [#15271](https://github.com/etcd-io/etcd/issues/15271),
    [#15402](https://github.com/etcd-io/etcd/issues/15402),
    [#15492](https://github.com/etcd-io/etcd/pull/15492),
    [#15820](https://github.com/etcd-io/etcd/issues/15820),
    [#16839](https://github.com/etcd-io/etcd/issues/16839),
    [#17247](https://github.com/etcd-io/etcd/issues/17247),
    [#17475](https://github.com/etcd-io/etcd/issues/17475),
    [#17507](https://github.com/etcd-io/etcd/issues/17507),
    [#17508](https://github.com/etcd-io/etcd/issues/17508),
    [#17529](https://github.com/etcd-io/etcd/issues/17529),
    [#17555](https://github.com/etcd-io/etcd/pull/17555),
    [#17611](https://github.com/etcd-io/etcd/issues/17611),
    [#17622](https://github.com/etcd-io/etcd/pull/17622),
    [#17791](https://github.com/etcd-io/etcd/issues/17791),
    [#18089](https://github.com/etcd-io/etcd/issues/18089),
    [#18274](https://github.com/etcd-io/etcd/pull/18274),
    [#18879](https://github.com/etcd-io/etcd/issues/18879),
    [#19179](https://github.com/etcd-io/etcd/issues/19179),
    [#19188](https://github.com/etcd-io/etcd/pull/19188),
    [#19487](https://github.com/etcd-io/etcd/issues/19487),
    [#19710](https://github.com/etcd-io/etcd/issues/19710),
    [#20221](https://github.com/etcd-io/etcd/issues/20221),
    [#20229](https://github.com/etcd-io/etcd/pull/20229),
    [#20240](https://github.com/etcd-io/etcd/pull/20240),
    [#20242](https://github.com/etcd-io/etcd/issues/20242),
    [#20281](https://github.com/etcd-io/etcd/pull/20281),
    [#20349](https://github.com/etcd-io/etcd/issues/20349),
    [#20418](https://github.com/etcd-io/etcd/issues/20418),
    [#20573](https://github.com/etcd-io/etcd/issues/20573),
    [#20716](https://github.com/etcd-io/etcd/issues/20716),
    [#21065](https://github.com/etcd-io/etcd/pull/21065),
    [#21399](https://github.com/etcd-io/etcd/pull/21399),
    [#21813](https://github.com/etcd-io/etcd/issues/21813),
    [#22184](https://github.com/etcd-io/etcd/issues/22184),
    [#22264](https://github.com/etcd-io/etcd/issues/22264),
    [#22268](https://github.com/etcd-io/etcd/issues/22268),
    [#22294](https://github.com/etcd-io/etcd/issues/22294),
    [#22614](https://github.com/etcd-io/etcd/issues/22614),
    [#22635](https://github.com/etcd-io/etcd/issues/22635),
    [#22673](https://github.com/etcd-io/etcd/issues/22673)
  - **jetcd issues & PRs:**
    [#946](https://github.com/etcd-io/jetcd/issues/946),
    [#1055](https://github.com/etcd-io/jetcd/issues/1055),
    [#1227](https://github.com/etcd-io/jetcd/issues/1227),
    [#1352](https://github.com/etcd-io/jetcd/issues/1352),
    [#1379](https://github.com/etcd-io/jetcd/issues/1379),
    [#1393](https://github.com/etcd-io/jetcd/issues/1393),
    [#1616](https://github.com/etcd-io/jetcd/issues/1616),
    [#1617](https://github.com/etcd-io/jetcd/pull/1617)
  - **Kubernetes issues & KEPs:**
    [kubernetes/kubernetes#59848](https://github.com/kubernetes/kubernetes/issues/59848),
    [kubernetes/kubernetes#76624](https://github.com/kubernetes/kubernetes/issues/76624),
    [kubernetes/kubernetes#89488](https://github.com/kubernetes/kubernetes/issues/89488),
    [kubernetes/kubernetes#89881](https://github.com/kubernetes/kubernetes/pull/89881),
    [kubernetes/kubernetes#111116](https://github.com/kubernetes/kubernetes/issues/111116),
    [kubernetes/kubernetes#123072](https://github.com/kubernetes/kubernetes/issues/123072),
    [kubernetes/kubernetes#123448](https://github.com/kubernetes/kubernetes/issues/123448),
    [kubernetes/kubernetes#123513](https://github.com/kubernetes/kubernetes/issues/123513),
    [KEP-2340 (Consistent Reads from Cache)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2340-Consistent-reads-from-cache)

---

## Scope

This document evaluates the architecture, correctness guarantees, and failure modes of the `etcd` (`v3.7.3`) Watch subsystem and its official Go (`clientv3 v3.7.3`) and Java (`jetcd 0.8.7`) client libraries. It examines how Watch interacts with Raft consensus across leader failovers, network partitions, follower lag, and history compaction, and compares etcd's design against Google's Chubby lock service (`doc/chubby-osdi06.md`) and Kubernetes's production control-plane watch cache (`v1.37.2`).

---

## Summary

### Executive Summary

A common architectural suspicion is that etcd Watch, like etcd Lease renewal, bypasses Raft consensus and therefore loses state during leader failovers. Source verification shows that Watch splits its architecture across two planes with very different properties:

1. **The Mutation Plane (Raft-Backed):** Every key-value mutation (`Put`, `DeleteRange`, `Txn`, `LeaseRevoke`, `Compact`) is committed through Raft and stored in each member's local Multi-Version Concurrency Control (`bbolt`) database under a cluster-wide monotonic `Revision`. Unlike Leases, etcd does not need to replicate watcher registrations through Raft for historical events to survive a leader failover, provided the client knows the exact revision at which to resume.
2. **The Stream Control & Bookmark Plane (Raft-Bypassing):** Watch streams themselves can terminate on any cluster member (leader or follower) or `grpcproxy` without Raft coordination or linearizable `ReadIndex`. Watch creation at the "current" revision (`startRev == 0`), progress notifications (`RequestProgress` / `ProgressNotify`), future-revision watch classification, and stream liveness all read the serving replica's **unfenced local state**.

This split creates two structural categories of production failures:
- **Contract Gaps (Silent Staleness in a Quorum-Healthy Cluster):** Etcd's specification guarantees ordering and completeness of delivered events (*Ordered, Unique, Atomic, Reliable, Resumable, Bookmarkable*), but disclaims bounded delivery time (`etcd-io/website` `content/en/docs/v3.7/learning/api_guarantees.md`: *"there is no limit and events in unhealthy clusters might never arrive"*). While that doc phrases the disclaimer as *"unhealthy clusters"*, the critical failure occurs when **the cluster HAS a healthy majority quorum and is committing writes normally**: by default, a watch connected to a single partitioned follower or deposed leader (1 of 3 nodes) hangs silently forever while receiving `ProgressNotify` messages with the isolated node's frozen revision. Opting into `WithRequireLeader` does not solve this cleanly: it fails across nine distinct client, proxy, and server code paths—including destroying the client's watch channel instead of failing over to the surviving quorum, discarding the internal resumption revision on `startRev == 0` watches, and being completely ignored by `grpcproxy`.
- **Direct Contract Violations (Event Loss, Contamination, and Deadlocks):** Because watch continuity across failovers depends entirely on the client and server maintaining a consistent revision cursor, subtle races and state-machine bugs cause silent event loss or duplication. Most critically, Java `jetcd` (`0.8.7`) overwrites its resumption cursor with the server's tip revision (`Header.Revision`) upon receiving the initial `Created` message, silently dropping all historical events in `[startRev, serverRev - 1]` if the stream reconnects before catch-up completes. On the server side, `etcd v3.7.3` contains open bugs that suppress `ProgressNotify` cluster-wide whenever any single watcher is unsynced (`#20242`), leak or cross-contaminate events from canceled watches (`#22673`), leak compacted watchers when $\ge 512$ watchers are unsynced (`#22294`), scan unbounded `bbolt` history under the exclusive lock blocking Raft apply (`#22264`), and actively migrate healthy watchers onto dead broadcasts in `grpcproxy` (`#21813`).

### Summary of Claims

1. **[High confidence — Verified in code & docs]** **Watched key-value mutations go through Raft, but Watch stream operations (`WatchCreateRequest`, `WatchProgressRequest`, `ProgressNotify`, and stream liveness) bypass Raft and linearizable `ReadIndex`—creating a direct conceptual contradiction between etcd's claim of "eventual consistency for Watch" (`tests/robustness/README.md:102, 111`) and its disclaimer that events "might never arrive" (`api_guarantees.md:38`), even when the cluster maintains a healthy quorum.**
   Every watched write is committed via Raft and applied deterministically to each replica's local MVCC `bbolt` store. However, because any replica can serve a watch stream from its unsynchronized local MVCC state without `ReadIndex`, a watch connected to a single partitioned or apply-stalled member in a quorum-healthy cluster can go permanently silent without notifying the client. In distributed systems theory (and etcd's own definition at `tests/robustness/README.md:102`), **eventual consistency** requires that components *"converge to the same view eventually"* (a liveness guarantee). A watch stream whose events *"might never arrive"* while the cluster has quorum provides only **prefix safety** (if events arrive, they are ordered and gap-free), not eventual consistency—negating the core utility of a push-based watch primitive and forcing production clients to bolt on periodic polling (linearizable `KV.Get` `ReadIndex` checks or scheduled watch restarts) to guarantee liveness.
   - *Evidence:* `server/storage/mvcc/watchable_store_txn.go:22-47`, `server/etcdserver/api/v3rpc/watch.go:340-400, 558-570`, `server/storage/mvcc/watchable_store.go:118-167, 507-540`, `tests/robustness/README.md:100-111`, `client/v3/watch.go:78-84 @ v3.7.3`; `etcd-io/website` `content/en/docs/v3.7/learning/api_guarantees.md:36-38`, `design-client.md:48, 80, 106, 136`; `kubernetes/kubernetes#59848`.
   - *Condition under which this claim fails:* None for `v3.7.3`; verified directly in server source and specification docs.

2. **[High confidence — Verified in code & issues]** **`WithRequireLeader` is off by default and, when enabled, fails to preserve Watch continuity and liveness across nine distinct client, proxy, and server paths.**
   Without `WithRequireLeader`, a watch on a partitioned member never closes and continues receiving `ProgressNotify` at the isolated node's frozen revision (`#17611`). With `WithRequireLeader`, `monitorLeader` cancels existing streams after 3 election timeouts with `s.Leader() == raft.None` (~3–5 s under default settings), but exposes nine verified failure modes (detailed in §3.1):
   1. *Terminal watch destruction instead of failover* (`clientv3` and `jetcd` treat `ErrNoLeader` as fatal rather than failing over to the surviving quorum).
   2. *Silent loss of the internal resumption bookmark (`ws.initReq.rev`)* on `startRev == 0` watches when `closeSubstream` emits `Header.Revision == 0`.
   3. *Pinned `READY` gRPC `SubConn` and ineffective retry loop* in `openWatchClient()` because `ErrGRPCNoLeader` arrives in HTTP/2 trailers rather than closing the TCP connection.
   4. *Silent discard of buffered events (`ws.buf`) and drop of `ErrNoLeader`* if the 1-slot `ws.outc` channel is full for $> 250\text{ ms}$.
   5. *Permanent client goroutine deadlock* in `w.wg.Wait()` when `ErrNoLeader` races with a rejected/canceled `Created` response (`#17611` / `#17622`).
   6. *Silent failure of `RequestProgress(ctx)`* when called without `WithRequireLeader(ctx)` due to stream-key partitioning (`#12254`).
   7. *Server and transport blind spots* during apply-loop stalls (`s.Leader() != raft.None`), process pauses (`DialKeepAliveTime == 0` in `clientv3`), or total client isolation.
   8. *Complete failure in `grpcproxy`*, which reads `metadata.FromOutgoingContext` instead of `FromIncomingContext` on server streams and places its internal `__lostleader` watch on a separate gRPC stream from data watches.
   9. *Zero grace period in `newStreamInterceptor`*, which immediately rejects reconnecting streams with `ErrGRPCNoLeader` if they land on a surviving member during a normal 1-second leader election.
   - *Evidence:* `server/etcdserver/api/v3rpc/interceptor.go:239-258, 315-353`, `server/proxy/grpcproxy/watch.go:112`, `server/proxy/grpcproxy/leader.go:42-80`, `client/v3/watch.go:478-541, 667, 838-872, 1039-1062 @ v3.7.3`; `WatchImpl.java:354 @ jetcd-0.8.7`; `#8980`, `#12254`, `#12534`, `#17611`, `kubernetes/kubernetes#111116`.
   - *Condition under which this claim fails:* None for `v3.7.3` / `jetcd-0.8.7`; each path is verified directly in source code.

3. **[High confidence — Verified in code]** **Opening a watch at the "current" revision (`startRev == 0`) or disconnecting immediately after `Created` can either replay pre-existing events or silently drop committed events.**
   `watchableStore.watch()` binds `startRev == 0` to the replica's local `s.store.currentRev + 1` ($R_1 + 1$) without `ReadIndex`, and `serverWatchStream.recvLoop` subsequently reads `sws.watchStream.Rev()` ($R_2 \ge R_1$) outside the lock to populate `Created.Header.Revision`. If concurrent writes commit between those two reads ($R_2 > R_1$) and the stream disconnects after `Created`, `clientv3` resumes at `nextRev = R_2`, silently dropping events in $[R_1 + 1, R_2 - 1]$. Conversely, if no writes intervene ($R_2 == R_1$), `clientv3` sets `nextRev = R_1` without adding `+ 1` (mistakenly treating `start_revision` as exclusive), replaying revision $R_1$ on reconnect.
   - *Evidence:* `server/storage/mvcc/watchable_store.go:141-147`, `server/etcdserver/api/v3rpc/watch.go:340, 358`, `client/v3/watch.go:849-860, 871`, `api/etcdserverpb/rpc.proto:784 @ v3.7.3`.
   - *Condition under which this claim fails:* Requires `startRev == 0` (no explicit `WithRev`) and a stream disconnect between the `Created` message and the first delivered event.

4. **[High confidence — Verified in code & issues]** **Reconnecting a watch to a lagging follower (`startRev > s.store.currentRev`) and member-local recovery paths have repeatedly broken Watch safety guarantees, and the `v3.7.3` fix (`#20229`) introduces server-wide `ProgressNotify` starvation (`#20242`) and an `mvcc` lock-order inversion (`#22184`).**
   Prior to `v3.7.0`, placing future-revision watches into `s.synced` caused lagging followers to emit stale `ProgressNotify` revisions that rewound `clientv3`'s `nextRev` cursor (`#20221`). Commit `973847aa2` (`#20229`, in `v3.7.3`) suppresses progress when `rev < w.startRev`, but checks server-wide `s.unsynced`, `s.synced`, and `s.victims` maps while holding `watchableStore.mu.RLock()` before acquiring `store.mu.RLock()`. This starves progress notifications across the entire server whenever any single watcher is catching up (`#20242`) and inverts the documented lock order (`#22184`). **Qualification verified in `v3.7.3`:** While the lock inversion in `progressIfSync()` is real at the `mvcc` layer, issue `#22184`'s specific production trigger (`createMergedSnapshotMessage` calling `s.KV().Commit()` concurrently with `Write()`) does not occur in `EtcdServer` at `v3.7.3` because `applyAll()` serializes `Restore`, `Write`, `Compact`, and `Commit` on the single `s.applySched` FIFO goroutine.
   - *Evidence:* `server/storage/mvcc/watchable_store.go:59-60, 141, 374, 486, 515-539`, `server/etcdserver/server.go:972-993, 1062-1069, 2101-2105 @ v3.7.3`; `#8411`, `#15220`, `#15271`, `#17529`, `#18089`, `#19179`, `#19487`, `#20221`, `#20229`, `#20240`, `#20242`, `#22184`.
   - *Condition under which this claim fails:* Triggering the `#22184` deadlock in `EtcdServer` requires a caller outside `s.applySched` to acquire `store.mu.Lock()` concurrently with `watchableStoreTxnWrite.End()` and `progressIfSync()`.

5. **[High confidence — Verified in code & issues]** **Multiple open server, proxy, and protocol bugs in `etcd v3.7.3` cause silent event omission, cross-watch event contamination, stream deadlocks, or unbounded apply-loop stalls.**
   Specifically: (a) `sendLoop` retains `pending[wid]` across watch cancellation, leaking events or flushing them onto a reused `WatchId` (`#22673`); (b) `WithPrevKV()` silently omits `PrevKv` when `ModRevision - 1` is compacted (`#10681`); (c) `watcherGroup.choose()` deletes compacted watchers from a temporary copy instead of `s.unsynced` when $\ge 512$ watchers are unsynced (`#22294`); (d) `syncWatchers()` scans unbounded `[minRev, curRev+1)` revisions under `watchableStore.mu.Lock()` while delivering at most 1,000 (`#22264`); (e) burst watch cancellations deadlock `clientv3` and `serverWatchStream` in a 4-way channel cycle (`#18879`, `#20716`); (f) `grpcproxy` resets `wb.nextrev = 1` on upstream cancel, drops the `Canceled` response, and actively coalesces healthy broadcasts onto the dead broadcast (`#21813`); (g) `etcd/cache` permanently wedges as not-ready after compaction resync (`#22614`); and (h) `etcdutl snapshot restore` defaults to `--bump-revision=0`, turning resumed watches into silent future watches (`#14303`).
   - *Evidence:* `server/etcdserver/api/v3rpc/watch.go:42, 122, 383, 450-456, 470-477, 519, 534-553`, `server/storage/mvcc/watcher_group.go:28, 57-60, 226-269`, `server/storage/mvcc/watchable_store.go:346-414`, `server/proxy/grpcproxy/watch_broadcast.go:84`, `server/proxy/grpcproxy/watcher.go:107`, `server/proxy/grpcproxy/watch_broadcasts.go:67-79`, `cache/cache.go:349-369, 405-421`, `etcdutl/etcdutl/snapshot_command.go:83-84 @ v3.7.3`.
   - *Condition under which this claim fails:* Each bug has an exact trigger condition documented in §4.1–§4.8.

6. **[High confidence — Verified in code]** **Java `jetcd` (`jetcd-0.8.7`) contains a critical revision-tracking bug in `WatchImpl.java` that silently drops all historical events `[startRev, serverCurrentRev - 1]` if a watch stream disconnects after receiving `Created`.**
   When a watch is opened at `startRevision = R` against a server at revision $H > R$, the server immediately sends `Created: true` with `Header.Revision = H` while queueing the watcher in `s.unsynced` to stream events $[R, H]$ asynchronously. Upon receiving `Created`, `jetcd` executes `revision = Math.max(revision, response.getHeader().getRevision())`, jumping `this.revision` from $R$ to $H$ before any historical events arrive. If the stream disconnects before catch-up completes, `jetcd` reconnects with `start_revision = H`, silently losing $[R, H - 1]$. Additionally, `jetcd` has an off-by-one bug on `ProgressNotify` (missing `+ 1`), abandons the watcher if `resume()` throws inside `reschedule()` (`jetcd#1227`), lacks fragmentation support, and tests resumption only at `startRev == 0` against `etcd v3.5.14`.
   - *Evidence:* `jetcd-core/src/main/java/io/etcd/jetcd/impl/WatchImpl.java:179-181, 256-268, 293, 312, 357-368`, `WatchResumeTest.java:37-69`, `Etcd.java:31 @ jetcd-0.8.7`; commit `70be01a` (`jetcd#946`).
   - *Condition under which this claim fails:* Historical event loss occurs whenever `startRevision < serverCurrentRev` and the gRPC stream disconnects after `processCreated` before all unsynced historical batches up to `serverCurrentRev` have been delivered.

---

## 1. Architectural Premise: How Watch Relates to Raft and Leader Failover

A central question in evaluating etcd's architecture is whether Watch suffers from the same design flaw as Leases—bypassing Raft consensus and consequently losing state on leader failover. Comparing etcd's source code (`v3.7.3`) with its issue history reveals a sharp boundary between **watched data mutations** (which go through Raft) and **watch stream operations** (which bypass both Raft and linearizable `ReadIndex`).

### 1.1 What Goes Through Raft: Mutating Operations and the MVCC Event Log

In etcd's Leases subsystem, `LeaseRenew` mutates volatile, leader-local TTL state in memory without proposing a Raft entry, so a newly elected leader has no record of renewals granted by the old leader.

Watch handles **write operations** differently:
1. Every mutation that can produce a watch event—`Put`, `DeleteRange`, `Txn`, `LeaseRevoke`, and `Compact`—is committed through Raft and deterministically applied by every cluster member in its Raft apply pipeline.
2. As each member applies a committed entry to its local Multi-Version Concurrency Control (`bbolt`) store, `watchableStoreTxnWrite.End()` runs synchronously inside the MVCC write transaction, increments the revision, constructs `mvccpb.Event` records, and notifies local watchers under `watchableStore.mu.Lock()`.
3. Because every voting member and learner retains the full historical revision log `[compactMainRev, currentRev]` in `bbolt`, etcd does **not** need to replicate active watcher registrations (`WatchCreateRequest`) through Raft for a surviving member to replay missed events after a leader failover—*provided* the client presents an accurate resumption revision (`StartRevision = R + 1`) to a member that has applied at least up to $R$.

- **Code References (`v3.7.3`):**
  - Lease renewal comparison: `server/etcdserver/v3_server.go:489`
  - Raft apply pipeline: `server/etcdserver/raft.go:218`, `server/etcdserver/server.go:843-845, 1945`
  - Synchronous watch notification on write commit (`server/storage/mvcc/watchable_store_txn.go:22-47 @ v3.7.3`):
    ```go
    func (tw *watchableStoreTxnWrite) End() {
        changes := tw.Changes()
        if len(changes) == 0 {
            tw.TxnWrite.End()
            return
        }
        rev := tw.Rev() + 1
        evs := make([]mvccpb.Event, len(changes))
        for i, change := range changes {
            evs[i].Kv = &changes[i]
            if change.CreateRevision == 0 {
                evs[i].Type = mvccpb.Event_DELETE
                evs[i].Kv.ModRevision = rev
            } else {
                evs[i].Type = mvccpb.Event_PUT
            }
        }
        // end write txn under watchable store lock so the updates are visible
        // when multiplexing the event to all watchers
        tw.s.mu.Lock()
        tw.s.notify(rev, evs)
        tw.TxnWrite.End()
        tw.s.mu.Unlock()
    }
    ```

### 1.2 What Bypasses Raft and `ReadIndex`: Watch Streams, Bookmarks, and Liveness

While data mutations go through Raft, **watch stream control operations completely bypass Raft and linearizable `ReadIndex`**. Any replica (leader or follower) can terminate a `Watch` gRPC stream and evaluate requests purely against its local, unfenced MVCC state:

| Watch Operation | Code Path (`v3.7.3`) | Uses Raft or `ReadIndex`? | Requires Leader? | Consequence on Failover, Partition, or Lagging Follower |
|---|---|---|---|---|
| **`WatchCreateRequest` (`startRev == 0`)** | `server/etcdserver/api/v3rpc/watch.go:340` $\to$ `watchable_store.go:118-167` | **No** (reads local `s.store.currentRev`) | **No** | Binds "current revision" to a potentially stale follower or partitioned ex-leader; unlocked `Created` header read (`watch.go:358`) races with concurrent commits (§3.2). |
| **`WatchCreateRequest` (`startRev > 0`)** | `server/storage/mvcc/watchable_store.go:141-155` | **No** (compares `startRev` against local `currentRev`) | **No** | If `startRev > s.store.currentRev` (failover to a lagging follower), the watcher is placed in `s.synced` (`line 141`) even though the follower has not yet applied up to `startRev` (§3.3). |
| **`WatchProgressRequest` (`RequestProgress`)** | `server/etcdserver/api/v3rpc/watch.go:395-400` $\to$ `watchable_store.go:507-540` | **No** (reads local `s.rev()` under `s.mu.RLock()`) | **No** | A partitioned member with no leader (`s.Leader() == raft.None`) replies with `ProgressNotify` at its frozen local revision (`#17611`); a lagging follower either regresses client revision (`< v3.7.0`, `#20221`) or suppresses progress server-wide (`v3.7.3`, `#20242`). |
| **Periodic `ProgressNotify` (`WithProgressNotify`)** | `server/etcdserver/api/v3rpc/watch.go:558-570` | **No** (reads local `s.rev()`) | **No** | Emits unfenced local revision every `progressReportInterval` (default 10m), convincing clients that a partitioned watch stream is healthy. |
| **Stream Liveness on Partition / Leader Loss** | `server/etcdserver/api/v3rpc/interceptor.go:239-258, 315-353` | **No** (polls local `s.Leader() == raft.None`) | **Opt-in only** (`WithRequireLeader`) | Off by default (watcher hangs forever on partitioned node), broken in `grpcproxy` (`watch.go:112`), and when enabled on direct connections, aborts with `ErrGRPCNoLeader`, which `clientv3` and `jetcd` treat as a fatal error that destroys the watch (§3.1). |

**Architectural takeaway:** Etcd relies on the client's monotonic `nextRev` cursor to bridge stateless watch streams across failovers, while serving those streams and `ProgressNotify` bookmarks from **unfenced, non-linearizable local replica state**. Almost every failover, partition, and cache-inconsistency bug in etcd's watch history arises from the collision between these two design choices.

---

## 2. Watch API Guarantees vs. Explicit Contract Disclaimers

Evaluating production readiness requires distinguishing between **direct violations of etcd's promised invariants** and **hazards that are legally permitted by etcd's specification disclaimers**.

### 2.1 The Eight Documented and Validated Watch Promises

Across its documentation, protobuf specification, client contract, and robustness test suite, etcd defines eight core watch guarantees:

| # | Promise | Definition & Invariant | Specification & Validator References (`v3.7.3`) |
|---|---|---|---|
| **1** | **Ordered** | Events on a watch stream are delivered in strictly non-decreasing `ModRevision` order (`event[i].Kv.ModRevision <= event[i+1].Kv.ModRevision`). | `etcd-io/website` `content/en/docs/v3.7/learning/api_guarantees.md`; `tests/robustness/validate/watch.go:30-39, 167-170` |
| **2** | **Unique** | An event for a given key at a given `ModRevision` is never delivered more than once on the same logical watch (unless split across fragments when `WithFragment()` is enabled). | `tests/robustness/validate/watch.go:229-239` |
| **3** | **Atomic** | All watched events produced by the same revision (e.g., a multi-key `Txn` or `LeaseRevoke`) are delivered in the same `WatchResponse` unless `WithFragment()` is set. | `tests/robustness/validate/watch.go:171-173`; enforced by `eventBatch.add` at `server/storage/mvcc/watcher_group.go:53-60` |
| **4** | **Reliable (No Gaps)** | A watch stream never drops matching events within the retained history window: if events at revisions $R_a$ and $R_c$ ($R_a < R_c$) are delivered, every matching event $R_b \in (R_a, R_c)$ must be delivered between them. | `tests/robustness/validate/watch.go:268-281` |
| **5** | **Resumable** | After a stream breaks at revision $R$, opening a new watch with `WithRev(R + 1)` (`start_revision` is **inclusive**) delivers all matching events at revisions $\ge R + 1$ without gaps or duplicates, unless $R + 1$ has been compacted (which must explicitly cancel the watch with `CompactRevision > 0`). | `api/etcdserverpb/rpc.proto:784, 850-858`; `tests/robustness/validate/watch.go:30-39` |
| **6** | **Bookmarkable (`ProgressNotify`)** | An empty progress response (`IsProgressNotify() == true`) with `Header.Revision = R` guarantees that all matching events with `ModRevision <= R` have already been delivered on the stream, and all subsequent events will have `ModRevision > R`. | `api/etcdserverpb/rpc.proto:424-429`; `tests/robustness/validate/watch.go:159-165, 255-267` |
| **7** | **Metadata & Filter Accuracy** | `WithPrevKV()` populates `Event.PrevKv` from `ModRevision - 1` on non-create events (unless compacted, §4.5); filters (`WithFilterPut`, `WithFilterDelete`) and RBAC (`isWatchPermitted`) restrict delivery to authorized matching keys in `[Key, RangeEnd)`. | `api/etcdserverpb/rpc.proto:805-807`; `tests/robustness/validate/watch.go:205-227`; `server/etcdserver/api/v3rpc/watch.go:287-328` |
| **8** | **Partition Detection & Error Delivery** | Wrapping a watch context with `WithRequireLeader` prevents a watch from staying stuck on a partitioned node by closing with `ErrNoLeader`; whenever any watch is canceled, the final message on `WatchChan` must have `Canceled: true` and `Err() != nil`. | `client/v3/watch.go:57-92` (specifically lines `67-70` and `78-83`) |

### 2.2 What the Contract Disclaims—and Why Equating "Eventual Consistency" With "Might Never Arrive" Is a Conceptual Contradiction

While the eight promises above constrain the **sequence** of events delivered on a stream, etcd's official documentation simultaneously claims that Watch implements **eventual consistency** while disclaiming that events will ever arrive:

> - *"Eventual Consistency: Components may temporarily see different data after an update **but converge to the same view eventually**."* (`tests/robustness/README.md:102 @ v3.7.3`)
> - *"Etcd provides strict serializability for KV operations and **eventual consistency for Watch**."* (`tests/robustness/README.md:111 @ v3.7.3`)
> - *"Calling KV API will take an immediate effect, while Watch API will return with some unbounded delay. In correctly working etcd cluster you should expect to see watch events to appear with 10ms delay after them happening. **However, there is no limit and events in unhealthy clusters might never arrive.**"* (`etcd-io/website` `content/en/docs/v3.7/learning/api_guarantees.md:36-38`)
> - *"etcd does not ensure linearizability for watch operations. Users are expected to verify the revision of watch events to ensure correct ordering with other operations."* (`etcd-io/website` `content/en/docs/v3.7/learning/api_guarantees.md:115-116`)
> - *"Liveness. Servers may fail or disconnect briefly. Clients should make progress in either way. Clients should never deadlock waiting for a server to come back from offline, unless configured to do so."* (`etcd-io/website` `content/en/docs/v3.7/learning/design-client.md:48`)
> - *"The balancer does not understand node’s health status or cluster membership. So, it is possible that balancer gets stuck with one failed or partitioned node."* / *"Client-side keepalive ping still does not reason about network partitions. Streaming request may get stuck with a partitioned node."* (`etcd-io/website` `content/en/docs/v3.7/learning/design-client.md:80, 106, 136`, originally `Documentation/dev-guide/apispec/grpc/design-client.md`)
> - *"In order to prevent a watch stream being stuck in a partitioned node, make sure to wrap context with 'WithRequireLeader'."* (`client/v3/watch.go:82-83 @ v3.7.3`)

#### 2.2.1 The Conceptual Contradiction: Eventual Consistency (Liveness) vs. Prefix Safety ("Might Never Arrive")

There is a fundamental distributed-systems contradiction across these statements:

1. **Eventual consistency is a *liveness* guarantee (`converge eventually`), not merely a safety property:**
   By definition—including etcd's own definition in `tests/robustness/README.md:102 @ v3.7.3`—eventual consistency requires that if a replicated system remains operational (maintains a majority quorum) and a client remains connected to the system, every committed update **must eventually reach the client** so that all components converge to the same view.
   - A specification that guarantees *Ordered*, *Unique*, *Atomic*, and *Reliable* (no interior gaps between delivered events $R_a$ and $R_c$) while allowing the stream to stop delivering new events forever provides **monotonic prefix safety with zero liveness**. An empty or frozen stream vacuously satisfies every safety rule in `tests/robustness/validate/watch.go:30-39` while violating eventual convergence.
   - Either the etcd documentation conflates *asynchrony* (no hard real-time bound $\Delta$, even though events eventually arrive) with *non-termination / permanent silence* ("might never arrive"), or `tests/robustness/README.md:102, 111` and `design-client.md:48` directly contradict `api_guarantees.md:38`.

2. **Why "unhealthy clusters" is a false excuse — events fail to arrive when the cluster *has* a healthy quorum:**
   If *"unhealthy clusters"* in `api_guarantees.md:38` only meant a cluster that has lost quorum (e.g., 2 of 3 nodes failed), the statement would be trivial and uninteresting: a consensus cluster without quorum has failed and cannot make progress on any API. The critical flaw in etcd is that **watch events can fail to arrive forever in a cluster that *maintains a healthy majority quorum* and is actively committing writes**:
   - **Scenario A — 1-of-3 Minority Network Partition (Quorum Intact, Default Watch):** In a 3-node cluster $\{M_1, M_2, M_3\}$ where $M_1$ is partitioned from $\{M_2, M_3\}$ while the client $\leftrightarrow M_1$ TCP link stays up, $\{M_2, M_3\}$ form a healthy quorum and commit writes normally. For KV operations (`Put`, linearizable `Get`), the cluster masks the single-node fault ($f = 1$). For Watch, because stream operations bypass Raft and `ReadIndex` (`server/etcdserver/api/v3rpc/watch.go:340-400, 558-570 @ v3.7.3`) and `WithRequireLeader` is off by default (`server/etcdserver/api/v3rpc/interceptor.go:239-258 @ v3.7.3`), $M_1$ **never closes the watch stream** and continues replying to `RequestProgress` / `WithProgressNotify` with $M_1$'s frozen local revision.
   - **Scenario B — 1-of-3 Follower Apply-Loop Stall (Quorum Intact, Even *With* `WithRequireLeader`):** Even when the client enables `WithRequireLeader`, `monitorLeader` (`server/etcdserver/api/v3rpc/interceptor.go:315-354 @ v3.7.3`) only checks `s.Leader() == raft.None`. If follower $M_1$ still receives Raft heartbeats from leader $M_2$ (`s.Leader() != raft.None`), the cluster has quorum and $M_1$ knows the leader—**but** if $M_1$'s local `applyAll` goroutine is stalled (e.g., blocked on a `bbolt` commit/defrag or scanning 500k revisions under `watchableStore.mu.Lock()` in `#22264`, `server/storage/mvcc/watchable_store.go:346-414 @ v3.7.3`), $M_1$ never applies new entries and never cancels the watch stream.
   - **Scenario C — Dead Upstream Broadcast in `grpcproxy` (`#21813`, Quorum Intact):** When watching through `grpcproxy`, an upstream cancel resets `wb.nextrev = 1`, drops the `Canceled` response (`server/proxy/grpcproxy/watcher.go:107 @ v3.7.3`), and `coalesce()` (`server/proxy/grpcproxy/watch_broadcasts.go:67-79 @ v3.7.3`) actively migrates receivers from healthy broadcasts onto the dead broadcast while the cluster is 100% healthy.

3. **Why this negates the core utility of Watch and forces clients back to polling:**
   The entire reason to use a `Watch` subscription rather than polling `KV.Get` is so the client can rely on the server to push changes without periodic polling. Because any etcd watcher can silently stop delivering events even while the cluster maintains quorum—and because the client has no built-in way to know its watcher has entered this silent zombie state (`RequestProgress` itself reads unfenced local state and can be starved server-wide by `#20242`), **clients cannot rely on Watch alone for eventual consistency**. Instead, production clients are forced to implement **periodic polling or scheduled watch restarts**:
   - Either polling a quorum-backed linearizable `KV.Get` (to force a Raft `ReadIndex`, compare the cluster's true revision against the watcher's local revision, and bypass or restart the watch when it lags—as Kubernetes had to engineer in `KEP-2340` `ConditionalProgressRequester` + `delegator.go:160-172`), or
   - Periodically tearing down and restarting watchers / re-listing on a fixed schedule to escape silent zombie streams (without which stale watch caches cause production split-brain bugs such as `kubernetes/kubernetes#59848` / `etcd#8673`).

---

## 3. Leader Failover, Network Partitions, and `ReadIndex` Bypass Failures

### 3.1 `WithRequireLeader`: Mechanism, Target Scenario, and Nine Failure Modes

#### 3.1.1 How `WithRequireLeader` Works and Why It Was Added

**Target Scenario:**
Consider a 3-node cluster $\{M_1, M_2, M_3\}$ where a client's watch stream is connected to $M_1$. A network partition isolates $M_1$ from $\{M_2, M_3\}$, while the TCP connection between the client and $M_1$ remains healthy. $\{M_2, M_3\}$ form a majority quorum and continue committing writes. Because the client–$M_1$ network link is intact, HTTP/2 keepalive pings succeed. Without `WithRequireLeader`, $M_1$ never closes the watch stream, and any `RequestProgress` or `WithProgressNotify` tick on $M_1$ returns `ProgressNotify` responses bearing $M_1$'s frozen local revision (`#17611`).

**Mechanism & Raft Timing (`v3.7.3`):**
To detect this minority partition, `WithRequireLeader` was introduced in May 2016 (PRs `#5324`, `#5328`, `#5332`):
1. **Client metadata:** Calling `clientv3.WithRequireLeader(ctx)` (or `WatchOption.withRequireLeader(true)` in `jetcd`) attaches outgoing gRPC metadata `"hasleader": "true"` to the request. In `clientv3`, `Watcher.streams` partitions underlying gRPC streams by context metadata, placing `WithRequireLeader` watches on a dedicated `watchGRPCStream` keyed by `"[hasleader true]"`.
2. **Server stream registration:** When `newStreamInterceptor` sees `"hasleader": "true"`, it checks whether `s.Leader() == raft.None` (immediately returning `ErrGRPCNoLeader` if so) and otherwise wraps the stream in a cancellable context registered in `smap.streams`.
3. **Raft leader loss + `monitorLeader` cancellation (~3–5 seconds):**
   Under default server settings (`--heartbeat-interval=100ms`, `--election-timeout=1000ms`), when $M_1$ is isolated from $\{M_2, M_3\}$:
   - If $M_1$ was a **follower**, its randomized election timer expires after $1\text{–}2\text{ s}$ and `becomePreCandidate` sets `r.lead = None`.
   - If $M_1$ was the **leader**, `CheckQuorum` (unconditionally enabled in etcd) causes $M_1$ to step down after one election timeout ($1\text{ s}$) via `becomeFollower(r.Term, None)`, setting `s.Leader() = raft.None`.
   - The server's background `monitorLeader` goroutine ticks once per election timeout ($1\text{ s}$). After `maxNoLeaderCnt = 3` consecutive ticks where `s.Leader() == raft.None` ($3\text{ s}$), it cancels all streams in `smap.streams` with `rpctypes.ErrGRPCNoLeader` (`codes.Unavailable`, `"etcdserver: no leader"`). Total time from partition to stream cancellation is **$\approx 3\text{–}5\text{ seconds}$**.

- **Code & Issue References:**
  - Historical introduction & adoption: etcd PRs `#5324` (`19221b33c`), `#5328` (`68eaf4083`), `#5332` (`9c103dd0d`), `#9872`; issues `#5354`, `#7247`, `#8673`; Kubernetes `#89488`, PR `#89881`.
  - Client metadata & stream keying: `client/v3/ctx.go:30-41`, `client/v3/watch.go:330, 1086-1091 @ v3.7.3`; `WatchOption.java:154`, `WatchImpl.java:155-161 @ jetcd-0.8.7`.
  - Raft step-down & election timers: `server/embed/config.go:533-534`, `server/etcdserver/bootstrap.go:548`, `server/etcdserver/raft.go:181-206 @ v3.7.3`; `raft.go:917-929, 1281-1284 @ raft v3.7.0`.
  - Server interceptor & `monitorLeader` (`server/etcdserver/api/v3rpc/interceptor.go:63-67, 239-258, 315-354 @ v3.7.3`):
    ```go
    if hasLeaderRequirement(md) {
        if s.Leader() == types.ID(raft.None) {
            return rpctypes.ErrGRPCNoLeader
        }
        cctx := newCancellableContext(ss.Context())
        ss = &serverStreamWithCtx{ctx: cctx, ServerStream: ss}
        smap.mu.Lock()
        smap.streams[ss] = struct{}{}
        smap.mu.Unlock()
        ...
    }
    ```

#### 3.1.2 Why `WithRequireLeader` Fails: Nine Verified Client, Proxy, and Server Failure Modes

Tracing `etcd v3.7.3` and `jetcd 0.8.7` in the exact 1-node-partitioned or 1-node-crash scenario where the remaining 2 members of a 3-node cluster still form a healthy quorum exposes **nine distinct ways** `WithRequireLeader` fails to preserve Watch API promises.

##### Group A: Client-Side Teardown & Resumption Failures on `ErrNoLeader` (Failures #1–#5)

###### Failure #1: Terminal Watch Destruction Instead of Transparent Failover
- **Finding:** When isolated member $M_1$ aborts the stream with `ErrGRPCNoLeader`, both Go `clientv3` and Java `jetcd` treat that per-member error as a fatal cluster error and permanently close the caller's watch, even when configured with healthy surviving endpoints `[M1, M2, M3]`.
- **Mechanism & Evidence:**
  - When a member crashes cleanly, `watchGRPCStream` transparently switches to $M_2$ or $M_3$ and resumes every substream at `ws.initReq.rev` without closing the user's `WatchChan` (`TestBalancerUnderServerShutdownWatch`, `tests/integration/clientv3/connectivity/server_shutdown_test.go:33-114 @ v3.7.3`).
  - `clientv3`'s general retry helper `isHaltErr(ctx, err)` (`client/v3/client.go:570-585`, `client_test.go:162-165 @ v3.7.3`) explicitly classifies `v3rpc.ErrNoLeader` (`codes.Unavailable`) as **non-halting (`false`)** so RPCs retry on another endpoint.
  - Yet `watchGRPCStream.run()` (`client/v3/watch.go:665-670 @ v3.7.3`) explicitly overrides `isHaltErr` to force a terminal exit on `ErrNoLeader`:
    ```go
    case err := <-w.errc:
        if isHaltErr(w.ctx, err) || errors.Is(ContextError(w.ctx, err), v3rpc.ErrNoLeader) {
            w.closeErr = err
            return
        }
    ```
  - Java `jetcd` copies the same override (`jetcd-core/src/main/java/io/etcd/jetcd/impl/WatchImpl.java:353-355 @ jetcd-0.8.7`):
    ```java
    private boolean shouldReschedule(final Status status) {
        return !Errors.isHaltError(status) && !Errors.isNoLeaderError(status);
    }
    ```
  - Upstream issues: `etcd#8980`, `etcd#12534`, `jetcd#1352`, `kubernetes/kubernetes#111116`.

###### Failure #2: Silent Loss of the Resumption Bookmark (`ws.initReq.rev`) on `startRev == 0` Watches
- **Finding:** When `ErrNoLeader` terminates a `startRev == 0` watch created without `WithCreatedNotify()`, `clientv3` discards the creation revision it recorded internally and delivers a close response with `Header.Revision == 0`. A caller that catches `ErrNoLeader` and re-opens `Watch()` on the surviving quorum has no revision bookmark and **silently misses every write committed during the 3–5 s partition window**.
- **Mechanism & Evidence:**
  1. When a caller opens `wch := cli.Watch(WithRequireLeader(ctx), "a")` on $M_1$ at revision `100`, the server sends `WatchResponse{Created: true, Header: {Revision: 100}}`.
  2. Because `ws.initReq.createdNotify` defaults to `false`, `serveSubstream` (`client/v3/watch.go:846-872 @ v3.7.3`) swallows the `Created` message and stores `100` privately in `ws.initReq.rev` (`line 871`). The caller has received nothing on `wch` and does not know revision `100`.
  3. While $M_1$ is partitioned and waiting 3 election timeouts, the surviving quorum $\{M_2, M_3\}$ commits `Put("a", "b")` at revision `101`.
  4. When $M_1$ aborts the stream with `ErrGRPCNoLeader`, `run()` refuses to resume internally (Failure #1) and calls `closeSubstream(ws)` (`client/v3/watch.go:488-503 @ v3.7.3`):
     ```go
     closeResp := &WatchResponse{
         Header:   &pb.ResponseHeader{},
         Canceled: true,
         closeErr: w.closeErr,
     }
     ```
  5. `closeSubstream` sends an empty `&pb.ResponseHeader{}` (`Header.Revision == 0`), throwing away `ws.initReq.rev` (`100`). When the caller re-opens `cli.Watch(WithRequireLeader(ctx), "a")` on $\{M_2, M_3\}$, it starts at the quorum's new revision (`101+`), permanently missing revision `101`.

###### Failure #3: Pinned `READY` gRPC `SubConn` and Broken Retry Loop in `openWatchClient()`
- **Finding:** Even if the application knows the exact revision $R$ and manually calls `cli.Watch(WithRequireLeader(ctx), "a", WithRev(R))` on a multi-endpoint client `[M1, M2, M3]`, the gRPC `SubConn` to partitioned $M_1$ remains `READY`, and `openWatchClient()`'s retry loop fails to mask `ErrGRPCNoLeader`, causing roughly $1/N$ of manual re-watch attempts to immediately die again.
- **Mechanism & Evidence:**
  - When $M_1$ cancels the stream with `ErrGRPCNoLeader`, gRPC sends HTTP/2 stream trailers (`RST_STREAM`), **not** a connection-level `GOAWAY` frame. Because the TCP link between the client and $M_1$ is intact, `grpc-go`'s `round_robin` balancer keeps $M_1$'s `SubConn` in state `READY`.
  - In `openWatchClient()` (`client/v3/watch.go:1039-1062 @ v3.7.3`), the loop checks `if ws, err = w.remote.Watch(w.ctx, w.callOpts...); ws != nil && err == nil { break }`.
  - In gRPC, `w.remote.Watch()` (`ClientConn.NewStream()`) only allocates a local HTTP/2 stream ID and writes request `HEADERS` to a `READY` `SubConn`; it **does not wait for the server's `newStreamInterceptor` (`interceptor.go:240`) to run**. Thus `w.remote.Watch()` immediately returns `ws != nil, err == nil` even when `round_robin` selects partitioned $M_1$.
  - `serveWatchClient(wc)` (`watch.go:772, 933`) then calls `wc.Recv()`, receives $M_1$'s `ErrGRPCNoLeader` trailer asynchronously, and kills the newly created `WatchChan` again.
  - **Test suite confirmation (`tests/integration/clientv3/connectivity/network_partition_test.go:205-248 @ v3.7.3`, commit `a57e967d84`):** When etcd switched to standard gRPC `round_robin`, `testBalancerUnderNetworkPartitionWatch` failed whenever the new watch landed on partitioned `M0`. Rather than fixing `watchGRPCStream`, commit `a57e967d84` commented out `watchCli.SetEndpoints(eps...)` and hard-coded a non-partitioned target endpoint (`targetEp := eps[(lead+1)%3]`, lines `228-238`).

###### Failure #4: Silent Discard of Buffered Events (`ws.buf`) and Drop of `ErrNoLeader` After `250ms`
- **Finding:** When `ErrNoLeader` aborts `watchGRPCStream`, `clientv3` immediately discards any unread event batches queued in `ws.buf`, and if the 1-slot output channel `ws.outc` is occupied for more than `250ms`, `clientv3` silently drops the final `Canceled: true` / `ErrNoLeader` response and closes `WatchChan` with no error.
- **Mechanism & Evidence (`client/v3/watch.go:40, 478-500, 520-541, 818-836 @ v3.7.3`):**
  1. When `run()` exits on `ErrNoLeader`, its `defer` (`line 525`) closes `ws.recvc`.
  2. Closing `ws.recvc` causes `serveSubstream` (`line 835`) to `return` immediately without draining `ws.buf` to `ws.outc`, losing all buffered events that had already arrived from the server.
  3. Next, `closeSubstream` calls `sendCloseSubstream` (`lines 478-485`) with `closeSendErrTimeout = 250 * time.Millisecond` (`line 40`):
     ```go
     func (w *watchGRPCStream) sendCloseSubstream(ws *watcherStream, resp *WatchResponse) {
         select {
         case ws.outc <- *resp:
         case <-ws.initReq.ctx.Done():
         case <-time.After(closeSendErrTimeout):
         }
     }
     ```
  4. Because `ws.outc` has buffer capacity `1` (`line 558`), if `ws.outc` already holds 1 unconsumed event while the caller is busy for $> 250\text{ ms}$, `sendCloseSubstream` times out, drops `closeResp`, and closes `ws.outc` (`line 500`), violating the documented contract (`client/v3/watch.go:67-70 @ v3.7.3`: *"The last message will be a watch response with Canceled set to true, and Err() will return a non-nil error"*).

###### Failure #5: Permanent Client Goroutine Deadlock in `w.wg.Wait()` (`#17611` / `#17622`)
- **Finding:** If `ErrNoLeader` arrives on `w.errc` right after a substream receives a rejected or canceled `Created` response, `watchGRPCStream.run()` deadlocks permanently in `w.wg.Wait()`.
- **Mechanism & Evidence (`client/v3/watch.go:466-476, 520-541, 601-609, 803 @ v3.7.3`):**
  - When `w.resuming[0]` receives a `Created` response with `WatchId == InvalidWatchID` or `Canceled && CancelReason != ""`, `addSubstream` (`lines 468-472`) closes `ws.recvc` and returns `false` (so `ws` is not added to `w.substreams`), and `line 607` sets `w.resuming[0] = nil` (removing `ws` from `w.resuming`).
  - `serveSubstream` exits on closed `ws.recvc` and blocks in its `defer` at `w.closingc <- ws` (`line 803`) before calling `w.wg.Done()`.
  - If `ErrNoLeader` arrives on `w.errc` before `run()` selects `<-w.closingc`, `run()` enters its `defer` (`lines 520-541`), which only iterates over `w.substreams` and `w.resuming` (neither contains `ws`), never drains `w.closingc`, and blocks forever at `w.wg.Wait()` (`line 539`).

##### Group B: Context Multiplexing & Proxy Failures (Failures #6 & #8)

###### Failure #6: `RequestProgress(ctx)` Silently Fails Unless `ctx` Also Wraps `WithRequireLeader` (`#12254`)
- **Finding:** Calling `cli.RequestProgress(ctx)` on a plain `context.Context` never requests progress for watches created with `cli.Watch(clientv3.WithRequireLeader(ctx), ...)`.
- **Mechanism & Evidence (`client/v3/watch.go:330, 413-428, 1086-1091 @ v3.7.3`):**
  - `Watcher.streams` indexes `watchGRPCStream` instances by `streamKeyFromCtx(ctx)`. Watches with `WithRequireLeader` live on stream `"[hasleader true]"`.
  - Calling `RequestProgress(ctx)` without `WithRequireLeader(ctx)` looks up stream `""`, lazily creates a second, completely empty `watchGRPCStream` with zero watchers, and sends `WatchProgressRequest` on that empty stream.

###### Failure #7: Server-Side and Transport Blind Spots (Apply Stall, Paused Node, or Unreachable Cluster)
- **Finding:** Even with `WithRequireLeader` on a direct connection, watches hang indefinitely in four non-minority-partition failure states:
  1. *Apply-loop stall while Raft keeps ticking:* `monitorLeader` only checks `s.Leader() == raft.None` (`interceptor.go:329`), which is updated in the Raft loop (`raft.go:181-206`), whereas committed entries are applied on the separate `s.applySched` FIFO goroutine (`server.go:843-845`). If `s.applySched` blocks or stalls (`#22184`, `#22635`), `s.Leader()` stays non-zero and `monitorLeader` never fires while all watches on that member go silent.
  2. *Intermittent heartbeats on a degraded link:* Because `monitorLeader` resets `noLeaderCnt = 0` whenever a single sample sees `s.Leader() != raft.None` (`interceptor.go:329-333`), a follower receiving occasional small `MsgHeartbeat` packets over a lossy link resets `noLeaderCnt` and never cancels its streams.
  3. *Paused server (`SIGSTOP` / VM freeze) or black-holed TCP link:* A paused process cannot run `monitorLeader`, leaving detection to client HTTP/2 keepalive pings—which Go `clientv3` disables by default (`DialKeepAliveTime == 0` at `client/v3/client.go:231-236 @ v3.7.3`, whereas `jetcd` defaults to `30s`/`10s` at `ClientBuilder.java:66-68 @ jetcd-0.8.7`).
  4. *Client partitioned from all members:* Because `streamKeyFromCtx` strips deadlines (`client/v3/watch.go:274`) and `openWatchClient` retries forever with `WaitForReady(true)` (`watch.go:85-91, 1039`, `options.go:24`), complete client isolation blocks silently forever (`#8980`).

###### Failure #8: `grpcproxy` Completely Ignores `WithRequireLeader` and Splits Its Leader Detector Across Streams
- **Finding:** Any watch routed through `etcd grpc-proxy` silently loses `WithRequireLeader` protection because the proxy inspects the wrong gRPC metadata context on incoming streams, and furthermore routes its internal leader-loss watch on a separate gRPC stream from its data watches.
- **Mechanism & Evidence (`server/proxy/grpcproxy/watch.go:111-125`, `lease.go:160-174`, `leader.go:42-80`, `watch_broadcast.go:58-68`, `adapter/chan_stream.go:142-155 @ v3.7.3`):**
  1. In `watchProxy.Watch` (`watch.go:111-114`) and `leaseProxy.LeaseKeepAlive` (`lease.go:160-163`):
     ```go
     var lostLeaderC <-chan struct{}
     if md, ok := metadata.FromOutgoingContext(stream.Context()); ok {
         v := md[rpctypes.MetadataRequireLeaderKey]
         if len(v) > 0 && v[0] == rpctypes.MetadataHasLeader {
             lostLeaderC = wp.leader.lostNotify()
     ```
     `stream` is a **server-side** stream (`pb.Watch_WatchServer`). Client metadata arrives in the **incoming** context (`metadata.FromIncomingContext`, used by `interceptor.go:228` and `grpcproxy/util.go:27`), never in the outgoing context. When PR `#7896` (commit `5e059fd8d`, `v3.3.0`) migrated from `metadata.FromContext`, it mistakenly used `FromOutgoingContext`. On a real networked proxy (`server/etcdmain/grpc_proxy.go:507-525`), `ok` is always `false` and `lostLeaderC` is always `nil`.
  2. *Why tests missed it:* Proxy integration tests use the in-process channel adapter (`adapter/chan_stream.go:142-155`), which derives the server stream's `sctx` directly from the client's `ctx`, leaking outgoing metadata into the server context.
  3. *Split-stream defect:* Even if `FromIncomingContext` were used, `newLeader` (`leader.go:44, 62`) watches `"__lostleader"` **with `WithRequireLeader(ctx)`**, whereas `newWatchBroadcast` (`watch_broadcast.go:58-68`) opens upstream data watches **without `WithRequireLeader`**. Because `clientv3` keys `w.streams` by context metadata, the proxy's leader detector and data watches run on **two different gRPC connections**, which `round_robin` can assign to different backend etcd members.

###### Failure #9: Zero Grace Period on New Streams Causes Routine Leader Crashes to Kill `WithRequireLeader` Watches
- **Finding:** While established streams get a 3-election-timeout (~3 s) grace period before `monitorLeader` cancels them, `newStreamInterceptor` rejects newly opened streams with **zero grace period** if `s.Leader() == raft.None`. Consequently, when a leader crashes cleanly and `clientv3` attempts a transparent failover to a surviving member while an election is in flight, `newStreamInterceptor` immediately returns `ErrGRPCNoLeader` and `clientv3` terminally kills the watch.
- **Mechanism & Evidence (`server/etcdserver/api/v3rpc/interceptor.go:239-242`, `client/v3/watch.go:667, 678 @ v3.7.3`):**
  1. A client watches `[M1, M2, M3]` with `WithRequireLeader` and is connected to leader $M_1$.
  2. $M_1$ crashes or restarts cleanly. `serveWatchClient` sees a transport EOF (`codes.Unavailable`, not `ErrNoLeader`), so `watchGRPCStream.run()` (`watch.go:678`) calls `w.newWatchClient()` to transparently fail over to $M_2$.
  3. At that moment, $M_2$ and $M_3$ are campaigning (`s.Leader() == raft.None`) for ~100–1000 ms to elect $M_2$.
  4. `newStreamInterceptor` (`interceptor.go:240`) immediately rejects the new stream on $M_2$ with `ErrGRPCNoLeader` (`0` grace period).
  5. `run()` (`watch.go:667`) receives `ErrGRPCNoLeader` and permanently closes the caller's `WatchChan` (`kubernetes/kubernetes#111116`).

---

### 3.2 Watch Creation at Current Revision (`startRev == 0`): Non-Linearizable Binding and the `Created` Header Race

When an application starts a watch without specifying a revision (`startRev == 0`, meaning "receive events after the current state"), two mechanisms in `etcd v3.7.3` and `clientv3 v3.7.3` break linearizability and event continuity:

#### 1. Local `s.store.currentRev` Binding Without `ReadIndex`
When `startRev == 0`, `watchableStore.watch()` binds the watcher's starting revision (`wa.minRev`) directly to the connected replica's local `s.store.currentRev + 1` without running a Raft `ReadIndex`:
- If a client writes key $K$ at revision `100` (on the leader) and then opens a `startRev == 0` watch that lands on a lagging follower at revision `95`, `wa.minRev` is set to `96`. As the follower catches up, the watch delivers historical writes `96..100` that occurred *before* the watch was opened.

#### 2. Unlocked `Created` Header Read + `clientv3` Off-by-One Resume
In `serverWatchStream.recvLoop`, the server registers the watcher under `watchableStore.mu.Lock()` (line 340), releases the lock, and then reads `sws.watchStream.Rev()` **outside the lock** (line 358) to populate `Created.Header.Revision`. Meanwhile, when `clientv3` receives `Created` for a `ws.initReq.rev == 0` watch, it sets `nextRev = wr.Header.Revision` **without adding `+ 1`**—because the comment author at `client/v3/watch.go:849-857` mistakenly assumed `StartRevision: 100` watches *after* revision `100`, overlooking that `start_revision` (`rpc.proto:784`) is **inclusive**.

If the stream disconnects right after `Created` (before the first event arrives), two failure cases occur depending on whether concurrent writes committed between lines 340 and 358:
- **Case A (No concurrent write between line 340 and line 358 — $R_2 == R_1$):** At line 340, `currentRev` was $R_1$, so `wa.minRev = R_1 + 1`. At line 358, `Rev()` returns $R_1$. On `Created`, `clientv3` sets `ws.initReq.rev = R_1` (missing `+ 1`). On reconnect, `clientv3` requests `StartRevision = R_1` (inclusive), **replaying the event at revision $R_1$ which committed before the watch was created**.
- **Case B (Concurrent writes commit between line 340 and line 358 — $R_2 > R_1$):** At line 340, `wa.minRev` was set to $R_1 + 1$. Before line 358 runs, the Raft apply loop commits writes at revisions $R_1 + 1, \dots, R_2$. Line 358 then reads `Rev() == R_2` and sends `Created { Header.Revision: R_2 }`. If the stream disconnects before the buffered event frames are read, `clientv3` sets `ws.initReq.rev = R_2` and reconnects at `StartRevision = R_2`, **silently dropping all committed events in $[R_1 + 1, R_2 - 1]$**.

- **Code Evidence (`v3.7.3`):**
  - Local `currentRev` binding (`server/storage/mvcc/watchable_store.go:141-147 @ v3.7.3`):
    ```go
    s.mu.Lock()
    s.revMu.RLock()
    synced := startRev > s.store.currentRev || startRev == 0
    if synced {
        wa.minRev = s.store.currentRev + 1
        if startRev > wa.minRev {
            wa.minRev = startRev
        }
        wa.startRev = wa.minRev
    }
    ```
  - Unlocked `Created` header read (`server/etcdserver/api/v3rpc/watch.go:340-366 @ v3.7.3`):
    ```go
    id, err := sws.watchStream.Watch(mvcc.WatchID(uv.WatchId), uv.Key, uv.RangeEnd, uv.StartRevision, filters...)
    ...
    wr := &pb.WatchResponse{
        Header:   sws.newResponseHeader(sws.watchStream.Rev()),
        WatchId:  int64(id),
        Created:  true,
        Canceled: err != nil,
    }
    ```
  - `clientv3` `Created` vs. `ProgressNotify` cursor update (`client/v3/watch.go:846-872 @ v3.7.3`):
    ```go
    if wr.Created {
        if !resuming {
            // 1) client establishes a watch on key "a" with revision 0
            // 2) etcd returns a watch response with created=true, revision=100
            // ...
            // 5) client reconnects with etcd with revision 100, and will not miss the event at revision 101
            if ws.initReq.rev == 0 {
                nextRev = wr.Header.Revision
            }
            if ws.initReq.createdNotify {
                ws.outc <- *wr
            }
        }
        resuming = false
    } else if len(wr.Events) == 0 && wr.CompactRevision == 0 {
        // current progress of watch; <= store revision
        nextRev = wr.Header.Revision + 1
    }
    if len(wr.Events) > 0 {
        nextRev = wr.Events[len(wr.Events)-1].Kv.ModRevision + 1
    }
    ws.initReq.rev = nextRev
    ```

---

### 3.3 Failover to a Lagging Follower: "Future Revision" Watches and Server-Wide Progress Starvation

When a client observes revision $R_{\text{leader}}$ (from a `Put`/`Get` or a watch stream on Member A), disconnects, and reconnects its watch with `StartRevision = R_{\text{leader}} + 1` to a lagging follower Member B whose local MVCC store is still at $R_{\text{follower}} < R_{\text{leader}}$, etcd creates a **"future revision" watch** (`startRev > s.store.currentRev`).

#### 1. Why Future Revision Watches Are Placed in `s.synced`
Because Member B has not yet applied revisions $(R_{\text{follower}}, R_{\text{leader}}]$, those revisions are not in Member B's `bbolt` database yet. `watchableStore.watch()` (`watchable_store.go:141-155 @ v3.7.3`) therefore classifies `startRev > s.store.currentRev` as `synced = true` and places the watcher in `s.synced` with `wa.minRev = wa.startRev = startRev`, relying on `notify()` (`line 465`) to filter out events with `ModRevision < w.minRev` until Member B catches up.

#### 2. Historical Bugs Caused by Un-Caught-Up Watchers in `s.synced` (`< v3.7.0`)
Prior to `v3.5.22` / `v3.6.2` / `v3.7.0`, placing a future-revision watcher in `s.synced` broke two invariants:
- **`ProgressNotify` revision regression (`#20221`, `#19487`, `#15271`):** Because the watcher was in `s.synced`, `progressIfSync()` returned `true` with `Header.Revision = R_{\text{follower}} < \text{startRev}` (e.g., emitting revision `66` for a watch started at `148`). When `clientv3` received that `ProgressNotify` (`client/v3/watch.go:864`), it unconditionally executed `nextRev = wr.Header.Revision + 1`, **rewinding `nextRev` backwards from `148` to `67`** and replaying events `67..147` on the next reconnect.
- **Snapshot `Restore()` lowering `wa.minRev` (`cf0369e4a`, PR `#20281`):** If lagging Member B restored a leader snapshot at $R_{\text{snap}} < \text{startRev}$, `Restore()` moved all `s.synced` watchers to `s.unsynced` and `syncWatchers()` overwrote `w.minRev = curRev + 1` ($R_{\text{snap}} + 1 < \text{startRev}$), delivering events older than `startRev`. Fixed in `cf0369e4a` (`watchable_store.go:374 @ v3.7.3`) via `w.minRev = max(curRev+1, w.minRev)`.

#### 3. How the `v3.7.3` Fix (`973847aa2`, PR `#20229`) Causes Server-Wide `ProgressNotify` Starvation (`#20242`, `#20240`)
To fix `#20221`, commit `973847aa2` rewrote `progressIfSync()` to check `s.unsynced`, `s.synced`, and `s.victims`. Because `s.unsynced`, `s.synced`, and `s.victims` are **server-wide** structures shared by every client connected to that etcd member, the `v3.7.3` implementation introduces severe cross-client starvation:
1. **Any single unsynced watcher blocks progress for the entire server:** If *any* client on the server has a watcher in `s.unsynced` (`len(s.unsynced.watchers) > 0`), `progressIfSync()` immediately returns `false` (`line 521`) for every other watcher on the server.
2. **Any single future-revision watcher blocks `RequestProgressAll()` and randomly blocks `RequestProgress(id)`:** The loop `for w := range s.synced.watchers` (`line 524`) iterates over a Go map in randomized order. If `id == 0` (`RequestProgressAll()`, used by `WatchProgressRequest` at `v3rpc/watch.go:397`), or if `id != 0` and map iteration visits *any* other client's watcher with `rev < w.startRev` before visiting `w.id == id`, `progressIfSync()` returns `false`.
3. **Any single victim batch blocks `RequestProgressAll()`:** If `len(s.victims) > 0`, `progressIfSync(0)` returns `false` (`line 532`) for every `WatchProgressRequest` on the server.
4. **10-minute silent drop on `progressTicker`:** When `sendLoop`'s `progressTicker` fires (default `10m`, `v3rpc/watch.go:46, 558-568`), a `false` return from `RequestProgress(id)` silently drops the notification and resets `sws.progress[id] = true` for another 10 minutes (`#20242`). This forced etcd to disable progress notification validation in its robustness test suite (`b51415c3f`, PR `#20240`).

- **Code Evidence (`server/storage/mvcc/watchable_store.go:515-539 @ v3.7.3`):**
  ```go
  func (s *watchableStore) progressIfSync(id WatchID) (int64, bool) {
      s.mu.RLock()
      defer s.mu.RUnlock()
      rev := s.rev()
      //Check if all watchers are synced
      if len(s.unsynced.watchers) > 0 {
          return rev, false
      }
      for w := range s.synced.watchers {
          if rev < w.startRev {
              return rev, false
          }
          if id != 0 && w.id == id {
              return rev, true
          }
      }
      if id == 0 && len(s.victims) == 0 {
          return rev, true
      }
      return rev, false
  }
  ```

---

### 3.4 Historical Ledger of Repaired Watch Guarantee Violations

While watched KV mutations have always gone through Raft, etcd's member-local watch catch-up, snapshot restore, compaction, and cross-member resumption paths repeatedly violated the documented Watch safety promises prior to `v3.7.1`. All entries below are verified against `tests/robustness/README.md:19-39 @ v3.7.3` and git commit history:

| Issue / PR & Commit | Promise Violated | Root Cause & Failover / Recovery Trigger | Introduced | Fixed In |
|---|---|---|---|---|
| [#8411](https://github.com/etcd-io/etcd/issues/8411) | **Reliable** (permanent silence after heal) | `watchableStore.Restore` did not exist; when a partitioned member (e.g., deposed leader) rejoined and restored a leader snapshot, its existing watchers never received events again. | `v3.1.x` | 2017 (`watchableStore.Restore`) |
| [#15220](https://github.com/etcd-io/etcd/issues/15220) / PR [#15237](https://github.com/etcd-io/etcd/pull/15237) (`74feb229c`) | **Bookmarkable** & **Resumable** (silent event loss on reconnect) | `WatchProgressRequest` sent `ProgressNotify` via `ctrlStream` ahead of pending events in `watchStream.Chan()`, advancing the client's bookmark revision past undelivered events (cited in Kubernetes KEP-2340). | $\le$ `v3.4` | `v3.4.25`, `v3.5.8`, `v3.6.0` |
| [#17507](https://github.com/etcd-io/etcd/issues/17507) (`579b22cf3`) | **Bookmarkable** liveness | Regression from `#15237` where a newly created watch that had not yet received any events never answered `RequestProgress`. | `v3.5.8` | `v3.5.13`, `v3.6.0` |
| [#15271](https://github.com/etcd-io/etcd/issues/15271) / PR [#15492](https://github.com/etcd-io/etcd/pull/15492) (`a690707c5`) | **Ordered** & **Unique** ("watch traveling back in time") | `notify()` did not advance `w.minRev` while a watcher was in `s.synced`; when a member restored a leader snapshot after a partition (`watchableStore.Restore`), moving synced watchers to `s.unsynced` replayed already-delivered historical events (`watchable_store.go:486 @ v3.7.3`). | $\le$ `v3.4` | `v3.4.25`, `v3.5.8`, `v3.6.0` |
| [#17247](https://github.com/etcd-io/etcd/issues/17247) | **Unique** (duplicate event) | Bug in `Txn` range caching emitted duplicate watch events on `main` (caught by robustness tests before release). | `main` only | Never released |
| [#17529](https://github.com/etcd-io/etcd/issues/17529) / PR [#17555](https://github.com/etcd-io/etcd/pull/17555) (`405862e80`) | **Reliable** (silent interior gap across compaction) | When `w.ch` was full while `chooseAll` attempted to send a `CompactRevision` cancel (`watcher_group.go:255`), `syncWatchers()` failed to skip `w` (`watchable_store.go:369-373 @ v3.7.3`), advancing `w.minRev = curRev + 1` and moving `w` to `s.synced` without sending `ErrCompacted`. Caused `kube-apiserver` to retain deleted pods for $>24\text{h}$ (`kubernetes/kubernetes#123072`). | $\le$ `v3.4` | `v3.4.31`, `v3.5.13`, `v3.6.0` |
| [#18089](https://github.com/etcd-io/etcd/issues/18089) / PR [#18274](https://github.com/etcd-io/etcd/pull/18274) | **Reliable** (dropped `DELETE` event) | Compacting at the exact revision of a `DELETE` tombstone dropped the tombstone from the index, causing unsynced watchers catching up across the compaction boundary to silently miss the `DELETE`. | $\le$ `v3.4.0` | `v3.4.34`, `v3.5.16`, `v3.6.0` |
| [#19179](https://github.com/etcd-io/etcd/issues/19179) / PR [#19188](https://github.com/etcd-io/etcd/pull/19188) (`83cf7bb4f`) | **Resumable** (dropped `DELETE` on crash/restore) | Same tombstone-at-compaction-revision loss triggered via member crash and `treeIndex` restore when a watch opened at the compaction revision. | $\le$ `v3.4` | `v3.4.36`, `v3.5.18`, `v3.6.0` |
| [#19487](https://github.com/etcd-io/etcd/issues/19487) & [#20221](https://github.com/etcd-io/etcd/issues/20221) (`cf0369e4a`, `973847aa2`) | **Bookmarkable**, **Ordered**, **Unique**, **Reliable** | Future-revision watch on a lagging follower emitted stale `ProgressNotify` below `startRev` (rewinding `clientv3` `nextRev`) or lowered `w.minRev` on snapshot `Restore()` (§3.3). | $\le$ `v3.4` | `v3.5.22`, `v3.6.2`, `v3.7.0` |
| [#20418](https://github.com/etcd-io/etcd/issues/20418) / PR [#21399](https://github.com/etcd-io/etcd/pull/21399) (`5944c74e3`) | Linearizable `ReadIndex` freshness check | `ReadIndex` retries reused the same request ID when a process paused, allowing a stale `ReadIndex` response to satisfy a later linearizable read (undermining KEP-2340's linearizable `Get` + `RequestProgress` check). | `v3.4.20`, `v3.5.0` | `v3.4.42`, `v3.5.28`, `v3.6.9`, `v3.7.0` |
| [#20573](https://github.com/etcd-io/etcd/issues/20573) | **Resumable** (Antithesis snapshot-restore failure) | Antithesis deterministic simulation caught a `Resumable` violation involving snapshot restore in 2025; closed in Nov 2025 as stale without an identified root cause. | Unknown | **Unresolved / closed without root cause** |
| Watch RBAC bypass (`afeaa624d`) | **Access Control** (`isWatchPermitted`) | `recvLoop` normalized `creq.RangeEnd = "\x00"` to `[]byte{}` *before* calling `sws.isWatchPermitted(creq)` (`server/etcdserver/api/v3rpc/watch.go:287-328 @ v3.7.3`), allowing a user with read permission on one key `K` to watch all keys $\ge K$. | $\le$ `v3.5` | `v3.5.33`, `v3.6.14`, `v3.7.1` |

---

## 4. Open Server, Proxy, and Protocol Bugs in `etcd v3.7.3`

Each of the following eight defects is present in `etcd v3.7.3` (`0a173d0410a9`)—and remains unfixed on `origin/main` as of `1573d5941`—and is verified directly against the source code.

### 4.1 Canceled Watch Events Retained in `sendLoop.pending` and Delivered to a Subsequent Watch or Leaked (`#22673`)

- **Summary:** When a watch is canceled on a gRPC stream, `sendLoop` deletes the watcher's ID (`wid`) from `ids` without deleting `pending[wid]`. Any in-flight events for the canceled watch that arrive on `watchStream.Chan()` after `delete(ids, wid)` are buffered into `pending[wid]`. If a raw-gRPC client reuses `wid` on the same stream, those stale events from the old watch are flushed onto the new watch; if `AutoWatchID` is used (`clientv3` default), `pending[wid]` leaks for the lifetime of the stream.
- **Promises Violated:** **Filter**, **Ordered**, **Unique**, and **Reliable** (when `WatchId` is reused); bounded server memory (for all clients).
- **Step-by-Step Mechanism:**
  1. `serverWatchStream.sendLoop()` multiplexes two internal channels onto the gRPC wire: `sws.watchStream.Chan()` (carrying `mvcc.WatchResponse` event batches) and `sws.ctrlStream` (carrying `Created` and `Canceled` control responses from `recvLoop`).
  2. Because events can arrive on `watchStream.Chan()` before `Created` arrives on `ctrlStream`, `sendLoop` tracks active watches in `ids map[mvcc.WatchID]struct{}` and buffers pre-creation events in `pending map[mvcc.WatchID][]*pb.WatchResponse`.
  3. When `sendLoop` receives a `Canceled` message on `ctrlStream`, it executes `delete(ids, wid)` and `continue` **without calling `delete(pending, wid)`**.
  4. Any events already queued in `watchStream.Chan()` for `wid` before `Cancel(wid)` completed—or events racing with `ctrlStream` in Go's `select`—arrive after `wid` has been removed from `ids`. Seeing `!okID`, `sendLoop` treats the canceled watch as "not yet registered" and appends its events to `pending[wid]`.
  5. If a client subsequently sends a `WatchCreateRequest` specifying `WatchId: wid` (explicitly allowed by `rpc.proto:809` and `watcher.go:102-136` once the prior `wid` is canceled), `sendLoop` receives the new `Created` message and flushes `pending[wid]` from the old watch onto the new watch.
- **Code Evidence (`server/etcdserver/api/v3rpc/watch.go:470-478, 530-555 @ v3.7.3`):**
  ```go
  // Event channel branch (lines 470-478):
  wid := wresp.WatchID
  if _, okID := ids[wid]; !okID {
      // buffer if watch not yet registered
      wrs := append(pending[wid], wr)
      pending[wid] = wrs
      continue
  }

  // Control channel branch (lines 530-555):
  wid := mvcc.WatchID(c.WatchId)
  if c.Canceled && wid != clientv3.InvalidWatchID {
      delete(ids, wid)
      continue
  }
  if c.Created {
      // flush buffered events
      ids[wid] = struct{}{}
      for _, v := range pending[wid] {
          if err := sws.gRPCStream.Send(v); err != nil {
              ...
              return
          }
      }
      delete(pending, wid)
  }
  ```

---

### 4.2 Storage-Layer Lock-Order Inversion in `progressIfSync` (`#22184`) and `EtcdServer.applyAll` Serialization

- **Summary:** Commit `973847aa2` (`#20229`, in `v3.7.3`) introduced a lock-order inversion in `watchableStore.progressIfSync()` by acquiring `watchableStore.mu.RLock()` *before* calling `s.rev()` (`store.mu.RLock()`), directly violating the documented lock hierarchy (`store.mu` before `watchableStore.mu`). However, auditing every `store.mu.Lock()` site in `server/` confirms that issue `#22184`'s claimed production trigger (`createMergedSnapshotMessage` calling `s.KV().Commit()` concurrently with `Write()`) does **not** occur in `EtcdServer` at `v3.7.3`, because `applyAll()` executes `Restore`, `Write`, `Compact`, and `Commit` sequentially on the single `s.applySched` FIFO goroutine.
- **Promises Affected:** Latent storage-layer deadlock of `watchableStore.mu` and `store.mu` (which, if triggered by any caller outside `s.applySched`, freezes the Raft apply loop and all watches while `s.Leader()` remains non-zero).
- **Step-by-Step Mechanism & Verification:**
  1. **The Documented Lock Hierarchy (`watchable_store.go:59-60`):** `watchableStore.mu` must never be acquired before `store.mu` (`kvstore.mu`).
  2. **Goroutine 1 (`watchableStoreTxnWrite.End()`):** `store.Write()` acquires `store.mu.RLock()`. At transaction end, `watchableStoreTxnWrite.End()` acquires `watchableStore.mu.Lock()` while still holding `store.mu.RLock()`:
     $$\text{holds } \texttt{store.mu.RLock()} \longrightarrow \text{waits for } \texttt{watchableStore.mu.Lock()}$$
  3. **Goroutine 2 (`progressIfSync()`):** Acquires `watchableStore.mu.RLock()` and then calls `s.rev()`, which calls `store.Read()` and acquires `store.mu.RLock()`:
     $$\text{holds } \texttt{watchableStore.mu.RLock()} \longrightarrow \text{waits for } \texttt{store.mu.RLock()}$$
  4. **Goroutine 3 (A concurrent `store.mu.Lock()` writer):** In Go's `sync.RWMutex`, a pending `Lock()` call on `store.mu` blocks behind Goroutine 1's `store.mu.RLock()` and simultaneously blocks new `RLock()` calls (Goroutine 2) from acquiring `store.mu.RLock()`, while Goroutine 1 blocks behind Goroutine 2's `watchableStore.mu.RLock()`.
  5. **Why `EtcdServer` Does Not Hit This Today (`server.go:972-993, 2101-2105`):** Inside `EtcdServer`, `applyAll()` runs on the single `s.applySched` FIFO goroutine (`server.go:844`). Within `applyAll()`, `s.applySnapshot()` (`Restore`), `s.applyEntries()` (`Write` and `Compact`), `s.snapshotIfNeededAndCompactRaftLog()` (`Commit`), and `s.createMergedSnapshotMessage()` (`Commit`) all execute **sequentially on that single goroutine** (`server.go:2103`: `// So KV().Commit() cannot run in parallel with toApply`). The only other `store.mu.Lock()` site is `compactBarrier` (`kvstore.go:146`), which runs on `s.fifoSched` only when a compaction was interrupted prior to completion. Thus the bug is a confirmed storage-layer (`mvcc`) defect and a latent trap if any future refactor moves `Commit()` or `Compact()` off `s.applySched`.
- **Code References (`v3.7.3`):**
  - Lock invariant & inversion: `server/storage/mvcc/watchable_store.go:59-60, 515-520`, `server/storage/mvcc/watchable_store_txn.go:43-46`, `server/storage/mvcc/kv_view.go:32`, `server/storage/mvcc/kvstore_txn.go:30, 46, 174`, `server/storage/mvcc/kvstore.go:146, 283, 297, 303`.
  - `applyAll` FIFO serialization: `server/etcdserver/server.go:844, 972-993, 1062-1069, 2101-2105`, `server/etcdserver/snapshot_merge.go:38`, `server/etcdserver/apply/backend.go:73`.

---

### 4.3 `syncWatchers` Scans Unbounded Revisions `[minRev, curRev+1)` Under Exclusive Lock (`#22264`, `#16839`)

- **Summary:** Every 100 ms, `syncWatchers()` acquires the exclusive `watchableStore.mu.Lock()`—blocking every Raft write commit in `watchableStoreTxnWrite.End()`—and scans `bbolt` from `minRev` all the way to `curRev + 1`, even though `eventBatch.add()` stops accepting events after `watchBatchMaxRevs = 1000` revisions.
- **Promises Affected:** Watch catch-up throughput and Raft apply-loop latency; triggers cascading watch victim overflow (`s.victims`).
- **Step-by-Step Mechanism:**
  1. `syncWatchers()` acquires `s.mu.Lock()` (`watchable_store.go:347`), selects up to `maxWatchersPerSync = 512` watchers from `s.unsynced`, and finds the minimum `minRev` across them (`lines 359-360`).
  2. It calls `s.store.rangeEvents(minRev, curRev+1, wg)` (`line 366`), which scans every revision in `bbolt` in `[minRev, curRev+1)` and unmarshals the `mvccpb.KeyValue` protobuf for every matching key (`lines 419-442`).
  3. As `rangeEvents` feeds events into `evs.add(ev)`, `eventBatch.add()` caps each batch at `watchBatchMaxRevs = 1000` distinct revisions and sets `eb.moreRev = evRev` (`watcher_group.go:53-60`).
  4. Despite the batch being full after 1,000 revisions, `rangeEvents` has no early exit and continues scanning and unmarshaling all the way to `curRev + 1`.
  5. A watcher reconnecting 500,000 revisions behind `curRev` forces `syncWatchers()` to scan 500,000 revisions under `s.mu.Lock()` to deliver 1,000, then 499,000 revisions on the next tick to deliver the next 1,000—performing $\mathcal{O}(N^2)$ bbolt work ($1.25 \times 10^8$ revision visits) while holding the exclusive lock that blocks the Raft apply loop.
- **Code Evidence (`server/storage/mvcc/watchable_store.go:346-414`, `watcher_group.go:28, 53-60 @ v3.7.3`):**
  ```go
  // watcher_group.go:53-60 @ v3.7.3
  if eb.revs > watchBatchMaxRevs {
      // max revisions exceeded
      eb.moreRev = evRev
      return
  }
  ```

---

### 4.4 Compacted Watchers Leaked in `s.unsynced` When $\ge 512$ Watchers Are Unsynced (`#22294`)

- **Summary:** When `len(s.unsynced.watchers) >= 512` (e.g., immediately after a follower with $\ge 512$ active watches restores a leader snapshot), `watcherGroup.choose()` copies 512 watchers into a temporary `ret` group and calls `ret.chooseAll()`. For any compacted watcher (`w.minRev < compactRev`), `ret.chooseAll()` deletes `w` from the temporary `ret` copy rather than from `s.unsynced`, trapping `w` in `s.unsynced` forever.
- **Promises Violated:** **Unique** cancellation (`CompactRevision` response sent repeatedly every 100 ms), **Bookmarkable** liveness (`len(s.unsynced.watchers) > 0` permanently disables `RequestProgress` across the entire server per §3.3), and bounded CPU/memory.
- **Step-by-Step Mechanism:**
  1. When `len(s.unsynced.watchers) >= maxWatchersPerSync` (`512`), `s.unsynced.choose()` allocates a new temporary `ret := newWatcherGroup()`, copies 512 watchers into `ret`, and calls `ret.chooseAll(curRev, compactRev)`.
  2. Inside `ret.chooseAll()`, any watcher `w` with `w.minRev < compactRev` receives `WatchResponse{WatchID: w.id, CompactRevision: compactRev}` on `w.ch`, and `ret.chooseAll()` calls `wg.delete(w)`—which deletes `w` from **`ret`**, not from `s.unsynced`.
  3. Because `w` was removed from `ret`, when `syncWatchers()` later iterates over `wg` (`ret`) at `watchable_store.go:368, 403` to remove processed watchers from `s.unsynced`, `w` is not in `ret`, so `s.unsynced.delete(w)` is never called either.
  4. Furthermore, when `sendLoop` (`v3rpc/watch.go:492`) sends the `CompactRevision` response to the client, it only executes `delete(ids, wid)` without calling `sws.watchStream.Cancel(wid)`. Unless the client tears down the entire gRPC stream or sends an explicit `WatchCancelRequest`, `w` stays in `s.unsynced` forever, emitting duplicate `CompactRevision` responses every 100 ms (which `sendLoop` buffers infinitely in `pending[wid]` per §4.1).
- **Code Evidence (`server/storage/mvcc/watcher_group.go:226-269 @ v3.7.3`):**
  ```go
  func (wg *watcherGroup) choose(maxWatchers int, curRev, compactRev int64) (*watcherGroup, int64) {
      if len(wg.watchers) < maxWatchers {
          return wg, wg.chooseAll(curRev, compactRev)
      }
      ret := newWatcherGroup()
      for w := range wg.watchers {
          if maxWatchers <= 0 {
              break
          }
          maxWatchers--
          ret.add(w)
      }
      return &ret, ret.chooseAll(curRev, compactRev)
  }

  func (wg *watcherGroup) chooseAll(curRev, compactRev int64) int64 {
      minRev := int64(math.MaxInt64)
      for w := range wg.watchers {
          ...
          if w.minRev < compactRev {
              select {
              case w.ch <- WatchResponse{WatchID: w.id, CompactRevision: compactRev}:
                  w.compacted = true
                  wg.delete(w) // deletes from temporary `ret` when len >= 512!
              default:
                  // retry next time
              }
              continue
          }
  ```

---

### 4.5 Silent Omission of `PrevKv` When Compaction Races With `sendLoop` (`#10681`)

- **Summary:** `WithPrevKV()` does not attach `PrevKv` when an event is created during transaction commit; instead, `sendLoop` looks up `ModRevision - 1` lazily right before sending the event on the gRPC wire and silently ignores `mvcc.ErrCompacted`.
- **Promises Violated:** **`WithPrevKV()` accuracy** (delivers update `PUT` and `DELETE` events with `PrevKv == nil` instead of canceling the watch with `ErrCompacted`, causing `kube-apiserver` errors in `kubernetes/kubernetes#76624` and `#123448`).
- **Step-by-Step Mechanism:**
  1. Suppose key `K` was created at revision `10`, updated or deleted at revision `100`, and the store is compacted at revision `90` (or at `100` while revision `100` is queued in `w.ch` or `s.victims`, or while `syncWatchers` catches up from revision `95`).
  2. Because `w.minRev` (`95` or `100`) is $\ge$ `compactRev` (`90`), `syncWatchers` does not cancel the watch.
  3. When `sendLoop` calls `sws.watchable.Range(K, nil, RangeOptions{Rev: 99})`, `Range` returns `mvcc.ErrCompacted`.
  4. `sendLoop` ignores `err != nil` and sends the event with `PrevKv: nil`.
- **Code Evidence (`server/etcdserver/api/v3rpc/watch.go:445-460 @ v3.7.3`):**
  ```go
  evs := wresp.Events
  for i := range evs {
      if sws.prevKV[wresp.WatchID] && !evs[i].IsCreate() {
          opt := mvcc.RangeOptions{Rev: evs[i].Kv.ModRevision - 1}
          r, err := sws.watchable.Range(evs[i].Kv.Key, nil, opt)
          if err == nil && len(r.KVs) != 0 {
              evs[i].PrevKv = &(r.KVs[0])
          }
      }
  ```

---

### 4.6 Silent Event Loss & Active Coalescing Poison-Pill in `grpcproxy` (`#21813`, `#19710`) and Readiness Hang in `etcd/cache` (`#22614`)

Two official etcd watch-multiplexing and caching packages contain severe liveness and event-loss bugs in `v3.7.3`:

#### 1. `grpcproxy`: Upstream Cancel Resets `wb.nextrev = 1`, Drops `Canceled`, and Actively Poisons Healthy Broadcasts (`#21813`, `#19710`)
- **Summary:** When an upstream `clientv3.WatchChan` in `grpcproxy` terminates with a non-compaction cancel (`Header.Revision == 0`), `watchBroadcast` resets its `wb.nextrev` cursor to `1`, silently swallows the `Canceled` response instead of forwarding it to downstream clients, leaves the dead `watchBroadcast` in `watchBroadcasts.bcasts`, and **actively migrates receivers from healthy broadcasts onto the dead broadcast while stopping the healthy ones**.
- **Step-by-Step Mechanism:**
  1. When an upstream watch closes with a non-compaction cancel, `clientv3`'s `closeSubstream` (`client/v3/watch.go:494`) emits `&WatchResponse{Header: &pb.ResponseHeader{Revision: 0}, Canceled: true, CompactRevision: 0}`.
  2. In `watchBroadcast`, `wb.bcast(wr)` (`watch_broadcast.go:83-85`) executes:
     ```go
     if wb.responses > 0 || wb.nextrev == 0 {
         wb.nextrev = wr.Header.Revision + 1
     }
     ```
     Because `wr.Header.Revision == 0`, **`wb.nextrev` is overwritten back to `0 + 1 = 1`**.
  3. Next, `wb.bcast(wr)` calls `r.send(wr)` (`watcher.go:107-110`) for each downstream watcher `r`:
     ```go
     if !wr.IsProgressNotify() && !wr.Created && len(events) == 0 && wr.CompactRevision == 0 {
         return
     }
     ```
     Because `wr.Canceled == true` (`!IsProgressNotify()`), `!wr.Created`, `len(events) == 0`, and `wr.CompactRevision == 0`, **`r.send(wr)` silently drops the `Canceled` response and returns early**.
  4. The upstream goroutine exits (`watch_broadcast.go:75`), closing `wb.donec` without removing `wb` from `watchBroadcasts.bcasts`.
  5. **Active Coalescing Poison-Pill:** Because the dead broadcast (`wbswb`) now has `wbswb.nextrev == 1` and `wbswb.responses > 0`:
     - Any new downstream watcher `w` with `w.nextrev >= 1` satisfies `wb.nextrev (1) <= w.nextrev` (`watch_broadcast.go:100`), receives a fabricated `Created: true` response, and attaches to the dead broadcast.
     - Whenever any **healthy** broadcast `wb` in the same key range receives an event and runs `wbs.coalesce(wb)` (`watch_broadcasts.go:67-79`), the check `if wb.nextrev >= wbswb.nextrev && wbswb.responses > 0` (`wb.nextrev >= 1`) is **always true**. `coalesce()` **moves all receivers from the healthy broadcast `wb` onto the dead broadcast `wbswb` and calls `wb.stop()` to kill the healthy broadcast**.
  6. Additionally, `grpcproxy` completely drops `WatchProgressRequest` messages (`server/proxy/grpcproxy/watch.go:293-296`, `#19710`).
- **Code References (`v3.7.3`):** `server/proxy/grpcproxy/watch_broadcast.go:55-128`, `server/proxy/grpcproxy/watcher.go:53-120`, `server/proxy/grpcproxy/watch_broadcasts.go:53-99`, `server/proxy/grpcproxy/watch.go:293-296`.

#### 2. Experimental `etcd/cache`: Permanent Readiness Hang After Compaction Resync (`#22614`)
- **Summary:** When `etcd/cache` resyncs after compaction, `c.demux.Init(rev)` purges the newly registered storage watcher `storeW`, causing `watchEvents` to call `c.ready.Reset()` and loop around with an already-spent `sync.Once` (`readyOnce`), leaving `Cache.Ready()` permanently `false`.
- **Step-by-Step Mechanism & Code References (`cache/cache.go:348-369, 405-421`, `cache/demux.go:119-143, 296-308 @ v3.7.3`):**
  1. `c.watch(rev)` initializes `readyOnce := sync.Once{}` *outside* its `for` loop (`cache.go:349`).
  2. Inside the loop, it registers `storeW` via `c.demux.Register(storeW, rev)` (`line 352`) *before* `c.demux.Init(rev)` is called inside `watchEvents` (`line 406`).
  3. On a post-compaction resync (`d.minRev != 0` and `rev != d.maxRev + 1`), `c.demux.Init(rev)` (`demux.go:130-143`) calls `d.purge()` (`lines 296-308`), which stops all registered watchers—including the just-registered `storeW`.
  4. Stopping `storeW` closes `applyErr` (`err == nil`), causing `watchEvents` (`cache.go:418-420`) to call `c.ready.Reset()` and return `nil`. The outer `for` loop starts a new iteration with the already-executed `readyOnce`, so `c.ready.Set()` can never run again.

---

### 4.7 Bidirectional Stream Deadlock on Mass Watch Cancellation (`#18879`, `#20716`, PR `#21065`)

- **Summary:** Because `clientv3` multiplexes all watches with identical context metadata onto a single `watchGRPCStream`, cancelling a burst of watches ($> 16$) while events or control responses are in flight creates a 4-goroutine cyclic deadlock between `clientv3` and `serverWatchStream` that freezes every watch on the stream.
- **Promises Affected:** Stream liveness for all active and newly created watches sharing that `watchGRPCStream`.
- **The 4-Goroutine Deadlock Cycle:**
  1. **Client `watchGRPCStream.run()` (`client/v3/watch.go:685-706`):** Single-goroutine `select` loop that both reads incoming responses from `w.respc` (`line 548`) and sends `WatchCancelRequest` synchronously via `wc.Send(req)` inside `case ws := <-w.closingc:` (`line 703`). When TCP/HTTP2 send buffers fill, `run()` blocks inside `wc.Send(req)` and stops draining `w.respc`.
  2. **Client `serveWatchClient()` (`client/v3/watch.go:770-790`):** Reads from `wc.Recv()` and writes to **unbuffered** `w.respc <- wresp` (`lines 258, 789`). Because `run()` is blocked in `wc.Send()`, `serveWatchClient()` blocks on `w.respc <- wresp` and stops calling `wc.Recv()`.
  3. **Server `sendLoop()` (`server/etcdserver/api/v3rpc/watch.go:500, 519`):** Calls `sws.gRPCStream.Send(...)`. Once transport buffers fill (immediately on the 1-slot `chanStream` adapter at `adapter/chan_stream.go:145` in `#20716`, or when HTTP/2 flow-control windows fill in `#18879`), `sendLoop()` blocks in `Send()` and stops draining `sws.ctrlStream`.
  4. **Server `recvLoop()` (`server/etcdserver/api/v3rpc/watch.go:367, 383`):** Reads `CancelRequest` from `sws.gRPCStream.Recv()` and writes the cancel response into `sws.ctrlStream` (capacity `ctrlStreamBufLen = 16`, `watch.go:42, 122`). On the 17th cancel response while `sendLoop()` is blocked, `recvLoop()` blocks at `sws.ctrlStream <- wr` (`line 383`) and stops calling `Recv()`, which blocks the client's `wc.Send(req)` forever.
- **Code References (`v3.7.3`):** `client/v3/watch.go:255, 258, 343, 548, 685-706, 770-790`; `server/etcdserver/api/v3rpc/watch.go:42, 122, 367, 383, 500, 519`; `server/proxy/grpcproxy/adapter/chan_stream.go:107, 145`.

---

### 4.8 Operator Snapshot Restore Without Revision Bump Turns Resumed Watches Into Silent Future Watches (`#14303`)

- **Summary:** When an operator restores a cluster from an older backup snapshot (e.g., at revision `500` after the cluster had reached revision `1000`), `etcdutl snapshot restore` defaults to `--bump-revision=0` and `--mark-compacted=false`. Reconnecting clients resume at `StartRevision = 1001`, which the restored cluster accepts as a valid "future revision" watch, silently ignoring all new writes committed at revisions `501..1000`.
- **Promises Affected:** **Reliable** and **Resumable** across disaster-recovery restores.
- **Mechanism & Code References (`v3.7.3`):**
  - Neither `clientv3` (`client/v3/watch.go`) nor `jetcd` (`WatchImpl.java`) checks `Header.ClusterId` across reconnects.
  - `etcdutl/etcdutl/snapshot_command.go:83-84, 133-136 @ v3.7.3` defaults `--bump-revision` to `0`.
  - When the client reconnects with `StartRevision = 1001` against the restored store at revision `500`, `watchableStore.watch()` (`server/storage/mvcc/watchable_store.go:141`) sees `startRev (1001) > s.store.currentRev (500)` and places the watcher in `s.synced` without error. Only when `--bump-revision <delta> --mark-compacted` is explicitly passed does the restored cluster force clients to relist via `ErrCompacted`.

---

## 5. Client-Side Watch State Machines: Go `clientv3` vs. Java `jetcd`

Because etcd's server-side watch streams are stateless across reconnections, the client library's state machine is solely responsible for tracking the resumption revision (`nextRev` / `this.revision`) and re-establishing broken streams.

### 5.1 Java `jetcd` (`0.8.7`): Silent Loss of Historical Events on Reconnect After `Created`

#### Finding
When `jetcd` opens (or resumes) a watch at an explicit historical revision `startRevision = R` against an etcd server currently at revision $H > R$, `jetcd` overwrites `this.revision = Math.max(revision, response.getHeader().getRevision())` ($H$) as soon as the initial `Created: true` message arrives—**before a single historical event in $[R, H]$ has been delivered**. If the gRPC stream disconnects before `syncWatchersLoop` finishes streaming those historical events, `jetcd` reconnects with `start_revision = H`, **silently and permanently dropping all events in $[R, H - 1]$**.

#### Step-by-Step Trace (`WatchImpl.java @ jetcd-0.8.7`)
1. Suppose a Java application opens a watch at `startRevision = 50` (e.g., after a `Get` at revision `49`, or while reconnecting a watch that had reached revision `49` before a leader failover) against an etcd member whose current revision is `100`.
2. `WatcherImpl.resume()` (`WatchImpl.java:179-185`) sends `WatchCreateRequest { start_revision: 50 }`.
3. On the server (`v3rpc/watch.go:340-366`), `watchStream.Watch(..., startRev=50)` places the watcher in `s.unsynced` with `minRev = 50` (to be caught up asynchronously by `syncWatchersLoop` every 100 ms in batches of up to 1,000 revisions) and **immediately sends `WatchResponse { Created: true, Header: { Revision: 100 } }` on `ctrlStream`**.
4. Upon receiving `Created`, `jetcd`'s `processCreated()` (`WatchImpl.java:256-268`) executes line 267:
   ```java
   revision = Math.max(revision, response.getHeader().getRevision());
   ```
   **`this.revision` immediately jumps from `50` to `100` before any event in `[50..100]` has arrived.**
5. **Two ways this loses events on disconnect:**
   - *Disconnect before Batch 1 arrives:* If the stream breaks before the first `syncWatchersLoop` batch arrives, `resume()` reconnects with `start_revision = 100`, losing all events in `[50..99]`.
   - *Disconnect during multi-batch catch-up (>1,000 revisions behind):* Even after Batch 1 arrives, `syncWatchersLoop` (`watchable_store.go:417`) stamps *every* catch-up batch's `Header.Revision` with the store's current tip (e.g., `5000`), while `processEvents()` (`WatchImpl.java:315`) sets `this.revision = lastEvent.getModRevision() + 1` (e.g., `1100`). If the stream disconnects after Batch 1, `resume()` sends `start_revision = 1100`, immediately receives `Created { Header.Revision: 5000 }` on the new stream (jumping `this.revision` to `5000` again!), and any second blip before Batch 2 arrives drops `[1100..4999]`.

#### Historical Origin and Why Tests Missed It
- **Origin (`jetcd#946`, commit `70be01a`, July 2021):** Originally, `processCreated` executed `revision = response.getHeader().getRevision()`. When `jetcd#946` reported that a future-revision watch (`revision > Header.Revision`) had its cursor lowered by `Created`, commit `70be01a` changed the line to `Math.max(revision, response.getHeader().getRevision())`—fixing future watches while leaving historical watches (`0 < revision < Header.Revision`) broken.
- **Why Go `clientv3` is not affected for historical watches:** Go `clientv3` (`client/v3/watch.go:858 @ v3.7.3`) guards its `Created` assignment with `if ws.initReq.rev == 0 { nextRev = wr.Header.Revision }`, preserving `ws.initReq.rev` whenever an explicit starting revision was provided.
- **Why `jetcd` tests missed it (`WatchResumeTest.java:37-69`, `Etcd.java:31 @ jetcd-0.8.7`):** `WatchResumeTest` only tests `watchClient.watch(key, ...)` with the default `startRevision == 0`, never testing reconnection of a watch started at $0 < R < \text{currentRev}$. Furthermore, `jetcd-launcher` (`Etcd.java:31`) runs against `quay.io/coreos/etcd:v3.5.14`, which predates the server-side watch fixes in `v3.5.16`, `v3.5.18`, and `v3.5.22`.

- **Code Evidence (`jetcd-core/src/main/java/io/etcd/jetcd/impl/WatchImpl.java:175-185, 256-318 @ jetcd-0.8.7`):**
  ```java
  private void processCreated(io.etcd.jetcd.api.WatchResponse response) {
      ...
      if (response.getWatchId() == -1) {
          listener.onError(newWatcherCreateException());
          this.close();
          return;
      }
      revision = Math.max(revision, response.getHeader().getRevision());
      watchId = response.getWatchId();
  }
  ```

---

### 5.2 Additional State-Machine Defects in `jetcd` (`0.8.7`) and `clientv3` (`v3.7.3`)

1. **`jetcd` Off-by-One Revision Update on `ProgressNotify` (`WatchImpl.java:291-313 @ jetcd-0.8.7`):**
   When `jetcd` receives an empty `ProgressNotify` response (`response.getEventsCount() == 0`), it sets:
   ```java
   if (response.getEventsCount() == 0 && this.option.isProgressNotify()) {
       revision = Math.max(revision, response.getHeader().getRevision());
       listener.onNext(new WatchResponse(response));
   } else if (response.getEventsCount() == 0) {
       revision = response.getHeader().getRevision();
   }
   ```
   Because `start_revision` in `WatchCreateRequest` is **inclusive** (`rpc.proto:784`), a `ProgressNotify` at `Header.Revision = 100` means all events $\le 100$ have already been delivered, so the next watch must start at **`100 + 1 = 101`** (which Go `clientv3` does at `client/v3/watch.go:864` after commit `36fc3cae6`). By omitting `+ 1`, `jetcd` resumes at `100` after a disconnect and **re-delivers a duplicate event if a write occurred at revision `100`**, violating **Unique**. Furthermore, if `option.isProgressNotify()` is `false` (line 312), `revision = response.getHeader().getRevision()` lacks `Math.max` and can regress `this.revision` backwards.
2. **`jetcd` Uncaught Exception in `reschedule()` Leaves Watcher Permanently Dead (`WatchImpl.java:357-368 @ jetcd-0.8.7`, `jetcd#1227`):**
   When a stream disconnects with a retryable status, `reschedule()` schedules `this::resume` after 500 ms. If `this::resume` throws any synchronous runtime exception, `Futures.addCallback`'s `onFailure` handler (`lines 362-366`) only logs `LOG.warn("scheduled resume failed", t)` without calling `reschedule()`, `listener.onError(t)`, or `close()`, leaving `WatcherImpl` open (`closed == false`) but permanently dead.
3. **`jetcd` Missing Watch Fragmentation Support (`WatchImpl.java:164-185 @ jetcd-0.8.7`):**
   `WatcherImpl.resume()` never calls `.setFragment(...)` on `WatchCreateRequest.Builder`, so large multi-key transactions or catch-up batches cannot be fragmented across responses.
4. **Go `clientv3` Unbounded Per-Substream Memory Buffer (`client/v3/watch.go:818-885 @ v3.7.3`):**
   In `clientv3`, `serveSubstream` appends incoming responses to an **unbounded** slice `ws.buf = append(ws.buf, wr)` whenever the caller has not drained the 1-slot `ws.outc` channel, allowing a single slow consumer to grow client heap memory without bound.

---

### 5.3 Side-by-Side Comparison Matrix: Go `clientv3` vs. Java `jetcd`

| Dimension / Behavior | Go `clientv3` (`v3.7.3`) | Java `jetcd` (`jetcd-0.8.7`) |
|---|---|---|
| **Stream Multiplexing** | Multiplexes all watches with identical `streamKeyFromCtx(ctx)` onto 1 `watchGRPCStream` (`watch.go:1086-1091`); 1 burst cancellation (`#18879`) stalls all watches on that stream. | Opens 1 dedicated gRPC `Watch` stream per `WatcherImpl` (`WatchImpl.java:186-194`). |
| **`WithRequireLeader` Default** | Off by default (`client/v3/ctx.go:39`). | Off by default (`requireLeader = false`, `WatchOption.java:154`). |
| **HTTP/2 Keepalive Default** | **Off by default** (`DialKeepAliveTime == 0`, `client.go:231-236`); hangs forever on `SIGSTOP`/black-holed member unless configured. | **On by default** (`keepaliveTime = 30s`, `keepaliveTimeout = 10s`, `keepaliveWithoutCalls = true`, `ClientBuilder.java:66-68`). |
| **Resume Cursor After `Created`** | Sets `nextRev = wr.Header.Revision` (missing `+ 1`) **only when `ws.initReq.rev == 0`** (`watch.go:858-860`). Preserves `ws.initReq.rev` for explicit-revision watches. | Sets `revision = Math.max(revision, response.getHeader().getRevision())` on **all** watches (`WatchImpl.java:267`), **silently skipping `[startRev, Header.Revision)` on reconnect**. |
| **Resume Cursor After `ProgressNotify`** | Correctly sets `nextRev = wr.Header.Revision + 1` (`watch.go:864`). | **Off-by-one bug:** sets `revision = Math.max(revision, Header.Revision)` (`WatchImpl.java:293`) or `revision = Header.Revision` (`line 312`) without `+ 1`, replaying revision `Header.Revision` on reconnect. |
| **Behavior on `ErrNoLeader`** | Overrides `isHaltErr` to terminally close all substreams (`watch.go:667`), discards `ws.buf` (`line 835`), emits `Header.Revision = 0` (`line 495`), and drops `ErrNoLeader` if `ws.outc` is full for $> 250\text{ ms}$ (`line 478`). | Treats `Errors.isNoLeaderError(status)` as fatal (`WatchImpl.java:354`), calls `listener.onError()`, and permanently closes the watcher (`#1352`). |
| **Behavior on Transient Disconnect / Unreachable Cluster** | Silently retries in background forever (`watch.go:85-91, 1039`) with no error surfaced to caller. | Calls `listener.onError()` on each failure and schedules `this::resume` after `500ms` (`WatchImpl.java:334, 347, 358`). If `resume()` throws, logs a warning and abandons the watcher (`#1227`). |
| **Fragmentation & Buffering** | Supports `WithFragment()` (`watch.go:590`); buffers unread responses in an unbounded slice `ws.buf` (`watch.go:879-880`). | Does not support `WithFragment()` (`WatchImpl.java:164-185`). |

---

## 6. Production Impact: Kubernetes Control-Plane Defenses and Residual Hazards (`v1.37.2`)

Kubernetes uses etcd Watch as the foundation of `kube-apiserver`'s `cacher` / `watchCache`. Inspecting the Kubernetes `v1.37.2` storage layer (`staging/src/k8s.io/apiserver/pkg/storage/{etcd3,cacher,storagebackend}/`) shows how Kubernetes has had to build explicit application-level workarounds for etcd's watch failure modes—and where residual hazards remain:

1. **Mandatory `WithRequireLeader`, HTTP/2 Keepalives, and Matching `RequestProgress` Context:**
   - To prevent watches from hanging forever on partitioned or paused etcd members (`#89488`, `#8980`), `kube-apiserver` wraps every watch context in `clientv3.WithRequireLeader(ctx)` (`store.go:1156-1165`, PR `#89881`) and configures gRPC keepalives to `30s` / `10s` (`factory/etcd3.go:63-64`).
   - To avoid Failure #6 (`#12254`, where `RequestProgress` without `WithRequireLeader` targets an empty gRPC stream), `kube-apiserver` saves `s.watchContext` and passes it to `RequestProgress(s.watchContext)` (`store.go:102-106`).
   - **Operational cost (`kubernetes/kubernetes#111116`):** Because `clientv3` treats `ErrNoLeader` as fatal (Failure #1) and `newStreamInterceptor` gives zero grace period to reconnecting streams during an election (Failure #9), any etcd election or leader crash that hits `ErrGRPCNoLeader` forces `kube-apiserver` reflectors to tear down and re-list across the cluster.
2. **Explicit `WithRev(initialRev + 1)`, Bare-Channel-Close Handling, and `PrevKv == nil` Rejection:**
   - `kube-apiserver` never uses `startRev == 0` for its cacher reflectors; it lists at `initialRev` and watches at `WithRev(initialRev + 1)` with `WithPrevKV()` (`watcher.go:470-477`), avoiding Failure #2 and the `startRev == 0` `Created` race (§3.2).
   - When `for wres := range wch` exits (`watcher.go:521`), `kube-apiserver` closes `watchClosedCh` (`watcher.go:269`) and triggers a reflector relist (`cacher.go:503`) even if `clientv3`'s 250 ms `sendCloseSubstream` timeout dropped the `Canceled: true` error response (Failure #4).
   - When etcd bug `#10681` (§4.5) delivers a non-create event with `PrevKv == nil` due to compaction, `parseEvent` (`event.go:64-67`) returns an explicit error (`"kv is nil in delete event"`), aborting the watch and forcing a relist rather than corrupting `watchCache` (`kubernetes/kubernetes#76624`, `#123448`).
3. **Why Kubernetes Could Not Trust `RequestProgress` Alone for Consistent Cached Reads (`KEP-2340`):**
   - Because etcd's `RequestProgress()` bypasses Raft `ReadIndex` and returns the watch-serving member's unfenced local revision (§1.2), Kubernetes could not implement `ConsistentReadsFromCache` (KEP-2340) using `RequestProgress()` alone.
   - Instead, `kube-apiserver` first issues a linearizable `KV.Get(ctx, prefix, WithCountOnly(), WithLimit(1))` (`store.go:728-745`, `tests/robustness/traffic/kubernetes.go:118-137 @ v3.7.3`) solely to force a Raft `ReadIndex` and obtain the cluster's true revision $R_{\text{linearizable}}$. If `watchCache` is behind $R_{\text{linearizable}}$, `ConditionalProgressRequester` polls `RequestProgress` every `100ms` (`watch_progress.go:36, 83-115`) for up to `3s` (`watch_cache.go:49, 318-360`), falling back to a direct etcd read (`delegator.go:160-172`) if the watch stream fails to catch up (e.g., due to `#20242` progress starvation in §3.3 or `#22264` `syncWatchers` stalls in §4.3).
4. **Unresolved `resourceVersion=0` Staleness Hazard (`watch_cache.go:333-339`, `kubernetes/kubernetes#59848`):**
   - For reads with `resourceVersion="0"`, `watch_cache.go:333-339` bypasses the KEP-2340 freshness check (`// In case resourceVersion is 0, we accept arbitrarily stale result`). During the 3–5 s window before `WithRequireLeader` cancels a partitioned stream—or indefinitely during an apply-loop stall where `s.Leader() != raft.None`—`watchCache` serves arbitrarily stale objects to `resourceVersion=0` callers (`kubernetes/kubernetes#59848`).

---

## 7. Architectural Comparison with Google's Chubby

Comparing etcd's Watch subsystem with Google's Chubby lock service (`doc/chubby-osdi06.md`, §2.5, §2.7–§2.9, §4.5–§4.6) highlights how the two systems make opposite trade-offs between **event continuity across failover** and **session-fenced cache freshness**:

| Design Dimension | Chubby (`doc/chubby-osdi06.md`) | etcd (`v3.7.3` / `jetcd-0.8.7`) |
|---|---|---|
| **Primary Client Cache Consistency Mechanism** | **Master-driven blocking cache invalidations** piggybacked on Master-bound Session KeepAlives (§2.7–§2.8, pp. 6–8). Before committing a write, the Chubby master blocks the write until every client caching that file/directory acknowledges invalidation on its KeepAlive reply or its session lease expires. | **Asynchronous MVCC Watch streams** (`Watch` RPC). Writers never block on watchers; watchers stream asynchronously from any replica's MVCC history (`[compactMainRev, currentRev]`). |
| **Role of Events (`Watch`)** | **Advisory, best-effort hints** (§2.5, p. 5; §4.5, pp. 13–14). Burrows explicitly notes that Chubby avoided using events as a general publish/subscribe replication log. | **Primary state-replication log** (*Ordered, Unique, Atomic, Reliable, Resumable, Bookmarkable*), used by Kubernetes `watchCache`, `etcd/cache`, and `client/v3/leasing` to maintain in-memory state replicas. |
| **Where Streams Terminate** | **Strictly at the Master** (§2.2, §2.8, pp. 3, 7). Replicas do not serve client sessions. | **Any member (leader or follower) or `grpcproxy`**, without Raft or `ReadIndex` coordination (`server/etcdserver/api/v3rpc/watch.go`). |
| **Coupling Between Event Delivery and Liveness** | **Tightly coupled** (§2.8, pp. 7–8): invalidations and events ride on Session KeepAlive replies. If the master is lost, the client's local lease timer enters **Jeopardy** (45 s grace period), immediately disabling the client cache and blocking API calls so the application cannot act on stale state. | **Completely decoupled**: a watch stream has no session lease. Without `WithRequireLeader` (default), a partitioned watch stays open forever; with `WithRequireLeader`, it checks only local `s.Leader() == raft.None`, fails in `grpcproxy`, and has no jeopardy signal before destroying the watch. |
| **Behavior on Master / Leader Failover** | **Explicit `fail-over` event (lossy by design)** (§2.5, §2.9 step 5, §4.6, pp. 5, 9, 14). On master failover, Chubby flushes caches and emits a `fail-over` event warning applications to rescan. In §4.6 (p. 14), Burrows reflects: *"We might have done better... to ensure that no events were lost during a fail-over."* | **Revision-indexed lossless history (`Resumable`) paired with fragile client cursor tracking.** Etcd's replicated MVCC log provides the exact lossless failover capability Burrows wished for in §4.6—provided the client cursor (`nextRev`) and replica state remain consistent. |

---

## 8. Master Verification Matrix and Open Follow-Up Targets

### 8.1 Master Verification Matrix of Watch Failure Modes

| Failure Mode | Section | Promise(s) Violated / Affected | Affected Versions | Verification Source (`v3.7.3` / `jetcd-0.8.7`) |
|---|---|---|---|---|
| Watcher on partitioned member without `WithRequireLeader` stalls forever while receiving unfenced `ProgressNotify` (`#17611`) | §1.2, §2.2, §3.1 | Liveness & freshness (contract gap fatal for state caches) | `etcd v3.7.3`, `clientv3 v3.7.3`, `jetcd 0.8.7` | `interceptor.go:239-258`, `v3rpc/watch.go:395-400, 558-570`, `watchable_store.go:507-540` |
| `WithRequireLeader` closes client watch channel terminally on `ErrNoLeader` instead of failing over to surviving quorum (`#8980`, `#12534`, `k8s#111116`) | §3.1 (Failure #1) | Resumable / transparent failover | `clientv3 v3.7.3`, `jetcd 0.8.7` | `client/v3/watch.go:665-670`, `WatchImpl.java:353-355` |
| `ErrNoLeader` teardown discards `ws.initReq.rev` and emits `Header.Revision == 0` on `startRev == 0` watches, losing writes committed during partition | §3.1 (Failure #2) | **Reliable**, **Resumable** | `clientv3 v3.7.3` | `client/v3/watch.go:494-496, 667, 846-872` |
| Partitioned member's gRPC `SubConn` stays `READY` after `ErrGRPCNoLeader` trailer; `openWatchClient()` returns `err == nil` on `NewStream` and fails on `Recv()` | §3.1 (Failure #3) | Resumable / manual retry liveness | `clientv3 v3.7.3` | `client/v3/watch.go:772, 1039-1062`, `network_partition_test.go:228-238` (`a57e967d84`) |
| `ErrNoLeader` teardown discards queued events in `ws.buf` and drops `Canceled: true` / `ErrNoLeader` response after `250ms` (`closeSendErrTimeout`) | §3.1 (Failure #4) | **Reliable**, terminal error delivery contract (`watch.go:57-70`) | `clientv3 v3.7.3` | `client/v3/watch.go:40, 478-485, 520-541, 824-836` |
| `ErrNoLeader` racing with rejected/canceled `Created` substream deadlocks `watchGRPCStream.run()` at `w.wg.Wait()` (`#17611`, `#17622`) | §3.1 (Failure #5) | Client liveness (permanent goroutine deadlock) | `clientv3 v3.7.3` | `client/v3/watch.go:466-476, 520-541, 601-609, 803` |
| `RequestProgress(ctx)` without `WithRequireLeader(ctx)` opens a separate empty `watchGRPCStream` (`#12254`) | §3.1 (Failure #6) | **Bookmarkable** | `clientv3 v3.7.3` | `client/v3/watch.go:330, 413-428, 1086-1091` |
| Apply-loop stall (`s.Leader() != raft.None`), `SIGSTOP` with `DialKeepAliveTime == 0`, or total client isolation hangs watch forever | §3.1 (Failure #7) | Liveness | `etcd v3.7.3`, `clientv3 v3.7.3` | `interceptor.go:329`, `server.go:843-845`, `client.go:231-236`, `watch.go:85-91, 274, 1039` |
| `grpcproxy` reads `metadata.FromOutgoingContext` instead of `FromIncomingContext`, ignoring `WithRequireLeader` (`PR #7896`), and splits `__lostleader` across streams | §3.1 (Failure #8) | Documented `WithRequireLeader` contract (`watch.go:78-83`) | `etcd v3.3.0`–`v3.7.3` | `grpcproxy/watch.go:112`, `lease.go:161`, `leader.go:42-80`, `watch_broadcast.go:58-68`, `chan_stream.go:142-155` (`5e059fd8d`) |
| `newStreamInterceptor` gives zero grace period (`s.Leader() == raft.None`) to new streams, killing failover watches during routine 1s leader elections | §3.1 (Failure #9) | Resumable / transparent failover (`k8s#111116`) | `etcd v3.7.3`, `clientv3 v3.7.3`, `jetcd 0.8.7` | `interceptor.go:239-242`, `client/v3/watch.go:667, 678` |
| `startRev == 0` unlocked `Created` header read (`v3rpc/watch.go:340, 358`) + `clientv3` `nextRev = wr.Header.Revision` (`client/v3/watch.go:859`) | §3.2 | **Reliable** (drops $[R_1+1, R_2-1]$) or **Unique/Ordered** (replays $R_1$) | `etcd v3.7.3`, `clientv3 v3.7.3` | `v3rpc/watch.go:340, 358`, `client/v3/watch.go:845-872` |
| Failover to lagging follower (`startRev > s.store.currentRev`) regresses `nextRev` via `ProgressNotify` or snapshot `Restore` (`#20221`, `#19487`, `#15271`) | §3.3, §3.4 | **Bookmarkable**, **Ordered**, **Unique**, **Reliable** | `< v3.5.22`, `< v3.6.2`, `< v3.7.0` (fixed in `v3.7.3`) | `watchable_store.go:141, 374, 486, 525-527` (`cf0369e4a`, `973847aa2`) |
| Server-wide `RequestProgress` / `ProgressNotify` suppression when any watcher on the server is in `unsynced`, `victims`, or `rev < w.startRev` (`#20242`, `#20240`) | §3.3 | **Bookmarkable** liveness | `etcd v3.7.3` | `watchable_store.go:515-539`, `v3rpc/watch.go:558-568` |
| Canceled watch events buffered in `sendLoop.pending[wid]` and flushed to a reused `WatchId` or leaked (`#22673`) | §4.1 | **Filter**, **Ordered**, **Unique**, **Reliable**; memory | `etcd v3.7.3` | `v3rpc/watch.go:470-477, 534-553` |
| Lock-order inversion in `progressIfSync` (`watchableStore.mu.RLock` $\to$ `store.mu.RLock`, `#22184`) | §4.2 | Storage-layer deadlock (`mvcc`), serialized in `EtcdServer` by `s.applySched` | `etcd v3.7.3` | `watchable_store.go:59-60, 516-519`, `watchable_store_txn.go:43-46`, `server.go:972-993, 2101-2105` |
| `syncWatchers` scans unbounded `[minRev, curRev+1)` under `watchableStore.mu.Lock()` while delivering at most 1,000 revisions (`#22264`, `#16839`) | §4.3 | Apply-loop latency & watch catch-up liveness | `etcd v3.7.3` | `watchable_store.go:346-414`, `watcher_group.go:28, 57-60` |
| Compacted watchers leaked in `s.unsynced` when $\ge 512$ watchers are unsynced (`#22294`) | §4.4 | **Unique** cancellation, **Bookmarkable** liveness, CPU/memory | `etcd v3.7.3` | `watcher_group.go:226-269`, `watchable_store.go:215-218, 368, 403` |
| `WithPrevKV()` silently returns `PrevKv == nil` when `ModRevision - 1` was compacted (`#10681`, `k8s#76624`, `k8s#123448`) | §4.5 | `WithPrevKV` accuracy | `etcd v3.7.3` | `v3rpc/watch.go:450-456` |
| `grpcproxy` resets `wb.nextrev = 1` on cancel, drops `Canceled` (`watcher.go:107`), actively coalesces healthy broadcasts onto dead broadcast (`#21813`), and drops `RequestProgress` (`#19710`) | §4.6 | **Reliable** (silent total event loss & active poisoning), **Bookmarkable** | `etcd v3.7.3` | `watch_broadcast.go:55-128`, `watcher.go:107-110`, `watch_broadcasts.go:53-99`, `grpcproxy/watch.go:293-296` |
| `etcd/cache` permanently wedges as not-ready after compaction resync (`#22614`) | §4.6 | Cache liveness after compaction | `etcd v3.7.3` | `cache/cache.go:349-369, 405-421`, `cache/demux.go:119-143, 296-308` |
| 4-way cancellation deadlock between `watchGRPCStream.run()`, `serveWatchClient`, `recvLoop`, and `sendLoop` (`#18879`, `#20716`, PR `#21065`) | §4.7 | Stream liveness for all watches sharing the `watchGRPCStream` | `etcd v3.7.3`, `clientv3 v3.7.3` | `client/v3/watch.go:343, 548, 703, 789`, `v3rpc/watch.go:42, 122, 383, 519` |
| Operator `etcdutl snapshot restore` without `--bump-revision --mark-compacted` rolls revision backwards while clients resume at old revision (`#14303`) | §4.8 | **Reliable**, **Resumable** (watches silently wait for future revision) | `etcd v3.7.3`, `clientv3 v3.7.3`, `jetcd 0.8.7` | `etcdutl/snapshot_command.go:83-84`, `watchable_store.go:141` |
| `jetcd` overwrites `this.revision = Math.max(revision, Header.Revision)` on `Created` (`WatchImpl.java:267`, commit `70be01a`) | §5.1 | **Reliable**, **Resumable** (silently drops `[startRev, serverRev - 1]` on reconnect) | `jetcd 0.8.7` | `WatchImpl.java:179-181, 256-268` (`70be01a`) |
| `jetcd` off-by-one `ProgressNotify` revision (`missing + 1`) and uncaught exception in `reschedule()` (`#1227`) | §5.2 | **Unique** (replays `Header.Revision`); liveness (watcher left open and dead) | `jetcd 0.8.7` | `WatchImpl.java:293, 312, 357-368` |

### 8.2 Open Questions and Follow-Up Reproduction Targets

1. **Minimal Functional Reproducers for Client Event Loss and Proxy Poisoning:**
   - *`clientv3` `WithRequireLeader` event loss (§3.1 Failure #2) and `250ms` error drop (§3.1 Failure #4):* Verify both the skipped-write schedule on `startRev == 0` watches across a 1-node minority partition and the silent drop of `ErrNoLeader` when `ws.outc` is held full for $> 250\text{ ms}$.
   - *`jetcd` historical event loss (§5.1):* Cut the gRPC stream immediately after `Created(Header.Revision = H)` for a watch opened at `startRevision = R < H` (before the 100 ms `syncWatchersLoop` batch arrives) and confirm that `resume()` skips $[R, H)$.
   - *`grpcproxy` active coalescing poison-pill (§4.6):* Trigger upstream watch cancellation on one broadcast, then emit an event on a second healthy broadcast for the same key range and confirm that `coalesce()` migrates the healthy broadcast's receivers onto the dead `wb.nextrev == 1` broadcast and stops the healthy broadcast.
2. **Unresolved Antithesis `Resumable` Failure (`#20573`):** Issue `#20573` (a snapshot-restore `Resumable` violation discovered by Antithesis in 2025) was closed as stale in November 2025 without an identified root cause; whether a subtle race remains in `watchableStore.Restore()` remains open.
3. **Upstream Direction on Leader-Bound Watches and Apply-Stall Health Checks (`#22268`, `#22635`):** Upstream `main` recently merged an advisory `leader_id` field in `ResponseHeader` (`ee374bdc1`, `7135f1a61`, targeting v3.8) and proposed `apply_stall` / `apply_lag` health checks (`#22635`), but neither client-side leader-aware watch routing nor apply-progress-based stream cancellation is implemented.
4. **Benchmarking `syncWatchers` Lock Contention (`#22264`):** Once dedicated benchmark hardware is available (`AGENTS.md` §7), measuring Raft apply-loop tail latency (`Put` p99) while lagging watchers catch up across large revision spans will quantify the production impact of `syncWatchers()` holding `watchableStore.mu.Lock()`.

---

## Conclusions

1. **Watch's Architectural Strength Is Its Raft-Backed MVCC Log; Its Architectural Weakness Is Its Unfenced Stream Control Plane:**
   Because etcd commits every key-value mutation through Raft into each replica's multi-version `bbolt` store under a cluster-wide monotonic `Revision`, historical watch events survive leader failovers without needing to replicate watcher registrations through Raft. That design directly solves the lossy failover limitation that Burrows noted in Chubby (§4.6). However, etcd undermines this strength by allowing any replica (leader, lagging follower, or partitioned member) to terminate watch streams, bind `startRev == 0`, and emit `ProgressNotify` bookmarks from its **unsynchronized local state** without Raft `ReadIndex` or session leases.
2. **Claiming "Eventual Consistency" While Admitting Events "Might Never Arrive" in a Quorum-Healthy Cluster Is a Fundamental Contradiction That Forces Clients Back to Polling:**
   Etcd explicitly claims that Watch implements **eventual consistency** (`tests/robustness/README.md:111 @ v3.7.3`), which its own specification defines as the liveness property that components *"converge to the same view eventually"* (`tests/robustness/README.md:102 @ v3.7.3`). Yet `etcd-io/website` `content/en/docs/v3.7/learning/api_guarantees.md:36-38` states that *"events in unhealthy clusters might never arrive"*, and `content/en/docs/v3.7/learning/design-client.md:80, 106, 136` admits that even when the cluster **maintains a healthy majority quorum**, a watch stream can *"get stuck with one failed or partitioned node"* forever. This conflates **prefix safety** (which an empty, permanently silent stream satisfies vacuously) with **eventual consistency** (which requires eventual delivery). Worse, `WithRequireLeader` does not restore transparent watch liveness: it is disabled by default, non-functional through `grpcproxy`, blind to apply-loop stalls, and when triggered on direct connections, destroys the caller's watch (and discards `ws.initReq.rev` on `startRev == 0` watches) rather than failing over to the surviving quorum—while causing newly reconnecting streams to die with zero grace period if they hit a member during a normal 1-second election. Because a client cannot establish a watcher with guaranteed liveness or detect silent staleness from the watch stream alone, the push-subscription utility of Watch is negated unless the client bolts on **periodic polling** (linearizable `KV.Get` `ReadIndex` checks or scheduled watch restarts).
3. **Server-Side Fixes for Multi-Member Reconnection (`v3.7.0`–`v3.7.3`) Traded Safety Regressions for Global Progress Starvation:**
   Prior to `v3.5.22` / `v3.6.2` / `v3.7.0`, reconnecting a watch to a lagging follower or recovering a member via snapshot restore / compaction repeatedly violated *Ordered*, *Unique*, *Reliable*, and *Bookmarkable* (`#15220`, `#15271`, `#17529`, `#18089`, `#19179`, `#20221`). While `v3.7.3` repairs those historical corruption paths, the fix in `progressIfSync()` (`#20229`) inspects server-wide `s.unsynced`, `s.synced`, and `s.victims` maps under an inverted lock order (`#22184`), starving `RequestProgress` and `ProgressNotify` across the entire server whenever any single watcher is catching up (`#20242`). Meanwhile, open `v3.7.3` defects in `sendLoop.pending` (`#22673`), `watcherGroup.choose` (`#22294`), `syncWatchers` (`#22264`), `WithPrevKV` (`#10681`), `grpcproxy` (`#21813`), and `etcd/cache` (`#22614`) remain unfixed.
4. **Java `jetcd` (`0.8.7`) Is Not Production-Safe for Resumable Historical Watches Without Client-Side Workarounds:**
   Unlike Go `clientv3` (which preserves explicit starting revisions across `Created`), `jetcd` `0.8.7` overwrites `this.revision = Math.max(revision, response.getHeader().getRevision())` upon receiving `Created` (`WatchImpl.java:267`). Any stream disconnect after `Created` and before `syncWatchersLoop` finishes streaming historical events causes `jetcd` to reconnect at the server's tip revision, silently losing all intervening history.
5. **Production Consumers Must Apply Kubernetes-Style Defensive Patterns:**
   Any production system using etcd Watch to maintain an in-memory cache or coordination state must:
   - Never open watches with `startRev == 0`; always obtain an explicit revision $R$ (via `Get`) and watch from `WithRev(R + 1)`.
   - Enable HTTP/2 keepalive pings (`30s`/`10s`) and `WithRequireLeader(ctx)` (on both `Watch` and `RequestProgress`), and treat any watch channel closure (even with `Err() == nil`) as a signal to re-establish the watch from `(lastProcessedRevision + 1)` rather than from "now".
   - Never rely on `RequestProgress` alone for linearizable cache freshness; pair `RequestProgress` with a linearizable quorum `ReadIndex` (`KV.Get`) as Kubernetes does in KEP-2340.
   - Avoid `grpcproxy`, `etcd/cache`, and `jetcd`'s built-in reconnection cursor (`WatchImpl.java:267`) until their respective event-loss bugs are fixed.
