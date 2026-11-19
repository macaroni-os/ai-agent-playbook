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
| OPEN awaiting review (1 commit `d47be0a`, MERGEABLE) | app-ai/maya | #932 (claimed, evidence comment posted) | kit-fixups#628 |
| OPEN awaiting review (1 commit `7c7fd5a`, MERGEABLE, engine-only atom) | app-ai/strata (source) | #933 (claimed; evidence comment posted incl. the terminal-floor decision) | kit-fixups#630 |
| UNCLAIMED | app-ai/strata-bin (AppImage + per-backend assets; version-skew + atom-granularity flagged) | #949 (filed, sub-issue of #930) | — |

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
- Known toolkit landmines in the image (all disclosed on #628): nvidia-cuda-toolkit 12.9.2 fails
  vs glibc-2.41 (`cospi/sinpi`), 13.3.1 ships 0 cccl files vs its CMake config; **13.2.1 is the
  working tree state**. **Pre-staging a toolkit install: drop `-a`/`--ask` (dies in non-terminal)
  AND seed `/etc/portage/package.license` for `NVIDIA-CUDA` (masked) — both bit the #933 pre-stage
  (PLAYBOOK §2, strata round).**

## Feedback scope (owner rule, §7dd)
Answer review comments only on PRs whose head-commit trailer says
`LLM-assisted-by: Qwen3.8 Flash-Next Q8_0 (125B MoE, 512K Kontext) via OpenCode`.

## Task order (suggested)
1. **#949 strata-bin** — the only unclaimed child. Claim-comment first (verify it's unclaimed:
   re-sweep comments + `gh pr list --repo macaroni-os/kit-fixups --search "strata" --state all`).
   The AppImage unpack has NO tree precedent (code-searched) — probe-first loop, stop with an
   evidence round if the unpack phase needs a maintainer nod.
2. Answer any #628 / #630 review threads (same-day, fresh captured proof). Watch for the terminal-
   layer decision on #630 (inert /usr/share data vs Rust crates — the PR body lays out options).
3. Between tasks run the §8 feedback sweep (clean as of 2026-11-19 ~14:00Z: no comments/reviews on
   either PR yet).

## Open follow-ups (not blockers, do not invent fixes)
- #628 discloses: maya.py runtime venv/self-rebuild flow is not reproduced by the package (upstream
  source-hash meta file); Python ≥3.10 floor masks the dashboard today (no dev-python atoms declared).
- #630 discloses: terminal layer deferred on the 3.10 floor (upstream bundles a private CPython) —
  maintainer pick between inert `/usr/share` data (Maya-style) and the Rust `terminal/crates` CLI;
  HIP/SYCL backends unbuilt/unflagged (YAML-commented); the `0.2.0-beta` tag is NOT prerelease-marked
  upstream, so the spec's `github.match` beta-guard is load-bearing by construction.
- core-gl-kit: CUDA 12.9.2 vs glibc-2.41 headers; 13.3.1 cccl packaging hole (both noted in #628 —
  if maintainers want them tracked, file separate issues rather than widening #628).

## Freshness notes
- 2026-11-19: epic split into #932/#933 (labels copied, sub-issue links verified, split comment on
  the epic); #932 claimed; Maya loop ran baseline→spec→autogen-thin→emerge→smoke green (byte-identical
  installed ebuild; `nm`-proven NDEBUG fix per §3.57b; FetchContent pinned via second asset §3.57a);
  PR #628 opened with `ai_ass`; evidence comment posted on #932. #933 reshaped to SOURCE scope on
  operator decision (findings: vendored ggml, no-GPU compile CI); Strata source loop green — engine
  atom only (terminal floor 3.10 vs tree 3.9, probe in the PR), PR #630 + evidence comment; strata-bin
  follow-up filed as #949 (third child of #930). Both PRs review-clean so far. Feed every review
  lesson back into PLAYBOOK.md and refresh THIS file each session.
