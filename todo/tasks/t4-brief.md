<!-- aidevops:brief-schema=v2 -->

# t4: Package Nostr VPN FIPS transit node as a Cloudron app through Community Apps submission

## Pre-flight (auto-populated by briefing workflow)

- [x] Memory recall: `aidevops init new repo cloudron app community directory publishing` → 0 hits — no relevant lessons (session memories on nvpn ports/aliases are client-side only)
- [x] Discovery pass: 0 commits / 0 merged PRs / 0 open PRs touch target files (new repository; `gh search repos "cloudron nostr"` returned no existing package)
- [x] File refs verified: 8 refs checked in `mmalmi/nostr-vpn`, `marcusquinn/cloudron-netbird-app` and `marcusquinn/aidevops` at current default branches
- [x] Tier: `tier:thinking` — whether a transit-only node runs without TUN/`net_admin` is unresolved and decides the package architecture
- [x] Seeded draft PR decision recorded: skipped — the feasibility result determines the package shape

## Origin

- **Created:** 2026-09-27
- **Session:** opencode:ses_f1f59ba62ffeapMbfcXvy05cN5
- **Created by:** ai-interactive (Marcus Quinn session)
- **Parent task:** none
- **Blocked by:** none
- **Conversation context:** Two Macs now mesh over Nostr VPN (nvpn 4.1.16). Off-LAN connectivity still depends on the third-party FIPS bootstrap/transit peers `fips1.iris.to` and `fips2.iris.to`. The maintainer wants a self-hosted transit node, packaged for Cloudron like `cloudron-netbird-app`.

## What

A publishable Cloudron app package that runs `nvpn` as a FIPS transit and bootstrap peer we control:

- Stable node identity (Nostr keypair) generated on first run under `/app/data` and shown to the operator (logs + docs) as the npub to put in client `[fips_bootstrap_peers]`.
- FIPS WebSocket transport exposed at `wss://<app domain>/fips` through Cloudron TLS, using nvpn's `fips_websocket_bind_addr` (plain WS on the container `httpPort`) and `fips_websocket_public_url`.
- Optional direct UDP transport via a manifest `udpPorts` entry (default 51820; NetBird clients use 51820 on their own hosts, not on the Cloudron box, so verify no Cloudron-side clash).
- Operator documentation for the exact client config stanza, e.g. `[fips_bootstrap_peers]` `"npub1…" = ["<app domain>:51820", "websocket:wss://<app domain>/fips"]`.
- The same managed release pipeline shape as `cloudron-netbird-app`.

## Why

Mesh remote workers over nvpn work direct-only on the LAN, but off-LAN they need bootstrap/transit peers. The defaults in upstream `crates/nostr-vpn-core/src/config/types.rs` (`DEFAULT_FIPS_BOOTSTRAP_PEERS`) are two iris.to hosts we do not control. `DEFAULT_RELAYS` is empty, so the transit peer, not a Nostr relay, is the remaining third-party dependency. A self-hosted transit node removes it.

## Tier

### Tier checklist (verify before assigning)

- [ ] **Exact execution contract supplied?** No.
- [x] **Targets and reference pattern verified?** Yes — NetBird package and upstream `umbrel/` packaging.
- [ ] **No semantic or design decision remains?** No — TUN/capability requirement is unknown.
- [x] **Bounded, reversible, low-consequence impact?** New repository.
- [ ] **No stateful coordination to invent?** Transit node identity and peering behaviour must be proven.
- [x] **Focused verification and rollback are explicit?** Yes.
- [x] **No dispatch-path risk override?** Not an aidevops dispatch-path file.

**Selected tier:** `tier:thinking`

**Tier rationale:** The package architecture depends on an unresolved runtime question (can nvpn transit without a TUN device and `net_admin`), which must be answered with evidence before building.

## PR Conventions

Leaf task: the implementation PR uses `Resolves #<this issue>`.

## How (Approach)

### Progressive Context Plan

