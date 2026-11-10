# An etcd-compatible rewrite on NuRaft and RocksDB (C++ or Rust): feasibility, cost, test reuse, and wire vs. library compatibility

- **Method:** static analysis of local source clones plus desk research (GitHub API, web pages). Nothing was built, run or benchmarked. Four research passes (sub-agents) inventoried NuRaft, etcd's test trees, the Jepsen suite and analyses, and prior art. I re-checked their load-bearing claims against code or primary pages; the list is in §13.1.
- **Author:** Claude (Claude Code), 2026-11-08. Written independently. `analysis/nuraft-rocksdb-rewrite-codex.md` appeared concurrently and was not read.
- **Versions analyzed:**
  - etcd `v3.7.3` = `0a173d0410a9`. On 2026-11-08 there was no newer stable tag.
  - jetcd `jetcd-0.8.7` = `23b61f1499a5`.
  - NuRaft, ClickHouse fork (`github.com/ClickHouse/NuRaft`), `master` = `e937522` (2026-11-06). Compared through the GitHub API with eBay/NuRaft `master` `46c73ad37c` (2026-10-12).
  - RocksDB tag `v11.13.2` = `17b624b5c` (2026-10-28). The clone is detached at this release tag, not at `main`. The repository is now `meta/rocksdb`.
  - Jepsen etcd suite, `jepsen-io/etcd` `master` = `b47cc40` (2026-04-17).
  - Kubernetes `master` = `dbe731cf4b02` (2026-11-09 UTC). Selected files were read through the GitHub API; they are cited where used.
