# K8S loop handoff — macaroni-os/mark-issues#862 (living doc, refresh every session)

**STATUS 2026-11-08 evening: LOOP COMPLETE.** All merged: kit-fixups #525, #530–#546 + macaroni-commons
#4/#5 (the full core-server-kit Kubernetes/Cilium autogen stack: etcd, cri-o, kubectl, kubelet, kubeadm,
kube-proxy, cilium-cli, cilium-agent across mark-unstable + mark-xl/iii/31). Final summary comment posted
on the closed issue (#issuecomment-6722244106 — first version had the §7p slash-list trap `#531/#532/#533`
→ tails auto-linked to mark-issues!; PATCHed to fully-qualified form; LESSON: audit must run BEFORE the
first post, and rendered link-count is NOT a proxy — same-repo bare #NNN auto-links to the WRONG repo's
issues). Playbook §3.48 + this doc → ai-agent-playbook PR #18 (open at time of writing).
Leftovers offered on the issue, awaiting maintainer if ever: iii/31 legacy plain cri-o atom → 3-track
alignment; cilium socket-group entity; -bin flavours; control-plane trio (apiserver/cm/scheduler,
entities 432–434 ready); kubelet completions symmetry; mark-devkit 0.36.1 Manifest-drift issue;
tcp-wrappers gcc-16 rot.
Container `mark-contrib-862` still up (snapshot bootstrap-clean) — keep until reviews on #18 land, then
`incus delete`.

--- historical board below (kept for reference) ---

Task: `[core-server-kit] autogen kubernetes and cillium stuff` — issue
https://github.com/macaroni-os/mark-issues/issues/862, **CLOSED completed 2026-11-08T08:53Z by the
#534 merge — intentional; thread stays alive (geaaru asked + answered follow-ups there)**.
Session state as of 2026-11-08 ~11:00 CET:

## Fresh from 2026-11-08 afternoon/evening
- MERGED by geaaru overnight: #531–#533, #534 (→ issue closed), #535–#537, #538 (cri-o on mark-xl),
  commons#4 + **commons#5 (kube-proxy gid/uid 518)**.
- **kube-proxy round**: [kit-fixups#539](https://github.com/macaroni-os/kit-fixups/pull/539) (main,
  `d7241d9ba`) + [#540](https://github.com/macaroni-os/kit-fixups/pull/540)/[#541](https://github.com/macaroni-os/kit-fixups/pull/541)/[#542](https://github.com/macaroni-os/kit-fixups/pull/542) (xl/iii/31)
  — 3 tracks, completions shipped (kube-proxy HAS cobra completions — briefing premise disproven),
  root-run + 518 account owns /var/lib/kube-proxy, OpenRC `before kubelet`. NOTE from that agent:
  installed **kubelet also answers `completion bash` rc=0** — merged #530 ships no kubelet completions;
  one-line follow-up if geaaru wants symmetry (NOT done unasked).
  - **REVIEW ROUND 2 (geaaru comment 4506242370: "create a new definition key … keep less lines")**:
    DONE. Replaced the `kubectl:` cluster with a `kubernetes:` key; shared machinery hoisted once into
    `defaults.vars`, one sprig-parametrized body (`completions`/`service`/`uid`/`entity_pr` guards),
    per-entry deltas only. net-base.yml 1484→1234 (−250), cluster 461→211, body copies 7→1; whole PR
    now net −2 (was +249/−1). **Byte-identity PASS=16/16** (12 ebuilds + 4 Manifests, all tracks,
    cmp-clean vs a pre-refactor scratch-worktree baseline; also == the 4 installed sandbox ebuilds) →
    **no rebuild, kit tree untouched.** Head `d7241d9ba`→`5945f7953` (lease), single commit, ai_ass,
    mergeable/clean. Threaded reply `r4506377862`. #540/#541/#542 atom files UNCHANGED (spec-only refactor,
    no resolution change, §3.38). Baselines kept under `/root/539-review/` (baseline/post/final + worktree).
