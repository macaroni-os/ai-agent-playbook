# AGENT CONTRIBUTOR PLAYBOOK — Macaroni OS (generalized)

> A reusable playbook for any Macaroni OS team member who wants to run an AI coding agent
> (OpenCode or similar) as a package contributor: fixing `mark-issues`, building autogen specs
> in `kit-fixups`, and surviving maintainer review — with an AI agent doing the loop work.
>
> Conventions herein were established through 20 merged kit-fixups PRs (Oct 2026) and direct
> maintainer review feedback; they are what the maintainers actually asked for, not guesses.

---

## 1. Mission & ground rules

Work the `macaroni-os/mark-issues` queue (typically label `mark-unstable`) top-to-bottom:
reproduce → fix → verify in a disposable sandbox → PR into `kit-fixups` (or the owning kit repo)
→ answer every review thread with captured proof. One issue at a time via sequential subagents.

Non-negotiables:
- **Nothing is ever built/tested on the workstation.** Everything package-related happens in a
  throwaway container you can snapshot and revert.
- **Reproduce before you fix.** Several open issues are stale (fixed by tree drift); the
  reproduce step rejects those honestly instead of filing cosmetic PRs.
- **Duplicate check before everything**: `gh pr list --repo macaroni-os/kit-fixups --search "<pkg>" --state all`
  and an issue search. (Duplicated issues cost maintainer trust.)
- **Only captured output goes into public comments.** If something cannot be verified
  (license-walled downloads, missing toolchains), say so explicitly — a `BLOCKED` report with
  evidence is respected; a faked green run is not.
- **Human in the loop**: the human reviews public artifacts. Agents make posting mistakes;
  expect to catch them (see §7 failure modes).

## 2. Sandbox setup (one-time)

1. **Incus** on the workstation host (Docker is not adequate — you need a full OpenRC/portage
   container with systemd-free init and file sharing off).
   - Prereq gotchas seen in the field: `/etc/subuid` + `/etc/subgid` need a wide range
     (e.g. `root:1000000:10000000`) or the Macaroni image fails to unpack; firewall/bridge rules
     must permit container egress; the operator needs `incus-admin` group membership.
2. Launch from the official image family (e.g. `macaroni:macaroni/mark-unstable`), name it
   `mark-contrib` (substitute your own name everywhere).
3. Inside: meta-repo lives at `/var/git/meta-repo`; **each kit is its own git repo** under
   `kits/<kit>/` — your revert tool is `git checkout -- <path>` per kit.
4. Add: a clone of your `kit-fixups` fork at `/kit-fixups` (for autogen runs), snapshot the clean
   state (`incus snapshot create …`), snapshot again before every risky phase.
5. Convenience: `MAKEOPTS="-j28"` (or your core count); leave `emerge --quiet-build` as default.

### Exec pattern (every single container command)
```
printf '%s\n' 'incus exec <CONTAINER> -- bash -c "COMMANDS"' | newgrp incus-admin
```
Outer single quotes, inner double quotes escaped as `\"` whenever the line is assembled in a
double-quoted host string. Long-running commands: set your agent tool's timeout generously
(~900000 ms for emerge; the default will kill builds mid-compile).

Known rough edges:
- `incus file pull/push` (SFTP) may fail (307 redirect / missing agent). Workaround: base64 in
  chunks (`echo <B64> | base64 -d > f`); huge single-arg base64 hits ARG_MAX — split.
- MARK's `emerge` rejects bare `=cat/pkg` atoms: use exact versions or plain atoms.
- Handy tools: `qlist -Iv <cat/pkg>`, `qdepends -Qt <pkg>` (reverse deps), logs under
- **Non-trivial container work: don't triple-nest quotes.** Write the script on the host, transfer
  it `base64 -w0` → `echo <B64> | base64 -d > /root/x.sh` → `bash /root/x.sh; shred -u /root/x.sh`.
  Every quoting bug (silent host-side `$()` expansion, empty `$GT` from over-escaping) vanished
  when we switched to this for multi-step push/verify scripts.
