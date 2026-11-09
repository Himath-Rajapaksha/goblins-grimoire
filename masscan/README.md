# masscan — asynchronous internet-scale port scanner

- **Version:** 1.9.6#stable (`masscan --version`)
- **Binary:** `/usr/bin/masscan` (Debian package `masscan`)
- **Full help snapshot:** [`help.txt`](help.txt); `--echo` prints the active
  configuration: [`help-echo.txt`](help-echo.txt)
- **Upstream:** https://github.com/robertdavidgraham/masscan
- **Docs:** `man masscan`; https://github.com/robertdavidgraham/masscan#readme

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

masscan sends TCP SYN probes as fast as the NIC and link allow, using its own
userland TCP/IP stack rather than the kernel's. It is designed for one job:
sweep very large address spaces for open ports quickly, then hand the results to
nmap for version detection and NSE. It can also grab banners (`--banners`) and
write packet captures (`--pcap`).

It is not a replacement for nmap. It has no service/version database, no OS
fingerprinting, no NSE, and no rich host-discovery logic. Its value is speed and
its cost is noise: a careless `--rate` will saturate a link, trip IDS, and lose
packets.

## Use cases

| Situation | Why masscan |
|-----------|-------------|
| Large authorized range | Sweep a /16 or a cloud VPC faster than nmap |
| Internet-wide research | Measure exposure of one port across address space (with permission) |
| Time-boxed engagement | Find any host with 443/22/3389 before deeper scanning |
| Cloud egress testing | Map which ports a provider allows from a source |
| Banner triage | `--banners` gives a first-pass HTTP/FTP/SMTP banner |
| Evidence capture | `--pcap` records the probe traffic for review |

## Requirements & install layout

- Debian package `masscan`; binary `/usr/bin/masscan`. Version 1.9.6.
- **Root is required** for raw sockets and the userland stack.
- Uses the kernel's routing table by default. On multi-homed hosts, pin the
  interface with `--adapter-ip`, `--adapter-mac`, `--router-mac`, and
  `--adapter-port`.
- No service database or config directory; a config file (`-c`) is just the
  same `key = value` parameters as the CLI.
- `--echo` prints the resolved configuration and exits — the fastest way to see
  defaults and generate a config file.

## Basic syntax

```bash
masscan <targets> -p<ports> [options]

masscan 10.0.0.0/8 -p80
masscan 10.0.0.0/16 -p22,80,443 --rate 10000
masscan -iL scope.txt -p0-65535 --rate 50000 -oL scan.txt
masscan 10.0.0.0/8 -p80 --banners --source-port 61000
```

Targets can be CIDR, ranges (`10.0.0.1-10.0.0.255`), or a file (`-iL`).
`--exclude` and `--excludefile` remove addresses.

## Flag reference

### Targets and ports

| Flag | Meaning |
|------|---------|
| `<ip>/<mask>` | CIDR target, e.g. `10.0.0.0/8` |
| `--range <ip>` | Single address or range |
| `-iL <file>` | Read targets from a file |
| `--exclude <ip/range>` | Exclude one target or range |
| `--excludefile <file>` | Read exclusions from a file |
| `-p <ports>` / `--ports <ports>` | Ports: `80`, `22,80,443`, `0-65535`, `U:53` |
| `--top-ports <n>` | Scan the `n` most common ports |
| `--port-ratio <ratio>` | Scan ports above a popularity ratio |

### Rate, retries, and sharding

| Flag | Meaning |
|------|---------|
| `--rate <pps>` | Packets per second (default 100; raise deliberately) |
| `--retries <n>` | Retransmissions per port |
| `--wait <seconds>` | Wait after the last packet for replies |
| `--seed <n>` | Reproducible probe order |
| `--shard <n>/<total>` | Split one scan across `total` machines |
| `--infinite` | Run until stopped |
| `--offline` | Generate packets without sending (testing) |

### Adapter and packet control

| Flag | Meaning |
|------|---------|
| `--adapter-ip <ip>` | Source IP for the userland stack |
| `--adapter-mac <mac>` | Source MAC |
| `--router-mac <mac>` | Gateway MAC |
| `--adapter-port <port>` | Source port |
| `--adapter-vlan <id>` | VLAN tag |
| `--source-ip <ip>` | Spoof source IP |
| `--source-port <port>` | Spoof source port (e.g. 53) |
| `--ttl <n>` | IP TTL |
| `--badsum` | Bogus TCP checksum |
| `--hello <string>` | Custom payload for `--banners` |
| `--http-user-agent <ua>` | User-Agent for HTTP banner grabs |

### Output

| Flag | Meaning |
|------|---------|
| `-oL <file>` | List output (easy to parse) |
| `-oJ <file>` | JSON output |
| `-oX <file>` | XML output |
| `-oG <file>` | Grepable output |
| `-oB <file>` | Binary output |
| `--output-format <fmt>` | `list`, `json`, `xml`, `grepable`, `binary` |
| `--output-filename <file>` | Output path when not using `-o*` |
| `--pcap <file>` | Write probe traffic to a pcap |
| `--pcap-show` | Print the pcap filename |
| `--resume <file>` | Resume from a previous scan |
| `--regress` | Regression-test the scanner |

