# COREKIT-KICKOFF — core-kit autogen epic loop (macaroni-os/mark-issues#880)

Living session-handoff for the #880 loop: board state + task order. **Refresh this every session**
so a reboot can't erase it (same discipline as MFS-KICKOFF.md). Conventions: PLAYBOOK.md §3.52
(epic→children workflow, renumbered from 3.50 post-merge), §3 conversions, §5 verification, §6 one-container rule.

## Mission
Epic: convert every legacy merge-only core-kit atom to an autogen spec in kit-fixups
(`core-kit/autogen.kit.d/*` + atom moved to `core-kit/merge.kit.d/core_autogen.yml`).
47 child issues (#881–#927) are linked as sub-issues of #880; each carries its tree versions,
the conversion precedent, and maintainer-decision notes. `Closes:` goes on each child, never on
the epic. Conversions need NO release-branch PRs (§3.38 + confirmed after #573: the kit-fixups
release branches carry merge-atom recipes pointing at `core-kit@mark-unstable`).

## Sandboxes (one-container rule, §6)
- `mark-contrib` — shared/contested: in use by the GLM-5.3-Flash session (children #882 cpio +
  #886 lz4 both MERGED; #926 zlib open with its own review thread — theirs to answer).
- `mark-contrib-b` — this loop's dedicated clone (from the `contrib-base` image =
  `mark-contrib/clean-base` snapshot; kit-fixups pulled to head; own `clean-base` snapshot taken).
  Session model: `Qwen3.8 Flash-Next Q8_0 (125B MoE, 512K Kontext)`.
- To spawn another: `incus publish mark-contrib-b/clean-base --alias contrib-base` →
  `incus launch contrib-base mark-contrib-c` → sanity per §6, then renew its `clean-base`.

## Feedback scope (owner rule, §7dd)
Proceed review comments ONLY on PRs whose head commit trailer says
`LLM-assisted-by: Qwen3.8 Flash-Next Q8_0 (125B MoE, 512K Kontext) via OpenCode` (October variant
`Qwen3.8-Flash-Next Q8_0 (125B, local)` counts as ours too). Foreign trailers (GLM-5.3-Flash,
DeepSeek V4.1) = other sessions' loops, even under the same GitHub author.

## Board (captured 2026-11-14; re-verify via API at session start)
| state | package | child | PR |
|---|---|---|---|
| MERGED (2026-11-14) | app-arch/gzip | #884 | kit-fixups#579 |
| MERGED (2026-11-14) | sys-apps/diffutils | #904 | kit-fixups#584 |
| MERGED (2026-11-14, GLM session) | app-arch/lz4 | #886 | kit-fixups#580 |
| MERGED (2026-11-14, GLM session) | app-arch/cpio | #882 | kit-fixups#583 |
| REVIEW ANSWERED (round 2: multilib→autotools, rebased, `941488b`) — maintainer's turn | app-arch/bzip2 | #881 | kit-fixups#582 |
| OPEN, GLM session's own feedback (econf thread — NOT ours) | sys-libs/zlib | #926 | kit-fixups#581 |
| IN PROGRESS — mark-contrib-b (this session) | sys-apps/findutils | #906 | — |
| free | app-arch/dpkg, libarchive, pax, pxz, rar, snappy, unzip, xz-utils, zstd | #883, #885, #887–#893 | — |
| free | app-shells/bash, dash, tcsh, zsh | #894–#897 | — |
| free | dev-libs/gmp, libpcre, libpcre2, libsigc++ | #898–#901 | — |
| free | net-dns/libidn2 | #902 | — |
| free | sys-apps/baselayout … sysvinit (9, minus diffutils #904) | #903, #905–#912 | — |
| free | sys-devel/autoconf … patch (8) | #913–#920 | — |
| free | sys-libs/zlib(-ng) … see #581 in review; binutils-libs, minizip-ng, ncurses, readline, timezone-data | #921, #923–#925, #927 | — |

Full number↔package mapping: `gh api "repos/macaroni-os/mark-issues/issues/880/sub_issues?per_page=100" --jq '.[] | "\(.number) \(.title)"'`
Before claiming: re-sweep the child's comments (someone else may have claimed it) + `gh pr list
--repo macaroni-os/kit-fixups --search "<pkg>" --state all`.

## Maintainer-decision queue (flagged in child bodies; do not decide unasked, §9)
- autoconf SLOT policy (#913, geaaru asked on the epic) — collect cross-kit slot-dependency
  evidence FIRST, then let maintainers pick; tree serves 2.13/2.69/2.72 today.
- binutils / binutils-libs slot policy (#915, #921) — preserve the served track unless told.
- libpcre EOL vs retire (#899); baselayout has-no-upstream (#903); pxz dormant + patch (#888);
  rar proprietary-binary (#889); bash/readline patch-series shape (#894, #924).

## Task order (suggested)
1. Finish bzip2 (#881) — in flight.
2. Simple GNU dirlisting conversions (same cluster/precedent as gzip #579): diffutils #904,
   findutils #906, gawk #907, grep #908, sed #911, cpio #882, m4 #918, patch #920, groff #909,
   make #919 (toolchain floor! see child note), autoconf #913/automake #914/libtool #917 AFTER
   the SLOT decision.
3. GitHub-source library conversions (model on the `app_arch_github` cluster): snappy #890,
   libarchive #885, zstd #893, xz-utils #892, minizip-ng #922, zlib #926 / zlib-ng #927,
   libpcre2 #900, gmp #898, libidn2 #902, ncurses #923, readline #924 (patch series),
   timezone-data #925 (rolling year names), unzip #891 (SF-era source — probe first).
4. Special cases last, with maintainer input: pax #887, pxz #888, rar #889, dpkg #883,
   bash #894 / dash #895 / tcsh #896 / zsh #897, baselayout #903, file #905, less #910,
   sysvinit #912, gnuconfig #916, binutils #915 + binutils-libs #921, libpcre #899,
   libsigc++ #901 (two slots — probe the GNOME download layout).

## Loop recipe (per package)
Claim-comment on the child issue → one foreground subagent (briefing = §2 + §3 + §5 + §7 verbatim
refs + lessons bb/cc + the container it owns + fork `org-tekeli-borisp/macaronios-kit-fixups`,
cross-named head label, `--input` PR creation) → reproduce baseline first → autogen-thin identity
proof + deployed build + smoke → one-commit PR with `Closes:` on the child + `ai_ass` →
orchestrator re-verifies public state (§6) and updates THIS file + the epic comment when merged.

## Freshness notes
- 2026-11-14: epic triaged (29 atoms already autogen / 50 legacy atoms / 47 children created and
  linked); gzip PR #579 up (upstream tip 1.15 released 2026-09-20 will be added by the first bot
  run — disclosed in PR); bzip2 dispatched on mark-contrib-b.
- 2026-11-15: bzip2 PR #582 up (sourceware dirlisting, 1.0.8 is tip so no bump; 7 tree patches
  ship flat in `autogen.kit.d/bzip2/`, semantic diff posted as a PR comment). Loop caught+fixed a
  generated-ebuild symlink bug pre-push (`libbz2.so{,.1,1.0}` flattened wrong → `libbz2.so1.0`;
  `bash -n` passed it, only `ls /lib64/libbz2*` caught it — reinforces §7bb/cc: verify the
  installed file set, not syntax). lz4 (#886) is the parallel session's — PR #580. diffutils
  (#904) — merged as kit-fixups#584 (the cancelled subagent had already pushed+opened it; see the
  dual-session note below).
- Feed every new lesson into PLAYBOOK.md. Branch `playbook/corekit-880-round2`: renumbers the
  epic-workflow entry §3.50→§3.52 (post-merge collision with #22's Rust §3.50/§3.51 — §4's table
  cites the Rust one), adds §7dd, refreshes this board. Keep pushing board updates to whatever
  playbook branch is current.
- DUAL-SESSION HAZARD (learned the loud way): both sessions share the GitHub account, so a
  package can be converted by the OTHER session and merged while our subagent runs — a merged PR
  under our own name may be OUR JUST-COMPLETED WORK (diffutils: claim 23:03, #584 opened 23:34 by
  our own cancelled-but-already-pushed subagent, merged 23:35; the cancel killed only the report).
  Before (re)dispatching a package: re-check child-issue state + `gh pr list --search <pkg>
  --state all` IMMEDIATELY before dispatch, and when a subagent dies/is cancelled, inspect fork
  branch sha vs container worktree sha vs PR head sha before concluding anything.
