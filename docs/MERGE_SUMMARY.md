# Merge summary: cyberblob changes ported onto upstream 1.1.16

## Versions
| Role | Ref | Commit |
|---|---|---|
| Original base | upstream `rustdesk/rustdesk-server` tag `1.1.15` | `9bae9f2` |
| Your previous fork master (backed up) | `cyberblob/rustdesk-server` branch `backup/master-pre-1.1.16` | `965432a` |
| New base | upstream tag `1.1.16` | `73523b3` |
| Submodule `libs/hbb_common` | as pinned by 1.1.16 | `83419b6` |

## How changes were identified
Each file was compared across three versions: upstream 1.1.15, your modified source (`rustdesk-server - Copy`), and upstream 1.1.16. Fork commit `965432a` was used for the systemd units. A line counts as your change only if it differs from **both** upstream versions. Upstream work your fork had already pulled in is therefore not treated as yours: toolchain 1.90, the GHCR manifest steps, and the removed `punch-requests` cleanup.

## Changes applied

### `src/relay_server.rs`
- Fixed a log typo in `make_pair_`: `"Relayrequest …"` is now `"Relay request …"`.

### `src/peer.rs`
- `PeerMap` gained two reverse indexes:
  - `addr_index`: exact `SocketAddr` to peer ID.
  - `ip_index`: `IpAddr` to the peer IDs at that address.
- `update_addr_index(id, peer)` replaces a peer's index entries with its current address. Entries with port 0 are skipped.
- `get_id_by_addr(addr)` returns the exact `ip:port` match if there is one. Otherwise it returns the peer ID only when exactly one peer is registered at that IP. When several peers share the IP, it logs a warning and returns `None`.
- `update_pk` refreshes the index after every successful key registration.
- Unit tests:
  - `get_id_by_addr_exact_fallback_and_ambiguous`
  - `update_addr_index_moves_peer_to_new_address`

### `src/rendezvous_server.rs`
- `update_addr`: the peer's write lock is now released in its own block. When the address is accepted (`!request_pk`), the index is refreshed afterwards.
- `handle_punch_hole_request`: looks up the requesting peer's ID with `get_id_by_addr`. The "Fetch local addr" and "Punch hole" log lines now include that ID (`unknown` for punch hole when there is no match). Both lines were raised from `debug` to `info`.

### `.github/workflows/build.yaml`
- In the deb-package job, `debuild … -a${{ … }}` is now `-a ${{ … }}`, so the architecture is passed as a separate argument.

### `debian/rustdesk-server-hbbs.postinst`, `debian/rustdesk-server-hbbr.postinst`
- Added `setup_environment_override()`. When `/etc/default/rustdesk-hbbs` (or `-hbbr`) exists, it writes `/etc/systemd/system/<service>.d/override.conf` with `EnvironmentFile=` pointing at that file. It then runs an extra `daemon-reload`.

### `debian/rustdesk-server-hbbs.prerm`, `debian/rustdesk-server-hbbr.prerm`
- Added an `upgrade)` case that stops the service before the package is upgraded.

### `systemd/rustdesk-hbbr.service`
- Added `RUST_LOG=info`, `ENCRYPTED_ONLY=Y` and `DOWNGRADE_START_CHECK=1`.
- Changed `ExecStart=/usr/bin/hbbr -k _`.

### `systemd/rustdesk-hbbs.service`
- Added `RUST_LOG=info` and `ALWAYS_USE_RELAY=Y`.
- Changed `ExecStart=/usr/bin/hbbs -k _`.

## Not carried over
| Item | Reason |
|---|---|
| Version 1.1.17 in `Cargo.toml`/`Cargo.lock` and its `debian/changelog` entry | That was your own build number. The tree stays at 1.1.16. |
| `[hbbs]`/`[hbbr]` sections in `Cargo.toml` | Cargo ignores these sections, and they contained a hardcoded LAN relay IP. |
| `.gitignore` additions (`libs/hbb_common/`, etc.) | Ignoring the submodule path would hide `libs/hbb_common`. |
| Whitespace edits to `debian/control.tpl` and `debian/source/format`, and the garbled `#Â` comment in `build.yaml` | No functional change. |

## Notes
- The hbbs and hbbr binaries never read `ENCRYPTED_ONLY`; only the Docker s6 image's startup scripts use it. Under systemd, `-k _` is what does the work.
- `-k _` on **hbbr** turns relay key validation on, because hbbr's default key is empty. On **hbbs** it is redundant, because hbbs already defaults to `-`.
- `update_addr` runs on every peer heartbeat, and `update_addr_index` scans the whole `ip_index` each time. That is fine for small servers. For thousands of peers, tracking each peer's previous address would make the cleanup O(1).
- The punch-hole log lines are now at `info` level and include peer IDs and IP addresses.

## Verification
- `cargo check` passes on 1.1.16, both before and after the changes. No new warnings were introduced.
- `cargo test` reports 3 passed and 0 failed (2 new peer tests and the existing database test).

## Local housekeeping (D:\MyApps\rustdesk-server)
- `origin` now points to `cyberblob/rustdesk-server` and `upstream` to `rustdesk/rustdesk-server`.
- The local branch `upstream-master-snapshot` (`680b603`) keeps the previous local upstream-master tree, including `AGENTS.md`.
- The stash `pre-1.1.16 local changes` holds the earlier local `Cargo.lock` and `.gitignore` edits.
- The plain-copy `libs/hbb_common` was renamed to `libs/hbb_common.pre-1.1.16`. It is untracked and not committed.