- `incus exec` flag position is load-bearing: `incus exec <c> --env K=V -- bash -c "…"` —
  `--env` AFTER the command becomes a harmless argument and the variable silently stays unset
  (observed: push failing with "Invalid username or token" while the script "looked right").
  Its sibling trap: FORGETTING `--env` entirely. Named remotes for public repos are anonymous
  https — `git ls-remote` (and lease-value fetching) succeed without any token, only the push
  fails, as a bare `rc=128`. When push returns 128, check the token plumbing first, not the
  network and not the lease.
- **Fork renames break pushes as a bare rc=128 too** (firebird session 2026-10-30): after a GitHub
  repo rename the old URL 404s — `could not read Username for ...` means the REMOTE IS GONE, not a
  bad token. Check `gh api repos/<owner>/<repo> --jq .full_name` before touching credentials, and
  note cross-repo PR heads may need the `owner:repo:branch` label form after a rename.
- **Never `shred -u` a script that bash is still executing** — the running wrapper's inode vanishes
  and the detached chain dies mid-phase without an error line. Detach a small launcher instead.
- Detached-build polling: `pgrep -f "emerge --quiet-build"` ALSO matches the `bash -c` wrapper
  of the very exec call that runs the pgrep (the pattern is in its own command line) → phantom
  "STILL-RUNNING". Trust sentinels instead: log tail sentinels (`>>> Installing`, `Calculating
  … done!`) plus `qlist -Iv <cp>` for ground truth.
  `/var/tmp/portage/<cat>/<pn>-<ver>/temp/`.
- `mark-devkit` (install/build from source if the image lacks it): **use `autogen-thin`** for
  spec verification. Full `autogen` runs the merge phase, which needs `anise-portage-converter`
  (often absent; the `app-misc/anise` currently installable provides only the luet CLI).
- If issue-filing experiments left **stale `/etc/portage/patches/<atom>/` dirs**, they
  double-apply against ebuild PATCHES and fail the build. Do not delete system state to verify —
  use a sandbox config: copy `/etc/portage` → `/root/pcr`, drop the stale dir there, run
  `PORTAGE_CONFIGROOT=/root/pcr emerge …` (the `PORTAGE_USER_PATCHES_PATH` env is ignored by
  this Portage).

### Identity & credentials
- **Git identity is set by the HUMAN and used verbatim by the agent — never invented.**
  Before the loop commits anything, the operator sets it explicitly on the host AND inside
  every build container (they start with none):
  `git config --global user.name "<name>" && git config --global user.email "<email>"`.
  The agent reads `git config user.name` / `user.email` in the repo it operates on and commits
  as exactly that identity (if a repo carries a stale local override, remove it rather than
  passing a guessed `--author`). If no identity is configured → STOP and ask; never
  "hallucinate" a plausible name/email — commit authorship is DCO-relevant and lives in the
  org's history forever. (macaroni-os also expects attribution trailers — see §3.19.)
- `gh` CLI logged in via **browser OAuth** (`gh auth login -p https -w`) — gives API + push rights
  for your fork; kit repos (e.g. `python-modules-kit`, `net-kit`, `core-hw-kit`) often accept
  **direct branches from team members** (verify per repo: `gh api repos/macaroni-os/<kit> --jq .permissions`);
  some (e.g. `media-kit`) do not → those go through kit-fixups only.
- Token hygiene for agents: never interpolate `$(gh auth token)` inside `printf "…"` strings
  (it can echo into transcripts); assign to a variable first, silence push output
  (`git push … >/dev/null 2>&1; echo rc=$?`), and scrub anything printed
  (`sed 's/gho_[A-Za-z0-9]*//g'`). If a token ever appears in logs: rotate immediately
  (`gh auth refresh`) — everything should read the token live from `gh auth token`, so rotation
  is zero-effort.
- Force-push ONLY as: `git push --force-with-lease=<br>:$(git ls-remote $U refs/heads/<br> | cut -f1) $U <br>`
  (maintainers may push to your PR branches — fetch before every push; remote-tracking refs in
  long-lived clones go stale).

## 3. Maintainer conventions (kit-fixups autogen) — learned from review, not docs

