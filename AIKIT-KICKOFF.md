# AIKIT-KICKOFF — ai-kit Maya/Strata autogen loop (macaroni-os/mark-issues#930)

Living session-handoff for the #930 epic loop: board state + task order. **Refresh this every
session** (same discipline as COREKIT-KICKOFF.md). Conventions: PLAYBOOK.md §3.52 (epic→children),
§3.57 (autogen.kit.d flat-spec layout + new-package shapes), §5 verification, §6 one-container rule.

## Mission
Epic #930: autogen two upstream AI apps into ai-kit (`app-ai`) in kit-fixups. Split 2026-11-19 into
children (sub-issues of #930, epic stays open, `Closes:` on each child's own PR):
- #932 project-maya → `app-ai/maya` (source-build, CUDA/HIP-only engine, near-daily tags — no pins).
- #933 Strata → `app-ai/strata-bin` proposed (asset flavors: AppImage 0.2.16, strata-control,
  per-backend strata-engine tarballs; atom-split scope is a flagged maintainer decision).
Never package model weights (GGUF/`.pack`, tens–hundreds of GB, runtime downloads).

## Board (captured 2026-11-19; re-verify via API at session start)
| state | item | issue | PR |
|---|---|---|---|
| OPEN awaiting review (1 commit `d47be0a`, MERGEABLE) | app-ai/maya | #932 (claimed, evidence comment posted) | kit-fixups#628 |
| IN PROGRESS (source loop dispatched 2026-11-19) | app-ai/strata (source) | #933 (retitled from strata-bin per operator decision; scope-change comment posted) | — |
| PLANNED follow-up | strata-bin (AppImage + engine/control assets, incl. engine-version-skew 0.1.62 vs 0.2.16) | child of #930, to open after the source PR | — |

## Sandboxes (one-container rule, §6)
- `mark-contrib-932` — Maya loop. `/kit-fixups` on branch `maya-autogen-932` (clean, = PR head).
  **`clean-base` snapshot must NOT be renewed**: the current state (nvidia-cuda-toolkit-13.2.1,
  installed app-ai/maya, `/root/maya932` logs) IS the verified build environment for review
  rounds. Prune the container after #628 merges. Meta-repo ai-kit carries untracked
  `app-ai/maya/` + `/etc/portage/package.use/maya932` (revert: `git clean` in the kit).
- `mark-contrib-933` — Strata source loop. `/kit-fixups` at head `09cac49`, branch
  `strata-autogen-933` when the loop runs. Background `emerge -1av =dev-util/nvidia-cuda-toolkit-13.2.1`
  launched at dispatch (poll `/root/toolkit.log` for `DONE`); **no GPU in this container** — the
  engine proof is compile+link only. Keep until the strata PR merges.
- Known toolkit landmines in the image (all disclosed on #628): nvidia-cuda-toolkit 12.9.2 fails
  vs glibc-2.41 (`cospi/sinpi`), 13.3.1 ships 0 cccl files vs its CMake config; **13.2.1 is the
  working tree state**.

## Feedback scope (owner rule, §7dd)
Answer review comments only on PRs whose head-commit trailer says
`LLM-assisted-by: Qwen3.8 Flash-Next Q8_0 (125B MoE, 512K Kontext) via OpenCode`.

## Task order (suggested)
1. #933 Strata SOURCE — loop dispatched 2026-11-19 (session `ses_e0c1049d5…`): baseline engine
   build with CUDA 13.2.1 (headless, explicit arch, no `native`), Python terminal probe (Maya hit
   the 3.9 tree floor), `builtin-github` riding the v-tags (4-part `v0.2.14.1` parser check),
   autogen-thin render + ride proof, emerge byte-identical + smoke. Precedents: maya (#628) for the
   CUDA/`gpus:` shape, ollama cluster for cmake/user.
2. Open the `strata-bin` follow-up child once #933's source shape lands (AppImage unpack +
   per-backend engine/control assets; opencode-bin/pi-bin idiom; version-skew note).
3. Answer any #628 / strata-PR review threads (same-day, fresh captured proof).
4. Between tasks run the §8 feedback sweep.

## Open follow-ups (not blockers, do not invent fixes)
- #628 discloses: maya.py runtime venv/self-rebuild flow is not reproduced by the package (upstream
  source-hash meta file); Python ≥3.10 floor masks the dashboard today (no dev-python atoms declared).
- core-gl-kit: CUDA 12.9.2 vs glibc-2.41 headers; 13.3.1 cccl packaging hole (both noted in the PR
  body — if maintainers want them tracked, file separate issues rather than widening #628).

## Freshness notes
- 2026-11-19: epic split into #932/#933 (labels copied, sub-issue links verified, split comment on
  the epic); #932 claimed; Maya loop ran baseline→spec→autogen-thin→emerge→smoke green (byte-identical
  installed ebuild; `nm`-proven NDEBUG fix per §3.57b; FetchContent pinned via second asset §3.57a);
  PR #628 opened with `ai_ass`; evidence comment posted on #932. First loop on this epic — feed
  every review lesson back into PLAYBOOK.md and refresh THIS file each session.
