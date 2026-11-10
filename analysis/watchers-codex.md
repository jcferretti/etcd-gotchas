# etcd watches: guarantees, failure modes, and production implications

- **Investigation dates:** original audit 2026-10-31; peer review and expansion 2026-11-01; eventual-consistency and liveness review 2026-11-03; Jepsen and upstream-fix review 2026-11-08, America/New_York. The latest review adds documentation and source checks, not runtime results (§A.6).
- **Versions analyzed:** etcd **v3.7.3**, commit `0a173d0410a96d01091b484daae81365bbcde57e`; jetcd **jetcd-0.8.7**, commit `23b61f1499a5d340ae2f267d88d45ca3900425a4`. Selected Kubernetes source at **v1.37.2**, commit `c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32`, read remotely. Historical reports retain their own version qualifications.
- **Upstream references:** principal current findings include [#21813](https://github.com/etcd-io/etcd/issues/21813), [#22673](https://github.com/etcd-io/etcd/issues/22673), and [jetcd #1227](https://github.com/etcd-io/jetcd/issues/1227). Jepsen follow-up includes [etcd #11496](https://github.com/etcd-io/etcd/issues/11496)/[#11559](https://github.com/etcd-io/etcd/pull/11559) and [jetcd #1445](https://github.com/etcd-io/jetcd/pull/1445). The source inventory and issue-state dates are in Appendix A; references also accompany each finding.
- **Summary:** **High confidence:** committed KV history alone does not establish correct watch delivery, eventual progress, or cache freshness. Eventual consistency requires convergence under its failure assumptions; a watch that stays silently stale forever does not satisfy that requirement. The executive summary below separates verified defects from untested failure scenarios.

- **Confidence review:** 2026-11-06. Ratings reassessed against the cited evidence, with targeted checks of the pinned leader monitor, WAL/leader-state ordering, Go termination path, jetcd recovery code, and API/client-design documentation. Historical issue states and experiments retain their original dates; no runtime tests were added.

## Scope

Evaluate the guarantees, failure modes, and production implications of etcd v3.7.3 watches, its Go client and optional gRPC proxy, and jetcd 0.8.7, with Kubernetes examples and a Chubby comparison.

## Summary

etcd watches provide a revisioned change stream that applications can use to maintain derived state. Production consumers need both correct history and eventual progress. A silent watch can leave derived state permanently obsolete even when the application only requires eventual consistency, without requiring a linearizable cache or a fixed delivery deadline.

- **The liveness contract is incomplete — High confidence in the documentation and mechanisms.** etcd's own robustness README explicitly claims eventual consistency for Watch and defines it as eventual convergence. The API documentation permits nonarrival in unhealthy clusters without defining a sufficient condition for subscription recovery. A surviving quorum and `WithRequireLeader` do not ensure that each watch progresses or reports failure. Exact quotations and failure assumptions are in §1.5; a direct-member counterexample with the option enabled is in §2.6.
- **The contract separates event continuity from freshness — High confidence.** Events originate from Raft-committed writes, while subscriptions and delivery are member-local. Correct cursors, retained history, ordered replay, and truthful progress can preserve continuity across failover. The API nevertheless permits unbounded notification delay. A live connection or a response from a stale member cannot certify current cluster state (§1).
- **Failure detection is incomplete — High confidence in the mechanisms.** `WithRequireLeader` reacts to the member's stored leader ID, not storage, apply, or delivery progress. A stalled member can retain a nonzero leader ID and leave a direct watch open without new events or an error while the majority advances. Even successful leader-loss signaling terminates the Go watch; application recovery is still required. The separate 250 ms final-error drop leaves an observable channel close and must not be confused with an open, silent subscription (§2).
- **The standalone proxy adds verified hazards — High confidence.** An independent v3.7.3 experiment showed a proxied leader-required watch staying open and silent during a partition while the direct watch terminated. The proxy also lacks explicit progress-request handling. Other source findings cover ended proxy broadcasts, server/client cancellation blocking, and watch-ID reuse; each has distinct trigger conditions and verification limits (§4).
- **Recovery and progress need careful client state — High confidence in the mechanisms; Moderate for untested failure schedules.** A second experiment confirmed that a future watch suppresses requested progress only on its own stream. Go also has source-derived risks around initial revision-zero cursors and rejected-creation teardown. jetcd can advance a historical replay cursor on `Created` before receiving the events, and can stop retrying after an exception in scheduled resume. These client schedules still need targeted reproductions (§3 and §6).
- **Historical Jepsen evidence has a limited scope — High confidence.** Its 3.4.3 single-key watch result is positive evidence for the tested histories (§1.6). jetcd's default KV retry defect was fixed in 0.8.5; that change does not alter the separate watch-resumption mechanism in 0.8.7 (§6.5).
- **Kubernetes demonstrates both the impact and a workable defense — High confidence.** Stalled watches and corrupted caches have had distinct causes, including missing previous values, false progress, and delivery pressure. For consistent cache reads, Kubernetes obtains a quorum-derived revision and waits for the cache to reach it, with timeout handling. That defense does not bound all watch delivery or apply to reads that explicitly permit stale state (§5).

**Moderate confidence — recovery-policy recommendation:** The practical requirement is to track the application's **processed** revision, supervise progress independently where silent stalls are possible, and rebuild after a discontinuity. A separate freshness policy is needed when decisions require current data. Progress supervision can trigger recovery selectively; it need not mean blindly restarting every watch on a schedule (§7.2). Chubby addresses cache validity through session uncertainty and invalidation acknowledgments; etcd's retained replay history addresses a different part of the problem (§8).

The analysis is primarily a source and issue audit. Two small functional experiments establish the specific observations above; no benchmark or incident-frequency estimate is claimed. Confidence labels distinguish verified mechanisms from unexecuted end-to-end scenarios.

**Reading guide:** §§1–3 explain the contract and client behavior; §4 catalogs server, proxy, and cache findings; §§5–8 cover consumers and design implications; §9 lists remaining validation. Appendices retain the methods, historical fixes, reproduction details, and peer-review decisions. The closing Conclusions section states the assessment.

## 1. Watch contract and data path

Three questions guide the analysis: was the change committed, was its event delivered correctly, and has the application caught up far enough to use its view? A guarantee about one does not answer the others.

### 1.1 Terms

| Term | Meaning in this report |
|---|---|
| Watch | A subscription to matching key changes from an inclusive starting revision. |
| MVCC revision | A versioned-store revision identifying a committed change; it is not a Raft log index. |
| Serving member | The etcd member hosting the client's watch stream, whether leader or follower. |
| Resume cursor | The revision from which the next watch request should replay. Library receipt and application processing can have different cursors. |
| Progress notification | An empty watch response certifying delivery through its revision for the affected watch or watches. It does not certify quorum freshness. |
| Safety / liveness | Safety concerns the truth and completeness of delivered data; liveness concerns whether and when data or failure signals arrive. |

### 1.2 What the API promises

**High confidence in the documented guarantees; Moderate in the assessment of their combined liveness scope:** The [v3.7 API guarantees](https://etcd.io/docs/v3.7/learning/api_guarantees/#watch-apis) promise revision ordering, uniqueness, no missing intervening events within retained history, complete revisions, resumption from a known revision, and truthful progress notifications. They explicitly exclude watch linearizability and allow unbounded delay, including nonarrival in an unhealthy cluster. The safety guarantees constrain delivered events. They do not establish eventual progress or set a general deadline for the next event or failure signal. These are separate obligations: absence of a finite deadline does not, by itself, permit permanent nonconvergence (§1.5).

Several protocol details determine whether a client can use those promises correctly:

| Protocol detail (High confidence in the cited contract/code) | Consequence for client state | Verified source |
|---|---|---|
| `start_revision` is inclusive; zero means start from the receiving member's current point | A reconnect must preserve the replay position. Starting again at zero can intentionally skip the outage interval. | `api/etcdserverpb/rpc.proto:784 @ v3.7.3`; `server/storage/mvcc/watchable_store.go:128 @ v3.7.3` |
| `Created` acknowledges registration | It does not say that requested historical events have already been replayed. | `api/etcdserverpb/rpc.proto:839 @ v3.7.3`; `server/etcdserver/api/v3rpc/watch.go:340 @ v3.7.3` |
| Ordinary event responses and progress responses have different cursor significance | During catch-up, the response header can be ahead of the events in that batch. Resume from the last complete event revision, or a valid progress watermark, rather than indiscriminately from every header. | `server/storage/mvcc/watchable_store.go:360 @ v3.7.3`, `:384`, `:388`; `client/v3/watch.go:863 @ v3.7.3` |
| A compacted starting point or an irrecoverably lagging watcher is canceled with `compact_revision` | This is explicit loss of replayability. Rebuild state from a new snapshot; do not pretend that restarting at a newer revision preserves the missing history. | `api/etcdserverpb/rpc.proto:845 @ v3.7.3`; `client/v3/watch.go:63 @ v3.7.3` |
| Previous values can be absent after compaction | Receiving the delete event does not guarantee receiving the deleted object's full prior contents. | `api/etcdserverpb/rpc.proto:805 @ v3.7.3` |
| Raw gRPC clients may request fragmentation | They must assemble all fragments before treating a revision as complete. The Go client reassembles them. | `api/etcdserverpb/rpc.proto:816 @ v3.7.3`; `client/v3/watch.go:590 @ v3.7.3` |
| Watch delivery has no application-processing acknowledgment | A library's receipt cursor is not a durable record that the application has updated its own state. | Watch request variants in `api/etcdserverpb/rpc.proto:761 @ v3.7.3`; buffering in `client/v3/watch.go:825 @ v3.7.3`, `:867`, `:879` |

**High confidence:** Three examples separate the possible failures:

- Events at 100 and 120 arrive, but a matching retained event at 110 is omitted: **continuity is violated**.
- Progress through 120 arrives before that event at 110: **the progress guarantee is violated**.
- A final matching update commits at 110, but the watch remains at 100 forever: **eventual convergence has failed**, if the execution satisfies the promised liveness assumptions. This needs neither an interior event gap nor a missed fixed deadline. A finite observation of silence alone cannot prove permanent failure.

Each case needs different evidence and a different remedy.

Revision gaps by themselves are not evidence of loss: unrelated keys also advance the global revision. A prefix watcher needs an independent expected history or a consistent snapshot comparison to establish that a matching event was omitted.

### 1.3 How committed writes become local watch events

**Verified, High confidence:** The normal data path is:

```text
Raft-committed write
  -> member applies the committed entry
  -> MVCC transaction records changes and emits watch events
  -> local watcher queues / historical replay
  -> gRPC stream
  -> client buffer
  -> application state
```

Each member applies Raft-committed entries to its local store. MVCC transaction completion constructs and publishes the corresponding mutation events. Watch registration calls the local MVCC watch stream directly: it neither replicates the subscription through Raft nor obtains a linearizable read barrier for each response.

**Source evidence:** `server/etcdserver/raft.go:218 @ v3.7.3` passes `CommittedEntries` into the apply machinery; `server/etcdserver/server.go:1945 @ v3.7.3` applies normal entries; `server/storage/mvcc/watchable_store_txn.go:22 @ v3.7.3` publishes events; `server/etcdserver/api/v3rpc/watch.go:340 @ v3.7.3` registers the local watch.

Each watcher tracks the next revision it needs. A watcher behind the local store enters the unsynchronized group, where `syncWatchers` reads and sends retained history. Blocked output enters a retry path. When a member installs a leader's snapshot, its synchronized watchers move into the unsynchronized group so they can reconcile against the new store.

**Source evidence:** `server/storage/mvcc/watchable_store.go:128 @ v3.7.3`, `:207`, `:284`, `:346`, and `:468`.

**Inference, High confidence:** A new serving member can resume a subscription from committed history without replicating the subscription itself. This requires a correct cursor, ordered replay, truthful progress, and an explicit compaction error when the needed history is gone. Raft establishes the underlying history; the watch implementation and client must preserve those delivery properties.

This argument assumes a surviving committed history, a reachable member that eventually makes progress, retention of the needed revisions, and correct delivery/client implementations. It does not cover restoring an old backup as though it were the same uninterrupted history.

### 1.4 How reconnect preserves a replay position

**Verified, High confidence:** The Go client recreates streams on recoverable receive errors and resubmits watches. Its resume revision follows the last event's modification revision plus one, with progress responses handled separately. The already-received buffer remains on the watcher across stream recreation. Evidence: `client/v3/watch.go:665 @ v3.7.3`, `:789`, `:863`, and `:928`.

Reconnection succeeds at several different stages: transport opens, the server registers a watch, the library receives events, and the application processes them. Completion of an earlier stage does not prove completion of a later one. A compaction error correctly reports that replay is no longer possible; the application must then reconstruct its state.

### 1.5 Eventual consistency requires eventual progress

**Assessment, High confidence:** The conceptual objection is justified: eventual consistency permits unknown convergence time, not permanent nonconvergence under the conditions in which convergence is promised. The earlier version of this report emphasized freshness and bounded failure notification too narrowly. A consumer can accept stale data temporarily and still be incorrect if it remains stale forever.

#### Exact upstream wording and its provenance

| Claim | Exact quotation | Source and scope |
|---|---|---|
| Watch provides eventual consistency | “Etcd provides strict serializability for KV operations and eventual consistency for Watch.” | Official repository documentation, [`tests/robustness/README.md:111 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/README.md#L111). Verified in the pinned local checkout; this explicit documentation claim was missed in the first liveness review. |
| Eventual consistency requires convergence | “converge to the same view eventually” | The same README's definition at [`:102 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/README.md#L102). |
| Maintainer explanation | “However as watch is eventually consistent we cannot guarantee linearizability.” | Maintainer `serathius`, accepted answer in [discussion #15423](https://github.com/etcd-io/etcd/discussions/15423), 2023-03-08. This corroborates the repository documentation; the claim is not limited to an informal discussion. |
| Events may never arrive | “events in unhealthy clusters might never arrive” | [v3.7 API guarantees, APIs to consider](https://etcd.io/docs/v3.7/learning/api_guarantees/#apis-to-consider). The paragraph does not define the failure conditions under which eventual watch delivery resumes. |
| A watch can remain on a partitioned member | “In order to prevent a watch stream being stuck in a partitioned node, make sure to wrap context with "WithRequireLeader".” | Public Go API documentation, [`client/v3/watch.go:82 @ v3.7.3`](https://github.com/etcd-io/etcd/blob/v3.7.3/client/v3/watch.go#L82). This warning also identifies a mitigation. |
| A partition can preserve service | “The majority side becomes the available cluster and the minority side is unavailable.” | [v3.7 failure guide, Network partition](https://etcd.io/docs/v3.7/op-guide/failures/#network-partition). This describes cluster availability, not automatic migration of every existing subscription. |
| Client liveness is a stated requirement | “Clients should never deadlock waiting for a server to come back from offline, unless configured to do so.” | [v3.7 client design, Client Requirements](https://etcd.io/docs/v3.7/learning/design-client/#client-requirements). This is a design requirement; the page also describes historical balancers, so its implementation descriptions need version-specific source checks. |

These sources support the concern jointly. There is no single located documentation sentence asserting that every leader-required watch can remain silent forever while its client can reach a functioning majority. The stronger cases require the code and failure-path analysis below.

#### What eventual consistency requires

Werner Vogels defines the convergence obligation as: “if no new updates are made to the object, eventually all accesses will return the last updated value.” See [*Eventually Consistent — Revisited*](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html), 2008-12-23. Bailis and Ghodsi distinguish this eventual-convergence obligation from safety properties in [*Eventual Consistency Today: Limitations, Extensions, and Beyond*](https://people.eecs.berkeley.edu/~alig/papers/eventual-consistency-limitations-extensions.pdf), pp. 1, 4.

**Application to watches — High confidence in the conditional convergence argument:** These definitions concern observed state. For a cache maintained by a watch, consider a final update from `x=old` to `x=new`, followed by no more updates to x. Under the service's liveness assumptions, a functioning consumer must eventually reflect `x=new`. Delivering an ordered, internally complete prefix ending before that update does not meet this condition. A lack of a known upper time bound is compatible with convergence; an infinite wait is not. Recovering every intermediate transition is a separate, stronger requirement.

#### A surviving quorum does not ensure subscription recovery

Consider three members A/B/C. A client watches A at revision 10. A loses its peer links, while B/C retain quorum and commit a final matching update at 11. The client remains running and can reach B/C as well as A, but its existing stream to A remains transport-healthy. Without leader-required monitoring, that stream need not terminate or migrate. Adding endpoints does not itself reassign an already open stream. If the peer partition persists, the watched state can remain at 10 indefinitely, despite a reachable majority and a consumer capable of processing the update.

**High confidence in the source-derived mechanism; this exact multi-endpoint topology was not runtime-tested:** unmarked streams bypass leader-loss monitoring (`server/etcdserver/api/v3rpc/interceptor.go:239,315 @ v3.7.3`), registration is member-local (§1.3), and Go stream recreation follows receive errors (`client/v3/watch.go:665 @ v3.7.3`). No such error is required for this peer-only partition. This is a counterexample to treating quorum survival and client reachability as sufficient for default-watch convergence.

`WithRequireLeader` requests termination after locally observed leader loss; that is a conditional failure signal, not a guarantee of continued delivery or eventual recovery (§2.1). In the peer-only partition experiment the direct watch terminated, but that result establishes neither failover nor convergence. A direct leader-required watch also has silent-stall paths (§2.6). The proxy experiment (§C.1) establishes a separate signaling discrepancy during a finite observation window; it does not prove infinite silence or test every multi-endpoint configuration. Ended proxy broadcasts, blocked dispatch, and stopped client retries have their respective trigger qualifications (§§3–4, §6).

Every meaningful convergence guarantee needs environmental assumptions: the client must eventually communicate with a usable service, its processing must progress, and recovery must have retained history or a valid snapshot. A permanently disconnected or stopped client cannot be made current merely by keeping quorum. Conversely, failure of one member does not necessarily remove the client's ability to reach the majority, as the scenario above shows.

**Conclusion — High confidence in the logical conflict; Moderate in the assessment of documentation adequacy:** If permanent nonarrival is permitted in an execution covered by the eventual-consistency promise, the two claims contradict each other. If those executions are excluded as unhealthy, the promise needs explicit failure assumptions and recovery obligations. The defensible finding is an **incompletely specified end-to-end liveness contract**, with concrete silent-stall mechanisms—not evidence about the authors' understanding. Watches retain their value for efficient push delivery and replay, but the consumer cannot infer self-sufficient eventual convergence from an open subscription and a surviving quorum. Where failure notification is incomplete, independent progress supervision or reconciliation is needed (§7.2).

### 1.6 What the Jepsen 3.4.3 watch result establishes

**High confidence — historical report, not a current-version test:** [Jepsen's etcd 3.4.3 analysis (2020-01-30)](https://jepsen.io/analyses/etcd-3.4.3), §§2.5, 3.1, 4, and 4.2, compared single-key watch sequences across revision-based reconnects and found ordered, complete delivery while history remained available. It did not test multi-key watches or rigorously cover deletes. It also identified the undocumented special meaning of revision zero. These results do not prove general watch liveness or correctness of later versions.

The associated fixes were traced independently:

| Historical finding | Verified change and present behavior |
|---|---|
| [#11496](https://github.com/etcd-io/etcd/issues/11496): revision zero did not mean the beginning of history. | **High confidence — documentation fix:** [PR #11559](https://github.com/etcd-io/etcd/pull/11559), commit `342c2464aee1`, merged 2020-01-27, documents initial revision 1 and adds the zero/unspecified caveat to the Go Watch interface. The caveat remains at `client/v3/watch.go:68–69 @ v3.7.3`; `server/storage/mvcc/watchable_store.go:139–145 @ v3.7.3` still maps zero to local current revision plus one. This was not a change making zero replay all history. |
| Inaccurate consistency terminology. | **High confidence — documentation fix:** [PR #11474](https://github.com/etcd-io/etcd/pull/11474), commit `23810ea28548`, merged 2020-01-27, revises the guarantees description. Current [v3.7 documentation](https://etcd.io/docs/v3.7/learning/api_guarantees/#strict-serializability) specifies strict serializability for KV and separately excludes linearizability for Watch. Strict serializability means an atomic serial execution of transactions respecting real-time precedence; it does not describe asynchronous notification freshness. [Definition](https://jepsen.io/consistency/models/strong-serializable). |

**High confidence — scope of the inference:** The old zero-value surprise is different from the creation-header races in §3.1 and §6.2. The former concerns which revision the caller requested; the latter concern advancing a recovery cursor past undelivered events. Documenting zero does not repair those races. Similarly, single-key success cannot establish atomic delivery of multi-key lease-revocation events (§5.1), correct progress watermarks, or recovery after compaction and restart. Appendix B traces later defects and their shipped fixes rather than treating the earlier test result as permanent certification. The partial-failure liveness schedules in §2.6 remain independently source-derived and unexecuted.

## 2. Failure detection and its limits

Consider three members A/B/C and a watch connected to A. A becomes isolated from B/C but remains reachable by its client. B/C retain quorum and commit a change. A cannot apply that change, yet its client connection can remain alive.

**High confidence:** Without `WithRequireLeader`, a transport-healthy watch can remain attached to A with no event or failure notification. Ordinary TCP/HTTP/2 keepalive cannot discover that A has lost its peers. The option supplies a server-side reason to terminate that otherwise usable stream, so the consumer can attempt recovery. A surviving majority does not, by itself, move an existing watch there.

The behavior below concerns direct connections to a member. The standalone proxy has additional failures (§4.1); the option’s introduction and adoption history are retained in Appendix B.1.

### 2.1 What WithRequireLeader does

**Verified, High confidence:** `clientv3.WithRequireLeader(ctx)` adds outgoing gRPC metadata, `hasleader=true`, asking the serving member to reject or terminate the stream when it detects no leader. A follower may still serve the watch. The option adds neither a client watchdog nor a freshness deadline, and registration remains local.

**Source evidence:** the helper is at `client/v3/ctx.go:26 @ v3.7.3`; the metadata constants are at `api/v3rpc/rpctypes/md.go:18 @ v3.7.3`.

For a direct connection to an etcd member, the mechanism is:

1. **Admission:** The stream interceptor rejects a new marked stream if `s.Leader() == raft.None` (`server/etcdserver/api/v3rpc/interceptor.go:239 @ v3.7.3`). A follower that knows a leader is allowed to serve it.
2. **Ongoing monitoring:** The interceptor registers the stream with `monitorLeader`. Once per configured election interval, the monitor samples the member's locally stored leader ID. Three consecutive samples with no leader cancel marked streams with `ErrGRPCNoLeader` (`:315`; `maxNoLeaderCnt = 3` at `:38`). Thus it applies to established streams as well as new ones. The code deliberately waits longer for established streams because reconnecting them is expensive.
3. **Server response:** The watch handler observes the canceled context and returns its error (`server/etcdserver/api/v3rpc/watch.go:220 @ v3.7.3`). Delivery still requires the handler and transport to make progress.
4. **Go-client behavior:** Receipt of `ErrNoLeader` is terminal for the affected logical watches; it is not the recoverable-error branch that silently reconnects them (`client/v3/watch.go:665 @ v3.7.3`). The intended result is a canceled response followed by channel closure. The application must restart from its last processed revision or rebuild its state.

`s.Leader()` reads a local atomic field (`server/etcdserver/server.go:1705 @ v3.7.3`, `:1735`), updated from Raft's `Ready.SoftState` (`server/etcdserver/raft.go:185 @ v3.7.3`). The check is therefore **local knowledge of a leader**, not proof that this member has contacted a quorum now or applied all current commits. Three monitoring samples also do not give a three-second end-to-end guarantee: Raft must first notice the failure, the monitor must run, and the error must reach and be processed by the client.

### 2.2 Why server cancellation may not reach the application

The failure paths must be separated:

| Failure conditions | Expected effect of `WithRequireLeader` | Remaining client problem |
|---|---|---|
| A loses B/C; A keeps running; the client–A transport stays usable; the application drains responses | A detects no leader and terminates the stream. | The consumer still has to invalidate its confidence and resume explicitly. Failure detection is delayed. |
| A loses B/C and the client cannot receive A's terminal status or reach another usable endpoint | The server may cancel, but the client sees a transport failure instead of `ErrNoLeader`. | The Go client can retry indefinitely while the public watch channel remains open and silent. |
| A still reports a leader but its apply or delivery path is stalled | The no-leader predicate can remain false. | The option does not measure replication lag, delivery progress, or cache freshness. |
| A sends `ErrNoLeader`, but the public watch channel is full | The library receives the intended terminal status. | The separate error-delivery timeout can discard it; see §2.3. |

**Verified mechanism, High confidence:** The Go receive loop terminates on `ErrNoLeader`, but retries other recoverable errors. `openWatchClient` has no finite retry budget (`client/v3/watch.go:1039 @ v3.7.3`); default calls use `grpc.WaitForReady(true)` (`client/v3/options.go:24 @ v3.7.3`). The shared transport context preserves values but strips the original deadline (`client/v3/watch.go:274 @ v3.7.3`). Individual watch cancellation is separately honored; this does not mean user cancellation is universally ignored. It means the option itself supplies no deadline for recovery or failure notification.

[etcd #8980](https://github.com/etcd-io/etcd/issues/8980) documents this client-visible indefinite retry, including a 2020 report using `WithRequireLeader` and transport keepalive settings. [Kubernetes #89488's final analysis](https://github.com/kubernetes/kubernetes/issues/89488#issuecomment-606491110) likewise reports a watch stuck in transport failure when the client shares the disrupted failure domain. Both issues are closed; the current retry-policy TODO still cites #8980 at `client/v3/watch.go:90 @ v3.7.3`. Closure is not evidence that a bounded failure-notification policy was implemented.

**Limit of the evidence:** A client located beside the minority member should still receive `ErrNoLeader` if that member is running and their connection remains usable, even when the client cannot reach B or C. Explaining a failure with that local connection intact requires another fault or race. Reports of `TRANSIENT_FAILURE` establish a transport/retry problem; they do not show that established streams lack server monitoring.

The current leader- and follower-partition tests explicitly keep the client pinned to one member and assert `ErrNoLeader` (`tests/integration/clientv3/connectivity/network_partition_test.go:201 @ v3.7.3`, `:226`, `:249`, `:260`). They exercise the first row, with a consumer already waiting to receive. They do not establish the second or fourth row's client guarantees. I did not execute them.

**Assessment — High confidence in the implementation limit:** The implemented guarantee ends at the server’s attempt to terminate a marked stream after detecting leader loss. It does not ensure that the application receives a failure within a deadline. A local client deadline could bound how long the application trusts an unverified view, even though it could not distinguish a slow system perfectly from a failed one. `WithRequireLeader` supplies no such deadline or uncertainty state.

### 2.3 The final error can be dropped after 250 ms

**Verified source contradiction, High confidence; full partition reproduction not performed:** The public Go API promises a final response carrying failure before channel closure. `Watcher.Watch` specifically promises `ErrNoLeader` for the leader-required failure path (`client/v3/watch.go:78 @ v3.7.3`); `WatchResponse.Canceled` documents a final canceled response with non-nil `Err()` (`:108`). Yet the implementation allows that response to be dropped:

- Each application's output channel has capacity one (`client/v3/watch.go:558 @ v3.7.3`).
- Terminal stream cleanup calls `closeSubstream`, which starts `sendCloseSubstream` with the stored error (`:487`, `:494`).
- `sendCloseSubstream` selects between sending the error, cancellation of the application's context, and a timer. When the timer wins, it closes the channel without sending the error (`:478`).
- That timer is `closeSendErrTimeout = 250 * time.Millisecond` (`:39`). This is a source constant, not a measured latency.

A sufficient failure schedule is:

1. An established watch has an ordinary event or creation notification queued in its one-slot output channel. The application's context is still active, but its consumer is temporarily busy and does not drain the slot.
2. A loses its leader. The server and client transport correctly deliver `ErrNoLeader`; the Go watch receive loop begins terminal cleanup.
3. The application leaves the output slot full for more than 250 ms after the final-error sender starts. The sender times out and closes the channel without inserting the canceled/error response.
4. The application resumes, drains the already-buffered response, and sees channel closure. It never receives the promised `ErrNoLeader` or `Canceled=true` response.

A temporary consumer pause is sufficient for this schedule. It needs no compaction, permanent disconnection, or loss of the cluster majority. The public API does not say that a 250 ms pause forfeits error delivery, so the code contradicts its final-error promise independently of the absence of a general notification deadline.

**Verification limit:** No upstream issue or runtime reproduction of this exact schedule was found. The finding is derived from source; the inspected `main` and `release-3.7` versions of `client/v3/watch.go` have no change to this path.

**Consumer consequence:** Channel closure remains observable. A consumer must treat any unsolicited close while its own context is active as loss of the subscription, even without a preceding error. It should resume from its own last processed revision.

**High confidence — source-derived delivery boundary:** Terminal cleanup can also abandon internal buffered responses when `recvc` closes (`client/v3/watch.go:832 @ v3.7.3`). Library receipt therefore does not establish application delivery. The demonstrated problem is unreliable terminal signaling and possible loss of buffered delivery at closure; this schedule does not establish an interior event gap on a stream that stays open.

Kubernetes v1.37.2 has an additional defense against a bare close: after its range loop ends, it closes `watchClosedCh`, causing the outer watch to cancel and close its result channel (`staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:521 @ v1.37.2`, `:269`). Therefore this source finding does not, by itself, establish permanent Kubernetes cache corruption. It does refute the assumption in the nearby comment that reaching that point necessarily means the caller ended the watch, and it explains why consumers must handle closure independently of `Err()`.

Related [#12534](https://github.com/etcd-io/etcd/issues/12534) and [#17611](https://github.com/etcd-io/etcd/issues/17611) are closed reports of blocking despite the option. #17611 describes a different creation/cancellation teardown race using an old client and etcd 3.4.16. Neither reproduces the 250 ms error-loss path. The peer review prompted a fresh trace of that separate race in current code; see §3.3.

### 2.4 Why a healthy majority does not make a stale observer safe

**High confidence:** The majority can be healthy and committing while one observer is permanently unable to distinguish “nothing changed” from “my observation path stopped.” Describing this only as an unhealthy-cluster exception obscures the application consequence: the rest of the system can make decisions that invalidate the observer's cached authority.

[Kubernetes #59848](https://github.com/kubernetes/kubernetes/issues/59848), observed open during this follow-up, gives a concrete safety consequence of stale cached reads: a restarted kubelet can receive an obsolete pod assignment and start an old pod incarnation after another node has started its replacement. [etcd #8673](https://github.com/etcd-io/etcd/issues/8673) explicitly links a variant in which a quorum-derived snapshot at revision 10 is followed by changes at 11/12 and a watch reestablished against an isolated member still at 10. The prior successful snapshot does not certify the later watch's freshness. These are historical documented scenarios, not reproductions against today's Kubernetes release.

**Inference, High confidence:** Even when `WithRequireLeader` works exactly as intended, the interval before error delivery allows decisions based on stale state. After the error, the option does not revoke decisions already made, invalidate a downstream cache, stop a worker acting on an old assignment, or fence that worker's external writes. Reconnecting alone also cannot repair arbitrary already-corrupted application state. Those are consumer-protocol responsibilities that must be made explicit if the advertised higher-level guarantee requires them.

**High confidence — local-progress limitation and read-point reasoning:** Local progress notifications do not fix the uncertainty: `progressIfSync` certifies the receiving member's applied revision (`server/storage/mvcc/watchable_store.go:515 @ v3.7.3`). Repeated truthful progress at 10 does not establish that the majority has not reached 12. A consumer needing a current consistent view must establish a quorum-derived target revision and wait for its own processed state to reach it, with a local deadline that changes the cache to unavailable/uncertain on failure. This yields a view valid at a read point; it does not grant continuing ownership or prevent subsequent concurrent changes. Exclusive external actions need their own concurrency/fencing protocol.

### 2.5 Which promises these failures violate

The evidence supports four separate verdicts:

1. **High confidence:** A failed observation path can leave an application indefinitely uninformed despite a functioning quorum, including with `WithRequireLeader` on a direct connection (§2.6). The option supplies conditional leader-loss signaling; it does not ensure eventual delivery or recovery. This matters even when the application requires only eventual convergence, without fresh reads or safe external ownership.
2. **High confidence:** The narrower Go-client promise of a final failure response has a concrete contradiction in the current 250 ms drop path. This is a client API defect even without assuming a bounded-delay watch contract.
3. **High confidence, conditional on the stated failure assumptions:** Permanent nonconvergence violates eventual consistency even if every delivered event is correctly ordered and no interior event is missing. Lack of a deadline cannot excuse that liveness failure. The documentation does not adequately delimit the eventual-consistency claim against its unhealthy-cluster exception (§1.5).
4. **Not established by this partition argument alone:** That every configured watcher can become permanently stuck, that a finite silent interval proves permanent failure, or that the server emits a later event while omitting an earlier retained one. Nor can quorum survival guarantee delivery to a client permanently cut off from all usable members. A bounded “deliver or declare uncertainty” policy is an additional application requirement, distinct from eventual convergence.

The absence of a general delivery deadline excuses neither permanent nonconvergence within the promised conditions nor failure to deliver an explicitly promised terminal error. Reliable stored history, eventual observation, and safe use of current state are separate obligations.

### 2.6 Direct leader-required watches can remain silently stalled

**Correction, High confidence:** Describing a direct `WithRequireLeader` watch as handling member isolation overstates the result. The verified narrow behavior is cancellation after the serving member records leader loss, provided the monitor, handler, and client cleanup can run. Cancellation itself does not establish eventual consistency: the Go client makes `ErrNoLeader` terminal, leaving resumption to the application (`client/v3/watch.go:665 @ v3.7.3`). Other member failures can prevent even that cancellation.

#### Concrete counterexample: a storage-stalled member retains its leader ID

Consider A/B/C with leader B, a direct leader-required watch on follower A, and a functioning client able to reach all three endpoints. Assume no application deadline or external watchdog retires the watch.

1. A has recorded B as leader. `s.Leader()` reads an atomic field, not a live quorum check (`server/etcdserver/server.go:1705,1709,1735 @ v3.7.3`).
2. A's Raft processing goroutine blocks in synchronous WAL persistence. That same goroutine processes `Ready.SoftState` and updates the recorded leader ID before calling storage (`server/etcdserver/raft.go:185,198,256 @ v3.7.3`). The persistence path calls `WAL.Save`, then `sync`, then `Fdatasync` (`server/storage/storage.go:91 @ v3.7.3`; `server/storage/wal/wal.go:995,1029,881 @ v3.7.3`). A storage call that remains blocked does not return an error for the fatal-error branch to handle.
3. B/C continue committing subsequent updates without A. Because A's Raft processing goroutine cannot process subsequent ready batches, those later updates cannot reach its local watch history. Its stored leader ID can nevertheless remain B. This case does not require continuing heartbeats to A: the field itself can be stale.
4. A's independent gRPC and leader-monitor goroutines may still run. The monitor sees a nonzero ID and resets its failure counter; it does not inspect WAL completion, applied revision, or watch delivery (`server/etcdserver/api/v3rpc/interceptor.go:315,329 @ v3.7.3`). The transport can remain responsive, while the watch has no new matching event and no required failure notification.
5. The Go client has no receive error requiring stream recreation (`client/v3/watch.go:665 @ v3.7.3`). Other configured endpoints do not migrate this existing stream. The watch can remain stale for as long as the storage stall persists; if the stall persists indefinitely without process termination or external intervention, so can the silence.

**Evidence limit:** This is a source-derived counterexample under an explicitly assumed partial member failure, not a newly executed disk-failure test or an estimate of its frequency. The conditional code mechanism is **High confidence**; the unexecuted end-to-end failure schedule is **Moderate confidence**. Its occurrence rate and duration on a particular storage stack are unknown. An I/O error that returns, a process restart, a client deadline, or successful external failover changes the scenario. Quorum loss and permanent client disconnection are not prerequisites.

**High confidence in the separated code paths; Moderate in an unexecuted indefinite-stall scenario:** An apply-only stall has a related blind spot: apply jobs run on a separate FIFO scheduler (`server/etcdserver/server.go:843 @ v3.7.3`), while leader state is updated in the Raft loop. As long as the local leader predicate remains nonzero, `WithRequireLeader` does not test whether those jobs complete. A long scan or defrag demonstrates delay, not automatically permanent deadlock; claims of indefinite blocking need their own cause and reachability evidence (§4.4–§4.5).

#### Which other failures are actually silent?

| Failure path | What the consumer can observe | Why the flag is insufficient |
|---|---|---|
| **High in the predicate; Moderate in the untested partial-failure schedule:** stored leader ID remains nonzero while storage/apply/delivery stalls | Open stream without new matching events; transport may remain usable. | The leader predicate does not measure progress. Source-derived case above. |
| **Moderate — untested schedule:** rejected creation races with terminal `ErrNoLeader` | Client cleanup can deadlock before completing the expected watch lifecycle. | The flag does not repair client bookkeeping; current schedule remains untested (§3.3). |
| **Moderate — topology-dependent reachability:** cancellation blocks shared dispatch | Event and control delivery can stop on the affected stream. | A server-side leader signal cannot guarantee execution of blocked client recovery; deployment reachability remains qualified (§4.2). |
| **High for the reproduced flag failure; Moderate for the ended-broadcast schedule:** networked proxy ignores the flag, or retains an ended broadcast | Downstream stream can stay open without new events or terminal status. | Flag propagation and broadcast lifecycle are separate defects; the first has a limited partition reproduction (§4.1). |
| **Moderate — untested current trigger:** jetcd's scheduled resume throws | No new retry or listener notification for that scheduled failure; an earlier transport error may already have been reported. | `isClosed()` is not a progress indicator; current trigger needs reproduction (§6.3). |
| **High — source-derived termination behavior:** Go drops the final error after 250 ms, or normally terminates on `ErrNoLeader` | **Channel closure remains observable.** | These require consumer recovery but are not evidence of an open, undetectably dead watch (§2.3). |

**Observability conclusion — High confidence in the indistinguishability argument; Moderate in the untested watchdog integration:** The affected watch's callbacks, connection state, and local leader/progress reports do not reliably distinguish a quiet key range from a failed observation path. That is the missing built-in client signal. It does not make independent monitoring impossible: a deadline-bounded linearizable read through a usable endpoint can establish a revision the consumer has not reached. Waiting for watch coverage through that revision, then declaring uncertainty on timeout, detects lack of verified progress without identifying the internal cause (§7.2). This is extra client machinery, not a guarantee supplied by `WithRequireLeader`.

## 3. Go-client recovery and progress

Recovery depends on where the watch starts, which revision the client remembers, and whether progress or failure signals reach the application. The following findings concern the Go client; jetcd is examined in §6.

### 3.1 Starting and restarting a revision-zero watch

#### Local “now” and the creation header are different observations

**Verified, High confidence:** For `start_revision=0`, registration chooses the serving member's `currentRev + 1` while holding the watchable-store and revision locks (`server/storage/mvcc/watchable_store.go:139 @ v3.7.3`). The RPC handler then reads the member's revision **again**, after registration, for the `Created` header (`server/etcdserver/api/v3rpc/watch.go:340 @ v3.7.3`, `:358`). That read is synchronized internally; the problem is a non-atomic pair of observations, not an unsynchronized memory access. Concurrent applies can advance the revision between them.

The Go client binds an initially zero resume revision to the creation header **without adding one** (`client/v3/watch.go:838 @ v3.7.3`, `:858`). Thus there are two source-derived cases if a recoverable disconnect occurs after creation but before any event or progress response reaches the library:

| Registration and creation header | Transparent reconnect | Consequence |
|---|---|---|
| Member revision is R at both observations; original subscription starts at R+1 | Client requests inclusive revision R | A retained matching event at R, outside the original subscription, can be replayed. This alone is not a duplicate of an event already delivered to this logical watch. |
| Registration sees R; header sees H, with H ≥ R+2 | Client requests inclusive revision H | Matching events in [R+1, H−1], included in the original subscription but not received, are no longer requested. |

**Moderate confidence in the unexecuted failure schedule:** The second case requires concurrent matching writes and a carefully placed disconnect. It is a narrower concern than jetcd's historical-watch cursor jump (§6.2): the Go path affects the initial revision-zero watch, whereas an explicitly requested Go revision survives creation unchanged. If event responses reach the library first, their last modification revision replaces this cursor; if a valid progress response arrives, it establishes delivery through its watermark. Recoverable reconnection retains already-received buffered events.

The API does not promise a cluster-linearizable “now”; merely receiving older cluster history from a lagging member is not evidence of a violation. The second schedule nevertheless conflicts with continuity from the **member-selected subscription boundary**, and with the Go comment promising events after the server receives the request (`client/v3/watch.go:68 @ v3.7.3`). A deterministic test should establish the exact skipped interval before calling it a reproduced current bug. Neither changing every creation cursor to `H+1` nor treating `Created(H)` as a progress watermark repairs the mismatch.

#### Terminal restart discards the library's private starting point

**Verified, High confidence:** The library consumes the initial `Created` internally unless `WithCreatedNotify` was requested. On `ErrNoLeader`, it terminates the watch and constructs an error response with an empty header (`client/v3/watch.go:845 @ v3.7.3`, `:495`, `:667`). A revision-zero watch that has delivered no application response can therefore leave the application without the library's private resume point.

**High confidence — consequence of restarting from revision zero:** If the majority writes a matching key during the partition, then the application opens a new default watch after the terminal error, that new watch starts after the receiving member's current revision and can omit the partition's write. This is an application recovery trap, **not an undocumented refusal to perform transparent failover**: `ErrNoLeader` termination is explicitly documented at `client/v3/watch.go:78 @ v3.7.3`. The new request at zero also explicitly selects a new starting point.

For continuity, obtain a known snapshot/revision before watching and resume from the application's last complete processed revision. `WithCreatedNotify` exposes useful creation information, but the non-atomic boundary above means its header must not be represented as an exact registration watermark. If the application has no trustworthy starting cursor, a new snapshot restores present state; it cannot recover every intervening transition.

New marked streams also get no three-sample admission grace: they are rejected immediately when the member currently reports no leader (§2.1). A retry can therefore terminate during an ordinary election or when it selects another still-reachable leaderless member. **High confidence in the admission/termination logic; no claim about selection frequency:** Opening a transport stream is not confirmation that its server has accepted the leader requirement. This does not establish that `openWatchClient`'s retry loop is generally broken, nor that a multi-endpoint client remains permanently pinned to one member. The current partition test intentionally uses a single endpoint (`tests/integration/clientv3/connectivity/network_partition_test.go:226 @ v3.7.3`); it is not a test of successful reselection from a mixed set of healthy and partitioned endpoints.

### 3.2 Progress scope, routing, and response timing

**Verified, High confidence:** A progress watermark certifies delivered history only once it arrives. Requested and periodic progress have different scopes, and Go metadata also determines where an explicit request goes.

#### Explicit progress covers one stream

**High confidence — source-verified stream scope:** `RequestProgressAll` passes the requesting stream's `ws.watchers` map to `progressIfSync`. If any of those watches is unsynchronized or starts beyond the local revision, the function sends nothing and returns false. The RPC handler ignores the result rather than keeping a request pending until catch-up. A later response therefore requires another request or another notification mechanism.

**Source evidence:** `server/storage/mvcc/watcher.go:206 @ v3.7.3`; `server/storage/mvcc/watchable_store.go:515 @ v3.7.3`; `server/etcdserver/api/v3rpc/watch.go:395 @ v3.7.3`.

#### Periodic progress is checked per watch

**High confidence — source-verified per-watch scope and scheduling:** `progress(w)` passes a singleton map, so an unrelated future watch does not suppress its check. Periodic notification is opt-in, with a default base interval of ten minutes plus jitter of less than one minute. Recent event delivery can suppress a tick, and a failed synchronization check creates no immediate retry. Ten minutes is therefore not an upper bound on silence.

**Source evidence:** `server/storage/mvcc/watchable_store.go:507 @ v3.7.3`; `server/etcdserver/api/v3rpc/watch.go:94 @ v3.7.3`, `:104`, `:503`, `:558`. These are source constants and control paths, not measured latency.

#### Go metadata routes explicit requests

**High confidence — source-verified metadata routing:** Watch creation and `RequestProgress` both select a stream using `streamKeyFromCtx`. A plain-context progress request does not reach a watch created with `WithRequireLeader`. It can instead create a separate empty stream and return nil after queuing the request locally. That return value acknowledges neither server progress nor application freshness.

**Source evidence:** `client/v3/watch.go:330 @ v3.7.3`, `:413`, `:432`, `:1086`. [#12254](https://github.com/etcd-io/etcd/issues/12254) documents this usage trap; Kubernetes deliberately matches the metadata (§5.2).

#### Observation and verification limits

**Independently reproduced, High confidence:** On a single v3.7.3 member, a quiet watch initially received requested progress. Adding a future-revision watch on the same stream suppressed subsequent explicit progress responses. A second client stream still received progress, and both ordinary key watches still received a subsequent put (§C.2). This directly distinguishes stream-local progress coupling from member-wide suppression and from stalled event delivery.

[#20242](https://github.com/etcd-io/etcd/issues/20242), now closed, proposed individual responses when a stream-wide response cannot be sent. The release still uses the all-or-nothing check. Its safety purpose is sound: a progress response must not falsely certify a watch that has not caught up. The cost is response predictability and isolation among multiplexed watches. A future watch can prevent an explicit response indefinitely, even with no load or leader failure.

The scope follows the caller-supplied watcher map. `progressIfSync` checks each supplied watcher against the member’s synchronized group; it does not scan all member watchers, reject every request whenever `s.unsynced` is nonempty, or impose a global victim-list check. Shared locks can still cause member-wide contention (§4.4–§4.5). Appendix D records the rejected claim of member-wide progress suppression.

The relevant testing change was narrower than removal of progress validation. [PR #20240](https://github.com/etcd-io/etcd/pull/20240), commit `b51415c3f603eb6d7bd60ebbcdeb7e364fdcefed`, removed a robustness assertion that at least one progress response must arrive. It did **not** remove correctness validation for progress responses that do arrive: `validateBookmarkable` remains at `tests/robustness/validate/watch.go:111 @ v3.7.3`. This is evidence of a liveness-testing limitation, not abandonment of bookmark safety checks.

### 3.3 Rejected creation can strand client cleanup

**Verified source mechanism, High confidence; current runtime reachability, Moderate confidence:** The schedule reported in closed [#17611](https://github.com/etcd-io/etcd/issues/17611) still has a plausible path in v3.7.3. Its proposed fix [PR #17622](https://github.com/etcd-io/etcd/pull/17622) is closed **without merging**; its discussion repeatedly requested a deterministic test. The original report used a 2020 Go client and etcd 3.4.16, so its deployment evidence must not be relabeled as a v3.7.3 reproduction.

Current code permits this sequence:

1. A creation response is rejected: `WatchId == InvalidWatchID`, or `Canceled` with a nonempty reason. `addSubstream` closes `ws.recvc` and returns without adding the watcher to `w.substreams` (`client/v3/watch.go:466 @ v3.7.3`).
2. The creation-handling branch removes its entry from `w.resuming` (`:601`, `:607`). The watch is now absent from both collections.
3. `serveSubstream` exits on the closed receive channel, closes `donec`, and blocks sending itself to `w.closingc` before reaching `w.wg.Done()` (`:797`, `:803`, `:832`).
4. Before `run` receives that closing notification, it selects a terminal error, such as `ErrNoLeader`, and enters teardown. The teardown builds its closing set from tracked/resuming watches, drains that many notifications, and then waits for the wait group (`:520`, `:535`, `:539`). In the minimal case with only this untracked rejected watcher, the set is empty: no one drains its `closingc` send, and `wg.Wait` cannot finish.

This is a potential **client cleanup deadlock**, not the server cancellation/flow-control cycle in §4.2 and not the final-error timeout in §2.3. A rejected creation alone is insufficient: normally `run` receives `closingc` and cleans it up. The adverse ordering requires a subsequent terminal failure to win first. Authentication rejection supplies a possible rejected response; the specific server/transport interleaving still needs an injected or controlled reproduction. The relevant code has no corrective change in the inspected upstream comparison refs.

## 4. Server, proxy, and cache findings

These findings apply to different layers and must not be treated as one failover bug. The proxy paths require the optional gRPC proxy; the experimental cache is separate from Kubernetes's watch cache. Each subsection distinguishes the code mechanism, the reported or inferred trigger, and the limit of the evidence. Issue states are those recorded during the audit (Appendix A).

| Area | Main risk | Detail |
|---|---|---|
| gRPC proxy | Silence after upstream termination, ignored leader requirement, unsupported requested progress | §4.1 |
| Cancellation and watch IDs | Shared-stream blocking or events delivered to a different watch | §4.2–§4.3 |
| Watch-store synchronization | Lock inversion and catch-up work that can obstruct apply | §4.4–§4.5 |
| Compaction cleanup and experimental cache | Retained watcher state or failure to regain readiness | §4.6–§4.7 |
| Previous values | An event arrives without data the consumer requires | §4.8 |

### 4.1 gRPC proxy: three separate failure paths

#### 4.1.1 Ended upstream broadcasts (#21813)

**Issue state:** [#21813](https://github.com/etcd-io/etcd/issues/21813) is open; its reported deployment used v3.5.4. This concerns the optional **gRPC proxy**, which combines downstream watches into shared upstream subscriptions, rather than a direct connection to an etcd member.

**Verified mechanism, High confidence:** In `server/proxy/grpcproxy/watch_broadcast.go:55 @ v3.7.3`, the upstream receive goroutine exits when its Go watch channel closes and closes `donec`. It does not remove the broadcast from its parent's registry or install a stopped-state check. `add`, at `:97`, checks revision eligibility but not `donec`; it can even generate a synthetic creation acknowledgment. Coalescing at `server/proxy/grpcproxy/watch_broadcasts.go:53 @ v3.7.3` similarly checks revision and response counts without checking whether the recipient broadcast is alive.

**Failure schedule, Moderate confidence pending reproduction:** An upstream watch terminates while its broadcast remains registered. Before it is removed through downstream cleanup, an eligible new watch joins it, or another active watch is moved to it. The new receiver gets no subsequent changes because there is no upstream receive goroutine. Its connection to the proxy may still be healthy. The missing prerequisite is not another election; it is restoration of the subscription or explicit downstream failure.

**High confidence in the response filter; Moderate in an untested terminal-error propagation failure:** There is an additional source-level concern: `server/proxy/grpcproxy/watcher.go:107 @ v3.7.3` filters empty non-created, non-progress responses when `CompactRevision == 0`, without exempting other canceled responses. This makes propagation of terminal failures another necessary reproduction target.

**Qualification:** A routine recoverable transport break normally triggers Go-client retry; it does not necessarily close the upstream watch channel. Therefore the issue's generic “restart or partition” recipe is not enough to conclude that every such failure leaves a dead broadcast. The concrete condition is **terminal upstream closure plus a still-eligible registered broadcast**. Direct member watches do not traverse this code. No corrective liveness check was present in the inspected upstream branches.

#### 4.1.2 Ignored leader requirement

**Verified in source and independently reproduced, High confidence:** The proxy Watch handler calls `metadata.FromOutgoingContext(stream.Context())` to look for `hasleader=true` (`server/proxy/grpcproxy/watch.go:112 @ v3.7.3`). A networked gRPC server receives client metadata in the **incoming** context. The ordinary etcd interceptor and the proxy's own auth-token extraction use the incoming context (`server/etcdserver/api/v3rpc/interceptor.go:228 @ v3.7.3`; `server/proxy/grpcproxy/util.go:26 @ v3.7.3`). The standalone proxy installs metrics and optional logging stream interceptors, with no conversion of that metadata into outgoing context (`server/etcdmain/grpc_proxy.go:507 @ v3.7.3`).

Consequently, the proxy leaves `lostLeaderC` nil; its leader-loss branch never becomes ready for this downstream watch (`server/proxy/grpcproxy/watch.go:111 @ v3.7.3`, `:139`). Its upstream data subscription also does not inherit the requirement: it derives its context from the proxy context and copies client authentication separately (`server/proxy/grpcproxy/watch_broadcast.go:46 @ v3.7.3`, `:58`, `:66`). It can therefore stay attached to a reachable member that has lost its peers.

The experiment in §C.1 used three v3.7.3 members, a standalone proxy with **one backend endpoint**, and direct/proxied `etcdctl watch foo` clients. Both clients set `WithRequireLeader` (`etcdctl/ctlv3/command/watch_command.go:163 @ v3.7.3`). With the initial leader isolated from its peers but reachable by both clients, the majority committed `foo=v2`. The direct watcher exited with `ErrNoLeader`; the proxied watcher remained open and did not deliver v2 during the observation window. After healing, it delivered v2 and then v3. This establishes a failure of the advertised leader-loss notification through this proxy topology; it does **not** reproduce #21813's dead-broadcast mechanism or permanent history loss.

The bad context lookup is present in historical commit `5e059fd8dca7907c034d1d4b2458e755548a4a5e`, which converted the old metadata API in 2017, and remains unchanged in the inspected stable and upstream refs. The proxy integration framework uses in-process adapters (`tests/framework/integration/cluster_proxy.go:68 @ v3.7.3`, `:80`); their server contexts inherit the client context directly (`server/proxy/grpcproxy/adapter/chan_stream.go:142 @ v3.7.3`). **Inference, High confidence:** That adapter can preserve outgoing metadata and mask this network-context distinction. Passing adapter tests does not validate the standalone path.

**Separate design concern, Moderate confidence; not tested:** Even after correcting the context lookup, the proxy's leader detector is a separate watch on `__lostleader`, carrying `WithRequireLeader`, at a very high future revision (`server/proxy/grpcproxy/leader.go:42 @ v3.7.3`, `:60`). Data broadcasts lack that metadata and use a different client watch stream. With multiple backend endpoints, the detector and data stream can select different members; health of the detector's member would not prove health of the member serving events. The single-backend experiment neither establishes nor rules out that additional schedule. Merely fixing the metadata lookup is therefore insufficient evidence of complete multi-endpoint watch-health semantics.

#### 4.1.3 Unsupported explicit progress requests

**Verified, High confidence:** The proxy receive loop handles create and cancel requests but has no `WatchRequest_ProgressRequest` case. It logs an unsupported request and continues (`server/proxy/grpcproxy/watch.go:290 @ v3.7.3`). Closed [#19710](https://github.com/etcd-io/etcd/issues/19710) reported this with etcd 3.5.21 and Kubernetes 1.32; closure did not add support to v3.7.3 or the inspected upstream refs.

This is distinct from direct-member progress suppression in §3.2. A freshness protocol that repeatedly calls `RequestProgress` through the proxy cannot rely on that request reaching the backend, even if its Go call returns nil. Periodic upstream progress is separately requested by broadcasts (`server/proxy/grpcproxy/watch_broadcast.go:60 @ v3.7.3`), so “the proxy has no progress notifications at all” would be too strong. This missing request capability was source-verified; it was not separately exercised in the partition experiment.

### 4.2 Cancellation blocking (#18879, #20716)

**Issue state:** [#18879](https://github.com/etcd-io/etcd/issues/18879) is open, originally reported with v3.5.11 and a Kubernetes ConfigMap-watch reproduction. The reporter confirmed the Kubernetes watch cache was disabled. Related [#20716](https://github.com/etcd-io/etcd/issues/20716), about an in-process watch used by election campaigning, is closed as not planned; that status is not evidence of a fix. [PR #21065](https://github.com/etcd-io/etcd/pull/21065), proposing a cancellation-deadlock fix, remains open in the issue-search result.

**Verified mechanism, High confidence; deployment reachability, Moderate confidence:** The client sends cancellation synchronously from the same loop that dispatches received responses (`client/v3/watch.go:685 @ v3.7.3`). The server's receive loop queues cancellation acknowledgments into a bounded control channel (`server/etcdserver/api/v3rpc/watch.go:122 @ v3.7.3`, `:375`). Its send loop can block sending a response (`:519`). The in-process adapter implements sends using channels (`server/proxy/grpcproxy/adapter/chan_stream.go:107 @ v3.7.3`).

A cancellation burst can form a cycle:

1. The client waits for the server to accept more cancels.
2. The server receive loop waits for space in the control buffer.
3. The server send loop waits for the client to consume responses.
4. The client cannot consume responses because its dispatch loop is still sending cancels.

The inspected code retains these dependencies.

**Client consequence — Moderate confidence, conditional on reaching the blocking cycle:** Unrelated watches multiplexed onto the affected stream can stop seeing changes even after the cancellation burst ends. Application election failover can then fail because a waiter never observes its predecessor's deletion. This does not imply that the etcd cluster failed to elect its own Raft leader.

The in-process path has the strongest upstream evidence. The precise behavior of a normal networked Kubernetes deployment depends on transport buffering and flow control; the original report's description of embedded etcd was itself questioned upstream. I do not generalize its reproduction to every Kubernetes topology or claim to have reproduced it on v3.7.3.

### 4.3 Events delivered to a reused watch ID (#22673)

**Issue state:** [#22673](https://github.com/etcd-io/etcd/issues/22673) is open. Its author reports unit and end-to-end reproductions on `main`; I checked the release code independently.

**Verified mechanism, High confidence:** `sendLoop` uses separate channels for control acknowledgments and MVCC events. A cancellation acknowledgment deletes an ID from `ids`, but an event encountered later for an absent ID is interpreted as waiting for creation and stored in `pending`. Creating a watch with that ID flushes `pending` without checking the event's original watch incarnation. Evidence: `server/etcdserver/api/v3rpc/watch.go:411 @ v3.7.3`, `:470`, `:534`, and `:538`.

The concrete schedule is: create ID 5 for key A; queue an A event; process the cancel acknowledgment first; classify the queued event as pending; reuse ID 5 for key B; flush the old event to the new watch. Reuse after cancellation is permitted by the protocol's restriction against IDs **currently in use**, and the MVCC duplicate-ID check is against live watchers (`api/etcdserverpb/rpc.proto:809 @ v3.7.3`; `server/storage/mvcc/watcher.go:128 @ v3.7.3`).

**Assessment — High confidence in the violation given the described event/ID ordering; Moderate in untested v3.7.3 runtime reachability:** This is a safety violation, not merely delayed notification: the receiver can get an event outside its requested range or starting revision. Without ID reuse, the same classification retains otherwise dead event data until the whole stream closes. The ordinary Go client requests automatic IDs (`client/v3/watch.go:1067 @ v3.7.3`), so its exposure is primarily retention, while raw clients using explicit IDs expose the cross-watch delivery problem. No leader failure is necessary.

### 4.4 Lock inversion with an unconfirmed production trigger (#22184)

**Issue state:** [#22184](https://github.com/etcd-io/etcd/issues/22184) is open. It supplies a constructed three-goroutine test and attributes production reachability to concurrent snapshot generation.

**Verified, High confidence:** `progressIfSync` takes the watchable-store lock and then reads the revision through a read transaction, acquiring the underlying store lock. A write transaction holds the store read lock and later needs the watchable-store write lock. A third goroutine waiting for the store write lock can close the cycle because Go's `RWMutex` blocks new readers behind a waiting writer. Sources: `server/storage/mvcc/watchable_store.go:515 @ v3.7.3`; `server/storage/mvcc/kv_view.go:32 @ v3.7.3`; `server/storage/mvcc/kvstore_txn.go:46 @ v3.7.3`, `:174`; `server/storage/mvcc/watchable_store_txn.go:43 @ v3.7.3`; `server/storage/mvcc/kvstore.go:296 @ v3.7.3`.

**Correction — High confidence in the inspected scheduling and call path:** The cited `createMergedSnapshotMessage` call is inside `applyAll`, after applying entries, at `server/etcdserver/server.go:972 @ v3.7.3` and `:989`. `applyAll` jobs are submitted to the FIFO apply scheduler at `:844`. `createMergedSnapshotMessage` calls `KV().Commit()` synchronously at `server/etcdserver/snapshot_merge.go:38 @ v3.7.3`, before sending the snapshot. This does not establish a separate concurrent snapshot goroutine performing that commit while the same apply scheduler has an unfinished write transaction.

**Assessment — High confidence in the lock-order inversion; Moderate in the conditional production risk:** The lock-order defect is real at the storage API level, and the supplied artificial schedule is consistent with the code. **A production deadlock caused by the specific snapshot path alleged in the issue remains unconfirmed.** Another concurrent store write-lock caller would need to be demonstrated. If reached, the result is stalled application of writes and stalled watches; this report does not count it as a demonstrated leader-failover failure.

The future-revision fix [PR #20229](https://github.com/etcd-io/etcd/pull/20229) did not introduce the inverted lock order. Its commit `973847aa246b1255df648643f900958035f89888` moved `s.rev()` earlier within `progressIfSync`, but the preceding code already called it while holding the watchable-store read lock. The change can alter which paths acquire the second lock.

### 4.5 Catch-up scanning and buffering (#22264, #16839)

[#22264](https://github.com/etcd-io/etcd/issues/22264) and [#16839](https://github.com/etcd-io/etcd/issues/16839) remain open. The former reports a lagging Kubernetes API-server watch; the latter tracks starvation and memory pressure. The later clarification in #16839 retracts an initial attribution to previous-value lookup alone and instead identifies inefficient repeated history scanning under backpressure.

**Verified mechanism, High confidence:** `syncWatchers` holds the watchable-store mutex while `rangeEvents` scans from the minimum requested revision through the current revision. The backend range uses an unlimited result limit. Only afterward does event batching restrict delivery to 1,000 distinct matching revisions per watcher. The next pass can reread a large remaining interval. Sources: `server/storage/mvcc/watchable_store.go:346 @ v3.7.3`, `:363`, `:417`, `:430`; `server/storage/mvcc/watcher_group.go:25 @ v3.7.3`, `:39`. Write-transaction completion needs the same watchable-store mutex (`server/storage/mvcc/watchable_store_txn.go:43 @ v3.7.3`).

**High confidence — source-verified buffering behavior:** The Go client also appends received responses to a per-watcher buffer without a hard bound; its source retains a TODO to pause when the buffer becomes too large (`client/v3/watch.go:879 @ v3.7.3`). Server catch-up pressure and client buffering are different accumulation points.

**Qualification, High confidence:** Sharing a Go gRPC stream does not mean that every slow application consumer immediately blocks every other watch. `serveSubstream` can keep receiving into its private buffer while its application output is blocked (`client/v3/watch.go:818 @ v3.7.3`, `:832`, `:879`). That trades direct backpressure for memory growth. Shared dispatch, transport, cancellation, progress-request coupling, and resource exhaustion can still affect other watches; those are the mechanisms to test. “One slow consumer always stalls the entire stream” is not supported by this buffering design.

**Inference, High confidence about the coupling, Moderate confidence about operational severity:** Recovery after an outage can increase lag, scanning, allocation, and contention precisely when the consumer needs to catch up. Compaction can then force a full state rebuild. This is an availability and resource-isolation weakness; the inspected scanning code does not itself establish silent interior event loss. No benchmark numbers from the reports are adopted as measurements of this host or of v3.7.3.

### 4.6 Compaction cancellation can leave watcher state behind (#22294)

[#22294](https://github.com/etcd-io/etcd/issues/22294) is open.

**Verified, High confidence:** When the unsynchronized group contains at least `maxWatchersPerSync` watchers—512 by default—`choose` copies a selected subset into a temporary group. `chooseAll` can send a compaction response and remove a watcher from that copy, leaving the original unsynchronized group unchanged. The subsequent sync loop only visits watchers remaining in the returned group. Evidence: `server/storage/mvcc/watcher_group.go:226 @ v3.7.3`, `:255`; `server/storage/mvcc/watchable_store.go:40 @ v3.7.3`, `:363`.

**High confidence in the collection mismatch; Moderate in the unexecuted repeated-response schedule:** This permits repeated cancellation responses and retained state until cleanup, rather than silently delivering a complete but incorrect event sequence. A client cancel can remove the original entry (`server/storage/mvcc/watchable_store.go:161 @ v3.7.3`). Snapshot installation can move many watches into the unsynchronized group, but **a leader election alone is not the same operation as snapshot installation**. The issue's grouping of those situations should not be read as proof that every election triggers this condition.

### 4.7 Experimental cache fails to regain readiness (#22614)

[#22614](https://github.com/etcd-io/etcd/issues/22614) and its proposed fix [PR #22619](https://github.com/etcd-io/etcd/pull/22619) are open. This is the etcd repository's **experimental `cache/` library**, not Kubernetes's API-server watch cache.

**Verified mechanism, High confidence:** `Cache.watch` registers the new internal store watcher before the first response calls `demux.Init`. After a discontinuity, initialization can purge registered watchers, including the just-created one. Readiness is set through a `sync.Once`; an ensuing internal retry can remain unready. Sources: `cache/cache.go:348 @ v3.7.3`, `:388`; `cache/demux.go:119 @ v3.7.3`.

**High confidence in the upstream report; Moderate in untested reachability at the pinned release:** The upstream reproduction reports a successful relist and new watch followed by `Ready() == false` and blocking `WaitReady`/`Watch`. This is an exposed failure to recover readiness, rather than proof that a ready cache silently omits changes. The proposed initialization-order fix is not in the examined stable or upstream tips.

### 4.8 Missing previous values (#10681)

[#10681](https://github.com/etcd-io/etcd/issues/10681) is open and directly links Kubernetes [#76624](https://github.com/kubernetes/kubernetes/issues/76624).

**Verified, High confidence:** Previous values are obtained later, in the gRPC send path, by reading revision `event.ModRevision - 1`. A failed historical read leaves `PrevKv` unset. Evidence: `server/etcdserver/api/v3rpc/watch.go:450 @ v3.7.3`. Compaction can therefore intervene after a delete is applied and before the previous object is fetched for delivery. The protobuf explicitly permits omission after compaction (`api/etcdserverpb/rpc.proto:805 @ v3.7.3`).

The server can meet its contract while the consumer lacks the previous object needed to update its cache. If the consumer skips that event but advances its revision, it can retain a deleted object indefinitely. Rejecting the stream and rebuilding avoids that form of silent corruption. Section 5 distinguishes Kubernetes’s historical failure from its inspected error handling.

### 4.9 Other reports and coverage limits

The inventory also included [#20349](https://github.com/etcd-io/etcd/issues/20349), [#20622](https://github.com/etcd-io/etcd/issues/20622), and [#15820](https://github.com/etcd-io/etcd/issues/15820), which concern robustness coverage and the Kubernetes contract rather than newly established defects. #20349 specifically calls out incomplete coverage across future, retained, and compacted revisions.

Other results include [#22296](https://github.com/etcd-io/etcd/issues/22296), reporting a watch-resync ticker panic on coarse clocks; [#22225](https://github.com/etcd-io/etcd/issues/22225), concerning benchmark latency accounting; [#22266](https://github.com/etcd-io/etcd/issues/22266), a flaky watch test; and [#19597](https://github.com/etcd-io/etcd/issues/19597), a lock waiter that does not stop promptly when its own session expires. These were screened as adjacent issues, not independently reproduced or promoted into additional failover-corruption findings. CLI features, filtering proposals, general server stalls, and broad authorization questions are outside this report's detailed audit.

## 5. Kubernetes: impact and defenses

**High confidence in the described architecture; Moderate in the conditional propagation of a particular failure:** Kubernetes has two additional layers between etcd's event stream and most controllers: the API server's **watch cache**, a cached resource view populated from storage, and client **informers**, which maintain further local views. Corruption or a stall at an earlier layer can propagate to many otherwise functioning consumers.

### 5.1 Incident reports and current error handling

| Primary source | Reported observation (High confidence in what the report documents) | Limit on the conclusion |
|---|---|---|
| [#89488](https://github.com/kubernetes/kubernetes/issues/89488), closed; linked change [#89881](https://github.com/kubernetes/kubernetes/pull/89881) | Identified the need to use `WithRequireLeader` for storage watches that could otherwise remain attached to partitioned members. | This is a failure-detection issue, not proof of lost committed mutations. |
| [#76624](https://github.com/kubernetes/kubernetes/issues/76624), closed | Reported delete events with no object, leaving deleted pods in caches and impairing scheduling. The report associated occurrences with controller-manager leader changes and high churn. | A controller-manager election is not an etcd Raft election. Its timing does not establish the proposed Raft cause. |
| [#123072](https://github.com/kubernetes/kubernetes/issues/123072), closed as not planned | Reported caches that received recent objects while retaining some long-deleted pods; the problem propagated to informer clients. | This is stronger evidence than a wholly stalled stream, but the report alone does not isolate which layer omitted a delete or establish a v3.7.3 reproduction. |
| [#123448](https://github.com/kubernetes/kubernetes/issues/123448), closed in June 2026 with follow-ups | Maintainer investigation reproduced control-plane damage from slow direct watches with empty `resourceVersion`, including memory growth and stalled resource watches. | Closing the umbrella issue did not mean every etcd starvation mechanism was fixed; it explicitly delegates remaining work. |

The June 2026 closing comment on #123448 points to [#130939](https://github.com/kubernetes/kubernetes/issues/130939), removing previous-value dependence when the watch cache is used, and [#138264](https://github.com/kubernetes/kubernetes/issues/138264), aligning storage metrics. Those are follow-up references, not claims here that their work has shipped.

**Current-source verification, High confidence:** In Kubernetes v1.37.2:

- Storage watches are wrapped in `WithRequireLeader`, and progress requests use matching context metadata: [staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go:102 @ v1.37.2](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go#L102) and [:1153](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go#L1153).
- The direct storage watch starts at `initialRev + 1`, requests previous values, and optionally requests progress. It forwards watch errors and stops the receive loop: [staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go:470 @ v1.37.2](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go#L470).
- A non-create event without `PrevKv` is rejected: [staging/src/k8s.io/apiserver/pkg/storage/etcd3/event.go:64 @ v1.37.2](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/etcd3/event.go#L64). The caller sends an error and stops at [watcher.go:508 @ v1.37.2](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher.go#L508). Thus the inspected code does not simply ignore the missing object and continue advancing the stream as in the historical failure report.
- The cache's reflector runs ListAndWatch and reinitializes after errors: [staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go:503 @ v1.37.2](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go#L503). A correct explicit failure can trigger recovery; a watch that remains open forever without an error can prevent that recovery path from being invoked.

**Assessment, High confidence:** Kubernetes supplies direct evidence that the consumer-visible failure matters: deleted objects can persist, schedulers can stop responding to changes, and a cache that advances for other objects can remain wrong about one key. It also supplies evidence against the broad causal claim: the documented remedies concern error propagation, progress ordering, buffering, and reconstruction. No cited Kubernetes report proves that ordinary Raft leader election necessarily loses watch events because watch registration is not replicated.

### 5.2 Freshness checks for cache reads

**Verified in the selected Kubernetes v1.37.2 source, High confidence:** The relevant mechanisms are more specific than “Kubernetes watches are protected”:

1. The storage layer obtains a current resource revision through a KV read at [etcd3/store.go:728 @ v1.37.2](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go#L728). The inspected etcd Kubernetes helper uses a default linearizable Get, without `WithSerializable` (`client/v3/kubernetes/client.go:47 @ etcd v3.7.3`). This establishes a read-point target; a local progress response by itself does not.
2. While readers wait for sufficient cache freshness, the conditional requester sends progress requests with a 100 ms timer. It only runs those requests while waiters exist: [cacher/progress/watch_progress.go:36 @ v1.37.2](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/cacher/progress/watch_progress.go#L36), [:83](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/cacher/progress/watch_progress.go#L83). The storage wrapper deliberately reproduces the watch's context metadata: [etcd3/store.go:102](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go#L102).
3. Cache freshness waiting uses a three-second timeout, after which an insufficient revision produces an error. The consistent-list delegator either falls back to storage or, with `ConsistentListFromCacheSkipTimeoutFallback` enabled, returns a throttling error. Sources: [cacher/watch_cache.go:49 @ v1.37.2](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go#L49), [:318](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go#L318), [cacher/delegator.go:160](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/cacher/delegator.go#L160). This is a configured cache-wait interval, not a hard bound on total request latency, storage fallback, or scheduler delays.
4. A requested resource version of zero bypasses that freshness wait and allows an arbitrarily stale cache result: [cacher/watch_cache.go:319 @ v1.37.2](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go#L319), [:333](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go#L333). The consistent-read defense must not be generalized to this mode or to uninterrupted informer event delivery.

**High confidence in the cited configuration and its scope:** Kubernetes also configures Go gRPC keepalive at 30 seconds with a ten-second timeout: [storagebackend/factory/etcd3.go:63 @ v1.37.2](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/storagebackend/factory/etcd3.go#L63), [:344](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/storagebackend/factory/etcd3.go#L344). These are configuration values, not a measured or guaranteed forty-second recovery bound. A paused process cannot execute the server's leader monitor; client transport detection therefore matters. Transport keepalive still cannot certify apply progress or quorum freshness, and it does not turn an indefinitely retrying Go watch into a terminal error.

**Inference, High confidence:** This design separates correctness of a particular cache-served read from availability of the long-lived event stream. It depends on truthful progress, correctly processed events, compatible context metadata, and an endpoint that supports explicit progress requests. The proxy limitation in §4.1 is therefore materially relevant even though direct-member safety fixes are present.

## 6. jetcd: recovery and client-library readiness

The creation cursor is the main continuity concern. Retry lifecycle, API coverage, and test configuration introduce separate questions; each is assessed below.

### 6.1 Error handling and the limits of the failover report

[jetcd #1352](https://github.com/etcd-io/jetcd/issues/1352) is open and explicitly reports failure to reconnect after leader loss or quorum recovery. Its text claims jetcd 0.7.5, while its pasted trace names `jetcd-core-0.5.7.jar`. It also recursively opens a new watch from the error listener. Those details prevent treating it as a clean reproduction against 0.8.7.

**Verified, High confidence:** Current jetcd already schedules a retry for recoverable errors, but reports the error to the listener first. `ErrNoLeader` is excluded from automatic rescheduling. Compaction cancellation is terminal. Evidence: `jetcd-core/src/main/java/io/etcd/jetcd/impl/WatchImpl.java:272 @ jetcd-0.8.7`, `:323`, `:334`, and `:353`.

**High confidence in the retry interaction; Moderate in the application-policy recommendation:** Consequently, application retry code must account for the library's own retries and preserve a revision. A fresh default watch created on each error does not establish continuity across the failed interval. I have not confirmed that #1352's underlying connection failure persists in 0.8.7.

### 6.2 Creation can advance the cursor before replay

**Verified code, High confidence; skipped-event schedule, Moderate confidence pending runtime reproduction:** `WatcherImpl` initializes its revision from the requested option, uses that field as the next `start_revision`, and unconditionally raises it to `max(revision, creationHeader.revision)` on every creation response. Sources: `jetcd-core/src/main/java/io/etcd/jetcd/impl/WatchImpl.java:141 @ jetcd-0.8.7`, `:164`, and `:256`. Actual event responses update it to the last event revision plus one at `:313`.

A concrete schedule follows:

1. The application requests an existing history interval beginning at revision 101. The member's current revision is 200. Matching events at 110 and 150 remain retained.
2. The server creates that historical watch and sends `Created` with header revision 200. This response acknowledges registration, not completion of replay (`server/etcdserver/api/v3rpc/watch.go:340 @ v3.7.3`, `:358`).
3. jetcd changes its internal resume revision from 101 to 200.
4. A recoverable connection failure occurs before either historical event reaches `onNext`.
5. jetcd resumes automatically from 200. Events 110 and 150 are outside its new request, despite never having been delivered and still being available.

The same condition can arise during recovery of an established watch: it resumes from its genuine last event, receives a new creation acknowledgment from a member further ahead, then disconnects again before replay arrives. The server can obey each individual request while the client library loses the logical subscription's continuity. A transport error callback may occur, but it does not identify the omitted interval.

**High confidence — source-derived client distinction:** The Go client's explicit-revision path does not make this same creation-response jump (`client/v3/watch.go:838 @ v3.7.3`). jetcd's historical [#946](https://github.com/etcd-io/jetcd/issues/946), now closed, concerned the opposite direction—lowering a requested future revision. The current `max` protects that lower bound but is not evidence that raising an undelivered replay cursor is safe.

`WatchImpl.java` is unchanged between the analyzed jetcd release and inspected `main`. This schedule needs a deterministic test that cuts the stream after `Created` and before the first replayed response. No issue was filed and no claim of an executed reproduction is made here.

**Failure-window limit — High confidence in the cursor assignment:** The first nonempty historical response resets the cursor to the last event’s modification revision plus one, without taking the maximum with the creation header (`WatchImpl.java:313 @ jetcd-0.8.7`). For example, events through 60 after `Created(100)` set the cursor to 61.

The skip schedule therefore needs a disconnect before that first response corrects the cursor, or another creation acknowledgment that advances it again. A disconnect after any arbitrary partial replay is not sufficient. Adding `Math.max` to the event assignment would retain the premature high cursor and worsen the problem.

### 6.3 A scheduled-resume exception can stop recovery

**Verified code, High confidence; reachable trigger in 0.8.7, Moderate confidence:** Open [jetcd #1227](https://github.com/etcd-io/jetcd/issues/1227) reports one unsuccessful reconnect followed by no more attempts, using jetcd 0.7.5 and etcd 3.5.9. The current scheduled-future failure callback still only logs a warning (`jetcd-core/src/main/java/io/etcd/jetcd/impl/WatchImpl.java:357 @ jetcd-0.8.7`). It neither schedules another attempt, closes the watcher, nor calls the listener with that scheduled failure.

Two paths need separate treatment:

- **Asynchronous RPC error:** `onError` resets `started` and schedules another attempt when the error is recoverable (`:323`, `:327`, `:339`, `:353`).
- **Exception escaping scheduled `resume()`:** the scheduled future completes exceptionally and its callback only logs. Because `resume()` sets `started` before building and opening the stream (`:160`, `:164`, `:186`), an exception can also leave that flag set.

**High confidence — source-verified status predicate:** `isClosed()` checks closure flags; it does not check whether a stream is active or another retry is scheduled (`:150`).

This is a credible recovery-state gap, but it does **not** establish that every failed network attempt takes the exception path or that all disconnects get only one retry. A test must inject an exception at the actual current resume boundary and check subsequent retry scheduling, listener signals, and `isClosed()`. The old issue report is supporting evidence of the class, not a reproduction on the target versions.

**Upstream update, 2026-11-08 — High confidence:** [PR #1617](https://github.com/etcd-io/jetcd/pull/1617), previously open, merged on 2026-11-07 as `dc4fc009cc54`. It resets channel connect backoff for `Impl.retryPolicy` calls and is absent from 0.8.7. Its description explicitly excludes `WatcherImpl.reschedule()`; the diff changes `Impl.java`, while `WatchImpl.java` remains unchanged at inspected main `b4ccc3dea4ce`. This is an **unreleased unary-retry improvement**, not a fix for watch resumption or the scheduled-exception path.

### 6.4 API coverage, defaults, and test gaps

**High confidence in the source-verified API and default differences below.** None is a performance comparison; the suggested deployment checks are recommendations.

| Aspect | Go client v3.7.3 | jetcd 0.8.7 | Production implication |
|---|---|---|---|
| Watch stream allocation | Logical watches share a stream when outgoing metadata matches (`client/v3/watch.go:330`). | Each watcher opens its own RPC (`jetcd-core/src/main/java/io/etcd/jetcd/impl/WatchImpl.java:186`). | Go's explicit progress coupling can span multiple logical watches. This does not establish relative throughput or memory cost. |
| Leader requirement | Opt-in context metadata (`client/v3/ctx.go:26`). | `requireLeader` defaults false (`jetcd-core/src/main/java/io/etcd/jetcd/options/WatchOption.java:154`). | Both need an explicit choice of leader-loss behavior. |
| Transport keepalive | Applied by the client only when `DialKeepAliveTime > 0` (`client/v3/client.go:231`). | Builder defaults to 30-second interval, ten-second timeout, and keepalive without calls (`jetcd-core/src/main/java/io/etcd/jetcd/ClientBuilder.java:66`). | Different defaults for transport-failure detection; neither verifies event freshness. |
| Requesting progress | Context routes to a shared stream (§3.2). | `Watcher.requestProgress()` writes to that watcher's stream (`WatchImpl.java:226`). | jetcd avoids this particular cross-context routing trap. An attempted write is still not proof that progress was received. |
| Recoverable error reporting | Ordinary recoverable receive failures retry without a public terminal response (§2.2). | Listener sees `onError` before automatic rescheduling (`WatchImpl.java:327`). | Application retry must be coordinated with library retry; launching an additional watch on every error can create competing subscriptions. |
| Fragmentation | Has a request option and reassembles responses (`client/v3/watch.go:590`, `:1073`). | The inspected `WatchOption` and create builder expose/set no fragmentation option (`WatchImpl.java:164`). | Test large revisions against the configured receive-message limit; absence of fragmentation is an API limitation, not proof that every response above etcd's request-size limit fails. |

**High confidence — source-verified message handling and configuration:** For the last row, an unfragmented server watch sends the complete response rather than enforcing `maxRequestBytes` as a response cap (`server/etcdserver/api/v3rpc/watch.go:481 @ v3.7.3`). jetcd permits configuring `maxInboundMessageSize` (`jetcd-core/src/main/java/io/etcd/jetcd/ClientBuilder.java:332 @ jetcd-0.8.7`; `impl/ClientConnectionManager.java:176`). Effective transport limits, response sizes, and application behavior need testing before asserting a concrete failure threshold. Fragmentation also does not relax the uniqueness guarantee: splitting a response is not permission to duplicate an event.

#### Why the progress cursor does not establish a duplicate-event bug

**High confidence under a truthful server stream:** jetcd uses `max(currentCursor, progressRevision)` for a progress response (`WatchImpl.java:291 @ jetcd-0.8.7`), whereas Go uses progress revision plus one. That difference alone does not establish that an already delivered event at revision R will be replayed:

- After processing an event at R, jetcd's cursor is already R+1. Progress at R leaves `max(R+1, R) = R+1`.
- If a later progress watermark P advances the cursor to P, a truthful watermark means every matching event through P was already delivered. If a matching event at P had been delivered, its event branch would already have advanced the cursor to P+1. Otherwise replay beginning at P has no matching event at P to duplicate.

This reasoning assumes correct progress, serial response handling, and successful listener return; it is not a proof that every jetcd interleaving is safe. The ordinary positive-revision empty response takes the progress branch, as confirmed by `jetcd-core/src/main/java/io/etcd/jetcd/watch/WatchResponse.java:88 @ jetcd-0.8.7`; the later empty-response branch must not be substituted for that path. Resume at P rather than P+1 is conservative relative to such a watermark, but it does not cure the separate creation-cursor bug.

#### What the existing restart test covers

**Testing gap, High confidence:** `jetcd-core/src/test/java/io/etcd/jetcd/impl/WatchResumeTest.java:49 @ jetcd-0.8.7` opens a default watch, restarts the cluster, then writes a key. That test does not create retained history before registration or cut the stream between `Created` and replay. The launcher's default image is etcd v3.5.14 (`jetcd-launcher/src/main/java/io/etcd/jetcd/launcher/Etcd.java:31 @ jetcd-0.8.7`), predating the compaction-boundary and future-watch backports in §B.6–§B.7. Actual CI can override the image; the default alone does not prove what every upstream job runs. Any targeted test must explicitly pin etcd v3.7.3.

### 6.5 The Jepsen KV retry fix does not change watch resumption

**High confidence — historical scope:** [Jepsen's jetcd 0.8.2 report (2024-08-08)](https://jepsen.io/analyses/jetcd-0.8.2) investigates repeated KV transaction execution, not a watch-delivery defect. Its reported anomalies therefore need a different version/fix trail from §§6.1–6.3.

[jetcd #1444](https://github.com/etcd-io/jetcd/issues/1444) explicitly connects that report and [#1072](https://github.com/etcd-io/jetcd/issues/1072) to [etcd #14092](https://github.com/etcd-io/etcd/issues/14092), [#14890](https://github.com/etcd-io/etcd/issues/14890), and [#18424](https://github.com/etcd-io/etcd/issues/18424). [PR #1445](https://github.com/etcd-io/jetcd/pull/1445), patch `05f92e044328`, merge `08def01ee10d`, shipped in [0.8.5](https://github.com/etcd-io/jetcd/releases/tag/jetcd-0.8.5) and is in 0.8.7. **High confidence — verified release and code:** default Put/Delete/Txn exclude ambiguous `UNAVAILABLE` retries, retaining selected authentication-rejection retries; `withAutoRetry()` opts into broader retry. Source **@ jetcd-0.8.7**: [jetcd-core/src/main/java/io/etcd/jetcd/impl/KVImpl.java:66,98,123](https://github.com/etcd-io/jetcd/blob/jetcd-0.8.7/jetcd-core/src/main/java/io/etcd/jetcd/impl/KVImpl.java#L66), [support/Errors.java:29–40](https://github.com/etcd-io/jetcd/blob/jetcd-0.8.7/jetcd-core/src/main/java/io/etcd/jetcd/support/Errors.java#L29), and [options/TxnOption.java:31–46](https://github.com/etcd-io/jetcd/blob/jetcd-0.8.7/jetcd-core/src/main/java/io/etcd/jetcd/options/TxnOption.java#L31).

**High confidence — separate mechanisms:** `ClientBuilder.retryMaxAttempts(0)` limits the Failsafe policy in [jetcd-core/src/main/java/io/etcd/jetcd/impl/Impl.java:121–155 @ jetcd-0.8.7](https://github.com/etcd-io/jetcd/blob/jetcd-0.8.7/jetcd-core/src/main/java/io/etcd/jetcd/impl/Impl.java#L121). Watch recovery instead uses [WatchImpl.java:323–367 @ jetcd-0.8.7](https://github.com/etcd-io/jetcd/blob/jetcd-0.8.7/jetcd-core/src/main/java/io/etcd/jetcd/impl/WatchImpl.java#L323), with its own error predicate and scheduled `resume()`. The KV fix neither disables that recovery nor changes its cursor assignments. It also does not fix #1227 or establish that #1352's reported failure is gone. Those two issues remained open when checked on 2026-11-08; their current-version reproduction limits remain as stated above.

**High confidence — diagnostic distinction:** If one application write is executed twice and creates two revisions, a watch delivering both committed mutations has not duplicated a single event. Deduplicating by key and value would erase legitimate history. To diagnose a repeated notification, compare event revisions and the producer's actual RPC attempts, then check the consumer's own replay cursor. Correct watch delivery cannot make an unsafe producer retry into one logical transaction. Conversely, fixing producer retries cannot restore events skipped by a bad watch cursor. The source distinction is the KV execution policy above versus the event/resume paths in §§1.3–1.4 and §6.2; no new repeated-write experiment was run.

## 7. Consumer recovery and freshness policy

**For a state cache:**

1. Read a consistent snapshot at revision R and subscribe from R+1.
2. Apply complete responses in order and preserve the application's processed cursor.
3. On compaction or uncertain continuity, replace the snapshot.
4. Supervise progress independently where silent stalls are possible; trigger recovery when the watch cannot demonstrate catch-up (§7.2).
5. For a decision requiring current information, obtain a new quorum-read revision after the read begins and wait until this cache is complete through it.
6. If the deadline expires, report unavailable rather than certify freshness.

This is the reasoning behind the Kubernetes cache-read design discussed in §5.2 and §B.2. It does not make ordinary asynchronous watch callbacks linearizable.

No generic full relist can recover the exact sequence of transitions that occurred and was compacted between snapshots. Applications requiring every transition need sufficient retention and an explicit policy for unrecoverable gaps; “eventual current state” and “every transition delivered” are different requirements.

### 7.1 Failure states and required responses

The following are conclusions from the verified code paths and failure schedules above, not measurements of incident frequency.

| Client state | What can silently go wrong | Required distinction or response |
|---|---|---|
| Connected to a member that is not applying new majority commits | Cache remains old while TCP and local progress still work. | **High confidence:** establish a target revision through a linearizable read; demand watch progress to that target or declare the cache unavailable. |
| Retrying a broken transport indefinitely | No channel closure arrives to trigger application recovery. | **High confidence:** use explicit freshness/deadline policy; a transport keepalive alone is insufficient. |
| Leader-required watch goes through the standalone gRPC proxy | The metadata check is bypassed and the watch remains silent during member isolation. | **High confidence, reproduced:** direct-member failure-detection expectations do not carry through this proxy (§4.1). |
| Waiting for explicit progress on the wrong metadata stream, through the proxy, or beside a future watch | Repeated calls succeed locally without supplying the expected watermark. | **High confidence:** match metadata, use a compatible path, account for stream scope, and enforce a local deadline (§3.2). |
| Attached to an ended proxy broadcast | Successful downstream creation can be followed by permanent silence. | **Moderate confidence:** exercise #21813; require terminal-state propagation or subscription reconstruction. |
| Sharing a stream caught in cancellation blocking | New and existing logical watches cannot receive responses. | **Moderate confidence for deployment reachability:** test the actual adapter/transport path; an ordinary reconnect policy cannot execute if its dispatcher is blocked. |
| Resume cursor taken from `Created` or an ordinary batch header | Requested retained events are skipped on reconnect. | **High confidence as a protocol rule; Moderate for the jetcd schedule:** track a complete delivered event revision or valid progress watermark. |
| Revision-zero watch terminates before the application obtains a usable cursor | Restarting at zero omits the outage interval. | **High confidence:** establish an explicit starting revision or rebuild state; a creation header is not an exact registration watermark (§3.1). |
| Rejected Go creation races with terminal stream failure, or jetcd scheduled resume throws | Cleanup/retry can stop without the expected application-visible state transition. | **Moderate confidence in untested current triggers:** instrument lifecycle state; `WithRequireLeader` and `isClosed()` do not cover these paths (§3.3, §6.3). |
| Compaction occurred before replay completed | The missing history no longer exists. | **High confidence:** honor the error and replace the cached snapshot. A full relist recovers current state, not every intervening transition. |
| Delete received without enough data to update the derived cache | Application skips it but continues advancing other objects. | **High confidence:** fail/rebuild, or use a representation that can correctly apply the delete without the prior value. |
| Cache was already corrupted, then merely reconnected from its latest cursor | The skipped delete is older than the cursor and never replayed. | **High confidence:** resumption preserves continuity from a cursor; it does not repair arbitrary incorrect local state. |
| Library received events but application crashed before applying them | Restart from the library's latest received revision skips application work. | **High confidence:** persist application state and its processed cursor consistently, or relist. |
| Operator restored an old backup with lower revisions | Existing consumers wait for revisions from the former history or retain inconsistent cached objects. | **High confidence:** treat this as history replacement, not ordinary failover; use the documented revision-bump and compaction procedure. |

**High confidence — documented recovery mechanism:** The final row is independently documented in [etcd's v3.7 disaster-recovery guide](https://etcd.io/docs/v3.7/op-guide/recovery/#restoring-with-revision-bump): `--bump-revision` combined with `--mark-compacted` forces consumers away from obsolete replay positions. This cannot recreate events missing from the backup. It provides an explicit discontinuity so caches can be invalidated.

### 7.2 Progress supervision without blind scheduled restarts

**High confidence in the observability limit; Moderate confidence in the proposed recovery policy:** A quiet key range and a stalled subscription can look identical to an application waiting only for change callbacks. Where the stack lacks reliable failure notification, the consumer needs an independent progress check, periodic reconciliation, or a bounded subscription policy. Waiting for the next event alone cannot start recovery from a permanently silent stream.

**Moderate confidence — proposed, untested integration:** for a state cache, a progress watchdog can work as follows, provided its read and recovery tasks remain independent of the stalled client path:

1. **Keep a usable recovery path.** Prefer direct member connections with reachable alternative endpoints, leader-required contexts, and transport failure detection. Treat unexpected channel closure as loss of the subscription even without a final error (§2.3).
2. **Periodically establish a fixed target.** Make a deadline-bounded linearizable read and retain its revision R. Request progress on the watch's actual stream, with matching metadata (§3.2). Wait until the application has processed all matching events through R, using a valid progress response when no matching event establishes that coverage. Do not continually move R forward while waiting: unrelated writes can keep advancing the cluster revision.
3. **Recover when the read or catch-up does not complete within the deadline.** Mark the view uncertain; stop accepting callbacks from the retired subscription into the replacement state. Resume against a usable member from the application's last complete processed revision plus one, or replace the snapshot if compaction or uncertain continuity prevents replay. Coordinate this with library retries and preserve cursor/state consistency. An unavailable quorum prevents recovery; keep the uncertainty visible until service returns.
4. **Bound the recovery machinery too.** A blocked dispatcher or client close cannot be assumed to repair itself. Isolate the affected client and provide a recovery/escalation path whose progress does not depend on that same blocked task (§3.3, §4.2, §6.3). Timeout should not silently certify successful recovery.

A local progress response at an old revision and a successful keepalive do not establish progress toward the majority's state. Conversely, a timeout indicates inability to verify progress within the chosen policy, not proof that the watch will never recover. Successful catch-up still depends on reachable healthy members, sufficient processing capacity, and replay retention or snapshot availability. Deadline selection needs workload testing; overly short deadlines can repeatedly interrupt useful catch-up.

Keep the watchdog independent of the watched callback/dispatcher and put a deadline on its read path too. A check queued behind the same deadlocked client task is not an independent check. Establishing that the watch has not caught up is possible without observing the internal failure; automatically restoring progress still requires a recovery path that can run. This policy therefore adds detection and recovery machinery rather than turning the flag into a liveness guarantee.

This adds periodic independent checking, which is the operational burden identified in the critique. It need not poll the entire watched state or restart every watch at fixed intervals. Blind rotation can reconnect to the same stale endpoint and can lose continuity if it resets the cursor. Watches still carry changes efficiently; a watchdog supplies the missing recovery trigger.

**Moderate confidence — conditional design benefit:** periodic supervision can support recovery and eventual convergence under the reachability, scheduling, and replay/snapshot assumptions above. It does **not** make a later cached read linearizable: that stronger requirement needs a new quorum-derived target after that read begins (§5.2). This protocol is a recommendation, not a newly tested implementation. Its building blocks are the current linearizable-read/progress design in §5.2, stream-scoped progress in §3.2, and application cursor rules in §1.2.

## 8. Design assessment and the Chubby comparison

**High confidence:** The strongest supported criticism concerns both eventual progress and knowledge of cache validity. The watch API principally offers an asynchronous revision stream and error-driven replay; it does not provide a complete end-to-end protocol that detects every stalled observation path and initiates recovery. A surviving quorum and healthy unrelated reads do not ensure that a particular watcher is still advancing. Permanent silence can defeat eventual convergence even when immediate freshness is unnecessary (§1.5).

**High confidence — protocol described by the paper:** Chubby's client protocol makes cache validity explicit. On local session uncertainty, it disables the cache; master failover includes a cache-flushing event and acknowledgment/expiry barrier. The paper also warns that some other events may have been lost across failover. See Chubby §2.7–2.9, pp. 6–9, especially `doc/chubby-osdi06.md:229`, `:265`, and `:303`. Thus Chubby should not be described as guaranteeing an identical lossless event stream through every failover. Its relevant strength here is explicit management of whether cached state remains usable.

**Verified design distinction, High confidence:** Chubby waits for cache invalidation acknowledgment or cache-lease expiry before a modification proceeds (§2.7, p. 7, `doc/chubby-osdi06.md:239`). Events and invalidations travel in session KeepAlive replies, coupling continued session maintenance to invalidation acknowledgment (§2.8, p. 7, `:261`). etcd's Watch protocol has no application-processing acknowledgment (§1.2). Its writes therefore do not wait for applications to invalidate their derived caches, although local watch locks and delivery work can still delay apply (§4.5). Chubby's invalidation barrier and etcd's resumable history solve different parts of the problem: neither should be declared uniformly superior from this comparison, and adding a periodic heartbeat alone would not reproduce Chubby's cache-validity protocol.

**Inference, High confidence:** etcd can support correct replicated-state consumers without putting subscriptions through Raft, provided replay positions and progress are truthful and clients reconstruct state on discontinuity. Kubernetes's quorum-revision-plus-progress approach is a concrete example. Making registrations consensus operations would not by itself repair an incorrect cursor, a dead proxy receive loop, a blocked response dispatcher, or a client that advances its checkpoint before applying a delete.

**Moderate confidence — architectural assessment grounded in the preceding mechanisms:** The narrower design concern is therefore justified: **watch continuity, eventual progress, and client knowledge of cache validity require an additional protocol beyond the consistency of the stored KV data.** The current open issues show that this protocol spans multiple independently fallible components. The broader proposition—that local watches inherently cannot maintain their documented safety promises through ordinary leader failover—is not established by the evidence examined.

## 9. Remaining validation work

These tests would resolve the material uncertainties. Appendix C's two limited experiments are completed; they do not complete the broader interleaving, replay, and recovery tests below:

### 9.1 Event continuity and cursor recovery

- **Ordinary failover baseline:** On three v3.7.3 members, watch a range with known transaction contents; isolate a leader or follower while preserving quorum; test both leader-required and ordinary contexts; then heal the partition. Verify exact matching-event coverage or explicit compaction failure, not just successful writes and an open watch channel.

- **Explicit-ID reuse:** Port #22673's controlled cancellation/event interleaving to the release; verify key range, start revision, cancellation boundary, and retained-event accounting.

- **jetcd creation cursor:** Force `Created(header=200)` for a watch starting at 101, cut before replay, and verify that retained events at 110 and 150 are still delivered after retry.

- **Recovery under compaction and backpressure:** Validate both event completeness and terminal error delivery; check bounded versus unbounded buffering separately. Performance characterization belongs on the project's eventual benchmark hardware.

- **Go revision-zero boundary:** Pause between local registration and construction of `Created`; commit at least two matching revisions; disconnect after the client receives only `Created`. Compare replay with an explicit-revision control. Separately test R=H to identify replay of the initial boundary revision. The key assertion is the exact requested/delivered interval, not merely that reconnection succeeded.

### 9.2 Failure notification and lifecycle recovery

- **Direct leader-required watch during a WAL stall:** On three members, block the watched follower's WAL persistence while leaving its gRPC service and the client runnable. Commit a subsequent matching update through the other two members. Check the stored leader ID, watch events/errors, transport keepalive, and an independent quorum-read/progress watchdog. Restore storage to distinguish recoverable delay from a nonprogress path that persists while the injected fault remains. Reuse existing failpoints such as `walBeforeSync` before adding a harness; this is a correctness test, not a storage benchmark.

- **Proxy termination:** Force a terminal upstream watch response/channel closure, retain the downstream stream, and create or coalesce a compatible watch before broadcast cleanup. Assert delivery or explicit failure. A recoverable TCP break alone is not a sufficient trigger.

- **Cancellation blocking:** Reuse the upstream #21064/#21065 reproduction against the in-process adapter, then separately test ordinary gRPC flow control. Demonstrate whether stream cancellation or reconnect actually unwinds the cycle.

- **#22184 production reachability:** Identify an actual concurrently executing caller that obtains the underlying store write lock while an apply transaction is active. The snapshot-send assertion in the issue is insufficient for the inspected release.

- **Leader-loss error delivery under brief consumer backpressure:** Extend the existing leader/follower partition test by keeping the one-slot application output channel full. Leave the application's context active, induce `ErrNoLeader`, and delay draining until after the 250 ms final-error timeout. Check that channel closure can occur without the promised canceled/error response; separately verify that a defensive consumer resumes from its last processed revision.

- **Partition topology versus retry policy:** Keep the client–minority connection intact in one case and disrupt that connection as well in another. Observe the public watch channel, not just the server's cancellation or gRPC connectivity state. Verify whether the application receives a terminal status, remains indefinitely in retry, or reaches an explicit local uncertainty deadline supplied by the application.

- **Rejected-creation teardown:** Deliver a rejected `Created` response, then force terminal stream failure before the closing-channel handoff. Assert that the watch call/channel, goroutines, and client close all terminate. This needs a controlled schedule; #17622's historical discussion is not a current passing regression test.

- **jetcd lifecycle and message size:** Inject a synchronous exception in the scheduled resume and observe retry state and callbacks. Separately exercise a large complete revision with explicitly recorded receive-message limits. Neither test is covered by the default restart-then-put test.

### 9.3 Progress routing and proxy topology

- **Quorum-preserving convergence:** Keep the client able to reach all three members; isolate only the watched member's peer links, commit a final matching update through the majority, then stop matching writes. Compare ordinary direct, leader-required direct, and proxied watches, with and without a progress watchdog. Check catch-up or a recovery-triggering signal, not merely interior event ordering. A finite timeout demonstrates a missed operational target; combine the observation with a source-level nonprogress path before claiming indefinite failure.

- **Progress and proxy coverage beyond the experiments:** Test metadata mismatch explicitly; compare periodic per-watch progress with requested stream-wide progress; exercise a real networked proxy and multiple backend endpoints. The completed same-stream test verifies neither periodic timing nor multi-backend detector affinity.

### 9.4 Existing test foundations

Existing foundations include `tests/integration/clientv3/watch/watch_test.go:340 @ v3.7.3` for disconnect resumption, `:395` for compaction during resumption, `tests/integration/clientv3/watch/v3_watch_restore_test.go:56 @ v3.7.3` for snapshot recovery, and `tests/robustness/validate/watch.go:30 @ v3.7.3` for ordering, uniqueness, atomicity, reliability, resumability, and progress validation. Reuse those before inventing a separate harness. The missing empirical result is how often, and under precisely which deployed conditions, the current source-level failure schedules are reachable—not whether consistent KV storage alone proves watch correctness.

## Appendix A. Evidence, versions, and research record

“Verified” means checked in source, repository history, a primary report, or an explicitly identified experiment. It does not mean that every failure scenario was reproduced. High confidence in a mechanism can coexist with Moderate confidence in its operational trigger.

### A.1 Search and evidence limits

I searched GitHub's open etcd issues using `watch`, `watcher`, and `watches` in titles and bodies, followed linked historical reports and PRs, searched Kubernetes reports, and inspected the corresponding stable-release source. The broad `watch` query returned 37 issues, including unrelated mentions; the additional queries found an adjacent lock-wait issue. This is an issue-led source audit, not an exhaustive proof over every watch interleaving.

**Two small Docker functional experiments were run during the expansion; no benchmarks were run.** Appendix C records their exact scope, topology, commands, observations, and limits. Everything else marked “verified” was checked in source, repository history, or an upstream primary source. The original audit ran no tests. Upstream reproductions belong to their authors, and unexecuted failure schedules remain explicitly identified as inferences. GitHub's unauthenticated core API quota became exhausted during the original research; not every original issue's full comment history was retrievable.

### A.2 Version and upstream checks

The original audit verified the local commit IDs and checked releases with read-only remote listings. On 2026-11-01, `git fetch --tags --prune origin` in both clones, stable-tag sorting, GitHub's latest-release API, and jetcd's Maven Central metadata again confirmed v3.7.3 and jetcd-0.8.7 as the latest stable releases. Neither checkout moved. This review changed no source files, worktrees, shared project files, or peer analyses; its only project edit is this report.

Upstream comparisons use etcd `main@1573d59416efa220421c451f5266b24a5d8374ef`, `release-3.7@34d8004033dd3693e81bc12f00472b439167ff2d`, and jetcd `main@9f509770201e8fba1b7f67340eff086f0d29d976`; remote branch tips matched the available local refs. These are **unreleased comparison points**, not the versions evaluated as production releases. The code paths for current defects discussed in the main text have no corrective change in those inspected refs. Differences in the examined etcd files include failpoint comments, a response-header leader ID on `main`, and unrelated comments/lint annotations.

### A.3 Upstream reference inventory

Original issue states retain their 2026-10-31 observation date. Newly discussed issue/PR states were checked during the 2026-11-01 expansion.

Open etcd issues [#21813](https://github.com/etcd-io/etcd/issues/21813), [#18879](https://github.com/etcd-io/etcd/issues/18879), [#22673](https://github.com/etcd-io/etcd/issues/22673), [#22184](https://github.com/etcd-io/etcd/issues/22184), [#22264](https://github.com/etcd-io/etcd/issues/22264), [#16839](https://github.com/etcd-io/etcd/issues/16839), [#22294](https://github.com/etcd-io/etcd/issues/22294), [#22614](https://github.com/etcd-io/etcd/issues/22614), and [#10681](https://github.com/etcd-io/etcd/issues/10681); historical correctness issues [#15220](https://github.com/etcd-io/etcd/issues/15220), [#15271](https://github.com/etcd-io/etcd/issues/15271), and [#17529](https://github.com/etcd-io/etcd/issues/17529). Additional client, Kubernetes, and PR references appear with their findings.

### A.4 Peer review and editorial revision

The expansion first read this report, then `watchers-claude.md` and `watchers-agy.md` as investigation leads. Each adopted addition was checked independently against the pinned source, history, primary reports, or the experiments. Some peer code excerpts do not match v3.7.3, and several proposed conclusions do not follow from the actual code. Appendix D records the adoption and rejection decisions. All supporting mechanisms and qualifications needed to read this report are included here.

The subsequent clarity pass separated the executive summary and purpose from this evidence record, grouped the main text by reader question, moved historical fixes and test recipes into appendices, and added closing Conclusions. It retained the version pins, findings, qualifications, data, and source references. No new runtime tests were run for the editorial pass.

For citations to the previous organization, use this section map:

| Previous location | Current location |
|---|---|
| 1: scope and evidence standard | Scope; §1.1; Appendix A |
| 2; 3.1; 3.3: contract, data path, replay | §1.2; §1.3; §1.4 |
| 3.2.1; 3.2.3–3.2.6: leader detection and client consequences | §2.1–§2.5 |
| 3.2.2: WithRequireLeader history | §B.1 |
| 3.4–3.6: Go recovery and progress | §3.1–§3.3 |
| 4: current server/proxy/cache findings | §4; three proxy paths now have §4.1.1–§4.1.3 |
| 5.1–5.7: historical fixes and reports | §B.2–§B.8 |
| 6; 7: Kubernetes; jetcd | §5; §6 |
| 8; 9: consumer responses; design assessment | §7; §8 |
| 10; 11; 12: validation; experiments; peer review | §9; Appendix C; Appendix D |

### A.5 Eventual-consistency and liveness review, 2026-11-03

The follow-up checked the v3.7 API/API-guarantees and failure documentation, the accepted maintainer answer in discussion #15423, and the two primary definitions cited in §1.5. After the user's correction, the latest `watchers-agy.md` changes supplied the missed official eventual-consistency statement in `tests/robustness/README.md:102,111 @ v3.7.3`; both lines were verified in the local checkout. The client-design liveness requirement was also checked. The report now distinguishes this explicit repository documentation from the corroborating maintainer answer.

The same follow-up re-read the leader predicate, Raft loop, WAL persistence, apply scheduler, Go terminal-error branch and cleanup, and the peer's nine failure categories. Section 2.6 adds a direct-member storage-stall counterexample with `WithRequireLeader` enabled and corrects the earlier implication that leader-loss termination establishes convergence. This counterexample is pinned to v3.7.3 and was not runtime-executed. The independent-observability distinction and remaining peer qualifications are in §D.3.

Tag fetches, GitHub's latest-release endpoints, and jetcd Maven metadata still identify v3.7.3 and jetcd-0.8.7 as the latest stable releases; the pinned checkouts did not move. At the fetched etcd `main@4b0b74502571` and `release-3.7@d8d8cef36250`, the examined `client/v3/watch.go`, `client/v3/ctx.go`, `server/proxy/grpcproxy/watch.go`, `server/proxy/grpcproxy/watch_broadcast.go`, and `server/etcdserver/api/v3rpc/interceptor.go` have no differences from v3.7.3. This is a narrow source comparison, not a fresh audit of every issue or upstream path; earlier issue-state dates remain as recorded.

This review updates this report and `etcd-issues-and-mitigations-codex.md`. It adds no runtime tests, benchmarks, worktrees, or source changes. The new partition/storage-stall scenarios and watchdog protocol are source-derived analysis and recommendations; Appendix C's finite experiments retain their original limits.

### A.6 Jepsen and upstream-fix review, 2026-11-08

Both reports were read in full. Their linked issues, documentation patches, jetcd #1444/#1445, and containing release tags were checked independently against GitHub and the local histories. `git fetch --tags --prune origin`, stable-tag sorting, GitHub latest releases, and jetcd Maven metadata still identify etcd v3.7.3 and jetcd 0.8.7. HEADs did not move. The historical findings are integrated in §1.6 and §6.5; the lock-specific fix is traced in [leases §5.6](leases-codex.md#56-jepsen-lock-and-retry-findings-at-the-pinned-releases).

At jetcd main `b4ccc3dea4ced976348beb323d6e36ecb29040d2`, `WatchImpl.java`, `KVImpl.java`, and `Errors.java` have no differences from 0.8.7. `Impl.java` now includes the unreleased #1617 backoff change (§6.3). The separate unreleased keepalive clean-end fix #1628 is recorded in [leases §5.5](leases-codex.md#55-jetcd-deadlines-and-errorretry-behavior); neither change is a correction to the watch cursor or scheduled-resume exception. This review adds no runtime tests, benchmarks, or worktrees and does not advance unrelated issue-state dates. Appendix C's two experiments retain their original date and scope; no post-fix Jepsen certification is claimed.

## Appendix B. Historical fixes and unresolved reports

Historical reports explain the known failure classes. A fixed defect is not an unfixed v3.7.3 finding; a closed issue without an identified corrective change is not proof of a fix.

### B.1 Why WithRequireLeader was introduced

**Verified history, High confidence:** The option predates Kubernetes's 2020 change. The etcd client helper was introduced in May 2016 by commit [`68eaf4083a73`](https://github.com/etcd-io/etcd/commit/68eaf4083a7316403d8b9517dcc3c9045ddd0dd3); cancellation of existing leader-required streams was introduced in commit [`9c103dd0dedf`](https://github.com/etcd-io/etcd/commit/9c103dd0dedfc723cd4f33b6a5e81343d8a6bae7). Both dates and diffs were checked in local history.

The partition-isolation rationale is explicit in [etcd #7247's maintainer explanation](https://github.com/etcd-io/etcd/issues/7247#issuecomment-276154894): default minority-side watches wait; requiring a leader permits their termination. The 2018 documentation change [`11ead62b9d67`](https://github.com/etcd-io/etcd/commit/11ead62b9d67cec8eff6bfdfcd3f552314f4152e) made the partition motivation explicit.

Kubernetes later enabled it for API-server storage watches in [PR #89881](https://github.com/kubernetes/kubernetes/pull/89881), patch commit `70c9f770d7aa2194bfd3f58fe01756a7d200b866`, authored 2020-03-31. The patch describes improving detection and termination when etcd loses quorum and links the unresolved transport-retry discussion. It is a responsiveness improvement, not a claim that all partitioned clients now receive a timely failure. The current call remains at `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go:1156 @ Kubernetes v1.37.2`.

### B.2 #15220: a false progress watermark could make reconnect lose events

[#15220](https://github.com/etcd-io/etcd/issues/15220) is closed. Its failure sequence directly matches the requested focus on client state: request progress while events are still pending; receive a watermark ahead of them; disconnect; resume using that watermark; omit the undelivered interval.

**Verified fix, High confidence:** [PR #15237](https://github.com/etcd-io/etcd/pull/15237), commit `74feb229c723150cfd89d1370e3aa575312c5d79`, routes requested progress through the watch event stream and waits for synchronization instead of sending it through the independent control channel. That change is in v3.7.3: `server/etcdserver/api/v3rpc/watch.go:395 @ v3.7.3`; `server/storage/mvcc/watchable_store.go:515 @ v3.7.3`.

The Kubernetes proposal [KEP-2340, Consistent Reads from Cache](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/2340-Consistent-reads-from-cache/README.md#bug-in-etcd-progress-notification) explicitly identifies this as a silent correctness hazard for cache reads and records the historical fixes in etcd 3.4.25 and 3.5.8. Its algorithm obtains a quorum-read revision and waits until the cache has processed a sufficient watch watermark. A falsely advanced watermark invalidates that reasoning.

The root cause was ordering between local response paths, not an uncommitted write being exposed or a requirement to replicate watch registration.

### B.3 #15271: snapshot catch-up could replay events backward

[#15271](https://github.com/etcd-io/etcd/issues/15271) is closed by, among other work, [PR #15492](https://github.com/etcd-io/etcd/pull/15492). A synchronized watch's internal minimum revision was not kept current. Moving it into replay after a leader snapshot could therefore resend older events.

**Verified fix, High confidence:** Commit `a690707c57524ed5aee3263220612979df4e4d4e` updates the next-needed revision even while a watcher remains synchronized. The update remains at `server/storage/mvcc/watchable_store.go:486 @ v3.7.3`. Later commit `cf0369e4ad43463f6d3b1e3745b21e6fcc9ffe7d`, also included, preserves a requested future revision during restore/catch-up; see `:374`.

This is genuine evidence that failover/recovery paths have violated watch ordering. It is not evidence that this particular defect remains in the target release. Snapshot catch-up within a surviving cluster must also be distinguished from restoring an older disaster-recovery backup.

### B.4 #17529: failed compaction notification could turn into silent event loss

[#17529](https://github.com/etcd-io/etcd/issues/17529) is closed by [PR #17555](https://github.com/etcd-io/etcd/pull/17555). Its original discussion includes a proposed victim-batch-loss explanation that the author later withdrew; that is not the fix's mechanism.

**Verified fix, High confidence:** Commit `405862e80779a3dfae83cf50f4c57bb0c9a4de0f` keeps a watcher whose requested revision is compacted from being advanced as though synchronization succeeded when its compaction response could not be queued. The guard remains at `server/storage/mvcc/watchable_store.go:369 @ v3.7.3`. Without it, backpressure could cause the server to skip the error and later resume delivery beyond a missing interval. The release has a focused regression test at `server/storage/mvcc/watchable_store_test.go:488 @ v3.7.3`.

This historical bug illustrates why “history is compacted” must produce a visible discontinuity, not merely a new resume position. The different retirement defect in #22294 is still open; it should not be confused with this repaired event-loss mechanism.

### B.5 #15402: transport scheduling can starve an otherwise correct watch

[#15402](https://github.com/etcd-io/etcd/issues/15402) is closed as completed. It reported TLS watch starvation when large read responses share the connection and linked Go HTTP/2 scheduling issue [golang/go #58804](https://github.com/golang/go/issues/58804). Both mutation events and progress notifications could be delayed.

The inspected release retains support for separating HTTP and gRPC listeners (`server/embed/config.go:614 @ v3.7.3`) and a combined HTTP handler (`server/embed/serve.go:317 @ v3.7.3`). **Moderate confidence:** This history remains relevant when choosing and testing listener topology, but neither the old issue's “all versions” statement nor the presence of a combined handler establishes that the same starvation remains reproducible with v3.7.3's current Go/gRPC dependencies. That needs a current transport test. It is not a Raft consistency failure.

### B.6 #20221 / #19487: resuming on a lagging member must preserve a future start revision

**Verified fixes, High confidence:** A resume revision can be ahead of the new member's local store. This is legal: the watch waits for that member to apply the requested revisions. Two historical bugs broke that behavior:

- A future watch could receive progress below its requested starting revision. Because the Go client consumes progress as `header.revision + 1`, that could move its next replay request backward. [PR #20229](https://github.com/etcd-io/etcd/pull/20229), commit `973847aa246b1255df648643f900958035f89888`, stores the original start revision and suppresses such progress. The guard is at `server/storage/mvcc/watchable_store.go:525 @ v3.7.3`; the client cursor update is at `client/v3/watch.go:864 @ v3.7.3`.
- Following a leader snapshot, synchronization could lower a future watch's minimum revision to `currentRev + 1`, delivering events below its requested start. [PR #20281](https://github.com/etcd-io/etcd/pull/20281), commit `cf0369e4ad43463f6d3b1e3745b21e6fcc9ffe7d`, preserves the greater revision. The current assignment is at `server/storage/mvcc/watchable_store.go:374 @ v3.7.3`.

These changes address [#20221](https://github.com/etcd-io/etcd/issues/20221) and [#19487](https://github.com/etcd-io/etcd/issues/19487). Local commit ancestry and first stable containing tags were checked: both mainline changes ship in v3.7.0, with backports in v3.6.2 (`68075c999`, `438287a59`) and v3.5.22 (`d1d3dc296`, `2d72ea1b8`). They are present in v3.7.3. Saying this affects every release “before 3.7” would incorrectly include patched 3.5 and 3.6 releases.

This history strengthens the need to test switching members, including to a member behind the client's cursor. It does not show that local progress is intrinsically unsafe. A truthful local watermark can be useful without being a quorum-fresh watermark. The current tradeoff is suppressed responses for future watches and their stream-mates (§3.2), not permission to publish a false watermark.

### B.7 #18089 / #19179: delete tombstones at the compaction boundary

**Verified fixes, High confidence:** These are additional historical missing-delete mechanisms beyond the delivery/backpressure bug in §B.4:

| Historical defect | Fix checked in history and v3.7.3 source | First stable releases checked |
|---|---|---|
| [#18089](https://github.com/etcd-io/etcd/issues/18089): dropping a delete tombstone when compacting at its revision | [PR #18274](https://github.com/etcd-io/etcd/pull/18274), merge `1d0a6b39780e730c590578c107de461323247012`; the index retains a generation whose tombstone is **at** the compaction revision (`server/storage/mvcc/key_index.go:269 @ v3.7.3`). | v3.6.0; backports v3.5.16 (`8d98510e1`) and v3.4.34 (`2828a12ad`). |
| [#19179](https://github.com/etcd-io/etcd/issues/19179): losing that retained tombstone from the reconstructed index after restart | [PR #19188](https://github.com/etcd-io/etcd/pull/19188), merge `83cf7bb4fc621f017fcaa9a735194fcec23420b8`; restore explicitly handles a tombstone that is a key's first surviving revision (`server/storage/mvcc/kvstore.go:490 @ v3.7.3`; `server/storage/mvcc/key_index.go:119 @ v3.7.3`). | v3.6.0; backports v3.5.18 (`c6fcd2701`) and v3.4.36 (`f3cae135e`). |

First releases above were checked with `git tag --contains` on the corresponding mainline and backport commits, not inferred from issue closure. The robustness ledger separately identifies both defects (`tests/robustness/README.md:32 @ v3.7.3`, `:35`). A useful regression test must cover a watch starting **at** the compaction revision and repeat it after restart; ordinary reconnect away from that boundary is insufficient.

### B.8 An unresolved historical report is not a verified fix

[#20573](https://github.com/etcd-io/etcd/issues/20573) reported a resumability failure in fault testing. Its comments include a maintainer reproduction with snapshot activity, followed later by closure because failures had not been seen for a long time. The retrieved discussion supplies no identified root cause and corrective commit. **Moderate confidence in the historical classification; no current reproduction:** retain it as an open validation question, not as a demonstrated v3.7.3 defect and not as proof that every historical watch failure has been fixed. The original logs also do not establish that ordinary leader election alone caused the omission.

## Appendix C. Functional experiments and reproduction details

Both experiments ran on 2026-11-01 on this project's Linux/Docker analysis host. They used the existing official image `gcr.io/etcd-development/etcd:v3.7.3`, digest `sha256:0d27b9478551f431f82d1e0ddb6c87ad07ebae8b904c637f3a99213349523910`. The binary reported etcd 3.7.3, Git SHA `0a173d0`, Go `go1.26.10`, Linux/amd64. Client commands used `etcdctl` from the same image. No jetcd runtime was involved.

These were small correctness checks using container writable storage and tiny values, not throughput or latency measurements. Existing `etcdctl` and `etcd grpc-proxy` supplied the clients/server. Python orchestration only launched isolated containers, issued commands, checked output, and cleaned up; it did not implement a replacement watch client. This avoided modifying or building either source checkout. Containers and networks created for the checks were removed afterward; scratch scripts and logs were kept outside the project.

### C.1 Direct versus standalone-proxy leader-loss notification

**Topology and configuration:** Three etcd members on a private Docker peer network. Member M1 also joined a client network under alias `isolated`. The proxy joined only that client network and used the single endpoint `http://isolated:2379`. A direct watcher ran inside M1 against localhost; a proxied watcher ran inside the proxy container against its listener. Thus disconnecting M1 from the peer network preserved both client paths. An initial status response recorded M1's member ID equal to its leader ID: the isolated member was the initial leader.

Each member was created using the following command shape; `MEMBER` was its unique container/name and `CLUSTER` listed all three `name=http://name:2380` pairs. Containers were attached to their networks before startup, with no host ports published:

```sh
docker create --name "$MEMBER" --network "$PEER" --network-alias "$MEMBER" \
  "$IMAGE" /usr/local/bin/etcd --name "$MEMBER" --data-dir=/data \
  --listen-client-urls=http://0.0.0.0:2379 \
  --advertise-client-urls="http://$MEMBER:2379" \
  --listen-peer-urls=http://0.0.0.0:2380 \
  --initial-advertise-peer-urls="http://$MEMBER:2380" \
  --initial-cluster="$CLUSTER" --initial-cluster-token="$PREFIX" --log-level=error
docker network connect --alias isolated "$CLIENT" "$M1"
docker run -d --name "$PROXY" --network "$CLIENT" "$IMAGE" \
  /usr/local/bin/etcd grpc-proxy start \
  --endpoints=http://isolated:2379 --listen-addr=0.0.0.0:23790
```

Here `IMAGE` was the versioned image above, and `PREFIX`, `PEER`, `CLIENT`, `M1`, `M2`, and `PROXY` were fresh resource names. After starting the members and waiting for `etcdctl endpoint health`, the harness launched these two long-lived commands with output captured separately:

```sh
docker exec "$M1" /usr/local/bin/etcdctl \
  --endpoints=http://127.0.0.1:2379 watch foo
docker exec "$PROXY" /usr/local/bin/etcdctl \
  --endpoints=http://127.0.0.1:23790 watch foo
```

The harness wrote `foo=v1` through M2 and waited until **both** outputs contained it, ensuring registration rather than assuming it from a sleep. It recorded M1's peer IP with `docker inspect`, then performed:

```sh
docker network disconnect "$PEER" "$M1"
docker exec "$M2" /usr/local/bin/etcdctl \
  --endpoints=http://127.0.0.1:2379 --command-timeout=8s put foo v2
docker network connect --ip "$ORIGINAL_PEER_IP" --alias "$M1" "$PEER" "$M1"
docker exec "$M2" /usr/local/bin/etcdctl \
  --endpoints=http://127.0.0.1:2379 --command-timeout=8s put foo v3
```

There was a three-second orchestration pause between disconnect and the v2 put. Before healing, the harness waited for the direct watcher to exit, verified its no-leader error, then observed for a further five seconds and verified that the proxied watcher was still running without v2. After healing, it waited for v2 before issuing v3. These waits define the test schedule; they are not service-level latency bounds.

**High confidence in the recorded finite observation (2026-11-01; not rerun):** The majority put succeeded. The direct watcher exited with code 5 and printed `watch was canceled (etcdserver: no leader)`. The proxied watcher stayed open through the observation period. After healing it emitted v2 and then v3. Relevant event outputs were:

```text
direct: PUT foo v1; then no-leader cancellation and exit
proxy:  PUT foo v1; silence during partition; PUT foo v2 after heal; PUT foo v3
```

**Limits:** This confirms the metadata-related proxy discrepancy under a peer partition with healthy client transport. It does not establish permanent omission, cover a dead upstream broadcast, measure failure-detection latency, or test multiple proxy endpoints. No claim depends on the peer analysis's own runtime observations.

### C.2 Future watch suppresses requested progress only within its stream

**Topology:** One fresh standalone etcd container from the same image, with two independent `etcdctl` processes inside it. Each process used its own client/stream; all watched keys were otherwise quiet. The actual command forms were:

```sh
docker run -d --name "$NODE" "$IMAGE" /usr/local/bin/etcd --log-level=error
docker exec "$NODE" /usr/local/bin/etcdctl --command-timeout=1s endpoint health
docker exec -i "$NODE" /usr/local/bin/etcdctl watch --interactive
```

After health succeeded, the first interactive process received `watch foo` followed by `progress`. It printed `progress notify: 1`. The harness then sent `watch future --rev=1000000` and `progress` on that same process, leaving the future watch open. A second independent interactive process received `watch foo` and `progress`; it also printed `progress notify: 1`.

The harness next ran:

```sh
docker exec "$NODE" /usr/local/bin/etcdctl put foo event-after-future-watch
```

Both processes printed that put. Five more `progress` commands, spaced 200 ms apart, were sent to the first process; its output still contained only the original progress notification. Thus an ordinary watch on that stream could deliver events while requested progress remained suppressed by its future watch, and an independent stream on the same member could still receive progress.

**Confidence and limits:** High confidence for this finite observation plus the explicit source predicate in §3.2. The test does not infer infinite silence from a short wait: that follows conditionally from retaining the future watch while the member remains below its start revision. It does not measure periodic notification timing, metadata mismatch, overload, or catch-up speed. Both interactive clients set matching leader-required metadata for watch and progress (`etcdctl/ctlv3/command/watch_command.go:130 @ v3.7.3`, `:163`).

## Appendix D. Peer-review decisions

The peer files were useful prompts, not evidence substitutes. The comparisons below refer to their pre-review snapshots in project commit `98a5d32e9d44f6082132aeff1b86acbd832eceb3`, read from the initially clean working tree. Both peer files were edited concurrently later in this review; those edits were left untouched. This table records the material decisions about the reviewed snapshots, rather than asserting that later revisions retain the same claims.

### D.1 Included or qualified findings

| Lead from the peer analyses | Decision after independent checking | Evidence and qualification in this report |
|---|---|---|
| Claude: standalone proxy ignores `WithRequireLeader`; adapter tests can hide it | **Include.** Source-verified and independently reproduced. | §4.1 and §C.1. The experiment shows missing failure notification followed by recovery, not permanent lost history. |
| Antigravity: the proxy does not handle explicit progress requests | **Include.** The current handler confirms the missing case. | §4.1; #19710 is closed, but the capability is still absent. The peer's quoted TODO is not the actual current default-branch text. |
| Both: future/slow watchers can suppress explicit progress; context mismatch can misroute it | **Include with corrected scope.** Same-stream coupling and metadata routing are real. | §3.2 and §C.2. Periodic progress checks one watch at a time. |
| Both: Go revision-zero creation has a registration/header gap; terminal `ErrNoLeader` discards the private start cursor | **Include, with separate verdicts.** The first is an untested continuity schedule; the second is an application restart trap. | §3.1. Lack of cluster-linearizable “now” is documented. Replaying an initial boundary event is not automatically a duplicate or order violation. |
| Antigravity: rejected `Created` plus terminal failure can deadlock cleanup | **Include as a current source-derived candidate.** Trace the actual closing-set bookkeeping, not its mismatched code excerpt. | §3.3. #17622 was not merged; a deterministic current test remains necessary. |
| Claude: a slow Go application consumer necessarily delays all watches sharing its stream | **Qualify.** Per-watch unbounded buffering can keep dispatch running. | §4.5. Memory growth and shared transport/control-path risks remain; they are not identical to direct application backpressure. |
| Both: more historical failures around future revisions, compaction, restart, and multi-member resume | **Include verified mechanisms and fixes.** | §B.6–§B.7 includes actual commits and containing release tags. §B.8 retains #20573 without inventing a fix or a current reproduction. |
| Claude: current Kubernetes cache reads have explicit freshness waiting and fallback; zero revision permits stale results | **Include with scope.** Re-read the selected v1.37.2 code. | §5.2 distinguishes the cache-read protocol from a general watch-delivery or total request deadline. |
| Both: jetcd scheduled-resume failure and weak restart-test coverage | **Include with trigger qualification.** | §6.3–§6.4. A thrown scheduled task differs from an ordinary asynchronous network error. The default launcher version does not prove every CI job's image. |
| Antigravity: missing jetcd fragmentation proves failure above etcd's request-size limit | **Include the API gap; reject the asserted threshold.** | §6.4. Server send behavior and configurable client receive-message limits must be distinguished. |
| Both: Chubby ties cache validity to session maintenance, while etcd offers stronger retained event replay | **Include the concrete protocol comparison, not a universal ranking.** | §8 adds the invalidation-acknowledgment barrier and preserves Chubby's own failover-loss qualification. |

### D.2 Rejected conclusions

| Lead from the peer analyses | Decision after independent checking | Evidence and qualification in this report |
|---|---|---|
| Antigravity: every member watcher is scanned by `progressIfSync`, so one future/unsynced/victim watch disables all server progress | **Reject.** Its purported function body/signature does not match v3.7.3. | The actual function takes a caller-supplied watcher map. An independent stream progressed in the experiment. Do not confuse this with shared-lock contention. |
| Antigravity: terminal `ErrNoLeader` itself violates promised transparent failover; round-robin always loses a fixed fraction of retries | **Reject those conclusions.** Terminal behavior is explicitly documented, and no fixed selection rate was established. | §2 and §3.1. Immediate admission failure during election is real, but does not prove a generally broken retry loop. |
| Both: #20229 introduced the lock inversion and snapshot send proves production deadlock | **Do not adopt that causal claim.** The old function already called `s.rev()` under the watcher lock; the claimed concurrent snapshot caller remains unproved. | §4.4 preserves the API-level inversion and its exact missing reachability condition. |
| Antigravity: jetcd's progress cursor necessarily duplicates event R after progress at R | **Reject for the stated schedule.** `max(R+1, R)` remains R+1. | §6.4 explains why the normal truthful-watermark path does not duplicate a delivered event. |
| Antigravity: jetcd must use `Math.max` on historical event cursors; a disconnect after any partial replay still skips the whole old interval | **Reject.** The event branch corrects the premature creation cursor to the last event plus one. | §6.2 narrows the real failure window. Keeping the higher creation cursor would worsen it. |
| Antigravity: `PrevKv` is unconditionally guaranteed, delayed/absent progress violates bookmark truthfulness, or fragmentation permits duplicate events | **Reject.** These conflate distinct protocol properties or contradict the protobuf. | §1.2 and §4.8 retain the explicit previous-value compaction exception; §3.2 separates response arrival from watermark safety; §6.4 preserves uniqueness. |

No production-frequency estimate, performance number, universal election-recovery bound, or proof of complete watch correctness is adopted from the peers. The added source findings remain version-pinned, and the two independent experiments establish only their explicitly recorded conditions.

### D.3 Follow-up on liveness and WithRequireLeader

The 2026-11-03 peer update correctly identifies official documentation claiming eventual consistency and emphasizes that `WithRequireLeader` cannot guarantee it. Its apply-stall concern prompted the additional stable-source trace in §2.6, which shows that even the local leader ID can remain stale behind blocked WAL persistence. The earlier wording about handling direct-member isolation was too broad: the experiment established a terminal signal under its particular conditions, not an end-to-end convergence guarantee.

The valid core criticism does not require adopting every supporting assertion. Current source and §C.2 still contradict member-wide progress suppression by a single future watch; normal termination and the 250 ms error drop leave observable closure; a large finite scan is not proof of permanent deadlock; and eventual convergence of current state does not require recovering every intermediate transition. The default-watch, direct flagged-watch, proxy, and client-state-machine cases retain their distinct prerequisites and confidence levels. Independent quorum-derived checks can expose lack of progress even though the affected stream supplies no reliable failure signal.

## Conclusions

**High confidence:** Eventual consistency requires eventual convergence under its stated failure assumptions. Correctly ordered delivery of a permanently stale prefix is insufficient. The upstream eventual-consistency description and the documented possibility of nonarrival leave the failure assumptions and recovery obligation inadequately specified (§1.5). This is a liveness concern even for applications that never require linearizable cached reads.

**High confidence in the mechanism:** `WithRequireLeader` does not close this gap. A direct watch can remain open and stale on a partially failed member while the client and a majority remain functional (§2.6). Its local leader-loss check is neither a progress check nor automatic recovery. The storage-stall counterexample is source-derived, with no new runtime reproduction. A consumer cannot rely on the option, transport health, or silence alone to know that its subscription will converge.

**High confidence:** The evidence does not establish that keeping watch subscriptions outside Raft inherently makes leader failover incorrect. Committed history can support local subscriptions and revision-based replay. Correct KV replication, however, does not prove correct delivery, error propagation, or application state.

**High confidence:** Production consumers need an explicit policy for progress and uncertainty. An open watch, transport keepalive, local progress, and `WithRequireLeader` each provide limited information. Applications should preserve a processed revision, supervise progress where silent stalls are possible, treat unexpected closure as subscription loss, and rebuild after a replay discontinuity. A quorum-derived freshness target for each read is a separate requirement when decisions need current data (§7).

**High confidence in the verified mechanisms; Moderate in the untested scenarios:** v3.7.3 and jetcd 0.8.7 still have material defects and recovery risks. The direct-versus-proxy partition experiment and the progress-scope experiment confirm two specific behaviors. The final-error drop, cursor races, rejected-creation teardown, and scheduled-resume exception require their own controlled tests before their operational reach can be established. Historical fixes must remain distinct from these current findings.

**High confidence — version and scope:** Jepsen's earlier positive watch evidence does not displace the later source findings (§1.6). Its jetcd KV retry defect is fixed by default in 0.8.7, while watch resumption still uses separate, unchanged code (§6.5). Neither the old all-retries workaround nor the newly merged unary backoff change establishes watch continuity or progress.

**Moderate confidence — architectural assessment and supervision recommendation:** The design concern supported by this analysis is the gap between reliable stored history, eventual observation, and a client's knowledge that its derived state remains usable. Chubby makes the cache-validity transition part of its session and invalidation protocol; etcd consumers assemble their recovery and validity policy from revision tracking, replay, reads, deadlines, and reconstruction. Watches retain useful push and replay capabilities, but that does not eliminate the need for independent progress supervision where automatic recovery can stall. The remaining work is to test the identified failure schedules and characterize their impact on the intended production topology. No benchmark result or production-frequency estimate follows from this audit.
