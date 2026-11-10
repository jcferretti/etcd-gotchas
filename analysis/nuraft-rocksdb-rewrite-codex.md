# An etcd-compatible rewrite using NuRaft and RocksDB

**Date:** 2026-11-08.

**Versions analyzed:** etcd server and Go client `v3.7.3`, commit `0a173d0410a96d01091b484daae81365bbcde57e`; jetcd `jetcd-0.8.7`, commit `23b61f1499a5d340ae2f267d88d45ca3900425a4`; ClickHouse/NuRaft `master`, commit `e9375226c8851be59c6a7a15b4322f66d3eef5e6`; RocksDB `v11.13.2`, commit `17b624b5c0d41bd42db5cf88aa867f3b26eaf1fa`; Jepsen's etcd harness, commit `b47cc401f27bfb7c28afd9ac088b5f0dca4501b8`. Selected Kubernetes tests are pinned to `v1.37.2`, commit `c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32`.

**Upstream references:** etcd [#22332](https://github.com/etcd-io/etcd/issues/22332), alternative storage investigation; [#22339](https://github.com/etcd-io/etcd/pull/22339), proposed storage correctness tests; jetcd [#1444](https://github.com/etcd-io/jetcd/issues/1444), unsafe default retries, fixed by [`05f92e044328`](https://github.com/etcd-io/jetcd/commit/05f92e04432849d4d091efc49397554df9682c6f); historical [Jepsen etcd 3.4.3 report](https://jepsen.io/analyses/etcd-3.4.3) and [jetcd 0.8.2 report](https://jepsen.io/analyses/jetcd-0.8.2). These are evidence and test leads, not certification of the proposed implementation.

**Summary:** Technically feasible, but a new distributed database product rather than a two-library integration (**High confidence in the missing components; Moderate in feasibility**). Preserve the client gRPC protocol initially (**Moderate confidence**). A broadly compatible production candidate merits an initial budget of **80–160 engineer-months**, including testing and operational work (**Low-confidence planning estimate**, §7).

## Scope

Assess a complete etcd server rewrite in C++ or Rust using ClickHouse's NuRaft fork for consensus and C++ RocksDB for persistence, comparing client wire compatibility with replacement language APIs, and identifying implementation cost, reusable tests, and production acceptance gates.

## Summary

1. **The architecture is feasible; the proposed libraries cover only part of the product.** NuRaft supplies consensus machinery and peer transport but requires application implementations of the durable log store, state manager, and state machine. RocksDB supplies local storage primitives. Revisioned transactions, historical reads, watches, leases, authentication, maintenance, recovery orchestration, and compatibility remain substantial work. **Verified facts: High confidence; overall feasibility: Moderate confidence.** (§§2–3)
2. **Preserve etcd's client gRPC protocol first.** This permits unmodified clients and substantial reuse of existing API tests while allowing a completely different peer protocol and storage format. Dropping the wire protocol removes some server compatibility work but adds a client product for every supported language; it does not remove the shared semantic core. **Recommendation: Moderate confidence.** (§1)
3. **Language API compatibility is narrower than it initially sounds.** Public types, errors, asynchronous behavior, cancellation, retry rules, and transport escape hatches belong to the observable API. Rebuilding a Go client does not make an already compiled Kubernetes API server work with a new wire protocol. **High confidence in the dependencies; Moderate in integration effort.** (§§1.3, 4)
4. **NuRaft requires specific correctness choices.** Its `async_replication_` mode returns pre-commit results before quorum agreement and is unsuitable for etcd success acknowledgments. Its asynchronous completion-handler mode is different. I did not find a public etcd-style `ReadIndex` facility in the inspected fork; begin with a committed read barrier and separately qualify a faster read protocol. **High confidence in source/API observations; Moderate confidence in the proposed integration.** (§2.1)
5. **Tests are a major reusable asset, with substantial adaptation cost.** Prioritize etcd's `tests/common`, robustness models and validators, issue regressions, Jepsen workloads, actual Go/jetcd clients, and Kubernetes storage tests. Embedded-Go fixtures, etcd WAL decoders, Go failpoints, process flags, and assumptions about file formats cannot simply follow the replacement binary. **High confidence.** (§5)
6. **Heavy coding-agent use does not justify budgeting this as a short rewrite.** Agents are useful for adapters, fixture ports, generators, and constrained implementation tasks. The durable-state protocol, semantic specification, independent oracles, integration debugging, and operational qualification still require accountable engineering. No measured productivity multiplier for this project is available. **Assessment: Moderate confidence; all numerical estimates: Low confidence.** (§§4.3, 7)
7. **The performance case remains unproved.** A rewrite can change the MVCC representation, avoid the Go/native crossings of a Go/RocksDB fork, and remove bbolt-specific maintenance costs. It retains quorum, ordered-state, history, and watch-delivery obligations. RocksDB introduces compaction and resource-management tradeoffs. Benchmark representative mixed workloads before funding broad compatibility. **Scope facts: High confidence; prospective benefit: Moderate confidence with no numerical speedup forecast.** (§6)

This is a source-based feasibility study. No replacement was implemented, and no build, functional test, fault campaign, or performance benchmark was run. “Verified” means source or upstream metadata inspected; proposed designs and effort estimates are not observed results.

## 1. Define compatibility before choosing the implementation

### 1.1 Four independent compatibility boundaries

An **etcd-compatible replacement** needs a stated contract: which applications and operational procedures can use it without modification. The following boundaries must not be conflated.

| Boundary | What compatibility requires | Needed for the proposed replacement? |
|---|---|---|
| Client wire protocol | Service/method names, protobuf field meanings, gRPC status and metadata, streaming behavior, and associated consistency guarantees | Yes for unmodified network clients; optional with replacement libraries |
| Language API | Public symbols/types, options, return values/errors, threading or async behavior, cancellation and recovery semantics | Existing libraries provide this when the wire protocol is preserved; otherwise each supported library needs implementation and qualification |
| Peer protocol and Raft membership | Nodes understand the same replication/configuration protocol | Not required for a separate homogeneous replacement cluster; required for mixed stock-etcd/replacement membership |
| Operational and storage interfaces | Backup/restore formats, admin behavior, flags, metrics, health endpoints, upgrades and migration | Must be specified separately; client wire compatibility alone does not provide it |

**High confidence:** The user's first approach does not require etcd's server-to-server protocol. A NuRaft-only cluster may expose the existing client protocol. It cannot join an etcd Raft group merely because both implement Raft. They have different messages, configuration machinery, logs, and snapshot integration. A mixed implementation cluster would be a separate, much larger project; it is excluded from the estimates below.

There is a concrete architectural precedent: ClickHouse Keeper exposes ZooKeeper's client protocol while using Raft and incompatible interserver and snapshot formats. Its official documentation explicitly excludes mixed ZooKeeper/Keeper clusters. This supports the separation of compatibility boundaries, not an inference that Keeper implements etcd's semantics or proves this design correct. **High confidence in the documented precedent.** [Keeper implementation details](https://clickhouse.com/docs/guides/oss/deployment-and-scaling/keeper#implementation-details).

### 1.2 Approach A: preserve the gRPC client protocol

**Recommendation — Moderate confidence:** Generate the client-facing service interfaces from the pinned etcd protobuf definitions and implement their behavior over an independent server core. Keep the internal command format, peer transport, durable representation, and scheduling independent of the external request classes.

Generated stubs handle serialization; they do not implement compatibility. The surface includes:

| Area | Behavior that must be implemented and tested |
|---|---|
| KV and transactions | Arbitrary byte keys/values; range endpoints, sorting, filtering, limits and counts; historical revisions; compare predicates; nested transactions; ordered responses; duplicate-write validation; previous values; ignore-value/ignore-lease rules |
| Revisions | Global store revision, per-key creation/modification revisions and version, transaction revision grouping, compaction boundary, and response headers |
| Watches | Bidirectional streams; creation and cancellation; inclusive replay; ordering and revision atomicity; previous values; filters; fragmentation; progress requests; compaction errors; slow-consumer recovery |
| Leases | Grant/revoke/list/TTL; multiplexed keepalive; key attachment; expiration; leader changes and restart behavior |
| Coordination | Lock and Election services, plus compatibility with client algorithms that construct coordination from KV/Txn/Watch/Lease |
| Authentication | TLS and client certificates; user/role/range permissions; tokens; auth revisions; permission changes during streams and transactions |
| Administration | Membership and learners; leader transfer; alarms, status and hashes; compaction, defragmentation, snapshots and version handling |
| Transport details | Deadlines and cancellation, message limits, status descriptions, authentication metadata, leader-required streams, reconnect behavior, and any promised HTTP/JSON gateway |

**Verified, High confidence:** The target is not an old minimal v3 interface. `v3.7.3` includes `KV.RangeStream` at `api/etcdserverpb/rpc.proto:42–47`. The same file defines KV, Watch, Lease, Cluster, Maintenance, and Auth at lines 33, 92, 106, 163, 205, and 279. Lock and Election have separate protobuf files under `server/etcdserver/api/`. A compatibility checklist copied from a 3.4-era implementation would omit current surface area. [Pinned main services](https://github.com/etcd-io/etcd/blob/v3.7.3/api/etcdserverpb/rpc.proto#L33), [Lock](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/api/v3lock/v3lockpb/v3lock.proto#L10), [Election](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/api/v3election/v3electionpb/v3election.proto#L10).

**Verified, High confidence:** Error descriptions are part of effective interoperability. The Go client maps errors by their descriptions (`api/v3rpc/rpctypes/error.go:256 @ v3.7.3`); jetcd matches specific descriptions when deciding whether authentication failures are retryable (`jetcd-core/src/main/java/io/etcd/jetcd/support/Errors.java:39–60 @ jetcd-0.8.7`). Returning a generic status code can change recovery behavior even when the protobuf payload is correct. [Go error mapping](https://github.com/etcd-io/etcd/blob/v3.7.3/api/v3rpc/rpctypes/error.go#L256), [jetcd classification](https://github.com/etcd-io/jetcd/blob/jetcd-0.8.7/jetcd-core/src/main/java/io/etcd/jetcd/support/Errors.java#L39).

**Proposal:** Publish a versioned feature matrix with three distinct states: supported with compatibility tests, intentionally different with documented consequences, and unsupported. A restricted initial deployment can omit features its applications do not use. It should be described as a supported application profile, not universal etcd compatibility. For example, a successful Kubernetes boot is evidence about that execution, not about Auth, Lock, snapshot restore, or every Kubernetes recovery path.

### 1.3 Approach B: replace the language clients

**Assessment — Moderate confidence:** This is attractive for a controlled application fleet and a deliberate semantic redesign. It is usually less attractive for a general etcd replacement.

There is no single “language-level etcd API.” The Go client and jetcd expose different object models. Other clients add independent surfaces. For each supported library, decide whether compatibility means source compatibility after rebuilding, binary compatibility, or a narrower interface used by selected applications.

**Verified, High confidence:** Go's `clientv3.Client` embeds service interfaces and owns a gRPC connection; `ActiveConnection()` publicly returns `*grpc.ClientConn` (`client/v3/client.go:51,565 @ v3.7.3`). Applications can also import generated protobuf clients. A replacement library that eliminates gRPC cannot transparently satisfy arbitrary uses of those escape hatches. It may retain the public protobuf data types while replacing transport, but that is different from supporting raw generated stubs. [Public Go client](https://github.com/etcd-io/etcd/blob/v3.7.3/client/v3/client.go#L51), [connection escape hatch](https://github.com/etcd-io/etcd/blob/v3.7.3/client/v3/client.go#L565).

The replacement client scope includes endpoint selection, TLS/authentication, deadlines, cancellation, backpressure, reconnection, lease keepalive multiplexing, watch replay, exception/error identity, builders/options, futures/channels, and coordination helpers. Matching method names while changing these behaviors is insufficient. A Java jar substitution also requires checking binary signatures and dependency interactions; a Go module replacement normally requires rebuilding consumers. **High confidence in the compatibility distinction; Moderate confidence in the resulting workload.**

Keeping the same semantics leaves the server's hardest obligations intact. Meaningful simplification comes from explicitly dropping historical reads, transaction forms, watch guarantees, lease features, or admin behavior. That is a smaller product scope, not a benefit intrinsic to replacing protobuf.

**A hybrid is viable — Moderate confidence:** Implement one semantic core, expose the etcd gRPC facade, and later offer a native protocol/client for stronger or cheaper operations. New request identifiers, acknowledged delivery, or richer sessions can be optional capabilities. Do not maintain two independently implemented transaction or lease engines. A facade also preserves an independent compatibility-test path while native clients evolve.

### 1.4 Compatibility does not require reproducing known bugs

**Recommendation — Moderate confidence:** Separate documented guarantees, observable behavior relied upon by supported clients, and historical defects. Fix the last category; record intentional differences rather than interpreting every difference from stock etcd as a regression. Differential tests need adjudication against a specification, not an assumption that the reference binary is always right.

The preceding project analyses make this distinction necessary:

| Earlier analysis | Consequence for a rewrite |
|---|---|
| [Storage](storage-codex.md), backend replacement assessment | RocksDB primitives do not supply etcd's historical-revision model, recovery boundary, snapshot tools or maintenance meanings |
| [WAL replacement](wal-replacement-codex.md), durable-log/state integration | Removing duplicate logging requires a recovery protocol; a database library does not remove persistence ordering obligations |
| [Leases](leases-codex.md), authority and expiration analysis | Replicated deletion is not proof that a timer decision was safe; renewal promises and failover must agree |
| [Watches](watchers-codex.md), delivery and liveness analysis | Correct committed data does not establish correct replay, truthful progress, or eventual delivery |
| [Chubby comparison](chubby-perspectives-codex.md), session/cache authority analysis | An invalidation-acknowledgment protocol requires cooperating clients; it cannot be silently imposed on unmodified etcd clients |

**High confidence in the architectural constraint:** A wire-compatible server can improve timer handling, progress detection and resource bounds. It cannot make old clients acknowledge application-level cache invalidation when their protocol provides no such acknowledgment. A replacement language library could add that internal protocol, but preserving every externally visible operation and latency expectation would still require analysis. Chubby's coupled invalidation/KeepAlive protocol is described in §2.7–§2.8, `doc/chubby-osdi06.md:229–263`; the prior comparison explains why the coupling, rather than a transport choice, matters.

## 2. What NuRaft and RocksDB actually provide

### 2.1 NuRaft: consensus foundation with important application obligations

**Verified, High confidence:** NuRaft explicitly assigns three modules to its user: the log store, state machine, and state manager. Raft and the Asio network/timer layer are supplied. `docs/how_to_use.md:14–26 @ e9375226c885` is unusually clear about this boundary. Its examples keep log/state in memory and are not production persistence implementations. [Module contract](https://github.com/ClickHouse/NuRaft/blob/e9375226c8851be59c6a7a15b4322f66d3eef5e6/docs/how_to_use.md#L14), [in-memory manager](https://github.com/ClickHouse/NuRaft/blob/e9375226c8851be59c6a7a15b4322f66d3eef5e6/examples/in_memory_state_mgr.hxx#L43).

| Interface or facility | Reused functionality | Work the replacement must own |
|---|---|---|
| `log_store` | Interface for append/read/overwrite/packing/compaction/flush | Durable encoding and indexes, suffix truncation, recovery, corruption handling, I/O errors and sync semantics |
| `state_mgr` | Hooks to load/save Raft state and configuration | Durable term/vote/configuration, identity mapping, bootstrap and restart policy |
| `state_machine` | Ordered commit callbacks and snapshot hooks | Entire etcd semantic engine, applied-index tracking, snapshot creation/install and recovery |
| Elections, membership, learners and transfer | Existing consensus machinery | Mapping to etcd admin semantics, promotion gates, persisted configuration transitions and operational tests |
| Asio transport and threading | Peer RPC transport and timers | Client gRPC frontend, bounded queues, callback ownership, shutdown, TLS configuration and observability |

Sources: `include/libnuraft/log_store.hxx:62,70,180,217`; `state_mgr.hxx:48–88`; `state_machine.hxx:74,92,107,225,303,312 @ e9375226c885`. [Log interface](https://github.com/ClickHouse/NuRaft/blob/e9375226c8851be59c6a7a15b4322f66d3eef5e6/include/libnuraft/log_store.hxx#L62), [state manager](https://github.com/ClickHouse/NuRaft/blob/e9375226c8851be59c6a7a15b4322f66d3eef5e6/include/libnuraft/state_mgr.hxx#L48), [state machine](https://github.com/ClickHouse/NuRaft/blob/e9375226c8851be59c6a7a15b4322f66d3eef5e6/include/libnuraft/state_machine.hxx#L74).

Three choices deserve explicit gates:

**First, do not confuse asynchronous execution with early acknowledgment. High confidence.** `cluster_config::async_replication_` returns the result of `pre_commit()` before replication completes. That conflicts with etcd's acknowledged-write contract. In contrast, `raft_params::async_handler` avoids blocking the caller while delivering the committed result later. Use the latter if needed; acknowledge success only after commitment and appropriate application. `docs/async_replication.md:4–39` and `src/handle_client_request.cxx:242–275 @ e9375226c885` establish the difference. [NuRaft's mode distinction](https://github.com/ClickHouse/NuRaft/blob/e9375226c8851be59c6a7a15b4322f66d3eef5e6/docs/async_replication.md#L4).

**Second, linearizable reads need a protocol. High confidence in the missing integration; Moderate in the proposed starting design.** Inspection/search of the pinned public headers and implementation did not locate an etcd-style public `ReadIndex` API. `is_leader()` checks local identity/role; it is not a fresh quorum acknowledgment. Leadership expiration is a failure detector and is not, by itself, an audited linearizable-read protocol. Start with a command that serves as a read barrier: commit it, wait until the serving state machine has applied through it, then capture a consistent state view. The barrier must not advance the client-visible KV revision. A later optimization can implement a quorum-confirmed read-index protocol with current-term, membership, apply-position and concurrency rules, or a separately proved lease-read protocol with explicit timing assumptions. [Local leader check, `raft_server.hxx:659`](https://github.com/ClickHouse/NuRaft/blob/e9375226c8851be59c6a7a15b4322f66d3eef5e6/include/libnuraft/raft_server.hxx#L659), [leadership expiration](https://github.com/ClickHouse/NuRaft/blob/e9375226c8851be59c6a7a15b4322f66d3eef5e6/docs/leadership_expiration.md#L24).

etcd already obtains a confirmed index and waits for application (`server/etcdserver/read/read.go:117–139 @ v3.7.3`). A logged barrier is a simple compatibility baseline, but adds log/storage work to reads; its performance must not be mistaken for the eventual optimized design. [etcd read coordination](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/read/read.go#L117).

**Third, defer parallel log appending until the durable-index contract is tested. High confidence in the contract; Moderate in the sequencing recommendation.** The feature is marked experimental and permits commitment on a quorum that excludes the leader while its own disk write is pending. The adapter must report the actual durable prefix and notify completion correctly. The default `log_store::last_durable_index()` returns `next_slot()-1`; inheriting it with asynchronous persistence would report appended data as durable. `raft_params.hxx:732–761`, `log_store.hxx:224–231`, and `docs/parallel_log_appending.md:37–39 @ e9375226c885`. [Parallel appending contract](https://github.com/ClickHouse/NuRaft/blob/e9375226c8851be59c6a7a15b4322f66d3eef5e6/include/libnuraft/raft_params.hxx#L732), [default durable index](https://github.com/ClickHouse/NuRaft/blob/e9375226c8851be59c6a7a15b4322f66d3eef5e6/include/libnuraft/log_store.hxx#L224).

Keep ordinary majority quorum rules for the baseline. Custom quorum and early-acknowledgment features are not free performance switches. Keeper's similarly named configuration options must not be mapped to NuRaft flags by name alone.

### 2.2 RocksDB: suitable primitives, not etcd MVCC

**Verified, High confidence:** RocksDB `v11.13.2` provides atomic `WriteBatch`, byte-ordered storage, stable read snapshots, WAL controls and openable checkpoints. These are sufficient building blocks for a candidate implementation. They do not directly implement etcd's revisions or cluster consistency. Evidence: `include/rocksdb/write_batch.h:9`; `db.h:1566`; `options.h:2687–2712`; `utilities/checkpoint.h:141–169 @ v11.13.2`. [Atomic batches](https://github.com/meta/rocksdb/blob/v11.13.2/include/rocksdb/write_batch.h#L9), [read snapshots](https://github.com/meta/rocksdb/blob/v11.13.2/include/rocksdb/db.h#L1566), [durability options](https://github.com/meta/rocksdb/blob/v11.13.2/include/rocksdb/options.h#L2687), [checkpoints](https://github.com/meta/rocksdb/blob/v11.13.2/include/rocksdb/utilities/checkpoint.h#L141).

**High confidence:** Keep these three numbers separate:

- **Raft index:** position in the replicated command log, including configuration and barrier commands.
- **etcd revision:** application-visible version of the KV store, grouping the mutations of an etcd transaction.
- **RocksDB sequence number:** engine-local write ordering, including maintenance/metadata updates and batching choices.

A RocksDB snapshot handle is a local read view, not a durable public historical-revision service or a Raft snapshot. Historical reads and replay after restart require explicitly retained application versions. RocksDB transactions likewise do not replace etcd's compare/branch/response rules. With one ordered apply stream, an atomic batch plus a carefully implemented transaction view is a plausible starting point; `TransactionDB` is not required merely because the public API calls an operation `Txn`. **Design assessment: Moderate confidence.**

**High confidence in the storage distinction:** MVCC compaction removes application history; RocksDB compaction reorganizes engine files; Raft log compaction discards commands recoverable from durable state. An old key version may still be the latest live value. Deleting all records below a revision or letting RocksDB TTL delete attached keys locally would violate the intended model. The earlier [storage analysis, §§3.2–3.3](storage-codex.md#32-the-on-disk-key-order-is-revision-order), supplies the pinned etcd evidence.

**Recommendation — Moderate confidence:** Pin a tested NuRaft/RocksDB/compiler combination, rather than continuously following branch tips. NuRaft's production use elsewhere is useful evidence about the library, but does not transfer correctness to a new log adapter, snapshot protocol, MVCC implementation or Rust bridge.

## 3. A concrete architecture and its proof obligations

### 3.1 Begin with one ordered semantic engine

```text
etcd gRPC clients / optional native clients
                  |
          API and admission layer
                  |
     leader proposal / linearizable read barrier
                  |
      NuRaft quorum + durable log adapter
                  |
      ordered etcd semantic state machine
        |                         |
 atomic state batch        committed event dispatch
        |                         |
 RocksDB versions, metadata   watch queues / replay
 and applied Raft index

Snapshot publication and recovery connect the durable log
to a complete, versioned image of the applied state.
```

**Proposal — Moderate confidence:** Keep a transport-independent command representation and a small reference state machine. Validate requests, authenticate, and enforce admission limits at the frontend, but evaluate state-dependent predicates and authoritative permission/lease decisions in a well-defined order with committed state. Never let replicas independently decide transaction branches from local clocks, disk free space, or differently advanced read views.

Start with a single Raft group. Sharding would change global revisions, multi-key transactions and watch ordering and require a separate distributed transaction design. NuRaft and RocksDB do not eliminate that additional problem.

### 3.2 Durable log and applied state

**Proposal — Moderate confidence:** For the first correctness prototype, use RocksDB with WAL enabled for both the NuRaft log adapter and the applied state. Two RocksDB instances, one for Raft records/metadata and one for application state, make their lifecycles and snapshot contents explicit. A single instance with separated namespaces is a valid alternative, but shares compaction/write-stall resources and requires careful exclusion of node-local Raft state from transferable snapshots. Neither design is established as faster here.

For the conservative, non-parallel-appending baseline:

1. Persist term/vote changes before the Raft messages that depend on them. Persist log entries before treating the relevant append as durable. Implement suffix overwrite/truncation atomically enough that recovery cannot retain a contradictory tail.
2. Apply only commands known committed. Do not expose speculative `pre_commit()` changes through reads or watches.
3. Atomically write each applied transition's KV/history changes, lease/auth metadata, global revision and **applied index**, the Raft position represented by that database state.
4. Publish the resulting read view and watch events after successful application. An I/O failure must not become a normal success or an inconsistent applied-index advance.
5. Initially synchronize state batches at a clear boundary as well. Later, allow replayable unsynchronized application only after establishing a retained durable log/checkpoint prefix and testing every reclamation path.
6. Reopen from a valid state image, read its applied index, and obtain commitment information through the defined Raft recovery path before replay. Never execute every locally stored entry merely because it survived; an uncommitted tail may be replaced.

**High confidence in the obligations:** The recovered state and applied index must describe the same complete transition prefix. An index that survives without its mutations loses acknowledged data; mutations replayed twice can advance revisions and emit incorrect events. Data-directory deletion, snapshot installation, membership restoration and log reclamation must preserve this relationship, not just ordinary writes. The prior [WAL analysis, §§2 and 9](wal-replacement-codex.md), explains these invariants in etcd's existing design.

**High confidence:** A RocksDB write is not necessarily durable by default: `WriteOptions.sync` defaults to false. With WAL disabled, writes can be lost even across a process crash. Atomic multi-column-family flush is another distinct feature; its option documentation explicitly discusses WAL-disabled writes (`options.h:1605–1617 @ v11.13.2`). Avoid enabling WAL removal, asynchronous durability and multiple independent flush domains simultaneously. [Flush contract](https://github.com/meta/rocksdb/blob/v11.13.2/include/rocksdb/options.h#L1605).

Using RocksDB for both stores does not make replicated commands and their applied representation one physical write. There are two logical transitions separated by consensus, and usually two representations of the payload. Eliminating that duplication is the journal/checkpoint research problem described in the previous WAL analysis, not an automatic consequence of this rewrite.

### 3.3 MVCC layout and transactions

**Proposal — Moderate confidence:** Evaluate a representation with current-key metadata, retained versions ordered by user key/revision, and a revision-ordered event index. Use one atomic application batch across the related records. A revision-first representation closer to etcd is also viable and easier to compare initially. The layout decision trades range-read locality, history replay, memory, disk duplication and compaction complexity; it needs workload evidence.

Unlike a backend-only swap, the rewrite can remove etcd's existing in-memory index architecture. This is an opportunity to evaluate, not a promise of lower memory or faster reads. An additional persistent index can trade memory consumption for extra writes and read I/O.

The semantic test matrix must cover at least: nonexistent-key comparisons; binary range boundaries; sorting before/after limits and filters; version reset on recreation; empty deletes; failed transaction branches that themselves write; nested transaction validation; read-your-writes where etcd exposes it; atomic revision grouping; previous KV values; lease reassignment; and reads at, before and after the compaction boundary. Request-level validation matters too: etcd rejects prohibited duplicate writes (`server/etcdserver/api/v3rpc/key.go:275–310 @ v3.7.3`). Its transaction read view can advance when the transaction has changes (`server/storage/mvcc/kvstore_txn.go:189–194 @ v3.7.3`). **High confidence in these source examples.** [Validation](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/api/v3rpc/key.go#L275), [transaction view](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore_txn.go#L189).

### 3.4 Watches, leases and coordination

**Assessment — Moderate confidence:** These should be separate workstreams with explicit shared invariants, not a final convenience layer after Put/Get works.

For watches, join historical replay to live delivery without gaps or duplicates, including across snapshot install. Define a cursor at a complete revision boundary and ensure progress never overtakes undelivered matching events. Isolate slow streams from the apply path using bounded queues and retained-history replay; if history is unavailable, return the appropriate compaction/cancellation outcome. Test stream cancellation, fragmentation, re-registration and independent watchers on the same connection. Add a stated recovery-progress objective under restored connectivity and adequate resources. A finite test can check that objective, not prove eventual delivery under arbitrary schedules. **Proposal: Moderate confidence; inherited watch obligations: High confidence**, with details and source citations in [watchers-codex.md](watchers-codex.md).

For leases, choose and document how accepted renewals constrain expiration across leader changes. Options include conservative failover extension with correctly managed local renewal state, or replicated renewal information with batching and an explicit timing model. Logging renewals can simplify some authority questions but adds consensus traffic and does not itself solve clock, pause or expiration-race problems. Local timers must propose authoritative revocations; replicas must not independently expire RocksDB records. Recheck expiration eligibility against concurrent renewal and leadership changes. Grant, attach, revoke and cleanup must preserve transaction/event semantics. **Design assessment: Moderate confidence.** The earlier [lease analysis](leases-codex.md) supplies the failure cases to turn into regression tests.

For locks and elections, support the service APIs and test interaction with Go concurrency helpers. Ownership keys and revisions are externally visible and can be used in transactions; they cannot be replaced by an unrelated in-memory mutex. A lease-bound lock does not stop a paused former holder from affecting an external service. External effects still require a fencing/checking protocol at that service. **High confidence in the contract and limitation.** `server/etcdserver/api/v3lock/v3lockpb/v3lock.proto:14–18 @ v3.7.3` explicitly describes guarding etcd transactions with the ownership key. [Lock contract](https://github.com/etcd-io/etcd/blob/v3.7.3/server/etcdserver/api/v3lock/v3lockpb/v3lock.proto#L14).

### 3.5 Snapshot, membership and operational integration

**Proposal — Moderate confidence:** Snapshot publication needs a manifest tying together the exact applied Raft index/term, committed configuration, logical revision, compaction boundary and complete application state. Capture a matching state/configuration boundary, write and synchronize the complete image, publish it atomically, and durably synchronize the publication metadata and containing directory as required by the filesystem protocol before permitting log reclamation. Validate received images in a staging generation; installation must recover to either the old or new complete generation after a crash. Preserve the receiving node's identity and term/vote rules rather than overwriting them with the donor's local metadata.

NuRaft's logical snapshot callbacks and RocksDB checkpoints can implement parts of this. The application still owns consistency of the captured boundary, checksums, retry/cleanup, interrupted installation, format versions, and ordering against log deletion. A checkpoint obtained concurrently with application cannot simply be labeled with whichever Raft index was convenient before or after the call.

Membership is not just forwarding `MemberAdd`. NuRaft uses `int32` server IDs (`state_mgr.hxx:88 @ e9375226c885`), while etcd exposes 64-bit member IDs (`api/etcdserverpb/rpc.proto:984 @ v3.7.3`). Preserve a durable mapping, learner status, advertised client/peer addresses and correct configuration-transition ordering. Qualify removal of the leader, failed promotion, snapshot catch-up and restart during membership changes. **High confidence in the ID mismatch; Moderate confidence in adapter design.** [etcd member ID](https://github.com/etcd-io/etcd/blob/v3.7.3/api/etcdserverpb/rpc.proto#L980).

Operational behavior needs explicit choices. A RocksDB implementation can offer useful defragmentation/space-reclamation semantics, but should not claim that a successful no-op freed space. Logical `HashKV` can be implemented independently of file layout; physical hashes and database-size meanings may differ. Alarms, quota admission, health/readiness, metrics, diagnostics, certificate rotation and upgrade compatibility require their own acceptance tests. Any divergence in `Snapshot` or `Downgrade` must be advertised rather than hidden behind a successful response. **Assessment: Moderate confidence.**

## 4. C++, Rust and coding agents

### 4.1 C++ throughout

**Assessment — Moderate confidence:** C++ is the lower-integration-risk starting point for a team already competent in modern C++ storage and concurrency code. NuRaft's virtual interfaces, callbacks and ownership types and RocksDB's native API can be used directly; a C++ gRPC frontend avoids a second language boundary inside the server.

The cost is responsibility for memory ownership and concurrency across gRPC, NuRaft/Asio, application workers, RocksDB and shutdown. Use a small number of explicit owners and queues, RAII for lifetimes, bounded work admission and sanitizer builds. Define which callbacks may block or reenter other components. A C++ implementation does not become unsafe merely because agents write code, but fast code generation makes reviewable ownership rules more valuable.

### 4.2 Rust above a deliberately narrow C++ bridge

**Assessment — Moderate confidence:** Rust is equally credible when the team is stronger in Rust or intends to put most new semantic and orchestration code there. It does not remove the C++ libraries' failure modes or make their callbacks safe automatically. There must still be a C++ adapter implementing NuRaft's interfaces and a binding to the selected RocksDB version.

A reasonable split is Rust for request handling, deterministic command evaluation, watch/lease orchestration and validation, with a C++ facade for NuRaft lifecycle/persistence callbacks and RocksDB operations. Alternatively, keep consensus and the state machine together in C++ and put only the frontend in Rust; that reduces semantic crossings but gives Rust less ownership of the critical new code. Choose one boundary deliberately.

Bridge design requirements:

- Pass batches or owned command/result buffers, not an FFI call per field or key/value comparison. **FFI**, the foreign-function interface, is the boundary where both languages must agree on representation and ownership.
- Specify buffer ownership after callbacks, iterator/snapshot lifetimes, thread affinity, permitted concurrent calls and cancellation. NuRaft explicitly says the caller owns buffers passed to `commit()` (`state_machine.hxx:59–65 @ e9375226c885`).
- Preserve application ordering. If a NuRaft commit callback queues work to Rust and returns before it finishes, the integration must not falsely advance application or complete the client result. A synchronous callback may instead wait on a dedicated apply worker, with a reviewed deadlock/shutdown protocol.
- Convert errors at the boundary. Do not let exceptions/panics cross uncontrolled or turn RocksDB errors into apparent success. Avoid blocking synchronous storage work on an async runtime's general event-loop workers.
- Pin native libraries and bridge/binding revisions together. Existing Rust RocksDB bindings are candidates to inspect, not evidence that every `v11.13.2` facility or callback is already supported.

**Verified, High confidence:** CXX, one possible Rust/C++ bridge, documents explicit exception-to-`Result` behavior; unhandled exceptions on non-`Result` bridge calls terminate, and panics in exported Rust functions abort. It does not automatically convert RocksDB's `Status` into the desired domain errors. [CXX error-handling contract](https://cxx.rs/binding/result.html). A small safe-facing wrapper can reduce the exposed unsafe surface; its implementation and C++ dependencies still need review.

**Recommendation — Moderate confidence:** Choose C++ if native integration speed is the decisive criterion and the team has the required expertise. Choose Rust if Rust expertise and ownership of the new semantic layer outweigh bridge work. Do not claim a performance ranking or an agent-productivity advantage for either language without a representative vertical slice. Both choices need the same distributed-system specification and external tests.

### 4.3 Use agents to increase verified throughput

**Planning judgment — Moderate confidence:** Agents should expand the amount of independently checked implementation work, not determine the semantics from whatever makes the current tests pass.

Good assignments include service adapters, request validation tables, error mappings, fixture conversion, declarative regression cases, randomized operation generators, build packaging and diagnostics. Higher-risk assignments include the durable applied-prefix protocol, snapshot publication/install, expiry authority, read barriers and retry/deduplication. Those require a written design and independent review before implementation is treated as established.

Recommended workflow:

1. Maintain a versioned compatibility specification and a requirement-to-test inventory. Give each implementation task explicit invariants and permitted differences.
2. Keep the reference model and test oracle independent of the optimized implementation. Translating the same incorrect algorithm into two languages is not independent validation.
3. Give agents bounded modules and explicit ownership. Integrate small changes through differential tests, fault injection, sanitizer checks and review.
4. For every defect, retain the minimized failing history, seed, binary/library pins and recovery image. Add regression coverage before accepting the fix.
5. Audit test changes as carefully as production changes: weakened assertions, silently skipped faults and misclassified ambiguous results can create false confidence.
6. Keep human owners for consensus/storage invariants, client semantics, test infrastructure and production operation. Require a reviewer who can explain the failure schedule, not merely read the patch.

There is no project-specific measured speedup from agent use. The estimates in §7 already assume extensive assistance and do not apply an additional optimistic discount. As an arithmetic illustration only, if 40% of total effort were accelerated fourfold and the remainder were unchanged, total effort would fall to 70%, not 25%. Neither input is a forecast; the example shows why code-generation speed alone cannot price the project.

## 5. Reusing tests: substantial value, explicit adaptation

### 5.1 Keep test languages independent of server language

**Recommendation — High confidence in feasibility; Moderate in effort saved:** Keep Go tests in Go, Java tests in Java and Jepsen tests in Clojure wherever possible. A complete C++/Rust server rewrite does not require rewriting its external tests. Preserving independent client implementations and test languages helps expose accidental assumptions in the new server.

| Existing asset | Reuse with gRPC compatibility | Necessary adaptation / coverage limit | Relative effort judgment |
|---|---|---|---|
| etcd `tests/common` | Reuse many KV/Txn/Watch/Lease/Auth/admin cases | Implement a process-backed runner/cluster/client fixture; some methods return concrete `clientv3` types and some tests use runner-specific features | Low–medium |
| etcd `tests/e2e` and `etcdctl` | Reuse external requests, command behavior and selected lifecycle cases | Flags, readiness log text, paths, metrics, snapshots and upgrade assumptions are etcd-specific | Medium |
| etcd `tests/integration`, Go client/concurrency tests | Reuse behavioral test bodies and expectations | Many create embedded Go servers or inspect internals; extract external-server cases or replace fixtures | Medium–high |
| etcd `server/storage/mvcc` tests | Reuse revision/compaction/restore scenarios and model expectations | Direct Go implementation tests cannot exercise a C++ store unchanged; translate cases into black-box sequences or a semantic-engine harness | Medium–high |
| etcd `tests/robustness` | Reuse operation generation, histories, models, Porcupine linearizability checks, watch validators and issue regressions | Replace cluster lifecycle, failpoints, transport fault controls and persisted-request extraction | High, especially recovery evidence |
| Jepsen etcd harness | Reuse clients/workloads/checkers for registers, transactions, watches and coordination | Change installation/startup, member management and file corruption selection; pin the actual client/version; qualify each nemesis | Medium–high |
| jetcd tests | Reuse real client behavior against a compatible endpoint | Replace launcher/cluster fixtures where needed; mocks do not test the server; control the server image/version | Medium |
| Kubernetes storage tests and API-server integration | Reuse application-level semantics, paging, TTL, watches, progress and resource versions | Replace embedded-etcd fixtures; then exercise actual API-server binaries against replacement endpoints | Medium–high |
| NuRaft tests | Reuse election, failure, membership, snapshot and network-scheduling cases | Run the new durable adapters and state machine; tests using in-memory stores do not validate production persistence | Medium |
| RocksDB tests/crash tools | Reuse engine stress, filesystem fault injection and diagnostic techniques | Configure the selected engine options and add application invariants; engine consistency is not etcd consistency | Medium |
| Xline compatibility scripts | Additional `etcdctl` scenarios and compatibility inventory | Hardcoded deployment assumptions, uneven assertions and historical documentation; review and update rather than importing a certification claim | Low–medium |

The effort column is an engineering judgment (**Moderate confidence**), not a measured porting result. The existence and coupling of the named assets were inspected (**High confidence**); representative source anchors follow.

### 5.2 Start with `tests/common`, not a port of the entire etcd repository

**Verified, High confidence:** `tests/framework/interfaces/interface.go:25–86 @ v3.7.3` defines `TestRunner`, `Cluster`, `Member` and `Client`. `tests/common/main_test.go:24` holds the selected runner; the e2e and integration build-tag files initialize different implementations. These are useful seams for a replacement fixture. They are not a fully server-neutral conformance standard: response types include `clientv3` objects, and the e2e matrix includes mixed etcd versions. [Interfaces](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/framework/interfaces/interface.go#L25), [e2e selection](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/common/e2e_test.go#L31).

Create an adapter that starts one or more replacement processes, returns endpoints and uses the unmodified Go client. Retain test assertions. Separate genuinely inapplicable implementation tests from semantic failures, and report the exclusion list. Do not spend time imitating etcd's exact startup log string merely because the old fixture waits for it: adapt the fixture's readiness check. That dependency is explicit at `tests/framework/e2e/etcd_process.go:41,257 @ v3.7.3`. [Readiness dependency](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/framework/e2e/etcd_process.go#L41).

**Verified, High confidence:** MVCC tests include revision, Put/Range/Delete, compaction, restore and unfinished-compaction recovery scenarios (`server/storage/mvcc/kvstore_test.go:48,61,183,256,332,374,488 @ v3.7.3`). They are valuable specifications to extract, but importing their Go package tests does not test the replacement engine. [MVCC cases](https://github.com/etcd-io/etcd/blob/v3.7.3/server/storage/mvcc/kvstore_test.go#L48).

Upstream [#22339](https://github.com/etcd-io/etcd/pull/22339) remained open and unmerged when checked; its head was `c80ad786f30a4d7a57ec726780b9470a9863e8d9`. Treat it as an additional test-design source, not shipped validation of another backend.

### 5.3 Preserve the robustness suite's evidence, not just its green result

**Verified, High confidence:** The robustness suite's log reader imports etcd WAL and Raft protobuf packages and reconstructs requests from member data directories (`tests/robustness/report/wal.go:32–39,64–96,368–399 @ v3.7.3`). A NuRaft/RocksDB directory cannot be read by that decoder. [Persisted-request reconstruction](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/report/wal.go#L64).

**A particularly important trap — High confidence:** `ValidateAndReturnVisualize` performs linearizability checking, but if persisted requests are empty it returns before watch and historical-read validation (`tests/robustness/validate/validate.go:41–59 @ v3.7.3`). Dropping WAL extraction to make the harness run can silently remove major validation. [Validator control flow](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/validate/validate.go#L41).

Adapt this in stages:

1. Preserve client histories and run black-box linearizability checks with honest unknown outcomes. A timed-out mutation may have executed; do not mark it definitely aborted.
2. Define a durable, versioned extraction format for replicated commands, indexes/terms and the information needed to reconstruct the accepted history. Implement NuRaft/RocksDB extraction separately from the normal read/watch paths.
3. Reconcile divergent member tails and commitment uncertainty correctly. A record on one disk is not evidence that its effect belongs in the final committed history. Handle truncated logs and snapshot boundaries explicitly; arrange test retention or archive sufficient evidence.
4. Re-enable the independent replay-based watch and historical-read checks. Make missing evidence an explicit incomplete result, not a pass.
5. Translate Go failpoints into equivalent C++/Rust injection points at persist, acknowledgment, apply, snapshot publication/install and reclamation boundaries. Preserve the ordering being tested, not merely the old failpoint name.

**High confidence in the coverage limitation:** The model is intentionally smaller than the full API. `tests/robustness/model/types.go:43–125 @ v3.7.3` represents a selected KV/transaction/lease/compaction surface; lease grant contains an ID but no TTL/renewal timing state. Passing it cannot establish lease renewal promises, complete Auth behavior, every transaction form or application-level lock safety. [Model types](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/model/types.go#L43).

Reuse issue-specific cases from the robustness README as regression seeds. They include compaction/revision recovery, missing watch events, false progress and process-pause stale reads. Preserve the behavior being checked while replacing implementation-specific triggers. **High confidence in the inventory**, `tests/robustness/README.md:19–39 @ v3.7.3`. [Regression history](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/README.md#L19).

### 5.4 Jepsen: test the server and the actual client separately

The 2020 report on etcd `3.4.3` found no violations of the tested KV and watch properties while identifying lock-related problems. It is historical evidence about that version, workload and fault campaign, not a certificate for `v3.7.3` or a rewrite. Its pause/partition/membership/clock cases and lock lessons remain useful test inputs. **High confidence in the report's scope.** [Jepsen etcd 3.4.3](https://jepsen.io/analyses/etcd-3.4.3).

The 2024 jetcd `0.8.2` report traced transactional anomalies to retries of operations whose first execution was uncertain. It establishes why a server test using a client library is a test of their combination. The same server can produce different observed histories with different retry policies. **High confidence in the historical finding.** [Jepsen jetcd 0.8.2](https://jepsen.io/analyses/jetcd-0.8.2).

**Do not carry the old retry verdict forward unchanged. Verified, High confidence:** jetcd commit `05f92e04432849d4d091efc49397554df9682c6f`, dated 2025-02-18 and contained in tags `0.8.5` through the evaluated `0.8.7`, changed default non-idempotent retry behavior. At `jetcd-0.8.7`, `TxnOption` defaults `autoRetry=false`; `KVImpl.txn()` selects the non-safe-redo predicate, which does not retry generic `UNAVAILABLE`. Authentication failures still have specific retry rules. This is source verification of the fix, not a newly executed Jepsen validation of `0.8.7`. [Fix](https://github.com/etcd-io/jetcd/commit/05f92e04432849d4d091efc49397554df9682c6f), [`TxnOption.java:31`](https://github.com/etcd-io/jetcd/blob/jetcd-0.8.7/jetcd-core/src/main/java/io/etcd/jetcd/options/TxnOption.java#L31), [`KVImpl.java:129`](https://github.com/etcd-io/jetcd/blob/jetcd-0.8.7/jetcd-core/src/main/java/io/etcd/jetcd/impl/KVImpl.java#L129), [`Errors.java:29`](https://github.com/etcd-io/jetcd/blob/jetcd-0.8.7/jetcd-core/src/main/java/io/etcd/jetcd/support/Errors.java#L29).

The cloned Jepsen harness has useful concrete features and limits:

- **High confidence:** `project.clj:13 @ b47cc401f27b` pins jetcd **0.8.5**, not 0.8.2 or 0.8.7. Change and record that dependency deliberately when evaluating current clients. `src/jepsen/etcd/client.clj:212–231` wires client selection and retry configuration. [Dependency pin](https://github.com/jepsen-io/etcd/blob/b47cc401f27bfb7c28afd9ac088b5f0dca4501b8/project.clj#L13), [client construction](https://github.com/jepsen-io/etcd/blob/b47cc401f27bfb7c28afd9ac088b5f0dca4501b8/src/jepsen/etcd/client.clj#L212).
- **High confidence:** Workloads include register/set/append/watch and coordination examples. Nemeses, the fault injectors, include pause, kill, clock, partitions, membership and WAL/snapshot corruption; LazyFS support can discard unsynchronized writes. The process installer and corruption selectors are etcd-specific. [Harness README:33–80](https://github.com/jepsen-io/etcd/blob/b47cc401f27bfb7c28afd9ac088b5f0dca4501b8/README.md#L33), [`db.clj:72,199`](https://github.com/jepsen-io/etcd/blob/b47cc401f27bfb7c28afd9ac088b5f0dca4501b8/src/jepsen/etcd/db.clj#L72), [`nemesis.clj:146`](https://github.com/jepsen-io/etcd/blob/b47cc401f27bfb7c28afd9ac088b5f0dca4501b8/src/jepsen/etcd/nemesis.clj#L146).
- **High confidence:** The harness explicitly excludes some lock workloads from its expected-to-pass set (`src/jepsen/etcd.clj:54–56`). Those cases can demonstrate unsafe application use; requiring them all to pass unchanged would misstate the server's contract. Preserve them as negative examples and add correctly fenced/transactionally guarded variants. [Expected-workload selection](https://github.com/jepsen-io/etcd/blob/b47cc401f27bfb7c28afd9ac088b5f0dca4501b8/src/jepsen/etcd.clj#L54).

Run two complementary tracks: low-level requests with controlled retries to investigate server semantics, and actual supported clients with their intended defaults to qualify end-to-end behavior. Add explicit lost-response-after-commit tests to both. Replacing gRPC requires a new Jepsen client adapter or a sufficiently complete compatible Java library, in addition to process/fault changes.

**High confidence in the protocol limitation:** The existing Txn request carries predicates and operation branches but no mandatory stable client operation identifier (`rpc.proto:663–675 @ v3.7.3`). A server cannot generally distinguish a retry from a deliberate second identical request by hashing its payload. A native protocol can add request IDs and replicated result deduplication, but must also define session identity, retention, garbage collection and behavior after the deduplication window. That is a useful optional redesign, not a transparent server-side cure for arbitrary old-client retries.

### 5.5 Kubernetes and additional external test sources

**Verified, High confidence:** Kubernetes `v1.37.2` has reusable storage tests for creation with TTL, deletion conflicts, list paging and watches. `staging/src/k8s.io/apiserver/pkg/storage/etcd3/store_test.go:112–147,248` and `watcher_test.go:60–104` call the common storage test helpers. The fixture starts embedded etcd and explicitly uses `UnsafeNoFsync=true` for a disposable single-member test (`.../etcd3/testserver/test_server.go:52–62,88–98`). Replace that fixture; do not treat its configuration as a durability test. [Store tests](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/etcd3/store_test.go#L112), [watch tests](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/etcd3/watcher_test.go#L60), [fixture](https://github.com/kubernetes/kubernetes/blob/v1.37.2/staging/src/k8s.io/apiserver/pkg/storage/etcd3/testserver/test_server.go#L52).

Then test an unmodified API server against the replacement: list/watch handoff, continuation tokens and retained revisions, optimistic updates, deletion/recreation, TTL cleanup, watch-cache progress and rebuild, API-server restart, member failure, compaction and backup restore. Add representative controllers and load. Kubernetes conformance is useful application evidence but does not replace direct durability or all-etcd-API tests. **Recommendation: Moderate confidence.**

**Verified, High confidence:** jetcd's launcher still defaults to `quay.io/coreos/etcd:v3.5.14` (`jetcd-launcher/src/main/java/io/etcd/jetcd/launcher/Etcd.java:31 @ jetcd-0.8.7`). Adapt the fixture or image and print the actual server version in test artifacts. Otherwise a successful jetcd test run may have exercised the old stock server rather than the replacement. [Launcher pin](https://github.com/etcd-io/jetcd/blob/jetcd-0.8.7/jetcd-launcher/src/main/java/io/etcd/jetcd/launcher/Etcd.java#L31).

Xline, an existing etcd-API alternative, provides additional `etcdctl` validation scripts. The inspected `scripts/validation_test.sh` contains KV, compaction, watch, lease and auth scenarios. Its `VALIDATION_REPORT.md` also distinguishes snapshot saving from incompatible status/restore, but contains historical feature-status entries; do not infer present coverage from its “passed” rows. Use reviewed cases as supplementary smoke tests. **High confidence in inspected artifacts; Moderate confidence in reuse value.** [Script @ `71479a4fabdd`](https://github.com/xline-kv/Xline/blob/71479a4fabdd12fe67eec0bee95e652976937541/scripts/validation_test.sh#L75), [report:42](https://github.com/xline-kv/Xline/blob/71479a4fabdd12fe67eec0bee95e652976937541/VALIDATION_REPORT.md#L42).

### 5.6 New tests that the libraries and existing suites cannot supply

**Recommendation — Moderate confidence:** Build the following layers, with separately reported outcomes. Existing tests should seed them; new harness code is justified where no current adapter reaches the new persistence or language boundary.

| Layer | Required evidence |
|---|---|
| Deterministic semantic model | Generated operation sequences agree with a small model and, where appropriate, pinned etcd; all persisted revisions/metadata and event sequences are checked |
| Concurrent public API | Linearizability/transaction checks, cancellation, retries and lost replies; separate allowed stale reads from linearizable reads |
| Durable boundary tests | Crashes between append/sync/ack/apply/index update/checkpoint publish/log deletion; no acknowledged effects lost within the declared fault model; no uncommitted effects exposed |
| Elections and membership | Old-leader reads, paused leaders, failover during renewal/revoke, changing configurations, learners, interrupted transfer and snapshot catch-up |
| Streaming and liveness | Replay/live handoff, fragmentation, false progress, slow readers, reconnect after compaction, snapshot installation during watches and progress after faults heal |
| Native runtime and bridge | Buffer/iterator lifetime, use-after-close, callback reentry, shutdown with pending RPCs, panics/exceptions, I/O error propagation and resource leaks |
| Operations | Disk full, corrupted files, interrupted restore, format upgrade/rollback, TLS/auth changes, sustained overload and bounded resource use |

**Verified, High confidence:** NuRaft has fake-network controls for delayed/failed/reordered delivery (`tests/unit/fake_network.hxx:70–91 @ e9375226c885`) and snapshot/failure tests. Its example log store's `flush()` only updates an in-memory durable-index variable (`examples/in_memory_log_store.cxx:289`). Exercise the production adapter under the scheduler instead of assuming those tests cover real persistence. [Network controls](https://github.com/ClickHouse/NuRaft/blob/e9375226c8851be59c6a7a15b4322f66d3eef5e6/tests/unit/fake_network.hxx#L70), [example flush](https://github.com/ClickHouse/NuRaft/blob/e9375226c8851be59c6a7a15b4322f66d3eef5e6/examples/in_memory_log_store.cxx#L289).

RocksDB has `tools/db_crashtest.py`, `db_stress_tool/` and `utilities/fault_injection_fs.h @ v11.13.2`. Reuse their facilities and failure models, but add invariants about the combined log/state/snapshot protocol. RocksDB recovering a self-consistent database is necessary and insufficient for recovering the correct replicated prefix. **High confidence.** [Crash driver](https://github.com/meta/rocksdb/blob/v11.13.2/tools/db_crashtest.py), [filesystem injection](https://github.com/meta/rocksdb/blob/v11.13.2/utilities/fault_injection_fs.h).

**Do not substitute process killing for power-loss testing. High confidence.** Unsynchronized data can remain in the OS cache after a process exits. etcd's existing LazyFS kill path clears that cache, but its default scenario builder adds LazyFS only for selected low-load single-member cases and removes compaction (`tests/robustness/failpoint/kill.go:51`; `scenarios/scenarios.go:154–168 @ v3.7.3`). Extend the matrix to quorum failures, snapshots and compaction. [Kill integration](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/failpoint/kill.go#L51), [scenario restriction](https://github.com/etcd-io/etcd/blob/v3.7.3/tests/robustness/scenarios/scenarios.go#L154).

Use deterministic schedules and crash-point enumeration around the adapter boundaries, plus longer external fault campaigns. Model checking of a small snapshot/expiry protocol is worthwhile; passing it is evidence about that model, not a proof of the full C++/Rust implementation. Existing Antithesis configurations in etcd and the Jepsen harness are integration leads if that service is available; no service access or cost is assumed here.

Finally, test the testers. Deliberately inject an early success acknowledgment, a lost event, false progress, a stale read and an applied-index/data mismatch; require the corresponding checker to fail. Run the reference implementation through the adapted harness. Record tests skipped, failed injections, incomplete histories and checker timeouts separately from passes. Arbitrary corruption or loss of acknowledged data on a quorum is outside ordinary crash-fault Raft guarantees; distinguish expected fail-stop detection from promised recovery.

## 6. Performance rationale and comparison with narrower changes

**Assessment — Moderate confidence:** A complete rewrite has its strongest case when the desired improvement requires changing several layers together: storage layout, memory/index ownership, transaction execution, watch scheduling, lease management and native runtime integration. Its weakest case is an identified bbolt or WAL bottleneck that a narrower fork can address while preserving the existing semantic engine.

The previous [backend analysis](storage-codex.md) and [WAL/storage analysis](wal-replacement-codex.md) describe smaller interventions. They retain much more of etcd's implementation and test coverage. Their effort estimates are older judgment-based ranges for different scopes, not quotations, but the scope difference is real: a complete rewrite must replace MVCC, watch, lease, auth and API behavior that those alternatives retain. **High confidence in scope; Moderate in relative cost.**

| Potential opportunity | What must be demonstrated | Condition that weakens the case |
|---|---|---|
| Native runtime and memory control | Lower CPU/memory or better tail behavior for the actual request/stream mix | Quorum/storage latency dominates; C++ allocation or synchronization becomes the new bottleneck |
| RocksDB compression and storage organization | Better sustained space and write/read behavior with equivalent durability and retained history | Incompressible values, read amplification, cache misses, compaction stalls or excess index duplication |
| Revised MVCC indexes | Lower memory or better range/history locality | Extra indexes cost more than the removed in-memory structure |
| Grouped application and pipelining | Higher throughput with bounded queues and correct per-request responses/revisions | Larger batches increase tail latency or hide apply lag |
| Watch/lease redesign | Bounded resource use and better recovery without weakening client guarantees | Old clients cannot participate in the proposed protocol; mass expiry/replay remains dominant |
| New wire protocol | Measurable client/frontend savings or stronger request identity/session semantics | Semantics and persistence dominate; migration and client maintenance exceed the savings |

These are hypotheses, all **Moderate confidence as plausible opportunities**, not measured advantages. No claim is made that NuRaft is faster or more correct than etcd's Raft implementation, or that RocksDB is universally faster than bbolt. The rewrite would exchange one extensively exercised composition for another library composition plus new integration code.

**Proposed evaluation:** On the later benchmark hardware, compare stock pinned etcd, any justified narrower fork, and the replacement under the same acknowledged-durability contract. Record version/commit and compiler, cluster size, CPU/RAM, disks and filesystem, client/version, key/value distributions and compressibility, retained history, concurrency, batching, TLS, exact commands and all durability settings.

Measure sustained mixed read/write/transaction/watch/lease workloads, not only Put throughput. Include p50/p99/p99.9 latency, maximum sustainable load under the agreed latency objective, read latency during writes, watch lag, committed-to-applied backlog, CPU/RAM, write amplification, disk headroom and recovery time. Run through steady-state compaction, snapshot creation/transfer, member failure, reconnect storms and mass expiry. Compare the logged-read baseline separately from any later optimized read protocol. **Recommendation: Moderate confidence.**

Performance qualification is conditional on correctness qualification. A faster pre-quorum acknowledgment, stale read or unbounded queue is not an equivalent result. No benchmark from this analysis host is used or proposed as production evidence.

Conversely, the conservative prototype must not be mistaken for an optimized competitor. etcd already batches backend commits and recovers unapplied durable-log work; it does not require a fresh synchronous bbolt commit for every Put. A prototype that synchronizes both stores for every command may add persistence costs relative to that baseline. Compare a correctly qualified batched/replayable state path before judging the architecture's performance potential. **High confidence in the etcd baseline; Moderate confidence in the comparison recommendation.** [Previous source audit, storage §3.1](storage-codex.md#31-the-write-path-has-two-persistence-layers).

## 7. Cost, staffing and schedule

### 7.1 Planning assumptions

All numbers in this section are **Low-confidence planning estimates**, derived from the work breakdown rather than a completed implementation or measured coding-agent productivity. An engineer-month is one experienced engineer working full time for one month; it measures effort, not elapsed time.

The baseline assumes one Linux deployment family, one Raft group, C++ throughout or a team already capable of both Rust and C++, extensive coding-agent use, reuse of existing external tests, conventional majority Raft, and a planned maintenance window for first migration. It includes useful native backup/restore and a documented compatibility matrix. It excludes mixed stock-etcd/replacement clusters, arbitrary binary-compatible clients for every language, automatic zero-downtime bidirectional migration, byte-compatible bbolt snapshots, a new consensus algorithm, and a new storage engine. Those are additional projects.

The estimate also assumes access to representative hardware and engineers with distributed storage experience. If the team must learn consensus, RocksDB internals and native concurrency while building, these ranges are not a reliable ceiling.

### 7.2 Milestones, with cumulative effort

| Milestone | Cumulative effort | Illustrative elapsed time / team | Evidence delivered |
|---|---:|---|---|
| Demonstration | 1–3 engineer-months | 1–2 engineers, roughly 1–2 months | Compatible Put/Get/basic Txn on a small cluster; illustrative only, no production claim |
| Decision-quality prototype | 6–12 engineer-months | 2–3 engineers, roughly 3–6 months | Durable adapters; revisioned KV; linearizable reads; basic watch/lease behavior; restart/snapshot path; initial differential and fault tests; representative bottleneck evidence |
| Restricted production candidate | 30–60 engineer-months | 3–5 engineers, roughly 9–18 months | One explicitly bounded application/client profile; tested failure/recovery, backups, deployment tooling and pilot operation |
| Broad client-compatible production candidate | 80–160 engineer-months | 5–8 engineers, roughly 18–36 months | Broad v3 API/client coverage, operational tooling, upgrade/migration policy, substantial fault and sustained-load qualification, independent review and staged rollout |

**Forecast confidence: Low for every row and elapsed-time range.** The last row is a budget envelope for the specified scope, not a date on which correctness becomes guaranteed. Elapsed ranges are not obtained by dividing effort by the maximum headcount: review, discovering integration faults, migration rehearsals and operational observation constrain parallelism. A restricted candidate is only cheaper if the restrictions are real; calling all missing features “later” while depending on them in production does not reduce the work.

### 7.3 Broad candidate work breakdown

| Workstream | Engineer-months | Why it remains after choosing the libraries |
|---|---:|---|
| Contract, gRPC frontend and compatibility inventory | 8–14 | Observable behavior, errors, streaming, limits, current API surface and independent reference cases |
| NuRaft durability, reads and snapshot integration | 10–20 | Production log/state adapters, read authority, crash recovery and configuration transitions |
| MVCC, transactions, indexing and compaction | 12–24 | Entire versioned semantic engine and its recovery/retention rules |
| Watches, leases, locks/elections and client interaction | 14–28 | Cross-component timing, history and delivery behavior under failures |
| Authentication, membership/admin, resource management | 10–20 | Permissions, operations, admission/backpressure, diagnostics and maintenance meanings |
| Independent validation and harness adaptation | 18–36 | Differential/model tests, robustness/Jepsen ports, deterministic faults, client/Kubernetes qualification and failure analysis |
| Packaging, migration, release and pilot operation | 8–18 | Versioned formats, backup/restore, rollout/rollback, runbooks and operational debugging |
| **Total** | **80–160** | Includes the earlier milestones |

**Forecast confidence: Low.** Component-level tests and ordinary review are included in each implementation row; the validation row is additional cross-component qualification, not the whole testing budget. This is why test reuse should be funded explicitly rather than treated as a free final step. Severe library defects or a failed performance hypothesis can invalidate the schedule.

### 7.4 Effect of changing language or compatibility strategy

**Rust bridge allowance — Low-confidence planning estimate:** If the team lacks an established NuRaft/RocksDB bridge, reserve approximately **4–10 engineer-months** for the initial facade, ownership/error contracts, build integration and boundary testing. Some of this can fit within the baseline integration rows; it is not necessarily additive, and later Rust implementation savings may offset it. There is no defensible universal C++/Rust cost multiplier.

**Replacement-client strategy — Low-confidence planning estimate:** A deliberately small, controlled single-language API profile could fit near the restricted-product range. Broad Go and jetcd replacement libraries plus a new protocol could add roughly **12–30 engineer-months** of client/protocol/qualification work while removing only part of the frontend row. This estimate assumes application-facing compatibility, not transparent support for every raw gRPC escape hatch or Java binary/dependency configuration. Every additional language expands the continuing support matrix. Dropping semantics can save much more, but changes the product being estimated.

**Financial translation, an illustrative budget assumption:** At an assumed fully loaded engineering cost of **$20,000 per engineer-month**, 80–160 engineer-months corresponds arithmetically to **$1.6–3.2 million**, before hardware/CI, coding-agent usage, external review and contingency. This is not a verified labor-market rate or procurement quote; substitute the organization's own rate. The effort range is Low confidence, and no separate dollar estimate for infrastructure or agent use is justified from the available inputs.

**Continuing ownership — Low-confidence staffing allowance:** Budget at least **2–4 experienced engineers' worth of ongoing capacity** for a supported product with active users, dependency upgrades, compatibility drift, incidents and regression qualification. This is neither a 24×7 support roster nor a universal minimum; operating scope and incident experience can move it substantially. A rewrite avoids repeatedly merging all of upstream etcd, but still needs to track its API/client/security changes and the native dependencies.

## 8. Migration and the meaning of operational replacement

**High confidence:** A new client-compatible cluster is not a rolling replacement of existing etcd members. Use a separate target cluster and an explicit cutover protocol. Existing etcd backup tools understand bbolt data and etcd metadata (`etcdutl/snapshot/v3_snapshot.go:118,257 @ v3.7.3`); a RocksDB directory cannot be substituted. [Snapshot status/restore implementation](https://github.com/etcd-io/etcd/blob/v3.7.3/etcdutl/snapshot/v3_snapshot.go#L118).

Even `Snapshot` wire compatibility requires more than streaming arbitrary files: the Go save helper checks the expected checksum-bearing size convention (`client/v3/snapshot/v3_snapshot.go:33–38,102–103 @ v3.7.3`), and restore utilities interpret the data. **Verified, High confidence.** [Save-helper check](https://github.com/etcd-io/etcd/blob/v3.7.3/client/v3/snapshot/v3_snapshot.go#L102). Choose between a compatible logical export/conversion path and replacement operational tools. Supporting stock snapshot export may justify a separate Go conversion utility even if the server is entirely C++/Rust. The design must say which unmodified tools work and which require replacement. **Assessment: Moderate confidence.**

For the first deployment, prefer a maintenance-window migration (**Moderate-confidence recommendation**):

1. Quiesce writers and establish a consistent source boundary. Inventory keys, retained history, revisions, lease associations, auth state and application assumptions about cluster/member identity.
2. Import using a purpose-built format or snapshot reader that preserves the required metadata. Ordinary Put calls cannot set arbitrary creation/modification revisions and versions, so a key/value copy is not a general semantic migration.
3. Validate logical state, revisions, permissions and the chosen lease policy. Do not copy old lease deadlines as though clocks and elapsed downtime were irrelevant.
4. Repoint/restart clients and explicitly rebuild watches/caches or preserve sufficient history according to the migration contract. Test Kubernetes resource-version and restore behavior, including the revision-bump/compaction policy if used.
5. Keep a tested rollback procedure. Once writes occur only on the replacement, rolling back to an old source snapshot loses those writes unless a verified reverse migration or equivalent recovery path exists.

A watch-fed online migration is a separate design. KV watches do not by themselves transfer the complete lease/auth/membership state; compaction can remove needed history; and naive dual writes do not atomically preserve conditional transactions across two independent consensus groups. Do not assume a proxy or duplicated RPCs solve that problem. **High confidence in these limitations; Moderate confidence in migration effort.**

**Failure condition for the recommended initial scope:** If zero-downtime migration, stock backup compatibility or unrestricted existing administrative tooling is mandatory, resolve that before implementation. The baseline estimate explicitly does not include all of those guarantees.

## 9. Recommended sequence and go/no-go criteria

**Recommendation — Moderate confidence:** Fund a bounded prototype first, preserving gRPC and ordinary majority Raft. Treat a full rewrite as a product investment conditional on the prototype, not as the assumed solution to the earlier lease/storage concerns.

| Gate | Deliverable | Stop or redesign if… |
|---|---|---|
| 0. Product boundary | Supported etcd/client/Kubernetes versions, required APIs, permitted differences, migration constraints and performance objectives | The team cannot state what “compatible” means or who owns production support |
| 1. Minimal semantic core | NuRaft/RocksDB vertical slice with deterministic revisions, Txn and a safe read barrier; independent differential runner | Basic behavior requires unbounded exceptions to the compatibility contract |
| 2. Durability and snapshots | Written persistence protocol; actual adapters; repeatable crash/overwrite/install/reclamation tests | Success can precede required durability, or recovered index/state can disagree |
| 3. Coordination and streaming | Lease authority, watch replay/progress/backpressure, cancellation and client-failure cases | Fixes require semantic changes that supported clients cannot tolerate |
| 4. Performance decision | Representative sustained mixed-load results against stock etcd and the most relevant narrower alternative | Gains disappear with equivalent durability, stable compaction, fault recovery or required read latency |
| 5. Compatibility and operation | Explicit test inventory, robustness and Jepsen results, supported client/API-server qualification, migration/restore and upgrade rehearsals | Evidence is dominated by skipped tests, unknown histories, unexercised failpoints or missing recovery paths |
| 6. Staged adoption | Isolated pilot, operational observation and a rehearsed rollback decision | The team cannot diagnose failures or bound backlog/disk/memory during normal disturbances |

No fixed number of green test runs proves correctness. Before a pilot, require no known unresolved violations of the chosen safety contract, demonstrably effective fault/checker coverage, repeatable recovery drills and performance within the agreed objectives. Long campaigns should report seeds, durations, workloads, cluster/storage configuration, actual injected faults, exclusions and checker results. An incomplete or inconclusive run is not a pass.

### Open decisions retained for the project

- Required deployment target: Kubernetes specifically, a controlled application fleet, or a general etcd alternative; exact client libraries and versions.
- Required API/admin coverage, including RangeStream, gateway behavior, Auth, Lock/Election, snapshot export and downgrade/upgrade expectations.
- Performance objective and representative hardware/workload. Benchmark hardware is still an open project input.
- Required retained-history window, watch recovery/progress expectations, lease timing behavior and external fencing responsibilities.
- Migration downtime, preservation of historical revisions/leases, restore semantics and acceptable rollback limitations.
- Team expertise and staffing for C++, Rust, storage/consensus, testing and ongoing operation.
- Any additional known lease defects or performance problems from the earlier workstream; none were assumed supplied by this request.

These are decision inputs, not requests to block this analysis. The architectural and budget recommendations are conditional on the stated assumptions rather than silently resolving them.

## 10. Verification record and limitations

The source repositories were read without editing tracked files or moving their checkouts. etcd and jetcd tags were fetched as required by the project instructions and checked against GitHub latest-release metadata; jetcd was also checked against Maven Central. Both stable pins remained current on 2026-11-08. No source worktree was created, and no commit, push or publication was performed.

| Source | Verified pin / treatment |
|---|---|
| etcd | `v3.7.3` / `0a173d0410a96d01091b484daae81365bbcde57e`; API, semantic implementation, clients and test infrastructure |
| jetcd | `jetcd-0.8.7` / `23b61f1499a5d340ae2f267d88d45ca3900425a4`; API/retry code, tests and launcher |
| ClickHouse/NuRaft | Default branch is `master`, not `main`; local and upstream tip matched `e9375226c8851be59c6a7a15b4322f66d3eef5e6` |
| RocksDB | Local target is `v11.13.2` / `17b624b5c0d41bd42db5cf88aa867f3b26eaf1fa`, as clarified by the user |
| Supplemental RocksDB check | Upstream `main` was `e591f5f1ab40640a62ee5b4bd1f4e00c15c21423`; selected API/test files were downloaded to scratch for comparison before the version clarification. It is not the target or a claimed validated upgrade |
| Jepsen etcd harness | `b47cc401f27bfb7c28afd9ac088b5f0dca4501b8`; local source, including jetcd 0.8.5 dependency |
| Kubernetes test examples | `v1.37.2` / `c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32`; selected remote source files, not a claim about the latest Kubernetes release |
| Xline test examples | `71479a4fabdd12fe67eec0bee95e652976937541`; selected remote validation script/report, not a production evaluation of Xline |

Supplemental files were kept under `/tmp/ftcd-rewrite-20261108/`. The durable evidence is the pinned citations in this document, not continued availability of that scratch directory. Prior analyses were used for project context; their historical experiments and confidence qualifications remain their own. This report adds source inspection and design synthesis, not new reproductions of their bugs.

**Limits:** This is not a full NuRaft correctness audit, a complete RocksDB option audit, an inventory of every etcd test, or an executed interoperability qualification. Negative API-search findings are limited to the pinned code inspected. Effort ranges are judgments; there is no measured project velocity, workload baseline or agent productivity study. Reused source/tests also retain their own license and dependency requirements; their presence does not make them all interchangeable artifacts.

## Conclusions

**Proceed to a bounded gRPC-compatible prototype if the organization is considering a maintained database product. Confidence: Moderate.** NuRaft and RocksDB are credible building blocks, but they leave most etcd-specific semantics and the critical integration proofs to the new implementation. The project is feasible in engineering terms; production readiness is not inherited from either library.

**Preserve the client wire protocol initially. Confidence: Moderate.** It retains existing clients and the strongest test-reuse path while leaving internal transport and storage free to change. Replacement language libraries are justified by a controlled client fleet or deliberate new semantics, not by an expectation that abandoning protobuf removes most server complexity.

**Budget the validation and operational work as part of the implementation. Forecast confidence: Low.** A decision-quality prototype is plausibly 6–12 engineer-months; a restricted production candidate 30–60; a broadly client-compatible candidate 80–160, under the explicit assumptions above. Coding agents can help execute that work, but no evidence supports reducing it to a short assembly exercise.

**The decisive comparison is the rewrite versus the smallest change that meets the actual objective. Confidence: Moderate.** If a backend/WAL change meets the workload and operational requirements, retaining etcd's semantic engine is the less extensive intervention. A full rewrite becomes compelling only when measured benefit or required architectural change justifies replacing that engine and owning its long-term compatibility and failure behavior.