### Configuration and diagnostics

| Flag | Meaning |
|------|---------|
| `-c <file>` | Read `key = value` configuration |
| `--echo` | Print configuration and exit |
| `--interactive` | Interactive keyboard control |
| `--debug <level>` | Debug output |

## Recipes

Authorized, in-scope targets only. Start at a low rate and measure loss.

**1. Fast top-port sweep of a /16.**

```bash
masscan 10.10.0.0/16 -p22,80,443,445,3389 --rate 10000 -oL evidence/masscan.txt
```

**2. Full TCP sweep of one host.**

```bash
masscan 10.10.10.5 -p0-65535 --rate 20000 -oL evidence/full.txt
```

**3. Banner grab on discovered web ports.**

```bash
masscan 10.10.0.0/16 -p80,443,8080 --banners --rate 5000 \
  --http-user-agent 'Mozilla/5.0' -oJ evidence/banners.json
```

**4. Reproducible sharded scan.** Split the same target set across hosts; each
shard uses the same seed and its own index.

```bash
masscan 10.10.0.0/16 -p443 --rate 25000 --seed 1234 --shard 1/4 -oL shard1.txt
masscan 10.10.0.0/16 -p443 --rate 25000 --seed 1234 --shard 2/4 -oL shard2.txt
```

**5. Exclude sensitive ranges.**

```bash
printf '10.10.10.0/24\n10.10.20.0/24\n' > ~/masscan-demo/exclude.txt
masscan 10.10.0.0/16 -p443 --excludefile ~/masscan-demo/exclude.txt \
  --rate 10000 -oL evidence/scan.txt
```

**6. Capture the probes for evidence.**

```bash
masscan 10.10.10.5 -p80,443 --rate 1000 --pcap evidence/masscan.pcap \
  -oL evidence/scan.txt
```

**7. Generate a config file.**

```bash
masscan -p80 --rate 1000 --echo > ~/masscan-demo/masscan.conf
masscan -c ~/masscan-demo/masscan.conf
```

**8. Hand off to nmap.** Extract IPs and open ports, then run targeted `-sV`.

```bash
awk '/open/{print $4}' evidence/masscan.txt | sort -u > evidence/ips.txt
nmap -iL evidence/ips.txt -sV -sC -oA evidence/nmap-followup
```

## Output & pipelines

- `-oL` lines look like `open tcp 443 10.10.10.5 1700000000` — `awk '$1=="open"{print $4}'`
  extracts hosts.
- `-oJ` is JSON-lines; `-oX` is XML for import; `-oB` is for `--resume`.
- `--banners` adds a banner line after the open record; it is not a full
  service/version fingerprint.
- `--pcap` writes the probe traffic, not the replies; use `tcpdump` on the
  interface for a bidirectional capture.

## Gotchas & operational security

- **`--rate` is packets per second, not hosts per second.** A /16 with 1000
  ports at `--rate 100000` is still a very large number of packets; measure.
- **Loss looks like closed ports.** Increase `--retries` and `--wait`, or lower
  the rate, before concluding a port is closed.
- **The userland stack can confuse the local kernel.** On a multi-homed or
  container host, pin `--adapter-ip`/`--adapter-mac`/`--router-mac`; otherwise
  replies may go to the wrong interface.
- **SYN cookies and rate limits** on the target can make masscan report fewer
  open ports than nmap.
- **No version detection or NSE.** A masscan "open" is only a TCP handshake
  observation; confirm with nmap.
- **`--banners` is protocol-guessy.** It sends generic probes and may not match
  the actual service.
- **Sharding is manual.** Every shard must use the same target set, seed, and
  total; a mismatched shard silently scans the wrong slice.
- **The scan is extremely loud.** Even a small range at a high rate is a burst
  that IDS, NetFlow, and upstream providers will notice.

## Legal & safety

Use masscan only on address space you own or have **explicit written
authorization** to scan. Internet-wide scans require permission from the network
owner and may violate ISP terms, even for "research." Do not use spoofed source
addresses to hide origin. Keep rates low enough not to degrade the target or
your own link.

## Related tools

- [`nmap/`](../nmap/README.md) — version detection, NSE, OS fingerprinting.
- [`tcpdump/`](../tcpdump/README.md) — verify what masscan put on the wire.
- [`wireshark/`](../wireshark/README.md) — decode `--pcap` output.
- [`netexec/`](../netexec/README.md) — authenticated follow-up on discovered SMB.
- [`httpx/`](../httpx/README.md) — probe discovered HTTP ports at scale.

## References

- Installed help snapshot: [`help.txt`](help.txt)
- `man masscan`
- Upstream README: https://github.com/robertdavidgraham/masscan
- Kali tool page: https://www.kali.org/tools/masscan/
