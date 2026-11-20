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
| REVIEW — geaaru asked (16:49Z) to rebuild on `macaroni/mark-experimental` (python 3.11). DONE 2026-11-19: merged spec rendered BYTE-IDENTICAL on the experimental image (ebuild sha `c470c46f…`) and `emerge`-built green (`cuda cuda_sm_120`); `import serve.server` OK on python 3.11.16 (TypeError on 3.9 reproduced fresh in 932). One amendment: RDEPEND += `dev-python/jinja` (only unguarded third-party import; serve/frontend.py:29) → head `b6c520e`, re-verified BOTH trees (ebuild sha `b8c625b5…`, EMERGE-RC=0 + byte-identical installs). Reply draft delivered to orchestrator, NOT yet posted → **POSTED 2026-11-19** (`issuecomment-7017251111`). geaaru followed up (17:50Z): the venue's 3.11 is applied by the mark-stages `python_upgrade` hook and **`ego sync` reverts it** — verified in 932-exp (make.defaults back to `python3_9`, installed/eselect 3.11 persists) and answered with the recovery recipe + the proof that this spec is target-insensitive (`issuecomment-7018254099`). **Round 3 ANSWERED (2026-11-19 ~21:00Z)**: distutils-r1 = negative-proofed (setup.py has no setup()/pyproject — installer helper; pydeps/python_compat render NOTHING under builtin-github — throwaway probe; §3.59) + dep determination (static AST closure + headless mock-serve on 3.11 with optionals ABSENT) → jinja-only necessary&sufficient, NO atoms added; amendment `1ab51041` (comment-only, byte-identical render both trees), reply posted (`issuecomment-7020500555`) incl. the PY_PACKAGES-parity offer + honest tree gaps (jsonschema 4.5.1 < upstream pin; tqdm/textual absent) → **Round 4 LANDED 2026-11-20**: layout re-implemented + re-verified per operator decision — head `6ae0cd4e` (1 commit): seds of the `BUILD = ROOT/"build"` chain + 3 `--exe` defaults → engines resolve in `/usr/bin`, `build/` dir gone (0 files, grep 0 hits, spawn trace), NEW `/usr/bin/maya-server` launcher (name note: `/usr/bin/maya` = the engine), downloads default `~/Maya-data/models` (fallback sed; `--models-dir` wins; `/var/lib/maya`-with-user offered). Mid-round the tag line rode **v1.1.68** — generated+emerged in BOTH containers with equal renders (`73419564…`). Body updated (file-set map, mm3 rc-2 wording). Threaded reply posted (`discussion_r4639133257`). Loop-caught traps → §3.62/§7bb (EAPI7 cmake_src_prepare, newins×insinto, double-write_text revert bug). **Rebased 2026-11-20 (operator ask)** onto post-#630 `mark-unstable` → head `059dda1` (1 commit, MERGEABLE, clean): top-level-cluster EOF conflict with strata — the naive marker-strip CORRUPTED the file (strata's block stranded inside maya; §7cc), fixed by deterministic reconstruction (upstream file + our maya cluster), verified by regenerating BOTH clusters + byte-identical maya render vs old head re-targeted to the same tip (v1.1.69 — tag line rode again 1.1.68→69; §7cc). **Round 5 RUNNING 2026-11-20** (geaaru 13:17-13:28Z): (a) body: comment diet — minimal operational notes, history to PR description (§3.63 expansion pushed; playbook PR re-opened after #31 was MERGED); (b) conf.d/init.d/.service payload + `systemd` USE "following what is been done for strata" — loop `ses_e06c0b8db…` on 932+932-exp2 → **Round 5 DONE 2026-11-20**: head `2bdcae9` (1 commit, MERGEABLE; 3× amended by real failures — user-eclass `enewuser` miss, initd logfile perms via `checkpath`; §7q held) — body: diet to ≤3-line WHAT+WHY + file:line cites (render `e904a25f→2da65846`, both venues byte-equal, cross-container); `maya.{confd,initd,service}` payload, `inherit systemd user`, ollama-style daemon user, `systemd` added to IUSE (it was NOT there — briefing corrected), every confd knob probe-backed (MAYA_MODELS_DIR omitted on 0-hit evidence), full OpenRC runtime cycle on exp2 (start→dash 200→stop; invalid-port refusal; 401/200 key proof), USE=systemd placement on 932; PR body now canonical history (his ask); all three replies + body applied publicly (`discussion_r4645322448`, `discussion_r4645322803`, `issuecomment-7045570495`). | app-ai/maya | #932 (claimed, evidence comment posted) | kit-fixups#628 |
| **MERGED 2026-11-20** (`b9f9fa37`, issue auto-closed by Closes:) — engine-only atom landed; services follow-up filed as **kit-fixups#635** (OPEN, `97cc5e1`) per geaaru's files_dir request — see #631 row | app-ai/strata (source) | #933 (closed) | kit-fixups#630 |
| OPEN awaiting review — REWORKED per geaaru on #949 ("tarballs instead of AppImage"): now `strata-control`-tarball-based (version-free matcher rides the component skew per tag; engines stay runtime-fetched — per-release asset fluctuation + collision evidence in body); 1 commit `bcfc755` (was `44bc023`), reply posted (`issuecomment-7020417748`) + pointer on the PR → **Round 2 ANSWERED 2026-11-20**: engines IN per geaaru — USE surface (`cuda12 cuda13 hip_rx6000 hip_rx9070 hip_rx7900 hip_halo sycl arm64_cuda13`, `_` not `-` since USEs can't hyphenate; REQUIRED_USE arch ties; `{{ .Values.p }}` distfiles; version-free matchers stay), KEYWORDS line dropped (`KEYWORDS="*"` render proof), default-USE + `USE=cuda13` emerges green byte-identical; fluctuation = hard-fail on gated-but-absent (v0.2.15/0.2.14 repros) → no sanctioned opt-out → **mark-devkit ask drafted** (`assets[].optional: true` — NOT filed yet, maintainer's call). Head `04a8a5c`; threaded replies posted on both inline threads → **Round 3 RUNNING 2026-11-20** (geaaru 08:50Z): OpenRC conf.d/init.d + systemd .service payloads via a SHARED files_dir (`ai-kit/autogen.kit.d/strata/`), `systemd` USE on BOTH atoms + follow-up PR updating the merged strata source atom too — loop `ses_e07cfdc66…` on 949 + 933 (933 container REUSED for the services PR, not pruned) → **Round 3 DONE 2026-11-20**: head `b7f7429` (services wired; shared literal `files_dir: strata` proven for both PNs; per-PN units + portage-proven co-install `comm -12`=0; strata-bin unit = `strata-control serve --profile` with required-profile refusal contract; both sides of `systemd` USE placement-proven; QA-notice correction restated honestly in body — libpython `$ORIGIN` self-ref + driver-absent `libcuda.so.1`), source-atom side filed as NEW **kit-fixups#635** @ `97cc5e1` (follow-up to merged #630, no Closes, merge order: #631→#635; both atoms also generate green at the new v0.2.17 tip; ollama latent bug spotted: `systemd_dounit` without `inherit systemd` — flagged, untouched). Reply posted (`issuecomment-7039047874`) + pointer on #635 (`issuecomment-7039057657`). **Round 4 ANSWERED 2026-11-20** (geaaru inline 12:32Z: drop the AppImage comment archaeology) → head `f3b39b4` (1 commit, CLEAN; spec AppImage-mentions 0; one "comment" was ARTIFACT text inside `body:` → the only render diff = one ebuild comment, §3.63; re-verified at new tip **v0.2.17**: emerge rc 0, 0 QA/ERROR, cmp-OK, 6957 files, unit files placed, `--version` rc 0; threaded reply `discussion_r4642887514`). Also merged meanwhile via OTHER sessions' loops (account-shared): tqdm #633 (DeepSeek trailer; #950 closed) & textual #634 open (GLM trailer; pointer posted on #951 — not ours to review per §7dd) | app-ai/strata-bin | #949 (claimed; probe + rework evidence posted) | kit-fixups#631 |

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
- `mark-contrib-933` — Strata source loop; **#630 MERGED, container REUSED for the services
  follow-up**: `/kit-fixups` on branch `strata-services-933` @ `97cc5e1` = PR **#635** head (porcelain
  clean; old `strata-autogen-933` @ 7c7fd5a kept locally as history). **No GPU** — compile+link venue.
  CUDA 13.2.1 + `app-ai/strata-0.2.16` (cuda cuda_sm_120, with the new conf.d/init.d placement)
  installed; `/etc/portage/package.license` + `933-*` package.use persist. `clean-base` intact —
  prune after #635 merges.
