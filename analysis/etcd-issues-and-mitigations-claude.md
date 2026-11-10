# etcd in production: lease, watch and write-performance risks, and how to deploy around them (Claude)

**Versions analyzed:** etcd `v3.7.3` = `0a173d0410a9`; jetcd `jetcd-0.8.7` =
`23b61f1499a5`; Kubernetes `v1.37.2` (selected files, as in the source
analyses). Tag check on 2026-11-02: no newer stable release of either. Tag
check on 2026-11-09: jetcd `jetcd-0.8.8` was released that day (GitHub and
Maven Central); it is not analyzed here, and the clone stays at 0.8.7 until
the user confirms the move (AGENTS.md §4).

**Method:** a synthesis of the seven Claude analysis files listed in
Appendix A, as committed at `5102a18`. No new code analysis, except that the
flag names, feature-gate names and defaults recommended below were re-checked
at `v3.7.3` (`server/embed/config.go`, `server/etcdmain/config.go:227`,
`server/features/etcd_features.go:89-99`,
`etcdutl/etcdutl/snapshot_command.go:83-84`). The §3 discussion of the
watch delivery disclaimer was added on 2026-11-02, after the user's review.
It re-read the v3.7 API guarantees page and three files at `v3.7.3`:
`etcdctl/ctlv3/command/ep_command.go`, `server/etcdserver/server.go` and
`tests/robustness/README.md`. The **Jepsen pass** (2026-11-09) read the
two Jepsen reports, [etcd 3.4.3](https://jepsen.io/analyses/etcd-3.4.3)
(2020-01-30) and [jetcd 0.8.2](https://jepsen.io/analyses/jetcd-0.8.2)
(2024-08-08), in full; traced every issue they cite, and the other issues
Jepsen filed against etcd and jetcd, through the GitHub API; and checked
the fixes in code at `v3.7.3` and `jetcd-0.8.7`, plus the v3.7 user docs
(`etcd-io/website` `main` @ `41e5cd6ab9ef`) and Jepsen's 2019 test source
(`jepsen-io/etcd` @ `16149cb`). Its results are in §5 ("Independent
testing") and the rows and items marked "Jepsen". A **governance pass**
(2026-11-09) read every comment thread on the issues Jepsen filed, the
SIG etcd charter and README, `GOVERNANCE.md`, `OWNERS`, the stale-bot
workflow history, and the 2022 maintainer messages; its results are in §5
("Project scope and maintenance capacity"). The Jepsen details are in
`leases-claude.md` §2.3, §5.4, §8 and `watchers-claude.md` §2.5. Nothing was run for this file. Each claim keeps the confidence its source gives it; the "See" column
or bracket points to the evidence. **This project has benchmarked nothing**
(AGENTS.md §7): every number below is upstream's, a reporter's, or derived
from code.

**Upstream references (most important):** etcd
[#15247](https://github.com/etcd-io/etcd/issues/15247),
[#12528](https://github.com/etcd-io/etcd/issues/12528),
[#22649](https://github.com/etcd-io/etcd/issues/22649),
[#22081](https://github.com/etcd-io/etcd/issues/22081),
[#22094](https://github.com/etcd-io/etcd/issues/22094),
[#19597](https://github.com/etcd-io/etcd/issues/19597),
[#22082](https://github.com/etcd-io/etcd/issues/22082),
[#19091](https://github.com/etcd-io/etcd/issues/19091),
[#21813](https://github.com/etcd-io/etcd/issues/21813),
[#19710](https://github.com/etcd-io/etcd/issues/19710),
[#17611](https://github.com/etcd-io/etcd/issues/17611),
[#22264](https://github.com/etcd-io/etcd/issues/22264),
[#18879](https://github.com/etcd-io/etcd/issues/18879),
[#22673](https://github.com/etcd-io/etcd/issues/22673),
[#22450](https://github.com/etcd-io/etcd/issues/22450),
[#22223](https://github.com/etcd-io/etcd/pull/22223),
[#22332](https://github.com/etcd-io/etcd/issues/22332),
[#11456](https://github.com/etcd-io/etcd/issues/11456),
[#11457](https://github.com/etcd-io/etcd/issues/11457),
[#11496](https://github.com/etcd-io/etcd/issues/11496),
[#14092](https://github.com/etcd-io/etcd/issues/14092),
[#14890](https://github.com/etcd-io/etcd/issues/14890),
[#18424](https://github.com/etcd-io/etcd/issues/18424),
[#12534](https://github.com/etcd-io/etcd/issues/12534),
[#17611](https://github.com/etcd-io/etcd/issues/17611),
[#13915](https://github.com/etcd-io/etcd/issues/13915),
[#20242](https://github.com/etcd-io/etcd/issues/20242),
[#21051](https://github.com/etcd-io/etcd/issues/21051); CNCF TOC
[#898](https://github.com/cncf/toc/issues/898); SIG etcd
[charter](https://github.com/kubernetes/community/blob/master/sig-etcd/charter.md); jetcd
[#1379](https://github.com/etcd-io/jetcd/issues/1379),
[#1072](https://github.com/etcd-io/jetcd/issues/1072),
[#1444](https://github.com/etcd-io/jetcd/issues/1444) (fix PR
[#1445](https://github.com/etcd-io/jetcd/pull/1445)); Jepsen
[etcd 3.4.3](https://jepsen.io/analyses/etcd-3.4.3) and
[jetcd 0.8.2](https://jepsen.io/analyses/jetcd-0.8.2); Kubernetes
[#65497](https://github.com/kubernetes/kubernetes/issues/65497),
[#110210](https://github.com/kubernetes/kubernetes/issues/110210),
[#59848](https://github.com/kubernetes/kubernetes/issues/59848). Full lists
are in the source files (Appendix A).

**Purpose.** For a prospective production user who reads only this file:
what goes wrong with etcd leases, watches and write throughput at the current
release, what to do about it when deploying, and what to find out before
committing.

## Summary

- **Stored data is safe; timing and liveness are not.** No lease or watch
  defect found corrupts the replicated key-value state. — **High**.
- **Leases.** TTL is a minimum, not a deadline: every leader change resets
  it. Lease-based locks give mutual exclusion only with fencing. Both client
  libraries can keep renewing a lease the application believes is gone.
  — **High** (code); **Moderate** for the residual failover windows, which are
  inferred and unreproduced.
- **Watches.**
  - On direct connections to a current release, the events that arrive are
    in order and complete. — **High** for the fixed bugs; **Moderate** that
    no unknown server defect remains.
  - etcd documents no delivery guarantee. A watch can go **silent with no
    error** while the cluster keeps its quorum and other members could serve
    it, and the default configuration does nothing to detect this.
  - That is a safety-only contract. It contradicts etcd's own description
    of watches as eventually consistent (§3).

  — **High**.
- **The gRPC proxy is unfit for watches and leases that need failure
  signals.** — **High** (one defect reproduced, others from code).
- **Write throughput** is capped per cluster by one Raft log, one apply
  goroutine and disk `fdatasync` latency. Lease revocations and compaction
  force synchronous backend commits, and defrag blocks the member.
  — **High** (mechanism); magnitudes unmeasured.
- **Every mitigation below is deployable without patching etcd.** Most of
  them live in application and client code, and none was tested by this
  project. — **Moderate**.
- **Upstream capacity is limited, and its scope is Kubernetes** (rewritten
  2026-11-09, §5).
  - etcd is governed as a Kubernetes SIG whose charter scope begins "how
    it is used by Kubernetes". In 2022 its maintainers described it as
    "effectively unmaintained".
  - Issues close automatically after about four months without activity.
  - Defects outside Kubernetes' usage have taken years to assess or fix, or
    closed without a fix. Examples: Jepsen's 2022 anomaly reports, closed
    in 2024 as "inaccurate" six days before Jepsen showed the cause; a
    lease failover bug fixed about three years after its first report.

  — **High** for the record; **Moderate** that defects off Kubernetes'
  path fare worse (examples, not a systematic count). Plan to own the
  fixes you need.
- **Independent testing agrees, at older versions** (added 2026-11-09, §5).
  - Jepsen found etcd 3.4.3's key-value operations strict serializable
    under partitions, pauses, crashes, clock skew and membership changes,
    and single-key watches complete and in order.
  - It found that etcd locks do not give mutual exclusion, even in a
    healthy cluster, and lost about 18 % of acknowledged updates protected
    by an unfenced lock. Fencing through a guarded transaction was safe in
    its tests.
  - Its 2024 report traced lost updates, cyclic information flow and
    aborted reads to jetcd retrying writes that may already have executed,
    not to the server. jetcd stopped retrying such writes by default in
    0.8.5 (2025-03-03).

  — **High** for what the reports say and for the fixes being in the code
  analyzed here. Jepsen tested neither v3.7.3 nor jetcd 0.8.7.

## Terms

- **Member / leader / follower:** an etcd server process, and its Raft role.
  The **serving member** is the member that holds a given client stream.
- **Quorum:** a majority of voting members that can communicate. With it,
  the cluster commits writes; without it, nothing commits. This file does
  not use etcd's word "unhealthy", which etcd applies to unrelated
  conditions (§3).
- **Lease:** a server object with a TTL. Keys attached to it are deleted when
  it is revoked: explicitly, or by the leader when the TTL elapses without a
  renewal (**keepalive**).
- **Revision:** etcd's cluster-wide logical clock. Every write increments it.
- **Watch:** a stream of change events from a start revision. **"From now"**
  means from the serving member's current revision.
- **Fencing:** making a protected write conditional, in the same transaction,
  on the lock key's create revision, so that a deposed holder's write fails.
- **`WithRequireLeader`:** an opt-in client flag. The serving member rejects
  or cancels the stream while it has no Raft leader.
- **Apply loop:** the single goroutine that applies committed Raft entries to
  the store. **WAL:** the Raft write-ahead log. **Backend:** the on-disk store
  (bbolt, a copy-on-write B+tree file).
- **Alpha / Beta:** etcd feature-gate maturity. Alpha gates are off by
  default.
- **Indefinite failure:** an error after which the client cannot tell
  whether the server executed the request, for example a timeout or a
  connection lost after sending (Jepsen's term). Retrying a write after one
  can execute it twice.
- **Jepsen:** an independent distributed-systems testing practice; its
  reports on etcd 3.4.3 and jetcd 0.8.2 are summarized in §5.
- **ET:** the Raft election timeout, 1 s by default.

IDs in this file (LS, WT, WR) are local to it. They do not match the L/W
numbering of `chubby-perspectives-claude.md` or the P/R numbering of
`leases-claude.md`.

## 1. Bottom line

1. **Run a current release.** The minimum versions are **v3.7.1, 3.6.14 or
   3.5.33**; this project analyzed v3.7.3. Older releases carry fixed bugs in
   lease failover safety, watch ordering and completeness, and stale
   linearizable reads (note under the §3 table; [watchers §13.1,
   Appendix A]). For jetcd, the floor is **0.8.5**: earlier versions retry
   writes after indefinite failures (§5, Jepsen).
2. **Fence every lock-protected write.** Never trigger an external side
   effect on "my session is alive" alone (LS2, LS4).
3. **Do not use lease TTL for bounded cleanup or failure detection** where
   leader changes are frequent (LS1).
4. **Assume any watch can go silent.** Give every watch consumer its own
   revision cursor, `WithRequireLeader`, client keepalive and a freshness
   check (WT1–WT4).
5. **Connect clients directly to members, not through `etcd grpc-proxy`**
   (WT5).
6. **Pick disks by `fdatasync` latency, and measure before optimizing.** The
   write ceiling is per cluster; the only scale-out is several clusters
   (WR1–WR3).
7. **Plan to test your client library** (Go or jetcd) under partition,
   process pause and leader change. It is the layer with the most open
   defects (§5).
8. **Treat a failed write as "outcome unknown", never as "did not
   happen".** Make writes safe to repeat (guard them with a compare on the
   revision you read), and resolve the outcome by reading back. Do not turn
   on client retries for writes (§5, Jepsen).
9. **Budget to fix what you depend on outside Kubernetes' usage**: etcd
   leases and keep-alive, locks, several watchers per stream, the proxy and
   jetcd. Upstream capacity is limited and issues close on inactivity, so
   keep the issues you need tracked and active (§5, "Project scope and
   maintenance capacity").

## 2. Leases

| ID | Issue | Scenario | Impact | Status at v3.7.3 | Conf. | See |
|---|---|---|---|---|---|---|
| LS1 | **TTL is a minimum, not a deadline** | Every leader change or restart resets each lease to full TTL + ET | Expiry-driven cleanup and failure detection are late; under repeated leader changes, indefinitely (Kubernetes Events, k8s#65497) | By design. The fix, `LeaseCheckpoint`, is Alpha and off, and does not help short leases or frequent leader changes | High | leases §1.3–1.4, §2.2 |
| LS2 | **A leader change can revoke a live lease** | Historically, a stalled leader revoked leases that its successor was renewing (#12528, #15247). At v3.7.3, five narrow windows remain, inferred from code: a stall ending, a large expiry batch, a renewal forwarded to a stalled ex-leader, a renew/revoke race, and a correct revocation committing after a leader change | Without fencing: two holders of one lock or election | Reported cases fixed in 3.4.31, 3.5.13 and 3.6.0, by a heuristic leadership check rather than a fence; none of the residual windows reproduced | Moderate (exists); Low (likelihood) | leases §3.1–3.4 |
| LS3 | **#22649: checkpoint bug** | Only with `LeaseCheckpoint` on: a renewal reports the full TTL while the server keeps a shorter one | Early revocation | Open; fix PR #22650 open | High (code) | leases §3.5 |
| LS4 | **Client lease deadlines are optimistic** | Go and jetcd set the deadline to *response receipt + TTL* and check it once a second; jetcd uses the wall clock | A holder still believes it holds the lock for about 1 s plus the reply delay after the server expired it, and without bound under GC pauses or clock steps | By design | High (code); Moderate (size) | leases §5.1 |
| LS5 | **Lease renewed after the application thinks it is gone** | Go #22081, #22094: only when the application passes a `WithRequireLeader` context to `KeepAlive`. jetcd: any keep-alive stream error calls `onError`, but jetcd keeps renewing | Lock keys never expire; other waiters block; keys leak | Go: open, fix PRs open. jetcd: by design since 0.8.0 | High (code); Moderate (consequence) | leases §5.2–5.3 |
| LS6 | **Renewals through followers fail under leader apply backlog** | A follower forwards each renewal to the leader, which waits up to 1 s for its apply loop; after 7 s of retries the follower fails the stream. `FastLeaseKeepAlive` (Beta, on) helps only renewals sent to the leader itself | If a backlog outlasts the TTL, leases of clients that renew on schedule expire. About (N−1)/N of clients go through followers (inferred) | Structural | High (code); Moderate (impact; backlog frequency unmeasured) | leases §6.1 |
| LS7 | **Expiry is costly, and stalls under overload** | One Raft entry per expired lease, at most 16 in flight and 1000/s. All attached keys are deleted in one transaction, which forces a synchronous backend commit (WR3). Revocations are rejected like any write once the apply loop lags by more than 5000 entries. With JWT auth, one token signature per revocation (#21735) | Expiry waves consume write capacity. Expired keys linger exactly when the cluster is overloaded. A lease with very many keys can stall the cluster (k8s#96836) | Structural; `PriorityRequest` gate is Alpha, off | High (code); magnitude unmeasured | leases §6.2 |
| LS8 | **Lock recipe defects** (`concurrency.Mutex`; the server Lock API that jetcd calls runs the same code) | A waiter keeps waiting after its own session expired (#19597). A retried `Unlock` can delete a newer acquisition by the same session (#22082). Waiters wake through a watch, so they inherit WT1. Historical, found by Jepsen: a waiter whose lease expired while it waited was told it held the lock (#11456). jetcd retries a `Lock` call after `UNAVAILABLE`; if the first attempt's handler is still waiting on its member, its cleanup can delete the lock key that the retry reused (added 2026-11-09; inferred) | Stalled lock waits; one lock released wrongly; a spurious "session expired", or rarely a false grant, after a jetcd retry | #19597, #22082 open. #11456 fixed in v3.5.0, never backported to 3.4. jetcd retry race not reported upstream | High; Low for the jetcd retry race (inferred, not reproduced) | chubby-perspectives §7; leases §2.3, §5.4 |
| LS9 | **`client/v3/leasing` is unsafe** (also exposed as `grpc-proxy --experimental-leasing-prefix`) | A cut-off owner serves stale "linearizable" reads (#19091, closed by the stale bot). Writes by non-owners are applied twice, with wrong `Delete` and `PrevKV` responses (reproduced). `EvictRange` evicts the wrong key | Incorrect reads and duplicated writes | Unfixed, also on `main` | High | chubby-perspectives §8 |
| LS10 | **Unfenced locks fail in practice, not only in theory** (Jepsen, etcd 3.4.3; added 2026-11-09) | Locks with 1–3 s TTLs failed to give mutual exclusion within minutes, in healthy clusters, with no faults injected; pauses and partitions made it faster. With 2 s TTLs, five processes and a pause every 5 s, a lock-protected read-modify-write of an in-memory set lost about 18 % of acknowledged updates. A read-modify-write of a set stored in etcd, whose write was one transaction guarded on the lock key, was safe | Two holders at once; lost updates in whatever the lock protects | Inherent to leases (LS2, LS4). The docs have warned since 2020 (`learning/why.md:82-98`, v3.7 docs), but `etcdctl/README.md:1175-1179 @ v3.7.3` still shows an unfenced `etcdctl lock mylock ./etcdctl put foo bar`, the pattern Jepsen flagged, and the same "why etcd" page still lists "APIs for safe distributed shared locks" (`:46`) | High (Jepsen's measurement on 3.4.3; mechanism unchanged at v3.7.3) | leases §2.3 |

**Kubernetes.**
- **What does not apply.** kube-apiserver never calls `KeepAlive`, and its
  leader election and node heartbeats use `coordination.k8s.io` Lease
  objects, which are ordinary keys. So LS2–LS5 do not apply to it.
  — **High** (code); **Moderate** for "never", since the whole repository was
  not searched.
- **What does.**
  - LS1: Event TTLs reset on every etcd leader change, and kubeadm does not
    enable checkpointing.
  - LS7: k8s#96836 led Kubernetes to cap leases at 1000 objects.
  - k8s#110210 (open): a watch broken in the middle of a shared lease's
    revocation can miss the remaining DELETE events.

  [leases §7]

**Mitigations.** None tested; derived from code. — **Moderate**.
1. **Fence.** Put `Mutex.IsOwner()`, or
   `Compare(CreateRevision(lockKey), "=", rev)`, in the same `Txn` as every
   protected write. For resources outside etcd, pass the revision as a
   fencing token and have the resource reject older ones. Do not gate
   external actions on `Session.Done()` alone (LS2, LS4, LS10). Jepsen's
   safe variant guarded each write with `Version(myLockKey) > 0`, which is
   equivalent once the lock has been granted, because the lock key is
   unique to its lease. `etcdctl lock` passes the token to its command as
   `ETCD_LOCK_REV`.
2. **Do not treat TTL as a deadline.** Where cleanup must be bounded,
   delete explicitly and use the lease only as a backstop (LS1).
3. **Leave `LeaseCheckpoint` off** until PR #22650 ships, unless keeping long
   leases accurate across leader changes matters more than LS3.
4. **Go:** do not pass `WithRequireLeader` contexts to `KeepAlive` until
   #22081 and #22094 are fixed. `concurrency.Session` is unaffected by
   default.
5. **jetcd:** treat any `onError` other than `NOT_FOUND` as transient. To
   give up a lease, `close()` the keep-alive *and* revoke the lease.
6. **Use a margin of your own:** consider a lease lost at *send time of the
   last acknowledged renewal + TTL − margin*, not when the library says (LS4;
   chubby-perspectives §10, B1).
7. **Locks:** give `Lock()` a deadline and abandon it when the session ends
   (LS8). Release with an owner-compare delete. jetcd: treat a lock
   returned after a retried call as unproven until a fenced write succeeds
   (LS8; leases §5.4).
8. **Avoid grant/revoke churn:** reuse leases, but cap the keys per lease
   (LS7, WR3). With JWT auth, budget CPU for expiry waves.
9. **Alert on apply lag.** Expect clients connected to followers to lose
   leases first (LS6).
10. **Do not use `client/v3/leasing`** or the proxy's leasing prefix (LS9).

## 3. Watches

**What holds.**
- Events come from Raft-committed entries, at revisions identical on every
  member.
- A client that resumes from its own last revision + 1 loses nothing across
  a leader change, on any member, when connected directly to a release at or
  above the floor in §1.
- All known ordering and completeness bugs are fixed (snapshot restore,
  compaction, resume on a lagging member).

— **High** [watchers §3, §7, Appendix A].

**Jepsen agrees on safety, and could not have seen the delivery problem**
(added 2026-11-09). Its etcd 3.4.3 tests found that watches "deliver every
update to a key in order" under partitions, pauses, crashes, clock skew and
membership changes. But each test watch lasted under 5 s before being
re-created from its last revision, on a single key, with no compaction, and
the check compared logs only after faults had healed. A watch that went
silent was closed within seconds anyway and resumed from the same revision,
so silence left no trace in what was checked. Several ordering and completeness bugs fixed later (§1 floor)
were present in 3.4 releases, per upstream; whether each was in 3.4.3 was
not checked. Their triggers, compaction and snapshot restore on a lagging
member, were outside or at the edge of that workload. — **High** for the
test design (Jepsen's test source read) [watchers §2.5].

**What does not hold: delivery.** The table below lists the cases. What
etcd documents about them needs care.

**The delivery disclaimer, disambiguated.**
- **The text.** The v3.7 API guarantees page lists six properties of *what*
  a watch delivers: ordered, unique, reliable, atomic, resumable,
  bookmarkable. It has one sentence on *whether* it delivers: "there is no
  limit and events in unhealthy clusters might never arrive"
  (`content/en/docs/v3.7/learning/api_guarantees.md:37`, `etcd-io/website`
  `main`, re-read 2026-11-02).
- **"Unhealthy" is undefined there, and etcd uses the word for unrelated
  conditions.**
  - `etcdctl endpoint health` exits with "unhealthy cluster" when any one
    checked endpoint fails, whether or not a quorum remains
    (`etcdctl/ctlv3/command/ep_command.go:166-178 @ v3.7.3`).
  - The server returns "etcdserver: unhealthy cluster" when a leadership
    transfer has no connected target, or when a member add would break the
    active quorum (`server/etcdserver/server.go:1301-1303`, `:1427-1434`).

  This file does not use the term. It distinguishes two cases:
  - **No quorum.** No write commits, so there is nothing new to deliver.
    Events committed just before the loss can also wait for quorum to
    return, on members not yet told they were committed. No API can promise
    more, and the disclaimer adds nothing here.
  - **Quorum, with a failed component.** The cluster keeps committing writes.
    What failed is the serving member, the connection, the proxy or the
    client library (WT1–WT3, WT5, WT6), and other members could serve the
    watch. Only in this case does the disclaimer have any effect. etcd's
    defaults, and the way these defects have been handled upstream, treat
    it as covered.
- **So the documented contract is safety-only.** — **High** for the reading
  of the text and code; the judgement of its value is mine.
  - It constrains what arrives, never whether anything arrives. A watch that
    delivers nothing, forever, in a cluster with quorum satisfies it.
  - As a liveness contract it is therefore empty, in exactly the case where
    clients need one.
  - The safety half still has value. It makes resuming from a stored
    revision lossless, which is what lets a client build the missing
    liveness check itself, as Kubernetes does (KEP-2340).
  - But every consumer must build that check, and etcd's defaults do not:
    `WithRequireLeader` is off, Go keepalive is off, and progress
    notifications come every 10 min.
- **It is not eventual consistency.**
  - etcd's robustness suite states: "Etcd provides strict serializability
    for KV operations and eventual consistency for Watch"
    (`tests/robustness/README.md:111 @ v3.7.3`).
  - Eventual consistency is a liveness guarantee. An update the system has
    accepted eventually reaches every correct replica that can communicate
    with it (Vogels, "Eventually Consistent", CACM 2009; "eventual
    delivery" in Shapiro et al., "Conflict-free Replicated Data Types", SSS
    2011; both recalled, not re-read). "Might never arrive" is its negation,
    not a description of it.
  - WT1 meets it only in the weakest reading: events arrive once the
    partition heals, although the client could have resumed on a working
    member all along.
  - Other paths stay silent after every network fault has healed, so they
    violate even that reading:
    - a proxy watch attached to a dead broadcast (#21813);
    - a Go client deadlocked in teardown (#17611);
    - a connection that died unnoticed, with no client keepalive (WT3);
    - a member whose apply loop wedges, if #22184 has a trigger (WT2).
- **What a useful contract would say.** — **Moderate** (design inference).
  - The promise: for a client that can reach at least one member connected
    to the quorum, a watch either keeps delivering committed events, or ends
    with an error within a bounded time. The client can then resume
    elsewhere from its last revision.
  - That is implementable:
    - Chubby's session lease bounds it (Chubby §2.8).
    - Kubernetes approximates it per request, with linearizable reads and
      progress requests.
    - Watch mitigations 7–9 below build it client by client.
    - The server-side form would cancel streams when apply stops
      progressing, rather than when the leader is lost [watchers §13.5].

| ID | Issue | Scenario | Impact | Status at v3.7.3 | Conf. | See |
|---|---|---|---|---|---|---|
| WT1 | **Silent watch on a cut-off member** | The serving member, follower or deposed leader, is isolated from the quorum | No events and no error until the partition heals | Permitted by the documented contract (§3); `WithRequireLeader` is off by default in Go and jetcd | High | watchers §5.1 |
| WT2 | **`WithRequireLeader` is a partial fix** | It cancels after about 3–5 s. It misses a member that has a leader but does not apply (one candidate cause: the #22184 lock inversion; no trigger confirmed). It also misses a member whose raft loop is blocked, for example on a WAL `fdatasync` that does not return: that member's leader view freezes, and it keeps answering keepalive pings (added 2026-11-03). It ends the watch instead of resuming it, and Go discards the resume point, so a re-watch "from now" skips the writes made meanwhile. Watches that set it can also be silenced by WT5, WT6 and the #18879 stream freeze (WT8) | Indefinite silence on such a member despite the option; silent gaps for clients that re-watch from "now" | By design | High (code); Moderate (the apply-stall case); blocked raft loop not reproduced | watchers Summary, §5.2–5.3, §6 |
| WT3 | **Paused member or dead connection** | The Go client has no gRPC keepalive by default and retries forever; it raises no error when no member is reachable | Indefinite hang | Default behaviour | High | watchers §5.4 |
| WT4 | **Freshness signals are local** | The "now" start point, the `Created` revision and progress notifications reflect only the serving member. Periodic progress notifications are off unless requested, and the server interval is 10 min. `RequestProgress` goes unanswered if any watcher on the stream is behind (#20242) | "No changes" looks the same as "stalled" | By design | High | watchers §1, §2.3, §8.2 |
| WT5 | **The gRPC proxy breaks failure signalling** | It ignores `WithRequireLeader` (reproduced; not reported upstream). It attaches watchers to dead upstream watches (#21813). It swallows upstream errors. It drops `RequestProgress` (#19710, closed not planned) | Permanent silent loss of updates; no freshness check possible | Open | High (code); Moderate (how often #21813 occurs) | watchers §9 |
| WT6 | **The Go client can lose failure signals** | If the application does not read within 250 ms, the final error is dropped and the channel closes bare, which looks like the caller's own cancellation. A narrow race deadlocks teardown, and later `Watch()` calls block (#17611, closed without a fix) | A missed failure; a hang | Open | High (code); Moderate (the race) | watchers §10.2 |
| WT7 | **jetcd watch defects** | Resume skips historical events if the stream breaks after `Created` and before the first batch (not reproduced). A response over 4 MiB ends the watch, and re-watching fails at the same revision (jetcd#1379). jetcd can stop retrying after one failed resume | Silent gaps; a watch that cannot progress | Open | Moderate–High per item | watchers §10.3–10.5 |
| WT8 | **Slow and far-behind watchers** | Multi-second delivery lag (#16839). A watcher far behind blocks writes: the reporter of #22264 saw writes blocked more than 90 % of the time on 3.5.21, and says it is still present in 3.7.1. Bursts of cancellations freeze a shared stream (#18879; fix PR #21065 open). The Go client buffers without limit. A long delay lets compaction remove `PrevKv` | Delay; write stalls; errors in consumers that need the previous value | Open | Moderate | watchers §8.1 |
| WT9 | **Raw-gRPC watch-ID reuse** | Reusing an explicit `watch_id` delivers a cancelled watch's events to the new watch (#22673) | Wrong events; the Go client is unaffected | Open | Moderate | watchers §8.2 |
| WT10 | **Restore without a revision bump** | An operator restores a snapshot, so revisions go backwards for watchers | Silent inconsistency in watch-fed caches | Operator choice | High | watchers §7.4 |

Note on versions: 3.5 < 3.5.22 and 3.6 < 3.6.2 still have watch ordering
bugs; the watch-auth fix is in 3.7.1, 3.6.14 and 3.5.33 [watchers
Appendix A].

**Kubernetes.**
- **Well defended on the direct path:** `WithRequireLeader`, gRPC keepalive
  (30 s + 10 s), explicit start revisions, every error ends the watch, and a
  missing `PrevKv` is an error. Consistent reads from cache pair a
  linearizable `Get` with `RequestProgress`, capped at 3 s.
- **Residual exposure:**
  - WT2 apply stalls;
  - WT3 paused members (about 40 s until keepalive notices);
  - WT5, through the proxy;
  - `resourceVersion=0` reads, which may be arbitrarily stale (k8s#59848,
    open);
  - mass watch teardown when an etcd election is prolonged (k8s#111116).

— **Moderate** [watchers §11].

**Mitigations.** None tested; from code. — **Moderate**.

Operators:
1. Run at or above the floor in §1.
2. Point watch and lease clients at members, not `etcd grpc-proxy` (WT5).
3. Restore with
   `etcdutl snapshot restore --bump-revision <large> --mark-compacted`
   (WT10).
4. On TLS listeners with heavy range reads, consider
   `--listen-client-http-urls` to separate gRPC from HTTP (#15402; untested
   on v3.7.3, Low–Moderate).

Clients:

5. **Keep your own cursor.** Persist the last *applied* revision atomically
   with the state derived from events, and resume from it + 1. The library's
   cursor records receipt, not application. Take it from the events'
   `ModRevision`, not from response headers: Jepsen's harness saw
   non-monotonic revisions when it used header revisions [watchers §2.5].
6. **Never re-watch from "now" after an error.** List, then watch from the
   list revision + 1; or record the `Created` revision. Revision 0 means
   "now", not "from the beginning": the first revision is 1 (Jepsen,
   #11496; documented since 2020).
7. **Set `WithRequireLeader`** (jetcd: `withRequireLeader(true)`), and on
   `ErrNoLeader` re-watch from your cursor.
8. **Go:** set `DialKeepAliveTime` and `DialKeepAliveTimeout` (WT3). Treat
   every channel close you did not cause as a failure (WT6). Send
   `RequestProgress` with the same context metadata as the watch.
9. **Freshness monitor.** Periodically read a revision R with a linearizable
   `Get`, then request progress, and require the watch to reach R within a
   deadline. Otherwise cancel and re-watch on another member; if the
   re-watch itself blocks (Go, #17611), create a new client. Nothing else
   detects WT2 or WT8.
10. **Handle compaction and missing values.** A compaction error means
    relist; a missing `PrevKv` is an error if you need the old value.
11. **Isolate watchers.** Keep unrelated watchers on separate streams, and do
    not leave watches far behind on a busy cluster (WT8).
12. **jetcd:** raise `maxInboundMessageSize` well above 4 MiB. Track the last
    `ModRevision` in the application, and on `onError` close the watcher and
    re-create it from an explicit revision (WT7).
13. **Raw gRPC:** never reuse explicit `watch_id`s (WT9).

## 4. Write performance

**The write path.** Client → leader → Raft replication, with a WAL
`fdatasync` on a quorum of members → the single apply loop → backend commit
→ reply. Nothing here was measured.

| ID | Issue | Mechanism | Impact | Conf. | See |
|---|---|---|---|---|---|
| WR1 | **One log, one apply loop per cluster** | Every write goes through one Raft log and one apply goroutine. Writes are rejected (`ErrTooManyRequests`) when apply lags commit by more than 5000 entries | A fixed ceiling per cluster; the only scale-out is splitting data across clusters | High | storage §2.1, §1 |
| WR2 | **`fdatasync` latency sets the pace** | One WAL `fdatasync` per Raft batch on every member. The leader counts its own copy of an entry toward the quorum only once it is durable. The Raft goroutine waits during the sync (asynchronous storage writes are not enabled in v3.7.3). Device `fdatasync` latency spans microseconds (enterprise NVMe with power-loss protection) to milliseconds (consumer NVMe, cloud network block storage) | Disk choice matters more than CPU | High (mechanism); Moderate (device figures, desk research) | wal-replacement §1.2–1.3, §11.8 |
| WR3 | **Backend commits block applies and new reads** | Each bbolt commit runs 2 `fdatasync`s while holding locks that block the next apply and the start of new reads. It runs every 100 ms, plus synchronously for every entry that deletes from the backend (**every lease revocation**) and every 1000-key compaction batch | Lease churn and compaction cause commit storms. At about 1 ms per `fdatasync`, revocations cap near 500/s and stall other writes meanwhile | High (mechanism); Low–Moderate (magnitude) | storage §2.3, §3.1 |
| WR4 | **Defrag stops the member** | In 3.7.x, defrag locks the backend for the whole copy; non-blocking defrag exists only on `main`, for v3.8 | The member is unavailable for a time that grows with DB size | High | storage §3.1 |
| WR5 | **Full scans that stall** | MVCC compaction rescans from revision 0 every time. HashKV corruption checks (`--corrupt-check-time`, default 0 = off) hold a lock for a full scan, stalling applies | Periodic latency spikes that grow with DB size | High (code) | storage §2.2–2.3 |
| WR6 | **WAL file mechanics** | Segments are preallocated with `fallocate`. On ext4 and XFS the first write into that space forces a filesystem-journal commit at `fdatasync` (one external measurement: 20k vs 99k IOPS). Every 64 MB, segment rotation runs 2 `fdatasync`s and a rename on the Raft goroutine | Periodic tail-latency spikes; extra sync cost | High (mechanism); Low (gain for etcd, unmeasured) | wal-replacement §1.1, §11.3 |
| WR7 | **Watches and leases load the write path** | WT8 (a far-behind watcher blocks writes, #22264); LS7 expiry waves; #22450, a leader apply backlog that fails to recover after a transient stall (open; reported on 3.5.21 and 3.6.6, not checked on 3.7) | Write stalls caused by read-side or lease activity | Moderate | watchers §8.1; leases §6 |
| WR8 | **All keys live in RAM** | The key index (every key and revision) is in memory whatever the backend; the DB size is bounded by the quota | Memory, not disk, caps the key count | High | storage §2.2 |

**Reference numbers.**
- The only official write numbers are for **etcd 3.2.0**: 44–50k QPS with
  256-byte values, 1000 clients, on 8-vCPU GCE machines. A source comment
  assumes a maximum of about 10k [wal-replacement §11.7].
- Upstream's own health thresholds: WAL fsync p99 < 10 ms, backend commit
  p99 < 25 ms (etcd FAQ) [wal-replacement §11.7]. These are alarm
  thresholds, not performance targets.

**Not shipped yet.**
- **Asynchronous Raft storage writes**, PR #22223 (open, behind an Alpha
  gate). The Raft library's own PR reported 20–25 % lower latency and 73 %
  higher maximum throughput on AWS gp3. Two safety reports against the mode
  are open (raft #551, #543).
- **Pebble backend prototypes** (#22332). At the MVCC layer: about 13 %
  faster steady writes, steady throughput during compaction and defrag, and
  5–6.5× slower reads.
- Forecast that upstream ships an alternative engine within about 12
  months: **Low** [storage Summary 9; wal-replacement Summary 1, 7].

**Mitigations.** — **Moderate**: standard practice plus code-derived
reasoning; none benchmarked here.
1. **Choose storage by `fdatasync` latency.** Measure it with `fio` on a
   WAL-like pattern. Prefer local enterprise NVMe with power-loss protection,
   and put the WAL on its own device (`--wal-dir`).
2. **Measure before tuning or forking.**
   - f_commit = `rate(etcd_disk_backend_commit_duration_seconds_sum[5m])`,
     the fraction of time spent in backend commits.
   - Commits per second well above 10 mean forced commits from lease
     revocations or compaction.
   - Also watch the WAL fsync histogram, the apply lag (committed − applied)
     and `etcd_server_lease_expired_total`.
   - Any backend improvement can raise throughput by at most 1/(1 − f_commit).
3. **Partition data that needs no cross-key atomicity** into separate
   clusters. Kubernetes can move Events to their own cluster (kube-apiserver
   `--etcd-servers-overrides`; recalled). AWS EKS reports up to 5× write
   throughput from partitioning alone [storage §4.3, §8].
4. **Shape the workload:**
   - batch related operations into one `Txn`;
   - skip writes of unchanged values;
   - keep values small, compressing them client-side;
   - reuse leases instead of a grant and revoke per object.
5. **Compaction and defrag:**
   - tune auto-compaction retention, `--compaction-batch-limit` and
     `--compaction-sleep-interval`;
   - defragment one member at a time, off peak, with clients steered away;
   - set `--quota-backend-bytes` with headroom.
6. **Decide `--corrupt-check-time` explicitly.** Periodic checks detect
   corruption but stall applies (WR5).
7. **Change `--backend-batch-interval` and `--backend-batch-limit` only
   after measuring.**
8. **Benchmark on the target hardware with open-loop load**
   (`tools/benchmark --rate`), including lease churn, compaction and
   watchers, not only puts [storage §10].

## 5. Cross-cutting risks

- **gRPC proxy.**
  - WT5.
  - The same `WithRequireLeader` defect applies to lease keep-alive streams.
  - It serves LS9's leasing cache.
  - The defect has been present since v3.3.0 and is not covered by e2e tests.

  [watchers §9; leases §5.2.4]
- **Client libraries are the weakest layer.**
  - Go: LS5, WT3, WT6.
  - jetcd 0.8.7, which is pre-1.0: LS4 (wall clock), LS5 and WT7. Its tests
    run against an etcd image with known watch bugs.

  — **High** (code) [leases §11.2; watchers §15.3].
- **Automatic retries of writes** (Jepsen, jetcd 0.8.2; added 2026-11-09).
  - **What Jepsen found.** jetcd 0.6.0 to 0.8.3 (there is no 0.8.4)
    retried any call that failed with `UNAVAILABLE`, including a `Txn`
    already sent and possibly executed. With only `kill -9` of servers, one
    `commit()` executed twice, about 3 s apart. Applications saw lost
    update, cyclic information flow (G1c), transactions that read their
    own future writes, and aborted reads: a guarded transaction reported
    as failed whose first copy had committed. Guards do not prevent this,
    because the caller is told the retry's outcome, not the first
    attempt's.
  - **Fixed in jetcd 0.8.5** (2025-03-03; PR #1445 for #1444, by this
    project's user). At 0.8.7, `Put`, `Delete` and `Txn` retry only on auth
    errors unless the caller opts in with `withAutoRetry()`; election calls
    never retry on `UNAVAILABLE`.
  - **Still retried on `UNAVAILABLE` in jetcd 0.8.7:** `Lock`, `Unlock`,
    lease `grant` and `revoke`, and reads.
  - **The Go client at v3.7.3** retries a write only on an error showing
    that no connection was available, or on a deadline or cancellation
    error while the caller's own context is live. The maintainers concluded
    that neither follows a request the server executed, and could not
    reproduce the problem with the Go client (#18424). It does retry lease
    grant and revoke on any `Unavailable`.
  - **Consequences of the remaining retries:** an orphan lease that lives
    until its TTL runs out; a "lease not found" error for a revoke that
    succeeded; and, in jetcd, the LS8 lock race. — **High** (code);
    consequences inferred, not reproduced.
  - **The server cannot deduplicate.** etcd has no request IDs, so an
    indefinite failure must be resolved by the application. #18424, which
    proposed client-side rules, was closed by the stale bot in 2025.

  [leases §5.4]
- **Project scope and maintenance capacity** (rewritten 2026-11-09; it
  replaces the earlier "Maintenance signals" bullet, which wrongly called
  #13915's closure a decision). The adopter's question: how likely is a
  defect you report, outside Kubernetes' usage, to be assessed and fixed in
  time? The facts below come from etcd's governance documents and public
  issue threads, read 2026-11-09. People are named by role; each quote
  links to its source. The assessment at the end is inference.
  - **Governance.** Since 2023-09 etcd has been governed as Kubernetes SIG
    etcd ([announcement](https://etcd.io/blog/2023/introducing-sig-etcd/)).
    - The [charter](https://github.com/kubernetes/community/blob/master/sig-etcd/charter.md)'s
      scope begins: "Owns the etcd project and how it is used by
      Kubernetes". It lists "the implicit Kubernetes-ETCD Contract" among
      its responsibilities. The SIG's README adds that etcd should also
      serve "as a standalone solution".
    - The SIG's chairs and tech leads are etcd's approvers
      (`OWNERS`, `OWNERS_ALIASES @ v3.7.3`). The project uses Kubernetes'
      merge bot and Slack, and has a Kubernetes steering-committee liaison.
    - jetcd is a SIG etcd subproject on paper. Its own `OWNERS` names none
      of the SIG leads, and one volunteer made 29 of the roughly 50 non-bot
      commits since 2024-08. In #14890 a maintainer wrote: "I doubt that the
      jetcd is well maintained by existing etcd maintainer either"
      ([2023-01-27](https://github.com/etcd-io/etcd/issues/14890#issuecomment-1405879641)).
  - **Capacity.** On 2022-03-07 a maintainer told the Kubernetes steering
    committee that etcd had "only one active and two occasionally-reviewing
    maintainers" and was "effectively unmaintained"
    ([thread](https://groups.google.com/a/kubernetes.io/g/steering/c/e-O-tVSCJOk);
    read through a summarizing fetch). That led to
    [CNCF TOC #898](https://github.com/cncf/toc/issues/898) and, per the
    announcement, to the SIG. Jepsen's 2022 reports arrived three months
    after that message.
  - **Process: issues close on inactivity.**
    - A bot marks an issue stale after 90 days without activity and closes
      it 21 days later, unless it is labelled `stage/tracked` or
      `help wanted` (`.github/workflows/stale.yaml`, `origin/main`).
    - From 2025-05-13 to 2026-01-13 the workflow used input names the
      GitHub Action does not recognize, so those exemptions did not apply
      (fix: commit `55c5fabe5`). About 50 `stage/tracked` issues were closed
      in that window.
    - Maintainers defend auto-closure as policy: "Reason to auto-close is
      that the author or contributors stopped caring about the issue enough
      to address it" ([#21051](https://github.com/etcd-io/etcd/issues/21051#issuecomment-3691314377)); "If an
      issue is important, the author should take responsibility for keeping
      it moving forward" ([#21051](https://github.com/etcd-io/etcd/issues/21051#issuecomment-3738979541)). That
      proposal to stop auto-closing was itself closed by the bot on
      2026-05-08.
    - The bot is part of the project's process, so its closures count as
      the project's. For an adopter, an issue it closes, by policy or by
      misconfiguration, is as unresolved as one a maintainer declines.
  - **Cases.** Each row is a public record; "the bot" is the stale bot.

    | Case | Timeline | Outcome |
    |---|---|---|
    | **Jepsen's anomaly reports** ([#14092](https://github.com/etcd-io/etcd/issues/14092), jetcd [#1072](https://github.com/etcd-io/jetcd/issues/1072), [#14890](https://github.com/etcd-io/etcd/issues/14890), [#18424](https://github.com/etcd-io/etcd/issues/18424)) | **2022-06-06:** aborted reads reported against etcd 3.5.3. Next day a maintainer: "From brief read this looks like bug in jetcd and not etcd" ([comment](https://github.com/etcd-io/etcd/issues/14092#issuecomment-1148426533)). Closed 2022-09-28 "as a jetcd issue". The jetcd report got no maintainer reply and was bot-closed 2022-08-22. **2022-12-04:** lost update and cyclic information flow reported (#14890). Maintainers tried to reproduce it in etcd's own framework, without success. One concluded: "It turned out to be **not an issue** to me" ([2022-12-07](https://github.com/etcd-io/etcd/issues/14890#issuecomment-1340300176)). **2023-01-26:** Jepsen withdrew after "months of unpaid work" ([comment](https://github.com/etcd-io/etcd/issues/14890#issuecomment-1405084453)). **2023-01-27:** a Kubernetes engineer: "I think the most likely explanation is that Jepsen attempted a write, it believed the write failed and therefore it was free to re-use the value, but etcd actually recorded the write" ([comment](https://github.com/etcd-io/etcd/issues/14890#issuecomment-1406838149)). That is the actual cause. **Then 18 months without activity.** **2024-07-22:** an outside developer traced the anomalies to the client's automatic retry ([comment](https://github.com/etcd-io/etcd/issues/14890#issuecomment-2242269686)). **2024-08-02:** closed. A maintainer: "at the current moment I think we can official reject Jepsen report as inaccurate" ([comment](https://github.com/etcd-io/etcd/issues/14890#issuecomment-2265016141)). A second: "Agreed. … The problem might be related to etcd java client or Jepsen. Based on all the discussion so far, I would bet Jepsen" ([comment](https://github.com/etcd-io/etcd/issues/14890#issuecomment-2265039691)). **2024-08-08:** Jepsen's report: "It's jetcd" ([comment](https://github.com/etcd-io/etcd/issues/14890#issuecomment-2276009336)). The same day a maintainer opened #18424; the Go client was found unaffected; the bot closed it 2025-06-01 | The report was correct. jetcd was fixed by an outside contributor (PR #1445, merged 2025-02-18; released in 0.8.5 on 2025-03-03): **2 years 9 months** after the first report, with the right diagnosis on the thread **18 months** before it was confirmed |
    | **Lease revoked by a deposed leader** ([#12528](https://github.com/etcd-io/etcd/issues/12528) → [#15247](https://github.com/etcd-io/etcd/issues/15247); LS2) | **2020-12-07:** reported on v3.4.6. A fix, PR [#12531](https://github.com/etcd-io/etcd/pull/12531), was opened the next day. The bot closed the issue 2021-06-14 and the PR 2021-09-16, unmerged. **2023-02-06:** the same bug re-reported as #15247, citing #12528. Fix merged 2023-12-15 | Released in v3.4.31 and v3.5.13 (March 2024) and v3.6.0 (May 2025): about **3 years 3 months** from report to fixed release, with a candidate fix from the second day. The fix chosen is a heuristic leadership check; the term fence proposed in #15247 was not adopted |
    | **Watch blocked forever despite `WithRequireLeader`** ([#12534](https://github.com/etcd-io/etcd/issues/12534) → [#17611](https://github.com/etcd-io/etcd/issues/17611); WT2, WT6) | **2020-12-09:** reported. A then-listed maintainer: "It is a nice to have feature. Probably etcd should check if the server maintains leadership for long running requests" ([2021-04-19](https://github.com/etcd-io/etcd/issues/12534#issuecomment-822107665)). Bot-closed 2021-08-08. **2024-03-19:** re-filed under the same title, now with a client-side cause; labelled `type/bug` and `priority/important-longterm`; bot-closed *not planned* 2025-06-01 | Never fixed. The 2021 classification answered a misreading: the server already cancelled flagged streams on leader loss (watchers §6.1), and #17611 later located the hang in the Go client's teardown, a race (WT6). Either way, a report of a watch silent forever aged out twice over about four and a half years |
    | **"Make leases linearizable"** ([#13915](https://github.com/etcd-io/etcd/issues/13915)) | Opened 2022-04-09 and labelled `stage/tracked`. Bot-closed 2025-09-11 while the exemption was broken; reopened by a maintainer 2025-09-23; bot-closed again 2025-12-19. The checkpointing stabilization tracker ([#13776](https://github.com/etcd-io/etcd/issues/13776); `stage/tracked`, `help wanted`) was bot-closed 2025-09-30 | Neither closure was a verdict on the design, but for an adopter the outcome is the same: tracked lease redesigns closed and not reopened. Both questions remain open; checkpointing has been experimental or Alpha since v3.4.0 (2019) |
    | **Lost un-fsynced writes** ([#14140](https://github.com/etcd-io/etcd/issues/14140), [#14143](https://github.com/etcd-io/etcd/issues/14143)) | 2022: labelled `priority/important-soon` and `stage/tracked`. A maintainer later reported that a contributor had reproduced #14143 ([2023-07-04](https://github.com/etcd-io/etcd/issues/14143#issuecomment-1620148375)). Little activity after 2023, apart from a 2024 maintainer comment proposing to close #14140; both bot-closed in July 2025 while the exemption was broken | No root cause (crash-consistency bullet below) |
    | **Watch progress guarantee** ([#20242](https://github.com/etcd-io/etcd/issues/20242); WT4) | **2025-06-30:** filed by a maintainer: "Note this doesn't affect Kubernetes as it setups only 1 watcher per stream." The same day, the robustness suite's progress-notification check was disabled "until we can guarantee response" (PR #20240). Bot-closed 2025-11-23 with no human comment | At v3.7.3 the guarantee does not exist, and the robustness suite's check that progress notifications arrive (`ValidateGotAtLeastOneProgressNotify`, removed by #20240) is absent |
    | **`client/v3/leasing` stale read** ([#19091](https://github.com/etcd-io/etcd/issues/19091); LS9) | Reported 2024-12-19. The report and its first fix were bot-closed (2025-05-17) without maintainer review; a second fix was closed unmerged | Unfixed, also on `main` |

  - **Also:** the robustness harness does not exercise lease expiry or
    keepalive, and adding leases to its Kubernetes traffic profile (#20621)
    was bot-closed.
  - **Assessment.**
    - **Capacity is the binding constraint**, by the maintainers' own
      account, and the backlog is managed by auto-closure. — **High**.
    - **Defects on Kubernetes' path get fixed; defects off it often do
      not.** Server panics and the watch ordering bugs that matter to the
      apiserver's watch cache were fixed and backported. Lease keep-alive
      and expiry, locks, several watchers per stream, the gRPC proxy,
      `client/v3/leasing` and jetcd more often stay open or close without a
      fix. — **Moderate**: drawn from the cases above, not from a
      systematic count. It would fail if a classification of all
      bot-closed bugs showed the same rate on and off Kubernetes' path.
    - **Assessment can take years.** "Not caused by the etcd server" has
      been treated as grounds to close even when the cause was a client
      hosted in etcd-io, and one closure called a correct report
      inaccurate. — **High** for the record; **Moderate** that this
      generalizes.
    - **Not evidence of indifference by individuals.** Most replies came
      within days and were technically engaged. Several maintainers tried
      to reproduce the Jepsen results, and the tooling (Clojure, Jepsen's
      Elle checker) was a real barrier. The reporter withdrew in January
      2023. — **Moderate**.
  - **What an adopter can do.** — **Moderate** (derived from the record;
    untested as policy).
    1. **Plan to own fixes off Kubernetes' path:** budget time to diagnose,
       write the fix and its test, and carry it as a patch until it ships.
       jetcd's retry fix came this way.
    2. **Keep the upstream issues you depend on alive.** Ask for
       `stage/tracked` (honored again since 2026-01-13), comment before the
       90-day mark, and if one is closed, ask a maintainer to reopen it or
       file a new issue that links it.
    3. **Report with a reproduction in etcd's own frameworks** (e2e or
       robustness tests), not only external tooling.
    4. **Treat "not an etcd-server bug" as unresolved** while the client
       library involved is one of etcd-io's.
    5. **Test the features Kubernetes does not exercise** before relying on
       them (LS1–LS10, WT1–WT9).
    6. **Consider a support vendor that employs etcd maintainers**, and
       attend the biweekly SIG etcd triage meeting for issues that matter
       to you.

  [leases §3.1, §3.7, §8; watchers §10.2, Appendix B;
  chubby-perspectives §8.2; this file §5 "Independent testing"]
- **Crash consistency** (adjacent to the write path).
  - Power-loss fault-injection research (LazyFS, PVLDB 2024) found etcd
    restart failures.
  - A split-brain found with Jepsen and LazyFS (#14143) was closed not
    planned, without a root cause.
  - Jepsen's 2020 etcd analysis did not test storage faults.

  — **Moderate** (desk research, not re-read for this file)
  [wal-replacement §11.6].

  Added 2026-11-09, from the GitHub API: Jepsen filed the LazyFS issues in
  2022, while testing etcd 3.5.3 in a private engagement whose report was
  never published (jetcd report, footnote 1).
  - #14098: the WAL read back as `max entry size limit exceeded`. Closed
    after Jepsen traced it to LazyFS filling truncated bytes with `0x30`
    instead of zeros.
  - #14102, "Handle disk corruption": closed by the stale bot.
  - #14140 (stale read with loss of un-fsynced writes) and #14143: closed
    *not planned* in July 2025, after stale-bot warnings. A maintainer
    wrote in 2024 that #14140 had not been reproduced in two years; for
    #14143 a maintainer reported in 2023 that a contributor had reproduced
    it, but no root cause was found.
  - Two read-only-transaction panics Jepsen filed in 2022 were fixed
    (#14110, #14891).

  The jetcd report, in 2024, still says etcd "experiences a broad array of
  crashes when nodes encounter filesystem corruption or the loss of
  un-fsynced writes". — **High** for the states; Jepsen's statement is
  untested here.
- **Independent testing: Jepsen** (added 2026-11-09). The issues Jepsen
  filed against etcd and jetcd since 2019 that bear on correctness, and
  where they stand. Omitted: jetcd #673 (a duplicate of #674), jetcd #677
  (a channel-shutdown warning), and the two panics in the paragraph above.

  | Report | Finding | Upstream | Status at v3.7.3 / jetcd 0.8.7 |
  |---|---|---|---|
  | etcd 3.4.3 (2020) | KV reads, writes and transactions strict serializable under partitions, pauses, crashes, clock skew and membership changes | — | No published Jepsen test of the server since; the 2024 report attributes its anomalies to jetcd. etcd's robustness suite now validates linearizability of KV operations under injected faults (`tests/robustness/README.md:111`) |
  | etcd 3.4.3 | Single-key watches deliver every update in order | — | Consistent with this project's findings for safety; the tests could not see silent watches (§3) |
  | etcd 3.4.3 | Docs claimed "sequential consistency, the strongest"; defined linearizability by wall clock | (docs) | Fixed: the v3.7 API guarantees page says strict serializability, links Jepsen's definition, and quotes Herlihy and Wing for linearizability (`learning/api_guarantees.md:41`, `:68-74`, `:155`) |
  | etcd 3.4.3 | Locks do not give mutual exclusion; unfenced use lost updates | #11457 (docs) | Docs note added 2020 (PR #11490); unfenced `etcdctl` example still in the README (LS10) |
  | etcd 3.4.3 | A lock waiter whose lease expired was told it held the lock | #11456 | Fixed in v3.5.0 by a re-check after waiting (`client/v3/concurrency/mutex.go:96-105`); not in any 3.4 release |
  | etcd 3.4.3 | Revision 0 means "now", not "from the start"; undocumented | #11496 | Documented (PR #11559, 2020): first revision is 1; Go `Watch` comment (`client/v3/watch.go:68-69`) |
  | etcd 3.4.3 (2019 work) | jetcd 0.4.1 watch creation threw "Stream is already completed" | jetcd #674 | Closed by the stale bot. jetcd's watch client has since moved to Vert.x streams (`jetcd-core/src/main/java/io/etcd/jetcd/impl/WatchImpl.java:44-45 @ jetcd-0.8.7`); not re-tested |
  | 2022 engagement | Aborted read; lost update and cyclic information flow on crash | etcd #14092, #14890, jetcd #1072 | All closed without a fix: #14092 "as a jetcd issue" (2022), jetcd #1072 by the stale bot, #14890 on 2024-08-02 with a maintainer betting the cause was Jepsen, six days before the jetcd report showed it was jetcd's retries |
  | jetcd 0.8.2 (2024) | jetcd retries non-idempotent writes | etcd #18424, jetcd #1444 | Fixed in jetcd 0.8.5; residual lease and lock retries above |
  | 2022 engagement | Crashes and anomalies with lost un-fsynced writes or corruption | #14098, #14102, #14140, #14143 | Closed without fixes (above) |

  — **High** (issue states and code read 2026-11-09).

## 6. Common misconceptions

| Claim | Correction | Conf. |
|---|---|---|
| "Watches bypass Raft, so they lose events on failover." | Half right (corrected 2026-11-03). Events that arrive are never missing, reordered or retracted: they come from committed entries. But the watch path skips Raft's quorum check, so a watch on a member that is cut off from the quorum (a deposed leader included) or has stalled can stay silent indefinitely. That breaks the eventual consistency etcd claims for watches, and failover is only one trigger (WT1–WT4, §3). | High |
| "Lease renewals bypass Raft, and that is the bug." | Chubby does the same. The flaw is that expiry revocations carry no Raft term, and that a leader's authority is measured in Raft ticks while expiry runs on real time. | Moderate |
| "A partitioned leader breaks the renewal guarantee." | A new leader extends every lease by TTL + ET, which covers that case to within about one tick. | Moderate |
| "Kubernetes relies on etcd leases for leader election and node heartbeats." | It uses Lease API objects: ordinary keys updated by compare-and-swap. | High |
| "Watches are eventually consistent." (etcd's robustness README) | The documented contract guarantees what arrives, not that it arrives, and some silent paths persist after every fault has healed (§3). | High |
| "`WithRequireLeader` makes watches safe." | It is partial (WT2), and ignored through the proxy (WT5). A watch that sets it can still stay silent indefinitely with the quorum intact; only a freshness monitor (§3 mitigation 9) detects every such case. | High |
| "Replacing bbolt or the WAL format will fix write throughput." | Only if backend commits dominate (f_commit). The WAL already group-commits. | High (mechanism) |
| "Jepsen tested etcd, so its locks are safe." | Jepsen tested them and found the opposite: mutual exclusion failed within minutes with short TTLs, even in a healthy cluster. What it found safe was KV operations, and lock-guarded transactions (LS10). | High |
| "Jepsen found lost updates and aborted reads in etcd." | It found them in applications using jetcd ≤ 0.8.2, and traced them to the client retrying writes; the server behaved correctly. Fixed in jetcd 0.8.5 (§5). | High |
| "A failed write did not happen." | After a timeout or a lost connection the outcome is unknown, in any client. Retrying it blindly is how jetcd produced lost updates (§5). | High |
| "Watching from revision 0 replays the whole history." | 0 means "now"; the first revision is 1 (#11496). | High |

[leases §2.3, §4, §5.4, §7; leases-critique; chubby-perspectives-critique
E15; watchers §2.5, §3; wal-replacement Summary 1; storage §1]

## 7. Before deploying: questions to answer

**Coordination (leases and locks)**
- Do you use leases for mutual exclusion? Then:
  - Is every protected write fenced?
  - Can external resources check a fencing token?
- Do you rely on expiry for timely cleanup or failure detection? How often
  do leader changes occur in your environment (disk stalls, GC, network)?
  Each one extends every lease (LS1).
- How many lease grants and revokes per second, and how many keys per
  lease (LS7, WR3)?

**Watch consumers**
- Does each consumer persist its own revision cursor?
- Can it detect a silent watch?
- Does it need every intermediate change, or only the current state? After a
  compaction error only the current state is recoverable.
- How far behind can a consumer fall, and what does it do when it does
  (WT8)?

**Clients and topology**
- Which client library and version?
  - Is jetcd's pre-1.0 status acceptable?
  - Have you tested its resume and its 4 MiB limit (WT7)?
- Is a gRPC proxy or other intermediary in the path?
- Do you use `concurrency.Mutex` and `Election`, or the server Lock API, as
  jetcd does?
- jetcd: is it at 0.8.5 or later? Does any code call `withAutoRetry()`, or
  set `retryMaxAttempts` above 0 on an older version (§5)?
- What does your code do when a write fails with a timeout or
  `UNAVAILABLE`? Is every write either guarded or harmless to repeat, and
  is the outcome resolved by reading back?

**Capacity**
- What is the peak write rate, value size distribution, history retention
  and DB size?
- Can the data be split across clusters?
- What is the `fdatasync` p99 of the WAL device under load? Is the storage
  shared or networked?
- What is the round-trip time between members, for example across
  availability zones?

**Operations**
- Do your dashboards and alerts cover:
  - WAL fsync and backend commit latency;
  - apply lag;
  - lease expiry rate;
  - watch lag?
- Is there a written procedure for restore (WT10), defrag (WR4) and
  version upgrades (§1 floor)?

**Upstream**
- Track the fixes that would retire items above: #22650 (LS3); #22087,
  #22253 and #22291 (LS5); #21065 (WT8); PR #22223 and #22332 (WR2, WR3).
  Who will watch these, and is there a vendor to escalate to?
- Which of the features you rely on are outside Kubernetes' usage? Who
  will diagnose and fix defects in them, and carry the patch until it
  ships upstream?
- Who keeps your upstream issues active (or labelled `stage/tracked`), so
  that the stale bot does not close them (§5)?
- If you use jetcd: who maintains it in practice, and are you prepared to
  contribute fixes yourself?

**Concepts to understand first**
- Fencing tokens.
- Linearizable versus serializable reads.
- Watch cursors and compaction.
- That lease TTL is a lower bound.
- That etcd's write cost is mostly synchronous disk flushes.

## 8. Limits of this file

- **No measurements.** Performance statements are code structure plus
  others' numbers. Two functional Docker tests exist: the proxy
  `WithRequireLeader` defect and `client/v3/leasing`.
- **Mostly static analysis.** The residual lease windows (LS2), the apply
  stall case (WT2) and the jetcd resume skip (WT7) are inferred from code,
  not reproduced.
- **The maintenance assessment (§5) is drawn from cases**, read in issue
  threads and governance documents, not from a systematic count of issue
  outcomes on and off Kubernetes' path. Such a count is the test that would
  confirm or refute it.
- **Jepsen's results are external and older.** They cover etcd 3.4.3 and
  jetcd 0.4.1 (2019–2020), and jetcd 0.5.0–0.8.2 (2024; the report does
  not name the server version, and its harness at `fbe693e` defaults to
  etcd 3.5.15).
  This project re-ran none of them; their fixes were checked in code only.
- **Not covered:** membership changes, upgrades and downgrades, auth beyond
  revocation cost, TLS beyond #15402, multi-region deployment, and backup
  tooling.
- **What would change these conclusions:**
  - a reproduction of a residual lease window would raise LS2 to High;
  - benchmarks on the target hardware would replace the WR magnitudes;
  - the fixes listed in §7 landing would retire their rows.

## Appendix A. Where the details are

| Topic | File and section |
|---|---|
| Lease API promise, failover mechanics, properties | [`leases-claude.md`](leases-claude.md) §1–§2 |
| Failover bugs #12528/#15247, residual windows, #22649 | `leases-claude.md` §3 |
| Design flaw relative to Chubby (term-less revocation, tick-based authority) | `leases-claude.md` §4, §11.1 |
| Go and jetcd lease clients (LS4, LS5) | `leases-claude.md` §5 |
| Renewal and expiry cost (LS6, LS7) | `leases-claude.md` §6 |
| Kubernetes and leases | `leases-claude.md` §7 |
| Lease issue survey with states | `leases-claude.md` Appendix A |
| Lease claims rejected from other tools' analyses | [`leases-critique-claude.md`](leases-critique-claude.md) |
| Watch contract and what it disclaims | [`watchers-claude.md`](watchers-claude.md) §2 |
| Watch failure states, one table | `watchers-claude.md` §4 |
| `WithRequireLeader`, cut-off and stuck members (WT1–WT3) | `watchers-claude.md` §5–§6 |
| Restore, compaction and resume bugs; version floor | `watchers-claude.md` §7, Appendix A |
| Slow watchers, stream protocol (WT8, WT9) | `watchers-claude.md` §8 |
| gRPC proxy (WT5), with Docker reproduction | `watchers-claude.md` §9, Appendix C |
| Go and jetcd watch clients (WT6, WT7) | `watchers-claude.md` §10 |
| Kubernetes watch defences and gaps | `watchers-claude.md` §11 |
| Full watch recommendation list | `watchers-claude.md` §13 |
| Open upstream watch items | `watchers-claude.md` Appendix B |
| Write pipeline, backend commit triggers (WR1, WR3–WR5, WR8) | [`storage-claude.md`](storage-claude.md) §2 |
| Backend performance model; tuning without code changes | `storage-claude.md` §3, §4.3 |
| Metrics for attribution; experiments E0–E6 | `storage-claude.md` §3.4, §10 |
| Replacing bbolt (Pebble vs RocksDB), upstream #22332 | `storage-claude.md` §4–§9 |
| WAL internals, Raft loop, cost per member (WR2, WR6) | [`wal-replacement-claude.md`](wal-replacement-claude.md) §1 |
| Async storage writes, a two-store design, staging | `wal-replacement-claude.md` §3–§8 |
| Prior art: crash consistency, device `fdatasync` figures, etcd's own numbers | `wal-replacement-claude.md` §11.6–§11.8 |
| Chubby comparison of leases, watches and locks; per-issue verdicts | [`chubby-perspectives-claude.md`](chubby-perspectives-claude.md) §1, §5–§7 |
| Lock recipe defects (LS8) | `chubby-perspectives-claude.md` §7 |
| `client/v3/leasing` defects, with functional test (LS9) | `chubby-perspectives-claude.md` §8; program in [`chubby-perspectives-critique-claude.md`](chubby-perspectives-critique-claude.md) Appendix A |
| Client-side borrowings (conservative deadline, session jeopardy) | `chubby-perspectives-claude.md` §10 |
| Jepsen: lock results, #11456, docs (LS8, LS10) | `leases-claude.md` §2.3, §8 |
| Jepsen: client retries, lease and lock RPC retry classes | `leases-claude.md` §5.4 |
| Jepsen: watch test design, revision 0 (#11496) | `watchers-claude.md` §2.5 |

Independent analyses of the same questions by other tools (`-codex`,
`-agy` suffixes) are listed in [`INDEX.md`](INDEX.md). This file draws only
on the Claude files.
