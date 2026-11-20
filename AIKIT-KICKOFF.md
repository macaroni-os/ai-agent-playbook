# AIKIT-KICKOFF — ai-kit Maya/Strata autogen loop (macaroni-os/mark-issues#930)

Living session-handoff for the #930 epic loop: board state + task order. **Refresh this every
session** (same discipline as COREKIT-KICKOFF.md). Conventions: PLAYBOOK.md §3.52 (epic→children),
§3.57 (autogen.kit.d flat-spec layout + new-package shapes), §5 verification, §6 one-container rule.

## Mission
Epic #930: autogen two upstream AI apps into ai-kit (`app-ai`) in kit-fixups. Split 2026-11-19 into
children (sub-issues of #930, epic stays open, `Closes:` on each child's own PR):
- #932 project-maya → `app-ai/maya` (source-build, CUDA/HIP-only engine, near-daily tags — no pins).
- #933 Strata → `app-ai/strata` source flavour (engine-only atom; retitled from the original
  strata-bin scope on operator decision — see the scope-change comment on the issue).
- #949 `app-ai/strata-bin` (follow-up, filed 2026-11-19): AppImage + per-backend engine/control
  assets; atom-granularity + engine-version-skew (engine 0.1.62 vs app 0.2.16) flagged as
  maintainer decisions.
Never package model weights (GGUF/`.pack`, tens–hundreds of GB, runtime downloads).

## Board (captured 2026-11-19; re-verify via API at session start)
| state | item | issue | PR |
|---|---|---|---|
| REVIEW — geaaru asked (16:49Z) to rebuild on `macaroni/mark-experimental` (python 3.11). DONE 2026-11-19: merged spec rendered BYTE-IDENTICAL on the experimental image (ebuild sha `c470c46f…`) and `emerge`-built green (`cuda cuda_sm_120`); `import serve.server` OK on python 3.11.16 (TypeError on 3.9 reproduced fresh in 932). One amendment: RDEPEND += `dev-python/jinja` (only unguarded third-party import; serve/frontend.py:29) → head `b6c520e`, re-verified BOTH trees (ebuild sha `b8c625b5…`, EMERGE-RC=0 + byte-identical installs). Reply draft delivered to orchestrator, NOT yet posted → **POSTED 2026-11-19** (`issuecomment-7017251111`). geaaru followed up (17:50Z): the venue's 3.11 is applied by the mark-stages `python_upgrade` hook and **`ego sync` reverts it** — verified in 932-exp (make.defaults back to `python3_9`, installed/eselect 3.11 persists) and answered with the recovery recipe + the proof that this spec is target-insensitive (`issuecomment-7018254099`). **Round 3 ANSWERED (2026-11-19 ~21:00Z)**: distutils-r1 = negative-proofed (setup.py has no setup()/pyproject — installer helper; pydeps/python_compat render NOTHING under builtin-github — throwaway probe; §3.59) + dep determination (static AST closure + headless mock-serve on 3.11 with optionals ABSENT) → jinja-only necessary&sufficient, NO atoms added; amendment `1ab51041` (comment-only, byte-identical render both trees), reply posted (`issuecomment-7020500555`) incl. the PY_PACKAGES-parity offer + honest tree gaps (jsonschema 4.5.1 < upstream pin; tqdm/textual absent) → **Round 4 LANDED 2026-11-20**: layout re-implemented + re-verified per operator decision — head `6ae0cd4e` (1 commit): seds of the `BUILD = ROOT/"build"` chain + 3 `--exe` defaults → engines resolve in `/usr/bin`, `build/` dir gone (0 files, grep 0 hits, spawn trace), NEW `/usr/bin/maya-server` launcher (name note: `/usr/bin/maya` = the engine), downloads default `~/Maya-data/models` (fallback sed; `--models-dir` wins; `/var/lib/maya`-with-user offered). Mid-round the tag line rode **v1.1.68** — generated+emerged in BOTH containers with equal renders (`73419564…`). Body updated (file-set map, mm3 rc-2 wording). Threaded reply posted (`discussion_r4639133257`). Loop-caught traps → §3.62/§7bb (EAPI7 cmake_src_prepare, newins×insinto, double-write_text revert bug). | app-ai/maya | #932 (claimed, evidence comment posted) | kit-fixups#628 |
| **MERGED 2026-11-20** (`b9f9fa37`, issue auto-closed by Closes:) — engine-only atom landed; geaaru's follow-up (service payloads) is now the services round, see #631 row | app-ai/strata (source) | #933 (closed) | kit-fixups#630 |
| OPEN awaiting review — REWORKED per geaaru on #949 ("tarballs instead of AppImage"): now `strata-control`-tarball-based (version-free matcher rides the component skew per tag; engines stay runtime-fetched — per-release asset fluctuation + collision evidence in body); 1 commit `bcfc755` (was `44bc023`), reply posted (`issuecomment-7020417748`) + pointer on the PR → **Round 2 ANSWERED 2026-11-20**: engines IN per geaaru — USE surface (`cuda12 cuda13 hip_rx6000 hip_rx9070 hip_rx7900 hip_halo sycl arm64_cuda13`, `_` not `-` since USEs can't hyphenate; REQUIRED_USE arch ties; `{{ .Values.p }}` distfiles; version-free matchers stay), KEYWORDS line dropped (`KEYWORDS="*"` render proof), default-USE + `USE=cuda13` emerges green byte-identical; fluctuation = hard-fail on gated-but-absent (v0.2.15/0.2.14 repros) → no sanctioned opt-out → **mark-devkit ask drafted** (`assets[].optional: true` — NOT filed yet, maintainer's call). Head `04a8a5c`; threaded replies posted on both inline threads → **Round 3 RUNNING 2026-11-20** (geaaru 08:50Z): OpenRC conf.d/init.d + systemd .service payloads via a SHARED files_dir (`ai-kit/autogen.kit.d/strata/`), `systemd` USE on BOTH atoms + follow-up PR updating the merged strata source atom too — loop `ses_e07cfdc66…` on 949 + 933 (933 container REUSED for the services PR, not pruned). Also merged meanwhile via OTHER sessions' loops (account-shared): tqdm #633 (DeepSeek trailer; #950 closed) & textual #634 open (GLM trailer; pointer posted on #951 — not ours to review per §7dd) | app-ai/strata-bin | #949 (claimed; probe + rework evidence posted) | kit-fixups#631 |

