# Chubby perspectives: sessions, KeepAlives, and the etcd lease and watch issues (Claude)

**History:** Written 2026-11-01. Revised 2026-11-02 after cross-checking two independent analyses (Appendix B), then reorganized for readability. Later on 2026-11-02, a caveat on etcd's watch delivery disclaimer was added to §6.1; it points to `etcd-issues-and-mitigations-claude.md` §3. Corrected 2026-11-10 after an accuracy review: the upstream report #22714 was added to the lease failover rows (§1, §5), and the account of how #19091's first fix was closed was corrected (§8.2).

**Versions analyzed:**

| Component | Version | Notes |
|---|---|---|
| etcd | `v3.7.3` = `0a173d0410a9` (local clone `~/github/etcd`) | All `path:line` citations are at this tag unless marked otherwise. `git fetch --tags` on 2026-11-01 showed no newer stable release. `origin/main` at `a2e3642bc` (2026-11-02) was consulted only to check whether `client/v3/leasing` had changed. |
| jetcd | `jetcd-0.8.7` = `23b61f1499a5` | Checked: `LockImpl.java` and `ElectionImpl.java` call the server's Lock and Election gRPC services; jetcd has no session class; the lease keepalive code in `LeaseImpl.java:240-287`. |
| Chubby | Burrows, OSDI 2006, `doc/chubby-osdi06.md` | Sections 2.1–2.9, 3, 3.1 and 4.1–4.6 were read in full. |
| Functional test | Official image `gcr.io/etcd-development/etcd:v3.7.3`, one member; Go client `go.etcd.io/etcd/client/v3 v3.7.3` built with go1.26.10 in the `golang:1.26` container | Correctness tests of `client/v3/leasing` (§8.3): three short runs on 2026-11-02 on this host. Counts of revisions and Raft proposals only; no timings. Not a benchmark (AGENTS.md §7). |

Method, inputs and citation conventions are in Appendix A.

