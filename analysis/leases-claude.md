# Leases: Raft, leader failover, and correctness (Claude)

**Versions analyzed:**

| Component | Version | Notes |
|---|---|---|
| etcd | `v3.7.3` = `0a173d0410a9` | All `path:line` citations are at this tag unless stated otherwise. |
| jetcd | `jetcd-0.8.7` = `23b61f1499a5` | `LeaseImpl.java` read in full (§5.3). For §5.4 (2026-11-09): `support/Errors.java`, `impl/{Impl,KVImpl,LockImpl,ElectionImpl}.java`, `ClientBuilder.java`, and commit `05f92e0`. `jetcd-0.8.8` was released on 2026-11-09 and is not analyzed. |
| Raft library | `go.etcd.io/raft/v3` `v3.7.0`, the version pinned in `server/go.mod:30` | Only `raft.go`, fetched from the `etcd-io/raft` tag. Used for proposal forwarding (§3.2) and for tick and CheckQuorum timing (§3.4, R1). |
| Kubernetes | `master` at `3c690c189e5d` (fetched 2026-10-31); two files at tag `v1.37.2` | The `master` files are "current Kubernetes", not a release (§7). |

Upstream state was re-checked on 2026-11-01:
- `git fetch --tags` showed no newer stable release of etcd or jetcd.
- etcd `origin/main` (`1573d5941`) has no change to lease semantics after
  `v3.7.3`. Its only commits to `server/lease` are a protobuf
  regeneration, a semver-library swap and a peer-HTTP body-size limit.
- `origin/release-3.7` has no lease commits after `v3.7.3`.

**Method:**
- Code reading of the checkouts.
- The GitHub REST API, for issue and PR bodies and comment threads in
  `etcd-io/etcd`, `etcd-io/jetcd` and `kubernetes/kubernetes`.
