# tor — anonymising overlay network daemon

- **Version:** Tor 0.4.9.15 (Debian package `tor` `0.4.9.15-0+deb13u1`)
- **Binary:** `/usr/sbin/tor`
- **Help snapshot:** [`help.txt`](help.txt) — `tor --help` (terse by design)
- **Option list:** [`help-torrc-options.txt`](help-torrc-options.txt) —
  `tor --list-torrc-options` (377 option names)
- **Upstream:** https://gitlab.torproject.org/tpo/core/tor
- **Docs:** `man tor`, `man torrc`, https://spec.torproject.org/
- **Config:** `/etc/tor/torrc`
- **Service:** `tor.service` (enabled and active on this host)
- **Local ports (this install):** SOCKS `127.0.0.1:9050`,
  DNS `127.0.0.1:5353`, TransPort `127.0.0.1:9040`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Tor builds a circuit of three relays (guard, middle, exit) and carries
**TCP streams** through it. Each relay knows only its predecessor and
successor, so no single relay sees both who you are and what you connect
to. The client learns the circuit from a directory consensus signed by the
directory authorities.

It is not a VPN. It does not tunnel IP, it does not carry UDP, and it does
not hide the fact that you are using Tor from your local network. What it
does give you is:

- **TCP proxying** through a SOCKS5 interface (`SocksPort`),
- **DNS resolution** that does not leak to the local resolver (`DNSPort`),
- **transparent TCP proxying** for a whole host or network (`TransPort`),
- **onion services** that are reachable without a public IP
  (`HiddenServiceDir`/`HiddenServicePort`), and
- **a control protocol** for scripting circuit and stream state
  (`ControlPort`).

For pentest work the usual cases are reaching a `.onion` service, sending
traffic from a different exit, or standing up a disposable onion service
for a callback.

```text
tor [options]
tor -f /etc/tor/torrc
```

Almost every option has the same name in `torrc` and on the command line.
Command-line options are prefixed with `--` (or written as
`SocksPort 9999` in the file).

## Standalone modes

| Invocation | What it does |
|------------|--------------|
| `tor` | Run with the default config (`/etc/tor/torrc`) |
| `tor -f <file>` | Run with a specific config |
| `tor --verify-config` | Parse and validate the config, then exit |
| `tor --dump-config [short\|non-builtin]` | Print the effective config |
| `tor --list-torrc-options` | Print every option name |
| `tor --hash-password <pw>` | Produce a `HashedControlPassword` value |
| `tor --keygen [--newpass]` | Generate relay identity keys in `DataDirectory` |
| `tor --list-fingerprint [rsa\|ed25519]` | Generate keys and print nickname/fingerprint |
| `tor --version` | Version |

## Core option groups

### Client ports

| Option | Meaning |
|--------|---------|
| `SocksPort [addr:]port [flags]` | SOCKS5 listener (default 9050) |
| `SocksPolicy <policy>` | Who may use the SOCKS port |
| `DNSPort [addr:]port` | UDP DNS resolver that resolves through Tor |
| `TransPort [addr:]port` | Transparent TCP proxy (needs firewall redirect) |
| `NATDPort [addr:]port` | NATD transparent proxy (macOS/BSD) |
| `HTTPTunnelPort [addr:]port` | HTTP CONNECT tunnel listener |
| `ControlPort [addr:]port` | Control protocol listener |
| `ControlSocket <path>` | Control protocol over a Unix socket |
| `CookieAuthentication 0\|1` | Cookie-based control auth |
| `CookieAuthFile <path>` | Where the control cookie lives |
| `HashedControlPassword <hash>` | Password-based control auth |
| `ControlPortWriteToFile <path>` | Write the chosen control port |

### Circuit and path selection

| Option | Meaning |
|--------|---------|
| `EntryNodes`, `ExitNodes`, `ExcludeNodes`, `ExcludeExitNodes` | Path constraints |
| `StrictNodes 0\|1` | Fail rather than fall back when constraints cannot be met |
| `GeoIPFile`, `GeoIPv6File` | Country database for `{cc}` selection |
| `NewCircuitPeriod <seconds>` | How often to consider a new circuit |
| `MaxCircuitDirtiness <seconds>` | Maximum age of a circuit |
| `CircuitBuildTimeout <seconds>` | Circuit build deadline |
| `LearnCircuitBuildTimeout 0\|1` | Adaptive timeout |
| `EnforceDistinctSubnets 0\|1` | Keep relays in distinct /16s |
| `ClientUseIPv4`, `ClientUseIPv6` | Address family policy |
| `ConnectionPadding 0\|1`, `CircuitPadding 0\|1` | Traffic padding |

