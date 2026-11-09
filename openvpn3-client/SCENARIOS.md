# openvpn3-client — command scenarios for deep research

Installed build: OpenVPN3/Linux v24.1, core v3.10.4 (Debian
`openvpn3-client` `24.1+dfsg-1`). Every command and flag below appears in
[`help.txt`](help.txt) or one of the `help-<command>.txt` snapshots. The
`openvpn2` front end has its own snapshot in
[`help-openvpn2.txt`](help-openvpn2.txt).

> **Scope:** Importing a profile and starting a session are authorisation
> changes on a shared D-Bus system. Use only profiles and servers you are
> authorised to use, and remove imported profiles and sessions when the
> engagement ends.

## How to invoke

```text
@deep-research For an authorised assessment on a Linux host running
OpenVPN 3 Linux 24.1, explain how the D-Bus configuration, session, backend,
netcfg, and log services divide responsibility, what an unprivileged user
can do without sudo, and how a profile's DNS and route changes are applied
by netcfg. Compare the 3.x session model with classic OpenVPN 2.x, and cite
the openvpn3-linux source, the D-Bus service documentation shipped in
/usr/share/doc/openvpn3/dbus/, and the OpenVPN 3 man pages. Separate
confirmed behaviour from inference and state confidence.
```

Capture the baseline before importing anything:

```bash
mkdir -p ~/ovpn3-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/ovpn3-demo/evidence/run-start.txt
openvpn3 configs-list --json > ~/ovpn3-demo/evidence/configs-before.json
openvpn3 sessions-list > ~/ovpn3-demo/evidence/sessions-before.txt
ip route > ~/ovpn3-demo/evidence/route-before.txt
cat /etc/resolv.conf > ~/ovpn3-demo/evidence/resolv-before.txt
```

## Scenario 1 — service inventory and health

`openvpn3` fails at the D-Bus layer long before it reaches the network.
Confirm the services and the version first.

| # | Command | How to use |
|---|---------|------------|
| 1 | `openvpn3 version` | Client and core library versions |
| 2 | `openvpn3 help` | Command list |
| 3 | `openvpn3 shell-completion --list-commands` | Machine-readable command list |
| 4 | `openvpn3 shell-completion --list-options session-start` | Options for one command |
| 5 | `openvpn3 configs-list` | Configuration service reachable? |
| 6 | `openvpn3 sessions-list` | Session service reachable? |
| 7 | `systemctl status openvpn3-autoload.service` | Autoload unit state |
| 8 | `systemctl list-units 'openvpn3*' --all` | All OpenVPN 3 units |
| 9 | `busctl list \| grep openvpn` | D-Bus services on the bus |
| 10 | `sudo openvpn3-admin variables` | Build and runtime variables |
| 11 | `sudo openvpn3-admin journal --since 10m` | Service journal |
| 12 | `openvpn3 --help` | Top-level help |

**Research questions**
- Which service owns profile storage, and which owns the tunnel process?
- What does an unprivileged user see that root sees differently?
- Which D-Bus name is missing when `configs-list` fails?
- Does `openvpn3-autoload` need the directory to exist before it starts?
- Which version mismatch between client and service would break the API?

**Sources:** `openvpn3 --help`; D-Bus service docs in
`/usr/share/doc/openvpn3/dbus/`; systemd unit files.
**Output:** a service-health record with the D-Bus names observed.

## Scenario 2 — import and inspect profiles

A profile becomes a configuration object with its own path. Importing is
not the same as connecting.

| # | Command | How to use |
|---|---------|------------|
| 13 | `openvpn3 config-import --config client.ovpn` | Import with the default name |
| 14 | `openvpn3 config-import --config client.ovpn --name lab-client` | Import with a chosen name |
| 15 | `openvpn3 config-import --config client.ovpn --persistent` | Survive service restarts |
| 16 | `openvpn3 config-import --config client.ovpn --tag engagement-a` | Tag for filtering |
| 17 | `openvpn3 configs-list` | Confirm the object exists |
| 18 | `openvpn3 configs-list --count` | Just the count |
| 19 | `openvpn3 configs-list --json` | JSON for scripts |
| 20 | `openvpn3 configs-list --verbose` | Per-profile detail |
| 21 | `openvpn3 configs-list --filter-config lab` | Name-prefix filter |
| 22 | `openvpn3 configs-list --filter-tag engagement-a` | Tag filter |
| 23 | `openvpn3 configs-list --filter-owner alice` | Owner filter |
| 24 | `openvpn3 config-dump --config lab-client` | Dump the stored profile |
| 25 | `openvpn3 config-dump --config lab-client --json` | JSON dump |
| 26 | `openvpn3 config-manage --config lab-client --show` | Effective settings |
| 27 | `openvpn3 config-manage --config lab-client --exists` | Existence check for scripts |
| 28 | `openvpn3 config-manage --config lab-client --rename lab-client-2` | Rename |
| 29 | `openvpn3 config-manage --config lab-client --tag reviewed` | Add a tag |
| 30 | `openvpn3 config-manage --config lab-client --remove-tag reviewed` | Remove a tag |

