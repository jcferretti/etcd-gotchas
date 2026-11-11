# etcd in production: lease, watch and write-performance risks, and how to deploy around them (Claude)

**Versions analyzed:** etcd `v3.7.3` = `0a173d0410a9`; jetcd `jetcd-0.8.7` =
`23b61f1499a5`; Kubernetes `v1.37.2` (selected files, as in the source
analyses). Tag check on 2026-11-02: no newer stable release of either. Tag
check on 2026-11-09: jetcd `jetcd-0.8.8` was released that day (GitHub and
Maven Central); it is not analyzed here, and the clone stays at 0.8.7 until
the user confirms the move (AGENTS.md §4). Its diff against 0.8.7 was read on
2026-11-10 (main sources only): it fixes the keep-alive defect LS11, and it
touches neither `WatchImpl.java` nor the code that decides which calls are
retried (`Errors.java`, `KVImpl`, `LockImpl`, `ElectionImpl`).

**Method:** a synthesis of the seven Claude analysis files listed in
Appendix A, first written from them as committed at `5102a18`. The section
numbers cited below are those of the current files, which later passes
extended (`leases-claude.md` §5.4 and `watchers-claude.md` §2.5 did not exist
at that commit). No new code analysis, except that the
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
`leases-claude.md` §2.3, §5.4, §8 and `watchers-claude.md` §2.5. An
**accuracy review** (2026-11-10) re-checked this file's claims against the
code at `v3.7.3` and `jetcd-0.8.7`, Kubernetes `v1.37.2`, the two Jepsen
reports and the issue threads as they stood that day. Its corrections are
in place below; those that change a conclusion are marked "2026-11-10". It
also added what upstream reported after the source analyses were written:
etcd #22714 (LS2), jetcd 0.8.8 (LS11) and etcd #22687 (WT5). Nothing was run for this file. Each claim keeps the confidence its source gives it; the "See" column
or bracket points to the evidence. **This project has benchmarked nothing**
(AGENTS.md §7): every number below is upstream's, a reporter's, or derived
from code.

