# torsocks — route a single application through Tor

- **Version:** 2.5.0
- **Binary:** `/usr/bin/torsocks` (plus `libtorsocks.so` LD_PRELOAD shim)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://gitlab.torproject.org/tpo/core/torsocks
- **Man pages:** `man torsocks`, `man torsocks.conf`, `man torsocks(8)`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

torsocks wraps a command and transparently redirects its **TCP** connections
(and DNS lookups, via the shim) to a Tor SOCKS5 listener. It works by setting
`LD_PRELOAD` to a library that intercepts `connect()`, `getaddrinfo()`, and
related calls. The application keeps using normal hostnames and ports; the
shim rewrites them to Tor.

Use it when you want *one* program to use Tor while the rest of the system
keeps its normal connectivity — the opposite of `tor-mode`, which reroutes the
whole machine.

## Use cases

| Situation | Command |
|-----------|---------|
| Fetch a page anonymously | `torsocks curl https://check.torproject.org/api/ip` |
| Run a scanner through Tor | `torsocks nmap -sT -Pn target` |
| Interactive session through Tor | `torsocks --shell` |
| One-off git clone over Tor | `torsocks git clone https://...` |
| Per-connection Tor circuits | `torsocks --isolate curl ...` |
| Verify Tor is in use | `torsocks curl -s https://check.torproject.org/api/ip` |

## Basic syntax

```bash
torsocks [OPTIONS] command [arg ...]
torsocks --shell
```

```bash
torsocks curl https://example.com
torsocks ssh user@host.example
torsocks nmap -sT -Pn -p80,443 example.com
```

## Options (from `torsocks --help`)

| Option | Meaning |
|--------|---------|
| `-h, --help` | Show help |
| `--shell` | Spawn a new shell with torsocks active |
| `--version` | Print version |
| `-d, --debug` | Enable debug output (shows interception details) |
| `-u, --user NAME` | SOCKS5 username (for authenticated SOCKS proxies) |
| `-p, --pass NAME` | SOCKS5 password |
| `-i, --isolate` | Request a new Tor circuit per connection (cannot be combined with `-u`/`-p`) |
| `-6, --ipv6` | Allow IPv6 socket creation |
| `-a, --address HOST` | Tor SOCKS address (numeric IP only) |
| `-P, --port PORT` | Tor SOCKS port |
| `on` / `off` | Set/unset torsocks for the **current shell** (must be sourced: `. torsocks on`) |
| `show` / `sh` | Show current `LD_PRELOAD` value |

## Shell-wide mode

```bash
. torsocks on        # note the leading dot — must be sourced
curl https://check.torproject.org/api/ip
. torsocks off
. torsocks show      # prints the current LD_PRELOAD value
```

`on`/`off` only affect the shell they are sourced in. They do **not** survive a
new terminal, and they do not affect GUI applications launched elsewhere.

## How it behaves

| Traffic | Through Tor? | Notes |
|---------|--------------|-------|
| TCP connect | Yes | Redirected to the SOCKS5 listener |
| DNS (`getaddrinfo`) | Yes | Resolved by Tor, not the local resolver |
| UDP | **No** | SOCKS5 in torsocks does not proxy UDP; DNS is the exception because it is intercepted at the resolver level |
| ICMP (ping) | **No** | Raw sockets bypass the shim |
| Unix sockets | No | Not network traffic |
| Statically linked binaries | **No** | `LD_PRELOAD` is ignored |
| setuid/setgid binaries | **No** | Loader strips `LD_PRELOAD` for security |

This is the key limitation versus `tor-mode`: torsocks only covers dynamically
linked programs that use libc networking calls and do not drop privileges.

## Configuration: `torsocks.conf`

System config lives at `/etc/torsocks.conf`; a per-user copy can be placed at
`~/.torsocks.conf`. torsocks reads `~/.torsocks.conf` first, then the system
file.

Typical contents:

```ini
# Tor SOCKS listener used by the shim
TorAddress 127.0.0.1
TorPort    9050

# Allow connections to these (Tor will still refuse many .onion/private cases)
AllowInbound 0
AllowOutboundLocalhost 1
AllowOutbound 0.0.0.0/0

# Do not torify these destinations (direct connections)
IsolatePID 0
```