**Research questions**
- What is stored in the configuration object versus the original file?
- Does `--persistent` change where the profile lives?
- What happens when two profiles share a name?
- Which fields in `config-dump --json` are safe to share?
- Does `--filter-owner` accept a username, a UID, or both?

**Sources:** `openvpn3-config-import(1)`, `openvpn3-configs-list(1)`,
`openvpn3-config-dump(1)`; configuration-service D-Bus docs.
**Output:** a profile inventory with paths and ownership.

## Scenario 3 — start and verify a session

Two paths exist: one-shot import-and-start, or start an imported profile.
Verify the interface and the route before trusting the tunnel.

| # | Command | How to use |
|---|---------|------------|
| 31 | `openvpn3 session-start --config client.ovpn` | Import and start in one step |
| 32 | `openvpn3 session-start --config lab-client` | Start by profile name |
| 33 | `openvpn3 session-start --config-path /net/openvpn/v3/configuration/<id>` | Start by object path |
| 34 | `openvpn3 session-start --config client.ovpn --timeout 30` | Bound the connect wait |
| 35 | `openvpn3 session-start --config client.ovpn --background` | Background after credentials |
| 36 | `openvpn3 session-start --config client.ovpn --persist-tun` | Keep the tun device |
| 37 | `openvpn3 session-start --config client.ovpn --dco true` | Request DCO |
| 38 | `openvpn3 sessions-list` | Confirm the session object |
| 39 | `ip addr show tun0` | Confirm the interface |
| 40 | `ip route get 10.10.0.5` | Confirm the path to a target |
| 41 | `openvpn3 session-stats --config lab-client` | Counters |
| 42 | `openvpn3 session-stats --config lab-client --json` | JSON counters |
| 43 | `openvpn3 log --config lab-client` | Follow session log |
| 44 | `openvpn3 log --config lab-client --log-level 6` | Verbose log |
| 45 | `openvpn3 session-auth --auth-req <id>` | Answer a pending auth request |

**Research questions**
- When does `session-start` return: after the handshake, or after the
  request is queued?
- What does `--background` do with a pending auth request?
- Does `--dco true` fail hard when the kernel module is absent?
- Which tun name does netcfg choose, and can it collide?
- How do you prove the route came from netcfg and not from a leftover
  process?

**Sources:** `openvpn3-session-start(1)`, `openvpn3-session-stats(1)`,
`openvpn3-log(1)`; netcfg D-Bus docs.
**Output:** a session record with interface, route, and counter evidence.

## Scenario 4 — manage a live session

Sessions are owned by the session manager, not by the shell that started
them. These commands are the only way to stop one cleanly.

| # | Command | How to use |
|---|---------|------------|
| 46 | `openvpn3 sessions-list` | Re-read the current session set |
| 47 | `openvpn3 session-manage --config lab-client --pause` | Pause |
| 48 | `openvpn3 session-manage --config lab-client --resume` | Resume |
| 49 | `openvpn3 session-manage --config lab-client --restart` | Disconnect and reconnect |
| 50 | `openvpn3 session-manage --config lab-client --disconnect` | Clean disconnect |
| 51 | `openvpn3 session-manage -I tun0 --disconnect` | Select by interface |
| 52 | `openvpn3 session-manage --config lab-client --log-level=6` | Change log level live |
| 53 | `openvpn3 session-manage --config lab-client --timeout 60 --resume` | Bounded resume |
| 54 | `openvpn3 session-manage --cleanup` | Remove stale sessions |
| 55 | `openvpn3 sessions-list` | Confirm the session is gone |

**Research questions**
- What survives `--pause`: the tun device, the routes, or the TCP flow?
- Does `--restart` re-read the configuration or reuse the imported copy?
- What makes a session "stale" for `--cleanup`?
- Can one user disconnect another user's session?
- Does `--disconnect` remove routes and DNS as reliably as a clean exit?

**Sources:** `openvpn3-session-manage(1)`; session-manager D-Bus docs.
**Output:** a session lifecycle record with before/after state.

## Scenario 5 — per-profile overrides

Overrides let one imported profile serve several targets without editing
the file. They are also a common source of "why is it connecting there?"

