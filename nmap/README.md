# nmap — network discovery, port scanning, service/OS detection, NSE

- **Version:** Nmap 7.95 (`nmap --version`)
- **Binaries:** `/usr/bin/nmap`, `/usr/bin/nping` (packet generator)
- **NSE data:** `/usr/share/nmap/scripts/`, `/usr/share/nmap/nselib/`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/nmap/nmap
- **Book / reference:** https://nmap.org/book/man.html
- **NSE docs:** https://nmap.org/nsedoc/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Nmap discovers live hosts, determines which TCP/UDP ports are open, identifies
services and versions, guesses operating systems, and runs Lua NSE scripts that
turn raw ports into findings (TLS weaknesses, SMB misconfigurations, default
credentials, vulnerable services, and more). It is the reference scanner for
scope validation, service inventory, and evidence capture.

Nmap has four moving parts:

1. **Host discovery** — which addresses answer at all (`-sn`, `-Pn`, `-PS`...).
2. **Port scanning** — which ports accept traffic (`-sS`, `-sT`, `-sU`, ...).
3. **Fingerprinting** — what is listening (`-sV`, `-O`, `--script`).
4. **Output** — normal, XML, grepable, and script-kiddie formats (`-oN/-oX/-oG/-oS`).

## Use cases

| Situation | Why nmap |
|-----------|----------|
| Scope validation | Confirm which hosts and services are actually reachable before deeper testing |
| Service inventory | Build a port/service/version table for a subnet or host list |
| Pre-exploitation | Find the exact version behind a port before choosing an exploit or NSE script |
| SMB/AD surface | `smb-*`, `ldap-*`, `msrpc-*` NSE scripts enumerate shares, users, and policy |
| TLS hygiene | `ssl-*` scripts flag deprecated protocols, weak ciphers, and certificate issues |
| Firewall/IDS testing | Fragmentation, decoys, source-port, and timing controls |
| Output for other tools | `-oX`/`-oG` feeds Metasploit, testssl, and custom parsers |

## Requirements & install layout

- Debian package `nmap`; binary `/usr/bin/nmap`. Version 7.95.
- **Raw-socket scans need root or `CAP_NET_RAW`:** `-sS`, `-sU`, `-sA`, `-sW`,
  `-sM`, `-sN`, `-sF`, `-sX`, `-O`, `--send-eth`, and most spoofing options.
  Unprivileged users can still run `-sT` (TCP connect) and `-sn`.
- `-sI` (idle scan) needs a suitable zombie host. `-b` (FTP bounce) needs a
  vulnerable FTP server.
- NSE scripts live under `/usr/share/nmap/scripts`; update the script DB with
  `nmap --script-updatedb` after adding scripts.
- No persistent config; profiles are just shell aliases or wrapper scripts.

## Basic syntax

```bash
nmap [Scan Type(s)] [Options] {target specification}

nmap 10.10.10.5
nmap -sV -sC -p- 10.10.10.5
nmap -sn 10.10.10.0/24
nmap -sU --top-ports 50 10.10.10.5
nmap -iL scope.txt -oA evidence/scan
```

Targets can be hostnames, IPs, CIDR, ranges (`10.0.0.1-254`), or a file
(`-iL`). Exclude with `--exclude` / `--excludefile`.

## Flag reference

### Target specification

| Flag | Meaning |
|------|---------|
| `<target>` | Hostname, IP, CIDR, range, or multiple targets |
| `-iL <file>` | Read targets from a file (one per line) |
| `-iR <n>` | Scan `n` random public addresses (use only for research) |
| `--exclude <list>` | Comma-separated hosts/networks to skip |
| `--excludefile <file>` | Read exclusions from a file |
| `-6` | IPv6 scanning |

### Host discovery

| Flag | Meaning |
|------|---------|
| `-sL` | List scan: resolve targets, do not send packets |
| `-sn` | Ping scan only (no port scan) |
| `-Pn` | Treat every host as online (skip discovery) |
| `-PS/PA/PU/PY[ports]` | TCP SYN/ACK, UDP, or SCTP discovery probes |
| `-PE/PP/PM` | ICMP echo, timestamp, netmask probes |
| `-PO[proto]` | IP-protocol ping |
| `-n` / `-R` | Never / always resolve DNS |
| `--dns-servers <list>` | Use specific resolvers |
| `--traceroute` | Trace hops to each host |

### Scan techniques

