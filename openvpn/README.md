# openvpn — TLS VPN client and multi-client server

- **Version:** OpenVPN 2.6.14 (Debian package `openvpn` `2.6.14-1+deb13u3`)
- **Binary:** `/usr/sbin/openvpn`
- **Help snapshot:** [`help.txt`](help.txt) — `openvpn --help` (536 lines)
- **Upstream:** https://github.com/OpenVPN/openvpn
- **Docs:** `man openvpn`, https://openvpn.net/community-resources/reference-manual-for-openvpn-2-6/
- **Config:** `/etc/openvpn/` (`client/`, `server/`, `easy-rsa/`)
- **Systemd:** `openvpn@.service`, `openvpn-client@.service`,
  `openvpn-server@.service`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

OpenVPN builds a virtual tun/tap interface and carries IP (tun) or Ethernet
(tap) traffic over a single UDP or TCP flow, protected by TLS for key
negotiation and a symmetric cipher for the data channel. It can run as:

- a **client** connecting to a remote server (`--client`),
- a **peer-to-peer** endpoint with a pre-shared static key (`--secret`), or
- a **multi-client server** that hands out addresses from a pool
  (`--mode server` + `--server`).

In a pentest context it is mostly the **transport into an internal network**
after you obtain a `.ovpn` profile, and occasionally the thing you have to
stand up yourself for a lab. It is also a common finding: an exposed OpenVPN
daemon, a weak `tls-auth` key, or a profile that trusts any server
certificate.

```text
openvpn [options]
openvpn --config client.ovpn
```

Options may come from the command line or from a config file
(`--config`). One directive per line in the file, `#` or `;` starts a comment.

## Modes at a glance

| Mode | Key options | Typical use |
|------|-------------|-------------|
| TLS client | `--client`, `--remote`, `--ca`, `--cert`, `--key` | Connect to a normal VPN |
| TLS server | `--mode server`, `--server`, `--tls-server` | Lab or engagement server |
| Static key P2P | `--secret key`, `--remote`, `--ifconfig` | Minimal point-to-point tunnel |
| Bridge | `--dev tap`, `--server-bridge` | Layer-2 access to a segment |
| Management | `--management` | Automate or observe a running client |

`--client` is a macro that expands to `--pull --tls-client` plus a few
defaults; it does not supply certificates. `--server` is a macro for
`--mode server --tls-server --push "..."` and a pool.

## Flag reference

### General and config

| Option | Meaning |
|--------|---------|
| `--config <file>` | Read options from a file |
| `--help` | Show the option summary |
| `--version` | Version, library versions, compile-time features |
| `--cd <dir>` | Change directory before reading relative paths |
| `--chroot <dir>` | Chroot after initialization |
| `--daemon [name]` | Detach after initialization |
| `--syslog [name]` | Log to syslog without detaching |
| `--log <file>` | Truncate and write log |
| `--log-append <file>` | Append to log |
| `--verb <n>` | Verbosity 0–11 (3 is a good summary) |
| `--mute <n>` | Cap repeated identical messages |
| `--writepid <file>` | Write the main PID |
| `--nice <n>` | Process priority |
| `--user <user>`, `--group <group>` | Drop privileges after init |
| `--script-security <0-3>` | How much external programs are trusted |
| `--setenv <name> <value>` | Pass a variable to scripts |
| `--ignore-unknown-option <opt>` | Tolerate a future directive |

### Tunnel and transport

| Option | Meaning |
|--------|---------|
| `--local <host>`, `--remote <host> [port]` | Endpoints |
| `--remote-random` | Pick one of several `--remote` entries at random |
| `--proto <p>` | `udp`, `tcp-server`, `tcp-client`, plus `4`/`6` variants |
| `--port <n>`, `--lport <n>`, `--rport <n>` | Port selection |
| `--bind`, `--nobind` | Bind a local port or not |
| `--dev <tunX\|tapX>` | Virtual device |
| `--dev-type <tun\|tap>` | Device type when the name is non-standard |
| `--dev-node <node>` | Explicit device node |
| `--topology <net30\|p2p\|subnet>` | tun addressing model |
| `--ifconfig <l> <r>` | Point-to-point addresses (or address + mask for tap) |
| `--ifconfig-ipv6 <l> <r>` | IPv6 endpoints |
| `--route <net> [mask] [gw] [metric]` | Add a route after connect |
| `--route-ipv6 <net/bits> [gw] [metric]` | IPv6 route |
| `--route-gateway <gw\|dhcp>` | Gateway for `--route` |
| `--route-delay <n> [w]` | Wait before adding routes |
| `--route-noexec` | Do not install routes; pass them to scripts |
| `--route-nopull` | Ignore routes/DNS/DHCP pushed by the server |
| `--redirect-gateway [flags]` | Send all traffic through the tunnel |
| `--block-ipv6` | Drop IPv6 instead of leaking it |
| `--tun-mtu <n>`, `--link-mtu <n>` | MTU control |
| `--mssfix [n]`, `--fragment <n>` | TCP MSS / UDP fragmentation |
| `--mtu-test` | Measure path MTU empirically |
| `--sndbuf <n>`, `--rcvbuf <n>` | Socket buffers |
| `--mark <value>` | fwmark on encrypted packets |
| `--bind-dev <dev>` | Bind to a VRF/device |
| `--txqueuelen <n>` | tun/tap TX queue length (Linux) |