- **cilium daemon round**: probe GREEN — cilium release tarballs are build-ready (committed vendor/,
  committed codegen; agent lives at `daemon/` NOT `cmd/cilium-agent` since 1.20.x; plain go build
  ~6 min, ldflags stamp version + GIT_VERSION-file `<sha7>` from unpack root). Shipped
  [#543](https://github.com/macaroni-os/kit-fixups/pull/543) sys-cluster/cilium (cilium-agent, init trio, root-run no-account,
  go 1.26.0 floor passes all branches) + [#544](https://github.com/macaroni-os/kit-fixups/pull/544)/[#545](https://github.com/macaroni-os/kit-fixups/pull/545)/[#546](https://github.com/macaroni-os/kit-fixups/pull/546).
  cilium-cli non-regression byte-identical.
- **OPEN awaiting review (8)**: #539–#546. One crash mid-round (§7n): dead agent left only recon
  drafts (kp-* == merged files, no work lost); continuation agent wrote everything fresh.
- api.github.com tarball endpoint 504-flapped all day — autogen-thin needs retry-until-clean; 0.36.1
  cannot reuse DISTDIR to rescue a failed download.
- geaaru on #862 (08:44): **cilium DAEMON wanted** ("having a way to build everything is better"),
  musing about dirlisting; asked whether **kube-proxy is a separate package**.
- My reply (10:0x): kube-proxy etc. = separate binaries of the same tree → cluster-entry follow-up
  (kube-proxy needs ONE entity; apiserver/cm/scheduler entities 432–434 exist); cilium needs NO
  dirlisting — cilium/cilium HAS tags, `query: tags` + #525 tarball-recovery works, the risk is the
  daemon build itself (container-tooling assumptions); committed to probe + spec if nobody beats me.

## NEXT (waiting on reviews)
1. §8 sweep on #539–#546 (+ any reply-threads same-day). geaaru merges fast — expect rebase asks
   when his own merges land between our open PRs (§7c procedure; keep-both worked 3× today).
2. Offered-but-unasked (only on his word): iii/31 legacy plain cri-o atom → track shape (offer in #538);
   kubelet completions symmetry; control-plane trio kube-apiserver/controller-manager/scheduler
   (entities 432–434 exist); cilium-operator/hubble-relay/cni siblings; -bin flavours anywhere;
   mark-devkit Manifest-drift issue; tcp-wrappers gcc-16 rot fix.
3. Housekeeping: two stale /root/chain*.sh from prior sessions sit in the container; /root/kp-done/
   holds the dead agent's recon. Harmless; purge on next clean phase.

## Board state

| piece | state |
|---|---|
| PR-A = [kit-fixups#525](https://github.com/macaroni-os/kit-fixups/pull/525) | **MERGED** (e86b720fb): etcd 3.7.3/3.6.16/3.5.35, cri-o 1.37.3/1.36.8/1.35.11, kubectl 1.37.2/1.36.6/1.35.10 — source flavours, tracks in selector AND merge conditions. |
| PR-B = [#530](https://github.com/macaroni-os/kit-fixups/pull/530) kubelet+kubeadm | OPEN, MERGEABLE, `ai_ass`, 1 commit `9a15fbb92`. Extends the net-base.yml k8s cluster; 7/7 merges green, USE-side init proof, static account via → |
| PR-B entities = [macaroni-commons#4](https://github.com/macaroni-os/macaroni-commons/pull/4) | OPEN: `kubelet` gid/uid **517** (first globally free, 435–516 contiguous). #530 cites this OPEN gid — dependency disclosed in body. kubeadm needs no account (root CLI). |
| PR-B propagation = [#531](https://github.com/macaroni-os/kit-fixups/pull/531) xl / [#532](https://github.com/macaroni-os/kit-fixups/pull/532) iii / [#533](https://github.com/macaroni-os/kit-fixups/pull/533) 31 | OPEN, one atom-file each (kubelet+kubeadm). Comment posted on #531: **cri-o atom absent on mark-xl** (present iii/31) — pre-existing gap, question asked, untouched otherwise. |
| PR-C = [#534](https://github.com/macaroni-os/kit-fixups/pull/534) cilium-cli | OPEN, `Closes: macaroni-os/mark-issues#862` (series-completer). **Source** flavour (operator: bin only later if asked, maybe for all k8s): dedicated `autogen.kit.d/cilium.yml`, vendored build (`GOFLAGS=-mod=vendor`, plain go-module, no go.work), PN `cilium-cli` → `/usr/bin/cilium`, completions wired, single 0.20.2 track. Smoke: `cilium version` → v0.20.2 + expected cluster-unreachable. |
| PR-C propagation = [#535](https://github.com/macaroni-os/kit-fixups/pull/535)/[#536](https://github.com/macaroni-os/kit-fixups/pull/536)/[#537](https://github.com/macaroni-os/kit-fixups/pull/537) | OPEN: cilium-cli atom on xl/iii/31 (+2 lines each). Per-branch go captured: xl 1.27.3, iii 1.26.9, 31 1.26.10 → eclass toolchain-rewrite exercised in merge phase there. |
| cilium DAEMON | NOT packaged — open question to @geaaru on #862 (stable releases ship zero assets; operator-deployed images). Answer pending; if he wants it: source build + init + entity (needs new free gid) as follow-up PR. |

## ⚠ Merge-order nuance (§3.23)
`Closes:` lives on #534. If #534 merges BEFORE #530–533, #862 auto-closes while kubelet/kubeadm
PRs are still open — recovery = status comment on the issue pointing at the still-open PRs (the
awscliv2 pattern), not a silent reopen. Ideally #534 merges last.

## Environment
- OWN Incus container **`mark-contrib-862`** (NEVER share, §6; `mark-contrib` = the #524 agent).
  Exec: `printf '%s\n' 'incus exec mark-contrib-862 -- bash -c "..."' | newgrp incus-admin`; anything
  with `$` via base64 script (§7 host-side `$()` bit twice). Snapshot `bootstrap-clean`.
  mark-devkit **0.36.1** (0.35.4 probes may not carry — re-probe before asserting key semantics;
  0.36.1 known quirk: regenerated go-bundle Manifest digest drift ±12–456 B, disclosed in #525 body,
  mark-devkit issue candidate).
- `/kit-fixups` remotes: fork=`org-tekeli-borisp/macaronios-kit-fixups` (RENAMED repo — old URL 404s
  as bare rc=128), origin=macaroni-os/kit-fixups. Open branches: 862b-kubelet-kubeadm-mark-unstable,
  862c-cilium-cli-mark-unstable + 3 propagation. Identity borisp <boris.pigin@gmail.com>, trailer
  block per merged #525. `GITHUB_TOKEN` per-exec via `--env` BEFORE `--`.
- Stacking watch: #530 (net-base.yml header + net-base_autogen.yml mid-file) vs #534 (net-base_autogen.yml
  EOF) — zero shared hunks by construction; rebase per §7c on any review round.

## Resume procedure
1. **§8 feedback sweep** on #530–#537 (+ commons#4; #525 merged; other open PRs: media-kit#977,
   net-kit#2506/#2476 — #524 is another agent's, long closed). Same-day threaded replies, captured proof.
2. Review asks land → amend-one-commit + fresh-lease push + head-sha verify + thread answers (§6/§7j/k).
3. Merges land → close-suggestion on #862 once #534 merges (or recovery comment if it fires early);
  propagate awareness to kit repos only if maintainer asks (§3.40).
4. Possible follow-ups (only if asked): -bin flavours for k8s/etcd (assets exist), kube-apiserver/
   controller-manager/scheduler/kube-proxy (entities 432–434 exist, near-zero cost), cilium daemon if
   geaaru answers yes, mark-devkit Manifest-drift issue, tcp-wrappers gcc-16 rot (blocks socat[tcpd]).