1. PR title `[<kit>] <category>/<pn>: <short fix>`; **exactly one squashed commit**; body starts
   from `Fixes https://github.com/macaroni-os/mark-issues/issues/NNN` + verification evidence.
2. Generator preference: `builtin-github` (`query: releases`|`tags`) > `builtin-dirlisting` >
   `builtin-noop` (only for dormant upstreams, justify in a comment). Autogen exists so the bot
   auto-tracks releases — a spec that freezes a version defeats the purpose.
3. Templates: `simple-github.tmpl` for github/api tarballs (it generates
   `post_src_unpack() { mv <user>-<repo>-* ${S}; }`). Binary/asset-based packages: `simple.tmpl`
   + explicit `s:` (often `'${WORKDIR}'` for flat tarballs) + `newbin` in `src_install` — keep
   default `src_unpack` behavior where possible.
4. `github.user: X` (+ `repo:` only if != PN); the leading `v` of tags is stripped automatically —
   write transforms only for what remains. **The release TITLE often drives version parsing**
   (e.g. `Bun v1.4.4`, `pnpm 12.6.0`) — transform titles accordingly. Exclude pre-releases with
   `exclude:` lists (`rc`, `beta`); huge tag-spam repos need `github.ignore_tags: true`
   (precedent: llama-cpp, opencode).
5. Dependencies: `depend: |` as `${RDEPEND}` + your extra conditionals. **Beware render order:
   `simple.tmpl` emits `RDEPEND` before `DEPEND`** — so `${DEPEND}` inside `rdepend` renders
   empty; put the literals in `rdepend:` and `depend: '${RDEPEND}'` (libedit pattern) or write
   both literally.
6. Atom bookkeeping: an autogen package's atom goes into `<kit>/merge.kit.d/base_autogen.yml`
   **and its old atom is removed from `merge.kit.d/base.yml`** — the legacy file must list only
   what still needs conversion. **Exception: ai-kit is autogen-native** — atoms live directly in
   `ai-kit/merge.kit.d/base.yml`, and autogen-thin there runs with `--kitfile merge.kit.d/base.yml`.
7. Parallel version tracks (e.g. 19c and 21c of one product): duplicate the merge atom with
   `conditions:` ranges (patterns: nvidia-cuda 11/12/13, mesa `<26`/`>=26`, oracle 19/21).
8. Render-time template macros (`{{ .Values.pv }}`, `{{ .Values.p }}`, `regexReplaceAll …`) —
   NOT shell variables like `${PV}` — so generated ebuilds contain literals and read clean.
9. House platform stance: **no multilib** (plain `autotools`/`cmake` eclasses, `cmake` supports
   EAPI 7), **no FreeBSD** in `mark-*` branches, prefer simplicity when asked (dropping doc/test
   USE + `src_test` is welcome; docs are read online; tests belong in workflows).
10. Zip assets: add `app-arch/unzip` to `bdepend` explicitly. Don't spell out defaults
    (`query: releases` can be omitted).
11. Rust/cargo packages: use the `rust` extension (`extensions_defs: rust` + package-level
    `extensions: - rust`) which vendors a `mark-rust-bundle-${PN}` tarball — precedent:
    `thin-provisioning-tools` in core-server-kit; the ebuild inherits `cargo` and calls
    `cargo_src_unpack`.
12. Version bumps of build-tool deps (meson cap etc.) are legitimate fixes — check for
    `selector:` caps freezing a generator first, and mind the toolchain floor of the new version
    (meson 1.12 requires python ≥3.10 → cap raised only to `<1.12`).
13. Merges-only packages (just an atom in `merge.kit.d/base.yml`, no spec) that need ebuild
    changes get **converted to autogen** in the same PR (this is the accepted standard, see
    libdbi-drivers/gtest/libedit/xvid/mpg123 merged PRs); verify the generated ebuild matches the
    current tree semantics (same phases/deps; Manifest DIST bytes identical where applicable).
