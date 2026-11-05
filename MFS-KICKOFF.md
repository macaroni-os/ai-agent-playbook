# MFS loop — session handoff (2026-11-04, second refresh)

Read PLAYBOOK.md first — §10 is the macaroni-fs loop, §5/§6/§7/§8 apply as written
(§2–§4 and the kit-fixups parts of §7 do NOT apply to macaroni-fs). Then:

## Board state (verify all of it live before acting — it moves daily)
- Sandbox: Incus container `mfs-dev` — RECREATED clean 2026-11-04 (old one deleted by the
  human). Privileged, image `macaroni/phoenix-builder` — plain alias launch works
  (re-verified 2026-11-05); the image has NO `/root` — mkdir first. Snapshots: clean-base,
  repo-ready, docker-ready, cross-tools-green, cross-tools-green-r2, cross-tools-green-r3.
  dockerd runs with `--storage-driver btrfs` and DOES NOT survive container restarts;
  netfilter modules die on host reboot — if dockerd fails with "Table does not exist",
  ask the human to run:
  `sudo sh -c 'modprobe -a br_netfilter ip_tables iptable_nat xt_nat'`.
- Repo: `/root/macaroni-fs`, branch `step1-cross-tools`, HEAD `e177157`; remotes
  `upstream` (macaroni-os/macaroni-fs) + `fork` (org-tekeli-borisp/macaroni-fs).
  Build invocation: `make build PACKAGES=<cat>/<name> BACKEND=docker REPO_CACHE=`
  — always verify artifacts (rc=0 lies, §10.2/§7 class).
- macaroni-os/macaroni-fs: PR #1/#2 MERGED, **PR #3 OPEN** — head `e177157` (seed `+14`):
  gcc limits.h wrapper, step1/m4, ncurses, bash, self-contained tool list (+11), and the
  review-round arc +12→+14: GCC-runtime prelude added→canonicalized→DROPPED (the terragon
  image `sha256:ea4a393c…` shipped the conf all along; the real failure was `/tmp` missing
  before `gcc-config` — guard hoisted, `virtual/base` dropped per maintainer). Three review
  rounds answered; all 4 threads resolved (2026-11-04); body carries the +14 13/13
  clean-slate verification incl. `whip h linking.check`. Awaiting re-review.
- macaroni-os/ai-agent-playbook: PR #11 MERGED, PR #12 OPEN (this file + §10 rotation lessons);
  **refresh this file at the end of every session** (that's the whole point; /tmp drafts died on
  reboots before).
- macaroni-os/anise#36 OPEN — answered 2026-11-04 with captured `anise-repo-devkit`
  evidence; geaaru is UNDECIDED between `--all-pkgs-tree` and leaving it — watch the thread,
  don't build on `--all`.
- Seed probe before EVERY chain run (§10.7 recipe): rotation content changes without any spec
  change; the +14 seed + current image cover tools, GCC conf and /tmp — still trust the seed,
  never the image, and add `whip h linking.check` to the post-chain battery.

## Task order
1. §8 sweep FIRST — all three endpoints (`issues/N/comments`, `pulls/N/comments`,
   `pulls/N/reviews`) on PR #3, on this repo's open PRs, and anise#36 — plus upstream
   master drift (`git fetch upstream` in the sandbox; geaaru pushes seeds/specs directly —
   rebase before any push; expect seed `+N` collisions). macaroni-fs has Issues disabled —
   feedback arrives as review threads on the PR only. Reply same day, captured evidence only.
2. When geaaru responds on PR #3: implement the ask or reply with a probe (§6 both-end-state
   style). Any "the image is fixed now" claim gets probed against the live rotation digest
   before we drop a workaround (§10.7).
3. When PR #3 merges (or geaaru says go): continue ch.6 in book order — diffutils, file,
   findutils, gawk, grep, gzip, make, patch, sed, tar, xz, then binutils-pass2 / gcc-pass2
   (the `-pass2` naming is agreed). Pull exact versions/procedures from
   linuxfromscratch.org `/lfs/view/systemd/` (rolling — today it is kernel 7.3.1 / glibc
   2.44 / gcc 16.2, already drifted from the 13.1 release) — do NOT trust `/view/stable/`
   (SysV 12.4). New specs: house double-install pattern (§10.5), quote 2-segment versions
   (§10.6), pin `CC_FOR_BUILD` / `--with-build-cc` (§10.7), remember binutils/gcc pass-2
   install under `${MFS}/usr` in 13.x. Chunk it: ~4-5 tools per PR, full-chain build +
   loader smoke as the verification bar, per-package commits with the attribution trailers,
   PR body = root causes + captured evidence + open questions.
4. After the tool list is complete: `step1/sources` capture package at the END of step1
   (geaaru's architecture: package_dir `${MFS}/sources`, requires the full step1 set) —
   ask in-thread whether it rides the last tools PR or lands as its own.
5. Beyond ch.6 (ch.7 equivalents, step2/): structural decisions — propose with evidence in
   PR comments and let geaaru decide (§9).

## Hard rules
- Nothing runs on the host except gh/git/incus plumbing. Container does all builds. Never
  triple-quote into the container: base64-push a script (§2) — even one-liners that touch `$`.
- Git identity in the sandbox: use human's git name and git email — commits carry the
  `LLM-assisted-by:` + `Signed-off-by:` trailers; no `ai_ass` label (undefined in
  macaroni-fs). Token push pattern: `--env GITHUB_TOKEN` BEFORE `--`, inline-URL, output
  silenced, verify with `git ls-remote` + `pulls/<n>` head sha. No bare-#NNN refs in public
  text (§7p).
- Only captured output goes public; re-verify a claim on the exact head being shipped before
  quoting it (§5, §7e/m). Update PR-body claims with every head push (§10.9).