- **Read first:** `mmalmi/nostr-vpn` `umbrel/Dockerfile`, `umbrel/docker-compose.yml`, `umbrel/README.md` at tag `v4.1.16` — how upstream builds `nvpn`/`nvpn-web` and runs the daemon (`network_mode: host`, `cap_add: NET_ADMIN`, `/dev/net/tun`, `nvpn daemon --config /data/config/nvpn/config.toml`).
- **Read first:** `crates/nostr-vpn-core/src/config/types.rs` — fields `fips_host_tunnel_enabled`, `fips_websocket_bind_addr`, `fips_websocket_public_url`, `fips_bootstrap_enabled`, `fips_bootstrap_peers`, `fips_nostr_discovery_enabled`, `lan_discovery_enabled`; and `config/types/nostr.rs` `NostrPubsubMode` (`off|client|relay`).
- **Load only if:** `marcusquinn/cloudron-netbird-app` `CloudronManifest.json`, `docs/PUBLISHING.md`, `.github/workflows/cloudron-catalog-publish.yml`, `scripts/publish-cloudron-catalog.sh` — when building the package and pipeline.
- **Load only if:** `~/.aidevops/agents/tools/deployment/cloudron-app-packaging-skill/manifest-ref/05-behavior.md` (`capabilities`: `net_admin`, `mlock`, `ping`, `vaapi`) and `02-ports.md` (`udpPorts`) — when the feasibility result requires capabilities or ports.
- **Why:** decide the smallest privilege set, then mirror a proven pipeline.
- **Stop when:** the feasibility decision is recorded with evidence and the manifest shape follows from it.

### Worker Quick-Start

```bash
gh api 'repos/mmalmi/nostr-vpn/contents/umbrel/Dockerfile?ref=v4.1.16' --jq .content | base64 -d
gh api 'repos/mmalmi/nostr-vpn/contents/crates/nostr-vpn-core/src/config/types.rs?ref=v4.1.16' --jq .content | base64 -d | rg -n 'fips_|DEFAULT_'
# Transit/relay-related code paths
gh api 'search/code?q=repo:mmalmi/nostr-vpn+fips_host_tunnel_enabled' --jq '.items[].path'
```

Critical facts:

- Upstream latest stable is `v4.1.16` (2026-09-25). Pin the build to that tag.
- The upstream Umbrel app runs the full daemon with host networking, `NET_ADMIN` and `/dev/net/tun`.
- The client CLI has no flag for bootstrap peers; clients edit `[fips_bootstrap_peers]` in `config.toml` directly.

### Files to Modify

- `NEW: docs/FEASIBILITY.md` — Phase 0 evidence: can `nvpn daemon` run as a transit/bootstrap peer with `fips_host_tunnel_enabled = false` in an unprivileged container (no `/dev/net/tun`, no `NET_ADMIN`)? Record commands, logs, and the resulting decision.
- `NEW: CloudronManifest.json` — id `com.marcusquinn.cloudron.nostrvpn`, `httpPort` for the FIPS WebSocket (and optional web UI), `udpPorts.FIPS_UDP_PORT` (default 51820), addon `localstorage`; add `capabilities: ["net_admin"]` only if Phase 0 proves it necessary.
- `NEW: Dockerfile` — multi-stage build of `nvpn` from the pinned upstream tag (model on `umbrel/Dockerfile`), final stage on the digest-pinned Cloudron base image.
- `NEW: start.sh` — generate `/app/data/config.toml` on first run; set transit-node settings every start (WebSocket bind/public URL from `CLOUDRON_APP_DOMAIN`, UDP listen from `FIPS_UDP_PORT`, LAN discovery off, bootstrap to iris.to off by default); print the node npub; run as `cloudron`.
- `NEW: CHANGELOG`, `NEW: CHANGELOG.md`, `NEW: logo.png`, `NEW: media/hero.png` — as in the NetBird package; artwork licence-checked or original, provenance in `DESIGN.md`.
- `NEW: docs/README.md`, `NEW: docs/PACKAGING-NOTES.md`, `NEW: docs/PUBLISHING.md`, `NEW: docs/CLIENT-CONFIG.md` — client `[fips_bootstrap_peers]` stanza and direct-only/off-LAN usage.
- `NEW: scripts/publish-cloudron-catalog.sh`, `NEW: .github/workflows/cloudron-catalog-publish.yml`, `NEW: .github/workflows/cloudron-package-release.yml`, `NEW: .github/workflows/linked-issue-check.yml`, `NEW: .github/dependabot.yml` — adapt from NetBird; release caller from the aidevops template.
- `NEW: test/package-test.sh`, `NEW: test/transit-e2e.sh` — static checks, plus a docker-compose check where two nvpn clients with only this node as bootstrap peer (Nostr discovery and LAN discovery off, separate docker networks) establish a mesh path through it.
- `NEW: SECURITY.md`, `NEW: CONTRIBUTING.md`, `NEW: .dockerignore`, `NEW: .editorconfig`.
- `EDIT: README.md`, `EDIT: AGENTS.md`, `EDIT: DESIGN.md`.

### Complete Write Surface

