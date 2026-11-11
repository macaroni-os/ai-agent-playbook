# MFS loop — session handoff (2026-11-06, fifth refresh — PR #5/#6 merged)

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
- Isolated build sandbox: **`mfs-build`** (unprivileged, `default` + `docker` profile,
  own btrfs daemon) — created 2026-11-06 because a second agent session shares `mfs-dev`
  and swaps its daemon (XFS/btrfs benchmarks), killing in-flight builds. Snapshot
  `step1-makeopts-green` (clean 24/24 MAKEOPTS chain), `step1-pass2-green`,
  `step1-pass2-green-r2` and `step1-pass2-dropintree-green` (clean 27/27 through
  gcc-pass2; the last one after the GMP/MPFR/MPC decision).
  Use `mfs-build` for macaroni-fs
  builds; leave `mfs-dev` to the other session.
- Repo: `/root/macaroni-fs`, master `a0f102d` (PR #7 merge); `step1-pass2`
  (`341c339`) is merged and stale. Remotes `upstream`
  (macaroni-os/macaroni-fs) + `fork` (org-tekeli-borisp/macaroni-fs).
  Build invocation: `make build PACKAGES=<cat>/<name> BACKEND=docker REPO_CACHE=`
  — always verify artifacts (rc=0 lies, §10.2/§7 class).
- macaroni-os/macaroni-fs: **step1 is complete — PR #1–#8 MERGED** (PR #4
  `d50cac4`, PR #5 `e7bd201`, PR #6 `7ddee9a`, PR #7 `a0f102d`, PR #8
  `0e32e73`; 2026-11-06/08/11). Master is `a0f102d`; the 27-package chain ends
  at `step1/gcc-pass2` (binutils 2.46.0, gcc 16.2.0). GMP/MPFR/MPC are not
  unpacked in-tree: the step1 packages provide the build-machine copies under
  `${MFS}/cross-tools`, and the chroot must export
  `LD_LIBRARY_PATH=/cross-tools/lib` (without it `cc1` fails on
  `libmpc.so.3`, captured). Remaining step1 follow-ups: binutils 2.47 book
  drift and the empty `/usr/lib64` created by base.
- macaroni-os/ai-agent-playbook: PR #11–#15 MERGED (PR #13 `0a1c3be`,
  PR #14 `a4ed3af`, PR #15 `4f9de49`); PR #20 carries this refresh.
  Refresh this file at the end of every session; /tmp drafts die on reboots.
- **Book ↔ stage mapping**: `step1` is LFS ch.5 + ch.6 (cross toolchain +
  cross-compiled temporary tools, ending at `gcc-pass2`); `step2` starts at
  ch.7 (enter chroot + additional temporary tools) and continues into ch.8.
  That is why binutils-pass2/gcc-pass2 live under `packages/step1/`.
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
  and GNU make reads `MAKEFLAGS`, not `MAKEOPTS`. geaaru asked for a specific PR;
  it is macaroni-os/macaroni-fs#6, merged as `7ddee9a`. Verified on a clean 24/24
  chain: gcc pass-1
  2066.5 s → 236.2 s, glibc 102.8 s → 17.6 s, libstdcxx 100.4 s → 49.7 s; sum over
  all packages 3297.4 s → 1352.5 s. gcc needed its goals split (`all-gcc` before
  `all-target-libgcc`): under a real `-j` the combined invocation races
  `libgcc/configure` against the not-yet-built `gcc/xgcc`.
- **Docker storage driver**: XFS + overlay2 was built and BENCHMARKED 2026-11-06
  (loopback shape, see §10.1): it works (`xfs_info` `ftype=1`, docker info
  `Backing Filesystem: xfs` / `Supports d_type: true`) but is **not faster** than the btrfs
  driver on this host — pulls and small-file churn are a wash, the synthetic layered build
  ran 1.7–4.5x slower on loop-XFS. Live shape on `mfs-dev` is the original
  `dockerd --storage-driver btrfs` on `/var/lib/docker`; the test loop devices, the
  `/xfs-docker` mount and the per-instance `linux.kernel_modules` were removed again.
  The maintainer's real-partition claim stays open (single LUKS/btrfs disk here): the
  production device-add shape and both prerequisites (`linux.kernel_modules xfs` for the
  module, hotplugged loop nodes for loopback) are recorded in §10.1.

## Task order
1. §8 sweep FIRST — upstream master drift (`git fetch upstream` in the sandbox;
   geaaru pushes seeds/specs directly — rebase before any push) and any new review
   activity on the merged PRs. All three endpoints, always.
2. Step1 is complete. Next is the **ch.7 / step2 bootstrap**, which is a structural
   decision, not a spec batch: the chroot entry needs
   `LD_LIBRARY_PATH=/cross-tools/lib` for the pass-2 compiler, the missing `/bin`
   plumbing (`/bin/bash`,`/bin/sh`), ownership/virtual-fs setup
   (`changingowner`, `kernfs`), and then the temporary tools from the rolling
   book: Gettext 1.0, Bison 3.8.2, Perl 5.44.0, Zlib 1.4, mpdecimal 4.0.1,
   Python 3.15.2, Texinfo 7.3, Util-linux 2.42.4, then cleanup. Work out how
   anise-build represents “build inside the assembled chroot” (dependency image
   + `chroot ${MFS}` steps + per-package payload dir) and propose it in a PR
   comment/Discord for geaaru to choose (§9) before writing a batch of specs.
3. After the ch.7 structure is agreed: implement the temporary tools as step2
   packages, then close step1 with `step1/sources` (package_dir `${MFS}/sources`,
   requires the full step1 set; ask whether it rides the last tools PR or lands
   alone).
4. Nothing further for MAKEOPTS unless geaaru asks; keep the default `-j$(nproc)`
   (override via `.Values.makeopts`) and re-measure if the host core count changes.
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
