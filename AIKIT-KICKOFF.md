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
| REVIEW — geaaru asked (16:49Z) to rebuild on `macaroni/mark-experimental` (python 3.11); rebuild loop RUNNING (session `ses_e0b408742…`, container `mark-contrib-932-exp`) | app-ai/maya | #932 (claimed, evidence comment posted) | kit-fixups#628 |
| OPEN awaiting review (1 commit `7c7fd5a`, MERGEABLE, engine-only atom) | app-ai/strata (source) | #933 (claimed; evidence comment posted incl. the terminal-floor decision) | kit-fixups#630 |
| OPEN awaiting review (1 commit `44bc023`, MERGEABLE; AppImage-only minimal shape, engine non-package recommendation + control-bin option on the issue) | app-ai/strata-bin | #949 (claimed; probe evidence posted) | kit-fixups#631 |

## Sandboxes (one-container rule, §6)
- `mark-contrib-932` — Maya loop. `/kit-fixups` on branch `maya-autogen-932` (clean, = PR head).
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
  @ `44bc023` = PR head (porcelain 0), `/root/b949` probe artifacts (assets + extracted trees + logs),
  squashfs-tools/mesa-25.3.6/X-stack/strata-bin installed as verification byproducts; ai-kit working
  copy restored. Keep until #631 merges, then prune. NOTE the mesa pin was sandbox-only and removed.
- `mark-contrib-932-exp` — experimental-verification host (new image family: launched 2026-11-19
  from `macaroni:macaroni/mark-experimental` @ 20260909, python 3.11.16 preferred, **no /root at
  first boot (§2 quirk — mkdir done), no meta-repo materialized — `ego sync` bootstraps it**
  (detached `/root/egosync.log`), no kit-fixups/mark-devkit by default — loop bootstraps those).
- Known toolkit landmines in the image (all disclosed on #628): nvidia-cuda-toolkit 12.9.2 fails
  vs glibc-2.41 (`cospi/sinpi`), 13.3.1 ships 0 cccl files vs its CMake config; **13.2.1 is the
  working tree state**. **Pre-staging a toolkit install: drop `-a`/`--ask` (dies in non-terminal)
  AND seed `/etc/portage/package.license` for `NVIDIA-CUDA` (masked) — both bit the #933 pre-stage
  (PLAYBOOK §2, strata round).**

## Feedback scope (owner rule, §7dd)
Answer review comments only on PRs whose head-commit trailer says
`LLM-assisted-by: Qwen3.8 Flash-Next Q8_0 (125B MoE, 512K Kontext) via OpenCode`.

## Task order (suggested)
1. **FEEDBACK (top priority)**: #628 geaaru rebuild-on-experimental — running (session
   `ses_e0b408742…`). When it returns: post the reply on #628; if the spec amended (dev-python
   atoms now declarable on 3.11), confirm the force-push moved the PR head + mark-unstable proof
   intact.
2. Watch #630 / #631 for maintainer threads — same-day, fresh captured proof. Open decisions the
   loops surfaced for maintainers: #630 terminal layer (inert /usr/share vs Rust crates); #631
   atom-granularity (control-bin split? engine confirmed non-packaged).
3. Between tasks run the §8 feedback sweep.

## Open follow-ups (not blockers, do not invent fixes)
- #628 discloses: maya.py runtime venv/self-rebuild flow is not reproduced by the package (upstream
  source-hash meta file); Python ≥3.10 floor masks the dashboard today (no dev-python atoms declared).
- #630 discloses: terminal layer deferred on the 3.10 floor (upstream bundles a private CPython) —
  maintainer pick between inert `/usr/share` data (Maya-style) and the Rust `terminal/crates` CLI;
  HIP/SYCL backends unbuilt/unflagged (YAML-commented); the `0.2.0-beta` tag is NOT prerelease-marked
  upstream, so the spec's `github.match` beta-guard is load-bearing by construction.
- core-gl-kit: CUDA 12.9.2 vs glibc-2.41 headers; 13.3.1 cccl packaging hole (both noted in #628 —
  if maintainers want them tracked, file separate issues rather than widening #628).
- **python-3.10 floor is now a tree-wide blocker (all three loops hit it)**: mark-unstable accepted
  interpreter is 3.9.25, so #932 maya dashboard (≥3.10), #933 strata terminal (≥3.10), and #949
  `media-libs/mesa-26.2.2` (`Python >= 3.10 not found` in its build) ALL fail on the same wall.
  `mark-experimental` (3.11) is the maintainer-supplied relief — geaaru pointed #628 there; if a
  bump to mark-unstable is wanted, file it as a core-kit/lang-kit issue rather than per-package.

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
  (rebuild maya on `macaroni/mark-experimental`, python 3.11) — rebuild loop running in
  `mark-contrib-932-exp`. Feed every review lesson back into PLAYBOOK.md and refresh THIS file each
  session.