- **Callers/readers:** nvpn clients dial the WebSocket and UDP endpoints; Cloudron reads the manifest and `CloudronVersions.json`; aidevops `cloudron-package-monitor-helper.sh` reads the manifest.
- **Writers/mutation paths:** `start.sh` writes `/app/data/config.toml` and identity; nvpn writes state under `/app/data`; only the catalog workflow writes `CloudronVersions.json` and tags.
- **Existing verification/tests:** none here yet; upstream has backend Docker end-to-end tests (see `umbrel/README.md` "Mesh packet-path behavior is covered by the backend Docker end-to-end tests") to model `test/transit-e2e.sh` on.
- **Schemas/config:** nvpn `config.toml` schema from `config/types.rs`; `CloudronManifest.json`.
- **Generated/deployed mirrors:** GHCR image `ghcr.io/marcusquinn/cloudron-nostr-vpn-app` and `CloudronVersions.json` produced by `.github/workflows/cloudron-catalog-publish.yml` only; never hand-written.
- **Migrations/backfills:** nvpn migrates its own config (e.g. `is_legacy_fips_bootstrap`); `start.sh` must not rewrite operator keys it does not own.
- **Cleanup/rollback paths:** Cloudron backup/restore of `/app/data` keeps the node identity; losing it changes the npub and breaks every client stanza, so docs must say so.

### Implementation Steps

1. Phase 0: build `nvpn` from `v4.1.16` in a plain container without `/dev/net/tun` or `NET_ADMIN`, run `nvpn daemon` with host tunnel disabled, and test transit between two client containers. Record the outcome in `docs/FEASIBILITY.md`.
2. Decide the privilege model: unprivileged if Phase 0 passes; otherwise `capabilities: ["net_admin"]` plus the TUN device, with the evidence and Community Apps acceptability noted. If transit cannot work in Cloudron at all, stop and report with evidence rather than shipping a partial package.
3. Write manifest, Dockerfile, `start.sh`, docs and pipeline following the NetBird pattern.
4. Run the verification block and open the PR with Phase 0 and e2e evidence.

### Hazards and Compatibility

- **Concurrency/atomicity:** single daemon; config written via temp file + `mv`.
- **Migration/rollback:** identity must persist across updates; never regenerate keys when present.
- **Mixed-version/backward compatibility:** clients and node may run different nvpn versions; document the tested client version range (at least 4.1.15–4.1.16).
- **Idempotency/retry:** `start.sh` safe on every restart; catalog workflow already idempotent.
- **Partial failure/recovery:** if the UDP port is disabled by the operator, keep WebSocket transit; if config generation fails, exit non-zero.

### Verification Before Dispatch

```bash
cloudron-package-helper.sh validate
cloudron-package-helper.sh check-compatibility
bash test/package-test.sh
bash test/transit-e2e.sh
shellcheck start.sh scripts/*.sh test/*.sh
```

- **Surface mapping:** `validate`/`check-compatibility` prove manifest and pinned base; `package-test.sh` proves static invariants; `transit-e2e.sh` proves the transit role, identity persistence and WebSocket/UDP endpoints; the pull-request run of `cloudron-catalog-publish.yml` proves the pipeline validates without publishing.
- **Broad verification trigger:** Not required — new standalone repository.

### Recoverability Checkpoint

- [ ] Focused functional verification passes: `bash test/transit-e2e.sh`
- [ ] WIP commit created before broad gates: `wip: nvpn transit feasibility and package`
- [ ] Evidence-triggered broad verification then run: not required — new repository

### Safety-Stop Recovery

- **Original objective:** a Cloudron-installable nvpn transit/bootstrap node that removes the iris.to dependency.
- **Preserved user directions:** self-hosted, peer-to-peer, private; no centralised third-party hosts.
- **Trigger and evidence:** not triggered.
- **Completed and verified:** none yet.
- **Remaining acceptance criteria:** all.
- **Unsafe route not to repeat:** none.
- **Next safe route:** if Cloudron cannot host transit, report evidence and propose a non-Cloudron container deployment instead.
- **Resume condition:** Phase 0 evidence recorded.
- **Owner and status:** worker; not-triggered.

### Scope Boundaries

**Hard boundaries:** do not hand-write `CloudronVersions.json`; do not commit keys or secrets; do not modify the aidevops repository (client-side helper support for custom bootstrap peers is a separate aidevops follow-up).

**AI brief owner:** marcusquinn interactive session (this brief).

**Recovery:** preserve the current PR and use the structured runtime request and Pulse intake in `reference/worker-discipline.md` when local recovery is unsafe.

### Files Scope

