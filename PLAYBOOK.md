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
     **BUT: `.Values.sha` is the GitHub release `target_commitish`, not a commit sha.** For a repo
     using `ignore_tags: true` the releases' target_commitish is a branch name (mimir: literally
     `main`), so the stamp renders `3.2.4-main` — for such repos fall back to plain `{{ .Values.pv }}`
     (or a tag-pinned sha from elsewhere). The distfile auto-name suffix (`<pn>-<pv>-<sha7>.tar.gz`)
     is the tag-object sha, which can differ from the unpack-root sha7 (peeled vs tag object) — the
     `mv <user>-<repo>-*` wildcard in §3.36 absorbs either.
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
      scratch dirs). Proof bar: generated ebuilds `cmp`-identical to `/var/db/pkg/<cat>/<pn>-<pv>/<pn>.ebuild`
      (§5.2) or to what you built; `-bin` trees regenerate byte-identical.

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
| `ebuild manifest` dies on a USE-gated distfile whose USE is OFF | the manifest phase fetches EVERY conditional SRC_URI regardless of USE — one dead optional-module host blocks the whole conversion (nginx `sticky`'s bitbucket tarball 404s today, aborting even the 1.26.2 manifest). Fetch-probe every module tarball as part of the reproduce step; dead host → repoint to a mirror of the SAME pinned commit (align `mod_p`/`mod_wd` to that archive's unpack root) or drop the module with justification (kit-fixups#513) |
| "upstream has no GitHub releases" — memory or a stale repo page | verify live before falling back to `builtin-dirlisting`: `gh api repos/<u>/<r>/releases`. nginx had quietly published the official tarballs as release assets (nginx.org tarball + `.asc`, current through the security lines) and the whole #849 spec flipped from dirlisting to `builtin-github` on one API call. Branch/tag listings are NOT evidence about releases |
| `go: go.mod requires go >= X (running go Y; GOTOOLCHAIN=local)` — builds on unstable, fails on a release branch | inherit `go-module` (§3.37): its `src_prepare` rewrites the go.mod directive to the local toolchain; do NOT version-pin or downgrade the source to appease a tree |

## 5. Verification standard (per PR)

1. Reproduce failing build; capture exact error lines.
2. Fix in the meta-repo kit copy → `ebuild <file> manifest` → `emerge --quiet-build -1 =cat/pkg-ver`
   → **0 errors** → `qlist -Iv` → functional smoke (binary `--version`/`-V`, link test, upstream
   `runtests.sh`/ctest, rebuild one cheap reverse dep if <5 min).
   **Byte-identity oracle:** portage installs the *exact* ebuild it used to
   `/var/db/pkg/<cat>/<pn>-<pv>/<pn>.ebuild` — `cmp` a freshly generated ebuild against it to
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
4. Restore sandbox state (git checkout per kit, remove test dirs, shred temp token files).
5. Only then: PR / review replies with the captured evidence (PR body carries the
   `Closes:`/`Bugs:` line per §3.23; label the PR `ai_ass` where the repo defines it, §3.39).

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
   `pulls/comments/<id>` (the nested `pulls/<n>/comments/<id>` form 404s).
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

## 10. macaroni-fs (MFS) — stage1/stage2 spec work

A different loop from kit-fixups: `macaroni-os/macaroni-fs` builds Macaroni from scratch with
`anise-build` (luet fork) specs — plain anise YAML + Go templates, no autogen/mark-devkit.
The `step1/` tree tracks the **LFS 13.x systemd-track book** (13.1: gcc 16.2, glibc 2.44,
kernel 7.1.8): read `linuxfromscratch.org/lfs/view/systemd/…` — `/view/stable/` is the frozen
SysV 12.4 book. The book drifts (rolling snapshots bump versions) — MFS pins its own.
Distilled from macaroni-os/macaroni-fs PRs #1–#3 (Oct–Nov 2026).

1. **Sandbox**: `macaroni/phoenix-builder` Incus container, `security.privileged=true`
   (ships anise-build 0.45.1 + docker 24 — zero install needed). The pool is btrfs → run
   `dockerd --storage-driver btrfs` with `/var/lib/docker` as a btrfs subvolume. Declare the
   netfilter prerequisites on the Incus side — maintainer recipe (macaroni-os/ai-agent-playbook#12):
   a `docker` profile with `security.nesting=true` plus `linux.kernel_modules:
   ip_tables,ip6_tables,netlink_diag,nf_nat,overlay,br_netfilter` — incusd runs `modprobe -b <mod>`
   for every entry at each container start — demonstrated end-to-end 2026-11-05: with all five
   unloaded on an idle host, a throwaway launch with the profile brought them back; an entry
   absent from the host kernel aborts the launch with `Failed to load kernel module 'xt_stat'…`), so the modules follow
   the container, not the boot — keep the list matched to the host kernel. Manual
   `sudo modprobe -a …` is the fallback only (in-container `modprobe` fails
   `Function not implemented`); without the modules, Docker-in-incus NAT dies with the cryptic
   `iptables … Table does not exist`.
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
      exporting `MAKEFLAGS`; raised with the maintainer, with the one-line fix, on
      macaroni-os/macaroni-fs#5.
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