- **Citation forms:** `path:line @ v3.7.3` (etcd), `@ jetcd-0.8.7`, `@ e937522` (NuRaft), `@ v11.13.2` (RocksDB), `@ b47cc40` (Jepsen suite), `@ k8s dbe731cf4b02` (Kubernetes).
- **Upstream references:**
  - Jepsen analyses: [etcd 3.4.3](https://jepsen.io/analyses/etcd-3.4.3) (2020-01-30) and [jetcd 0.8.2](https://jepsen.io/analyses/jetcd-0.8.2) (2024-08-08).
  - etcd: [#22332](https://github.com/etcd-io/etcd/issues/22332) (alternative storage engines, "mostly vibed" prototypes); the 17 correctness issues tabulated in `tests/robustness/README.md:21-39 @ v3.7.3`; [#11456](https://github.com/etcd-io/etcd/issues/11456) and [#11457](https://github.com/etcd-io/etcd/issues/11457) (locks); [#14140](https://github.com/etcd-io/etcd/issues/14140) and [#14143](https://github.com/etcd-io/etcd/issues/14143) (lost un-fsynced writes, never resolved); [#18424](https://github.com/etcd-io/etcd/issues/18424) (client retry safety).
  - jetcd: [#1444](https://github.com/etcd-io/jetcd/issues/1444), fixed by commit `05f92e0` (first released in jetcd-0.8.5).
  - NuRaft (eBay): [#185](https://github.com/eBay/NuRaft/issues/185) (rollback of committed entries); [PR #668](https://github.com/eBay/NuRaft/pull/668) (RequestVote freshness after full log compaction, merged 2026-10-08, not in the fork); [#651](https://github.com/eBay/NuRaft/pull/651), [#660](https://github.com/eBay/NuRaft/pull/660), [#662](https://github.com/eBay/NuRaft/pull/662), [#653](https://github.com/eBay/NuRaft/issues/653), [#553](https://github.com/eBay/NuRaft/issues/553), [#463](https://github.com/eBay/NuRaft/issues/463), [#288](https://github.com/eBay/NuRaft/issues/288), [#475](https://github.com/eBay/NuRaft/issues/475), [#654](https://github.com/eBay/NuRaft/issues/654).
  - ClickHouse: [PR #18491](https://github.com/ClickHouse/ClickHouse/pull/18491) (NuRaft added, 2020-12-26); [PR #56626](https://github.com/ClickHouse/ClickHouse/pull/56626) (RocksDB Keeper storage, 2024); [PR #108000](https://github.com/ClickHouse/ClickHouse/pull/108000) (its removal, 2026-06-26); durability issues [#111339](https://github.com/ClickHouse/ClickHouse/issues/111339), [#111454](https://github.com/ClickHouse/ClickHouse/issues/111454), [#112101](https://github.com/ClickHouse/ClickHouse/issues/112101), [#112535](https://github.com/ClickHouse/ClickHouse/issues/112535).
  - Kubernetes: [PR #140085](https://github.com/kubernetes/kubernetes/pull/140085) (`EtcdRangeStream` on by default, milestone v1.37, merged 2026-07-14).
  - Xline: [#467](https://github.com/xline-kv/Xline/issues/467) (Jepsen run without faults).
- **Builds on these project files:**
  - `storage-claude.md`: replacing bbolt; prior art in its §8.
  - `wal-replacement-claude.md`: a synced log with an unsynced state store.
  - `leases-claude.md`: the term-less revocation flaw.
  - `watchers-claude.md`: watch liveness.
  - `etcd-issues-and-mitigations-claude.md`.

## Scope

Should one build an etcd replacement from scratch, in C++ or Rust, on NuRaft (consensus) and RocksDB (storage), with coding agents doing most of the writing? This file covers:
- what such a project costs and how complex it is;
- how much existing test infrastructure can validate it, from the etcd repository, Jepsen and elsewhere;
- which compatibility contract to adopt. **(A)** is gRPC wire compatibility of the client API, with the server-to-server protocol free. **(B)** is language-level client API compatibility only, with replacement client libraries.

Out of scope: benchmarks (this host is not the benchmark environment, `AGENTS.md` §7) and a detailed design.

## Summary

**Verdict.**
- The project is feasible. The cost lies in matching etcd's behavior under faults, not in writing the code.
- Choose wire compatibility (A). Language-level compatibility (B) costs more and buys almost nothing that A cannot also provide.
- Rebuild only if the goals need something a fork of etcd cannot give. A storage-engine fork of etcd is estimated at about 6–12 engineer-months, against 15–60 for the rewrite (§10.4).

| # | Claim | Confidence | § |
|---|---|---|---|
| 1 | **Feasible.** ClickHouse Keeper is a NuRaft-based, wire-compatible ZooKeeper replacement in production, built in C++. Kine and others reimplement the Kubernetes slice of the etcd API. | High | 9 |
| 2 | **Writing the code is not the bottleneck.** etcd's server is 48,173 lines of non-test, non-generated Go. About 33k of them is the semantic core a rewrite must reproduce; the raft and bbolt libraries live in other repositories. Agents can produce code of that size quickly. Validation is the bottleneck. etcd's robustness suite lists 17 correctness bugs, most latent since v3.4 or earlier, 8 of them in watch. ClickHouse Keeper was still finding acknowledged-write-loss durability bugs in 2026, five years after it started. | High (sizes, history); Moderate (that validation dominates) | 2, 9, 10 |
| 3 | **Choose A.** B does not free the data model: the Go and Java client APIs are defined over etcd's protobuf messages. The Go client API exposes gRPC types, and Kubernetes passes gRPC dial options (metrics interceptors, tracing, custom dialer) through it. B needs a new client library per language, gives up the ecosystem, and loses most unmodified test reuse. Anything B could change can be added under A as optional new gRPC services or fields. | High | 6 |
| 4 | **Decide early: Kubernetes backend or full etcd replacement.** The Kubernetes slice is about 5 Txn shapes, Range and its variants (now including `RangeStream`), Watch with progress, `Lease.Grant`, Compact, Status and MemberList. Full etcd adds roughly 40 more of its 49 RPCs (auth, election and lock among them), the HTTP/JSON gateway, and ops tooling. Xline aimed at the full API with about 26 contributors and stalled after about 2.5 years. | High (the slice); Moderate (scope ratio) | 2, 9 |
| 5 | **NuRaft is usable, with five gaps**, each with a workaround: (a) no ReadIndex; (b) int32 member IDs, no endpoint update, no removal of the current leader, and `add_srv` requires a running node; (c) the ClickHouse fork lacks several upstream safety fixes, including PR #668 (vote freshness); (d) the log-durability contract is undocumented, and its failure mode (#185) is rollback of committed entries; (e) no deterministic simulation. Only (a) shapes performance. | High (gaps exist, verified in code); Moderate (workarounds) | 4 |
| 6 | **RocksDB fits the state machine well.** Atomic multi-column-family WriteBatches can carry the applied index. Disabling the state store's WAL leaves the Raft log as the only fsync on the write path. Checkpoints serve snapshots. User-defined timestamps implement etcd's compaction rule exactly, but should not be used in v1: transaction limits, open 2026 bugs, and no public production MVCC user. Watch needs a revision-ordered table anyway. The read-path regression the upstream Pebble prototype measured is the main performance risk. | Moderate | 5 |
| 7 | **Language:** Rust for the service and semantic layers (about 80% of the code), with a thin C++ shim around NuRaft. All-C++ becomes the better choice if the team will patch NuRaft heavily. If NuRaft is negotiable, raft-rs (a Rust port of etcd's raft) closes gaps (a) and (b) by construction. | Moderate–Low | 7 |
| 8 | **Large parts of the existing tests carry over unmodified under A**, given a thin `etcd`-flag-compatible launcher (a "shim"): `tests/common` (≈93 of 95 cases), 86 + 42 e2e functions, the robustness suite with about 1k lines of adapter, Antithesis configs, jetcd's tests (`ETCD_IMAGE`), 63% of the Jepsen suite, Kubernetes integration and conformance tests, and `tools/benchmark`. The highest-leverage new test is **differential testing against real etcd v3.7.3**. | High (mechanisms); Moderate (adapter sizes) | 8 |
| 9 | **Cost**, to a production candidate, before ≥6 months of shadow running: Kubernetes slice under A ≈ 7–14 months for 2–3 senior engineers with agents (≈15–35 engineer-months). Full API under A ≈ 10–20 months for 3–4 engineers (≈30–60). B costs more than full A. Ongoing upkeep ≈ 1.5–3 FTE. | Low | 10 |
| 10 | **A rewrite under A can fix most of the etcd design flaws found in earlier analyses**, because they are internal. Examples: term-fenced lease revocation via NuRaft's `expected_term_`, a TTL upper bound, no bbolt commit stalls, no defrag. It cannot remove what the API mandates: one totally ordered revision sequence (one Raft group, so a write ceiling of one leader's log), unsafe lease-based locks, and bugs in unmodified clients. A mixed etcd/replacement cluster is impossible, so migration needs a cutover that preserves revisions. | High (constraints); Moderate (fixes) | 3.5, 11 |

---

## Terms

- **A / B:** the two compatibility contracts in Scope.
- **Shim:** a launcher named `etcd` that accepts etcd's flags and `ETCD_*` environment variables, prints the log line and `--version` format that harnesses wait for, and starts the replacement. Ideally the replacement binary does this natively.
- **Semantic core:** etcd server code whose observable behavior must be reproduced: MVCC, apply, txn, watch, lease, auth, membership logic and the gRPC service layer. It excludes code that NuRaft and RocksDB replace (rafthttp, wal, snap, backend) and legacy code (v2store, proxy).
- **ReadIndex:** Raft's mechanism for linearizable reads without writing to the log. The leader confirms with a quorum that it is still leader, then serves the read once it has applied up to the commit index it recorded.
- **Revision:** etcd's cluster-wide, monotonically increasing counter. Each write transaction increments it once. Kubernetes `resourceVersion` is the object's `mod_revision`.
- **Differential testing:** sending identical request sequences to the replacement and to real etcd v3.7.3, then comparing responses, revisions and watch streams.

---

## 1. Premise check

Four premises in the request deserve comment before the analysis.

1. **"Two distinct approaches" (A vs B) are less distinct than they look.**
   - B still means keeping etcd's protobuf data model. Go's `clientv3` response types are defined from the protobuf messages: `type GetResponse pb.RangeResponse`, and likewise Put, Delete, Txn and Compact (`client/v3/kv.go:28-35 @ v3.7.3`). jetcd's public `GetResponse` has a public constructor that takes the generated `io.etcd.jetcd.api.RangeResponse` (`jetcd-core/src/main/java/io/etcd/jetcd/kv/GetResponse.java:27,33 @ jetcd-0.8.7`).
   - B also does not remove gRPC from the Go API. `Config.DialOptions []grpc.DialOption` (`client/v3/config.go:85`), `Dial` returns `*grpc.ClientConn` (`client.go:277`), and so does `ActiveConnection` (`client.go:566`).
   - So B frees only the transport and framing, and costs nearly all the ecosystem (§6). *(High.)*
2. **"The server-to-server API is up for grabs" has a cost the request does not mention.**
   - A different peer protocol means no mixed cluster of etcd members and replacement members.
   - Rolling migration is therefore impossible. Migration becomes a cutover, and for Kubernetes it must preserve revisions (§3.6). *(High.)*
3. **NuRaft is a reasonable but not neutral choice.**
   - NuRaft is a different Raft implementation from etcd's (`go.etcd.io/raft`), with a different feature set. It lacks ReadIndex. Its membership model does not match etcd's (§4).
   - If the language ends up being Rust, raft-rs is a port of etcd's raft to Rust (*recalled, not checked here*). It brings ReadIndex, learners and 64-bit IDs as a closer semantic match.
   - I treat NuRaft as a given below, but its gaps are priced in.
4. **"Heavy use of coding agents" shortens implementation more than validation.**
   - The sourced evidence (§9.4) shows large agent-built systems succeeding when there is an external oracle: GCC for a C compiler, byte-identical output for the Ladybird port, a ported upstream test suite for vinext.
   - etcd provides an unusually good oracle (§8). Even so, the hardening tail of a consensus store is dominated by fault-injection runs and diagnosis. Agents speed those up less. *(Moderate.)*

---

## 2. What "etcd-compatible" has to cover

### 2.1 The surface, tiered

The v3.7.3 API has 49 RPCs:
- `api/etcdserverpb/rpc.proto`: 42 RPCs in 6 services (KV 6, Watch 1, Lease 5, Cluster 5, Maintenance 8, Auth 17);
- `v3lock.proto`: 2;
- `v3election.proto`: 5.

41 of the 42 core RPCs have HTTP/JSON gateway mappings; the exception is the new `RangeStream` (`rpc.proto:44-47`, added in `84e612f39`, 2026-04-24). *(High; counted.)*

| Tier | What | Needed by | Notes |
|---|---|---|---|
| **T0: data plane** | Range (all options), RangeStream, Put, DeleteRange, Txn (including nested txns), Compact; Watch (prev_kv, filters, progress notify, RequestProgress, fragment, compaction cancel); Lease Grant/Revoke/KeepAlive/TimeToLive/Leases; Status; MemberList; `grpc.health.v1` | Every client; Kubernetes uses a subset (§2.2) | Revision, version and create/mod revision semantics must match exactly. Limits: 128 txn ops, 1.5 MiB requests (`server/embed/config.go:64,67`). |
| **T0: "soft" wire contract** | Error **message strings**; `hasleader` metadata; ResponseHeader fields; the version string | Go client, jetcd, Jepsen, Kubernetes | See §2.3 |
| **T1: operations** | MemberAdd/Remove/Update/Promote, MoveLeader, Defragment, Alarm (NOSPACE quota), HashKV, Snapshot, Downgrade | Operators, etcdctl, backup tools, robustness tests | Snapshot stream format in §2.3. Downgrade can be refused. |
| **T2: rest of the API** | Auth (17 RPCs, simple and JWT tokens); Lock and Election (7 RPCs; in etcd these run the Go `concurrency` package inside the server, `server/etcdserver/api/v3lock/lock.go:36-46`); the HTTP/JSON gateway (`server/embed/serve.go:357-364`) | Non-Kubernetes users, curl tests | The gateway uses grpc-gateway v2 with a custom marshaler |
| **T3: process and operations contract** | Command-line flags and `ETCD_*` variables; `/metrics` names, `/health`, `/readyz`, `/livez`; the log line `ready to serve client requests`; `--version` output | Test harnesses (§8), dashboards and alerts | Cheap to provide, and it multiplies test reuse |
| **Not on the wire** | bbolt snapshot and data-dir format (`etcdutl`), WAL format, data-dir layout | etcdutl, robustness persisted-request reader, Jepsen corruption nemesis, e2e C5 tests | A rewrite cannot keep these; adapters are listed in §8 |

### 2.2 The Kubernetes slice

kube-apiserver at Kubernetes `master` (`staging/src/k8s.io/apiserver/pkg/storage/etcd3/`, built against etcd client v3.7.3) uses:
- Range: single key, prefix with limit and revision, `CountOnly`, `KeysOnly`.
- `KV.GetStream` → **RangeStream**, falling back when the server returns `Unimplemented` (`store.go:924-932 @ k8s dbe731cf4b02`).
- Three Txn shapes on `ModRevision` (create, update, delete), plus the compaction txn on `Version(compact_rev_key)`, then `Compact`.
- Watch with `WithRev`, `WithPrevKV`, `WithPrefix` and `WithProgressNotify`; `RequestProgress`.
- `Lease.Grant` only (leases are reused across objects).
- `Maintenance.Status`, a health Get, and `WithRequireLeader`.

The research pass compiled this list from Kubernetes source; I re-checked the RangeStream fallback and the feature check below. *(High.)*

The independent mem_etcd project placed a logging proxy under a full Kubernetes conformance run and observed the same shapes. Kine's README lists essentially this subset as its whole scope. *(High; §9.)*

The slice also moves: `EtcdRangeStream` went on by default in the Kubernetes v1.37 milestone ([#140085](https://github.com/kubernetes/kubernetes/pull/140085), merged 2026-07-14). That is less than three months after the RPC appeared in etcd. *(High.)*

### 2.3 Obligations hidden in the clients

These were verified at v3.7.3 unless marked otherwise.

- **Errors are matched by message text, not by code.** `rpctypes.Error` looks up the gRPC status message in `errStringToError`, a 57-entry table, and takes the code from the table rather than from the wire (`api/v3rpc/rpctypes/error.go:256-279`). A server that returns the right code with a different message breaks:
  - retry decisions;
  - token refresh;
  - lease keep-alive teardown;
  - Jepsen's definite/indefinite classification (`src/jepsen/etcd/client.clj:288-389 @ b47cc40`).
- **Retry safety rests on the server's choice of code.**
  - The Go client retries immutable RPCs only on `Unavailable` (`client/v3/retry.go:54-62`).
  - It retries mutable RPCs only when no connection was established (`:85-94`).
  - etcd uses `Unavailable` for outcomes the server cannot know: no leader, leader changed, timed out (`error.go:77-86`).
  - So a replacement must map NuRaft's `TIMEOUT` and `CANCELLED` to these indefinite codes. NuRaft's `TIMEOUT` does not mean "not committed" (§4.4). Mapping them to a definite code would reproduce, from the server side, the G1a anomalies Jepsen found in jetcd 0.8.2 (§8.5).
- **Metadata.**
  - `hasleader=true`: set by `WithRequireLeader` and on by default for keep-alive streams. The server must reject calls when it has no leader, and cancel streams after three election timeouts without one (`server/etcdserver/api/v3rpc/interceptor.go:38,68-72,315-345`).
  - `token`: carries the auth token.
- **Version string.**
  - Kubernetes enables `RequestWatchProgress` only if `Status.Version` parses as ≥3.4.31 or ≥3.5.13 (`staging/src/k8s.io/apiserver/pkg/storage/feature/feature_support_checker.go:37-39,193-198 @ k8s dbe731cf4b02`).
  - The Go client's `RejectOldCluster` rejects versions below the previous minor (`client/v3/client.go:505-560`).
  - The e2e harness gates behavior on `--version`.
  - So the replacement must report an etcd version. That version is a claim about semantics; reporting it is a deliberate compatibility decision, not a detail.
- **Snapshot stream.**
  - `etcdctl snapshot save` checks only that the size is ≡ 32 (mod 512). It neither parses bbolt nor recomputes the digest (`client/v3/snapshot/v3_snapshot.go:35-38,102-104`).
  - etcd appends a sha256 digest (`server/etcdserver/api/v3rpc/maintenance.go:156,205-209`).
  - So `save` works against any stream of N×512 bytes plus a 32-byte digest.
  - `etcdutl snapshot restore` and `status` open the result as bbolt, so they work only on the real format.
- **Header fields.** Jepsen targets leaders using `Status.leader` and the highest `ResponseHeader.raft_term` (`db.clj:38-61 @ b47cc40`). `WaitLeader` in the e2e framework needs consistent `MemberId` and `Leader` values (`tests/framework/e2e/cluster.go:1120-1160`).

### 2.4 Sizes of what is reimplemented

| Component (etcd v3.7.3, non-test, non-generated Go) | Lines | Replacement |
|---|---|---|
| `server/` total | 48,173 | — |
| ...of which NuRaft/RocksDB replace rafthttp, wal, snap, backend | 8,462 | NuRaft integration, log store, RocksDB layer |
| ...of which legacy or optional: v2store, v2stats, v2error, proxy, mock | 6,744 | Not needed |
| **Semantic core (remainder, approximate)** | **≈33,000** | Must be reimplemented |
| `client/v3` (with `concurrency`: 1,063) | 11,279 | Reused under A; rewritten under B |
| `etcdctl` | 6,329 | Reused under A |
| `etcdutl` (offline bbolt and WAL tooling) | 1,744 | Rewritten in all cases |
| jetcd `jetcd-core/src/main` (Java) | 10,383 | Reused under A; rewritten under B |
| Generated protobuf, gRPC and gateway code | 16,999 | Regenerated from the same `.proto` files |

Sources: research-pass count of `server/` subdirectories, which I re-counted (total 48,173); client counts are mine (`wc -l`). Raft (`go.etcd.io/raft/v3`) and bbolt are separate modules, not counted. For scale: ClickHouse Keeper's `src/Coordination` is 38,682 non-test lines of C++ today, and NuRaft itself is 17,211 (`src/`) + 9,503 (`include/`) lines. *(High.)*

---

## 3. Architecture of the replacement and its key decisions

### 3.1 Component map

| etcd component | Replacement | Notes |
|---|---|---|
| `go.etcd.io/raft` + `rafthttp` + WAL + snap | NuRaft (`raft_server`, asio transport), a log store, `state_mgr`, a snapshot adapter | §4 |
| bbolt backend + schema | RocksDB, one state DB with column families | §5 |
| MVCC (`treeIndex` + `key` bucket), watchable store | Ported semantics on RocksDB; in-memory key index in v1 (§3.3) | Highest-risk area (§8.6) |
| Apply loop (`apply`, `txn`) | NuRaft `commit_ext` on its single commit thread | Must be deterministic and fast; blocking stalls every apply (§4.3) |
| Lessor | Ported, with term-fenced revocation (§3.5) | Expiry stays leader-local |
| Auth store | Ported | T2 |
| Membership | Replicated membership table in the state machine, mapped onto NuRaft `srv_config` (§3.4) | |
| `v3rpc` services + interceptors + gateway | tonic (Rust) or gRPC C++ callback API; a gateway (§7) | |
| Follower → leader proposal path | A replacement peer service, or NuRaft forwarding (§3.4) | |
| `embed`/`etcdmain` configuration | Native etcd flag and `ETCD_*` compatibility for the subset harnesses use (§8.1) | |
| `etcdutl` | A new offline tool; bbolt↔replacement converter written in Go, reusing the bbolt library (§3.6) | |

### 3.2 Durability: one fsync stream

Design: synced NuRaft log store; RocksDB state DB with its WAL **disabled**; applied index written in the same WriteBatch as the state. This is the same split as `wal-replacement-claude.md`.
- NuRaft initializes its commit cursor from `state_machine::last_commit_index()` (`src/raft_server.cxx:61-62 @ e937522`). It re-commits from there to the commit index (`src/handle_commit.cxx:226-246`).
- So unflushed state lost in a crash is rebuilt by replay, provided the log still holds those entries.
- NuRaft compacts the log only after `create_snapshot`. If a snapshot is a RocksDB checkpoint, which flushes memtables, compaction never discards entries the state DB still needs. *(Moderate: mechanism read in code, not tested.)*
- The same rule holds on a follower that installs a snapshot from the leader: the installed state must be durable before NuRaft compacts the follower's log.
- `last_commit_index()` must report the applied index as recovered from the flushed state, never an in-memory value.
- Multiple column families must flush together: `atomic_flush` (`include/rocksdb/options.h:1617 @ v11.13.2`).
- The result: on the client write path the only fsync is the Raft log. etcd's bbolt commit costs two `fdatasync` calls, holds locks, and is forced on every lease revocation (`storage-claude.md` §2.3). That cost leaves the apply path. *(Moderate.)*

**The log store is where durability bugs concentrate.**
- NuRaft's headers do not state when `append`/`write_at` must be durable. NuRaft itself calls `flush()` only for config entries (`src/raft_server.cxx:2243-2252`). A follower acknowledges right after `end_of_append_batch` (`src/handle_append_entries.cxx:1242-1244`).
- So `append` plus `end_of_append_batch` must fsync before returning. The alternative is `parallel_log_appending_` with `last_durable_index` (`include/libnuraft/raft_params.hxx:732-761`), the mode ClickHouse Keeper uses.
- Getting this wrong produces NuRaft #185: committed entries rolled back.
- ClickHouse's hand-written changelog still had missing directory fsyncs, non-atomic term/vote rewrites and segment-unlink races in 2026 (§9.1).

**Choosing where the log lives.**
- **Start with a separate, synced RocksDB instance** (`WriteOptions::sync`, `options.h:2704`). RocksDB's own crash testing covers its WAL.
- **Move to a custom segmented log only if measurement on the target hardware justifies it.** TiKV moved its logs out of RocksDB into Raft Engine, reporting 25–40% less write I/O (§9.3). Note the trade: a custom log is where Keeper's 2026 durability bugs were.
- *(Moderate–Low: I weigh correctness first; the research pass recommended the custom log first.)*

### 3.3 MVCC layout on RocksDB

etcd keys its main bucket by revision: 8-byte main revision, `_`, 8-byte sub-revision. It keeps all keys and their revision lists in an in-memory B-tree (`treeIndex`), rebuilt at startup by scanning everything (`storage-claude.md` §2.2).

| Option | Idea | For | Against |
|---|---|---|---|
| **L1 (recommended for v1)** | CF `rev`: revision → marshaled `mvccpb.KeyValue`, byte-identical to etcd's encoding. In-memory key index as in etcd. Other CFs: meta (applied index, compaction marks), lease, auth, members, alarm | Faithful port, the easiest to diff against etcd. Watch catch-up is a sequential scan of `rev`. HashKV can match etcd's values exactly (next paragraph) | Keys must fit in RAM; startup scans; one point lookup per returned revision (the read regression in §5.3) |
| L2 | L1, plus the key index persisted in a CF `idx` (user key + revision suffix) | Lifts the RAM limit, starts fast | More writes per op; every Range seeks in an LSM |
| L3 | User-defined timestamps (UDT): user key with revision as timestamp | RocksDB's GC rule matches etcd compaction exactly (§5.2) | Watch still needs `rev` (by-revision order); UDT limits and open bugs (§5.2) |

Mirroring etcd's encoding has a payoff. etcd's HashKV is a CRC over the backend's revision keys and values. If `rev` stores the same bytes, a single-node replacement can produce **the same HashKV as etcd for the same history**. That turns the whole keyspace into a cheap differential check (§8.2). *(Moderate: depends on reproducing the hash's exact iteration and compaction-boundary rules.)*

Compaction:
- Port etcd's algorithm: index compaction yields the revisions to keep, and the rest are deleted in batches.
- In RocksDB these deletes are cheap tombstones. They do not cause the forced bbolt commits behind the 47% write drop during compaction measured in #22332.
- The cost moves to background LSM compaction, which competes with the log's fsyncs (§5.3).

### 3.4 Consensus integration decisions

| Decision | Recommended | Why | Confidence |
|---|---|---|---|
| **Linearizable reads** | Phase 1: a read-barrier log entry per batch of reads, batched like etcd's `LinearizableReadLoop` (`server/etcdserver/read/read.go:96`). Phase 2: patch ReadIndex into the NuRaft fork. Do not rely on `leadership_expiry_` as a lease. | NuRaft has no ReadIndex (grep found none). Its lease mode can overlap leaders at defaults: expiry 2,500 ms vs election timeout 250–500 ms (`raft_params.hxx:531-540,69-72`; `docs/leadership_expiration.md:40-44`). The real vote ignores the heartbeat-alive check (`src/handle_vote.cxx:334-343`). A barrier costs a log append and fsync per batch, but is plainly correct. Kubernetes serves most reads from its watch cache; a consistent list still costs one linearizable read. | High (gap); Moderate (plan) |
| **Member IDs** | Replicated table of etcd uint64 ID ↔ NuRaft int32 ID, plus peer URLs, in the state machine | `srv_config::id_` is `int32`, "should be positive" (`include/libnuraft/srv_config.hxx:93-96`) | High |
| **MemberAdd before start** | Record a pending member in the state machine; the leader calls `add_srv` when the node is up | `add_srv` actively catches up a *running* node and times out otherwise (`src/handle_join_leave.cxx:95-135`). etcd adds first, starts later, and Jepsen's membership nemesis relies on that order (`db.clj:146-157`) | High |
| **MemberUpdate (peer URLs)** | Endpoint indirection: a custom `rpc_client_factory` resolves member ID to its current URL from the replicated table | NuRaft rejects endpoint changes (`src/raft_server.cxx:2146-2152`) | Moderate |
| **Remove the leader** | Transfer leadership first (`yield_leadership`), then remove | NuRaft returns `CANNOT_REMOVE_LEADER` (`src/handle_join_leave.cxx:497-502`). Yield is not synchronized with membership changes (#463) | Moderate |
| **MemberPromote** | Check learner readiness at the application level via `get_peer_info()`, as etcd does | `flip_learner_flag` has no catch-up check. Upstream #553 says a learner can become leader | Moderate |
| **Proposal routing** | A replacement peer service: a follower forwards `InternalRaftRequest` to the leader, then replies only after its own applied index reaches the entry | NuRaft `auto_forwarding_` is marked "not recommended" (`src/handle_user_cmd.cxx:230`) and its pool can reorder requests. Waiting for local apply keeps etcd's read-your-writes behavior for serializable reads on the same member | Moderate |
| **Log payload** | etcd's `InternalRaftRequest` protobuf (`api/etcdserverpb/raft_internal.proto`) | Lets the robustness suite's persisted-request decoder work unchanged once a log reader is supplied (§8.3) | High |
| **Fork baseline** | Pin `e937522`; cherry-pick upstream #668, #651, #660 and #662; fix the #653 iterator pattern; budget an ongoing merge | §4.5 | High |

### 3.5 Leases: what can be fixed internally

`leases-claude.md` located etcd's lease flaw in two places:
- revocations decided by one member's timer but committed as Raft entries that carry no term;
- authority measured in ticks while expiry runs in real time.

NuRaft offers a direct fence. `append_entries_ext` with `req_ext_params::expected_term_` rejects the proposal with `TERM_MISMATCH` if the term has changed (`include/libnuraft/raft_server.hxx:337-341`; `src/handle_client_request.cxx:143-153 @ e937522`). `commit_ext` hands the state machine each entry's `log_term`, so the apply step can also drop a revocation whose decision term differs from its entry term.

This fix is invisible on the wire, so it fits under A. Also compatible:
- Persisting the remaining TTL, which etcd only does behind the Alpha checkpoint gates. That bounds TTL across failovers (P2 in `leases-claude.md`).
- Reproducing etcd's minimum TTL rule, ⌈1.5 × election ticks × heartbeat⌉ (`server/etcdserver/server.go:340`), from NuRaft's timeouts.

*(Moderate: hooks verified in code; the design is mine and untested.)*

Some lease problems survive any server:
- Lease-based locks are not mutexes (Jepsen #11457).
- Clients compute optimistic local deadlines.
- Go #22081/#22094 and jetcd `onError` keep "lost" leases alive.

These are client-side or inherent to the API. *(High.)*

### 3.6 Migration from etcd

- **No mixed clusters.** The peer protocols differ, so a rolling member-by-member migration is impossible.
- **Revisions must be preserved.** Kubernetes `resourceVersion` is `mod_revision`. A store that renumbers revisions invalidates every client's cached resource versions.
- **`etcdctl make-mirror` is not enough.** It re-issues writes on the target, so the target assigns its own revisions (*inferred from how it works; not re-read here*).

Two routes:

| Route | How | Downtime | Confidence |
|---|---|---|---|
| Offline conversion | Go tool that reads the bbolt snapshot with the bbolt library and writes the replacement's format. It keeps revisions, compaction point, leases, auth and members, assigning new member IDs | Minutes for GB-scale databases (my estimate, unmeasured) | Moderate |
| Revision-preserving live mirror | Watch etcd from revision R with `prev_kv`. Apply each revision's event group as one transaction at a *forced* revision through an internal import API. Sync leases (`Leases` and `TimeToLive`) and auth separately. Then a brief write freeze and an endpoint switch | Seconds | Moderate–Low: TTLs are approximate, and the import API is new |

The same Go converter also restores portability for backups: it can convert replacement snapshots into bbolt files that `etcdutl` can read.

---

## 4. NuRaft fit (ClickHouse fork `e937522`)

### 4.1 What fits

- **Pluggable everything.** The application supplies the state machine, log store, state manager, transport (`rpc_client_factory`, `rpc_listener`), scheduler and logger (`include/libnuraft/context.hxx:45-52`).
- **etcd-style features exist:**
  - pre-vote, always on (`src/handle_vote.cxx:48-130`);
  - learners (`srv_config.hxx:116-120`);
  - priority elections;
  - leadership transfer (`yield_leadership`, `request_leadership`);
  - check-quorum-like `leadership_expiry_`;
  - batching, streaming and parallel log appending;
  - logical-object snapshot transfer.
- **Term and index exposure covers etcd's `Status` fields:** `get_term`, `get_last_log_idx` and the committed indices (`raft_server.hxx:535-585`). `commit_ext` receives `log_idx` and `log_term`, from which the revision and `raft_term` headers can be built.
- **Production use:** ClickHouse Keeper and eBay.

### 4.2 Gaps

| Gap | Evidence | Workaround |
|---|---|---|
| No ReadIndex | grep for `read_index` / `readindex` / `linearizab` over `include/`, `src/` and `docs/`: no hits. The maintainer points users to leadership expiry (eBay #288, #475). ClickHouse's `leader_status_request` returns the *applied* index with no quorum check | §3.4 |
| int32 IDs, single endpoint string, no endpoint update, no direct leader removal, active-catch-up `add_srv`, no promotion readiness check | §3.4 citations | §3.4 shims |
| No joint consensus | grep for `joint`: nothing; eBay #463 | etcd's server also changes one member at a time, so this matches |
| Durability contract undocumented | §3.2 | Synced log store; crash tests (§8.7) |
| Not deterministic | Timers read `steady_clock` (`include/libnuraft/internal_timer.hxx:49-106`); real commit and append threads; tests sleep in real time. The test suite does have `FakeNetwork`/`FakeTimer` (`tests/unit/fake_network.hxx`) | Antithesis (§8.4), lazyfs, Jepsen |

### 4.3 Threading and failure behavior that shape the design

- **One commit thread per server** applies entries strictly in order (`src/handle_commit.cxx:199-297`).
  - In `async_handler` mode, result handlers run inline on that thread (`include/libnuraft/async.hxx:173-185`), so a slow handler stalls every apply.
  - Watch fan-out and reply delivery must therefore be handed off to other threads.
- **An exception thrown by `commit` exits the process** (`src/handle_commit.cxx:185-192`). That is crash-stop, which is safe, but it turns apply bugs into availability incidents.
- **On a follower, log appends run under the main Raft lock** (`src/raft_server.cxx:823`). Without parallel appending, a slow fsync delays heartbeat and vote handling on that node. *(Inferred, Moderate.)*
- **Snapshot transfer is stop-and-wait per object**, so throughput is roughly object size ÷ RTT. GB-scale state needs large objects. Upstream etcd names followers unable to catch up via multi-GB snapshots as a current problem (#22332). Measure it.

### 4.4 Request results

- **`TIMEOUT` is not "not committed."** The entry is already in the log and may still commit (`src/handle_client_request.cxx:354-359`). Map it to etcd's indefinite `Unavailable` errors (§2.3).
- **Results fire after the leader's apply**, one per batch (`src/handle_commit.cxx:397-500`). This matches etcd, where a reply follows apply.

### 4.5 Fork health

| | eBay/NuRaft | ClickHouse/NuRaft |
|---|---|---|
| Last master commit / push | 2026-10-12 / 2026-11-01 | 2026-11-06 |
| Releases | v3.0.0 (2025-03-31) | none |
| Divergence | — | 307 commits ahead, 19 behind (merge base 2026-05-28) |

Upstream fixes missing from the fork:
- **PR #668: RequestVote freshness after full log compaction (merged 2026-10-08).** The fork still compares against `log_store_->last_entry()->get_term()` (`src/handle_vote.cxx:334-337`). That is a dummy term-0 entry when the log is empty, which can let a stale candidate win a vote. *(High that the gap exists; Moderate on how reachable it is.)*
- **#651**: inherited uncommitted config.
- **#660**: int32 overflow in the not-responding check.
- **#662**: a new leader waits for its state machine to catch up.
- **#653**: iterator invalidation pattern, still present.

ClickHouse-only fixes cover snapshot-install livelock, duplicate appends, commit-index rewind and removed-server rejoin. The two forks' wire formats already differ: ClickHouse adds message types 30–32. **Adopting NuRaft means owning a fork and auditing both histories.** *(High.)*

---

## 5. RocksDB fit (`v11.13.2`)

### 5.1 What fits

- **Atomic multi-column-family WriteBatch** carries the applied index with the data, matching etcd's consistent index (`storage-claude.md` §2.4).
- **WAL off plus `atomic_flush`** (`include/rocksdb/options.h:1617,2712`) gives the one-fsync-stream design in §3.2.
- **Checkpoints** (`include/rocksdb/utilities/checkpoint.h:167`) produce hard-linked snapshots that can be streamed as NuRaft logical objects and as backups.
- **No defrag.** LSM compaction replaces bbolt's stop-the-world defragmentation. `Defragment` can become a `CompactRange` or a no-op.
- **Fault-injection support:**
  - `utilities/fault_injection_fs.h`;
  - `test_util/sync_point.h`;
  - RocksDB's own crash-test infrastructure, which covers the engine's WAL and flush logic.

### 5.2 User-defined timestamps: semantically ideal, practically premature

**The GC rule.** RocksDB's compaction keeps every version at or above `full_history_ts_low`, plus "the largest one older than full_history_ts_low" for each key (`db/compaction/compaction_iterator.cc:784-786 @ v11.13.2`). Reads below the cutoff fail (`FailIfReadCollapsedHistory`, `db/db_impl/db_impl.cc:3892`). This is etcd's compaction contract: keep the latest version at or below the compaction revision, and return `ErrCompacted` below it. *(High: read in code.)*

**Reasons not to use it in v1:**
1. **Watch still needs a by-revision table.** UDT orders data by user key, but watch catch-up reads every change after revision R. So `rev` is needed anyway, and UDT would only replace the key index.
2. **Limited transaction support.** UDT works only with write-committed transactions.
3. **`persist_user_defined_timestamps=false` has restrictions.** It is not supported with atomic flush or concurrent memtable writes (`include/rocksdb/advanced_options.h:1310-1332`).
4. **Open 2026 bugs on the paths compaction would use**, as reported by the research pass: meta/rocksdb #15424 (`full_history_ts_low` with Merge) and #15558 (key corruption at transaction commit).
5. **No public production MVCC user found.**

*(Moderate.)*

### 5.3 Performance risks (unmeasured)

- **Reads.** A Range does one point lookup per result revision (`storage-claude.md` §2.2). The upstream Pebble prototype measured Get at about 6.5× and Range at about 5× lower throughput than bbolt (#22332, MVCC layer, compression on). RocksDB with a block cache sized to the working set may do better, but nobody has measured it. Kubernetes is list-heavy. *(Moderate.)*
- **Compaction I/O against fsync.** Background compaction competes with the Raft log's fsync, which sits on the client latency path (`storage-claude.md` §3.3). Use a rate limiter and separate devices where possible.
- **Quota semantics.** etcd's NOSPACE alarm is based on the bbolt file size. In RocksDB the closest measures are `rocksdb.total-sst-files-size` (`include/rocksdb/db.h:1790`) or the live-data estimate, which include space amplification. `tests/common` `TestAlarm` expects a 13-page quota and recovery by compact + defrag (`tests/common/alarm_test.go:37,77-80`), so this must be emulated deliberately.

### 5.4 Precedent

ClickHouse Keeper added RocksDB-backed storage in 2024 (#56626, experimental). It removed it in 2026-06 (#108000) in favor of a custom LSM, for performance and control. In both designs Keeper's own changelog plus snapshots were the source of truth (§9.1). That is a caution on RocksDB's performance for one specific workload, not on its correctness. *(High for the facts; Low on their transfer to etcd's workload.)*

---

## 6. Approach A vs Approach B

### 6.1 Comparison

| Criterion | A: gRPC wire-compatible | B: language-level API only |
|---|---|---|
| Unmodified clients | All: Go, jetcd, Python, Rust and C++ clients, etcdctl, kube-apiserver, operators | None. Each language needs a replacement library with the same API |
| Data model freedom | None | **None either**: the client APIs expose the protobuf messages (§1) |
| Transport freedom | Must speak gRPC with etcd's messages, errors and metadata | Yes |
| Kubernetes | Works unmodified | The Go API takes `[]grpc.DialOption`. kube-apiserver passes Prometheus gRPC interceptors, an OpenTelemetry stats handler and a custom dialer (`staging/src/k8s.io/apiserver/pkg/storage/storagebackend/factory/etcd3.go:310-347 @ k8s dbe731cf4b02`). So either keep gRPC underneath (then why B?) or patch and rebuild Kubernetes |
| Test reuse (§8) | Highest: raw-gRPC tests, curl gateway tests, etcdctl, jetcd suite, Jepsen and robustness all run as they are | Loses raw-gRPC tests (≥74 functions), the 42 curl tests, unmodified etcdctl, jetcd's suite and the gRPC health check. Jepsen needs a Java library with jetcd's class names. The test module is compile-coupled to the server module |
| Extra code | Exact error strings, metadata semantics, gateway (T2) | Per language: Go `client/v3` (11.3k lines), jetcd-core (10.4k), plus Python and others, each with its own correctness campaign. Jepsen found real client bugs in jetcd 0.8.2 (§8.5) |
| Ecosystem adoption | Drop-in | Every tool and operator must relink or be rewritten |

### 6.2 What only B buys, and how A gets most of it

B's real advantages are wire changes old clients cannot express:
- **Exactly-once semantics.** etcd has no request IDs, so a server cannot deduplicate a client's retries. Client session IDs with sequence numbers would fix that, the approach of the Raft dissertation §6.3 (*recalled*).
- **Chubby-style coupling** of session KeepAlive with cache invalidation (`chubby-perspectives-claude.md`).
- **A leaner framing.**

Under A, each can be offered as an **optional new gRPC service, or new fields** that old clients ignore. Proto3 tolerates unknown fields, and old clients never call new services. New clients opt in; old clients keep the old semantics.

What A cannot do is drop the legacy obligations. That is the price of the ecosystem, and it is the reason to choose A. *(High.)*

**Verdict:** A, with optional extensions where needed. B is dominated unless the project is a closed system that controls every client and has no Kubernetes or tooling dependency. *(High.)*

---

## 7. Language: C++, Rust, or both

| | All C++ | Rust + C++ shim over NuRaft |
|---|---|---|
| NuRaft integration | Native: virtual interfaces, `std::shared_ptr` (`ptr<>`), exceptions | A hand-written C++ shim subclasses `state_machine`, `log_store`, `state_mgr` and the transport and logger interfaces, and forwards to Rust. cxx cannot subclass. A few dozen virtual methods, many optional, plus the control API |
| FFI hazards | None | Callbacks from asio and commit threads (`Send + Sync`); `catch_unwind` at every Rust entry; C++ exceptions caught at every call; copies of caller-owned `buffer&` (`include/libnuraft/state_machine.hxx:62-66`); bridging `cmd_result` handlers to futures |
| RocksDB | Native, current (11.13.2) | `rocksdb` crate 0.25.0 bundles RocksDB **11.8.1** and has no commit-timestamp API (research pass). Accept the lag or build against 11.13.2 |
| gRPC server | C++ callback API (`ServerBidiReactor`); harder to get right for Watch and KeepAlive streams | tonic 0.14 (now `grpc/grpc-rust`); used by the `etcd-client` crate and linkerd2-proxy. Open stream-lifecycle issues exist (#2079, #2983). TiKV uses grpcio instead |
| HTTP/JSON gateway | grpc-httpjson-transcoding library, or an Envoy transcoder in front | Young crates, or hand-written: 41 POST endpoints, mechanical work for agents |
| Agent-written code | Memory and thread bugs surface at run time (ASan/TSan) on the exercised paths only | Ownership and `Send`/`Sync` errors surface at compile time, a fast and cheap feedback loop exactly where agents write the most code |
| Precedent | ClickHouse Keeper (C++ on NuRaft) | Xline (Rust, RocksDB, own consensus); mem_etcd (Rust, single node) |
| Builds and debugging | One toolchain (CMake) | Cargo + CMake; stack traces cross FFI under fault injection |

**Recommendation: Rust for everything above the consensus boundary**: gRPC services, MVCC, watch hub, lessor, auth, apply logic, shim, tooling. Use a thin C++ shim for NuRaft and keep NuRaft fork patches in C++. *(Moderate–Low.)*

Why:
- Most of the code is concurrent, semantics-heavy service logic. There, compile-time checking compounds with agent-driven iteration.
- The FFI surface is small and stable.

This fails if:
- the team must rewrite substantial NuRaft internals (ReadIndex, fork merges), so C++ expertise is needed anyway and the shim becomes a moving target; or
- reviewers are stronger in C++.

In either case, all-C++ following Keeper is the safer call. If the NuRaft constraint is negotiable and the language is Rust, raft-rs removes the shim and two of NuRaft's five gaps (§1). That is the larger simplification.

---

## 8. Test strategy and reuse

### 8.1 The enabler: a flag-compatible launcher

Most reusable harnesses start a binary named `etcd` and talk to it:
- **etcd's e2e framework** runs whatever is at `--bin-dir/etcd` (`tests/framework/e2e/flags.go:71-83`). etcd uses this to test older releases (`tests/robustness/Makefile:17-29`).
- **Kubernetes integration tests** look up `etcd` on `PATH` (`test/integration/framework/etcd.go:55-56 @ k8s dbe731cf4b02`) and pass:
  - `--data-dir`
  - `--listen-client-urls unix://…`
  - `--advertise-client-urls`
  - `--listen-peer-urls`
  - `-log-level`
  - `--quota-backend-bytes`

  (`:117-138`). They also accept `KUBE_INTEGRATION_ETCD_URL` for an already-running server (`:63`).
- **jetcd's launcher** runs the container named by `ETCD_IMAGE` (default `quay.io/coreos/etcd:v3.5.14`; `jetcd-launcher/src/main/java/io/etcd/jetcd/launcher/Etcd.java:31,40 @ jetcd-0.8.7`) with:
  - `--name`
  - `--advertise-client-urls`
  - `--listen-client-urls`
  - `--data-dir`
  - `--cert-file`
  - `--key-file`
  - `--initial-advertise-peer-urls`
  - `--listen-peer-urls`
  - `--initial-cluster`
  - `--initial-cluster-state`
  - `--initial-cluster-token`

  (`EtcdContainer.java:173-217`).
- **etcd's Antithesis setup** and **Jepsen's `antithesis/`** configure the server only through `ETCD_*` environment variables in docker-compose.

What the launcher must provide:
- those flags and variables;
- the line `ready to serve client requests` (`tests/framework/e2e/etcd_process.go:41,259`);
- `--version` output starting `etcd Version: X.Y.Z` (`:505-530`);
- SIGTERM shutdown.

The e2e framework also turns any non-default `embed.Config` field a test sets into a flag, by reflection (`tests/framework/e2e/cluster.go:139,635-645`). The flag subset therefore grows with the tests adopted. *(High.)*

### 8.2 Differential testing against etcd (new, highest leverage)

Run every generated request sequence against both the replacement and etcd v3.7.3. Compare responses, revisions, `version`/`create_revision`/`mod_revision`, errors (exact strings) and watch streams. Normalize cluster and member IDs and `raft_term`.

Sources of sequences:
- the robustness traffic generators (§8.3);
- the Kubernetes request shapes (§2.2);
- property-based generators;
- recorded production traffic.

Two oracles cheaper than a second server:
- **`tests/robustness/model/deterministic.go:140-236`**: a pure step function for Range at the current revision, Txn with Version/ModRevision compares, Put/Delete, LeaseGrant/Revoke and Compact. Its own comment calls it suitable for "writing a fake".
- **HashKV equality**, if the encoding is mirrored (§3.3).

This is the "external oracle" that made the strongest agent-built systems work (§9.4), and it targets the layer where Xline failed (§8.5). *(Moderate–High.)*

### 8.3 What the etcd repository provides

| Test tree (v3.7.3) | Size | Reuse under A | Reuse under B |
|---|---|---|---|
| `tests/common` | 4,000 + 169 lines; 95 cases | Through the e2e runner and the launcher: ≈93 of 95 cases with no Go changes. Drop `TestAuthority`, which greps a Go `http2debug` log line; emulate the alarm quota | Needs etcdctl rebuilt against the replacement library, plus compile coupling |
| `tests/e2e` | 12,013 lines; 221 functions | 86 (etcdctl/client only) as they are; +42 curl tests if the gateway exists; +11 with a gofail-compatible endpoint; 81 not reusable (bbolt/WAL/etcdutl, old binaries, etcd-specific flags and logs) | 86 with rebuilt etcdctl; no curl tests |
| `tests/integration` | 25,750 lines; 486 functions | Only through a new process-backed version of `framework/integration` (concrete structs; in-process server today): ≈293 client-only + ≈71 client-side-network functions. `clientv3/examples` and `clientv3/concurrency` need only an endpoint swap | ≈234 (clientv3 API only); no raw-gRPC tests |
| `tests/robustness` | 7,441 test + 7,962 other lines | **Biggest payoff:** porcupine linearizability checking, 9 watch-guarantee validators, kill / peer-network (needs peer TLS on plain TCP) / LazyFS faults, and member replacement. Adapters: (1) a log reader replacing `ReadWAL`, about 70 lines in etcd. It enables watch and serializable validation, which are otherwise skipped (`tests/robustness/validate/validate.go:41-55`), and the payload decoder carries over if the log stores `InternalRaftRequest`. (2) Optionally a gofail-compatible HTTP endpoint (`GET /`, `PUT /<name>`, `DELETE /<name>`, address from `GOFAIL_HTTP`) with failpoints named after their NuRaft/RocksDB analogues; the bbolt-specific ones have none | Same, minus the gRPC health-check path |
| `tests/antithesis` | 483 lines + compose files | A server image honoring `ETCD_*`, plus the log reader | Same |
| `tools/benchmark` | 1,928 lines | As is, except the `mvcc*` subcommands: the same Go workload against both servers | Rebuild with `replace` |
| `server/**` unit tests | 32,235 lines | Not reusable; a source of semantics to port | Same |

These counts and classifications come from the research pass. I re-checked the totals and several cited lines; per-file classes are heuristic. *(Moderate.)*

**The robustness bug table as a regression checklist.** `tests/robustness/README.md:21-39 @ v3.7.3` lists 17 correctness issues (13766 through 20418). Most have reproduction targets (`make test-robustness-issueNNNNN`).
- By area (areas overlap): 8 involve watches; 5 involve compaction or defrag interactions; 4 crash recovery; 1 auth; 2 panics; 1 stale reads under process pause.
- Who found them: users found 6, robustness tests 8, Antithesis 3.

Each is a scenario the replacement must pass. Porting the scenarios is mechanical agent work. *(High for the table.)*

### 8.4 Jepsen, Antithesis, Kubernetes, client suites

- **Jepsen (`jepsen-io/etcd`, 3,152 lines of Clojure).**
  - **Reusable unchanged under A:** 1,982 lines (63%): workloads (append, wr, register, set, lock-set, lock-etcd-set, watch), checkers (Elle strict-serializable, Knossos), and the jetcd client wrapper.
  - **To rewrite:** about 300–400 lines in `db.clj` and nearby. That covers install, flags, data, log and pid paths, the process name used for pause, the file targets of the corruption nemesis, and the zap-JSON crash regex (`etcd.clj:162-168`).
  - **Upgrades needed:**
    - The suite pins jetcd 0.8.5 and Jepsen 0.3.11.
    - Move to jetcd ≥ 0.8.7 and Jepsen ≥ 0.3.15. Skip 0.3.12–0.3.14, where `grepkill!` (used for pauses) silently did nothing.
    - The setup also passes `--enable-v2` and `--experimental-*corrupt-check*` flags, which were removed in etcd v3.6+, so it must be updated even to test real etcd v3.7.3.
  - **Gaps:** no lease workload; no deletes, range reads, multi-key or `prev_kv` watches, progress notifications or auth. Write new workloads, or rely on the robustness traffic, which covers leases, deletes and defrag.
  - *(High; line counts verified by the research pass.)*
- **Antithesis.** Both etcd's `tests/antithesis` and Jepsen's `antithesis/` need only a replacement server image. A deterministic hypervisor compensates for NuRaft's lack of deterministic simulation (§4.2). It is a commercial service. *(Moderate.)*
- **Kubernetes.**
  - Integration tests (`test/integration`) run against the launcher on `PATH`. Unix-socket client URLs are required (§8.1).
  - Conformance (`sig-api-machinery`) is the gate Kine uses (`scripts/test-conformance`).
  - These are the tests that matter for the Kubernetes slice. *(High.)*
- **Client suites.**
  - jetcd's tests (31 files, 4,868 lines) run against a replacement image via `ETCD_IMAGE`.
  - Other language clients' integration suites can be pointed at it the same way.
  - Under A these validate the server. Under B they would have to be rewritten together with the new libraries.

### 8.5 Jepsen findings and what they mean for a replacement

| Finding | Replacement |
|---|---|
| etcd 3.4.3: KV strict serializable under all tested faults; single-key watches complete and in order | **The bar to meet** |
| Watch `start_revision=0` means "now + 1" (#11496) | **Replicate**: proto3 default semantics |
| Lock granted without re-checking the lease after waiting (#11456) | **Avoid**: under A the Lock RPC is server-side; re-check ownership after waiting |
| Lease locks are not mutexes (#11457) | **Inherited** from the API; support fencing through the lock key's revision |
| jetcd ≤0.8.4 retried non-idempotent RPCs → lost update, G1c, G1a (jetcd 0.8.2 analysis; fixed by `05f92e0`, in 0.8.5+) | Client-side. **Under A:** pin jetcd ≥0.8.5. **Under B:** the new libraries must never retry an operation that may have been sent. **The server can cause the same G1a pattern** by reporting an indefinite NuRaft outcome as definite (§2.3) |
| Lost un-fsynced writes → stale read or lost commit (#14140, #14143; never resolved) | **Open risk** for any new log store: make lazyfs + kill a gating test from day one |
| Bit flips crash bbolt (#14102) | RocksDB checksums blocks (*recalled*); the log store needs per-entry CRCs and must refuse to serve or vote after detecting corruption (the NuRaft #185 lesson) |
| Xline (etcd-compatible, Rust) failed Jepsen `append` and `wr` **with no faults injected**: G2-item from conflict-key construction, txn ops not applied in order, broken revision generation (Xline #467, #470, #471) | **Where a rewrite fails first:** revision assignment and txn semantics. This is exactly what §8.2 targets |

### 8.6 New tests to write

- **The differential harness** (§8.2).
- **Lease semantics** P2/P3 across failover, from `leases-claude.md`. Neither Jepsen nor the deterministic model checks TTLs.
- **Watch liveness** under leader loss and stalled members, from `watchers-claude.md`: what `WithRequireLeader` does and does not guarantee.
- **Membership flows:** add-before-start, update URLs, remove leader, promote readiness. NuRaft's semantics differ from etcd's (§3.4).
- **Log-store crash tests** at the filesystem level: torn writes, missing directory fsync, rename atomicity. These are the bug classes ClickHouse found in 2026, which Jepsen running every 6 hours had not caught (§9.1).
- **Migration converter tests:** etcd snapshot → replacement → compare every key's revisions, plus HashKV.

### 8.7 Test effort

| Item | Size | Confidence |
|---|---|---|
| Launcher with etcd flag and `ETCD_*` compatibility | 1–3k lines | Moderate |
| Robustness adapters (log reader, gofail-compatible endpoint, failpoint mapping) | ≈1k lines | Moderate |
| Process-backed `framework/integration` | 2–4k lines (the exported struct API is large) | Low |
| Jepsen `db.clj` rewrite + version bumps + lease and multi-key watch workloads | 0.5–1.5k lines of Clojure | Moderate |
| Differential harness + generators | 3–6k lines | Low |
| New unit, property and crash tests | ≈1–1.5× production code | Low |

---

## 9. Prior art

### 9.1 ClickHouse Keeper: the closest precedent

A wire-compatible ZooKeeper replacement, in C++, on NuRaft.

| Item | Fact | Source |
|---|---|---|
| Timeline | NuRaft added to ClickHouse 2020-12-26 (#18491) → first `src/Coordination` commit 2021-01-13 → Jepsen tests 2021-03-27 → "production ready" in 22.3 LTS, 2022-04 (in production "more than half a year" before) | GitHub API; ClickHouse 22.3 blog |
| Effort | 74% of the 408 pre-GA non-merge commits by one engineer; it reused a ZooKeeper codec ClickHouse had owned since 2018 | commits API |
| Size now | `src/Coordination` 38,682 non-test lines + 18,281 test lines; 103 authors over its lifetime | sparse clone at ClickHouse `89e3dfc65ef1` |
| Testing | Jepsen in CI (every 6 hours per a 2023 blog); 114 integration-test directories; stress tests. Corruption nemeses are excluded by default because of NuRaft #185 | `tests/jepsen.clickhouse` |
| Late durability bugs | 2026-07: no directory fsync (#111339); non-atomic term/vote file, with split-brain risk after a kill (#111454); log gap after async segment removal (#112101); async unlink deleting the live segment (#112535). All closed Aug–Sep 2026 | GitHub issues |
| Reads | ZooKeeper semantics: local reads, `sync`, optional `quorum_reads` that send reads through Raft. No ReadIndex | `CoordinationSettings.cpp:70` |
| RocksDB | Experimental storage 2024 (#56626); removed 2026-06 (#108000, body: "This is one-shot by claude, I mostly didn't review it"); replaced by a custom LSM | PRs |

Read through the research pass; I re-checked #108000's merge date and title.

**Lesson.** About 15 months to "production ready" for a smaller API, by a strong team that already owned the codec, on top of an existing database's infrastructure. Then a durability-bug tail lasting years. ZooKeeper's API has no revision history, no ranges at a revision, and no MVCC watch replay. For semantic surface, etcd's T0 tier is larger than ZooKeeper's whole API (my judgement). *(Moderate.)*

### 9.2 etcd-compatible servers

| System | Approach | Status | Lesson |
|---|---|---|---|
| **Xline** (Rust, CURP, RocksDB) | Full etcd API intended; validation by etcdctl scripts and a partial Jepsen run | Last master commit 2024-12-03 (verified); CNCF lists it as archived on 2025-09-19 (per the research pass; the GitHub repo itself is not archived). Never finished Compact, member APIs or snapshot compatibility in its validation report; Jepsen `append`/`wr` "mostly failed" without faults; had watch event loss, repeated revisions, missing ReadIndex | Full compatibility is a large target; failures concentrated in txn and revision semantics |
| **Kine** (k3s) | Kubernetes subset over SQL/NATS; accepts only the Kubernetes Txn shapes, everything else `ErrNotSupported` | Active (v0.17.5, 2026-11-05); gate is Kubernetes `sig-api-machinery` conformance, on sqlite only | The Kubernetes slice is small and testable by conformance |
| **KubeBrain** (ByteDance) | Kubernetes subset on Badger/TiKV | Dormant since 2024-03 | — |
| **mem_etcd** (k8s-1m) | Rust, single node, in memory, about 2,945 lines | Research artifact | Measured the slice Kubernetes uses |
| **GKE** | Spanner-based store "implementing the etcd API" | Production at 65k–130k nodes | Built on a mature distributed database |
| **EKS** | Kept etcd's code; replaced Raft with an internal journal; bbolt on tmpfs | Production for ultra-scale clusters | Replaced one layer, not two |
| **Azure Cosmos DB etcd API** | Claimed wire compatibility | Preview withdrawn after about 18 months (2020) | — |

Neither hyperscaler built new consensus *and* new storage at the same time. *(Moderate; inference from public posts.)*

### 9.3 Raft logs in LSM stores

- **TiKV** moved Raft logs from RocksDB to Raft Engine, citing double writes and compaction write amplification. It reported write I/O down 25–40% and tail latency down 20%.
- **Dragonboat** went RocksDB → Pebble → an experimental custom log, "tan".
- **CockroachDB's** log-separation issue has been open since 2017.

Taken together: a log in an LSM store works but costs I/O, and separating it later is hard. Hence §3.2's advice: keep the log in its own store from the start, even if that store is RocksDB at first. *(Moderate.)*

### 9.4 Coding agents on large systems work

| Case | Outcome | Oracle |
|---|---|---|
| etcd #22332: Pebble/Badger backends, "mostly vibed" | Working MVCC-layer prototypes with benchmark numbers; not integrated | bbolt implementation + linearizability harness |
| Anthropic C compiler (2026-02) | About 100k lines of Rust, about $20k, two weeks; builds Linux; output slower than `gcc -O0`; "new features and bugfixes frequently broke existing functionality" | GCC |
| Ladybird LibJS → Rust (2026-02) | About 25k lines in about 2 weeks; 0 regressions on test262 | Byte-identical output vs the C++ engine |
| Cloudflare vinext (2026-02) | 94% of the Next.js API claimed in under a week; "experimental" | Ported Next.js test suite |
| METR randomized trials (2025–26) | Small or uncertain effects for experienced developers on mature codebases | — |

These were read by the research pass from primary posts; I did not re-read them. **Pattern:** strong results where an external oracle exists; weak or contested results otherwise. etcd supplies such an oracle (§8.2), which is the best argument that agents can cut implementation cost substantially here. It is not evidence that they shorten the fault-hardening tail. *(Moderate.)*

---

## 10. Cost and complexity

### 10.1 Size of the build

| Work package | Production lines (estimate) | Notes |
|---|---|---|
| NuRaft integration: state machine, log store, `state_mgr`, snapshots, membership shims, read barrier, proposal routing, fork patches | 6–10k | Highest-risk code per line |
| MVCC on RocksDB + compaction + HashKV | 6–10k | Port of `storage/mvcc` (3,972 Go lines) plus schema |
| Watch subsystem | 4–6k | Most bug-prone area historically |
| Apply / txn semantics, limits, errors | 4–6k | Port of `apply` + `txn` (2,141) and parts of `etcdserver` |
| Lessor with fencing and TTL persistence | 2–4k | Port of `lease` (1,522) |
| gRPC services, interceptors, metadata, health | 4–7k | Port of `v3rpc` (2,884) |
| Auth (T2) | 3–4k | Port of `auth` (2,208) |
| Lock / Election (T2), gateway (T2) | 2–4k | Mostly generated or mechanical |
| Ops: flags/launcher, TLS (client and peer, reload), metrics with etcd's names, logging, quota/alarm, corruption check, `etcdutl` equivalent, Go migration converter | 8–12k | `embed` + `etcdmain` alone are 5,450 Go lines |
| **Total: Kubernetes slice / full API** | **≈30–45k / ≈40–60k** | Low confidence; Rust and C++ are usually somewhat larger than Go |

Test code adds about 1–1.5× production code, plus the adapters in §8.7.

### 10.2 Where agents help, and where they do not

| Activity | Share of a traditional project (my estimate) | Agent compression | Why |
|---|---|---|---|
| Porting semantics module by module, with etcd source as the spec and differential tests as acceptance | 30–40% | Large (3–10×) | Oracle-backed; matches the §9.4 successes |
| Test-adapter and harness porting, scenario porting from the 17-issue table | 10–15% | Large | Mechanical |
| Design of durability, read path, membership mapping, lease fencing; review of safety-critical code | 10–15% | Small | Judgement; errors are silent until a fault test hits them |
| Fault-injection campaigns: run, diagnose, fix, re-run (robustness, Jepsen, lazyfs, Antithesis) | 25–35% | Moderate (1.5–2×) | Wall-clock bound by test runs; agents help triage porcupine/Elle histories and logs |
| Performance work on target hardware | 5–10% | Small–moderate | Needs the benchmark machine (`AGENTS.md` §7) |

By Amdahl's law the overall speed-up is about **1.7–2.5×**, not the implementation phase's 3–10×. *(Low: the shares and factors are judgements, informed by but not measured from §9.4.)*

### 10.3 Scenarios

All figures are to a **production candidate**: passing the reused etcd suites, robustness, Jepsen with all nemeses including lazyfs, and (for S1) Kubernetes integration and conformance. All assume senior distributed-systems engineers directing agents. After this, plan **≥6 months of shadow or soak running** before trusting it as the primary store. Keeper ran in production for about six months before being declared ready.

| Scenario | Scope | Team | Calendar | Engineer-months | Confidence |
|---|---|---|---|---|---|
| **S1** | Kubernetes slice, A, 3–5 members, ops tooling, migration converter | 2–3 | 7–14 months | 15–35 | Low |
| **S2** | Full v3.7 API, A (adds auth, lock/election, gateway, full Range/Txn generality, all lease and member RPCs, etcdctl/etcdutl parity) | 3–4 | 10–20 months | 30–60 | Low |
| **S3** | B: S2's server minus about 1–2 months of wire-detail work, plus Go and Java client libraries (≈3–6 months each including correctness campaigns), plus patched Kubernetes | 4–5 | 14–26 months | 45–90 | Low |
| **Ongoing** | NuRaft fork merges, etcd API tracking (e.g. RangeStream), Kubernetes behavior changes, RocksDB upgrades, security fixes, on-call | 1.5–3 FTE | — | — | Low |

Phase plan behind S1:
1. Phase 0, spikes plus harness: 1–2 months.
2. Single-node semantics under differential testing: 1–2 months.
3. Replication on NuRaft with crash tests: 2–4 months.
4. Hardening and Kubernetes gates: 3–6 months.

These estimates fail if:
- NuRaft integration surfaces protocol-level bugs beyond those listed in §4.5 (the estimates go up);
- the read barrier is too slow and ReadIndex must be built early (up);
- the differential oracle catches most semantic bugs before fault testing (down).

### 10.4 Against cheaper alternatives

| Alternative | Cost (from earlier analyses) | What it does not give |
|---|---|---|
| Measure first, then tune etcd (E0 in `storage-claude.md`) | Days | — |
| Partition by domain across several etcd clusters | ≈0 code | Cross-partition transactions |
| Decouple bbolt commits from the apply loop (`storage-claude.md` §4.4) | Small | LSM benefits |
| Pebble backend in an etcd fork (`storage-claude.md` §11) | ≈6–12 engineer-months to production | A non-Go runtime; replaced consensus |
| WAL plus backend replacement in a fork (`wal-replacement-claude.md`) | Larger than Pebble alone | Same |
| **Rewrite (S1/S2)** | **15–60 engineer-months + 1.5–3 FTE ongoing** | Nothing a fork does — but it keeps every API-mandated limit (§11.2) |

**Decision rule.** A rewrite earns its cost only if a goal requires what a fork cannot give:
- a non-Go runtime (no garbage-collection pauses, C++/Rust embedding);
- a different replication substrate (NuRaft's priority elections, custom quorums, full-consensus mode);
- a keyspace larger than RAM (L2 in §3.3);
- strategic ownership of the codebase.

Write-throughput or defrag pain alone does not meet that bar. *(Moderate.)*

---

## 11. What a rewrite can and cannot change

### 11.1 Fixable internally under A

| etcd issue (from this project's analyses) | Rewrite mechanism | Confidence |
|---|---|---|
| Term-less, timer-driven lease revocations (`leases-claude.md`) | `expected_term_` + apply-time term check (§3.5) | Moderate |
| No TTL upper bound across failovers (P2) | Persisted remaining TTL by default | Moderate |
| bbolt commit stalls; forced commits on deletes and revocations; double fdatasync | Single fsync stream, RocksDB WAL off (§3.2) | Moderate |
| Stop-the-world defrag | LSM compaction | High |
| Keys bounded by RAM; slow startup rebuilding the index | L2 persisted index (later) | Moderate |
| Go GC pauses as a source of stalls | C++/Rust runtime | Moderate (#20418 shows pauses matter; its root cause was read-path logic) |
| Watch progress and liveness gaps (`watchers-claude.md`) | Server-side progress guarantees within the existing API (progress notify, RequestProgress) | Low–Moderate |

### 11.2 Not fixable without breaking compatibility

- **One totally ordered revision sequence.** Each write increments the cluster-wide revision, watches are totally ordered, and Txn spans arbitrary keys. That pins the design to one Raft group, so write throughput is capped by one leader's log and fsync. A rewrite raises constant factors, not this ceiling. *(High.)*
- **Lease-based locks without fencing** (Jepsen #11457). *(High.)*
- **Bugs in unmodified clients**: Go #22081/#22094, jetcd `onError`, optimistic client lease deadlines, pre-0.8.5 jetcd retries. Fix them upstream or ship patched clients. *(High.)*
- **No rolling migration** from etcd (§3.6). *(High.)*

---

## 12. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Log-store durability bug (lost acked writes, rolled-back commits) | High during development | Critical | lazyfs + kill gating; filesystem-level crash tests; RocksDB-based log first (§3.2) |
| Txn/revision semantic drift (the Xline failure mode) | High early | Critical for Kubernetes | Differential testing from day one (§8.2) |
| Watch edge cases around compaction, partitions and resumption (8 of 17 etcd bugs) | High | High | Robustness watch validators; the 17-issue scenarios |
| Read path: barrier too slow, or unsafe lease reads | Moderate | High | Barrier first; ReadIndex patch gated by Jepsen and robustness |
| NuRaft fork safety debt (#668 and others) and ongoing divergence | Moderate | High | Pinned fork, cherry-picks, an audit budget |
| Membership semantic mismatches break operators' runbooks | Moderate | Moderate | Shims in §3.4; e2e member tests; Jepsen membership nemesis |
| Kubernetes moves the target (RangeStream-like additions) | High over years | Moderate | Track etcd `api/` and Kubernetes `etcd3/` changes as part of ongoing cost |
| Read throughput regression vs bbolt | Moderate | Moderate–High for list-heavy clusters | Benchmark on target hardware before committing (§5.3) |
| Agent-written code accepted unreviewed in safety paths | Moderate | Critical | Mandatory human review for durability, reads, membership and leases; adversarial review agents |
| Version impersonation (`Status.Version`) enables client behaviors the server does not implement | Low–Moderate | Moderate | Report the lowest etcd version whose semantics are fully met |

---

## 13. Verified vs inferred; open questions

### 13.1 Verified directly

**etcd v3.7.3:**
- `server/` LOC (48,173); 42 RPCs in `rpc.proto`; RangeStream at `rpc.proto:44-47` and its commit date;
- error mapping by message (`rpctypes/error.go:256-279`) and the `Unavailable` set (`:77-86`);
- Go retry rules (`client/v3/retry.go:54-62,85-94`); gRPC types in the client API (`config.go:85`, `client.go:277,566`);
- ReadIndex read loop (`server/etcdserver/read/read.go:69-160`); lease min TTL (`server.go:340`); limits (`embed/config.go:64,67`);
- Snapshot RPC digest (`maintenance.go:135-209`); gateway registration (`embed/serve.go:357-364`);
- the v3lock implementation via `concurrency` (`lock.go:36-46`); the robustness bug table (`tests/robustness/README.md:21-39`).

**Clients and Kubernetes:**
- jetcd: `Errors.java:29-40`; commit `05f92e0` and the first tag containing it; `GetResponse.java:27,33`; `Etcd.java:31,40`; `EtcdContainer.java:173-217`; test LOC.
- Kubernetes `dbe731cf4b02`: integration launcher flags, the version feature check, DialOptions, the RangeStream fallback; PR #140085 merged 2026-07-14 (milestone v1.37).

**NuRaft `e937522`:**
- no ReadIndex (grep); int32 IDs; `CANNOT_REMOVE_LEADER`; vote freshness code; endpoint-change rejection; the leadership-expiry doc;
- `expected_term_` fence; commit cursor initialized from `last_commit_index()`; "not recommended" forwarding;
- eBay #668 merged 2026-10-08 and #185's title.

**RocksDB `v11.13.2`:**
- the UDT GC rule (`compaction_iterator.cc:776-800`); `FailIfReadCollapsedHistory`; the `persist_user_defined_timestamps` restrictions;
- option and checkpoint and fault-injection locations.

**Prior art:**
- ClickHouse #108000 title and merge date; Xline's last master commit and that its GitHub repo is not archived.

### 13.2 From research passes, not re-checked line by line

- Per-file e2e and integration classifications (heuristic greps).
- Jepsen suite line splits.
- ClickHouse Keeper timeline, authorship and size; Kine, KubeBrain and mem_etcd details; hyperscaler posts.
- RocksDB open-issue numbers and Rust-crate versions; tonic issues.
- Agent-productivity case studies.
- NuRaft citations other than those in §13.1.

### 13.3 Inferred (my judgement)

- All effort and line-count estimates (§10).
- The read-path, log-store and MVCC-layout recommendations.
- Language choice.
- The revision-preserving live mirror.
- HashKV byte-identity.
- The share and compression table.

### 13.4 Open questions

1. **What is the goal?** Kubernetes backend (S1) or general etcd replacement (S2)? Which workload pain motivates it: write throughput, defrag, scale beyond RAM, runtime, ownership? The decision rule in §10.4 depends on the answer.
2. **Is NuRaft a hard constraint?** If not, and the language is Rust, raft-rs should be evaluated first.
3. **What read semantics are acceptable?** Barrier-cost linearizable reads in v1, or ReadIndex required from the start?
4. **What is the team's C++/Rust split** for review of safety-critical code?
5. **Is Antithesis available**, given NuRaft's lack of deterministic simulation?
6. **Does migration need to be live**, or is an offline cutover acceptable (§3.6)?
7. **Benchmark hardware** (`AGENTS.md` §11): no performance claim in this file can be settled without it.

---

## Conclusions

1. **The rewrite is feasible.** The risk sits in semantics under faults, not in code volume.
   - Writing about 30–60k lines of C++/Rust is the cheap part with coding agents, because etcd itself is an executable specification to diff against.
   - The expensive part is the long tail: durability of the new log store, watch and compaction edge cases, and transaction and revision semantics.
   - That tail is where etcd (17 tabulated bugs), Xline (Jepsen failures without faults) and ClickHouse Keeper (durability bugs five years in) all spent their effort.
2. **Adopt gRPC wire compatibility (A).** Language-level compatibility (B) keeps etcd's data model and still runs into gRPC types in the Go API that Kubernetes depends on. It multiplies client-library work, loses the ecosystem, and discards most reusable tests. Whatever B could offer can be added under A as optional extensions.
3. **NuRaft and RocksDB are workable foundations with known costs.**
   - NuRaft needs: a read barrier now and ReadIndex later; membership shims for etcd's uint64 IDs and add-before-start; and an owned fork with upstream safety fixes, starting with PR #668.
   - RocksDB fits a single-fsync design (WAL off, applied index in the batch, checkpoints for snapshots) and removes defrag and bbolt commit stalls. Use explicit revision-keyed tables, not user-defined timestamps, and measure range-read performance early.
4. **Test reuse is unusually good under A, but it hinges on a cheap investment:** a launcher that speaks etcd's flags, log line and version format. With it:
   - etcd's `tests/common`, most e2e client tests and the robustness suite run with about 1k lines of adapter;
   - Jepsen needs only a ~300–400-line setup rewrite;
   - Kubernetes integration and conformance tests and jetcd's suite run unmodified.

   The highest-value new artifact is a differential harness against etcd v3.7.3.
5. **Cost is set by scope, then by validation.** Roughly 15–35 engineer-months for the Kubernetes slice and 30–60 for the full API, plus at least six months of soak and 1.5–3 FTE ongoing *(Low)*. A storage fork of etcd costs a fraction of that. So build the rewrite only for goals a fork cannot meet: a non-Go runtime, a different replication substrate, a keyspace larger than RAM, or ownership. Even then the API's single revision order caps write throughput, as it does for etcd today.