- `docs/FEASIBILITY.md`
- `CloudronManifest.json`
- `Dockerfile`
- `.dockerignore`
- `.editorconfig`
- `start.sh`
- `CHANGELOG`
- `CHANGELOG.md`
- `README.md`
- `AGENTS.md`
- `DESIGN.md`
- `SECURITY.md`
- `CONTRIBUTING.md`
- `logo.png`
- `media/hero.png`
- `docs/README.md`
- `docs/PACKAGING-NOTES.md`
- `docs/PUBLISHING.md`
- `docs/CLIENT-CONFIG.md`
- `scripts/publish-cloudron-catalog.sh`
- `test/package-test.sh`
- `test/transit-e2e.sh`
- `.github/workflows/cloudron-catalog-publish.yml`
- `.github/workflows/cloudron-package-release.yml`
- `.github/workflows/linked-issue-check.yml`
- `.github/dependabot.yml`
- `TODO.md`
- `todo/tasks/t4-brief.md`

## Acceptance Criteria

- [ ] `docs/FEASIBILITY.md` records whether transit works without TUN/`net_admin`, with commands and logs, and the manifest privilege set matches that result.

  ```yaml
  verify:
    method: codebase
    pattern: "net_admin|NET_ADMIN|/dev/net/tun"
    path: "docs/FEASIBILITY.md"
  ```

- [ ] Two nvpn clients configured with only this node as bootstrap peer (Nostr and LAN discovery off, separate networks) exchange traffic through it.

  ```yaml
  verify:
    method: bash
    run: "bash test/transit-e2e.sh"
  ```

- [ ] The package does not default to the third-party iris.to bootstrap peers.

  ```yaml
  verify:
    method: codebase
    pattern: "iris\\.to"
    path: "start.sh"
    expect: absent
  ```

- [ ] Node identity persists across container restart (same npub before and after), shown in the PR evidence.
- [ ] `cloudron-package-helper.sh validate` and the pull-request run of `cloudron-catalog-publish.yml` pass.
- [ ] `docs/PUBLISHING.md` lists the operator hand-off steps: `CLOUDRON_RELEASE_PAT` secret, live install via `cloudron install --versions-url <PUBLIC_VERSIONS_URL> --location nvpn-test`, upgrade/restart/backup-restore checks, and Community Apps submission at ca.cloudron.io.
- [ ] Changed-file lint is clean (`shellcheck`, markdownlint where configured).

## Context & Decisions

- The Cloudron app is a transit/bootstrap peer, not a Nostr relay; a private relay is packaged separately in `cloudron-nostr-relay-app`.
- Use the upstream `mmalmi/nostr-vpn` build, not jmcorgan's standalone FIPS; their transit compatibility is unverified.
- Prefer the WebSocket path through Cloudron TLS (works on restrictive networks); UDP is optional for performance.
- Least privilege first: only add `net_admin`/TUN when Phase 0 proves it is required.
- The optional `nvpn-web` control panel is a non-goal for 0.1.0; if included later, protect it with the Cloudron `proxyauth` addon.
- Operator-only steps (secret, live Cloudron qualification, Community Apps submission) are tracked in the follow-up operator task.

## Relevant Files

- `mmalmi/nostr-vpn:umbrel/docker-compose.yml` — daemon invocation and privileges.
- `mmalmi/nostr-vpn:umbrel/Dockerfile` — multi-stage build to adapt.
- `mmalmi/nostr-vpn:crates/nostr-vpn-core/src/config/types.rs` — transit-relevant config fields and default bootstrap peers.
- `marcusquinn/cloudron-netbird-app:CloudronManifest.json` — manifest and `udpPorts` pattern.
- `marcusquinn/cloudron-netbird-app:docs/PUBLISHING.md` — publication flow.
- `marcusquinn/aidevops:.agents/services/networking/nostr-vpn.md` — client-side direct-only mode and bootstrap peer notes.

## Dependencies

- **Blocked by:** none
- **Blocks:** operator qualification and Community Apps submission task (t5); an aidevops follow-up to let `nostr-vpn-setup-lib.sh` set custom bootstrap peers.
- **External:** Docker on the worker host for Phase 0 and e2e; `CLOUDRON_RELEASE_PAT` provided by the operator for catalog publication.

## Estimate Breakdown

| Phase | Time | Notes |
|-------|------|-------|
| Research/read | 1h | upstream umbrel packaging and config types |
| Implementation | 5h | Phase 0, package, e2e, pipeline, docs |
| Verification | 1h | e2e evidence, CI |
| **Total** | **~7h** | |
