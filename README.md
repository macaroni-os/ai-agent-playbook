# Macaroni OS — Agent Contributor Playbook

A reusable playbook for running an AI coding agent (OpenCode or similar) as a Macaroni OS package
contributor: fixing [`mark-issues`](https://github.com/macaroni-os/mark-issues) with an autogen
first approach in [`kit-fixups`](https://github.com/macaroni-os/kit-fixups), verifying everything in
a disposable Incus sandbox, and handling maintainer review loops.

Distilled from 20 merged kit-fixups PRs (October 2026) and direct maintainer feedback — these are
the conventions the maintainers actually asked for.

## Contents
| file | what |
|---|---|
| [PLAYBOOK.md](PLAYBOOK.md) | the full playbook: sandbox setup, maintainer conventions, fix-pattern table, verification standard, subagent operating model, known agent failure modes, macaroni-fs stage1/stage2 spec work (§10) |
| [KICKOFF.md](KICKOFF.md) | short prompt to hand a fresh agent session so it boots the loop from the playbook |

## Quick start
1. `git clone` this repo on your workstation.
2. Make sure the one-time sandbox prerequisites (playbook §2) are met — Incus, subuid/subgid ranges, `incus-admin` group, Macaroni container image.
3. Open an AI coding agent session in this repo's directory and paste `KICKOFF.md` as the first message.
4. Stay in the loop: **you** review what the agent posts publicly — that safety net is part of the design (playbook §7).

Feedback welcome via PR/issue — the playbook explicitly depends on being updated whenever a new
convention is learned ("feed back into this playbook" is part of the loop).

*Originally a practitioner's playbook (20 agent PRs, Oct 2026), now a macaroni-os doc — keep feeding
it: every review cycle should leave it sharper than it found it.*
