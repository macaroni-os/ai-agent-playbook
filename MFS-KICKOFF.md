# MFS loop — session handoff (2026-11-06, fourth refresh)

Read PLAYBOOK.md first — §10 is the macaroni-fs loop, §5/§6/§7/§8 apply as written
(§2–§4 and the kit-fixups parts of §7 do NOT apply to macaroni-fs). Then:

## Board state (verify all of it live before acting — it moves daily)
- Sandbox: Incus container `mfs-dev` — image `macaroni/phoenix-builder`, created with
  `security.privileged=true` on 2026-11-04, but **privileged is NOT required** (verified
  2026-11-06: an unprivileged throwaway with only the `docker` profile ran
  `dockerd --storage-driver btrfs` and a NAT container; recreate `mfs-dev` unprivileged when
  convenient). Plain alias launch works, and the image has NO `/root` (mkdir first). Snapshots:
  clean-base, repo-ready, docker-ready, cross-tools-green, cross-tools-green-r2,
  cross-tools-green-r3, step1-ch6-green, **step1-ch6b-green** (2026-11-06, after the
  clean 24/24 chain). dockerd runs with `--storage-driver btrfs` and DOES NOT survive
  container restarts. The `docker` Incus profile carries `security.nesting=true` +
  `linux.kernel_modules: ip_tables,ip6_tables,netlink_diag,nf_nat,overlay,br_netfilter`;
  incusd modprobes them at every start. A host-kernel gap only shows up as a launch
  failing with `Failed to load kernel module …`. Do not add `xt_stat`: it is not a module
  in this kernel (7.2.15-1-omarchy) and a literal entry aborts every launch of the
  instance (`modprobe: FATAL: Module xt_stat not found`); the six-entry list is enough.
- Repo: `/root/macaroni-fs`, branch `step1-ch6-tools-2`, HEAD `4d3ad3f` (6 commits on
  top of merged master `d50cac4`); remotes `upstream` (macaroni-os/macaroni-fs) +
  `fork` (org-tekeli-borisp/macaroni-fs). Build invocation:
  `make build PACKAGES=<cat>/<name> BACKEND=docker REPO_CACHE=`
  — always verify artifacts (rc=0 lies, §10.2/§7 class).
- macaroni-os/macaroni-fs: PR #1/#2/#3/#4 MERGED (PR #4 as `d50cac4`, 2026-11-05);
  **PR #5 OPEN** — head `4d3ad3f`
  (https://github.com/macaroni-os/macaroni-fs/pull/5): ch.6 gzip 1.15, make 4.4.1,
  patch 2.8.1, sed 4.10, tar 1.35, xz 5.8.4; six commits, one per package; body
  carries the clean-slate 24/24 chain + loader smoke + `whip` evidence. Awaiting
  review.
- macaroni-os/ai-agent-playbook: PR #11 MERGED, PR #12 OPEN (this file + §10 cache
  and MAKEOPTS lessons) — this refresh rides it. Refresh this file at the end of every
  session; /tmp drafts die on reboots.
- macaroni-os/anise#36 OPEN — maintainer undecided between `--all-pkgs-tree` and
  leaving it; watch the thread, don't build on `--all`.
- Rotation probe before EVERY chain run (§10.7 recipe): base image
  `macaronios/terragon-minimal-devel:latest-amd64` is still
  `sha256:5f7e82be3b9bc9f0f773e9e61c817161ee7d360f9fc42313d487bf8a9f31591b`
  on 2026-11-06 (unchanged since 2026-11-05). It ships NO grep/awk/find/gzip/
  make/patch/tar/xz/file/diff, but does ship `/tmp` (777), the canonical
  `05gcc-x86_64-pc-linux-gnu.conf`, and libstdc++/libgcc_s resolving through it —
  seed `+14` remains self-sufficient.
- **Cache-masking lesson**: anise-build reuses local `anise-build/cache:*` images
  whenever the package spec/tree hash matches, even when the base image digest
  changed. A chain run can silently verify the OLD rotation. For any "full chain
  from seed" claim, run `docker system prune --all --volumes -f` first and quote the
  digest actually used. Reusing the cache is fine for quick spec iteration only.
- **MAKEOPTS lesson**: the support from macaroni-os/macaroni-fs commit `84af51a`
  is inert as merged — the specs export `MAKEOPTS`, but the calls are plain `make`
  and GNU make reads `MAKEFLAGS`, not `MAKEOPTS`. Verified 2026-11-06:
  `MAKEFLAGS=[]` with `MAKEOPTS=-j4`; gcc pass-1's `RUN` step still took 2066.5 s
  with the export in place. Evidence + proposed `make ${MAKEOPTS}` fix are in a
  comment on PR #5. Wait for geaaru's call before changing the toolchain specs.
- **Docker storage driver**: XFS + overlay2 is now the working setup on `mfs-dev`
  (verified 2026-11-06; faster than the btrfs driver). The operator added
  `loop-control` / `loop0` / `loop1` unix devices to the container; inside,
  `/dev/loop0` is an XFS image mounted at `/xfs-docker` (`ftype=1`) and dockerd runs
  `--data-root /xfs-docker/docker --storage-driver overlay2`. Captured:
  `Storage Driver: overlay2`, `Backing Filesystem: xfs`, and
  `docker run --rm busybox echo XFS-OVERLAY-NET-OK` passes. The loop mount and
  dockerd do NOT survive container restarts (re-create both; xfsprogs is installed
  in the container). For future unprivileged containers use a host XFS mount plus an
  Incus disk device instead of the in-container loop.

## Task order
1. §8 sweep FIRST — all three endpoints (`issues/N/comments`, `pulls/N/comments`,
   `pulls/N/reviews`) on PR #5 (and PR #12), plus upstream master drift
   (`git fetch upstream` in the sandbox; geaaru pushes seeds/specs directly —
   rebase before any push).
2. When PR #5 is reviewed: implement the ask or reply with a probe (§6
   both-end-state style). Open items for geaaru: the MAKEOPTS fix and the
   still-unanswered `coreutils` 6.5 gap before diffutils in the book order.
3. After PR #5 merges (or geaaru says go): next tools are **binutils-pass2** and
   **gcc-pass2** (the `-pass2` naming is agreed). Pull exact versions/procedures
   from linuxfromscratch.org `/lfs/view/systemd/` (rolling — do NOT trust
   `/view/stable/`, SysV 12.4); remember binutils/gcc pass-2 install under
   `${MFS}/usr` in 13.x. Full-chain build + loader smoke remain the verification
   bar, one commit per package with the attribution trailers.
4. If geaaru accepts the MAKEOPTS finding: small follow-up (or fold-in) changing
   the gcc/glibc/libstdcxx `make` calls to `make ${MAKEOPTS}`, with before/after
   timing on the same host and a re-verified chain.
5. After the tool list is complete: `step1/sources` capture package at the END of
   step1 (package_dir `${MFS}/sources`, requires the full step1 set) — ask whether
   it rides the last tools PR or lands alone.
6. Beyond ch.6 (ch.7 equivalents, step2/): structural decisions — propose with
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
