# proxychains4 — force TCP through SOCKS/HTTP proxies via LD_PRELOAD

- **Version:** proxychains4 4.17-3 (Debian package `proxychains4`)
- **Binary:** `/usr/bin/proxychains4`; `/usr/bin/proxychains` is an
  alternatives symlink
- **Library:** `/usr/lib/x86_64-linux-gnu/libproxychains.so.4`
- **Daemon:** `/usr/bin/proxychains4-daemon`
- **Config:** `/etc/proxychains4.conf`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/rofl0r/proxychains-ng
- **Docs:** `man proxychains4`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

proxychains4 is a `LD_PRELOAD` shim. It intercepts a program's TCP `connect()`
calls (and, optionally, DNS resolution) and routes them through one or more
SOCKS4, SOCKS5, or HTTP CONNECT proxies. The program does not need proxy
support, because it never knows the connection was redirected.

That design is also the limit. proxychains4 is not a VPN, not a transparent
proxy, and not a packet filter. It works only for dynamically linked programs
that make ordinary TCP connections.

## Use cases

| Situation | Why proxychains4 |
|-----------|------------------|
| CLI tool has no `--proxy` flag | Preload the shim |
| Reach an internal host through one SOCKS hop | One `[ProxyList]` entry |
| Chain two or three proxies | Multiple entries plus a chain mode |
| Send DNS through the proxy | `proxy_dns` |
| Tor-only egress | Default config already points at `127.0.0.1:9050` |
| Keep a lab tool from leaking direct connections | `strict_chain` plus `localnet` review |
| Test an IDS with rotating egress | `random_chain` plus `chain_len` |

## Chain modes

Exactly one mode should be active in the config. The last uncommented one wins.

| Mode | Behavior |
|------|----------|
| `strict_chain` | Every connection traverses every proxy, in list order; all must be up |
| `dynamic_chain` | Same order, but dead proxies are skipped; at least one must be up |
| `round_robin_chain` | Chains of `chain_len`, starting after the last used proxy |
| `random_chain` | Random proxy or chain per connection; useful for IDS testing |

`chain_len` applies only to `random_chain` and `round_robin_chain`.

## DNS modes

| Directive | Method |
|-----------|--------|
| `proxy_dns` | Threaded remote resolver; returns an address in `remote_dns_subnet` |
| `proxy_dns_old` | Legacy `proxyresolv` script plus `dig`; slower, no `.onion` |
| `proxy_dns_daemon <addr>` | Use a running `proxychains4-daemon`; hangs if absent |

Disable all three to let DNS resolve locally — which leaks the names you look
up to the local resolver.

## Config directives

| Directive | Meaning |
|-----------|---------|
| `strict_chain` / `dynamic_chain` / `round_robin_chain` / `random_chain` | Chain mode |
| `chain_len = N` | Chain length for random/round-robin |
| `quiet_mode` | Suppress library output (same as `-q`) |
| `proxy_dns`, `proxy_dns_old`, `proxy_dns_daemon` | DNS handling |
| `remote_dns_subnet N` | First octet of the synthetic DNS range (default 224) |
| `tcp_read_time_out MS` | Read timeout in milliseconds (default 15000) |
| `tcp_connect_time_out MS` | Connect timeout in milliseconds (default 8000) |
| `localnet ADDR[/MASK]` | Destination that bypasses the proxy |
| `dnat FROM TO` | Rewrite a destination before connecting |
| `[ProxyList]` | Proxy entries, one per line |

Proxy-list format:

```text
type  ip  port [user pass]
```

Types: `http`, `socks4`, `socks5`, `raw`. HTTP supports basic auth; SOCKS
supports user/pass. Only numeric IPv4 addresses are accepted in this file.

## Config search order

1. `$PROXYCHAINS_CONF_FILE`
2. the `-f <file>` argument
3. `./proxychains.conf`
4. `$HOME/.proxychains/proxychains.conf`
5. `/etc/proxychains.conf`
6. `/etc/proxychains4.conf`

On this install only `/etc/proxychains4.conf` exists, so it is the effective
default.

## Flag reference

| Flag | Meaning |
|------|---------|
| `-q` | Quiet; overrides `quiet_mode` in the config |
| `-f <file>` | Use a specific config file |
| `--help` | Print usage |

There is no `--version`. `proxychains4 -h` treats `-h` as the program name and
fails; use `--help`.

## Recipes

**1. Run a tool through the default Tor entry.**

```bash
proxychains4 -q curl -sS https://check.torproject.org/api/ip
```

**2. Use a specific config.**

```bash
proxychains4 -q -f ~/proxychains-lab.conf curl -sS https://target.example/
```

**3. SOCKS5 with authentication.**

```bash
# in the config:
# [ProxyList]
# socks5 127.0.0.1 1080 alice Summer2026!
proxychains4 -q ssh -o BatchMode=yes alice@target.example
```

**4. Two-hop chain.**

```bash
# [ProxyList]
# socks5 127.0.0.1 1080
# http   10.0.0.5  3128
proxychains4 -q -f ~/proxychains-chain.conf curl -sS https://target.example/
```

**5. Keep internal destinations direct.**

```bash
# localnet 10.0.0.0/255.0.0.0
# localnet 192.168.0.0/255.255.0.0
proxychains4 -q -f ~/proxychains-split.conf nmap -sT -Pn 10.0.0.5
```

**6. Verify the egress address.**

```bash
proxychains4 -q curl -sS https://ifconfig.me
```

**7. Prove the proxy is in the path.**

```bash
proxychains4 curl -sS https://ifconfig.me
# stderr shows: config file found / preloading / proxy chain used
```

**8. Chain a non-proxy-aware scanner.**

```bash
proxychains4 -q -f ~/proxychains-chain.conf nmap -sT -Pn -p 80,443 target.example
```

## Gotchas

- **TCP only.** UDP, ICMP, and raw sockets are not proxied. `nmap -sU`,
  `ping`, and most VPN protocols bypass the chain.
- **Dynamically linked programs only.** A static binary never loads the shim.
- **Setuid/setgid programs need root.** The loader ignores `LD_PRELOAD` for
  privileged binaries otherwise.
- **`-q` is not the same as `quiet_mode` alone.** The wrapper prints its
  "config file found / preloading" lines before the library reads the config;
  `PROXYCHAINS_QUIET_MODE=1` did not suppress those lines in testing, while
  `-q` did.
- **`strict_chain` fails closed but loudly.** If any proxy is down, the app
  gets `EINTR`; it does not fall back to a direct connection.
- **`proxy_dns` changes what the app sees.** It returns synthetic addresses
  from `remote_dns_subnet`; an app that validates the resolved IP may refuse
  them.
- **`localnet` is a bypass, not a block.** Every matching destination goes
  direct, which may be the opposite of what you want.
- **The default config points at Tor.** If no Tor daemon is listening on
  `127.0.0.1:9050`, every proxied connection fails.
- **`--version` does not exist.** It is interpreted as a program name.
- **Logging can identify targets.** The default stderr lines show chain
  decisions; treat them as engagement data.
- **Do not use it to evade authorization.** Proxying does not change scope.

## Related docs

- [`socat`](../socat/README.md) — explicit relays and `SOCKS5-CONNECT`
- [`nc`](../nc/README.md) — test whether a destination is reachable
- [`nmap`](../nmap/README.md) — TCP connect scans work through the chain
- [`tcpdump`](../tcpdump/README.md) — verify what actually leaves the host
