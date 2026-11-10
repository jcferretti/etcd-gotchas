# Change log

Project change log, moved out of `AGENTS.md` §12 on 2026-11-01. Append new
entries at the end. A bare `§N` means a section of `AGENTS.md`; "its §N"
means a section of the analysis file the entry names. Every worktree is
recorded here (`AGENTS.md` §9, rule 1).

- 2026-10-31: `AGENTS.md` created. Environment and repository state recorded.
- 2026-10-31: Decisions: target latest stable etcd and jetcd; jetcd is an
  evaluation subject; checkouts read-only, changes via worktrees; benchmarks on
  other hardware; one analysis file per feature/bug; no git repo yet; user's
  communication rules copied into §2; leases is the first workstream.
- 2026-10-31: Clones moved from `main` to stable tags: etcd `main@1573d59416ef`
  → `v3.7.3`; jetcd `main@9f509770201e` → `jetcd-0.8.7`. Fetch showed no newer
  upstream commits.
- Worktrees: none.
- 2026-11-01: Added storage-codex.md; static only, no tests or checkout changes.
- 2026-11-01: Added and audited `analysis/wal-replacement-codex.md`; peer
  ideas and source/issue corrections recorded in its §13. Static only; checkouts unchanged.
- 2026-10-31: Added and expanded `analysis/leases-codex.md`: renewal promises,
  checkpoint persistence, historical fixes, and an untested delayed-revocation
  schedule (its §3.4). Static only; stable releases and checkouts unchanged.
- 2026-10-31: Added `analysis/leases-claude.md`, independent of
  `leases-codex.md`. No tests; releases unchanged; raft `v3.7.0` `raft.go`
  fetched read-only.
- 2026-10-31: Added `analysis/leases-agy.md` (independent of the other two
  analyses). No tests run; checkouts unchanged (`v3.7.3` / `jetcd-0.8.7`).
- 2026-11-01: Added `analysis/storage-agy.md` evaluating replacing `bbolt`
  (`v1.5.0`) with `pebble` (`v2.1.7`) or `grocksdb` (`v1.11.1`) against etcd
  `v3.7.3`. No tests or benchmarks run; checkouts unchanged.
- 2026-11-01: Added `analysis/storage-claude.md` and
  `analysis/wal-replacement-claude.md`, written independently; the latter was
  then revised using codex/agy ideas (its §16). Static only.
- 2026-11-01: Added `analysis/wal-replacement-agy.md` (independent analysis of
  replacing both the Raft WAL and `bbolt` in etcd `v3.7.3`). Static only;
  checkouts unchanged.
- 2026-11-01: Git repository created; first commit `e1bfcc9` pushed to
  `origin/main`.
- 2026-11-01: Shortened: removed the copied user rules from §2 (each tool
  has them); moved orientation maps to `orientation_map.md`, Chubby notes
  to `references.md`, the index to `analysis/INDEX.md` (adds watchers-*).
- 2026-11-01: Moved this log out of `AGENTS.md` §12 into `log.md`;
  `AGENTS.md` §12 keeps the maintenance rules and points here.
- 2026-11-01: `AGENTS.md` §9 rule 8: agents may run concurrently; shared
  files change only on instruction or after asking; each tool solely owns
  its `analysis/*-<suffix>.md` files.
- 2026-11-01: `analysis/leases-claude.md` revised using leads from
  `leases-codex.md` and `leases-agy.md`, each re-verified (its Appendix C.2);
  then restructured for readability (Scope, Summary, Conclusions,
  appendices; old→new section map in its Appendix C.1); §2.5 added after
  Antigravity disputed one rejection. Static: raft `v3.7.0` `raft.go` and two
  Kubernetes `v1.37.2` files fetched read-only; GitHub API used; checkouts
  unchanged.
- 2026-11-01: Added `analysis/leases-critique-claude.md` (peer claims Claude
  rejected or qualified, with reasons). `analysis/watchers-claude.md`:
  cross-reference to `leases-claude.md` updated (§12 → §5.2).