14. Use the dedicated spec keys, never hand-write the ebuild equivalent in `body`: `cdepend:` →
    renders `CDEPEND` (and reference it from `rdepend`/`depend`), `patches:` → renders the eclass
    `PATCHES=( "${FILESDIR}/…" )` array, `iuse/license/required_use/extra_envs:` likewise. Beware:
    switching hand-written `PATCHES=` to `patches:` makes previously-inert patches **live** —
    `patch --dry-run` them against every version track first and use per-track `patches:` (a
    `patches: []` track is legitimate when upstream absorbed the fix).
15. Never set an `eapi` var (the template applies it). Prefer `body:` over `body_raw:` (body runs
    a per-line leading-space-pair→tab normaliser, which is the house render style). Template
    variables inside `body:` need the `$` form — `{{ $php_mv := first ( splitList "." .Values.pv ) }}`
    then `{{ $php_mv }}` — and `extra_envs:` entries are also rendered in the values context (pv
    available), but a render-var in `body:` is the maintainer-preferred style.
16. `exclude:` is an **atom-level** key (inherit it via `defaults:` so every version track gets
    it). A `github:`-section `exclude:` is silently ignored by current mark-devkit (no field on
    the github props struct; verified by discriminating probe 2026-10-25) — where the maintainer
    asks for that placement anyway, keep both and prove which one filters. Whenever placement or
    regex semantics of a generator key is in doubt, run the probe: exclude the currently-selected
    version and check the selection steps to the next candidate.
17. Prefer a simple `string` transform over regex when the release titles allow it (match the real
    title format — read a few titles first: "libnl 3.12.0" needs only `match: 'libnl '`).
