# openvpn3-client — OpenVPN 3 Linux client and session manager

- **Version:** OpenVPN3/Linux v24.1, OpenVPN core v3.10.4 (Debian package
  `openvpn3-client` `24.1+dfsg-1`)
- **Binaries:** `/usr/bin/openvpn3`, `/usr/bin/openvpn2`,
  `/usr/sbin/openvpn3-admin`, `/usr/sbin/openvpn3-autoload`
- **Help snapshots:** [`help.txt`](help.txt) plus one `help-<command>.txt`
  per subcommand, [`help-openvpn2.txt`](help-openvpn2.txt),
  [`help-openvpn3-admin.txt`](help-openvpn3-admin.txt)
- **Upstream:** https://github.com/OpenVPN/openvpn3-linux
- **Docs:** `man openvpn3`, `man openvpn3-<command>`,
  `/usr/share/doc/openvpn3/QUICK-START.md.gz`,
  `/usr/share/doc/openvpn3/dbus/`
- **D-Bus services:** `net.openvpn.v3.configuration`,
  `net.openvpn.v3.sessions`, `net.openvpn.v3.backends`,
  `net.openvpn.v3.netcfg`, `net.openvpn.v3.log`
- **Systemd:** `openvpn3-autoload.service`, `openvpn3-session@.service`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

OpenVPN 3 Linux is a different architecture from the classic
[`openvpn`](../openvpn/README.md) binary. It splits the VPN into D-Bus
services:

| Service | Role |
|---------|------|
| `net.openvpn.v3.configuration` | Stores and manages imported profiles |
| `net.openvpn.v3.sessions` | Creates and tracks VPN sessions |
| `net.openvpn.v3.backends` | Starts one backend process per session |
| `net.openvpn.v3.netcfg` | Applies addresses, routes, and DNS with minimal privilege |
| `net.openvpn.v3.log` | Central log service |

The `openvpn3` command is a client for those services, not a VPN process
itself. That is why an **unprivileged user can start a VPN session**: the
session manager and the netcfg service hold the privileges, and the user
only needs access to their own configuration objects.

The trade-off is that it is **not a drop-in replacement** for OpenVPN 2:
TAP/bridged setups are unsupported, some 2.x options are ignored, and the
profile must be imported into the configuration manager before it can be
started (or started in one shot with `session-start --config`).

```text
openvpn3 <command> [options]
```

`openvpn2` is a compatibility front end that accepts most 2.x client
options and translates them to the 3.x service model.

## Commands

| Command | Purpose |
|---------|---------|
| `config-import` | Import a `.ovpn`/`.conf` profile |
| `configs-list` | List imported profiles |
| `config-manage` | Show or change profile properties and overrides |
| `config-acl` | Grant/revoke access to a profile, seal it, lock it down |
| `config-dump` | Dump the profile (optionally JSON) |
| `config-remove` | Delete a profile |
| `session-start` | Import (optionally) and start a session |
| `sessions-list` | List active sessions |
| `session-manage` | Pause, resume, restart, disconnect, clean up |
| `session-auth` | Answer an ongoing authentication request |
| `session-acl` | Grant/revoke access to a session |
| `session-stats` | Show session counters (optionally JSON) |
| `log` | Follow log events for a session |
| `shell-completion` | Completion helper |
| `version` | Version information |
| `help` | Command list |

## Flag reference

### `config-import`

| Flag | Meaning |
|------|---------|
| `-c, --config <file>` | Profile to import |
| `-n, --name <name>` | Profile name (default: file name) |
| `-p, --persistent` | Survive service restarts |
| `--tag <tag>` | Attach a tag for filtering |

### `configs-list`

| Flag | Meaning |
|------|---------|
| `--count` | Only the number of profiles |
| `--json` | JSON output |
| `-v, --verbose` | More detail per profile |
| `--filter-config <prefix>` | Filter by name prefix |
| `--filter-tag <tag>` | Filter by tag |
| `--filter-owner <owner>` | Filter by owner |

### `config-manage`