## Sandboxes (one-container rule, §6)
- `mark-contrib-932-exp2` — THE python-3.11 venue since 2026-11-20 (image `macaroni/mark-experimental`
  @ 20260909; the old `mark-contrib-932-exp` DELETED after cross-checks captured). Bootstrapped with
  the `python_upgrade` hook reproduced step-by-step at target 3.11 (full recipe + its traps now in
  PLAYBOOK §3.61: `--noreplace` semantics, `package.use` is a DIRECTORY here — bare `>>` silently
  loses USE flags, the hook's bare `emerge --unmerge python` would kill the ACTIVE interpreter).
  State: `make.defaults` → `python3_11`, eselect python3.11, only `/usr/lib/python3.11` (pillow
  libdir-drift fixed, 3.13-REQUIRES count 0), mark-devkit 0.36.1, toolkit 13.2.1, gcc 12.3.0,
  `app-ai/maya-1.1.67` emerged green (876 files) + mock-serve proven. `/kit-fixups` at PR head.
  Keep until #628 merges.
- `mark-contrib-932` — Maya loop. `/kit-fixups` on branch `maya-autogen-932` (clean, = PR head `1ab51041`
  after the jinja amendment + force-with-lease push 2026-11-19 ~17:45Z).
  **`clean-base` snapshot must NOT be renewed**: the current state (nvidia-cuda-toolkit-13.2.1,
  installed app-ai/maya, `/root/maya932` logs) IS the verified build environment for review
  rounds. Prune the container after #628 merges. Meta-repo ai-kit carries untracked
  `app-ai/maya/` + `/etc/portage/package.use/maya932` (revert: `git clean` in the kit).
- `mark-contrib-933` — Strata source loop. `/kit-fixups` on branch `strata-autogen-933` (= PR #630
  head `7c7fd5a`, porcelain clean). **No GPU** — the engine proof is compile+link only. CUDA 13.2.1
  + `app-ai/strata-0.2.16` installed; sandbox ai-kit
  reverted (0 dirt); `/etc/portage/package.license` (NVIDIA-CUDA) + `933-*` package.use persist —
  needed for any rebuild here. `clean-base` intact — **#630 MERGED 2026-11-20 but container REUSED**
  for the services follow-up PR (`strata-services-933` branch off post-merge head); prune after
  that PR merges.
- `mark-contrib-949` — Strata-bin loop DONE (PR #631). Outcome kept: branch `strata-bin-autogen-949`
  @ `bcfc755` = PR head (porcelain 0; tarball rework — AppImage leftovers irrelevant, `/opt/strata-bin`
  + `/opt/strata-engine-cuda13` relocation probes kept), `/root/b949` probe artifacts (assets + extracted trees + logs),
  squashfs-tools/mesa-25.3.6/X-stack/strata-bin installed as verification byproducts; ai-kit working
  copy restored. Keep until #631 merges, then prune. NOTE the mesa pin was sandbox-only and removed.
- Known toolkit landmines in the image (all disclosed on #628): nvidia-cuda-toolkit 12.9.2 fails
  vs glibc-2.41 (`cospi/sinpi`), 13.3.1 ships 0 cccl files vs its CMake config; **13.2.1 is the
  working tree state**. **Pre-staging a toolkit install: drop `-a`/`--ask` (dies in non-terminal)
  AND seed `/etc/portage/package.license` for `NVIDIA-CUDA` (masked) — both bit the #933 pre-stage
  (PLAYBOOK §2, strata round).**

## Feedback scope (owner rule, §7dd)
Answer review comments only on PRs whose head-commit trailer says
`LLM-assisted-by: Qwen3.8 Flash-Next Q8_0 (125B MoE, 512K Kontext) via OpenCode`.

## Task order (suggested)
1. **FEEDBACK: #628 geaaru rounds 1–3 — ALL ANSWERED 2026-11-19** (experimental rebuild + jinja
   amendment `b6c520e`; hook warning + venue recovery recipe; distutils-r1/RDEPEND determination →
   `1ab51041`, replies posted). Maintainer's turn; open offer on the table = PY_PACKAGES parity set.
2. **RUNNING 2026-11-20**: geaaru's morning-before rounds — #628 install-layout inline (kill
   /usr/share/maya/build via sed, direct /usr/bin entry, downloads out of /usr/share → var/home)
   on `ses_e09d3344affe…` (containers 932+932-exp2); #631 engines-back-IN per maintainer call
   (USE-gated per-backend assets + arm64 custom USEs + required_use arch ties, drop KEYWORDS line,
   fluctuation problem = central probe) on `ses_e09d3344affd…` (container 949). Reply drafts →
   orchestrator posts. **Done direct**: geaaru's tqdm/textual request → filed #950/#951
   (package-request shape per #439; confirmation posted on #628).
3. Watch #628 / #630 / #631 for further threads — same-day, fresh captured proof. Remaining open
   decisions: #628 PY_PACKAGES-parity offer (now gated on #950/#951 landing); #630 terminal layer
   (inert /usr/share vs Rust crates).
4. Between tasks run the §8 feedback sweep.

## Open follow-ups (not blockers, do not invent fixes)
- #628 discloses: maya.py runtime venv/self-rebuild flow is not reproduced by the package (upstream
  source-hash meta file); Python ≥3.10 floor still masks the dashboard on mark-unstable (only
  `dev-python/jinja` is declared — the sole unguarded third-party import; PIL/psutil/jsonschema are
  runtime-guarded upstream-side).
- #630 discloses: terminal layer deferred on the 3.10 floor (upstream bundles a private CPython) —
  maintainer pick between inert `/usr/share` data (Maya-style) and the Rust `terminal/crates` CLI;
  HIP/SYCL backends unbuilt/unflagged (YAML-commented); the `0.2.0-beta` tag is NOT prerelease-marked
  upstream, so the spec's `github.match` beta-guard is load-bearing by construction.
- core-gl-kit: CUDA 12.9.2 vs glibc-2.41 headers; 13.3.1 cccl packaging hole (both noted in #628 —
  if maintainers want them tracked, file separate issues rather than widening #628).
- **python-3.10 floor is now a tree-wide blocker (all three loops hit it)**: mark-unstable accepted
  interpreter is 3.9.25, so #932 maya dashboard (≥3.10), #933 strata terminal (≥3.10), and #949
  `media-libs/mesa-26.2.2` (`Python >= 3.10 not found` in its build) ALL fail on the same wall.
  `mark-experimental` (3.11) is the maintainer-supplied relief — geaaru pointed #628 there; it is the
  mark-stages `python_upgrade` hook applied over mark-unstable kits (re-applied after every `ego
  sync`), i.e. a venue, not tree policy yet; if a bump to mark-unstable is wanted, file it as a
  core-kit/lang-kit issue rather than per-package.

## Freshness notes
- 2026-11-19: epic split into #932/#933 (labels copied, sub-issue links verified, split comment on
  the epic); #932 claimed; Maya loop ran baseline→spec→autogen-thin→emerge→smoke green (byte-identical
  installed ebuild; `nm`-proven NDEBUG fix per §3.57b; FetchContent pinned via second asset §3.57a);
  PR #628 opened with `ai_ass`; evidence comment posted on #932. #933 reshaped to SOURCE scope on
  operator decision (findings: vendored ggml, no-GPU compile CI); Strata source loop green — engine
  atom only (terminal floor 3.10 vs tree 3.9, probe in the PR), PR #630 + evidence comment; strata-bin
  follow-up filed as #949 (third child of #930). #949 strata-bin loop GREEN probe-first: AppImage
  self-extracts headless (no squashfs BDEPEND) but portage `unpack()` skips `.AppImage` → spec stages
  from DISTDIR; engine payloads evidence-backed non-packaged (embedded control dup + signed feed),
  control-bin split left to maintainers; PR #631 + evidence comment. Then **geaaru feedback on #628**
  (rebuild maya on `macaroni/mark-experimental`, python 3.11) — rebuilt green on the venue (spec
  rendered byte-identical cross-tree), `import serve.server` proven on 3.11, amendment `b6c520e`
  (+`dev-python/jinja`, closure-proven) verified on BOTH trees, reply posted same day. Same evening:
  geaaru's hook-warning round (answered; §2 venue entry + recovery recipe), his distutils-r1/
  RDEPEND round on #628 (answered — determination, no atoms added, head `1ab51041`; §3.59), and the
  #949 tarball-rework of #631 (head `bcfc755`, version-free matcher; §3.58 follow-up). Feed every
  review lesson back into PLAYBOOK.md and refresh THIS file each session.

