# Chubby Perspectives: How Session-Anchored Blocked KeepAlives Prevent etcd's Lease and Watcher Failure Modes

- **History:** revised 2026-11-02; self-contained synthesis cross-validated against `doc/chubby-osdi06.md`, `etcd v3.7.3`, `jetcd 0.8.7`, and peer analyses `chubby-perspectives-claude.md`, `chubby-perspectives-codex.md`, and `chubby-perspectives-critique-claude.md`
- **Versions analyzed:**
  - Google Chubby design reference: Burrows, *The Chubby lock service for loosely-coupled distributed systems*, OSDI 2006 (`doc/chubby-osdi06.md`, pp. 1–16)
  - etcd `v3.7.3` (`0a173d0410a9`); `origin/main` (`a2e3642bc`, 2026-11-02) checked for `client/v3/leasing`
  - `go.etcd.io/raft/v3` `v3.7.0` (`server/go.mod:30 @ v3.7.3`)
  - jetcd `jetcd-0.8.7` (`23b61f1499a5`)
  - Selected Kubernetes `v1.37.2` (`c50e5395f1ae`) / `master` (`3c690c189e5d`) files for production comparison
- **Upstream references:**
  - **Companion analyses in this repository:** [`leases-agy.md`](leases-agy.md), [`watchers-agy.md`](watchers-agy.md), [`chubby-perspectives-claude.md`](chubby-perspectives-claude.md), [`chubby-perspectives-codex.md`](chubby-perspectives-codex.md), [`chubby-perspectives-critique-claude.md`](chubby-perspectives-critique-claude.md)
  - **etcd Lease, Session, Lock, and Leasing issues/PRs:**
    [#9888](https://github.com/etcd-io/etcd/issues/9888),
    [#12528](https://github.com/etcd-io/etcd/issues/12528) (closed),
    [#12594](https://github.com/etcd-io/etcd/issues/12594),
    [#13294](https://github.com/etcd-io/etcd/issues/13294),
    [#13675](https://github.com/etcd-io/etcd/issues/13675),
    [#13776](https://github.com/etcd-io/etcd/issues/13776) (closed, not planned),
    [#13915](https://github.com/etcd-io/etcd/issues/13915),
    [#14758](https://github.com/etcd-io/etcd/issues/14758),
    [#15247](https://github.com/etcd-io/etcd/issues/15247) (closed by [#16822](https://github.com/etcd-io/etcd/pull/16822)),
    [#16822](https://github.com/etcd-io/etcd/pull/16822) (merged),
    [#18069](https://github.com/etcd-io/etcd/issues/18069) (closed by [#18428](https://github.com/etcd-io/etcd/pull/18428)),
    [#18100](https://github.com/etcd-io/etcd/issues/18100),
    [#18428](https://github.com/etcd-io/etcd/pull/18428) (merged),
    [#19091](https://github.com/etcd-io/etcd/issues/19091) (closed by stale bot; PRs [#19092](https://github.com/etcd-io/etcd/pull/19092), [#21589](https://github.com/etcd-io/etcd/pull/21589), [#21869](https://github.com/etcd-io/etcd/pull/21869) closed unmerged),
    [#19597](https://github.com/etcd-io/etcd/issues/19597) (open),
    [#20273](https://github.com/etcd-io/etcd/issues/20273),
    [#20589](https://github.com/etcd-io/etcd/pull/20589),
    [#20590](https://github.com/etcd-io/etcd/pull/20590),
    [#21389](https://github.com/etcd-io/etcd/pull/21389),
    [#21638](https://github.com/etcd-io/etcd/issues/21638),
    [#21735](https://github.com/etcd-io/etcd/issues/21735),
    [#22081](https://github.com/etcd-io/etcd/issues/22081) (open),
    [#22082](https://github.com/etcd-io/etcd/issues/22082) (open),
    [#22083](https://github.com/etcd-io/etcd/issues/22083) (open),
    [#22084](https://github.com/etcd-io/etcd/issues/22084) (open),
    [#22094](https://github.com/etcd-io/etcd/issues/22094) (open),
    [#22382](https://github.com/etcd-io/etcd/issues/22382),
    [#22417](https://github.com/etcd-io/etcd/issues/22417) (open),
    [#22649](https://github.com/etcd-io/etcd/issues/22649) (open; PR [#22650](https://github.com/etcd-io/etcd/pull/22650) open)
  - **etcd Watch and Cache issues/PRs:**
    [#8673](https://github.com/etcd-io/etcd/issues/8673),
    [#8980](https://github.com/etcd-io/etcd/issues/8980),
    [#10681](https://github.com/etcd-io/etcd/issues/10681),
    [#12254](https://github.com/etcd-io/etcd/issues/12254),
    [#12534](https://github.com/etcd-io/etcd/issues/12534),
    [#15220](https://github.com/etcd-io/etcd/issues/15220),
    [#15271](https://github.com/etcd-io/etcd/issues/15271),
    [#15402](https://github.com/etcd-io/etcd/issues/15402),
    [#16839](https://github.com/etcd-io/etcd/issues/16839),
    [#17529](https://github.com/etcd-io/etcd/issues/17529),
    [#17611](https://github.com/etcd-io/etcd/issues/17611) (closed, not planned),
    [#18089](https://github.com/etcd-io/etcd/issues/18089),
    [#18879](https://github.com/etcd-io/etcd/issues/18879) (open; PR [#21065](https://github.com/etcd-io/etcd/pull/21065) open),
    [#19179](https://github.com/etcd-io/etcd/issues/19179),
    [#19710](https://github.com/etcd-io/etcd/issues/19710),
    [#20221](https://github.com/etcd-io/etcd/issues/20221) (closed by [#20229](https://github.com/etcd-io/etcd/pull/20229)),
    [#20242](https://github.com/etcd-io/etcd/issues/20242),
    [#20716](https://github.com/etcd-io/etcd/issues/20716) (closed, not planned),
    [#21813](https://github.com/etcd-io/etcd/issues/21813),
    [#22184](https://github.com/etcd-io/etcd/issues/22184),
    [#22264](https://github.com/etcd-io/etcd/issues/22264),
    [#22294](https://github.com/etcd-io/etcd/issues/22294),
    [#22614](https://github.com/etcd-io/etcd/issues/22614),
    [#22673](https://github.com/etcd-io/etcd/issues/22673)
  - **jetcd issues:**
    [jetcd#946](https://github.com/etcd-io/jetcd/issues/946),
    [jetcd#1227](https://github.com/etcd-io/jetcd/issues/1227),
    [jetcd#1322](https://github.com/etcd-io/jetcd/issues/1322),
    [jetcd#1352](https://github.com/etcd-io/jetcd/issues/1352)
  - **Kubernetes issues & KEPs:**
    [kubernetes#47532](https://github.com/kubernetes/kubernetes/issues/47532),
    [kubernetes#59848](https://github.com/kubernetes/kubernetes/issues/59848) (open),
    [kubernetes#65497](https://github.com/kubernetes/kubernetes/issues/65497) (closed),
    [kubernetes#96836](https://github.com/kubernetes/kubernetes/issues/96836) (closed),
    [kubernetes#110210](https://github.com/kubernetes/kubernetes/issues/110210),
    [kubernetes#111116](https://github.com/kubernetes/kubernetes/issues/111116) (closed as completed),
    [KEP-2340](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2340-Consistent-reads-from-cache)

---

## Scope

This document evaluates whether Google's Chubby architecture—in which all client state (locks, ephemeral files, client caches, and event notifications) belongs to a single master-anchored **Session** maintained by a server-held **KeepAlive** RPC—prevents or simplifies the correctness and performance failures identified in etcd's decoupled `Lease`, `Watch`, `concurrency`, and `leasing` subsystems. It also evaluates the reverse trade-offs of Chubby's design and identifies which Chubby mechanisms etcd can adopt without blocking writes on slow or failed readers.

---

## Summary

1. **Chubby's Core Guarantee Rests on Conditional Renewal (`Ack-or-Expired`) Across a Single Master-Anchored Session (**High confidence — Verified in `doc/chubby-osdi06.md` §§2.7–2.9, 3, 4.6**):**
   A single parked Chubby `KeepAlive` RPC unites five functions: client liveness proof (**F1**), lease grant (**F2**), server-to-client upcall delivery (**F3**), client-to-server causal acknowledgement (**F4**), and server-controlled renewal pacing (**F5**). Because a client cannot extend its session lease without acknowledging pending invalidations, the Chubby master maintains a global **`Ack-or-Expired`** invariant: before any write $W$ completes, every session that cached pre-$W$ data has either flushed its cache or had its lease expire. In etcd, by contrast, `LeaseKeepAlive` and `Watch` are independent streams that gRPC `round_robin` load balancing routinely places on **different cluster members**, allowing a client's lease to stay alive on the leader while its watch stream stalls silently on a partitioned follower.
2. **Six Nuances in the Chubby Paper Qualify the Comparison (**High confidence — Verified in `doc/chubby-osdi06.md` §§2.2–2.9, 4.4, 4.6**):**
   - *Pre-write invalidations vs. post-write events:* Pre-write cache invalidations block the write until acknowledged (§2.7), whereas post-write application events are asynchronous hints that never block writers (§2.5).
   - *Cache lists vs. open handles:* The master invalidates every session on its `client-is-caching-file` list (which includes metadata and negative file-absence caches), not merely sessions with open file handles (§2.7, §4.1).
   - *Ephemeral files vs. elections:* Ephemeral files are reference-counted across *all* open handles (§2.3), wait ~1 minute after failover before deletion (§2.9), and are not required for primary election, which locks a permanent file (§2.6).
   - *Conditional renewal vs. physical piggybacking:* Chubby §4.6 later proposed moving events to a TCP `GetEvent()` stream while keeping unacknowledged event IDs in the UDP `KeepAlive` reply so renewal stayed conditional on acknowledgement.
   - *External fencing:* `KeepAlive` expiry reclaims locks inside Chubby, but protecting external services from delayed network packets still requires **sequencers** or a 1-minute **`lock-delay`** (§2.4).
   - *Consensus fencing and Chubby's §4.4 flaw:* Session KeepAlives do not prevent deposed-leader split-brain; Chubby prevents that with **real-time master leases** (§2.2) and **master epochs** (§2.9). Moreover, when Chubby deferred disk logging for young read-only sessions (§4.4), it created a design hazard where discarded read-only sessions could read stale cached data across fast failovers (noted as rare in practice), forcing Chubby to make new masters wait a full worst-case lease timeout (`12s–60s`) before allowing operations to proceed.
3. **Leases: Eliminating Standalone Lease Objects Solves Renewal Amplification, Passive-TTL Reset Bugs, and Client "Zombie Renewals" (**High confidence — Verified in `v3.7.3` & `jetcd-0.8.7`**):**
   - *Why etcd cannot simply hold `KeepAlive` replies today:* Both Go `client/v3` and Java `jetcd` advance `nextKeepAlive` **only when a response arrives, never when a request is sent**, while running a `500ms` send ticker. Whenever a response is overdue past `TTL / 3`, each lease is re-sent every `500ms` (`2 requests/s` per lease, e.g., $40\times$ the normal rate at `TTL = 60s`) with no backoff. Holding the initial reply beyond `firstKeepAliveTimeout` (`5s` by default in `client/v3`, `v3client`, and `jetcd`; `6s` in `client/v3` when `DialTimeout = 5s`) also causes clients to drop the lease before the first response arrives.
   - *No passive TTL leases:* Because Chubby has no passive data-expiration leases (such as Kubernetes's 1-hour `Event` leases), extending session leases on failover adds at most one session lease period (`12s–60s`) instead of resetting 1-hour timers (`kubernetes#65497`). This avoids the entire `LeaseCheckpoint` subsystem, open bug `#22649`, and 1,000-key shared-lease revocation stalls.
   - *Monotone single-session client state machine:* Maintaining one session per client (`Safe` $\leftrightarrow$ `Jeopardy` $\rightarrow$ `Expired`) eliminates per-lease subscriber multiplexing bugs (`#22081`, `#22094`, and `jetcd` `onError` zombie renewals).
4. **Watchers & `client/v3/leasing`: Master-Coupled Delivery + `Jeopardy` Prevent Silent Watch Stalls and Expose Severe Flaws in `client/v3/leasing` (**High confidence — Verified in `v3.7.3`**):**
   - Coupling notification delivery to the master-held `KeepAlive` guarantees (given correct session bookkeeping and bounded clock drift) that any client cut off from updates enters `Jeopardy` within its conservative local lease timeout, flushing its cache and blocking API calls.
   - Auditing etcd's own Chubby-style cache (`client/v3/leasing`, exposed by `etcd grpc-proxy --experimental-leasing-prefix`) proves what goes wrong when Chubby's cache protocol is built over decoupled `Watch` and `LeaseKeepAlive` streams: (a) a partitioned follower hosting the `monitorLease` watch (which omits `WithRequireLeader`) while the leader hosts `LeaseKeepAlive` **blocks competing writers in `waitRescind` until their caller context expires**; (b) optimistic client lease deadlines (`receipt_time + TTL` swept every `1s`) allow a partitioned owner to **serve stale cached reads as "linearizable" for `RTT + up to 1s`** after the leader already revoked its lease and committed a write (`#19091`); (c) because `put` and `delete` test `wc == nil` instead of `resp == nil` after `tryModifyOp`, **every non-owner write is applied twice** (costing **2 Raft proposals** when uncontested and **5 Raft proposals** when contested, incrementing `Version` by `+2`, and returning wrong `Delete` and `WithPrevKV` responses), while **every non-owner linearizable read of an owned key is a Raft write proposal**; and (d) `leaseCache.EvictRange` deletes range start `key` instead of loop variable `k`.
5. **Locks: Native Session Locks Eliminate Watch-Dependent Lock Stalls (`waitDeletes`, `#19597`, `#18879`/`#20716`) (**High confidence for wait path; Moderate for `Release` fencing**):**
   In etcd, lock contenders wait by watching predecessor keys (`waitDeletes`), making lock acquisition vulnerable to silent follower watch stalls, expired-waiter hangs (`#19597`), unfenced `Unlock()` deletes (`#22082`), and the in-process `chanStream` cancellation deadlock (`#18879`/`#20716`) used by `v3lock` and `v3election`. Chubby's server-side lock wait eliminates the watch dependency entirely.
6. **Reverse Trade-Offs: Synchronous Pre-Write Cache Invalidation Would Stall Kubernetes Writes for Up to 12s–60s on Reader Failure, Whereas Session-Fenced Liveness and Freshness Detection Are Safely Portable (**High confidence**):**
   Chubby trades silent reader staleness for **bounded writer stalls** (for the duration of a cacher's unresponsiveness, up to its remaining `12s–60s` lease on crash or partition) and **drops events on failover**. That fit Chubby's `< 0.1%` write workload (`SetContents` at `680 ppm`), but would stall cluster-wide writes in Kubernetes whenever a caching `kube-apiserver` paused or became unreachable. While true linearizable reads from a client cache without Chubby's write barrier still require a per-read `ReadIndex` check, etcd can adopt Chubby's conservative client deadlines, request-send throttling, `Jeopardy` state machine, leader-revision watch freshness bounding, and deterministic entry-term revocation fencing (`§6, B1–B6`) without blocking writes on readers.

---

## 1. How Chubby's Session-Anchored KeepAlive Works (and Six Nuances from the Paper)

### 1.1 The Core Mechanism: Five Functions on One Master-Held RPC vs. etcd's Split Streams

In Chubby, every stateful interaction between a client library and the Chubby cell belongs to a single per-client **Session** terminated exclusively at the Chubby master. Instead of the client polling the server with periodic heartbeat ticks that return immediately, the client maintains a continuously parked `KeepAlive` RPC at the master (`doc/chubby-osdi06.md:259-262`, §2.8, p. 7). The master holds that RPC blocked until either:
1. The session's current lease is approaching expiration (at which point the master replies with a new lease extension, and the client immediately issues a new `KeepAlive`), or
2. The master needs to deliver a **pre-write cache invalidation** (§2.7) or **post-write event notification** (§2.5) to the client (at which point the master returns the parked `KeepAlive` early with those payloads piggybacked; the client library flushes invalidated cache entries synchronously and records events for asynchronous application delivery before re-issuing its next `KeepAlive`).

```mermaid
sequenceDiagram
    participant C as Reader's Chubby Library (Session S)
    participant M as Chubby Master
    participant W as Writer Client

    Note over C,M: Pre-State: Session S has cached file F; KeepAlive #1 is parked at Master
    C->>M: KeepAlive #1 (held at Master)
    W->>M: SetContents(handle F, new_value)
    Note over M: 1. Pre-Write Invalidation Barrier (§2.7, pp. 6-7)<br/>Marks F uncachable (concurrent reads served uncached);<br/>blocks SetContents; returns early from S's KeepAlive #1
    M-->>C: KeepAlive #1 Reply: INVALIDATE(F) + new_lease_timeout
    Note over C: Library flushes F from RAM cache BEFORE re-issuing KeepAlive
    C->>M: KeepAlive #2 (acknowledges INVALIDATE(F); held at Master)
    Note over M: All caching sessions ACKed (or expired);<br/>commits SetContents via Paxos
    M-->>W: SetContents Reply (OK)

    Note over C,M: 2. Post-Write Application Event (§2.5, p. 5 — Writer does NOT wait)
    M-->>C: KeepAlive #2 Reply: EVENT(FileContentsModified: F)
    C->>M: KeepAlive #3 (queues event for async upcall & acknowledges; held at Master)
    C->>M: GetContentsAndStat(handle F) [On demand]
    M-->>C: Returns new_value (or newer) + re-records S in F's cache list
```

A single parked Chubby `KeepAlive` RPC therefore bundles five distinct protocol functions (`F1–F5`) into one sequential state machine, whereas etcd decomposes them across independent, uncoordinated subsystems:

| Function | Direction | Chubby Mechanism (`doc/chubby-osdi06.md`) | etcd (`v3.7.3`) & `jetcd` (`0.8.7`) Counterpart | Confidence |
|---|---|---|---|---|
| **F1. Liveness Proof** | Client $\to$ Master | The parked `KeepAlive` request itself (`:259-260`), scoped to the whole client session. | Periodic `LeaseKeepAliveRequest{ID}` sent per *lease ID*, not per client (`client/v3/lease.go:600`, `LeaseImpl.java:245`). | **High** |
| **F2. Lease Grant** | Master $\to$ Client | `KeepAlive` reply carrying new session lease timeout (`:259-260`), interpreted with conservative client math (`:263`). | Immediate `LeaseKeepAliveResponse{ID, TTL}` (`v3rpc/lease.go:136-147`), interpreted optimistically as `receipt_time + TTL` (`client/v3/lease.go:543`, `LeaseImpl.java:265`). | **High** |
| **F3. Upcall Delivery** | Master $\to$ Client | Early return from parked `KeepAlive` carrying cache invalidations (§2.7, `:231`) and subscribed events (§2.5, `:163`; §2.8, `:261`). | Separate, unleased `Watch` gRPC stream (`v3rpc/watch.go`), served by whichever member (leader or follower) the stream landed on without `ReadIndex`. | **High** |
| **F4. Causal ACK** | Client $\to$ Master | Sending the *next* `KeepAlive` request proves the client flushed invalidated cache entries and received events (`:232-239, 261, 546`). | **None.** The etcd server never learns which watch events a client has processed. Moreover, Go `clientv3` defeats HTTP/2 backpressure by buffering all incoming watch responses into an unbounded slice `ws.buf` (`client/v3/watch.go:879-880`). | **High** |
| **F5. Server Pacing** | Master | Master holds the RPC until near expiry and scales TTL from `12s` to `~60s` under load (`§2.8, :259`; `§3, :352`). | **Client-paced at `TTL / 3`**, jumping to **2 requests/s per lease (`500ms` ticker) with no backoff** whenever a response is overdue (`client/v3/lease.go:542, 590-610`; `LeaseImpl.java:234-264`; §2.2). | **High** |

#### The `Ack-or-Expired` Invariant vs. etcd's Split-Stream Routing
Why can't an etcd client simply pair a `LeaseKeepAlive` stream (**F1+F2**) with a `Watch` stream (**F3**) to replicate Chubby's guarantees? Two structural gaps prevent it (**High confidence**):
1. **No conditional renewal (`F1` is decoupled from `F4`) (**High confidence**):** In Chubby, *"clients cannot maintain a session without acknowledging cache invalidations"* (§2.8, `doc/chubby-osdi06.md:261`). This enforces the **`Ack-or-Expired` invariant**: before any write $W$ to cached state $x$ completes, every session $S$ that may have cached $x$ has either acknowledged discarding $x$ or had its lease expire at the master (and, by conservative client timing, entered `Jeopardy` locally *before* the master timer expired). In etcd, a client's `LeaseKeepAlive` stream continues extending its lease even if its `Watch` stream is dead or hours behind.
2. **Independent stream placement across cluster members (**High confidence**):** Go `client/v3` configures gRPC with `{"loadBalancingPolicy": "round_robin"}` across all configured endpoints. Consequently, a single client's `LeaseKeepAlive` stream and its `Watch` stream routinely land on **different etcd members**—for instance, `LeaseKeepAlive` on healthy leader $M_1$ and `Watch` on follower $M_2$. If $M_2$ is partitioned from $\{M_1, M_3\}$ (while still reachable by the client) or suffers an apply stall, the client's lease stays healthy forever while its watch stream is silently frozen in the past.
   - *Code evidence:* `EtcdManualResolver.Build` in `client/v3/internal/resolver/resolver.go:44 @ v3.7.3`.

---

### 1.2 Six Precision Distinctions Required by the Chubby Paper

Testing every clause of the prompt's premise against `doc/chubby-osdi06.md` confirms the core thesis while identifying six nuances where the paper's actual mechanics are subtler than the prompt's summary (**High confidence**):

| # | Prompt Premise | What `doc/chubby-osdi06.md` Actually States | Why the Distinction Matters for etcd | Confidence |
|---|---|---|---|---|
| **1** | Watcher updates and cache invalidations are piggybacked on `KeepAlive` replies in the same way. | Chubby separates **two notification classes** with opposite ordering relative to a write:<br/>• **Pre-write cache invalidations (§2.7, pp. 6–7, `:231-240`):** Sent *before* the write executes; the write blocks until every caching session ACKs or expires. Meanwhile, the node is marked **uncachable** (`:241`) so concurrent reads proceed uncached.<br/>• **Post-write application events (§2.5, pp. 5–6, `:163-173`):** Delivered asynchronously via an upcall *after* the write commits (`:163, 172`). Writers **never** wait for application event callbacks. | Pre-write invalidation provides strict cache consistency at the cost of blocking writers on dead readers; post-write events are asynchronous hints that can be lost on failover (§2.5, `:167`). | **High** |
| **2** | Before a write, the server invalidates *"every session that has the file open."* | The master maintains `client-is-caching-file` (`230k` entries in §4.1, `:401`) and invalidates *"every client that may have cached it"* (§2.7, p. 6, `:229-231`). Clients cache **file data, metadata, and file absence (negative caching, `32k` entries, `:403`)** (§2.7, §3, `:229, 353`), which need not match open handles (`:247`). | Master-side cache tracking covers negative/metadata caches without open handles and costs **zero consensus writes** on read misses, unlike `client/v3/leasing` (§3.2). | **High** |
| **3** | Ephemeral files *"are associated to a session and get removed if the session ends"*, forcing master election to use session-owned objects. | • Ephemeral nodes are deleted only when ***no* client** has them open (§2.3, p. 4, `:118`); they are reference-counted across **all** open handles (which is why §2.5 `:166` monitors parent directories to avoid keeping ephemeral files open).<br/>• After master failover, deletion of unreferenced ephemeral files is delayed by **~1 minute** (§2.9 step 9, p. 9, `:313`).<br/>• Chubby's primary election recipe (§2.6, p. 6, `:225`) acquires a lock on a file and calls `SetContents()`—the file itself is typically **permanent** while the **lock** is session-bound. | Ephemeral files in Chubby can be kept alive if any other client opens them, and also incur a ~1-minute cleanup delay across failovers (§2.9, `:313`). | **High** |
| **4** | An unresponsive session forces a write to wait *"the session TTL (typically 12 seconds)."* | The write waits for the unresponsive client's *remaining* lease (§2.7, p. 7, `:239`). However, while the default lease extension is `12s`, **an overloaded master scales lease extensions up to `~60s`** (§2.8, p. 7, `:259`; §3, p. 10, `:352`), delaying writes *"by up to tens of seconds if a recently-failed client cached the file"* (§4.1, p. 11, `:440`). | Dynamic TTL scaling under load directly increases worst-case write stalls when a caching reader dies. | **High** |
| **5** | Locks are *"taken away"* on KeepAlive TTL expiry, and KeepAlives are what make distributed algorithms on Chubby safe. | • On abnormal holder failure, the master holds the lock unavailable for **`lock-delay`** (up to **1 minute**, §2.4, p. 5, `:159-160`).<br/>• Burrows notes (§2.4, p. 5, `:155-158`) that KeepAlives and locks alone **cannot** protect external resources against delayed network packets from a dead holder; safety requires passing and checking a **sequencer** (`GetSequencer()` / `CheckSequencer()`, §2.4, §2.6, `:157, 211-215`) or relying on `lock-delay`. | Takeover after failure takes `remaining lease + lock-delay` (up to ~72s–120s), and external resources still require fencing tokens (Chubby sequencers $\leftrightarrow$ etcd `CreateRevision`). | **High** |
| **6** | Physical transport piggybacking on the held `KeepAlive` RPC is inseparable from Chubby's guarantees. | In §4.6 (`:542-546`, p. 14), coupling events to time-critical `KeepAlive` RPCs over TCP caused lost sessions during congestion, forcing `KeepAlive` onto UDP (`:544`). The authors proposed moving normal event delivery to a TCP `GetEvent()` RPC while keeping **the list of unacknowledged events in the `KeepAlive` reply** (`:546`). | The essential invariant is **conditional renewal (`Ack-or-Expired`)**, not physical transport piggybacking. | **High** |

---

## 2. Impact on Leases: Pacing, Passive TTLs, and Client State Machines

### 2.1 Overview: Which etcd Lease Failures Chubby Prevents

Cross-referencing the lease failure modes identified in [`leases-agy.md`](leases-agy.md) shows three distinct patterns: issues prevented by Chubby's master lease and parked `KeepAlive` (`L1`, `L4`, `L5`, `L7`), issues avoided because Chubby has no passive data-TTL leases (`L2`, `L3`, `L8`), and consensus-fencing issues (`L6`) that Chubby addresses via master leases and epochs rather than client KeepAlives.

| # | etcd Lease Issue ([`leases-agy.md`](leases-agy.md)) | Prevented by Chubby's Session / KeepAlive? | Chubby Mechanism vs. etcd Implementation | Confidence |
|---|---|---|---|---|
| **L1** | **Per-renewal `ensureLeadership()` ReadIndex cost (`#18069`, relaxed in `#18428`), client-paced `TTL / 3` renewal rate, and `500ms` overdue-reply resend amplification** | **Yes — By Master Lease (`§2.2, :94`) + Parked KeepAlive Pacing (F5)** | Chubby's **master lease** (`:94`) makes master-local renewal safe without a quorum round (inferred from `:94`), while holding `KeepAlive` until near expiry and scaling TTL from `12s` to `60s` (§2.8, §3, `:259-260, 352`) reduces renewal frequency. Etcd clients pace at `TTL / 3` and resend every `500ms` (`2/s` per lease) whenever replies are overdue (§2.2). | **High** (pacing); **Moderate** (master-lease renewal inference) |
| **L2** | **P2 vs. P3 collision:** `Promote()` resets 1-hour passive Kubernetes `Event` leases (`k8s#65497`, `#9888`) | **Yes — Avoided by having no passive TTL leases** | Both systems extend leases on failover (§2.9, `doc/chubby-osdi06.md:285, 307`; `lessor.go:480-492`). Because Chubby only has active client sessions, failover extension adds at most one session lease period (`12s–60s`, §2.9, `:307`), whereas `Promote()` adds each lease's full `GrantedTTL` (`3600s` for Kubernetes `Event`s) (§2.3). | **High** |
| **L3** | **`LeaseCheckpoint` complexity (abandoned graduation `#13776`) and checkpoint-clear bug `#22649`** | **Yes — Avoided by omission** | Without long-TTL passive leases, Chubby needs no remaining-TTL checkpoints; in §4.4 (`:490`) Chubby stopped persisting sessions altogether (§2.3). | **High** |
| **L4** | **Go `#22081`/`#22094` and `jetcd` `onError` "zombie renewals"** | **Yes — Prevented by 1-session-per-client monotone lifecycle** | One library-owned session per client (`Safe` $\leftrightarrow$ `Jeopardy` $\rightarrow$ `Expired`, §2.8, `doc/chubby-osdi06.md:265-282`) eliminates per-lease subscriber lists (`ka.chs`/`ka.ctxs`) and post-error background renewal leaks (§2.4). | **High** |
| **L5** | **Optimistic client deadlines (`receipt_time + TTL`)** | **Yes — Prevented by conservative lease math** | Chubby subtracts reply transit time and clock drift rate (§2.8, `doc/chubby-osdi06.md:263`). Directly portable to etcd clients (§2.4, §6 B1). | **High** |
| **L6** | **Deposed-leader premature revocations (historical `#12528`, `#15247` mitigated by `#16822`; residual post-check windows `R1` and `R2`)** | **No — Addressed in Chubby by Master Leases & Epochs** | Holding a client `KeepAlive` RPC does not fence a stalled ex-leader. Chubby prevents overlapping masters via **real-time master leases** (`§2.2, :90`) and fences stale client calls via **master epochs** (`§2.9 step 1, :305`); rejection of a deposed master's internal expiry writes by Paxos ballots is inferred (§2.5). | **Moderate** |
| **L7** | **Follower-forwarded renewal bugs:** unchecked `leasehttp` (`R3`), `1s` `ApplyWaitCommit` timeout asymmetry, follower-stamped headers | **Yes — Prevented by Master-Direct Topology (in 2006 deployment)** | In Chubby's 2006 production deployment, clients send all session RPCs directly to the master (§2.2, §2.8, §3.1, `doc/chubby-osdi06.md:94, 261, 364`). Note that Chubby §3 and §3.1 (`:356-364`) and §4.4 (`:492`) also designed proxies that forward `KeepAlive` and reads. | **High** |
| **L8** | **Expiration storm costs:** 1 Raft entry per expired lease, 1,000-key single-txn deletes (`k8s#96836`, `k8s#110210`), per-revoke JWT signing (`#21735`) | **Partially shared; shared-lease amplification avoided** | Session expiry in Chubby also writes lock releases and ephemeral-file deletions to Paxos, but Chubby forbids attaching 1,000 passive objects to one shared lease. | **Moderate** |

---

### 2.2 Renewal Pacing, Master Leases, and etcd's 500ms Overdue-Reply Resend Amplification (**High confidence**)

In etcd, renewal load has two distinct sources:
1. **Per-renewal leadership verification (`#18069` — **High confidence** on etcd `#18069`/`#18428`; **Moderate confidence** on Chubby local-renewal inference):** After PR [#16822](https://github.com/etcd-io/etcd/pull/16822) added `ensureLeadership()` (a quorum `ReadIndex`) to `LeaseRenew`, [#18069](https://github.com/etcd-io/etcd/issues/18069) reported +20% leader CPU and a $10\times$ jump in peer network traffic at 1,500 renewals/sec. PR [#18428](https://github.com/etcd-io/etcd/pull/18428) relaxed `ensureLeadership()` to a local `isActive()` check, re-opening the residual post-partition window `R1`. In Chubby, by contrast, the **real-time master lease** (`doc/chubby-osdi06.md:90, 94`, §2.2) guarantees that no other master can exist while the lease is valid, allowing the master to serve local operations safely without a per-request quorum round (**Moderate confidence** — inferred from §2.2 `:94`, since the paper does not state whether every `KeepAlive` runs a Paxos round, though §4.1 `:396` notes that read-only sessions are not logged to disk).
2. **Client-driven renewal frequency vs. server-held pacing (`F5` — **High confidence**):** In Chubby, holding the `KeepAlive` RPC at the master until near lease expiration makes server-side rate control automatic: when a cell is overloaded, the master holds parked `KeepAlive` calls longer and returns a larger lease extension (`12s` $\to$ `60s`, §2.8, §3, `doc/chubby-osdi06.md:259-260, 352`).

Why can't etcd simply hold `LeaseKeepAliveResponse` messages on the server today, and how do existing etcd clients behave when a server reply is delayed? Inspecting the renewal loops in Go `client/v3` and Java `jetcd` reveals three acute obstacles (**High confidence**):

1. **`nextKeepAlive` is updated only on response receipt, never on request send (**High confidence**):**
   Both Go `client/v3` and Java `jetcd` run a background ticker every **500 milliseconds** that checks whether `now > nextKeepAlive` and, if so, writes a `LeaseKeepAliveRequest{ID}` to the gRPC stream. Crucially, neither client advances `nextKeepAlive` when it sends that request; `nextKeepAlive` is only advanced to `now + TTL / 3` when a `LeaseKeepAliveResponse` arrives from the server.
2. **500ms resend amplification while a reply is overdue (**High confidence**):**
   As soon as `now` passes `TTL / 3`, while a reply remains overdue—whether because the leader is under CPU/apply-wait load, because a follower's `leasehttp` forwarder is waiting up to `1s` on `ApplyWaitCommit()`, or because a hypothetical server tried to hold the RPC Chubby-style—**each lease is re-sent every 500ms (`2 requests/s` per lease, e.g., $40\times$ the normal rate for a `60s` TTL or $10\times$ for a `15s` TTL) with no backoff**.
3. **`firstKeepAliveTimeout` forbids holding initial KeepAlives (**High confidence**):**
   When `KeepAlive` is first invoked, `client/v3`, `v3client.New` (the server's internal loopback client), and `jetcd` all default to a **`5s` initial keepalive timeout** (`NewLeaseFromLeaseClient` replaces an exact `1s` timeout—passed by `v3client.New` or by `NewLease` when `DialTimeout == 0`—with `defaultTTL = 5s`, while `DialTimeout = 5s` produces **`6s`**). If an etcd server held the first `KeepAlive` reply until near expiry of a 12s–60s lease, all three clients would drop the lease after 5–6 seconds.

```go
// client/v3/lease.go:542-557, 586-618 @ v3.7.3
	nextKeepAlive := time.Now().Add((time.Duration(karesp.TTL) * time.Second) / 3.0)
	ka.deadline = time.Now().Add(time.Duration(karesp.TTL) * time.Second)
	for _, ch := range ka.chs {
		...
		ka.nextKeepAlive = nextKeepAlive // Updated ONLY when a response arrives!
	}
...
func (l *lessor) sendKeepAliveLoop(stream pb.Lease_LeaseKeepAliveClient) {
	for {
		var tosend []LeaseID
		now := time.Now()
		l.mu.Lock()
		for id, ka := range l.keepAlives {
			if ka.nextKeepAlive.Before(now) { // Remains < now on every 500ms tick until reply!
				tosend = append(tosend, id)
			}
		}
		l.mu.Unlock()
		for _, id := range tosend {
			r := &pb.LeaseKeepAliveRequest{ID: int64(id)}
			if err := stream.Send(r); err != nil { ... }
		}
		select {
		case <-time.After(retryConnWait): // retryConnWait = 500ms (lease.go:86)
		...
		}
	}
}
```

- **Code & Paper References:**
  - Chubby master lease (`:90, 94`), server-held KeepAlive & dynamic TTL scaling (`12s` $\to$ `60s`): `doc/chubby-osdi06.md:90-94` (§2.2), `:259-260` (§2.8), `:352` (§3).
  - Go `client/v3` 500ms loop & `firstKeepAliveTimeout`: `client/v3/lease.go:81, 86, 192-205, 296, 542-557, 586-619 @ v3.7.3`; internal loopback `v3client.New` passing `time.Second` (replaced with `defaultTTL = 5s`): `server/etcdserver/api/v3client/v3client.go:37 @ v3.7.3`.
  - Java `jetcd` 500ms periodic timer & `DEFAULT_FIRST_KEEPALIVE_TIMEOUT_MS = 5000`: `jetcd-core/src/main/java/io/etcd/jetcd/impl/LeaseImpl.java:61, 234-266, 340-346 @ jetcd-0.8.7`.
  - Server-side unimplemented dynamic TTL comment & follower `1s` wait: `server/lease/lessor.go:290-291, 396-455 @ v3.7.3`; `server/lease/leasehttp/http.go:37, 84 @ v3.7.3`; `server/etcdserver/v3_server.go:510-536 @ v3.7.3`.

---

### 2.3 Active Sessions vs. Passive TTL Leases (P2 vs. P3 and Checkpointing) (**High confidence** on mechanisms; **Moderate confidence** on comparative design assessment)

Both Chubby and etcd extend lease timers across a leader failover (§2.9, `doc/chubby-osdi06.md:285, 307`; `lessor.Promote` in `server/lease/lessor.go:480-492 @ v3.7.3`). Why did failover extension cause unbounded memory bloat in etcd ([kubernetes#65497](https://github.com/kubernetes/kubernetes/issues/65497), [etcd#9888](https://github.com/etcd-io/etcd/issues/9888)) while causing no such problem in Chubby?

- **Chubby leases represent active client sessions only (`12s–60s` — **High confidence**):**
  In Chubby, a lease exists *solely* to bound the liveness of an active client session. When a new master extends session leases to the maximum timeout the old master may have used (`§2.9 step 3, p. 9`), it adds at most **one session lease period (`12s` default, `~60s` under load)**, plus ~1 minute before deleting unreferenced ephemeral files in step 9.
- **etcd overloads a single `Lease` primitive for both active sessions (`P3`) and 1-hour passive data timers (`P2`) (**High confidence**):**
  Because an etcd `Lease` is an anonymous `int64` timer that requires no active client session, different etcd users rely on the same primitive for opposite purposes: direct etcd coordination clients (`client/v3/concurrency` `Session`/`Mutex`/`Election`, `v3lock`, `v3election`, `client/v3/leasing`) use short actively renewed leases that require `P3` non-revocation across elections, whereas Kubernetes (`lease_manager.go`) uses etcd leases *only* as passive 1-hour (`3600s`) unrenewed TTL timers for `Event` objects (Kubernetes node heartbeats and controller leader elections use `coordination.k8s.io` `Lease` API objects updated via ordinary KV writes, not etcd `LeaseKeepAlive`). Because `lessor.Promote()` resets every uncheckpointed lease to `now + ElectionTimeout + GrantedTTL`, each leader change adds a full **3,600 seconds** to every passive `Event` lease.
- **Eliminating `LeaseCheckpoint` (abandoned graduation `#13776`) and open bug `#22649` (**High confidence**):**
  To stop `Promote()` from resetting 1-hour passive leases, etcd added Raft-replicated `LeaseCheckpoint` proposals (whose graduation from Alpha was tracked in [#13776](https://github.com/etcd-io/etcd/issues/13776) and closed as *not planned*). In `v3.7.3`, when an active lease with a prior `RemainingTTL` checkpoint is renewed, `lessor.Renew` invokes the checkpointer callback installed by `SetCheckpointer` (`server/etcdserver/server.go:392-403 @ v3.7.3`) to clear `RemainingTTL = 0` via `srv.raftRequest`. Because that `SetCheckpointer` closure ignores the error from `srv.raftRequest` and returns `nil` unconditionally, a failed or timed-out `raftRequest` leaves the stale `RemainingTTL` checkpoint in place while `Renew()` returns the full `l.ttl` to the client—causing the lease to be armed with the stale shorter `RemainingTTL` on the next leader election ([#22649](https://github.com/etcd-io/etcd/issues/22649), open fix PR [#22650](https://github.com/etcd-io/etcd/pull/22650)). By restricting leases to short-lived active sessions, Chubby needs no remaining-TTL checkpoints at all.

- **Code & Paper References:**
  - Chubby failover lease extension: `doc/chubby-osdi06.md:285, 307, 313` (§2.9), `:490` (§4.4).
  - etcd `Promote()`, Alpha feature gate, & `SetCheckpointer` bug `#22649`: `server/lease/lessor.go:367-455, 480-492, 753-770 @ v3.7.3`; `server/etcdserver/server.go:392-403 @ v3.7.3`; `server/features/etcd_features.go:94-95 @ v3.7.3`; Kubernetes 1-hour event lease sharing (`1000` objects/lease, `60s` window): `staging/src/k8s.io/apiserver/pkg/storage/etcd3/lease_manager.go:39-145 @ v1.37.2`.

---

### 2.4 Single-Session Lifecycle vs. Per-Lease Subscriber Multiplexing (**High confidence**)

In Chubby, the client library owns a single session per cell with a strict monotone state machine (`Safe` $\leftrightarrow$ `Jeopardy` $\rightarrow$ `Expired`, `doc/chubby-osdi06.md:265-282`, §2.8, pp. 7–8) and computes its local deadline **conservatively** by deducting RPC transit time and clock drift rate (`doc/chubby-osdi06.md:263`).

In etcd, both Go `client/v3` and Java `jetcd` allow callers to invoke `KeepAlive(ctx, id)` multiple times for arbitrary lease IDs on a shared background stream and compute deadlines **optimistically** (**High confidence**):
1. **Per-subscriber compaction and orphan bugs (`#22081`, `#22094` — **High confidence**):** In `client/v3`, `lessor.KeepAlive` appends each caller's context and channel to `ka.ctxs` and `ka.chs`. When `closeRequireLeader()` removes subscribers whose context carried `WithRequireLeader`:
   - In [#22081](https://github.com/etcd-io/etcd/issues/22081), the compaction loop pairs each surviving channel `ka.chs[i]` with `ka.ctxs[newIdx]` (`lease.go:406`), indexing the old `ctxs` slice with `newIdx` instead of `i`. That misaligns `ka.chs` and `ka.ctxs`, so a later context cancellation in `keepAliveCtxCloser` closes the **wrong subscriber's channel**.
   - In [#22094](https://github.com/etcd-io/etcd/issues/22094), when *all* subscribers of a lease carried `WithRequireLeader`, `closeRequireLeader()` empties `ka.chs` and `ka.ctxs` without deleting the entry from `l.keepAlives`. Because `recvKeepAlive` (`lease.go:544-556`) only advances `ka.nextKeepAlive` inside the loop `for _, ch := range ka.chs`, once `ka.chs` is empty `ka.nextKeepAlive` is never updated again—leaving an orphaned entry that sends a `LeaseKeepAliveRequest` every `500ms` forever!
2. **Post-error "zombie renewals" in `jetcd` (**High confidence**):** In `jetcd-0.8.7`, `KeepAlive.handleException` delivers `onError(throwable)` to every user `StreamObserver`—telling the application its lease renewal failed—and then immediately schedules a `500ms` retry that **resumes renewing all leases in `keepAlives` in the background**!
3. **Optimistic client deadlines (`L5` — **High confidence**):** Both `client/v3` (`ka.deadline = time.Now().Add(time.Duration(karesp.TTL) * time.Second)`) and `jetcd` (`ka.setDeadLine(System.currentTimeMillis() + ttl * 1000)`, swept every `1000ms` by `DeadLine.doStart`) anchor local expiration to **response receipt time**, guaranteeing that a partitioned client believes its lease is valid for `one-way transit delay + up to 1s sweep interval + clock drift` *after* the server has already revoked it.

- **Code & Paper References:**
  - Chubby conservative lease math & `Jeopardy`: `doc/chubby-osdi06.md:263-292` (§2.8–§2.9).
  - Go `client/v3` subscriber handling & optimistic deadline: `client/v3/lease.go:369-371, 376-411, 542-583 @ v3.7.3`.
  - Java `jetcd` `handleException` zombie renewal & deadline math: `jetcd-core/src/main/java/io/etcd/jetcd/impl/LeaseImpl.java:146-152, 251-287 @ jetcd-0.8.7`.

---

### 2.5 What KeepAlives Alone Do Not Prevent: Deposed-Leader Split-Brain (**High confidence** on paper mechanisms; **Moderate confidence** on inferred Paxos ballot fencing and etcd `R1`/`R2` race schedules)

Holding a client `KeepAlive` RPC at the server does **not** by itself prevent a deposed leader from proposing premature lease revocations (historical issues [#12528](https://github.com/etcd-io/etcd/issues/12528) and [#15247](https://github.com/etcd-io/etcd/issues/15247), which PR [#16822](https://github.com/etcd-io/etcd/pull/16822) mitigated in `v3.7.3` via an `ensureLeadership()` pre-check, leaving the residual post-check windows `R1` and `R2`) or serving stale follower-forwarded renewals (`R3` in [`leases-agy.md` §2.2–§2.3](leases-agy.md)).

Chubby addresses deposed-leader hazards through two **consensus-level** mechanisms documented in the paper (**High confidence**), plus standard Paxos ballot fencing (**Moderate confidence** — inferred):
1. **Real-time master leases (`doc/chubby-osdi06.md:90`, §2.2, p. 3 — **High confidence**):** Replicas promise not to elect a new master until the old master's real-time master lease has expired, preventing two masters from operating concurrently.
2. **Master epoch numbers (`doc/chubby-osdi06.md:305`, §2.9 step 1, p. 9 — **High confidence** on client epoch check; **Moderate confidence** on internal Paxos ballot fencing):** A new master increments the persistent epoch number in the database and rejects *client* requests carrying older epoch numbers. While the paper does not detail how a deposed master's internal expiry work is fenced if its master lease expires mid-operation (noting at `:315` that failover code is *"a rich source of interesting bugs"*), in Paxos any log write from an old ballot is rejected once a majority has promised a higher ballot. In etcd `v3.7.3`, by contrast, internal expiry-driven `LeaseRevokeRequest` proposals carry no term field (`server/lease/lessor.go:640-677`, `server/etcdserver/v3_server.go:471-482 @ v3.7.3`), so a `MsgProp` queued across a step-down is forwarded by Raft followers and committed in the new term (`go.etcd.io/raft/v3/raft.go:1718-1729 @ v3.7.0`, **High confidence** on code path; **Moderate confidence** on race schedule).

---

## 3. Impact on Watchers and Client Caches: Freshness vs. History

### 3.1 Overview: Freshness Guarantees vs. Lossless Event Replay

Evaluating the watcher and cache failure modes from [`watchers-agy.md`](watchers-agy.md) reveals a fundamental architectural divide:
- **Where Chubby excels (`W1–W5` — Stronger Freshness Contract and Partition Detection):** Etcd's watch API explicitly disclaims bounded delivery latency (*"there is no limit and events in unhealthy clusters might never arrive"*), so silent follower watch stalls (`W1–W4`) are consequences of a weak watch contract rather than violations of it. Chubby provides a much stronger contract: because invalidations and events ride on the master-held `KeepAlive` RPC and the client enters `Jeopardy` if that channel stalls, **silent cache/event staleness is excluded by design** (given correct session bookkeeping and the clock-rate assumption, §2.8, §4.4, `doc/chubby-osdi06.md:263, 488`).
- **Where etcd's MVCC log excels (`W6–W7` — Lossless Historical Stream Within Retained History):** Because Chubby has no multi-version history log (`[compactMainRev, currentRev]`), Chubby **loses events on master failover** and forces clients to perform full rescans (§2.5, §2.9, §4.6, `doc/chubby-osdi06.md:167, 309, 534`). Etcd's historical catch-up and cursor bugs (`W7`) exist precisely because etcd supports lossless revision replay across disconnects and elections within its uncompacted history.

| # | etcd Watcher / Cache Issue ([`watchers-agy.md`](watchers-agy.md)) | Prevented by Chubby's Session / KeepAlive? | Mechanism & Comparison | Confidence |
|---|---|---|---|---|
| **W1** | **Silent watch stall on partitioned follower or ex-leader** (default `WithRequireLeader` off in `interceptor.go:239-258, 315-354`; unfenced `ProgressNotify` at frozen local revision; downstream stale cache reads `k8s#59848`) | **Yes — Excluded by master-only session + `Jeopardy`** | Delivery and liveness share the master-held `KeepAlive`. If cut off from the master, the client's conservative lease expires into `Jeopardy`, flushing its cache and blocking API calls (§2.8, §2.9, `doc/chubby-osdi06.md:263-265, 289`). | **High** (in Chubby); **Moderate** (transfer to etcd) |
| **W2** | **Silent watch stall during (a) member apply-loop stall or (b) whole-process pause / dead connection (`DialKeepAliveTime == 0`, `client.go:231-236`)** | **Yes — Prevented by (a) master-only topology and (b) KeepAlive heartbeat** | (a) In Chubby, only the master serves reads and commits writes, so a slow master stalls writes rather than serving stale state on a lagging follower.<br/>(b) On a whole-process pause or dead connection, the master cannot answer `KeepAlive` and clients enter `Jeopardy` within their conservative local lease (`12s` default, up to `~60s` under load, §2.8, §3, `:263, 352`). | **High** |
| **W3** | **The 9 `WithRequireLeader` failure modes:**<br/>• *Implementation defects:* `250ms` error drop, `#17611` `wg.Wait` teardown deadlock, `#12254` `RequestProgress` stream split, `grpcproxy` metadata loss<br/>• *Lifecycle design:* immediate rejection of new streams during elections (`interceptor.go:239-241`) and mass stream cancellation after 3 election timeouts (`maxNoLeaderCnt = 3`, `interceptor.go:38, 315-354`, triggering `k8s#111116`) | **Implementation bugs avoided by omission; lifecycle improved by 45s `Jeopardy` grace period (with `:534` caveat)** | Chubby avoids etcd's `WithRequireLeader` implementation bugs by not having a bolt-on stream interceptor. For elections, Chubby's 45s `Jeopardy` grace period (§2.8, §2.9, `:265-292`) suspends calls instead of rejecting streams; if the *same* master recovers, it emits `safe` (§2.8, `:279`). Across a *master change*, however, Chubby flushes caches and emits a `fail-over` event (§2.9, `:309`)—which caused many Google developers to crash their apps (`§4.6, :534`), reproducing `k8s#111116` one layer up! | **Moderate** |
| **W4** | **Unfenced member-local `RequestProgress` / `ProgressNotify` + server-wide progress starvation (`#20242`) + `KEP-2340` quorum `Get` polling** | **Yes — By KeepAlive delivery + Pre-Write Invalidation Barrier (`§2.7`)** | In Chubby, a `KeepAlive` reply is the master-anchored liveness/delivery signal, and the **pre-write invalidation barrier (`§2.7, Layer C`)** guarantees that no write can complete between replies without invalidating the client cache first (§2.7, §2.8, `doc/chubby-osdi06.md:229-240, 261`). Without Layer C, a heartbeat revision bounds staleness rather than providing per-read linearizability (§6 `B3`). | **High** |
| **W5** | **`client/v3/leasing` (`and grpc-proxy --experimental-leasing-prefix`) double-write bug, 5-proposal contested writes, and split-stream bugs** | **Yes — Prevented by master-tracked cache + `Ack-or-Expired`** | Detailed in §3.2 below: `client/v3/leasing` suffers indefinite writer blocks, stale "linearizable" reads (`#19091`), double application of every non-owner write (`2` proposals uncontested, `5` proposals contested), Raft proposals on every non-owner linearizable read of an owned key, and an `EvictRange` bug (`#21869`). | **High** |
| **W6** | **Event completeness across master failover** | **No — etcd's MVCC log is stronger (within retained history)** | Chubby **loses events on master failover** and emits a `fail-over` event forcing clients to rescan (§2.5, §2.9, §4.6, `doc/chubby-osdi06.md:167, 309, 534`). Etcd's Raft-replicated MVCC history enables lossless resumption across elections for any revision $\ge$ `compactMainRev` (`mvcc/watcher_group.go:255-262`). | **High** |
| **W7** | **Historical catch-up, compaction, restore, and cursor bugs** (`#15220`, `#15271`, `#17529`, `#18089`, `#19179`, `#20221` [fixed by `#20229` in `v3.7.3`], `#22184`, `#22294`, `startRev == 0` race, `jetcd` `WatchImpl.java:267` skip) | **Avoided by omission (Chubby has no historical log or resume cursor)** | Chubby avoids cursor and catch-up bugs only because it never replays historical mutations; if an application needs every state transition, Chubby cannot provide it (§4.5, `doc/chubby-osdi06.md:516-524`). | **High** |
| **W8** | **Slow/lagging watchers stalling the write path (`#22264` `syncWatchers` holding `watchableStore.mu.Lock()`)** | **Inverted (Accidental bug in etcd vs. Intentional bounded contract in Chubby)** | In etcd, a watcher 500k revisions behind accidentally stalls writers via lock contention in `syncWatchers()`. In Chubby, **every write to cached data intentionally waits for all caching readers to ACK or expire (`12s–60s`)** (§5.1). | **High** |
| **W9** | **`grpcproxy` watch bugs (`#21813` dead-broadcast coalescing poison pill, `#19710` dropped `RequestProgress`)** | **Partially shared (§3.1, §4.4, `doc/chubby-osdi06.md:364, 488-492`)** | Any proxy must preserve session/invalidation invariants. Burrows notes Chubby's original failover design was *"not ideal for proxies"* (§3.1, `:364`) and had to be redesigned in §4.4 (`:488-492`). | **Moderate** |

---

### 3.2 Natural Experiment: Auditing etcd's Own Chubby-Style Cache (`client/v3/leasing`) (**High confidence**)

Etcd ships an official package—`client/v3/leasing` (also exposed by `etcd grpc-proxy` via `--experimental-leasing-prefix`), described in `client/v3/leasing/doc.go` as providing linearizable client-side read caching—that attempts to implement Chubby §2.7's write-invalidated cache on top of etcd's `KV`, `Lease`, and `Watch` primitives.

Because `client/v3/leasing` decouples **invalidation delivery** (a per-key `Watch` stream in `monitorLease`) from **session liveness** (a `concurrency.Session` `LeaseKeepAlive` stream), auditing its implementation at `v3.7.3` (and `origin/main` at `a2e3642bc`, where the code is identical) provides a direct empirical test of our thesis (**High confidence**).

```mermaid
sequenceDiagram
    participant R as Caching Owner (leasingKV 1)
    participant M1 as etcd Member 1 (Leader)
    participant M2 as etcd Member 2 (Partitioned Follower)
    participant W as Writer (leasingKV 2)

    Note over R,M2: 1. Split-Stream Placement via round_robin (resolver.go:44)
    R->>M1: Session LeaseKeepAlive stream (lands on Leader M1)
    R->>M1: Get("k") -> acquire Txn writes "pfx/k" with Lease1 (1st Raft write!)
    R->>M2: monitorLease Watch("pfx/k") (lands on Follower M2; NO WithRequireLeader!)

    Note over M1,M2: Network Partition isolates M2 from {M1, M3}, but R <-> M2 TCP stays up!
    W->>M1: Put("k", "v2") -> tryModifyOp fails (1st proposal); revoke writes Put("pfx/k", "REVOKE") (2nd proposal)
    W->>M1: waitRescind Watch("pfx/k") waiting for EventTypeDelete (kv.go:207)

    Note over R,M2: Failure Mode L-A: Indefinite Writer Block!
    M1-->>R: LeaseKeepAlive responses continue on M1 (Lease1 never expires!)
    Note over M2: Partitioned M2 never receives "REVOKE" and never closes Watch!
    Note over W: Writer blocks in waitRescind until caller ctx cancels!
```

#### Failure Mode L-A: Indefinite Writer Block Under Split-Stream Routing (**High confidence** on code mechanism; **Moderate confidence** on multi-endpoint partition schedule)
1. When a reader caches a key in `leasingKV.get`, it spawns `monitorLease(ctx, key, rev)`, which opens a watch `lkv.cl.Watch(cctx, lkv.pfx+key, v3.WithRev(rev+1))` **without `WithRequireLeader`** (**High confidence**).
2. Because `lkv.cl` uses gRPC `round_robin`, the caching owner's `LeaseKeepAlive` stream can land on healthy leader $M_1$ while its `monitorLease` watch stream lands on follower $M_2$ (**High confidence**).
3. If $M_2$ becomes network-partitioned from $\{M_1, M_3\}$ (while its TCP socket to the caching owner remains open) or suffers an apply stall, $M_2$ never receives the competing writer's `"REVOKE"` put and never closes the watch stream (**High confidence**).
4. Meanwhile, the owner continues renewing its lease on $M_1$ indefinitely. Because the owner neither sees `"REVOKE"` nor lets its lease expire, **the competing writer blocks in `waitRescind()` until its caller context times out** (**Moderate confidence** — derived from verified code paths; requires a multi-endpoint client where `round_robin` places `LeaseKeepAlive` and `monitorLease` on different members).
- **Code References:** `client/v3/leasing/kv.go:159-188` (`monitorLease`, no `WithRequireLeader`), `kv.go:204-216` (`waitRescind`), `kv.go:323-336` (`get`), `client/v3/internal/resolver/resolver.go:44 @ v3.7.3` (`round_robin`).

#### Failure Mode L-B: Stale "Linearizable" Reads on a Partitioned Owner (`#19091` — **High confidence**)
1. `etcd grpc-proxy` advertises `--experimental-leasing-prefix` as providing *"disconnected linearized reads"*. When `leasingKV.get` serves a cached key from RAM, its sole freshness check is `lkv.readySession()`, which checks whether `lkv.session.Done()` is closed.
2. As shown in §2.4, `Session.Done()` is driven by `client/v3`'s optimistic deadline `ka.deadline = receipt_time + TTL`, swept once per second by `deadlineLoop` (`timer.Reset(time.Second)`).
3. If the caching owner is completely partitioned from the cluster after renewing its lease at server time $t_0$ (with reply transit delay $\Delta_{\text{transit}}$), the etcd leader revokes the owner's lease at $t_0 + TTL$ and immediately allows a competing writer to commit a new value `v2`.
4. On the isolated owner, however, `lkv.readySession()` continues returning `true` until $t_0 + TTL + \Delta_{\text{transit}} + \text{up to } 1\text{s}$. Throughout that window, **`lkv.Get(ctx, key)` serves the stale value `v1` as a linearizable read after `v2` has already been committed** (**High confidence** — verified in code and reported upstream in [#19091](https://github.com/etcd-io/etcd/issues/19091), which was closed by the stale bot; fix PRs [#19092](https://github.com/etcd-io/etcd/pull/19092) and [#21589](https://github.com/etcd-io/etcd/pull/21589) were closed unmerged).
- **Code References:** `server/etcdmain/grpc_proxy.go:179, 488 @ v3.7.3`; `client/v3/leasing/kv.go:306-311, 467-479 @ v3.7.3` (`get`, `readySession`); `client/v3/lease.go:543, 562-583 @ v3.7.3` (`ka.deadline`, `deadlineLoop`); upstream [#19091](https://github.com/etcd-io/etcd/issues/19091), [#19092](https://github.com/etcd-io/etcd/pull/19092), [#21589](https://github.com/etcd-io/etcd/pull/21589).

#### Failure Mode L-C: Double-Applied Non-Owner Writes, 5-Proposal Contested Writes, Raft Proposals on Non-Owner Reads, and the `EvictRange` Bug (**High confidence**)
Even when the network is healthy, building cache invalidation out of client-side `KV` transactions and watches causes severe protocol bugs and consensus amplification (**High confidence** — verified in source code and reproduced against `etcd v3.7.3`):
1. **Every non-owner write (`Put` or `Delete`) is applied twice (2 Raft proposals uncontested; 5 Raft proposals contested) and returns corrupted `Delete`/`PrevKv` responses (**High confidence**):**
   In `leasingKV.put` (`kv.go:241-244`) and `leasingKV.delete` (`kv.go:389-392`), the client first calls `tryModifyOp(ctx, op)`, which locks the local cache entry (`wc, rev := lkv.leases.Lock(key)`, returning **`wc = nil, rev = 0` whenever the writer does not currently cache `key`**) and commits a Raft `Txn`: `If(CreateRevision(pfx+key) < rev+1).Then(op)`.
   Then `put` and `delete` execute:
   ```go
   // client/v3/leasing/kv.go:241-244 @ v3.7.3 (and kv.go:389-392 for delete)
   resp, wc, err := lkv.tryModifyOp(ctx, op)
   if err != nil || wc == nil { // BUG: wc is ALWAYS nil for a non-owner, even when tryModifyOp succeeded!
       resp, err = lkv.revoke(ctx, string(op.KeyBytes()), op)
   }
   ```
   Because `wc` is `nil` for any writer that did not already own `key` in its local cache (`cache.go:53-61`), **even when `tryModifyOp` succeeds and applies `op`**, `put` and `delete` unconditionally call `revoke(ctx, key, op)` (`kv.go:413-421`), which commits a second `Txn` (`If(CreateRevision(pfx+key) < rev+1).Then(op).Else(Put("REVOKE"))`) that **applies `op` a second time**!
   - **Uncontested non-owner write (**High confidence**):** Commits **2 Raft proposals (2 revisions)**, applies the `Put` or `Delete` twice, increments the key's `Version` by `+2`, emits two `PUT` watch events, causes `Delete(..., WithPrevKV())` on an existing key to return **`Deleted = 0` and an empty `PrevKvs`** (because the second `Txn` deletes an already-deleted key!), and causes `Put(k, "new", WithPrevKV())` to return **`PrevKv.Value = "new"`** instead of `"old"`!
   - **Contested non-owner write (when another client A caches `key` — **High confidence**):** Commits **5 sequential Raft proposals (4 revisions)** and still applies the write twice:
     1. Writer B's initial `tryModifyOp` commits via Raft (because `IsTxnReadonly` in `server/etcdserver/txn/txn.go:352-363` is `false` whenever either `Txn` branch contains a non-`Range` op) and fails its `If` compare;
     2. Writer B's `revoke` commits `Put(pfx+key, "REVOKE")` and waits in `waitRescind`;
     3. Owner A's `monitorLease` watch receives `"REVOKE"` and commits `rescind` (`Delete(pfx+key)`);
     4. Writer B's loop repeats and `tryModifyOp` succeeds, applying `op` the first time (`wc` is still `nil`);
     5. Because `wc == nil`, Writer B calls `revoke` again, which succeeds and **applies `op` a second time**!
2. **At most 1 client can cache a hot key, and every non-owner linearizable read of an owned key is a Raft write proposal (**High confidence**):**
   In Chubby, caching $N$ files on read miss costs $N$ read RPCs and **0 consensus writes**, and an average of $\approx 10$ clients (up to thousands) cache the same file simultaneously. In `client/v3/leasing`, `acquire()` (`kv.go:270-280`) requires `CreateRevision(pfx+key) == 0`, so **at most one `leasingKV` client can cache a key**. Worse, for any other client performing a linearizable `Get(ctx, key)`, `MayAcquire(key)` (`cache.go:113-118`) returns `true` (except for `revokeBackoff = 2s` after a local revoke), so **every non-owner linearizable read calls `acquire()`**, which contains `OpPut` in its `Then` branch and therefore goes through **Raft consensus as a write proposal** (`txn.go:352-363`)!
3. **Loop-variable deletion bug in `leaseCache.EvictRange` (`#21589`, `#21869` — **High confidence**):**
   When `deleteRangeRPC` fails on a range delete (`kv.go:349`), `EvictRange(key, end)` iterates `for k := range lc.entries` and calls `delete(lc.entries, key)`—deleting the range start `key` instead of the loop variable `k`, leaving every other cached key in `[key, end)` stale in memory (two upstream PRs, [#21589](https://github.com/etcd-io/etcd/pull/21589) and [#21869](https://github.com/etcd-io/etcd/pull/21869), attempted to fix this and were closed unmerged; the bug remains on `v3.7.3` and `origin/main`).

```go
// client/v3/leasing/cache.go:178-187 @ v3.7.3
func (lc *leaseCache) EvictRange(key, end string) {
	lc.mu.Lock()
	defer lc.mu.Unlock()
	for k := range lc.entries {
		if inRange(k, key, end) {
			delete(lc.entries, key) // BUG: deletes `key` instead of loop variable `k`!
			lc.revokes[key] = time.Now()
		}
	}
}
```

- **Code & Paper References:** `client/v3/leasing/kv.go:191-299, 317-350, 380-421 @ v3.7.3`; `client/v3/leasing/cache.go:28, 53-61, 113-118, 178-187 @ v3.7.3`; `server/etcdserver/txn/txn.go:352-363 @ v3.7.3`; upstream issues/PRs [#21589](https://github.com/etcd-io/etcd/pull/21589), [#21869](https://github.com/etcd-io/etcd/pull/21869), [#22083](https://github.com/etcd-io/etcd/issues/22083), [#22084](https://github.com/etcd-io/etcd/issues/22084), [#22417](https://github.com/etcd-io/etcd/issues/22417); Chubby cache fanout: §2.1, §2.7, §4.1, `doc/chubby-osdi06.md:59, 229-240, 430`; empirical reproduction in [`chubby-perspectives-critique-claude.md` Appendix A](chubby-perspectives-critique-claude.md).

---

## 4. Impact on Locks and Leader Election: Native Primitives vs. Client Recipes

### 4.1 How etcd Synthesizes Locks out of Keys, Leases, and Watches (**High confidence**)

In Chubby, advisory locks are first-class server primitives (`Acquire()`, `TryAcquire()`, `Release()`, `doc/chubby-osdi06.md:139-160, 209-215`, §§2.4, 2.6). In etcd, the replicated state machine has no native lock table (**High confidence**):
1. **Client-side recipe (`client/v3/concurrency` — **High confidence**):** `Mutex.tryAcquire` writes an ephemeral key `<pfx>/<hex(leaseID)>` attached to the caller's session lease and queries the key under `<pfx>/` with the smallest `CreateRevision`. If an earlier key exists, `Mutex.Lock` enters `waitDeletes`, which watches each predecessor key for `Event_DELETE` before verifying that its own key still exists.
2. **Server-side `v3lock` and `v3election` gRPC proxies (`server/etcdserver/api/v3lock`, `v3election` — **High confidence**):** Thin gRPC clients such as Java `jetcd` call etcd's server-side `Lock` and `Election` gRPC services. Internally, however, `v3lock` and `v3election` simply invoke the Go `client/v3/concurrency` recipe over `v3client.New(s)`—an in-process loopback client backed by `adapter.WatchServerToWatchClient` (`chan_stream.go`). Worse, on every `Lock` or `Campaign` RPC, `v3lock` and `v3election` call `concurrency.NewSession(..., WithLease(req.Lease))` (spawning a background `KeepAlive` goroutine) and **immediately call `s.Orphan()` on the very next line**! Read-only `Election.Leader` and `Election.Observe` even call `es.session(ctx, -1)`, starting and orphaning a `KeepAlive` for bogus lease ID `-1` on every call.

- **Code References:** `client/v3/concurrency/mutex.go:75-155 @ v3.7.3`; `client/v3/concurrency/key.go:25-64 @ v3.7.3`; `server/embed/serve.go:148-150 @ v3.7.3`; `server/etcdserver/api/v3client/v3client.go:30-40 @ v3.7.3`; `server/etcdserver/api/v3lock/lock.go:35-58 @ v3.7.3`; `server/etcdserver/api/v3election/election.go:40-136 @ v3.7.3`; `jetcd-core/src/main/java/io/etcd/jetcd/impl/LockImpl.java:60-87 @ jetcd-0.8.7`; `ElectionImpl.java:76-166 @ jetcd-0.8.7`.

---

### 4.2 Five Failure Modes Compared Against Native Session Locks

| Lock / Election Dimension | Chubby (`doc/chubby-osdi06.md` §§2.4, 2.6, 2.8) | etcd (`v3.7.3`) & `jetcd` (`0.8.7`) | Concrete Bug or Operational Hazard in etcd | Confidence |
|---|---|---|---|---|
| **1. How a Contender Waits for a Lock** | **Server-side wait (`Acquire()` on handle, `§2.6, :209`).** The master grants the lock directly when it becomes free. | **Client-side predecessor watch loop (`waitDeletes`, `key.go:25-64`).** | **Lock acquisition depends on Watch liveness:** if a Go contender's watch stream lands on a partitioned or stalled follower (no `WithRequireLeader` by default), `Lock()` hangs after the lock is released until the member recovers, the stream breaks, or the caller's context ends. Moreover, `v3lock`'s in-process `v3client` (`chan_stream.go`) is the exact path where the 4-way watch-cancellation deadlock ([#18879](https://github.com/etcd-io/etcd/issues/18879) open, PR [#21065](https://github.com/etcd-io/etcd/pull/21065) open; [#20716](https://github.com/etcd-io/etcd/issues/20716) closed) was reported and freezes in-process watches. | **High** |
| **2. Contender's Own Session Expires While Waiting** | **Immediate failure:** session expiry terminates all pending calls on the session's handles (`§2.8, :281`). | **`waitDeletes` ignores own-session expiry ([#19597](https://github.com/etcd-io/etcd/issues/19597), `mutex.go:88-105`).** | `waitDeletes` only watches keys with `CreateRevision <= m.myRev - 1` (`// TODO: early termination if the session key is deleted` at `mutex.go:88`), and `v3lock/lock.go:44` already closed `s.Done()` via `s.Orphan()`. If the waiter's lease expires while waiting, `Lock()` stays blocked until all earlier holders unlock. | **High** |
| **3. Unlock Safety (Releasing a Lock)** | **Handle-based `Release()` (`§2.6, :209`) with file-instance (§2.3, §2.6, `:124, 217`) and handle sequence numbers (§2.3, §2.9, `:134, 312`).** | **Unfenced `Delete(ctx, m.myKey)` ([#22082](https://github.com/etcd-io/etcd/issues/22082), `mutex.go:115, 134-150`, `v3lock/lock.go:52-57`).** | In etcd, `m.myKey` is `<pfx>/<leaseID>` (omitting `myRev`) and `Unlock()` deletes `m.myKey` without `If(m.IsOwner())` (`CreateRevision == myRev`), so a delayed or retried `Unlock()` deletes a newer lock re-acquired under the same session. (In Chubby, closed handles are remembered so delayed packets cannot recreate them, §2.9, `:312`, though the paper does not detail whether `Release()` checks the lock generation number on an unclosed handle.) | **Moderate** |
| **4. Protection Against Delayed Packets (Fencing & `lock-delay`)** | **Sequencers (`GetSequencer()` / `CheckSequencer()`, `§2.4, :157`) + `lock-delay` (up to `1 min` on holder failure, `:159`).** | **`CreateRevision` compare (`Mutex.IsOwner()`, `mutex.go:152-153`); zero `lock-delay`.** | **Shared requirement:** Neither Chubby nor etcd provides mutual exclusion for external resources without fencing tokens (`sequencer` $\leftrightarrow$ `CreateRevision`). For legacy external services that cannot check tokens, Chubby's `lock-delay` provides up to 1 minute of protection on holder failure; etcd wakes the next waiter immediately on lease revocation with zero delay. | **High** |
| **5. Operation Ordering After Session Expiry** | **Handle suffix-failure rule (`§2.8, p. 8, :281-282`):** once a session expires, *all* subsequent calls on handle `H` fail, so only a suffix of operations is lost. | **Stateless KV calls:** `Put`/`Get`/`Delete`/`Txn` take no session handle. | Unless the developer manually includes `If(m.IsOwner())` in every `Txn`, operations issued after session loss still execute on the server. | **High** |

---

## 5. Reverse Trade-Offs: Where Chubby Pays the Price (and Where It Failed)

Testing our architectural hypothesis requires examining the **reverse trade-offs**: what operational costs did Chubby pay for coupling caches, events, and locks to master-held KeepAlives, and where did Chubby's own design fail?

### 5.1 Blocking Writes on Slow or Failed Readers (Up to 12s–60s Stalls) and Why That Breaks Kubernetes (**High confidence**)

Under Chubby's pre-write cache invalidation barrier (`doc/chubby-osdi06.md:231-240`, §2.7, pp. 6–7), if $K$ sessions currently cache a node and session $S_i$ takes time $d_i$ to either acknowledge invalidation via its next `KeepAlive` or allow its remaining lease to expire, a write must wait:

$$\text{InvalidationWait}(x) = \max_{i \in \{1 \dots K\}} d_i$$

A transient pause shorter than the lease delays writes by the length of the pause; a clean process shutdown ends the session explicitly (`doc/chubby-osdi06.md:255`, §2.8); and if even **one** of those $K$ caching readers crashes or is cut off by a network partition, $d_i$ equals that dead reader's remaining session lease—**`12s` by default, and up to `~60s` when the master has scaled up lease extensions under load** (**High confidence**):
> *"Writes (which include lock operations) are delayed a further 5-10ms by the database log update, **but by up to tens of seconds if a recently-failed client cached the file.** Even this variability in write latency has little effect on the mean request latency at the server because writes are so infrequent."* (`doc/chubby-osdi06.md:440`, §4.1, p. 11)

Furthermore, because the master marks a node **uncachable** while invalidations are pending so concurrent reads never block, a 12s–60s invalidation stall caused by one dead reader forces **all other readers polling that hot file to bypass their caches and hit the master with uncached read RPCs** (§2.7, `doc/chubby-osdi06.md:241`, **High confidence**).

Why did this trade-off succeed in Chubby, and why would it fail in Kubernetes? (**High confidence**)
- **Chubby's read-dominated workload (**High confidence**):** In Google's production cells (where a single master could communicate with up to `90,000` clients at peak, §4.3, `:468`, and an average of `~10` clients cached a given file, §4.1, `:430`), `KeepAlive` accounted for `93%` of RPCs, reads for `~3.4%`, `SetContents` for **`680 ppm` (`0.068%`)**, and `Acquire` for **`31 ppm` (`0.0031%`)**. Because writes were `< 0.1%` of traffic, trading write tail latency on reader crashes for consistent cached reads was a clear win.
- **Kubernetes's continuous-write full-replica workload (**High confidence**):** In Kubernetes, the etcd caching clients are `kube-apiserver` instances (`watchCache`), each caching **every object of every resource type**, while the cluster commits hundreds to thousands of writes per second (`Pod` status updates, node `Lease` heartbeats, `Events`). Under Chubby's blocking invalidation rule, **every etcd write would wait for every caching `kube-apiserver` to acknowledge invalidation**—so a single `kube-apiserver` experiencing a pause or network partition would **stall writes across the entire Kubernetes cluster for as long as it remained unresponsive, up to its remaining lease (`12s–60s`)**. Moreover, because a `watchCache` is a full state replica requiring `PrevKv` to compute watch diffs, invalidating without pushing the new value would force every `kube-apiserver` to issue an extra read RPC on every write.
- **Paper References:** `doc/chubby-osdi06.md:59` (§2), `:231-243` (§2.7), `:255, 259` (§2.8), `:352` (§3), `:393-440` (§4.1), `:468` (§4.3), `:524` (§4.5).

---

### 5.2 Chubby's §4.4 Failover Stale-Read Design Flaw and the Mandatory Lease-Timeout Barrier (**High confidence**)

Does holding a `KeepAlive` RPC at the master automatically guarantee cache consistency across master failovers? Chubby's own history in §4.4 proves it does not (**High confidence**):

1. **Original disk-logged session design (§2.9, p. 9 — **High confidence**):**
   Originally, every session was written to the replicated Berkeley DB log upon creation. On failover, the new master read the recorded sessions from disk, emitted a `fail-over` event (flushing client caches), and waited in step 6 until every recorded session either acknowledged the `fail-over` event or expired before allowing writes (step 7).
2. **The §4.4 startup-overload optimization and the resulting stale-read hazard (§4.4, pp. 12–13 — **High confidence**):**
   Logging new sessions to disk caused severe Paxos/DB overload when thousands of clients started simultaneously. Google therefore changed the master to log a session to disk only when it first attempted a write, lock acquisition, or ephemeral-file open (plus probabilistically on `KeepAlive` for read-only sessions). That introduced a design flaw that permitted stale reads (described as rare in practice):
   > *"Though it was necessary to avoid overload, this optimization has the undesirable effect that young read-only sessions may not be recorded in the database, and so may be discarded if a fail-over occurs. Although such sessions hold no locks, **this is unsafe; if all the recorded sessions were to check in with the new master before the leases of discarded sessions expired, the discarded sessions could then read stale data for a while.**"* (`doc/chubby-osdi06.md:488`, §4.4, p. 13)
3. **The availability price of the fix (**High confidence**):**
   To eliminate that stale-read window (and allow proxies to manage sessions unknown to the master, §4.4, `:492`), Chubby stopped logging sessions to disk altogether and forced every newly elected master to **wait a full worst-case lease timeout (`12s–60s`) before allowing operations to proceed** (§4.4, `doc/chubby-osdi06.md:490`). Whereas etcd can resume serving operations as soon as a new leader completes its election (~1–2 seconds estimated from default election timeouts), Chubby's failover barrier pauses operations for a full lease period on every master failover (typical failover `14s`).
- **Paper References:** `doc/chubby-osdi06.md:301-311` (§2.9), `:396` (§4.1), `:476-493` (§4.4).

---

### 5.3 Lossy Failover Events, TCP-to-UDP Migration, and Client Library Complexity (**High confidence**)

1. **Lossy events on failover (**High confidence**):** Because Chubby has no multi-version history log, events are lost across master failovers (§2.5, §2.9, `doc/chubby-osdi06.md:167, 309`). Many Google developers crashed their processes upon receiving the `fail-over` event (§4.6, `:534`), leading Burrows to conclude: *"We might have done better to send redundant 'file change' events instead, **or even to ensure that no events were lost during a fail-over.**"*
2. **TCP congestion backoff vs. KeepAlive deadlines (**High confidence**):** Multiplexing events and invalidations onto time-critical `KeepAlive` RPCs over TCP caused widespread session drops during network congestion, forcing Chubby to move `KeepAlive` onto UDP and propose a separate TCP `GetEvent()` stream (`doc/chubby-osdi06.md:542-546`, §4.6).
3. **Delicate 7,000-line C++ client library and Java sidecar (**High confidence**):** Chubby's session, cache, `Jeopardy`, and handle-recreation state machine required a delicate 7,000-line C++ client library, forcing Java callers to run a local C++ protocol-conversion sidecar rather than maintain a second implementation (`doc/chubby-osdi06.md:452-459`, §4.2). In etcd, while `jetcd` delegates lock and election recipes to the server's `v3lock` and `v3election` gRPC services (§4.1), re-implementing lease keepalive and watch resumption independently in Go (`client/v3`) and Java (`jetcd`) introduced distinct client defects in each binding (§2.4, §3.1).

---

## 6. Actionable Borrowings for etcd (Without Blocking Writers on Readers)

We can separate Chubby's architecture into three layers:
- **Layer A (Session Liveness & Consensus Fencing):** Conservative client deadlines, request-send throttling, a single per-client session state machine with `Jeopardy`, separation of active sessions from passive TTL expiration, and deterministic entry-term revocation fencing.
- **Layer B (Coupling Watch Freshness Detection to Session Liveness):** Validating watch stream progress against a leader-asserted session heartbeat and entering `Jeopardy` (disabling cached reads) whenever watch freshness lags behind.
- **Layer C (Synchronous Pre-Write Invalidation Barrier):** Blocking every server write until all caching readers acknowledge invalidation or expire.

**Layer C** is what guarantees that a cached read is strictly linearizable between heartbeats, at the price of 12s–60s write stalls on dead readers (**High confidence**). **Layers A and B**, while not providing zero-RPC linearizable reads on their own (since a write can commit between two heartbeats unless a per-read `ReadIndex` or Layer C is used), fix **stall detection, bounded staleness, and session liveness** while keeping etcd's write path completely non-blocking (**High confidence** on `B1`/`B6`; **Moderate confidence** on unprototyped `B2`–`B5` architectural synthesis).

| # | Chubby-Inspired Improvement for etcd | Target Layer | Exact Mechanism | Bugs & Bottlenecks Eliminated | Trade-Offs / Failure Conditions | Confidence |
|---|---|---|---|---|---|---|
| **B1** | **Conservative Client Lease Deadlines + Request-Send Throttling** | Client (`client/v3/lease.go:542-607`, `jetcd` `LeaseImpl.java:234-266`) | 1. Compute `ka.deadline = send_time + TTL - drift_margin` instead of `receipt_time + TTL`.<br/>2. Advance `ka.nextKeepAlive` (with backoff) when a `LeaseKeepAliveRequest` is *sent* so overdue replies do not trigger 500ms (`2 req/s`) resend floods. | Closes the stale-read window in `client/v3/leasing` (§3.2 Failure Mode L-B, `#19091`) and optimistic `Session.Done()` overrun (`L5`); eliminates the 500ms overdue-reply resend amplification (§2.2 `L1`). | Cannot prevent stale actions during a local OS/JVM process pause between checking `Done()` and acting (which requires fencing tokens). | **High** |
| **B2** | **Unified Client Session State Machine (`Safe` $\leftrightarrow$ `Jeopardy` $\rightarrow$ `Expired`)** | Client (`client/v3/concurrency`, `jetcd`) | Bind 1 lease and all associated watches/caches to a single session object. Enter `Jeopardy` (pausing cached reads and notifying the app without destroying the session or watch cursor) when the conservative deadline nears or the watch stream disconnects during an election; transition to `Safe` on reconnect or `Expired` after a grace period. | Eliminates per-subscriber `WithRequireLeader` bugs ([#22081](https://github.com/etcd-io/etcd/issues/22081), [#22094](https://github.com/etcd-io/etcd/issues/22094)), `jetcd` `onError` zombie renewals (`LeaseImpl.java:273`), and `WithRequireLeader` election watch teardowns ([kubernetes#111116](https://github.com/kubernetes/kubernetes/issues/111116), `W3`). | Applications must handle the `Jeopardy` callback by quiescing unfenced side effects and not crashing on failover (`§4.6, :534`). | **High** (bug elimination); **Moderate** (unprototyped API) |
| **B3** | **Leader-Asserted `(RaftTerm, Revision)` Freshness in `LeaseKeepAliveResponse` Coupled to Watch Progress** | Server (`v3rpc/lease.go:127-145`, `leasehttp/http.go:101`) + Client | After `ensureLeadership()`, stamp `LeaseKeepAliveResponse.Header` with the **leader's current `Revision` and `RaftTerm`** (and propagate it through `leasehttp`). A client session compares its watch stream's processed revision (via `ProgressNotify`) against the leader revision from its `KeepAlive` reply; if the watch lags behind an old `KeepAlive` revision by more than one heartbeat interval, the session enters `Jeopardy` and switches the watch to another endpoint. | Detects silent watch stalls on partitioned followers or apply-stalled nodes (`W1`, `W2`, [kubernetes#59848](https://github.com/kubernetes/kubernetes/issues/59848)), eliminates split-stream writer blocks in `client/v3/leasing` (§3.2 Failure Mode L-A), and bounds watch-cache staleness to the heartbeat interval. | Requires fixing server-wide `ProgressNotify` suppression ([#20242](https://github.com/etcd-io/etcd/issues/20242)). Bounds staleness to the heartbeat interval; strict linearizable cached reads (`KEP-2340`) still need a per-read `ReadIndex` unless Layer C is used. | **Moderate** (sound design, unprototyped) |
| **B4** | **Separate Active `Session` Leases (P3) from Passive Key Expiration Deadlines (P2)** | Server (`server/lease`, `mvcc`) | Keep `Session` leases in memory (extended on `Promote()`, never checkpointed). Implement passive TTL garbage collection (for Kubernetes `Event`s) as an expiration timestamp in MVCC key metadata swept in bounded batches by the leader. | Removes the P2/P3 conflict under leader churn ([kubernetes#65497](https://github.com/kubernetes/kubernetes/issues/65497), [#9888](https://github.com/etcd-io/etcd/issues/9888)) and eliminates `LeaseCheckpoint` and bug [#22649](https://github.com/etcd-io/etcd/issues/22649); bounded-batch sweeping avoids 1,000-key single-txn revocation stalls ([kubernetes#96836](https://github.com/kubernetes/kubernetes/issues/96836), [kubernetes#110210](https://github.com/kubernetes/kubernetes/issues/110210)). | Requires an `mvccpb.KeyValue` storage/API field change, leader-term fencing on sweeper deletes (`B5`), and reasonably synchronized wall clocks across members. | **Moderate** (sound design, unprototyped) |
| **B5** | **Deterministic Raft-Entry-Term Fencing on Expiry-Driven `LeaseRevokeRequest` Proposals** | Server (`lessor.go`, `v3_server.go:471`, `server.go:1904`, `apply/backend.go:94`) | Record the decision term `req.LeaderTerm` on expiry-driven `LeaseRevokeRequest`s and, at apply time, plumb the committed log entry's term `e.GetTerm()` (`server.go:1904`) into the applier, rejecting the revoke if `req.LeaderTerm < e.GetTerm()`. | Deterministically fences stale revocation `MsgProp`s queued in term $T$ (`R1`/`R2`) that cross a step-down and are forwarded by followers into term $T' > T$ (where they receive `e.GetTerm() = T'`). | Requires a proto/WAL field on `LeaseRevokeRequest`. Does not fence `R3` (stale follower-forwarded renewals) or an entry already appended to the Raft log in term $T$ before election and committed in term $T'$ (which retains `e.GetTerm() = T` and is ordered before the new leader's no-op entry). | **High** (on `MsgProp` fencing); **Moderate** (unprototyped change) |
| **B6** | **Fenced `Unlock`, Own-Key Deletion Watch in `waitDeletes`, and Optional `lock-delay`** | Client & Server (`mutex.go:88, 134-150`, `v3lock/lock.go:35-58`) | 1. Guard `Unlock()` with `If(m.IsOwner())` (`CreateRevision == myRev`).<br/>2. In `waitDeletes`, also watch the waiter's own key `m.myKey` (per the `TODO` at `mutex.go:88`)—or stop calling `s.Orphan()` before `m.Lock(ctx)` in `v3lock/lock.go:44`—so own-lease expiry aborts the wait without breaking `v3lock`.<br/>3. Support optional `lock-delay` on expiry. | Fixes [#22082](https://github.com/etcd-io/etcd/issues/22082) (unfenced `Unlock` deleting newer lock) and [#19597](https://github.com/etcd-io/etcd/issues/19597) (waiter hanging after own-lease expiry), and provides Chubby's `lock-delay` buffer for unfenced external resources. | `v3lock.UnlockRequest` (`v3lock.proto`) currently passes only `key` (`<pfx>/<leaseID>`), so fencing `v3lock.Unlock` requires encoding `myRev` into the returned lock key/token. | **High** (on `Unlock`/`waitDeletes` fixes) |

---

## 7. Verification Matrix and Reproducible Functional Tests

### 7.1 Verification vs. Inference Matrix

| Claim | Verified vs. Inferred & Confidence | Primary Evidence (`doc/chubby-osdi06.md`, `etcd @ v3.7.3`, `jetcd @ 0.8.7`) | Condition Under Which Conclusion Fails |
|---|---|---|---|
| Chubby holds `KeepAlive` RPCs blocked at the master, distinguishes pre-write blocking cache invalidations (§2.7) from post-write events (§2.5), and enforces the `Ack-or-Expired` invariant (`§2.8, §4.6`). | **Verified** in paper (**High confidence**). | §2.5, §2.7, §2.8, §4.6, `doc/chubby-osdi06.md:163-182, 229-262, 542-546`. | None; explicitly stated in the paper. |
| Chubby's own §4.4 optimization (deferring disk writes for read-only sessions) introduced a design flaw that permitted stale cached reads across rapid failovers (noted as rare in practice), forcing new masters to wait a full worst-case lease timeout (`12s–60s`) before allowing operations to proceed. | **Verified** in paper (**High confidence**). | `doc/chubby-osdi06.md:476-493` (§4.4, pp. 12–13). | None; documented directly by Burrows in §4.4. |
| Both Go `client/v3` (`lease.go:556, 590-610`) and Java `jetcd` (`LeaseImpl.java:234-266`) update `nextKeepAlive` only upon *receiving* a response and run a 500ms send ticker, causing them to re-send `LeaseKeepAliveRequest` every 500ms (`2 req/s` per lease) whenever a reply is overdue, and default to a `5s` (`6s` when `DialTimeout = 5s`) first-keepalive timeout. | **Verified** in source code (`v3.7.3` & `jetcd-0.8.7`, **High confidence**). | `client/v3/lease.go:81, 86, 192-205, 296, 542-557, 586-619 @ v3.7.3`; `server/etcdserver/api/v3client/v3client.go:37 @ v3.7.3`; `LeaseImpl.java:61, 234-266, 340-346 @ jetcd-0.8.7`. | None for `v3.7.3` / `jetcd-0.8.7`; verified directly in source. |
| In `client/v3/leasing` (`kv.go`, `cache.go`), (a) read misses cost 1 Raft `Txn` + 1 `Watch`, allow only 1 caching client per key, and force every non-owner linearizable read of an owned key through Raft (`acquire`); (b) because `put`/`delete` check `wc == nil`, every non-owner write is applied twice (`2` Raft proposals uncontested, `5` proposals contested) with wrong `Delete`/`PrevKv` responses; (c) split-stream routing (`resolver.go:44`) between `LeaseKeepAlive` and `monitorLease` (`kv.go:160, 174`) can block writers in `waitRescind` until context expiry; (d) `readySession()` (`kv.go:467-479`) serves stale cached reads for `RTT + up to 1s` after server lease expiry (`#19091`); and (e) `EvictRange` (`cache.go:183`, `#21869`) deletes `key` instead of `k`. | Items (a), (b), and (e) **verified in code and reproduced against `v3.7.3` (High confidence)**; schedule (d) **verified in code and upstream `#19091` (High confidence)**; schedule (c) **code path verified (High confidence), multi-endpoint partition schedule inferred (Moderate confidence)**. | `client/v3/leasing/kv.go:159-350, 380-479 @ v3.7.3`; `client/v3/leasing/cache.go:28, 53-61, 113-118, 178-190 @ v3.7.3`; `server/etcdserver/txn/txn.go:352-363 @ v3.7.3`; `client/v3/internal/resolver/resolver.go:44 @ v3.7.3`; [`chubby-perspectives-critique-claude.md` Appendix A](chubby-perspectives-critique-claude.md). | Schedule (c) requires a multi-endpoint client where `round_robin` places `LeaseKeepAlive` and `monitorLease` on different members (with a single endpoint or behind a single proxy, both streams share fate). |
| Etcd's server-side `v3lock` and `v3election` services are loopback wrappers over `v3client.New(s)` (`chan_stream.go`) that start and immediately `Orphan()` a `concurrency.Session` on every call, depend on `Watch` delivery (`waitDeletes`), and expose unfenced `Unlock` (`#22082`) and expired-waiter hangs (`#19597`). | **Verified** in source code (**High confidence**). | `server/embed/serve.go:148-150 @ v3.7.3`; `server/etcdserver/api/v3client/v3client.go:30-40 @ v3.7.3`; `server/etcdserver/api/v3lock/lock.go:35-58 @ v3.7.3`; `server/etcdserver/api/v3election/election.go:40-136 @ v3.7.3`; `client/v3/concurrency/{mutex,key}.go @ v3.7.3`. | None for `v3.7.3`. |

---

### 7.2 Functional Verification Tests (Runnable in Docker)

While performance benchmarking is reserved for dedicated hardware (`AGENTS.md` §7), four small functional tests can be executed in Docker against `etcd v3.7.3` to demonstrate the failure mechanisms analyzed above:

1. **T1 (`client/v3/leasing` Double-Applied Non-Owner Writes, 5-Proposal Contested Writes, and Non-Owner Read Proposals — §3.2 Failure Mode L-C, reproduced in [`chubby-perspectives-critique-claude.md` Appendix A](chubby-perspectives-critique-claude.md)):**
   - Start a `v3.7.3` container and create two `leasing.NewKV` clients (`lkvA`, `lkvB`).
   - Verify via `etcd_server_proposals_committed_total`, store `Revision`, key `Version`, and a plain watcher that: (i) `lkvB.Put(ctx, "k1", "v1")` on an unowned key commits **2 proposals (+2 revisions, `Version = 2`)**; (ii) after `lkvA.Get(ctx, "k2")` acquires ownership, `lkvB.Put(ctx, "k2", "v2")` commits **5 proposals (+4 revisions, `Version` $1 \to 3$)**; (iii) `lkvB.Delete(ctx, "k3", WithPrevKV())` deletes `"k3"` but returns **`Deleted = 0, len(PrevKvs) = 0`**; (iv) `lkvB.Put(ctx, "k4", "new", WithPrevKV())` over `"old"` returns **`PrevKv.Value = "new"`**; and (v) 5 linearizable `lkvB.Get(ctx, "k5")` calls while `lkvA` owns `"k5"` commit **5 Raft proposals**.
2. **T2 (`client/v3/leasing` Indefinite Writer Block Under Split-Stream Partition — §3.2 Failure Mode L-A):**
   - Start a 3-node `v3.7.3` cluster `[M1, M2, M3]`. Connect `lkv1` (`leasing.NewKV`) with all three endpoints and call `lkv1.Get(ctx, "k")` so `lkv1` caches `"k"`.
   - Confirm via gRPC streams that `lkv1`'s `LeaseKeepAlive` stream is on $M_1$ (leader) and its `monitorLease` `Watch` stream is on follower $M_2$.
   - Isolate $M_2$ from $\{M_1, M_3\}$ using `iptables` while keeping `lkv1 <-> M2` open.
   - Have `lkv2` call `lkv2.Put(ctx, "k", "v2")` on $M_1$. Verify that `lkv2.Put` blocks in `waitRescind` until its context expires while `lkv1` continues renewing its session lease on $M_1$.
3. **T3 (`client/v3/leasing` Stale "Linearizable" Read Window After Partition — §3.2 Failure Mode L-B, upstream `#19091`):**
   - Have `lkv1` cache `"k" = "v1"` with a 5s lease TTL. Inject a 300ms artificial network delay on `LeaseKeepAliveResponse` (or pause traffic between `lkv1` and the cluster after a renewal, as in `TestLeasingGetChecksForExpiration` from `#19091`).
   - Immediately after the leader revokes `lkv1`'s lease at $t = 5.0\text{s}$ and `lkv2.Put(ctx, "k", "v2")` commits on the leader, call `lkv1.Get(ctx, "k")` on the isolated owner during $[5.0\text{s}, 6.2\text{s}]$. Verify that `lkv1.Get` returns the stale value `"v1"` as a linearizable read.
4. **T4 (`client/v3` and `jetcd` 500ms Overdue-Reply Resend Amplification — §2.2):**
   - Grant a 6s lease (`TTL / 3 = 2s`) and delay the server's `LeaseKeepAliveResponse` at $t = 2\text{s}$ by 3 seconds (e.g., via a proxy or follower `leasehttp` stall). Verify that between $t = 2.0\text{s}$ and $t = 5.0\text{s}$, both `client/v3` (`lease.go:590-610`) and `jetcd` (`LeaseImpl.java:234-247`) transmit 6 back-to-back duplicate `LeaseKeepAliveRequest`s (one every 500ms).

---

## Conclusions

1. **The Hypothesis Holds for Session Liveness, Watch Freshness Detection, and Lock Wait Safety (**High confidence**):**
   Most of etcd's highest-severity client coordination failures—silent watch stalls on partitioned followers (`k8s#59848`), the lifecycle failure modes of `WithRequireLeader` (`k8s#111116`), the split-stream writer hang and stale-read window (`#19091`) in `client/v3/leasing`, watch-dependent lock hangs (`#19597`, `#18879`/`#20716`), and per-lease subscriber bugs (`#22081`, `#22094`, `jetcd` `onError`)—trace directly to etcd's decision to split **liveness (`LeaseKeepAlive`)**, **notification (`Watch`)**, and **mutual exclusion (`concurrency.Mutex`)** into independent, unlinked primitives that can even land on different cluster members under `round_robin` load balancing. Chubby prevents this class of failures by anchoring liveness and notifications to a single master-bound **Session** governed by conditional renewal (**`Ack-or-Expired`**), conservative client lease math, and the **`Jeopardy`** grace-period state machine.
2. **Separating Active Sessions from Passive Key Expiration Eliminates the `LeaseCheckpoint` Quagmire (**High confidence** on mechanisms; **Moderate confidence** on comparative design assessment):**
   Because Chubby restricts leases to short-lived active client sessions (`12s–60s`) rather than overloading them for 1-hour passive data garbage collection, failover lease extension in Chubby never triggers the unbounded object retention (`k8s#65497`, `#9888`), checkpoint-clear bugs (`#22649`), or 1,000-key batch revocation stalls (`k8s#96836`, `k8s#110210`) seen in etcd.
3. **Consensus Fencing, Per-Read Linearizability, and Lossless History Still Require Mechanisms Beyond KeepAlives (**High confidence**):**
   Session KeepAlives do not by themselves prevent deposed-leader split-brain (which requires Chubby's **master leases** and **master epochs**, or deterministic entry-term fencing on `LeaseRevokeRequest` proposals in etcd), nor do they fence external services against delayed network packets (which still requires **sequencers** or **`CreateRevision`** checks). Conversely, Chubby's lack of an MVCC history log means Chubby drops events on failover (causing many applications to crash on `fail-over` events, `§4.6, :534`) and blocks writes for up to `12s–60s` whenever a caching reader fails—a trade-off that worked for Chubby's `< 0.1%` write rate but would stall a Kubernetes control plane.
4. **The Best Path Forward for etcd Is Selective Synthesis (§6, `B1–B6` — **High confidence** on `B1`/`B6`; **Moderate confidence** on unprototyped `B2`–`B5` architectural synthesis):**
   By adopting Chubby's **conservative client lease deadlines and request-send throttling (`B1`)**, **single-session `Jeopardy` state machine (`B2`)**, **leader-revision freshness stamped onto `LeaseKeepAliveResponse` and coupled to watch progress (`B3`)**, **separation of active sessions from passive TTL expiration (`B4`)**, **deterministic entry-term fenced lease revocations (`B5`)**, and **fenced `Unlock` with own-key deletion watch (`B6`)**, etcd and `jetcd` can eliminate their dominant lease and watcher liveness hazards while preserving etcd's non-blocking write path and lossless MVCC event stream.
