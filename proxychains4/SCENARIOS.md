# proxychains4 — command scenarios for deep research

Installed build: proxychains4 4.17-3 (Debian package `proxychains4`). The
wrapper options are in [`help.txt`](help.txt); directives below are read from
`/etc/proxychains4.conf` and `man proxychains4`.

> **Boundary tool:** proxychains4 redirects traffic but does not grant
> authorization. Use it only to reach targets that are already in scope, and
> remember that a proxy does not make an out-of-scope connection lawful.

## How to invoke

```text
@deep-research For an authorized assessment that must egress through a SOCKS5
proxy chain, compare proxychains4 4.17-3 with ssh -D, socat SOCKS5-CONNECT, and
a transparent proxy for TCP redirection and DNS handling. Which connections
escape an LD_PRELOAD shim, how do strict/dynamic/random chains fail, and what
evidence proves traffic actually traversed the proxy? Cite the proxychains-ng
source and README, the ssh dynamic-forwarding documentation, and independent
tests; return a chain design with leak checks.
```

Create a lab config and inputs, replacing addresses with in-scope values:

```bash
mkdir -p ~/proxychains-demo/evidence
cp /etc/proxychains4.conf ~/proxychains-demo/lab.conf
```

## Scenario 1 — identify the build and effective config

The wrapper, the preload library, and the config all matter. Confirm each one
before trusting a result.

| # | Command | How to use |
|---|---------|------------|
| 1 | `proxychains4 --help` | Usage (there is no `--version`) |
| 2 | `dpkg-query -W -f='${Version}\n' proxychains4` | Package version |
| 3 | `ls -l /usr/bin/proxychains /usr/bin/proxychains4 /usr/lib/x86_64-linux-gnu/libproxychains.so.4` | Wrapper, alternatives, library |
| 4 | `grep -v '^#' /etc/proxychains4.conf \| sed '/^$/d'` | Active directives only |
| 5 | `proxychains4 -q /bin/echo ok` | Quiet smoke test |
| 6 | `proxychains4 /bin/echo ok` | Show config and preload lines |
| 7 | `proxychains4 -h /bin/echo ok` | Negative test: `-h` is treated as a program |

**Research questions**
- Which config file is actually loaded, and why?
- What do the two startup lines prove about the mechanism?
- Why does `-h` fail while `--help` works?
- Which alternative does `/usr/bin/proxychains` point to?
- What is the difference between the wrapper binary and the shared library?

**Sources:** `man proxychains4` (`FILES`); `/etc/proxychains4.conf`;
`update-alternatives --query proxychains`.
**Output:** a resolved config path and a mechanism diagram.

## Scenario 2 — chain modes

Chain mode decides failure behavior. `strict_chain` is all-or-nothing;
`dynamic_chain` skips dead hops; `random_chain` and `round_robin_chain` spread
connections across entries.

| # | Command | How to use |
|---|---------|------------|
| 8 | `grep -nE 'strict_chain\|dynamic_chain\|round_robin_chain\|random_chain\|chain_len' ~/proxychains-demo/lab.conf` | See shipped modes |
| 9 | `proxychains4 -q -f ~/proxychains-demo/lab.conf curl -sS https://target.example/` | Run with the lab config |
| 10 | `proxychains4 -q -f ~/proxychains-demo/dynamic.conf curl -sS https://target.example/` | Dynamic chain |
| 11 | `proxychains4 -q -f ~/proxychains-demo/random.conf curl -sS https://ifconfig.me` | Random chain |
| 12 | `proxychains4 -q -f ~/proxychains-demo/round.conf curl -sS https://ifconfig.me` | Round-robin chain |
| 13 | `grep -c '^socks\|^http\|^raw' ~/proxychains-demo/lab.conf` | Count proxy entries |

**Research questions**
- What does each mode do when the first proxy is down?
- Does `chain_len` affect `strict_chain`?
- How does `random_chain` interact with multi-connection tools like a browser?
- Which mode is safest when a dead hop must not cause a direct fallback?
- What does `EINTR` look like to the proxified program?