18. Provenance comments in generated ebuilds say **Mark**, never Gentoo/Funtoo ("Drop old Gentoo
    reference and replace them with Mark"). USE flags gating packages not in the tree get
    simplified to the house equivalent (no `libressl` in tree → plain `ssl? ( dev-libs/openssl:= )`).
19. Org commits carry an attribution trailer block (requested on macaroni-os/ai-agent-playbook#1):
    `LLM-assisted-by: <model id (size, location)> via <agent harness>` plus
    `Signed-off-by: <the operator's git identity>` at the end of the commit message. Exact
    keywords are the maintainer's call — but the identity is always the human's configured one
    (§2), never one the agent made up.
20. Version selection at generation time is driven by the SPEC's `selector:` — autogen-thin ignores
    the merge atom's `conditions:`. That does NOT make `conditions:` optional: it drives the
    post-generation phases — the **clean** phase (keeps a specific release available instead of
    comparing ebuild counts across all versions) and the **merge phase onto the other release
    branches**. When a version bound matters, write it in BOTH places. New packages additionally
    need merge-phase PRs for `mark-xl`, `mark-iii` and `mark-31`, chosen per branch audience:
    `mark-xl` is container/server-oriented; `mark-iii`/`mark-31` carry those packages *and* desktop
    stuff. (Maintainer note on macaroni-os/ai-agent-playbook#2.)
21. mark-devkit 0.35.4 field quirks: `github.per_page` is broken (off-by-one `page < NumPages`
    yields zero tags — don't set it; raise `num_pages` only: GitHub's tag page size is 30, so
    older maintained lines need ~12 pages, openssl uses 10). Template auto-applies `EAPI` — never
    set an `eapi` var. `exclude:` parses on the ATOM (github-section placement is silently
    ignored — proven by discriminating probe, and the maintainer accepted the evidence).
22. No `#` comments inside ebuild dep strings (portage rejects them): dependency rationale goes
    in YAML comments above the spec key, not in the rendered dep line.
23. PR bodies reference their issue in the house form from the mark-issues README — a standalone
    line `Closes: macaroni-os/mark-issues#<N>` (or `Bugs: macaroni-os/mark-issues#<N>`). Not a
    bare `Fixes <full issue URL>`: it works mechanically but is not the convention. GitHub
    auto-closes the issue on merge for EVERY PR carrying `Closes` — so when one issue needs
    several PRs (dependency-layer enabler + the actual fix), put `Bugs:` on the enabler and
    `Closes:` only on the completing PR, or the issue dies with the first merge. (It has fired
    for real: the aws-crt-python enabler closed the awscliv2 issue on merge — the recovery is a
    status comment on the issue pointing at the still-open completing PR, not a silent reopen.)
24. `github.match:` (package/cluster level) filters candidate versions as a regex on the
    NORMALIZED version (leading `v` already stripped — `match: '^1.*'` matches tag `v1.2026.9`
    and excludes `v8059`, proven by removing the selector and probing). Maintainer prefers it
    over selector bounds for upstreams with broken/mixed version histories (plantuml: legacy
    v8059/v2017.x tags sort above the modern 1.x line). Selector bounds remain the right tool
    for compatibility caps with real semantics (openvdb `<12.0.0` vs openimageio).
    Beware (freetds probe 2026-10-30): `github.match:` is evaluated against the RAW release
    title, NOT the title-transformed candidate — combining the two yields "No versions found".
    Prerelease/tag-page garbage is already filtered natively; don't add `match:` to rescue it.
25. Without an explicit `github.assets:` override the generator falls back to the
    `api.github.com` auto-generated source tarball, whose name differs from the uploaded asset
    (freetds: uploaded `freetds-<ver>.tar.bz2`). Set `assets:` when the uploaded asset name/
    hash matters — and probe which one the selection actually fetched. When the needed source
    asset carries a BUILD COUNTER not derivable from the tag (Firebird: tag `v4.0.7` but asset
    `Firebird-4.0.7.3271-0.tar.xz`, and the tag auto-tarball lacks pre-generated `configure`),
    bind it with `github.assets[].matcher` regex — the matcher resolves to the asset's
    `browser_download_url` (verified in mark-devkit `GetAssets`) — rename the distfile to
    `${PN}-${PV}` in `s:`/body and `post_src_unpack` mv the real directory.
26. `github.assets[]` entry: `matcher:` and `url:` are **alternatives, not complements** —
    `matcher` picks an asset from the release's asset list, `url` pins the download URL directly
    (maintainer note on kit-fixups#510, 2026-10-30: "This is not needed"). Carrying both is
    redundant; keep whichever the source actually needs and prove the generated `SRC_URI` is
    unchanged when removing one (freetds probe: dropping `matcher` alongside `url` → generated
    ebuild byte-identical).
27. **No `${EPREFIX}` in new autogens** (maintainer convention, kit-fixups#507, 2026-10-30):
    custom-path installation is no longer supported — users needing it use an `anise`-setup
    chroot. Write plain `/usr`, `/etc/...`, `/var/...` in spec bodies. Pre-existing `${EPREFIX}`
    in base/restored lines is not this PR's business (keep verbatim; a sweep is a separate ask).
    Verify purges against the **generated** ebuild: sed the old baseline with the same
    substitution and diff — byte-identity proves the change touched nothing else.

## 4. Fix-pattern quick table

| Symptom | Standard answer |
|---|---|
| `'bool'/'true'/'false' … keyword`, implicit-decl, `f()` takes no args | `append-flags -std=gnu17` in `src_prepare` + comment citing the issue |
| missing `#include <cstdint>`/`<sys/wait.h>` in C++/C | upstream fix exists? bump via tracking generator; else small patch via `files_dir` |
| shipped-but-unused patch files in `files/` | wire into `patches:`/`PATCHES` (classic bug class) |
| upstream dead, tarball only | `builtin-dirlisting` on the listing page (anchor matcher, exclude junk) or pinned noop |
| release assets renamed upstream (e.g. `linuxstatic`→`linux`) | fix matchers in spec, verify asset names on the live release |
| package needs newer build tool (meson/cmake floor) | raise the tool's autogen `selector` cap (check the tool's own deps!) |
| USE flag for software not in tree | rename to house convention, add real dep via follow-up PRs (oracle pattern) |
| deprecation `#warning` + `-Werror` | remove the unused deprecated include (upstream commits often exist) |
| parallel-build race upstream-unfixed | serialize the racy phase (`-j 1` guard) with upstream-issue citation |
| CMake `find_package(X CONFIG)` fails | is the provider even packaged? declare the DEPEND (pybind11 case); if no tree package provides it (gdk-pixbuf-xlib case) constrain the USE combo via `REQUIRED_USE` and document — never leave a silently-unbuildable combination |
| `aclocal: You must have libxml2 …` after a libxml2 upgrade | libxml2 ≥ 2.12 stopped installing the `AM_PATH_XML2` m4 macro — replace with `PKG_CHECK_MODULES([XML2],[libxml-2.0])` (dvdauthor pattern); the error text is a red herring, libxml2 IS installed |
| PEP517 wheel build imports the package itself (autocomplete/index generation) | every runtime dep is also a BUILD-time dep — declare them all (`_build_ac_index` awscliv2 pattern); a wheel phase failing on ModuleNotFoundError of a "runtime-only" dep is a spec bug |
| Python dep dies at build with `TypeError: unsupported operand type(s) for \|` | that dep's own `Requires-Python` exceeds the distro interpreter (urllib3 ≥2.7 wants ≥3.10 vs tree py3.9; PEP604 unions evaluated at import). Version caps are then MANDATORY, not conservatism — prove it once by removing the cap and building (capture the TypeError), encode the cap with the rationale as a YAML comment (never in the dep string, §3.22) |
| `.jar` (or other zip-magic) distfile explodes into WORKDIR at unpack | portage auto-unzips by file magic — `src_unpack() { cp "${DISTDIR}/${A}" "${WORKDIR}"/ \|\| die; }` and install the intact archive (jq-bin/plantuml pattern) |
| custom-make package dies with a bizarre guard error only under portage (e.g. Firebird examples `ifndef ROOT` → extauth fatal) | Portage exports `ROOT=/`; upstream makefiles using bare `ROOT` collide — `unset ROOT` in `src_compile` (or `emake ROOT=`) and comment-cite the issue |
| upstream ships staged libs as absolute libtool symlinks into the build tree (libtommath/libtomcrypt pattern) | `cp -L` the real objects at install time — clean/merge phases dereference dangling symlinks later |
| consumer fails `#include <pkg/sub/...>` after install although `doheader` ran | the API headers live in a SUBDIR (`include/firebird/`) that `doheader include/*` skips — recursive `doins -r include/<subdir>` into `/usr/include` |
| old ebuild's SRC_URI host is dead (01.org → intel.com redirect) but the package still "exists" in-tree | check fetchability BEFORE treating the ebuild as functional; upstream may have consolidated (dleyna split repos → gitlab.gnome.org/World/dLeyna monorepo) — follow Gentoo's consolidated replacement package, add `!old/packages` blockers, and file the kit purge PR alongside |

## 5. Verification standard (per PR)

1. Reproduce failing build; capture exact error lines.
2. Fix in the meta-repo kit copy → `ebuild <file> manifest` → `emerge --quiet-build -1 =cat/pkg-ver`
   → **0 errors** → `qlist -Iv` → functional smoke (binary `--version`/`-V`, link test, upstream
   `runtests.sh`/ctest, rebuild one cheap reverse dep if <5 min).
3. Port the change to a kit-fixups branch → verify the **generated** ebuild via
   `mark-devkit autogen-thin --specfile autogen.kit.d/base.yml --kitfile <merge file> --pkg <PN>`
   (PN not spec-key; `GITHUB_TOKEN` via `incus exec --env`) → check selected version, `SRC_URI`,
   `post_src_unpack`/`S` → deploy generated ebuild → build again.
   **Merged-into-kit-fixups ≠ in-the-tree**: when a PR B depends on a dependency your PR A just
   added to kit-fixups, A's ebuild is NOT in the kit until the MARK bot's merge/clean phase runs
   (may be days). To verify B meanwhile, regenerate A's package from A's merged spec in a scratch
   worktree (`git worktree add --detach /tmp/a-wt upstream/mark-unstable` → autogen-thin → deploy
   its ebuild + files_dir into the kit copy) rather than blocking on the bot. (awscliv2 #492 vs
   aws-crt-python #491.)
4. Restore sandbox state (git checkout per kit, remove test dirs, shred temp token files).
5. Only then: PR / review replies with the captured evidence (PR body carries the
   `Closes:`/`Bugs:` line per §3.23).

## 6. Subagent operating model (if your agent supports them)

- **Sequential, one issue per foreground subagent.** Shared build container + review-focus means
  parallelism buys nothing and corrupts evidence attribution.
- Each briefing includes verbatim: §2 exec/credential recipes, §3 conventions (numbered list is
  compact), §5 checklist, reference PRs to imitate (`gh pr diff <N> --repo macaroni-os/kit-fixups`
  works for merged ones), and the escape hatch: “stale/maintainer-decision → STOP, report with
  evidence, propose an issue comment instead of a PR.”
- Fixed report format keeps them accountable: `STATUS | PR_URL | EVIDENCE | NOTES`.
- Review-fix runs: pass exact comment IDs and bodies; require **threaded replies**
  (`gh api -X POST repos/macaroni-os/kit-fixups/pulls/N/comments/<ID>/replies`), one-commit
  amend + lease-push, and re-verification after the change.
- Feed every newly learned convention back into this playbook (it is the loop's memory).
- **Orchestrator verification is not optional**: after every subagent that touched anything
  public, re-read the public state via API (PR head sha, commit count, author, thread
  answered-count, **file list via `pulls/<n>/files`**, issue comment rendered) and check it
  against the report. Subagent `rc=0` and "pushed successfully" lines have both been wrong
  while the report was still honest — and an honest report can still omit what the commit
  INCLUDED (see §7m).
- Review asks that contradict observed tool behavior (maintainer requests a key placement that
  probes show is inert): implement the ask **and** run a discriminating probe, reply with the
  evidence, offer both end-states, let the maintainer choose. This has converted a wrong instruction
  into "Yeah, correct. my fault." plus a settled convention — evidence-based review replies are
  welcome; silent deviation is not.

## 7. Known agent failure modes (all actually observed — check before assuming gremlins)

a. `gh api -f body=@file` does **not** expand `@file` → the literal path gets posted publicly.
   Use `-f body="$(cat file)"` or `gh pr comment --body-file`. (Caught by human PR review; fix =
   delete comment + repost real content as threaded replies.)
b. Shell-quoting slips echoing `$(gh auth token)` into transcripts → rotate (`gh auth refresh`).
c. EOF-append rebase conflicts in `<kit>/autogen.kit.d/base.yml` (several PRs append the same
   file end): resolve = keep both blocks, strip ALL conflict markers, then **verify**:
   `grep -cE "^(<<<<<<<|=======|>>>>>>>)"` must be 0, `yaml.safe_load` must pass, diff your block
   against the pre-rebase version (watch trailing nested list items getting eaten), amend to one
   commit, lease-push, re-run autogen-thin as functional proof.
d. `sed -n 'A,Bp'`/line-number edits drift after a rewrite — prefer pattern-based deletion.
e. Agent claims “verified X” in the same message that performed no verification — re-run the claim
   before it becomes a public reply.
f. Container dirt accumulates (stale workdirs, test dirs, leftover ebuilds) — restore per §5.4;
   it poisons the next reproduce.
g. Template render errors in `body:` are only WARNINGS → a silently truncated ebuild (observed: a
   3.7 KB output where ~700 lines are expected). Always validate every generated file: size/line
   count vs baseline, `bash -n`, grep for `{{` residue. Template assignments need the `$` sigil —
   `{{ $x := … }}` — otherwise the render fails with `function "x" not defined` (see §3.15).
h. GitHub CLI/git mechanics biting real runs: `gh pr create --head owner:repo:branch` is rejected
   for cross-named forks (cli/cli#10093) → create the PR via
   `gh api repos/<org>/<repo>/pulls --input <json>` with the body from a file via jq `--rawfile`;
   git ≥2.49 in containers may silently ignore netrc-style `/root/.git-credentials` → use
   inline-URL credentials or a shred-after askpass helper, push output silenced.
i. Agent INVENTS a git identity — plausible-looking name + domain never supplied by the
   operator (or lets identity drift between branches within one session: GitHub-noreply vs
   org alias vs invented). Both observed 2026-10-25. Prevent per §2 (mirror host → container
   once; read, never guess). Repair while PRs are OPEN: amend author + lease-push, then verify
   via `gh api .../pulls/<n>/commits` that the head author moved — and check which repo/branch
   the PR head actually tracks first (intra-repo vs fork PRs — see the #489 lesson; an amended
   branch the PR does not track fixes nothing). NEVER rewrite merged history.
j. `git push` returns **rc=0 and pushes nothing** (observed: transient auth oddity during the
   openvdb session). Trust nothing but verification: after every push, `git ls-remote` the branch
   AND check the PR head via `gh api repos/<org>/<repo>/pulls/<n> --jq .head.sha`. Take the lease
   value from a FRESH `ls-remote` at push time, never from memory — an incorrectly remembered full
   sha silently aborts a legitimate fix-push.
k. Flag-order self-destruct: `git commit -qam --amend -m …` consumes `--amend` as the `-m`
   **message** → commits titled `--amend` and/or an unintended second commit on a
   single-commit PR branch (hit twice in one day, both times the push looked successful). After
   any amend chain on a PR branch, verify `git log --format='%h %s' <base>..HEAD` shows exactly
   one commit with the intended subject.
l. Counting only `issues/<n>/comments` misses review feedback: inline comments live in
   `pulls/<n>/comments`, and a maintainer submitting them produces `pulls/<n>/reviews` entries
   with EMPTY bodies (the content is in the comment list, timestamped at submit). A sweep that
   checks one or the other reports "all clear" while 3 new threads wait (observed on PR #485
   round 4 and macaroni-site #15). Sweep all three endpoints, always.
m. `git add -A` in a kit-fixups checkout that has a live autogen `workdir/` commits GENERATOR
   OUTPUT — generated ebuilds, Manifests, multi-MB distfile tarballs — straight into the PR
   (observed: a 4.6 MB openvdb tarball + 4 artifact files reached a pushed commit; `git diff
   --stat` before the add doesn't save you, `git show --stat` can scroll past it). Either rm the
   workdirs before staging or stage targeted paths; ALWAYS audit the pushed PR via
   `pulls/<n>/files` (§6). Repair without rewriting anything public-visible in semantics:
   `git rm -r --cached <path>` + `--amend --no-edit` + lease-push.
n. A subagent can DIE mid-task and return corrupt output (tool-call fragments, no report) —
   but its sandbox work is usually intact and salvageable. Before re-dispatching anything,
   inspect actual state: branch-vs-remote diff, wip commits, `/root/*.log` tails, PR head sha,
   kits porcelain. One crashed review-round agent had already made both spec edits AND both
   full builds (openvdb's 40-min compile included); finishing its push/reply phase manually
   cost minutes versus redoing everything.
o. Whitespace is NOT inert in spec `body:` fields: the templates normalize space-pairs to tabs,
   so "cosmetically restoring" blank lines that a previous edit removed changed the generated
   ebuild (`\tcode` → `\t code`). If the generated ebuild from the current spec is byte-identical
   to the artifact you actually built, the whitespace state IS the verified state — don't churn
   it. Judge formatting only by what lands in the generated ebuild.

## 8. Feedback sweep (do it between every task)

```
gh api "repos/macaroni-os/kit-fixups/issues/<N>/comments?since=<ISO>"
gh api "repos/macaroni-os/kit-fixups/pulls/<N>/comments?since=<ISO>"
gh api "repos/macaroni-os/kit-fixups/pulls/<N>/reviews?since=<ISO>"   # empty-body COMMENTED = inline arrivals (§7l)
gh pr view <N> --repo macaroni-os/kit-fixups --json state,mergeable
```
…for every open PR of yours (also the kit repos!). Track the last-check timestamp. Review replies
the same day, each in its own thread, with fresh evidence. When your fix merges, post a
close-suggestion on the linked mark-issue (fix summary + “ready to close once it propagates to
the release”). Expect maintainers to ask for: squashes, cap changes, generator swaps, platform
simplifications — §3 covers the standing versions of those asks.

## 9. What NOT to delegate to the loop

Version policies with API breaks, default-python/compiler toolchain migrations, whole new kits,
RFC issues, package removals touching >1 kit's policy — produce evidence-rich reports and options
instead, and let maintainers decide (they respond fast and precisely on GitHub).
