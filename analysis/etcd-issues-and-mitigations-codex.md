# etcd: production risks and deployment mitigations

**Versions analyzed:** etcd server and Go client `v3.7.3` (`0a173d0410a9`); jetcd `jetcd-0.8.7` (`23b61f1499a5`). Still the latest stable releases when checked on 2026-11-03.

**Upstream references:** linked beside the issues below; their open/merged states were rechecked on 2026-11-02. Watch liveness documentation and selected source paths were reviewed on 2026-11-03. Detailed evidence comes from the five Codex analyses in the appendix.

**Summary — High confidence:** consistent KV storage does not automatically provide timely expiry, safe external lock ownership, eventual watch progress, or fresh application caches. Application controls are essential. Performance benefits and the severity of untested scenarios remain **Moderate-confidence**, workload-dependent assessments.

Scope: leases, watches, and writes. **High** means verified mechanisms or documented behavior; **Moderate** marks untested scenarios. Mitigations are recommendations; no capacity or incident-frequency measurements are available.

## Leases and ownership

Ordinary renewal is leader-local; grants and revocations are replicated. Expiration decisions still need separate correctness checks. [Evidence](leases-codex.md#2-replicated-state-and-local-lease-timing).

| Risk, scenario, and impact | Deployment or application action |
|---|---|
| **Late expiry — High.** Failover restarts surviving lifetimes by default. Continuing elections can indefinitely delay cleanup or takeover; overload also delays deletion. | Do not use TTL as a hard deadline. Enforce an application expiration time, with explicit clock assumptions, when records must become unusable on time. Physical cleanup still requires availability. |
| **Early expiry with checkpointing — High, conditional.** A failed checkpoint clear can acknowledge more lifetime than installed ([#22649](https://github.com/etcd-io/etcd/issues/22649)). | Leave Alpha `LeaseCheckpoint` off until fixed and validated; [PR #22650](https://github.com/etcd-io/etcd/pull/22650) is unmerged. It defaults off. Checkpointing provides no hard deadline and normally excludes newly granted leases of five minutes or less. |
| **Stale ownership belief — High.** Delayed replies extend apparent client lifetime. Default-path stale-revocation and renewal/expiry races are additional **Moderate, untested** candidates for premature loss. | Check ownership in the **same transaction** as each protected etcd write. External destinations must enforce fencing that rejects obsolete owners. Timers, `Session.Done()`, and separate ownership reads are insufficient. |
| **Unwanted renewal — High mechanisms.** Go cleanup bugs misassociate subscribers or leave orphan renewal running ([#22081](https://github.com/etcd-io/etcd/issues/22081), [#22094](https://github.com/etcd-io/etcd/issues/22094)). jetcd can renew after an error callback. | On intentional release, explicitly revoke when all attached uses may end; otherwise verify cleanup or use a corrected client. Close jetcd registrations. Test server-side disappearance rather than trusting local notification. |
| **Stale unlock — High mechanism.** An ambiguous unlock retried after same-session reacquisition can delete the newer lock ([#22082](https://github.com/etcd-io/etcd/issues/22082)). | Use generation-conditional deletion, or prevent session/key reuse while old unlocks can execute. Bound acquisition waits and cancel on session loss. |

## Watches and caches

**Liveness gap — High in the mechanisms.** etcd's [own documentation claims eventual consistency for Watch](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/README.md#L111), which requires eventual convergence; the [API documentation permits nonarrival in unhealthy clusters](https://etcd.io/docs/v3.7/learning/api_guarantees/#apis-to-consider). **Even a direct `WithRequireLeader` watch can remain silently stale while a reachable majority commits updates.** A storage-stalled member can retain a nonzero leader ID, leaving the flag's detector inactive. Client and proxy failures add other paths. [Exact quotations](watchers-codex.md#15-eventual-consistency-requires-eventual-progress); [counterexample and verification limits](watchers-codex.md#26-direct-leader-required-watches-can-remain-silently-stalled).

Use direct member connections, reachable alternative endpoints, and `WithRequireLeader` as a **partial failure detector, not a liveness guarantee**. Add independent progress supervision: periodically obtain a linearizable revision, require application-processed watch coverage through it, and recover or rebuild on timeout. Watch callbacks and local keepalives/progress alone cannot reliably expose these stalls. Independent checks detect unverified progress; they cannot guarantee recovery from a blocked client. [Recovery protocol](watchers-codex.md#72-progress-supervision-without-blind-scheduled-restarts).

For a **linearizable cached read**, obtain the target revision **after that read begins**; wait for coverage or fail/read directly on timeout. Periodic checks cannot authorize later reads. [Freshness reasoning](chubby-perspectives-codex.md#84-a-progress-watermark-cannot-authorize-future-cached-reads).

Read a consistent snapshot at **R**, watch from **R+1**, and resume after the last **complete, application-processed** revision. Persist state and cursor consistently. Rebuild after compaction, uncertain continuity, or unusable events. Snapshots restore current state, not missed transitions. [Recovery](watchers-codex.md#7-consumer-recovery-and-freshness-policy).

| Current concern and impact | Action and remaining limit |
|---|---|
| **Optional gRPC proxy — High mechanisms; leader-loss discrepancy reproduced.** Leader-required metadata is mishandled, explicit progress requests are unsupported, and ended upstream broadcasts can accept new watchers ([#21813](https://github.com/etcd-io/etcd/issues/21813)). | Prefer direct member connections. If a proxy is required, validate failure signaling through the actual networked topology. |
| **Client recovery gaps.** Go can close without its final error (**High**). jetcd can skip history after disconnecting between creation acknowledgment and replay, or stop retries after a scheduled-resume exception ([#1227](https://github.com/etcd-io/jetcd/issues/1227)); these schedules are **Moderate, untested**. | Treat unexpected closure as failure. Start Go watches at explicit revisions. jetcd needs a validated correction or reconstruction on uncertainty; an explicit revision alone does not fix its cursor jump. Coordinate library/application retries. |
| **Stream interference.** Cancellation bursts can block dispatch ([#18879](https://github.com/etcd-io/etcd/issues/18879), **Moderate**); future/lagging watches suppress stream-wide requested progress (**High**). Reused explicit IDs can receive old events ([#22673](https://github.com/etcd-io/etcd/issues/22673), **High mechanism**). | Isolate critical streams from bulk cancellation and future watches. Match Go watch/progress metadata. Do not reuse explicit IDs within a stream. |
| **Optional caches.** `cache/` can remain unready ([#22614](https://github.com/etcd-io/etcd/issues/22614)); `client/v3/leasing` has eviction defects and extra writes (**High mechanisms**), plus stale-read/writer-stall candidates (**Moderate**). | Avoid these unvalidated shortcuts. They are separate from Kubernetes's watch cache. Lease health alone cannot certify cache validity. |

## Write performance and operational headroom

**High confidence in the mechanisms; workload impact unmeasured.** These costs can combine during recovery:

| Pressure and impact | Mitigation |
|---|---|
| **Slow watchers can slow writes.** Catch-up scans hold a write-completion lock; Go buffers events without a hard limit. [#22264](https://github.com/etcd-io/etcd/issues/22264), [#16839](https://github.com/etcd-io/etcd/issues/16839). | Drain consumers, monitor lag/memory, and rebuild disposable state when excessively behind. Budget retention for outages **plus catch-up**; longer history also costs storage and replay work. |
| **Expiration bursts.** Each revoke replicates a request and deletes every attached key in one transaction; JWT can add signing work. | Spread expirations where semantics permit. Reuse leases judiciously: fewer leases reduce requests, but oversized attachment groups create expensive deletions. Test mass expiry under load. |
| **Storage/apply stalls delay reads too.** WAL sync, quorum latency, ordered application, backend commits, and watches can each limit capacity. | Use durable low-latency SSDs and reliable networking; isolate I/O contention. Preserve fsync, reduce redundant writes, bound concurrency/retries, and batch carefully. Measure before tuning. [Hardware guidance](https://etcd.io/docs/v3.7/op-guide/hardware/). |
| **Maintenance/space exhaustion.** Compaction removes replay history; defrag blocks its member; quota exhaustion restricts writes. | Alert on database size/free disk. Compact deliberately; defrag one member at a time, preserving quorum and checking recovery. Rehearse restore and cache invalidation. [Maintenance guidance](https://etcd.io/docs/v3.7/op-guide/maintenance/). |

Monitor **WAL/backend latency, committed-to-applied backlog, leader changes, client read/write tail latency, watch lag, memory, and disk headroom** together. Test production-like hardware through maintenance, expiry bursts, failure, and catch-up.

Backend/WAL replacements and asynchronous Raft storage require development and correctness validation. Mostly non-blocking defrag is on **upstream main** ([#22425](https://github.com/etcd-io/etcd/pull/22425), `d6af8028d9ce`), **absent from v3.7.3**. These are not deployment switches. [Storage](storage-codex.md), [WAL assessment](wal-replacement-codex.md).

## Questions to resolve before deployment

- Do you need current state, every transition, bounded expiry, or exclusive external effects? Who enforces each guarantee?
- What staleness, takeover delay, outage, and data-loss limits are acceptable?
- Who detects silent watch stalls and triggers recovery? Does the client recover through partitions, ambiguous replies, compaction, and restore? Test jetcd against the actual server version; its launcher defaults to an older release.
- Does sustained mixed load meet read/write targets with bounded memory, backlog, and disk use? Who owns recovery and client fixes?

In the inspected Kubernetes v1.37.2 paths, **coordination Lease objects use ordinary writes**; Events and selected endpoint records use native TTL leases. Their exposure differs. [Specifics](leases-codex.md#7-kubernetes-consequences).

## Appendix: detailed evidence

All five analyses informed this brief. They retain historical fixes and verification limits. No new functional tests or benchmarks were run. Code anchors are **@ etcd v3.7.3**, unless stated otherwise.

| Analysis | Read for specifics | Representative source anchors |
|---|---|---|
| [leases-codex.md](leases-codex.md) | §§3–6: timing, checkpoint error, client cleanup, expiration costs; §7: Kubernetes; Appendix B: locks. | `server/etcdserver/server.go:394`; `server/lease/lessor.go:326,480`; `client/v3/concurrency/mutex.go:134,152`. |
| [watchers-codex.md](watchers-codex.md) | §1.5: eventual consistency and quotations; §2.6: silence despite `WithRequireLeader`; §§2–4: signaling, proxy, cancellation, IDs, memory; §6: jetcd; §7: recovery/watchdog; Appendix C: two functional experiments. Includes #22294 and the unconfirmed production trigger for #22184. | `client/v3/watch.go:82,478,879`; `server/storage/mvcc/watchable_store.go:346`; `jetcd-core/src/main/java/io/etcd/jetcd/impl/WatchImpl.java:267,357 @ jetcd-0.8.7`. |
| [chubby-perspectives-codex.md](chubby-perspectives-codex.md) | §§3–5, 8: cache validity, fencing, limits of freshness checks; Appendix B: leasing-cache defects and extra transactions. | `client/v3/leasing/cache.go:178`; `client/v3/leasing/kv.go:218,467`. |
| [storage-codex.md](storage-codex.md) | §§3, 6–8: existing write costs, maintenance, lower-cost mitigations, measurements needed before a backend replacement. | `server/storage/backend/backend.go:34,476`; `server/storage/wal/wal.go:995`. |
| [wal-replacement-codex.md](wal-replacement-codex.md) | §§2, 6–7, 9–11: durability/apply distinction, asynchronous-storage risks, read latency, recovery and test requirements. | `server/etcdserver/raft.go:218,256`; `server/etcdserver/read/read.go:117,133`. |