| Flag | Meaning |
|------|---------|
| `-o, --path <path>` / `--config-path <path>` | Profile object path |
| `-c, --config <name>` | Profile name instead of path |
| `-r, --rename <name>` | Rename the profile |
| `--tag <tag>` / `--remove-tag <tag>` | Add/remove a tag |
| `-s, --show` | Show current settings |
| `--exists` | Exit status reports existence |
| `--quiet` | Minimal output |
| `--dco <true\|false>` | Data Channel Offload |
| `--server-override <host>` | Replace the remote server |
| `--port-override <port>` | Replace the remote port |
| `--proto-override <tcp\|udp>` | Force transport |
| `--ipv6 <yes\|no\|default>` | IPv6 policy |
| `--persist-tun <true\|false>` | Keep the tun device |
| `--log-level <1-6>` | Override `--verb` |
| `--dns-fallback-google <true\|false>` | Google DNS fallback |
| `--dns-setup-disabled <true\|false>` | Do not touch DNS |
| `--dns-scope <global\|tunnel>` | Split-DNS scope |
| `--dns-sync-lookup <true\|false>` | Synchronous DNS lookup |
| `--auth-fail-retry <true\|false>` | Retry on auth failure |
| `--allow-compression <no\|asym\|yes>` | Compression policy |
| `--enable-legacy-algorithms <true\|false>` | Allow legacy crypto |
| `--tls-version-min <tls_1_0…tls_1_3>` | Minimum TLS |
| `--tls-cert-profile <legacy\|preferred\|suiteb>` | Certificate policy |
| `--proxy-host`, `--proxy-port`, `--proxy-username`, `--proxy-password`, `--proxy-auth-cleartext` | HTTP proxy overrides |
| `--enterprise-profile <value>` | Device-posture profile |
| `--unset-override <name>` | Remove an override |

### `config-acl`

| Flag | Meaning |
|------|---------|
| `-s, --show` | Show the ACL |
| `-G, --grant <uid\|user>` | Grant access |
| `-R, --revoke <uid\|user>` | Revoke access |
| `--public-access <true\|false>` | Make the profile public |
| `-T, --transfer-owner-session <true\|false>` | Move ownership to the session |
| `--lock-down <true\|false>` | Disable profile retrieval |
| `-S, --seal` | Permanently read-only |

### `config-dump` / `config-remove`

| Flag | Meaning |
|------|---------|
| `-o, --path` / `--config-path`, `-c, --config` | Select the profile |
| `-j, --json` | JSON dump (`config-dump`) |
| `--force` | Skip confirmation (`config-remove`) |

### `session-start`

| Flag | Meaning |
|------|---------|
| `-c, --config <file>` | Import and start in one shot |
| `-p, --config-path <path>` | Start an imported profile |
| `--persist-tun` | Persistent tun (requires `--config`) |
| `--timeout <secs>` | Connection timeout (default infinite) |
| `--background` | Background after basic credentials |
| `--dco <bool>` | Start with DCO |

### `session-manage`

| Flag | Meaning |
|------|---------|
| `-o, --path` / `--session-path`, `-c, --config`, `-I, --interface` | Select the session |
| `--log-level[=level]` | View/set the session log level |
| `--timeout <secs>` | Resume/restart timeout |
| `-P, --pause` | Pause |
| `-R, --resume` | Resume |
| `--restart` | Disconnect and reconnect |
| `-D, --disconnect` | Disconnect |
| `--cleanup` | Remove stale sessions |

### `session-acl`

| Flag | Meaning |
|------|---------|
| `-o, --path` / `--session-path`, `-c, --config`, `-I, --interface` | Select the session |
| `-s, --show` | Show the ACL |
| `-G, --grant <uid\|user>` | Grant access |
| `-R, --revoke <uid\|user>` | Revoke access |
| `--public-access <true\|false>` | Public session |
| `--allow-log-access <true\|false>` | Let granted users read the log |

### `session-stats`, `session-auth`, `log`, `sessions-list`

| Flag | Meaning |
|------|---------|
| `-o, --path` / `--session-path`, `-c, --config`, `-I, --interface` | Select the session |
| `-j, --json` | JSON output (`session-stats`) |
| `--auth-req <id>` | Continue a specific auth request (`session-auth`) |
| `--log-level <n>` | Log verbosity (default 4) (`log`) |
| `-h, --help` | Help (`sessions-list` has only this) |

## Recipes

**1. Import a profile and note the returned path.**

```bash
openvpn3 config-import --config client.ovpn --name lab-client
```

**2. Import persistently with a tag.**

```bash
openvpn3 config-import --config client.ovpn --persistent --tag engagement-a
```

**3. List profiles, filtered and verbose.**

```bash
openvpn3 configs-list --verbose --filter-tag engagement-a
```

**4. Start a session in one shot.**

```bash
openvpn3 session-start --config client.ovpn
```

**5. Start an already-imported profile by name.**

```bash
openvpn3 session-start --config lab-client
```

**6. Start by object path.**

```bash
openvpn3 session-start --config-path /net/openvpn/v3/configuration/<id>
```

**7. List active sessions.**