**Sources:** `/etc/proxychains4.conf` comments; `man proxychains4`;
proxychains-ng source (`src/core.c`).
**Output:** a failure-mode table per chain type.

## Scenario 3 — proxy types and authentication

The `[ProxyList]` accepts `http`, `socks4`, `socks5`, and `raw`, with optional
credentials. Only numeric IPv4 addresses are valid in this file.

| # | Command | How to use |
|---|---------|------------|
| 14 | `printf 'socks5 127.0.0.1 1080\n' >> ~/proxychains-demo/socks5.conf` | SOCKS5, no auth |
| 15 | `printf 'socks5 127.0.0.1 1080 alice Summer2026!\n' >> ~/proxychains-demo/socks5auth.conf` | SOCKS5 user/pass |
| 16 | `printf 'socks4 127.0.0.1 1080\n' >> ~/proxychains-demo/socks4.conf` | SOCKS4 |
| 17 | `printf 'http 10.0.0.5 3128 proxyuser proxypass\n' >> ~/proxychains-demo/http.conf` | HTTP CONNECT with basic auth |
| 18 | `printf 'raw 10.0.0.5 9000\n' >> ~/proxychains-demo/raw.conf` | Raw forward |
| 19 | `proxychains4 -q -f ~/proxychains-demo/http.conf curl -sS https://target.example/` | Test the HTTP entry |
| 20 | `proxychains4 -q -f ~/proxychains-demo/socks5auth.conf ssh -o BatchMode=yes alice@target.example` | Test SOCKS5 auth |

**Research questions**
- Which auth method does each proxy type support?
- Why are hostnames not allowed in the proxy list?
- What does `raw` do that the other types do not?
- How do you confirm the proxy accepted the credentials rather than the target?
- What happens when the proxy requires auth and none is configured?

**Sources:** `/etc/proxychains4.conf` (`ProxyList` format); RFC 1928 (SOCKS5);
RFC 1929 (SOCKS5 auth); RFC 7231 §4.3.6 (CONNECT).
**Output:** a proxy-type matrix with an auth result per entry.

## Scenario 4 — DNS handling

DNS is the usual leak. `proxy_dns` sends names through the proxy; disabling all
DNS directives resolves locally.

| # | Command | How to use |
|---|---------|------------|
| 21 | `grep -nE 'proxy_dns\|remote_dns_subnet' ~/proxychains-demo/lab.conf` | Current DNS mode |
| 22 | `proxychains4 -q -f ~/proxychains-demo/lab.conf python3 -c 'import socket; print(socket.gethostbyname("target.example"))'` | Resolve through the proxy |
| 23 | `proxychains4 -q -f ~/proxychains-demo/olddns.conf curl -sS https://target.example/` | `proxy_dns_old` |
| 24 | `proxychains4-daemon &` | Start the DNS daemon |
| 25 | `proxychains4 -q -f ~/proxychains-demo/daemon.conf curl -sS https://target.example/` | `proxy_dns_daemon` |
| 26 | `proxychains4 -q -f ~/proxychains-demo/localdns.conf curl -sS https://target.example/` | Local DNS (leak test) |
| 27 | `tcpdump -i any -n port 53 -c 5` | Watch for local DNS |

**Research questions**
- What address does `proxy_dns` return, and why is it synthetic?
- When does an application reject a synthetic DNS answer?
- What breaks with `proxy_dns_old` that `proxy_dns` handles?
- Why does `proxy_dns_daemon` hang when the daemon is absent?
- How do you prove DNS did not leak locally?

**Sources:** `/etc/proxychains4.conf` DNS comments; `man proxychains4`;
`proxy_dns` source; `tcpdump` capture.
**Output:** a DNS path diagram and a leak-test result.

## Scenario 5 — localnet, dnat, and timeouts

`localnet` bypasses the proxy for matching destinations. `dnat` rewrites a
destination. Both change what the target sees, so record them.

