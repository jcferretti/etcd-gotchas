# References

Notes on the reference material in `doc/` (read-only inputs). Moved out of
`AGENTS.md` §6 on 2026-11-01.

## 1. The Chubby paper

`doc/chubby-osdi06.md` (≈617 lines). The conversion inserts page markers as
headings of the form `### --- [Page N] ---`; ignore them for structure but use
them to cite pages (e.g. "Chubby §2.8, p. 7").

Official copies: [USENIX PDF](https://www.usenix.org/legacy/event/osdi06/tech/full_papers/burrows/burrows.pdf),
[Google Research](https://research.google/pubs/the-chubby-lock-service-for-loosely-coupled-distributed-systems/).
In the public copy of this repository, `doc/chubby-osdi06.md` is a placeholder;
the full text is not republished there.

**Citing rule:** give the section in the same clause as any line of the
conversion, e.g. "Chubby §2.8, `doc/chubby-osdi06.md:263`". The public copy has
no conversion, so a line number alone cannot be resolved there, and a bare
`:263` does not even say what it refers to. `p. N` counts PDF pages from 1; the
USENIX copy prints 335–350 on those pages (p. 7 is printed 341).

Section index: 2.1 Rationale · 2.2 System structure · 2.3 Files, directories,
handles · 2.4 Locks and sequencers · 2.5 Events · 2.6 API · 2.7 Caching ·
2.8 Sessions and KeepAlives · 2.9 Fail-overs · 2.10 Database implementation ·
2.11 Backup · 2.12 Mirroring · 3 Scaling (3.1 Proxies, 3.2 Partitioning) ·
4 Use, surprises and design errors (4.2 Java clients, 4.3 Name service,
4.4 Fail-over problems, 4.5 Abusive clients, 4.6 Lessons learned).

Natural comparison axes (starting points, not conclusions):

| Chubby | etcd counterpart to examine |
|---|---|
| Sessions + KeepAlives, grace period (§2.8, §2.9) | Leases + `LeaseKeepAlive` stream; lease behavior across leader change |
| Locks + sequencers (§2.4) | `concurrency.Mutex`, lock service; revisions / lease IDs as fencing tokens |
| Events (§2.5) | Watch API, revisions, compaction interaction |
| Client cache with server-driven invalidation (§2.7) | No server-driven invalidation; `client/v3/leasing`, `cache/` |
| Proxies, partitioning (§3) | gRPC proxy, learners; etcd has no built-in partitioning |
| Abusive clients, Java clients (§4.2, §4.5) | Quotas, rate limits, request size limits; jetcd |
| Database / backup (§2.10–2.11) | bbolt + WAL + snapshots; `etcdutl` snapshot/restore, defrag |
