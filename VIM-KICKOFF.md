# 868 loop handoff — macaroni-os/mark-issues#868 (vim autogen; living doc, refresh every session)

Read PLAYBOOK.md first — §2/§3/§4/§5/§6/§7/§8 apply as written.

## Board state (verify live before acting)
- **PR: [kit-fixups#567](https://github.com/macaroni-os/kit-fixups/pull/567)**
  `[editors-kit] app-editors/vim, app-editors/vim-core, app-editors/gvim: autogen`
  → mark-unstable, head `14b47d5dd93d6571b2d7f34ed40517e609c75966`, 1 commit, `ai_ass`,
  MERGEABLE, OPEN awaiting review (submitted 2026-11-12). `Closes:` on #567 = sole closer.
  9 files: `editors-kit/autogen.kit.d/base.yml` (718-line `vim:` cluster: `defaults:`
  github `query: tags` — upstream has **zero GitHub releases** (probed 2×) + `simple-github.tmpl`
  api-tarball recovery `mv vim-vim-* ${S}`; 3 package entries, `~vim-core-{{ .Values.pv }}`
  cross-pin), flat `autogen.kit.d/<pn>/` files dirs (NO patches — all 6 legacy Gentoo-bug
  patches dropped with per-patch dry-run justification), `merge.kit.d/editors_autogen.yml`
  (new; core-kit source on mark-unstable per net-kit precedent — the legacy editors.yml
  keeps mark-testing sources + fixups: that split is CORRECT, do not "fix" it),
  editors.yml −3 atoms.
- Version truth generated: **9.2.1332** for all three PNs (identical selection proven).
  Dropped USE (provider missing, probed): racket, motif, selinux, aqua. Compat narrowed to
  tree reality: lua5-1+luajit, `PYTHON_COMPAT=( python3+ )` (profile masks >=3.10 —
  core-kit funtoo-kit-python), USE_RUBY="ruby31" (also fixes gvim's ruby30 fallback bug).
- **§3.38 proof already in body**: ls-tree ×5 release branches — all carry only
  `editors.yml` plain atoms (mark-xl/mark-v have NO gvim atom) → NO propagation PRs filed.
  If geaaru wants gvim on those branches anyway, that's an atom-ADD ask: ask first.
- Evidence: container `/root/868e/` (282 MB, incl. wd-*/srcprobe trees) + host
  `.868g/ev_*.txt` mirrors. Baseline tree-build (9.1.1420) + generated (9.2.1332) merges
  both GREEN with `cmp` byte-identity vs `/var/db/pkg`; default USE + spot combo
  (acl crypt cscope nls perl python vim-pager / gvim +sound session) proven; NOT exercised:
  lua/ruby/tcl/X/gpm/minimal USE sides (disclosed).
- Container **`mark-contrib-868`** (mine, do not share — mark-contrib-867 is another live
  session; never touch). Direct `incus exec mark-contrib-868 -- bash -c "..."` works host-side
  (no newgrp needed); anything with `$` via base64 script. Snapshot `bootstrap-clean`
  (pre-mark-devkit). NOTE: the container now has the 9.2.1332 stack + mark-devkit 0.36.1 +
  go 1.27.3 installed — restore `bootstrap-clean` before a fresh reproduce if you want the
  exact pristine baseline (then re-emerge mark-devkit).
- /kit-fixups: branch `868-vim-autogen-mark-unstable` pushed (fork remote =
  org-tekeli-borisp/macaronios-kit-fixups); identity configured; meta-repo ego-synced.

## Task order
1. **§8 sweep FIRST** on #567 — all three endpoints (`issues/567/comments`,
   `pulls/567/comments`, `pulls/567/reviews`), geaaru reviews fast; expect the standing asks
   (generator/template placement, body simplifications, selector caps). Same-day threaded
   replies with fresh captured proof; one-commit amend + fresh-lease push + head-sha verify;
   re-run autogen-thin AND re-merge after any spec change (§6). Watch the EOF-append
   conflict class (§7c) if he merges other editors-kit PRs meanwhile.
2. On merge: post the close-suggestion on mark-issues#868 (fix summary + "ready to close
   once it propagates"); `Closes:` should auto-fire — verify, and if it fired on merge but
   propagation is pending, keep the thread informative (§3.23).
3. Then take the NEXT `mark-unstable` issue from the queue (KICKOFF.md loop). Known open
   (2026-11-12): #869 codex — `.869g/proposals-draft.md` + `reply-draft.md` sit waiting on
   the OPERATOR's review; do not post without him.

## Hard rules (this loop)
- Nothing public without captured proof; issue-side comms are ORCHESTRATOR-side — subagents
  file/verify PRs only.
- Sandbox-only fixes (cairo X USE, the mesa-python unmask, rustc `-j4`/codegen-units=1 for
  the futex abort under -j28) are NOT spec content and must never leak into ebuilds;
  disclosed in #567 body — keep them out of any "fix".
- Commits: identity borisp <boris.pigin@gmail.com> verbatim + `LLM-assisted-by:` /
  `Signed-off-by:` trailers (house form copied from merged kit-fixups commits).
- No bare `#NNN` in public text; render-audit via graphql bodyHTML but scan TEXT nodes —
  GitHub's `aria-label` around Closes keywords contains `#NNN` (false positive, hit 2026-11-12).
