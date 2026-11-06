# Critique of `chubby-perspectives-agy.md` (Claude)

**Date:** 2026-11-02.

**Purpose.** A review of the Antigravity analysis `analysis/chubby-perspectives-agy.md`, written so that the Antigravity agent can correct its own file. Each item gives:
- the location and the quoted claim;
- what is wrong, with evidence that can be re-checked from this repository, the etcd/jetcd checkouts, the GitHub API, or the functional test in Appendix A;
- a suggested correction;
- a priority and a confidence.

The reviewer also wrote `chubby-perspectives-claude.md`. Where an item relies on it, the underlying evidence is repeated here so that this file stands alone.

**Target version.** `analysis/chubby-perspectives-agy.md` as modified 2026-11-02 09:44:30 -0500: 508 lines, 75,958 bytes, SHA-256 prefix `90650a470cdc9831`. Line numbers below refer to that version. Every item also quotes the text, so it can be found if lines move. An earlier version of the same file (about 530 lines, read the morning of 2026-11-02) contained further claims that the revision already removed; they are not discussed here.

**Versions analyzed:**

| Component | Version | Notes |
|---|---|---|
| etcd | `v3.7.3` = `0a173d0410a9` | All `path:line` citations are at this tag unless marked. `origin/main` at `a2e3642bc` (2026-11-02) consulted only for `client/v3/leasing`. |
| jetcd | `jetcd-0.8.7` = `23b61f1499a5` | `LeaseImpl.java`, `WatchImpl.java`. |
| Chubby | Burrows, OSDI 2006, `doc/chubby-osdi06.md` | Cited by section and Markdown line (`:N`). |
| Functional test | `gcr.io/etcd-development/etcd:v3.7.3`, one member; Go client `v3.7.3` built with go1.26.10 in `golang:1.26` | Appendix A. Counts only (proposals, revisions, versions, responses); no timings; not a benchmark (AGENTS.md §7). |

**Method.** Re-read of the target file in full. Re-reading of every cited paper line and code path behind the items below. GitHub REST API reads of the issue and PR states and texts cited, on 2026-11-02. Three runs of the functional test in Appendix A, the last of them the consolidated program reproduced there.