| # | Command | How to use |
|---|---------|------------|
| 28 | `grep -nE 'localnet\|dnat\|tcp_read_time_out\|tcp_connect_time_out' ~/proxychains-demo/lab.conf` | Shipped examples |
| 29 | `proxychains4 -q -f ~/proxychains-demo/split.conf nmap -sT -Pn 10.0.0.5` | Local net bypass |
| 30 | `proxychains4 -q -f ~/proxychains-demo/dnat.conf curl -sS https://target.example/` | Destination rewrite |
| 31 | `proxychains4 -q -f ~/proxychains-demo/timeout.conf curl -sS https://slow.target.example/` | Tight timeouts |
| 32 | `proxychains4 -q -f ~/proxychains-demo/lab.conf curl -m 10 -sS https://target.example/` | App-level bound |
| 33 | `grep -n 'localnet 127' ~/proxychains-demo/lab.conf` | Loopback exclusion |

**Research questions**
- Does `localnet` match before or after `dnat`?
- Why must `remote_dns_subnet` differ from a `localnet` range?
- What happens to a proxied connection when `tcp_connect_time_out` expires?
- Is `localnet` a security control or a convenience?
- How do you verify a `dnat` rule took effect?

**Sources:** `/etc/proxychains4.conf` (`localnet`, `dnat`); `man proxychains4`;
proxychains-ng source (`src/core.c`).
**Output:** a destination-handling table with bypass and rewrite entries.

## Scenario 6 — verification, limits, and cleanup

A proxy claim needs evidence. Watch the proxy port, compare egress addresses,
and demonstrate what the shim does **not** cover.

| # | Command | How to use |
|---|---------|------------|
| 34 | `proxychains4 curl -sS https://ifconfig.me` | Egress address with chain lines |
| 35 | `tcpdump -i any -n 'tcp port 9050' -c 10` | Traffic to the proxy |
| 36 | `ss -tnp \| grep 9050` | Live proxy sockets |
| 37 | `proxychains4 -q nmap -sT -Pn -p 80,443 target.example` | TCP connect scan through the chain |
| 38 | `proxychains4 -q nmap -sU -p 53 target.example` | Negative test: UDP is not proxied |
| 39 | `proxychains4 -q ping -c 1 target.example` | Negative test: ICMP is not proxied |
| 40 | `shred -u ~/proxychains-demo/socks5auth.conf ~/proxychains-demo/http.conf` | Remove credentials |

**Research questions**
- Which evidence proves the connection used the proxy rather than going direct?
- Why do UDP and ICMP bypass an `LD_PRELOAD` connect hook?
- What does `nmap -sT` do differently from `-sS` that makes it work here?
- Which statically linked or setuid tools would also bypass the shim?
- What must be removed from the lab config before sharing it?

**Sources:** `tcpdump(8)`; `ss(8)`; `nmap` documentation on connect vs SYN
scans; `ld.so(8)` on `LD_PRELOAD` and secure-execution mode.
**Output:** a leak-test report with proxy-port captures and negative results.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| `-q` | 1–6 |
| `-f` | 2–6 |
| `--help` | 1 |
| `strict_chain`, `dynamic_chain`, `round_robin_chain`, `random_chain` | 2 |
| `chain_len` | 2 |
| `proxy_dns`, `proxy_dns_old`, `proxy_dns_daemon` | 4 |
| `remote_dns_subnet` | 4 |
| `localnet`, `dnat` | 5 |
| `tcp_read_time_out`, `tcp_connect_time_out` | 5 |
| `[ProxyList]` types and auth | 3 |

Config search order: `$PROXYCHAINS_CONF_FILE`, `-f`, `./proxychains.conf`,
`~/.proxychains/proxychains.conf`, `/etc/proxychains.conf`,
`/etc/proxychains4.conf`. Full wrapper text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Identify build and config | 7 |
| 2 | Chain modes | 6 |
| 3 | Proxy types and auth | 7 |
| 4 | DNS handling | 7 |
| 5 | localnet, dnat, timeouts | 6 |
| 6 | Verification and limits | 7 |
| **Total** | | **40** |

## Legal and safety

A proxy chain hides the source address, not the authorization boundary. Use it
only for in-scope targets, keep the config free of production credentials, and
delete proxy passwords after the engagement. UDP, ICMP, and raw-socket tools
bypass the shim; do not describe a proxychains run as covering them.