- Web sources (Jepsen, etcd.io docs).
- **Jepsen pass (2026-11-09):** the reports
  [etcd 3.4.3](https://jepsen.io/analyses/etcd-3.4.3) (2020-01-30) and
  [jetcd 0.8.2](https://jepsen.io/analyses/jetcd-0.8.2) (2024-08-08), read
  in full; Jepsen's lock and client source at `jepsen-io/etcd` @ `16149cb`
  (2019) and its 2024 harness defaults at `fbe693e`; every issue the
  reports cite, through the GitHub API; the fixes, in code at `v3.7.3` and
  `jetcd-0.8.7`; and the v3.7 lock docs (`etcd-io/website` @
  `41e5cd6ab9ef`). Results: §2.3, §5.4, §8.

**Nothing was run or reproduced.** "Inferred" and "traced" mean derived from
code reading only. Confidence tags (High / Moderate / Low) apply to the claim
they follow, not to the whole document.

**Upstream references:** the most important are:
- etcd [#15247](https://github.com/etcd-io/etcd/issues/15247), [#12528](https://github.com/etcd-io/etcd/issues/12528), [#22649](https://github.com/etcd-io/etcd/issues/22649), [#18428](https://github.com/etcd-io/etcd/pull/18428), [#13915](https://github.com/etcd-io/etcd/issues/13915) and [#9888](https://github.com/etcd-io/etcd/issues/9888);
- Kubernetes [#65497](https://github.com/kubernetes/kubernetes/issues/65497) and [#110210](https://github.com/kubernetes/kubernetes/issues/110210);
- from the Jepsen reports: etcd [#11456](https://github.com/etcd-io/etcd/issues/11456), [#11457](https://github.com/etcd-io/etcd/issues/11457), [#14890](https://github.com/etcd-io/etcd/issues/14890), [#18424](https://github.com/etcd-io/etcd/issues/18424); jetcd [#1444](https://github.com/etcd-io/jetcd/issues/1444) / PR [#1445](https://github.com/etcd-io/jetcd/pull/1445).

Appendix B has the full list with states and dates. Appendix A is the issue
survey.

**Provenance:**
- First written 2026-10-31.
- §5.2 (Go client and `WithRequireLeader`) was added the same day by the
  session that wrote `analysis/watchers-claude.md`.
- Revised 2026-11-01, using two independent analyses by other tools as a
  source of leads. Every adopted item was re-verified, so this file does not
  depend on them.
- Restructured for readability the same day.
- Extended 2026-11-09 with the Jepsen reports and the state of their fixes
  (§2.3, §5.4, §8), at the user's request.

Appendix C has the revision history and a map from the old section numbers
to the new ones.

## Scope

This document asks one question. **Do etcd leases keep their promise to
clients across leader failover while a quorum survives?** It also tests the
hypothesis that etcd's lease design has a basic mistake compared with
Chubby's sessions.

It covers:
- the etcd v3.7.3 server;
- the Go client and jetcd 0.8.7;
- how Kubernetes uses leases.

Performance is covered only as code structure; nothing was measured
(AGENTS.md §7). It does not cover the lease-backed client cache
(`client/v3/leasing`), or watch semantics beyond what revocation affects.

**Terms used throughout**
- **Lease:** etcd's KV lease (`LeaseGrant`, `KeepAlive`, `LeaseRevoke`). Keys
  attached to a lease are deleted when it is revoked. Two things share the
  name but are different:
  - Raft's "leader lease" (CheckQuorum);
  - Kubernetes `Lease` objects (`coordination.k8s.io`), which are ordinary
    etcd keys.
- **Lessor, primary lessor:** the lessor is etcd's lease manager. The primary
  lessor is the one on the member that believes it is Raft leader. Only the
  primary tracks expiry times and revokes.
- **Promote / Demote:** the lessor becoming / ceasing to be primary. Promote
  extends every lease (§1.3).
- **Checkpoint:** an optional Raft entry that records a lease's remaining TTL
  (§1.4).
- **ET:** the Raft election timeout, 1 s by default. **TickMs:** the Raft tick,
  100 ms by default.
- **P1–P5:** the correctness properties defined in §2.1. The one that locks
  and elections depend on is **P3**: a lease is not revoked within TTL of a
  renewal that the server acknowledged.
- **Case 1, Case 2:** the two failure cases of the #15247 bug (§3.1).
- **R1–R5:** residual failure windows at v3.7.3 (§3.4).
- **Fencing:** making a protected write conditional on the lock key's
  revision, so that a deposed holder's write fails (§2.3).

## Summary

**Question.** Do etcd leases break their API promise across leader failover
while a quorum holds? If so, is the cause a design mistake relative to
Chubby?

**Short answer.**
- **The promise is a lower bound, and it was broken.** No lease may be
  revoked within TTL of an acknowledged renewal (P3). Reproduced bugs broke
  this on failover in every release before v3.4.31, v3.5.13 and v3.6.0
  (#12528, #15247). — **High**.
- **At v3.7.3 it holds, with caveats.** With the default configuration and a
  client renewing on schedule, no violation has been reproduced. But the fix
  is a heuristic leadership check, not a fence, and narrow windows remain
  (R1–R5). With the Alpha checkpoint feature on, the open bug #22649 breaks
  the promise outright. — **Moderate** for the residual windows.
- **The replicated key-value state is never at risk (P1).** Every
  lease-driven deletion is a Raft entry, applied identically on every member.
  — **High**.
- **There is no upper bound (P2).** Every leader change or restart resets
  each lease to its full TTL plus ET, so an unrenewed lease can live
  indefinitely under repeated leader changes. The mitigation, checkpointing,
  is Alpha and off by default. It also does not help short leases or
  frequent leader changes. — **High**.
- **The hypothesis is right in substance but mislocated.** Keeping renewals
  out of the consensus log is Chubby's design too. The mistake is elsewhere:
  - expiry-driven revocations are decided from one member's local timer, but
    committed as commands that carry no term;
  - that member's authority is measured in Raft ticks, while lease expiry
    runs on real time.

  Chubby avoids both, with master leases and epoch numbers. — **Moderate**.

**Key findings at v3.7.3**

| # | Finding | Status | Confidence | § |
|---|---|---|---|---|
| 1 | A stalled ex-leader revokes live leases, and acknowledges renewals that the new leader never sees (#12528, #15247) | Fixed for the reported scenarios (v3.4.31, v3.5.13, v3.6.0) by a leadership check before proposing | High | 3.1–3.3 |
| 2 | Residual windows R1–R5 around that check | Open; none reproduced | Moderate existence; Low impact | 3.4 |
| 3 | A leader cut off by a network partition skips the ReadIndex check (#18428) | Not a P3 violation: the ET extension added by Promote covers it, to within about one tick | Moderate | 3.4 (R1) |
| 4 | #22649: a renewal reports the full TTL while the server keeps a shorter one | Open; only with Alpha `LeaseCheckpoint` on; fix PR #22650 open | High (code) | 3.5 |
| 5 | TTL reset on every leader change and restart (P2) | By design; checkpointing is Alpha, off, and limited | High | 1.3, 1.4, 2.2 |
| 6 | Go client #22081, #22094: the application is told its keep-alive ended while the lease is renewed forever | Open; affects only applications that pass `WithRequireLeader` contexts to `KeepAlive` | High (code); Moderate (consequence) | 5.2 |
| 7 | jetcd reports every stream error through `onError` but keeps renewing | By design since jetcd 0.8.0; same end state as #22094 for applications that treat `onError` as terminal | High (code); Moderate (consequence) | 5.3 |
| 8 | Client lease deadlines are optimistic (receipt time + TTL, checked once a second) | By design in both clients; a window of about 1 s, unbounded with process pauses | High (code); Moderate (size) | 5.1 |
| 9 | Keepalives forwarded by followers still block on the leader's apply backlog. Expiry costs one Raft proposal per lease, and one root token per revocation when auth is on | Code structure; not measured | High (code); Low–Moderate (magnitude) | 6 |
| 10 | Kubernetes is insulated from the P3 bugs (it never calls `KeepAlive`). It is exposed to P2 (Events), to revocation cost, and to the open watch bug #110210 | — | High (code); Moderate ("insulated") | 7 |
| 11 | Process: the core failover bug was closed once by a stale bot; both lease redesign trackers were closed by the project's stale bot while their `stage/tracked` label should have exempted them, and neither has been reopened since; the correctness harness does not exercise expiry or keepalive | — | High | 3.7, 8 |
| 12 | Automatic retries (Jepsen, jetcd 0.8.2). jetcd ≤ 0.8.3 retried writes after indefinite failures, which Jepsen showed as lost updates and aborted reads. Both clients still retry lease grant and revoke, and jetcd retries `Lock`, whose in-flight first attempt can delete the key the retry reused | jetcd write retries fixed in 0.8.5 (#1444). Lease and lock retries: by design; the lock race is not reported upstream | High (code); Low (lock race likelihood; inferred) | 5.4 |

**What protects applications is fencing.** No lease design gives mutual
exclusion by itself. With fencing through etcd transactions, writes to etcd
stay correct even under finding 1 (§2.3). Jepsen measured the difference on
etcd 3.4.3: unfenced locks lost about 18 % of acknowledged updates, and
lock-guarded transactions lost none in its tests (§2.3).

---

## 1. How leases work at v3.7.3

### 1.1 The API promise

| Source | Text |
|---|---|
| `Grant` (`server/lease/lessor.go:92`) | "grants a lease that expires at least after TTL seconds" |
| Keepalive response (`api/etcdserverpb/rpc.proto:936`) | the TTL "is the new time-to-live for the lease" |
| Lease TTL field (`rpc.proto:872`) | the TTL is "advisory" |
| Failure guide ([op-guide/failures, v3.6](https://etcd.io/docs/v3.6/op-guide/failures/), fetched 2026-10-31) | "The new leader extends timeouts automatically for all leases. This mechanism ensures a lease will not expire before the granted TTL even if it was granted by the old leader" |

So the promise is a **lower bound**, and no upper bound is promised. The
documented safety mechanism is the extension a new leader applies to every
lease (§1.3). It protects the lower bound (P3) at the cost of the upper
bound (P2).

### 1.2 What does and does not go through Raft

| Operation / state | Path | Through Raft? | Evidence |
|---|---|---|---|
| `LeaseGrant` | `raftRequest(LeaseGrant)` → apply → `lessor.Grant` | Yes | `server/etcdserver/v3_server.go:435-457`, `server/etcdserver/apply/backend.go:83` |
| `LeaseRevoke` (client) | `raftRequest(LeaseRevoke)` → apply → `lessor.Revoke` (deletes attached keys in one txn) | Yes | `v3_server.go:471-487`, `apply/backend.go:94-96`, `server/lease/lessor.go:326-365` |
| Revocation on expiry | Primary lessor's timer finds expired leases → `EtcdServer.revokeExpiredLeases` → `LeaseRevoke` proposal | **Decision outside Raft; effect through Raft** | `lessor.go:620-662`, `server/etcdserver/server.go:838-905` |
| Attaching keys (`Put` with lease) | Normal KV write | Yes | — |
| `KeepAlive` / renew | Leader: `lessor.Renew` in memory. Follower: forwards over peer HTTP to the leader, whose handler calls `lessor.Renew`. Exception: with `LeaseCheckpoint` on, renewing a checkpointed lease first proposes a "clear" checkpoint through Raft (§1.4) | **No** (except the checkpoint clear) | `v3_server.go:489-560` (comment at :541: "renewals don't go through raft"), `server/lease/leasehttp/http.go:91`, `lessor.go:436-440` |
| Expiry time of each lease | Leader memory only; followers hold every lease at expiry "forever" | **No** | `lessor.go:534-550` (`Demote`), `lessor.go:305-309` (`Grant` on non-primary) |
| Remaining TTL | Leader memory; replicated only via optional checkpoints | **Only if `LeaseCheckpoint` gate on** (Alpha, default off) | `lessor.go:367-383`, `server/etcdserver/server.go:392-403`, `server/features/etcd_features.go:94-95` |
| `LeaseTimeToLive` | Leader memory after a local apply-index wait, with no ReadIndex. Fails with `ErrLeaderChanged` if the lessor was demoted during the lookup, so a stalled ex-leader that has not yet noticed its demotion answers from its own view. Followers forward over peer HTTP | No | `v3_server.go:637-697`, demotion check :664 |
| `LeaseLeases` (list) | Local member's lessor, no ReadIndex | No (serializable) | `v3_server.go:737-749` |

Two consequences follow.

- **The state machine is safe.** Keys disappear because of a lease only when
  a `LeaseRevoke` entry is applied.
  - Apply is deterministic: keys are sorted before deletion
    (`lessor.go:345-351`).
  - Lease deletion is in the same backend transaction as key deletion
    (`lessor.go:353-361`).

  — **High**. This would fail only through an apply-path bug, such as the
  v3.2→v3.3 upgrade revocation corruption
  ([#11689](https://github.com/etcd-io/etcd/issues/11689); title only, not
  examined).
- **The time-based part of the contract lives in one process's memory.**
  Which leases get revoked, and when, is decided by whichever member's lessor
  believes it is primary, on that member's clock. The lease contract is
  therefore only as correct as the sync between "I am the primary lessor"
  and "I am the Raft leader". The failover bugs are in that sync.

### 1.3 Failover mechanics

1. **Demote.** When the Raft-ready handler sees that this member is no longer
   leader, it calls `lessor.Demote()` (`server.go:769-773`). That sets every
   lease's expiry to "forever" and clears the expiry and checkpoint queues
   (`lessor.go:534-550`). A demoted member never revokes.
2. **Promote timing.** A new leader promotes only when it applies the empty
   entry of its own term, i.e. after applying all entries from earlier terms
   (`server.go:1959-1967`; comment: "promote lessor when the local member is
   leader and finished applying all entries from the last term"). So any
   revocation committed in the previous term is applied before the new
   leader starts its timers. — **High**.
3. **Promote extends every lease.** `Promote(s.Cfg.ElectionTimeout())`
   (`server.go:1965`) calls `refresh(extend)` on every lease
   (`lessor.go:480-492`). `refresh` sets `expiry = now + extend +
   remainingTTL-or-TTL` (`server/lease/lease.go:76-89`). Without a checkpoint,
   the remaining TTL is the **full TTL**. With 1000 or more leases, Promote
   also spreads expiries to avoid a revocation pile-up (`lessor.go:494-531`).
4. **Restart resets too.** On restart, leases are recovered with expiry
   "forever" and refreshed on promotion (`lessor.go:808-835`; `expiry:
   forever` at :826).
5. **Minimum TTL versus election timeout.** Granted TTLs are raised to
   `ceil(1.5 × ET)` (`server.go:340`, applied at `lessor.go:301-303`). With
   CheckQuorum on (`server/etcdserver/bootstrap.go:548`), a healthy but
   partitioned leader steps down within about 2 × ET, typically before any
   lease could expire on its clock. That is why a plain network partition is
   mostly harmless, and why *stalls* are not (§3).

### 1.4 Checkpointing and its limits

**What it does.** Both gates are Alpha and off by default
(`etcd_features.go:94-95`).

| Aspect | Behaviour | Evidence |
|---|---|---|
| Cadence | The leader proposes `LeaseCheckpoint{ID, remaining}` every 5 minutes | `lessor.go:52`, `:664-686`, `:772-806` |
| Coverage | Only for leases whose remaining TTL exceeds the interval | `lessor.go:758` |
| Persistence to bbolt | When the `LeaseCheckpointPersist` gate is on, **or** the cluster version is ≥ 3.6 | `lessor.go:385-388` |
| Renew | If a lease has a checkpointed remaining TTL, `Renew` first proposes a checkpoint of 0 through Raft, then refreshes. That puts a synchronous Raft round trip in the keepalive path, which matters for #22649 (§3.5) | `lessor.go:410-449` |

**Limits.** The code is verified; the consequences are inferred.
- **What is recorded is a duration, not a deadline.** `Checkpoint` stores
  only `remainingTTL` (comment at `lessor.go:372`: "Promote is responsible
  for applying this to lease expiry"). Each `Promote` re-applies it from
  "now" plus ET. Time between the last checkpoint and the leader change is
  lost: up to one full interval per change.
- **Churn defeats it.** `Promote` schedules each lease's first checkpoint one
  interval after promotion (`lessor.go:491`, `:753-770`), and `Demote`
  discards the schedule (`lessor.go:543`). If leader changes come faster than
  the interval, no new checkpoint is ever taken, and every new leader
  re-applies the same old remaining TTL. Retention is then as unbounded as
  with checkpointing off.
- **Short leases are never covered.** `concurrency.Session`'s default TTL is
  60 s (`client/v3/concurrency/session.go:26`). So default sessions, locks
  and elections never get a checkpoint.
- **The interval is not an operator setting.** It is a server config field
  (`server/config/config.go:164`), passed to the lessor at `server.go:346`,
  with a 5-minute default when unset (`lessor.go:217`). I found no
  command-line flag or `embed.Config` field that sets it at v3.7.3; only
  tests set it.
- **The reset is tested-for behaviour.** `TestV3LeaseCheckpoint` has a case
  named "Checkpointing disabled, lease TTL is reset"
  (`tests/integration/v3_lease_test.go:506`, `:518`).

## 2. Correctness properties

### 2.1 Five properties

"Lease correctness" is several properties that fail differently.

| # | Property | Status at v3.7.3 | Confidence |
|---|---|---|---|
| P1 | Replicas agree on which keys exist; KV reads stay linearizable despite lease activity | Holds (§1.2) | High |
| P2 | **Upper bound:** a lease that is not renewed is revoked about TTL after its last renewal | **Violated by design** on every leader change and restart (reset to TTL + ET). Unbounded under repeated leader changes. The mitigation is off by default, and even when on it does not cover TTL ≤ 5 min or leader changes more frequent than the checkpoint interval (§1.4) | High |
| P3 | **Lower bound:** a lease is not revoked within TTL of a keepalive the server acknowledged (explicit client `LeaseRevoke` excluded) | Violated on failover before the fixes (#12528, #15247). Fixed for the reported scenarios in v3.4.31, v3.5.13 and v3.6.0. Narrow residual windows remain (§3.4: R1, R2, R5). With the Alpha checkpoint gate on, #22649 violates it outright (§3.5) | High (history); Moderate (residual) |
| P4 | Lease reads (`TimeToLive`, `Leases`, the TTL in keepalive responses) are linearizable | Not provided ([#13915](https://github.com/etcd-io/etcd/issues/13915), closed *not planned* by the stale bot while its tracked-issue exemption was ignored; §3.7). `FastLeaseKeepAlive` adds a deliberate exception (§2.4). Response-header revisions on lease RPCs are not snapshot points, and on "not found" answers can precede the revocation (§2.5) | High |
| P5 | Mutual exclusion for locks built on leases | Not provided by leases in any design (etcd docs, Jepsen, Chubby). Requires fencing with the lock key's revision (§2.3) | High |

P1 holds, but it is **not** the property that leader election and locks
depend on. They depend on P3, and P3 is the one that was broken.

### 2.2 P2 and P3: one knob, two uses

- **One knob.** P2 and P3 trade against each other through one knob: the
  extension applied at promotion. Extending more protects P3; extending less
  protects P2.
- **Why the knob exists.** The replicated state records each lease's grant
  (ID, TTL) but not its last renewal. A new leader cannot tell a lease
  renewed a millisecond ago from one abandoned an hour ago, so it must treat
  them alike.
- **Checkpointing is the only exception.** It is the only mechanism that
  improves P2 without hurting P3. It does so only for leases longer than
  5 minutes, and only while leader changes are less frequent than the
  checkpoint interval.
- **The P2 cost in practice.** After a client dies, takeover of its role can
  take TTL + ET *per etcd leader change*, which is unbounded under repeated
  changes. This is documented liveness behaviour, not a P3 violation.

The two uses of etcd leases want opposite settings of the knob:
- **Coordination sessions** (locks, elections) need P3 and can tolerate P2
  slack.
- **TTL garbage collection** (Kubernetes Events, §7) needs P2, and has no
  renewing client for P3 to protect.

One promotion policy serves both, so each use pays for the other. Chubby
avoids the conflict: its sessions serve only live clients, and the paper
describes no per-object TTL mechanism. Its closest analogue, ephemeral
files, has the same fail-over delay: "ephemeral files may not disappear
promptly if the last client on such a file loses its session during a
fail-over" (Chubby §2.9 step 9, p. 9). — **Moderate** (design inference).

### 2.3 Mutual exclusion requires fencing

**No lease, bug-free or not, guarantees mutual exclusion.** The holder's
belief is computed locally and can be stale (pauses, delays). etcd's docs,
Jepsen (§8) and Chubby (§2.4: sequencers, lock-delay) all say so.

**etcd provides fencing.**
- `Election.Proclaim` and `Mutex.IsOwner` compare the role key's create
  revision (`client/v3/concurrency/election.go:115`, `mutex.go:152-153`).
- `Proclaim` applies the compare itself.
- For any other write, the application must put the compare (`IsOwner()`, or
  `CreateRevision(key) == rev`) into the transaction.
- A lease ID alone is not ordered, and is not a fencing token.

**With fencing, writes to etcd stay correct even under #15247 Case 1.** The
deleted key makes the deposed holder's guarded transactions fail. — **High**
(verified code; follows from linearizable `Txn` compares).

**What #15247 breaks is everything not fenced through etcd:** external side
effects, and any etcd write that omits the compare.
- A correctly behaving holder keeps believing it holds the role until its
  next keepalive reports the lease gone. With the Go client, that is up to
  TTL/3 later (`client/v3/lease.go:542`).
- Meanwhile, a successor has already been elected.
- The result is dual leadership caused by the server, not by a slow client.
  — **Moderate** (derived from code, not measured).

**Jepsen's measurements** (etcd 3.4.3, 2020; added 2026-11-09). Jepsen's
lock workloads took a lease per acquisition, kept it alive with jetcd
0.4.1's keep-alive, and called the server Lock API (`jepsen-io/etcd` @
`16149cb`, `src/jepsen/etcd/lock.clj`; jetcd version from its
`project.clj`).
- **Healthy clusters are enough.** With TTLs of 1, 2 and 3 s, locks
  "generally failed to provide mutual exclusion after only a few minutes of
  testing, even in healthy clusters"; pauses and partitions made it faster.
  The report does not say why the leases expired without faults; the
  #11456 thread says only that keep-alive failed to maintain the lease.
  — **High** (Jepsen's observation, on 3.4.3).
- **Etcd raises short TTLs.** The server grants at least
  `ceil(1.5 × election ticks × heartbeat)`, 2 s at the defaults
  (`server/etcdserver/server.go:340`, `:345`;
  `server/lease/lessor.go:301-302`; the same formula at `v3.4.3`). Jepsen's
  harness passed no timing flags to etcd (`src/jepsen/etcd/db.clj:74-84`),
  so its 1 s TTL ran as 2 s (inferred from the defaults; Jepsen does not
  say).
- **Unfenced use loses data.** Five processes updated an in-memory set
  under the lock, reading it, then writing it back about a second later.
  With 2 s TTLs and a process pause every 5 s, about 18 % of acknowledged
  updates were lost.
- **Fenced use did not.** A variant wrote a set stored in etcd with one
  transaction guarded on the holder's lock key (`Version(key) > 0`). That
  "safely isolated read-modify-write operations" in Jepsen's tests. The
  guard is equivalent to `IsOwner()` once the lock has been granted: the
  key is `name/leaseID`, unique to the lease, and every later waiter waits
  for its deletion (`client/v3/concurrency/mutex.go:115-123`). Jepsen adds
  that fencing gives isolation, not atomicity: a holder that loses the lock
  part-way through a multi-step update leaves the steps already done.

**#11456: a waiter told it held the lock after losing its lease.**
- **The bug.** `Mutex.Lock` returned success after `waitDeletes` without
  checking that its own key still existed, so a waiter whose lease expired
  while it waited was told it held the lock (Jepsen, 2019).
- **The fix.** Commit `b14c7cd44` ("concurrency: make lock more reliable")
  adds a `Get` of the waiter's own key after waiting, and returns
  `ErrSessionExpired` if the key is gone (`mutex.go:96-105`).
  - First release v3.5.0 (2021-06-15). It is in no 3.4 release: absent at
    `v3.4.4`, `v3.4.10`, `v3.4.42` and `origin/release-3.4`.
  - jetcd's `Lock` is served by the same code: the server's Lock handler
    calls `concurrency.NewMutex(...).Lock`
    (`server/etcdserver/api/v3lock/lock.go`).
- **Its limit.** The check is a point-in-time read. As the maintainer who
  wrote the fix said (quoted in the issue), it "wont completely solve the
  problem": the
  lease can still expire before the reply arrives, so fencing stays
  necessary. The waiter is still not woken when its own key is deleted
  (#19597; TODO at `mutex.go:88`). — **High** (code and git).

**Documentation** ([#11457](https://github.com/etcd-io/etcd/issues/11457);
v3.7 docs at `etcd-io/website` @ `41e5cd6ab9ef`).
- **What changed.** PR #11490 (2020-03-05) added "Notes on the usage of
  lock and lease" to the "why etcd" page (`learning/why.md:82-98`). The
  notes say "the lease mechanism itself doesn't guarantee mutual exclusion"
  and that resources outside etcd must validate the revision themselves.
- **What did not change**, at v3.7.3:
  - The same page lists "APIs for safe distributed shared locks" (`:46`).
  - The Lock RPC reference describes no guarantee for the lock itself
    (`dev-guide/api_concurrency_reference_v3.md:13`).
  - `etcdctl/README.md:1175-1179` still shows
    `etcdctl lock mylock ./etcdctl put foo bar`, the unfenced pattern
    Jepsen cited. `etcdctl lock` exports the fencing token to its command
    as `ETCD_LOCK_REV` (`:1157`), but no example uses it.

— **High** (text read).

### 2.4 `FastLeaseKeepAlive` and P4

`FastLeaseKeepAlive` is Beta and on by default in v3.7
(`etcd_features.go:97`). On the leader, a renewal skips the apply-index wait
whenever the lease is still in the local map (`v3_server.go:512-523`). The PR
states the consequence and accepts it
([#20589](https://github.com/etcd-io/etcd/pull/20589), merged 2025-10-31):

> When a lease is found but in revoking state, the revoke request has been
> committed but not yet applied. Allowing the current renewal to proceed
> doesn't compromise correctness since the lease will still be properly
> revoked

- **For P3, harmless.** The revocation was either explicit (excluded from
  P3), or driven by expiry, and `Renew` already refuses expired leases
  through its wait-for-revocation branch (`lessor.go:415-430`).
- **For P4, one real-time-order case** (inferred, not reproduced).
  1. Client A's `LeaseRevoke` is acknowledged by the member that proposed it,
     as soon as *that* member applies it.
  2. Suppose the leader's apply lags behind that member's.
  3. A later renewal at the leader still succeeds, with a positive TTL, for a
     lease whose revocation A has already been told is complete.

  Without the gate, `waitAppliedIndex` (`v3_server.go:459`) makes the leader
  apply up to its commit index first, which closes this case. —
  **Moderate**; impact **Low**.

### 2.5 Response-header revisions on lease RPCs

**The contract leaves this undefined.** Every lease response carries a
`ResponseHeader`. The API defines its `revision` as "the key-value store
revision when the request was applied, and it's unset (so 0) in case of calls
not interacting with key-value store" (`api/etcdserverpb/rpc.proto:424-425`).
Renewals and TTL queries are not applied through Raft, so the contract does
not say what their revision means. What the code does:

| RPC | When the header is filled | Evidence |
|---|---|---|
| `LeaseKeepAlive` | On the member that received the stream, *before* the renewal is attempted or forwarded. Intent, per the code comment: the revision is "strictly smaller or equal to when the keepalive happened", so that a lease "revoked at rev 3" cannot appear renewed "at rev 4" | `server/etcdserver/api/v3rpc/lease.go:127-134` |
| `LeaseKeepAlive`, lease not found | The same pre-filled header is sent with TTL 0 | `v3rpc/lease.go:136-140` |
| `LeaseTimeToLive` | On the member that received the call, *after* the result is in. The result comes from the local lessor, or from the leader over peer HTTP; both leave the header empty ("TODO: fill out ResponseHeader") | `v3rpc/lease.go:62-75`; `v3_server.go:650`; `server/lease/leasehttp/http.go:129` |

**Consequences** (code verified; not run):
- **Successful renewal (TTL > 0).** The revision is a valid lower bound, as
  intended.
- **"Not found" renewal (TTL 0).** The same lower bound now points the
  wrong way: the revision can precede the revocation that made the lease
  disappear.
  - This needs no lagging follower. On the leader, a renewal of an expired
    lease fills the header and then waits for the revocation to apply
    (`lessor.go:415-430`). So a lease with attached keys is reported gone at
    a revision at which it and its keys still existed.
  - A lagging follower widens the gap, e.g. a follower at revision 10
    answering for a revocation at revision 15 (Antigravity's example).
- **`LeaseTimeToLive`: no consistent direction.**
  - On a follower, the TTL and key list come from the leader's newer state,
    while the revision is the follower's.
  - On the leader, the header is filled after the lookup. A positive result
    can therefore carry a revision later than a revocation that happened in
    between.
  - Either way, the key list is not a snapshot at the header revision.

**Exposure.**
- etcd's own recipes do not read lease-response headers:
  `client/v3/concurrency` uses KV, Txn and watch headers only.
- The Go streaming `KeepAlive` drops TTL ≤ 0 responses without delivering
  them (`client/v3/lease.go:534-538`).
- jetcd reports "not found" as an error, without the response
  (`LeaseImpl.java:267-269`, and `keepAliveOnce`).
- The revision is visible through Go `KeepAliveOnce`, which returns the
  response together with `ErrLeaseNotFound` (`client/v3/lease.go:323-326`),
  through `TimeToLive`, and through raw gRPC.

**Assessment.** A P4 metadata inconsistency. It does not affect P3, and it
does not affect fenced writes. A client must not read a lease response's
header revision as the revision at which the result held. — **High** for the
code; impact **Low**.

## 3. Failover-safety bugs (P3)

### 3.1 The reported bugs: #12528 and #15247

**Setup.** A 3-member cluster whose leader's disk (`fdatasync`) or CPU
stalls. The other two members keep quorum and elect a new leader.

| Case | What happens | Effect |
|---|---|---|
| **Case 1:** wrong revocation by the old leader | Clients connected to other members renew at the new leader. The stalled old leader's lessor is never demoted; its timer keeps running on real time, and it finds those leases "expired". When the stall ends, its revocation proposals reach the new leader and commit | Keys of live, actively renewed leases are deleted (election and lock keys) |
| **Case 2:** wrong revocation by the new leader | Clients connected to the stalled old leader keep getting successful keepalive responses from it. The new leader never sees those renewals | The new leader revokes those leases |

**Reports.**
- [#15247](https://github.com/etcd-io/etcd/issues/15247) (2023). It affects
  3.4 and 3.5 per the maintainers, and was reproduced with an injected
  `fdatasync` stall. The maintainer's root-cause write-up
  ([comment 1777862093](https://github.com/etcd-io/etcd/issues/15247#issuecomment-1777862093)):

  > When the leader gets blocked on the raft loop such as writing WAL
  > entries, it can't process any events or messages from the raft, such as
  > leader changing … the older leader thinks it's still the leader.
- [#12528](https://github.com/etcd-io/etcd/issues/12528) (2020, v3.4.6,
  CPU-stall injection) is Case 1. It describes "the etcd old leader will send
  leaseRevoke request to new leader, which results in lease repired [sic] of
  the key, although the client is still sending lease keepalive to the new
  leader". The stale bot closed it. Its fix PR #12531 ("ignore old leader's
  request to revoke lease") was never merged.

**Affected releases:** every release before v3.4.31, v3.5.13 and v3.6.0. —
**High**.

**Trigger profile.**
- Crash failures do not trigger it: a dead leader proposes nothing.
- The trigger is a leader that is alive but stalled: a disk stall, CPU
  starvation, or a VM or GC pause. That is a "gray" failure, which
  quorum-based fault tolerance is meant to tolerate.
- A 5-member cluster is just as exposed as a 3-member one.

### 3.2 Why a deposed leader's proposal commits

The code's own safety argument is a comment at
`server/lease/lessor.go:253-262`:

> there might be two primary leaders at the same time (raft allows concurrent
> leader but with different term) for at most a leader election timeout. The
> old primary leader cannot affect the correctness since its proposal has a
> smaller term and will not be committed.
>
> TODO: raft follower do not forward lease management proposals. There might be
> a very small window (within second normally which depends on go scheduling)
> that a raft follow is the primary between the raft leader demotion and lessor
> demotion.

All three premises are false in the stall scenario.

| Premise | Why it fails |
|---|---|
| "for at most a leader election timeout" | It assumes the old leader learns of its demotion promptly. A stalled Ready loop delays that by the length of the stall (60 s in the #15247 reproduction) |
| "its proposal has a smaller term" and "raft follower do not forward lease management proposals" | False once the old leader's Raft instance has stepped down. Raft's follower step function *forwards* any `MsgProp` to the known leader, unless there is no leader or forwarding is disabled. The leader then appends the proposal in the **new** term |
| "within second normally" | The delay is set by the stall, not by Go scheduling |

Evidence for the forwarding row:
- `go.etcd.io/raft/v3` `v3.7.0`, `raft.go:1718-1729`, fetched from the
  `etcd-io/raft` tag `v3.7.0`, the version pinned in `server/go.mod:30`.
- etcd never sets `DisableProposalForwarding`. `bootstrap.go:540-551` sets
  only `CheckQuorum` and `PreVote`, and the field occurs nowhere in non-test
  etcd code.
- Forwarding is also how client writes sent to followers work.

— **High** (verified; also observed in #12528 and described in the
maintainer's root-cause text).

**Nothing distinguishes a stale revocation at apply time.**
- The `LeaseRevokeRequest` message carries only `ID`
  (`api/etcdserverpb/rpc.proto:889-894`).
- The internal request header carries `ID`, `username` and `auth_revision`,
  but not the proposer's term (`api/etcdserverpb/raft_internal.proto:10-18`).
- The apply path revokes unconditionally (`apply/backend.go:94-96`).

So an expiry-driven revocation is indistinguishable from a client's explicit
`LeaseRevoke`. — **High** (verified).

### 3.3 The fix that shipped, and the fixes that did not

**Shipped.** Commit `67f17166b`, "Safeguard lease operations by double
checking the leadership", PR [#16822](https://github.com/etcd-io/etcd/pull/16822),
merged 2023-12-15. It is in v3.6.0, and was backported to v3.5.13 and
v3.4.31. It adds `ensureLeadership()` in three places:

| Call site | Behaviour | Evidence |
|---|---|---|
| `revokeExpiredLeases` | Checks before proposing a batch | `server.go:858-873` (cites #15247 at :865) |
| `LeaseRenew` on the leader | Returns `ErrNotPrimary` on failure, so the client fails over | `v3_server.go:496-506` |
| The checkpointer | Checks first | `server.go:394-400` |

`ensureLeadership` is a ReadIndex (`LinearizableReadNotify`) plus a check
that Raft still reports this member as leader (`server.go:915-941`). It is a
check before acting, not a fence. A fence would be a term tag on
expiry-driven revocations, checked at apply.

**Not shipped:** two structural alternatives raised in the #15247 thread.
- **Term fencing** (mitake,
  [comment 1510362310](https://github.com/etcd-io/etcd/issues/15247#issuecomment-1510362310)):
  "let LeaseRevokeRequest have a field of Raft term when it was issued … If
  the term from request is older, it simply discard the request".
  - Cons cited: a WAL format change; "Using term information inside the
    state machine is unusual".
  - The maintainer's response cited a "potential compatibility issue".
- **Conditional apply** (serathius,
  [comment 1735268056](https://github.com/etcd-io/etcd/issues/15247#issuecomment-1735268056)):
  a `SkipIfLeaderNotMatch` field, checked at apply. The same comment assesses
  the ReadIndex approach that was eventually chosen: "This reduces the chance
  of this happening, but doesn't fully resolve a possibility of a race."

### 3.4 Residual windows at v3.7.3

All five windows are inferred from code; none was reproduced. Each exists
because the shipped check is a heuristic:
- it is skipped whenever the member's own Raft tick is fresh (#18428);
- it is evaluated once per batch of up to 500 revocations, which are then
  proposed 16 at a time with 7-second timeouts;
- it is absent on the path where a follower forwards a keepalive to the
  leader.

Whether a stalled leader can still revoke a live lease at v3.7.3 is
**unresolved**. A failpoint reproduction would settle it (§10).

| Window | Trigger | What goes wrong | Existence | Impact |
|---|---|---|---|---|
| R1 | A leader stall ends, and the tick is processed before the step-down | Leadership check skipped; a stale revocation batch is dispatched | Moderate | Low |
| R1, partition variant | A leader cut off by the network, otherwise healthy | It acknowledges renewals the new leader never sees | High (mechanism) | Covered by Promote's extension, to within about one tick |
| R2 | Leadership lost during a large expiry batch | Later proposals go out seconds after the single check | Moderate | Low; it stretches R1 and R5 |
| R3 | A follower forwards a keepalive to a stalled ex-leader | The peer renewal path has no leadership check | High | Low |
| R4 | A renewal races a revocation's key deletion | Positive TTL for a lease being revoked | Moderate | Low; only in the last moments of the TTL |
| R5 | A leader change during dispatch of a *correct* expiry batch | The revocation commits after the new leader acknowledged a renewal | Moderate | Low |

#### R1. The leadership check is skipped while the local tick loop is live

**Mechanism.**
- PR [#18428](https://github.com/etcd-io/etcd/pull/18428) (merged
  2024-08-14; in v3.6.0; backported as #18439 in v3.5.16 and #18440 for 3.4)
  made `ensureLeadership` return `true` without any ReadIndex whenever the
  member processed a Raft tick within 3 × `TickMs` (`server.go:908-923`).
- The motive was performance. The unconditional ReadIndex on every renewal
  cost about 20 % leader CPU at 1500 renewals/s
  ([#18069](https://github.com/etcd-io/etcd/issues/18069)).
- Ticks and Ready processing share one `select` loop
  (`server/etcdserver/raft.go:159-162`, `:183-185`). So during a stall,
  `isActive()` is false and the full check runs. That is why the fix still
  covers the #15247 reproduction.

**The residual window.**
- After the stall ends, the loop can take the buffered tick before the Ready
  that carries the step-down state, because Go's `select` chooses randomly
  among ready cases.
- Between that tick and `Demote()`, the lessor is still primary, and
  `ensureLeadership()` returns `true` with no quorum check. Any revocation
  batch dispatched in that window proceeds.
- The window is about one Raft-loop iteration, sub-millisecond to
  milliseconds (inferred). The lessor scans every 500 ms (`lessor.go:623`),
  so the probability per incident looks small.

**Upstream objection.** A maintainer objected to #18428 on safety grounds
after it was merged
([review r1719518498](https://github.com/etcd-io/etcd/pull/18428#discussion_r1719518498)):
"until heath probes fail for ElectionMs, the old leader will still think it's
a leader before they resign". Also:
"[Please consider reverting and redesign](https://github.com/etcd-io/etcd/pull/18428#issuecomment-2291124275)".
The author replied that the check only targets self-stalls, and that Raft
handles partitions.
- For revocations, I agree with the author on the partition case. A
  minority-side leader's proposals cannot commit, and it steps down before
  typical TTLs elapse (§1.3, item 5).
- For renewals, Promote's extension is designed to cover the window; the
  next paragraph checks that.

— **Moderate**.

**Partition variant (checked 2026-11-01).**

*Setup.* Leader L1 is cut off by the network, with healthy disk and CPU. Its
tick loop keeps running, so `isActive()` stays true and `ensureLeadership()`
returns true without a ReadIndex. Until L1's Raft steps down, L1
acknowledges renewals that the new leader F never sees. The question is
whether that breaks P3. The timing below is derived from raft `v3.7.0`.

| Step | Bound | Evidence |
|---|---|---|
| L1 steps down | Only at a CheckQuorum check, which `tickHeartbeat` runs every ET. The check passes if a quorum was heard from since the previous check, then resets the activity marks. If the last response arrives just before the partition at t_p, the next check passes and the one after fails. So L1 steps down by **t_p + 2·ET** at the latest | `raft.go:862-880`, `:1281-1292` |
| F's election starts | Followers campaign after a randomized timeout of [ET, 2·ET) ticks since they last heard from L1. With CheckQuorum, they refuse votes while within ET of hearing from a leader. Their last contact was about t_p minus half a round trip, and counting ticks can shorten ET by up to one tick in real time. So the election starts no earlier than about **t_p + ET − 1 tick** | `raft.go:850-859`, `:2049-2055`, `:1103` |
| F promotes | Later still: after pre-vote, vote, and committing and applying its empty entry | — |
| Comparison | F's expiry is t_promote + ET + TTL. L1's last acknowledgement is at t_ack ≤ t_p + 2·ET and promises t_ack + TTL. So F's expiry ≥ t_ack + TTL − (1 tick − election duration) | — |

*Result.*
- With the default 100 ms tick, the shortfall is at most about one tick, and
  zero once the election takes longer than a tick.
- L1's lessor demotes in the same Ready iteration as the step-down, which is
  negligible when L1 is not stalled.
- So the ET extension covers the partition case. That is presumably why the
  extension is ET, but this is my inference; the code does not say so.
- The worst-case residue, about 100 ms, has the same order and direction as
  the client-side optimism in §5.1. It matters only if the client also fails
  to renew anywhere for the rest of its TTL.
- Stalls differ, because L1's ticks stop: its step-down is delayed by the
  length of the stall, which ET does not cover. That is #15247.

— **Moderate** (derived, not measured).

#### R2. One check per batch; later proposals in the batch go out seconds later

**Mechanism.**
- A batch can contain up to `leaseRevokeRate/2 = 500` leases
  (`lessor.go:644`).
- They are proposed through a 16-slot semaphore (`maxPendingRevokes = 16`,
  `server.go:103`; loop at `:875-900`), after a single `ensureLeadership()`
  call (`server.go:867`).
- A proposal that never commits holds its slot until `ReqTimeout`, which is
  5 s + 2 × ET, 7 s by default (`server/config/config.go:320-324`, used at
  `v3_server.go:1109`). An example is a proposal appended in a deposed
  leader's own log at its old term, which is later discarded.

So revocations later in the batch go out seconds after the check. Nothing
re-checks leadership, lessor primacy or term:
- `LeaseRevoke` → `processInternalRaftRequestOnce` → `Propose` has no such
  check (`v3_server.go:471-487`, `:1059-1134`);
- if the member has meanwhile become a follower that knows the new leader,
  those proposals are forwarded, and they commit.

**Assessment.**
- R2 does not open a window by itself. It stretches R1 and R5 from
  milliseconds to `⌈batch/16⌉ × 7 s`. It does the same for any case where
  the check passes and leadership is lost before the batch is fully
  proposed.
- Existence **Moderate**. Impact **Low**, unless R1 or R5 occurs with more
  than 16 expired leases in the batch.

#### R3. Forwarded renewals skip the leadership check

**Mechanism.**
- A follower forwards a keepalive to the member it believes is leader
  (`v3_server.go:541-560`).
- The leader-side peer handler calls `lessor.Renew` directly, without
  `ensureLeadership` (`leasehttp/http.go:91`). The handler is built from the
  bare lessor at `server.go:646`.
- `lessor.Renew` checks only the lessor's own primary flag
  (`lessor.go:397-402`). That flag is exactly the stale state in Case 2.
- The handler's only wait is for its own apply index to reach its own commit
  index, with a 1-second timeout (`leasehttp/http.go:37`, `:84-89`; the wait
  is built from `ApplyWaitCommit`, `server.go:632-634`, `:646`). A leader
  stalled in its Raft loop commits nothing new, so it has usually applied
  everything already, and the handler renews at once.

**Assessment.**
- This matters only if a follower still routes to a stalled ex-leader.
- Followers that forward to L1 before they elect a new leader are covered by
  Promote's extension, by the same timing argument as the partition variant
  of R1.
- After voting, they know the new leader. So the window looks negligible.

Existence **High** (verified); impact **Low**.

#### R4. Renew/revoke race (not specific to failover)

**Mechanism.**
- `Revoke` releases the lessor lock while it deletes keys. It removes the
  lease from the map only afterwards (`lessor.go:326-361`), and closes
  `revokec` only at return (:335).
- `Renew` re-checks only map membership (`lessor.go:442-448`). That re-check
  was added by [#20590](https://github.com/etcd-io/etcd/pull/20590) in
  v3.7.0, v3.6.5 and v3.5.23.
- So a keepalive that arrives during a revocation's key deletion can still
  return a positive TTL for a lease whose keys are already gone.
- PR [#21389](https://github.com/etcd-io/etcd/pull/21389) reproduced exactly
  this with failpoints ("Fixes #14758"). It was closed unmerged (stale,
  2026-06-19).

**Automatic revocation.** The race applies here too, but narrowly.
- A renewal that *arrives* after the lease expired waits for the revocation
  and is refused (`lessor.go:415-430`).
- Only a renewal that passed that expiry check just before expiry, and was
  then delayed until the key deletion was in progress, gets through.
- So R4 can violate P3 without any leader change, but only for a renewal sent
  in the last moments of the TTL.

— **Moderate** (the code path is verified; the reproduction is the PR
author's, not mine).

#### R5. A correct revocation delayed past a leader change

No stall is needed. This is traced from code, not reproduced; the code for
each step is verified.

1. A is leader. Client C's lease L genuinely expires on A. Either C stopped
   renewing for TTL (it was paused or cut off), or the server delayed its
   renewals, e.g. forwarded keepalives timing out under apply backlog (§6.1).
2. A's lessor puts L in an expiry batch. `revokeExpiredLeases` passes
   `ensureLeadership()` once (`server.go:867`), then dispatches one goroutine
   per lease through the 16-slot semaphore (`server.go:874-900`). The
   goroutines run under `GoAttach`, which stops them on server shutdown, not
   on demotion.
3. Leadership moves to B before L's proposal enters the log, e.g. by a leader
   transfer or an election after a blip.
   - With more than 16 leases in the batch, this needs no scheduling delay.
   - The proposals A appended in its old term that never commit hold their
     slots for up to 7 s (R2), so later ones are dispatched after the change.
4. B applies every entry of earlier terms, and only then promotes
   (`server.go:1959-1967`). L has not been revoked, so `Promote` extends it
   to TTL + ET (`lessor.go:480-492`).
5. C's renewal reaches B. On B, L is not expired, so `Renew` succeeds and
   promises a full TTL. On a single leader the same late renewal would have
   been refused, because `Renew` on an expired lease waits for the revocation
   and returns "not found" (`lessor.go:415-430`).
6. A's goroutine calls `LeaseRevoke(L)`. A is now a follower, so Raft forwards
   the proposal to B (§3.2). It commits in B's term, and apply revokes L
   unconditionally (`apply/backend.go:94-96`). L's keys are deleted within TTL
   of B's acknowledgement, which violates P3 for C.

**Exposure and bounds.**
- **C must still be renewing after its lease expired on A.** A Go client
  stops on its own once the deadline sweep reaps the keep-alive entry: at
  most about one second after the server's expiry, plus the reply delay
  (§5.1).
- **The realistic victim** is an application that reacts to a closed
  keep-alive channel by calling `KeepAlive` again on the same lease ID, or
  that renews with `KeepAliveOnce`. Such a renewal succeeds at B, and the
  lease is revoked shortly afterwards.
- **The gap is short unless R2 stretches it.** With at most 16 leases in the
  batch, the gap between the check and the proposal is a goroutine hop,
  microseconds to milliseconds (inferred). During a leader change, larger
  batches stretch it to seconds (R2).
- **Only a fence closes it.** Unlike R1–R3, A's view was *correct* when it
  decided. The only fault is that the decision takes effect in a later term,
  after B has re-validated the lease. A term carried in the revocation and
  checked at apply would reject it (§3.3, §4.3). Another check before
  proposing would only narrow the window.

Existence **Moderate**; impact **Low**.

### 3.5 Checkpointing bugs (only when the Alpha gate is on)

**[#17132](https://github.com/etcd-io/etcd/issues/17132):** a historical bug.
With checkpointing on but persistence off, restarting members lost the
checkpointed remaining TTL, which reset to full on a later leader change. The
maintainer's response: "known issue … The fix is the
`--experimental-enable-lease-checkpoint-persist` flag". At v3.7.3,
persistence is automatic for cluster version ≥ 3.6 (`lessor.go:385-388`).

**[#22649](https://github.com/etcd-io/etcd/issues/22649)** (open, opened
2026-10-23).
- **Mechanism.**
  - The checkpointer closure discards `raftRequest`'s result and returns
    `nil` (verified: `server.go:401-402`).
  - So if the "clear remaining TTL" proposal in `Renew` fails (timeout,
    overload, leader change), `Renew` proceeds as if it had succeeded.
  - It refreshes with the stale remaining TTL (`lease.go:76-89`), and reports
    the full TTL to the client (`lessor.go:455`).
  - The server-side window is therefore shorter than the TTL the client was
    told. The reporter reproduced it on a real server; I verified the code
    path, not the reproduction. — **High** (code).
- **Margin under steady renewal (my refinement; not in the issue).**
  - Checkpoints exist only for leases with TTL > 300 s (`lessor.go:758`).
    Under steady keepalives every TTL/3, a checkpointed remaining TTL is at
    least about 2/3 × TTL.
  - So one silent failure shrinks the margin but does not by itself revoke a
    steadily renewed lease.
  - Premature revocation needs a stale checkpoint taken while renewals were
    irregular, or further delays. — **Moderate**.
  - The failure is most likely during leader changes and overload, which are
    exactly when checkpointing is supposed to help.
- **A late clear does not repair the timer.** If the timed-out clear applies
  later, `Checkpoint` only sets `remainingTTL` (`lessor.go:372`). The
  shortened expiry installed by `refresh` stays until the next successful
  renewal. — **High** (code).
- **A leader cut off from the quorum triggers it for every checkpointed lease
  renewed there before it steps down.**
  - The clear can never commit there: the proposal is dropped, or times out
    after `ReqTimeout` (7 s). The error is discarded, and `Renew` continues.
  - After the clear, `Renew` re-checks only that the lease is still in the map
    (`lessor.go:442-449`), not that the lessor is still primary. So it reports
    the full TTL even if the member was demoted while waiting.
  - The new leader holds the uncleared remaining TTL, and extends from that.
  - Under steady renewal the margin argument above still applies: the margin
    shrinks, but no lease is revoked by this alone. — **Moderate** (traced).
- **Fix status.** PR [#22650](https://github.com/etcd-io/etcd/pull/22650),
  "server: propagate raftRequest error from lease Checkpointer", opened
  2026-10-23, is open and unmerged. I checked its title and state, not its
  diff. The defect is unchanged on `main` at `1573d5941`.

### 3.6 Other narrow violations

| Issue | What happened | Status |
|---|---|---|
| [#20590](https://github.com/etcd-io/etcd/pull/20590) | A keepalive that raced with a revocation could report success | Fixed in v3.7.0, v3.6.5, v3.5.23 |
| [#21389](https://github.com/etcd-io/etcd/pull/21389) | The same kind of race at the expiry boundary (R4) | Fix PR closed unmerged; still present |
| [#13675](https://github.com/etcd-io/etcd/issues/13675) (v3.5.2) | A follower acknowledged a `LeaseGrant` before the leader had applied it. The client's first keepalive reached the leader, which answered "not found" (TTL 0), and `concurrency.Session` closed immediately. The lease itself was never revoked, and no leader change was involved | Fixed in v3.5.3 by [#13690](https://github.com/etcd-io/etcd/pull/13690): wait for the apply index before renewing an unknown lease. `FastLeaseKeepAlive` keeps that wait at v3.7.3 (`server/etcdserver/v3_server.go:512-523`). — **High** (maintainer diagnosis; fix verified in code) |

None of these affects a client that renews on schedule and whose lease nobody
else revokes.

### 3.7 Timeline and process signal

| Date | Event |
|---|---|
| 2018-03 / 2018-06 | TTL reset on leader change and restart reported (#9395); Kubernetes Events never expiring (#9888, Kubernetes #65497) |
| 2019-08 | Checkpointing ships behind an experimental flag (v3.4.0, #9924) |
| 2020-12 | Old-leader revocation reported (#12528) with a reproduction; fix PR #12531 opened |
| 2021-06 / 2021-09 | #12528 closed by the stale bot; #12531 closed unmerged |
| 2022-02 | Checkpoint persistence flag (v3.5.2, #13508); grant/keepalive "not found" race reported (#13675), fixed in v3.5.3 (#13690) |
| 2022-04 | Redesign tracker "Make leases linearizable" (#13915) opened |
| 2023-02 | Same bug re-reported (#15247); labelled important-soon |
| 2023-12 / 2024-03 | ReadIndex safeguard merged (#16822); backported (v3.5.13, v3.4.31) |
| 2024-03 | jetcd commit `10b07f5` (first released in 0.8.0) starts reporting every keep-alive stream error through `onError` while continuing to renew (jetcd #1322, §5.3) |
| 2024-08 | Safeguard short-circuited for performance (#18428); post-merge safety objection |
| 2025-09 | Renew/revoke TOCTOU partially fixed (#20590); checkpoint tracker #13776 closed *not planned* by the stale bot (its `stage/tracked` exemption was not being applied; see "Reading the timeline"); #13915 closed by the bot, then reopened by a maintainer |
| 2025-10 | `FastLeaseKeepAlive` merged (#20589): leader skips the apply wait for known leases; the follower-forwarding path keeps it (§6.1) |
| 2025-12 | #13915 closed *not planned* by the stale bot again, with the exemption still not applied |
| 2026-06 | Fix PR for the remaining renew/revoke race (#21389) closed unmerged (stale) |
| 2026-10 | #22649 (checkpoint error swallowed) open; fix PR #22650 open |

**Reading the timeline.**
- **Slow fix, no structural fix.** It took about three years from the first
  reproduction of the failover bug to a released fix, and the structural fix
  was never adopted.
- **The open-issue count understates the problem.** Only 5 open etcd issues
  have "lease" in the title (search on 2026-10-31). Yet:
  - the core failover bug was closed by the stale bot once (2021) before
    being re-reported and fixed (2023);
  - both redesign trackers,
    [#13915](https://github.com/etcd-io/etcd/issues/13915) and
    [#13776](https://github.com/etcd-io/etcd/issues/13776), were closed *not
    planned* by the project's stale bot. Both carried `stage/tracked`, the
    label meant to exempt an issue from auto-closing. From 2025-05-13 to
    2026-01-13 the bot's workflow used input names the GitHub Action does
    not recognize, so the exemption was ignored (fixed in commit
    `55c5fabe5`), and both closures fall in that window. #13915 was closed
    on 2025-09-11, reopened by a maintainer on 2025-09-23, and closed again
    on 2025-12-19. Neither has been reopened since. "Not planned" is the
    reason the bot records, not a maintainer's verdict on the designs; for
    a user of etcd the outcome is the same, because the bot is part of the
    project's process (corrected 2026-11-09);
  - etcd's linearizability harness does not exercise lease expiry or
    keepalive at all (§8).

— **High**.

## 4. Is there a design mistake relative to Chubby?

### 4.1 What etcd shares with Chubby (argues against the hypothesis)

| Shared trait | Chubby | etcd |
|---|---|---|
| **Session timer at the master, outside the replicated log** | "The authoritative timer for session leases runs at the master, so until a new master is elected the session lease timer is stopped; this is legal because it is equivalent to extending the client's lease" (§2.9, p. 8). The database records "each session, held lock, and ephemeral file" (p. 9): existence, not timers | Persists lease ID, TTL and (with checkpoints) a remaining TTL, never the expiry instant (`lease.go:55-61`). Same split |
| **Extension on fail-over** | "Session leases are extended to the maximum that the previous master may have been using" (§2.9 step 3, p. 9) | `Promote(extend)` does the same, conservatively: full TTL plus ET |
| **Fail-over code is bug-prone** | "the fail-over code, which is exercised far less often than other parts of the system, has been a rich source of interesting bugs" (§2.9, p. 9) | The history in §3.7 matches |
| **Unreplicated sessions cost a conservative recovery rule** | §4.4, pp. 12–13. The original design wrote each session to the database at creation. Under load, this changed to recording a session at its first modification, and active sessions "with some probability on each KeepAlive". The revised design avoids "recording sessions in the database at all", and a new master "must now wait a full worst-case lease timeout before allowing operations to proceed" | `Promote`'s full-TTL extension: no lease expires until TTL + ET after a leader change. Unlike Chubby, etcd does not block other operations meanwhile. This is the P2 cost (§2.2), shared with Chubby by design |

Chubby thus moved further away from replicating session state, not closer.
So "renewals do not go through consensus" is not, by itself, the mistake.

**Why not put renewals through Raft.**
- **Cost.** Every keepalive would become one replicated log entry, persisted
  (fsync, batched) on a quorum. The volumes involved, e.g. 1500 renewals/s
  in #18069, are the load Chubby deliberately avoided.
- **Chubby sheds that load instead.** It lets a loaded master hand out longer
  extensions: "The default extension is 12s, but an overloaded master may
  use higher values" (§2.8, p. 7), up to about 60 s (§3, p. 10).
- **etcd has no such load control.**
  - A TODO in `Grant` names it: "when lessor is under high load, it should
    give out lease with longer TTL to reduce renew load"
    (`lessor.go:290-291`).
  - The request [#12594](https://github.com/etcd-io/etcd/issues/12594),
    "Give increased lease timeout if etcd is under load", was closed by the
    stale bot in 2021.

### 4.2 Where etcd departs from Chubby (argues for a narrower design flaw)

| Aspect | Chubby | etcd v3.7.3 | Consequence in etcd |
|---|---|---|---|
| How a master knows it is still master | **Master lease**: replicas promise not to elect another master for a time interval; reads are served "provided the master lease has not expired" (§2.2, p. 3). Real-time bounded | Raft leadership. CheckQuorum step-down counts **Raft ticks** (`bootstrap.go:548`; raft `v3.7.0` `raft.go:866-868`), and ticks are processed by the same loop that stalls on fsync (`server/etcdserver/raft.go:183-185`) | A stalled leader's "am I leader?" clock stops, while its lease-expiry clock (real time) keeps running. That is the precondition for both cases of #15247 |
| Fencing of stale actions | Client calls carry a master epoch: "The master rejects calls from clients using older epoch numbers" (§2.9 step 1, p. 9). Writes are issued by the master itself through consensus (§2.2); a deposed master's proposals would lose to the new master's (my inference from Paxos ballots; the paper does not discuss it) | Revocations carry no term (`rpc.proto:889-894`). Raft forwards a follower's proposal into the current term (raft `raft.go:1718-1729`) | A deposed leader's time-based decision can commit in the new term (§3.2) |
| Client's view of its lease | "a conservative approximation of the master's lease timeout", allowing for reply flight time and clock-rate bounds (§2.8, p. 7) | Deadline = response *receipt* time + TTL (`client/v3/lease.go:543`; same in jetcd) | The client's belief can outlast the server's; about 1 s in typical conditions, unbounded with pauses (§5.1) |
| Fail-over grace for clients | Jeopardy event plus a 45 s grace period; API calls block (§2.8–2.9, pp. 7–8) | None at the protocol level. `concurrency.Session` exposes only `Done()` | Applications get no "unsure" state; they see either success or expiry |
| Lock release when a holder fails | **lock-delay**, up to 1 min, before another client can take a lock freed by a failed holder (§2.4, p. 5) | None. A waiter's watch fires on the revocation's delete | Shrinks the safety margin for unfenced resources (P5) |
| Sequencers | Lock generation number passed to servers (§2.4) | Lock key revision. The etcd docs equate a sequencer with "revision number and lease ID" ([why](https://etcd.io/docs/v3.6/learning/why/)) | Equivalent fencing exists, but its use is optional. A lease ID alone is not ordered and is not a fencing token |
| KeepAlive load control | The master holds each KeepAlive until the lease is nearly due, and lengthens leases (12 s default, up to ~60 s) when overloaded (§2.8, p. 7; §3, p. 10) | Immediate reply with the fixed granted TTL; the client renews every TTL/3 (`client/v3/lease.go:542`); TODO at `lessor.go:290-291` | Renewal load scales with lease count ÷ TTL and cannot be shed under overload. Overload instead delays forwarded renewals (§6.1) |

### 4.3 Verdict on the hypothesis

**Strongest case for a design mistake.**
- etcd adopted Chubby's leader-local timers without Chubby's two supporting
  mechanisms: time-bounded mastership, and epoch-fenced actions.
- It then gave a stale leader a channel, proposal forwarding, that turns its
  stale judgement into replicated state.
- The code's own correctness argument (`lessor.go:253-262`) is still in the
  source, and the Raft library's forwarding contradicts it.
- The structural repairs (term fencing, conditional apply) were proposed but
  not adopted; a maintainer cited possible compatibility issues.
- The adopted mitigation was later partly bypassed for performance.

**Strongest case against.**
- A lease timer outside the log is the right architecture, and Chubby proves
  it workable.
- The specific holes have been patched for the reported scenarios.
- The residual windows (§3.4) need a multi-second stall (R1, R2), or a late
  client coinciding with a leader change (R5), *and* a narrow interleaving.
- A plain network partition is covered by the promotion extension (R1,
  partition variant).
- Chubby, too, pays for unreplicated sessions with a conservative recovery
  rule (§4.1).
- No lease design gives mutual exclusion without fencing (Jepsen; etcd docs;
  Chubby §2.4), so applications that need safety must fence anyway.

**Assessment.** The suspected mistake is real but mislocated. It is not
"leases bypass Raft". It is this: **time-driven decisions (expiry →
revocation) are made from one member's soft state but committed as epoch-less
commands, while that member's authority to decide is not bounded on the same
clock.**

The two halves of the flaw need different repairs.

| Half | Affected | Repair |
|---|---|---|
| **Revocations** | Case 1, R1, R2, R5 | A fence. R5 shows the hole exists even when the deciding member's view was right when it decided: the defect is check-then-act across terms. Another check before proposing only narrows the window. Carrying the deciding term in the command, and checking it at apply, closes it |
| **Acknowledgements** | Case 2, R3 | A real-time bound on the acknowledging member's authority, as Chubby's master lease provides. etcd approximates it with a ReadIndex when the tick loop looks stalled, and with the promotion extension otherwise. The partition analysis (R1) suggests the combination is adequate, except on the follower-forwarding path, which has no check at all |

— **Moderate**. The mechanism is verified; "this is the flaw" is a
judgement. It would fail if forwarded proposals were rejected somewhere
between Raft's forwarding (`raft.go:1718-1729` @ raft `v3.7.0`) and etcd's
apply. I found no such check in etcd (`v3_server.go:1059-1134`,
`apply/backend.go:94-96`), and #12528 and #15247 observed such proposals
committing.

## 5. Client libraries

### 5.1 Optimistic lease deadlines (Go and jetcd)

**Go client.**
- On each keepalive response it sets `deadline = time.Now() + TTL`
  (`client/v3/lease.go:542-543`), whereas the server set its expiry when it
  refreshed the lease on the leader (`lessor.go:449`, `lease.go:84-89`).
- So the client's deadline is later than the server's by the response path
  delay, plus any client scheduling delay before `recvKeepAlive` runs.
- Chubby instead requires the client's local timeout to be conservative,
  allowing for reply flight time and for the master's clock running faster
  (§2.8, p. 7). — **High** (code).

**The 1-second sweep widens the gap.**
- `deadlineLoop` runs once per second (`client/v3/lease.go:562-583`; timer
  at `:563`), so the keep-alive channel closes up to about 1 s after the
  local deadline. `Session.Done()` follows that channel.
- On the server, the lessor scans every 500 ms (`lessor.go:623`) and
  proposes the revocation. A contender's watch fires when it commits.
- **Window.** A holder that has stopped receiving keepalive responses can
  still say "held" while a successor already holds the lock, for up to about
  1 s plus the reply delay.
- **The window is typical, not a bound.** It assumes no process pause or
  clock step on the client. Pauses lengthen it without limit, and a slow
  server revocation shortens it.
- **Who is exposed.** Fenced etcd writes are unaffected. Unfenced external
  actions gated on `Session.Done()` are exposed for that long, even without a
  pause.

— **High** for the code; **Moderate** for the magnitude (derived, not
measured).

**jetcd 0.8.7.**
- Same scheme, using `System.currentTimeMillis()`, a wall clock subject to
  NTP steps (`jetcd-core/src/main/java/io/etcd/jetcd/impl/LeaseImpl.java:263-265`).
- Deadlines are also swept once per second (`:299-312`).

### 5.2 Go client: `WithRequireLeader` on keep-alive streams

The session that wrote `analysis/watchers-claude.md` contributed this
subsection on 2026-10-31. §13 of that file covers the general mechanism, its
2016 origin, and a Docker reproduction for watches. Everything below was
verified in code at `v3.7.3`, unless marked "traced" (consequence derived
from code, not run).

**The mechanism.**
- `WithRequireLeader(ctx)` adds the gRPC metadata `hasleader=true`
  (`client/v3/ctx.go:26-38`).
- A member that has no Raft leader rejects new flagged streams.
- It cancels open flagged streams with `ErrGRPCNoLeader` after 3 consecutive
  leaderless samples, one per election timeout
  (`server/etcdserver/api/v3rpc/interceptor.go:38`, `:239-258`, `:315-354`).
- **Timing.** Measured on watch streams: 3.3 s and 4.3 s after isolating the
  leader (watchers-claude.md Appendix C). Lease streams use the same
  interceptor, but their timing was not measured.

**Overall effect.**
- Both bugs below, #22081 and #22094, push the lease's life past what the
  application believes, in the same direction as P2. Neither revokes early.
- The gRPC proxy also ignores the flag on keep-alive streams (§5.2.4).
- Exposure is limited to applications that pass flagged contexts to
  `KeepAlive`. `concurrency.Session` does not by default, and kube-apiserver
  does not call `KeepAlive` (§7).

— **High** for the code; **Moderate** for the traced consequences.

#### 5.2.1 How the Go lease client uses the flag

1. **The shared keep-alive stream is always flagged.** `l.stopCtx` is
   derived from `WithRequireLeader(context.Background())`
   (`client/v3/lease.go:210-211`). Lease-client support dates from v3.2.0
   (commit `8024a0d15`, 2017-03-30). This is what gets keep-alives off a
   member that is cut off from the quorum.
2. **On `ErrNoLeader`:**
   - `recvKeepAliveLoop` calls `closeRequireLeader`, and re-opens the stream
     after `retryConnWait` = 500 ms (`lease.go:478-481`, `:488-489`, `:86`).
     The new stream may land on another endpoint.
   - `closeRequireLeader` closes only the subscriber channels whose *own*
     `KeepAlive(ctx, id)` context was flagged (`lease.go:376-410`).
   - Unflagged subscribers are not told. They learn of trouble only through
     the local deadline, TTL after the last response (`lease.go:542-543`),
     which `deadlineLoop` checks every second (`:562-583`).
3. **The server stops renewing from a leaderless member even without the
   flag.**
   - A follower forwards renewals until `ReqTimeout()` expires: 5 s + 2 × ET
     = **7 s** by default (`server/etcdserver/v3_server.go:538-548`,
     `:785-803`; `server/config/config.go:320-324`).
   - It then returns `ErrNoLeader`, and the handler ends the *whole*
     keep-alive stream (`server/etcdserver/api/v3rpc/lease.go:136-143`).
   - So keep-alives through a partitioned member stop within about 3–7 s
     either way. The leader's lessor then revokes at TTL. That is the
     intended contract; the bugs below concern what the client *believes*
     afterwards.

#### 5.2.2 #22081: misaligned compaction in `closeRequireLeader`

Code confirmed; consequence traced, **Moderate**.
- `lease.go:406` copies `ka.ctxs[newIdx]` where it should copy `ka.ctxs[i]`.
  After compaction, channels and contexts are misaligned whenever a flagged
  subscriber precedes an unflagged one.
- `keepAliveCtxCloser` finds a subscriber by its context, and closes
  `ka.chs[i]` at that index (`lease.go:342-372`).
- **Example.** Before: `chs=[A(flagged), B]`, `ctxs=[cA, cB]`. After a leader
  loss: `chs=[B]`, `ctxs=[cA]`. Then:
  - cancelling `cA` closes **B's** channel, while B's context is live, and
    deletes the entry. Keep-alives stop, and the lease expires at TTL,
    although B never asked for that;
  - cancelling `cB` matches nothing. B's channel stays open and the entry is
    never deleted, so the lease is **renewed indefinitely** after its last
    live subscriber cancelled. This lasts until `cA` is cancelled, if ever,
    which removes it.
- **Requires** one lease with at least two `KeepAlive` subscribers on the
  same client, of mixed flag status, and a leader loss on the serving member.
- Fix PR [#22087](https://github.com/etcd-io/etcd/pull/22087) is open.

#### 5.2.3 #22094: orphaned entry renewed forever

Code confirmed; consequence traced, **Moderate–High**.
- When every subscriber of a lease is flagged, `closeRequireLeader` empties
  `chs`/`ctxs` but never deletes the entry from `l.keepAlives`
  (`lease.go:376-410`). Compare `keepAliveCtxCloser`, which deletes the
  entry when no listeners remain (`:369-371`).
- `sendKeepAliveLoop` keeps sending requests for the entry
  (`lease.go:586-619`).
- `recvKeepAlive` refreshes `ka.deadline` (`:543`), but advances
  `ka.nextKeepAlive` only *inside* the per-channel loop (`:544-556`). With no
  channels, `nextKeepAlive` never advances, so a request goes out on every
  500 ms pass instead of every TTL/3.
- Because the deadline keeps moving forward, `deadlineLoop` never reaps the
  entry.
- **Effect.**
  - After a leader loss the application sees its keep-alive channel close,
    i.e. "lease lost".
  - Yet the client renews the lease every 500 ms for the life of the
    process, and keys attached to it never expire unless the application
    calls `Revoke`.
  - For a lock or election key, other contenders are blocked indefinitely.
    That is a liveness failure in the same direction as P2, not a
    mutual-exclusion violation.
- **Condition.** The renewed stream must reach a member with a leader before
  the entry's deadline (TTL after the last response) passes; otherwise
  `deadlineLoop` reaps it. With several endpoints, and TTLs much longer than
  the 3–5 s cancellation, that is the normal case.
- **Exposure.** Only applications that pass flagged contexts to `KeepAlive`.
  `concurrency.NewSession` uses `client.Ctx()` unless `WithContext` is given
  (`client/v3/concurrency/session.go:43`, `:57-58`), so it is unflagged by
  default.
- Fix PRs [#22253](https://github.com/etcd-io/etcd/pull/22253) and
  [#22291](https://github.com/etcd-io/etcd/pull/22291) are open.

#### 5.2.4 The gRPC proxy ignores the flag on keep-alive streams

Code **High**; the lease variant was not run.
- `server/proxy/grpcproxy/lease.go:160-173` looks for the flag with
  `metadata.FromOutgoingContext` on a *server* stream, where client metadata
  is in the incoming context.
  - This is the same defect as in the watch proxy
    (`server/proxy/grpcproxy/watch.go:112`).
  - It was introduced by PR #7896 (v3.3.0), and reproduced for watches on
    v3.7.3 (watchers-claude.md §9.1 and Appendix C).
- The proxy also forwards each lease through its own Go lessor. It
  subscribes with a context derived from the client stream, which carries no
  flag (`lease.go:276-279`). So even its own backend `ErrNoLeader` would not
  close that subscription early.
- **Effect (traced).** A client on the proxy whose backend member is
  partitioned gets neither keep-alive responses nor an early error. Its
  channel closes only at its local deadline, TTL after the last response,
  instead of after about 3–5 s.

#### 5.2.5 Older servers report any cancellation as leader loss

Code **Moderate**; practical impact **Low**, not checked.
- Before v3.7.0, the server's `LeaseKeepAlive` reported **any** cancellation
  of the stream context as `ErrGRPCNoLeader`: "the only server-side
  cancellation is noleader for now"
  (`server/etcdserver/api/v3rpc/lease.go:100-105 @ v3.6.0`).
- v3.7.0 (PR #21122, commit `b54c88406`) returns the real reason
  (`lease.go:97-108 @ v3.7.3`). The change was not backported to
  release-3.4, release-3.5 or release-3.6.
- So against older servers, a cancellation unrelated to leader loss can look
  like leader loss to a Go client. It then triggers `closeRequireLeader`, and
  with it #22081/#22094. I did not check whether a member shutdown actually
  delivers that status to clients.

### 5.3 jetcd keep-alive error semantics

All citations are to
`jetcd-core/src/main/java/io/etcd/jetcd/impl/LeaseImpl.java @ jetcd-0.8.7`
unless stated. Code facts are **High**; consequences are traced, not run.

**How it behaves.**
- **One shared stream, always flagged.** All `keepAlive` traffic goes over
  one bidirectional stream, on a stub with `WithRequireLeader` metadata
  (`:74`; `support/Util.java:115-122`), as with the Go client's shared
  stream (§5.2.1). Unlike Go, there is no per-subscriber flag.
- **Every stream error goes to every observer.** `handleException` runs for
  any stream error: leader loss on the serving member (`ErrNoLeader`, §5.2),
  endpoint failure, or a transport error.
  - It calls `onError` on every observer of every lease (`:278`).
  - It keeps all entries in the `keepAlives` map.
  - It restarts the stream after 500 ms (`:280-286`).

  `KeepAliveObserver.onError` does not remove observers (`:392-397`). When
  renewals resume, the same observers receive `onNext` after `onError`.
- **Only four events end a keep-alive:**

  | Event | What jetcd does | Evidence |
  |---|---|---|
  | TTL-0 response | Removes the entry, then calls `onError(NOT_FOUND)` | `:267-269` |
  | Local deadline passes | Removes the entry, then calls `onCompleted` | `:303-311` |
  | `close()` on the returned `CloseableClient` | Removes the observer, and the entry once no observers remain | `:368-374` |
  | `Lease.close()` | Closes everything | — |

  So `onError` is non-terminal unless it carries `NOT_FOUND`. That departs
  from grpc-java's `StreamObserver` convention, under which
  `onError`/`onCompleted` end the stream (recalled). The interface Javadoc
  says only "keep the given lease alive forever"
  (`jetcd-core/src/main/java/io/etcd/jetcd/Lease.java:79-86`).

**Where it comes from.**
- **Origin.** jetcd #1322 (2024-02) asked for keep-alive exceptions to be
  visible to the caller. Commit `10b07f5` (2024-03-05, "Fixes #1322"; first
  tag `jetcd-0.8.0`) added the `onError` fan-out and left the entries in
  place.
- **Intended, but not exercised.** The unit test
  `testKeepAliveResetOnStreamErrors` encodes exactly this contract. It
  injects a stream error, expects keep-alive requests to continue ("expect
  keep alive requests are still sending even with reset"), and expects
  `onError` to have been called
  (`jetcd-core/src/test/java/io/etcd/jetcd/impl/LeaseUnitTest.java:233-249`).
  But the whole class is `@Disabled("TODO: does not seems to add much
  value")` (`:57`).

**Consequence (Moderate).**
- Take an application that treats `onError` as lease loss and does not call
  `close()`. It reaches the same end state as Go #22094 (§5.2.3): it
  believes the lease is lost, while jetcd renews it for the life of the
  client, and keys attached to it, such as lock keys, never expire.
- The trigger is wider than in Go: any stream error, including every leader
  loss on the serving member, not only flagged subscribers.
- #22081's misalignment cannot happen here: each lease has one observer list
  and no parallel context array.
- An application that ignores non-`NOT_FOUND` errors and waits for
  `onCompleted` gets the Go semantics: loss is reported at the deadline.

**Other jetcd points.**
- **First-response deadline.**
  - A new entry's deadline is "now +" the gRPC channel's `keepaliveTimeout`,
    10 s by default (`:341-346`; `ClientBuilder.java:67`, `:589-591`). The
    5 s constant applies only if that setting is null.
  - If the first response does not arrive in time, the entry is reaped with
    `onCompleted`, whatever the lease's TTL.
  - That errs toward reporting loss early. It is not unsafe, but it couples
    lease handling to an HTTP/2 ping setting.
- **`keepAliveOnce` sends its request twice (minor).** — **Moderate**.
  - The stream handler writes the request (`:156-158`). On completion,
    `ref.get().end(req)` (`:171`) runs, and Vert.x `WriteStream.end(T)`
    writes its argument before ending the stream (recalled, not checked in
    the Vert.x source). So every `keepAliveOnce` renews twice.
  - That is harmless, because renewal is idempotent, but it doubles the load
    from this call.
  - If the stream never opened, `ref.get()` is null and the completion action
    throws. The future has already failed with the connection error in that
    case, so the caller sees the original error (Java `whenComplete`
    semantics).
- **Open report.** jetcd [#1383](https://github.com/etcd-io/jetcd/issues/1383)
  (open; jetcd 0.7.7, etcd 3.5.13) reports `onCompleted` firing after several
  minutes of keep-alive. I read only the report; it is not analyzed here.

### 5.4 Automatic retries of non-idempotent RPCs (Jepsen, jetcd 0.8.2)

Added 2026-11-09. Code facts are at `jetcd-0.8.7` and `v3.7.3` (**High**);
jetcd paths are under `jetcd-core/src/main/java/io/etcd/jetcd/`. The
consequences for lease and lock RPCs are traced from code, not run.

**What Jepsen found** ([jetcd 0.8.2](https://jepsen.io/analyses/jetcd-0.8.2),
2024-08-08).
- **The defect.** jetcd retried every unary call that failed with
  `UNAVAILABLE`, including writes already sent. With `kill -9` of servers as
  the only fault, one `Txn.commit()` went out twice, on separate
  connections, about 3 s apart, and both copies executed.
- **What applications saw:** lost update, cyclic information flow (Adya's
  G1c), a transaction that read its own future write, and aborted read
  (G1a).
- **Why guards do not help.** In the aborted-read case the first copy
  committed, the retry's guard then failed, and jetcd returned the retry's
  `isSucceeded() == false`. An indefinite failure was reported as a
  definite one.
- **Versions.** jetcd 0.6.0 (2021-12-15) through 0.8.2; 0.5.0 was also
  evaluated. Jepsen's workaround was `ClientBuilder.retryMaxAttempts(0)`.
- **History.** Jepsen reported the symptoms in 2022: etcd
  [#14092](https://github.com/etcd-io/etcd/issues/14092) and
  [#14890](https://github.com/etcd-io/etcd/issues/14890), jetcd
  [#1072](https://github.com/etcd-io/jetcd/issues/1072).
  - etcd closed #14092 "as a jetcd issue"; the stale bot closed jetcd
    #1072.
  - etcd closed #14890 on 2024-08-02, a maintainer agreeing to "reject
    Jepsen report as inaccurate" and betting the cause was Jepsen.
  - On the report's publication day, etcd opened
    [#18424](https://github.com/etcd-io/etcd/issues/18424), "etcd Go & Java
    client SDK's retry mechanism may break `Serializable`". The stale bot
    closed it *not planned* on 2025-06-01.

**The fix.** jetcd [#1444](https://github.com/etcd-io/jetcd/issues/1444) /
PR [#1445](https://github.com/etcd-io/jetcd/pull/1445), by this project's
user. Commit `05f92e0`, merged 2025-02-18; first release `jetcd-0.8.5`
(2025-03-03, `git tag --contains`). It splits the retry predicate in two
(`support/Errors.java:29-41`):
- `isRetryableForSafeRedoOp`: `UNAVAILABLE`, or an auth-token error;
- `isRetryableForNoSafeRedoOp`: auth-token errors only. The server rejects
  a request with such an error without applying it, so retrying is safe.

Defaults: 2 retries, with backoff from 500 ms to 2.5 s
(`ClientBuilder.java:62-64`).

**Which RPCs are retried after `UNAVAILABLE`.**

| Operation | jetcd 0.8.7 | Go client v3.7.3 | Safe to repeat? |
|---|---|---|---|
| `Put`, `Delete`, `Txn` | No, unless the caller sets `withAutoRetry()` (`impl/KVImpl.java:69`, `:101`, `:134`) | No, except on an error showing no connection was available (`client/v3/retry.go:85-94`) | No |
| `Get`; jetcd `compact` | Yes (`KVImpl.java:85`, `:120`) | `Range` yes (`retry.go:108`) | Yes |
| Lease `grant` | Yes (`impl/LeaseImpl.java:81-99`) | Yes (`retry.go:151`) | **No**: each copy creates a lease |
| Lease `revoke` | Yes (`LeaseImpl.java:103-110`) | Yes (`retry.go:155`) | The effect is; the reply is not |
| Lease `timeToLive` | Yes (`LeaseImpl.java:114-125`) | Yes (`retry.go:143`) | Yes |
| `Lock` (server Lock API) | Yes (`impl/LockImpl.java:61-72`) | — (`concurrency.Mutex` uses `Txn`) | Acquisition yes, by key reuse; see below |
| `Unlock` | Yes (`LockImpl.java:76-86`) | — (`Delete`, not retried) | The effect is, unless a delayed copy lands after a re-lock (#22082's pattern) |
| Election calls | No (`impl/ElectionImpl.java:88-91`, `:111-114`, `:126-129`, `:162-165`) | — (recipes use `Txn`) | — |

The Go client also retries any RPC that fails with a deadline or
cancellation status while the caller's own context is still live
(`client/v3/retry_interceptor.go:76-82`). The #18424 investigation
concluded that the server never returns such a status for a request it
executed, and could not reproduce the problem with the Go client.
— **Moderate** (their analysis). It is consistent with the server code: a
proposal's own timeout becomes `ErrTimeout` or one of its variants, all
`Unavailable` (`parseProposeCtxErr`, `server/etcdserver/server.go:2359-2389`),
and the Go client does not retry a write on `Unavailable` after connecting.

**Consequences for leases and locks** (inferred; not reproduced).
- **Orphan lease.** If a `LeaseGrant` executed and its reply was lost, the
  retry grants a second lease, which the caller receives. The first has no
  keys and no keep-alive, and expires after its TTL, or later if leader
  changes intervene (P2). Harmless except as a leak.
- **A revoke reported as failed.** If a `LeaseRevoke` executed and its reply
  was lost, the retry finds no lease and returns `NOT_FOUND`, "requested
  lease not found" (`server/lease/lessor.go:329-333`, mapped at
  `server/etcdserver/api/v3rpc/util.go:71`). The caller sees a failure for a revoke that
  succeeded.
- **jetcd `Lock`: the first attempt can delete the retry's key.**
  - *Why a retry is idempotent.* The server's Lock handler runs
    `concurrency.Mutex.Lock` with the call's context. The waiter key is
    `name/leaseID`, and a second call with the same lease finds that key
    and reuses its revision (`mutex.go:115-130`).
  - *Why the first attempt is still a hazard.* jetcd sees `UNAVAILABLE`
    when its connection fails, but the member's handler may still be
    waiting. It ends only when the member notices the dead connection,
    which with default settings can take about two hours (the server pings
    an idle connection after 2 h, `server/embed/config.go:70`). If it is
    cancelled while still waiting, its cleanup `m.Unlock(client.Ctx())`
    deletes `name/leaseID` (`mutex.go:89-93`, `:97-101`), the key the
    retry reused.
  - *If the retry is still waiting,* it later finds its key gone and returns
    `ErrSessionExpired` (`mutex.go:103-105`), although the session is
    alive: a spurious failure.
  - *If the retry has already been granted,* the lock key of a client that
    believes it holds the lock is deleted, and the next waiter is granted
    the lock. That needs the first handler to be cancelled after the
    previous holder released, but before it processed the release: a
    narrow window. A fenced holder is protected, because its guarded
    writes then fail.

  — **Low** (likelihood; inferred, not reproduced; not reported upstream).
  The Go recipe is not exposed: it does not retry its `Txn` calls once a
  connection was established, and its cleanup runs in the caller's own
  process. Jepsen's harness already guarded against the general problem:
  after a failed acquisition it revokes the lease, because "our lock request
  might still be outstanding, and could complete later"
  (`jepsen-io/etcd` @ `16149cb`, `src/jepsen/etcd/lock.clj:44-56`).

**What applications should do.** — **Moderate** (derived from code and the
report; untested).
- jetcd: use 0.8.5 or later, and set `withAutoRetry()` only on writes that
  are safe to repeat. On 0.6.0–0.8.3, set `retryMaxAttempts(0)`.
- Both clients: treat a write error that is not a definite rejection (an
  auth, permission or validation error) as "outcome unknown", and resolve
  it by reading. A failed compare is not an error: it is a definite answer
  (`Succeeded == false`), but only for the attempt that returned it. Writing a unique value or request ID makes the
  read conclusive; Jepsen gives "committing only if a unique ID for that
  operation has not already been applied" as an idempotent form.
- Treat `NOT_FOUND` from a revoke as success.
- jetcd `Lock`: if a call fails or was retried, either revoke the lease
  (which deletes every waiter key it owns) or confirm ownership with a
  fenced write before acting.
- etcd keeps no table of executed client requests, the remedy the Raft
  paper (§8) prescribes for duplicate commands, as a commenter on #18424
  pointed out. Deduplication therefore stays with the client.

## 6. Performance structure of renewal and expiry

Everything here is structure read from code at `v3.7.3`, plus field reports
quoted as such. Nothing was measured: this host is not the benchmark
environment (AGENTS.md §7).

### 6.1 Renewal

- **On the leader, a renewal is a map and heap update in memory**
  (`lessor.go:442-452`), with no Raft entry, except for checkpoint clears
  (§1.4). `ensureLeadership` costs nothing while ticks are fresh (#18428).
  Before that shortcut, it cost about 20 % leader CPU at 1500 renewals/s
  (#18069, reporter's numbers).
- **`FastLeaseKeepAlive`** (Beta, on by default) lets the leader skip the
  apply-index wait for leases it knows (`v3_server.go:512-523`). That stopped
  apply backlog from blocking keepalives (#18100, #20589).
- **Forwarded renewals do not get it.**

  | Step | Behaviour | Evidence |
  |---|---|---|
  | Follower | Forwards each renewal over peer HTTP | `v3_server.go:538-560` |
  | Leader's peer handler | Waits for its apply index to reach its commit index, with a 1 s timeout; answers HTTP 408 on timeout | `leasehttp/http.go:37`, `:84-89` |
  | Follower on error | Retries every 50 ms until `ReqTimeout` (7 s), then fails the keep-alive stream with `ErrTimeout` | `v3_server.go:541-572` |

  - When the leader's apply backlog exceeds 1 s, renewals through followers
    fail, while renewals sent to the leader succeed.
  - With clients spread evenly over N members, about (N−1)/N of keep-alive
    streams take the forwarded path (inferred).
  - So a backlog lasting longer than the TTL expires leases whose clients
    renew on schedule through followers. That is a loss of availability,
    not a P3 violation, because the failed renewals were never acknowledged.
  - It is also one of the "server delayed the renewals" routes into R5
    (§3.4).

  — **High** (code); impact **Moderate**: it depends on how often backlogs
  exceed 1 s, which was not measured.
- **Forwarded renewals also add apply waiters.** Each forwarded wait
  registers an `applyWait` waiter on the leader (`server.go:632-634`,
  `:646`).
  - The open [#22450](https://github.com/etcd-io/etcd/issues/22450) (v3.5.21,
    v3.6.6) reports a leader apply backlog that never recovers. The
    reporter's reproduction load included lease keepalives and linearizable
    reads, which grow `applyWait` waiters.
  - I have not checked this on v3.7.3, where keepalives sent to the leader
    no longer wait. Its only relevance here: forwarded keepalives still add
    waiters.
- **No renewal load control** (§4.2): the TTL is fixed, clients renew every
  TTL/3, and the TODO at `lessor.go:290-291` is unimplemented.

### 6.2 Expiry

- **One Raft entry per expired lease.**
  - `revokeExpiredLeases` proposes one `LeaseRevoke` per lease, with at most
    16 in flight (`server.go:874-900`; `maxPendingRevokes`, `:103`).
  - Checkpoints, by contrast, batch up to 1000 leases per entry
    (`lessor.go:55`, `:674`).
  - The lessor hands over at most `leaseRevokeRate/2` = 500 leases per
    500 ms scan (`lessor.go:644`). That is a rate cap, not a throughput
    guarantee.
- **Each revocation deletes all attached keys in one backend transaction**
  (`lessor.go:343-361`). Kubernetes hit this cost with shared Event leases
  (#96836, §7.2).
- **Back-pressure delays expiry; it does not drop it.**

  | Condition | Behaviour | Evidence |
  |---|---|---|
  | Server loop busy | The batch hand-off is skipped and retried at the next scan | `lessor.go:652-660` |
  | Lease handed over but not revoked | Re-examined every 3 s (`defaultExpiredleaseRetryInterval`) | `lessor.go:58`; re-registered at `:721` |
  | Apply backlog above 5000 entries | Revocations are ordinary proposals, so they are rejected with `ErrTooManyRequests` like any write | `v3_server.go:51`; check at `:1063-1064` |
  | Same, with the Alpha `PriorityRequest` gate (off by default) | `LeaseRevoke` is admitted up to a 10000-entry gap | `etcd_features.go:98`; `server/etcdserver/util.go:60-74`; `v3_server.go:53` |

  So expired keys linger exactly when the cluster is overloaded (inferred).
- **With auth on, every revocation creates a root token.**
  - Each revocation goroutine calls `authStore.WithRoot` (`server.go:884`),
    which assigns a fresh token for `root` on every call
    (`server/auth/store.go:1159-1188`). With JWT tokens, that is one
    signature per expired lease.
  - [#21735](https://github.com/etcd-io/etcd/issues/21735) reports one
    production data point. Caching the root JWT on this path cut hot-member
    CPU by about 72 % on average, at about 280–630 revocations/s.
  - The reporter's own caveat: "a rollout observation in a real workload,
    not a strict benchmark". The reporter closed the issue as not planned two
    days after opening it.

  — **High** for the per-lease call; the magnitude is the reporter's.

## 7. Kubernetes

### 7.1 What Kubernetes uses etcd leases for

Verified at `master` on 2026-10-31, except where tagged `v1.37.2`.

| Use | How | Evidence |
|---|---|---|
| TTL'd resources | kube-apiserver attaches an etcd lease only when the resource has a TTL, obtained from a `leaseManager` | `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go:322`, `:532` |
| Lease reuse | One lease is reused for a time window: reuse = min(5 % of TTL, 60 s), at most 1000 objects per lease; the manager grants TTL + reuse | `staging/.../etcd3/lease_manager.go:29-30`, `:90-131` |
| Events | The resource that sets a TTL; default TTL 1 h per #65497 | `pkg/registry/core/event/storage/storage.go:42` |
| API-server endpoint keys ("master lease") | The endpoint reconciler writes TTL'd keys and refreshes them by *rewriting* the key with a new TTL (comment: "we can't refresh the TTL without actually changing the object") | `pkg/controlplane/reconcilers/lease.go:96-120` |
| Leader election and node heartbeats | `client-go` `LeaseLock` and `kube-node-lease` use `coordination.k8s.io` Lease **objects**: ordinary keys, updated with resourceVersion compare-and-swap and client-side renew times. The Lease storage sets no `TTLFunc`, and `LeaseLock` only gets, creates and updates the object through the API | @ `v1.37.2`: `pkg/registry/coordination/lease/storage/storage.go:36-58`; `staging/src/k8s.io/client-go/tools/leaderelection/resourcelock/leaselock.go:42`, `:57`, `:73` |

- **No `KeepAlive`.** None of these files calls `KeepAlive`: there is no
  occurrence in `lease_manager.go`, `store.go` or `reconcilers/lease.go`. —
  **High** for those files; **Moderate** for "kube-apiserver never keeps an
  etcd lease alive", since I did not search the whole repository.
- **Lease objects are ordinary keys, not etcd leases.** — **High**.
- **That is no verdict on etcd leases.** Kubernetes components reach cluster
  state only through the API server, so any coordination primitive had to be
  an API object. The earlier Endpoints/ConfigMap locks were API objects too
  (recalled). — **Moderate**.

### 7.2 Consequences

- **The failover-safety class (P3, §3) largely does not apply** to
  kube-apiserver. — **Moderate** (reasoned).
  - A never-renewed lease has no live client whose renewals a stale leader
    could miss.
  - A stale-leader revocation fires at roughly the original TTL, which is
    the intended expiry.
  - Master-lease keys are rewritten with fresh leases, so revoking an old
    lease detaches nothing current.
  - Components that use etcd directly with `KeepAlive` remain exposed; I did
    not survey them.
- **TTL extension (P2) applies directly.** — **High**.
  - Kubernetes #65497 (2018): "if leader elections occur more often than the
    event-ttl … event leases will be renewed indefinitely and the number of
    events stored in etcd … will grow". It drove etcd #9888 and
    checkpointing.
  - Current kubeadm does **not** enable checkpointing: its default etcd flags
    set only `--feature-gates=InitialCorruptCheck=true`
    (`cmd/kubeadm/app/phases/etcd/local.go:265`). So kubeadm clusters on etcd
    v3.7.3 still reset Event TTLs on every etcd leader change.
  - A Red Hat OpenShift RFE
    ([ETCD-311](https://issues.redhat.com/browse/ETCD-311)) reportedly
    proposes enabling checkpointing for this reason. Known only from a
    search-result snippet; the page did not load. Unverified.
- **Revocation cost.** A sequence of fixes:
  1. One lease per Event created too many leases (#47532, 2017).
  2. Fix: reuse leases per time window (PR #64539, 2018).
  3. Reused leases with many keys made one `LeaseRevoke` a huge transaction,
     which blocked etcd long enough to fail health checks ("event etcd
     restarts", #96836, 2020).
  4. Fix: a cap of 1000 objects per lease.

  On the etcd side, v3.7 adds an Alpha `PriorityRequest` gate to prioritize
  `LeaseRevoke` under overload (#20492, `etcd_features.go:98`).
- **An open watch-correctness bug caused by lease revocation**
  ([#110210](https://github.com/kubernetes/kubernetes/issues/110210); open
  since 2022-05; `priority/important-soon`, `lifecycle/frozen`).
  - Revoking a shared lease deletes all its keys in one transaction, hence at
    one revision (`lessor.go:345-361`).
  - A Kubernetes watch that breaks mid-transaction resumes "from the last
    received RV", and can miss the remaining DELETE events.
  - An etcd maintainer noted that etcd's watch API can signal fragmented
    responses, and that the information is lost in the API server.
  - Only Events are exposed. This is a mismatch between lease semantics and
    the consumer, not a failover bug.
- **Restore interactions** (#86812): stale master-lease endpoints after an
  etcd restore. Labelled not-reproducible/rotten; not analyzed here.

## 8. Test coverage

| Suite | What it does with leases | Evidence |
|---|---|---|
| Robustness (linearizability) traffic | Grants leases with `DefaultLeaseTTL = 7200` s and never sends keepalives (no `KeepAlive` in `tests/robustness/traffic/` or `tests/robustness/client/`) | `tests/robustness/traffic/traffic.go:50` |
| Robustness model | Has `LeaseGrant` and `LeaseRevoke`, but no expiry | `tests/robustness/model/deterministic.go:146-149`, `:238-268` |
| Antithesis | Reuses the same traffic | `tests/antithesis/test-template/robustness/traffic/main.go:60` |
| `TestLeaseRevoke_IgnoreOldLeader`, `TestLeaseRevoke_ClientSwitchToOtherMember` | #15247 Cases 1 and 2, each on 3 and 5 members | `tests/e2e/v3_lease_no_proxy_test.go:36`, `:48` |
| `TestLeaseRevokeDuringRenew` | #20590, using the `beforeCheckpointInLeaseRenew` failpoint | `tests/e2e/v3_lease_no_proxy_test.go:169` |
| `TestV3LeaseCheckpoint` | Asserts that the TTL *is* reset when checkpointing is off | `tests/integration/v3_lease_test.go:506` (§1.4) |

**Findings.**
- **The correctness harness checks no lease property.** None of P2, P3 or P4
  is checked by the harness etcd uses to validate correctness under
  failpoints. — **High**.
- **No test covers R5 or the partition variant of R1.**
  - `revokeExpiredLeases` has no failpoint between its leadership check and
    its proposals (`server.go:858-903`), so testing R5 needs one added in a
    worktree (§10).
  - Adding leases to the Kubernetes traffic profile
    ([#20621](https://github.com/etcd-io/etcd/issues/20621)) was closed *not
    planned*.
- **jetcd's tests do not exercise v3.7.3.** `jetcd-launcher` defaults to etcd
  v3.5.14 (`jetcd-launcher/src/main/java/io/etcd/jetcd/launcher/Etcd.java:31`).
  - That server has the #17425 safeguard (v3.5.13) in its original form: a
    full ReadIndex on every renewal. The #18439 shortcut arrived in v3.5.16.
  - It lacks the renew re-check (v3.5.23).
  - So jetcd's own tests do not exercise v3.7.3's lease paths, and its
    `LeaseUnitTest` class is disabled (§5.3).
- **Jepsen** (etcd 3.4.3, 2020) tested locks, not lease timing during
  failover. Its findings bear on P5:
  - "etcd locks (like all distributed locks) do not provide mutual
    exclusion. Multiple processes can hold an etcd lock concurrently, even in
    healthy clusters with perfectly synchronized clocks". The healthy-cluster
    failures used 1–3 s TTLs; the ~18 % lost updates used 2 s TTLs and
    process pauses (§2.3). *Corrected 2026-11-09:* this bullet earlier
    attached the pauses to the healthy-cluster quote.
  - It recommends fencing with the lock key's revision.
  - It found a lock-after-lease-loss bug (#11456), fixed in v3.5.0 and not
    in 3.4 (§2.3).
  - Its faults included leader-targeted crashes and pauses, partitions,
    clock skew and membership changes, but its checks were for mutual
    exclusion, not for P2 or P3, so it says nothing about the failover
    windows of §3.
- **Jepsen** (jetcd 0.8.2, 2024) tested jetcd against etcd, with `kill -9`
  as the only fault, using list-append and read-write register workloads.
  It found the retry defect of §5.4 and no server anomaly; it did not test
  leases. Its harness at `fbe693e` defaults to etcd 3.5.15; the report does
  not name the version.

## 9. Where these conclusions could fail

| Conclusion | It fails if… | Current evidence |
|---|---|---|
| §3.2 mechanism; R1, R2 | Forwarded proposals from a deposed leader are rejected somewhere | Raft `v3.7.0` forwards them (`raft.go:1718-1729`), etcd adds no term check, and #12528 and #15247 observed such commits |
| R1 | `isActive()` is updated by something other than the Raft loop's tick | Verified: only `raft.go:159-162` |
| R1, partition variant | Real time and Raft ticks diverge substantially on the old leader or the followers, i.e. a stall (the #15247 regime, not a partition) | Also assumes etcd's default `PreVote` + `CheckQuorum` configuration (`bootstrap.go:540-551`) |
| R5 | Something between `revokeExpiredLeases` and apply cancels or rejects a proposal whose decision predates a leader change | None found: `GoAttach` workers stop only on shutdown, `LeaseRevoke` checks nothing, and apply revokes unconditionally (§3.4) |
| P2 "by default" | The cluster enables `LeaseCheckpoint` | Even then it holds for leases with TTL ≤ 5 min, and for all leases while leader changes come faster than the checkpoint interval |
| Kubernetes (§7) | Kubernetes uses etcd leases for more than TTL'd resources | A component that calls `KeepAlive` against the Kubernetes etcd would be exposed to §3 |
| jetcd consequence (§5.3) | Applications follow jetcd's de facto contract (only `NOT_FOUND` and `onCompleted` are terminal) | Then the #22094-like end state does not arise |
| jetcd `Lock` retry race (§5.4) | The member always ends the first attempt's handler, and its cleanup commits, before the retry's acquisition reaches the cluster | Plausible for a reset connection, which the member notices at once, if the cleanup commits within jetcd's 500 ms backoff. Not for a silently dead connection, which the member notices only through a keepalive ping after 2 h idle (`server/embed/config.go:70`) |

## 10. Open questions and next steps

1. **Reproduce R1/R2** on v3.7.3 in a worktree, with gofail. This would turn
   "Moderate" into a fact either way. It requires Go or a container build
   (AGENTS.md §11).
   - Start from the #15247 regression tests
     (`tests/e2e/v3_lease_no_proxy_test.go:36`, `:48`), which stall the
     leader with `raftBeforeSave` `sleep("30s")` (`:127`).
   - Make 17 or more leases expire on the stalled leader while clients renew
     at the new leader.
   - Repeat many times, and check for wrong revocations after the stall ends.
2. **Reproduce R5.** It needs a new failpoint in a worktree, placed in
   `revokeExpiredLeases` after `ensureLeadership()` and before `LeaseRevoke`
   (`server.go:867-885`); none exists. The test:
   - keep checkpointing off, and keep the old leader running;
   - let a lease expire on leader A, and hold its revoke worker at the
     failpoint;
   - transfer leadership to B, and renew the lease at B (it should succeed);
   - release the worker;
   - check whether the revocation commits and the key disappears within TTL
     of B's acknowledgement.

   Keep this separate from the stall and crash cases.
3. **Quantify P2** under a given leader-change rate (e.g. one change every N
   seconds), with and without `LeaseCheckpoint`, to bound how long Events
   survive. This is a functional measurement, so it is acceptable on this
   host.
   - Include a rate faster than the checkpoint interval: §1.4 predicts that
     checkpointing then stops helping.
   - The interval must be set programmatically, e.g. by extending
     `TestV3LeaseCheckpoint` (`tests/integration/v3_lease_test.go:506`).
   - Also test leases shorter than the interval, and an all-member restart.
4. **Evaluate term fencing.** What would a `term` field on expiry-driven
   `LeaseRevoke` break: mixed-version clusters, WAL replay, `etcdutl` tools?
   This is the structural fix, and the natural design proposal for this
   project.
5. **Reproduce the #22094 consequence (§5.2.3).** Use a Go client in a
   container, a 3-member cluster, and `KeepAlive(WithRequireLeader(ctx), id)`
   as the lease's only subscriber. Partition the serving member until
   `ErrNoLeader`, then heal. Expected:
   - the channel closes;
   - keep-alive requests for the lease continue every 500 ms;
   - `TimeToLive` keeps showing a refreshed TTL;
   - a lock key attached to the lease is never released.
6. **jetcd on leader change (§5.3).** What is already known:
   - jetcd has no per-subscriber require-leader handling, so #22081 has no
     equivalent;
   - #22094's end state is reached by design whenever an application treats
     `onError` as terminal.

   Still open:
   - reproduce it against v3.7.3, not the launcher's default v3.5.14;
   - check whether jetcd's own lock and election clients, or common
     frameworks built on jetcd, treat `onError` as terminal.
7. **Forwarded keepalives under apply backlog (§6.1).** Build a 3-member
   cluster on v3.7.3, and inject an apply delay on the leader longer than
   1 s, e.g. with an apply-path failpoint. Compare keepalive success for
   clients connected to the leader with clients connected to a follower.
   This is a functional test (success or failure, not latency), so it is
   acceptable on this host.
8. **Split into per-bug files** (AGENTS.md §5) if any item is pursued:
   - `leases-bug-stale-leader-revoke.md` (#12528/#15247 + R1/R2/R5);
   - `leases-bug-checkpoint-error-swallowed.md` (#22649);
   - `leases-design-ttl-reset-on-failover.md` (P2 + Kubernetes #65497);
   - `leases-bug-jetcd-onerror-keeps-renewing.md` (§5.3).

## 11. Conclusions

### 11.1 On the design hypothesis

The suspicion of a design mistake is justified, but "renewals bypass Raft"
is not it. Chubby also keeps session timers out of its replicated log, and
also extends leases on fail-over (§4.1). The mistake is narrower, and has two
parts (§4.3):

1. **Unfenced revocations.** Expiry-driven revocations are decided from one
   member's local timer, but committed as commands with no term. So a
   decision made under old authority can take effect in a later term (Case
   1, R1, R2, R5). The repair is to carry the deciding term in the revocation
   and check it at apply. It was proposed upstream in #15247, and not
   adopted.
2. **Authority measured in ticks, expiry in real time.** A leader's
   authority is measured in Raft ticks, which stop when the leader stalls,
   while lease expiry runs on real time (Case 2, R3). The repair is a
   real-time bound on authority, like Chubby's master lease. etcd
   approximates it with a ReadIndex when ticks look stalled, but the
   follower-forwarded renewal path has no check at all.

— **Moderate**.

### 11.2 On production readiness at v3.7.3

- **The replicated KV state is safe (P1).** — **High**.
- **Lease-based coordination is safe only with fencing.** — **Moderate**.
  - With fencing, the remaining server windows (R1–R5) cannot corrupt etcd
    writes.
  - Without it, they can produce dual ownership. Each window needs a stall,
    or a late client coinciding with a leader change, plus a narrow
    interleaving.
- **TTL is not an upper bound.** — **High**.
  - Anything that relies on timely expiry is delayed by every leader change,
    and can be postponed indefinitely. That includes failure detection and
    garbage collection such as Kubernetes Events.
  - Checkpointing does not fix this for short leases or frequent leader
    changes. Enabling it adds the #22649 risk until PR #22650 lands.
- **Client libraries are the weakest layer.** Two open Go client bugs
  (#22081, #22094) and jetcd's `onError` contract can leave a lease renewed
  forever after the application believes it is gone. Both clients' deadlines
  are optimistic. — **High** (code).
- **Retries are fixed where Jepsen found them, not everywhere.** jetcd no
  longer retries writes by default (0.8.5). Both clients still retry lease
  grant and revoke, and jetcd retries `Lock`, which opens a narrow race
  with the first attempt's cleanup (§5.4). — **High** (code); **Low** for
  the race's likelihood.
- **Maintenance signals are weak for lease timing.** — **High** for the
  facts; the interpretation is my inference.
  - The failover bug took about three years to fix after its first
    reproduction, and a stale bot closed it once.
  - The structural fix was not adopted.
  - Both lease redesign trackers were closed by the project's stale bot,
    whose exemption for tracked issues was not being applied, and neither
    has been reopened (§3.7).
  - Checkpointing has been experimental or Alpha since v3.4.0 (2019).
  - The main correctness harness does not exercise expiry or keepalive.

  Together these suggest that lease timing gets less maintenance and testing
  than the KV path.

### 11.3 Practical guidance

These recommendations follow from the code facts above; none was tested. —
**Moderate**.
- **Applications.**
  - Fence every protected write with the role key's create revision
    (`Mutex.IsOwner`, or `CreateRevision(key) == rev` in the transaction).
  - Do not gate external side effects on `Session.Done()` alone (§2.3,
    §5.1).
- **Go client users.** Avoid passing `WithRequireLeader` contexts to
  `KeepAlive` until #22081 and #22094 are fixed (§5.2).
  - The cost: leader loss is then noticed only at the local deadline.
  - `concurrency.Session` is unaffected by default.
- **jetcd users.**
  - Treat any `onError` other than `NOT_FOUND` as transient.
  - When giving up a lease, call `close()` on the keep-alive and revoke the
    lease explicitly (§5.3).
  - Use jetcd 0.8.5 or later, and do not set `withAutoRetry()` on writes
    that are not safe to repeat (§5.4).
  - After a failed or retried `Lock`, revoke the lease or prove ownership
    with a fenced write before acting (§5.4).
- **All client users.** Treat a write that fails with a timeout or
  `UNAVAILABLE` as "outcome unknown", and a `NOT_FOUND` from revoke as
  success (§5.4).
- **Operators.**
  - Do not rely on lease TTL for bounded cleanup under leader churn.
  - Leave `LeaseCheckpoint` off, unless retaining long leases matters more
    than the #22649 risk.
  - Expect keepalives through followers to fail first under apply backlog.
  - With JWT auth, expect per-revocation signing cost during expiry waves
    (§1.4, §3.5, §6).

### 11.4 What would change these conclusions

- The reproductions in §10, mainly R1/R2, R5, #22094 and forwarded
  keepalives under backlog.
- The failure conditions in §9.

A reproduction of R1 or R5 would raise the residual-risk findings from
Moderate to High. A failure to reproduce them after targeted failpoints
would lower them.

---

## Appendix A. Issue survey (etcd-io/etcd)

**Method.**
- **Original search (2026-10-31):** issues with "lease" in the title (5
  open, 146 closed), plus targeted searches.
- **Title-or-body search (2026-11-01):**
  `repo:etcd-io/etcd is:issue is:open lease` returns 27 open issues. The
  extra hits I examined are mostly client-recipe and `client/v3/leasing`
  cache issues. I did not classify all 27.
- **Legend.**
  - † = row added on 2026-11-01, with its state checked that day through
    the GitHub API.
  - ‡ = row added on 2026-11-09 from the Jepsen reports, with its state
    checked that day.
  - "Read": "body" means I read the issue text (and comments where noted);
    otherwise, title only.

| Issue | Title (abridged) | State | Class | Failover-related | Read |
|---|---|---|---|---|---|
| [#15247](https://github.com/etcd-io/etcd/issues/15247) | All leases revoked when leader stuck (fdatasync/CPU) | closed, fixed (#16822) | P3 safety | **Yes** | body + 44 comments |
| [#12528](https://github.com/etcd-io/etcd/issues/12528) | Old leader still revokes lease after stepping down (high CPU) | closed by stale bot, unfixed then | P3 safety | **Yes** | body |
| [#13915](https://github.com/etcd-io/etcd/issues/13915) | Make leases linearizable (excluding TTL) | closed not planned by the stale bot (twice; reopened once by a maintainer; exemption ignored, §3.7) | P4 / design | Yes (proxying to leader outside Raft) | body + comments |
| [#9888](https://github.com/etcd-io/etcd/issues/9888) | Leases may be auto-renewed indefinitely due to leader elections | closed (checkpointing) | P2 | **Yes** | body |
| [#9395](https://github.com/etcd-io/etcd/issues/9395) | TTL resetting on leader elections or restart, undocumented | closed | P2 | **Yes** | body |
| [#17132](https://github.com/etcd-io/etcd/issues/17132) | Checkpointing not working as expected (3rd leader change) | closed (needs persist flag) | P2 | **Yes** | body + comments |
| [#20273](https://github.com/etcd-io/etcd/issues/20273) | Lease not expired when node restarts | closed not planned | P2 (by design) | Restart | body |
| [#13776](https://github.com/etcd-io/etcd/issues/13776) | Track stabilization of lease checkpointing | closed not planned by the stale bot (exemption ignored, §3.7) | P2 process | Yes | body |
| [#22649](https://github.com/etcd-io/etcd/issues/22649) | Checkpointer discards raftRequest error | **open** | P3 (gate on) | Yes (likely during leader change) | body |
| [#18069](https://github.com/etcd-io/etcd/issues/18069) | +20 % leader CPU from leadership check in renew | closed (#18428) | perf vs P3 | Yes (weakened safeguard) | body + PR review |
| [#18100](https://github.com/etcd-io/etcd/issues/18100) | Make linearizable renew configurable (apply stalls block keepalives) | closed → `FastLeaseKeepAlive` (Beta, default on in v3.7, `etcd_features.go:97`) | perf / availability | Indirect | body |
| [#14758](https://github.com/etcd-io/etcd/issues/14758) | Attached keys deleted before lease expired | closed (not reproduced); race later reproduced in #21389 (unmerged) | P3 / P4 | No | body + comments |
| [#21638](https://github.com/etcd-io/etcd/issues/21638) | KeepAlive channel yields buffered pre-revoke success | **open** (doc note on `main`, `a81b6d623`; not in v3.7.3) | client | No | body |
| [#22081](https://github.com/etcd-io/etcd/issues/22081) | KeepAlive subscriber gets wrong channel closed after leader loss | **open** | client | Leader loss, client-side | body + code at v3.7.3 (§5.2.2) |
| [#22094](https://github.com/etcd-io/etcd/issues/22094) | `closeRequireLeader` leaves orphaned keepAlive entry | **open** | client | Leader loss, client-side | body + code at v3.7.3 (§5.2.3) |
| (none filed) | gRPC proxy ignores `WithRequireLeader` on lease keep-alive streams (same bug as the watch proxy) | not reported | proxy | Leader loss, client-side | code (§5.2.4); watch variant reproduced (watchers-claude.md Appendix C) |
| [#22082](https://github.com/etcd-io/etcd/issues/22082) | Stale `Mutex.Unlock` retry can delete a newer same-session lock | **open** | P5 client | No | body |
| [#14019](https://github.com/etcd-io/etcd/issues/14019) | Etcd loses track of leases (listed but "not found") | closed by stale bot | P4 | Unknown | body |
| [#14096](https://github.com/etcd-io/etcd/issues/14096) | Leases expire on adding cluster members | closed | P3? | Membership change | body (resolution not read) |
| [#8382](https://github.com/etcd-io/etcd/issues/8382) | Lease expires when cluster reorganized (2017) | closed | P3? | Membership change | body (resolution not read) |
| [#9360](https://github.com/etcd-io/etcd/issues/9360), [#9496](https://github.com/etcd-io/etcd/issues/9496) | Poor performance with many expiring leases; promotion exceeding timeouts with many leases | closed (stale for #9496) | perf | #9496 yes (Promote cost) | title |
| [#15993](https://github.com/etcd-io/etcd/issues/15993), [#16053](https://github.com/etcd-io/etcd/issues/16053) | Put-with-lease linear in attached keys; `itemMap` leak | closed; #16053 not planned | perf / memory | No | title |
| [#12291](https://github.com/etcd-io/etcd/issues/12291) | Lease expiry based on wall clock | closed | clocks | No | title |
| † [#13675](https://github.com/etcd-io/etcd/issues/13675) | `concurrency.Session` closed immediately after creation (keepalive saw an unapplied grant) | closed, fixed in v3.5.3 (#13690) | P3-like false "not found" | No | body + maintainer diagnosis |
| † [#13294](https://github.com/etcd-io/etcd/issues/13294) | Why the new leader restarts TTL counting | closed (2022-01) | P2 | **Yes** | body |
| † [#12594](https://github.com/etcd-io/etcd/issues/12594) | Give increased lease timeout if etcd is under load | closed by stale bot (2021-04) | perf / design (§4.1) | No | title |
| † [#21735](https://github.com/etcd-io/etcd/issues/21735) | Root JWT signing cost in expired-lease revoke | closed *not planned* by its reporter, 2 days after opening (2026-05) | perf (§6.2) | No | body |
| † [#22450](https://github.com/etcd-io/etcd/issues/22450) | Leader apply backlog fails to recover after a transient stall (v3.5.21, v3.6.6) | **open** | perf / availability; keepalive load among the reporter's inputs | Indirect | body (searched for lease mentions) |
| † [#19597](https://github.com/etcd-io/etcd/issues/19597) | `Mutex.Lock` keeps waiting after its own session expired | **open** | client recipe, progress | No | title; the code's own TODO confirms it (`client/v3/concurrency/mutex.go:88`) |
| † [#22382](https://github.com/etcd-io/etcd/issues/22382) | `concurrency.ResumeMutex` | **open** | feature request | No | title |
| † [#22083](https://github.com/etcd-io/etcd/issues/22083), [#22084](https://github.com/etcd-io/etcd/issues/22084), [#22417](https://github.com/etcd-io/etcd/issues/22417) | `client/v3/leasing` (client-side lease-backed cache): protocol keys deletable, data race, stale count | **open** | cache, not KV leases | No | titles |
| † [#22209](https://github.com/etcd-io/etcd/issues/22209) | `keys_only` range no longer returns the `lease` field (v3.7 regression) | **open** | API metadata | No | title |
| ‡ [#11456](https://github.com/etcd-io/etcd/issues/11456) | Locks return without checking whether the lease is still held (Jepsen) | closed, fixed by `b14c7cd44` (v3.5.0; not in 3.4) | P5 client recipe | No | body + fix commit (§2.3) |
| ‡ [#11457](https://github.com/etcd-io/etcd/issues/11457) | Document that locks aren't really locks (Jepsen) | closed by PR #11490 (2020-03-05) | P5 docs | No | timeline; current docs read (§2.3) |
| ‡ [#18424](https://github.com/etcd-io/etcd/issues/18424) | Go and Java clients' retries may break `Serializable` (after Jepsen jetcd 0.8.2) | closed *not planned* by the stale bot (2025-06-01) | client retries | No | body + comments (§5.4) |

Out of scope but easily confused:
[#10082](https://github.com/etcd-io/etcd/issues/10082) and
[#12704](https://github.com/etcd-io/etcd/issues/12704) concern Raft's
*leader lease* for reads, not KV leases.

## Appendix B. Upstream references

Most relevant first.

**Failover safety (P3)**
- etcd [#15247](https://github.com/etcd-io/etcd/issues/15247): all leases revoked when the leader stalls (closed 2024-02-23).
  - Fix [#16822](https://github.com/etcd-io/etcd/pull/16822) (main, 2023-12-15).
  - Backports [#17425](https://github.com/etcd-io/etcd/pull/17425) (v3.5.13) and [#17465](https://github.com/etcd-io/etcd/pull/17465) (v3.4.31).
- etcd [#12528](https://github.com/etcd-io/etcd/issues/12528): the same bug reported in 2020 on v3.4.6.
  - Closed by the stale bot 2021-06-14, unfixed.
  - Fix PR [#12531](https://github.com/etcd-io/etcd/pull/12531) closed unmerged 2021-09-16.
- etcd [#18069](https://github.com/etcd-io/etcd/issues/18069) / PR [#18428](https://github.com/etcd-io/etcd/pull/18428): leadership check made conditional for performance; a maintainer objected on safety.
- etcd [#14758](https://github.com/etcd-io/etcd/issues/14758) / PR [#21389](https://github.com/etcd-io/etcd/pull/21389) (closed unmerged 2026-06-19): renew/revoke race. PR [#20590](https://github.com/etcd-io/etcd/pull/20590) (2025-09-05) is a partial fix.
- etcd [#22649](https://github.com/etcd-io/etcd/issues/22649) (open): checkpointer swallows Raft errors. Fix PR [#22650](https://github.com/etcd-io/etcd/pull/22650) is open and unmerged.
- etcd [#13675](https://github.com/etcd-io/etcd/issues/13675) / PR [#13690](https://github.com/etcd-io/etcd/pull/13690) (v3.5.3): keepalive reported a just-granted lease as not found. Refined by `FastLeaseKeepAlive`, [#18100](https://github.com/etcd-io/etcd/issues/18100) / PR [#20589](https://github.com/etcd-io/etcd/pull/20589).

**Linearizability and TTL reset (P2, P4)**
- etcd [#13915](https://github.com/etcd-io/etcd/issues/13915): "Make leases linearizable (excluding TTL)". Closed *not planned* by the stale bot 2025-09-11, reopened by a maintainer 2025-09-23, closed again by the bot 2025-12-19; its `stage/tracked` exemption was not being applied at either closure (§3.7).
- etcd [#9888](https://github.com/etcd-io/etcd/issues/9888), [#9395](https://github.com/etcd-io/etcd/issues/9395), [#17132](https://github.com/etcd-io/etcd/issues/17132), [#20273](https://github.com/etcd-io/etcd/issues/20273): TTL reset on leader change or restart.
  - Checkpointing: [#9924](https://github.com/etcd-io/etcd/pull/9924), [#13508](https://github.com/etcd-io/etcd/pull/13508).
  - Stabilization tracker [#13776](https://github.com/etcd-io/etcd/issues/13776), closed *not planned* by the stale bot 2025-09-30, while its `stage/tracked` and `help wanted` exemptions were not being applied (§3.7).

**Clients**
- etcd [#22081](https://github.com/etcd-io/etcd/issues/22081), [#22094](https://github.com/etcd-io/etcd/issues/22094) (both open): Go lease client `closeRequireLeader` bugs (§5.2). Fix PRs [#22087](https://github.com/etcd-io/etcd/pull/22087), [#22253](https://github.com/etcd-io/etcd/pull/22253), [#22291](https://github.com/etcd-io/etcd/pull/22291), all open.
- etcd PR [#21122](https://github.com/etcd-io/etcd/pull/21122) (v3.7.0 only): keep-alive stream cancellation is no longer always reported as "no leader".
- etcd PR [#5332](https://github.com/etcd-io/etcd/pull/5332) (v3.0.0) and commit `8024a0d15` (v3.2.0): origin of `WithRequireLeader` stream cancellation, and its lease-client support.
- jetcd [#1322](https://github.com/etcd-io/jetcd/issues/1322) (commit `10b07f5`, jetcd 0.8.0) and [#1383](https://github.com/etcd-io/jetcd/issues/1383) (open): keep-alive error reporting (§5.3).
- jetcd [#1444](https://github.com/etcd-io/jetcd/issues/1444) / PR [#1445](https://github.com/etcd-io/jetcd/pull/1445) (commit `05f92e0`, merged 2025-02-18, first in jetcd 0.8.5): no default retry of non-idempotent operations (§5.4). Earlier reports of the same defect: jetcd [#1072](https://github.com/etcd-io/jetcd/issues/1072) (stale-closed 2022), etcd [#14092](https://github.com/etcd-io/etcd/issues/14092) (closed 2022 as a jetcd issue), [#14890](https://github.com/etcd-io/etcd/issues/14890) (closed 2024-08-02), [#18424](https://github.com/etcd-io/etcd/issues/18424) (closed not planned 2025-06-01).

**Locks (P5)**
- etcd [#11456](https://github.com/etcd-io/etcd/issues/11456) (fix `b14c7cd44`, v3.5.0) and [#11457](https://github.com/etcd-io/etcd/issues/11457) / PR [#11490](https://github.com/etcd-io/etcd/pull/11490) (docs, 2020-03-05): from the Jepsen etcd 3.4.3 analysis (§2.3).
- etcd [#19597](https://github.com/etcd-io/etcd/issues/19597), [#22082](https://github.com/etcd-io/etcd/issues/22082) (both open): lock recipe defects (Appendix A).

**Performance and load**
- etcd [#21735](https://github.com/etcd-io/etcd/issues/21735): root JWT signing per expired lease.
- etcd [#22450](https://github.com/etcd-io/etcd/issues/22450) (open): leader apply backlog.
- etcd [#12594](https://github.com/etcd-io/etcd/issues/12594): longer lease timeout under load.
- etcd [#13294](https://github.com/etcd-io/etcd/issues/13294): why a new leader restarts TTL counting.

These are covered in §6 and Appendix A.

**Kubernetes and external**
- Kubernetes [#65497](https://github.com/kubernetes/kubernetes/issues/65497), [#110210](https://github.com/kubernetes/kubernetes/issues/110210) (open), [#96836](https://github.com/kubernetes/kubernetes/issues/96836), [#47532](https://github.com/kubernetes/kubernetes/issues/47532), PR [#64539](https://github.com/kubernetes/kubernetes/pull/64539), [#86812](https://github.com/kubernetes/kubernetes/issues/86812).
- [Jepsen: etcd 3.4.3](https://jepsen.io/analyses/etcd-3.4.3) (2020-01-30); test source `jepsen-io/etcd` @ `16149cb`.
- [Jepsen: jetcd 0.8.2](https://jepsen.io/analyses/jetcd-0.8.2) (2024-08-08); harness `jepsen-io/etcd` @ `fbe693e`.
- Background for the anomaly names in §5.4: Jepsen's [consistency models](https://jepsen.io/consistency). G1a (aborted read) and G1c (cyclic information flow) are Adya's phenomena, as cited by the jetcd report.

## Appendix C. Revision history

### C.1 Versions of this file

| Date | Change |
|---|---|
| 2026-10-31 | First version, written independently of the other lease analyses. |
| 2026-10-31 | The session that wrote `analysis/watchers-claude.md` added the `WithRequireLeader` lease section (then §12, now §5.2). Its code claims were verified at `v3.7.3`. The only thing run was a Docker test of the *watch* path (watchers-claude.md Appendix C, then §13.5); nothing lease-specific. |
| 2026-11-01 | Revised using the Codex and Antigravity lease analyses as leads (C.2). Two corrections of earlier text: (1) the client-side window had been called "normally milliseconds"; it is about 1 s plus reply delay, because of the 1-second deadline sweep (§5.1). (2) An earlier chat summary led with the point that the KV state is safe (P1). That made the answer read as more equivocal than the evidence warrants, since P3 is the property that matters. |
| 2026-11-01 | Restructured for readability: Scope, Summary and Conclusions added; Direct answer and Summary merged; sections reordered. Content unchanged except one stale cross-reference: the §4.2 client-view row said "normally ms", contradicting the correction above. |
| 2026-11-01 | §2.5 added after Antigravity disputed my rejection of its response-header finding. My rejection ("deliberate lower bound") holds only for successful renewals; see C.2. |
| 2026-11-09 | **Jepsen pass**, at the user's request: the etcd 3.4.3 and jetcd 0.8.2 reports read in full, their issues traced, and their fixes checked at `v3.7.3` and `jetcd-0.8.7`. New: §2.3 "Jepsen's measurements", "#11456" and "Documentation"; §5.4 (automatic retries, including the lease grant/revoke and jetcd `Lock` consequences, inferred); key finding 12; a §9 row; §11.2, §11.3 bullets; Appendix A rows marked ‡; Appendix B entries. Corrected: the §8 Jepsen bullet attached process pauses to the healthy-cluster quote. No earlier finding or rating changed. Nothing run. |
| 2026-11-09 | Context added to the closures of #13915 and #13776 (key finding 11, §2.1 P4, the §3.7 timeline and "Reading the timeline", §11.2, Appendices A and B): both were closed by the stale bot during the 2025-05-13 to 2026-01-13 window when its `stage/tracked` exemption was ignored, and #13915 had been reopened once by a maintainer. The earlier text could be read as maintainers declining the designs. Finding 11 and its High rating are unchanged: a tracked issue closed by the project's own process, and left closed, counts for an adopter the same as one declined. Checked through the GitHub API (labels, closers, timeline) and the workflow fix `55c5fabe5`. |

**Old → new section numbers** (for references made before the
restructuring):

| Old | New |
|---|---|
| Direct answer | Summary; §1.1; §2.3; §3.1; §3.6 |
| Summary | Summary; Terms in Scope |
| §1 | §1.2 |
| §2 | §1.3, §1.4 |
| §3 | §2 |
| §4.1 | §3.1, §3.2 |
| §4.2 | §3.3 |
| §4.3 | §3.4 |
| §4.4 | §3.5 |
| §4.5 | §3.7 |
| §5 | §4 |
| §6 | §7 |
| §7.1 | §5.1 |
| §7.2 | §5.3 |
| §8 | §8 |
| §9 | Appendix A |
| §10 | §9 |
| §11 | §10 |
| §12 | §5.2 |
| §13 | §6 |
| §14 | Appendix C.2 |

### C.2 Use of the other lease analyses (2026-11-01)

**Sources.** `analysis/leases-codex.md` (Codex) and `analysis/leases-agy.md`
(Antigravity), both written independently of this file.
- **Main pass:** the versions committed at `e19f63d`, read in full.
- **Second pass:** both files were rewritten by their own tools during this
  revision (2026-11-01, about 20:17–20:18), partly using this file. I
  skimmed the rewrites for new leads only; rows marked "revised version"
  come from them.

Each item was re-checked against code, tests, Raft, the Chubby paper or the
GitHub API before use. Where my conclusion differs from the source's, the
difference is stated. A standalone summary of the rejections is in
`analysis/leases-critique-claude.md`.

| Change | Where (new §) | Idea from | Verification and my call |
|---|---|---|---|
| R5: correct revocation delayed past a leader change | §3.4 R5, Summary, §4.3 | Codex | Each step re-traced in code. Added the bound from the client's deadline sweep and the `KeepAliveOnce`/re-`KeepAlive` exposure. Impact rated Low because the victim's lease had already expired on the old leader. |
| Partition variant of R1: `isActive()` skips the ReadIndex on a partitioned leader | §3.4 R1, Summary | Antigravity | Mechanism confirmed. **Consequence rejected**: the source calls it a High-confidence P3 failure. My derivation from raft `v3.7.0` shows `Promote`'s ET extension covers it to within about one tick. |
| Checkpointing does not bound P2 under churn; recorded value is a duration | §1.4, §2, Summary | Codex | Verified in `Promote`/`Demote`/`scheduleCheckpointIfNeeded`. Added: `Session`'s 60 s default TTL is never checkpointed; the interval has no operator flag (the source's flag name does not exist at v3.7.3); the reset is asserted by `TestV3LeaseCheckpoint`. |
| #22649 details: late clear does not repair the timer; PR #22650 | §3.5 | Codex | Verified. Added on my own: the partitioned-leader trigger, and the missing primacy re-check after the clear. |
| #13675 (grant visible on a follower before the leader) | §3.6, §3.7, App. A | Codex | Maintainer diagnosis read; fix kept in `FastLeaseKeepAlive`. |
| `FastLeaseKeepAlive` renews leases whose revocation is committed but unapplied | §2.4 | Antigravity | The PR text confirms it is deliberate. Narrowed to a real-time-order P4 case. Not P3, because the revocation was explicit or the lease was already expired. |
| Forwarded keepalives keep the apply wait (1 s, then 408) | §6.1, §3.4 R3, Summary | Antigravity | Verified in `leasehttp`. Added the (N−1)/N exposure, the 7 s failure path and the link to R5. |
| Expiry cost: per-lease proposals, root token per revocation (#21735), retry behaviour | §6.2 | Antigravity | Verified `WithRoot` per call; read #21735 (closed not planned by the reporter). Added the 3 s retry and hand-off skip, which show that overload delays expiry rather than dropping it. |
| jetcd `onError` on every stream error | §5.3, Summary | Antigravity | Verified. Added on my own: entries are kept and renewed after `onError` (the #22094-like end state), the origin commit and #1322, and the first-response deadline tied to the gRPC `keepaliveTimeout`. |
| Client deadline sweep of 1 s | §5.1 | Antigravity | Verified for both clients. **Corrected my earlier "normally milliseconds"** to about 1 s plus reply delay. |
| Chubby §4.4 (sessions not persisted; worst-case wait) and §2.8/§3 load-based lease extension vs etcd's TODO and #12594 | §4.1, §4.2 | Codex (§4.4); Antigravity (TODO) | Quotes checked in `doc/chubby-osdi06.md`. |
| Kubernetes Lease objects have no TTL | §7.1 | Codex | Upgraded from "recalled" to verified at `v1.37.2`. |
| Open-issue survey additions | App. A | both | States checked on 2026-11-01. The open-issue search count was re-run myself (27). |
| Test inventory (3- and 5-member #15247 tests, `TestLeaseRevokeDuringRenew`) | §8 | Codex | Verified in `tests/e2e/v3_lease_no_proxy_test.go`. |
| R4 also reaches automatic revocation, if the renewal passed its expiry check just before expiry | §3.4 R4 | Codex (revised version) | Verified against `Renew`'s expired-wait branch. |
| One knob serves two uses (sessions vs. TTL garbage collection) | §2.2 | both (Codex original; Antigravity revised) | Design inference. Chubby ephemeral-file quote added by me. |
| jetcd's disabled `LeaseUnitTest` encodes the `onError`-and-continue contract | §5.3 | Codex (revised version) | Verified: the class-level `@Disabled` and the test body. |
| jetcd `keepAliveOnce` writes the request twice | §5.3 | Antigravity (revised version) | Code verified. Vert.x `end(T)` semantics recalled. I did not adopt the "NullPointerException" framing as a user-visible bug, because the original error wins in `whenComplete`. |
| §5.1 window is typical, not a bound | §5.1 | Codex (revised version) | Accepted: pauses and clock steps can enlarge it. |
| Header revision on lease responses: correct lower bound only for successful renewals | §2.5, §2.1 (P4) | Antigravity (its reply to `leases-critique-claude.md`) | First rejected as a deliberate lower bound. **Reconsidered**: verified that "not found" keep-alive answers reuse the pre-filled header, even on the leader, and that `TimeToLive` headers have no consistent direction. Kept at Low impact: no recipe reads these headers, and neither client delivers TTL-0 stream responses. |

**Considered and not adopted:**
- *"Kubernetes abandoned or refused native etcd leases, the clearest
  production verdict on them"* (Antigravity). Kubernetes components reach
  state only through the API server, so coordination had to be an API
  object. That is not evidence about etcd leases (§7.1).
- *The RBAC check on renewal uses the local lessor without an apply wait*
  (Antigravity). The code path is real: `checkLeaseRenew` skips key checks
  when the local lessor does not know the lease (`v3_server.go:575-606`,
  `:593`). But the effect is limited: a non-admin user who knows a lease ID
  can keep the lease alive during the apply lag after a grant or attach, and
  renewal only extends a lease. Recorded here; not tracked as a
  lease-correctness issue.
- *Events retained under leader churn "until etcd exhausted
  `QuotaBackendBytes`" (`NOSPACE`)* (Antigravity). Not in the #65497 text
  quoted in §7.2. I did not search its comments, so the claim stays
  unverified and is not adopted.
- *#13915 closed 2025-12-12* (Antigravity). The API gives 2025-12-19, as
  this file already said.
- *Kubernetes `lease_manager` "defaults to 60 s"* (Antigravity). 60 s is the
  cap on the reuse window, not the TTL (§7.1).
- Codex's external blog reference (amyangfei, 2020, v3.4.14). Not re-checked.
  Its points about `Session.Done()` are covered by §5.1 and §2.3.