- `TorPort 9050` matches the system Tor daemon (`/etc/tor/torrc`).
- Tor Browser's bundled Tor uses **9150**, so `torsocks -P 9150 ...` is needed
  if you want to reuse the browser's daemon.
- `IsolatePID 1` gives each process its own circuit automatically (similar to
  `--isolate`).

To check what port the system daemon uses:

```bash
grep -E '^(SocksPort|SocksListenAddress)' /etc/tor/torrc
ss -ltnp | grep -E '9050|9150'
```

## Environment variables

| Variable | Meaning |
|----------|---------|
| `TOR_SOCKS_PORT` | SOCKS port override |
| `TOR_SOCKS_HOST` | SOCKS host override (numeric) |
| `TOR_ISOLATE_PID` | Isolate circuits per PID (`1`/`0`) |
| `TORSOCKS_CONF_FILE` | Alternate config file |
| `TORSOCKS_DEBUG` | Debug output (`1`) |
| `LD_PRELOAD` | Set automatically by the wrapper; do not set by hand |

## Recipes

```bash
# Verify Tor exit
torsocks curl -s https://check.torproject.org/api/ip
# => {"IsTor":true,"IP":"..."}

# curl through Tor with DNS resolved by Tor
torsocks curl -s https://example.com

# git clone over Tor
torsocks git clone https://github.com/example/repo.git

# TCP connect scan through Tor (must use -sT; SYN scan needs raw sockets)
torsocks nmap -sT -Pn -p 80,443 example.com

# A new circuit per connection
torsocks --isolate curl https://example.com
torsocks --isolate curl https://example.com   # different exit IP

# Point at Tor Browser's SOCKS listener
torsocks -P 9150 curl https://check.torproject.org/api/ip

# Debug why a program is not being torified
torsocks -d curl https://example.com 2>&1 | head -30

# Interactive torified shell
torsocks --shell
```

## Verification

```bash
# 1. Tor daemon running and listening?
sudo systemctl status tor
ss -ltn | grep 9050

# 2. Is the exit actually Tor?
torsocks curl -s https://check.torproject.org/api/ip
# expect: {"IsTor":true,"IP":"<exit-ip>"}

# 3. Compare with direct connection
curl -s https://check.torproject.org/api/ip
# should report IsTor:false and your real IP

# 4. DNS leak check (requests should resolve through Tor, not local DNS)
torsocks curl -s https://dnsleaktest.com/ 2>/dev/null | head
```

## Gotchas

- **UDP and ICMP are not proxied.** Ping, QUIC, and many UDP protocols leak
  or fail outright.
- **Statics and setuid binaries ignore `LD_PRELOAD`.** Go binaries are
  typically statically linked — `torsocks` cannot torify them.
- **`-i` and `-u/-p` are mutually exclusive.**
- Tor refuses connections to private/LAN addresses by default; expect failures
  for intranet hosts.
- The wrapper only affects the process it launches; children inherit it, but
  unrelated terminals do not.
- SOCKS5 authentication (`-u`/`-p`) is for a *proxy*, not a replacement for
  Tor's own configuration.
- If the app spawns helpers via setuid or drops privileges, parts of the
  traffic may bypass Tor.
- For full-system anonymity, use `tor-mode` (root-only) instead; torsocks is
  per-application only.

## Differences from `tor-mode`

| | torsocks | tor-mode |
|--|----------|----------|
| Scope | One command | Whole host (nftables redirect) |
| Needs root | No | Yes |
| UDP/ICMP | Not proxied | ICMP dropped; UDP DNS redirected |
| Static binaries | Not covered | Covered (kernel-level) |
| DNS | Intercepted in libc | Redirected to Tor DNSPort |
| Reversible | Per-command | `tor-mode off` restores direct routing |

## References

- torsocks man pages: `man torsocks`, `man torsocks.conf`, `man torsocks(8)`
- Project: https://gitlab.torproject.org/tpo/core/torsocks
- Tor documentation: https://support.torproject.org/