### Bridges and pluggable transports

| Option | Meaning |
|--------|---------|
| `UseBridges 0\|1` | Enable bridge use |
| `Bridge <line>` | A bridge relay line |
| `ClientTransportPlugin <transport> exec <path>` | Pluggable transport binary |
| `UpdateBridgesFromAuthority 0\|1` | Fetch bridge descriptors |
| `BridgeDistribution <method>` | How a bridge is handed out |

### Relay operation

| Option | Meaning |
|--------|---------|
| `ORPort [addr:]port` | Relay listener |
| `DirPort [addr:]port` | Directory mirror listener |
| `Nickname <name>` | Relay name |
| `ContactInfo <text>` | Operator contact |
| `ExitRelay 0\|1` | Whether to allow exit traffic |
| `ExitPolicy <policy>` | Exit rules |
| `BandwidthRate`, `BandwidthBurst` | Rate limits |
| `AccountingMax`, `AccountingStart`, `AccountingRule` | Transfer quota |
| `RelayBandwidthRate`, `RelayBandwidthBurst` | Relay-specific limits |
| `MyFamily <fingerprints>` | Declare relay family |
| `Address <addr>` | Public address override |

### Onion services

| Option | Meaning |
|--------|---------|
| `HiddenServiceDir <dir>` | Service key and hostname directory |
| `HiddenServicePort <virtport> <target>` | Map a virtual port to a local target |
| `HiddenServiceVersion 3` | v3 onion address (the only current version) |
| `HiddenServiceEnableIntroDoSDefense 0\|1` | Introduction-point DoS defence |
| `HiddenServiceMaxStreams <n>` | Concurrent streams per circuit |
| `HiddenServiceSingleHopMode 0\|1` | Non-anonymous single hop (dangerous) |
| `HiddenServiceNonAnonymousMode 0\|1` | Required with single-hop |

### Logging and hardening

| Option | Meaning |
|--------|---------|
| `Log <level> <target>` | `notice file /var/log/tor/notices.log` |
| `RunAsDaemon 0\|1` | Detach |
| `User <user>`, `Group <group>` | Drop privileges |
| `DataDirectory <dir>` | State and key directory |
| `PidFile <path>` | PID file |
| `SafeLogging 0\|1` | Redact addresses in logs |
| `Sandbox 0\|1` | Seccomp sandbox |
| `HardwareAccel 0\|1` | Use hardware crypto |

## Recipes

**1. Check whether the running daemon is healthy.**

```bash
systemctl status tor
ss -ltnp | grep -E ':9050|:9051|:9040'
```

**2. Validate a config before restarting.**

```bash
tor --verify-config -f /etc/tor/torrc
```

**3. Print the effective config.**

```bash
tor --dump-config short
```

**4. List every option the build accepts.**

```bash
tor --list-torrc-options
```

**5. Send a request through the SOCKS port.**

```bash
curl --socks5-hostname 127.0.0.1:9050 https://check.torproject.org/api/ip
```

**6. Resolve a name through Tor's DNSPort.**

```bash
dig @127.0.0.1 -p 5353 example.com
```

**7. Reach a `.onion` service.**

```bash
curl --socks5-hostname 127.0.0.1:9050 http://exampleonion.onion/
```

**8. Add a control port with cookie auth.**

```bash
printf 'ControlPort 9051\nCookieAuthentication 1\n' | sudo tee -a /etc/tor/torrc
sudo systemctl restart tor
```

**9. Generate a control password hash.**

```bash
tor --hash-password 'correct horse battery staple'
```

**10. Request a new circuit over the control port.**

```bash
printf 'AUTHENTICATE\r\nSIGNAL NEWNYM\r\nQUIT\r\n' | nc 127.0.0.1 9051
```

