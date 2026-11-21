# MFS loop — session handoff (2026-11-21 refresh — step2 PR in review)

Read PLAYBOOK.md first — §10 is the macaroni-fs loop (§10.10 is the step2 pattern), §5/§6/§7/§8
apply as written (§2–§4 and the kit-fixups parts of §7 do NOT apply to macaroni-fs). Verify the
board below live before acting.

## Board state (verify live before acting — it moves daily)
- Issue https://github.com/macaroni-os/mark-issues/issues/954 (`[macaroni-fs] Create step2
  phase specs`, geaaru, 2026-11-21) — claimed by this loop; design/evidence follow-ups posted.
  **macaroni-fs issues are NOT disabled** (geaaru, same day): use that repo for follow-ups.
- **PR https://github.com/macaroni-os/macaroni-fs/pull/9 OPEN** — head `524bdf8` (branch
  `step2-specs`, one commit, MERGEABLE, CI spec validation green). **Round 1 review answered
  2026-11-21** (six inline threads: /build unpacking, no step2/sources, chained deps,
  step1/wget; zlib accepted as a top-level comment). Awaiting the next review. Round 2 answered 2026-11-22 (single `requires` on `step1/wget`, versions reset to the first valid ones).

macaroni-os/macaroni-fs: step1 PR #1–#8 all MERGED. The PR adds three step1 packages the
step2 preludes need, then the seven step2 temporary tools.

### step1 additions (for the source fetch)
| package | version | notes |
|---|---|---|
| step1/openssl | 3.5.9 | LTS branch, cross-built with the pass-1 toolchain; `AR/RANLIB/NM` from `cross-tools/${MFS_TARGET}/bin` (pass-1 binutils is unprefixed) |
| step1/ca-certificates | 20261105 | dated Mozilla bundle from curl.se; requires openssl and builds the hashed cert dir (bundle split into `cert-NNN.pem` first — `openssl rehash` skips bundles) |
| step1/wget | 1.25.0 | OpenSSL-backed via a single `requires: ca-certificates`; `--with-libssl-prefix` bypasses pkg-config; no CA option → chroot exports `SSL_CERT_FILE` |

### step2 packages (chained in book order)
| package | version | requires | notes |
|---|---|---|---|
| step2/gettext | 1.0 | step2/chroot | only msgfmt/msgmerge/xgettext |
| step2/bison | 3.8.2 | step2/gettext | |
| step2/perl | 5.44.0 | step2/bison | live install + snapshot-copied payload (Perl has NO DESTDIR support) |
| step2/zlib | 1.4 | step2/perl | |
| step2/python | 3.11.18 | step2/zlib | 3.11 line as requested; no mpdecimal (bundled libmpdec) |
| step2/texinfo | 7.3 | step2/python | |
| step2/util-linux | 2.42.4 | step2/texinfo | `--without-python`, ADJTIME_PATH per book |

Design points: each prelude downloads its tarball into `/sources` only if `step1/sources`
did not already carry it, then unpacks in `/build` and removes `/build` before the final
image. `step2/chroot` copies only `/mfs` (no sources package) and exports
`LD_LIBRARY_PATH=/cross-tools/lib` + `SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt`.
`step2/base` installs `step1/wget` + explicit `step1/openssl`/`step1/ca-certificates`
(anise install does not follow the tree requires) and carries the local-workflow fixes
(`yq4 -i`, `step1/mpr`→`mpc`, duplicate gzip removed). Version bumps (existing packages only): base `1.0.0+11`, chroot `1.0.0+2`.
Local revision bumps were used to bust the image cache during verification and restored
to the first valid versions before pushing (new packages carry no `+N`).

## Sandboxes (one-container rule, §6)
- **`mfs-build`** — THIS loop's step2 sandbox. Base image is the cached **MacaronOS
  `mark-unstable` amd64 (20260316)** image, NOT `phoenix-builder`: on 2026-11-21 the
  `macaroni` remote's simplestreams index 404'd and the `phoenix-builder` alias pointed at a
  deleted serial. Docker (`app-emulation/docker` + `docker-cli`, 29.10.0) and
  `app-admin/anise-build` (0.46.0) were installed via `emerge`;
  `dockerd --storage-driver=overlay2` (btrfs graphdriver is compiled out of the Macaroni
  docker build). The `docker` Incus profile is required (nesting + netfilter modules). No
  `/root` in the image until `mkdir -p /root`.
- Local repo for `local_build: true`: step1 artifacts in `/root/mfs-repo`, metadata via
  `anise-build create-repo --type http --urls http://172.17.0.1:8000`, served with
  `anise-build serve-repo --port 8000 --dir /root/mfs-repo`. Values in
  `/root/mfs-local-values.yaml`. Keep the served repo OUT of the checkout (`make build`
  deletes `build/`).
- Sandbox-only workaround: the step1/wget build hits overlay2's **128 lower-layer limit**
  because its builder stacks on the ca-certificates image (126 layers). The ca image is
  flattened in the sandbox and re-tagged, **restoring the config** (`docker import --change
  ENV MFS=…` etc. — a plain import drops `MFS`/`MFS_TARGET` and the dependent builds into
  `/`); then wget builds and exports normally. Do NOT use `flat_image: true` in the specs
  for intermediate step1 images: it drops the inherited `ENV` the rest of the chain needs.
- Evidence kept in `/root`: `build-step1*.log`, `build-step2-r4.log`, `probe-*.log`,
  `/root/mfs-probe/` (openssl/wget feasibility probe), `/root/mfs-testtree/` (throwaway
  combined image used before the chain existed), `payload-cmp3.sh`.

## Task order
1. **§8 sweep FIRST** — PR #9 comments/reviews + the issue thread (all three endpoints,
   always). Review asks: one-commit amend + force-with-lease + re-verify payload identity
   and the chained-image smoke; `make validate` is the CI gate (validation only, it does
   NOT build).
2. Once #9 merges: prune `mfs-build` when no review rounds are pending, refresh this file.
3. Next structural step: step3 base with Portage + Ego. That is a structural decision
   (repo wiring, subsets, package manager bootstrap): propose with evidence and let geaaru
   pick (§9). Do not start a batch of step3 specs before that.

## Hard rules
- Nothing runs on the host except gh/git/incus plumbing. Container does all builds. Never
  triple-quote into the container: base64-push a script (§2) — even one-liners that touch `$`.
- Git identity in the sandbox: use human's git name/email — commits carry the
  `LLM-assisted-by:` + `Signed-off-by:` trailers; **no `ai_ass` label** (undefined in
  macaroni-fs). Token push pattern: `--env GITHUB_TOKEN` BEFORE `--`, inline-URL, output
  silenced, verify with `git ls-remote` + `pulls/<n>` head sha. No bare `#NNN` refs in public
  text (§7p) — always `owner/repo#NNN` or a full URL.
- Only captured output goes public; re-verify a claim on the exact head being shipped before
  quoting it (§5, §7e/m). Update PR-body claims with every head push (§10.9).
- **MFS verification standard**: full local chain from the seed at final head + payload
  smoke; byte-identity where possible. Step2: payload vs live `cmp` per file (0 body diffs
  on the PR head) and the chained final image must have `/build` absent, 118 cert files +
  118 hashed symlinks in `/etc/ssl/certs`, wget HTTPS working and the tools working.
