# Leases: claims rejected from the Codex and Antigravity analyses (Claude)

**Versions analyzed:** etcd `v3.7.3` (`0a173d0410a9`); jetcd `jetcd-0.8.7`
(`23b61f1499a5`); raft `v3.7.0` (`raft.go`); Kubernetes `v1.37.2` (two
files). Claims were taken from the two peer files as committed at `e19f63d`.
Both files were rewritten on 2026-11-01 (Antigravity about 20:18, Codex
about 20:28); the last column says whether a claim survives in those
rewrites.

**Upstream references:** etcd [#13915](https://github.com/etcd-io/etcd/issues/13915),
[#18428](https://github.com/etcd-io/etcd/pull/18428),
[#20589](https://github.com/etcd-io/etcd/pull/20589),
[#22649](https://github.com/etcd-io/etcd/issues/22649),
[#22714](https://github.com/etcd-io/etcd/issues/22714) (added 2026-11-11);
Kubernetes [#65497](https://github.com/kubernetes/kubernetes/issues/65497).

**Summary:**
- The only substantive rejection is Antigravity's claim that a partitioned
  leader breaks the renewal guarantee. The ET extension (one election
  timeout) that a new leader adds when it takes over covers that case.
  — **Moderate**.
  - **Qualified 2026-11-11.** The rejection holds for the mechanism
    Antigravity gave: the extension covers the renewals that the cut-off
    leader acknowledged, so the new leader does not revoke early on their
    account. The headline does not hold in general. Upstream #22714 (filed
    2026-11-06, open, not re-run by this project) reports that expiry
    revocations queued on a partitioned leader are applied after it steps
    down, and delete leases that clients are renewing on the new leader.
    See [`leases-claude.md`](leases-claude.md) §3.4, R2. — **High**
    (mechanism, from code); **Moderate** (the reported loss rates).
- The other rejections are an over-read of Kubernetes design history and
  four factual errors.
- One rejection was withdrawn in part after Antigravity's reply. The
  response-header revision is a valid lower bound only for *successful*
  renewals; on "lease not found" responses it can precede the revocation.
  This is now a Low-impact P4 finding (P4: lease reads are linearizable).
- Nothing material from the Codex file was rejected. Two of its items were
  qualified.

The analysis that adopted the rest is
[`leases-claude.md`](leases-claude.md), Appendix C.2. Terms used below:
- **P3:** a lease is not revoked within TTL of a renewal the server
  acknowledged.
- **ET:** the Raft election timeout.
- **`Promote`:** the new leader's lease manager taking over. It extends every
  lease to now + ET + TTL.

## Rejected (Antigravity, `leases-agy.md`)

| Claim | Why rejected | In the rewrite? |
|---|---|---|
| **A partitioned leader breaks P3.** `isActive()` keeps returning true, so a deposed leader's acknowledged renewals are lost and the new leader revokes early (High confidence). | The mechanism is right, the consequence is not. Per raft `v3.7.0`, the isolated leader steps down by partition + 2·ET (`raft.go:862-880`, `:1281-1292`). Followers cannot elect before about partition + ET − 1 tick (`:850-859`, `:1103`, `:2049-2055`). `Promote` adds ET + TTL after that. So every acknowledgement the old leader gave is covered to within about one tick (100 ms). The real exposure is with checkpointing on, through #22649. **Qualified 2026-11-11:** this covers acknowledged renewals only. A partitioned leader can still break P3 by a different mechanism, its queued expiry revocations when more than 16 leases expire in one batch (`leases-claude.md` §3.4, R2; upstream #22714). | **Corrected.** The rewrite now concludes that uncheckpointed leases survive, and moves the violation to #22649. |
| **Kubernetes "abandoned"/"refused" native etcd leases, the "clearest production verdict"** on them. | Kubernetes components reach state only through the API server, so coordination had to be an API object. That is architecture, not a verdict on etcd leases. | Phrasing removed. |
| **Keep-alive response headers carry the follower's revision, a linearizability bug** ("Follower-Stamped" header). | **Rejection withdrawn in part; see "Reconsidered" below.** | Still present, narrowed to the negative-response case. |
| **`FastLeaseKeepAlive` breaks lease linearizability**, by renewing a lease whose revoke is committed but not applied. | Narrowed, not rejected. PR #20589 states this trade-off and accepts it. It cannot break P3: the revoke is either explicit (excluded) or expiry-driven, and `Renew` already refuses expired leases (`lessor.go:415-430`). What remains is a narrow real-time-order case: the revoke is acknowledged by a member that applied it before the leader did. | — |
| **#22649 example:** client told 600 s while the server revokes after about 10 s. | Overstated for default settings. Under steady renewal every TTL/3, a checkpointed remaining TTL is at least about 2/3 TTL. A value as small as 10 s needs irregular renewals or a non-default interval. | — |
| **RBAC check on follower-forwarded renewals** as a correctness finding. | The code path is real: `checkLeaseRenew` skips key checks for leases the local member doesn't know yet (`v3_server.go:593`). But a renewal only *extends* a lease, during the apply lag after a grant or attach. Recorded, not tracked as a lease-correctness issue. | Still present (as a bounded concern). |
| **Events kept "until `NOSPACE`"** under leader churn. | Not in the #65497 text I read. Unverified, so not used. | Still present. |
| **jetcd `keepAliveOnce` NullPointerException** (rewrite only). | The duplicate write is real and was adopted. The NPE is not user-visible: it fires only when the stream never opened, so the future has already failed, and `whenComplete` returns the original error. | Still present. |

**Factual errors** (Antigravity, committed version):
- **#13915 closing date.** The file says 2025-12-12. The GitHub API says
  2025-12-19.
- **`--experimental-lease-checkpoint-interval` flag.** It does not exist at
  `v3.7.3`. The interval is set only through `server/config/config.go:164`,
  which only tests use.
- **`lease_manager` "defaults to 60 s".** 60 s caps the *reuse window*
  (min(5 % of TTL, 60 s)). It is not the lease TTL.

All three are gone from the Antigravity rewrite. The Codex rewrite
independently reports that the flag was not found.

## Reconsidered: response-header revisions (Antigravity)

**What I wrote first.** The keep-alive header is filled before the renewal
on purpose, as a lower bound (code comment at
`server/etcdserver/api/v3rpc/lease.go:127-134`), and no client contract
depends on it.

**Antigravity's reply.** The lower bound protects only TTL > 0 responses. On
`ErrLeaseNotFound`, the handler sends TTL 0 with the same pre-filled header
(`:136-140`). From a lagging follower, that stamps a revision from before
the revocation onto a "lease gone" answer. `LeaseTimeToLive` likewise mixes
the leader's lease data with the local member's revision.

**My check.** The reply is right, and the problem is wider than its
example.
- **It happens on the leader too.** A renewal of an expired lease fills the
  header, then waits for the revocation to apply (`server/lease/lessor.go:415-430`).
  So a lease with attached keys is reported gone at a revision at which it
  and its keys still existed. No lagging follower is needed.
- **`LeaseTimeToLive` has no consistent direction.** The header is filled
  *after* the result (`v3rpc/lease.go:62-75`).
  - On a follower, the TTL and keys come from the leader's newer state,
    whose header the HTTP bridge drops (`server/lease/leasehttp/http.go:129`,
    "TODO: fill out ResponseHeader").
  - On the leader (`server/etcdserver/v3_server.go:650`, same TODO), a
    positive result can carry a revision later than a revocation that
    happened in between.
- **What still stands from my first answer:**
  - the pre-fill is deliberate, and correct for successful renewals;
  - the API defines `revision` as "the key-value store revision when the
    request was applied", unset for calls that do not touch the store
    (`api/etcdserverpb/rpc.proto:424-425`), so no documented contract makes
    these revisions snapshot points;
  - etcd's own recipes never read them.
- **Exposure is small.**
  - The Go streaming `KeepAlive` drops TTL ≤ 0 responses
    (`client/v3/lease.go:534-538`).
  - jetcd reports only the error.
  - The revision is visible through Go `KeepAliveOnce`, which returns the
    response together with `ErrLeaseNotFound` (`client/v3/lease.go:323-326`),
    through `TimeToLive`, and through raw gRPC.

**Verdict.** A P4 metadata inconsistency, not a correctness bug for P3. —
**High** for the code; impact **Low**. Two of the reply's citations are off:
- I cited `v3rpc/lease.go:127-134`, not `v3_server.go:500-505`;
- `v3_server.go:725` is `newHeader`, while the `TimeToLive` header is filled
  at `v3rpc/lease.go:74`.

Neither changes the substance. Now covered in `leases-claude.md` §2.5.

## Qualified (Codex, `leases-codex.md`)

- **R5 (delayed revocation after a leader change).** Adopted, with impact
  rated Low. The victim's lease had already expired on the old leader. A Go
  client also stops renewing within about 1 s of that expiry, when its local
  deadline is reaped. So realistic exposure is limited to applications that
  call `KeepAlive` or `KeepAliveOnce` again on the same lease ID.
- **External blog reference (amyangfei, 2020, v3.4.14).** Not re-checked,
  so not cited. Its points are covered by code-level evidence.