**Upstream references.** † marks states read through the GitHub REST API on 2026-11-02. The others are as recorded in `leases-claude.md` and `watchers-claude.md` on 2026-11-01.
- **etcd leases:**
  - [#12528](https://github.com/etcd-io/etcd/issues/12528);
  - [#15247](https://github.com/etcd-io/etcd/issues/15247), fixed by [#16822](https://github.com/etcd-io/etcd/pull/16822)† (merged 2023-12-15; merge commit `4e986363a371` is an ancestor of `v3.7.3`);
  - [#22649](https://github.com/etcd-io/etcd/issues/22649), with fix PR [#22650](https://github.com/etcd-io/etcd/pull/22650)† (open);
  - [#22081](https://github.com/etcd-io/etcd/issues/22081), [#22094](https://github.com/etcd-io/etcd/issues/22094), [#12594](https://github.com/etcd-io/etcd/issues/12594), [#9888](https://github.com/etcd-io/etcd/issues/9888);
  - [#18069](https://github.com/etcd-io/etcd/issues/18069)† (closed) and [#18428](https://github.com/etcd-io/etcd/pull/18428)† (merged 2024-08-14).
- **etcd locks:** [#19597](https://github.com/etcd-io/etcd/issues/19597)† (open), [#22082](https://github.com/etcd-io/etcd/issues/22082)† (open).
- **etcd `client/v3/leasing`:**
  - [#19091](https://github.com/etcd-io/etcd/issues/19091)† (closed by the stale bot as "not planned");
  - fix PRs [#19092](https://github.com/etcd-io/etcd/pull/19092)† and [#21589](https://github.com/etcd-io/etcd/pull/21589)†, both closed unmerged;
  - [#21869](https://github.com/etcd-io/etcd/pull/21869)† (closed unmerged);
  - [#22083](https://github.com/etcd-io/etcd/issues/22083)†, [#22084](https://github.com/etcd-io/etcd/issues/22084)†, [#22417](https://github.com/etcd-io/etcd/issues/22417)† (all open).
- **etcd watches:** [#21813](https://github.com/etcd-io/etcd/issues/21813), [#17611](https://github.com/etcd-io/etcd/issues/17611), [#18879](https://github.com/etcd-io/etcd/issues/18879), [#22264](https://github.com/etcd-io/etcd/issues/22264), [#16839](https://github.com/etcd-io/etcd/issues/16839), [#19710](https://github.com/etcd-io/etcd/issues/19710), [#20242](https://github.com/etcd-io/etcd/issues/20242), [#15402](https://github.com/etcd-io/etcd/issues/15402).
- **Kubernetes:** [#59848](https://github.com/kubernetes/kubernetes/issues/59848), [#65497](https://github.com/kubernetes/kubernetes/issues/65497), [#110210](https://github.com/kubernetes/kubernetes/issues/110210), [#111116](https://github.com/kubernetes/kubernetes/issues/111116)† (closed as completed 2022-07-28), [KEP-2340](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2340-Consistent-reads-from-cache).

## Scope

**Question.** In Chubby, every client's state hangs off one session. A KeepAlive RPC held at the master keeps the session alive and also carries events and cache invalidations to the client. Would that design prevent or simplify the etcd lease and watch issues found in `leases-claude.md` and `watchers-claude.md`? Which issues would it leave untouched, and what would it cost?

**Kind of document.** A design comparison, not a bug hunt. The one exception is §8, which examines etcd's own Chubby-style cache, `client/v3/leasing`, as a natural experiment, and found defects in it.

## Summary

**Short answer: partly.**
- **The key rule.** What makes Chubby's design work is not the held RPC but one rule: a client cannot renew its session without acknowledging the cache invalidations sent to it. etcd has server push (watches) but no acknowledgement, and its lease renewals are independent of watch delivery.
- **What it would fix.** Adopting the rule would remove etcd's main watch failure: a watch that goes silent while the client otherwise looks healthy. Separately, Chubby's lack of standalone lease objects avoids a whole line of etcd lease bugs.
- **What it would not fix.** The lease failover-safety bugs; Chubby handles those with other mechanisms. Chubby is also worse than etcd at delivering every event.
- **The price.** Write latency tied to the health of every caching client. That suits Chubby's read-mostly workload, not Kubernetes'.

**Findings.**
1. **The premise is mostly accurate; five corrections are in §2.** The most consequential two:
   - KeepAlives do not make lock-based algorithms safe against delayed requests; the paper needs sequencers or lock-delay for that (Chubby §2.4).
   - The authors wanted to move events *off* the KeepAlive RPC while keeping the acknowledgement rule (Chubby §4.6).

   — **High**.
2. **The essential mechanism is that liveness depends on acknowledgement.** It is not the held RPC or the server-to-client upcall. "Clients cannot maintain a session without acknowledging cache invalidations" (Chubby §2.8, p. 7). From this the master gets an invariant: every live session has acknowledged every invalidation sent before its latest KeepAlive, and every other session has expired. — **High**.
3. **etcd has push but neither half of that rule.**
   - Watch streams push events, but the client never acknowledges them.
   - Lease keepalives, watch streams and KV calls fail independently.
   - With the Go client's round-robin balancer, a client's lease and watch streams can be served by different etcd members.

   — **High** (code); **High** for the per-stream placement (recalled gRPC behaviour, not re-read in grpc-go).
4. **Watch freshness: Chubby's coupling would prevent the main failure class at v3.7.3, silent staleness.**
   - That covers a cut-off or stuck serving member, a dead connection, and the gRPC-proxy cases. A Chubby client cannot stay live for longer than its lease while cut off from deliveries, and its library then blocks calls and flushes its cache. — **Moderate** (that this transfers to etcd).
   - Chubby's library also has two stages: it stops trusting its cache first (jeopardy), and gives up the session only after a 45 s grace period. etcd's only fail-safe, the opt-in `WithRequireLeader`, has one terminal stage. Kubernetes' API server reacted to it by dropping all its watchers (#111116).
   - Chubby's own applications reproduced that failure one level up: many crashed on the fail-over event (Chubby §4.6). — **High** (code, paper, issue text); **Moderate** (the transfer).
5. **Event history: Chubby is worse.** Its events are lost across fail-over, and a "fail-over" event tells clients to rescan. etcd's revisioned, resumable history is stronger. The fixed etcd bugs in restore, compaction and resume sit in machinery Chubby simply lacks. — **High**.
6. **Lease failover safety: not addressed by sessions or KeepAlives.**
   - This covers #12528 and #15247 (both fixed; #16822 is in v3.7.3) and the residual windows R1–R5, which are inferred from code and not reproduced.
   - The paper places Chubby's defence against this class in master leases and epochs, which are separate mechanisms. It does not say how a deposed master's own pending work is fenced, so "Chubby prevents it" is an inference.

   — **Moderate**.
7. **The master lease, not the KeepAlive, is what makes a renewal both safe and cheap.**
   - A Chubby master serves reads alone, which "is safe provided the master lease has not expired" (Chubby §2.2, `:94`). I infer the same holds for answering KeepAlives.
   - An etcd leader proves leadership with a ReadIndex, a quorum round. Doing that per renewal cost 20 % more leader CPU at 1500 renewals/s (#18069). It was then made conditional on a local tick (#18428), which opened window R1.

   — **High** (paper, issue text); **Moderate** (the attribution).
8. **No standalone lease objects is Chubby's strongest simplification.** Chubby has one liveness primitive per client and no timer for deleting data. So it has:
   - no conflict between the expiry upper bound (P2) and the no-early-revocation lower bound (P3);
   - no checkpoint machinery, hence no #22649;
   - no per-lease client keepalive bookkeeping, where the Go bugs (#22081, #22094) and jetcd's `onError` behaviour sit.

   Chubby still has no upper bound on cleanup after fail-over. — **Moderate**.
9. **etcd's `client/v3/leasing` shows the protocol without the rule.** It copies Chubby's invalidating cache from leases and watches, and is also exposed by `etcd grpc-proxy --experimental-leasing-prefix` ("disconnected linearized reads"). Because its acknowledgement channel (a watch) and its liveness channel (a lease keepalive) are independent:
   - a writer can block for as long as the owner's watch is stalled while its keepalive works (**Moderate**, inferred from code);
   - a cut-off owner serves stale "linearizable" reads for about 1 s after the server expires its lease, and without bound under process pauses. This is **High**: an outside contributor reported it upstream with a failing test (#19091). The stale bot closed the report, and two fix PRs were closed unmerged.

   It also shows the cost of rebuilding the protocol from consensus writes:
   - only one client can cache a key;
   - every linearizable read of a key the client does not own is a Raft write;
   - a write by a non-owner is applied **twice**, and a non-owner `Delete` reports `Deleted=0`;
   - a contested write takes five Raft proposals.

   — **High** (reproduced on v3.7.3; unchanged on `origin/main`; no upstream report found).
10. **Performance: the coupling turns silent staleness into bounded write stalls.**
    - Every write to cached data waits for every caching client to acknowledge, or for its lease to expire. Chubby observed "tens of seconds".
    - Chubby could afford this because `SetContents` and `Acquire` together were about 0.07 % of its RPCs.
    - In Kubernetes, the etcd-level cachers are the API servers. One stalled API server would stall writes to everything it caches, for up to a lease period.

    — **Moderate** (inference).
11. **Locks: Chubby's server-side wait removes a dependency etcd has.**
    - An etcd lock waiter learns that its predecessor is gone through a watch (`waitDeletes`), and keeps waiting even after its own session has expired (#19597, open).
    - Chubby's nodes carry instance numbers and its handles sequence numbers, and the paper guards explicitly against delayed or duplicated packets. etcd's `Mutex.Unlock` deletes by a key name that is reused across acquisitions, so a stale retry can release a newer lock (#22082, open).
    - Neither design gives mutual exclusion without fencing; etcd's revision compare is equivalent to Chubby's sequencer. Chubby's lock-delay has no etcd counterpart and is independent of KeepAlives.

    — **High**.

**Verdict.** — **Moderate**.
- **Right** for the watch-freshness class, and for the cost of standalone lease objects.
- **Wrong** for the lease failover-safety bugs and for event completeness.
- **Costly.** The design trades etcd's silent staleness for write latency held hostage by client health. That suits Chubby's read-dominated, coarse-grained workload, not Kubernetes'.
- **What is portable is the invariant:** liveness conditional on acknowledgement, or on a server-asserted freshness bound. etcd could adopt a useful part of it in its client libraries without coupling writes to clients (§10): a conservative lease deadline, and a session object with a jeopardy state. On the server, it could separate session leases from data-expiry leases and term-fence expiry revocations.

Per-issue verdicts are tabulated in §1.

## Terms

**Chubby:**
- **Session:** "a relationship between a Chubby cell and a Chubby client". The client's handles, locks and cached data stay valid while the session does (Chubby §2.8, p. 7, `doc/chubby-osdi06.md:253`). There is one session per client process, and the server knows which client owns it.
- **KeepAlive:** the session-renewal RPC. The master holds it until the lease is nearly due, or until it has an event or invalidation to deliver (§3.1).
- **Invalidation:** a message telling a client to drop cached data before a write proceeds (Chubby §2.7).
- **Jeopardy / grace period:** the state a client enters when its local lease estimate runs out. The library flushes and disables its cache and blocks API calls for a grace period (45 s by default) while it tries to reach a master (Chubby §2.8, pp. 7–8).
- **Master lease:** the replicas' promise not to elect another master for an interval (Chubby §2.2, p. 3).
- **Epoch:** the number a new master hands out and requires on every client call (Chubby §2.9 step 1, p. 9). Both the master lease and epochs are separate from sessions.
- **Lock-delay:** a period during which a lock freed by a *failed* holder cannot be taken (Chubby §2.4, p. 5).
- **Sequencer:** a token carrying the lock generation, for servers to check (Chubby §2.4, p. 5).

**etcd:**
- **Lease:** a server object with an ID and a TTL, to which any number of keys can be attached. Anyone who knows the ID can renew or revoke it. Revocation deletes the attached keys.
- **P2 / P3:** properties from `leases-claude.md` §2.1.
  - P2 is the upper bound: an unrenewed lease is revoked about TTL after its last renewal.
  - P3 is the lower bound: a lease is not revoked within TTL of an acknowledged renewal.
- **R1–R5:** the residual P3 failure windows at v3.7.3 (`leases-claude.md` §3.4).
- **Promote:** what the etcd lessor does when its member becomes leader. It extends every lease by its full TTL plus the election timeout (ET), which is 1 s by default.
- **Serving member:** the etcd member that holds a client's watch stream.
- **`WithRequireLeader`:** an opt-in client flag. It asks the serving member to cancel the stream when it has no leader.
- **Owner (`leasing`):** in `client/v3/leasing` (§8), the one client that holds the ownership key for a data key and may serve that key from its cache.

**This file:**
- **Liveness, delivery and acknowledgement channels:** how a client proves it is alive; how the server pushes changes to it; how the server learns that the client has processed them.
- **F1–F5:** the five functions of a Chubby KeepAlive (§3.1).
- **I1–I5 / N1–N6:** what the coupling gives and does not give (§4).
- **L1–L11, W1–W12, B1–B6:** lease issues (§5), watch issues (§6), borrowable changes (§10).
- **Requirements A, B, C** (§3.5):
  - **A:** read current coordination state from a local cache;
  - **B:** consume every mutation;
  - **C:** delete data at a deadline.

## 1. Verdicts at a glance

| Issue class (source) | Does Chubby's session/KeepAlive design prevent it? | Confidence | Detail |
|---|---|---|---|
| Silent watch on a cut-off, stuck or paused serving member; proxy ignoring `WithRequireLeader` (watchers §5, §6, §9) | **Yes**: the delivery channel is the liveness channel | Moderate | §6.1 |
| Stale reads from watch-fed caches (Kubernetes #59848, KEP-2340 machinery) | **Yes, for first-tier caches**, at the price of write stalls | Moderate | §6, §9 |
| Leader loss is terminal for flagged watches; mass watch teardown (Kubernetes #111116) | **Yes in the library** (jeopardy, then a grace period). Chubby's applications reproduced it by crashing on the fail-over event | Moderate | §6.2 |
| Event loss across fail-over; restore, compaction and resume bugs | **No**: Chubby is lossy and has no history. The bugs are avoided by omission | High | §6.3 |
| Slow watchers stalling writes (#22264) | **Inverted**: Chubby stalls writes by design, but for a bounded time | Moderate | §6.4 |
| Lease failover safety, P3 (#12528, #15247 fixed; R1–R5 residual, of which R2 is reported reproduced upstream in #22714, 2026-11-06) | **No**. The paper relies on master leases and epochs here, and does not state how a deposed master's internal work is fenced | Moderate | §5.1 |
| Safe renewal needs a quorum round; its cost led to a weaker check (#18069 → #18428, R1) | **No, not through sessions**: the master lease lets Chubby's master answer alone | Moderate | §5.4 |
| No upper bound after leader change, P2 (#65497, #9888) | **No**; Chubby has the same rule. The P2/P3 *conflict* is avoided because Chubby has no data-expiry objects | High (paper) | §5.2 |
| Checkpointing and #22649 | **Avoided by omission** (nothing to checkpoint) | Moderate | §5.2 |
| Go #22081/#22094, jetcd `onError`: lease renewed after the app thinks it is gone | **Yes, in substance**: one library-owned, monotone session per client | Moderate | §5.3 |
| Optimistic client lease deadline | **Another Chubby rule** (a conservative client lease), portable to etcd clients | High | §5.3 |
| No renewal load control (`lessor.go:290-291`) | **Yes**: a held RPC makes the server the pacer | High | §5.4 |
| Forwarded renewals failing under a leader apply backlog | **No**; Chubby avoids it by talking only to the master | Moderate | §5 (L8) |
| Lock waiters depending on watch delivery (`waitDeletes`); waiter outliving its own session (#19597) | **Yes**: the master grants locks, and session expiry ends pending calls | Moderate | §7 |
| Stale `Unlock` retry deleting a newer lock incarnation (#22082) | **Not a session property**. Chubby's instance and handle sequence numbers target the same delayed-packet hazard; the paper does not say how `Release` uses them | Moderate | §7 |
| Mutual exclusion without fencing | **No**; Chubby also needs sequencers or lock-delay | High | §7 |

## 2. Premise check

Most of the premise is accurate. The held KeepAlive, the early return for events and invalidations, acknowledgement by the next KeepAlive, and write blocking are as described (Chubby §2.7–§2.8, pp. 6–7, `doc/chubby-osdi06.md:229-261`). Five points need correcting.

1. **Ephemeral files are not owned by one session.**
   - *Prompt:* ephemeral files "are associated to a session and get removed if the session ends".
   - *Paper:* "ephemeral nodes are also deleted if no client has them open" (Chubby §2.3, p. 4, `:118`). They are reference-counted across *all* sessions that hold them open.
     - A monitor that *opens* someone's presence file therefore keeps it alive. The paper provides child-node events precisely to "monitor ephemeral files without affecting their reference counts" (Chubby §2.5, `:166`).
     - After a fail-over, deletion waits about a minute, so "ephemeral files may not disappear promptly" (Chubby §2.9 step 9, p. 9, `:313`).
   - *Why it matters:* Chubby's closest analogue of an etcd lease-attached key is not strictly session-owned. It also has a fail-over cleanup delay, i.e. the P2 problem (§5).
2. **Invalidations follow caching, not open handles.**
   - *Prompt:* invalidations go to "every session that has the file open".
   - *Paper:* the master keeps "a list of what each client may be caching" and sends invalidations "to every client that may have cached it" (Chubby §2.7, p. 6, `:229-231`). Caching includes file absence and metadata.
   - *Why it matters:* it does not change the argument. It shows that the master keeps per-client cache state, more than etcd tracks for anything.
3. **A write waits for the remaining lease, and the lease length is the server's choice.**
   - *Prompt:* an unresponsive session makes the write wait "the session TTL (typically 12 seconds)".
   - *Paper:* the write proceeds once each client has acknowledged, or "allowed its cache lease to expire" (`:239`): the *remaining* lease, at most one lease period.
     - The period is 12 s by default, "up to around 60s" under load (Chubby §3, p. 10, `:352`).
     - Observed: writes delayed "by up to tens of seconds if a recently-failed client cached the file" (Chubby §4.1, p. 11, `:440`).
     - Reads are not blocked meanwhile; the node is served uncached (`:241`).
   - *Why it matters:* the worst-case write stall grows exactly when the master is overloaded.
4. **Lock hand-over after a failure also includes lock-delay.**
   - *Prompt:* locks are "taken away" from a client that fails to renew, on the same TTL.
   - *Paper:* yes, but a lock freed by a failed holder then stays unavailable for the **lock-delay**, up to one minute (Chubby §2.4, p. 5, `:159`).
   - *Why it matters:* hand-over after failure is lease expiry plus lock-delay. Lock-delay is a separate mechanism, and etcd has none.
5. **The KeepAlive is not where Chubby's lock safety comes from, and its piggybacking was negotiable.**
   - *Prompt:* the KeepAlive design is "central to the guarantees chubby provides and to how distributed systems algorithms implemented on top of chubby are possible".
   - *Paper:* partly.
     - It locates lock safety against delayed messages in **sequencers** and lock-delay, not in sessions (Chubby §2.4, `:155-159`).
     - It reports that coupling events to KeepAlives forced KeepAlives onto UDP. It proposes a separate TCP `GetEvent()` RPC, with the KeepAlive reply still listing unacknowledged events "so that events must eventually be acknowledged" (Chubby §4.6, p. 14, `:542-546`).
   - *Why it matters:* the authors kept the *acknowledgement rule* and were willing to drop the *piggybacking*. §3 builds on that distinction.

**One premise is right, and worth sharpening:** etcd's lock *is* built from leases and keys. A server-side `Lock` RPC exists, but `v3lock` runs the same client recipe inside the server, on an in-process client. The replicated state has no lock primitive. — **High**.
- `server/etcdserver/api/v3lock/lock.go:35-46`; wired at `server/embed/serve.go:150`.
- The in-process client is built from adapters at `server/etcdserver/api/v3client/v3client.go:30-39`.

## 3. The mechanism

### 3.1 What one Chubby KeepAlive does

One outstanding RPC per session carries five functions. The client "initiates a new KeepAlive immediately after receiving the previous reply. Thus, the client ensures that there is almost always a KeepAlive call blocked at the master" (Chubby §2.8, p. 7, `:259`).

| Function | Direction | Paper |
|---|---|---|
| F1. Liveness proof | client → master | The KeepAlive request itself |
| F2. Lease grant | master → client | The reply carries the new lease timeout; "The master may extend the timeout by any amount" (`:259`) |
| F3. Delivery | master → client | "the KeepAlive reply is used to transmit events and cache invalidations"; it may "return early" for them (`:261`) |
| F4. Acknowledgement | client → master | The *next* KeepAlive acknowledges what the previous reply carried (`:231`, `:239`) |
| F5. Pacing | master | The master chooses when to reply, and how far to extend under load (`:259`, `:352`) |

**What F4 proves, and what it does not.** — **High** (paper).
- **It proves only that the client *library* has flushed the invalidated cache entries.** It says nothing about the application: data the library already returned, and anything derived from it, are outside the protocol. The library cannot recall bytes it has handed over.
- **Only invalidations gate writes.** Invalidations go out *before* a write, and the write waits for them (Chubby §2.7). Events are "delivered after the corresponding action has taken place", and a later read returns the new data "(or data that is yet more recent)" (Chubby §2.5, `:172`). No write waits for an event.
- **Event acknowledgement is about delivery, not write ordering.** In the Chubby §4.6 redesign, "events must eventually be acknowledged" (`:546`).

### 3.2 How etcd splits the same functions

In etcd, each function lives on a separate gRPC stream or call.

| Function | etcd at v3.7.3 | Evidence |
|---|---|---|
| F1 + F2 | `LeaseKeepAlive` stream. The client sends a request every TTL/3, and the server renews and answers at once. There is one renewal per *lease*, not per client | `client/v3/lease.go:542`; server loop `server/etcdserver/api/v3rpc/lease.go:112-147` (renew at :136, send at :147) |
| F3 | `Watch` stream, per context metadata, served by whichever member the stream landed on | `watchers-claude.md` §1, §8.1 |
| F4 | **None.** The server never learns what a client has processed. gRPC flow control is the only back-pressure, and the Go client defeats it with an unbounded buffer | `watchers-claude.md` §8.1 (`client/v3/watch.go:879-880`) |
| F5 | Client-paced at TTL/3. A TODO notes the missing server-side control | `server/lease/lessor.go:290-291`; #12594, closed by the stale bot |

**The streams can land on different members.** — **High** (code for the policy; recalled for per-stream picking).
- The Go client's resolver sets `{"loadBalancingPolicy": "round_robin"}` (`client/v3/internal/resolver/resolver.go:44`).
- gRPC's `round_robin` picks a subchannel for each new call or stream (recalled; not re-read in grpc-go).
- So a client's lease stream and its watch stream are, in general, served by different members with independent health.

### 3.3 The invariant, and why the upcall is not the point

etcd already has server push: a watch stream is an upcall channel. What it lacks is F4, and the coupling of F1 to F4. Chubby's rule is that a session cannot be renewed past an unacknowledged invalidation. That gives the master a two-way partition of all clients at all times:

> **Ack-or-expired.** For every write *w* to cached data, each client that may cache it has either acknowledged *w*'s invalidation, or its session lease has expired at the master.

**Chubby uses the invariant twice.**
1. **Per write,** to keep client caches consistent: "clients see either a consistent view of Chubby state, or an error" (Chubby §2.7, `:229`).
2. **Per fail-over,** as a global barrier: "The master waits until each session acknowledges the fail-over event or lets its session expire" (Chubby §2.9 step 6, p. 9, `:310`). The revised design simply waits "a full worst-case lease timeout before allowing operations to proceed" (Chubby §4.4, p. 13, `:490`).

**The transport is incidental.** The Chubby §4.6 proposal moves events to a TCP `GetEvent()` RPC, and the KeepAlive reply carries only the list of unacknowledged events. That keeps the invariant with a different transport. — **High** (paper text); my reading that the invariant, not the transport, is what matters is **Moderate**.

**The invariant's weak point is the master's knowledge of who may cache, and Chubby's own design once violated it** (Chubby §4.4, pp. 12–13, `:478-490`):
1. To cut database load, the master stopped recording sessions at creation. A session was recorded at its first modification, lock acquisition or ephemeral-file open, and active sessions "with some probability on each KeepAlive".
2. A young read-only session could therefore be missing after a fail-over. If all *recorded* sessions checked in before its lease ran out, the new master proceeded, and "the discarded sessions could then read stale data for a while". The paper calls this "rare in practice" and reports no incident; it is a design flaw, not a recorded outage.
3. The fix gave up on knowing. Sessions are no longer recorded at all, and a new master "must now wait a full worst-case lease timeout before allowing operations to proceed".

Two readings follow:
- the session design names exactly what must be accounted for, which made the flaw diagnosable;
- holding a KeepAlive at the master did not make the bookkeeping correct, and the repair cost availability.

— **High** (paper); the readings are mine, **Moderate**.

### 3.4 What a per-client session adds beyond acknowledgement

The prompt's other point is that everything stateful hangs off a named, per-client session. That has consequences beyond acknowledgement.

| Property | Chubby | etcd |
|---|---|---|
| Who owns server-side client state | The session: handles, locks, ephemeral-file references, cache entries, event subscriptions (`:253`) | Nothing per client. Leases are anonymous: any client that knows an ID can renew or revoke it. Watches belong to a stream, not to a lease |
| What the server can reason about | Whole clients: release everything on expiry, rebuild per session on fail-over (Chubby §2.9) | Individual leases and streams. "This watcher and this lock holder are the same process" cannot be expressed |
| Monotone failure | Once a handle's session has expired, every later operation on it fails, so "only a suffix of a sequence of operations" is lost (Chubby §2.8, p. 8, `:281`) | A lease ID is just a number. A goroutine the application has forgotten can keep a lease alive (L5), and a watch can outlive or precede any lease |
| What it forbids | Sharing liveness across clients; timed objects with no live client | Nothing: Kubernetes shares one lease across up to 1000 objects for a time window, with no keepalive at all (`leases-claude.md` §7.1) |

The last row is the root of the main *lease* simplification (§5.2). Chubby cannot express "delete this data at time T, whether or not anyone is alive". etcd can, and that one primitive serves two uses with opposite needs.

### 3.5 Three requirements that etcd's primitives serve together

The comparison is clearer when three jobs are kept apart. The Codex analysis suggested the separation; the assignments below are mine.

| | Requirement | What a protocol for it needs | Chubby | etcd |
|---|---|---|---|---|
| A | Read current coordination state from a local cache | Invalidation, acknowledgement, a conservative client deadline, a recovery barrier | Built for this (Chubby §2.7–§2.9) | No server support. Rebuilt on top: the Kubernetes watch cache plus KEP-2340; `client/v3/leasing` (§8) |
| B | Consume every mutation, in order, resumably | Revision cursors, retained history, compaction handling, bounded buffering | Not provided: events are hints and are lost at fail-over (N4) | The watch API |
| C | Delete data at a deadline, whether or not anyone is alive | A persisted deadline, cleanup progress, a bound on extension across fail-over | Not provided (§3.4) | Leases, shared with client liveness |

Ownership (locks and elections) comes on top. Chubby ties it to A through the session; etcd builds it from leases plus B.

How the verdicts below split:
- most "prevented" verdicts in §5–§6 concern A and ownership;
- the "Chubby is worse" verdicts concern B (W7–W9);
- the main lease simplification concerns C (L3, L4).

Several etcd failures arise where one job's machinery is taken as evidence for another. An example is a healthy lease keepalive taken to mean that a watch-fed cache is current (§8.2 a). — **Moderate**.

## 4. What the coupling gives, and what it does not

**Five invariants Chubby gets from sessions and KeepAlives:**

| # | Invariant | Paper |
|---|---|---|
| I1 | **Consistent-or-error cache.** A cached read is valid, or the library returns an error | Chubby §2.7, `:229`; jeopardy in Chubby §2.9, `:289` ("blocks all application calls on its API to prevent the application from observing inconsistent data") |
| I2 | **Bounded undetected disconnection.** A client cut off from its delivery channel learns it may be stale within its *conservative* local lease timeout | Chubby §2.8, `:263-265` |
| I3 | **One monotone liveness state per client:** safe, then jeopardy, then safe or expired. No partial liveness | Chubby §2.8, `:279-281` |
| I4 | **Server-controlled renewal rate.** Under load, longer extensions and fewer KeepAlives | Chubby §2.8, `:259`; §3, `:352` |
| I5 | **Server knowledge of every cacher.** Writes can be ordered after invalidation, with bounded waiting. The most fragile of the five: Chubby lost it for unrecorded sessions and fell back to a worst-case lease wait (§3.3) | Chubby §2.7, `:229-239`; §4.4, `:486-490` |

**Six things it does not give:**

| # | Not given | Paper |
|---|---|---|
| N1 | Mutual exclusion against delayed requests; that needs sequencers or lock-delay. Expiry withdraws a client's rights under the protocol; it does not prove that the process has stopped | Chubby §2.4, `:155-159` |
| N2 | Safety of time-based decisions across a master change. That comes from master leases (Chubby §2.2, `:90`, `:94`) and epochs (§2.9 step 1, `:305`), and the fail-over code was still "a rich source of interesting bugs" | Chubby §2.9, `:315` |
| N3 | An upper bound on cleanup after fail-over. Leases are "extended to the maximum that the previous master may have been using", and ephemeral deletion is deferred | Chubby §2.9 steps 3 and 9, `:307`, `:313` |
| N4 | Event completeness across fail-over. A "fail-over" event says "other events may have been lost, so data must be rescanned" | Chubby §2.5, `:167` |
| N5 | Scalability for write-heavy, many-cacher workloads. Chubby's design rests on "write traffic constitutes much less than one percent" | Chubby §3.1, `:360`; §4.1 table: `SetContents` 680 ppm |
| N6 | Anything about the application. The acknowledgement is the library's; data already handed to the application is outside the barrier, and no write waits for an event (§3.1) | Chubby §2.5, `:172`; §2.7 |

## 5. Leases, issue by issue

Issue numbers and properties are from `leases-claude.md`; the section numbers in the first column refer to that file. For each issue, the question is whether Chubby's *session and KeepAlive design* prevents it, as opposed to some other Chubby mechanism.

| # | etcd issue (leases-claude §) | Verdict | Chubby mechanism that matters | Confidence |
|---|---|---|---|---|
| L1 | P3 violated on failover: a stalled ex-leader revokes live leases or acknowledges unseen renewals (#12528, #15247, both fixed; #16822 is in v3.7.3). Residual windows R1–R5 are inferred from code; this project reproduced none, and upstream #22714 (2026-11-06, open) reports R2 reproduced on v3.7.2 (leases-claude §3.4) | **Not prevented by sessions** | Master lease (a real-time bound on authority) and epochs (stale client calls rejected). The paper does not describe how a deposed master's own pending work is fenced. The etcd flaw: revocations commit as term-less commands (leases-claude §4.3). See §5.1 | Moderate |
| L2 | P2: no upper bound; every leader change resets each lease to TTL + ET (§1.3, §2.2) | **Shared** | Chubby extends leases to the maximum on fail-over and defers ephemeral deletion (N3). See §5.2 | High (paper and code) |
| L3 | P2 vs P3 on one knob: sessions need P3, data expiry (Kubernetes Events) needs P2 (§2.2) | **Avoided by omission** | No data-expiry primitive: liveness-scoped objects need a live client (§3.4). See §5.2 | Moderate |
| L4 | Checkpointing is Alpha, limited, and has #22649 (§1.4, §3.5) | **Avoided by omission** | Nothing to checkpoint. Chubby moved *away* from persisting sessions (Chubby §4.4, `:490`) | Moderate |
| L5 | Go #22081/#22094 and jetcd `onError`: the library keeps renewing a lease the application was told is gone (§5.2, §5.3) | **Prevented in substance** | I3: one library-owned session per client. No per-object keepalives with per-listener contexts; expiry is terminal for all handles (Chubby §2.8, `:281`). See §5.3 | Moderate |
| L6 | Optimistic client deadline: receipt time + TTL, checked once a second (§5.1). jetcd does the same (`LeaseImpl.java:263-265`) | **Another Chubby rule**, portable | A "conservative approximation" allowing for reply flight time and clock rate (Chubby §2.8, `:263`). Not tied to the held RPC; etcd clients could adopt it without a protocol change | High |
| L7 | No renewal load control; renewal rate = leases × 3 / TTL (§4.2, §6.1) | **Prevented** | I4: the master paces replies. etcd could retrofit a longer response TTL, since the Go client schedules from the *response* TTL (`client/v3/lease.go:542`), but not *held* replies. See §5.4 | High (mechanism); Moderate (retrofit) |
| L8 | Renewals forwarded by followers fail when the leader's apply backlog exceeds 1 s; leases of on-schedule clients expire (§6.1) | **Not a session property** | Chubby clients talk only to the master. Its proxies, which would reintroduce an intermediary, were not in production (Chubby §3, `:356`) | Moderate |
| L9 | Expiry cost: one Raft entry per lease, one transaction per lease's keys, a root token per revocation with auth (§6.2) | **Shared in structure** | Session expiry also writes: lock releases and ephemeral deletions. Chubby's workload (1k locks; 3 % of files ephemeral; Chubby §4.1) never stressed it, and the Kubernetes shared-lease pattern cannot exist there | Moderate |
| L10 | Mutual exclusion needs fencing (§2.3) | **Shared** | Sequencers (N1); etcd's create-revision compare is the equivalent. Chubby adds lock-delay for unfenced resources; etcd has nothing similar | High |
| L11 | A safe renewal needs proof of leadership. `ensureLeadership` is a ReadIndex; doing it per renewal cost 20 % more leader CPU at 1500 renewals/s (#18069, v3.5.11, arm64). #18428 skips it while the local tick is live, which opens R1 (§3.4) | **Not a session property** | The master lease. Reads, and by the same argument KeepAlive replies, are "satisfied by the master alone; this is safe provided the master lease has not expired" (Chubby §2.2, `:94`). See §5.4 | High (paper, issue); Moderate (the KeepAlive extension of the read rule) |

### 5.1 Failover safety (L1): the case the hypothesis cares most about

**Sessions do not bound a master's authority.** — **Moderate**.
- Nothing in F1–F5 does, so a held KeepAlive cannot stop a deposed master's stale decision from taking effect.
- Chubby relies on two other mechanisms:
  - **master leases:** a master serves only while the replicas promise not to elect another (Chubby §2.2, `:90`);
  - **epochs:** "The master rejects calls from clients using older epoch numbers" (`:305`).
- `leases-claude.md` §4.3 located etcd's flaw in exactly that gap:
  - expiry-driven revocations are decided on one member's timer but committed without a term;
  - the member's authority is measured in Raft ticks, while lease expiry runs on real time.

  Sessions would change neither half.

**What the paper does not show.** — **Moderate**.
- Epochs fence *client* calls. A deposed master's *own* pending work, such as a session-expiry decision that releases locks, is not discussed.
- That such work would fail is an inference. The master issues writes through consensus (Chubby §2.2, `:94`), and a proposal from a superseded Paxos coordinator loses to the new coordinator's ballot. But the paper never mentions ballots, and it calls the fail-over code "a rich source of interesting bugs" (Chubby §2.9, `:315`).
- In etcd, by contrast, Raft forwards a follower's queued proposal to the current leader, which commits it in its own term (`leases-claude.md` §3.2).
- The etcd analogue of an epoch check is a term tag on expiry-driven revocations, checked at apply. It was proposed in the #15247 thread and not shipped (`leases-claude.md` §3.4; B6).

### 5.2 One primitive, two jobs (L2–L4): the real simplification

Chubby's design is narrower, not cleverer. — **Moderate**.
- **Chubby's only timed state is client liveness.** P3 (do not take anything away from a live client) is the only promise it makes. P2 slack is acceptable, and the paper accepts it explicitly for ephemeral files (Chubby §2.9, `:313`).
- **etcd's lease is both a session and a timer for data.** The timer use (Kubernetes Events, #65497) needs P2. That need drove #9888 and the checkpoint feature, whose open bug #22649 breaks P3 when the feature is enabled.
- **The per-failover slack differs in size, for the same reason.** — **High** (paper and code); the attribution is mine, **Moderate**.
  - Both systems extend leases on fail-over, so P2 is unbounded under repeated master changes in both (L2).
  - Chubby's extension is at most one session lease period, "the maximum that the previous master may have been using" (Chubby §2.9, `:307`): 12 s by default, about 60 s under load.
  - etcd's Promote adds each lease's *full* TTL. For a data-expiry lease that can be hours; Kubernetes Events default to 1 h (`leases-claude.md` §7.1).
  - The reset is large only because the same primitive also carries long data timers.
- **A Chubby-style split would have removed that whole line of work:** sessions for liveness, and something else for data expiry. `leases-claude.md` §2.2 reaches the same point, as does Antigravity's lease analysis (as summarized in `INDEX.md`: "one primitive for sessions and TTL GC").

### 5.3 Client-side bookkeeping and deadlines (L5, L6)

**Why sessions remove a bug class (L5).** — **Moderate**.
- **The Go and jetcd bugs share one shape:** a keepalive continues in the library after the application was told it had ended.
  - In Go, this happens when a `WithRequireLeader` context's stream is closed (#22081, #22094).
  - In jetcd, `onError` is reported but renewal continues.
- **That shape needs a lease with several independent keepalive listeners,** each with its own end of life. A Chubby session has one owner (the library), one state machine (I3) and a terminal expiry, so there is nothing for a listener to drift away from.
- **An etcd client could adopt this on today's server.** It is a client-API design property: one library-owned session object, terminal on expiry, with no detachable per-lease listeners.

**A conservative deadline (L6) needs no held RPC.** Compute the client deadline from *send* time minus a drift margin. That is a one-line change in `client/v3/lease.go:543` and in jetcd's `LeaseImpl.java:265`. — **High**.

### 5.4 Renewal pacing and cost (L7, L11)

**A longer TTL under load is reachable in etcd; held replies are not.** — **High** (code); the retrofit **Moderate**.
- **Longer TTL.** The leader could return a longer TTL under load. The Go client already schedules its next keepalive from the response TTL (`:542`). The lessor would also need to set the matching expiry, and the API documents the response TTL as "the new time-to-live for the lease" (`leases-claude.md` §1.1).
- **Held replies are incompatible with today's clients,** although the bidirectional stream would allow them on the wire (`api/etcdserverpb/rpc.proto:131`).
  - The Go client advances `nextKeepAlive` only when a response arrives (`lease.go:542-556`). It re-sends every 500 ms while a renewal is overdue (`:586-610`; `retryConnWait` at `:86`).
  - It also drops the lease locally at receipt time + TTL (`:543`, `:562-583`).
  - So a held reply would turn each client into a 500 ms poller, and then expire its lease locally.
  - The same 500 ms resend loop is how #22094 renews an orphaned lease forever (`leases-claude.md` §5.2.3).
- **In short,** the held RPC makes L7 *inherent*, while etcd would have to *add* it.

**Message rate.** — **Moderate**. The rate model comes from the Codex analysis; the code facts were re-checked.
- Ignoring early replies for events, Chubby sends about one KeepAlive per session per lease period: *S/T* for *S* sessions and lease period *T*.
- etcd's Go client sends about three renewals per lease per TTL: *3L/T* for *L* kept-alive leases, multiplexed on one stream per client (`lease.go:586-619`).
- The gap is a factor of 3 from pacing, times *L/S* from renewing per lease. A process that keeps one lease per session gains only the factor of 3.
- Kubernetes never calls `KeepAlive` (`leases-claude.md` §7), so this load comes from session, lock and election users.
- Early replies that carry events raise Chubby's rate. These are message counts, not throughput predictions.

**Lengthening leases is not free.** Each extension granted under load also lengthens how long a dead client keeps its locks, and how long it stalls writers to files it cached (§2, correction 3). — **High** (follows from the mechanism).

**Cost per message.** — **Moderate**.
- A Chubby master can answer without consulting the replicas, because under its master lease "no other master can possibly exist" (Chubby §2.2, `:94`).
- An etcd leader has no such real-time promise, so a safe renewal needs a ReadIndex.
- #18069 measured that cost, and #18428 removed it at the price of window R1 (L11).

**Net.** etcd pays twice: more renewal messages, each costlier to make safe. Both savings come from Chubby mechanisms other than the session: server pacing (F5) and the master lease.

## 6. Watchers, issue by issue

Issue numbers and failure states are from `watchers-claude.md`; the section numbers in the first column refer to that file.

| # | etcd issue (watchers-claude §) | Verdict | Why | Confidence |
|---|---|---|---|---|
| W1 | Serving member cut off from the quorum: the watch stays open and silent unless `WithRequireLeader` is set (off by default) (§5.1) | **Prevented** | Delivery and liveness are one channel, from the one master that holds a master lease. A cut-off client enters jeopardy within its conservative lease (I2) | Moderate |
| W2 | Member with a leader but not applying: silent even with `WithRequireLeader` (§5.2) | **Prevented, by topology plus the coupling** | Chubby has no follower serving reads or events; the master is the only writer, and its state includes its own commits. An apply-stalled master stalls *writes*, not freshness | Moderate |
| W3 | Paused member or dead connection: the Go watch hangs with no keepalive (§5.4) | **Prevented** | The KeepAlive *is* the heartbeat; no extension means jeopardy (I2) | High (mechanism) |
| W4 | Freshness signals are member-local: `Created`, progress notifications; 10-minute default interval; requests dropped while any watcher is unsynced (§1, §2.3, §8.2; #20242) | **Prevented** | A KeepAlive reply with no invalidations is a **master-asserted** statement that everything the client caches is still current. In etcd terms: a linearizable progress notification plus a session renewal, at least once per lease period | High (mechanism); Moderate (equivalence) |
| W5 | gRPC proxy: ignores `WithRequireLeader` (reproduced), reuses dead upstream watches (#21813), drops `RequestProgress` (#19710) (§9) | **Not prevented by design** | Any intermediary must re-implement the coupling. The paper says Chubby's fail-over strategy "is not ideal for proxies" (Chubby §3.1, `:364`) and redesigned session recording partly for them (§4.4, `:488-492`) | Low–Moderate |
| W6 | Go client: final error dropped after 250 ms; #17611 teardown deadlock; #18879 cancellation-burst freeze (§8.1, §10.2) | **Partly simplified** | One sequential channel per session removes multi-stream teardown races. But Chubby's library was "7000 lines … and the client protocol is delicate" (Chubby §4.2, `:458`); there is no evidence it was bug-free | Low |
| W7 | Restore, compaction and cross-member-resume bugs, all fixed (§7, App. A) | **Avoided by omission** | Chubby has no history, no resume and no compaction. After fail-over the client flushes and rescans (Chubby §2.9 step 5, `:309`) | High |
| W8 | Event completeness across fail-over | **etcd is stronger** | Revisioned, resumable history is what Chubby's authors wished for: "or even to ensure that no events were lost during a fail-over" (Chubby §4.6, `:534`) | High |
| W9 | jetcd resume skip; 4 MiB response limit (§10.3, §10.4) | **Avoided by omission** | No resume. Whole-file reads with a 256 KB file limit (Chubby §4.5, `:514`) | Moderate |
| W10 | Slow or far-behind watchers stall the write path (#22264), or are starved (#16839) (§8.1) | **Inverted** | Chubby makes writes wait for slow cachers *by design*, bounded by the lease timeout (§2, correction 3). etcd's stall is lock contention in catch-up scans, unbounded but not intended. See §6.4 | Moderate |
| W11 | Watch-fed caches are eventually consistent: Kubernetes `resourceVersion=0` stale reads (#59848), and the KEP-2340 machinery needed to bound staleness (§11) | **Prevented for first-tier caches** | I1 and I5. Cost in §9 | Moderate |
| W12 | Loss of leader is terminal for watches that opted in. The Go client ends the watch on `ErrNoLeader` (`client/v3/watch.go:665-668`); new flagged streams are refused while the member has no leader (`v3rpc/interceptor.go:239-241`). Kubernetes' cacher then terminated all its watchers, and the re-creation storm destabilized large clusters (#111116) (§5.1, §6) | **Prevented in the library; reproduced by applications** | Jeopardy disables the cache but keeps the session; expiry comes only after a 45 s grace period (Chubby §2.8). Yet "many developers chose to crash their applications" on Chubby's fail-over event (Chubby §4.6, `:534`). See §6.2 | Moderate |

### 6.1 Freshness (W1–W4): the core of the case for the idea

**Chubby is an existence proof for the coupling etcd lacks.** `watchers-claude.md` concluded that the watch weakness is "no liveness or freshness coupling to Raft progress: no server-asserted 'applied ≥ recent ReadIndex', and no stream lease" (`watchers-claude.md` §12.2). Chubby's KeepAlive is precisely a stream lease issued by the authoritative server. So the coupling is buildable, and was found worth its cost at scale: 90,000 clients on one master (Chubby §3, `:349`).

**Kubernetes rebuilds part of it on the client side.** — **Moderate**.
- It does a linearizable `Get` for the current revision, then requests progress every 100 ms until the watch cache reaches that revision, for at most 3 s (`watchers-claude.md` §11.2).
- That is a *pull* version of W4, rebuilt by the consumer for each request. Chubby gets it once per lease period, by construction.

**W1–W4 are not etcd contract violations.** — **High** (documentation as quoted in `watchers-claude.md` §2.2).
- The v3.7 API guarantees page disclaims delivery time: "there is no limit and events in unhealthy clusters might never arrive". It also says "etcd does not ensure linearizability for watch operations".
- So a silent watch behind a lease that is still alive is permitted behaviour. Chubby makes freshness part of the contract (I1, I2); etcd leaves it to the consumer.
- The Chubby design would therefore not fix a bug. It would strengthen a contract that etcd deliberately left weak.
- **Caveat (2026-11-02).** "Not contract violations" means only that etcd documents no delivery guarantee these states could break. It does not mean that the contract is defensible.
  - The page does not define "unhealthy", and W1–W4 all occur in a cluster that keeps its quorum, where other members could serve the watch.
  - Read that way, the contract is safety-only.
  - It contradicts etcd's own description of watches as eventually consistent (`tests/robustness/README.md:111 @ v3.7.3`): "might never arrive" is the negation of eventual consistency.
  - Full analysis: [`etcd-issues-and-mitigations-claude.md`](etcd-issues-and-mitigations-claude.md) §3; short form in the `watchers-claude.md` §2.2 caveat.

### 6.2 Two failure stages versus one (W12)

— **High** (code, paper, issue text); **Moderate** (the transfer).
- **Chubby's library has two failure stages.**
  - In jeopardy, it stops trusting its cache and blocks calls, but keeps the session and its handles.
  - It gives up only when the grace period ends (Chubby §2.8, `:265-279`). A normal election fits inside the 45 s grace period: the paper reports 4–6 s typically and up to 30 s (Chubby §2.2, `:94`).
- **etcd's only freshness fail-safe has one stage.** `WithRequireLeader` turns "no leader" into a terminal `ErrNoLeader` (`watchers-claude.md` §6).
  - Kubernetes #111116 shows the consequence at scale. After three election cycles without a leader, the API server's cacher terminated all its watchers, and re-creating them overloaded the API server. Kubernetes fixed it in 2022 by making the cacher retry such errors for 30 s before it terminates its watchers (PR #111387; `staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go:451-454 @ v1.37.2`; added 2026-11-10): a grace period added on the application side.
  - The issue asked for a retry window before termination, i.e. a grace period. It was closed as completed in 2022-07; I did not check how it was resolved.
- **What an etcd equivalent would need.** A jeopardy-style state in the etcd client would mark caches stale **without** cancelling the watch, then resume from the last revision + 1 when a leader returns (B2). That combines Chubby's two stages with etcd's resumable history, which Chubby lacked (W8).
- **Chubby's own experience shows the library cannot do it alone.** Its fail-over event told applications to rescan, and "many developers chose to crash their applications on receiving this event, thus decreasing the availability of their systems substantially" (Chubby §4.6, `:534`). That is the same failure as #111116, one layer up.

### 6.3 History (W7–W9): Chubby avoids those bugs by lacking a feature worth having

— **High** (paper); the Kubernetes statement is my inference, **Moderate**.
- **etcd's watch is a replication feed:** ordered, resumable, history-backed. Chubby's events are notifications, and the paper treats them as such.
- **Chubby's attempts at publish/subscribe failed,** because "Chubby's heavyweight guarantees and its use of invalidation rather than update … make it a slow and inefficient for all but the most trivial publish/subscribe examples" (Chubby §4.5, pp. 13–14, `:516-524`).
- **Its fail-over event even hurt availability,** because "many developers chose to crash their applications on receiving this event" (`:534`).
- **Kubernetes' list-and-watch with `resourceVersion` cannot be built on Chubby's events.**

### 6.4 Slow watchers (W10): the inversion

— **Moderate**.
- **Both designs let a slow client degrade writers.**
- **Chubby makes it a *contract*, with a bound:** at most one lease period per write, during which reads are served uncached.
- **etcd has it as a *bug*, without a bound.** One watcher 500,000 revisions behind held `watchableStore.mu` against writers for more than 90 % of the time in #22264.
- **A designed, bounded stall is easier to operate than an accidental, unbounded one.** But Chubby's applies to every write to cached data, not only in pathological cases.

## 7. Locks and elections

| Aspect | Chubby | etcd v3.7.3 | Consequence |
|---|---|---|---|
| Where the lock lives | A server primitive: any node is a reader/writer lock (Chubby §2.4) | A recipe: the waiter's key under a prefix, attached to its session lease; the oldest create revision holds the lock (`client/v3/concurrency/mutex.go:75-106`). The server `Lock` RPC runs the same recipe in-process (§2) | — |
| How a waiter learns it may proceed | The master grants it, completing the blocked `Acquire()` | It watches the predecessor's key until it is deleted (`waitDeletes`, `client/v3/concurrency/key.go:25-64`) | **Progress depends on watch delivery** (note 1) |
| Waiter whose own session dies while waiting | Session expiry ends the session's pending operations (Chubby §2.8, `:281`) | It keeps waiting for its predecessors (#19597, open; note 2) | A liveness gap, closed only at the end |
| Server-side lock service | — | `v3lock`, on the in-process client; it orphans the caller's session and unlocks with a plain delete (note 3) | Thin clients such as jetcd inherit its exposure |
| Release by a stale or duplicated request | Instance and handle sequence numbers guard against delayed packets (note 4) | `Unlock` deletes by a key name reused across acquisitions (#22082, open; note 4) | A stale retry can release a newer lock |
| Holder learning it lost the lock | Jeopardy, then expired, from the session state machine; handles fail (`:279-281`) | `Session.Done()` closes at the optimistic local deadline, or on a "not found" renewal (note 5) | The etcd holder's view of ownership is optimistic, typically by about 1 s (L6) |
| Hand-over after holder failure | Lease expiry, then **lock-delay** of up to 1 min (`:159`) | Lease expiry, then an immediate grant via the waiter's watch | etcd gives unfenced resources a smaller safety margin. Unrelated to KeepAlives |
| Fencing | Sequencer: lock name, mode and generation, checked by the resource server (Chubby §2.4) | Create-revision compare in a `Txn` (`Mutex.IsOwner`, `Election.Proclaim`; `leases-claude.md` §2.3) | Equivalent for resources inside etcd. For outside resources, both depend on the resource checking the token |

**Notes.**
1. **Watch-dependent progress.** A waiter whose watch sits on a cut-off member never wakes, and the default context carries no `WithRequireLeader`. Safety is kept: after waking, the waiter re-reads its own key and returns `ErrSessionExpired` if it is gone (`mutex.go:96-104`).
2. **Expired waiter.** A TODO notes the missing "early termination if the session key is deleted" (`mutex.go:88`); #19597 tracks it. The re-check in note 1 closes the gap only after the predecessors are gone.
3. **`v3lock`.** It uses the in-process adapter client (`v3client.go:30-39`), the path on which the #18879 cancellation-burst freeze was reported (`watchers-claude.md` §8.1).
   - `Lock` builds a session on the caller's lease ID and calls `s.Orphan()` before waiting (`v3lock/lock.go:35-49`), so the server never keeps that lease alive.
   - `Unlock` is a plain `Delete` of the key (`:52-58`).
   - jetcd's `LockImpl` and `ElectionImpl` call these services. The lock's liveness is therefore the caller's own lease keepalive, and jetcd's renew-after-`onError` behaviour (L5) carries over to locks held through it (inferred).
4. **Stale release.**
   - **Chubby:** nodes carry an instance number, "greater than the instance number of any previous node with the same name" (Chubby §2.3, `:124`). Handles carry a sequence number, and a closed handle is remembered so that "a delayed or duplicated network packet" cannot recreate it (Chubby §2.3, `:134`; §2.9 step 8, `:312`). These identifiers are not part of the session design, but they show the paper designing against the same delayed-packet hazard. The paper does not say whether `Release` checks them.
   - **etcd:** the lock key is `<prefix><lease ID>` (`mutex.go:115`), so one session gets the same key name on every acquisition. `Unlock` deletes by name without the `IsOwner()` compare (`mutex.go:134-150`). If a delete commits but the client sees an error, a retry from the old `Mutex` deletes a newer acquisition's key. The fix is local: guard the delete with `IsOwner()` (`mutex.go:152-153`).
5. **Loss signal.** `Session.Done()` closes when the lease keepalive channel closes (`client/v3/concurrency/session.go:58-73`, `:97`; `client/v3/lease.go:534-543`, `:562-583`).

**Assessment.**
- **Liveness.** The prompt is right that Chubby's lock is simpler to use correctly: one server-side wait, tied to the one session, with an explicit loss signal. Most of etcd's lock *liveness* hazards come from building the wait out of a watch, and they would disappear. — **Moderate** (inferred from code).
- **Safety** is the same in both designs: fencing is required (N1). — **High**.

## 8. A natural experiment inside etcd: `client/v3/leasing`

etcd ships a client-side protocol that copies Chubby's invalidating cache (Chubby §2.7): `client/v3/leasing`.
- **Also served by the proxy.** The gRPC proxy serves it to many clients. Its flag `--experimental-leasing-prefix` is described as "leasing metadata prefix for disconnected linearized reads" (`server/etcdmain/grpc_proxy.go:179`, used at `:488`).
- **Why it matters here.** It shows directly what Chubby's protocol becomes without the coupling.

### 8.1 The protocol (verified in code)

| Chubby §2.7 step | `leasing` equivalent | Evidence |
|---|---|---|
| Client caches a file; the master records it | **Acquire.** On a linearizable `Get`, the client puts an ownership key `<prefix>/<key>`, attached to its own `concurrency.Session` lease, in the same transaction as the read | `client/v3/leasing/kv.go:265-299` (`acquire`), `:301-338` (`get`) |
| Master sends invalidations; write blocked | **Revoke.** A writer that is not the owner writes `"REVOKE"` into the ownership key, then waits for that key to be **deleted** | `kv.go:413-421` (`revoke`), `:204-216` (`waitRescind`) |
| Client flushes and acknowledges via its next KeepAlive | **Rescind.** The owner's per-key watch sees `"REVOKE"`, evicts the entry and deletes the ownership key | `kv.go:159-188` (`monitorLease`), `:191-202` (`rescind`) |
| …or the client's cache lease expires | **Lease expiry.** If the owner's session lease is revoked, the ownership key goes with it, and the writer's watch sees the delete | Lease semantics; `waitRescind` returns on any `DELETE` |
| Client cache valid only while its session is | Cached reads are served only while `Session.Done()` is open. There is no other validity check | `kv.go:306-308` (`readySession`), `:467-478`; `client/v3/leasing/cache.go:191-210` |
| Many clients may cache the same file (the master keeps "a list of what each client may be caching") | **One owner per key.** Acquisition requires that no ownership key exists (`CreateRevision(<prefix>/<key>) = 0`). Other clients read through to the server; after a revoke, a client waits 2 s before trying to acquire again | `kv.go:270-280`; `cache.go:28` (`revokeBackoff`), `:113-118` (`MayAcquire`) |

So the shape matches Chubby: a write waits for acknowledgement or lease expiry. The difference is that **the acknowledgement channel is the owner's watch and the liveness channel is the owner's lease keepalive**: two independent streams (§3.2).

### 8.2 Failure modes (inferred from code; (b) also reported upstream with a test)

**(a) A writer can block for as long as the owner's watch is stalled.** — **Moderate**.
1. The owner's lease keepalive works, while its `monitorLease` watch is on a member that is cut off from the quorum or not applying.
   - With round-robin, the two streams can be on different members (§3.2).
   - The watch context derives from the leasing KV's own context, with no `WithRequireLeader` (`kv.go:160`, `:174`).
2. The owner never sees `"REVOKE"`, and its lease never expires.
3. The writer's `waitRescind` has no timeout of its own, so its `Put` blocks until its context ends or the stall clears (`kv.go:236-262`, `:204-216`).

Chubby excludes this: an owner whose delivery channel is dead cannot renew (§3.3).

**(b) A cut-off owner can serve stale "linearizable" reads: about 1 s typically, unbounded under process pauses.** — **High** (existence); **Moderate** (the window sizes, which are inferred).
1. The owner is cut off entirely. The leader expires its lease at *last renewal + TTL*.
2. The revocation deletes the ownership key, and the waiting writer's next transaction commits (`kv.go:218-234`).
3. The owner's `Session.Done()` closes only when the Go client's local deadline passes: *last response receipt + TTL*, checked once a second (`client/v3/lease.go:543`, `:562-583`). That is one-way latency plus up to 1 s later.
4. Meanwhile, `get` serves the old value from cache.

This is the "disconnected linearized read" the proxy flag advertises, and the optimistic deadline (L6) makes it unsound in a short window.
- Chubby's conservative client lease and jeopardy rule (I1, I2) exist to close exactly this window.
- A pause between the check and the use defeats any lease-based cache, Chubby's included.

**Upstream history of (b)** (read 2026-11-02; found while checking another analysis's lead):
- **Reported.** An outside contributor reported it as #19091 ("Client lease library does not check for expiration", 2024-12-19). The report included an integration test (a three-member cluster with one member cut off) that "sometimes fails, seeing the stale 'abc'".
- **First fix.** PR #19092 explained the polling part of the cause: the `Done()` channel "is closed by a background loop that sleeps and periodically checks whether the lease is expired".
- **Closed by the bot.** The stale bot closed both the issue and the PR on 2025-05-17. The issue had no human reply. The PR had one maintainer review comment, on naming (2024-12-21), and was never approved for testing (corrected 2026-11-10: this line said "without maintainer review").
- **Second fix.** #21589 (2026-04) was closed unmerged by its author on 2026-06-28.
- **Unchanged.** The code is the same at `v3.7.3` and on `origin/main` (`a2e3642bc`).
- **What the fixes would leave open.** Both fixes compare the current time with the client's deadline. Since that deadline is receipt time + TTL (L6), they would close the 1 s polling gap but not the reply's flight time; B1 closes both.

— **High** (issue and PR text); **Moderate** that the fixes leave the flight-time gap (inferred from their descriptions; their code was not read).

**(c) Invalidation is voluntary.** — **High** (by construction).
- The protocol lives in the client. A plain client writing the same key bypasses it, and leasing owners keep serving the old value until something else evicts it.
- Chubby enforces invalidation in the server, for all writers (I5).

### 8.3 Costs and defects of the construction (functional tests)

The Antigravity analysis estimated the Raft cost of this protocol. Checking its count led to the measurements below.

**Code facts.** — **High** (code).
- **A cache miss is a Raft write.**
  - `acquire` is a `Txn` whose success branch puts the ownership key (`kv.go:270-280`).
  - A transaction with any non-`Range` operation in either branch is not read-only (`server/etcdserver/txn/txn.go:352-365`). So it goes through Raft even when the compare fails, i.e. when another client already owns the key.
  - Nothing records the failure, so a non-owner pays this on **every** linearizable read of that key, except within 2 s of a revoke (`MayAcquire`).
  - A plain client's linearizable read costs a ReadIndex but appends nothing to the log.
- **Range reads are never cached.** `isBadOp` sends any range or revision read to the server (`cache.go:189`). Chubby caches directory listings and the absence of files (Chubby §2.7, `:229`).
- **The write path applies the operation twice.**
  - `put` and `delete` first try `tryModifyOp`, a `Txn` guarded by "no ownership key, or my own" (`kv.go:218-234`).
  - If the writer does not own the key in its local cache (`wc == nil`), they then call `revoke` **unconditionally**, even when `tryModifyOp` has already succeeded (`kv.go:241-244`, `:389-392`).
  - `revoke` is a second `Txn` that applies the same operation again if no ownership key exists (`kv.go:413-421`).

**Functional test** (2026-11-02; setup in the header table). Two `leasing.NewKV` clients, A and B, share the prefix `pfx/`. A plain client watches the data keys and reads the server's `etcd_server_proposals_committed_total`.

| Case | Raft proposals | Revisions | Key `Version` | Watch events seen by a plain watcher | Response to the caller |
|---|---|---|---|---|---|
| B puts `k1`; nobody owns `k1` | 2 | +2 | 0 → 2 | Two `PUT k1=v1` | — |
| A owns `k2` (by reading it); B puts `k2` | 5 | +4 | 1 → 3 | Two `PUT k2=v2` | — |
| B deletes `k3` with `WithPrevKV`; nobody owns `k3` | — | — | — | — | `Deleted=0`, no `PrevKvs`; the key is gone |
| B puts `k4="new"` with `WithPrevKV`; previous value `"old"` | — | — | — | — | `PrevKv.Value="new"`: B's own first write |
| A owns `k`; B reads `k` five times (linearizable), then five times with `WithSerializable`; then A reads it five times (separate run) | 5, then 0, then 0 | 0 | — | — | Values correct |

**Consequences.**
- **Every write from a client that does not own the key is applied twice.** Watchers see a duplicate event, `Version` advances by two, and `Delete` and `WithPrevKV` results describe the second, redundant operation. — **High** (reproduced).
- **A contested write costs five proposals,** plus two watch round trips. — **High** (count reproduced); the Chubby side is from the paper (Chubby §2.7).
  - The five are: the failed `tryModifyOp`, the `REVOKE` put, the owner's rescinding delete, the successful `tryModifyOp`, and the duplicate from `revoke`.
  - Chubby's master does the same job with in-memory bookkeeping and one KeepAlive round trip per caching client.
- **Status.** Unchanged on `origin/main` (`a2e3642bc`). Searches of etcd issues found no report of the double application (**Moderate**: search terms can miss one).
- **No upstream test covers it.**
  - The two integration tests that assert `Version` both go through the owner path (`tests/integration/clientv3/lease/leasing_test.go:321-363` `TestLeasingConcurrentPut`, `:473-502` `TestLeasingOwnerPutResponse`).
  - `TestLeasingDeleteNonOwner` discards the `Delete` response (`:416-443`).

**Other defects in the package** (code and issue states checked 2026-11-02):
- **`EvictRange` deletes the wrong key.** It deletes `key`, the range start, instead of the loop variable `k` (`cache.go:178-187`). — **High** (code).
  - So when a range delete fails with an unknown outcome, the other cached keys in the range stay cached, and can be served although they may have been deleted.
  - Two PRs proposed the fix and were closed unmerged (#21869, #21589). It is unchanged on `origin/main`.
  - The Antigravity analysis presented this as new; it is not.
- **Open issues:**
  - #22083: a `Delete` with `WithFromKey` removes the protocol's own keys;
  - #22084: a race in `LockWriteOps`;
  - #22417: stale `Count` after a cached delete.

**Reading.** — **Moderate**.
- The protocol is expensive in Raft writes, and its exclusive ownership gives up the many-reader caching that made Chubby's design pay.
- It has basic correctness defects in its write path that no test catches, and its stale-read and `EvictRange` fixes went unreviewed.
- That is consistent with little or no production use; adoption itself is unknown (§12).

### 8.4 What the experiment shows

- **etcd's primitives can express Chubby's *protocol*, but not its *invariant*.** — **Moderate**.
  - The invariant needs the server to make renewal conditional on acknowledgement.
  - In etcd, renewal is decided on the leader, while acknowledgements would arrive on watch streams at any member. The server has no place where both meet.
- **The construction is costly.** — **High** (§8.3); the generalization is **Moderate**.
  - Chubby's invalidation is cheap because the master keeps the cacher list in memory, and the acknowledgement rides on a message the client sends anyway.
  - Rebuilt from replicated keys and watches, every step of the protocol becomes a consensus write.
- **The opposite approach exists.** The experimental `cache/` package is a pull-based consistent cache that relies on `RequestProgress`, the KEP-2340 pattern. Its README warns: "gRPC proxy is not supported. The cache relies on `RequestProgress` RPCs, which the gRPC proxy does not forward" (`cache/README.md:5`).

## 9. Performance: what the coupling costs

The coupling is not free, and its costs fall where etcd's workload is heaviest. None of this was measured; each item is inference from the paper and the cited analyses.

1. **Writes wait for every cacher.** — **Moderate**.
   - *Chubby:*
     - writes are delayed "by up to tens of seconds if a recently-failed client cached the file" (`:440`);
     - this was affordable because `SetContents` was 680 ppm and `Acquire` 31 ppm of RPCs (Chubby §4.1 table);
     - the wait is the *maximum* over the cachers, not the sum, since invalidations go out in parallel;
     - under repeated failures or continuing lease extensions, it has no finite bound.
   - *etcd and Kubernetes:*
     - the etcd-level cachers in Kubernetes are the API servers' watch caches (`watchers-claude.md` §11);
     - each caches every object of each resource it serves, so with Chubby semantics **every write would wait for every API server**;
     - one API server stalled by GC, CPU starvation or a slow network would stall writes across the cluster for up to a lease period. Today the same stall only delays that API server's own view.
2. **Invalidate, never update.** — **Moderate**.
   - *Chubby:* the paper chose invalidation because "update-only protocols can be arbitrarily inefficient" (Chubby §2.7, `:243`), and it is unsuitable for publish/subscribe (Chubby §4.5, `:524`).
   - *etcd and Kubernetes:*
     - a watch cache is a full replica and needs values, not invalidations;
     - each write would cost each cacher an extra read, and Kubernetes also needs `prev_kv` for its own watchers;
     - Chubby's model fits small, read-mostly, widely shared files, while etcd's main workload is the opposite.
3. **Uncached reads during invalidation.** — **Low–Moderate**.
   - *Chubby:* the node is served uncached while invalidations are outstanding, and the paper notes that over-eager clients may "bombard the master" (Chubby §2.7, `:241`).
   - *etcd:* hot keys read by many clients (configuration, endpoints) would send a burst of reads to the leader on every write.
4. **Everything goes through the master.** — **Moderate**.
   - *Chubby:* KeepAlives were 93 % of RPCs (Chubby §4.1). Scaling relied on longer leases, proxies and partitioning (Chubby §3).
   - *etcd:* it spreads watches and serializable reads across members. Requiring the leader to answer every session's deliveries and acknowledgements would concentrate load where etcd already funnels writes, lease renewals and `ReadIndex`.
5. **Fail-over barrier.** — **Moderate**.
   - *Chubby:* the new master blocks operations until all sessions acknowledge or expire; in the revised design, for a full worst-case lease timeout (Chubby §2.9, `:310`; Chubby §4.4, `:490`). A typical fail-over took 14 s (Chubby §4.1 table).
   - *etcd:* it serves writes as soon as a leader is elected (about 1–2 s with defaults; recalled, not measured here). A Chubby-style barrier would add a lease timeout to every election.
6. **Transport.** — **Low**.
   - *Chubby:* coupling time-bounded KeepAlives to TCP "led to many lost sessions at times of high network congestion", and KeepAlives moved to UDP (Chubby §4.6, `:544`).
   - *etcd:* keepalives and watches share HTTP/2 over TCP, possibly on one connection. Whether large responses can delay keepalive frames enough to expire leases is untested here; it is kin to the watch starvation in #15402 (`watchers-claude.md` §8.1).
7. **Thick client.** — **Moderate**.
   - *Chubby:* the client library was "7000 lines (comparable with the server), and the client protocol is delicate". Java users ran a protocol-conversion server instead of a port (Chubby §4.2, `:458`).
   - *etcd:*
     - a Chubby-style protocol would push every language binding toward a large, delicate state machine, or toward a conversion server;
     - etcd's wire protocol is thin, but its *correct use* is not. Lease keepalive bookkeeping, watch resume cursors and lock recipes live in each client library;
     - jetcd re-implements the first two, with its own defects: receipt-time deadline (L6), renewal after `onError` (L5), resume skip (W9). It delegates locks to `v3lock`/`v3election`, which play the conversion-server role;
     - so etcd also has delicate client state; it is spread across separate mechanisms and duplicated per language, rather than concentrated in one session state machine.

**Net.** — **Moderate** (inference; nothing measured).
- For the watch-freshness problems, the coupling trades *silent, unbounded staleness* for *visible, bounded write latency tied to client health*.
- For Chubby's workload (coarse-grained locks, names, configuration; `SetContents` plus `Acquire` about 0.07 % of RPCs), that trade is clearly right. The paper defends it on usability grounds: "we felt that programmers would find them harder to use" (Chubby §2.7, `:245`).
- For Kubernetes on etcd, write-heavy with few but critical cachers, the same trade would turn any API-server hiccup into a cluster-wide write stall. Applied wholesale, it is the wrong trade there.

**Where this would fail.** For read-dominated etcd uses outside Kubernetes, such as service discovery and configuration distribution, the cost argument weakens and the case for Chubby-style caching strengthens. That is the niche `leasing` targets.

## 10. What etcd could borrow without coupling writes to clients

These are directions, not recommendations, and none was prototyped. Each keeps etcd's write path independent of client health.
- **B1–B5** aim at I2 and I3 (detectable staleness, one liveness state) rather than I1 and I5 (server-enforced invalidation).
- **B6** borrows a mechanism that is not part of the session design but sits next to it in the paper.

**B1. Conservative client lease deadline.** — **High** (feasible); **Moderate** (sufficient).
- *Change:* compute the deadline from send time minus a drift margin, not from receipt time (`client/v3/lease.go:543`; jetcd `LeaseImpl.java:265`). Layer: client.
- *Buys:* closes the `leasing` stale-read window except under pauses (§8.2 b), and makes `Session.Done()` err on the safe side (L6).
- *Fails when:* the process pauses. Any lease-based client cache shares this limit.

**B2. A client session object with jeopardy.** — **Moderate**.
- *Change:* one client-side object (Go, jetcd) owns one lease and all watches, with states safe → jeopardy → safe or expired.
  - It enters jeopardy when the keepalive deadline is near, *or* when a watch cannot prove freshness within a bound (linearizable `Get` plus `RequestProgress`, the KEP-2340 pattern).
  - In jeopardy, cached reads (`leasing`) block and the application is told.
  - Watches are *not* cancelled. They resume from the last revision + 1 once freshness is proven again, and expiry comes only after a grace period.
- *Buys:*
  - I2 and I3, built from existing RPCs;
  - a fix for the L5 bug shape by construction;
  - Chubby's two stages in place of the terminal `ErrNoLeader`, while keeping etcd's resumable history (W12).
- *Costs:* `RequestProgress` goes unanswered while any watcher on the stream is unsynced (#20242), and the proxy drops it (#19710). It also costs one `ReadIndex` per client per bound period.

**B3. A leader-asserted revision in keepalive responses.** — **Low–Moderate**.
- *Change:* fill the response with the leader's revision, taken after the leadership check, so that B2 needs no separate linearizable read. Today the header is the receiving member's revision, filled *before* the renewal (`v3rpc/lease.go:133-134`; `leases-claude.md` §2.5). Layer: server and API semantics.
- *Buys:* it makes the lease keepalive a freshness heartbeat, which is what W4 lacks.
- *Costs:*
  - it changes the meaning of an existing field;
  - forwarded renewals must carry the leader's value back;
  - it bounds how fresh the client *should* be, not how fresh its watch member *is*.

**B4. Separate session leases from data-expiry leases.** — **Moderate**.
- *Change:* two kinds of lease on the server:
  - session leases (P3: extended on Promote, keepalive allowed);
  - data-expiry leases (P2: deadline semantics, no keepalive, no Promote extension).
- *Buys:* it removes the one-knob conflict (L3), and the need for checkpointing for data expiry (L4).
- *Costs:* it needs a persisted deadline, i.e. wall-clock time in replicated state, with its own clock-skew questions. It is an API addition, not a fix to existing leases.

**B5. Optional lock-delay in `v3lock`.** — **Moderate**.
- *Buys:* Chubby's mitigation for unfenced resources (§7).
- *Costs:* it only helps clients of the server lock API.

**B6. Term-fenced expiry revocations.** — **Moderate** (inferred from `leases-claude.md` §3.4).
- *Change:* tag each expiry-driven `LeaseRevoke` with the proposer's Raft term, and drop it at apply if that term is older than the term of the log entry carrying it. This is the etcd form of Chubby's epoch, applied to the server's own work rather than to client calls. It was proposed in the #15247 thread and not shipped (`leases-claude.md` §3.4). Layer: server, WAL and proto.
- *Buys:* it closes the P3 windows in which a revocation decided in one term commits in a later one: R5 directly, and the stale-batch cases of R1 and R2 (L1).
- *Costs:*
  - a WAL and proto change, with mixed-version and replay questions;
  - a con cited in the thread: "Using term information inside the state machine is unusual";
  - it does not touch R3 (an ex-leader acknowledging renewals) or R4 (no leader change).

**What no borrowing gives.** The full Chubby invariant (I5) would require tying renewal to acknowledgement in the server: lease renewal at the leader would depend on watch acknowledgements at arbitrary members. I see no cheap form of that in etcd's architecture (§8.4). — **Moderate**.

## 11. Limits of this analysis

- **The paper is not the implementation.** Chubby's details may differ from the 2006 description; for example, the paper does not say how a blocked `Acquire()` survives fail-over. Every Chubby claim here is a claim about the paper.
- **Imported findings.** Issue states and the L/W issue lists come from `leases-claude.md` and `watchers-claude.md` as of 2026-11-01. If those are wrong, or upstream has changed, rows in §5–§6 change with them.
- **Per-stream placement.** §8.2 (a) and the W1 framing assume that lease and watch streams can land on different members.
  - That holds with the default resolver and several endpoints.
  - With one endpoint, or behind a proxy, both streams share fate. The owner's lease then expires and the writer unblocks after about a TTL, so (a) degrades to a TTL-length stall.
- **The Kubernetes cost argument (§9).** It assumes the API servers are the only etcd-level cachers and that each caches every key it serves. Both come from `watchers-claude.md` §11 and general Kubernetes architecture, not re-verified here. With partitioned watch caches, the write fan-out shrinks.
- **"Prevented" means by design, not bug-free.** Chubby's own fail-over code was "a rich source of interesting bugs" (Chubby §2.9, `:315`), and Chubby §4.4 records a design flaw in exactly the invariant this file centres on (§3.3). A Chubby-style etcd would have its own bugs in the coupling.
- **The master-lease and ballot arguments (L1, L11) go beyond the paper.** It states the master-lease read rule (Chubby §2.2, `:94`) but not how a deposed master's pending writes are rejected. If Chubby's implementation allowed a stale coordinator's proposal to commit, as Raft forwarding does in etcd, L1 would be shared rather than avoided.
- **The functional tests (§8.3) were single runs on a one-member cluster.**
  - The double application follows from a code path that depends on neither cluster size nor timing, so one member suffices to show it.
  - The contested-write proposal count could differ under contention, through retries.

## 12. Open questions and proposed tests

Functional tests can run on this host (Docker, official `v3.7.3` image). Their timings are not benchmarks (AGENTS.md §7).

| # | Test | Expected if this analysis is right |
|---|---|---|
| T1 | `leasing` writer block. Three members, and an owner client with all three endpoints. Confirm from per-member gRPC stream metrics that the lease and watch streams are on different members. Cut the watch member off from its peers but not from the client; another client then `Put`s the leased key | The `Put` blocks until the partition heals; the owner's lease stays alive throughout |
| T2 | `leasing` stale read. Cut the owner client off completely; a writer `Put`s; the owner keeps calling `Get`. The test in #19091 (`TestLeasingGetChecksForExpiration`, in the reporter's fork) already does this | The owner returns the old value for up to about latency + 1 s after the writer's commit |
| T3 | Same as T2, after applying B1 (send-time deadline minus a margin) in a worktree build | The window disappears, except under injected pauses |
| T4 | Keepalive starvation: large concurrent `Range` or watch responses on the same connection as a lease keepalive stream (the analogue of Chubby §4.6) | Unknown; tests the Low-confidence transport item in §9 |
| T5 | **Done 2026-11-02** (§8.3): non-owner `leasing` writes | Observed: applied twice; wrong `Delete` and `WithPrevKV` responses; five proposals for a contested write |

**Open questions:**
- Did etcd's designers consider and reject a session primitive? Upstream discussions were not searched for this file.
- How much of Kubernetes' KEP-2340 machinery would B2 and B3 make redundant, and at what `ReadIndex` cost per API server?
- Is `client/v3/leasing` used in production anywhere?
  - Its proxy flag still carries the `experimental` prefix.
  - Every write from a non-owner is duplicated, and two stale-read fixes went unreviewed (§8.3), which suggests little use. Adoption itself is unknown.
- Whether to report the `leasing` double application upstream is the user's decision. AGENTS.md rules out PRs against `etcd-io/*`; an issue report is outward-facing and was not filed.

## Conclusions

- **On the mechanism.** Chubby's decisive property is that **a session cannot be renewed past an unacknowledged invalidation**. Held RPCs and piggybacking are its 2006 transport, and the authors were ready to change that transport (§3.3). etcd has push without acknowledgement, and liveness separate from delivery. — **High**.
- **On watchers.** — **Moderate**.
  - **Largely right.** The dominant production failure at v3.7.3, a watch that goes silent while the client otherwise looks healthy, cannot happen to a Chubby client for longer than one conservative lease period. Kubernetes' consistent-read machinery is a per-request, client-built substitute.
  - **Wrong on completeness.** Chubby loses events at fail-over, and etcd's resumable history is better.
  - **Most portable piece:** Chubby's two-stage failure handling, stop trusting first and give up later. etcd's opt-in fail-safe has one terminal stage, and Kubernetes paid for it in mass watch teardowns (#111116).
- **On leases.** — **Moderate**.
  - **Wrong for the failover-safety bugs (P3).** They come from term-less revocations and tick-measured authority. In the paper, the defence against them is master leases and epochs, not sessions, and the paper does not show how a deposed master's own pending work is fenced.
  - **The master lease also makes each renewal cheap to make safe,** which etcd lacks (#18069 → #18428).
  - **Right that standalone lease objects cost something:**
    - one primitive serves both liveness and data expiry, creating the P2/P3 conflict and the checkpoint machinery;
    - per-object keepalives create the client bookkeeping bugs and multiply renewal traffic.
  - **P2 is shared:** Chubby also extends leases on fail-over.
- **On locks.** — **Moderate** / **High**.
  - Chubby's server-side wait removes etcd lock waiters' dependence on watch delivery (#19597).
  - Chubby's per-incarnation identifiers guard against the stale-release bug etcd has (#22082).
  - Neither design gives mutual exclusion without fencing.
- **On performance.** Wholesale adoption would make etcd's write latency depend on the health of every caching client, which in Kubernetes means every API server. Chubby could afford that because its writes were rare. — **Moderate**.
- **Practical upshot.**
  - **The predicted failures are real.** etcd's own `leasing` package shows that Chubby's protocol without Chubby's invariant fails in exactly the predicted ways (§8.2).
  - **The rebuild is costly and buggy.** Rebuilding the protocol from replicated keys turns each step into a consensus write, and as shipped the package applies every non-owner write twice (§8.3).
  - **The cheapest borrowings are client-side:**
    - a conservative deadline (B1);
    - a single session state machine that enters jeopardy on lost freshness without cancelling watches (B2).
  - **On the server, the comparison most directly suggests two changes:**
    - separating session leases from data-expiry leases (B4);
    - term-fencing expiry revocations, the etcd form of Chubby's epochs (B6).

---

## Appendix A. Method, inputs and sources

**Method.**
- The paper, plus code reading at `v3.7.3` of:
  - the Go client: `client/v3/lease.go`, `client/v3/concurrency/{session,mutex,key}.go`, `client/v3/leasing/{kv,cache,txn,doc}.go`, `client/v3/watch.go` (the `ErrNoLeader` path) and `client/v3/internal/resolver/resolver.go`;
  - the server: `server/etcdserver/api/v3rpc/{lease,interceptor}.go`, `server/etcdserver/api/v3lock/lock.go`, `server/etcdserver/api/v3election/election.go`, `server/etcdserver/api/v3client/v3client.go`, `server/etcdserver/txn/txn.go`, `server/embed/serve.go`, `server/etcdmain/grpc_proxy.go` and `server/lease/lessor.go`;
  - jetcd's `LeaseImpl.java`;
  - `cache/README.md`.
- Upstream issue and PR states were read through the GitHub REST API on 2026-11-02.
- Apart from the functional tests in §8.3, nothing was run or reproduced.
- "Inferred" means derived from code reading only. Confidence tags (High / Moderate / Low) apply to the claim they follow.

**Inputs.**
- **Prior analyses:** `analysis/leases-claude.md` and `analysis/watchers-claude.md` (both revised 2026-11-01). They are the source of the "identified issues". Their findings, issue states and confidence levels are used as stated there, unless this file says it re-checked them.
- **Cross-checked analyses:** `analysis/chubby-perspectives-codex.md` (Codex) and `analysis/chubby-perspectives-agy.md` (Antigravity), as of 2026-11-02. They were used only as leads. Every claim taken from them was re-verified against the paper, the code or upstream before inclusion. Appendix B lists what was taken, what was rejected and why; this file does not depend on them. The objections to the Antigravity analysis are written up for its author in `analysis/chubby-perspectives-critique-claude.md`.

**Citation conventions.**
- "Chubby §x.y, p. N" is a section and page of the paper.
- `:L` is a line in `doc/chubby-osdi06.md` when it follows a paper reference. When it follows a code path, it is a line in that file.
- A bare "§N" refers to this file. "`leases-claude.md` §N" and similar refer to the named file.

**Chubby paper** (`doc/chubby-osdi06.md`), lines cited:
- §2.1 p. 3: `:74-80`
- §2.2 p. 3: `:90`, `:94`
- §2.3 p. 4: `:118`, `:122-124` (instance numbers), `:131-135` (handles)
- §2.4 p. 5: `:155-159`
- §2.5 pp. 5–6: `:163-182` (child events and reference counts `:166`; event ordering `:172`)
- §2.7 pp. 6–7: `:229-249`
- §2.8 pp. 7–8: `:253-281`
- §2.9 pp. 8–9: `:285-315`
- §3 p. 10: `:349-356`
- §3.1 p. 10: `:360-364`
- §4.1 p. 11: `:393-440`
- §4.2 p. 12: `:454-458`
- §4.3 p. 12: `:468-472`
- §4.4 pp. 12–13: `:478-492`
- §4.5 pp. 13–14: `:514-524`
- §4.6 p. 14: `:530-546`

**etcd `v3.7.3` code read for this file:**
- `client/v3/lease.go:86`, `:376-411`, `:519-583`, `:586-619`
- `client/v3/watch.go:665-668`
- `client/v3/internal/resolver/resolver.go:44`
- `client/v3/concurrency/session.go:26`, `:41-97`
- `client/v3/concurrency/mutex.go:75-153`
- `client/v3/concurrency/key.go:25-64`
- `client/v3/leasing/doc.go`; `client/v3/leasing/kv.go:129-460`; `client/v3/leasing/cache.go:28`, `:44-118`, `:178-210`; `client/v3/leasing/txn.go` (function list only)
- `tests/integration/clientv3/lease/leasing_test.go:321-502`
- `api/etcdserverpb/rpc.proto:131`
- `server/etcdserver/api/v3rpc/lease.go:112-147`; `server/etcdserver/api/v3rpc/interceptor.go:232-262`
- `server/etcdserver/txn/txn.go:352-365`
- `server/etcdserver/server.go:908-918` (`isActive`, `ensureLeadership`; existence only)
- `server/lease/lessor.go:290-291`
- `server/etcdserver/api/v3lock/lock.go:31-58`; `server/etcdserver/api/v3election/election.go:75-136`
- `server/etcdserver/api/v3client/v3client.go:30-48`
- `server/embed/serve.go:150`
- `server/etcdmain/grpc_proxy.go:179`, `:488`
- `cache/README.md:5`
- On `origin/main` (`a2e3642bc`): `client/v3/leasing/kv.go:241-244`, `:389-392` and `cache.go:178-186`, unchanged.

**jetcd `0.8.7`:**
- `jetcd-core/src/main/java/io/etcd/jetcd/impl/{LockImpl,ElectionImpl}.java` (use of the server Lock and Election gRPC services only);
- `LeaseImpl.java:240-287` (keepalive scheduling, deadline, error handling).

**Upstream, read through the GitHub REST API on 2026-11-02:**
- issue and PR states and descriptions for #16822, #18069, #18428, #19091 (and its comments), #19092 (and its comments), #19597, #21589 (and its comments), #21869, #22082, #22083, #22084, #22417, #22650 and Kubernetes #111116;
- issue searches for the `leasing` double write and for `EvictRange`;
- `git merge-base --is-ancestor 4e986363a371 v3.7.3` succeeded.

**Prior analyses consulted:**
- `analysis/leases-claude.md` (§1–§7, §11) and `analysis/watchers-claude.md` (§1–§12, §15), both as of 2026-11-01;
- `analysis/INDEX.md`, for the one-line summary of `leases-agy.md`.

## Appendix B. Cross-check against the Codex and Antigravity analyses (2026-11-02)

Both files answer the same question independently: `chubby-perspectives-codex.md` (Codex) and `chubby-perspectives-agy.md` (Antigravity). They were read as leads, not as sources. Nothing below depends on them: every item taken was re-verified, and the evidence cited in the body is the paper, the code, upstream, or the functional test.

**Taken, after verification:**

| Item | Lead from | Where now | What verification found |
|---|---|---|---|
| Child-node events let a client monitor an ephemeral file without holding it open | Codex | §2 (correction 1) | Chubby §2.5, `:166`, verbatim |
| The acknowledgement is the library's; events follow writes and gate nothing; application copies are outside the barrier | Codex | §3.1, §4 (N6) | Chubby §2.5, `:172`; Chubby §4.6, `:546` |
| Chubby's §4.4 stale-read flaw as evidence about the invariant | Codex | §3.3, §4 (I5), §11 | Chubby §4.4, `:478-490` |
| Three separate requirements (current-state cache, every mutation, deadline expiry) | Codex | §3.5 | A framing; the assignments are mine |
| The watch-freshness failures are within etcd's documented contract, so Chubby's design would strengthen a contract rather than fix a bug | Codex | §6.1 | The v3.7 API guarantees page, as quoted in `watchers-claude.md` §2.2 |
| #15247 is fixed and must not be counted as a live defect | Codex | Header, Summary, L1 | #16822 merged; `4e986363a371` is in `v3.7.3` |
| The paper does not show how a deposed master's internal work is fenced | Codex | §5.1, §11 | No mention of ballots in the paper (`grep`) |
| The message-rate model *S/T* vs *3L/T*, and its limits | Codex | §5.4 | Go client code at `lease.go:542`, `:586-619` |
| Longer leases under load also lengthen a dead holder's rights | Codex | §5.4 | Follows from Chubby §2.8, `:259`, and Chubby §3, `:352` |
| jetcd's deadline is receipt time + TTL | Codex, Antigravity | L6, B1 | `LeaseImpl.java:263-265` |
| A writer's wait is the maximum over cachers, and has no bound under repeated failures | Codex | §9 | Follows from Chubby §2.7 |
| Renewal safety cost: #18069 and #18428 | Antigravity | L11, §5.4 | Issue text: 20 % CPU at 1500 renewals/s, v3.5.11, arm64. The Antigravity file's "10× network amplification" is supported by the issue's comments (2024-05-27: peer bytes "an order of magnitude higher"; 2024-08-09: another user saw leader peer traffic go from about 2 KB/s to 100 KB/s). I first missed this by reading only the body |
| Held replies would make the Go client resend every 500 ms | Antigravity (as an open question) | §5.4 | `lease.go:542-556`, `:586-610`, `:86` |
| `ErrNoLeader` is terminal; Kubernetes #111116 | Antigravity | W12, §6.2, B2 | Code `watch.go:665-668`, `interceptor.go:239-241`; issue text |
| `leasing` costs: exclusive owner, Raft write per acquire, uncached ranges | Antigravity | §8.1, §8.3 | Code; `txn.go:352-365` |
| `EvictRange` bug | Antigravity | §8.3 | Real (`cache.go:183-184`), but not new: PRs #21869 and #21589 proposed the fix and were closed unmerged |
| Open `leasing` issues #22083, #22084, #22417 | Antigravity | §8.3 | Open on 2026-11-02 |
| `v3lock` orphans the session; `Unlock` is a plain delete; #19597; #22082 | Antigravity | §7 | Code and issue text. Chubby's instance and handle sequence numbers (Chubby §2.3, `:124`, `:134`; Chubby §2.9, `:312`) were added from the paper |
| etcd's protocol is thin, but its correct use is not | Antigravity | §9 (thick client) | Revised my earlier version, which overstated the advantage |
| Term-tagged revocations as the etcd form of epochs | Antigravity | B6 | Already in `leases-claude.md` §3.4 |

**Found while verifying, in neither file:**
- The `leasing` stale read (§8.2 b) is upstream #19091, with a failing test; two fixes were never reviewed.
- Every non-owner `leasing` write is applied twice, with wrong `Delete` and `WithPrevKV` responses. A contested write costs five proposals, not the three the Antigravity file counted (§8.3, reproduced).
- Chubby's applications reproduced the #111116 failure by crashing on the fail-over event (W12).

**Rejected or corrected:**

| Claim | From | Why not taken |
|---|---|---|
| "A deposed Chubby master can never forward a stale expiration decision"; "Paxos writes are fenced by the master's consensus ballot" | Antigravity | Not in the paper. It is a plausible inference, stated as such in §5.1 |
| Chubby "eliminates all nine `WithRequireLeader` failure modes" | Antigravity | Most are defects in an etcd feature Chubby lacks, so they are avoided by omission, not prevented. Chubby's own fail-over event produced the analogous availability failure (W12) |
| #111116 shows "zero grace period during routine 1 s leader elections" | Antigravity | The issue is about *prolonged* re-election (three election cycles) and the API server's reaction. The immediate rejection of new flagged streams is real (`interceptor.go:239-241`), but is not what #111116 reports |
| "A Chubby client can neither read stale cached data during a partition nor lose its session during a normal election" | Antigravity | Chubby §4.4 records a design flaw that allowed stale reads ("rare in practice", no incident reported); a partition longer than the grace period loses the session |
| A contested `leasing` write is "three sequential Raft transactions" | Antigravity | Measured five proposals; it also missed the double application (§8.3) |
| Server pacing alone "allowed a single master to sustain 90,000 sessions" | Antigravity | The 90,000 figure (`:468`) is given in support of explicit-invalidation caching under a constant KeepAlive rate, compared with DNS TTLs, not of lease lengthening. The sub-millisecond latency (`:438`) is a general statement, not tied to that figure. Chubby §3 lists several scaling mechanisms |
| "Where Chubby is strictly superior: … epoch/term fencing" | Antigravity | etcd has Raft terms. The gap is specific: expiry revocations carry none (L1, B6) |
| A single node crash would stall Kubernetes writes | Antigravity | The etcd-level cachers in Kubernetes are API servers, not nodes (§9) |