- 2026-11-01: `analysis/leases-codex.md` and `analysis/leases-agy.md` revised
  by their tools after cross-reading the peer lease analyses, then
  restructured; each file records its own changes. Neither reports running
  tests.
- 2026-11-01: `analysis/INDEX.md` updated for the four lease files.
- 2026-11-01: `analysis/watchers-claude.md` revised using leads from
  `watchers-codex.md` and `watchers-agy.md`, each re-verified (its Appendix
  E.2; rejected claims in E.3). New: Go client final-error drop and #17611
  teardown deadlock; proxy dead-broadcast, swallowed-error and
  `RequestProgress` findings; jetcd 4 MiB response limit; #22184
  reachability downgraded. Then restructured for readability (Scope,
  Summary, Conclusions, Appendices A–E; old→new section map in its
  Appendix E.1). Static: GitHub API; Kubernetes `v1.37.2` `etcd3/watcher.go`
  re-fetched read-only; one grpc-java constant read with `javap` from the
  local Gradle cache. No tests; checkouts unchanged.
  `analysis/leases-claude.md`: four cross-references to `watchers-claude.md`
  updated (§13.4/§13.5 → §9.1/Appendix C).
- 2026-11-01: `analysis/watchers-codex.md` expanded by Codex after reviewing
  the peer watch analyses (its Appendix D), then restructured (Scope,
  Summary, Conclusions; section map in its Appendix A.4). It ran two small
  Docker functional experiments with the official `v3.7.3` image (its
  Appendix C; containers and networks removed afterwards); no benchmarks.
  It re-ran `git fetch --tags --prune` in both clones: no newer stable
  release; checkouts unchanged.
- 2026-11-01: `analysis/watchers-agy.md` revised by Antigravity after
  cross-validating the peer watch analyses, then restructured for clarity
  (Scope, Summary, Conclusions). It does not report running tests.
- 2026-11-01: `analysis/INDEX.md` updated for the three watch files.
- 2026-11-01: Added `analysis/chubby-perspectives-claude.md` and
  `analysis/chubby-perspectives-codex.md`, written independently: would
  Chubby's per-session KeepAlive design prevent or simplify the etcd lease
  and watch issues? Static only; checkouts unchanged.
- 2026-11-01: Added `skills/clarity-of-exposition.md`, a reusable
  instruction for a clarity pass on an analysis file (Scope, Summary and
  Conclusions sections; reorganize and reword without dropping data or
  references).
- 2026-11-02: `analysis/chubby-perspectives-claude.md` revised using leads
  from `chubby-perspectives-codex.md` and `chubby-perspectives-agy.md`, each
  re-verified (its Appendix B), then reorganized with
  `skills/clarity-of-exposition.md` (Scope, Summary, Terms, Conclusions;
  method and sources moved to its Appendix A). Functional test of
  `client/v3/leasing`: official `v3.7.3` image, one member; Go client
  `v3.7.3` built in a `golang:1.26` container; three runs; counts of
  proposals, revisions and responses only, no timings, not a benchmark
  (§7). Containers removed afterwards. GitHub API reads; etcd
  `origin/main` `a2e3642bc` read for `client/v3/leasing`. Checkouts
  unchanged; no worktrees.
- 2026-11-02: Added `analysis/chubby-perspectives-critique-claude.md`:
  Claude's objections to `chubby-perspectives-agy.md` at SHA-256 prefix
  `90650a470cdc9831` (the 09:44 version), written for Antigravity to use
  when correcting its file. Its Appendix A holds the program and output of
  the functional test above.
- 2026-11-02: `analysis/chubby-perspectives-codex.md` expanded by Codex
  after checking leads from snapshots of the Claude and Antigravity files
  (SHA-256 prefixes and adopt/narrow/reject decisions in its Appendix A).
  No functional tests. It re-fetched tags in both clones: no newer stable
  release; checkouts unchanged.