| Flag | Meaning |
|------|---------|
| `-sS` | TCP SYN "half-open" scan (root; default when privileged) |
| `-sT` | TCP connect scan (no raw sockets) |
| `-sA` | TCP ACK scan (maps firewall rules) |
| `-sW` | TCP Window scan |
| `-sM` | TCP Maimon scan |
| `-sU` | UDP scan |
| `-sN/-sF/-sX` | TCP Null/FIN/Xmas scans |
| `--scanflags <flags>` | Custom TCP flags |
| `-sI <zombie[:port]>` | Idle scan through a zombie |
| `-sY/-sZ` | SCTP INIT/COOKIE-ECHO scans |
| `-sO` | IP protocol scan |
| `-b <ftp relay>` | FTP bounce scan |

### Ports and order

| Flag | Meaning |
|------|---------|
| `-p <range>` | Ports, e.g. `-p22`, `-p1-65535`, `-p U:53,T:80` |
| `--exclude-ports <range>` | Skip ports |
| `-F` | Fast mode (100 most common ports) |
| `-r` | Scan ports sequentially (no randomization) |
| `--top-ports <n>` | Scan the `n` most common ports |
| `--port-ratio <ratio>` | Scan ports more common than ratio |

### Service, OS, and scripts

| Flag | Meaning |
|------|---------|
| `-sV` | Service/version detection |
| `--version-intensity <0-9>` | Probe aggressiveness (default 7) |
| `--version-light` / `--version-all` | Intensity 2 / 9 |
| `--version-trace` | Debug version probes |
| `-O` | OS detection |
| `--osscan-limit` | Limit OS detection to promising hosts |
| `--osscan-guess` | Guess OS more aggressively |
| `-sC` | Run the `default` NSE script category |
| `--script <list>` | Scripts, files, directories, or categories |
| `--script-args <k=v,...>` | NSE arguments |
| `--script-args-file <file>` | NSE arguments from a file |
| `--script-trace` | Show NSE traffic |
| `--script-updatedb` | Rebuild the script database |
| `--script-help <list>` | Show script help |

### Timing and performance

| Flag | Meaning |
|------|---------|
| `-T<0-5>` | Timing template: paranoid (0) to insane (5); `-T4` is common on LANs |
| `--min-hostgroup/max-hostgroup <n>` | Parallel host group size |
| `--min-parallelism/max-parallelism <n>` | Probe parallelism |
| `--min-rtt-timeout/max-rtt-timeout/initial-rtt-timeout <time>` | RTT bounds |
| `--max-retries <n>` | Cap retransmissions |
| `--host-timeout <time>` | Give up on a host after this long |
| `--scan-delay/--max-scan-delay <time>` | Delay between probes |
| `--min-rate/--max-rate <n>` | Packets per second bounds |

### Evasion and spoofing

| Flag | Meaning |
|------|---------|
| `-f` / `--mtu <n>` | Fragment packets |
| `-D <decoys>` | Decoy source addresses (`ME` includes you) |
| `-S <ip>` | Spoof source address |
| `-e <iface>` | Choose interface |
| `-g/--source-port <port>` | Set source port (e.g. 53, 88) |
| `--proxies <urls>` | Relay through HTTP/SOCKS4 proxies |
| `--data <hex>` / `--data-string <s>` / `--data-length <n>` | Custom payloads |
| `--ip-options <opts>` | IP options |
| `--ttl <n>` | Set TTL |
| `--spoof-mac <mac/prefix/vendor>` | Spoof MAC |
| `--badsum` | Bogus checksums (tests firewalls/IDS) |

### Output

| Flag | Meaning |
|------|---------|
| `-oN <file>` | Normal text output |
| `-oX <file>` | XML output (feeds Metasploit, parsers) |
| `-oG <file>` | Grepable output |
| `-oS <file>` | Script-kiddie output |
| `-oA <base>` | All three major formats at once |
| `-v` / `-vv` | Verbosity |
| `-d` / `-dd` | Debug |
| `--reason` | Show why a port is in a state |
| `--open` | Only show open ports |
| `--packet-trace` | Show every packet |
| `--iflist` | List interfaces and routes |
| `--append-output` | Append instead of overwrite |
| `--resume <file>` | Resume an aborted scan |
| `--stylesheet <path>` / `--webxml` / `--no-stylesheet` | XML styling |

## Recipes

Authorized, in-scope targets only.

**1. Fast host discovery for a /24.**

```bash
nmap -sn 10.10.10.0/24 -oG - | awk '/Up$/{print $2}' > live.txt
```

**2. Top-1000 TCP ports with service and default scripts.**

```bash
nmap -sV -sC -T4 10.10.10.5 -oA evidence/top1000
```

**3. Full TCP port sweep, then targeted version detection.** Full-range scans
are slow; `-T4` and `--min-rate` keep them bounded.

```bash
nmap -p- -T4 --min-rate 2000 10.10.10.5 -oN evidence/allports.txt
nmap -sV -sC -p 22,80,443,445,3389 10.10.10.5 -oA evidence/services
```

