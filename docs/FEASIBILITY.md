# FIPS transit feasibility (nvpn v4.1.16)

## Decision: do not ship a Cloudron package yet

The proposed least-privilege transit daemon **does not start** without a TUN
device and `CAP_NET_ADMIN`. Disabling `fips_host_tunnel_enabled` is insufficient:
the Linux FIPS runtime unconditionally checks `/dev/net/tun` and constructs a
`SystemTun` at startup, before it can serve the WebSocket listener. The Cloudron
manifest reference lists `net_admin` as a capability, but does not document a
way for an app manifest to mount `/dev/net/tun`. A manifest with `net_admin`
alone is not evidence that Cloudron can run this daemon. Do not publish a
package or claim transit works until a supported TUN device path and a real
two-client packet-path test are demonstrated.

## Reproduction (2026-09-28)

Upstream checkout: `mmalmi/nostr-vpn` tag `v4.1.16`, commit
`076e5d1c6fe7b0a5fe3905569f5ea37953da089e`. From that checkout:

```sh
docker build --target rust-builder -f umbrel/Dockerfile -t nvpn-feasibility:4.1.16 .
docker run --rm --cap-drop ALL --read-only --tmpfs /tmp --tmpfs /run \
  nvpn-feasibility:4.1.16 bash -lc '
    /out/nvpn init --config /tmp/nvpn.toml &&
    /out/nvpn set --config /tmp/nvpn.toml \
      --autoconnect false --lan-discovery-enabled false \
      --fips-nostr-discovery-enabled false \
      --fips-host-tunnel-enabled false --fips-bootstrap-enabled false &&
    timeout 12 /out/nvpn daemon --config /tmp/nvpn.toml \
      --fips-websocket-bind 0.0.0.0:8080 \
      --fips-websocket-public-url wss://example.invalid/fips'
```

Observed: `nvpn init` and `nvpn set` succeed. The daemon warns that the UDP
receive buffer is clamped by the kernel, then exits before serving WebSocket:

```text
Error: Linux tunnel setup requires CAP_NET_ADMIN and /dev/net/tun before FIPS can create utun100: missing /dev/net/tun device. For a foreground session run `sudo nvpn start --connect` or `sudo nvpn connect`; for unattended use install/start the system service. In Docker add `--cap-add NET_ADMIN --device /dev/net/tun`.
```

The same failure occurs with `autoconnect` left enabled. Disabling it does not
bypass TUN creation. Source inspection at this exact tag: in
`crates/nostr-vpn-cli/src/fips_private_mesh/tunnel_runtime_unix_core.rs`,
`start_with_mesh` calls `ensure_linux_tun_permissions(&config.iface)` and then
`SystemTun::new(&config.iface)` unconditionally (lines 55 and 71-75). In
`crates/nostr-vpn-cli/src/fips_private_mesh/unix_tun.rs`, the guard checks
`/dev/net/tun`. The upstream Umbrel daemon explicitly uses both `NET_ADMIN`
and `/dev/net/tun` (`umbrel/docker-compose.yml`).

Granting the capability by itself did not supply the device in the local
Docker runtime: `docker run --rm --cap-add NET_ADMIN nvpn-feasibility:4.1.16
ls -l /dev/net/tun` returned `No such file or directory`. This is a local
Docker observation, **not** a Cloudron runtime test. The remaining device
mount and Cloudron acceptability questions are unresolved.

The tested binary's `nvpn --help` also hides the internal `daemon` command;
`nvpn daemon --help` confirms it remains callable. The actual client-facing
command is `nvpn start`. This difference matters when adapting the Umbrel
entrypoint.

## What remains to qualify

1. Establish with a supported Cloudron manifest/runtime contract that
   `/dev/net/tun` is available to the app with `net_admin` and that Community
   Apps accepts the privilege model. Do not rely on a host-side, untracked
   Docker modification.
2. If unsupported, obtain an upstream control-only FIPS runtime that does not
   allocate a TUN; repeat the no-capabilities/no-device test. Bypassing just
   the permission check is not a solution because `SystemTun::new` follows it.
3. Run two clients on separate networks, with only this node in
   `[fips_bootstrap_peers]`, Nostr and LAN discovery disabled, and prove packet
   exchange through it. Then check node npub survives restart/backup-restore.

No manifest, image, catalog, or release was produced from this negative test.
In particular, untested `capabilities: ["net_admin"]` would not satisfy the
packet-path acceptance criterion.
