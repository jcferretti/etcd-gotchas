# Watchers: API promises, leader failover, and the client states that break them (Claude)

**Versions analyzed:**

| Component | Version | Notes |
|---|---|---|
| etcd | `v3.7.3` = `0a173d0410a9` (local clone `~/github/etcd`) | All `path:line` citations are at this tag unless stated otherwise. `origin/main` @ `1573d5941` (2026-10-28) and `origin/release-*` branches were consulted only for fix and backport status. |
| jetcd | `jetcd-0.8.7` = `23b61f1499a5` (local clone `~/github/jetcd`) | Files read: `WatchImpl.java`, `WatchOption.java`, `ClientBuilder.java`, `ClientConnectionManager.java`, `Errors.java`, `Util.java`, `watch/WatchResponse.java`. |
| Raft library | `go.etcd.io/raft/v3` `v3.7.0` | Only `raft.go`, fetched read-only from GitHub, to confirm when a member considers itself leaderless. |
| Kubernetes | `v1.37.2`, selected files fetched read-only from GitHub | `storage/etcd3/{watcher.go,store.go,event.go}`; `storage/cacher/{cacher.go,watch_cache.go,delegator.go,progress/watch_progress.go}`; `storage/storagebackend/factory/etcd3.go`; `features/kube_features.go` (all under `staging/src/k8s.io/apiserver/pkg/`). `etcd3/watcher.go` was re-fetched raw on 2026-11-01. |
| etcd user docs | `etcd-io/website` `main`, fetched 2026-10-31 | `content/en/docs/v3.7/learning/api_guarantees.md` and `content/en/docs/v3.7/learning/design-client.md`. |
| KEP-2340 | `kubernetes/enhancements` `master`, fetched 2026-10-31 | "Consistent reads from cache". |
| Jepsen | [etcd 3.4.3 report](https://jepsen.io/analyses/etcd-3.4.3) (2020-01-30); test source `jepsen-io/etcd` @ `16149cb` (`src/jepsen/etcd/{watch,client}.clj`, `project.clj`); fetched 2026-11-09 | §2.5. The test used etcd 3.4.3 and jetcd 0.4.1. |
| grpc-java | `grpc-core` 1.76.2 (local Gradle cache) | One constant read with `javap` (§10.4). jetcd 0.8.7 declares grpc 1.82.0. |

**Path abbreviations** used in citations:

| Short form | Full path |
|---|---|
| `v3rpc/` | `server/etcdserver/api/v3rpc/` (the server's watch, interceptor and gRPC setup code) |
| `etcdserver/` | `server/etcdserver/` |
| `mvcc/` | `server/storage/mvcc/` (at v3.7.3; at v3.4.0 the same package lived at the repository root, `mvcc/`) |
| `grpcproxy/` | `server/proxy/grpcproxy/` |
| `client/v3/` | unchanged (the Go client) |
| `jetcd/` | `jetcd-core/src/main/java/io/etcd/jetcd/` |
| `apiserver/` | Kubernetes `staging/src/k8s.io/apiserver/pkg/` |

**Method:**
- Code reading of the checkouts and the fetched files.
- The GitHub REST API, for issue and PR bodies and comment threads in `etcd-io/etcd`, `etcd-io/jetcd` and `kubernetes/kubernetes`. A few threads could only be read through a tool that summarizes pages with a smaller model; claims from them are marked "(summarized fetch)" and are weaker evidence (details in Appendix D).
- `git tag --contains` on merge and backport commits, for every fix version stated as "verified".
- **One Docker functional test** (2026-10-31; Appendix C): the official `v3.7.3` image, a 3-member cluster and `etcd grpc-proxy` on this host, run twice. It checks behaviour only; it is not a benchmark (AGENTS.md §7). The 2026-11-01 revisions ran no tests.
- **Jepsen pass (2026-11-09):** the Jepsen etcd 3.4.3 report and its watch test source read in full, its watch issue (#11496) traced through the GitHub API, and the fix checked in the v3.7 docs and at `v3.7.3` (§2.5). The jetcd 0.8.2 report (2024) did not test watches. Nothing run.
- Confidence tags (High / Moderate / Low) apply to the claim they follow, not to the whole document. "Inferred" means derived from code reading only.

**Upstream references:** the most important are:
- etcd: [#21813](https://github.com/etcd-io/etcd/issues/21813), [#22184](https://github.com/etcd-io/etcd/issues/22184), [#22673](https://github.com/etcd-io/etcd/issues/22673), [#17611](https://github.com/etcd-io/etcd/issues/17611), [#19710](https://github.com/etcd-io/etcd/issues/19710), [#8980](https://github.com/etcd-io/etcd/issues/8980), [#15271](https://github.com/etcd-io/etcd/issues/15271), [#17529](https://github.com/etcd-io/etcd/issues/17529), [#20221](https://github.com/etcd-io/etcd/issues/20221);
- jetcd: [#1379](https://github.com/etcd-io/jetcd/issues/1379), [#946](https://github.com/etcd-io/jetcd/issues/946);
- Jepsen: [etcd 3.4.3](https://jepsen.io/analyses/etcd-3.4.3) and its watch issue [#11496](https://github.com/etcd-io/etcd/issues/11496);
- Kubernetes: [#59848](https://github.com/kubernetes/kubernetes/issues/59848), [#123072](https://github.com/kubernetes/kubernetes/issues/123072), [KEP-2340](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2340-Consistent-reads-from-cache).

Appendix D has the full list. Appendix A lists the fixed watch-guarantee violations, Appendix B the open upstream items.

**Provenance:**
- First written 2026-10-31, independently of the other watch analyses in this directory.
- A second pass the same day examined `WithRequireLeader` in depth and ran the Docker test.
- Revised 2026-11-01 using the Codex and Antigravity watch analyses (`watchers-codex.md`, `watchers-agy.md`) as leads. Every adopted item was re-verified, so this file does not depend on them.
- Restructured for readability the same day.
- Extended 2026-11-09 with the Jepsen etcd 3.4.3 results (§2.5), at the user's request.

Appendix E has the revision history, the use made of the peer analyses, and a map from old section numbers to new ones.

## Scope

**Purpose.** To test the claim that etcd watches "do not go through Raft" and therefore cannot stay correct across leader failover, and to identify the client states in which watch promises, documented or assumed, actually fail.

It covers the etcd v3.7.3 server, the Go client, jetcd 0.8.7, the etcd gRPC proxy, and how Kubernetes uses watches. Google's Chubby is the design reference. Performance is covered only where it changes what a client sees; nothing was measured.

**Terms used throughout**
- **Revision:** etcd's cluster-wide logical clock. Every committed modification gets one, assigned identically on every member as it applies the Raft log.
- **Serving member:** the etcd member that holds the client's watch gRPC stream.
- **"Now" watch:** a watch created without a start revision (`start_revision = 0`). It starts after the serving member's current revision. A **historical** watch starts below that revision; a **future** watch starts above it. Revision 0 never means "from the beginning": the first revision is 1 (Jepsen, #11496; §2.5).
- **`Created` response:** the server's acknowledgement that a watch was registered. Its header carries the serving member's current revision.
- **Progress notification** (also *bookmark*): a watch response with no events. It states that all events up to its header revision have been delivered on that stream.
- **Synced / unsynced watcher:** a watcher that is caught up with the member's store, or one that the server is still feeding from history in batches.
- **`WithRequireLeader`:** a client option. It asks the serving member to refuse or cancel streams while it has no Raft leader (§6).
- **Halt error:** an error the client libraries treat as fatal rather than retrying: any gRPC status other than `Unavailable` or `Internal`, plus `ErrNoLeader` for watches.
- **Safety guarantee, eventual delivery, timeliness:** the three kinds of promise this file distinguishes (revised 2026-11-03; earlier versions split promises into "documented" and "intuitive").
  - A *safety guarantee* is one of the six in §2.1: what arrives, and in what order.
  - *Eventual delivery* means that every event a watch matches eventually arrives while the cluster keeps its quorum. etcd claims it by calling watches eventually consistent (`tests/robustness/README.md:102`, `:111`); its API guarantees page disclaims it (§2.2). This file counts it as a promise.
  - *Timeliness* is a bound on delay. Nothing in etcd promises it.
- **Broadcast:** in the gRPC proxy, one upstream watch shared by all client watches on the same key range (§9.2).

## Summary

**Question.** Do etcd watches break their promises on leader failover because they bypass Raft? If not, what does break them, and in which client states?

**Short answer.**
- **The premise is right about liveness and wrong about safety** (revised 2026-11-03; §3 explains the change).
  - *Right:* etcd claims eventual consistency for watches, a liveness promise, and watches break it in a cluster that keeps its quorum. By default, a watch served by a member cut off from the quorum stays open and silent until that member rejoins, which may be never; the deposed leader after a partition-induced failover is one such member. The cause is the one the premise names: the watch path never consults Raft's quorum. It makes no ReadIndex check. Its only server-side liveness check is the opt-in `WithRequireLeader`, which tests the member's own view of whether it has a leader, not whether it applies what the quorum commits. — **High**.
  - *Wrong:* watch events come from Raft-committed entries, applied on each member at the same revision everywhere. A watcher never sees an uncommitted write, a failover cannot retract an event already delivered, and a broken watch can resume on any member. — **High**.
  - *Narrower than stated:* failover is one trigger, not the condition. A cut-off follower, a member that has a leader but does not apply, the gRPC proxy and the client libraries silence watches without any failover. A clean failover, where the old leader crashes and its connections reset or leadership is transferred, costs only a pause on the server side. — **High**.
- **`WithRequireLeader` does not restore eventual delivery** (added 2026-11-03). It turns one case, a member that knows it has lost its leader, into an error after about 3–5 s. A watch that sets it can still go silent indefinitely while the quorum keeps committing:
  - the serving member still reports a leader but does not apply (§5.2; Moderate, inferred). One way this happens is a blocked raft loop, for example a WAL `fdatasync` that does not return: the member's leader view then freezes instead of clearing, and the gRPC server keeps answering keepalive pings, so neither `WithRequireLeader` nor client keepalive ends the watch (§5.2; High for the mechanism, from code; not reproduced);
  - a burst of cancellations freezes the shared stream ([#18879](https://github.com/etcd-io/etcd/issues/18879); §8.1);
  - the Go client deadlocks in teardown when `ErrNoLeader` races a rejected creation ([#17611](https://github.com/etcd-io/etcd/issues/17611), titled upstream "Watch API blocked forever even with WithRequireLeader option"; §10.2), or it drops the `ErrNoLeader` error and closes the channel bare, which an application that follows the documentation reads as its own cancellation (§10.2);
  - the serving process is paused or the connection is dead, unless client keepalive is also on; it is off by default in Go (§5.4);
  - the watch goes through the gRPC proxy, which ignores the option and can attach watches to dead upstream watches (§9).

  A "from now" watch that the option does end also loses, for good, the writes made before the application re-watches (§6.3). Only a client-built freshness check, a linearizable read compared with a progress notification, detects all of these (§13.2, §15.4). — **High** that such paths exist; per-path confidence in §4 and §6.3.
- **What bypasses Raft is freshness and liveness.** The "now" start point, the `Created` revision and every progress notification reflect only the serving member's own applied revision, and no quorum check touches the watch path. The danger is a cut-off or stuck *serving member*, which is as likely to be a follower as a deposed leader. — **High**.
- **The documented contract permits silence; etcd's consistency claim does not.** The API guarantees page promises ordering and completeness, not delivery: "events in unhealthy clusters might never arrive". The main production failure, a watch that silently stops while the quorum keeps working, therefore breaks no guarantee on that page, but it does break eventual consistency. "Unhealthy" is undefined there, and every silent-watch state in this file occurs with the quorum intact, so the page's contract is safety-only (§2.2 caveat; full analysis in [`etcd-issues-and-mitigations-claude.md`](etcd-issues-and-mitigations-claude.md) §3). — **High**.
- **Ordering and completeness have been broken repeatedly**, almost always where member-local watch state meets replicated state: snapshot restore, compaction, and resuming on another member. These are probably the failover bugs the premise remembers. All known cases are fixed in v3.7.0, with backports to 3.4–3.6. — **High**.
- **At v3.7.3, most remaining open defects that silence watches sit in the client libraries and the gRPC proxy**, not in the direct server path. The direct path's silent states, a cut-off or stalled serving member, are by design (§5). — **Moderate**.
- **Design verdict.** The weakness is the part of the watch path that bypasses Raft: a watch stream has no freshness or liveness coupling to Raft progress, and the defaults leave even the existing signals off. The part that goes through Raft, replay of committed history, is what keeps the safety guarantees. — **Moderate**.

**Key findings at v3.7.3**

| # | Finding | Status | Confidence | § |
|---|---|---|---|---|
| 1 | A serving member cut off from the quorum keeps the watch open and silent until the partition heals, unless the client set `WithRequireLeader`, which is off by default in Go and jetcd | By design | High | 5.1 |
| 2 | `WithRequireLeader` cancels such a watch after about 3–5 s (measured 3.3 s and 4.3 s). But it misses a member that has a leader and does not apply, ends the watch instead of resuming it, and throws away the resume point. Watches that set it can still go silent indefinitely (Summary bullet above) | Partial remedy, by design | High (code); Moderate (apply-stall case) | 6 |
| 3 | A member that still sees a leader but has stopped applying gives watch-only clients no signal at all. The #22184 lock inversion is one candidate cause; no production trigger is confirmed. A blocked raft loop (for example a WAL `fdatasync` that does not return) freezes the member's leader view, so `WithRequireLeader` and client keepalive both miss it (added 2026-11-03) | Open | Moderate; Low–Moderate for #22184; High (code) for the frozen leader view, not reproduced | 5.2, 5.3 |
| 4 | A paused member or dead connection hangs a Go watch indefinitely: the Go client has no keepalive by default and retries forever | By design | High | 5.4 |
| 5 | The gRPC proxy ignores `WithRequireLeader` (reproduced, unreported), reuses dead upstream watches (#21813), swallows upstream errors, and drops `RequestProgress` (#19710) | Open | High (code; first item reproduced); Moderate (how often #21813 occurs) | 9 |
| 6 | The Go client drops its final error after 250 ms if the application is slow; the bare close looks like the caller's own cancellation | Open; no upstream report found | High (code); not reproduced | 10.2 |
| 7 | A narrow race deadlocks the Go client's stream teardown; later `Watch()` calls then block (#17611, titled "Watch API blocked forever even with WithRequireLeader option"; `ErrNoLeader` is one of the fatal errors that open the race) | Closed upstream without a fix | Moderate | 10.2 |
| 8 | jetcd skips historical events if the stream breaks after `Created` and before the first batch | Open; no upstream report found | Moderate (mechanism); Low (frequency) | 10.3 |
| 9 | jetcd ends a watch whose response exceeds 4 MiB, and re-watching fails the same way | Open (seen in jetcd#1379) | High (mechanism); Moderate (reachability) | 10.4 |
| 10 | Reusing an explicit watch ID can deliver a cancelled watch's events to the new watch (#22673) | Open; raw-gRPC clients only | Moderate | 8.2 |
| 11 | Far-behind or congested watchers: multi-second delays, write stalls (#22264, #16839), and streams frozen by cancellation bursts (#18879) | Open | Moderate | 8.1 |
| 12 | Historical violations on snapshot restore, compaction and cross-member resume | Fixed (3.4.25 – 3.7.0) | High | 7, App. A |
| 13 | Kubernetes defends itself explicitly; residual gaps are apply stalls, paused members, the proxy, and `resourceVersion=0` reads | — | High (code); Moderate (residual risk) | 11 |

**What protects applications.**
- Treat every unexpected channel close or error as the end of the watch, and resume from a revision the application itself has stored, never from "now".
- Set `WithRequireLeader` and client keepalive.
- Where freshness matters, check periodically that the watch has reached a revision obtained by a linearizable read.
- Connect directly to members rather than through the gRPC proxy.

With these measures, the silent-loss paths left on a current release are the jetcd resume skip (finding 8) and watch-ID reuse by raw-gRPC clients (finding 10) (§13, §15).

---

## 1. How watches work at v3.7.3

A watch is a subscription held by one member. Its *events* come from the replicated log; its *start point* and *freshness signals* come from that member alone. Five facts, all verified in code, establish this.

1. **Creation is member-local.** The server registers a watch directly with its local store. There is no ReadIndex or other linearizable barrier anywhere in the watch server or the MVCC watch store. The store picks the start point from its *own* current revision, so "now" means the serving member's applied revision.
   - `v3rpc/watch.go:340`: `recvLoop` calls `sws.watchStream.Watch(...)` directly. `grep` for `ReadIndex` and `linearizableRead` finds nothing in either file.
   - `mvcc/watchable_store.go:139-151`: `synced := startRev > s.store.currentRev || startRev == 0`; a "now" watch gets `minRev = currentRev + 1`.
   - `api/etcdserverpb/rpc.proto:784-785`: "No start_revision is 'now'".
2. **Events come from applying Raft-committed entries.** Every event a watcher receives corresponds to a committed log entry, applied at the same revision on every member. Lease expiry reaches watchers the same way: the leader proposes `LeaseRevoke` through Raft and each member deletes the keys.
   - `mvcc/watchable_store_txn.go:22-46`: every write transaction ends in `watchableStoreTxnWrite.End()`, which calls `s.notify(rev, evs)`.
   - `etcdserver/server.go:843-845`: write transactions run only when the apply goroutine applies committed entries, scheduled from the raft loop.
   - `mvcc/watchable_store.go:105-108`: lease revocations delete through the watchable store.
3. **Catching up is local and replayable.** A watcher that is behind is fed every 100 ms from the member's own history. A watch that starts below the compaction point is cancelled with a `CompactRevision`.
   - `mvcc/watchable_store.go:223-255`, `:346-414` (`rangeEvents`).
   - `mvcc/watcher_group.go:255-262`; `v3rpc/watch.go:459-466`.
4. **Freshness signals are local.** The `Created` header and every progress notification carry the member's own current revision. A progress notification is sent only if the watchers are synced, and the proto scopes its guarantee to one stream: "All future events received **in this stream** are guaranteed to have a higher revision".
   - `v3rpc/watch.go:357-358` (`Created` header: `sws.watchStream.Rev()`).
   - `mvcc/watchable_store.go:515-540` (progress: `s.rev()`).
   - `api/etcdserverpb/rpc.proto:424-429`.
5. **Leader state reaches a watch only through the opt-in `WithRequireLeader` flag.** The stream interceptor rejects new flagged streams while the member has no leader, and a monitor goroutine cancels flagged streams after three leaderless samples (§6.1). Without the flag, a stream is never told about leader loss.
   - `v3rpc/interceptor.go:38`, `:239-258`, `:315-354`.

## 2. The contract

### 2.1 Documented guarantees

Source: the v3.7 "API guarantees" page, "Watch APIs" section (`etcd-io/website` `content/en/docs/v3.7/learning/api_guarantees.md`, fetched 2026-10-31). The robustness checker uses the same wording (`tests/robustness/validate/watch.go:31-39`).

| Guarantee | Meaning |
|---|---|
| Ordered | Events are ordered by revision. |
| Unique | An event never appears twice. |
| Reliable | "a sequence of events will never drop any subsequence of events within the available history window". |
| Atomic | A revision's events are never split. |
| Resumable | A watch broken at revision *r* can be re-established at *r+1*, within the history window. |
| Bookmarkable | A progress notification guarantees that all events up to its revision have been delivered. |

Every one of these says *what* is delivered and in what order, given that something is delivered.

### 2.2 What is explicitly disclaimed

Delivery time and linearizability. From the same page (fetched 2026-10-31):

> "Calling KV API will take an immediate effect, while Watch API will return with some unbounded delay. In correctly working etcd cluster you should expect to see watch events to appear with 10ms delay after them happening. **However, there is no limit and events in unhealthy clusters might never arrive.**"

> "etcd does not ensure linearizability for watch operations. Users are expected to verify the revision of watch events to ensure correct ordering with other operations."

**Caveat on "unhealthy" (added 2026-11-02).** Read the disclaimer, and every
statement in this file that the contract "permits silence", as describing
etcd's documented position, not a defensible contract.
- **The term is undefined, and etcd uses it loosely.** The page does not say
  what "unhealthy" means. `etcdctl endpoint health` reports "unhealthy
  cluster" when any one endpoint fails, whether or not a quorum remains
  (`etcdctl/ctlv3/command/ep_command.go:166-178`). The server uses the same
  error for unrelated reconfiguration and leadership-transfer checks
  (`etcdserver/server.go:1301-1303`, `:1427-1434`).
- **It only has effect with a quorum.** A cluster without quorum commits
  nothing, so the disclaimer adds nothing there. It matters only for a
  cluster that keeps its quorum while some component fails: the serving
  member, the connection, the proxy or the client library. That is where
  every silent-watch state in §4 occurs, and other members could serve the
  watch.
- **The contract is therefore safety-only.** It constrains what arrives,
  never whether anything arrives.
- **It contradicts etcd's own description.** etcd calls watches eventually
  consistent (`tests/robustness/README.md:111`), but eventual consistency is
  a liveness guarantee, and "might never arrive" is its negation. Some
  silent paths (#21813, #17611, an unnoticed dead connection) persist after
  every fault has healed.

The full analysis, including what a useful liveness contract would say, is
in [`etcd-issues-and-mitigations-claude.md`](etcd-issues-and-mitigations-claude.md)
§3.

Two other documents say the same about partitions:
- The client design document (`design-client.md:136`, v3.7 docs): "Client-side keepalive ping still does not reason about network partitions. Streaming request may get stuck with a partitioned node."
- The Go client's interface comment (`client/v3/watch.go:63-92`): "In order to prevent a watch stream being stuck in a partitioned node, make sure to wrap context with 'WithRequireLeader'. Otherwise ... watch will retry on other recoverable errors forever." It also carries a TODO, still open since v3.4, to "configure watch retry policy, limit maximum retry number (see #8980)".

### 2.3 Implicit promises in the proto and the client

- **The start revision is inclusive, and "now" means the serving member's now** (`api/etcdserverpb/rpc.proto:784-785`). [#19487](https://github.com/etcd-io/etcd/issues/19487) was accepted as a bug because a response came back *below* the requested start revision.
- **The previous value (`prev_kv`) is best-effort.** "If the previous KV is already compacted, nothing will be returned" (`rpc.proto:805-807`). The server fetches it *at send time*, with a `Range` at `ModRevision-1`, and silently omits it on error (`v3rpc/watch.go:448-457`). [#10681](https://github.com/etcd-io/etcd/issues/10681), open since 2019, calls this "surprising behavior".
- **Compaction is signalled, never silent:** "The client should treat the watcher as canceled" (`rpc.proto:850-858`). Several fixed bugs violated exactly this (§7.3).
- **Periodic progress notifications are off unless requested per watch**, and the default server interval is **10 minutes** (`v3rpc/watch.go:94`; flag `--watch-progress-notify-interval`). Without `RequestProgress`, a quiet watch has no heartbeat: "no changes" looks the same as "stalled" for up to 10 minutes.

### 2.4 What the contract amounts to

A watch promises a gap-free, ordered **suffix of the replicated history**, starting from a revision the client chose or from the serving member's local "now". It promises nothing about *how far behind the cluster's current revision that suffix is*. A client can learn its lag only by comparing against a linearizable read, which goes through ReadIndex, i.e. through Raft.

So watches keep their ordering and completeness guarantees across failover. What breaks is eventual delivery, and with it the client's ability to know whether it is up to date: a stream whose member stops advancing stays open and silent (§4, §5).

Three consequences for how clients and testers should reason (raised by the Codex analysis; I agree):
- **Revision gaps are not evidence of loss.** Writes to unrelated keys also advance the global revision. Showing that a matching event was omitted needs an independent record of the expected history.
- **The library's cursor is not the application's cursor.** The resume revision a client library keeps records what the *library* received, not what the application has applied. A process that crashes between the two must resume from a cursor stored together with its own state, or relist.
- **A relist restores current state, not the transitions in between.** After a compaction error, an application that needs every intermediate change has no recovery path; one that needs only the current state does.

### 2.5 What Jepsen tested (etcd 3.4.3, 2020)

Added 2026-11-09. Sources: the [report](https://jepsen.io/analyses/etcd-3.4.3)
and its watch workload at `jepsen-io/etcd` @ `16149cb`
(`src/jepsen/etcd/watch.clj`, `client.clj`). Jepsen has published no later
watch test; its 2024 jetcd report did not test watches.

**Result.** "Watches appear correct, at least over single keys. So long as
compaction does not destroy historical data while a watch isn't running,
watches appear to deliver every update to a key in order."
- The faults: partitions (single nodes, majority and minority,
  non-transitive), crashes and pauses (including of the leader), clock skew
  up to hundreds of seconds, and membership changes.
- This is independent evidence for the safety half of §3, at 3.4.3.

— **High** (for what was observed).

**What the test design could and could not see** (from the source).
- **One key, one member per client.** Writers set one key to unique
  integers. Each client had a single endpoint (`client.clj:172-179`) and
  used jetcd 0.4.1 (`project.clj`).
- **Watches were rotated.** Each watch ran for a random 0–5 s, then was
  closed and re-created from the last revision seen + 1
  (`watch.clj:142-143`, `:199-204`, `:228`), on the same member. A watch
  that went silent was closed within 5 s anyway, and its successor resumed
  from the same revision, so silence during a fault left no trace in what
  was checked: the test could not tell it from slow delivery.
- **The check ran after the faults.** In the final phase, after the cluster
  was healed and a 10 s wait (`src/jepsen/etcd.clj:112-116`), every watcher
  kept re-watching until all reached the same revision, and the checker then
  compared their logs (`watch.clj:235-255`, `:316-345`). That tests that
  history is complete and ordered once the cluster is healthy. It does not
  test that a watch keeps delivering while its member is cut off.
- **The cursor came from events.** The harness advances its cursor to the
  highest `mod_revision` received. A comment records that using the
  response header's revision instead "sometimes" produced non-monotonic
  revisions (`watch.clj:153-157`).
  - Jepsen gives no cause. The likely one is the mechanism behind §10.3:
    while a watcher catches up, each batch's header carries the member's
    current revision, ahead of the batch's own events
    (`mvcc/watchable_store.go:388`; §1 fact 4). A cursor taken from that
    header jumps past events still to come. — **Moderate** (cause
    inferred).
- **Compaction was excluded, and snapshot restore was probably rare.**
  Restoring a snapshot on a lagging member requires the leader to have
  compacted its Raft log. At 3.4's defaults that happens only every 100,000
  applied entries, keeping 5,000 for slow followers
  (`etcdserver/server.go:69`, `:76 @ v3.4.3`). Whether Jepsen's runs got
  that far is unknown.

**Consequences for this file.** — **Moderate**.
- **Jepsen's result and the later bugs are compatible.** The violations
  fixed later (Appendix A) sit on paths this workload rarely or never
  exercised: compaction (#17529, #18089, #19179), and snapshot restore or
  cross-member resume (#15271, #19487 / #20221). Appendix A dates them to
  3.4 releases; whether each was present in 3.4.3 was not checked.
- **It says nothing about liveness** (§3). Its rotation is itself a
  mitigation, a weak form of the freshness monitor in §13.2 item 8.

**Revision 0** ([#11496](https://github.com/etcd-io/etcd/issues/11496),
filed by Jepsen 2020-01-06). — **High** (docs and code read).
- **The finding.** A watch from revision 0 starts at the serving member's
  current revision + 1, not at the first revision, and this was
  undocumented. Jepsen's harness says the same in a comment: "Definitely
  don't pass 0, or you'll get wherever it is currently" (`watch.clj:142`).
- **The fix, in docs only.** PR [#11559](https://github.com/etcd-io/etcd/pull/11559)
  (merged 2020-01-27):
  - the data model page says "the initial revision is 1" (v3.7 docs,
    `learning/data_model.md:15`, `etcd-io/website` @ `41e5cd6ab9ef`);
  - the Go client's `Watch` comment says that a revision of 0 or unspecified
    returns "watch events that happen after the server receives the watch
    request" (`client/v3/watch.go:68-69`).
- **Still imprecise.** That comment is looser than the code. "After" means
  after the serving member's applied revision, which on a lagging member
  precedes the request in real time (§1 fact 1). The v3.7 API reference
  describes only the unset case: "If not given, it will stream events
  following the revision of the watch creation response header revision"
  (`learning/api.md:412`).

## 3. The premise, tested

The premise: "watches don't go through Raft, so they break on leader failover".

**The correct part (High).** Watch freshness and liveness are not mediated by Raft.
- A watch is a local subscription to one member's apply stream. The watch path makes no ReadIndex or quorum check, and it checks for a leader only under the opt-in `WithRequireLeader` (§6). Nothing proves to the client that the stream keeps up with the quorum, and nothing ends it when it stops: a member cut off from the quorum keeps its watches open and silent (§5.1).
- That breaks a promise etcd makes: "Etcd provides strict serializability for KV operations and eventual consistency for Watch" (`tests/robustness/README.md:111`). The same file defines eventual consistency as a liveness property: components "converge to the same view eventually" (`:102`).
- The failover instance is a deposed leader that keeps its watch clients after the partition that deposed it. The same silence follows from any cut-off or stalled serving member (§5).

**The incorrect part (High).** Safety, and the restriction to failover.
- Leader failover by itself does not threaten watch correctness. Events derive from committed entries, so entries truncated on an old leader are never emitted. Revisions are identical on every member, so a client can resume anywhere at `lastRevision+1`.
- Jepsen's etcd 3.4.3 tests agree: under partitions, crashes, pauses, clock skew and membership changes, single-key watches resumed from their last revision delivered every update in order (§2.5).
- On members that stay healthy, a failover shows up only as a pause: nothing commits during the election.
- The member that matters is the *serving* member. A follower cut off by a partition is exactly as dangerous as a deposed leader.

**When the liveness half would fail (High).** Under the classical definition of eventual consistency, which assumes that every replica eventually communicates again, a cut-off member that delivers once it rejoins (§5.1) does satisfy it. That reading does not rescue etcd's claim. etcd's fault model tolerates a minority of members that never return, and a delivery guarantee that waits for them is no guarantee under that model. Some paths also stay silent after every fault has healed (#21813, #17611, an unnoticed dead connection; §2.2 caveat), which breaks eventual consistency even in the classical sense.

**Revision note (2026-11-03).** Earlier versions of this section, the Summary and §15.1 rated the premise "mostly wrong" (High). That rating counted only the six safety guarantees of §2.1 as promises, and it filed delivery under "intuitive" expectations, although this section already quoted etcd's eventual-consistency claim. Counting eventual delivery as a promise, as that claim requires, reverses the verdict on the liveness half.

**Probably what the premise remembers (Moderate).** The watch bugs that did break correctness "after a network partition" all sit on one path: a member that fell behind, often the old leader, downloads a snapshot from the new leader and rebuilds its watchers' state.
- [#8411](https://github.com/etcd-io/etcd/issues/8411) (2017): a member that rejoined never delivered events after restoring a snapshot.
- [#15271](https://github.com/etcd-io/etcd/issues/15271) (2023): "Watch traveling back in time after network partition".
- [#19487](https://github.com/etcd-io/etcd/issues/19487) / [#20221](https://github.com/etcd-io/etcd/issues/20221) (2025): a watch on a future revision received old events after a snapshot restore or a member switch.

These were implementation bugs in `watchableStore.Restore` and in the resynchronization of unsynced watchers, not consequences of bypassing Raft. All are fixed (§7, Appendix A).

## 4. Failure states at a glance

The table maps each client-visible state to the section that analyzes it. The "Promise affected" column uses the three kinds defined under Terms: "Safety" means the state breaks a guarantee in §2.1; "Eventual delivery" means matching events stop arriving while the quorum is intact, which breaks etcd's eventual-consistency claim (§3); "Timeliness" means delay only, which nothing promises. Two further labels name narrower signals: the progress signal (§2.3) and documented client-library behaviour. (Relabelled 2026-11-03; earlier versions used "documented" and "intuitive".)

**A. Serving member cut off or stuck (§5, §6)**

| State | Promise affected | Visible to client? | Versions | Confidence | § |
|---|---|---|---|---|---|
| Serving member isolated from quorum (incl. deposed leader), no `WithRequireLeader` | Eventual delivery | No; indefinite silence | all | High | 5.1 |
| Same, with `WithRequireLeader` | Eventual delivery, signalled (restored only if the app re-watches) | `ErrNoLeader` after ~3–5 s (measured 3.3 / 4.3 s); app must re-watch | all | High | 5.1, 6 |
| "From now" watch ended by `ErrNoLeader`; app re-watches from "now" | Eventual delivery (writes made during the partition never arrive) | No; writes made during the partition are skipped | all | High (follows from code) | 6.3 |
| Serving member has a leader but does not apply | Eventual delivery | No, even with `WithRequireLeader` | all | Moderate (inferred) | 5.2 |
| Serving process paused or connection black-holed; no client keepalive | Eventual delivery | No (Go default) | all | High | 5.4 |
| Client cannot reach any member | Error signalling only (no member reachable) | No error from the Go client; jetcd calls `onError` per retry | all | High | 5.4 |

**B. Resume, restore and compaction (§7)**

| State | Promise affected | Visible to client? | Versions | Confidence | § |
|---|---|---|---|---|---|
| Transparent resume on another (lagging) member | Safety (Bookmarkable, start revision) | No | fixed 3.5.22 / 3.6.2 / 3.7.0 | High | 7.1 |
| Serving member restores a leader snapshot | Safety (Ordered, Unique, Reliable) | No | fixed (App. A) | High | 7.2 |
| Watch at or below the compaction boundary | Safety (Reliable) | No | fixed 3.4.34+ / 3.5.16+ / 3.5.18 | High | 7.3 |
| ≥512 unsynced watchers, some compacted | Retirement only (repeated cancels) | Yes (noisy) | open [#22294](https://github.com/etcd-io/etcd/issues/22294) | Moderate | 7.3 |
| Cluster restored from snapshot without revision bump | Safety (Reliable, from the client's view) | No | operator choice | High | 7.4 |
| Member with divergent data (v3.5.0–3.5.2) | All | No | fixed 3.5.3 | High | 7.5 |

**C. Delivery path (§8)**

| State | Promise affected | Visible to client? | Versions | Confidence | § |
|---|---|---|---|---|---|
| Congested stream / slow watchers | Timeliness; historically documented ([#17529](https://github.com/etcd-io/etcd/issues/17529)) | Delay only (now) | open performance issues | Moderate | 8.1 |
| Burst of cancellations on a shared stream ([#18879](https://github.com/etcd-io/etcd/issues/18879)) | Eventual delivery (all watches on the stream stop) | No | open; fix PR #21065 open | High (in-process), Moderate (networked) | 8.1 |
| TLS, gRPC and HTTP on one listener, heavy concurrent reads ([#15402](https://github.com/etcd-io/etcd/issues/15402)) | Timeliness (starvation) | Delay only | mitigation since 3.4.25 / 3.5.8 / 3.6.0, opt-in | Low–Moderate for v3.7.3 (untested) | 8.1 |
| Explicit `watch_id` reuse on raw gRPC | Safety (start revision, key range) | No | open [#22673](https://github.com/etcd-io/etcd/issues/22673) | Moderate | 8.2 |
| `RequestProgress` while any watcher on the stream is unsynced | Progress signal (no response) | No | by design ([#20242](https://github.com/etcd-io/etcd/issues/20242)) | High | 8.2 |

**D. gRPC proxy (§9)**

| State | Promise affected | Visible to client? | Versions | Confidence | § |
|---|---|---|---|---|---|
| Serving member isolated, with `WithRequireLeader`, **through the proxy** | Go client's documented `WithRequireLeader` behaviour | No; silent for the whole partition | since v3.3.0, incl. v3.7.3 | High (reproduced) | 9.1 |
| Watch attached to an ended upstream watch | Eventual delivery (permanent silence) | No | open [#21813](https://github.com/etcd-io/etcd/issues/21813) | High (code), Moderate (frequency) | 9.2 |
| Upstream watch ends with a non-compaction error | Go client's documented final-error behaviour | No; the cancellation is filtered out | v3.7.3 (earlier versions not checked) | High (code) | 9.3 |
| `RequestProgress` through the proxy | Progress signal (no bookmark ever) | No; the request is logged and dropped | [#19710](https://github.com/etcd-io/etcd/issues/19710), closed not planned | High (code + report) | 9.3 |

**E. Client libraries (§10)**

| State | Promise affected | Visible to client? | Versions | Confidence | § |
|---|---|---|---|---|---|
| Go: fatal stream error while the application holds a response for > 250 ms | Go client's documented final response | Channel closes bare; documented meaning is "your context was cancelled" | v3.7.3 (and earlier) | High (code); not reproduced | 10.2 |
| Go: rejected watch creation races with a fatal stream error | Eventual delivery | No; later `Watch()` calls on the same metadata block | v3.7.3; [#17611](https://github.com/etcd-io/etcd/issues/17611) closed unfixed | Moderate (narrow race) | 10.2 |
| jetcd: resume of a historical watch before its first batch | Safety (Reliable, Resumable) | No | jetcd 0.8.7 (and earlier) | Moderate mechanism, Low frequency | 10.3 |
| jetcd: one watch response exceeds 4 MiB | Eventual delivery, signalled | Yes (`onError`, then closed); re-watch likely fails again | jetcd 0.8.7 | High (mechanism), Moderate (reachability) | 10.4 |

## 5. Serving member cut off or stuck

### 5.1 Isolated from the quorum (the "old leader" case)

**What happens by default (High).** A minority member stops receiving committed entries, so its apply stream, and every watch it serves, goes silent. Nothing on the server ends the stream. The member is alive and answers gRPC keepalive pings, so the client's balancer keeps the stream where it is. When the partition heals, the member catches up from the log or a leader snapshot, and events arrive late but, on current versions, are not lost.
- `design-client.md:106`, `:136` (client keepalive cannot detect this).
- This is the default in both clients. jetcd: `requireLeader = false` (`jetcd/options/WatchOption.java:154`).

**With `WithRequireLeader` (verified mechanism, derived timing).** The client gets `etcdserver: no leader` after **about 3–5 s**; measured 3.3 s and 4.3 s in two runs after isolating the then-leader (Appendix C). With the default timeouts (`--heartbeat-interval=100ms`, `--election-timeout=1000ms`; `server/embed/config.go:533-534`), the sequence is:
1. An isolated follower clears its leader when its randomized election timer fires, after about 1–2 s: `becomePreCandidate` sets `r.lead = None` (`raft.go:917-929` in raft v3.7.0).
2. An isolated old leader steps down through CheckQuorum, which etcd always enables (`etcdserver/bootstrap.go:548`). The step-down calls `becomeFollower(r.Term, None)` (`raft.go:1281-1284`).
3. The server's leader monitor then needs 3 consecutive leaderless samples taken 1 s apart (`v3rpc/interceptor.go:320-348`).

This matches the Kubernetes code comment on why it waits "3 election timeouts" (`apiserver/storage/etcd3/store.go:1156-1165`).

**After cancellation.** Neither client retries `ErrNoLeader`; both end the watch.
- Go: every watch on the stream is closed with that error (`client/v3/watch.go:667-669`).
- jetcd: `shouldReschedule` excludes no-leader errors, and the watcher is closed (`jetcd/impl/WatchImpl.java:353-355`).

So `WithRequireLeader` turns silent staleness into an *availability* event. The application must re-watch from its last revision + 1; re-watching from "now" creates a gap (§6.3 item 4). At scale this costs something: [kubernetes#111116](https://github.com/kubernetes/kubernetes/issues/111116) reports that a prolonged etcd election makes the apiserver terminate *all* watchers at once.

**History.**
- [#8980](https://github.com/etcd-io/etcd/issues/8980) (2017): a watch "blocks forever" on loss of connectivity. A maintainer replied, "watch is not really meant for detecting disconnects". In 2023 a maintainer verified in that thread that `WithRequireLeader` cancels watches on a partitioned member after ~3 s, and noted that an abruptly stopped single node still hangs the client.
- Kubernetes adopted `WithRequireLeader` in 2020: [#89488](https://github.com/kubernetes/kubernetes/issues/89488), PR [#89881](https://github.com/kubernetes/kubernetes/pull/89881) (summarized fetch). In that thread a Kubernetes maintainer stated that the option "only applies to establishing the initial watch connection". That was already wrong in 2020: established streams have been cancelled since commit `9c103dd0d` (PR #5332, May 2016, first in v3.0.0). It does hold *through the gRPC proxy*, where neither form of the check works (§9.1).

**Condition under which this fails:** if a partitioned member kept a non-zero leader ID. Under PreVote plus CheckQuorum it does not, except in the case of §5.2.

### 5.2 Has a leader but does not apply (Moderate; inferred)

**Mechanism.** Leader tracking and applying run on different goroutines, so a stuck apply does not clear the leader. The member keeps receiving heartbeats, `WithRequireLeader` never fires, and its watches go silent because events are produced only on apply. Writes routed through the member eventually fail, and linearizable reads on it block, but **a watch-only client has no signal**.
- `etcdserver/raft.go:181-206`: the leader ID (`s.Leader()`) is updated from `SoftState` in the raft goroutine, which also drives ticks.
- `etcdserver/server.go:843-845`: committed entries are handed to the `run` loop, which only *schedules* `applyAll` on a FIFO scheduler.
- `mvcc/watchable_store_txn.go:43-46`: events are produced only on apply.
- `etcdserver/v3_server.go:51`: proposals are rejected once commit − applied exceeds 5000.
- `etcdserver/raft.go:304-321`: a pending configuration change makes the raft loop wait for apply; even then the leader view is frozen, not cleared.

**Possible triggers.**
- **[#22184](https://github.com/etcd-io/etcd/issues/22184)** (open): a lock-order inversion that would wedge the apply loop permanently. The inversion is real, but no production trigger is confirmed (§5.3).
- **[#22326](https://github.com/etcd-io/etcd/issues/22326) / [#22214](https://github.com/etcd-io/etcd/issues/22214)** (open): a log sink that stops draining hangs etcd goroutines while "the member reports healthy". #22214 concerns the linearizable-read loop; whether apply is affected depends on which goroutine logs. Not verified.
- **The raft loop itself blocks (High for the mechanism, from code; not reproduced; added 2026-11-03).** This case is worse than an apply stall, because the leader view freezes instead of merely staying correct.
  - The member's leader ID is written only when the raft loop takes a new `Ready` (`etcdserver/raft.go:185`, `:198`). The same loop drives raft's ticks (`:183`), hands committed entries to apply (`:232`) and writes the WAL (`:256`), which ends in `fdatasync` (`server/storage/wal/wal.go:881`).
  - If that write does not return, the loop takes no further `Ready`. The member stops applying, raft stops ticking, and the leader ID keeps its last value, even if the member then also loses contact with the leader.
  - `monitorLeader` and the new-stream check (`v3rpc/interceptor.go:239-242`, `:315-354`) therefore keep seeing a leader: existing `WithRequireLeader` watches are not cancelled, and new ones are admitted. The gRPC server's goroutines do not touch the disk, so the member keeps answering client keepalive pings. Neither remedy ends the watch.
  - Linearizable reads on the member block, because ReadIndex results also pass through this loop (`:209-217`); only a client-built freshness check with a deadline notices (§13.2 item 8).
  - Evidence that the loop does stall on slow `fdatasync` in production: [#15247](https://github.com/etcd-io/etcd/issues/15247) (closed), on the leader, where it caused lease revocations (`leases-claude.md`). A write that never returns, which would make the silence permanent, needs a hung storage device; one search of etcd issues (2026-11-03) found no report of that.
  - `watchers-codex.md` §2.6, revised the same day, reaches the same counterexample independently; I read it after writing this.
- **Asymmetric failure (Low).** Raft caps the commit index carried in heartbeats at the follower's acknowledged match index (`raft.go:694-709`). A member that receives the leader's heartbeats but whose replies never reach the leader therefore stops advancing while still reporting a leader. Pure IP-level one-way loss breaks TCP in both directions, so this needs an application-level asymmetric failure. I have no evidence it occurs in practice.

**Upstream recognition.** Proposal [#22635](https://github.com/etcd-io/etcd/issues/22635) (open, 2026-10-21) asks for server health entries `apply_stall` ("applied keeps advancing while committed advances") and `apply_lag`. This corroborates, without confirming, that upstream sees "leader present, apply stalled" as a state clients cannot currently detect.

### 5.3 #22184 in detail: a real inversion without a confirmed trigger

*Revised 2026-11-01 after the Codex analysis pointed out the flaw in the issue's trigger; the lock-site survey is mine.*

**The inversion (High).** Two code paths take the same two locks in opposite orders.
- A progress request (`progressIfSync`) takes the watch store's lock, `watchableStore.mu`, and then the KV store's lock, `store.mu`, through `s.rev()` → `readView.Rev()` → `store.Read()` (`mvcc/watchable_store.go:515-519`, `mvcc/kv_view.go:32-35`, `mvcc/kvstore_txn.go:46-48`). The code comment forbids that order (`mvcc/watchable_store.go:59-61`).
- A write transaction holds `store.mu` for reading (`mvcc/kvstore_txn.go:174-175`) and, in `End()`, waits for `watchableStore.mu` (`mvcc/watchable_store_txn.go:43-46`).

**Its age (High).** The order is old. v3.4.0's `progress()` already called `s.rev()` under `s.mu.RLock()` (`mvcc/watchable_store.go:480-488` at tag v3.4.0, where the package lived at the repository root). PR #20229 (`973847aa2`, 2025) only moved the call to the top of the function, so it now runs on every progress request, not just on those that send a notification.

**What a deadlock needs.** Two read-lock holders do not block each other. A deadlock needs a *third* goroutine that waits for `store.mu` as a writer while the write transaction holds it; Go's `RWMutex` then blocks new readers such as `s.rev()`. The reporter's deterministic unit test constructs that third goroutine.

**The issue's production trigger does not fit (High).** It names a snapshot `Commit()`. But both `KV().Commit()` call sites run inside `applyAll`, on the same goroutine that runs write transactions, and the code says so: "KV().Commit() cannot run in parallel with toApply" (`etcdserver/server.go:2103`).
- Disk snapshot: `etcdserver/server.go:2096-2105`.
- Merged snapshot for a lagging follower: `etcdserver/server.go:986-990` → `etcdserver/snapshot_merge.go:38`.

**No other candidate found (Moderate).** The other `store.mu.Lock()` sites are:
- `Compact` and `Restore`, both applied on the same goroutine (`mvcc/kvstore.go:283`, `:303`);
- `NewStore`, at start-up only (`:127`);
- `compactBarrier`, only when the compaction scheduler has already been stopped (`:137-153`).

**Assessment.** The inversion is a latent defect. A production deadlock on v3.7.3 is **unconfirmed (Low–Moderate)** until someone identifies a concurrent `store.mu` writer. If it did occur, the result would be as the issue describes: a permanently wedged apply loop and silent watches. Kubernetes issues `RequestProgress` every 100 ms whenever a consistent read is waiting (`apiserver/storage/cacher/progress/watch_progress.go:36`), so the progress ingredient is common there.

### 5.4 Paused member, dead connection, or no reachable member (High)

**Paused process or black-holed connection.** A paused (`SIGSTOP`, VM freeze) or wedged process cannot run its leader monitor, so the server never cancels anything. Only client-side HTTP/2 keepalive can detect it, and the defaults differ:

| Client | Keepalive default | Evidence |
|---|---|---|
| Go | Off: sent only if `DialKeepAliveTime > 0`, default zero. A paused member hangs a Go watch indefinitely; this is the original [#8980](https://github.com/etcd-io/etcd/issues/8980) reproduction. | `client/v3/client.go:231-236` |
| jetcd | 30 s, 10 s timeout, also without active calls | `jetcd/ClientBuilder.java:66-68` |
| Kubernetes | 30 s + 10 s | `apiserver/storage/storagebackend/factory/etcd3.go:63-64`, `:344-345` |

**Client cut off from every member.**
- **Go retries forever and never closes the channel** (`client/v3/watch.go:85-91`, `:666-679`, `:1024-1036`). This is structural, not just a default: a watch's own context still cancels that watch, but nothing in the library bounds how long the application waits for events or an error.
  - The stream runs under a context that keeps the caller's values but has no deadline and is never done (`valCtx`, `client/v3/watch.go:265-280`).
  - `openWatchClient` loops until it gets a stream or a halt error (`client/v3/watch.go:1042-1062`).
  - Calls default to `WaitForReady(true)` (`client/v3/options.go:24-28`).
- **jetcd reports each failure and retries** every 500 ms through `listener.onError` (`jetcd/impl/WatchImpl.java:327-368`), so the application at least sees errors. Two exceptions:
  - **A failed scheduled resume stops recovery.** If `resume()` itself throws, jetcd only logs "scheduled resume failed" and never retries again; the watcher is dead but not closed ([jetcd#1227](https://github.com/etcd-io/jetcd/issues/1227), open; `WatchImpl.java:357-368`).
  - **Slow reconnect after a long outage.** A jetcd client can stay disconnected for up to about two minutes after the cluster is back, while grpc-java's channel sits in reconnect backoff (cap two minutes); jetcd offers no way to reset it ([jetcd#1616](https://github.com/etcd-io/jetcd/issues/1616), open; the reporter measured 138 s on 0.8.3, for KV calls). The proposed fix, PR [#1617](https://github.com/etcd-io/jetcd/pull/1617) (open), covers only calls retried by jetcd's retry policy; its description states that "a watch resumed by `WatcherImpl.reschedule()` ... still waits for the channel's own timer" (pointed out by Codex's revised analysis; PR read raw). jetcd stubs use gRPC wait-for-ready by default (`jetcd/ClientBuilder.java:71`, applied in `jetcd/impl/ClientConnectionManager.java:110-114`), so a resumed watch most likely *waits* for the channel instead of failing, and the delay would be silent, with no `onError`. That last step is inferred, not tested.

## 6. `WithRequireLeader`: the remedy for §5.1, and its limits

### 6.1 What it does (verified)

| Layer | Behaviour | Evidence |
|---|---|---|
| Go client | `WithRequireLeader(ctx)` adds the gRPC *outgoing* metadata pair `hasleader=true`. Nothing else changes on the client. Because the watch client groups watches into gRPC streams by their metadata, flagged watches share one stream, separate from unflagged ones. | `client/v3/ctx.go:26-38`; `client/v3/watch.go:330`, `:343-347`, `:1086-1091` |
| Server, unary calls | Rejected with `ErrGRPCNoLeader` if the serving member has no leader. | `v3rpc/interceptor.go:57`, `:68-72` |
| Server, stream open (watch, lease keep-alive) | Rejected if the member has no leader; otherwise wrapped in a cancellable context and registered. | `v3rpc/interceptor.go:228-258` |
| Server, open streams | One goroutine, `monitorLeader`, samples the leader ID once per election timeout. After `maxNoLeaderCnt = 3` consecutive leaderless samples it cancels every registered stream with `ErrGRPCNoLeader`. Any sample that sees a leader resets the count. The watch handler returns that reason as the stream status. | `v3rpc/interceptor.go:38`, `:315-354`; `v3rpc/watch.go:226-231` |
| Server, new watches on an open stream | Not checked at all. | `v3rpc/watch.go:259-370` |

**How the Go watch client ends a flagged watch.**
- `ErrNoLeader` is terminal and not retried (`client/v3/watch.go:667-669`). Every watch on the stream receives a final `Canceled` response whose `Err()` is `etcdserver: no leader` and whose header is empty; then its channel is closed (`client/v3/watch.go:494-495`, `:520-541`).
- Responses already received but not yet consumed by the application are discarded: closing the substream's receive channel makes `serveSubstream` return without flushing its buffer (`client/v3/watch.go:832-836`).
- The final `ErrNoLeader` response itself is best-effort: it is dropped if the application leaves a response unread for more than 250 ms (§10.2).
- Ending on `ErrNoLeader` is a watch-specific choice, and an old one: the same check is in v3.1.0's `clientv3/watch.go:482`. The client's general classifier treats `Unavailable`, "(e.g., can't connect, lost leader)", as retryable (`client/v3/client.go:577-584`); the watch loop overrides it (`client/v3/watch.go:667`).

**The Go lease client** also uses the flag; details are in `analysis/leases-claude.md` §5.2 (§12 before that file's 2026-11-01 restructuring).
- Its keep-alive stream *always* carries the flag (`client/v3/lease.go:210-211`). `closeRequireLeader` closes only the keep-alive channels whose own context carried it (`client/v3/lease.go:376-410`).
- That function has two open bugs: [#22081](https://github.com/etcd-io/etcd/issues/22081) (the index mix-up at `client/v3/lease.go:406` is present in v3.7.3) and [#22094](https://github.com/etcd-io/etcd/issues/22094). #22094 can keep a lease renewed every 500 ms for the life of the client process, after telling the application its keep-alive ended.
- The gRPC proxy also ignores the flag on keep-alive streams (§9.1).
- Before v3.7.0, servers reported *any* cancellation of a keep-alive stream as `ErrGRPCNoLeader`.

**Who sets it.**
- `etcdctl watch`, for the watch and for `RequestProgress` (`etcdctl/ctlv3/command/watch_command.go:130`, `:163`).
- Kubernetes, on every watch (`apiserver/storage/etcd3/store.go:1156-1165`).
- The gRPC proxy, for its internal leader detector (`grpcproxy/leader.go:42-80`).
- Not the Go client or jetcd by default (`jetcd/options/WatchOption.java:154`).
- HTTP/JSON gateway callers must send the header `Grpc-Metadata-hasleader: true` ([#10819](https://github.com/etcd-io/etcd/issues/10819)).

### 6.2 Why it was added (verified from git history and PR threads)

**The scenario** is a client pinned to a member that is partitioned from the quorum. That member keeps serving open streams and stale reads with no error.

**The commits** date from May 2016; all three are first tagged in `v3.0.0`.

| PR | Commit | What it did |
|---|---|---|
| #5324 (branch `xiang90/partitioned`) | `19221b33c` | "etcd member rejects unary call with leader requirement when it does not have leader". |
| #5328 | `68eaf4083` | The client helper. A reviewer argued against a client-wide setting: "I'm not sure about this being a client config option. Maybe the user can do something like `client.Get(v3.WithLeader(ctx), "abc")` instead?" That is why it is per-request and opt-in. |
| #5332 | `9c103dd0d` | "cancel required leader streams when member lost its leader". The code comment explains the 3-sample delay: "We are more conservative on canceling existing streams. Reconnecting streams cost much more than just rejecting new requests." At first the Watch handler mapped *every* context cancellation to `ErrGRPCNoLeader` ("the only server-side cancellation is noleader for now"). |

**Later discussion.**
- [#5354](https://github.com/etcd-io/etcd/issues/5354) (2016) followed up on how watch clients receive `ErrNoLeader`.
- [#7247](https://github.com/etcd-io/etcd/issues/7247#issuecomment-276154894) (2017-01-30) states the intent most directly (found by the Codex analysis; read raw): "By default, the watchers connected to the minority will hang there until the network partition recovers. However, you can provide option `requireLeader` ... to ensure they can be aborted, and then be switched to the majority when partition happens." The library never did the switching: `ErrNoLeader` has been terminal in the Go watch client since v3.1.0 at the latest (§6.1). The switch was always the application's job, without the resume revision it needs (§6.3 item 4).
- [#8673](https://github.com/etcd-io/etcd/issues/8673) (2017–2018, "Network-partition aware health service") established that client keepalive cannot detect partitions and that `WithRequireLeader` is the only remedy:
  ```go
  wch := cli.Watch(clientv3.WithRequireLeader(context.Background()), "foo")
  // will be closed when the node loses leader
  wch := cli.Watch(context.Background(), "foo")
  // block forever, even if a node loses leader
  ```
  In that thread a maintainer tied it to [kubernetes#59848](https://github.com/kubernetes/kubernetes/issues/59848): "apiserver becomes partitioned from the etcd majority but is able to reestablish a watch from the partitioned member at RV=10; kubelet contacts the apiserver and sees RV=10, launches the pod".
- PR [#9872](https://github.com/etcd-io/etcd/pull/9872) (2018, commit `11ead62b9`, 2018-06-20), "clarify WithRequireLeader for network partition", references #59848 and #8673.

**Why it is not the default:** I found no documented rationale. A 2019 maintainer comment only guessed at "backward compatibility reasons" ([#10819](https://github.com/etcd-io/etcd/issues/10819)).

### 6.3 Why it is not a full solution

Each limit is stated as claim, reason, and the condition under which it would not hold.

1. **It checks leadership, not freshness (Moderate).** It tests "the serving member currently has a leader"; the client needs "the serving member is applying what the quorum commits".
   - The two diverge whenever the member sees a leader but does not apply (§5.2); when its raft loop is blocked, which freezes the leader ID instead of clearing it (§5.2); and when the process is paused, because then the monitor cannot run at all (§5.4).
   - Would not hold if every way of not progressing also cleared the leader ID. The raft loop and the apply goroutine are decoupled (`etcdserver/server.go:843-845`), so it does not, and a blocked raft loop cannot clear it at all.
2. **A stale window of about 3–5 s remains (High; measured 3.3 / 4.3 s).**
   - In that window the member keeps serving its existing streams. It also creates *new* watches on them at its stale "now", because the create path has no leader check.
   - It answers progress requests with its stale revision: consistent with Bookmarkable, but uninformative about freshness.
   - Until raft clears the leader (1–2 s for a follower), even new streams are accepted.
3. **The counter resets (Low–Moderate; inferred; not tested).** Any sample that sees a leader resets the count (`v3rpc/interceptor.go:329-333`). A member that intermittently sees a leader, for example over a flaky link, can postpone cancellation indefinitely while receiving few entries.
4. **It ends the watch instead of resuming it, and throws away the resume point (High; follows from code).** Whether this is a library bug or a contract gap is a judgment call.
   - On `ErrNoLeader` the Go client closes the channel with an empty header and drops its internal resume revision (`ws.initReq.rev`, maintained at `client/v3/watch.go:858-871`). There is no API to retrieve it.
   - A "from now" watch that has delivered no events leaves the application nothing to resume from, unless it asked for `WithCreatedNotify`.
   - The natural retry, watching again from "now", silently skips every write committed between the original call and the retry:
     ```
     wch := cli.Watch(WithRequireLeader(ctx), "a")  // served by m1, which is (or becomes) partitioned
     // majority: Put("a","b") at revision R
     <-wch                                          // Canceled, Err()=etcdserver: no leader, Header.Revision=0
     wch = cli.Watch(WithRequireLeader(ctx), "a")   // lands on a healthy member; starts after R
     // the Put at R is never delivered
     ```
   - This is exactly the case the library's own comment says its *transparent* resume must handle: "if wch is disconnected before the Put is issued, then reconnects after it is committed, it'll miss the Put" (`client/v3/watch.go:849-857`). `WithRequireLeader` moves the reconnect to the application without handing over the information it needs.
   - Without the option, the same watch would stay stuck and then deliver the Put after the partition heals: late, but not lost.
   - The `Resumable` guarantee is phrased in terms of "the last revision received in a watch event", which does not exist for a watch that has received none.
   - jetcd behaves the same way: its resume `revision` is private, and a no-leader error closes the watcher (`jetcd/impl/WatchImpl.java:327-355`).
   - Kubernetes is immune: it always watches from an explicit revision (list revision + 1) and resumes from the last event or bookmark.
5. **New streams get no grace period (Moderate; inferred, timing-dependent).**
   - Established streams get 3 election timeouts; a new stream is rejected the moment the leader ID reads empty (`v3rpc/interceptor.go:239-242`).
   - If a stream breaks for an unrelated reason, such as its member (the old leader) crashing, and the transparent reconnect lands on a member that is a pre-candidate, the watch is *terminated*, not resumed. Terminations therefore cluster at failover; [kubernetes#111116](https://github.com/kubernetes/kubernetes/issues/111116) reports the cost at scale.
   - The same applies to an application's own re-watch after `ErrNoLeader`. If the client's endpoint list still includes the partitioned member, its connection stays healthy and the balancer can place the new stream there; it is rejected at admission and the application sees `ErrNoLeader` again. This is visible churn, not silence. Moderate: per-stream placement by gRPC's balancer is recalled, not read.
   - etcd's own partition test avoids the question by pinning the client to the single partitioned endpoint (`tests/integration/clientv3/connectivity/network_partition_test.go:225-243`). Commit `a57e967d8` (2021, "Integration test flakes fixes") removed `SetEndpoints(eps...)` with the comment: "We stick to the original endpoint, so when the one fails we don't switch under the cover to other available eps, but expose the failure to the caller."
6. **The metadata also chooses the stream (High).**
   - `RequestProgress(ctx)` is routed to the stream whose key matches `ctx`'s metadata, and creates a new empty stream if there is none (`client/v3/watch.go:413-425`).
   - Calling it with `context.Background()` while the watches carry `WithRequireLeader` sends the request to a stream with no watchers. The server sends nothing (`mvcc/watchable_store.go:515-540`) and the call returns `nil`.
   - Reported in 2020 as [#12254](https://github.com/etcd-io/etcd/issues/12254) (closed); the code still behaves this way at v3.7.3.
   - Kubernetes works around it deliberately: "Use watchContext to match ctx metadata provided when creating the watch" (`apiserver/storage/etcd3/store.go:102-106`).
7. **It is opt-in.** Off in the Go client, in jetcd, and for HTTP/JSON watches.
8. **It does not cover the client side of a partition.** If the client cannot reach any member, no server can cancel anything and the Go client retries forever (`client/v3/watch.go:85-91`). A dead TCP path needs keepalive (§5.4).
9. **It does not work through the gRPC proxy** (§9.1).
10. **Its error can be lost on the way to the application (High, code).** Even when the server cancels correctly and the library receives `ErrNoLeader`, the final response reaches the application only if it reads within 250 ms. Otherwise the channel closes bare, which the documentation reserves for the caller's own cancellation (§10.2).

### 6.4 What it achieves in the motivating scenario

A member partitioned from the quorum still, according to the code:
- serves its open streams, silently;
- accepts new streams until raft clears its leader (about 1–2 s, or the old leader's CheckQuorum step-down);
- creates new watches on open streams at its stale "now" until cancellation;
- answers serializable reads, unless they carry the flag;
- answers progress requests with its stale revision.

Only its linearizable reads fail, because ReadIndex needs a leader. Seen from the client:
- `WithRequireLeader` **bounds** how long a client connected *directly* to such a member can consume stale data without an error, to about 3–5 s. It makes nothing served in that window fresher.
- It turns a stall into a termination. With explicit-revision resumption that is harmless; with "from now" re-creation it silently loses the writes made during the partition (§6.3 item 4).
- Through the gRPC proxy it does nothing (§9.1).
- It does not resolve [kubernetes#59848](https://github.com/kubernetes/kubernetes/issues/59848), still open. That issue concerns apiserver watch-cache staleness served to kubelets. Kubernetes' real fix for consistent reads is KEP-2340, and `resourceVersion=0` reads remain arbitrarily stale (§11).
- Without the option, history is not lost. In the Docker test the proxied watcher (which never got the option's effect) received the partition-time write at heal in run 2, and a fresh watch from revision 1 showed complete history. Only freshness is lost.

## 7. Resume, restore and compaction

These are the paths where member-local watch state meets replicated state, and where etcd's documented guarantees have actually been broken. **Pattern (High):** the fixed violations cluster in snapshot restore, compaction, crash recovery and resuming against a different member. They were found once etcd's robustness tests started injecting those conditions; the "Discovered by" column of etcd's own bug ledger says so (`tests/robustness/README.md:19-39`, called the robustness "track record"). Those tests replay client traffic under injected faults and check the watch guarantees of §2.1 (`tests/robustness/validate/watch.go:31-39`); Antithesis is the deterministic-simulation fault-injection service etcd also runs (`tests/antithesis/`).

Counting from that ledger: 8 watch-guarantee violations reported in 2023–2025, 6 found by the robustness or Antithesis tests and 2 by users. All but one originated in v3.4 or earlier; the exception, #17247, was a regression on `main` that was never released. Two more were reported separately: #19487 and #17507. All are fixed by v3.7.0, with backports to 3.4/3.5/3.6. **Appendix A** lists each violation with its trigger and fix versions.

### 7.1 Resume on a different member

When the serving member dies or the stream breaks, the client library re-creates the watch on another endpoint from a stored "next revision". Resuming on a *lagging* member turns the watch into a future watch, which is legal: the member waits until it reaches that revision. The client-library side of this is in §10.1 (Go) and §10.3 (jetcd); this section covers the server.

**Server bugs triggered by cross-member resume (fixed).** [#20221](https://github.com/etcd-io/etcd/issues/20221) (2025) was found within a day of adding a robustness client that connects to multiple members. It had two parts:

| Part | Fix | Current code |
|---|---|---|
| A future-revision watch received a progress notification below its start revision, breaking Bookmarkable. | #20229, "Skip sending progress notification for watch with starting revision in the future" | guard at `mvcc/watchable_store.go:525-527` |
| A future-revision watch received old events. The same fix covers [#19487](https://github.com/etcd-io/etcd/issues/19487). | #20281, "Avoid lowering revision of watchers in the future after restore" | `mvcc/watchable_store.go:374`: `w.minRev = max(curRev+1, w.minRev)` |

Both are in v3.5.22, v3.6.2 and v3.7.0 (verified with `git tag --contains` on `d1d3dc296` / `2d72ea1b8` for 3.5, `68075c999` / `438287a59` for 3.6, and the main merges `1a2efc3c4` / `9fb8c3685`). The thread (summarized fetch) shows maintainers could not at first tell whether client or server was at fault. One wrote: "MemberId changes in an transparent way, meaning we switched etcd we talk to, resulting with incorrect watch stream response."

**Design note (Moderate).** Bookmarkable is defined per stream (`rpc.proto:426-428`), but the Go client hides stream replacement from the application. A client that records bookmark revisions therefore assumes a cross-stream monotonicity the proto does not promise. Since #20229 the server no longer sends a future watch a bookmark below its start revision, which restores monotonicity for the resume case.

### 7.2 Serving member restores a leader snapshot

This is how an old leader or a long-partitioned follower usually rejoins: the leader's log has been compacted, so it sends a snapshot. The member swaps its backend and moves every synced watcher to unsynced, flagged `restore`; the sync loop tolerates their future start revisions. Since the #15271 fix, `notify` always advances a watcher's next revision, "needed for Restore"; before it, restored watchers replayed from a stale revision and sent old events again.
- `mvcc/watchable_store.go:207-221` (`watchableStore.Restore`); `mvcc/watcher_group.go:241-254` (`chooseAll`); `mvcc/watchable_store.go:486-489` (`notify`).

**Bugs on this path.**
- [#8411](https://github.com/etcd-io/etcd/issues/8411): `Restore` did not exist, so watchers on a rejoined member got nothing. The 2017 discussion includes a patch adding `Restore`; I did not check which release first shipped it.
- [#15271](https://github.com/etcd-io/etcd/issues/15271): time travel. Fixed in v3.4.25 and v3.5.8, verified for `46ae7ebd9` and `15b3756ab`.
- [#19487](https://github.com/etcd-io/etcd/issues/19487): responses below the start revision. Fixed as in §7.1.
- [#20573](https://github.com/etcd-io/etcd/issues/20573) (Antithesis, 2025): a broken Resumable guarantee, with a snapshot implicated in the reproduction. Closed in 2025-11 as stale after "No failures for long time", **without an identified root cause** (raw read of comments). Confidence that this class is fully closed: **Moderate**.

### 7.3 Watches near the compaction boundary

Correct behaviour is an explicit cancel carrying `CompactRevision`, after which the client must relist. Three fixed bugs made etcd *silently skip* events instead:

| Issue | Trigger | Fix and versions (verified) |
|---|---|---|
| [#17529](https://github.com/etcd-io/etcd/issues/17529) (2024) | The watch channel was full when the compaction response was due, so the sync loop advanced the watcher past the compaction point. The next pass never sent the cancel, and the watcher continued with a gap. | #17555; v3.5.13, v3.4.31. Skip guard now at `mvcc/watchable_store.go:369-373`. |
| [#18089](https://github.com/etcd-io/etcd/issues/18089) (2024) | A DELETE whose tombstone sat exactly at the compaction revision was dropped. Present since at least v3.4.0, per a maintainer's manual reproduction. | #18274 and backports; v3.5.16, v3.4.34. |
| [#19179](https://github.com/etcd-io/etcd/issues/19179) (2025) | The same class, through crash and restore of the tombstone index. | v3.5.18, v3.4.36. |

#17529 had real consequences: [kubernetes#123072](https://github.com/kubernetes/kubernetes/issues/123072) (summarized fetch) reports watch caches that kept objects deleted more than 24 h earlier, with informers fed by them not recovering until restart. On #18089's frequency a maintainer wrote: "It's once every 5 minutes, ask anyone if they want to build a distributed system that makes a mistake every 5 minutes."

**Open: [#22294](https://github.com/etcd-io/etcd/issues/22294), compacted watchers not retired (Moderate).**
- With ≥512 unsynced watchers, the sync loop works on a *copy* of a subset and deletes compacted watchers from the copy. They stay unsynced and are re-sent `CompactRevision` every 100 ms. The code matches the report (`mvcc/watcher_group.go:226-239`, `:255-262`); the reporter states there is no event loss.
- The common way to reach 512 unsynced watchers is a snapshot restore, which moves *all* synced watchers to unsynced (`mvcc/watchable_store.go:207-221`).
- While such a watcher stays registered on a stream, that stream's `RequestProgress` goes unanswered, because the watcher is not synced (§8.2). The effect ends when the client cancels the watch; the Go client sends that cancel right after delivering the compaction error (`client/v3/watch.go:685-706`).
- The Antigravity analysis states that this disables progress "across the entire server" and that the duplicates pile up forever in the server's `pending` buffer. Both are wrong: progress checks only the requesting stream's watchers (`mvcc/watchable_store.go:515-528`), and compaction responses never remove the watch ID that `pending` is keyed on (`v3rpc/watch.go:459-476`, `:533-541`).

**Also open.**
- [#22614](https://github.com/etcd-io/etcd/issues/22614): the experimental `cache/` library never becomes ready again after a compaction cancels its upstream watch.
- A missing previous value (§2.3). Kubernetes treats it as an error (`apiserver/storage/etcd3/event.go:65-67`); other clients silently get a DELETE without the old value.

### 7.4 Operator restore: revision rollback (High)

Restoring a cluster from an older snapshot moves the revision backwards. A client that resumes at its last revision + 1 then holds a *future* watch on the restored cluster (`mvcc/watchable_store.go:141-147`). Every new write numbered between the restored revision and that resume point is never delivered, with no error ([#14303](https://github.com/etcd-io/etcd/issues/14303): "the client side doesn't receive any error"). The Go watch client does not check the cluster ID (`grep ClusterId client/v3/watch.go` finds nothing).

The remedy exists but is off by default: `etcdutl snapshot restore --bump-revision --mark-compacted` pushes the revision past every client's resume point and forces a compaction error, hence a relist (`etcdutl/etcdutl/snapshot_command.go:83-84`, `:134`).

### 7.5 Divergent member state; access control

- **Divergent state.** In v3.5.0–v3.5.2 a crash could leave a member with committed transactions missing (`Documentation/postmortems/v3.5-data-inconsistency.md:13`; fixed in v3.5.3). Watchers on that member would see a history different from the other members'. This impact is inferred; the postmortem does not discuss watches. Corruption checks detect divergence; they do not fix the watchers.
- **Access control.** A user with read permission on one key could watch every key from that key onward, by sending the "from key" range end `\x00`: range-end normalization ran before the permission check. Fixed in v3.7.1, v3.6.14 and v3.5.33 (commit `afeaa624d` and its backports; current order at `v3rpc/watch.go:287-328`). Not in any 3.4 release-branch commit I found.

### 7.6 Lease-driven DELETE events (pointer)

Watches are how clients observe lease expiry, and leader failover changes *when* those DELETEs happen: by default leases are extended on promotion (`server/lease/lessor.go:480`, `Promote`), and a stale leader can also revoke leases. The watch faithfully reports whatever Raft committed, so these are lease-contract problems, not watch problems. See `analysis/leases-claude.md`, `analysis/leases-codex.md` and `analysis/leases-agy.md`; they were not used for this analysis.

## 8. Delivery path: congestion and stream protocol

### 8.1 Congested streams and slow watchers (Moderate)

Most items here delay events rather than lose them. Two can stop a stream outright (cancellation bursts) or turn delay into errors (`PrevKv`).

- **Shared streams.** The Go client multiplexes every watch created with the same context metadata onto one gRPC stream (`client/v3/watch.go:1086-1091`), so one slow consumer delays all of them. Kubernetes uses one client per resource type, so its watches do not share a stream across resources (KEP-2340, "Performance").
- **Starvation** ([#16839](https://github.com/etcd-io/etcd/issues/16839), open, reopened in 2026; summarized fetch). Delivery can lag far behind writes with no error: p90 watch latency of 5.5–6.6 s was reported while puts stayed in milliseconds, with etcd at 4–6 GB of memory. The cause is the sync loop re-reading history every 100 ms.
- **Re-scan cost** ([#22264](https://github.com/etcd-io/etcd/issues/22264), open). An apiserver watch 500,000 revisions behind is re-scanned in full on each pass but advances only 1000 revisions per pass. The scan holds `watchableStore.mu`, which the write path also needs (`mvcc/watchable_store_txn.go:43-46`). The reporter saw writes blocked for more than 90% of the time on 3.5.21, and states it is still present in 3.6.14 and 3.7.1. One lagging watcher can thus degrade the cluster's write path: a failure in the opposite direction.
- **Unbounded client buffer (High, code).** The Go client appends every received response to a per-watch slice with no limit; the source carries "TODO pause channel if buffer gets too large" (`client/v3/watch.go:879-880`). A slow application therefore never pushes back on the server through gRPC flow control; memory grows on the client instead.
- **Delay turns into errors through `PrevKv` (Moderate).** The server looks up the previous value only at send time (§2.3). The longer delivery lags, the more likely a compaction removes that value first, and the event goes out without `PrevKv`, which Kubernetes treats as an error. [kubernetes#123448](https://github.com/kubernetes/kubernetes/issues/123448) (closed 2026-06-01; follow-ups #130939, #138264) describes this chain (issue text read raw; not reproduced):
  - an inactive direct watch with `resourceVersion=""` on a busy resource;
  - "direct etcd watches having an infinite buffer and never kicking clients";
  - starvation of the watch cache's own watch, which then breaks on `prevKV=nil` and can end all watches.
- **Cancellation bursts can freeze a shared stream** ([#18879](https://github.com/etcd-io/etcd/issues/18879), open; [#20716](https://github.com/etcd-io/etcd/issues/20716), closed "not planned"; fix PR [#21065](https://github.com/etcd-io/etcd/pull/21065) open). Mechanism raised by the Codex analysis; each link re-checked at v3.7.3. A burst of cancels can close a cycle: the client waits for the server to read, the server's receive loop waits for control-channel space, and its send loop waits for the client to read. Every watch on that stream then stops, with no error.
  - The Go client sends a cancel request synchronously, from the same loop that dispatches incoming responses (`client/v3/watch.go:685-706`).
  - The server's receive loop queues each cancel acknowledgement into a 16-slot control channel (`v3rpc/watch.go:124`, `:379-386`).
  - The server's send loop blocks on `Send` while the client is not reading (`v3rpc/watch.go:485-500`).
  - The reports involve the in-process adapter, which the election and lock gRPC services use internally (`grpcproxy/adapter/chan_stream.go`): High there. Over a real network, HTTP/2 flow-control buffers must fill first: Moderate.
- **TLS listener sharing** ([#15402](https://github.com/etcd-io/etcd/issues/15402), closed 2023; lead from the Codex analysis). With TLS, gRPC and HTTP on one listener, and about ten concurrent Range responses of roughly 10 MB each, a watch on the same connection got nothing, progress notifications included, for tens of seconds. The reporter tested "all v3.4+ versions"; the issue links [golang/go#58804](https://github.com/golang/go/issues/58804).
  - The mitigation is a separate listener, `--listen-client-http-urls`, added in v3.4.25, v3.5.8 and v3.6.0 (verified with `git tag --contains`). It is opt-in.
  - At v3.7.3 a TLS client listener without it still serves gRPC through `net/http` (`server/embed/serve.go:226-260`; `grpcHandlerFunc` at `:314-327`).
  - Whether Go 1.26's HTTP/2 server still starves the watch is untested (Low–Moderate).

### 8.2 Stream-protocol hazards

- **Watch-ID reuse** ([#22673](https://github.com/etcd-io/etcd/issues/22673), open, filed 2026-10-30; Moderate: code matches the report, not reproduced). Events of a cancelled watch can be delivered to a new watch that reuses its ID, even if they lie below its start revision and outside its key range.
  - Mechanism: the server's send loop buffers events for IDs it has not yet announced as created. Cancelling a watch deletes its ID from the announced set, so events already queued for it are then buffered in `pending` (`v3rpc/watch.go:468-477`, `:534-537`). A new watch with the same explicit ID flushes them (`:538-553`).
  - Exposure: the Go client always requests automatic IDs, which are never reused (`mvcc/watcher.go:128-136`), so only raw-gRPC or third-party clients can be misdelivered to. The unbounded growth of `pending` affects every long-lived stream.
- **Unanswered progress requests (by design; High).** A progress request is answered only if **all** watchers on the stream are synced and at or past their start revision; otherwise it is dropped silently and the client must ask again (`mvcc/watchable_store.go:515-540`; `v3rpc/watch.go:395-400`). [#20242](https://github.com/etcd-io/etcd/issues/20242) (closed) describes how one slow or future watcher starves the whole stream: "any setup with more than 1 watcher per stream doesn't make sense". Kubernetes' 100 ms re-request loop compensates. (Codex's revised analysis reports a functional Docker run on v3.7.3 confirming the stream scope: a future-revision watch suppressed progress on its own stream while a second stream still received it. I did not re-run it.)
- **Past progress-notification bugs (fixed).**
  - [#15220](https://github.com/etcd-io/etcd/issues/15220) (2023): a requested progress notification could overtake the events it claimed had been delivered, breaking Bookmarkable. Fixed in v3.4.25 / v3.5.8 per KEP-2340; on main by #15237, first tag v3.6.0.
  - [#17507](https://github.com/etcd-io/etcd/issues/17507): a follow-on regression where a new watch never answered progress requests. Fixed in v3.5.13, verified for `579b22cf3`.

## 9. The gRPC proxy

`etcd grpc-proxy` sits between clients and members and shares upstream watches among clients. For watch correctness it differs from a direct member connection on four counts, each covered below:
- `WithRequireLeader` is ignored (§9.1);
- upstream watches that have ended are reused (§9.2);
- upstream errors are swallowed (§9.3);
- progress requests are dropped (§9.3).

It is a separate and less-tested path. Earlier reports of delayed or missed events through it: [#9988](https://github.com/etcd-io/etcd/issues/9988) (2018), [#18231](https://github.com/etcd-io/etcd/issues/18231) (2024).

### 9.1 It ignores `WithRequireLeader` (High; reproduced; not reported upstream)

**Effect.** A client that follows the documented advice ("In order to prevent a watch stream being stuck in a partitioned node, make sure to wrap context with WithRequireLeader", `client/v3/watch.go:78-83`) gets neither an error nor events for as long as the proxy's backend member is partitioned. Lease keep-alives through the proxy are not closed early either; the client's own TTL-based deadline still applies. In the Docker test the direct watcher was cancelled after 3.3 s and 4.3 s, while the proxied watcher stayed open and silent for the whole partition, about 27 s (Appendix C).

**Cause.** The proxy looks for the flag in the wrong place.
- Its Watch handler reads it with `metadata.FromOutgoingContext(stream.Context())` (`grpcproxy/watch.go:111-125`); the lease keep-alive handler does the same (`grpcproxy/lease.go:160-173`).
- On a gRPC *server* stream, metadata sent by the client is in the *incoming* context. That is how the proxy's own auth-token forwarding reads it (`grpcproxy/util.go:26-35`), and how the etcd server interceptor reads it (`v3rpc/interceptor.go:57`, `:228`).
- The standalone proxy installs only metrics and logging interceptors (`server/etcdmain/grpc_proxy.go:507-525`), so nothing copies incoming metadata into the outgoing context.
- So the flag is never found, the proxy's leader-loss channel stays nil, and its teardown goroutine (`grpcproxy/watch.go:139-146`) never fires for leader loss.

**History.** PR #7896 (commit `5e059fd8d`, 2017-06-15, "use metadata Incoming/OutgoingContext"; first in v3.3.0) migrated from grpc's old `metadata.FromContext`. It correctly converted the etcd server interceptor to `FromIncomingContext` and the client lease code to `FromOutgoingContext`, but also converted the proxy's two *server-side* handlers to `FromOutgoingContext`, which is wrong for server streams.

**Why the tests miss it.**
- Proxy-mode integration tests connect clients to the proxy through in-process adapters (`tests/framework/integration/cluster_proxy.go:68-84`). There, the server-side stream context is derived from the *client's* context, so outgoing metadata is visible (`grpcproxy/adapter/chan_stream.go:142-153`, `grpcproxy/adapter/watch_client_adapter.go:34-39`).
- No e2e test uses `WithRequireLeader` (`grep` of `tests/e2e` finds nothing).
- A search for `FromOutgoingContext`, proxy "require leader", `lostleader` and `hasleader` found no upstream report.

**A second flaw would remain after the one-line fix (Moderate; inferred).**
- The proxy's leader signal comes from its own watch on `__lostleader`, at revision `MaxInt64-2` (`grpcproxy/leader.go:42-80`). That watch carries the flag, so it gets a different gRPC stream from the upstream broadcast watches, which carry none (`grpcproxy/watch_broadcast.go:46-68`).
- With several `--endpoints`, the round-robin balancer can place the two streams on different members. The signal then describes a member other than the one serving the events.
- The proxy also returns `rpctypes.ErrNoLeader`, a client-side `EtcdError` with no `GRPCStatus` method, so it would reach clients as code `Unknown`. That is cosmetic: the Go client maps the message back to the same error.

### 9.2 Ended upstream watches are reused ([#21813](https://github.com/etcd-io/etcd/issues/21813); mechanism High, frequency Moderate, not reproduced)

**Effect.** Once a shared upstream watch has ended, client watches on the same key range can be attached to it, or merged into it, and then receive nothing, forever, with no error. Open; reported against v3.5.4; verified in v3.7.3 code on 2026-11-01. Scope: watchers of the same key range on the same proxy process.

**How the proxy shares watches.** It groups client watches on the same key range into *broadcasts*, each served by one upstream Go-client watch (`grpcproxy/watch_ranges.go:36-45`, `grpcproxy/watch_broadcast.go:45-75`).

**Why ended broadcasts attract watchers.**
- *They stay registered.* When the upstream channel closes, the broadcast's goroutine exits and closes `donec` (`grpcproxy/watch_broadcast.go:56`, `:71-74`), but nothing removes the broadcast from its registry; `donec` is read only in `stop()` (`:151-165`).
- *New watchers can join.* `add()` checks only revisions (`grpcproxy/watch_broadcast.go:97-130`). For a broadcast that has already had a response, it even fabricates a `Created` response for the new watcher.
- *Existing watchers can be merged in.* Coalescing runs after every response on any broadcast of the range (`grpcproxy/watch_broadcasts.go:53-83`, `:130-135`). It moves the receivers of any broadcast with fewer than 5 receivers into another broadcast that has had a response and is not ahead of it. A dead broadcast's revision is frozen, so it is never ahead.

**What ends an upstream watch.** Not a transport error: the Go client retries those. The terminal cases are:
- a compaction cancel, for example when a client asks the proxy for an already-compacted revision. The original watcher receives the compaction error and cancels, but the window before that cancel is enough for a join or a merge;
- a halt error, such as an authentication failure;
- context cancellation.

**The halt-error case is worse: the dead broadcast's revision drops to 1.** (Raised in Antigravity's revised analysis, written during the 2026-11-01 revision; traced in code.)
- The Go client ends the upstream watch with a final `Canceled` response whose header is empty, so its revision is 0 (`client/v3/watch.go:494-495`).
- The broadcast passes that response to `bcast`. The broadcast has already had a response (its upstream watch uses `WithCreatedNotify`), so `bcast` sets its next revision to `0 + 1` (`grpcproxy/watch_broadcast.go:63`, `:79-86`).
- From then on, `add()` accepts every new watcher on the range that asks for an explicit revision, and coalescing moves every small live broadcast into it, because each is at or past revision 1. Only "from now" watchers (requested revision 0) are refused.
- Such a broadcast can stay registered indefinitely: its receivers never cancel, because the proxy never told them anything (§9.3).

### 9.3 Swallowed errors and dropped progress requests (High, code)

**Upstream errors are not forwarded** (lead from the Codex analysis). The proxy forwards an upstream response only if it is a progress notification, a creation, carries events, or carries a compaction revision (`grpcproxy/watcher.go:106-109`). The Go client ends an upstream watch on a halt error with a `Canceled` response that has none of these. So the proxy's clients are never told that the upstream watch ended. They stay open and silent, as in §9.2.

**`RequestProgress` is not supported** ([#19710](https://github.com/etcd-io/etcd/issues/19710), closed "not planned" 2026-07-13).
- The proxy's request switch handles only create and cancel. A progress request falls into the default branch, which logs "not supported request type by gRPC proxy" and sends nothing (`grpcproxy/watch.go:237`, `:293-296`). The reporter hit it with Kubernetes using the proxy as `--etcd-servers`.
- Consequence: through the proxy, Kubernetes' consistent reads cannot rely on the watch to confirm freshness (§11), and the freshness monitor recommended in §13.2 cannot work.
- Periodic progress notifications requested at creation (`WithProgressNotify`) still pass through (`grpcproxy/watcher.go:56-58`).
- (The Antigravity analysis quotes a "TODO: support ProgressRequest" comment; there is no such comment at v3.7.3, but the behaviour it describes is real.)

## 10. Client libraries

### 10.1 Go client: resume logic

**Verified (`client/v3/watch.go:789-892`).** The Go client keeps a next revision, `nextRev`, for each watch and resumes from it after a transient error.
- `nextRev` starts at the requested revision.
- On `Created`, it is replaced by the header revision **only for a "now" watch**. The code comment explains this protects a "now" watch from missing writes made during a reconnect (`:849-860`).
- Otherwise it follows the last event's `ModRevision + 1`, or a progress notification's header revision + 1 (`:862-871`).

That is correct for historical watches. Two edge cases remain for "now" watches, both **Low** and not reported upstream as far as I found.

**Skip window.** The store registers a "now" watcher at member revision R0 (`mvcc/watchable_store.go:141-147`); the `Created` header is computed afterwards, as H ≥ R0 (`v3rpc/watch.go:357-358`). Events in (R0, H] are queued behind `Created` and flushed right after it (`v3rpc/watch.go:538-553`). If the stream breaks between `Created` and those events, the client resumes at H and events in (R0, H) are skipped. This needs concurrent matching writes in the microseconds between registration and header creation, and a break between two consecutive sends on one stream.

**Pre-start event.** The resume point is H itself, not H+1 (`client/v3/watch.go:858-860`), and the start revision is inclusive. If no write landed between registration and header creation (H = R0), a resume after `Created` and before any event can deliver a matching event at revision H, which was committed *before* the watch was registered. It is not a duplicate: the application never received it. But an application that recorded H from `WithCreatedNotify` and expects only revisions above H will see one it did not expect. (Raised by the Antigravity analysis as an ordering/uniqueness violation; I rate it Low, as a pre-start event.)

### 10.2 Go client: the final error can be lost, and teardown can deadlock

Both defects are in how the Go client ends a gRPC watch stream after a fatal error (a halt error or `ErrNoLeader`). Raised by the Codex and Antigravity analyses (the deadlock by Antigravity only); I re-traced both at v3.7.3. Neither was reproduced.

#### (a) The final error is dropped after 250 ms (High, code)

**Effect.** If the application is slow to read, the channel closes without the promised final error. The documentation says a bare close means the caller's own context ended, so an application that follows it can treat the failure as its own shutdown and stop watching for good, with nothing logged.

**What the documentation promises** (`client/v3/watch.go`):
- "If the watch failed and the stream was about to close, before the channel is closed, the channel sends a final response that has Canceled set to true with a non-nil Err()" (`:108-111`).
- For `WithRequireLeader`: "error 'etcdserver: no leader' (ErrNoLeader) will be returned, and then 'WatchChan' is closed with non-nil 'Err()'" (`:78-81`).
- A bare close is the caller's doing: "If the context 'ctx' is canceled or timed out, returned 'WatchChan' is closed, and 'WatchResponse' from this closed channel has zero events and nil 'Err()'" (`:70-71`).

**What the code does** (`client/v3/watch.go`):
- Each watch's application channel holds one response (`make(chan WatchResponse, 1)`, `:558`).
- On a fatal error, `run()` returns. Its deferred cleanup closes each substream's receive channel, and a substream that sees that returns without flushing the responses it has buffered (`:832-836`).
- The cleanup then calls `closeSubstream`, which hands the final error to `sendCloseSubstream`. That waits for the first of: the application accepting the error, the watch's context ending, or `closeSendErrTimeout = 250 * time.Millisecond` (`:39`, `:478-485`). Then it closes the channel.

**Failure schedule.**
1. The application is processing one response.
2. The next response is already in the one-slot channel.
3. The stream fails with `ErrNoLeader`.
4. The application does not return to the channel within 250 ms.

It then reads the queued response, followed by a bare close.

**Notes.**
- A consumer written as `for wr := range wch { ... }` that treats loop exit as shutdown stops watching for good.
- Responses buffered inside the library are discarded too. That is harmless only if the application resumes from its *own* last processed revision.
- A 250 ms pause is ordinary for an application that does I/O per event.
- Only stream-level errors are affected. A compaction cancel goes through the normal blocking path and is not dropped.
- **Kubernetes is protected.** After its range loop ends without an error, it closes `watchClosedCh`, which ends the outer watch, and the reflector re-watches from its own resource version (`apiserver/storage/etcd3/watcher.go:522-526`, `:258-280`). The comment there, "it's only possible that client side ends the watch", is contradicted by this path, but the defence does not depend on it.
- **Status.** `git log v3.7.3..origin/main -- client/v3/watch.go` is empty. I found no upstream issue for this schedule.
- **Test gap.** etcd's partition tests assert `ErrNoLeader` with a consumer already blocked on the channel (`tests/integration/clientv3/connectivity/network_partition_test.go:201-262`), so they cannot see this path.

#### (b) Teardown deadlock when a rejected creation races a fatal error (Moderate; [#17611](https://github.com/etcd-io/etcd/issues/17611))

**Effect.** The stream's teardown blocks forever. Every later `Watch()` call with the same context metadata then blocks until its own context ends; with `context.Background()`, forever. That matches #17611's title, "Watch API blocked forever even with WithRequireLeader option". Moderate: two independent events must coincide within one `select` iteration.

**Mechanism** (`client/v3/watch.go` unless noted).
1. *Trigger.* The server answers a watch creation with `Created` and `Canceled` both set. It does so on authentication or permission failures (for example an expired token on a resume), on a negative start revision, and when the store refuses the watch (`v3rpc/watch.go:270-310`, `:357-365`).
2. *Orphaned substream.* `addSubstream` closes that substream's receive channel and does not register it (`:466-475`); the run loop also clears it from the resuming queue (`:600-607`). Its goroutine then blocks sending itself on the unbuffered `closingc`, before calling `wg.Done()` (`:797-806`).
3. *Race.* Normally the run loop receives from `closingc` (`:685`). If a fatal error is selected first, the deferred cleanup runs (`:520-541`). It drains `closingc` only once per substream it knows about and then waits on `w.wg.Wait()`, which the orphaned goroutine never releases.
4. *Aftermath.* The cleanup never reaches `closeStream`, so the stream stays registered under its context key and its `donec` stays open. A later `Watch()` sends on a request channel no one reads and waits on that `donec` (`:343-363`, `:374-386`).

**Status.** The reporter used a 2020 client against etcd 3.4.16. The issue was closed "not planned" (2025-06-01) and the fix PR [#17622](https://github.com/etcd-io/etcd/pull/17622) closed unmerged (2025-11-10). The structure the race needs is unchanged at v3.7.3.

### 10.3 jetcd: resume can skip historical events (mechanism Moderate; frequency Low; not reproduced)

**Effect.** A jetcd watch that starts in the past, and whose stream breaks after `Created` but before the first batch of historical events arrives, resumes at the member's current revision. The events in between are skipped with no error.

**Mechanism** (`jetcd/impl/WatchImpl.java`).
1. On every `Created`, `WatcherImpl` sets `revision = Math.max(revision, response.getHeader().getRevision())` (`:256-271`; the assignment is at `:267`).
2. On a transient error it resumes with `setStartRevision(this.revision)` (`:164-167`, `:341-348`).
3. For a watch whose requested start R lies *below* the member's current revision H, the `Created` header is H (`v3rpc/watch.go:357-358`). So the resume point jumps to H before any historical event has been delivered.
4. The historical events come later, from the 100 ms sync loop, at most 1000 revisions per batch and 512 watchers per pass (`mvcc/watchable_store.go:40-44`, `:223-255`; `mvcc/watcher_group.go:28`, `:226-239`).
5. If the stream fails in that window, the resumed watch starts at H and every matching event in [R, H) is **silently skipped**. Each further reconnect before the first batch re-arms the skip.

**When the window closes.** When the first historical batch arrives: the event branch assigns `revision = lastEvent.ModRevision + 1` *without* `max()` (`:313-315`), moving the resume point back below H. (The Antigravity analysis states that the skip persists "after only the first batch" arrives; the code says otherwise.)

**When it opens.** On every list-then-watch with writes in between, and on every resume after a disconnect. Both concentrate around failures: when a member dies, its watchers migrate together and grow the unsynced set on the surviving members. A second disruption must land inside the window.

**Not a defect: the progress branch.** It uses `max(revision, header)` without `+1` (`:291-293`). Because of the `max()`, it cannot move the cursor back onto an already-delivered event: after an event at R the cursor is already R+1. The other branch, for empty responses that are not progress notifications (`:294-312`), assigns the header revision without `max()`, but is reachable only for a response whose header revision is 0, which `isProgressNotify` excludes (`jetcd/watch/WatchResponse.java:88-91`). I found no server path that sends one. So I do not count the claimed jetcd progress off-by-one as a defect.

**History and tests.**
- The `max()` was introduced on 2021-07-12 in commit `70be01a`, "Don't overwrite watch revision on start with the latest", for [jetcd#946](https://github.com/etcd-io/jetcd/issues/946). Before that, jetcd overwrote the revision unconditionally, which was worse in both directions. The fix addressed future revisions, not past ones.
- jetcd's resume test watches only from "now" across a cluster restart (`jetcd-core/src/test/java/io/etcd/jetcd/impl/WatchResumeTest.java`).

**Condition under which this fails:** if jetcd consumed historical events before handling `Created`. It cannot, because the server buffers a watch's events until `Created` is sent (`v3rpc/watch.go:468-477`, `:538-553`).

### 10.4 jetcd: watch responses over 4 MiB end the watch (mechanism High; reachability Moderate; not reproduced)

*My finding, from a lead in the Antigravity analysis, which noted only that jetcd lacks fragmentation support.*

**Effect.** A watch response larger than 4 MiB makes jetcd call `onError` and close the watcher. Re-creating the watch at the same revision asks for essentially the same batch and fails again, until the application raises `maxInboundMessageSize`. The Go client is unaffected: its default receive limit is `math.MaxInt32` (`client/v3/options.go:40`).

**Mechanism.**
- *The server's batches can be large.* A catching-up watcher receives up to `watchBatchMaxRevs = 1000` revisions of matching events in a *single* response (`mvcc/watcher_group.go:28`, `mvcc/watchable_store.go:388`). With `PrevKV` each event also carries the previous value. A single revision can be large too: a prefix `DeleteRange` or a lease revocation deletes many keys at one revision.
- *The server splits a response only on request.* It splits responses above `--max-request-bytes` (default 1.5 MiB) only for watches created with `fragment = true` (`v3rpc/watch.go:485-490`; `sendFragments` at `:581-590`). Its own send limit is `math.MaxInt32` (`v3rpc/grpc.go:36`, `:75`).
- *jetcd neither requests fragmentation nor raises the limit.* It never sets `fragment` (`jetcd/impl/WatchImpl.java:164-167`; no reference anywhere in `jetcd-core`), and `WatchOption` has no such setting. It leaves the inbound limit at the gRPC default unless the application sets `ClientBuilder.maxInboundMessageSize` (`jetcd/impl/ClientConnectionManager.java:176-178`). The default is `GrpcUtil.DEFAULT_MAX_MESSAGE_SIZE = 4194304`, read with `javap` from `grpc-core` 1.76.2. jetcd 0.8.7 builds against grpc 1.82.0, whose default I did not read; the 4 MiB in the #1379 report below is consistent with it.
- *jetcd treats the error as fatal.* grpc-java fails the call with `RESOURCE_EXHAUSTED` ("gRPC message exceeds maximum size"; status code recalled, not read from source). jetcd's watcher reschedules only on `UNAVAILABLE` or `INTERNAL` (`jetcd/support/Errors.java:63-65`, `jetcd/impl/WatchImpl.java:353-355`), so it calls `onError` and closes the watcher (`WatchImpl.java:327-351`).

**Evidence it happens.** [jetcd#1379](https://github.com/etcd-io/jetcd/issues/1379) (open; jetcd 0.8.2, etcd 3.4.33): the watch's `onError` received "gRPC message exceeds maximum size 4194304: 6619136". The rest of that thread drifted to other symptoms and was never diagnosed. jetcd#745 (2020) is the same limit hit by a KV range read.

### 10.5 jetcd and the Go client compared

| Aspect | jetcd 0.8.7 | Go client v3.7.3 |
|---|---|---|
| Streams | One gRPC stream per watcher (`jetcd/impl/WatchImpl.java:186-194`) | Multiplexed per context metadata |
| `requireLeader` default | false (`jetcd/options/WatchOption.java:154`) | not set unless `WithRequireLeader` |
| Keepalive default | 30 s / 10 s, without calls (`jetcd/ClientBuilder.java:66-68`) | off (`client/v3/client.go:231`) |
| Balancer | `round_robin` default (`jetcd/impl/ClientConnectionManager.java:200-204`) | gRPC round robin |
| Resume revision after `Created` | `max(requested, header)` for all watches: **skip risk** (§10.3) | header revision only for "now" watches |
| Resume after progress notification | `max(revision, header)` (`WatchImpl.java:291-293`) | header + 1 |
| Transient errors | `onError` to the app **and** automatic reschedule after 500 ms | silent retry |
| Halt condition | any status other than UNAVAILABLE/INTERNAL, or no-leader (`jetcd/support/Errors.java:63-69`) | `isHaltErr` or `ErrNoLeader` |
| Reschedule failure | logged only; watcher left open and dead ([#1227](https://github.com/etcd-io/jetcd/issues/1227), open) | n/a |
| Fragmentation requested | never (`WatchImpl.java:164-167`) | yes; reassembled by the client (`client/v3/watch.go:590-600`) |
| Inbound message limit | gRPC default 4 MiB unless set; overflow is fatal (§10.4) | `math.MaxInt32` (`client/v3/options.go:40`) |
| Final error on stream failure | delivered through `onError` | can be dropped after 250 ms; channel closes bare (§10.2) |
| Reconnect after long outage | resume can wait up to ~2 min for grpc-java's backoff; likely silent under default wait-for-ready ([#1616](https://github.com/etcd-io/jetcd/issues/1616), open; PR #1617 excludes watches; §5.4) | not checked |

**Other jetcd reports (not verified).**
- [jetcd#674](https://github.com/etcd-io/jetcd/issues/674) (and duplicate #673), filed by Jepsen in 2019 with jetcd 0.4.1: creating a watch occasionally threw `IllegalStateException: Stream is already completed`, from `WatchImpl$WatcherImpl.resume`. Closed by the stale bot in 2020 without a fix. jetcd's watch client has since moved to Vert.x streams (`jetcd/impl/WatchImpl.java:44-45`); whether a similar race survives was not checked.
- [jetcd#1352](https://github.com/etcd-io/jetcd/issues/1352): watcher not reconnecting after the leader pod dies; status "waiting-for-feedback" (summarized fetch). The report claims jetcd 0.7.5, but its stack trace names `jetcd-core-0.5.7.jar`, and its error handler opens a new watch from inside `onError` (noted by the Codex analysis; not re-checked by me). Treat it as undiagnosed for 0.8.7.
- [jetcd#1379](https://github.com/etcd-io/jetcd/issues/1379): the original error was the 4 MiB limit (§10.4); after raising it, the reporter found the remaining "silence" was a logging problem, and the thread moved on to KV errors. (*Corrected 2026-11-01*: the first version of this file described it as "stops after 10–15 changes with no error".)
- [etcd#13822](https://github.com/etcd-io/etcd/issues/13822) / [jetcd#1055](https://github.com/etcd-io/jetcd/issues/1055) (2022, closed with no comments): jetcd 0.5.0 watchers received nothing after isolation followed by compaction, on etcd 3.4.13. Consistent with the then-unfixed #17529 or with an unhandled compaction cancel; never diagnosed.

**Testing gap.** jetcd's test launcher defaults to `quay.io/coreos/etcd:v3.5.14` (`jetcd-launcher/src/main/java/io/etcd/jetcd/launcher/Etcd.java:31`). That image predates the fixes for #18089 (3.5.16), #19179 (3.5.18) and #20221 (3.5.22), so jetcd's watch tests run against a server with known watch-guarantee bugs.

## 11. Kubernetes

Kubernetes is etcd's main watch consumer. Its apiserver feeds a *watch cache* from etcd watches, and serves most client watches and many reads from that cache. It defends itself against most of the states above, explicitly.

### 11.1 How the apiserver uses watches (verified at v1.37.2)

| Defence | What it does | Evidence |
|---|---|---|
| Leader requirement and keepalive | Every etcd watch context is wrapped with `WithRequireLeader`; gRPC keepalive is 30 s + 10 s (§5.4). | `apiserver/storage/etcd3/store.go:1156-1165`; `apiserver/storage/storagebackend/factory/etcd3.go:63-64` |
| Explicit start revision | Watches always request `WithRev(initialRev+1)` and `WithPrevKV()`, so they never depend on "now". | `apiserver/storage/etcd3/watcher.go:470-477` |
| Errors end the watch | Any `wres.Err()`, such as compaction or no leader, ends the watch with an error, and list-and-watch clients relist. | `apiserver/storage/etcd3/watcher.go:479-491` |
| Missing previous value is an error | A non-create event with `PrevKv == nil` fails the watch, forcing a relist. | `apiserver/storage/etcd3/event.go:65-67` |
| Bare close ends the watch | If the etcd channel closes without an error, Kubernetes ends its own watch, and the reflector re-watches from its own resource version. This covers the Go client's lost-final-error path (§10.2). | `apiserver/storage/etcd3/watcher.go:522-526`, `:258-280` |
| `RequestProgress` with matching metadata | Progress requests use the watch's own context, so they reach the right stream (§6.3 item 6). | `apiserver/storage/etcd3/store.go:102-106` |

**Why a missing previous value is an error.** [kubernetes#76624](https://github.com/kubernetes/kubernetes/issues/76624) (2019, closed): the apiserver's pod reflector received DELETE events with a nil object and, because it did not relist, "will cause it to *never* remove the object from its cache". The report describes informer clients that believed "hundreds of pods exist that no longer do", which starved scheduling. That was etcd's best-effort `PrevKv` (§2.3, [#10681](https://github.com/etcd-io/etcd/issues/10681)) turned into silent cache corruption by a consumer that skipped the event and carried on.

### 11.2 Consistent reads from cache (KEP-2340; stable since v1.34 per `kep.yaml`)

Kubernetes **routes the freshness check through Raft while leaving delivery local**: the client-side form of the coupling etcd lacks.
1. The apiserver obtains the current revision with a linearizable `Get` (`apiserver/storage/etcd3/store.go:728-745`). The etcd Kubernetes client does not set `WithSerializable` (`client/v3/kubernetes/client.go:48`).
2. It waits until the watch cache reaches that revision, sending `RequestProgress` every 100 ms (`apiserver/storage/cacher/progress/watch_progress.go:36`, `:83-115`).
3. The wait is capped at 3 s (`apiserver/storage/cacher/watch_cache.go:49`, `:318-360`). After that it falls back to reading etcd directly (`apiserver/storage/cacher/delegator.go:160-172`). An alpha gate added in 1.37 can skip the fallback and return 429 instead (`apiserver/features/kube_features.go:363-365`).

**What it depended on etcd fixing.** KEP-2340 on [#15220](https://github.com/etcd-io/etcd/issues/15220): "with the bug manual progress notification cannot be trusted at all. It causes a silent corruption that cannot be automatically detected prior to acting upon the corrupted data." The KEP requires etcd ≥ v3.4.25 / v3.5.8. Later fixes the feature implicitly relies on, Reliable at compaction (§7.3) and Bookmarkable on member switch (§7.1), arrived in 3.5.13 through 3.5.22; no version gate in KEP-2340 tracks them (Moderate; based on the KEP text, not on Kubernetes code).

**`resourceVersion=0` reads are exempt.** "In case resourceVersion is 0, we accept arbitrarily stale result" (`apiserver/storage/cacher/watch_cache.go:333-339`). [#59848](https://github.com/kubernetes/kubernetes/issues/59848) (open since 2018-02) shows the consequence: a kubelet restarted against an apiserver whose watch cache is behind can start a pod whose replacement is already running elsewhere: "This violates pod safety."

### 11.3 Residual risks (Moderate)

- **Not covered by `WithRequireLeader`:** a member with a leader that does not apply (§5.2), and a paused member, detected only by keepalive after up to ~40 s (§5.4).
- **Watchers of the apiserver are not protected.** Consistent reads have the 3 s timeout and fallback; a client watching through the apiserver just keeps receiving nothing.
- **The protection has a cost:** mass watcher termination when an etcd election is prolonged ([#111116](https://github.com/kubernetes/kubernetes/issues/111116); closed, resolution not checked).
- **Through the gRPC proxy** (§9): `WithRequireLeader` is ignored, `RequestProgress` is dropped ([#19710](https://github.com/etcd-io/etcd/issues/19710), reported with Kubernetes), and dead broadcasts are reused (#21813). With `RequestProgress` dropped, a consistent read can be served from the cache only if events for that resource happen to bring the cache up to the target revision. Otherwise, on any resource quieter than the cluster as a whole, it waits out the 3 s timeout and falls back to etcd, or returns 429 with the 1.37 alpha gate (inferred from the code in §11.2; not tested through a proxy). The silent-watch defects remain.
- **Slow direct watches** ([#123448](https://github.com/kubernetes/kubernetes/issues/123448), closed 2026-06-01 as tracked elsewhere, not as fixed; follow-ups #130939 and #138264). An inactive client watch that bypasses the watch cache can grow buffers in both the apiserver and etcd, starve the watch cache's own etcd watch, and break it through `prevKV=nil` errors (§8.1).

## 12. Design: comparison with Chubby, and the hypothesis

### 12.1 Chubby comparison

Source: `doc/chubby-osdi06.md`, the Markdown conversion of the Chubby paper in this project. On event loss across fail-over etcd is stronger than Chubby; on coupling events to liveness it is weaker.

| Axis | Chubby | etcd v3.7.3 |
|---|---|---|
| Events after the action | Guaranteed: "Events are delivered after the corresponding action has taken place. Thus, if a client is informed that file contents have changed, it is guaranteed to see the new data (or data that is yet more recent) if it subsequently reads the file." (§2.5, p. 5). | Only for **linearizable** reads. A serializable read on a member behind the watch's member can return older data than the event. |
| Events across fail-over | **Lossy by design.** One event type is "Chubby master failed over—warns clients that other events may have been lost, so data must be rescanned" (§2.5, p. 5). The new master "emits a fail-over event to each session; this causes clients to flush their caches (because they may have missed invalidations), and to warn applications that other events may have been lost" (§2.9 step 5, p. 9). In hindsight: "many developers chose to crash their applications on receiving this event ... We might have done better to send redundant 'file change' events instead, or even to ensure that no events were lost during a fail-over" (§4.6, p. 14). | Revisioned history plus Resumable **is** "ensure that no events were lost during a fail-over". Stronger than Chubby (High). |
| Liveness coupling | Events ride on session KeepAlive replies: "the KeepAlive reply is used to transmit events and cache invalidations ... Piggybacking events on KeepAlive replies ensures that clients cannot maintain a session without acknowledging cache invalidations" (§2.8, p. 7). A **jeopardy** event starts the grace period, and the library blocks API calls so the application cannot observe inconsistent data (§2.8, p. 8). | A watch stream is independent of any session or lease. Its only staleness signals are `WithRequireLeader` (coarse, opt-in) and progress notifications (local). |
| Cache consistency | "The modification proceeds only after the server knows that each client has invalidated its cache"; weaker models were rejected because "we felt that programmers would find them harder to use" (§2.7, p. 7). | Writes never wait for watchers. Every watch-fed cache (Kubernetes' watch cache, `client/v3/leasing`, `cache/`) is eventually consistent and must bound its staleness with linearizable reads, as KEP-2340 does. |
| What events are for | A convenience. Attempts to use them for publish/subscribe failed: "Chubby's heavyweight guarantees and its use of invalidation rather than update in maintaining cache consistency make it a slow and inefficient for all but the most trivial publish/subscribe examples" (§4.5, pp. 13–14, `doc/chubby-osdi06.md:516-524`; point raised by the Antigravity analysis, quote checked). | The replication feed for downstream caches. |
| Who serves events | The master only. | Any member, and proxies. That spreads fan-out, at the cost that "now" and bookmark revisions are per-member quantities. |

The last two rows explain why etcd's watch defects matter more than Chubby's lost events did. A Chubby client that missed an invalidation still had its cache flushed by the fail-over event. A watch-fed cache that misses an event, or never learns that its feed ended, stays wrong.

### 12.2 The design hypothesis

**Hypothesis.** "Watches not going through Raft is a design mistake that makes them incorrect on leader failover."

**Strongest case for.**
- A watch is the primary way Kubernetes-style systems learn about change, yet nothing in the protocol lets a client distinguish "no changes" from "my member is cut off or stuck". The remedies are opt-in or client-built: `WithRequireLeader`, which leaves gaps (§5.2, §5.4); keepalive; and comparing linearizable reads against `RequestProgress`, as Kubernetes does.
- The documented contract makes the failure *legal* ("might never arrive"), so contract testing cannot catch it. Legal here means only that no guarantee on the API page covers it, even with the quorum intact; etcd's own eventual-consistency claim does (§3; §2.2 caveat).
- The bug record shows the member-local catch-up machinery has been the least reliable part of etcd's consistency story for years (§7, Appendix A).
- A watch API that required, say, periodic linearizable progress, or a stream lease, would have made silent staleness detectable by construction.

**Strongest case against.**
- Ordering and completeness hold across failover precisely *because* watches replay Raft-committed history by revision.
- Routing watch creation through ReadIndex would only make the *start point* linearizable; it would not stop a member from becoming isolated a second later.
- The defect is not a failover property. It is a missing freshness and liveness signal on a long-lived local stream, which a follower partition or a stuck apply produces just as well.
- Chubby, the reference design, did *worse* on event loss across fail-over. It did better only on liveness coupling.

**Assessment (Moderate; revised 2026-11-03).** The hypothesis locates the mistake correctly and overstates what it breaks. The mistake is the part of the watch path that bypasses Raft: the watch protocol has **no liveness or freshness coupling to Raft progress**: no server-asserted "applied ≥ recent ReadIndex", and no stream lease. It breaks eventual delivery, not ordering or completeness, and it does so on any cut-off or stalled serving member, not only on failover (§3). Combined with defaults that leave even the existing signals off (`WithRequireLeader` off, Go keepalive off, progress notifications every 10 min), this makes silent staleness the default failure mode. §15.4 states what evidence would overturn this.

## 13. Recommendations and fix directions

### 13.1 For operators

1. **Run etcd ≥ v3.7.1** (or ≥ 3.6.14 / 3.5.33). These carry every fix in Appendix A, including the watch-auth fix. Treat 3.5 < 3.5.22 and 3.6 < 3.6.2 as having known watch-guarantee bugs.
2. **Do not put the gRPC proxy in front of watches that must not miss events** ([#21813](https://github.com/etcd-io/etcd/issues/21813)) **or that need failure and freshness signals**: it drops `WithRequireLeader` and `RequestProgress` and swallows upstream errors (§9). Connect such clients directly to members.
3. **Restore snapshots with `--bump-revision <large> --mark-compacted`** (§7.4).
4. **TLS deployments with heavy range reads:** consider `--listen-client-http-urls` to serve gRPC on a listener separate from HTTP ([#15402](https://github.com/etcd-io/etcd/issues/15402); whether v3.7.3 still needs it is untested, Low–Moderate; §8.1).

### 13.2 For every watch client

5. **Resume from your own cursor, stored with your state.** Persist the last applied revision atomically with whatever the application derives from events. The library's internal resume point (Go `nextRev`, jetcd `revision`) records receipt, not application, and is lost on restart or `ErrNoLeader` (§2.4, §6.3). Take the cursor from the events' `ModRevision` (or a progress notification), never from an event batch's header revision, which runs ahead while a watcher catches up; Jepsen's harness hit exactly this (§2.5).
6. **Never re-create a watch from "now" after an error.** Watch from an explicit revision (list, then watch from revision + 1, as Kubernetes does), or request `WithCreatedNotify` and record the `Created` header revision. Resume from the last revision you received: event, bookmark or `Created` (§6.3 item 4). To replay from the beginning, pass 1, not 0 (§2.5).
7. **Always set `WithRequireLeader`** (Go) or `withRequireLeader(true)` (jetcd), and handle `ErrNoLeader` by re-watching from your last revision + 1.
8. **Build a freshness monitor** where freshness matters; nothing else detects §5.2 or a congested stream (Moderate).
   - Periodically read a linearizable revision R, for example `Get(key, WithLimit(1))` and its header revision.
   - Request progress, and require that the watch reach R, through an event or a bookmark, within a deadline T.
   - Otherwise, cancel and re-watch, preferably on another endpoint. If the re-watch itself blocks (#17611; §10.2), create a new client.
   - In Go, call `RequestProgress` with the same context metadata as the watch, or it reaches a stream with no watchers and does nothing (§6.3 item 6).
9. **Treat a compaction error as "relist"**, and a missing `PrevKv` as an error if your logic needs the old value (§7.3).
10. **Keep unrelated watchers on separate streams** (different context metadata in Go), so one slow watcher does not starve progress or events for the others (§8).
11. **Raw gRPC clients: do not reuse explicit `watch_id`s** ([#22673](https://github.com/etcd-io/etcd/issues/22673); §8.2).

### 13.3 Go client

12. **Enable client keepalive** (`DialKeepAliveTime` / `DialKeepAliveTimeout`); it is off by default (§5.4).
13. **Treat every channel close as a failure unless you cancelled the context yourself** (§10.2). Do not rely on a final `Canceled` response or a non-nil `Err()`. On an unexpected close, resume from your own last processed revision + 1.
14. **Call `RequestProgress` with a context carrying the same metadata as the watches**, e.g. `WithRequireLeader(...)`. Otherwise the request silently goes to a different stream (§6.3 item 6).

### 13.4 jetcd

15. **Raise `ClientBuilder.maxInboundMessageSize` well above 4 MiB** (§10.4). The Go client's default is `MaxInt32`. A watch that fails with "message exceeds maximum size" will fail again at the same revision until you do.
16. **Until the resume skip is fixed (§10.3):** track the last delivered `ModRevision` in the application, and close and re-create watchers with an explicit revision on `onError` instead of relying on internal resume. Alternatively, start historical watches only after confirming the cluster revision.
17. jetcd's keepalive defaults are acceptable (§5.4).

### 13.5 Fix directions for upstream (not implemented)

| Component | Fix | For |
|---|---|---|
| gRPC proxy | Use `metadata.FromIncomingContext` at `grpcproxy/watch.go:112` and `grpcproxy/lease.go:161`, and add an e2e test through a networked proxy. Better still, forward the flag to the upstream broadcast watch, so the member serving the events cancels it itself and the second flaw goes away. | §9.1 |
| gRPC proxy | Remove a broadcast from its registry (`bcasts`) when its upstream goroutine exits; make `add` / `coalesce` skip broadcasts whose `donec` is closed; forward `Canceled` responses to client watchers; implement `ProgressRequest` (#19710). | §9.2, §9.3 |
| Go client | Put the resume revision in the closing response (for example `Header.Revision = nextRev-1`), or expose it otherwise. At minimum, document that "from now" watches combined with `WithRequireLeader` need `WithCreatedNotify` or an explicit revision. | §6.3 |
| Go client | Deliver the final error with a blocking send that ends only on the watch's own context, instead of the 250 ms timer, or record it so a bare close can be told apart from cancellation. Drain `closingc` for substreams rejected at creation (#17611). | §10.2 |
| jetcd | Either raise the inbound limit for watch channels, or set `fragment` on watch creation, and expose the choice in `WatchOption`. Fragmentation also needs jetcd to reassemble fragments before delivering them or advancing its cursor; today a fragment would be treated as a complete response, and a resume between fragments would skip the rest of that revision. | §10.4 |
| Server | Base stream cancellation on apply progress rather than leader presence, for example cancel when commit − apply grows or apply stalls while commit advances. Proposal [#22635](https://github.com/etcd-io/etcd/issues/22635) defines these health dimensions (`apply_stall`, `apply_lag`) but does not tie them to stream cancellation. | §5.2, §6.3 |

## 14. Open questions and proposed tests

Per AGENTS.md §7, the tests below would run on this host only as functional tests; timings from this host are not findings.

### 14.1 Proposed functional tests

| # | Test | Setup and expected result | Tests § |
|---|---|---|---|
| 1 | jetcd resume skip | 3-member v3.7.3 cluster in Docker; jetcd 0.8.7 through a fault-injecting TCP proxy such as toxiproxy. Write N keys, then watch from revision R < current. Cut the stream right after the `Created` frame, before the first sync batch (up to 100 ms later), then restore it. Expected: events in [R, H) missing. If not, find the code path that prevents it. | 10.3 |
| 2 | jetcd over-size response | Write 1000 revisions of about 8 KiB each under a prefix; open a default jetcd 0.8.7 watch from before them. Expected: `onError` with `RESOURCE_EXHAUSTED`, then the watcher closes. Repeat with `maxInboundMessageSize(64 MiB)` to confirm the fix. | 10.4 |
| 3 | Go lost final error | Extend `testBalancerUnderNetworkPartitionWatch` (`tests/integration/clientv3/connectivity/network_partition_test.go:201-262`): keep one response unread in the application channel, induce `ErrNoLeader`, drain only after 300 ms. Expected: a bare close with no `Canceled` response. (Proposed by the Codex analysis; my code trace agrees.) | 10.2 |
| 4 | Go teardown deadlock | Unit test with a fake `WatchClient`: deliver `Created` + `Canceled` with a cancel reason for a resuming watch, immediately followed by a halt error. Then check that a second `Watch()` on the same metadata returns. | 10.2 |
| 5 | Proxy dead broadcast via compaction | Through `etcd grpc-proxy`, open watch A on range X at an already-compacted revision. Before A's client cancels, open watch B on X at a recent revision, and let a third watch C on X receive events. Expected: B can be attached to A's dead broadcast, or C merged into it, and then receives nothing more, with no error. The proxy picks broadcasts in map order, so repeat the run. | 9.2 |
| 6 | `WithRequireLeader` latency, remaining cases | Done for an isolated *leader* (3.3 s and 4.3 s, Appendix C). Still open: an isolated *follower*, and the reconnect-during-election termination window. | 5.1, 6.3 item 5 |
| 7 | Leader present, apply stalled | A gofail failpoint in the apply path, or the #22184 deterministic test adapted to an e2e cluster. Confirm that a `WithRequireLeader` watch on that member stays silent while the member keeps a leader. For the blocked raft loop, set the existing `raftBeforeSave` failpoint (`etcdserver/raft.go:255`) to sleep on one follower, then partition that follower; expected: its `WithRequireLeader` watches stay open and silent, with client keepalive on, for as long as the sleep lasts. | 5.2 |
| 8 | #15402 on v3.7.3 | Is a watch on a shared TLS listener still starved by concurrent large ranges with Go 1.26? | 8.1 |

### 14.2 Open questions

9. **#22184 reachability (§5.3).** Is there any goroutine other than the apply goroutine that calls `store.mu.Lock()` while a write transaction can be open? My survey found none. A run with the race detector and lock-order instrumentation under robustness traffic would settle it.
10. **Go "now"-watch skip window (§10.1).** Is it worth an upstream issue? Fixing it would need the server to report the registration revision in `Created`.
11. **Robustness coverage.** Do etcd's robustness or Antithesis suites model a stuck apply with a leader present (§5.2) or a restore without a revision bump (§7.4)? Neither appears in the track record. Since PR #20240 the suite also no longer checks that progress notifications arrive at all (Appendix B).
12. **[#20573](https://github.com/etcd-io/etcd/issues/20573) (§7.2).** It was closed without a root cause. Is the snapshot-related Resumable failure gone, or merely unobserved?
13. **Reporting the proxy `WithRequireLeader` bug (§9.1).** Not yet reported upstream. Filing it is the user's decision: AGENTS.md §9 forbids agents opening PRs against `etcd-io`. A networked-proxy e2e test would also show whether the second flaw in §9.1 matters in practice.
14. **Post-heal delay in Docker run 1 (Appendix C).** In run 1 the proxied watcher had not received the partition-time write 8 s after the heal; in run 2 it got it 0.14 s after the heal. Not diagnosed.

## 15. Conclusions

### 15.1 On the premise

The premise is right about liveness and wrong about safety (revised 2026-11-03; §3).
- **Liveness: right.** etcd claims eventual consistency for watches, and watches break it with the quorum intact. The watch path never consults Raft's quorum, so a member cut off from it, such as a deposed leader after a partition-induced failover, keeps its watches open and silent until it rejoins, which may be never (§3, §5.1). `WithRequireLeader` closes that case only; with it, a member that keeps a leader but does not apply, a blocked raft loop, a frozen stream, the Go client's teardown and error-delivery defects, and the gRPC proxy still leave watches silent (Summary; §5.2, §6.3). What escapes Raft is the answer to "is this watch up to date, and is it still moving?": the start point, the `Created` revision and every bookmark are one member's local view (§1). — **High**.
- **Safety: wrong.** Watch events are derived from Raft-committed entries, so leader failover cannot make a watcher see uncommitted data, and a client that resumes from its own last revision loses nothing on any member (§1, §3). Jepsen's etcd 3.4.3 tests reached the same result independently, for single keys without compaction (§2.5). The failover bugs that did break safety guarantees were bugs in member-local recovery code, and all known ones are fixed (§7). — **High**.
- **Failover is a trigger, not the condition.** Cut-off followers, members that have a leader but do not apply, the gRPC proxy and the client libraries break eventual delivery without any failover (§4). — **High**.

### 15.2 On the design hypothesis

The design weakness is real, and it sits where the watch path bypasses Raft (revised 2026-11-03). The replicated history a watch replays does go through Raft; that is what makes watches resumable, and on that axis etcd improved on Chubby (§12.1). What bypasses Raft is the stream's liveness and freshness. A watch stream carries no proof of either: no stream lease, no server-asserted "applied ≥ recent ReadIndex", and a contract that explicitly permits silence (§2.2, §12.2), even in a cluster that keeps its quorum. That leaves the contract safety-only and at odds with etcd's claim of eventual consistency (§2.2 caveat; [`etcd-issues-and-mitigations-claude.md`](etcd-issues-and-mitigations-claude.md) §3). The defaults then switch off the signals that do exist (`WithRequireLeader`, Go keepalive, periodic progress). Silent staleness is therefore the default failure mode, and every robust consumer, Kubernetes included, has to rebuild the missing coupling on the client side with linearizable reads (§11.2). — **Moderate**.

### 15.3 On production readiness at v3.7.3

- **Direct server path: sound for safety, weak for liveness.** No open server defect silently breaks ordering or completeness for clients using automatic watch IDs. The open server issues delay delivery (§8.1), freeze shared streams under cancellation bursts (§8.1), or, if a trigger exists, wedge the apply loop (§5.3). A member whose apply or raft loop is stuck keeps its watches silent even under `WithRequireLeader` (§5.2). — **Moderate**.
- **gRPC proxy: not fit for watches that need failure or freshness signals.** It ignores `WithRequireLeader` (reproduced), reuses dead upstream watches, swallows upstream errors and drops progress requests (§9). — **High** (code); Moderate for how often #21813 bites.
- **Go client: correct resume logic, unreliable failure signalling.** It can drop its final error and close bare, can deadlock on teardown in a narrow race, retries forever with no keepalive by default, and discards its resume point on `ErrNoLeader` (§5.4, §6.3, §10.2). — **High** (code).
- **jetcd: the weakest layer for completeness.** It can silently skip historical events on resume (§10.3), ends watches whose responses exceed 4 MiB (§10.4), can stop retrying after one failed resume (§5.4), and its tests run against an etcd image with known watch bugs (§10.5). — **Moderate** to **High** per item.
- **Kubernetes: well defended on the direct path.** Its remaining exposure is apply stalls, paused members, the proxy, and `resourceVersion=0` reads (§11.3). — **Moderate**.

The practical recipe is in §13. Its core is to resume only from an application-owned revision, to treat every unexpected close as failure, and to verify freshness against a linearizable read.

### 15.4 What would change these conclusions

- **The premise verdict (§15.1)** has two halves.
  - The safety half would fail if a reproducible case showed a client, resuming correctly from its own last revision on a current release, receiving an out-of-order, duplicated or missing event across a leader change without a compaction error. None was found; [#20573](https://github.com/etcd-io/etcd/issues/20573), closed without a root cause, is the closest candidate (§7.2).
  - The liveness half would fail only if eventual consistency were read classically, as conditional on every failed member eventually returning (§3). Even then, #21813, #17611 and an unnoticed dead connection would remain violations, because they persist after every fault has healed.
- **The design assessment (§15.2)** would fail if a client using `WithRequireLeader`, keepalive, and a periodic linearizable-read-versus-progress check could still miss events silently on a current release, connected directly to members. On the evidence collected, the only such paths are:
  - the jetcd resume skip (§10.3);
  - the Go client's bare close after a fatal error (§10.2), which does not affect an application that resumes from its own revision on every close;
  - watch-ID reuse by raw-gRPC clients ([#22673](https://github.com/etcd-io/etcd/issues/22673); §8.2).

  Through the gRPC proxy the check itself cannot be built, because `RequestProgress` is dropped, and #21813 can silence watches (§9). The 2026-11-01 revision found more defects in the client libraries and the proxy, but none in the direct server path that would change this assessment.
- **The #22184 rating (§5.3)** would rise to High if a concurrent `store.mu` writer outside the apply goroutine were identified (§14.2 item 9).

## Appendix A. Fixed watch-guarantee violations

Sources: etcd's robustness track record (`tests/robustness/README.md:19-39`) and the issues. Fix versions verified with `git tag --contains` on release-branch commits, unless marked. Discussed in §7.

| Issue | Guarantee broken | Trigger | Introduced | Fixed in |
|---|---|---|---|---|
| [#8411](https://github.com/etcd-io/etcd/issues/8411) | Reliable (total silence) | Member restores leader snapshot after long partition | v3.1.x | 2017 (`watchableStore.Restore`; release not verified) |
| [#15220](https://github.com/etcd-io/etcd/issues/15220) | Bookmarkable | Requested progress notification overtakes events | ≤ v3.4 | v3.4.25, v3.5.8 (per KEP-2340); main → v3.6.0 |
| [#15271](https://github.com/etcd-io/etcd/issues/15271) | Ordered/Unique (time travel) | Snapshot restore after partition | ≤ v3.4 | v3.4.25, v3.5.8 |
| [#17247](https://github.com/etcd-io/etcd/issues/17247) | Unique | TXN caching bug | main only | never released |
| [#17507](https://github.com/etcd-io/etcd/issues/17507) | Progress liveness | Regression from #15237 | v3.5.8 | v3.5.13 |
| [#17529](https://github.com/etcd-io/etcd/issues/17529) | Reliable (silent skip) | Congested stream + compaction | ≤ v3.4 | v3.4.31, v3.5.13 |
| [#18089](https://github.com/etcd-io/etcd/issues/18089) | Reliable | Compaction on a delete tombstone | ≤ v3.4.0 | v3.4.34, v3.5.16 |
| [#19179](https://github.com/etcd-io/etcd/issues/19179) | Resumable | Same, via crash/restore | ≤ v3.4 | v3.4.36, v3.5.18 |
| [#19487](https://github.com/etcd-io/etcd/issues/19487) / [#20221](https://github.com/etcd-io/etcd/issues/20221) (events) | Start revision / Ordered | Future watch + restore / member switch | ≤ v3.4 | v3.5.22, v3.6.2, v3.7.0 |
| [#20221](https://github.com/etcd-io/etcd/issues/20221) (notifications) | Bookmarkable | Future watch on lagging member | ≤ v3.4 | v3.5.22, v3.6.2, v3.7.0 |
| watch auth (`afeaa624d`) | Access control | `\x00` range end | ? | v3.5.33, v3.6.14, v3.7.1 |

**Adjacent: [#20418](https://github.com/etcd-io/etcd/issues/20418), "Stale reads caused by process pausing"** (Antithesis, 2025).
- Cause: ReadIndex retries reused the same request ID, so a stale ReadIndex response could satisfy a later linearizable read. Fix: "Don't reuse same ReadIndex in retries", #21399, in v3.7.0, v3.6.9, v3.5.28 and v3.4.42 (verified).
- Why it matters here: clients and Kubernetes judge watch freshness by comparing against linearizable reads (§11.2). While this bug was present, that comparison could itself be stale.

## Appendix B. Open upstream items relevant to watch promises

As of 2026-10-31; rows added 2026-11-01. "Silent?" means whether the client gets no error.

| Issue | Effect | Silent? | § |
|---|---|---|---|
| [#22673](https://github.com/etcd-io/etcd/issues/22673) | Dead watch's events delivered to a watch reusing its ID; unbounded `pending` | Yes | 8.2 |
| [#22294](https://github.com/etcd-io/etcd/issues/22294) | Compacted watchers not retired with ≥512 unsynced; repeated cancels | No | 7.3 |
| [#22264](https://github.com/etcd-io/etcd/issues/22264) | Far-behind watcher stalls writes; quadratic rescans | Delay | 8.1 |
| [#22184](https://github.com/etcd-io/etcd/issues/22184) | Lock-order inversion could wedge apply → all watches on the member silent (no confirmed trigger) | Yes | 5.3 |
| [#21813](https://github.com/etcd-io/etcd/issues/21813) | Proxy watchers attached to, or merged into, a dead broadcast | Yes | 9.2 |
| [#19710](https://github.com/etcd-io/etcd/issues/19710) (closed not planned) | Proxy drops `RequestProgress` | Yes | 9.3 |
| [#16839](https://github.com/etcd-io/etcd/issues/16839) | Watch starvation, multi-second delays, OOM | Delay | 8.1 |
| [#18879](https://github.com/etcd-io/etcd/issues/18879), [#20716](https://github.com/etcd-io/etcd/issues/20716) (closed not planned); PR [#21065](https://github.com/etcd-io/etcd/pull/21065) open | Cancellation burst blocks a shared stream in both directions | Yes | 8.1 |
| [#17611](https://github.com/etcd-io/etcd/issues/17611) (closed not planned; PR #17622 unmerged) | Go client teardown deadlock; later `Watch()` calls block | Yes | 10.2 |
| (none found) | Go client drops the final error after 250 ms; bare close | Yes | 10.2 |
| [#20240](https://github.com/etcd-io/etcd/pull/20240) (merged 2025-06-30) | Robustness tests no longer check that progress notifications *arrive*; only the Bookmarkable ordering check remains (`tests/robustness/validate/watch.go:113-132`) | n/a (coverage) | 14.2 |
| [#10681](https://github.com/etcd-io/etcd/issues/10681) | `PrevKv` silently omitted if compacted | Yes | 2.3 |
| [#22614](https://github.com/etcd-io/etcd/issues/22614) | `cache/` library stuck not-ready after compaction | No (not ready) | 7.3 |
| [#22635](https://github.com/etcd-io/etcd/issues/22635) | Proposal: health entries incl. `apply_stall`, `linear_read` | n/a | 5.2 |
| [#22268](https://github.com/etcd-io/etcd/issues/22268) | Leader-aware balancer; `leader_id` in `ResponseHeader` is on `origin/main` (`ee374bdc1`), unreleased, "advisory ... not a fencing token" | n/a | — |
| [#15820](https://github.com/etcd-io/etcd/issues/15820), [#20349](https://github.com/etcd-io/etcd/issues/20349), [#20622](https://github.com/etcd-io/etcd/issues/20622) | Robustness coverage gaps for the Kubernetes contract and watch revision windows | n/a | — |

No fix for any open item above was on `origin/main` as of 2026-10-28 (verified with `git log`; re-checked for the 2026-11-01 additions).

## Appendix C. Docker reproduction of the proxy `WithRequireLeader` bug (2026-10-31; functional only)

Supports §9.1 and the timings in §5.1. Image: `gcr.io/etcd-development/etcd:v3.7.3` (Git SHA `0a173d0`, Go 1.26.10).

**Topology.**
- Two Docker networks:
  - `peer`: etcd1–3.
  - `cli`: etcd1's second interface, `etcd grpc-proxy start --endpoints=http://etcd1:2379`, and two `etcdctl watch foo` clients. One watcher is direct to etcd1; the other goes through the proxy. Both set `WithRequireLeader`.
- The partition is `docker network disconnect peer etcd1`: etcd1 is cut off from its peers but still reachable by the clients.

**Steps.**
1. `put foo v1` through etcd2. Both watchers print it.
2. Partition etcd1.
3. `put foo v2` through etcd2, about 2 s later.
4. Observe for about 27 s.
5. Heal with `docker network connect peer etcd1`.

**Results.** Times are measured from the partition.

| | Run 1 | Run 2 |
|---|---|---|
| Direct watcher | `watch was canceled (etcdserver: no leader)` at **+3.3 s**, exit 5 | same at **+4.3 s** |
| Proxy watcher during partition | running at +26.7 s, never printed v2, no error | running at +27.7 s, never printed v2, no error |
| Proxy watcher after heal | v2 not printed within 8 s (not diagnosed) | v2 printed 0.14 s after heal. `put v3` delivered. A fresh `watch --rev=1` through the proxy shows v1, v2, v3 (history intact) |

**Notes.**
- In run 1, etcd1 was the leader (`endpoint status`); run 2's initial roles were not captured.
- The scripts were in that session's scratchpad. The commands above are enough to rebuild them; they use only the official image.
- These are single functional observations on this host (AGENTS.md §7), not performance figures.

## Appendix D. Sources and upstream references

### D.1 Sources

- **etcd server and Go client code at v3.7.3**, read directly:
  - the watch server: `v3rpc/watch.go`;
  - the leader monitor: `v3rpc/interceptor.go`;
  - the MVCC watch store: `mvcc/{watchable_store.go,watcher_group.go,watcher.go,watchable_store_txn.go,kvstore.go}`;
  - the raft loop and apply path: `etcdserver/{raft.go,server.go}`;
  - the Go client: `client/v3/watch.go`;
  - the gRPC proxy: `grpcproxy/{watch.go,watcher.go,watch_broadcast.go,watch_broadcasts.go,watch_ranges.go,leader.go,lease.go}`.
- **GitHub issues.**
  - Searched through the GitHub REST search API: open etcd issues matching "watch", plus targeted searches (partition, leader, stale, missing or lost events, `WithRequireLeader`, progress notify, bookmark), jetcd watch issues, and Kubernetes etcd-watch issues.
  - Issue bodies and many comment threads were read raw from the API. The 2026-11-01 revision added 18 raw reads of issue, PR and comment records, plus 3 searches.
  - The unauthenticated API quota ran out during the first session (other agents share this IP). After that, a few threads were read through a web-fetch tool that **summarizes pages with a smaller model**: etcd #15220, #16839 and #20221; Kubernetes #89488, #97294 and #123072; jetcd #1352. Claims from those threads are marked "(summarized fetch)" and are weaker evidence than raw reads.
- **etcd's robustness track record** (`tests/robustness/README.md:19-39`): etcd's own ledger of correctness bugs found by its fault-injection tests (§7).
- **Peer analyses** (2026-11-01 revision): `analysis/watchers-codex.md` and `analysis/watchers-agy.md`, used only as leads (Appendix E).
- **Jepsen** (2026-11-09 revision): the [etcd 3.4.3 report](https://jepsen.io/analyses/etcd-3.4.3) and its test source at `jepsen-io/etcd` @ `16149cb` (`src/jepsen/etcd.clj`, `src/jepsen/etcd/{watch,client}.clj`, `project.clj`), fetched raw from GitHub (§2.5).

### D.2 Upstream references

**etcd, open:**
[#22673](https://github.com/etcd-io/etcd/issues/22673),
[#22294](https://github.com/etcd-io/etcd/issues/22294),
[#22264](https://github.com/etcd-io/etcd/issues/22264),
[#22184](https://github.com/etcd-io/etcd/issues/22184),
[#21813](https://github.com/etcd-io/etcd/issues/21813),
[#22614](https://github.com/etcd-io/etcd/issues/22614),
[#18879](https://github.com/etcd-io/etcd/issues/18879),
[#16839](https://github.com/etcd-io/etcd/issues/16839),
[#10681](https://github.com/etcd-io/etcd/issues/10681),
[#22635](https://github.com/etcd-io/etcd/issues/22635),
[#22214](https://github.com/etcd-io/etcd/issues/22214),
[#22326](https://github.com/etcd-io/etcd/issues/22326),
[#22268](https://github.com/etcd-io/etcd/issues/22268),
[#15820](https://github.com/etcd-io/etcd/issues/15820),
[#20349](https://github.com/etcd-io/etcd/issues/20349),
[#20622](https://github.com/etcd-io/etcd/issues/20622);
PR [#21065](https://github.com/etcd-io/etcd/pull/21065) (open fix for #18879/#20716).

**etcd, closed without a fix** (code unchanged at v3.7.3):
[#17611](https://github.com/etcd-io/etcd/issues/17611) (PR [#17622](https://github.com/etcd-io/etcd/pull/17622) closed unmerged),
[#19710](https://github.com/etcd-io/etcd/issues/19710),
[#20716](https://github.com/etcd-io/etcd/issues/20716).

**etcd, closed:**
[#8411](https://github.com/etcd-io/etcd/issues/8411),
[#8980](https://github.com/etcd-io/etcd/issues/8980),
[#9988](https://github.com/etcd-io/etcd/issues/9988),
[#13822](https://github.com/etcd-io/etcd/issues/13822),
[#14303](https://github.com/etcd-io/etcd/issues/14303),
[#15220](https://github.com/etcd-io/etcd/issues/15220),
[#15271](https://github.com/etcd-io/etcd/issues/15271),
[#15402](https://github.com/etcd-io/etcd/issues/15402),
[#17247](https://github.com/etcd-io/etcd/issues/17247),
[#17507](https://github.com/etcd-io/etcd/issues/17507),
[#17529](https://github.com/etcd-io/etcd/issues/17529),
[#18089](https://github.com/etcd-io/etcd/issues/18089),
[#18231](https://github.com/etcd-io/etcd/issues/18231),
[#19179](https://github.com/etcd-io/etcd/issues/19179),
[#19487](https://github.com/etcd-io/etcd/issues/19487),
[#20221](https://github.com/etcd-io/etcd/issues/20221),
[#20242](https://github.com/etcd-io/etcd/issues/20242),
[#20418](https://github.com/etcd-io/etcd/issues/20418),
[#20573](https://github.com/etcd-io/etcd/issues/20573),
[#12534](https://github.com/etcd-io/etcd/issues/12534);
PRs #15237, #17555, #18274, #20229, #20281, #21399, [#20240](https://github.com/etcd-io/etcd/pull/20240).

**etcd, `WithRequireLeader` history (§6.2):**
PRs [#5324](https://github.com/etcd-io/etcd/pull/5324), [#5328](https://github.com/etcd-io/etcd/pull/5328), [#5332](https://github.com/etcd-io/etcd/pull/5332), [#7896](https://github.com/etcd-io/etcd/pull/7896), [#9872](https://github.com/etcd-io/etcd/pull/9872);
issues [#5354](https://github.com/etcd-io/etcd/issues/5354), [#7247](https://github.com/etcd-io/etcd/issues/7247), [#8673](https://github.com/etcd-io/etcd/issues/8673), [#10819](https://github.com/etcd-io/etcd/issues/10819), [#12254](https://github.com/etcd-io/etcd/issues/12254);
lease-side issues still open: [#22081](https://github.com/etcd-io/etcd/issues/22081), [#22094](https://github.com/etcd-io/etcd/issues/22094).

**jetcd:**
[#745](https://github.com/etcd-io/jetcd/issues/745),
[#946](https://github.com/etcd-io/jetcd/issues/946),
[#1055](https://github.com/etcd-io/jetcd/issues/1055),
[#1227](https://github.com/etcd-io/jetcd/issues/1227),
[#1352](https://github.com/etcd-io/jetcd/issues/1352),
[#1379](https://github.com/etcd-io/jetcd/issues/1379),
[#1616](https://github.com/etcd-io/jetcd/issues/1616) / PR [#1617](https://github.com/etcd-io/jetcd/pull/1617).

**Kubernetes:**
[#59848](https://github.com/kubernetes/kubernetes/issues/59848),
[#76624](https://github.com/kubernetes/kubernetes/issues/76624),
[#89488](https://github.com/kubernetes/kubernetes/issues/89488) / PR [#89881](https://github.com/kubernetes/kubernetes/pull/89881),
[#97294](https://github.com/kubernetes/kubernetes/issues/97294),
[#111116](https://github.com/kubernetes/kubernetes/issues/111116),
[#123072](https://github.com/kubernetes/kubernetes/issues/123072),
[#123448](https://github.com/kubernetes/kubernetes/issues/123448) (follow-ups [#130939](https://github.com/kubernetes/kubernetes/issues/130939), [#138264](https://github.com/kubernetes/kubernetes/issues/138264)),
[KEP-2340](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2340-Consistent-reads-from-cache).

**Other:** [golang/go#58804](https://github.com/golang/go/issues/58804).

**Jepsen (§2.5):**
[Jepsen: etcd 3.4.3](https://jepsen.io/analyses/etcd-3.4.3) (2020-01-30);
etcd [#11496](https://github.com/etcd-io/etcd/issues/11496) / PR [#11559](https://github.com/etcd-io/etcd/pull/11559) (revision 0, docs, 2020-01-27);
jetcd [#673](https://github.com/etcd-io/jetcd/issues/673), [#674](https://github.com/etcd-io/jetcd/issues/674) (2019 watch-creation exception, stale-closed).

## Appendix E. Revision history and use of the other watch analyses

### E.1 Versions of this file

| Date | Change |
|---|---|
| 2026-10-31 | First version, written independently of the other watch analyses. |
| 2026-10-31 | Second pass: `WithRequireLeader` in depth (then §13, now §6 and §9.1), including the gRPC-proxy bug and the Docker reproduction (now Appendix C). |
| 2026-11-01 | Cross-reference to `leases-claude.md` updated (its §12 → §5.2). |
| 2026-11-01 | Revised using `watchers-codex.md` and `watchers-agy.md` as leads (E.2). In the numbering of that time: new §4.13, §4.14 and Summary items 10–11; extended Summary items 5, 6 and 9, §1, §3.4, the §4 table, §4.2, §4.3, §4.4, §4.6, §4.7, §4.8, §6, §7, §8, §9, §10, §11 (items 11–15), §12 (items 9–14), §13.1–13.3 and §13.7. Corrections to earlier text are listed in E.3. Static only: no tests run, checkouts unchanged. |
| 2026-11-01 | **Restructured for readability.** Added Scope, an executive Summary with a key-findings table, and Conclusions (§15). Reordered sections so that mechanism, contract and premise come first, then failure states grouped by where they arise, then client libraries, Kubernetes, design, recommendations and open questions. Moved data out of the exposition: the fixed-violation ledger, open-item list, Docker reproduction and full reference list are now Appendices A–D; code citations sit in evidence bullets or tables after the claim they support; path prefixes are abbreviated (header). Recommendations and open questions renumbered without gaps. No findings, confidence ratings or references were removed. Map below. |
| 2026-11-02 | Caveat on the delivery disclaimer ("events in unhealthy clusters might never arrive") added to the Summary, §2.2, §12.2 and §15.2. It covers: "unhealthy" is undefined; the disclaimer has effect only with the quorum intact; the contract is therefore safety-only and contradicts etcd's claim of eventual consistency. Points to `etcd-issues-and-mitigations-claude.md` §3. Re-read for it at `v3.7.3`: `etcdctl/ctlv3/command/ep_command.go`, `server/etcdserver/server.go`, `tests/robustness/README.md`. No findings or ratings changed. |
| 2026-11-03 | **Premise verdict revised** after the user pointed out that eventual consistency, which etcd claims for watches, is a liveness promise: from "mostly wrong" to "right about liveness, wrong about safety". The earlier verdict counted only the six safety guarantees of §2.1 as promises. Changed: Terms (the "documented"/"intuitive" split replaced by safety guarantee / eventual delivery / timeliness); the Summary's first, second, third, fifth and last bullets (the 2026-11-02 caveat folded into the third); §2.4; §3 (correct and incorrect parts, a failure condition for the liveness half, a revision note); the §4 "Promise affected" labels; §12.2 (one bullet and the assessment); §15.1, §15.2 and §15.4. No new code reading: the evidence is the existing findings of §5–§10, plus `tests/robustness/README.md:102`, `:111` at `v3.7.3`, re-read. No other findings or confidence ratings changed. |
| 2026-11-03 | **`WithRequireLeader` limits made explicit in the Summary**, after the user read the revised first bullet as implying that the option restores liveness. New Summary bullet listing the paths that stay silent with the option set; first bullet reworded; key findings 2, 3 and 7 extended. **New finding:** a blocked raft loop (for example a WAL `fdatasync` that does not return) freezes the member's leader view, so neither `WithRequireLeader` nor client keepalive ends its watches (§5.2, §6.3 item 1; High for the mechanism, from code, not reproduced). Also: §13.2 item 8 (recreate the client if a re-watch blocks; match `RequestProgress` metadata), §14.1 test 7 (a `raftBeforeSave` sleep), §15.1 and §15.3. Code read at `v3.7.3`: `etcdserver/raft.go:174-260`, `v3rpc/interceptor.go:236-243`, `:315-354`, `server/storage/wal/wal.go:869-881`. GitHub API: #17611 (title and thread re-read) and one issue search for disk stalls. `watchers-agy.md` §3.1.2 (its nine `WithRequireLeader` failure modes) checked against §6.3: each was already covered here, and none includes the blocked raft loop. `watchers-codex.md` §2.6 (2026-11-03) reached the same blocked-raft-loop counterexample independently; I read it only after writing §5.2. |
| 2026-11-09 | **Jepsen pass**, at the user's request. New §2.5: what Jepsen's etcd 3.4.3 watch test showed and what its design could not see (watch rotation under 5 s, one key, one member per client, a check only after healing, no compaction), its header-revision comment explained from code, and the revision-0 finding (#11496) with its 2020 doc fix. Also: Terms ("now" watch), §3, §10.5 (jetcd #673/#674), §13.2 items 5 and 6, §15.1, Appendix D. Read: the report, `jepsen-io/etcd` @ `16149cb` source, PR #11559, the v3.7 docs at `etcd-io/website` @ `41e5cd6ab9ef`, `mvcc/watchable_store.go:294-297`, `:388` and `etcdserver/server.go @ v3.4.3`. No earlier finding or rating changed. Nothing run. |

**Old → new section numbers** (for references made before the restructuring):

| Old | New |
|---|---|
| Header, upstream references | Header (key references); Appendix D.2 |
| Summary (items 1–11) | Summary (executive, with key-findings table); §15 |
| §1 Scope and method | Scope; header "Method"; Appendix D.1 |
| §2.1 | §1 |
| §2.2 | §3 |
| §3.1–§3.3 | §2.1–§2.3 |
| §3.4 | §2.4 |
| §4 (overview table) | §4, split into groups A–E |
| §4.1 | §5.1 |
| §4.2 | §5.2; #22184 detail in §5.3 |
| §4.3 | §5.4 |
| §4.4 | §7.1 (server bugs, design note); §10.1 (Go); §10.3 (jetcd) |
| §4.5 | §7.2 |
| §4.6 | §7.3 |
| §4.7 | §8.1 |
| §4.8 | §9 (introduction), §9.2, §9.3 |
| §4.9 | §8.2 |
| §4.10 | §7.4 |
| §4.11 | §7.5 |
| §4.12 | §7.6 |
| §4.13 | §10.2 |
| §4.14 | §10.4 |
| §5 | §7 (introduction and pattern); Appendix A |
| §6 | Appendix B |
| §7 | §11 |
| §8 | §10.5 |
| §9 | §12.1 |
| §10 | §12.2; §15.4 |
| §11 | §13.1–§13.4 |
| §12 | §14 |
| §13.1 | §6.1 |
| §13.2 | §6.2 |
| §13.3 | §6.3 |
| §13.4 | §9.1 |
| §13.5 | Appendix C (summarized in §9.1) |
| §13.6 | §6.4 |
| §13.7 | §13.5 |
| Appendix A | Appendix E |

**Old → new item numbers.** Recommendations (old §11 → new §13): 1→1, 2→7, 3→12 and 17, 4→8, 5→16, 6→9, 7→2, 7a→6, 7b→14, 8→11, 9→10, 10→3, 11→13, 12→15, 13→5, 14→2, 15→4. Open questions (old §12 → new §14): 1→1, 2→6, 3→7, 4→10, 5→11, 6→12, 7→13, 8→14, 9→3, 10→4, 11→2, 12→9, 13→5, 14→8.

### E.2 Use of the other watch analyses (2026-11-01)

**Sources.** `analysis/watchers-codex.md` (Codex) and `analysis/watchers-agy.md` (Antigravity).
- **Main pass:** the versions committed in `e1bfcc9`, read in full. Both were written independently of this file.
- **Second pass:** both tools rewrote their files during this revision (about 21:10 on 2026-11-01), partly using this file. I skimmed the rewrites for new leads only; rows marked "revised file" come from them. E.3 refers to the committed versions and was not re-checked against the rewrites. Antigravity's rewrite, for example, now makes #22184 conditional on a `store.mu` writer outside the apply scheduler.

Each item was re-checked against the v3.7.3 or jetcd 0.8.7 code, git history, the GitHub API, the Chubby paper or Kubernetes v1.37.2 source before use. Where my conclusion differs from the source's, the difference is stated. Section numbers are the current ones.

| Item | Where | Idea from | Verification and my call |
|---|---|---|---|
| Go client drops the final error after 250 ms; the channel closes bare | §10.2(a), Summary finding 6, §6.1, §6.3 item 10, §13.3 item 13 | both | Code re-traced (`client/v3/watch.go:478-485`, `:558`, `:832-836`). Added: the interface documents a bare close as the caller's own cancellation, so a documented-behaviour consumer can stop watching for good. Kubernetes' defence re-verified in its v1.37.2 source. |
| Go client teardown deadlock (#17611) | §10.2(b), Summary finding 7, Appendix B | Antigravity | Race re-traced at v3.7.3. Added the consequence that later `Watch()` calls on the same metadata block (`client/v3/watch.go:343-386`). Status from the API: closed "not planned", fix PR #17622 closed unmerged. Rated Moderate (narrow race), not High as in the source. |
| #22184: the issue's snapshot trigger cannot run concurrently with apply | §5.3, Summary finding 3 | Codex | Verified both `KV().Commit()` sites run in `applyAll`. Added on my own: the survey of all other `store.mu.Lock()` sites, and that the inversion dates from v3.4.0. **Downgraded** my earlier "Moderate-to-High" to Low–Moderate reachability. |
| #21813 mechanism in code | §9.2, Summary finding 5 | both | Verified. Added on my own: coalescing can *move* live receivers into a dead broadcast; compaction is a concrete terminal trigger; the scope is one key range on one proxy. |
| Proxy filters out non-compaction `Canceled` responses | §9.3 | Codex | Verified (`grpcproxy/watcher.go:106-109`). Linked to the Go client's close responses. |
| A fatal upstream close resets the dead broadcast's revision to 1, so it absorbs nearly every watcher on the range | §9.2 | Antigravity (revised file, seen after my main pass) | Traced in `bcast` and the Go client's close response. Adopted. |
| jetcd#1617 excludes watch resumes | §5.4, §10.5 | Codex (revised file) | PR text read raw. Added on my own: the wait-for-ready default, under which the watch delay is likely silent. This **corrected** my own first wording ("reported through `onError`"). |
| Proxy drops `RequestProgress` (#19710) | §9.3, §11.3, §13.1 item 2 | Antigravity | Verified in code and in the issue (closed "not planned"). The source's quoted "TODO" comment does not exist at v3.7.3. |
| #18879 / #20716 cancellation cycle | §8.1, Appendix B | Codex | Each blocking point re-checked. Kept the source's split: High for the in-process adapter, Moderate for networked clients. |
| #15402 TLS listener starvation | §8.1, §13.1 item 4 | Codex | Added on my own: the mitigation's release versions (`git tag --contains`), and that the default TLS path still serves gRPC through `net/http` at v3.7.3. Rated Low–Moderate for v3.7.3, untested. |
| Unbounded Go client buffer; #123448 | §8.1, §11.3 | both | TODO verified; issue body read raw. Added: the delay → compaction → `prevKV=nil` chain. |
| Kubernetes #76624 | §11.1 | both | Issue body read raw. It explains why Kubernetes now fails on a missing `PrevKv`. |
| Go stream context has no deadline; `openWatchClient` loops; `WaitForReady(true)` | §5.4 | Codex | Verified. |
| `isHaltErr` treats lost leader as retryable; the watch overrides it | §6.1 | Antigravity | Verified. Added on my own: the override dates from v3.1.0 or earlier. |
| #7247 maintainer comment (2017) and the 2018 doc commit `11ead62b9` | §6.2 | Codex | Comment read raw; commit verified locally. |
| Go "now" watch resumes at H, not H+1 | §10.1 | Antigravity | Verified. **Reframed**: a pre-start event, rated Low; not an ordering or uniqueness violation as the source states. |
| jetcd has no fragmentation support | §10.4, §10.5 | Antigravity | Verified. **Extended into a new finding of mine:** with the 4 MiB default inbound limit, large catch-up batches end the watch. The error itself was found in jetcd#1379. |
| jetcd#1616 reconnect backoff | §5.4, §10.5 | Antigravity | Issue read raw; not verified in code. |
| Robustness-suite gap after PR #20240 | Appendix B, §14.2 item 11 | Antigravity | **Narrowed**: the PR removed only the check that progress notifications *arrive*. The Bookmarkable ordering check remains (`tests/robustness/validate/watch.go:113-132`). |
| Revision gaps are not loss; library vs application cursor; a relist does not recover transitions | §2.4, §13.2 item 5 | Codex | Reasoning points; adopted. |
| Chubby §4.5: events were not a publish/subscribe mechanism | §12.1 | Antigravity | Quote checked (Chubby §4.5, `doc/chubby-osdi06.md:516-524`). |
| Test inventory and proposed tests | §14.1 items 2–5 and 8, §14.2 item 9, §6.3 item 5 | both | The `network_partition_test.go` lines and the `a57e967d8` diff were read. |

### E.3 Considered and not adopted, and corrections to this file

**Rejected claims (Antigravity).**
- *"`progressIfSync` blocks progress for every client on the member when any watcher anywhere is unsynced or a victim."* False. The code iterates only the requesting stream's watchers (`mvcc/watchable_store.go:515-528`). The periodic ticker checks one watcher at a time (`mvcc/watcher.go:196-204`). The source's quoted code for `progressIfSync` does not match v3.7.3.
- *"PR #20229 (`973847aa2`) introduced the #22184 lock inversion."* False. `progress()` called `s.rev()` under `s.mu.RLock()` in v3.4.0. The PR only hoisted the call (§5.3).
- *"#22294 disables `RequestProgress` across the entire server and leaks `pending` entries forever."* False, for the reasons in §7.3.
- *"Commit `a57e967d84` deleted `SetEndpoints` because new watches landed on the partitioned member, and pointed the test at a healthy follower."* The comment the source quotes does not exist anywhere in the repository's history (`git log --all -S`). The actual commit says the opposite: it pins the client to the partitioned member so that the failure is exposed (§6.3 item 5).
- *"jetcd's progress handling is off by one and re-delivers the event at R."* Unreachable (§10.3).
- *"After only the first batch arrives, jetcd still loses events."* The event branch resets the cursor (§10.3).
- *"`WithRequireLeader` partition detection is an eighth documented watch guarantee."* It is not in the API guarantees page. It is a statement in the Go client's documentation, treated as such in §6.
- *"Every major failover and partition bug stems from serving watches from unfenced local state."* Overgeneralized. #17529, #18089 and #19179 are compaction bugs (Appendix A).
- Severity labels: "catastrophic" (jetcd) and High (the Go "now" window). I keep Moderate/Low, for the frequency reasons in §10.1 and §10.3.
- *"A pinned READY connection makes `openWatchClient`'s retry loop dead code."* The loop does not retry admission rejections. But the effect is a repeated, visible `ErrNoLeader`, already covered by §6.3 item 5. It is not a separate silent failure.

**Not adopted for lack of evidence.**
- *"#22184 wedges all linearizable reads on the member"* (Antigravity). Plausible if the deadlock occurred, but not traced, and reachability itself is unconfirmed.

**Codex.** Nothing material was rejected. Two items were qualified:
- #18879's networked reachability, kept at Moderate as the source did;
- the jetcd#1352 version inconsistency, taken from the source without re-checking (§10.5).

**Corrections to earlier versions of this file.**
- #22184 (now §5.3, Summary finding 3) was rated "Moderate-to-High" with the snapshot path given as the trigger. Corrected as described above.
- The description of jetcd#1379 (now §10.5), "stops after 10–15 changes with no error", was wrong. The thread's original error was the 4 MiB limit; the silence was a logging problem.
- #21813 (now §9.2) was marked "not verified in code"; it is now verified.