**Upstream references** (states read 2026-11-02):
- etcd: [#12528](https://github.com/etcd-io/etcd/issues/12528), [#13776](https://github.com/etcd-io/etcd/issues/13776) (closed, not planned), [#15247](https://github.com/etcd-io/etcd/issues/15247), [#16822](https://github.com/etcd-io/etcd/pull/16822) (merged 2023-12-15), [#17611](https://github.com/etcd-io/etcd/issues/17611) (closed, not planned), [#18069](https://github.com/etcd-io/etcd/issues/18069) (closed), [#18428](https://github.com/etcd-io/etcd/pull/18428) (merged), [#18879](https://github.com/etcd-io/etcd/issues/18879) (open), [#19091](https://github.com/etcd-io/etcd/issues/19091) (closed by stale bot), [#19092](https://github.com/etcd-io/etcd/pull/19092), [#21589](https://github.com/etcd-io/etcd/pull/21589), [#21869](https://github.com/etcd-io/etcd/pull/21869) (all three closed unmerged), [#20221](https://github.com/etcd-io/etcd/issues/20221) (closed, completed), [#22081](https://github.com/etcd-io/etcd/issues/22081), [#22082](https://github.com/etcd-io/etcd/issues/22082), [#22094](https://github.com/etcd-io/etcd/issues/22094), [#22649](https://github.com/etcd-io/etcd/issues/22649) (open), [#22650](https://github.com/etcd-io/etcd/pull/22650) (open).
- Kubernetes: [#59848](https://github.com/kubernetes/kubernetes/issues/59848) (open), [#96836](https://github.com/kubernetes/kubernetes/issues/96836) (closed), [#111116](https://github.com/kubernetes/kubernetes/issues/111116) (closed as completed 2022-07-28).

## Summary

**Overall.** The revised file is much improved over its first version. Its central thesis matches the other two analyses: conditional renewal ("ack-or-expired"), the decomposition into F1–F5, the six premise corrections, Chubby's §4.4 flaw, and the costs of the write barrier. Most of its code facts check out (§4 below lists them). The problems are concentrated in four places:
1. **Upstream issues cited for something they do not show.** #17611, #111116, #18069, #13776, and the descriptions of #22081/#22094 and #22649.
2. **The `client/v3/leasing` cost analysis.** It undercounts the Raft cost and misses that every non-owner write is applied twice, with wrong responses. Both were reproduced (Appendix A).
3. **Two of the six proposed borrowings would not work as written.** B6's `s.Done()` check conflicts with `v3lock`. B5's apply-time term check, read literally, is non-deterministic. A third, B3, claims linearizable reads that only the write barrier the file excludes can provide.
4. **"Prevented" verdicts at High confidence for things Chubby avoids by omission, or that the paper does not document.**

**Counts.** 15 items to fix (P1), 12 to qualify (P2), 9 precision fixes (P3), and 6 additions. Confidence is per item.

**The items that change conclusions:**
- **E1:** `leasing` applies non-owner writes twice; a contested write is 5 proposals, not 3; each non-owner read of an owned key is a Raft write. — **High** (reproduced).
- **E10:** session-bound freshness heartbeats (B3/W4) bound staleness; they do not make cached reads linearizable without Chubby's write barrier ("Layer C"). The claim that B3 removes KEP-2340's per-read quorum read is wrong. — **High**.
- **E2, E3:** #17611 is a client teardown deadlock, not partition silence. #111116 is about *prolonged* leaderlessness (three election cycles), not routine 1 s elections. — **High** (issue texts).
- **E8, E9:** the B6 and B5 mechanisms as written. — **High** / **Moderate**.
- **E15:** Kubernetes does not use etcd leases for node heartbeats or leader election. — **High**.

---

## 1. P1: factual errors, misattributions, and proposals that would not work as written

### E1. `client/v3/leasing`: costs undercounted; double application and wrong responses missing

**Where.**
- Line 103: "every read miss costs a Raft `Txn` write and allows only **one client** to cache a hot key".
- Line 302 (W5): "$3\times$ Raft write amplification".
- Line 355: "all other readers fail the `Txn` and hit the server on every read".
- Line 356: "**3 sequential Raft `Txn` writes + 2 `Watch` deliveries per contested write**".
- Line 475 (matrix, item b): "contested writes cost 3 Raft `Txn`s + 2 `Watch` deliveries".

**What the code does.** — **High** (code).
- `put` and `delete` first call `tryModifyOp`, a `Txn` guarded by "no ownership key, or my own" (`client/v3/leasing/kv.go:218-234`).
- If the writer does not own the key locally (`wc == nil`), they then call `revoke` **unconditionally**, even when `tryModifyOp` already succeeded (`kv.go:241-244` for `put`, `:389-392` for `delete`).
- `revoke` is another `Txn` that applies the same operation again if no ownership key exists, or else writes `"REVOKE"` (`kv.go:413-421`).
- A `Txn` with any non-`Range` op in either branch is not read-only (`server/etcdserver/txn/txn.go:352-365`). So `tryModifyOp` and `acquire` go through Raft even when their compare fails.

**Measured** (Appendix A; v3.7.3, one member; counts from `etcd_server_proposals_committed_total`, the store revision, and a plain watcher):

| Case | Proposals | Revisions | `Version` | Plain watcher sees | Caller receives |
|---|---|---|---|---|---|
| Non-owner B puts `k1`; nobody owns `k1` | 2 | +2 | 0 → 2 | two `PUT k1=v1` | — |
| A owns `k2`; B puts `k2` | **5** | +4 | 1 → 3 | two `PUT k2=v2` | — |
| B deletes `k3` with `WithPrevKV`; nobody owns it | — | — | — | one `DELETE` | **`Deleted=0`, no `PrevKvs`**; the key is gone |
| B puts `k4="new"` with `WithPrevKV`; old value `"old"` | — | — | — | two `PUT k4=new` | **`PrevKv.Value="new"`** |
| A owns `k5`; B reads it 5× linearizable, then 5× serializable; A reads 5× | **5**, then 0, then 0 | 0 | — | — | values correct |

The five proposals of a contested write are:
1. `tryModifyOp`, which fails;
2. `revoke`, which puts `"REVOKE"`;
3. the owner's rescinding delete;
4. `tryModifyOp`, which succeeds;
5. the duplicate from the unconditional `revoke`.

**Also relevant.**
- **No upstream test catches it.** The integration tests that assert `Version` use the owner path (`tests/integration/clientv3/lease/leasing_test.go:321-363`, `:473-502`). `TestLeasingDeleteNonOwner` discards the `Delete` response (`:416-443`).
- **Unchanged upstream.** The code is the same on `origin/main` at `a2e3642bc`.
- **No report found.** GitHub issue searches for a leasing double write found nothing (**Moderate**: a search can miss a report).

**Suggested correction.**
- Replace "3 sequential Raft `Txn` writes" with "five Raft proposals (four revisions), and the write is applied twice".
- Add that every write by a non-owner is applied twice: two revisions, two watch events, `Version` +2, a wrong `Delete` count, and a wrong `PrevKv`.
- Replace "hit the server on every read" with "every linearizable read by a non-owner of an owned key is a Raft proposal, more than a plain etcd linearizable read, which needs a ReadIndex but appends nothing to the log; the only exception is the 2 s after a revoke (`MayAcquire`, `cache.go:113-118`, `revokeBackoff` at `cache.go:28`)".
- Drop "$3\times$" everywhere it appears.

**Priority:** P1. **Confidence:** High.

### E2. #17611 is cited as evidence of silent partition stalls; it is a client teardown deadlock

**Where.**
- Line 298 (W1): "unfenced `ProgressNotify` at frozen local revision, `#17611`, `k8s#59848`".
- Line 459 (B3): "Eliminates silent watch stalls on partitioned followers ... ([#17611] ...)".
- Line 502 (Conclusions): "silent watch stalls on partitioned followers (`#17611`, `k8s#59848`)".

**Evidence.**
- #17611 is titled "Watch API. blocked forever even with WithRequireLeader option" and was closed as not planned.
- Its body: a response with both `Created` and `Canceled` set, followed by a halt error or `ErrNoLeader`, leaves `clientv3/watch.go` `run()` "blocked at w.wg.Wait() in defer" or with `ws.outc` never closed. That is a client teardown deadlock.
- The file itself uses #17611 correctly in W3 (line 300, "`#17611` deadlock").
- k8s #59848 ("Kubernetes is vulnerable to stale reads, violating critical pod safety guarantees") concerns `resourceVersion=0` reads served from an API server's watch cache that "can be arbitrarily delayed". It is the *consequence* of any watch-cache lag, not specifically of a partitioned etcd member.

**Suggested correction.**
- Remove #17611 from W1, B3 and the Conclusions. Keep it only as the deadlock in W3.
- For partition silence, cite the code: without the flag, a stream is never told about leader loss (`v3rpc/interceptor.go:239-258`, `:315-354`).
- Keep #59848 as the downstream consequence, worded as such.

**Priority:** P1. **Confidence:** High.

### E3. #111116 is attributed to "zero grace period" during "1 s elections"

**Where.**
- Line 300 (W3): "zero election grace period `k8s#111116`" and "rejecting new streams during a 1s election (`interceptor.go:240`)".
- Line 502: "the nine failure modes of `WithRequireLeader` (`k8s#111116`)".

**Evidence.**
- #111116, "Prolonged etcd reelection causes api-server to close all watchers": "When etcd is unable to elect leader for 3 election cycles, etcd returns 'no leader' error through watches", and the API server then "terminates all watchers", whose mass re-creation overwhelms it. It asked for a retry window before termination.
- In the code, these are two different behaviours:
  - **New** flagged streams are refused at once while the member has no leader (`v3rpc/interceptor.go:239-241`). That is the "zero grace period", and it is not what #111116 reports.
  - **Existing** flagged streams are cancelled only after `maxNoLeaderCnt = 3` consecutive election timeouts without a leader (`interceptor.go:38`, `:315-354`). That *is* #111116's trigger.
- "1 s election": 1 s is the default election *timeout*, not a measured election duration.

**Suggested correction.** Describe #111116 as the API server's reaction to prolonged leaderlessness: after three election timeouts, every flagged watch is cancelled and its watchers are terminated en masse. Keep the immediate rejection of new streams as a separate, code-cited point.

**Priority:** P1. **Confidence:** High.

### E4. #12528 and #15247 are presented as live defects that B5 would eliminate; both are fixed in v3.7.3

**Where.**
- Line 188 (L6): "Deposed-leader premature revocations (`#12528`, `#15247`) & residual windows `R1` and `R2`".
- Line 282: "does **not** by itself prevent a deposed leader from proposing premature lease revocations (`#12528`, `#15247`)".
- Line 461 (B5): "Eliminates deposed-leader spurious revocations ([#12528], [#15247]) and residual windows `R1` ... and `R2`".

**Evidence.**
- PR #16822 ("Ignore old leader's leases revoking request") fixes #15247; it was merged 2023-12-15.
- Its merge commit `4e986363a371` is an ancestor of `v3.7.3` (`git merge-base --is-ancestor 4e986363a371 v3.7.3` succeeds).
- The fix is a leadership pre-check (`ensureLeadership`) before expiry revocation and before renewal on the leader.
- What remains are residual windows around that check, inferred from code and not reproduced (`leases-claude.md` §3.4, R1–R5).

**Suggested correction.**
- Mark #12528 and #15247 as historical: fixed by a pre-check in v3.7.3.
- State B5's value as replacing a check-before-acting with a fence, which closes the residual windows. A fence on revocations closes R5 and the stale-batch cases of R1/R2. It does not close R3 (an ex-leader acknowledging forwarded renewals) or R4 (no leader change).

**Priority:** P1. **Confidence:** High.

### E5. The #22649 mechanism is attributed to the wrong function

**Where.** Line 256: "Because `scheduleCheckpointIfNeeded` ignores any error from that `raftRequest`, a transient timeout leaves the stale `RemainingTTL` checkpoint in place...".

**Evidence.**
- `scheduleCheckpointIfNeeded` only pushes the lease onto the checkpoint heap when its remaining TTL exceeds the checkpoint interval (`server/lease/lessor.go:753-770`).
- The error is discarded in the checkpointer closure installed by `SetCheckpointer`. It calls `srv.raftRequest(ctx, &pb.InternalRaftRequest{LeaseCheckpoint: cp})` and returns `nil` regardless (`server/etcdserver/server.go:392-403`).
- PR #22650 ("server: propagate raftRequest error from lease Checkpointer", open) describes exactly that.

**Suggested correction.** Attribute the swallowed error to the `SetCheckpointer` closure (`server.go:392-403`). For the consequence, cite #22650's own description.

**Priority:** P1. **Confidence:** High.

### E6. #22081 and #22094 are misdescribed

**Where.** Line 269: "When one subscriber cancels its context (`#22094`), `keepAliveCtxCloser` removes that subscriber, but if `WithRequireLeader` fails on the shared stream (`#22081`), `closeRequireLeader` only tears down `ka` entries whose *first* context in `ka.ctxs` carried the metadata—leaving mixed subscribers in a broken state."

**Evidence** (issue titles and code at `client/v3/lease.go:376-411`):
- **#22081**, "KeepAlive subscriber may get the wrong channel closed after leader loss". When compacting the surviving subscribers, the code pairs each surviving channel with `ka.ctxs[newIdx]`, indexing the old slice with the *new* index (`lease.go:406`). Channels and contexts become misaligned, so a later context cancellation closes the wrong subscriber's channel. Nothing in the code tears down entries based on the *first* context.
- **#22094**, "closeRequireLeader leaves an orphaned keepAlives entry that keeps sending keepalive requests forever". When *every* subscriber of a lease is flagged, `closeRequireLeader` empties `chs`/`ctxs` but never deletes the entry from `l.keepAlives`. `recvKeepAlive` advances `nextKeepAlive` only inside the loop over channels (`lease.go:544-556`), so the request goes out on every 500 ms pass, and the lease never expires. It is not triggered by "one subscriber cancels its context". That path, `keepAliveCtxCloser`, deletes the entry correctly when no listeners remain (`lease.go:369-371`).

**Suggested correction.** Replace the sentence with the two descriptions above. Line 186 (L4) can stay as it is.

**Priority:** P1. **Confidence:** High.

### E7. #13776 is cited as the checkpoint feature or its "complexity"; it is a stabilization-tracking issue

**Where.**
- Line 185 (L3): "`LeaseCheckpoint` complexity (`#13776`)".
- Line 256: "etcd introduced Raft-replicated `LeaseCheckpoint` proposals ([#13776])".

**Evidence.** #13776 is titled "Track stabilization of lease checkpointing" ("tracks graduation of this feature"). It was closed as *not planned*. It neither introduced checkpointing nor reports a defect.

**Suggested correction.** Cite #13776 for what it shows: graduation of the feature was tracked and abandoned, consistent with the Alpha gate (`server/features/etcd_features.go:94-95`). Do not cite it for the introduction of checkpointing.

**Priority:** P1 (wrong attribution); low impact on conclusions. **Confidence:** High.

### E8. B6's "select on `s.Done()` inside `waitDeletes`" would break the server-side lock

**Where.**
- Line 462 (B6): "select on `s.Done()` inside `waitDeletes`".
- Line 394 notes the reason itself: "`v3lock/lock.go:44` already closed `s.Done()` via `s.Orphan()`".

**Evidence.**
- `v3lock.Lock` builds a session on the caller's lease and calls `s.Orphan()` *before* `m.Lock(ctx)` (`server/etcdserver/api/v3lock/lock.go:35-49`).
- `Orphan()` is `s.cancel(); <-s.donec` (`client/v3/concurrency/session.go:102-105`). So `Done()` is already closed when the wait starts.
- A `waitDeletes` that aborts on `s.Done()` would therefore make every contended `v3lock.Lock` return at once, for jetcd and every other thin client.

**Suggested correction.** Make the early-termination signal independent of the session's local keepalive goroutine. Two options:
- also watch the waiter's *own* key and stop on its deletion, as the TODO at `mutex.go:88` says ("early termination if the session key is deleted");
- or change `v3lock` so it does not orphan the session before waiting.

Keep the fenced `Unlock` part of B6. It is correct, and its note that `UnlockRequest` carries only `key` (`v3lockpb/v3lock.proto`) is right.

**Priority:** P1. **Confidence:** High.

### E9. B5's apply-time check, as written, is not deterministic

**Where.** Line 461 (B5): "reject them at apply time if `proposal.LeaderTerm != currentTerm`".

**Problem.**
- The apply step must give the same result on every member, including a member replaying its WAL after a restart or catching up from far behind.
- If "currentTerm" means the applying member's live Raft term, members applying the same entry at different times can see different terms and diverge.
- The deterministic quantity is the term of the **log entry** being applied. It is in the entry and is available in the apply loop (`e.GetTerm()`, `server/etcdserver/server.go:1892-1920`).
- `s.getTerm()` is only the term of the *previously applied* entry (set after each entry at `server.go:1920`), so it is not the right comparison either.

**Suggested correction.**
- Record the decision term in the request, and reject at apply if it is older than the term of the log entry that carries it (`req.term < entry.Term`). `!=` is equivalent, since the decision term can never exceed the entry term.
- Plumb the entry term into the lease applier, which today receives only the request.
- Note the cost: a proto and WAL change (`LeaseRevokeRequest` has only `ID` today, `api/etcdserverpb/rpc.proto`). It also meets the objection raised in the #15247 thread, "Using term information inside the state machine is unusual" (quoted in `leases-claude.md` §3.4).

**Priority:** P1 (design correctness). **Confidence:** Moderate. The determinism requirement is basic; the exact plumbing was not prototyped.

### E10. Freshness heartbeats are presented as giving linearizable cached reads without Chubby's write barrier

**Where.**
- Line 301 (W4): "A Chubby `KeepAlive` reply with no invalidations is a **master-asserted linearizable proof** that every cached entry is current ... No separate `ReadIndex` + `RequestProgress` polling loop is needed."
- Line 459 (B3): "removes the need for `KEP-2340`'s per-read `KV.Get` quorum `ReadIndex` (`W4`)".
- Line 453: "Layers A and B, however, are completely independent of Layer C".

**Problem.**
- A Chubby client's cached read is consistent because **writes wait** for every cacher's acknowledgement or lease expiry (§2.7, `:229-239`). A KeepAlive reply is a statement about one instant. Between replies, the cache stays valid only because no write can complete without invalidating it first.
- That write barrier is the file's own "Layer C", which it excludes for etcd (lines 451-453).
- Without Layer C, a leader-asserted revision on each keepalive (B3) tells the client how stale it *might* be: at most one heartbeat interval plus delivery lag. A write can still commit between the last heartbeat and the read.
- KEP-2340's purpose is *consistent* (linearizable) reads from the watch cache. These need the current revision at read time, which is why it does a quorum read per request.
- So B3 can bound staleness, detect a stuck watch, and replace KEP-2340's polling for callers who accept bounded staleness. It cannot remove the per-read quorum step for linearizable reads.

**Suggested correction.**
- Rephrase W4: the KeepAlive reply is a *liveness and delivery* signal, and consistency comes from the write barrier.
- In B3, replace "removes the need for KEP-2340's per-read quorum ReadIndex" with "bounds watch-cache staleness to the heartbeat interval and detects stalled watches; linearizable reads still need a current revision per read unless Layer C is adopted".
- Line 453 should say that Layers A and B fix detection and liveness, not linearizability.

**Priority:** P1 (changes a conclusion). **Confidence:** High.

### E11. First-keepalive timeouts: the in-process client's value is wrong, and the Go client's depends on configuration

**Where.**
- Line 98: "`6s` in `client/v3`, `5s` in `jetcd`, `1s` in `v3client`".
- Line 205.
- Line 241: "internal loopback `firstKeepAliveTimeout = time.Second`: `v3client.go:37`".
- Line 474.

**Evidence.**
- `v3client.New` passes `time.Second` (`server/etcdserver/api/v3client/v3client.go:37`, the line the file cites). But `NewLeaseFromLeaseClient` replaces exactly `time.Second` with `defaultTTL` = 5 s (`client/v3/lease.go:203-205`, `:81`). The in-process client's effective value is therefore **5 s**.
- The Go client passes `DialTimeout + 1s` (`lease.go:193`). With `DialTimeout` unset this is 1 s, and so also becomes 5 s. It is 6 s only when `DialTimeout` is 5 s.
- jetcd's default of 5000 ms is correct (`LeaseImpl.java:61`, `DEFAULT_FIRST_KEEPALIVE_TIMEOUT_MS`).

The conclusion (a reply held for 12–60 s would make the clients drop the lease) is unaffected.

**Suggested correction.** "5 s by default in all three; the Go client uses `DialTimeout + 1s` when that is not exactly 1 s".

**Priority:** P1 (wrong number), low impact. **Confidence:** High.

### E12. jetcd does not implement lock recipes itself

**Where.** Line 442: "Etcd's choice to implement complex session, watch, and lock recipes independently in Go (`client/v3`) and Java (`jetcd`) without a shared sidecar directly produced the client bug divergence".

**Evidence.** jetcd's `LockImpl` and `ElectionImpl` call the server's `Lock` and `Election` gRPC services. The file says so at line 383, and `jetcd-core/.../impl/LockImpl.java` and `ElectionImpl.java` confirm it. In the paper's terms, those services play the protocol-conversion role (§4.2, `:458`). Lease keepalive and watch resume *are* re-implemented in jetcd, with their own defects.

**Suggested correction.** "re-implement lease keepalive and watch resume in each language; locks and elections are delegated to the server's `v3lock`/`v3election` services". Drop "directly produced", or support it.

**Priority:** P1 (contradicts the file's own §4.1). **Confidence:** High.

### E13. Chubby §4.4 is described as an observed incident ("post-mortem", "served stale data"); the paper describes a hazard

**Where.**
- Line 96: "those sessions served stale cached data across fast failovers".
- Line 425: "Chubby's own production post-mortem in §4.4 proves it does not".
- Line 473: "caused a stale-read cache bug".

**Evidence.** §4.4 (`:488`) says the discarded sessions "could then read stale data for a while", calls the optimization "unsafe", and adds "This is rare in practice; in a large system it is almost certain that some session will fail to check in". No incident is reported.

**Suggested correction.** "a design flaw that permitted stale reads (described as rare in practice)". Keep the rest of §5.2. Its point stands: holding a KeepAlive did not make the bookkeeping correct, and the fix cost availability.

**Priority:** P1 (misstates the source). **Confidence:** High.

### E14. W2's reasoning conflates transport keepalives with apply stalls

**Where.** Line 299 (W2): "Silent watch stall during apply-loop stall or process pause (`DialKeepAliveTime == 0`)", with the explanation "if the master stalls or pauses, it cannot answer `KeepAlive` and clients enter `Jeopardy` within `12s`".

**Problems.**
- **`DialKeepAliveTime == 0`** (gRPC keepalive pings off by default, `client/v3/client.go:231-236`) concerns detecting a dead connection or a paused process. It is unrelated to an apply-loop stall on a member that still answers.
- **The Chubby reason is wrong.** The relevant property is topology. Only the master serves reads, and its state includes its own commits, so a Chubby master with a slow apply stalls *writes*, not freshness. It may keep answering KeepAlives normally. A whole-process pause is the only case where "cannot answer KeepAlive" applies.
- **"Within 12s"** should be "within the client's conservative local lease", which can be up to about 60 s under load (`:352`).

**Suggested correction.** Split W2 into two rows:
- an apply stall, prevented by topology;
- a process pause or dead connection, prevented because the KeepAlive is the heartbeat.

Fix the bound.

**Priority:** P1 (wrong mechanism). **Confidence:** High.

### E15. Kubernetes does not use etcd leases for node heartbeats or leader election

**Where.** Line 254: "Kubernetes attaches 1-hour (`3600s`) unrenewed leases to `Event` objects (`P2` bounded expiration) alongside 10–15s actively renewed leases for node heartbeats and leader election (`P3` non-revocation)."

**Evidence.**
- Node heartbeats and controller leader election in Kubernetes use `coordination.k8s.io` `Lease` **API objects**. These are stored as ordinary etcd keys and renewed by updating the object, not by etcd `LeaseKeepAlive`.
- The API server never calls etcd `KeepAlive`. Its etcd leases are TTL leases granted and reused by `lease_manager.go` for objects with a TTL, mainly Events (`leases-claude.md` §7, code-verified there; Kubernetes source not re-read for this critique).
- The P2/P3 conflict therefore arises *across* etcd users (sessions and locks in other systems versus Kubernetes TTL objects), not inside Kubernetes.

**Suggested correction.** "Kubernetes uses etcd leases only as TTL timers (mainly Events, 1 h by default); its node heartbeats and leader election are Kubernetes `Lease` objects renewed by writes. Active, P3-dependent etcd leases come from other etcd users: `concurrency` sessions, locks, elections."

**Priority:** P1. **Confidence:** High (recalled Kubernetes architecture, consistent with the code-verified statement in `leases-claude.md` §7).

---

## 2. P2: overstatements to qualify

### O1. "All 9 `WithRequireLeader` failure modes ... prevented by the 45 s Jeopardy grace period"

**Where.** Line 300 (W3, High), line 502.

**Problems.**
- **Most are implementation defects in etcd client or proxy code**: the 250 ms final-error drop, the #17611 teardown deadlock, the #12254 stream split, the grpcproxy metadata bug. A grace period has nothing to do with them. Chubby avoids them by not having the feature; any Chubby-style etcd client would have its own implementation to get right. The paper calls Chubby's own library "delicate" (§4.2, `:458`).
- **One item is a design decision, not a defect.** Ending the watch on `ErrNoLeader` instead of failing over has been the client's deliberate behaviour since v3.1 (`watchers-claude.md` §6).
- **"Resumes seamlessly with a `safe` event"** holds only if the same master answers within the grace period. Across a master fail-over the client receives a fail-over event, its cache is flushed, its handles are recreated, and the application is told to rescan (§2.9, `:309-312`).
- **Chubby's applications reproduced the #111116 failure one layer up.** "Many developers chose to crash their applications on receiving this event, thus decreasing the availability of their systems substantially" (§4.6, `:534`). The file quotes this at line 440 but does not connect it to W3.

**Suggested correction.**
- Split W3 into "avoided by omission" (the implementation defects) and "prevented by design" (two-stage failure handling: jeopardy, then expiry after a grace period).
- Add the `:534` analogue.
- Lower the confidence to Moderate.

**Confidence in the objection:** High.

### O2. "Silent watch staleness is impossible by construction" at High confidence

**Where.** Line 293; W1–W4 rated High.

**Problem.** The file's own §5.2 shows Chubby's bookkeeping failing once (§4.4), and the guarantee also rests on the clock-rate assumption (Chubby §2.8, `:263`).

**Suggested correction.** "excluded by design, given correct session bookkeeping and the clock-rate assumption". Use Moderate for the transfer to etcd.

**Confidence:** High.

### O3. #18069 is presented as "client-paced renewal load" and "renewal floods aggravated by 500 ms retry storms"

**Where.** Line 183 (L1), line 502.

**Evidence.**
- #18069 reports leader CPU **+20 %** at **1500 renewals/s** on v3.5.11 (arm64). The reporter attributes it to the per-renewal `ensureLeadership()` (a ReadIndex) added by #16822. The comments add peer traffic "an order of magnitude higher" (2024-05-27), and another user saw it go from about 2 KB/s to 100 KB/s (2024-08-09).
- It was resolved by #18428, which skips the check while the local Raft tick is fresh. That opens residual window R1 (`leases-claude.md` §3.4).
- No retry storm is involved.
- The Chubby mechanism that addresses this cost is the **master lease**, not pacing: "Read requests are satisfied by the master alone; this is safe provided the master lease has not expired, as no other master can possibly exist" (§2.2, `:94`). That a Chubby master can answer KeepAlives without a quorum round by the same argument is an inference.

**Suggested correction.** Cite #18069 for the cost of making each renewal safe without a leader lease, and #18428 for the weaker check that followed. Attribute the Chubby counterpart to the master lease. Pacing (F5) addresses the *number* of renewals, a separate point.

**Confidence:** High (issue text); Moderate (the KeepAlive extension of `:94`).

### O4. "500 ms retry storm", "positive feedback", "unthrottled"

**Where.** Lines 98, 150 ("**unthrottled `500ms` retry amplification**"), 201-203, 457, 495.

**The mechanism is correct.** It was verified:
- Go `lease.go:542-556`, `:586-610`, `retryConnWait` = 500 ms at `:86`;
- jetcd's `setPeriodic(0, 500, ...)` sender with `nextKeepAlive` advanced only in `handleResponse` (`LeaseImpl.java:234-266`).

**What to change.** The rate is **bounded**: one request per lease per 500 ms (2/s) while a reply is overdue, against 3/TTL normally. That is about 40× for a 60 s TTL and about 3× for a 5 s TTL, and it does not grow with further delay. "Positive feedback" implies growth. "Unthrottled" contradicts the 500 ms tick.

**Suggested correction.** "While a reply is overdue, each lease is re-sent every 500 ms (2 requests/s per lease, e.g. 40× the normal rate at TTL 60 s); the clients do not back off." Drop "unthrottled" and "positive feedback".

**Confidence:** High.

### O5. L6 and §2.5 rate "Chubby fences ex-masters via master leases and epochs" at High

**Where.** Line 188 (High), lines 283-284.

**Problem.**
- The paper describes two mechanisms: master leases, under which replicas promise not to elect another master for an interval (Chubby §2.2, `:90`), and epochs, which make the master reject *client* calls carrying older epoch numbers (Chubby §2.9, `:305`).
- It does not describe how a deposed master's own queued internal work, such as a session-expiry decision that releases locks, is prevented from committing.
- "Paxos ballots would reject it" is a reasonable inference. But the paper never mentions ballots, and it calls fail-over code "a rich source of interesting bugs" (Chubby §2.9, `:315`).

**Suggested correction.** Keep the verdict "not prevented by KeepAlives". Lower the Chubby side to Moderate and say the internal-work fencing is inferred.

**Confidence:** High (the paper text).

### O6. "Chubby's server-side lock table eliminates all four failure classes"; "Handle- and instance-scoped `Release()`"

**Where.** Line 105; line 395 (table row 3).

**Evidence.**
- `:209` only names `Acquire()`, `TryAcquire()` and `Release()`.
- `:217` says a handle "is associated with an instance of a file". That concerns a node deleted and recreated under the same name. #22082's scenario is the same lock acquired again by the same session, where the instance does not change.
- The lock generation number exists (it is in the sequencer, §2.4), but the paper does not say whether `Release()` checks it.
- #20716 (closed, not planned) and #18879 (open; reported with the in-process `v3client`) are etcd watch-plumbing defects. That a Chubby-style lock table in etcd would avoid them is an inference about a design that does not exist.

**Suggested correction.**
- Chubby's server-side wait removes the watch dependency (classes 1 and 2), and session expiry ends pending calls (Chubby §2.8, `:281`).
- For stale releases (#22082), the paper shows the relevant identifiers (instance numbers, Chubby §2.3, `:124`; handle sequence numbers, Chubby §2.3, `:134`; closed handles remembered so that "a delayed or duplicated network packet cannot" recreate them, Chubby §2.9, `:312`) but does not document `Release` fencing.
- Lower to Moderate.

**Confidence:** High (the paper text).

### O7. Lock waiters "hang forever"

**Where.** Line 393: "`Lock()` hangs forever after the lock is released".

**Correction.** "until the member recovers, the stream breaks, or the caller's context ends". The file already qualifies this for L-A at line 342.

**Confidence:** High.

### O8. "Freeze all writes ... for 12–60 seconds" on a GC pause, network blip, or rolling restart

**Where.** Line 107; line 418.

**Problem.**
- Under Chubby's rule, a writer waits for a slow cacher's acknowledgement or for its *remaining* lease (Chubby §2.7, `:239`). A pause shorter than the lease delays writes by the length of the pause.
- Only a crashed or cut-off cacher costs the remaining lease.
- A clean restart ends the session explicitly (§2.8, `:255`: a client "ends the session explicitly ... when it terminates").

**Suggested correction.** "stall writes to everything it caches for as long as it is unresponsive, up to its remaining lease (12 s by default, up to ~60 s under load)".

**Confidence:** High.

### O9. "Before allowing writes to proceed"; "etcd resumes writes within ~1–2 seconds"

**Where.** Line 433.

**Correction.**
- The paper says "before allowing operations to proceed" (Chubby §4.4, `:490`), i.e. all operations, not only writes.
- The etcd figure is unmeasured here; tag it as an estimate from default timeouts.

**Confidence:** High (paper text).

### O10. B4 "eliminates" the 1,000-key revocation stalls; a sweeper is "asynchronous"

**Where.** Line 460.

**Problem.**
- Deleting expired keys is still a replicated write proposed by the leader. Each deletion produces a revision and watch events, and leader changes and fencing (E9) still apply to the sweeper.
- Whether deletions are batched in small transactions is an implementation choice the timestamp alone does not imply.
- A per-key expiry field is also a storage and API change (`mvccpb.KeyValue`), and it puts wall-clock decisions into replicated behaviour.

**Suggested correction.** "removes the P2/P3 conflict and the need for checkpoints; revocation cost depends on how the sweeper batches deletions". List the format change and the clock dependence under trade-offs.

**Confidence:** Moderate (design reasoning).

### O11. "Lossless resumption across elections"

**Where.** Line 303 (W6).

**Correction.** "within the retained history": a watch whose start revision has been compacted is cancelled with `CompactRevision` (`mvcc/watcher_group.go:255-262`, as cited in `watchers-claude.md` §1).

**Confidence:** High.

### O12. "Forcing the client library to process them before re-issuing its next KeepAlive"

**Where.** Line 117.

**Correction.**
- For invalidations, yes: the library must flush before acknowledging.
- Events are "delivered to the client asynchronously via an up-call" (`:163`). The paper does not say the application processes them before the next KeepAlive. §4.6 only says that "events must eventually be acknowledged" (`:546`).
- Say "flush invalidated entries; record events for delivery".

**Confidence:** High.

---

## 3. P3: precision and citations

| # | Where | Issue | Fix |
|---|---|---|---|
| C1 | Line 271 | jetcd deadline given as "`deadLine = nextKeepAlive + TTL * 1000 * 2 / 3`" | The code is `ka.setDeadLine(System.currentTimeMillis() + ttl * 1000)` (`LeaseImpl.java:265`). Same value; cite the actual expression. jetcd's deadline sweep runs every 1000 ms (`DeadLine.doStart`, `setPeriodic(0, 1000, ...)`) |
| C2 | Line 304 (W7) | Cites #18089 and #19179, absent from the header's references | Add them to the header with states, or drop them |
| C3 | Lines 54, 304 | #20221 ("Watch on future revision might receive old events or notifications ... multiple members") is closed as completed (2025-07-22) | Present it as fixed, and state whether the fix is in v3.7.3 |
| C4 | Line 393 | #18879 / #20716 presented as freezing "all in-process watches" without qualification | #18879 was reported with the in-process `v3client`. Reachability over the network is rated Moderate in `watchers-claude.md` §8.1. #20716 is closed as not planned; fix PR #21065 is open |
| C5 | Lines 103, 357, 493 (T3) | The `EvictRange` bug is presented as a finding | It is real (`cache.go:183-184`), but two upstream PRs already fixed it and were closed unmerged: #21869 (with unit tests calling `EvictRange` directly, a simpler T3) and #21589. Unchanged on `origin/main` (`a2e3642bc`) |
| C6 | Lines 103, 345-350, 489-491 (L-B, T2) | The stale-read window is presented as a new derivation | Upstream #19091 (2024-12-19) reported it with a failing integration test (`TestLeasingGetChecksForExpiration`, in the reporter's fork). The stale bot closed it 2025-05-17; fix PRs #19092 and #21589 were closed unmerged. Cite them; T2 can reuse that test |
| C7 | Line 189 (L7) | "Prevented by master-direct topology" | True of the 2006 deployment. Note that the paper's designed proxies "add an additional RPC" and would forward KeepAlives and reads (Chubby §3, §3.1, `:356-364`, "We do not yet use them in production"), and that §4.4 lets proxies hold sessions the master does not know (`:492`) |
| C8 | Line 190 (L8) | Cites #21735, absent from the header's references | Add it with its state |
| C9 | Line 417 | "zero-RPC consistent cached reads across 90,000 clients" | The 90,000 figure (`:468`) is one master's peak client count, quoted in support of invalidation-based caching under a constant KeepAlive rate, compared with DNS TTLs. Keep it as a scale figure; do not tie it to consistent reads or to the latency at `:438` |

---

## 4. Claims re-checked and found correct

Kept here so that corrections stay targeted. Each was verified for this critique unless marked.

**Paper:**
- the F1–F5 decomposition and the ack-or-expired reading (§2.7–§2.8, §4.6);
- the six premise corrections in §1.2;
- "discards its in-memory state" on loss of mastership (`:285`);
- the §4.6 quote at `:542`;
- reads ≈ 3.4 % of RPCs (`GetStat` 2 % + `Open` 1 % + `GetContentsAndStat` 0.4 %, §4.1 table);
- fail-over duration 14 s (`:396`);
- the costs of the write barrier and the uncachable window (`:241`);
- takeover after failure taking up to lease + lock-delay.

**etcd v3.7.3:**
- `round_robin` (`client/v3/internal/resolver/resolver.go:44`);
- `nextKeepAlive` advanced only on response, and resend every 500 ms (`lease.go:542-556`, `:586-610`, `:86`);
- receipt-time deadline and 1 s sweep (`lease.go:543`, `:562-583`);
- the `closeRequireLeader` index bug at `lease.go:406`;
- gRPC keepalive pings off by default (`client.go:231-236`);
- `LeaseRevokeRequest` carrying no term (`rpc.proto`);
- the ID-only `UnlockRequest` (`v3lockpb/v3lock.proto`);
- `v3lock`/`v3election` orphaning their sessions, and `Leader`/`Observe` using lease `-1` (`v3lock/lock.go:35-58`, `v3election/election.go:75-136`);
- `Mutex.Unlock` as a plain delete of `<pfx><leaseID>` (`mutex.go:115`, `:134-150`), i.e. #22082;
- the #19597 TODO (`mutex.go:88`);
- checkpoints scheduled only when the remaining TTL exceeds the interval (`lessor.go:758`; default interval 5 min, `:52`);
- the `lessor.go:290-291` TODO, present since the initial lessor commit `bf70b5127` (2015-11-08).

**`client/v3/leasing`:**
- exclusive ownership (`kv.go:270-280`);
- range reads never cached (`cache.go:189`);
- `revokeRange` revoking keys one at a time (`kv.go:423-451`);
- the L-A schedule, plausible from code (not reproduced; it requires two endpoints with the streams on different members);
- the L-B window (also upstream, C6).

**jetcd 0.8.7:**
- default first keepalive timeout 5000 ms (`LeaseImpl.java:61`);
- 500 ms sender (`:231-240`);
- `handleException` reporting `onError` to every observer and restarting after 500 ms without removing leases (`:273-287`);
- the cursor jump on `Created` (`WatchImpl.java:267`);
- progress notifications setting the resume revision to the header revision without +1 (`:293`, `:312`), which makes a replay at that revision possible on reconnect. The end-to-end effect was not reproduced.

**Upstream:** the #18069 numbers. The file's earlier "10× network" figure is supported by the issue's comments. This reviewer's first pass wrongly said it was not, and `chubby-perspectives-claude.md` has been corrected.

---

## 5. Additions worth making

| # | Addition | Evidence | Where it fits |
|---|---|---|---|
| A1 | Non-owner `leasing` writes are applied twice, with wrong `Delete` and `WithPrevKV` responses; non-owner reads of owned keys are Raft writes | E1, Appendix A | §3.2, L-C |
| A2 | The stale-read window is a known, unfixed upstream report | C6 | §3.2, L-B |
| A3 | The master lease is what makes each Chubby renewal cheap to make safe | O3; Chubby §2.2, `:94` | §2.1 (L1), §2.2 |
| A4 | Chubby's applications reproduced #111116's failure by crashing on the fail-over event | O1; Chubby §4.6, `:534` | §3.1 (W3) |
| A5 | etcd's watch contract disclaims delivery time: "there is no limit and events in unhealthy clusters might never arrive" (v3.7 API guarantees page, quoted in `watchers-claude.md` §2.2). So W1–W4 are a weak contract, not contract violations, and Chubby's design would strengthen the contract rather than fix a bug | `watchers-claude.md` §2.2 | §3.1 |
| A6 | Per-failover extension: Chubby adds at most one session lease period (Chubby §2.9, `:307`), while `Promote` adds each lease's full TTL; the size difference comes from one primitive carrying long data timers. The file says the first half (lines 99, 252); the second half explains why | `lessor.go:480-492`; `:307` | §2.3 |

---

## Appendix A. Reproduction of the `client/v3/leasing` measurements

Runs on any host with Docker; nothing is installed on the host. The program needs one etcd member on `127.0.0.1:2379` with `/metrics` on the same port, which is the default.

```sh
mkdir leasingrepro && cd leasingrepro      # put go.mod and main.go below here
docker run -d --rm --name etcd-leasingrepro gcr.io/etcd-development/etcd:v3.7.3 \
  /usr/local/bin/etcd --name s1 --data-dir /tmp/d \
  --listen-client-urls http://127.0.0.1:2379 --advertise-client-urls http://127.0.0.1:2379
sleep 2
docker run --rm --network container:etcd-leasingrepro -v "$PWD":/src -w /src \
  -e GOFLAGS=-mod=mod golang:1.26 sh -c 'go mod tidy >/dev/null 2>&1; go run .'
docker rm -f etcd-leasingrepro
```

`go.mod`:

```
module leasingrepro

go 1.26

require go.etcd.io/etcd/client/v3 v3.7.3
```

`main.go`:

```go
// Reproduces the client/v3/leasing write and read costs against one etcd v3.7.3 member
// listening on 127.0.0.1:2379. Prints Raft proposals committed, store revision and key
// Version around each operation, and the responses returned to the caller.
package main

import (
	"bufio"
	"context"
	"fmt"
	"net/http"
	"strings"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
	"go.etcd.io/etcd/client/v3/leasing"
)

var ctx = context.Background()

func proposals() string {
	r, err := http.Get("http://127.0.0.1:2379/metrics")
	if err != nil {
		return "err"
	}
	defer r.Body.Close()
	sc := bufio.NewScanner(r.Body)
	for sc.Scan() {
		if l := sc.Text(); strings.HasPrefix(l, "etcd_server_proposals_committed_total ") {
			return strings.Fields(l)[1]
		}
	}
	return "?"
}

func state(c *clientv3.Client, key string) string {
	time.Sleep(300 * time.Millisecond) // let the owner's rescind settle
	g, err := c.Get(ctx, key)
	if err != nil {
		return err.Error()
	}
	v := int64(0)
	if len(g.Kvs) > 0 {
		v = g.Kvs[0].Version
	}
	return fmt.Sprintf("proposals=%s rev=%d version=%d", proposals(), g.Header.Revision, v)
}

func must(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	mk := func() *clientv3.Client {
		c, err := clientv3.New(clientv3.Config{Endpoints: []string{"127.0.0.1:2379"}, DialTimeout: 5 * time.Second})
		must(err)
		return c
	}
	plain, a, b := mk(), mk(), mk()
	lkvA, closeA, err := leasing.NewKV(a, "pfx/")
	must(err)
	defer closeA()
	lkvB, closeB, err := leasing.NewKV(b, "pfx/")
	must(err)
	defer closeB()
	time.Sleep(time.Second)

	events := make(chan string, 100)
	go func() {
		for wr := range plain.Watch(ctx, "k", clientv3.WithPrefix()) {
			for _, ev := range wr.Events {
				events <- fmt.Sprintf("%s %s=%s mod=%d ver=%d", ev.Type, ev.Kv.Key, ev.Kv.Value, ev.Kv.ModRevision, ev.Kv.Version)
			}
		}
	}()
	time.Sleep(500 * time.Millisecond)

	fmt.Println("== 1. B (not owner) puts k1; nobody owns k1")
	fmt.Println("   before:", state(plain, "k1"))
	_, err = lkvB.Put(ctx, "k1", "v1")
	must(err)
	fmt.Println("   after: ", state(plain, "k1"))

	fmt.Println("== 2. A owns k2 (by reading it); B puts k2")
	_, err = plain.Put(ctx, "k2", "v0")
	must(err)
	_, err = lkvA.Get(ctx, "k2")
	must(err)
	fmt.Println("   before:", state(plain, "k2"))
	_, err = lkvB.Put(ctx, "k2", "v2")
	must(err)
	fmt.Println("   after: ", state(plain, "k2"))

	fmt.Println("== 3. B deletes k3 WithPrevKV; nobody owns k3")
	_, err = plain.Put(ctx, "k3", "x")
	must(err)
	dr, err := lkvB.Delete(ctx, "k3", clientv3.WithPrevKV())
	must(err)
	g, _ := plain.Get(ctx, "k3")
	fmt.Printf("   resp.Deleted=%d len(PrevKvs)=%d; key still present=%v\n", dr.Deleted, len(dr.PrevKvs), len(g.Kvs) > 0)

	fmt.Println("== 4. B puts k4=new WithPrevKV; value before the call is \"old\"")
	_, err = plain.Put(ctx, "k4", "old")
	must(err)
	pr, err := lkvB.Put(ctx, "k4", "new", clientv3.WithPrevKV())
	must(err)
	prev := "<nil>"
	if pr.PrevKv != nil {
		prev = string(pr.PrevKv.Value)
	}
	fmt.Println("   PrevKv.Value =", prev)

	fmt.Println("== 5. A owns k5; B reads k5 five times (linearizable), then five times serializable; then A five times")
	_, err = plain.Put(ctx, "k5", "v")
	must(err)
	_, err = lkvA.Get(ctx, "k5")
	must(err)
	time.Sleep(300 * time.Millisecond)
	p0 := proposals()
	for i := 0; i < 5; i++ {
		_, err = lkvB.Get(ctx, "k5")
		must(err)
	}
	p1 := proposals()
	for i := 0; i < 5; i++ {
		_, err = lkvB.Get(ctx, "k5", clientv3.WithSerializable())
		must(err)
	}
	p2 := proposals()
	for i := 0; i < 5; i++ {
		_, err = lkvA.Get(ctx, "k5")
		must(err)
	}
	p3 := proposals()
	fmt.Printf("   proposals: start=%s after B linearizable=%s after B serializable=%s after A=%s\n", p0, p1, p2, p3)

	time.Sleep(500 * time.Millisecond)
	fmt.Println("== watch events seen by a plain watcher on prefix \"k\":")
	for {
		select {
		case e := <-events:
			fmt.Println("  ", e)
		default:
			return
		}
	}
}
```

Output of the run on 2026-11-02 (etcd v3.7.3, go1.26.10):

```
== 1. B (not owner) puts k1; nobody owns k1
   before: proposals=6 rev=1 version=0
   after:  proposals=8 rev=3 version=2
== 2. A owns k2 (by reading it); B puts k2
   before: proposals=10 rev=5 version=1
   after:  proposals=15 rev=9 version=3
== 3. B deletes k3 WithPrevKV; nobody owns k3
   resp.Deleted=0 len(PrevKvs)=0; key still present=false
== 4. B puts k4=new WithPrevKV; value before the call is "old"
   PrevKv.Value = new
== 5. A owns k5; B reads k5 five times (linearizable), then five times serializable; then A five times
   proposals: start=23 after B linearizable=28 after B serializable=28 after A=28
== watch events seen by a plain watcher on prefix "k":
   PUT k1=v1 mod=2 ver=1
   PUT k1=v1 mod=3 ver=2
   PUT k2=v0 mod=4 ver=1
   PUT k2=v2 mod=8 ver=2
   PUT k2=v2 mod=9 ver=3
   PUT k3=x mod=10 ver=1
   DELETE k3= mod=11 ver=0
   PUT k4=old mod=12 ver=1
   PUT k4=new mod=13 ver=2
   PUT k4=new mod=14 ver=3
   PUT k5=v mod=15 ver=1
```

Two earlier separate runs gave the same counts for cases 1, 2 and 5 and the same responses for cases 3 and 4.

**Limits.**
- One member, one run per configuration.
- The double application follows from a code path that depends on neither cluster size nor timing, so one member suffices to show it.
- The contested-write count could differ under concurrent contention, through retries.