- `mark-contrib-949` — Strata-bin loop. Branch `strata-bin-autogen-949` @ `f3b39b4` = PR #631 head
  (porcelain 0; services wired; `/opt/strata-bin` + `/opt/strata-engine-cuda13` relocation probes
  kept), `/root/b949` probe artifacts, logs verify949/collide949/gate949/amend949 + gen7-9.
  **Live co-install evidence kept intentionally: strata-bin-0.2.16 AND strata-0.2.16 (no-cuda) both
  installed** (the collision proof) — don't clean. squashfs/mesa-25.3.6/X-stack byproducts remain;
  the mesa pin was sandbox-only and removed. Keep until #631 merges, then prune.
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
- **`join` is undefined in this tree's eclass set** — the `cuda_sm_*` arch-list join in the
  ollama-pattern specs silently falls back to the package default (visible only in full build
  logs; affects in-tree ollama too; harmless with a single enabled `cuda_sm_*`, bites with multi-
  arch). Correct form: `"${flagged[*]}"` joined via IFS. #628's body offers the one-line follow-up;
  worth its own mark-issues entry if maintainers want it tracked (do NOT fix inside package PRs).
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
- **2026-11-20: PLAYBOOK PR #31 MERGED** (geaaru, `bf8def33` on `phoenix`) — §3.57-63, §7cc, this
  handoff file and the README row are upstream. Follow-up doc branch = `playbook/aikit-930-round1`
  re-pushed for §3.63's maintainer-expansion + board refreshes (successive small PRs as before).
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
- **#634 review round ANSWERED 2026-11-20 ~10:30Z** (geaaru: platformdirs fix wanted in-PR + 2
  inline notes): commit 2 `cc856ef` = platformdirs spec declares `hatch-fancy-pypi-readme` build
  hook + `selector: '<4.14.6'` cap (>=4.14.6 needs `python-sbom[hatch]>=0.1.2` — every python-sbom
  release requires py >=3.11, unpackageable on 3.9; cap drops with #879). Discriminating proof:
  plugin unmerged → `emerge -1 =dev-python/platformdirs-4.14.5` pulls it via BDEPEND → rc=0,
  byte-identical. Commit 1 amended: pygments `s:` DROPPED per maintainer (atom-level
  `pypi_name: pygments` — the 0.36.1 default rendered capital-`Pygments` and died, evidence in
  threaded reply) + mdit `s:` on `{{ .Values.pv }}`. Head `cc856ef` (2 commits), MERGEABLE clean;
  threaded replies + issue reply posted. **Waiting on geaaru merge.**

