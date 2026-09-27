# cloudron-nostr-vpn-app

<!-- AI-CONTEXT-START -->

## Quick Reference

- **Validate**: `cloudron-package-helper.sh validate`
- **Build**: `cloudron-package-helper.sh build`
- **Test install**: `cloudron-package-helper.sh install nvpn-test`
- **Release checks**: `cloudron-package-helper.sh check-compatibility` and
  `cloudron-package-helper.sh preflight-release vX.Y.Z`

## Project Overview

Cloudron app package that runs [Nostr VPN](https://github.com/mmalmi/nostr-vpn)
(`nvpn`) as a self-hosted FIPS transit and bootstrap peer. Nostr VPN nodes can
list it in `[fips_bootstrap_peers]` instead of the default third-party
`fips1.iris.to`/`fips2.iris.to` peers, so off-LAN mesh connections stay on
infrastructure you control.

## Architecture

Mirror the structure of `marcusquinn/cloudron-netbird-app`: `Dockerfile` pinned to
the Cloudron base image, `start.sh`, `CloudronManifest.json`, `docs/` for
operator guides, and the Cloudron release and catalog-publish workflows. The
upstream Umbrel package (`umbrel/` in `mmalmi/nostr-vpn`) is the reference for
the daemon build and its host-networking and TUN requirements.

## Conventions

- Commits: [Conventional Commits](https://www.conventionalcommits.org/)
- Branches: `feature/`, `bugfix/`, `hotfix/`, `refactor/`, `chore/`
- Documentation: human/operator guides in `docs/`, AI-only context in `.agents/`;
  retain conventional root entrypoints and respect existing repository conventions.

## Key Files

| File | Purpose |
|------|---------|
| `.agents/AGENTS.md` | Project-specific agent instructions |
| `docs/` | Human/operator guides, linked from root entrypoints |
| `TODO.md` | Task tracking |
| `CHANGELOG.md` | Version history |

<!-- AI-CONTEXT-END -->