**11. Run a client-only instance with a custom SOCKS port.**

```bash
tor --SocksPort 9051 --DataDirectory /tmp/tor-test --RunAsDaemon 0
```

**12. Use a bridge with obfs4.**

```bash
cat <<'EOF' | sudo tee -a /etc/tor/torrc
UseBridges 1
ClientTransportPlugin obfs4 exec /usr/bin/obfs4proxy
Bridge obfs4 192.0.2.10:443 FINGERPRINT cert=CERT iat-mode=0
EOF
```

**13. Create a v3 onion service for a local port.**

```bash
sudo mkdir -p /var/lib/tor/demo
sudo tee -a /etc/tor/torrc <<'EOF'
HiddenServiceDir /var/lib/tor/demo
HiddenServicePort 80 127.0.0.1:8080
HiddenServiceVersion 3
EOF
sudo systemctl restart tor
sudo cat /var/lib/tor/demo/hostname
```

**14. Generate a service key without starting Tor.**

```bash
tor --keygen --DataDirectory /tmp/tor-relay-keys
```

**15. Constrain the exit country.**

```bash
printf 'ExitNodes {nl}\nStrictNodes 1\n' | sudo tee -a /etc/tor/torrc
```

**16. Disable circuit prediction (useful for scripted tests).**

```bash
tor -f /dev/null --SocksPort 9150 --DataDirectory /tmp/tor-demo/data \
  --__DisablePredictedCircuits 1
```

**17. Check the relay fingerprint.**

```bash
tor --list-fingerprint --DataDirectory /tmp/tor-relay-keys
```

**18. Confirm Tor's own view of the circuit.**

```bash
sudo journalctl -u tor -n 50 --no-pager
```

## Gotchas

- **Tor carries TCP only.** UDP, ICMP, and most peer-to-peer protocols do
  not work through `SocksPort`; `DNSPort` is the exception because DNS is
  explicitly handled.
- **DNS leaks are the classic failure.** A tool that resolves names through
  the system resolver before connecting to the SOCKS proxy leaks the name.
  Use `socks5h` (hostname resolution at the proxy), not `socks5`.
- **`TransPort` needs firewall redirects.** Setting the option alone does
  not transparently capture traffic.
- **The control port is a remote code execution surface if exposed.**
  Bind it to localhost, use cookie or password auth, and never expose it.
- **`SIGNAL NEWNYM` does not guarantee a new exit.** It asks for a new
  circuit; the path may still share relays.
- **`.onion` v3 addresses are self-authenticating.** The address is derived
  from the service key; a wrong address cannot be made to resolve by a
  malicious directory.
- **Running as root is discouraged.** The shipped `torrc` and the daemon
  expect to drop to the `debian-tor` user; `--verify-config` warns when run
  as root.
- **Tor is not a VPN.** It does not hide that you use Tor, and it does not
  protect traffic that bypasses the proxy.
- **`ExitNodes {cc}` is a preference unless `StrictNodes 1`.** Without it,
  Tor falls back when no relay matches.
- **Onion service keys are the service identity.** Losing
  `HiddenServiceDir` loses the address; copying it clones the service.
- **`HiddenServiceSingleHopMode` is not anonymity.** It exists for
  performance testing and explicitly trades anonymity away.
- **The clock matters.** Large clock skew breaks consensus download and
  circuit building; check `timedatectl` before debugging Tor.
- **The Debian package runs as a system service.** A second `tor` with the
  same `DataDirectory` and ports will fail; give a test instance its own
  directory and ports.
- **`--dump-config` shows built-in defaults too.** Use `short` to see only
  what differs.

## Related docs

- [`proxychains4`](../proxychains4/README.md) — force TCP through SOCKS
- [`socat`](../socat/README.md) — relay and port-forward primitive
- [`openvpn`](../openvpn/README.md) — IP-level VPN
- [`openvpn3-client`](../openvpn3-client/README.md) — OpenVPN 3 session
  manager
- [`torsocks`](../torsocks/README.md) — wrap an arbitrary program in Tor
- [`tor-browser`](../tor-browser/README.md) — hardened browser bundle
- [`wireshark`](../wireshark/README.md) — confirm what leaves the host
