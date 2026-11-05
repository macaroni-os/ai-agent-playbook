# MFS loop — session handoff (2026-11-05, third refresh)

Read PLAYBOOK.md first — §10 is the macaroni-fs loop, §5/§6/§7/§8 apply as written
(§2–§4 and the kit-fixups parts of §7 do NOT apply to macaroni-fs). Then:

## Board state (verify all of it live before acting — it moves daily)
- Sandbox: Incus container `mfs-dev` — privileged, image `macaroni/phoenix-builder`;
  plain alias launch works, and the image has NO `/root` (mkdir first). Snapshots:
  clean-base, repo-ready, docker-ready, cross-tools-green, cross-tools-green-r2,
  cross-tools-green-r3, **step1-ch6-green** (2026-11-05, after the clean 18/18 chain).
  dockerd runs with `--storage-driver btrfs` and DOES NOT survive container restarts.
  Netfilter modules: the `docker` Incus profile (`security.nesting=true` +
  `linux.kernel_modules: ip_tables,ip6_tables,netlink_diag,nf_nat,overlay,br_netfilter`)
  is attached to `mfs-dev`; incusd modprobes them at every start. Only a launch that
  itself fails with `Failed to load kernel module …` means a host-kernel gap.
- Repo: `/root/macaroni-fs`, branch `step1-ch6-tools`, HEAD `408d876` (5 commits on
  top of merged master `b0341fd`); remotes `upstream` (macaroni-os/macaroni-fs) +
  `fork` (org-tekeli-borisp/macaroni-fs). Build invocation:
  `make build PACKAGES=<cat>/<name> BACKEND=docker REPO_CACHE=`
  — always verify artifacts (rc=0 lies, §10.2/§7 class).
- macaroni-os/macaroni-fs: PR #1/#2/#3 MERGED (PR #3 as merge commit `b0341fd`,
  2026-11-05); **PR #4 OPEN** — head `408d876`
  (https://github.com/macaroni-os/macaroni-fs/pull/4): ch.6 diffutils 3.12,
  file 5.48, findutils 4.11.0, gawk 5.4.1, grep 3.12; one commit per package;
  body carries the clean-slate 18/18 chain + loader smoke + `whip` evidence.
  Awaiting review. Open questions in the body: `coreutils` 6.5 is missing from
  step1 (pre-diffutils in the book), and the next slice size (5–6 tools).
- macaroni-os/ai-agent-playbook: PR #11 MERGED, PR #12 OPEN (this file + §10
  rotation lessons) — this refresh rides it; keep it updated at the end of every
  session (that is the whole point; /tmp drafts died on reboots before).
- macaroni-os/anise#36 OPEN — maintainer undecided between `--all-pkgs-tree` and
  leaving it; watch the thread, don't build on `--all`.
- Rotation probe before EVERY chain run (§10.7 recipe): base image
  `macaronios/terragon-minimal-devel:latest-amd64` was
  `sha256:5f7e82be3b9bc9f0f773e9e61c817161ee7d360f9fc42313d487bf8a9f31591b`
  on 2026-11-05 (previous `ea4a393c…`). It still ships NO grep/awk/find/gzip/
  make/patch/tar/xz/file/diff, but does ship `/tmp` (777), the canonical
  `05gcc-x86_64-pc-linux-gnu.conf`, and libstdc++/libgcc_s resolving through it —
  seed `+14` remains self-sufficient.
- **Cache-masking lesson (new, 2026-11-05)**: anise-build reuses local
  `anise-build/cache:*` images whenever the package spec/tree hash matches, even
  when the base image digest changed. A chain run can therefore silently verify
  the OLD rotation. For any "full chain from seed" claim, run
  `docker system prune --all --volumes -f` first and quote the digest actually
  used (`FROM …@sha256:…` in the build log / `docker image inspect`). Reusing the
  cache is fine for a quick spec iteration, never for the PR evidence.

## Task order
1. §8 sweep FIRST — all three endpoints (`issues/N/comments`, `pulls/N/comments`,
   `pulls/N/reviews`) on PR #4 (and any new review activity on PR #12), plus
   upstream master drift (`git fetch upstream` in the sandbox; geaaru pushes
   seeds/specs directly — rebase before any push).
2. When PR #4 is reviewed: implement the ask or reply with a probe (§6
   both-end-state style). Open questions in it are maintainer's call: coreutils
   placement and the next slice size.
3. Next slice (when geaaru says go or PR #4 merges): gzip 1.15, make 4.4.1,
   patch 2.8.1, sed 4.10, tar 1.35, xz 5.8.4 (the `tukaani.org/xz/` tarball path
   now 404s — use the GitHub release asset), then binutils-pass2 / gcc-pass2
   (the `-pass2` naming is agreed). Chunk ~5–6 tools per PR, one commit per
   package with the attribution trailers, full-chain build + loader smoke as the
   verification bar. `xz` needs `liblzma.la` removed from both staged trees.
4. After the tool list is complete: `step1/sources` capture package at the END of
   step1 (geaaru's architecture: package_dir `${MFS}/sources`, requires the full
   step1 set) — ask in-thread whether it rides the last tools PR or lands alone.
5. Beyond ch.6 (ch.7 equivalents, step2/): structural decisions — propose with
   evidence in PR comments and let geaaru decide (§9).

## Hard rules
- Nothing runs on the host except gh/git/incus plumbing. Container does all
  builds. Never triple-quote into the container: base64-push a script (§2) —
  even one-liners that touch `$`.
- Git identity in the sandbox: use human's git name and git email — commits carry
  the `LLM-assisted-by:` + `Signed-off-by:` trailers; no `ai_ass` label
  (undefined in macaroni-fs). Token push pattern: `--env GITHUB_TOKEN` BEFORE
  `--`, inline-URL, output silenced, verify with `git ls-remote` + `pulls/<n>`
  head sha. No bare-#NNN refs in public text (§7p).
- Only captured output goes public; re-verify a claim on the exact head being
  shipped before quoting it (§5, §7e/m). Update PR-body claims with every head
  push (§10.9).