### Keepalive, timeouts, and lifecycle

| Option | Meaning |
|--------|---------|
| `--keepalive <n> <m>` | Ping every n s, restart after m s |
| `--ping <n>`, `--ping-exit <n>`, `--ping-restart <n>` | Explicit timers |
| `--ping-timer-rem` | Only run timers when a remote address exists |
| `--inactive <n> [bytes]` | Exit after n s below a byte threshold |
| `--session-timeout <n>` | Hard connection time limit |
| `--connect-retry <n> [max]` | Retry interval and cap |
| `--connect-retry-max <n>` | Maximum attempts |
| `--connect-timeout <n>` | Per-server wait when polling |
| `--resolv-retry <n\|infinite>` | DNS retry |
| `--float` | Accept a peer IP/port change |
| `--persist-tun`, `--persist-key`, `--persist-remote-ip`, `--persist-local-ip` | Keep state across restarts |
| `--up <cmd>`, `--down <cmd>` | Run scripts on device open/close |
| `--up-delay`, `--down-pre`, `--up-restart` | Script timing |
| `--ipchange <cmd>` | Run on remote address change |

### Proxy and DNS

| Option | Meaning |
|--------|---------|
| `--http-proxy <s> <p> [up] [auth]` | HTTP proxy, optional NTLM |
| `--http-proxy-option <type> [parm]` | `VERSION`, `AGENT` |
| `--socks-proxy <s> [p] [up]` | SOCKS5 proxy (default port 1080) |
| `--socks-proxy-retry` | Retry forever on proxy errors |
| `--dns server <n> <option> <value>` | Push DNS server settings (2.6) |
| `--dns search-domains <domain>` | Search list (2.6) |

### Server mode

| Option | Meaning |
|--------|---------|
| `--mode server` | Multi-client server |
| `--server <net> <mask>` | Pool + push defaults for IPv4 |
| `--server-ipv6 <net/bits>` | IPv6 pool |
| `--server-bridge [ip mask start end]` | tap bridging pool |
| `--push "<option>"` | Send an option to clients |
| `--push-reset`, `--push-remove <opt>` | Per-client push control |
| `--ifconfig-pool <start> <end> [mask]` | Explicit pool |
| `--ifconfig-pool-persist <file> [s]` | Persist leases |
| `--ifconfig-push <l> <r>` | Static address for one client |
| `--iroute <net> [mask]` | Route a client subnet |
| `--client-config-dir <dir>` | Per-client config directory |
| `--ccd-exclusive` | Require a per-client config |
| `--client-to-client` | Route client traffic internally |
| `--duplicate-cn` | Allow duplicate certificate CNs |
| `--max-clients <n>` | Connection cap |
| `--connect-freq <n> <s>` | Rate-limit new connections |
| `--verify-client-cert <none\|optional\|require>` | Client cert policy |
| `--username-as-common-name` | Use auth username as CN |
| `--auth-user-pass-verify <cmd> <method>` | External credential check |
| `--client-connect <cmd>`, `--client-disconnect <cmd>` | Per-client hooks |
| `--learn-address <cmd>` | Validate client addresses |
| `--port-share <host> <port>` | Share a TCP port with HTTPS |

### Client authentication

