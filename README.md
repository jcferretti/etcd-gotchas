# etcd-gotchas

AI-generated analyses of the production-readiness and performance of
[etcd](https://github.com/etcd-io/etcd) and its Java client
[jetcd](https://github.com/etcd-io/jetcd), with Google's Chubby paper as the
design reference point. Independent work, not affiliated with the etcd project.

Start at [`analysis/INDEX.md`](analysis/INDEX.md).

- Different AI tools analyzed the same questions separately. The filename
  suffix (`-claude`, `-codex`, `-agy`) names the tool, and files can disagree.
- Each file states the versions it analyzed; later releases may differ.
- Claims carry a confidence level: High, Moderate or Low. The tools' standing
  instructions ask for these. Each level is the model's own judgment, with no
  numeric definition.
- This repository is a periodic snapshot of a private working repository.
