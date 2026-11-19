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
| REVIEW — geaaru asked (16:49Z) to rebuild on `macaroni/mark-experimental` (python 3.11). DONE 2026-11-19: merged spec rendered BYTE-IDENTICAL on the experimental image (ebuild sha `c470c46f…`) and `emerge`-built green (`cuda cuda_sm_120`); `import serve.server` OK on python 3.11.16 (TypeError on 3.9 reproduced fresh in 932). One amendment: RDEPEND += `dev-python/jinja` (only unguarded third-party import; serve/frontend.py:29) → head `b6c520e`, re-verified BOTH trees (ebuild sha `b8c625b5…`, EMERGE-RC=0 + byte-identical installs). Reply draft delivered to orchestrator, NOT yet posted → **POSTED 2026-11-19** (`issuecomment-7017251111`). geaaru followed up (17:50Z): the venue's 3.11 is applied by the mark-stages `python_upgrade` hook and **`ego sync` reverts it** — verified in 932-exp (make.defaults back to `python3_9`, installed/eselect 3.11 persists) and answered with the recovery recipe + the proof that this spec is target-insensitive (`issuecomment-7018254099`). **Round 3 ANSWERED (2026-11-19 ~21:00Z)**: distutils-r1 = negative-proofed (setup.py has no setup()/pyproject — installer helper; pydeps/python_compat render NOTHING under builtin-github — throwaway probe; §3.59) + dep determination (static AST closure + headless mock-serve on 3.11 with optionals ABSENT) → jinja-only necessary&sufficient, NO atoms added; amendment `1ab51041` (comment-only, byte-identical render both trees), reply posted (`issuecomment-7020500555`) incl. the PY_PACKAGES-parity offer + honest tree gaps (jsonschema 4.5.1 < upstream pin; tqdm/textual absent) | app-ai/maya | #932 (claimed, evidence comment posted) | kit-fixups#628 |
| OPEN awaiting review (1 commit `7c7fd5a`, MERGEABLE, engine-only atom) | app-ai/strata (source) | #933 (claimed; evidence comment posted incl. the terminal-floor decision) | kit-fixups#630 |
| OPEN awaiting review — REWORKED per geaaru on #949 ("tarballs instead of AppImage"): now `strata-control`-tarball-based (version-free matcher rides the component skew per tag; engines stay runtime-fetched — per-release asset fluctuation + collision evidence in body); 1 commit `bcfc755` (was `44bc023`), reply posted (`issuecomment-7020417748`) + pointer on the PR | app-ai/strata-bin | #949 (claimed; probe + rework evidence posted) | kit-fixups#631 |

## Sandboxes (one-container rule, §6)
- `mark-contrib-932-exp` — experimental-verification host (image `macaroni/mark-experimental` @ 20260909).
  End state 2026-11-19: egosync RC=0 (34 kits, **all on `mark-unstable`** — no `mark-experimental` kit
  branch exists anywhere; the IMAGE supplies python 3.11.16, mask `>=dev-lang/python-3.10` still in
  core-kit). Installed: mark-devkit 0.36.1, nvidia-cuda-toolkit 13.2.1 (cccl OK), gcc profile
  switched 16.2.0→**12.3.0** (nvcc 13.2.1 hard-refuses gcc>15 — configure abort), `app-ai/maya-1.1.67`
  from the amended spec (`/var/db/pkg` ebuild sha `b8c625b5…` == 932's). `/kit-fixups` at PR head
  `1ab51041` (porcelain 0), ai-kit carries untracked deployed `app-ai/maya/`, evidence tree
  `/root/exp932/` (gen, gen2, emerge*.log, pre-stage.log, closure.sh). Keep until #628 merges.
  **Venue caveat (geaaru, hook `mark-stages/jobs/hooks/extras/python_upgrade.yml`):** the session's
  bootstrap `ego sync` REVERTED the hook's make.defaults edit — `PYTHON_TARGETS/SINGLE_TARGET="python3_9"`
  again (verified 2026-11-19; installed 3.11.16 + eselect persist). Any PYTHON_TARGETS-sensitive rebuild
  here must re-apply the hook steps first (recipe in PLAYBOOK §2 venue entry).
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
  needed for any rebuild here. `clean-base` intact — keep container until #630 merges, then prune.
- `mark-contrib-949` — Strata-bin loop DONE (PR #631). Outcome kept: branch `strata-bin-autogen-949`
  @ `bcfc755` = PR head (porcelain 0; tarball rework — AppImage leftovers irrelevant, `/opt/strata-bin`
  + `/opt/strata-engine-cuda13` relocation probes kept), `/root/b949` probe artifacts (assets + extracted trees + logs),
  squashfs-tools/mesa-25.3.6/X-stack/strata-bin installed as verification byproducts; ai-kit working
  copy restored. Keep until #631 merges, then prune. NOTE the mesa pin was sandbox-only and removed.
- `mark-contrib-932-exp` — experimental-verification host — end state recorded above (first bullet of
  this section).
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
2. Watch #628 / #630 / #631 for maintainer threads — same-day, fresh captured proof. Open decisions
   now: #628 PY_PACKAGES-parity offer (geaaru's call; honest tree gaps disclosed — jsonschema 4.5.1,
   no tqdm/textual) + real-GPU/vision execution still unexercised; #630 terminal layer (inert
   /usr/share vs Rust crates); #631 REWORKED per geaaru to the control-tarball shape (reply posted
   on #949 + pointer on the PR) — engines stay runtime unless he asks for USE gating (matrix
   evidence in the PR body).
3. Between tasks run the §8 feedback sweep.

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
