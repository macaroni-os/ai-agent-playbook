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
- The `macaroni/mark-unstable` image family has **no `/root` at first boot** (same quirk as
  phoenix-builder, §10.1): every redirect into `/root/` dies with `No such file or directory`
  and silently kills the detached command whose log it was (observed 2026-11-12: the first
  `ego sync` never started). `mkdir -p /root` before anything writes there. And `incus launch`
  inside the piped-`newgrp` wrapper **swallows the remaining stdin** — the next line is parsed
  as instance config (`cannot construct !!str … into api.InstancePut`) — append `</dev/null`
  to `incus launch`. Where the operator's account has direct incus access, plain
  `incus exec <c> -- bash -c "…"` needs no newgrp pipe at all — probe once, per machine.
- `incus file pull/push` (SFTP) may fail (307 redirect / missing agent). Workaround: base64 in
  chunks (`echo <B64> | base64 -d > f`); huge single-arg base64 hits ARG_MAX — split.
- MARK's `emerge` rejects bare `=cat/pkg` atoms: use exact versions or plain atoms.
- Handy tools: `qlist -Iv <cat/pkg>`, `qdepends -Qt <pkg>` (reverse deps), logs under
- **Non-trivial container work: don't triple-nest quotes.** Write the script on the host, transfer
  it `base64 -w0` → `echo <B64> | base64 -d > /root/x.sh` → `bash /root/x.sh; shred -u /root/x.sh`.
  Every quoting bug (silent host-side `$()` expansion, empty `$GT` from over-escaping) vanished
  when we switched to this for multi-step push/verify scripts. The rule also covers "simple"
  one-liners: when the assembled host command is double-quoted, any `$f`/`$?` inside
  `bash -c "…"` expands HOST-SIDE (observed: literal `DONE-RC=$?` recorded in a log, empty
  `== ` lines) — once a container command touches `$`, go base64-script.
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
- **`macaroni/mark-experimental` is a venue, not a kit branch** (maya #932 feedback round, 2026-11-19):
  the Incus image from the `macaroni` remote syncs all 34 kits on `mark-unstable` (`ego sync`, `release =
  mark-unstable` in `/etc/ego.conf`). Its python 3.11 is the **mark-stages `python_upgrade` hook**
  (`jobs/hooks/extras/python_upgrade.yml`, geaaru 2026-11-19) applied at image-build time over those
  kits — and **`ego sync` REVERTS its tree-side edits**: verified in-container — after sync,
  `core-kit/profiles/funtoo/kits/python-kit/mark/make.defaults` reads `PYTHON_TARGETS="python3_9"`
  again (kit git clean), while the installed `dev-lang/python-3.11.16`, the `eselect python set
  python3.11` pick and `/etc/portage/package.unmask` persist. Recovery = the hook itself
  (`python_version2upgrade: 3.11`; the LIVE hook is now `3.13`-targeted — full fresh-container
  reproduction sequence in §3.61, and beware the drift check there before trusting python smokes):
  sed make.defaults, unmask, `emerge -j python python-exec
  --nodeps`, `locale-gen`, rebuild `gobject-introspection glib --nodeps -j`,
  `PYTHON_TARGETS=python3_11 PYTHON_SINGLE_TARGET=python3_11 emerge -DuvN @world`, eselect-set,
  then unmerge python:3.9 + `rm -rf /usr/lib/python3.9`. Specs that inherit NO python eclass and
  declare slot-less dev-python RDEPENDS are insensitive to the revert — #628's amended build proved
  green post-sync with `python3_9` targets back. Two traps: the image defaults **gcc 16.2.0, which
  nvcc ≤13.2 rejects** (`unsupported GNU version! gcc versions later than 15`) — `gcc-config` to the
  shipped 12.x before any CUDA configure; and the image ships **no /root, no meta-repo, no
  mark-devkit** — bootstrap order: `mkdir -p /root`, detached `ego sync` (sentinel), then
  `emerge mark-devkit nvidia-cuda-toolkit` (license seed §2). Cross-tree proof that works: gen the
  same spec in both containers and assert `sha256sum` of the generated ebuild+Manifest are equal.
- **Background emerges die on interaction and masks** (strata #933 pre-stage, 2026-11-19): a detached
  `emerge -1av` never prompts — `--ask` in a non-terminal silently kills the install; masked licenses
  (e.g. `NVIDIA-CUDA` for `dev-util/nvidia-cuda-toolkit`) abort in the resolver instead. Pre-staged
  dependency installs must drop `-a` AND seed `/etc/portage/package.license` first, and the
  pre-stage script should end with a `qlist` assertion + `DONE` sentinel, never rely on rc alone.
- **Never `shred -u` a script that bash is still executing** — the running wrapper's inode vanishes
  and the detached chain dies mid-phase without an error line. Detach a small launcher instead.
- **`autogen-thin` with `GITHUB_TOKEN` unset fails SILENTLY** (patch #601, 2026-11-15): logs
  `Missing git token! Push interrupted!` but **exits 0 and produces no output dir** — a generation
  "success" with zero artifacts. Always assert the generated ebuild file exists (and is non-empty)
  after every autogen-thin run, independent of rc, and treat an empty `--to` dir as a token-plumbing
  failure (§2 `--env` position trap), not a generator bug. Same round's §7e-on-tooling near-miss: an
  extraction that yields a **0-byte file** made a "clean" render-audit verdict vacuous — assert the
  artifact backs the check BEFORE trusting a green (re-derive from raw, cf. §7v).
- Detached-build polling: `pgrep -f "emerge --quiet-build"` ALSO matches the `bash -c` wrapper
  of the very exec call that runs the pgrep (the pattern is in its own command line) → phantom
  "STILL-RUNNING". Trust sentinels instead: log tail sentinels (`>>> Installing`, `Calculating
  … done!`) plus `qlist -Iv <cp>` for ground truth.
  `/var/tmp/portage/<cat>/<pn>-<ver>/temp/`.
- `mark-devkit` (install/build from source if the image lacks it): **use `autogen-thin`** for
  spec verification. Full `autogen` runs the merge phase, which needs `anise-portage-converter`
  (often absent; the `app-misc/anise` currently installable provides only the luet CLI).
  **`autogen-thin` does not run extensions**: Rust/Go bundles (`mirror://macaroni/…-mark-*-bundle-…`)
  are only produced by full `autogen --skip-merge --keep-workdir --sync=false` (the merge phase is
  skipped, so no converter is needed). `--sync=false` is mandatory locally — the default uploads
  generated artifacts to the shared S3/minio mirror (§3.44).
  **Correction (mark-devkit 0.36.1 probe, 2026-11-10):** `autogen-thin --to <dir>` DOES run the
  golang extension bundle build for a cluster that declares it — the whole proxy closure
  downloads (~20 min/repo) and the repacked bundle tarball drifts ±200 B in size/digest per run
  (the known nondeterminism) while distfile NAMES stay stable; budget for it on every spec
  iteration. Confirmed for the **Rust** extension on the same 0.36.1 (#865 net-fs/rustfs,
  2026-11-11): autogen-thin produces the `mark-rust-bundle` tarball AND its generated Manifest —
  because of the re-tar drift, re-stage bundle + Manifest into DISTDIR **as a set**, never one
  against the other's older digest.
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
  (`sed 's/gho_[A-Za-z0-9]*//g'`). `set -x` in a wrapper holding an inline-token URL writes the
  live token to the log in plaintext (observed: `/root/p4_push.out`) — audit any old evidence
  file with a pattern count (`grep -c 'gho_'`) BEFORE `cat`-ing it, and never `cat` an
  unproven file into a tee'd inspect log. If a token ever appears in logs: rotate immediately
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
   default `src_unpack` behavior where possible. Set `s:` ONLY when the unpack root differs from
   portage's default `${WORKDIR}/${PN}-${PV}` — a source-tarball flavour usually needs NOTHING
   (archive root == `${P}`); the versionless wrapper dir of a `-bin` asset is the case that
   needs it (maintainer words: "This is the default. It's only needed for pi-bin",
   macaroni-os/kit-fixups#512 review, 2026-10-31).
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
   EAPI 7), **no FreeBSD** in `mark-*` branches, prefer simplicity when asked. **ALWAYS drop
   `test` USE + `src_test` (and `restrict` added for it) in EVERY new autogen** — standing rule,
   not a courtesy (geaaru, kit-fixups#586/#599/#601/#617, 2026-11-15: "Always drop `test` USE
   for every new autogen. Add a note in the playbook"); docs read online, tests in workflows.
   **PYTHON_COMPAT is always `( python3+ )` — python 2.x is not supported anymore** (geaaru
   ruling on file #603 after a loop left the tree's `python2+` range verbatim: "Just set always
   `python3+`").
   Ported bodies drop `userland_BSD`/`elibc_FreeBSD`/`elibc_Darwin` **and `kernel_FreeBSD`/
   `kernel_Darwin`** conditional branches verbatim (#601/#617; the `kernel_*` variant came from
   geaaru on #608: "drop kernel_FreeBSD support, please").
   **Always prefer an ebuild WITHOUT multilib support** — when a tree ebuild still carries
   `multilib-minimal`/per-ABI machinery, collapsing it to a plain build is part of the
   conversion, not an optional extra (maintainer ask on macaroni-os/kit-fixups#587 review,
   2026-11-15; ncurses #923 hit the corollary too: the eclass pre-creates per-ABI `BUILD_DIR`
   which a plain build then lacks — re-create what the eclass was providing or the phases break).
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
18. Provenance strings in generated ebuilds: NEVER Gentoo/Funtoo, and the packager/distro
    string is **`MacaroniOS`** — geaaru 2026-11-15 (kit-fixups#586/#599): "I prefer to use common
    `MacaroniOS` string instead of `Mark`" — superseding earlier rounds' `Packaged by Mark`
    (and the #567-era split where only DISTRO STAMPS used MacaroniOS: now everything does,
    e.g. `--with-modified-by=MacaroniOS-...`). USE flags gating packages not in the tree get
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
    branches**. When a version bound matters, write it in BOTH places. **HARD RULE (geaaru, 2026-11-16, #593/#603/#605 — his #605 note asked for this entry):
    a merge atom with N release-line `conditions:` REQUIRES the autogen definition to carry N
    `packages:` entries with matching `selector:` bounds — EVERY served line is GENERATED by
    the spec, the old line is not left as a frozen tree ebuild. sysvinit #608 (own key,
    `<2.99` + `>=3.0` entries) and patch #601 (dual same-PN entries) are the working shapes.** New packages additionally
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
    The same auto-close fires from a `Closes:` line in the **commit message** once that commit
    reaches the default branch (caught pre-merge on macaroni-os/mark-devkit#9: the feature commit
    said `Closes: macaroni-os/mark-issues#848` while the kit-side swap was still pending — amend
    to `Bugs:` and lease-push before review settles).
24. `github.match:` (package/cluster level) filters candidate versions as a regex on the
    NORMALIZED version (leading `v` already stripped — `match: '^1.*'` matches tag `v1.2026.9`
    and excludes `v8059`, proven by removing the selector and probing). Maintainer prefers it
    over selector bounds for upstreams with broken/mixed version histories (plantuml: legacy
    v8059/v2017.x tags sort above the modern 1.x line). Selector bounds remain the right tool
    for compatibility caps with real semantics (openvdb `<12.0.0` vs openimageio).
    Beware (freetds probe 2026-10-30): `github.match:` is evaluated against the RAW release
    title, NOT the title-transformed candidate — combining the two yields "No versions found".
    Prerelease/tag-page garbage is already filtered natively; don't add `match:` to rescue it.
    SCOPE: `match:` is GITHUB-section only — `AutogenDirlistingProps` has NO Match field in
    mark-devkit 0.36.1 (struct-proven, sys-devel/make macaroni-os/kit-fixups#611): a `match:`
    inside a dirlisting block is SILENTLY INERT; use `dir.exclude` there. Discriminating probe
    (uncapped vs capped selection) validates whichever key you write.
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
    (maintainer note on macaroni-os/kit-fixups#510, 2026-10-30: "This is not needed"). Carrying both is
    redundant. **Maintainer's settled preference (geaaru, same PR thread, later that day): keep
    `matcher`, drop `url`** — "less text is better" and upstream URL changes are easier to fix.
    Prefer `name: '{{ .Values.p }}.tar.<ext>'` over `{{ .Values.pn }}-{{ .Values.version }}` —
    `.Values.p` explodes to `pn-PV` **without** any `-rX` suffix (verified: `revision: 1` yields
    `pkg-1.0-r1.ebuild` while the dist stays `pkg-1.0.tar.bz2`), so the same pair survives bumps.
    Matcher semantics (freetds probes 2026-10-30): **regex, NOT anchored** — substring matches
    select the asset (bare `freetds` matched); unescaped dots match literally-different chars
    (`freetds.1.5.16.tar.bz2` hit `freetds-1.5.16.tar.bz2`), so escape your dots; zero-match fails
    with log line `no asset found for matcher <X>` and no ebuild (run a negative control to prove
    the matcher filters). Matcher resolves to `browser_download_url` of the uploaded asset →
    prove any such swap by regenerating and diffing against the baseline ebuild (freetds:
    byte-identical).
27. **No `${EPREFIX}` in new autogens** (maintainer convention, macaroni-os/kit-fixups#507, 2026-10-30):
    custom-path installation is no longer supported — users needing it use an `anise`-setup
    chroot. Write plain `/usr`, `/etc/...`, `/var/...` in spec bodies. Pre-existing `${EPREFIX}`
    in base/restored lines is not this PR's business (keep verbatim; a sweep is a separate ask).
    Verify purges against the **generated** ebuild: sed the old baseline with the same
    substitution and diff — byte-identity proves the change touched nothing else.
28. **Server services use STATIC uid/gid** (maintainer ask, macaroni-os/kit-fixups#511): read the ids from
    `macaroni-commons/packages/entities/groups.yaml` (the `entities` tool's list, per-service
    `gid:` + `user.uid:` + homedir/shell). If the account is missing there, PR it FIRST, then use
    it — never `enewuser <svc> -1 -1` for a server. Macaroni eclass `enewuser` =
    `user uid shell home groups` — NO gid slot and a literal `/sbin/nologin` shell dies; the
    static gid lands via a pre-created `enewgroup <svc> <gid>` (ntp/epmd precedent), homedir stays
    `/dev/null` with data dirs handled by `diropts`/`keepdir`.
29. **Init system is USE-SELECTED, not additive** (maintainer ask + follow-up, macaroni-os/kit-fixups#511):
    `if use systemd ; then systemd_dounit ...; else newinitd ...; fi` — exactly ONE init system
    installs (geaaru explicitly moved `newinitd` into the `else`). Prefer the unit from the
    upstream tarball (`*.service.in` — instantiate the @VARS@), `Type=` matched to how the daemon
    actually forks (fbguard pattern); proof standard: one merge per USE side, `qlist` shows only
    the unit OR only the initd.
30. `simple-github.tmpl`'s generated `post_src_unpack` is unconditional `mv <user>-<repo>-* ${S}`
    with no override var — WRONG for matcher-pinned uploaded assets whose tarball root differs
    (firebird: internal dir `Firebird-4.0.7.3271-0`, so `FirebirdSQL-firebird-*` never matches;
    `tar tjf` is the ground truth). Keep `simple.tmpl` + hand-written `post_src_unpack` for such
    assets and show the probe when asked to switch (§6 both-end-states reply).
31. Shared-upstream flavour pairs (`<pn>` + `<pn>-bin` built from the same repository) get a
    **dedicated autogen cluster key** whose `defaults:` define the common parts once — github
    user/repo, `template`, and the shared `restrict`/`extra_envs` — while each entry under
    `packages:` keeps only its flavour-specific keys (maintainer ask on macaroni-os/kit-fixups#512,
    geaaru, 2026-10-31). `defaults:` IS an atom (mark-devkit `AutogenDefinition.Defaults` is
    `*AutogenAtom`), so vars-style keys belong under `defaults.vars:` — merged per-key with each
    package's own `vars`; placed directly under `defaults:` they are silently INERT
    (discriminating probe: the regenerated ebuild loses RESTRICT/QA_PREBUILT). Proof standard:
    both flavours regenerate byte-identical after the move (verified on
    macaroni-os/kit-fixups#512).
32. **Every extra distfile must be an `assets:` spec key — USE-conditional ones included**
    (maintainer ask, macaroni-os/kit-fixups#513, 2026-10-31): a top-level `SRC_URI+=` append in
    `body` is invisible to the generator's Manifest — the autogen-generated Manifest silently
    lacks everything behind a USE. Express optional module tarballs as
    `assets: [{name, url, use: <flag>}]`; pin the versions/shas as `vars:` (`ndk_version: 0.3.3`)
    and interpolate them `{{ .Values.ndk_version }}` into `url:`/`name:` (both render in the
    values context — lang-kit go/zig-bin precedents), so the generated ebuild carries literals
    and a bump has ONE source. Two sharp edges: **the main release asset must be re-listed as
    the first entry once the assets list is non-empty** (mark-devkit does NOT keep the release
    tarball implicitly — maintainer words), and when the distfile name and the pinned commit
    differ, use two vars (name: `push_stream_pv`, url/workdir: `push_stream_sha`). Never assume
    a release ASSET equals the tag ARCHIVE — in either direction: nginx's official assets ship a
    generated top-level `./configure` and `man/`, its tag archives have neither (they carry
    `auto/configure`, `docs/man`); firebird was the reverse case (§3.25). Probe the tarball you
    pin.
33. **Autogen conversion PRs carry the files tree too** (macaroni-os/kit-fixups#513): the
    CURRENT convention is a **flat package-named directory** — cluster `files_dir:
    '{{ .Values.pn }}'` and files live at `<kit>/autogen.kit.d/<pn>/` (NO category segment,
    NO `files/` subdir; contents sit directly under `<pn>/`, e.g. `autogen.kit.d/nginx/nginx.conf`,
    `autogen.kit.d/caddy/Caddyfile`, `autogen.kit.d/glusterfs/*.patch`). Precedents: caddy,
    glusterfs, thin-provisioning-tools. The older `autogen.kit.d/<category>/<pn>/files/` form
    (debian-sources, anise, glibc) still exists but is NOT what maintainers want in new
    autogen PRs — I first used it and geaaru corrected me: "the convention for files_dir is
    `autogen.kit.d/<pn>`, not with category name and/or `files` subdir." `mark-devkit` resolves
    `files_dir` relative to `autogen.kit.d/`, and `autogen-thin` copies that dir into the
    generated tree as `files/` (so `${FILESDIR}/…` still resolves). "The files are already in
    the kit repo" does not fly — the reviewer flags it as "patches are not pushed in this PR".
34. **Review hygiene bar** (geaaru, macaroni-os/kit-fixups#513): drop EVERY commented-out code
    line from a ported body (inert table entries, dead postinst blocks, old `mod_uri`/`mod_doc`
    variants — prose comments stay); tracker-reference comments (bugs.funtoo.org FL-####,
    bugs.gentoo.org #####) are dropped too (§3.18 extended); a stale blocker atom gets removed
    even though it was a verbatim tree-ebuild line — the "keep verbatim for scope discipline"
    instinct gets flagged (`!www-server/nginx:0` — a typo at a nonexistent category: "This seems
    a deprecated line that could be removed").

 35. **Version strings render from `.Values`, never `${PV}`** (maintainer ask,
     macaroni-os/kit-fixups#514, 2026-11-01): in a spec `body:` write `{{ .Values.pv }}` (and
     `{{ .Values.version }}`) — NOT the shell var `${PV}` — so the generated ebuild carries the
     literal (§3.8/§3.15 principle applied to version literals too). When a source flavour must
     reference the exact tag build, stamp `{{ .Values.pv }}-{{ substr 0 7 .Values.sha }}` (sprig
     `substr`, already used in-repo by the fish cluster) — e.g. `-X main.Version=3.1.1-9db7df7`.
     **BUT check the generator version:** mark-devkit ≥0.36.1 `builtin_github.go` sets
     `sha = tag.Commit.GetSHA()` — the 40-char TAG COMMIT sha — whenever the tag resolves
     (probed live on kube-ovn + ovn-kubernetes, kit-fixups#553/#557 round 2, 2026-11-10: renders
     the real commit on all six tracks, and the maintainer explicitly prefers
     `COMMIT={{ .Values.sha }}` over any unpack-time sha stash). The older trap stands only for
     the `ignore_tags: true` fallback, where sha falls back to the release `target_commitish`
     and can be a BRANCH name (mimir: literally `main`) — there, fall back to plain
     `{{ .Values.pv }}`. Related edge proven the same day: an ANNOTATED tag's recovered tarball
     unpack root is named by the tag-OBJECT sha while the distfile auto-name suffix
     (`<pn>-<pv>-<sha7>.tar.gz`) carries the commit sha — the `mv <user>-<repo>-*` wildcard in
     §3.36 absorbs either, but never stash `${d##*-}` from the unpack dir as "the commit".
 36. **Source flavour = generator tarball recovery + `post_src_unpack`, assets live under `-bin`**
     (settled convention, macaroni-os/kit-fixups#514 review, geaaru 2026-11-01, verified on
     tempo/loki/mimir): do NOT list the GitHub `archive/refs/tags/...` tarball as an explicit source
     asset. Put the whole `assets:` list on the `-bin` package entry (the source entry simply has no
     `assets:`), and let `builtin-github` recover the release's
     `api.github.com/repos/<user>/<repo>/tarball/<tag>` for the source entry.
     Its unpack root is `<user>-<repo>-<sha7>` (≠ `${P}`) so the source body needs
     `post_src_unpack() { mv {{ .Values.github_user }}-{{ .Values.github_repo }}-* "${S}"; }` (the
     same shape as `simple-github.tmpl` and the prometheus-exporters cluster) — that wildcard also
     makes `s:` unnecessary for the source flavour. PROOF the recovery is equivalent to the tag
     archive (the maintainer's condition "if it works the same way"): fetch both and
     `diff -r -q` — tempo: 15740 vs 15740 files, 0 deltas. `github_user` renders (`grafana`), no
     literal needed. Bin ebuilds must regenerate BYTE-IDENTICAL after moving assets into them (§3.31
     bar) — if not, the move changed semantics; revert. For a **prefixed-tag** repo (mimir tags are
     `mimir-3.2.4`, not `v...`), plain `query: releases` yields "No versions found" because the
     tag↔release cross-check skips releases whose tag isn't on the tags-endpoint page — set
     `github.ignore_tags: true` (see §3.4/§7s).

 37. **Go source flavours inherit `go-module` — even vendored, even when no bundle is needed**
      (maintainer ask, macaroni-os/mark-issues#848 reopen, geaaru 2026-11-02: "This probably could
      be fixed if the ebuild uses the `go-module` eclass where we try to override existing go.mod"
      and "The `go-module` eclass could be used also if the go bundle is not needed"). A
      hand-written `go build -mod=vendor` is green on the sandbox tree and dies in the merge phase
      on a release branch the moment upstream's `go` directive exceeds that tree's toolchain
      (observed: `tempo-3.1.1` → `go.mod requires go >= 1.27.3 (running go 1.27.2;
      GOTOOLCHAIN=local)` while my unstable sandbox had 1.27.3). The eclass's
      `_go-module_check_gotoolchain` (exported `src_prepare`) rewrites the `go` line in go.mod to
      the local `GOVERSION` on ANY mismatch — `EGO_OVERRIDE_GOMOD=0` disables it — ewarning
      "Package needs go X. Forcing local version Y." Sharp edges: EAPI **7 only**;
      `EXPORT_FUNCTIONS src_unpack src_prepare pkg_postinst`, and `go-module_src_unpack` falls to
      `default` when A has no `mark-go-bundle` distfile and `EGO_SUM`/`EGO_VENDOR` are empty — so
      the §3.36 `post_src_unpack` mv-shape keeps working untouched; the check reads go.mod at
      `${S}`, so multi-module builds (alloy, §7s) must mirror the override across EVERY local
      module wired via `replace ../..` — a collector-only override still died on `../syntax`;
      proven pattern is a `post_src_prepare()` loop `find "${S}" -name go.mod -not -path
      '*/vendor/*'` with the eclass's own sed, one ewarn per module; add the inherit on the
      SOURCE cluster entries only — package-level `vars.inherit` REPLACES the `defaults.vars` list (mark-devkit 0.35.4 probe: `[go-module]`
      alone generated `inherit go-module`), so repeat `user`/`systemd` per entry; `-bin` ebuilds
      must regenerate byte-identical. The rewrite goes BOTH directions on mismatch — upward
      (1.26.10→1.27.2) verified harmless on loki/mimir. Tree precedents: gitlab-runner, yq-go. A maintainer
      toolchain bump can hotfix the branch meanwhile (macaroni-os/macaroni-funtoo@4b940f8) — that
      does NOT replace the eclass ask.

 38. **Not every fix needs release-branch PRs — check what the branch actually carries**
      (macaroni-os/mark-issues#848 round 2): the kit-fixups release branches hold only
      merge-atom recipes (`<kit>/merge.kit.d/*.yml`, e.g. added by kit-fixups#518/#519/#520) whose
      `sources:` point at `<kit>@mark-unstable` — a SPEC content fix merges on unstable alone and
      the bot re-propagates. Before filing per-branch PRs, `git ls-tree -r origin/<branch>` the
      kit-fixups branch and edit only what exists (§3.20's per-branch PRs are for the ATOM SET).
      Caveat when the promoted kit already carries a same-PV ebuild (observed:
      core-server-kit@mark-xl `tempo-3.1.1.ebuild` pre-fix): whether the merge pipeline re-copies
      changed content under the same PV is maintainer territory — surface it with the evidence and
      offer the `-r1` bump, do not execute it unasked (§9).

 39. **Agent PRs carry the `ai_ass` (AI Assisted) label where the repo defines it** (operator
      direction, 2026-11-02; the label exists in macaroni-os/kit-fixups, not in every org repo):
      at PR-creation time check `gh api repos/<org>/<repo>/labels --jq '.[].name'` and, if
      `ai_ass` is defined, set it on the issue side of the PR:
      `gh api -X PUT repos/<org>/<repo>/issues/<pr>/labels -f 'labels[]=ai_ass'`. Do not create
      the label in repos that lack it, and do not label issues/commits — PRs only. Applies to
      every PR the agent opens that day, propagation rounds included; verify after setting
      (`pulls/<n>` → `.labels`).

 40. **Aligning promoted kit ebuilds with a merged spec = a kit-repo PR in the BOT's shape**
      (macaroni-os/core-server-kit#3448 per maintainer ask, mark-issues#848 round 2: "open a fix
      PR in mark-unstable to align the ebuilds … in order to propagate the fix with the bot" —
      the bot does NOT re-copy changed content under the same PV, cf. §3.38). Generate with
      autogen-thin from the merged kit-fixups spec, deploy into the kit working copy, and file
      the ebuild onto the kit repo's branch. Kit conventions observed: promoted manifests are
      **DIST-only** (0 `^EBUILD` lines kit-wide; this portage's `ebuild manifest` preserves that
      shape — a content-only fix yields an ebuild-only diff, matching the bot's own bump PRs);
      `ebuild manifest` must run in the real git-managed kit copy (fails as `::x-root` in
      scratch dirs). Proof bar: generated ebuilds `cmp`-identical to `/var/db/pkg/<cat>/<PF>/<PF>.ebuild`
      (§5.2) or to what you built; `-bin` trees regenerate byte-identical.

 41. **Go bundles v2 (mark-devkit ≥0.36.0): `gosum_paths` + `markdevkit_min_version`**
      (released from macaroni-os/mark-devkit#9, alloy swap-in kit-fixups#526):
      `extensions_defs.golang.opts.gosum_paths: go.sum,collector/go.sum` — paths relative to the
      UNPACKED repo root (pin the entry dir with `unpack_srcdir_prefix`), all sums share one
      bundle dir deduped by proxy-filename → the bundle is the EXACT UNION (alloy: 49
      collector-only module@versions land; verify with a fresh set-diff, bundle==union±0).
      `markdevkit_min_version` is a CLUSTER-level key (`AutogenDefinition`) and a HARD gate:
      `ProcessDefinition` refuses the whole definition (autogen-thin too) when the running tool
      is older — pin it only on clusters that require the newer option, or you break their
      generation on older releases. Verifying a FIRST-EVER bundle locally: stage the
      extension-generated `mark-go-bundle` tarball + generated Manifest into DISTDIR and
      `emerge -1` (assume-digests; `GOPROXY=file://` then works WITH network-sandbox on — that
      IS the proof the interim `restrict: strip network-sandbox` crutch is gone); the public
      mirror upload stays the bot's post-merge job (#848 answer). `portageq envv FEATURES` can
      print empty — `emerge --info` is the FEATURES ground truth.

 42. **A "please autogen `<pn>`" issue covers the helper stack the package needs to RUN**
      (mark-issues#861, podman 6.1.4). Podman 6 fails at startup without `conmon`, an OCI runtime
      and `netavark` (the ONLY supported network backend — `Error: could not find "netavark" …`),
      and rootless additionally needs `crun`, `pasta` and `fuse-overlayfs`; **`slirp4netns` support
      was REMOVED in podman 6** (`slirp4netns support has been removed, use --network=pasta
      instead`). When none of those atoms exists in any kit, ship the minimal working set in the
      same PR (nine here: podman, conmon, catatonit, netavark, aardvark-dns, crun, fuse-overlayfs,
      passt, containers-shortnames) — the issue's `Closes:` stays on the completing PR (§3.23).
      House category for container tooling is **`app-emulation`** (these kits have no
      `app-containers` category; operator-confirmed). `rootlesskit` is NOT a podman dependency
      (that is Docker's rootless helper) — don't add it just because a rootless list mentions it.

 43. **Vendored Go sources must force `GOFLAGS=-mod=vendor` — `go-module` sets `-mod=readonly`**
      (podman 6.1.4, kit-fixups#524). The eclass exports `GOFLAGS="-v -x -mod=readonly"`, which
      defeats Go's automatic vendor-directory selection even when a complete `vendor/` is present;
      the build then reaches `proxy.golang.org` and dies offline (`dial tcp … network is
      unreachable`). Re-export the mode in the spec `body:`'s `src_compile`:
      `export GOFLAGS="-v -x -mod=vendor"` (same shape as docker/runc). Proved by a full
      `emerge -1`, `qlist -Iv`, and a rootful smoke (§5).

 44. **`autogen-thin` does not run extensions — generate Rust/Go bundles with full `autogen`**
      (`--skip-merge --keep-workdir --sync=false`, podman stack #861). `autogen-thin` emits the
      ebuild + Manifest but leaves `mirror://macaroni/…-mark-rust-bundle-….tar.xz` unfetched, so a
      Rust source flavour cannot be built. Full `autogen --skip-merge --keep-workdir --sync=false
      --skip-reposcan-generation --to <dir>` runs the extension locally and drops the bundle in
      `<dir>/downloads`; copy `<dir>/downloads/*` into DISTDIR before `emerge -1`, then prove
      `cmp` identity between the installed `/var/db/pkg/…` ebuild and the generated one (§5.2).
      **Never run a review-side `autogen` with the default `--sync=true`**: it uploads generated
      artifacts to the shared S3/minio mirror, which belongs to the bot's post-merge job.
      *(Correction on mark-devkit 0.36.1: per the §2 note, `autogen-thin --to` now runs golang
      AND rust extensions — full `autogen` remains the proven route for shipped artifacts.)*

 45. **cgit-only upstreams are trackable with `builtin-dirlisting` on the cgit tags page**
      (passt, #861). passt has no GitHub/Forgejo/GitLab repo, but
      `https://passt.top/passt/refs/tags/` lists per-tag `snapshot/passt-<tag>.tar.xz` links: use
      that as `dir.url` with `matcher: 'passt-.*\.tar\.xz$'`, strip the tag commit with
      `regex: '\.[0-9a-f]+$'`, then map `_`→`.` for the version (`2026_10_02.cba3570` →
      `2026.10.02`). The unpack root is the raw snapshot directory, not `${P}` — add
      `post_src_unpack() { mv ${WORKDIR}/passt-* ${S}; }` and pass `VERSION=<version>` to make
      (otherwise `--version` reports `unknown version`). Only fall back to a pinned
      `builtin-noop` when the upstream has no machine-readable listing at all (justify in a
      comment, §3.2).

 46. **Daemon-style tools on systemd: ship a `.socket` + socket activation, not
      `Environment=SOCKET=`** (geaaru review, kit-fixups#524). Upstream podman ships
      `podman.socket` (`ListenStream=%t/podman/podman.sock`, `SocketMode=0660`) and a
      `podman.service` with `Requires=`/`After=podman.socket`; the daemon takes the socket via
      systemd socket activation (`activation.Listeners()`), so the service must NOT bind a URI
      itself. Keep `EnvironmentFile=-/etc/conf.d/<pn>` on the Macaroni-owned unit for
      `LOG_LEVEL`-style knobs (the SAME `/etc/conf.d` file as the OpenRC initd), `Type=exec`,
      `Delegate=true`, `KillMode=process`. Verify what is verifiable: both units land in `files/`,
      the generated ebuild installs the pair under `use systemd`, and the OpenRC path still
      builds/runs; when `sys-apps/systemd` is absent from the kit, state that the systemd side was
      not exercised instead of implying a green test.

 47. **New-package release-branch atom PRs: check the target tree BEFORE filing** (§3.20, podman
      stack #861). On `mark-xl`/`mark-iii`/`mark-31`, `git ls-tree -r origin/<branch>` first — those
      branches carry merge recipes only (`<kit>/merge.kit.d/tools.yml`, often no `*_autogen.yml`),
      so edit the file that actually exists. Check the language floor against the BRANCH, not
      unstable (§7t): podman's `go.mod` floor is `1.26.0` and `lang-kit@<branch>` shipped up to
      1.27.3 / 1.26.9 / 1.26.10 — all pass; also confirm every build/runtime atom exists there
      (`contents/<cat>/<pn>?ref=<branch>` for protobuf, fuse, nftables, gpgme, sqlite, …). Record
      those checks in the PR body; propagation PRs take `Bugs:` (the completing PR owns `Closes:`).

 48. **Collapsing a shared-upstream cluster to ONE parametrized `defaults.vars.body` with sprig
       guards** (geaaru review on macaroni-os/kit-fixups#539: "create a new definition key …
       keep less lines is better"): when several `packages:` of one cluster share a big body and
       differ only by (a) an optional sub-block and (b) a couple of literals, hoist the body ONCE
       under `defaults.vars:` and toggle the deltas with sprig branches —
       `{{- if .Values.<flag> }} … {{- end }}` — driven by per-package `vars`. Binary/cmd names
       come from `.Values.pn` (`WHAT=cmd/{{ .Values.pn }}`). Byte-identity survives the
       space-pair→tab normaliser because `{{- if }}`/`{{- end }}` left-trim their own newline, so
       a false branch leaves NO blank line — verified: kubernetes `kubectl/kubelet/kubeadm/
       kube-proxy` (completions on/off, service+static-account on/off, uid/entity_pr as vars)
       regenerated 12/12 ebuilds + 4/4 Manifests `cmp`-clean across all tracks, cluster 461→211
       lines. Custom keys (`completions`, `service`, `uid`, `entity_pr`) are ignored by the
       template but read by the body; overriding a defaults scalar per-package (`completions:
       true`→`false`) works — a zero-delta run IS the proof the merge chain is live (an inert
       guard would leak the wrong block). `inherit`/`iuse` still must be REPEATED per entry
       (package `vars` replace those list keys, §3.31/§3.37). Keep the maintainer's own merged
       selector bounds untouched; a spec-only refactor that changes no resolution needs NO
       propagation-branch rework (§3.38) — the release branches carry the merge ATOM only.

 49. **Shared-cluster body edits are proven by a per-track identity table, and whole sprig branches
     delete cleanly** (kit-fixups#547 round 2): a `{{- if .Values.<flag> }}` guard in a cluster
     `defaults.vars.body` means non-flag entries are provably untouched (kubectl/kubeadm regenerated
     ZERO-delta) — state it per package per track. Deleting a complete `{{- if }}/{{- else }}/{{- end }}`
     block plus its rendered sentence leaves no whitespace residue IF the flanking literal lines keep
     identical indentation: the `{{-` trim markers own the surrounding newlines, so the remaining
     literals rejoin exactly (verified: all service entries COMMENT-ONLY(-1), the removed sentence
     line, and files//Manifests identical — 28/28 cells). Same-round re-selection must be asserted
     (both runs captured the same 3 versions) so the table cannot be polluted by upstream releases.
     "Create a specific key definition" (geaaru, sysvinit #608) = give the package its OWN
     autogen cluster key instead of living in a shared catch-all cluster, with one `packages:`
     entry per release line, each carrying its own `selector:` (+ per-line `patches:`/
     `revision:`) — same machinery as the §4 single-PN dual-track row; prove generation AND a
     real build for EVERY line, especially an init-adjacent new-major track the cap previously
     held back.
     Maintainer stance confirmed the same session: entity-provenance comments in generated ebuilds
     are NOT wanted ("drop all lines with `entity_pr` and the comment", §3.18/§3.34 applied to
     macaroni-commons PR refs) — the `enewgroup`/`enewuser` lines carry the ids; nothing else.
     Whitespace follow-up (geaaru on kit-fixups#582, 2026-11-15): template-control lines in spec
     bodies take the LEFT-TRIM form `{{-` (not `{{`) — with plain `{{` the directive's own newline
     survives rendering and the generated ebuild carries stray empty lines; "Just a note for the
     playbook for the next autogen".

 52. **Epic issue → per-package child issues is the accepted division-of-labor shape**
     (mark-issues#880, geaaru 2026-11-14: "This the list that we can use to create children issue
     of the present issue … Anyone want helps, keep a package, open an issue and do the job. Agent,
     maybe you can help on open issues"): one child issue per package directory, title
     `[<kit>] autogen \`<cat>/<pn>\``, labels copied from the epic (MARK, <kit>, new_autogen,
     mark-unstable, mark-xl, mark-iii, mark-31, mark-i, mark-v); the body carries the captured tree
     versions, the conversion precedent PR, and a `**Note:**` only where a maintainer decision is
     pending (slots/EOL/no-upstream/dormant). Link them as GitHub sub-issues:
     `gh api repos/macaroni-os/mark-issues/issues/<epic>/sub_issues -X POST -F sub_issue_id=<id>` —
     `sub_issue_id` is the issue's numeric `id` from the create response, NOT the `number`, and must
     be sent `-F` typed-integer (`-f` string → 422 "not of type integer"). Labels array likewise:
     `gh api ... -F labels='[...]'` 422s as "not an array" — build the JSON with jq and pipe to
     `--input -`. Claim a package with a comment on its child issue, work it, and put
     `Closes: macaroni-os/mark-issues#<child>` in the PR BODY (the epic stays open; each child dies
     with its own merge). Spec conversions need NO release-branch follow-up PRs: the kit-fixups
     release branches carry only merge-atom recipes pointing at `kit@mark-unstable` (verified on
     mark-xl/iii/31 after util-linux macaroni-os/kit-fixups#573 merged — restates §3.38); the
     per-branch `mark-*` labels on the children describe propagation, not extra PRs.


 50. **Rust MSRV floor: probe the gate before conceding, propagate per the branch's rust ceiling**
      (net-fs/rustfs #865, kit-fixups#564/#565/#566, 2026-11-11): upstream declared
      `rust-version = "1.99.0"` + `edition = "2024"` while lang-kit topped out at rust-bin 1.98.1.
      The cargo error `rustc 1.98.1 is not supported by the following packages` is only the
      *declaration gate* — edition 2024 compiles on ≥1.85 — so the house answer is
      `cargo_src_configure --ignore-rust-version` (cargo.eclass has NO `CARGO_IGNORE_RUST_VERSION`
      var; positional args land in `ECARGO_ARGS` and reach every cargo phase) with a YAML comment
      citing the upstream MSRV and the verified tree version, disclosed in the PR body. Distinguish
      gate-fail from real-compile-fail, and probe EACH release branch's actual rust-bin ceiling
      (`git ls-tree` the lang-kit branch): mark-xl/mark-31 at 1.96.1 → `cargo check
      --ignore-rust-version` green → propagated; mark-iii at 1.94.0 → real
      `error[E0658]: use of unstable library feature 'cfg_select'` in sysinfo → NOT propagated,
      disclosed in the issue comment pointing at the open lang-kit bump PR as the enabler. Never
      bump the lang-kit compiler inside a package PR (§9).
 51. **Two upstream-Rust shapes break `mark-rust-bundle` generation — both fixed spec-side**
      (rustfs #865): (a) git crates whose Cargo.lock rev lives on a NON-DEFAULT branch fail the
      extension's single-branch clone (`error on checkout repo <x> for hash <sha>: object not
      found`, prove the branch with a full clone + `git branch --contains`) — set
      `extensions_defs.rust.opts.crates_git_single_branch: 'false'`; (b) a `[patch.crates-io]`
      local crate SHARING a registry crate's name at a different version confuses the extension's
      name-keyed local-crate detection and the registry version pinned in Cargo.lock is SILENTLY
      dropped from the bundle (offline build: `failed to select a version for the requirement
      'zstd = "^0.14.3"' … candidate versions found which didn't match: 0.13.4`) — prove it with a
      Cargo.lock-registry-entries ↔ bundle-contents set-diff (rustfs: exactly 1 missing, 0 extra),
      then re-add the missing `.crate` as an explicit `assets:` entry (main source asset
      re-listed first per §3.32, version pinned in `vars:`) and feed it into the vendored registry
      with the cargo eclass' own `_cargo_process_crate` in `src_unpack`. Upstream improvement ask:
      key the local-crate detection by name+version.

 52. **Core-kit autogen review round: render the names, declare docs, drop Gentoo cross-compile**
      (geaaru review on macaroni-os/kit-fixups#585/#589/#600/#620, 2026-11-15):
      - **No inherited cross-compilation logic.** The toolchain comes from the Macaroni From
        Scratch project, so ported ebuilds must NOT carry the old Gentoo cross answers
        (`if tc-is-cross-compiler ; then … export bash_cv_func_sigsetjmp/… ; fi`) and the
        `! tc-is-cross-compiler` guards around a package's own sub-builds are dropped with them —
        the maintainer was explicit: "We don't want support the old Gentoo cross compilation logic."
      - **Names in spec paths come from the renderer**, not shell vars: write
        `s: '${WORKDIR}/{{ .Values.pn }}'` instead of `s: '${WORKDIR}/${PN}'` — the generated
        ebuild gets the literal and readers do not have to resolve `${PN}` mentally
        ("it's always better to use the mark-devkit render and generate more clear file for users").
      - **Documentation lists use the dedicated keys**: the `docs:` spec key renders `DOCS=( … )`
        instead of hand-writing the array inside `src_install`; env-style lists such as
        `HTML_DOCS="doc/*.html"` belong in `extra_envs:` (both render before the phases).
      - **Proprietary/prebuilt drops** do not need `mirror` in RESTRICT — `restrict: bindist`
        is enough when the distfile is already pinned in the spec ("you can drop `mirror` from
        `restrict`").
      - **Shared files mean a rebase queue**: every open PR that edits
        `core-kit/autogen.kit.d/core.yml`, `merge.kit.d/core.yml` or `core_autogen.yml` goes dirty
        the moment ANY sibling PR merges (the maintainer merges fast and the remaining branches
        conflict on the same anchors). Rebase + re-verify immediately after each merge — "You need
        to rebase it again and monitoring all PR using the same files" — and treat merge order as
        part of the PR's completion, not an afterthought.

 53. **Core packages split their Python bindings into `<pn>-python`** (geaaru on file #603,
     2026-11-16: "Considering that is a core package it makes sense to split the package having
     a `file-python` package"): the main spec loses the `python` USE + the bindings install; a
     NEW cluster entry + NEW merge atom `<cat>/<pn>-python` carries the module (DEPEND on the
     parent). In-tree precedent: `sys-apps/util-linux-python` — copy its shape (same tarball,
     single python target, own files payload if any).

 54. **CPU-flag USE surfaces are declared as a `vars` list and rendered into `iuse` — never
     expanded at bash level** (geaaru on macaroni-os/kit-fixups#618, 2026-11-18: "the logic is
     managed at render level not at bash level expanding variables"). For packages exposing
     `cpu_flags_*` USEs, keep the supported flags in one variable and let the template loop emit
     them:

     ```yaml
     vars:
       cpu_flags:
         - x86_avx2
         - x86_sse2
         - x86_ssse3
         - x86_sse4a
         - x86_pclmul
         - arm_crc32
         - arm_neon
         - ppc_vsx2
       iuse: |
         compat
         {{- range $f := .Values.cpu_flags }}
         cpu_flags_{{ $f }}
         {{- end }}
     ```

     The generated ebuild carries `IUSE="compat` plus one `cpu_flags_*` per line — a valid quoted
     whitespace list — and the `{{- range }}` trim markers keep the render clean. The rejected
     alternative was an `extra_envs: - 'CPU_USE=( … )'` array expanded with
     `${CPU_USE[@]/#/cpu_flags_}` in `iuse`: it works in bash, but hides the flag surface from the
     spec and was explicitly corrected by the maintainer (media-kit precedent:
     `media-kit/autogen.kit.d/base.yml`, the ffmpeg cluster). Same rule for `test` USE: core/tool
     packages drop it and pass the build-system toggle explicitly (e.g. `-DBUILD_TESTING=OFF`).

 55. **The spec schema is `pkg/specs/definition_autogen.go`, not the examples** (geaaru on
     macaroni-os/kit-fixups#627, 2026-11-19: "add a note on your playbook about check syntax based
     on this definition"). When the *level* of a key is in doubt, read the structs —
     `AutogenDefinition` (`template`, `generator`, `generator_opts`, `defaults`, `packages`,
     `extensions_defs`, `markdevkit_min_version`) vs `AutogenAtom` (`assets`, `selector`,
     `revision`, `files_dir`, `category`, `transform`, `exclude`, `ignore_artefacts`, `vars`, …)
     vs `AutogenAsset` (`name`, `url`, `matcher`, `use`, …). Unknown or misplaced keys do not
     fail the run: they are silently dropped and the generator still exits rc 0 with "All done".
     - **`assets` is an atom-level key** — a peer of `vars:` on a definition's `defaults:` or on a
       package entry, NOT a field inside `vars:`. `assets` under `vars:` generated an ebuild with
       **no `SRC_URI` at all** while the run reported success (libpcre, kit-fixups#627 probe).
     - **Manifest digests come from `assets`**: `pkg/autogen/package.go` downloads every asset and
       sets `.Values.src_uri` from the FIRST artefact. A hand-written `src_uri:` var renders
       verbatim, but its Manifest digest is not elaborated correctly, and `mirror://` is resolved
       only for legacy/old ebuilds (and macaroni bundles) — pin the real download address in
       `assets.url` instead, including for pinned `builtin-noop` conversions. Captured (libpcre
       8.44): `SRC_URI="https://downloads.sourceforge.net/pcre/pcre-8.44.tar.bz2 -> pcre-8.44.tar.bz2"`,
       then `emerge -1` rc 0 with the Manifest generated from the downloaded asset.

 56. **`global_settings` is the reserved file-level key for shared options — not a package
     definition** (same #627 review: "check the possibility in the autogen file if we can migrate
     options common to all packages to `global_settings` key and reduce lines"). It is not a
     definition: `ProcessDefinitions` skips it in the loop and `MergeDefinition`s it into every
     definition of the same file, so its `defaults:`, `template:`, `generator_opts:`,
     `extensions_defs:` and `markdevkit_min_version:` are inherited (packages are NOT merged).
     Use it only at file top level to hoist options common to all clusters and cut duplicated
     lines; never name a real cluster `global_settings`. Auditing an existing file for what can
     move there is a follow-up opportunity, not a prerequisite for a package conversion (the
     libpcre conversion was merged as-is).

  57. **`autogen.kit.d/` layout: spec FILES are flat; subdirs are files_dir payloads only**
      (ai-kit `app-ai/maya` #932, kit-fixups#628): a new atom's cluster goes into the kit's flat
      spec file (appended to `ai-kit/autogen.kit.d/base.yml` or a new flat `<name>.yml`), NOT a
      `<name>/base.yml` — the depth-2 directories there (e.g. `ollama/`) hold only the `files_dir`
      payloads (confd/initd/service/envd) that `mark-devkit --specfile` consumes as flat files.
      Same round, three reusable new-package shapes: (a) upstream CMake projects that
      `FetchContent`-clone a pinned dependency at configure time become hermetic by adding that
      exact commit as a SECOND spec `assets:` entry and pointing upstream's own local-directory
      switch at it (`STRATA_GGML_DIR` for project-maya; nvcc's include path inside the sandbox
      build is the proof); (b) a `undefined reference` link fail that appears ONLY through portage
      (raw build green) is usually the eclass build type lacking `NDEBUG` — fix spec-side with
      `extra_envs: ['CMAKE_BUILD_TYPE=Release']` (renders pre-`inherit`) and prove it with `nm`
      symbol counts on the raw vs failed object; (c) when upstream gates the main target behind
      GPU backends (`if(STRATA_ENABLE_CUDA OR STRATA_ENABLE_HIP)` → configure succeeds but builds
      NO target), the honest spec requires the GPU USE flag instead of pretending a CPU shape —
      say so in the PR and reproduce the no-backend build to show the missing target.

  58. **AppImage `-bin` atoms: two unpack traps, both spec-side** (ai-kit `app-ai/strata-bin` #949,
      kit-fixups#631): (a) portage `unpack()` does NOT dispatch `.AppImage` (empty WORKDIR, fails
      later with `chmod: cannot access '<app>.AppImage'`) — stage it explicitly with
      `cp "${DISTDIR}/${A}"` (the jq-bin/plantuml intact-archive idiom) then run `--appimage-extract`;
      (b) a Tauri/linuxdeploy AppImage **self-extracts headless with no system unsquashfs** (proved
      in a clean dir with unsquashfs removed → no BDEPEND needed) but **plain `unsquashfs` on the raw
      asset FAILS** (`Can't find a SQUASHFS superblock`) — the naive fallback is not an option, so
      `--appimage-extract` is the ONLY extraction path. Install to `/usr/lib/<pn>/` with a
      `/usr/bin` symlink (pi-bin idiom); the unpacked `usr/bin` binary is the `Exec=` name so pick
      that as the symlink target (upstream's own name → `strata-desktop`), avoiding a collision with
      a same-upstream SOURCE atom (here `/usr/bin/strata*` from #630 vs `/usr/bin/strata-desktop` —
      disjoint by file-set, so NO mutual-block, unlike opencode/opencode-bin which share the path).
      Two disclosures that held the verification bar honest: bundled webkit means the system RDEPEND
      is only the ~8-lib host base (ldd 0 unresolved), and a headless GUI launch aborts at `Failed to
      initialize GTK` — capture the startup trace (host served, engine-absence handled) as the smoke,
      never claim rendering. The RDEPEND closure can pull a mesa that hits the python-3.10 floor
      (see the loop's tree note) — pin the tree's older mesa in the sandbox to build, remove the pin
      before hand-off, and disclose it. FOLLOW-UP (same PR, geaaru: "use the linux tarballs instead
      of the AppImage"): the shipped shape became the `strata-control` TARBALL — and the rework
      produced the pattern for **-bin atoms whose component assets version independently of the app
      tags** (control 0.1.62 @ v0.2.16, 0.1.61 @ v0.2.15, dual 0.1.60/0.1.60.1 @ v0.2.14 — so
      `{{ .Values.version }}`-templated asset names are IMPOSSIBLE): use a VERSION-FREE `matcher:`
      (`strata-control-[0-9.]+-linux-x86_64\.tar\.gz`) resolved against the *selected release
      itself*, and rename the distfile to the app version (`name:`) so it stays unique per PV;
      prove the ride with two selector renders (`<0.2.16` → the 0.1.61 asset). Per-release asset
      fluctuation is a packaging argument too: v0.2.15 ships NO x86_64 engine assets at all, so
      `use:`-gated engine entries would hard-break autogen on such tags — verify the full per-tag
      asset matrix (Release-API sweep) before gating anything.

  59. **mark-devkit's python machinery is `builtin-pypi`-only — don't reach for distutils-r1**
      (maya #932 review round, kit-fixups#628, geaaru "Why doesn't use the distutils-r1 eclass?"):
      `python_compat`/`pydeps`/`pypi_name` are fed ONLY by the PyPI generator (`builtin_pypi.go`) and
      consumed ONLY by `pypi-generic.tmpl` (the sole tree template with `inherit distutils-r1`) —
      under `builtin-github` they render NOTHING (proved by a throwaway spec render, restored after:
      identical ebuild). For app-embedded Python that runs on the SYSTEM interpreter: declare
      slot-less `dev-python/*` RDEPENDs WITHOUT `${PYTHON_USEDEP}` (target-pinning would misclaim on
      release branches where `make.defaults` resolves only `python3_9`), and answer "missing deps"
      with a determination, not a guess: static AST closure over the INSTALLED tree (module-level
      unguarded imports only) PLUS a dynamic headless run of the app's serve entry (mock/no-model
      mode where it exists) on the version the app needs, with the optional packages absent —
      guarded imports surfacing their own `pip install …` errors are the proof they stay
      undeclared. Before offering a full `PY_PACKAGES`-parity list, sweep the tree for each atom:
      maya's parity would need `jsonschema>=4.18,<5` (tree serves 4.5.1), plus `tqdm`/`textual`
      which DON'T EXIST — offer parity only with the honest diff disclosed.

  60. **USE-gated `assets:` resolve at GENERATION time and hard-fail on absence** (strata-bin #949
      engines round, kit-fixups#631): `builtin_github.GetAssets` iterates every asset with no USE
      awareness — a matcher with zero hits ABORTS atom generation ("no asset found for matcher …")
      **while `autogen-thin` still exits rc 0** (log-only failure; the artifact-exists assert is the
      only gate — §7e family), and at tip the generator downloads/digests ALL gated assets (3.5 GB
      for the 10-asset Strata matrix) even for a default-USE build. `AutogenAsset` has no optional
      field; `ignore_artefacts` is atom-wide and drops every Manifest digest. Consequences for
      release-asset products whose per-tag matrix fluctuates (Strata: v0.2.15 zero linux-x86_64
      engines, v0.2.14 no sycl, aarch64 engines only from v0.2.7): USE-gated entries generate fine
      on full-matrix tips but CANNOT generate on otherwise-valid older tags — sweep the full matrix
      via the Release API BEFORE designing the USE surface (and note `hip-rdna2/3` were windows-only
      ids: name flags off the real linux suffixes). USE flags cannot contain hyphens — map
      `hip-rx6000` → `hip_rx6000`, arch-prefixed `arm64_cuda13` + `REQUIRED_USE` arch ties is the
      sanctioned stand-in for multi-USE asset conditions (geaaru's own prescription); aarch64
      verification without an arm64 sandbox stays gen-level, disclosed. Feature ask drafted (not
      filed): `assets[].optional: true` → zero-hit = skip with WARN, no artefact/digest/SRC_URI.

  61. **macaroni python venues drift; reproduce the `python_upgrade` hook, then check
      installed-vs-active libdir before trusting any python smoke** (maya #932, geaaru 2026-11-20,
      after a venue run passed only because the image's python and its dev-python files happened to
      agree): the `macaroni/mark-experimental` image ships `dev-lang/python-3.11.16` +
      `eselect python3.11` + 3.9 already unmerged, and the user's own container showed the drift:
      `dev-python/pillow-12.2.0-r1` installed against `python3.13` REQUIRES with a `3.13`-libdir
      file set (2602 files) while `python3` ran **3.11** — `import PIL` failed. The hook's own
      migration tools CANNOT fix that (`python-updater`/`@python-module-updates --use-old-libdir`
      only consider already-installed atoms; every emerge line is `--noreplace` = "do not
      re-emerge installed") — a same-version `emerge` is a no-op on the FILES: portage never
      rewrites what is already installed, so a re-emerge "copies" the old files in the sense that
      nothing is rebuilt; `--force-reinstall`/`--nodeps` don't target the drift either (the
      interpreter switch does not mark same-version deps out-of-date without the `changed-deps`
      feature). Surgical fix for a drifted package: `emerge --unmerge <cpv> && emerge -1av
      <cpv>` (re-merges clean against the active interpreter — pillow → 2595 files, `cmp`-equal
      to a reference install; maya itself was never drifted so a re-emerge was pointless — its
      serve smoke was already valid). Reproducing the hook on a fresh container (target 3.11);
      the LIVE hook (`jobs/hooks/extras/python_upgrade.yml` @ phoenix) now carries
      `python_version2upgrade: 3.13` and `--ignore-srmask '*/*::python-kit'` (NOT
      `--ignore-package-mask`): sed `PYTHON_(SINGLE_)TARGETS` in
      `core-kit/profiles/funtoo/kits/python-kit/mark/make.defaults` → unmask `=dev-lang/python-3.11*`
      + mask `>=dev-lang/python-3.9` (write INSIDE the `package.unmask`/`package.mask`
      DIRECTORIES — the image has them as dirs, so the hook's `echo > file` form silently fails) →
      `--noreplace python python-exec` (no `-1`: this is what lands python in `world`) →
      `ego update world` (rewrites world from installed — why the user's world says 3.11) → locale
      entries + `locale-gen` → `--noreplace gobject-introspection glib` → `--noreplace @world` →
      `eselect python set python3.11` → `python-updater -a --ignore-safepause --noreplace` →
      `emerge -1 --noreplace @python-module-updates --use-old-libdir` → final `--noreplace python`
      → old-python removal (`emerge --unmerge dev-lang/python:3.9` + `rm -rf /usr/lib/python3.9` —
      SKIP the hook's bare `emerge --unmerge python`: on an image-built state the only matching
      package IS the active interpreter) → `revdep-rebuild -q`. Drop `--ask` everywhere (detached).
      Venue-boot gotchas: `dev-util/mark-sdk` does not exist and its presence KILLS the whole
      emerge line (boot `dev-util/mark-devkit` alone); `autogen-thin` is not a binary — run
      `mark-devkit autogen-thin`; autogen hard-fails ("Missing git token! Push interrupted!")
      without `GITHUB_TOKEN` even in thin mode; `--specfile autogen.kit.d/base.yml --kitfile
      merge.kit.d/base.yml` are relative to the KIT dir (`cd /kit-fixups/ai-kit`); the version
      `selector:` goes INSIDE the `- <pn>:` package entry — at cluster level it is silently
      ignored. A near-daily upstream can bump mid-session (project-maya v1.1.67→v1.1.68 during
      this very run): autogen correctly rode it, so when the verification target is the
      PR-emerged artifact, pin a throwaway selector (`['<1.1.68']` → byte-identical 1.1.67 render,
      sha256 `b8c625b5…`) and report the new tag to the orchestrator. And on venue boot check
      whether `/etc/portage/package.{use,unmask,mask}` and `package.license` are FILES or
      DIRECTORIES (the experimental image ships `package.use`/`package.license` as dirs): a
      `echo "cat/pkg flag" >> /etc/portage/package.use` dies with "Is a directory", and behind
      `|| true` it silently builds WITHOUT the intended USE — maya exp2 lost its `cuda` this way
      (870 files, no engines vs the intended 876/6); write inside the dir
      (`package.use/<name>`) and assert the `qlist` file count against the reference venue.

  62. **Two portage phase traps when a generated `src_install`/`src_prepare` does more than the
      template default** (maya #932 layout round, kit-fixups#628, both caught only by a real
      merge — `bash -n` passes both): (a) `newins` **respects the active `insinto`** — a
      `src_install` that first does `insinto /usr/share/<pn>` and later `newins launcher` +
      `fperms 0755 /usr/bin/launcher` stages the file under /usr/share/<pn> and dies in fperms
      ("No such file or directory"); ship the executable with **`newbin`** (the house idiom,
      EXEOPTS 0755, sets its own into) or an explicit `into`/`exeinto`. (b) This tree's portage on
      **EAPI 7 does not dispatch the cmake eclass' exported `src_prepare` through `default`** — a
      custom `src_prepare() { default; ... }` then dies in configure with `FATAL:
      cmake_src_prepare has not been run`; call `cmake_src_prepare` explicitly *first* (pattern:
      dev-util/opencl-icd-loader ebuilds), then sed the sources (seds after it are fine — the
      eclass' module-strip only touches *.cmake). Related design rule learned the hard way mid-land:
      before sed'ing an upstream "default path" constant, grep the function that uses it —
      maya's `serve/models.py` fallback feeds `models_dirs()`, a storage listing that FILTERS
      home-based folders out by design; sed'ing it silently disabled the listing while the real
      download target (the `maya.py` chain) needed exactly one file's six sites.

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
| autogen `PYTHON_COMPAT` picks a python the tree refuses to merge | house convention is `PYTHON_COMPAT=( python3+ )` (tree-wide: 962 ebuilds) — core-kit's profile mask `funtoo-kit-python` hides `>=dev-lang/python-3.10`, so pinning `python3_11` by the kit's HEADLINE ebuilds is wrong; check the mask, then resolve (`python_single_target_python3_9` is the live target, vim #868 2026-11-12) |
| `.jar` (or other zip-magic) distfile explodes into WORKDIR at unpack | portage auto-unzips by file magic — `src_unpack() { cp "${DISTDIR}/${A}" "${WORKDIR}"/ \|\| die; }` and install the intact archive (jq-bin/plantuml pattern) |
| custom-make package dies with a bizarre guard error only under portage (e.g. Firebird examples `ifndef ROOT` → extauth fatal) | Portage exports `ROOT=/`; upstream makefiles using bare `ROOT` collide — `unset ROOT` in `src_compile` (or `emake ROOT=`) and comment-cite the issue |
| upstream ships staged libs as absolute libtool symlinks into the build tree (libtommath/libtomcrypt pattern) | `cp -L` the real objects at install time — clean/merge phases dereference dangling symlinks later |
| consumer fails `#include <pkg/sub/...>` after install although `doheader` ran | the API headers live in a SUBDIR (`include/firebird/`) that `doheader include/*` skips — recursive `doins -r include/<subdir>` into `/usr/include` |
| old ebuild's SRC_URI host is dead (01.org → intel.com redirect) but the package still "exists" in-tree | check fetchability BEFORE treating the ebuild as functional; upstream may have consolidated (dleyna split repos → gitlab.gnome.org/World/dLeyna monorepo) — follow Gentoo's consolidated replacement package, add `!old/packages` blockers, and file the kit purge PR alongside |
| `ebuild manifest` dies on a USE-gated distfile whose USE is OFF | the manifest phase fetches EVERY conditional SRC_URI regardless of USE — one dead optional-module host blocks the whole conversion (nginx `sticky`'s bitbucket tarball 404s today, aborting even the 1.26.2 manifest). Fetch-probe every module tarball as part of the reproduce step; dead host → repoint to a mirror of the SAME pinned commit (align `mod_p`/`mod_wd` to that archive's unpack root) or drop the module with justification (kit-fixups#513) |
| "upstream has no GitHub releases" — memory or a stale repo page | verify live before falling back to `builtin-dirlisting`: `gh api repos/<u>/<r>/releases`. nginx had quietly published the official tarballs as release assets (nginx.org tarball + `.asc`, current through the security lines) and the whole #849 spec flipped from dirlisting to `builtin-github` on one API call. Branch/tag listings are NOT evidence about releases |
| `go: go.mod requires go >= X (running go Y; GOTOOLCHAIN=local)` — builds on unstable, fails on a release branch | inherit `go-module` (§3.37): its `src_prepare` rewrites the go.mod directive to the local toolchain; do NOT version-pin or downgrade the source to appease a tree |
| Go package compiles on OLDER go but dies on the NEWEST with `undefined:` errors inside vendored deps (e.g. kube-openapi `go-json-experiment` shim on go ≥1.27, kit-fixups#557 round 2) | a toolchain **ceiling** BDEPEND bound (`>=dev-lang/go-A <dev-lang/go-B`) — from the upstream's pinned `toolchain` directive / CI — is the house answer when the committed `vendor/` can't be fixed from the ebuild; `GOTOOLCHAIN=auto` is NOT an escape (the `toolchain` line only switches when local < the `go` directive — a newer local never downswitches; PROBE both modes before conceding a maintainer's "CI builds it fine"), verify every release branch carries a go inside the window before propagating, and disclose it in the PR body |
| `vendor/` is complete but the build still hits `proxy.golang.org` / `network is unreachable` | `go-module` exports `GOFLAGS=…-mod=readonly`, defeating Go's vendor auto-selection — re-export `GOFLAGS="-v -x -mod=vendor"` in `src_compile` (§3.43) |
| upstream has no GitHub/Forgejo/GitLab, only a cgit repo | `builtin-dirlisting` on the cgit tags page (`<repo>/refs/tags/` lists `snapshot/…tar.xz` links — passt pattern, §3.45); no pinned noop needed |
| `builtin-dirlisting` selected but upstream home is an HTML **download page** (not an autoindex) | mark-devkit joins `dir.url + "/" + href` verbatim → a page URL yields `…/download.html/less-710.tar.gz` 404 + ZERO artifacts with rc 0 (§2 assert): keep the page as `dir.url` for version discovery but pin `assets[].url` with a `{{ .Values }}` template (precedents: libcap, elfutils, less macaroni-os/kit-fixups#605) |
| generated ebuild has **no `SRC_URI`** or the Manifest digest is wrong | `assets` is an atom-level key (peer of `vars:`, under `defaults:` or the package entry) — under `vars:` it is silently dropped and the run still prints "All done"; a hand-written `src_uri:`/`mirror://` is not digested correctly, so use `assets: - name/url` with the real download address (§3.55) |
| rootless container image pull dies `mkdir /var/tmp/container_images_storage…: permission denied` | the sandbox `/var/tmp` is not the standard 1777 — fix the container mode, not the spec (§7x) |
| systemd unit models the API socket as `Environment=SOCKET=…` | upstream shape is a `.socket` unit + socket activation, with `EnvironmentFile=-/etc/conf.d/<pn>` only for `LOG_LEVEL`-style knobs (§3.46) |
| one PN needs a **legacy CVE-served track + floating tip** simultaneously | multiple `packages:` entries for the SAME PN with per-entry `selector:` + `revision:` + `patches:` (gcc-cluster precedent; sys-devel/patch macaroni-os/kit-fixups#601 generated `2.7.6-r4` with 12 live patches AND the 2.8.1 tip from one `--pkg patch` run; `.Values.p` excludes `-rN`, so bumping a revisioned track keeps the distfile pair stable, §3.26) |
| `error: rustc X is not supported by the following packages: … requires rustc Y` | declaration gate ≠ compile fail: `cargo_src_configure --ignore-rust-version` (no eclass var — §3.50), per-branch ceiling probe, disclose; never bump the toolchain yourself (§9) |
| silent `rustc` SIGABRT (exit 134, no diagnostics) on random tiny deps ONLY inside the portage sandbox | nested-container sandbox-IPC flake: control build the same tree unsandboxed + merge `FEATURES="-sandbox"`, disclose the full build-attempt tally in the PR body — do not touch the spec or add `restrict:` |

## 5. Verification standard (per PR)

1. Reproduce failing build; capture exact error lines.
2. Fix in the meta-repo kit copy → `ebuild <file> manifest` → `emerge --quiet-build -1 =cat/pkg-ver`
   → **0 errors** → `qlist -Iv` → functional smoke (binary `--version`/`-V`, link test, upstream
   `runtests.sh`/ctest, rebuild one cheap reverse dep if <5 min).
   **Byte-identity oracle:** portage installs the *exact* ebuild it used to
   `/var/db/pkg/<cat>/<PF>/<PF>.ebuild` — the file name carries `-rN` when the ebuild has a
   revision (vim-core session 2026-11-12: observed `<PF>.ebuild`, not `<pn>.ebuild`) —
   `cmp` a freshly generated ebuild against it to
   prove "generated now == built back then" with no digests/binpkgs needed (§3.40 proof bar).
3. Port the change to a kit-fixups branch → verify the **generated** ebuild via
   `mark-devkit autogen-thin --specfile autogen.kit.d/base.yml --kitfile <merge file> --pkg <PN>`
   (PN not spec-key; `GITHUB_TOKEN` via `incus exec --env`; one `--pkg <PN>` run regenerates
   ALL selector tracks of that atom — e.g. loki yields 3.7.9 + 3.6.19 — while `-bin` PNs need
   their own run) → check selected version, `SRC_URI`,
   `post_src_unpack`/`S` → deploy generated ebuild → build again.
   **Merged-into-kit-fixups ≠ in-the-tree**: when a PR B depends on a dependency your PR A just
   added to kit-fixups, A's ebuild is NOT in the kit until the MARK bot's merge/clean phase runs
   (may be days). To verify B meanwhile, regenerate A's package from A's merged spec in a scratch
   worktree (`git worktree add --detach /tmp/a-wt upstream/mark-unstable` → autogen-thin → deploy
   its ebuild + files_dir into the kit copy) rather than blocking on the bot. (awscliv2 #492 vs
   aws-crt-python #491.)
   **Rust/Go source flavours**: `autogen-thin` does not run extensions, so a
   `mirror://macaroni/…-mark-*-bundle-…` distfile stays unfetched. Generate it with full
   `autogen --skip-merge --keep-workdir --sync=false … --to <dir>`, stage `<dir>/downloads/*` into
   DISTDIR, then `emerge -1` (§3.44); finish with the `cmp` byte-identity check against
   `/var/db/pkg/…/<PF>.ebuild` (§5.2).
4. Restore sandbox state (git checkout per kit, remove test dirs, shred temp token files).
5. Only then: PR / review replies with the captured evidence (PR body carries the
   `Closes:`/`Bugs:` line per §3.23; label the PR `ai_ass` where the repo defines it, §3.39).

## 6. Subagent operating model (if your agent supports them)

- **Sequential, one issue per foreground subagent.** Shared build container + review-focus means
  parallelism buys nothing and corrupts evidence attribution.
- **The one-container rule extends across sessions: never share a build container between
  concurrent agent sessions** (macaroni-fs incident 2026-11-06). One session's sanctioned
  clean-slate `docker system prune --all` wiped the other's 48-image layer cache mid-benchmark,
  and its daemon swaps killed the other session's live chain build (`rpc error … EOF`); both
  transcripts briefly described different worlds and the two sessions raced on the same
  living-handoff file, producing contradicting commits on one PR branch that needed a corrective
  push. Sequential subagents *within* a session are not enough: a second concurrent session gets
  its own container, and the session that does NOT own the loop's repo leaves its living docs
  (e.g. MFS-KICKOFF.md) to the owner. Same for working TREES: if a foreign living-doc file
  appears untracked in your repo CWD (three core-kit loops shared one checkout 2026-11-15),
  never commit, edit, or "reconcile" it — it encodes the OTHER loop's perception; re-sync your
  own state from issues/PRs ground truth. Two live incidents on the playbook repo itself: (a) pushing AFTER the
  branch's PR merged strands the commits (twice: #24, #25 — ALWAYS check `pulls/<n>.state`
  immediately before a playbook push); (b) concurrent loops on ONE checkout interleave commits
  into each other's branch history (a GLM-loop commit rode a Qwen-loop force-push). Run playbook
  work in a dedicated `git worktree` per loop; stranded commits are re-landed onto a fresh
  branch via `cherry-pick -x` (provenance preserved), not re-typed.
  Symptom on the kit-fixups loop (core-kit #880, 2026-11-14): the shared `/kit-fixups` checkout
  switches to a foreign branch mid-task and the working tree grows mixed hunks. Countermeasure
  while contaminated: capture evidence, remove ONLY your hunks from the shared copy, build/commit
  your PR from an isolated `git worktree` (never the shared checkout), and prove PR purity via
  `pulls/<n>/files`. To honor the rule cheaply, clone the live sandbox on demand —
  `incus publish <CONTAINER>/clean-base --alias contrib-base` (reads only the snapshot; the other
  session keeps running) → `incus launch contrib-base <NEW-NAME>` → sanity-probe `/kit-fixups`
  (git identity, porcelain), kit trees clean, `mark-devkit --version`, egress → pull kit-fixups to
  current head → `incus snapshot delete <NEW-NAME> clean-base && incus snapshot create <NEW-NAME>
  clean-base`. Keep the snapshot name identical across sandboxes so briefings stay portable, and
  say in every briefing WHICH container it owns and that touching the other is forbidden.
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
   Use `-f body="$(cat file)"` or `gh pr comment --body-file`. Mirror failure (observed 2026-11-15
   on mark-issues#908): a body with backticks inside **double quotes** command-substitutes → the
   mangled text posts and reads as gibberish ("`cmp`-identical" → "-identical"). Safe pattern for
   any body containing backticks/`$`: write it to a file and post via `--input` JSON
   (`jq -n --rawfile b file '{body:$b}'`) or single-quoted heredoc; PATCH-edit such comments
   immediately and re-fetch to confirm. (Caught by human PR review; fix =
   delete comment + repost real content as threaded replies.)
b. Shell-quoting slips echoing `$(gh auth token)` into transcripts → rotate (`gh auth refresh`).
c. EOF-append rebase conflicts in `<kit>/autogen.kit.d/base.yml` (several PRs append the same
   file end): resolve = keep both blocks, strip ALL conflict markers, then **verify**:
   `grep -cE "^(<<<<<<<|=======|>>>>>>>)"` must be 0, `yaml.safe_load` must pass (a plain
   mark-contrib container ships no PyYAML — a clean `mark-devkit` parse is the stand-in gate),
   diff your block against the pre-rebase version (watch trailing nested list items getting eaten), amend to one
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
p. Cross-repo references have TWO traps. (1) Bare `#NNN` resolves **in the repo you are
   posting to** — a status comment on mark-issues#844 saying "tracking #511" silently links
   mark-issues#511 (`[gpu-configurator]…`), not kit-fixups#511. (2) The short `kit-fixups#511`
   qualification does NOT autolink either — GitHub renders it as plain text (verified against
   the comment's rendered HTML). The only linked forms are bare `#NNN` (same-repo refs), full
   URLs, and **`owner/repo#NNN`** — write `macaroni-os/kit-fixups#511`. Ranges/slash-lists too:
   `#504/#505/#506` leaves tails unlinked — write each one out. Audit before posting: strip
   qualified `owner/repo#NNN` forms and URLs from the body; any remaining `#\d+` is a bug
   (observed 2026-10-30 in four of my own #844 comments; fixed via
   `gh api -X PATCH repos/.../issues/comments/<id>`, then RE-CHECK rendered HTML with
   `-H "Accept: application/vnd.github.html"` — the raw markdown never shows whether a link
   formed; edit history stays visible, own the correction). Caveat found later (PR
   macaroni-os/kit-fixups#513): via `gh api --jq`, that Accept header can still return the RAW
   markdown for PR bodies and review comments — the render ground truth is graphql
   `bodyHTML` (`pullRequest(number:N){ bodyHTML }`). Two review-thread API quirks (2026-11-02):
   `PullRequest` has no `reviewComments` connection — render-audit a review comment by fetching
   its REST `node_id` and querying `node(id:)`; and a reply is GET-able only at
   `pulls/comments/<id>` (the nested `pulls/<n>/comments/<id>` form 404s). Render-audit scan
   rule (kit-fixups#567, 2026-11-12): audit TEXT nodes, not raw `bodyHTML` — GitHub wraps the
   `Closes` keyword in a span whose `aria-label` reads "This pull request closes issue #NNN.",
   so a naive `#\d+` sweep over the HTML fires a false positive on EVERY well-formed closing
   line; strip tags/attributes first, then apply the §7p owner/repo-strip rule.
q. `bash -n` is NOT a functional gate: `declare -A mod{pn,pv}` brace-expands to `modpn`/`modpv`
   — the associative arrays are silently never created, `${!mod_pn[@]}` comes back empty, the
   `IUSE+=` loops no-op, and portage then masks the ebuild (`USE flag … not in IUSE`) while every
   syntax check passes. The underscores belong INSIDE the items (`mod{_pn,_pv}`) like the tree
   original had them. Ebuild changes are proven by dep-calc + merge (`emerge -1`), never less.
r. `emerge --changed-use =cat/pkg-<ver>` is a **no-op when a different version is installed** —
   changed-use only considers installed packages. A USE-gated test you believe you ran simply
   never ran, and the transcript still looks alive. To prove a specific flag: `package.use`
   entry + plain `emerge -1 =cat/pkg-<ver>`, then grep a functional trace in the built artifact
   (e.g. `nginx -V` showing `add-module=…njs-1.0.2…`) — not just the install line.

s. **Multi-module Go upstreams break mark-go-bundle** (alloy probe 2026-11-01): the golang
   extension bundles only the FIRST artefact's top-level `go.sum`, so a repo whose real `main`
   lives in a second module wired via `replace ../` (grafana/alloy's `collector/`) is missing every
   module@version that appears only in the submodule's go.sum (alloy: 49 — root 1228 zip-rows vs
   collector 1241, 0 hash mismatches) and the `GOPROXY=off` build dies with "module lookup disabled"
   at every engine-registry import site. Do NOT spec such a source flavour until the extension can
   feed multiple go.sums (feature ask: `extensions_defs: golang: opts:` extra-go-sum paths);
   ship the -bin flavour and report the set-diff evidence (macaroni-os/mark-issues#848).
   Follow-up finding (go-module round): the same multi-module shape breaks the go-directive
   override too — every `replace ../..` sibling module carries its own `go` floor, see §3.37.

t. **A green merge in the sandbox only proves the UNSTABLE tree** — merge-phase builds run against
   the release branches' older toolchains (observed: vendored `go build` passed on my 1.27.3
   sandbox and failed `GOTOOLCHAIN=local` on a 1.27.2 release branch — the #848 tempo reopen).
   Language floors (`go` directives, `requires-python`, C++ standards) must be checked against the
   release trees the packages propagate to (macaroni-os/core-kit / lang-kit branch contents), and
   the house escape hatch used, not a version freeze: `go-module` for Go (§3.37), selector caps
   with rationale for build tools (§3.12).

u. **A red CI on your PR may be repo-wide bit-rot — run control checks BEFORE touching your
   code**: (1) do unrelated branches fail identically? dependabot PRs are the perfect control
   group; (2) when was the base branch last green, and what hasn't changed since (runner images,
   unpinned `go install`/deps upstream releases); (3) does the failure text reference anything in
   your diff at all? Observed on macaroni-os/mark-devkit#9: `sudo -E env "PATH=$PATH" make deps`
   broke when runner images started resolving sudo's HOME to /root (`go install` → /root/go/bin,
   off PATH → Error 127 before any test ran), behind it an unpinned `go get gomega` desyncing a
   VENDED tree (compile failures), plus a `release --clean` job triggered on `pull_request` where
   it structurally cannot pass — all three predated the PR (same failures on a 2026-10-21
   dependabot branch). Fixing infra inside the feature PR is acceptable when the diagnosis is
   airtight: separate commit, local gate (`make deps` + suite) first, full disclosure in a PR
   comment, feature code untouched.

v. **A go.sum bundle set-diff is only as sound as its path decoding** — the `mark-go-bundle`
   tarball lists PROXY paths that are URL-encoded (`cel.dev%2Fexpr%2F@v%2Fv0.25.3.zip`); to compare
   against go.sum `module@version` you must, IN ORDER: (1) `sed %2F→/`, (2) strip `mark-go-bundle-<pn>/`
   + `/@v/`→`@` + `.zip`, (3) undo case-encoding `!x→X`, and filter to `.zip` (the `.mod` rows inflate
   the count). A mis-ordered decode matches NOTHING and every collector-only module shows
   "MISSING" — a false alarm that looks like a real bundle defect. Persist the RAW inputs (both
   go.sums + the bundle tar-list), not intermediate set files — observed: the derived
   set-root/coll/union.txt were 0 bytes while only the bundle list survived, yet the claim was
   already public. Re-derive from raw and assert `bundle_zips == union`, `collonly_NOT_in_bundle==0`,
   `bundle_NOT_in_union==0` before trusting any posted number (this is §7e applied to your OWN
   orchestrator check: verify the artifact backs a public claim BEFORE posting it, not after).

w. **A nested Incus sandbox cannot prove rootless container `run`** — rootless podman/crun must
   mount `proc` inside a user namespace, which the unprivileged container denies (`crun: mount
   'proc' to 'proc': Operation not permitted: OCI permission denied`). Prove it is the environment,
   not the package, with a bare `unshare -Urmpf --mount-proc` (same EPERM) and a second runtime
   (runc fails identically). Rootless `podman info` + `pull` ARE still provable (storage created
   under `$HOME`, netavark selected) — report exactly those greens and mark the `run` step
   sandbox-blocked instead of hiding it (podman stack #861).

x. **Sandbox `/var/tmp` mode 755 breaks rootless image pulls** — containers/image stages copies in
   `/var/tmp` (`mkdir /var/tmp/container_images_storage…: permission denied`); the standard mode is
   1777. Check `stat -c %a /var/tmp` before blaming the package and restore 1777 for the test
   (container state, never in the spec).

y. **`mark-devkit autogen` defaults to `--sync=true`** — a review-side local run uploads generated
   artifacts (Rust/Go bundles) to the shared S3/minio mirror. Always pass `--sync=false` for
   review-side generation; the mirror upload belongs to the bot's post-merge job (§3.44).

z. **A stray PENDING review by your own account silently blocks threaded replies** (kit-fixups#547
   round 2, 2026-11-08): `POST pulls/<n>/comments/<id>/replies` fails `422 user_id can only have one
   pending review per pull request` when an unsubmitted draft review exists on the PR (left behind by
   an earlier failed/aborted comment attempt — invisible unless you sweep `pulls/<n>/reviews`). The
   "ignore the draft" instinct is wrong once a reply is due: the draft must be DISMISSED
   (`gh api -X DELETE repos/<org>/<repo>/pulls/<n>/reviews/<id>` — safe for your own empty-body
   PENDING review), then the reply posts as an ordinary COMMENTED review record.

aa. **NEVER assert absence from truncated API output** (mark-issues#862, 2026-11-09, three hits in one
    session): `--jq '.[0:4]'` on releases made a live project look "frozen since 2022" and reached the
    MAINTAINER in a public comment before the next probe found five modern semver releases (v1.4.0
    published 2026-09 — the NAME looked old, the DATE was fresh: sort/compare on dates and shas, never
    on version-name vibes); default pagination did the same to tags; a mis-sliced `--jq` silently
    returned `[]` for "no PRs touching X". Rule: before any public or briefing claim of the form
    "X does not exist / is frozen / has no Y", re-run unbounded (`?per_page=100`, no slice, raw length
    printed) and quote the count. Correcting a posted public claim: own it in the same thread with the
    evidence (the maintainer reply "I see regular tags" was exactly this catch).
 bb. **Read the tree Manifest for distfile FILENAMES — not your eyes on the listing** (gzip #884,
    2026-11-14): a spec assuming `gzip-<ver>.tar.gz` from the upstream listing misses that the tree
    (and the listing tip) ship `.tar.xz` — the Manifest DIST line is the ground truth for the name
    the ebuild expects, and it doubles as the mirror bit-identity proof (GARR sha512 == tree
    Manifest). Anchoring the dirlisting matcher on `\.tar\.xz$` then structurally filters the
    sibling `.sig`/`.zip`/`.tar.gz` assets for free.
 cc. **`$$` automake escapes in tree-ebuild sed/awk payloads survive into the spec body verbatim**
    (gzip #884): the tree `src_prepare` sed carried `$$source`/`$$dest` (od-confirmed); rendering it
    with a single `$` silently no-ops the substitution — the build stays green, the installed
    behavior breaks (§7q family: syntax passes, semantics dead). Diff generated-vs-tree at character
    level for every sed/awk line, not just visually.
 dd. **Two sessions on ONE GitHub account make merged work look like someone else's — and a
    cancelled subagent may have already shipped** (core-kit #880 loop, 2026-11-14/15): a diffutils
    subagent was cancelled "because the package was already merged" — but PR #584 was that very
    subagent's own pushed-and-merged commit (merged 71 s after opening; the cancel killed only the
    report). After any cancel/crash, before any conclusion or re-dispatch: compare the fork branch
    sha, the container worktree commit, and `pulls/<n>/head.sha` — identity proves ownership of
    "mystery" work. Re-check issue state + PR search IMMEDIATELY before dispatch, not only at
    claim time; claims in the other session's transcripts are invisible to your board. The
    ownership discriminator is the commit TRAILER, not the account: process review comments ONLY
    on PRs whose head commit carries YOUR `LLM-assisted-by: <your model id>` — foreign-trailer
    PRs (#581 zlib was GLM-5.3-Flash) belong to their session even under the same author; a
    "takeover" of one was cancelled mid-flight before it touched anything.
    Two corollaries proven by the gmp #898 race (2026-11-15, ended STALE, zero public writes):
    (a) **branch names must carry the model tag** (`gmp-autogen-898-qwen`) — the default
    `<pkg>-autogen-<issue>` convention collides on the shared fork, and a push `rc=1
    non-fast-forward` IS the race signal: `ls-remote` + read the foreign head commit's trailer
    before anything else, STOP if it's not yours (force-pushing over a foreign-trailer branch is
    forbidden); (b) keep the full verification package anyway (`/root/<pkg>*/`) — a STALE round's
    evidence is directly reusable if the winning PR fails review.


bb. **Python edit-scripts that call `write_text` twice from a stale `t` silently revert the first
   edit** (maya #932 layout round, observed 2026-11-20): a spec-edit script did
   `p.write_text(t.replace(a, b))` and then, for a second edit, recomputed from the ORIGINAL `t`
   and overwrote the file — the assert on the first pattern passed, "DONE" printed, the amended
   commit and both container verifications ran the UNREVERTED change, and it surfaced only as an
   impossible installed-file content three verification rounds later. Chain all replacements on
   one variable, write once, and after any multi-edit script GREP the final file for both edits
   (and for what must be gone) before committing.

cc. **Rebase keep-both on a top-level-cluster conflict must be reconstructed, not
    marker-stripped** (maya #628 rebase onto post-#630 `mark-unstable`, 2026-11-20): the maya
      cluster and the merged strata cluster BOTH append a new top-level key at the EOF of
      `ai-kit/autogen.kit.d/base.yml`, so git reports a content conflict with the two keys
      side-by-side. Deleting the `<<<< ====> >>>>` lines keeps BOTH full cluster bodies — but they
      are already BOTH in the conflicted region, so the result was a file where `strata:` (line
      1143) and `maya:` (1144) sat adjacent with strata's whole block stranded INSIDE maya (1567
      lines vs the correct 1677; autogen then aborts with no ebuild — §7e silent-success caught it
      via the artifact assert). The correct fix is deterministic reconstruction, not merge
      cleverness: `git show origin/<br>:<file> > file` (the clean upstream file, strata intact)
      then `git show <our-head>:<file> | awk '/^maya:/{f=1} f' >> file` (append ONLY our cluster
      from our own pre-rebase commit) — then assert the two invariants: head-of-file `diff` against
      upstream is empty (PREFIX-IDENTICAL) and the tail `diff` against our commit is empty
      (TAIL-IDENTICAL). `merge.kit.d` conflicted the same way but is a flat list where the naive
      strip WAS right (verify by `diff` vs upstream = exactly the one added `- pkg:` line). ALWAYS
      re-verify by regenerating EVERY top-level cluster in the file (both maya AND strata from the
      rebased spec), not just yours — the strata generation is what proved the neighbor survived;
      the render diff against your own pre-rebase ebuild shows only the version-line bump (tip
      rode 1.1.68→1.1.69 mid-rebase) — regenerate the OLD head to the SAME tip and cmp
      BYTE-IDENTICAL to separate "my rebase broke it" from "upstream moved". autogen-thin needs
      `--kitfile merge.kit.d/base.yml` too (rc=1 "No kitfile param defined" without it, and `--ask`
      is never used).

## 8. Feedback sweep (do it between every task)

```
gh api "repos/macaroni-os/kit-fixups/issues/<N>/comments?since=<ISO>"
gh api "repos/macaroni-os/kit-fixups/pulls/<N>/comments?since=<ISO>"
gh api "repos/macaroni-os/kit-fixups/pulls/<N>/reviews?per_page=100"   # `since=` is IGNORED here — filter client-side on submitted_at (observed 2026-11-15); empty-body COMMENTED = inline arrivals (§7l)
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

## 10. macaroni-fs (MFS) — stage1/stage2 spec work

A different loop from kit-fixups: `macaroni-os/macaroni-fs` builds Macaroni from scratch with
`anise-build` (luet fork) specs — plain anise YAML + Go templates, no autogen/mark-devkit.
The `step1/` tree tracks the **LFS 13.x systemd-track book** (13.1: gcc 16.2, glibc 2.44,
kernel 7.1.8): read `linuxfromscratch.org/lfs/view/systemd/…` — `/view/stable/` is the frozen
SysV 12.4 book. The book drifts (rolling snapshots bump versions) — MFS pins its own.
Distilled from macaroni-os/macaroni-fs PRs #1–#3 (Oct–Nov 2026).

1. **Sandbox**: `macaroni/phoenix-builder` Incus container (ships anise-build 0.45.1 + docker 24
   — zero install needed). **`security.privileged` is NOT required** — verified 2026-11-06: a
   throwaway unprivileged container with only the `docker` profile ran `dockerd
   --storage-driver btrfs` and a NAT container (`docker run --rm busybox echo DOCKER-NET-OK`)
   on the btrfs pool. Declare the netfilter prerequisites on the Incus side — maintainer
   recipe (macaroni-os/ai-agent-playbook#12):
   a `docker` profile with `security.nesting=true` plus `linux.kernel_modules:
   ip_tables,ip6_tables,netlink_diag,nf_nat,overlay,br_netfilter` — incusd runs `modprobe -b <mod>`
   for every entry at each container start — demonstrated end-to-end 2026-11-05: with all five
   unloaded on an idle host, a throwaway launch with the profile brought them back; an entry
   absent from the host kernel aborts the launch with `Failed to load kernel module '<mod>'…`,
   so the modules follow the container, not the boot — keep the list matched to the host kernel.
   Do NOT add names the host does not have: `xt_stat` is not a module in the current Omarchy
   kernel (7.2.15), and a literal entry aborts every launch of the instance with
   `Failed to load kernel module 'xt_stat': modprobe: FATAL: Module xt_stat not found in
   directory /lib/modules/7.2.15-1-omarchy` (verified 2026-11-06); the six-entry list above is
   sufficient. Manual `sudo modprobe -a …` is the fallback only (in-container `modprobe` fails
   `Function not implemented`); without the modules, Docker-in-incus NAT dies with the cryptic
   `iptables … Table does not exist`.
     **Storage driver (measured 2026-11-06)**: XFS with overlay2 works end-to-end here, but it is
     **not faster than the btrfs driver on this host** — the sandbox stays on the btrfs driver.
     Benchmark (loopback shape below, daemons serialized, identical registry digests per pull
     re-checked across both storages; alpine 8.4 MB, golang:1.27-alpine 253 MB unpacked):
     pulls are network-bound and a wash (alpine 2.5-2.9 s vs 2.6 s btrfs; golang 10.2-11.6 s vs
     10.9-13.0 s); a 20k-file create+delete in the writable layer is a wash (3.0-3.5 s vs
     2.8-3.0 s); ten cold `docker run --rm` cost 7.8-15.0 s vs 6.0-6.4 s; and the synthetic
     build (6x200 MB COPY layers + extract/delete of a 20k-file tar) ran
     **27.0 / 55.8 / 70.2 s on overlay2-on-XFS vs 16.1 / 15.3 s on btrfs** — XFS degraded run
     over run as the sparse loop file grew (loop-over-CoW-btrfs double penalty). A real
     dedicated XFS partition would dodge that penalty, so the maintainer's partition claim
     (macaroni-os/ai-agent-playbook#12) stays open — but untestable here (single LUKS/btrfs
     disk, no spare partition) and nothing measured suggests XFS+overlay2 beats btrfs in this
     sandbox. Both shapes need `mkfs.xfs -n ftype=1` (overlay2 needs d_type; prove with
     `xfs_info` → `ftype=1`, `docker info` → `Supports d_type: true`):
     - production shape (the lxd-compose-galaxy `docker-xfs-fs` profile equivalent): host XFS
       partition/mount exposed to the container as a disk device —
       `incus config device add <c> docker-storage disk source=<host-xfs-mount> path=/var/lib/docker`
       — then `dockerd --storage-driver overlay2`. Works unprivileged, and the storage lives on
       its own device, independent of the container rootfs and the daily base-image rotation.
     - loopback shape (what was benchmarked; zero-partition variant): privileged container plus
       TWO prerequisites. (1) `xfs` on the host kernel — use the same module mechanism as the
       netfilter list above: `incus config set <c> linux.kernel_modules xfs,ip_tables,ip6_tables,netlink_diag,nf_nat,overlay,br_netfilter`
       (keep the six existing entries) + container restart — incusd runs `modprobe -b xfs` and
       the HOST gains `xfs` in `/proc/filesystems` without the agent needing host sudo
       (verified 2026-11-06; in-container `modprobe xfs` still fails, no modules dir).
       (2) loop devices are NOT passed by privileged alone — `losetup` fails
       `Operation not permitted` and even `open(/dev/loop0)` is EPERM until the host hotplugs
       them: `incus config device add <c> loop-control unix-char path=/dev/loop-control` and
       `incus config device add <c> loop0 unix-block path=/dev/loop0 major=7 minor=0` (explicit
       major/minor is required when the host has no `/dev/loop*` nodes). Then inside:
       `truncate -s 30G /xfs.img && losetup -f --show /xfs.img && mkfs.xfs -n ftype=1 <loop>
       && mount <loop> /xfs-docker && dockerd --data-root /xfs-docker/docker --storage-driver overlay2`.
       `docker info` reports `Backing Filesystem: xfs`, `Supports d_type: true`;
       `Native Overlay Diff: false` on XFS is expected (native diff is ext4/btrfs only).
     The loop mount (and dockerd) must be re-created after a container restart, same as before.
     Recreation note (re-verified 2026-11-05): `incus launch macaroni:macaroni/phoenix-builder`
     works plain — the 2026-11-04 `Image … not found` was transient, the fingerprint workaround is
     dropped per maintainer. The image has no `/root` at first boot: `mkdir /root` before the first
     exec that writes there.
2. **`rc=0` does not mean built**: `make build` with empty `PACKAGES`, anise-build `--all`
   (macaroni-os/anise#36), and cache-pull denials all print `warning` and may exit 0 with
   zero artifacts. Count `build/*.package.tar.zst` and match per-package ✔ lines against the
   DAG size before believing any run. Use `BACKEND=docker REPO_CACHE=` — the Makefile's
   `dockerv3` default is rejected by released anise-build. `--pull` does NOT invalidate local
   cache images: anise-build reuses `anise-build/cache:*` when the spec/tree hash matches, even
   after the base image rotates (verified 2026-11-05 — a chain run silently verified the old
   rotation). Run `docker system prune --all --volumes -f` before any clean-slate claim and quote
   the base-image digest actually used.
3. **CI (`make validate`) only parses specs — it NEVER executes steps.** Every functional
   claim must come from a local full-chain `make build` + payload smoke. Verification
   standard: rebuild the whole chain from the seed at final head, then run built binaries
   via the SYSROOT loader: `/mfs/usr/lib/ld-linux-x86-64.so.2 --inhibit-cache
   --library-path /mfs/usr/lib <bin> --version`. A target-tool `GLIBC_2.xx not found` on the
   sandbox ROOT is an old-host artifact — re-check inside the current env image before
   filing it as a bug. Maintainer-endorsed rootfs link audit (since 2026-11-04, run on every
   fresh seed): `whip h linking.check` — walks `/usr/lib*`, `/usr/bin`, `/bin`, `/usr/sbin`,
   `/usr/libexec` and ends `[linking.check] Completed correctly.` (whip ships in the builder seed).
4. **Layer model**: each package builds FROM the dependency closure's cache image; the local
   docker daemon holds them, so tail-of-chain iterations cost minutes, full chain ~2 h.
   Any seed change cascades everything. `REPO_CACHE=` keeps pushes out of
   `macaronios/mfs-anise-cache`.
5. **House spec pattern (double install)**: live `make DESTDIR=${MFS} install` + payload
   `make DESTDIR=/step1/<pn>${MFS} install`, `package_dir: /step1/<pn>/mfs`. Anything the
   build fixes post-install (compat symlinks, seds) must be applied to BOTH trees.
   **Clean the payload dir first** — `rm -rf /step1/<pn>/*`, not `mkdir -p`: the directory
   can already exist with files from an earlier build, and those stale files would land in
   the anise package (geaaru on macaroni-os/macaroni-fs#7, 2026-11-11).
   Vendored patches live in the package dir and arrive injected at `/anisebuild/` — stage
   them into `${MFS}/sources/patches/` (maintainer architecture: a future `step1/sources`
   capture package ships them to step2 so nothing re-vendors; PR #2 review).
6. **YAML float trap**: quote two-segment versions (`version: '6.6'`, label `'5.3'`) in
   both definition fields or they parse as numbers.
7. **Toolchain landmines (PR #3, check before blaming a new spec)**:
   - pass-1 `--with-newlib` gcc installs the BOOT `limits.h` (`MB_LEN_MAX 1`, no
     `include_next`); glibc ≥2.42 + GCC-16 default C23 auto-fortify then kills every
     gnulib package with `Assumed value of MB_LEN_MAX wrong` in `bits/stdlib.h`. Fix:
     proper wrapper — `include_next` the C library first (its values win), then
     `glimits.h` for compiler macros. The pass-1 driver also omits libstdc++ headers
     entirely → pass `-isystem $GXXD…` INSIDE `CXX=` (not CXXFLAGS) so `CXXCPP` sees them
     too (autoconf header probes otherwise take legacy `#include <iostream.h>` paths).
   - Build-machine tools: `PATH` puts cross gcc at bare `gcc` first → always pin
     `CC_FOR_BUILD=/usr/bin/gcc` (bash's mkbuiltins links against the sysroot libc and
     cannot run) and ncurses needs explicit `--with-build-cc/--with-build-cxx`.
   - ncurses' own g++ probe hardcodes `-I/usr/include` against the build container's old
     floatn.h → `-fpermissive` demotes the `_FloatNN` builtin-collision errors.
   - The seed must not assume base-image contents: `terragon-minimal-devel:latest-amd64`
     rotates daily (one shipped no grep/gawk/find/gzip + a dangling `/usr/bin/awk` that
     broke virtual/base's finalizer). Seed lists everything explicitly; non-load-bearing
     anise calls get `eval '… || true'` (house style). `--pull` means the chain can break
     with zero spec changes — probe the fresh image first:
     `docker run --rm --entrypoint bash <image> -c 'for t in grep awk sed find gzip; do
     command -v $t \|\| echo MISSING:$t; done'`.
   - Rotation takes more than tools: the same day's `latest-amd64` also shipped **no
       libstdc++/libgcc_s shared libs anywhere**, while pass-1 gcc installs them only under
       `/usr/lib/gcc/<triplet>/` — invisible to the loader, so every `g++` dies rc=127 and gmp's
       configure fails with `C++ compiler not available`. Seed-prelude fix: write
       `/etc/ld.so.conf.d/99-gcc-runtime.conf` + `ldconfig` (macaroni-os/macaroni-fs#3, seed `+12`)
       — base-image runtime libs are on the "must not assume" list too. And when a maintainer says
       an image-side fix has landed, probe the CURRENT digest (`docker pull` + file probe,
       captured) before dropping the in-repo workaround; if the rotation doesn't carry it yet, keep
       the seed self-sufficient and reply with the evidence (§6).
    - The `/tmp` guard belongs BEFORE `gcc-config` — it is the consumer, and its absence broke
      the toolchain while the guard sat at the END of the seed prelude (geaaru's diagnosis of the
      failure round 1 misattributed to a "missing 05gcc conf" the image in fact already shipped:
      `eval 'mkdir /tmp || true' && chmod a+rwx /tmp` early in the prelude, macaroni-os/macaroni-fs#3
      `+14`, real fix pending terragon-side). Root-cause humility: a rotation probe proves the
      symptom, the maintainer can reframe the cause — re-test attribution after each fix and drop
      workarounds written for hypotheses that don't survive (the `printf` conf prelude died in the
      same commit that hoisted the guard).
    - **`MAKEOPTS` is not read by GNU make.** A spec-level
      `export MAKEOPTS="-j$(nproc)"` is inert while the calls stay plain `make` (verified
      2026-11-06: `$(MAKEFLAGS)` stays empty, and gcc pass-1's `RUN` step still took 2066.5 s
      with the export in place). The value only reaches make via `make ${MAKEOPTS}` or by
      exporting `MAKEFLAGS`. Fixed in macaroni-os/macaroni-fs#6: gcc, glibc and libstdcxx pass
      `${MAKEOPTS}` now; measured on a clean 24/24 chain, gcc 2066.5→236.2 s, glibc
      102.8→17.6 s, libstdcxx 100.4→49.7 s. GCC needs its goals split (`all-gcc` before
      `all-target-libgcc`): a real `-j` otherwise races libgcc's configure against the
      not-yet-built `xgcc` (`No such file or directory .../gcc/xgcc`).
    - **Pass-2 toolchain landmines (verified 2026-11-06, macaroni-os/macaroni-fs#7)**:
      pass-1 binutils installs target tools *unprefixed* under
      `cross-tools/$TGT/bin` (the `$TGT-ar` names do not exist), so pass-2 configures
      fail `could not determine $TGT-ar` — export
      `AR/AS/LD/NM/RANLIB/STRIP/OBJCOPY/OBJDUMP/READELF` from there. The unprefixed
      `g++` first on PATH is the headerless pass-1 cross compiler and kills GCC's
      libcody (`fatal error: memory: No such file or directory`) — pin `CC`/`CXX` (and
      `*_FOR_BUILD`) to `/usr/bin`. A seed zstd leaks into the installed `cc1`
      (`libzstd.so.1: cannot open shared object file` in the chroot) — configure
      `--without-zstd`. The book's `m64=../lib` sed does not stop GCC's target libs
      (incl. `libgcc_s.so.1`) from landing in `${MFS}/usr/lib64` while the driver
      searches `/usr/lib` — consolidate into `/usr/lib` and drop `/usr/lib64` from the
      staged trees. Also `rm -rf` the pass-1 source tree before unpacking, or stale
      objects from pass 1 can leak into pass 2. GMP/MPFR/MPC: the book unpacks them
      in-tree, but the team chose the anise-tracked package path instead (review
      2026-11-08): build them as packages under `${MFS}/cross-tools`, configure
      `--with-gmp/--with-mpfr/--with-mpc=${MFS}/cross-tools`, and make the chroot
      export `LD_LIBRARY_PATH=/cross-tools/lib`. Without that variable the installed
      `cc1` dies on `libmpc.so.3` (verified 2026-11-07); with it the C/C++ chroot
      compile passes. With no `--with-*` flags at all, configure stops with
      “Building GCC requires GMP 4.2+, MPFR 3.1.0+ and MPC 0.8.0+”.
      Proof bar: a real chroot C **and** C++ compile+run (add a test-only
      `/lib/ld-linux-x86-64.so.2` link because the loader lives in `/usr/lib`).
8. **Reproduce pristine, once per hypothesis**: reusing a build dir across in-tree/out-of-tree
   repro attempts produced three contradictory m4 results; one throwaway container per
   hypothesis, extracting a fresh tree, beat days of polluted context. And when a toolchain
   binary misbehaves, dump its config.log section (`-I` flags!) before theorizing —
   `-I/usr/include` was the whole ncurses story.
9. **Maintainer flow**: geaaru reviews fast, pushes seeds/specs himself between your PRs —
   always `git fetch upstream` + rebase before pushing, and expect seed version collisions
   (the `+N` suffix bumps per build iteration; squash your own intermediate bumps before
   push). Issues are disabled on macaroni-fs: findings go in PR bodies, discussion on
   Discord; org attribution trailers + `Signed-off-by` apply (no `ai_ass` label there —
   only kit-fixups defines it). Sync the PR body with every head push (version `+N`, evidence
   blocks) — stale body claims land on the reviewer's desk (a "+13" body against a "+11" head).
ab. **`git commit --amend` commits the INDEX, not your working-tree edits** (two hits on
   2026-11-15, kit-fixups#587 + #602 rounds): a fix made with `sed -i`/editors but never
   `git add`-ed leaves the amend a no-op while the public reply claims the change — one of
   the two reached the remote before being caught. Gate: after amending, `git diff <old-head> HEAD`
   must show EXACTLY the intended delta (an empty diff = the amend was a no-op), then verify the
   pushed content from the API (contents/blob or `pulls/<n>` files), never from local intent.
ac. **`mark-devkit autogen` exits 0 even when a package's selection fails** ("All done" prints
   regardless — observed on kit-fixups#602 after a rebase misplaced the xz-utils atom into the
   `pxz` noop cluster, whose `defaults.vars.versions` leaked into it). A regen "rc=0" is NOT a
   regen proof. Strict check: the log contains `For package <cat>/<pn> selected version <v>`
   AND the generated ebuild file exists under the output tree; batch audits must grep the
   de-ANSI-fied log (raw logs carry escape bytes that poison version greps). Watch for cluster
   `defaults` leaking into atoms placed inside them during conflict resolution.
