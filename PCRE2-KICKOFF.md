# PCRE2-KICKOFF — PCRE1→PCRE2 autogen conversion loop (macaroni-os/mark-issues#931)

Living session-handoff for the #931 epic loop: board state + sandbox map + task order. **Refresh
this every session** (same discipline as COREKIT-KICKOFF.md). Conventions: PLAYBOOK.md §3.28
(entities-catalog static ids), §3.29 (USE-selected init system), §3.65 (no defaults/bounds in
specs), §5 verification, §6 one-container rule, §8 feedback sweep.

## Mission

Epic #931 (follow-up of #899 / kit-fixups#627, libpcre pinned noop at 8.44): migrate the PCRE1
consumers in the tree to `dev-libs/libpcre2` so PCRE1 can be dropped. Division of labor per
§3.52: one child issue per package (`[<kit>] autogen <cat>/<pn> with libpcre2` / `probe PCRE2
first`), `Closes:` on each child's own PR, epic stays open. Conversion = autogen the atom in
kit-fixups + PCRE2 swap; where the release probed has no PCRE2 support (slang, rasqal) the
conversion ships keeping `dev-libs/libpcre` with the probe disclosed in the PR title/body.

## Board (captured 2026-11-26; re-verify via API at session start)

| state | item | issue | PR |
|---|---|---|---|
| **Round-1 ANSWERED 2026-11-26** — geaaru 2026-11-25 (1 thread: drop `v` transform); regen proves 0.12.11, byte-identical | app-text/editorconfig-core-c | #939 | kit-fixups#637 @ `9ae7326` |
| **Round-1 ANSWERED 2026-11-26** — 6 threads: systemd USE+unit, libressl dropped, libev/zlib released, readme.gentoo→einfo, MacaroniOS wording; + static ids 302, slot dropped. Regen 1.4.85, both USE sides merged, initd cycled | www-servers/lighttpd | #940 | kit-fixups#638 @ `4e40299` |
| **Round-1 ANSWERED 2026-11-26** — 4 threads: `v` transform, slot, systemd USE (unit+serviced vs initd/confd USE-selected), stray `true`→if. Regen 0.8.2, both sides merged, tftpd cycled | net-ftp/atftp | #942 | kit-fixups#639 @ `c860f1f` |
| **Round-1 ANSWERED 2026-11-26** — 2 threads: slot, EPREFIX/`${PF}`→`{{ .Values.pf }}`. Regen 4.25.1, merge rc=0, docdir literal proven | x11-wm/i3 | #935 | kit-fixups#640 @ `01cef15` |
| **Round-1 ANSWERED 2026-11-26** — 2 threads: slot (+playbook note), deps without `>=`. Regen 2.3.3, slsh smoke green | sys-libs/slang | #943 | kit-fixups#641 @ `b9af904` |
| **Round-1 ANSWERED 2026-11-26** — 2 threads: slot, plain util-linux. Regen 0.9.33, roqet smoke green | dev-libs/rasqal | #944 | kit-fixups#642 @ `2deb96f` |
| **Round-1 ANSWERED 2026-11-26** — 2 threads: slot, systemd USE (`systemd_newunit`/`newinitd`). Regen 20260811, both sides placed; daemon+log line proven with `-N` (rc-service cycle sandbox-blocked: `/dev/kmsg` EPERM, disclosed) | app-admin/metalog | #945 | kit-fixups#643 @ `a146f9c` |
| **Round-1 ANSWERED 2026-11-26** — 4 threads: slot (+pf docdir), entities-catalog ids 392 (+playbook note), systemd USE. Regen 4.2.0, account re-created at 392 proven, proxy `http=200`, both sides placed | net-proxy/privoxy | #947 | kit-fixups#644 @ `e431f16` |

**All eight: single squashed commit, lease-pushed, every inline thread answered same-day with
captured proof, PR bodies carry a "Review round 1" section. Awaiting geaaru re-review/merge.**

Open siblings in the same repo (NOT this loop's review scope unless trailers say so):
kit-fixups#636 (openjdk-bin, #964).

## Remaining epic children (open, unclaimed as of 2026-11-26 — verify before claiming)

#934 haproxy (branch `autogen-934-haproxy` exists in the sandbox, no PR yet; net-kit tree copy has
a modified `haproxy-2.9.15.ebuild` from the probe), #936 syslog-ng, #937 mongodb, #938 proftpd,
#941 postfix, #946 kjs, #948 pulseaudio; plus the already-autogen specs from the epic table:
nginx, nmap, swig, glib (dep swap inside existing specs). `/root/gen-{nginx,nmap,swig,glib,gdbus,
hp,pulse,brave-bin}` in the sandbox are prior probe outputs.

## Sandbox map

- `mark-contrib-931` — THE loop container (Incus). `/kit-fixups` clone with all `autogen-9xx`
  branches (in-repo branches on macaroni-os/kit-fixups; fetch refspec is mark-unstable-only —
  push by explicit URL, take the lease from a fresh `ls-remote`). Git identity `borisp`.
  mark-devkit 0.36.1; `autogen-thin` needs `--env GITHUB_TOKEN` (§2 silent-fail trap).
  Snapshot `pre-review-round` (2026-11-26, before the round-1 fixes). Kit trees under
  `/var/git/meta-repo/kits/*/` carry the deployed ebuilds of every verified package (documented
  dirt; net-kit also has the haproxy probe edit). `/root/gen-*-r2/` = round-1 generated trees,
  `/root/*-b[123].log` = per-side build logs — keep as the evidence package while reviews run.
  Metalog sandbox state: `/var/spool/metalog_sockets` + `/var/log/{everything,critical,pwdfail,
  kernel}` dirs created for the runtime probe (container state, not spec content).
- Exec pattern + token recipe: §2 (`printf … | newgrp incus-admin`, base64 scripts, `--env`
  BEFORE `--`).

## Feedback scope (owner rule, §7dd)

Answer review comments only on PRs whose head-commit trailer says
`LLM-assisted-by: DeepSeek V4.1 - Maya-S (748B MoE, 1M Kontext) via OpenCode` (the loop's model
tag; playbook-repo commits carry the answering session's own trailer).

## Task order

1. §8 feedback sweep on all eight PRs (issues/comments + pulls/comments + reviews — §7l).
   Round 1 is fully answered as of 2026-11-26; next arrivals are round 2.
2. When any of the eight merges: post the close-suggestion on its child issue, and rebase the
   remaining branches that touch the same spec files (lighttpd+metalog share
   `core-server-kit/autogen.kit.d/*`; slang+rasqal share `dev-kit/autogen.kit.d/base.yml`) — §3.52
   shared-file rule.
3. Only then pick up remaining children (§duplicate-check first: `gh pr list --search "<pkg>"`).