**4. UDP top ports.** UDP is lossy; pair with `-sV` and a longer timeout.

```bash
nmap -sU --top-ports 100 --max-retries 2 --host-timeout 10m 10.10.10.5
```

**5. SMB/AD surface enumeration.**

```bash
nmap -p139,445 --script smb-os-discovery,smb-enum-shares,smb-enum-users 10.10.10.5
nmap -p389,636,3268 --script ldap-rootdse,ldap-search 10.10.10.5
```

**6. TLS hygiene check.**

```bash
nmap -p443 --script ssl-enum-ciphers,ssl-cert,ssl-heartbleed 10.10.10.5
```

**7. Firewall/IDS evasion (only with authorization).** Fragmentation, a
trusted source port, and decoys change the packet shape, not the legality.

```bash
nmap -sS -f --mtu 8 -g 53 -D RND:10 -T2 10.10.10.5
```

**8. Resume a long scan and keep all formats.**

```bash
nmap -p- -T4 --min-rate 1000 10.10.10.5 -oA evidence/full
# interrupted?
nmap --resume evidence/full.xml
```

**9. Feed other tools.** XML goes to Metasploit (`db_import`), grepable output
feeds `testssl --file`, and JSON-ish parsing can start from `-oX`.

```bash
nmap -sV -oX evidence/scan.xml 10.10.10.0/24
msfconsole -q -x 'db_import evidence/scan.xml; hosts; services; exit'
```

## Output & pipelines

- **Normal (`-oN`)** is for humans; **XML (`-oX`)** is for tools; **grepable
  (`-oG`)** is the easiest to `awk`.
- Port states: `open`, `closed`, `filtered`, `unfiltered`, `open|filtered`,
  `closed|filtered`. A `filtered` result is not proof the port is closed.
- `--reason` records the packet or response that produced the state — useful
  evidence when a firewall is in play.
- `-oA` writes `.nmap`, `.xml`, and `.gnmap` with one basename; use a per-target
  directory to avoid collisions.
- NSE output is appended under `PORT ... SERVICE` blocks; parse with
  `--script` + `-oX` rather than screen-scraping.

## Gotchas & operational security

- **Privilege changes the scan.** Without root, `-sS` silently falls back to
  `-sT`, and `-O`/`-sU` fail or degrade. Check `nmap --privileged` behavior
  before trusting a result.
- **`-Pn` hides host-down errors.** It is correct behind a firewall that drops
  ICMP, but it makes a dead host look like a host with all ports filtered.
- **`-T4`/`-T5` can lose packets and trip IDS.** Use `-T2`/`-T3` on fragile
  links or protected segments; `--max-rate` is a safer speed control than `-T5`.
- **Full `-p-` scans are loud and slow.** A `/24` full-port scan is 65k probes
  per host; scope it and record the rate.
- **UDP is unreliable by design.** Retransmissions and `open|filtered` results
  are normal; corroborate with a service-specific probe.
- **Version detection is not exploitation.** `-sV` banners can be spoofed; an
  NSE `vulns` hit is a lead, not proof.
- **Decoys and spoofing can break the scan** and may violate provider policy.
  Keep them for explicitly authorized evasion tests.
- **NSE scripts can be intrusive.** `--script vuln` and `smb-brute` send real
  attack traffic; check `--script-help` first.
- **Output files contain target data.** Store them in the engagement directory,
  not in a shared repo.

## Legal & safety

Scan only hosts and networks you own or have **explicit written authorization**
to test. Port scanning and NSE scripts are active operations; unauthorized use
can violate computer-misuse law and provider terms. Do not scan public IP space
or third-party infrastructure. If a script changes state (writes, brute-forces,
exploits), stop unless the rules of engagement explicitly allow it.

## Related tools

- [`masscan/`](../masscan/README.md) — internet-scale port sweeps; hand off
  discovered ports to nmap for version/script work.
- [`tcpdump/`](../tcpdump/README.md) — prove what nmap actually sent/received.
- [`wireshark/`](../wireshark/README.md) — decode the capture.
- [`testssl/`](../testssl/README.md) — deeper TLS checks after `ssl-*` NSE.
- [`netexec/`](../netexec/README.md) — authenticated SMB/LDAP/WinRM follow-up.
- [`enum4linux-ng/`](../enum4linux-ng/README.md) — SMB/AD enumeration beyond NSE.
- [`nuclei/`](../nuclei/README.md) — template-based web checks after port discovery.

## References

- Installed help snapshot: [`help.txt`](help.txt)
- Nmap book and man page: https://nmap.org/book/man.html
- NSE documentation: https://nmap.org/nsedoc/
- Upstream source: https://github.com/nmap/nmap
- Kali tool page: https://www.kali.org/tools/nmap/