| Option | Meaning |
|--------|---------|
| `--auth-user-pass [file]` | Username/password, prompt if omitted |
| `--auth-retry <none\|interact\|nointeract>` | Behaviour on auth failure |
| `--static-challenge <text> <echo>` | Challenge/response |
| `--auth-nocache` | Do not cache credentials |
| `--askpass [file]` | Ask for the private-key passphrase |
| `--pull` | Accept pushed options |
| `--pull-filter <accept\|ignore\|reject> <prefix>` | Filter pushed options |
| `--allow-pull-fqdn` | Accept DNS names in pushed routes |
| `--push-peer-info` | Send client details to the server |

### Data-channel crypto

| Option | Meaning |
|--------|---------|
| `--data-ciphers <list>` | Negotiable ciphers (2.5+) |
| `--data-ciphers-fallback <alg>` | Fallback for old peers |
| `--cipher <alg>` | Legacy single cipher |
| `--auth <alg>` | HMAC digest, default SHA1 |
| `--secret <file> [d]` | Static-key mode (deprecated) |
| `--replay-window <n> [t]` | Replay protection window |
| `--replay-persist <file>` | Persist replay state |
| `--mute-replay-warnings` | Silence replay warnings |
| `--test-crypto` | Crypto self-test |
| `--shaper <n>` | Rate-limit output |

### TLS negotiation

| Option | Meaning |
|--------|---------|
| `--tls-server`, `--tls-client` | TLS role |
| `--ca <file>`, `--capath <dir>` | Trust anchors |
| `--cert <file>`, `--key <file>` | Local identity |
| `--extra-certs <file>` | Chain completion |
| `--pkcs12 <file>` | Bundle key + cert + CA |
| `--dh <file>` | Diffie-Hellman parameters (server) |
| `--tls-version-min <v>`, `--tls-version-max <v>` | TLS bounds |
| `--tls-cipher <list>`, `--tls-ciphersuites <list>` | Cipher control |
| `--tls-cert-profile <legacy\|preferred\|suiteb>` | Certificate algorithm policy |
| `--tls-auth <file> [d]` | HMAC on the control channel |
| `--tls-crypt <file>` | Encrypt and authenticate the control channel |
| `--tls-crypt-v2 <file>` | Per-client wrapped control-channel keys |
| `--genkey <type> <file>` | Generate secret/tls-auth/tls-crypt keys |
| `--verify-x509-name <name>` | Pin the peer subject |
| `--verify-hash <hash> [algo]` | Pin a level-1 cert fingerprint |
| `--peer-fingerprint <sha256>` | Pin the peer cert (2.6, man page) |
| `--remote-cert-tls <client\|server>` | Enforce EKU/key usage |
| `--remote-cert-ku <v>`, `--remote-cert-eku <oid>` | Granular cert checks |
| `--crl-verify <crl> [dir]` | Revocation list |
| `--tls-verify <cmd>` | External certificate check |
| `--reneg-sec <max> [min]` | Key renegotiation interval |
| `--hand-window <n>`, `--tran-window <n>` | Handshake/key transition windows |
| `--single-session`, `--tls-exit` | Session policy |

### PKCS#11 and inspection

| Option | Meaning |
|--------|---------|
| `--pkcs11-providers <provider>` | Load a PKCS#11 module |
| `--pkcs11-id <id>` | Select a token identity |
| `--show-pkcs11-ids [provider]` | List available identities |
| `--show-ciphers` | List data-channel ciphers |
| `--show-digests` | List HMAC digests |
| `--show-tls` | List TLS ciphers/suites |
| `--show-engines` | List crypto engines |
| `--show-gateway` | Print the current default gateway |
| `--mktun`, `--rmtun` | Create/remove a persistent tun/tap |

## Recipes

**1. Connect with a standard profile.**

```bash
sudo openvpn --config client.ovpn
```

**2. Connect and stay in the foreground with useful logs.**

```bash
sudo openvpn --config client.ovpn --verb 3
```

**3. Supply credentials non-interactively.**

```bash
printf 'alice\nS3cret\n' > /tmp/creds
sudo openvpn --config client.ovpn --auth-user-pass /tmp/creds --auth-nocache
```

**4. Route all traffic through the tunnel and block IPv6 leaks.**

```bash
sudo openvpn --config client.ovpn --redirect-gateway def1 --block-ipv6
```

**5. Accept only the pushed routes, not the pushed default route.**

```bash
sudo openvpn --config client.ovpn --route-nopull \
  --route 10.10.0.0 255.255.0.0
```

**6. Use a SOCKS5 proxy to reach the server.**

```bash
sudo openvpn --config client.ovpn --socks-proxy 127.0.0.1 1080
```