- 2026-11-02: `analysis/chubby-perspectives-agy.md` (Antigravity; creation
  date not recorded) revised several times; the 10:09 version cites
  `chubby-perspectives-critique-claude.md` and the two peer files. It does
  not report running tests; its claims of `client/v3/leasing`
  reproductions cite the critique's Appendix A.
- 2026-11-02: `analysis/INDEX.md` updated for the four
  `chubby-perspectives-*` files; prefix list and critique-file note
  extended.
- 2026-11-02: `AGENTS.md` §3: added `skills/` to the directory layout.
- 2026-11-02: Removed the analysis "status" field (draft / in review /
  final) at the user's request: it was never defined and in practice
  reflected each tool's habit. Removed from `AGENTS.md` §5 (required header
  fields) and §10, from the `analysis/INDEX.md` table, and from the header
  of all 17 analysis files. Where a Status line also carried other
  information, that text was kept under a new label: `History:` (revision
  notes; `chubby-perspectives-agy`, `chubby-perspectives-claude`,
  `leases-agy`, `watchers-agy`), `Method:` (`storage-claude`,
  `storage-codex`, `wal-replacement-claude`, `wal-replacement-codex`),
  `Date:` (`chubby-perspectives-critique-claude`). One in-body mention
  ("both draft") removed from `chubby-perspectives-claude.md`. No other
  content changed, but file hashes did: the current
  `chubby-perspectives-agy.md` no longer matches its 10:09 version
  byte for byte.
- 2026-11-02: Added `analysis/etcd-issues-and-mitigations-claude.md`
  (Claude): a concise brief for prospective production users on lease,
  watch and write-performance risks and deployment mitigations. It is
  synthesized from the seven `*-claude.md` files: issue tables LS1–LS9,
  WT1–WT10 and WR1–WR8, cross-cutting risks, misconceptions,
  pre-deployment questions, and an appendix of pointers. No new code
  analysis, except that recommended flag names, feature-gate names and
  defaults were re-checked at `v3.7.3`. Nothing run. A tag fetch found no
  newer stable release; checkouts unchanged; no worktrees.
