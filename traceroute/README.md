# traceroute — trace the path packets take to a host

- **Version:** Modern traceroute for Linux 2.1.6 (`traceroute -V`); Debian
  package `traceroute` version `1:2.1.6-1`
- **Binaries:** `/usr/sbin/traceroute` and `/usr/bin/traceroute6` (both
  alternatives symlinks)
- **Full help snapshot:** [`help.txt`](help.txt) — `traceroute --help`
- **Upstream:** http://traceroute.sourceforge.net/ (Dmitry Butskoy's "modern"
  traceroute, the fork of LBNL traceroute)
- **Docs:** `man traceroute`, `man traceroute6`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

`traceroute` sends probes with increasing IP TTL values and listens for the
ICMP Time Exceeded messages that routers return when a packet dies. Each TTL
value maps to one hop, so the output is a hop-by-hop list of the path — plus
round-trip times to each hop.

The default method is **UDP** with a destination port that starts at 33434 and
increments per probe. Other methods exist because firewalls treat UDP, ICMP,
and TCP differently:

| Method | Flag | Default destination port |
|--------|------|--------------------------|
| UDP (incrementing) | default | 33434, +1 per probe |
| UDP (fixed) | `-U` | 53 |
| ICMP ECHO | `-I` | sequence from 1 |
| TCP SYN | `-T` | 80 |
| UDPLITE | `-UL` | 53 |
| DCCP Request | `-D` | 33434 |
| Raw protocol | `-P prot` | — |
| Module | `-M name` | module-specific |

Built-in modules on this install: `default`, `icmp`, `tcp`, `udp`, `udplite`,
`dccp`. Module options are printed by `-O help`.

```text
traceroute [options] host [packetlen]
```

## Use cases

| Situation | Method |
|-----------|--------|
| General path discovery | Default UDP |
| UDP is filtered but ICMP is allowed | `-I` |
| ICMP is filtered, test a web path | `-T -p 443` |
| Reach a DNS server path | `-U -p 53` |
| Find the path MTU | `--mtu` |
| Map hops to AS numbers | `-A` |
| See MPLS labels | `-e` |
| Check whether the reverse path differs | `--back` |
| Trace from a specific source or interface | `-s`, `-i` |
| Fast, quiet trace for scripts | `-n -q 1 -N 1` |

## Flag reference

### Target and method

| Flag | Meaning |
|------|---------|
| `-4` / `-6` | Force IPv4 / IPv6 |
| `-I`, `--icmp` | ICMP ECHO probes |
| `-T`, `--tcp` | TCP SYN probes (default port 80) |
| `-U`, `--udp` | UDP to a fixed port (default 53) |
| `-UL` | UDPLITE probes (default port 53) |
| `-D`, `--dccp` | DCCP Request probes (default port 33434) |
| `-P prot`, `--protocol=prot` | Raw packet of protocol `prot` |
| `-M name`, `--module=name` | Use a built-in or external module |
| `-O OPTS,...`, `--options=OPTS,...` | Module-specific options; `-O help` lists them |
| `-p port`, `--port=port` | Destination port (meaning depends on method) |

### Hop range and probe count

| Flag | Meaning |
|------|---------|
| `-f first_ttl`, `--first=first_ttl` | Start at this TTL instead of 1 |
| `-m max_ttl`, `--max-hops=max_ttl` | Maximum TTL (default 30) |
| `-q nqueries`, `--queries=nqueries` | Probes per hop (default 3) |
| `-N squeries`, `--sim-queries=squeries` | Probes in flight simultaneously (default 16) |
| `-w MAX,HERE,NEAR`, `--wait=MAX,HERE,NEAR` | Wait policy (defaults `5.0,3,10`) |
| `-z sendwait`, `--sendwait=sendwait` | Minimum gap between probes; ≤10 is seconds, >10 is milliseconds |
| `--sport=num` | Source port; implies `-N 1` |

### Path and packet controls

| Flag | Meaning |
|------|---------|
| `-F`, `--dont-fragment` | Set the Don't Fragment bit |
| `-g gate,...`, `--gateway=gate,...` | Source-route through gateways (max 8 IPv4, 127 IPv6) |
| `-i device`, `--interface=device` | Outgoing interface |
| `-r` | Bypass the routing table; send directly on an attached network |
| `-s src_addr`, `--source=src_addr` | Source address |
| `-t tos`, `--tos=tos` | IPv4 TOS / IPv6 traffic class |
| `-l flow_label`, `--flowlabel=flow_label` | IPv6 flow label |
| `--fwmark=num` | Firewall mark for outgoing packets |
| `--mtu` | Discover path MTU; implies `-F -N 1` |

