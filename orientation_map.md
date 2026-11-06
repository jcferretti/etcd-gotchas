# Orientation map: etcd and jetcd source trees

Where each subsystem lives in the local clones, plus module, dependency and
build notes. Moved out of `AGENTS.md` §4 on 2026-11-01. Valid for the pinned
releases in `AGENTS.md` §4; re-verify when the clones move (tracking step 4
there).

## 1. etcd (verified at `v3.7.3`)

Go multi-module workspace (`go.work`; `go 1.26.0`, toolchain `go1.26.10`).
Modules: `api`, `cache`, `client/pkg`, `client/v3`, `etcdctl`, `etcdutl`,
`pkg`, `server`, `tests`, `tools/mod`, `tools/rw-heatmaps`,
`tools/testgrid-analysis`.

Important dependencies **not** in this repo (from `server/go.mod`):
raft `go.etcd.io/raft/v3` v3.7.0 (repo `etcd-io/raft`), storage engine
`go.etcd.io/bbolt` v1.5.0 (repo `etcd-io/bbolt`), `google.golang.org/grpc`
v1.83.2. Their sources are not checked out, and with no Go installed (`AGENTS.md` §7)
there is no module cache.

| Area | Path (under `~/github/etcd`) |
|---|---|
| Server core, raft integration, request pipeline | `server/etcdserver/` |
| Apply path (state-machine application of committed entries) | `server/etcdserver/apply/` |
| gRPC API handlers | `server/etcdserver/api/v3rpc/` |
| Lock / election services (server side) | `server/etcdserver/api/v3lock/`, `server/etcdserver/api/v3election/` |
| MVCC store, watch implementation | `server/storage/mvcc/` |
| bbolt backend wrapper (batching, commit interval) | `server/storage/backend/` |
| Write-ahead log | `server/storage/wal/` |
| Leases | `server/lease/` (details in `AGENTS.md` §8) |
| Feature gates | `server/features/etcd_features.go` |
| Auth | `server/auth/` |
| gRPC proxy | `server/proxy/grpcproxy/` |
| Embedding / config | `server/embed/` |
| Go client; locks, elections, sessions, STM | `client/v3/`, `client/v3/concurrency/` |
| Client-side leasing cache | `client/v3/leasing/` |
| Experimental client cache library (needs `RequestProgress`; not via gRPC proxy) | `cache/` |
| Benchmark tool (`put`, `range`, `txn_*`, `watch*`, `lease`, `stm`, `mvcc*`) | `tools/benchmark/` |
| Read/write heatmap tooling | `tools/rw-heatmaps/` |
| Linearizability / watch-guarantee robustness tests with failpoints (porcupine-based) | `tests/robustness/` |
| Antithesis (deterministic-simulation) test setup | `tests/antithesis/` |
| E2E / integration tests | `tests/e2e/`, `tests/integration/` |
| Postmortems | `Documentation/postmortems/` (currently `v3.5-data-inconsistency.md`) |
| Threat model | `THREAT_MODEL.md` |

User-facing etcd documentation lives in a separate repo (`etcd-io/website`,
served at https://etcd.io/docs/), not in this checkout.

## 2. jetcd (verified at `jetcd-0.8.7`)

Gradle build (`./gradlew`), compiles with `options.release = 11`; gRPC
`1.82.0`, Vert.x `5.1.2` (`gradle/libs.versions.toml`). Modules: `jetcd-api`,
`jetcd-common`, `jetcd-core`, `jetcd-grpc`, `jetcd-ctl`, `jetcd-launcher`,
`jetcd-test`.

- Public API interfaces: `jetcd-core/src/main/java/io/etcd/jetcd/`
  (`KV`, `Watch`, `Lease`, `Lock`, `Election`, `Txn`, `Cluster`, `Auth`,
  `Maintenance`); implementations in `.../impl/`.
- `jetcd-launcher` starts etcd in containers (Testcontainers). Its default image
  is `quay.io/coreos/etcd:v3.5.14`
  (`jetcd-launcher/src/main/java/io/etcd/jetcd/launcher/Etcd.java:31`), two
  minor versions behind the target, so jetcd's own tests do not exercise the
  target server by default (itself a production-readiness data point). Tests
  using the launcher must state the etcd version used.