```bash
openvpn3 sessions-list
```

**8. Show session statistics as JSON.**

```bash
openvpn3 session-stats --config lab-client --json
```

**9. Follow a session log.**

```bash
openvpn3 log --config lab-client --log-level 6
```

**10. Pause and resume a session.**

```bash
openvpn3 session-manage --config lab-client --pause
openvpn3 session-manage --config lab-client --resume
```

**11. Restart or disconnect.**

```bash
openvpn3 session-manage --config lab-client --restart
openvpn3 session-manage --config lab-client --disconnect
```

**12. Clean up stale sessions.**

```bash
openvpn3 session-manage --cleanup
```

**13. Override the server without re-importing.**

```bash
openvpn3 config-manage --config lab-client --server-override vpn2.example.com
openvpn3 config-manage --config lab-client --proto-override tcp
```

**14. Disable DNS changes for a profile.**

```bash
openvpn3 config-manage --config lab-client --dns-setup-disabled true
```

**15. Show the effective profile.**

```bash
openvpn3 config-manage --config lab-client --show
```

**16. Dump the profile as JSON for review.**

```bash
openvpn3 config-dump --config lab-client --json
```

**17. Share a profile with another user.**

```bash
sudo openvpn3 config-acl --config lab-client --grant alice
```

**18. Seal a profile so it cannot be changed.**

```bash
openvpn3 config-acl --config lab-client --seal
```

**19. Remove a profile.**

```bash
openvpn3 config-remove --config lab-client --force
```

**20. Use the OpenVPN 2 compatibility front end.**

```bash
openvpn2 --config client.ovpn
```

**21. Check the service version and variables.**

```bash
openvpn3 version
sudo openvpn3-admin variables
```

**22. Follow the system journal for the services.**

```bash
sudo openvpn3-admin journal --since 10m
```

**23. Autoload a profile at boot.**

```bash
sudo install -d /etc/openvpn3/autoload
sudo cp client.ovpn /etc/openvpn3/autoload/
sudo systemctl restart openvpn3-autoload.service
```

## Gotchas

- **`openvpn3` is a D-Bus client, not a VPN daemon.** If the services are
  not running, every command fails with a service error, not a network
  error.
- **TAP and bridged profiles do not work.** OpenVPN 3 supports routed TUN
  only; a `.ovpn` that needs `dev tap` will import but not connect.
- **Not every 2.x option is honoured.** The core library is the same one
  used by OpenVPN Connect; unsupported directives are ignored. Check
  `config-dump` and the session log rather than assuming.
- **`session-start --config` imports a copy.** Later edits to the source
  file do not affect the imported profile.
- **`--persist-tun` requires `--config`, not `--config-path`.** The help
  states the restriction explicitly.
- **`--server-override` changes where the profile connects.** That is
  useful for lab redirects and dangerous if you forget it is set; clear it
  with `--unset-override server_override` (or the matching name).
- **`--seal` is permanent.** A sealed profile cannot be modified or
  unsealed; import a fresh copy if you need changes.
- **`--lock-down` and `--seal` are different.** Lock-down disables profile
  retrieval for users; seal makes the object read-only.
- **A session can outlive the terminal.** It is owned by the session
  manager, so closing the shell does not disconnect it; use
  `session-manage --disconnect`.
- **`session-manage --cleanup` is not a disconnect.** It removes stale
  session objects, not live tunnels.
- **Logs are per-session.** `openvpn3 log` needs a session selector; the
  service-wide journal is under `openvpn3-admin journal`.
- **The netcfg service changes DNS.** If split DNS matters, set
  `--dns-scope tunnel` or `--dns-setup-disabled true` deliberately.
- **`openvpn2` is a translator.** It accepts 2.x-style options but still
  creates 3.x configuration and session objects; do not expect 2.x
  `--up`/`--down` semantics.
- **`openvpn3-autoload` needs `/etc/openvpn3/autoload`.** The directory is
  not created by the package on this install.
- **Unprivileged users can start sessions but not manage the services.**
  `openvpn3-admin` commands that touch netcfg or the session manager need
  root.
- **D-Bus policy is the security boundary.** Granting a user access to a
  profile or session is a real authorisation change, not a convenience.

## Related docs

- [`openvpn`](../openvpn/README.md) — the classic 2.x client/server
- [`proxychains4`](../proxychains4/README.md) — proxy-chained TCP
- [`socat`](../socat/README.md) — relay and tunnel primitive
- [`sshuttle`](../sshuttle/README.md) — VPN-like access over SSH
- [`tor`](../tor/README.md) — anonymising overlay