### Output

| Flag | Meaning |
|------|---------|
| `-n` | Do not resolve hop addresses to names |
| `-e`, `--extensions` | Show ICMP extensions, including MPLS |
| `-A`, `--as-path-lookups` | Look up AS paths in routing registries |
| `--back` | Guess the reverse path and flag differences |
| `-d`, `--debug` | Socket-level debugging |
| `-V`, `--version` | Version and exit |
| `--help` | Help and exit |

`packetlen` is the full packet length. The default is the IP header plus 40
bytes; smaller values are raised to the minimum the method allows.

## Recipes

**1. Basic trace.**

```bash
traceroute target.example
```

**2. Numeric only — no reverse DNS.**

```bash
traceroute -n target.example
```

**3. ICMP method (often passes where UDP is filtered).**

```bash
traceroute -I target.example
```

**4. TCP SYN to 443.**

```bash
traceroute -T -p 443 target.example
```

**5. Fixed UDP port.**

```bash
traceroute -U -p 53 target.example
```

**6. Start at hop 5, stop at hop 20.**

```bash
traceroute -f 5 -m 20 target.example
```

**7. Fast, quiet trace for a script.**

```bash
traceroute -n -q 1 -N 1 -w 2 target.example
```

**8. AS-path lookups.**

```bash
traceroute -A -n target.example
```

**9. MPLS labels and ICMP extensions.**

```bash
traceroute -e -n target.example
```

**10. Path MTU discovery.**

```bash
traceroute --mtu target.example
```

**11. Compare forward and reverse paths.**

```bash
traceroute --back -n target.example
```

**12. Trace from a specific interface and source address.**

```bash
traceroute -i eth0 -s 10.0.0.10 target.example
```

**13. IPv6-only trace.**

```bash
traceroute6 -n target.example
```

**14. Module options.**

```bash
traceroute -M tcp -O help target.example
traceroute -M icmp -O raw -n target.example
```

## Gotchas

- **`* * *` is not proof of a dead hop.** It means no response arrived within
  the wait policy. ICMP rate limiting, filtering, and load balancers all
  produce stars on a working path.
- **The default UDP method is often filtered.** Try `-I` or `-T -p 443`
  before concluding a path is broken.
- **Methods have different privilege requirements.** On this install, UDP and
  ICMP via dgram sockets work unprivileged (`ping_group_range` is open), but
  **TCP SYN requires root or `CAP_NET_RAW`** and fails with "You do not have
  enough privileges".
- **Forward probes do not prove the reverse path.** `--back` estimates it; it
  cannot measure it directly.
- **`-A` queries external routing registries.** That leaks the target and your
  interest to third parties, and it is slow. Use it deliberately.
- **`--mtu` is slow.** It implies `-F -N 1`, so probes are serialized.
- **Load balancers produce multiple addresses per hop.** That is real path
  diversity, not a parsing bug. `-q` and `-N` control how much of it you see.
- **TTL is not always hop count.** Tunnels and MPLS can hide hops; `-e` may
  reveal labels but not the underlying topology.
- **Destination ports matter.** A filtered port can make the last hop look
  dead even though the host is up.
- **`-g` source routing is usually disabled.** Modern networks drop the
  option; do not rely on it.
- **`-r` only works on directly attached networks.** It bypasses the routing
  table by design.
- **Name resolution distorts timing.** Reverse DNS lookups add latency to the
  display, not to the probe RTT; use `-n` for clean measurements.
- **`traceroute` is not a loss monitor.** A single run is a snapshot. Use
  repeated runs or `mtr` for loss and jitter trends.
- **`traceroute6` is IPv6-only.** It is not a general-purpose alias.

## Related docs

- [`nmap`](../nmap/README.md) — port and service scanning, including
  `--traceroute`
- [`masscan`](../masscan/README.md) — fast internet-scale scanning
- [`tcpdump`](../tcpdump/README.md) — capture the probes and ICMP replies
- [`wireshark`](../wireshark/README.md) — decode ICMP extensions and MPLS
- [`whois`](../whois/README.md) — map hop addresses to networks
- [`nc`](../nc/README.md) — test whether a specific TCP port is reachable
- [`socat`](../socat/README.md) — build a relay when a path is blocked