**Upstream references (most important):** etcd
[#15247](https://github.com/etcd-io/etcd/issues/15247),
[#12528](https://github.com/etcd-io/etcd/issues/12528),
[#22714](https://github.com/etcd-io/etcd/issues/22714),
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
[#13915](https://github.com/etcd-io/etcd/issues/13915),
[#20242](https://github.com/etcd-io/etcd/issues/20242),
[#21051](https://github.com/etcd-io/etcd/issues/21051); CNCF TOC
[#898](https://github.com/cncf/toc/issues/898); SIG etcd
[charter](https://github.com/kubernetes/community/blob/master/sig-etcd/charter.md); jetcd
[#1379](https://github.com/etcd-io/jetcd/issues/1379),
[#1383](https://github.com/etcd-io/jetcd/issues/1383),
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

- **Lease and watch defects break timing and liveness, not stored data.** No
  lease or watch defect found corrupts the replicated key-value state.
  — **High**. Storage faults are a separate matter, with unresolved reports
  (§5, crash consistency).
- **Leases.** TTL is a minimum, not a deadline: every leader change resets
  it. Lease-based locks give mutual exclusion only with fencing. Both client
  libraries can keep renewing a lease the application believes is gone, and
  jetcd up to 0.8.7 can also stop renewing without saying so (LS11).
  — **High** (code). One residual failover window now has an upstream
  reproduction on v3.7.2, which this project has not re-run (#22714, LS2;
  added 2026-11-10). The others are inferred and unreproduced
  (**Moderate**).
- **Watches.**
  - On direct connections to a current release, the events that arrive are
    in order and complete, except for clients that reuse explicit watch
    IDs (WT9; neither the Go client nor jetcd sets one). — **High** for the
    fixed bugs; **Moderate** that no unknown server defect remains.
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
- **Upstream capacity is limited, and in practice its priorities follow
  Kubernetes** (rewritten 2026-11-09; headline corrected 2026-11-10; §5).
  - etcd is governed as a Kubernetes SIG. Its charter's scope is "Owns the
    etcd project and how it is used by Kubernetes", and the SIG says it
    also covers etcd "as a standalone solution". In 2022 its maintainers
    described it as "effectively unmaintained".
  - Issues close automatically after about four months without activity
    (after about ten weeks from 2025-05 to 2026-01, when the bot ran on its
    defaults).
  - Defects outside Kubernetes' usage have taken years to assess or fix, or
    closed without a fix. Examples:
    - Jepsen's 2022 anomaly reports. Two were closed in 2022 without a fix.
      The third was closed in 2024 with the report called "inaccurate",
      six days before Jepsen's published analysis confirmed the cause in
      etcd-io's own Java client, which the closing maintainer had also
      named.
    - A lease failover bug fixed about three years after its first report,
      with a related case reported again in 2026 (LS2).

  — **High** for the record; **Moderate** that defects off Kubernetes'
  path fare worse (examples, not a systematic count). Plan to own the
  fixes you need.
- **Independent testing agrees, at older versions** (added 2026-11-09, §5).
  - Jepsen found etcd 3.4.3's key-value operations strict serializable
    under partitions, pauses, crashes, clock skew and membership changes,
    and single-key watches complete and in order.
  - It found that etcd locks do not give mutual exclusion, even in a
    healthy cluster. With process pauses, an unfenced lock lost about 18 %
    of acknowledged updates. Fencing through a guarded transaction was safe
    in its tests.
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
- **Revision:** etcd's cluster-wide logical clock. Each transaction that
  changes keys increments it by one.
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
  reports on etcd 3.4.3 (funded by the CNCF) and jetcd 0.8.2 are summarized
  in §5.
- **ET:** the Raft election timeout, 1 s by default.

IDs in this file (LS, WT, WR) are local to it. They do not match the L/W
numbering of `chubby-perspectives-claude.md` or the P/R numbering of
`leases-claude.md`.

## 1. Bottom line

1. **Run a current release.** The minimum versions are **v3.7.1, 3.6.14 or
   3.5.33**; this project analyzed v3.7.3. Older releases carry fixed bugs in
   lease failover safety, watch ordering and completeness, and stale
   linearizable reads (note under the §3 table; [watchers §13.1,
   Appendix A]). No release yet fixes the lease failover case reported in
   #22714 (LS2). For jetcd, the floor is **0.8.5** for writes: earlier
   versions retry them after indefinite failures (§5, Jepsen). Applications
   that keep leases alive should use **0.8.8**, which fixes LS11; this
   project analyzed 0.8.7 and read only the 0.8.8 diff (corrected
   2026-11-10).
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
9. **Budget to fix what you depend on outside Kubernetes' usage**: lease
   keep-alive (Kubernetes only grants leases and lets them expire), locks,
   several watchers per stream, the proxy and jetcd. Upstream capacity is limited and issues close on inactivity, so
   keep the issues you need tracked and active (§5, "Project scope and
   maintenance capacity").

## 2. Leases

| ID | Issue | Scenario | Impact | Status at v3.7.3 | Conf. | See |
|---|---|---|---|---|---|---|
| LS1 | **TTL is a minimum, not a deadline** | Every leader change or restart resets each lease to full TTL + ET | Expiry-driven cleanup and failure detection are late; under repeated leader changes, indefinitely (Kubernetes Events, k8s#65497) | By design. The fix, `LeaseCheckpoint`, is Alpha and off, and does not help short leases or frequent leader changes | High | leases §1.3–1.4, §2.2 |
| LS2 | **A leader change can revoke a live lease** | Historically, a stalled leader revoked leases that its successor was renewing (#12528, #15247). At v3.7.3, five windows remain, inferred from code: a stall ending, a large expiry batch, a renewal forwarded to a stalled ex-leader, a renew/revoke race, and a correct revocation committing after a leader change. **The large-batch window is now reported upstream with a reproduction** (#22714, filed 2026-11-06; added 2026-11-10). A leader is partitioned just as more than 16 leases expire on it. The revocations queued behind its 16-slot limit are proposed after it has stepped down, forwarded to the new leader and applied. The reporter lost 168–184 of 200 leases that clients were renewing on the new leader, in 10 of 10 trials on v3.7.2 and on v3.6.15 | Without fencing: two holders of one lock or election. Keys of live, renewing clients deleted | The cases reported in 2020–2023 are fixed in 3.4.31, 3.5.13 and 3.6.0, by a heuristic leadership check rather than a fence. #22714 is open, with fix PR #22730 open. Its mechanism matches the code at v3.7.3 (one leadership check per batch, `server/etcdserver/server.go:867`; revocations applied unconditionally), but this project did not re-run it and no maintainer has confirmed it. The other windows are not reproduced | #22714: High (mechanism, from code); Moderate (the reported loss rates). Other windows: Moderate (existence; High for the forwarded renewal); Low (impact) | leases §3.1–3.4 |
| LS3 | **#22649: checkpoint bug** | Only with `LeaseCheckpoint` on: a renewal reports the full TTL while the server keeps a shorter one | Early revocation | Open; fix PR #22650 open | High (code) | leases §3.5 |
| LS4 | **Client lease deadlines are optimistic** | Go and jetcd set the deadline to *response receipt + TTL* and check it once a second; jetcd uses the wall clock | A holder still believes it holds the lock for about 1 s plus the reply delay after the server expired it, and without bound under GC pauses or clock steps | By design | High (code); Moderate (size) | leases §5.1 |
| LS5 | **Lease renewed after the application thinks it is gone** | Go #22081, #22094: only when the application passes a `WithRequireLeader` context to `KeepAlive`. jetcd: any keep-alive stream error calls `onError`, but jetcd keeps renewing | Lock keys never expire; other waiters block; keys leak | Go: open, fix PRs open; #22094 and two of the three fix PRs carry the stale label (2026-11-10). jetcd: by design since 0.8.0 | High (code); Moderate (consequence) | leases §5.2–5.3 |
| LS6 | **Renewals through followers fail under leader apply backlog** | A follower forwards each renewal to the leader, which waits up to 1 s for its apply loop; after 7 s of retries the follower fails the stream. `FastLeaseKeepAlive` (Beta, on) helps only renewals sent to the leader itself | If a backlog outlasts the TTL, leases of clients that renew on schedule expire. About (N−1)/N of clients go through followers (inferred) | Structural | High (code); Moderate (impact; backlog frequency unmeasured) | leases §6.1 |
| LS7 | **Expiry is costly, and stalls under overload** | One Raft entry per expired lease. At most 1000 leases/s are handed over for revocation, with 16 in flight per batch; batches can overlap under load, so the total is not capped at 16. All attached keys are deleted in one transaction, which forces a synchronous backend commit (WR3). Revocations are rejected like any write once the apply loop lags by more than 5000 entries. With JWT auth, one token signature per revocation (#21735) | Expiry waves consume write capacity. Expired keys linger exactly when the cluster is overloaded. A lease with very many keys can stall the cluster (k8s#96836) | Structural; `PriorityRequest` gate is Alpha, off | High (code); magnitude unmeasured | leases §6.2 |
| LS8 | **Lock recipe defects** (`concurrency.Mutex`; the server Lock API that jetcd calls runs the same code) | A waiter keeps waiting after its own session expired (#19597). A retried `Unlock` can delete a newer acquisition by the same session (#22082). Waiters wake through a watch, so they inherit WT1. Historical, found by Jepsen: a waiter whose lease expired while it waited was told it held the lock (#11456). `Election.Campaign` still has that pattern: it returns after waiting without checking that its own key exists (`client/v3/concurrency/election.go:93-106 @ v3.7.3`, unchanged on `main`; added 2026-11-10, from code, not reproduced). jetcd retries a `Lock` call after `UNAVAILABLE`; if the first attempt's handler is still waiting on its member, its cleanup can delete the lock key that the retry reused (added 2026-11-09; inferred) | Stalled lock waits; one lock released wrongly; a spurious "session expired", or rarely a false grant, after a jetcd retry | #19597, #22082 open. #11456 fixed in v3.5.0 for `Mutex.Lock` only, never backported to 3.4. jetcd retry race not reported upstream | High; for the missing `Campaign` check, High (code) with the consequence inferred; Low for the jetcd retry race (inferred, not reproduced) | chubby-perspectives §7; leases §2.3, §5.4 |
| LS9 | **`client/v3/leasing` is unsafe** (also exposed as `grpc-proxy --experimental-leasing-prefix`) | A cut-off owner serves stale "linearizable" reads (#19091, closed by the stale bot). Writes by non-owners are applied twice, with wrong `Delete` and `PrevKV` responses (reproduced). `EvictRange` evicts the wrong key | Incorrect reads and duplicated writes | Unfixed, also on `main` | High | chubby-perspectives §8 |
| LS10 | **Unfenced locks fail in practice, not only in theory** (Jepsen, etcd 3.4.3; added 2026-11-09) | Locks with 1–3 s TTLs failed to give mutual exclusion within minutes, in healthy clusters, with no faults injected; pauses and partitions made it faster. With 2 s TTLs, five processes and a pause every 5 s, a lock-protected read-modify-write of an in-memory set lost about 18 % of acknowledged updates. A read-modify-write of a set stored in etcd, whose write was one transaction guarded on the lock key, was safe | Two holders at once; lost updates in whatever the lock protects | Inherent to leases (LS2, LS4). Jepsen's runs also included #11456 (LS8), fixed in v3.5.0. The docs have warned since 2020 (`learning/why.md:82-98`, v3.7 docs), but `etcdctl/README.md:1175-1179 @ v3.7.3` still shows an unfenced `etcdctl lock mylock ./etcdctl put foo bar`, the pattern Jepsen flagged, and the same "why etcd" page still lists "APIs for safe distributed shared locks" (`:46`) | High (Jepsen's measurement on 3.4.3). That unfenced locks are unsafe is unchanged at v3.7.3; the rates were not re-measured, and they include #11456 | leases §2.3 |
| LS11 | **jetcd ≤ 0.8.7 stops renewing after a clean stream end** (added 2026-11-10) | jetcd opens its shared keep-alive stream with no end handler (`jetcd-core/src/main/java/io/etcd/jetcd/impl/LeaseImpl.java:203-207 @ jetcd-0.8.7`). If the stream ends without an error, jetcd neither reopens it nor reports anything. An etcd member ends the stream cleanly only when the client closes its side (`server/etcdserver/api/v3rpc/lease.go:113-116 @ v3.7.3`), so the likely source is an intermediary such as a proxy or load balancer. The upstream fix also names a member shutting down gracefully, which this project did not confirm | Every lease of that client stops being renewed and expires while client and cluster are healthy. Observers get `onCompleted` at the local deadline | jetcd #1383 (reported 2024-07 on 0.7.7), fixed in 0.8.8 by PR #1628 | High (code); Low–Moderate (reachability on a direct connection to v3.7.3) | leases §5.3 |

**Kubernetes.**
- **What does not apply.** kube-apiserver never calls `KeepAlive`, and its
  leader election and node heartbeats use `coordination.k8s.io` Lease
  objects, which are ordinary keys. So LS3–LS5 do not apply to it, and LS2
  largely does not: a lease that is never renewed has no renewal for a
  stale leader to undo.
  — **High** (code: a search of all non-vendor Go files at v1.37.2 finds no
  etcd `KeepAlive` call; corrected 2026-11-10); **Moderate** for LS2
  (reasoned).
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
   unique to its lease, provided each lease is used for one acquisition.
   `etcdctl lock` exports `ETCD_LOCK_KEY` and `ETCD_LOCK_REV` to its
   command. `ETCD_LOCK_REV` is the store revision at acquisition, not the
   key's create revision: the two are equal only when the lock was acquired
   without waiting (`etcdctl/ctlv3/command/lock_command.go:126-131 @
   v3.7.3`). It serves as an increasing token for an external resource
   (inferred), but not as `rev` in the compare above.
2. **Do not treat TTL as a deadline.** Where cleanup must be bounded,
   delete explicitly and use the lease only as a backstop (LS1).
3. **Leave `LeaseCheckpoint` off** until PR #22650 ships, unless keeping long
   leases accurate across leader changes matters more than LS3.
4. **Go:** do not pass `WithRequireLeader` contexts to `KeepAlive` until
   #22081 and #22094 are fixed. `concurrency.Session` is unaffected by
   default.
5. **jetcd:** use 0.8.8 or later if you keep leases alive (LS11). Treat any
   `onError` other than `NOT_FOUND` as transient. To give up a lease,
   `close()` the keep-alive *and* revoke the lease.
6. **Use a margin of your own:** consider a lease lost at *send time of the
   last acknowledged renewal + TTL − margin*, not when the library says (LS4;
   chubby-perspectives §10, B1).
7. **Locks:** give `Lock()` a deadline and abandon it when the session ends
   (LS8). Release with an owner-compare delete. After `Campaign` returns,
   confirm leadership with a fenced write (`Proclaim` is one) before acting
   on it. jetcd retries `Lock` without telling the caller, so either build
   the client with `retryMaxAttempts(0)` or treat every acquired lock as
   unproven until a fenced write succeeds (LS8; leases §5.4).
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
  compaction, resume on a lagging member), with one exception: a client
  that reuses an explicit watch ID over raw gRPC can receive a cancelled
  watch's events (WT9, #22673, open). The Go client and jetcd never set a
  watch ID.

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
    transfer has no connected target, or when a member add or remove would
    break the active quorum (`server/etcdserver/server.go:1301-1303`,
    `:1427-1434`, `:1634-1645`).

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
    notifications are sent only on request, then every 10 min.
- **It is not eventual consistency.**
  - etcd's robustness suite states: "Etcd provides strict serializability
    for KV operations and eventual consistency for Watch"
    (`tests/robustness/README.md:111 @ v3.7.3`).
  - Eventual consistency is a liveness guarantee.
    - Vogels: "if no new updates are made to the object, eventually all
      accesses will return the last updated value" ("Eventually
      Consistent", CACM 2009; quoted from the author's online copy,
      2026-11-10).
    - Shapiro et al. state it as eventual delivery: an update delivered at
      one correct replica is eventually delivered at all correct replicas
      ("Conflict-free Replicated Data Types", SSS 2011; recalled, not
      re-read).

    "Might never arrive" is the negation of both, not a description of
    either.
  - WT1 meets it only in the weakest reading: events arrive once the
    partition heals, although the client could have resumed on a working
    member all along.
  - Other paths stay silent after every network fault has healed, so they
    violate even that reading:
    - a proxy watch attached to a dead broadcast (#21813);
    - a Go client deadlocked in teardown (#17611);
    - a connection that died unnoticed, with no client keepalive (WT3);
    - a member whose apply loop wedges, if #22184 has a production trigger
      (WT2).
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
| WT2 | **`WithRequireLeader` is a partial fix** | It cancels after about 3–5 s. It misses a member that has a leader but does not apply (one candidate cause: the #22184 lock inversion; no production trigger confirmed). It also misses a member whose raft loop is blocked, for example on a WAL `fdatasync` that does not return: that member's leader view freezes, and it keeps answering keepalive pings (added 2026-11-03). It ends the watch instead of resuming it, and Go discards the resume point, so a re-watch "from now" skips the writes made meanwhile. Watches that set it can also be silenced by WT5, WT6 and the #18879 stream freeze (WT8) | Indefinite silence on such a member despite the option; silent gaps for clients that re-watch from "now" | By design | High (code); Moderate (the apply-stall case); blocked raft loop not reproduced | watchers Summary, §5.2–5.3, §6 |
| WT3 | **Paused member or dead connection** | The Go client has no gRPC keepalive by default and retries forever; it raises no error when no member is reachable | Indefinite hang | Default behaviour | High | watchers §5.4 |
| WT4 | **Freshness signals are local** | The "now" start point, the `Created` revision and progress notifications reflect only the serving member. Periodic progress notifications are off unless requested, and the server interval is 10 min. `RequestProgress` goes unanswered if any watcher on the stream is behind or starts at a future revision (#20242) | "No changes" looks the same as "stalled" | By design | High | watchers §1, §2.3, §8.2 |
| WT5 | **The gRPC proxy breaks failure signalling** | It ignores `WithRequireLeader` (reproduced; not reported upstream). It attaches watchers to dead upstream watches (#21813). It swallows upstream errors. It drops `RequestProgress` (#19710, closed not planned). A second event-loss report is open: a watcher that starts far behind is merged into a current broadcast and skips the events in between (#22687, filed 2026-11-02; read, not analyzed here; added 2026-11-10) | Permanent silent loss of updates; no freshness check possible | Open | High (code); Moderate (how often #21813 occurs) | watchers §9 |
| WT6 | **The Go client can lose failure signals** | If the application does not read within 250 ms, the final error is dropped and the channel closes bare, which looks like the caller's own cancellation. A narrow race deadlocks teardown, and later `Watch()` calls block (#17611, closed without a fix) | A missed failure; a hang | Open | High (code); Moderate (the race) | watchers §10.2 |
| WT7 | **jetcd watch defects** | Resume skips historical events if the stream breaks after `Created` and before the first batch (not reproduced). A response over 4 MiB ends the watch, and re-watching fails at the same revision (jetcd#1379). jetcd can stop retrying after one failed resume | Silent gaps; a watch that cannot progress | Open | Resume skip: Moderate (mechanism), Low (frequency). 4 MiB limit: High (mechanism), Moderate (reachability). Failed resume: High (code) | watchers §10.3–10.5 |
| WT8 | **Slow and far-behind watchers** | Multi-second delivery lag (#16839). A watcher far behind blocks writes: the reporter of #22264 saw writes blocked more than 90 % of the time on 3.5.21, and says it is still present in 3.7.1. Bursts of cancellations freeze a shared stream of the in-process client, which embedded etcd and the server-side Lock and Election APIs use (#18879; fix PR #21065 open); for networked clients the freeze is inferred. The Go client buffers without limit. A long delay lets compaction remove `PrevKv` | Delay; write stalls; errors in consumers that need the previous value | Open | Moderate; High for the #18879 freeze on the in-process client | watchers §8.1 |
| WT9 | **Raw-gRPC watch-ID reuse** | Reusing an explicit `watch_id` delivers a cancelled watch's events to the new watch (#22673) | Wrong events for clients that reuse explicit IDs; the Go client and jetcd never set one. The server buffer behind the defect also grows on any long-lived stream | Open | Moderate | watchers §8.2 |
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
  - mass watch teardown, but only when etcd stays without a leader for
    longer than the 30 s during which the watch cache now retries
    (k8s#111116, fixed in 2022 by PR #111387;
    `staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go:451-454 @
    v1.37.2`; corrected 2026-11-10).

— **High** (code at v1.37.2) for the defences; **Moderate** for the residual
exposure [watchers §11].

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
`fdatasync` on a quorum of members → the single apply loop → reply
(`server/etcdserver/server.go:1982 @ v3.7.3`). Backend commits are not a
step of that chain (corrected 2026-11-10). They run every 100 ms, or at once
for an entry that deletes from the backend, and they reach clients as apply
stalls, because they hold locks that the apply loop and new reads need
(WR3). Nothing here was measured.

| ID | Issue | Mechanism | Impact | Conf. | See |
|---|---|---|---|---|---|
| WR1 | **One log, one apply loop per cluster** | Every write goes through one Raft log and one apply goroutine. Writes are rejected (`ErrTooManyRequests`) when apply lags commit by more than 5000 entries | A fixed ceiling per cluster; the only scale-out is splitting data across clusters | High | storage §2.1, §1 |
| WR2 | **`fdatasync` latency sets the pace** | One WAL `fdatasync` per Raft batch on every member. The leader counts its own copy of an entry toward the quorum only once it is durable. The Raft goroutine waits during the sync (asynchronous storage writes are not enabled in v3.7.3). Device `fdatasync` latency spans microseconds (enterprise NVMe with power-loss protection) to milliseconds (consumer NVMe without it). For cloud network block storage the sources hold mostly vendors' I/O latency targets (single-digit milliseconds), which are not `fdatasync` measurements, and one measurement of about 47 µs | Disk choice matters more than CPU | High (mechanism); Moderate (device figures, desk research) | wal-replacement §1.2–1.3, §11.8 |
| WR3 | **Backend commits block applies and new reads** | Each bbolt commit runs 2 `fdatasync`s while holding locks that block the next apply and the start of new reads. It runs every 100 ms, plus synchronously for every entry that deletes from the backend (**every lease revocation**) and every 1000-key compaction batch | Lease churn and compaction cause commit storms. At about 1 ms per `fdatasync`, revocations cap near 500/s and stall other writes meanwhile | High (mechanism); Low–Moderate (magnitude) | storage §2.3, §3.1 |
| WR4 | **Defrag stops the member** | In 3.7.x, defrag locks the backend for the whole copy; non-blocking defrag exists only on `main`, for v3.8 | The member is unavailable for a time that grows with DB size | High | storage §3.1 |
| WR5 | **Full scans that stall** | MVCC compaction rescans from revision 0 every time. HashKV corruption checks (`--corrupt-check-time`, default 0 = off) hold a lock for a full scan, stalling applies | Periodic latency spikes that grow with DB size | High (code) | storage §2.2–2.3 |
| WR6 | **WAL file mechanics** | Segments are preallocated with `fallocate`. On ext4 and XFS the first write into that space forces a filesystem-journal commit at `fdatasync` (one external measurement, on ext4: 20k vs 99k IOPS). Every 64 MB, segment rotation runs 2 `fdatasync`s, a rename and a directory fsync on the Raft goroutine | Periodic tail-latency spikes; extra sync cost | High (mechanism); Low (gain for etcd, unmeasured) | wal-replacement §1.1, §11.3 |
| WR7 | **Watches and leases load the write path** | WT8 (a far-behind watcher blocks writes, #22264); LS7 expiry waves; #22450, a leader apply backlog that fails to recover after a transient stall (open; reported on 3.5.21 and 3.6.6, not checked on 3.7) | Write stalls caused by read-side or lease activity | Moderate | watchers §8.1; leases §6 |
| WR8 | **All keys live in RAM** | The key index (every key and revision) is in memory whatever the backend; the DB size is bounded by the quota | Memory, not disk, caps the key count | High | storage §2.2 |

**Reference numbers.**
- The newest official write numbers are for **etcd 3.2.0**: 44–50k QPS with
  256-byte values, 1000 clients, on 8-vCPU GCE machines. (The v3.7 docs
  also carry benchmark pages for 2.x releases.) A source comment assumes a
  maximum of about 10k [wal-replacement §11.7].
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
   - Commits per second well above 10 mean forced commits: backend deletes
     (mostly lease revocations), compaction batches, or the
     10,000-operation batch limit.
   - Also watch the WAL fsync histogram, the apply lag (committed − applied)
     and `etcd_debugging_lease_revoked_total`, which counts every revocation
     applied, on each member. (`etcd_debugging_server_lease_expired_total`
     counts expiries only, on the leader. Corrected 2026-11-10: the name
     given here before does not exist.)
   - If the apply loop is the bottleneck, a faster backend can raise
     throughput by at most about 1/(1 − f_commit). If it is not, the gain is
     about nothing.
3. **Partition data that needs no cross-key atomicity** into separate
   clusters. Kubernetes can move Events to their own cluster (kube-apiserver
   `--etcd-servers-overrides`). AWS EKS reports up to 5× write
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
8. **Benchmark on the target hardware at fixed offered rates**
   (`tools/benchmark put --rate`). The tool is a rate-capped closed loop
   that times each request from the moment it is sent, so it understates
   stalls; give it enough `--clients` that they never all block. Include
   compaction, watchers and lease churn, not only puts. Lease churn needs
   its own harness: `tools/benchmark` has no grant/revoke workload
   [storage §10].

## 5. Cross-cutting risks

- **gRPC proxy.**
  - WT5.
  - The same `WithRequireLeader` defect applies to lease keep-alive streams.
  - It serves LS9's leasing cache.
  - The defect has been present since v3.3.0 and is not covered by e2e tests.

  [watchers §9; leases §5.2.4]
- **Client libraries are the weakest layer.**
  - Go: LS5, WT3, WT6.
  - jetcd 0.8.7, which is pre-1.0: LS4 (wall clock), LS5, LS11 and WT7. Its
    test launcher defaults to etcd v3.5.14, which has known watch bugs. Its
    CI overrides that with v3.5.29 and v3.6.11
    (`.github/workflows/build-pr.yml:46-47 @ jetcd-0.8.7`), which lack only
    the watch-auth fix. No test runs against etcd 3.7 (corrected
    2026-11-10).

  — **High** (code) [leases §11.2; watchers §15.3].
- **Automatic retries of writes** (Jepsen, jetcd 0.8.2; added 2026-11-09).
  - **What Jepsen found.** jetcd retried calls that failed with
    `UNAVAILABLE`, including a `Txn` already sent and possibly executed.
    Jepsen evaluated 0.5.0, 0.6.0 and 0.8.2, and reported 0.6.0 to 0.8.2
    affected. By code the range is wider (corrected 2026-11-10): the retry
    entered in 0.5.8 (commit `70c463c`) and is still in 0.8.4, which exists
    on Maven Central only (2024-11-14), with no git tag or GitHub release.
    With only `kill -9` of servers, one
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
  - **The Go client at v3.7.3** retries a write in four cases (corrected
    2026-11-10; `client/v3/retry.go:85-94`,
    `client/v3/retry_interceptor.go:76-98`, `:341-343`):
    - on an error showing that no connection was available;
    - after refreshing an invalid auth token;
    - on a learner's refusal, when the client has several endpoints;
    - on a `Canceled` or `DeadlineExceeded` status while the caller's own
      context is live.

    The first three are definite rejections. For the fourth, a
    contributor's analysis in #18424, accepted by a maintainer, found that
    the client never saw such a status from the server, and that the
    problem could not be reproduced with the Go client. The maintainer
    added that this retry "might not be safe ... because the server side
    may have successfully processed the request". — **Moderate** (their
    analysis). Its premise no longer holds from v3.7.0: the server now maps
    a cancelled proposal to `Canceled`
    (`server/etcdserver/api/v3rpc/util.go:54 @ v3.7.3`; absent on
    `release-3.6`). Whether a client whose context is live can receive that
    status after the write executed is not established. The Go client also
    retries lease grant and revoke on any `Unavailable`.
  - **Consequences of the remaining retries:** an orphan lease that lives
    until its TTL runs out; a "lease not found" error for a revoke that
    succeeded; and, in jetcd, the LS8 lock race. — **High** (code);
    consequences inferred, not reproduced.
  - **The server cannot deduplicate.** Write requests carry no
    client-supplied request ID, and the server keeps no record of the
    requests it executed, so an indefinite failure must be resolved by the
    application. #18424, which
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
    etcd ([announcement](https://etcd.io/blog/2023/introducing-sig-etcd/),
    dated 2023-11-01).
    - The [charter](https://github.com/kubernetes/community/blob/master/sig-etcd/charter.md)'s
      scope begins: "Owns the etcd project and how it is used by
      Kubernetes". It lists "the implicit Kubernetes-ETCD Contract" among
      its responsibilities. The SIG's README adds that etcd should also
      serve "as a standalone solution", and the announcement says the scope
      "is not exclusively about etcd as a Kubernetes component".
    - etcd's approvers are the SIG's chairs and tech leads, plus one other
      person (`OWNERS`, `OWNERS_ALIASES @ v3.7.3`). The project uses
      Kubernetes' Prow bot for pull-request commands and approvals
      (maintainers merge by hand) and Kubernetes' Slack, and has a
      Kubernetes steering-committee liaison.
    - jetcd is a SIG etcd subproject on paper. Its own `OWNERS` names none
      of the SIG leads. One of its three approvers made 28 of the 52
      non-bot commits from 2024-08 to `jetcd-0.8.7`, almost all dependency,
      CI, documentation or release chores. In #14890 a maintainer wrote: "I doubt that the
      jetcd is well maintained by existing etcd maintainer either"
      ([2023-01-27](https://github.com/etcd-io/etcd/issues/14890#issuecomment-1405879641)).
  - **Capacity.** On 2022-03-07 a maintainer told the Kubernetes steering
    committee that etcd had "only one active and two occasionally-reviewing
    maintainers" and was "effectively unmaintained"
    ([thread](https://groups.google.com/a/kubernetes.io/g/steering/c/e-O-tVSCJOk);
    read through a summarizing fetch). That led to
    [CNCF TOC #898](https://github.com/cncf/toc/issues/898) and, per the
    announcement, to the SIG. Jepsen's first 2022 reports arrived three
    months after that message.
  - **Process: issues close on inactivity.**
    - A bot marks an issue stale after 90 days without activity and closes
      it 21 days later, unless it is labelled `stage/tracked` or
      `help wanted` (`.github/workflows/stale.yaml`, `origin/main`).
    - From 2025-05-13 to 2026-01-13 the workflow used input names the
      GitHub Action does not recognize, so neither those exemptions nor the
      90- and 21-day periods applied. The Action's defaults did: stale
      after 60 days, closed 7 days later (corrected 2026-11-10; for example
      #20621 was marked stale after 61 days and closed 7 days after that).
      Fix: commit `55c5fabe5`. About 50 `stage/tracked` issues were closed
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
    | **Jepsen's anomaly reports** ([#14092](https://github.com/etcd-io/etcd/issues/14092), jetcd [#1072](https://github.com/etcd-io/jetcd/issues/1072), [#14890](https://github.com/etcd-io/etcd/issues/14890), [#18424](https://github.com/etcd-io/etcd/issues/18424)) | **2022-06-06:** aborted reads reported against etcd 3.5.3. Next day a maintainer: "From brief read this looks like bug in jetcd and not etcd" ([comment](https://github.com/etcd-io/etcd/issues/14092#issuecomment-1148426533)). Closed 2022-09-28 "as a jetcd issue". The jetcd report got one reply the same day, from a jetcd approver who offered to take a patch but gave no diagnosis, and was bot-closed 2022-08-22. **2022-12-04:** lost update and cyclic information flow reported (#14890). One maintainer tried to reproduce it in etcd's own framework, without success; another analyzed the data dumps and concluded: "It turned out to be **not an issue** to me" ([2022-12-07](https://github.com/etcd-io/etcd/issues/14890#issuecomment-1340300176)). **2023-01-26:** Jepsen withdrew after "months of unpaid work" ([comment](https://github.com/etcd-io/etcd/issues/14890#issuecomment-1405084453)). **2023-01-27:** a Kubernetes engineer: "I think the most likely explanation is that Jepsen attempted a write, it believed the write failed and therefore it was free to re-use the value, but etcd actually recorded the write" ([comment](https://github.com/etcd-io/etcd/issues/14890#issuecomment-1406838149)). That is close to the actual cause: the write had executed although the client saw a failure. What re-sent it was jetcd's internal retry, not the test harness. **Then 18 months without activity.** **2024-07-22:** an outside developer traced the cyclic information flow to the client's automatic retry ([comment](https://github.com/etcd-io/etcd/issues/14890#issuecomment-2242269686)), and on 2024-07-24 the lost update too. **2024-08-02:** closed. A maintainer: "I think we have enough evidence to think that problem is related to Java client and not on etcdserver side. … at the current moment I think we can official reject Jepsen report as inaccurate" ([comment](https://github.com/etcd-io/etcd/issues/14890#issuecomment-2265016141)). A second: "Agreed. … The problem might be related to etcd java client or Jepsen. Based on all the discussion so far, I would bet Jepsen" ([comment](https://github.com/etcd-io/etcd/issues/14890#issuecomment-2265039691)). **2024-08-08:** Jepsen's report: "It's jetcd" ([comment](https://github.com/etcd-io/etcd/issues/14890#issuecomment-2276009336)). The same day a maintainer opened #18424; the Go client was found unaffected; the bot closed it 2025-06-01 | The anomalies were real, and their cause was jetcd, as the closing maintainer said (corrected 2026-11-10: this row used to quote only the "inaccurate" sentence). jetcd was fixed by an outside contributor (PR #1445, merged 2025-02-18; released in 0.8.5 on 2025-03-03): **2 years 9 months** after the first report, with a near-correct diagnosis on the thread **18 months** before it was confirmed |
    | **Lease revoked by a deposed leader** ([#12528](https://github.com/etcd-io/etcd/issues/12528) → [#15247](https://github.com/etcd-io/etcd/issues/15247); LS2) | **2020-12-07:** reported on v3.4.6. A fix, PR [#12531](https://github.com/etcd-io/etcd/pull/12531), was opened the next day. The bot closed the issue 2021-06-14 and the PR 2021-09-16, unmerged. **2023-02-06:** the same bug re-reported as #15247, citing #12528. Fix merged 2023-12-15 | Released in v3.4.31 and v3.5.13 (March 2024) and v3.6.0 (May 2025): about **3 years 3 months** from report to fixed release, with a candidate fix from the second day. The fix chosen is a heuristic leadership check; the term fence proposed in #15247 was not adopted. **2026-11-06:** #22714 (open) reports a case that the check does not cover, reproduced on v3.7.2 and v3.6.15, and names the term fence among its fix directions; fix PR #22730 is open (LS2; added 2026-11-10) |
    | **Watch blocked forever despite `WithRequireLeader`** ([#12534](https://github.com/etcd-io/etcd/issues/12534) → [#17611](https://github.com/etcd-io/etcd/issues/17611); WT2, WT6) | **2020-12-09:** reported. A then-listed maintainer: "It is a nice to have feature. Probably etcd should check if the server maintains leadership for long running requests" ([2021-04-19](https://github.com/etcd-io/etcd/issues/12534#issuecomment-822107665)). Bot-closed 2021-08-08. **2024-03-19:** re-filed under the same title, now with a client-side cause; labelled `type/bug` and `priority/important-longterm`; bot-closed *not planned* 2025-06-01. Its fix PR #17622, opened by the reporter, was bot-closed unmerged 2025-11-10 | Never fixed. The 2021 classification answered a misreading: the server already cancelled flagged streams on leader loss (watchers §6.1), and #17611 later located the hang in the Go client's teardown, a race (WT6). Either way, a report of a watch silent forever aged out twice over about four and a half years |
    | **"Make leases linearizable"** ([#13915](https://github.com/etcd-io/etcd/issues/13915)) | Opened 2022-04-09 and labelled `stage/tracked`. Bot-closed 2025-09-11 while the exemption was broken; reopened by a maintainer 2025-09-23; bot-closed again 2025-12-19. The checkpointing stabilization tracker ([#13776](https://github.com/etcd-io/etcd/issues/13776); `stage/tracked`, `help wanted`) was bot-closed 2025-09-30 | Neither closure was a verdict on the design, but for an adopter the outcome is the same: tracked lease redesigns closed and not reopened. Both questions remain open; checkpointing has been experimental or Alpha since v3.4.0 (2019) |
    | **Lost un-fsynced writes** ([#14140](https://github.com/etcd-io/etcd/issues/14140), [#14143](https://github.com/etcd-io/etcd/issues/14143)) | 2022: labelled `priority/important-soon` and `stage/tracked`. A maintainer later reported that a contributor had reproduced #14143 ([2023-07-04](https://github.com/etcd-io/etcd/issues/14143#issuecomment-1620148375)). Little activity after 2023, apart from a 2024 maintainer comment proposing to close #14140; both bot-closed in July 2025 while the exemption was broken | No root cause (crash-consistency bullet below) |
    | **Watch progress guarantee** ([#20242](https://github.com/etcd-io/etcd/issues/20242); WT4) | **2025-06-30:** filed by a maintainer: "Note this doesn't affect Kubernetes as it setups only 1 watcher per stream." The same day, the robustness suite's progress-notification check was disabled "until we can guarantee response" (PR #20240). Bot-closed 2025-11-23 with no human comment | At v3.7.3 the guarantee does not exist, and the robustness suite's check that progress notifications arrive (`ValidateGotAtLeastOneProgressNotify`, removed by #20240) is absent |
    | **`client/v3/leasing` stale read** ([#19091](https://github.com/etcd-io/etcd/issues/19091); LS9) | Reported 2024-12-19. The report got no human reply. Its first fix (PR #19092) got one maintainer review comment, on naming, and was never approved for testing. Both were bot-closed 2025-05-17. A second fix (PR #21589) was closed unmerged by its author (corrected 2026-11-10) | Unfixed, also on `main` |

  - **Also:** the robustness harness does not exercise lease expiry or
    keepalive, and adding leases to its Kubernetes traffic profile (#20621)
    was bot-closed.
  - **Assessment.**
    - **Capacity is the binding constraint**, by the maintainers' own
      account, and the backlog is managed by auto-closure. — **High**.
    - **Defects on Kubernetes' path get fixed; defects off it often do
      not.** Server panics and the watch ordering bugs that matter to the
      apiserver's watch cache were fixed and backported. Lease keep-alive,
      locks, several watchers per stream, the gRPC proxy,
      `client/v3/leasing` and jetcd more often stay open or close without a
      fix. — **Moderate**: drawn from the cases above, not from a
      systematic count. It would fail if a classification of all
      bot-closed bugs showed the same rate on and off Kubernetes' path.
    - **Assessment can take years.** "Not caused by the etcd server" has
      been treated as grounds to close even when the cause was a client
      hosted in etcd-io, and one closure called a report inaccurate whose
      anomalies were real, while rightly attributing them to that client.
      — **High** for the record; **Moderate** that this generalizes.
    - **Not evidence of indifference by individuals.** Most replies came
      within days and were technically engaged. One maintainer tried to
      reproduce the Jepsen results in etcd's own framework and another
      analyzed the data dumps, and the tooling (Clojure, Jepsen's
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
       them (LS1–LS11, WT1–WT9).
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

  Added 2026-11-09, from the GitHub API: Jepsen filed four storage-fault
  issues in June 2022, while testing etcd 3.5.3 in a private engagement
  whose report was never published (jetcd report, footnote 1). Three used
  LazyFS (#14098, #14140, #14143) and one used random bit flips (#14102).
  - #14098: the WAL read back as `max entry size limit exceeded`. Closed
    after Jepsen traced it to LazyFS filling truncated bytes with `0x30`
    instead of zeros.
  - #14102, "Handle disk corruption": closed by the stale bot.
  - #14140 (stale read with loss of un-fsynced writes) and #14143: closed
    *not planned* in July 2025, after stale-bot warnings. A maintainer
    wrote in 2024 that #14140 had not been reproduced in two years, though
    a maintainer had written in 2022 that a contributor found it
    "reproducible on v3.4"; for
    #14143 a maintainer reported in 2023 that a contributor had reproduced
    it, but no root cause was found.
  - Two read-only-transaction panics Jepsen filed in 2022 were fixed
    (#14110, #14891).

  The jetcd report, in 2024, still says etcd "experiences a broad array of
  crashes when nodes encounter filesystem corruption or the loss of
  un-fsynced writes". — **High** for the states; Jepsen's statement is
  untested here.
- **Independent testing: Jepsen** (added 2026-11-09). The issues Jepsen
  filed against etcd and jetcd since 2019 that bear on correctness, the two
  issues that followed its 2024 report (etcd #18424, jetcd #1444), and
  where they stand. Omitted: jetcd #673 (a duplicate of #674), jetcd #677
  (a channel-shutdown warning), and the two panics in the paragraph above.

  | Report | Finding | Upstream | Status at v3.7.3 / jetcd 0.8.7 |
  |---|---|---|---|
  | etcd 3.4.3 (2020) | KV reads, writes and transactions strict serializable under partitions, pauses, crashes, clock skew and membership changes | — | No published Jepsen test of the server since; the 2024 report attributes its anomalies to jetcd. etcd's robustness suite now validates the KV API guarantees under injected faults (`tests/robustness/README.md:3-4 @ v3.7.3`) |
  | etcd 3.4.3 | Single-key watches deliver every update in order | — | Consistent with this project's findings for safety; the tests could not see silent watches (§3) |
  | etcd 3.4.3 | Docs claimed "sequential consistency, the strongest"; defined linearizability by wall clock | (docs) | Fixed: the v3.7 API guarantees page says strict serializability, links Jepsen's definition, and quotes Herlihy and Wing for linearizability (`learning/api_guarantees.md:41`, `:68-74`, `:155`) |
  | etcd 3.4.3 | Locks do not give mutual exclusion; unfenced use lost updates | #11457 (docs) | Docs note added 2020 (PR #11490); unfenced `etcdctl` example still in the README (LS10) |
  | etcd 3.4.3 | A lock waiter whose lease expired was told it held the lock | #11456 | Fixed in v3.5.0 by a re-check after waiting (`client/v3/concurrency/mutex.go:96-105`); not in any 3.4 release |
  | etcd 3.4.3 | Revision 0 means "now", not "from the start"; undocumented | #11496 | Documented (PR #11559, 2020): first revision is 1; Go `Watch` comment (`client/v3/watch.go:68-69`) |
  | etcd 3.4.3 (2019 work) | jetcd 0.4.1 watch creation threw "Stream is already completed" | jetcd #674 | Closed by the stale bot. jetcd's watch client has since moved to Vert.x streams (`jetcd-core/src/main/java/io/etcd/jetcd/impl/WatchImpl.java:44-45 @ jetcd-0.8.7`); not re-tested |
  | 2022 reports (#14890 was filed after the engagement ended) | Aborted read; lost update and cyclic information flow on crash | etcd #14092, #14890, jetcd #1072 | All closed without a fix: #14092 "as a jetcd issue" (2022), jetcd #1072 by the stale bot, #14890 on 2024-08-02. Its closing maintainer attributed the anomalies to the Java client and called Jepsen's report inaccurate; a second bet that the cause was Jepsen. Six days later the jetcd report showed it was jetcd's retries |
  | jetcd 0.8.2 (2024) | jetcd retries non-idempotent writes | etcd #18424, jetcd #1444 | Fixed in jetcd 0.8.5; residual lease and lock retries above |
  | 2022 engagement | Crashes and anomalies with lost un-fsynced writes or corruption | #14098, #14102, #14140, #14143 | Closed without fixes (above) |

  — **High** (issue states and code read 2026-11-09).

## 6. Common misconceptions

| Claim | Correction | Conf. |
|---|---|---|
| "Watches bypass Raft, so they lose events on failover." | Half right (corrected 2026-11-03). Events that arrive are never missing, reordered or retracted: they come from committed entries. But the watch path skips Raft's quorum check, so a watch on a member that is cut off from the quorum (a deposed leader included) or has stalled can stay silent indefinitely. That breaks the eventual consistency etcd claims for watches, and failover is only one trigger (WT1–WT4, §3). | High |
| "Lease renewals bypass Raft, and that is the bug." | Chubby does the same. The flaw is that expiry revocations carry no Raft term, and that a leader's authority is measured in Raft ticks while expiry runs on real time. | Moderate |
| "A partitioned leader breaks the renewal guarantee." | Half right (corrected 2026-11-10). Renewals that the cut-off leader acknowledged are covered: a new leader extends every lease by TTL + ET, to within about one tick. Its revocations are not: expiry revocations it had queued can be applied by the new leader after it steps down, deleting leases that clients are renewing (#22714, reported 2026-11-06 on v3.7.2; LS2). | Moderate |
| "Kubernetes relies on etcd leases for leader election and node heartbeats." | It uses Lease API objects: ordinary keys updated by compare-and-swap. | High |
| "Watches are eventually consistent." (etcd's robustness README) | The documented contract guarantees what arrives, not that it arrives, and some silent paths persist after every fault has healed (§3). | High |
| "`WithRequireLeader` makes watches safe." | It is partial (WT2), and ignored through the proxy (WT5). A watch that sets it can still stay silent indefinitely with the quorum intact; only a freshness monitor (§3 mitigation 9) detects every such case. | High |
| "Replacing bbolt or the WAL format will fix write throughput." | Only if backend commits dominate (f_commit). The WAL already group-commits. | High (mechanism) |
| "Jepsen tested etcd, so its locks are safe." | Jepsen tested them and found the opposite: mutual exclusion failed within minutes with short TTLs, even in a healthy cluster. What it found safe was KV operations, and lock-guarded transactions (LS10). | High |
| "Jepsen found lost updates and aborted reads in etcd." | It found them in applications using jetcd 0.6.0–0.8.2 (by code, 0.5.8–0.8.4), and traced them to the client retrying writes; the server behaved correctly. Fixed in jetcd 0.8.5 (§5). | High |
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
- jetcd: is it at 0.8.5 or later, and at 0.8.8 or later if it keeps leases
  alive (LS11)? On 0.8.5 or later, does any code call `withAutoRetry()`?
  On an older version, is `retryMaxAttempts(0)` set (§5)? The default is
  2, and the option exists only from 0.7.6.
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
- Track the fixes that would retire items above: #22730 (LS2); #22650
  (LS3); #22087, #22253 and #22291 (LS5); #22115 and #22387 (LS8); #21065
  (WT8); PR #22223 and #22332 (WR2, WR3).
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
  not reproduced by this project. One lease window has an upstream
  reproduction (#22714), which this project did not re-run.
- **The maintenance assessment (§5) is drawn from cases**, read in issue
  threads and governance documents, not from a systematic count of issue
  outcomes on and off Kubernetes' path. Such a count is the test that would
  confirm or refute it.
- **Jepsen's results are external and older.** They cover etcd 3.4.3 and
  jetcd 0.4.1 (2019–2020), and jetcd 0.5.0, 0.6.0 and 0.8.2 (2024; the report does
  not name the server version, and its harness at `fbe693e` defaults to
  etcd 3.5.15).
  This project re-ran none of them; their fixes were checked in code only.
- **Not covered:** membership changes, upgrades and downgrades, auth beyond
  revocation cost, TLS beyond #15402, multi-region deployment, and backup
  tooling.
- **What would change these conclusions:**
  - a maintainer's confirmation of #22714, or a re-run of it here, would
    raise LS2 to High throughout; a failed re-run would lower it again;
  - benchmarks on the target hardware would replace the WR magnitudes;
  - the fixes listed in §7 landing would retire their rows.

## Appendix A. Where the details are

| Topic | File and section |
|---|---|
| Lease API promise, failover mechanics, properties | [`leases-claude.md`](leases-claude.md) §1–§2 |
| Failover bugs #12528/#15247, residual windows, #22649 | `leases-claude.md` §3 |
| Design flaw relative to Chubby (term-less revocation, tick-based authority) | `leases-claude.md` §4, §11.1 |
| Go and jetcd lease clients (LS4, LS5, LS11) | `leases-claude.md` §5 |
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