| # | Command | How to use |
|---|---------|------------|
| 56 | `openvpn3 config-manage --config lab-client --server-override vpn2.example.com` | Change the server |
| 57 | `openvpn3 config-manage --config lab-client --port-override 443` | Change the port |
| 58 | `openvpn3 config-manage --config lab-client --proto-override tcp` | Force TCP |
| 59 | `openvpn3 config-manage --config lab-client --ipv6 no` | Disable IPv6 in the tunnel |
| 60 | `openvpn3 config-manage --config lab-client --dns-setup-disabled true` | Do not touch DNS |
| 61 | `openvpn3 config-manage --config lab-client --dns-scope tunnel` | Tunnel-only DNS |
| 62 | `openvpn3 config-manage --config lab-client --dns-fallback-google false` | No Google fallback |
| 63 | `openvpn3 config-manage --config lab-client --allow-compression no` | Refuse compression |
| 64 | `openvpn3 config-manage --config lab-client --tls-version-min tls_1_2` | Minimum TLS |
| 65 | `openvpn3 config-manage --config lab-client --tls-cert-profile preferred` | Certificate policy |
| 66 | `openvpn3 config-manage --config lab-client --enable-legacy-algorithms false` | Block legacy crypto |
| 67 | `openvpn3 config-manage --config lab-client --persist-tun true` | Persist the tun |
| 68 | `openvpn3 config-manage --config lab-client --dco false` | Disable DCO |
| 69 | `openvpn3 config-manage --config lab-client --log-level 4` | Override verbosity |
| 70 | `openvpn3 config-manage --config lab-client --proxy-host 127.0.0.1 --proxy-port 8080` | Proxy overrides |
| 71 | `openvpn3 config-manage --config lab-client --unset-override <name>` | Remove one override |
| 72 | `openvpn3 config-dump --config lab-client --json` | Review the effective object |

**Research questions**
- Which overrides are stored in the profile and which apply per session?
- Does `--dns-setup-disabled true` leave the previous resolver in place?
- What is the difference between `--dns-scope global` and `tunnel`?
- Which override names does `--unset-override` accept?
- Can an override weaken TLS below the profile's own setting?

**Sources:** `openvpn3-config-manage(1)`; OpenVPN 3 core option
documentation; netcfg DNS docs.
**Output:** an override table with the reason for each change.

## Scenario 6 — ACLs, sharing, and sealing

OpenVPN 3's privilege model is object-level. These commands decide who
else can use a profile or a session.

| # | Command | How to use |
|---|---------|------------|
| 73 | `openvpn3 config-acl --config lab-client --show` | Current profile ACL |
| 74 | `openvpn3 config-acl --config lab-client --grant alice` | Grant a user |
| 75 | `openvpn3 config-acl --config lab-client --revoke alice` | Revoke a user |
| 76 | `openvpn3 config-acl --config lab-client --public-access true` | Make it public |
| 77 | `openvpn3 config-acl --config lab-client --transfer-owner-session true` | Move ownership to the session |
| 78 | `openvpn3 config-acl --config lab-client --lock-down true` | Disable profile retrieval |
| 79 | `openvpn3 config-acl --config lab-client --seal` | Permanently read-only |
| 80 | `openvpn3 session-acl --config lab-client --show` | Current session ACL |
| 81 | `openvpn3 session-acl --config lab-client --grant bob` | Grant session access |
| 82 | `openvpn3 session-acl --config lab-client --allow-log-access true` | Let bob read the log |
| 83 | `openvpn3 session-acl --config lab-client --public-access false` | Make it private again |
| 84 | `openvpn3 session-acl --config lab-client --revoke bob` | Revoke session access |

**Research questions**
- What does `--seal` prevent that `--lock-down` does not?
- Does granting profile access also grant the embedded credentials?
- What is the security impact of `--public-access true` on a shared host?
- Can a session ACL outlive the session object?
- Does `--transfer-owner-session` change who can disconnect the tunnel?

**Sources:** `openvpn3-config-acl(1)`, `openvpn3-session-acl(1)`;
D-Bus policy files under `/usr/share/dbus-1/system.d/`.
**Output:** an ACL record with the least-privilege decision.

## Scenario 7 — compatibility, automation, and cleanup

The `openvpn2` front end and `openvpn3-admin` cover the edges: legacy
invocation and service-level automation.

