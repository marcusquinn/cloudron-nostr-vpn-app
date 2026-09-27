# t5: Qualify Nostr VPN transit package on Cloudron and submit to Community Apps

## Origin

- **Created:** 2026-09-27
- **Session:** opencode:ses_f1f59ba62ffeapMbfcXvy05cN5
- **Created by:** ai-interactive (Marcus Quinn session)
- **Blocked by:** t4 (package implementation PR merged)
- **Conversation context:** Operator half of the Nostr VPN transit package lifecycle. These steps need the maintainer's GitHub secret, Cloudron server, real nvpn clients and Cloudron Community Apps account, so they are not worker-dispatchable.

## What

The transit package from t4 is published, qualified on a live Cloudron with real clients, and listed in Cloudron Community Apps.

## Why

A live off-LAN test is the only proof that the self-hosted node replaces the iris.to bootstrap peers, and submission needs maintainer accounts.

## How (operator checklist)

1. Create a fine-grained PAT limited to this repository with only `Contents: Read and write`; save it as the repository secret `CLOUDRON_RELEASE_PAT` (see `docs/PUBLISHING.md`).
2. Merge the t4 PR and confirm the catalog workflow published the image, `CloudronVersions.json`, tag and GitHub release.
3. Install: `cloudron install --versions-url <PUBLIC_VERSIONS_URL> --location nvpn-test`; note the node npub from the logs or docs location.
4. Qualify with the MacBook and Mac mini: back up each `~/Library/Application Support/nvpn/config.toml`, replace `[fips_bootstrap_peers]` with only this node (per `docs/CLIENT-CONFIG.md`), disable Nostr discovery, put one Mac on a different network (e.g. phone hotspot), block any remaining direct client path, record route or session evidence that this node carries the traffic, and confirm `ssh mini.nvpn` still works. Restore the configs afterwards unless keeping the new node as the default.
5. Check restart, update and backup/restore keep the same npub.
6. Sign in at [Cloudron Community Apps](https://ca.cloudron.io), add the versions URL, and verify the imported listing.
7. In local `~/.config/aidevops/repos.json`, set `cloudron_package.monitor_upstream` and `monitor_compatibility` to `true` for this repo (disabled until a manifest exists).
8. Record evidence in the issue; then file aidevops follow-ups to (a) let `nostr-vpn-setup-lib.sh` set custom bootstrap peers and (b) document the node in `.agents/services/networking/nostr-vpn.md` and `.agents/reference/mesh-remote-workers.md`.

## Acceptance Criteria

- [ ] Two real nvpn clients on different networks connect using only this node as bootstrap peer, with recorded route or session evidence that traffic transits this node.
- [ ] Node identity survives restart, update and backup/restore.
- [ ] The app appears in Cloudron Community Apps with correct metadata.
- [ ] Client configs are restored or intentionally switched, with backups kept.

## Dependencies

- **Blocked by:** t4
- **External:** GitHub PAT, Cloudron server admin access, two nvpn clients, Cloudron Community Apps account.