- 2026-11-02: The same file's §3 extended at the user's request. It
  disambiguates the watch delivery disclaimer ("events in unhealthy
  clusters might never arrive"):
  - "unhealthy" is undefined, and etcd uses it for unrelated conditions;
  - the disclaimer has effect only with the quorum intact;
  - so the contract is safety-only, and it contradicts etcd's claim of
    eventual consistency for watches.

  Re-read for it: the v3.7 API guarantees page (`etcd-io/website` `main`)
  and, at `v3.7.3`, `etcdctl/ctlv3/command/ep_command.go`,
  `server/etcdserver/server.go` and `tests/robustness/README.md`.
- 2026-11-02: Caveats added to `analysis/watchers-claude.md` (Summary,
  §2.2, §12.2, §15.2, Appendix E.1) and `analysis/chubby-perspectives-claude.md`
  (§6.1, History). Each points to `etcd-issues-and-mitigations-claude.md`
  §3. No findings or confidence ratings changed.
- 2026-11-02: Added `analysis/etcd-issues-and-mitigations-codex.md`
  (Codex): a short brief from the five `*-codex.md` analyses, with
  risk-and-action tables for leases, watches and caches, and write
  headroom. Upstream issue states rechecked 2026-11-02; no tests or
  benchmarks run (per the file).
- 2026-11-02: Added `analysis/etcd-issues-and-mitigations-agy.md`
  (Antigravity): an adopter guide from the five `*-agy.md` analyses, with
  per-area open-bug tables, staged write mitigations, adopter questions and
  a verified-vs-inferred matrix. It does not report running tests.
- 2026-11-02: `analysis/watchers-agy.md` edited by Antigravity. The
  delivery disclaimer is recast as silent staleness in a cluster with
  quorum intact, with three scenarios: a minority partition, a follower
  apply stall, and the proxy's dead broadcast (#21813). Its API-guarantees
  citations now point to `etcd-io/website`. Its `History:` line was not
  updated.
- 2026-11-02: `analysis/INDEX.md` updated: rows for the three
  `etcd-issues-and-mitigations-*` files; notes on the 2026-11-02 edits to
  `watchers-claude.md`, `chubby-perspectives-claude.md` and
  `watchers-agy.md`; the prefix list extended.
- 2026-11-03: At the user's request, so the files can move to another
  repository: removed the project's checkout path and its remote's host and
  URL from `AGENTS.md` §3 (the SSH rule kept, worded generically) and
  reworded the 2026-11-01 repository-creation entry above.
- 2026-11-03: `analysis/watchers-claude.md` premise verdict revised after
  the user's review, from "mostly wrong" to "right about liveness, wrong
  about safety". The earlier verdict counted only the six safety guarantees
  as promises; etcd's eventual-consistency claim
  (`tests/robustness/README.md:102`, `:111` @ `v3.7.3`) makes eventual
  delivery one too. Its promise categories became safety guarantee /
  eventual delivery / timeliness (§4 relabelled). The misconception row in
  `etcd-issues-and-mitigations-claude.md` §6 corrected to match.
- 2026-11-03: `analysis/watchers-claude.md` second revision, after the user
  asked whether `WithRequireLeader` restores liveness. It does not: a new
  Summary bullet lists the paths that stay silent with it set. New finding
  (§5.2; mechanism from code, not reproduced): a blocked raft loop, for
  example a WAL `fdatasync` that does not return, freezes the member's
  leader view, so neither `WithRequireLeader` nor client keepalive ends its
  watches. Codex reached the same counterexample independently
  (`watchers-codex.md` §2.6). `watchers-agy.md` §3.1.2 was checked against
  its §6.3. Also changed: §13.2, §14.1 and §15; and, in
  `etcd-issues-and-mitigations-claude.md`, WT2, the freshness-monitor
  mitigation and the "`WithRequireLeader` makes watches safe" row. Static:
  code read at `v3.7.3`; GitHub API for #17611 and one issue search. No
  tests; checkouts unchanged; no worktrees.
- 2026-11-03: `analysis/watchers-agy.md` and
  `analysis/etcd-issues-and-mitigations-agy.md` edited by Antigravity. Both
  recast the watch findings around the contradiction between etcd's
  eventual-consistency claim and "events ... might never arrive" in a
  cluster with quorum; the watch file keeps its three scenarios and adds
  the client-design liveness requirement; the brief recommends periodic
  liveness polling or watch rotation. Neither `History:` line was updated.
  Neither reports running tests.
- 2026-11-03: `analysis/watchers-codex.md` and
  `analysis/etcd-issues-and-mitigations-codex.md` revised by Codex (its
  §A.5): §1.5 treats eventual consistency as eventual progress, with sourced
  quotations; §2.6 shows silence despite `WithRequireLeader`, including the
  WAL-stall member above; §7.2 adds a progress-supervision watchdog; the
  brief adds a watch liveness gap and independent progress supervision.
  No runtime tests. Its tag fetch and release checks found no newer stable
  release; checkouts unchanged.
- 2026-11-03: `analysis/INDEX.md` updated for the six files above.
- 2026-11-05: `AGENTS.md`: private host details removed (§3, §7); `gh`
  `2.103.3` and Go `1.27.3` recorded (§7), with the `GOTOOLCHAIN=go1.26.10`
  caveat for building etcd; the §11 question about installing Go closed. Go
  smoke test passed: compile, `vet`, race detector, cgo and static build; etcd
  `v3.7.3` `etcd` and `etcdctl` built under go1.27.3 and go1.26.10;
  single-member put/get and lease grant/revoke; `server/lease` tests with
  `-race` on go1.26.10. The build used a `git archive` export, so no worktree
  was created.
- 2026-11-05: Public GitHub copy prepared; nothing published yet. Added
  `README.md` and `publish/`. `publish-github.sh` builds each snapshot in a
  temporary repository holding no history from here, replaces
  `doc/chubby-osdi06.md` with a placeholder (the paper's copyright is not ours
  and no license allowing redistribution was found), leaves out `publish/`,
  scans the result against `denylist.txt`, and commits with the GitHub noreply
  address. Tested against local bare repositories only. Rules in `AGENTS.md`
  §3.
- 2026-11-05: Chubby citation pass, at the user's request, so that citations
  resolve in the official PDF: the public copy has no conversion, so its line
  numbers mean nothing there. Audit: 310 line references in 10 files; page
  markers match both PDFs (236 of 242 testable lines on the marked page, none
  on another page). 92 references in 6 files gained a section label, each read
  against the cited paper text. Corrections: wrong sections in
  `chubby-perspectives-claude.md:298,328`; wrong lines in
  `chubby-perspectives-agy.md:153,172,309,444,446` and `leases-agy.md:174,179`;
  a misquote in `chubby-perspectives-critique-claude.md:402`. No claims changed
  and no lines were added or removed. Claude Code edited the `-agy` and
  `-codex` files on the user's instruction (`AGENTS.md` §9 rule 8). `History:`
  lines not updated. `references.md` gained the official links and a citing
  rule, also pointed to from `AGENTS.md` §5; `analysis/INDEX.md` needed no
  change.
- 2026-11-05: Public GitHub copy published by the user:
  `jcferretti/etcd-gotchas`, public, one commit (`7805a6e`, a snapshot of
  `3c4fd57`). Verified from an anonymous clone: 29 files, the paper placeholder
  in place, `publish/` absent, the noreply address on the commit, every other
  file identical to `origin`, and no denylisted text in any object. To update
  it, push to `origin`, then run `publish/publish-github.sh --push`.
- 2026-11-06: `~/.codex/AGENTS.md` and `~/.gemini/GEMINI.md` installed as copies
  of `~/.claude/CLAUDE.md`, so all three tools load the user's rules; loading
  verified for both (`codex debug prompt-input`; a one-shot `agy --print` run).
  Copy them again after editing `CLAUDE.md`.
- 2026-11-06: Confidence review, at the user's request: Codex and Antigravity
  reviewed their 12 `-codex` and `-agy` files and added or revised claim-level
  High / Moderate / Low tags. Not only tags: some claims were reworded to
  separate verified mechanisms from inference, several levels went from High
  to Moderate (`wal-replacement-codex.md`,
  `etcd-issues-and-mitigations-codex.md`), and the fence proposed at
  `leases-agy.md:550` now compares `proposal.LeaderTerm < entry.Term`. The
  `-agy` files also gained tags in headings and confidence columns in tables.
  Claude Code checked structure only, not the ratings: table columns, internal
  anchors, Chubby citation counts, line references from other files, and the
  denylist. On the user's instruction it also shortened the `Confidence review`
  line in `storage-codex.md` and `wal-replacement-codex.md`.
- 2026-11-07: Second confidence pass, at the user's request: Antigravity
  removed the High tags on settled facts from its six `-agy` files, including
  most of those the first pass had added to headings and table rows.
  Occurrences of "High" fell from 525 to 62 (90 before the first pass);
  Moderate (98) and Low (15) were kept, some moved inline from removed table
  columns, and the levels in each Summary are unchanged. Line counts are
  unchanged; headings with a tag fell from 78 to 9. Evidence pointers that the
  first pass had put inside tags went with them; no other citation was lost.
  One rename: `wal-replacement-agy.md:190` "MVCC Engine" became "State
  Machine". Claude Code checked tables, Chubby citations, Summary levels and
  the denylist, not the ratings.
- 2026-11-08: New question, analyzed independently by all three tools:
  should one build an etcd-compatible replacement from scratch, in C++ or
  Rust, on ClickHouse's NuRaft fork and RocksDB, written mostly by coding
  agents; and should it keep gRPC wire compatibility (A) or only
  language-level client API compatibility (B)? Files:
  `analysis/nuraft-rocksdb-rewrite-claude.md` (2026-11-08; written without
  reading the Codex file), `-codex.md` (2026-11-08) and `-agy.md`
  (2026-11-09). All three recommend A, and none reports building or running
  anything. They disagree on language: Antigravity says C++20 (High),
  Claude says Rust with a thin C++ shim (Moderate–Low), and Codex says
  either, depending on the team. Their estimates for a production candidate
  with the full or broad API differ by up to an order of magnitude:
  Antigravity 14–22 engineer-months, Claude 30–60, Codex 80–160 (Low
  confidence in Claude's and Codex's estimates; Antigravity gives Low to
  Moderate for its schedule). Not reconciled. New read-only source clones,
  not yet listed in `AGENTS.md` §4: `~/github/NuRaft` (ClickHouse/NuRaft
  `master` `e937522`), `~/github/rocksdb` (meta/rocksdb tag `v11.13.2` =
  `17b624b5c`), `~/github/jepsen-etcd` (jepsen-io/etcd `master` `b47cc40`).
  No worktrees were created.
- 2026-11-08 and 2026-11-09: Jepsen pass. All three tools extended their
  `leases-`, `watchers-` and `etcd-issues-and-mitigations-` files with the
  Jepsen reports on etcd 3.4.3 (2020) and jetcd 0.8.2 (2024), and checked
  the state of their fixes at `v3.7.3` and `jetcd-0.8.7`.
  - Codex (2026-11-08, `leases-codex.md` §5.6 and §A.4,
    `watchers-codex.md` §1.6, §6.5 and §A.6): ownership checks and recovering
    a lost write's outcome are separate problems; jetcd #1445 fixes KV
    retries only; jetcd 0.8.7 stops renewing after a clean keepalive stream
    end (fix #1628).
  - Claude (2026-11-09, `leases-claude.md` §2.3, §5.4 and §8,
    `watchers-claude.md` §2.5, brief LS8 and LS10): Jepsen measured about
    18 % of acknowledged updates lost under unfenced locks and none under
    lock-guarded transactions. Its watch test rotated each watch every
    0–5 s and checked only after healing, so it could not see silent
    watches. Lease and lock retries remain in jetcd 0.8.7; the lock retry
    race is inferred (Low).
  - Antigravity (2026-11-09, `leases-agy.md` §4.4–4.5, `watchers-agy.md`
    §2.2.2 and §3.2): jetcd lease and lock retries after #1445; says
    `Election.Campaign` lacks the post-wait ownership check that #11456
    added to `Mutex.Lock`; revision-0 watch doc defects that remain. Its
    `History:` lines now record this update, but not the edits of
    2026-11-02 and 2026-11-03.
  None of these edits reports running a build, test or benchmark.
- 2026-11-09: Maintenance and governance review in the three
  `etcd-issues-and-mitigations-` briefs: Claude §5 "Project scope and
  maintenance capacity", Codex "Adoption risk: maintenance coverage and issue
  resolution" and "Governance history and current responsibilities",
  Antigravity §4 with three case studies. All three trace the handling of
  Jepsen's reports (etcd #14092, jetcd #1072, etcd #14890, #18424): closed in
  2024 as "inaccurate", then partly accepted once Jepsen published.
- 2026-11-09: jetcd `jetcd-0.8.8` released: tag = `a2909b8`, GitHub
  release 2026-11-09T06:40Z, Maven Central `<release>0.8.8</release>`
  (checked by Claude Code). etcd is still at `v3.7.3`. The jetcd clone stays
  at `jetcd-0.8.7` until the user confirms the move (`AGENTS.md` §4, step 2).
  The Claude and Codex briefs say they do not analyze 0.8.8; the Antigravity
  files cite both 0.8.7 and 0.8.8.
- 2026-11-09: `analysis/INDEX.md` updated for the twelve files above, at
  the user's request: three new rows, notes on nine rows, the new prefix,
  and a paragraph on the Jepsen pass. A denylist scan of the working tree
  (the publish script's `grep -i -F`, paper excluded) found nothing.