| # | Command | How to use |
|---|---------|------------|
| 85 | `openvpn2 --config client.ovpn` | 2.x-style connect |
| 86 | `openvpn2 --config client.ovpn --daemon` | Background via the 2.x front end |
| 87 | `openvpn2 --config client.ovpn --verb 4` | Verbosity in the front end |
| 88 | `openvpn2 --config client.ovpn --profile-override <key> <value>` | Pass a 3.x override |
| 89 | `sudo openvpn3-admin init-config --write-configs` | Probe and write service configs |
| 90 | `sudo openvpn3-admin netcfg-service --help` | netcfg administration |
| 91 | `sudo openvpn3-admin sessionmgr-service --help` | Session-manager administration |
| 92 | `sudo openvpn3-admin log-service --help` | Log-service administration |
| 93 | `sudo install -d /etc/openvpn3/autoload` | Create the autoload directory |
| 94 | `sudo cp client.ovpn /etc/openvpn3/autoload/` | Stage a profile |
| 95 | `sudo systemctl restart openvpn3-autoload.service` | Load at boot |
| 96 | `openvpn3 config-remove --config lab-client` | Remove a profile (prompts) |
| 97 | `openvpn3 config-remove --config lab-client --force` | Remove without prompting |
| 98 | `openvpn3 configs-list --json` | Prove the profile is gone |
| 99 | `openvpn3 sessions-list` | Prove no session remains |
| 100 | `ip route \| diff - ~/ovpn3-demo/evidence/route-before.txt` | Prove routes were restored |

**Research questions**
- Which 2.x options does `openvpn2` silently ignore?
- Does autoload import or start the profile, and as which user?
- What does `init-config --write-configs` change on disk?
- Is `--force` on `config-remove` safe when a session is still using it?
- How do you prove the DNS state was restored after a session?

**Sources:** `openvpn2 --help`; `openvpn3-admin --help`;
`openvpn3-autoload(8)`; systemd unit files.
**Output:** an automation record and a teardown diff.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `version`, `help`, `shell-completion --list-commands`, `--list-options` | 1 |
| `configs-list` (`--count`, `--json`, `-v`, `--filter-config`, `--filter-tag`, `--filter-owner`) | 1, 2 |
| `sessions-list` | 1, 3, 4, 7 |
| `config-import` (`-c`, `-n`, `-p`, `--tag`) | 2 |
| `config-dump` (`-o`, `--config-path`, `-c`, `-j`) | 2, 5 |
| `config-manage` (`--show`, `--exists`, `--rename`, `--tag`, `--remove-tag`) | 2 |
| `session-start` (`-c`, `-p`, `--persist-tun`, `--timeout`, `--background`, `--dco`) | 3 |
| `session-stats` (`-o`, `--session-path`, `-c`, `-I`, `-j`) | 3 |
| `log` (`--session-path`, `-c`, `-I`, `--log-level`) | 3 |
| `session-auth --auth-req` | 3 |
| `session-manage` (`-o`, `--session-path`, `-c`, `-I`, `--log-level`, `--timeout`, `-P`, `-R`, `--restart`, `-D`, `--cleanup`) | 4 |
| `config-manage` (`--dco`, `--server-override`, `--port-override`, `--proto-override`, `--ipv6`, `--persist-tun`, `--log-level`) | 5 |
| `config-manage` (`--dns-fallback-google`, `--dns-setup-disabled`, `--dns-scope`, `--dns-sync-lookup`, `--auth-fail-retry`) | 5 |
| `config-manage` (`--allow-compression`, `--enable-legacy-algorithms`, `--tls-version-min`, `--tls-cert-profile`) | 5 |
| `config-manage` (`--proxy-host`, `--proxy-port`, `--proxy-username`, `--proxy-password`, `--proxy-auth-cleartext`, `--enterprise-profile`, `--unset-override`) | 5 |
| `config-acl` (`--show`, `--grant`, `--revoke`, `--public-access`, `--transfer-owner-session`, `--lock-down`, `--seal`) | 6 |
| `session-acl` (`--show`, `--grant`, `--revoke`, `--public-access`, `--allow-log-access`) | 6 |
| `openvpn2` (`--config`, `--daemon`, `--verb`, `--profile-override`) | 7 |
| `openvpn3-admin` (`init-config`, `netcfg-service`, `sessionmgr-service`, `log-service`, `variables`, `journal`) | 1, 7 |
| `config-remove` (`--force`) | 7 |

Full text: [`help.txt`](help.txt) and the per-command snapshots.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Service inventory and health | 12 |
| 2 | Import and inspect profiles | 18 |
| 3 | Start and verify a session | 15 |
| 4 | Manage a live session | 10 |
| 5 | Per-profile overrides | 17 |
| 6 | ACLs, sharing, and sealing | 12 |
| 7 | Compatibility, automation, cleanup | 16 |
| **Total** | | **100** |

## Legal and safety

OpenVPN 3 objects are shared on a system bus. `--public-access`,
`--grant`, `--lock-down`, and `--seal` change who can read credentials and
start tunnels; treat them as security changes, not convenience flags.
`--server-override` can redirect a profile to a host the profile author
never intended. Remove imported profiles and sessions when the engagement
ends, and prove the route and DNS state matches the pre-import baseline.
