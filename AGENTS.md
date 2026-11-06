# AGENTS.md — etcd production-readiness & performance analysis

Instructions and shared context for any coding/research agent (Claude Code,
Codex, Antigravity, others) working in this directory. Read this whole file
before starting work. It and the files it links are the single source of
truth; tool-specific files (e.g. `CLAUDE.md`) only point here.

## 1. Project purpose and scope

Analyze the **production-readiness** and **performance** of
[etcd](https://github.com/etcd-io/etcd), the strongly consistent (linearizable)
distributed key-value store whose primary production use is as the backing
store for Kubernetes, and of its Java client library
[jetcd](https://github.com/etcd-io/jetcd). Google's Chubby paper
(`doc/chubby-osdi06.md`) is the design reference point for comparison.

Scope decisions (2026-10-31):
- **Target versions: latest stable releases** of etcd and jetcd (§4). `main`
  is consulted only to check whether an issue is already fixed or changing
  upstream; say so explicitly when citing it.
- **jetcd is a subject of evaluation** as a production client library (API
  coverage, failure/retry handling, lease/watch/lock semantics, performance vs.
  the Go client), not merely a load generator.
- **Work is organized as individual analyses of specific features and bugs**,
  one Markdown file each, in `analysis/` (§5).
- **Benchmarks run on other hardware later.** This host is for code analysis
  and small functional tests only (§7).

Remaining open decisions are in §11; do not silently resolve them.

## 2. Communication and accuracy rules

Not repeated here: each tool loads the user's rules from its own
configuration (the copy formerly in this section was removed 2026-11-01).
Analysis files in `analysis/` are deliverables: write them completely (the
brevity rules do not apply), but keep the accuracy rules, including confidence
tags and verified-vs-recalled distinctions.

## 3. Directory layout

| Path | Contents |
|---|---|
| `AGENTS.md` | This file. Shared agent instructions; grows over time. |
| `CLAUDE.md` | One-line shim (`@AGENTS.md`) so Claude Code loads this file. Do not put content there. |
| `README.md` | Landing page of the public GitHub copy. Keep it minimal. |
| `doc/` | Reference material (read-only inputs). |
| `doc/chubby-osdi06.md` | Markdown conversion of Burrows, *The Chubby lock service for loosely-coupled distributed systems*, OSDI 2006. |
| `orientation_map.md` | etcd and jetcd source-tree maps (§4.1). |
| `references.md` | Notes on the `doc/` reference material (§6). |
| `log.md` | Project change log (§12). |
| `analysis/` | One Markdown file per analyzed feature or bug (§5). |
| `analysis/INDEX.md` | Index of the analysis files (§10). |
| `publish/` | Tooling for the public GitHub copy: `publish-github.sh`, `denylist.txt` and the Chubby paper placeholder. Not part of that copy. |
| `skills/` | Reusable task instructions in plain Markdown, applied when the user names one, e.g. `clarity-of-exposition.md` (a clarity pass on an analysis file: Scope, Summary and Conclusions sections; reorganize and reword without dropping data or references). |

Git: commit or push only when the user asks. For SSH remotes, set
`SSH_AUTH_SOCK=~/.run/ssh_auth_sock` and check that `ssh-add -l` lists
a key first: repeated authentication failures can get this host banned by the
Git server.

`origin` is private. The public GitHub copy receives snapshots only, built by
`publish/publish-github.sh` from commits already on `origin/main`: no history,
the Chubby paper replaced by a placeholder, `publish/` left out. Publish only
when the user asks. Never push this repository to GitHub directly or add GitHub
as a remote: the history and the paper must stay private. Text matching
`publish/denylist.txt` in any other file blocks publishing, so keep private
details out of tracked files.

## 4. Source repositories and version tracking

**Declaration of intent: this project tracks the latest stable releases of
etcd and jetcd.** The local clones are kept checked out (detached HEAD) at the
latest stable release tag, and are moved forward when a new stable release
appears.

| Repo | Local path | Upstream | Checked out (2026-10-31) |
|---|---|---|---|
| etcd | `~/github/etcd` | https://github.com/etcd-io/etcd | tag `v3.7.3` = `0a173d0410a9` (released 2026-10-22) |
| jetcd | `~/github/jetcd` | https://github.com/etcd-io/jetcd | tag `jetcd-0.8.7` = `23b61f1499a5` (commit 2026-06-17; GitHub release and Maven Central 2026-08-27) |

Notes:
- jetcd tag `jetcd-0.8.8-alpha` points at the same commit as `jetcd-0.8.7`, so
  `git describe` may print the alpha name. Use `git tag --points-at HEAD`.
- jetcd is pre-1.0 (`0.8.x`).
- Upstream branches are readable as remote refs without checkout:
  `git -C ~/github/etcd show origin/main:<path>`,
  `git -C ~/github/etcd log v3.7.3..origin/release-3.7 -- <path>` (unreleased
  3.7 backports), `git log v3.7.3..origin/main -- <path>` (upstream changes).
- When citing code, give `path:line` **plus the tag or commit**. Confirm with
  `git -C ~/github/<repo> rev-parse --short HEAD` before citing.

**Definition of "latest stable":** the highest-versioned tag with no
pre-release suffix — etcd `vX.Y.Z`, jetcd `jetcd-X.Y.Z` — cross-checked
against GitHub's latest release
(`curl -s https://api.github.com/repos/etcd-io/<repo>/releases/latest`) and,
for jetcd, Maven Central
(`https://repo1.maven.org/maven2/io/etcd/jetcd-core/maven-metadata.xml`).

**Tracking procedure:**
1. At the start of a session, check for a newer stable release:
   `git -C ~/github/<repo> fetch --tags --prune origin`, then
   `git tag -l 'v*' --sort=-v:refname | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$' | head -1`
   (jetcd: pattern `'jetcd-*'`, regex `'^jetcd-[0-9]+\.[0-9]+\.[0-9]+$'`).
2. If a newer one exists, **tell the user and get confirmation before moving
   the clone**: in-progress analyses or concurrent sessions in other tools may
   depend on the current checkout.
3. To move: verify `git status --porcelain` is empty, then
   `git switch --detach <tag>`.
4. After moving: update the table above, re-verify the paths, versions, and
   line numbers in `orientation_map.md` and §8, and add a `log.md` entry.
5. Existing analyses stay pinned to the versions they state; re-validating an
   analysis against a new release is a separate task, recorded in
   `analysis/INDEX.md`.

### 4.1 Orientation maps

Maps of the etcd and jetcd source trees (modules, dependencies, where each
subsystem lives) are in [`orientation_map.md`](orientation_map.md). Note from
there: `jetcd-launcher` defaults to etcd `v3.5.14`, so any test using it must
state the etcd version actually used.

## 5. Analysis files (`analysis/`)

- **One Markdown file per feature or bug analysis.** Flat directory, kebab-case,
  feature first so files group by feature: `<feature>-<topic>.md`, e.g.
  `leases-design-vs-chubby-sessions.md`, `leases-bug-<short-slug>.md`,
  `leases-perf-<short-slug>.md`. When several tools analyze the same
  question, each file ends in its tool's suffix (`-claude`, `-codex`,
  `-agy`; see `analysis/INDEX.md`).
- Each file starts with a header block:
  - **Versions analyzed:** etcd tag + commit; jetcd tag + commit if relevant.
  - **Upstream references:** etcd/jetcd issue and PR numbers, with links.
  - **Summary:** the claims, each with its confidence (High / Moderate / Low).
- Body: evidence with `path:line @ tag` citations, what was verified vs.
  inferred, the conditions under which each conclusion would fail, and open
  questions.
- Cite the Chubby paper by section as well as by line (`references.md` §1).
- Add every new file to [`analysis/INDEX.md`](analysis/INDEX.md) with a
  one-line summary.

## 6. Reference material

Notes on `doc/chubby-osdi06.md` (official copies, citing rule, page-marker
citation convention, section index, Chubby-to-etcd comparison axes) are in
[`references.md`](references.md).

## 7. Environment (verified 2026-10-31)

- Host: Linux container. Check capacity at run time (`nproc`, `free -h`,
  `df -h`).
- Installed: Docker `29.9.2`, OpenJDK `17.0.20.1`, system `gradle`, `python3`,
  `curl`, GitHub CLI `gh` `2.103.3` (added 2026-11-05; authenticated, use it
  for issue/PR lookups).
- Go `1.27.3` in `/usr/local/go` (added 2026-11-05, smoke-tested). If `go` is
  not found, `export PATH=/usr/local/go/bin:$PATH`. etcd `v3.7.3` pins
  `go1.26.10`, and the default `GOTOOLCHAIN=auto` silently uses the newer local
  Go instead, so build etcd with `GOTOOLCHAIN=go1.26.10` (downloaded on first
  use) and record the Go version with any result.

**This host is not the benchmark environment** (decision 2026-10-31). Use it
for code reading, building, and small functional/correctness tests (e.g.
Docker clusters, jetcd tests); never report its performance numbers as
findings. etcd write latency is dominated by WAL `fdatasync`, and fsync on
this host's virtualized storage need not resemble production disks. Benchmark
hardware will be added here when available.

## 8. Workstreams

### 8.1 Leases (first workstream, started 2026-10-31)

The user's position: etcd leases have known **bugs** and **performance
problems**, and, based on the Chubby architecture, the user suspects an
**underlying design mistake** in how etcd leases were conceived compared with
Chubby's sessions and KeepAlives. Treat the design claim as a **hypothesis to
test**: state the strongest case for and against it from code and evidence;
do not assume it. Ask the user which specific bugs and performance problems
they already know of before searching from scratch.

Planned outputs: a design comparison file
(`analysis/leases-design-vs-chubby-sessions.md`) plus one file per lease bug or
performance problem.

Starting points (verified at `v3.7.3`):
- Lessor (server lease manager), `server/lease/lessor.go`: `Grant` :281,
  `Revoke` :326, `Checkpoint` :367, `Renew` :396, `Promote` :480 (extends
  leases on leader change), `runLoop` :620, `revokeExpiredLeases` :640;
  `defaultLeaseRevokeRate = 1000` :46; `defaultLeaseCheckpointInterval = 5min`
  :52; `MaxLeaseTTL` :40.
- Server entry points, `server/etcdserver/v3_server.go`: `LeaseGrant` :435,
  `LeaseRevoke` :471, `LeaseRenew` :489 (the leader renews locally after
  `ensureLeadership()`; a follower forwards to the leader over peer HTTP via
  `server/lease/leasehttp/http.go:172` `RenewHTTP`), `LeaseTimeToLive` :699.
- Feature gates `LeaseCheckpoint` and `LeaseCheckpointPersist` are both Alpha,
  default off (`server/features/etcd_features.go:94-95`); config validation
  in `server/embed/config.go:1030-1035`.
- Persistence nuance: `server/lease/lessor.go:385-387 @ v3.7.3` persists
  generated checkpoints when the cluster version is >=3.6, independently of
  the persistence gate. Checkpoint generation still defaults off. The current
  checkpoint-clear error bug (#22649) and historical failover fixes are
  analyzed in `analysis/leases-codex.md`.
- Clients: `client/v3/lease.go`, `client/v3/concurrency/session.go`; jetcd
  `jetcd-core/.../Lease.java` and `impl/`. Lead to examine: the Go client sets
  its local lease deadline to *response receipt time + TTL*
  (`client/v3/lease.go:543`), whereas Chubby's client timeout is deliberately
  conservative (Chubby §2.8, `doc/chubby-osdi06.md:263`).
- Chubby: §2.8 (sessions, KeepAlives), §2.9 (fail-overs, grace period), §4.4.

## 9. Working rules for agents

1. **Source checkouts are read-only.** Never edit files in `~/github/etcd` or
   `~/github/jetcd`. The only permitted change to them is moving to a newer
   stable tag per §4. When code must be built or patched (instrumentation,
   benchmark changes, another version), create a git worktree, e.g.
   `git -C ~/github/etcd worktree add ~/github/worktrees/etcd-<purpose> <tag>`,
   and record every worktree and its purpose in `log.md`. Never push or open PRs
   against `etcd-io/*`.
2. **Cite evidence.** Claims about etcd/jetcd behavior must cite code
   (`path:line @ tag`), tests, upstream issues/PRs, docs, or measurements.
   Distinguish clearly between what was verified and what is recalled or
   inferred.
3. **Version-qualify findings.** State which release(s) a finding applies to;
   if it is fixed or changed on `main` or a release branch, say so with the
   commit/PR.
4. **Record benchmark context** with every number: etcd version/commit, cluster
   size, hardware, storage stack, client (Go `benchmark` / jetcd / other),
   key/value sizes, concurrency, and the exact command.
5. **Prefer existing etcd tooling** (`tools/benchmark`, `tests/robustness`)
   over writing new harnesses, and note why when you do write one.
6. Put temporary/scratch files outside this directory; durable outputs go in
   `analysis/` (§5).
7. Keep this file **tool-agnostic**: plain Markdown only, no tool-specific
   syntax or directives.
8. **Other agents may be running concurrently** in this directory (other
   tools, other sessions). Do not change shared files — everything except
   your own analysis files: `AGENTS.md`, `log.md`, `analysis/INDEX.md`,
   `orientation_map.md`, `references.md`, other tools' analysis files —
   unless the user instructed it or you asked first. This applies to the
   update steps in §5 (`INDEX.md`), rule 1 and §12 (`log.md`, this file):
   ask before doing them. When you do edit a shared file, re-read it
   immediately before writing. Your own files are fixed by suffix: an agent
   running in Claude Code is the only writer of `analysis/*-claude.md`, in
   Codex of `analysis/*-codex.md`, in Antigravity CLI of
   `analysis/*-agy.md`. Create and edit your own files without coordinating
   with other tools; the only concurrent writers can be sub-agents you
   started yourself, and coordinating them is your job.

## 10. Analysis index

[`analysis/INDEX.md`](analysis/INDEX.md): one row per file (feature, versions,
summary) and the meaning of the `-claude`/`-codex`/`-agy` suffixes.

## 11. Open questions / decisions pending

- [ ] Benchmark hardware (to be provided later).
- [ ] Additional lease bugs and performance problems the user already knows of,
  beyond the renewal/Raft/leader-failover concern supplied on 2026-10-31 (§8.1).

## 12. Change log and maintenance

Maintenance rules:
- Update this file when you learn something a fresh session would need (new
  conventions, environment changes, decisions, pointers to analyses). Do not
  paste long findings here; link to files in `analysis/`.
- Record changes (new analyses, decisions, checkout moves, worktrees) as
  entries at the end of [`log.md`](log.md).
- Keep it under **24,000 bytes**: Antigravity documents a per-file rules limit
  of that size and truncates beyond it (moderate confidence; check
  https://antigravity.google/docs/rules). Move detail into separate files
  and link to them if it grows (as done 2026-11-01; see `log.md`).
- Tool compatibility: Codex reads `AGENTS.md` natively; Antigravity reads
  `AGENTS.md` (also `GEMINI.md` and `.agents/rules/` — do not create those, their
  precedence vs. `AGENTS.md` is undocumented); Claude Code loads it through the
  `@AGENTS.md` import in `CLAUDE.md`.