**7. Pin the server certificate instead of a CA.**

```bash
sudo openvpn --config client.ovpn \
  --peer-fingerprint AB:CD:EF:...
```

**8. Generate a static key for a P2P tunnel.**

```bash
openvpn --genkey secret static.key
```

**9. Generate a tls-crypt key.**

```bash
openvpn --genkey tls-crypt ta.key
```

**10. Generate a tls-crypt-v2 server and client key.**

```bash
openvpn --genkey tls-crypt-v2-server v2-server.key
openvpn --tls-crypt-v2 v2-server.key --genkey tls-crypt-v2-client v2-client.key
```

**11. Inspect the TLS ciphers the build supports.**

```bash
openvpn --show-tls
```

**12. Inspect the data-channel ciphers.**

```bash
openvpn --show-ciphers
```

**13. Print the default gateway (useful before/after a tunnel).**

```bash
openvpn --show-gateway
```

**14. Start a minimal TLS server for a lab.**

```bash
sudo openvpn --mode server --tls-server --dev tun --port 1194 \
  --proto udp --server 10.8.0.0 255.255.255.0 \
  --ca ca.crt --cert server.crt --key server.key --dh dh.pem
```

**15. Push a route and DNS to clients.**

```bash
sudo openvpn --config server.conf \
  --push "route 10.10.0.0 255.255.0.0" \
  --push "dhcp-option DNS 10.8.0.1"
```

**16. Run the client under the systemd template.**

```bash
sudo systemctl start openvpn-client@client.service
sudo journalctl -u openvpn-client@client.service -f
```

**17. Check whether DCO is active.**

```bash
openvpn --version | grep 'DCO version'
```

**18. Test a config without connecting.**

```bash
openvpn --config client.ovpn --test-crypto --verb 4
```

## Gotchas

- **`--client` is a macro, not a complete config.** It adds `--pull` and
  `--tls-client`; you still need `--remote`, `--ca`, and usually
  `--cert`/`--key`.
- **The server pushes options by default.** `--route-nopull` is the blunt
  control; `--pull-filter` is the surgical one.
- **`--redirect-gateway` changes the default route.** Always pair it with a
  way back (`--route` to the VPN server via the original gateway) or you can
  lose the session.
- **IPv6 leaks are easy.** A tunnel that only handles IPv4 leaves IPv6
  traffic on the physical link unless `--block-ipv6` (or a firewall) stops
  it.
- **DCO is compile-time and runtime.** This build has `enable_dco=yes` but
  `DCO version: N/A`; the kernel module is not loaded, so `--disable-dco`
  has no effect until it is.
- **`--tls-auth` and `--tls-crypt` are not interchangeable.** `tls-auth`
  signs the control channel; `tls-crypt` encrypts and authenticates it and
  hides certificates.
- **`--secret` static-key mode is deprecated.** It has no forward secrecy
  and should only be used for a lab or a legacy peer.
- **`--verify-client-cert none` turns the server into password-only.**
  Combine it with `--auth-user-pass-verify` or you have an open door.
- **`--duplicate-cn` breaks per-client addressing assumptions.** Two clients
  with the same CN share identity and confuse `ifconfig-pool-persist`.
- **`--script-security 2` or higher is required for `--up`/`--down`
  scripts.** The default is 1.
- **`--auth-nocache` re-prompts on renegotiation.** In a scripted context,
  supply a credentials file or use the management interface.
- **A `.ovpn` profile is executable configuration.** It can contain `--up`,
  `--plugin`, `--script-security`, and routes. Read it before running it.
- **`--config` paths are relative to `--cd`.** A profile that assumes its
  own directory needs `--cd` (or `cd` first).
- **The systemd units read `/etc/openvpn/<name>.conf`.** `openvpn@name`
  does not take an arbitrary path.
- **`--float` is not a security feature.** It accepts a new peer address;
  authentication still decides whether the peer is legitimate.

## Related docs

- [`openvpn3-client`](../openvpn3-client/README.md) — the OpenVPN 3 Linux
  client and session manager
- [`proxychains4`](../proxychains4/README.md) — force TCP through a proxy
- [`socat`](../socat/README.md) — generic relay and tunnel building block
- [`sshuttle`](../sshuttle/README.md) — VPN-like access over SSH
- [`tor`](../tor/README.md) — anonymising overlay network
- [`wireshark`](../wireshark/README.md) — verify what actually leaves the
  interface