## 2026-11-20 — textual-951 loop (GLM-5.3-Flash via OpenCode, branch playbook/textual-951-glm)

- **#950 tqdm: MERGED** (macaroni-os/kit-fixups#633, mark-unstable-only, no release-branch PRs —
  the operative propagation precedent for python-modules-kit).
- **#951 textual: PR macaroni-os/kit-fixups#634 OPEN** (1 commit `838f17a`, rebased onto #633's
  merge, MERGEABLE, `ai_ass` set; evidence comment posted on #951). Scope: `dev-python/textual`
  8.2.8 + the full missing closure (markdown-it-py 3.0.0, mdit-py-plugins 0.4.2, linkify-it-py
  2.0.3, mdurl 0.1.2, uc-micro-py 1.0.3) + rich/pygments converted to autogen (15.0.0 / 2.21.0,
  atoms moved base.yml → python_autogen.yml per §3.6). Verified: full-chain emerge rc=0, 8/8
  byte-identity, headless App smoke (Markdown+linkify). Disclosed: in-tree platformdirs-4.14.5
  spec lacks `hatch-fancy-pypi-readme` build dep (pre-existing, untouched).
- **#628 gate**: PY_PACKAGES-parity offer now waits only on #634 review (tqdm already in).
- Sandbox `mark-contrib-951` (cloned from 932/clean-base): j2cli emerged, the 8 atoms installed,
  `/root/gen951-post/` (post-rebase generated trees) + `/root/build951*.sh` kept as evidence.
  Meta-repo python-modules-kit copy carries the deployed ebuilds (documented dirt). **Keep until
  #634 merges, then prune.** NOTE: the ai-agent-playbook checkout in `~/Development/src/ai-agent-playbook`
  sits on THIS branch (the aikit loop's); the §3.64 playbook lesson lives on `playbook/textual-951-glm`
  (worktree off phoenix) — merge that branch before/with this one; it does NOT touch this file's
  other lines.

