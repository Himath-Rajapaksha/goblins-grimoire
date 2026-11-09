# onesixtyone — fast SNMP community-string scanner

- **Version:** 0.3.3 (banner in [`help.txt`](help.txt); confirm the Debian
  packaging revision with `dpkg-query -W onesixtyone`)
- **Binary:** `/usr/bin/onesixtyone` (Debian package `onesixtyone`)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/trailofbits/onesixtyone
- **Docs:** `man onesixtyone`; https://www.kali.org/tools/onesixtyone/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

onesixtyone is an SNMP scanner for hosts and community strings, not a full SNMP
client. It sends an SNMPv1 `GET` for `sysDescr.0` (`1.3.6.1.2.1.1.0`) to every
host/community pair and logs any reply as `<ip> [<community>] <description>`.
That one OID identifies printers, switches, routers, UPSes, NAS boxes, and
BMC/IPMI controllers, and often leaks a software version in the banner.

Its design point is speed. Instead of waiting seconds per host, it sends packets
continuously with a short per-packet receive window and a brief final wait for
late replies. The trade-off is that UDP loss appears as a false negative, so
pacing (`-w`) and scope size matter.

## Use cases

| Situation | Why onesixtyone |
|-----------|-----------------|
| Internal network inventory | Find SNMP-enabled devices on an in-scope subnet quickly |
| Default-credential checks | Test `public`/`private` and a custom dictionary before enumeration |
| Pre-walk validation | Learn which `(ip, community)` pairs answer before `snmpwalk` |
| Printer/UPS/BMC discovery | `sysDescr` identifies device class without MIB parsing |
| Rogue-device sweeps | A short-mode IP list highlights unexpected SNMP listeners |

## Requirements & install layout

- Installed from the Debian `onesixtyone` package; binary at
  `/usr/bin/onesixtyone`. It uses unprivileged UDP sockets, so no `sudo`.
- No config file, MIB tree, or state directory; host and community lists are
  plain text you supply.
- Outbound UDP to port 161 (or `-p`); no listening port. IPv4 only, and
  hostnames are not resolved.
- Compiled limits: 65536 hosts, 16384 communities, 32 bytes per community;
  upstream ships `dict.txt`, and SecLists has vendor default lists.

## Basic syntax

```bash
onesixtyone 192.168.4.0/24 public
onesixtyone 10.10.10.5
onesixtyone -c communities.txt -i hosts.txt -o snmp-hits.log -w 100
onesixtyone -p 1161 10.10.10.5 public
onesixtyone -dd 10.10.10.5 public
```

With no community argument and no `-c`, the built-in list `public private` is
used. A positional community and `-c` cannot be combined.

## Flag reference

### Inputs and targets

| Argument / flag | Meaning |
|-----------------|---------|
| `<host>` | IPv4 address or IPv4/CIDR range, e.g. `10.10.10.5` or `10.10.10.0/24`; no DNS names |
| `<community>` | One community string to try; rejected when `-c` is used |
| `-c <communityfile>` | File of community names to try (whitespace-separated; up to 16384) |
| `-i <inputfile>` | File of targets, one IPv4 address or CIDR per line |

### Port and timing

| Flag | Meaning |
|------|---------|
| `-p <port>` | Alternate destination SNMP port. The help line prints bare `-p`; the option takes a port value (default 161) |
| `-w n` | Wait `n` milliseconds after each packet for replies (default 10). `0` sends as fast as the kernel accepts |

### Output and diagnostics

| Flag | Meaning |
|------|---------|
| `-o <outputfile>` | Write the log to a file (opened for writing, so an existing file is truncated) |
| `-q` | Quiet mode: do not print hit lines to stdout; intended for use with `-o` |
| `-s` | Short mode: add a bare IP-address line per hit (see gotchas for `-q` interaction) |
| `-d` | Debug output; repeat (`-dd`) for per-packet detail. Printed to stdout, mixed with results |

## Recipes

Every recipe below sends packets to a target and is an **active scan**. Use only
against in-scope, authorized addresses. Prepare demo inputs once:

```bash
mkdir -p ~/snmp-demo
printf '10.10.10.5\n10.10.10.6\n' > ~/snmp-demo/hosts.txt
printf 'public\nprivate\ncisco\n' > ~/snmp-demo/communities.txt
```

**1. Smoke-test one host with the built-in defaults.** Any reply means SNMP is
reachable and one of `public`/`private` is valid.

```bash
onesixtyone 10.10.10.5
```

**2. Sweep a subnet with a custom dictionary and a durable log.** `-w 100`
leaves 100 ms per packet for replies on a lossy or routed link.

```bash
onesixtyone -c ~/snmp-demo/communities.txt 10.10.10.0/24 \
  -o ~/snmp-demo/class-c.log -w 100
```

**3. Sweep a host list and emit clean IPs.** `-q` suppresses hit lines while
`-s` prints bare addresses; `grep` drops the startup line, and the log keeps
full records.

```bash
onesixtyone -q -s -c ~/snmp-demo/communities.txt -i ~/snmp-demo/hosts.txt \
  -o ~/snmp-demo/hits.log | grep -E '^[0-9]' > ~/snmp-demo/live-ips.txt
```

**4. Use a non-standard SNMP port.** For lab forwarders, NAT mappings, or
agents moved off 161.

```bash
onesixtyone -p 1161 10.10.10.5 public
```

**5. Debug a device that should answer.** One `-d` shows configuration; `-dd`
shows every send and reply attempt.

```bash
onesixtyone -dd -c ~/snmp-demo/communities.txt \
  10.10.10.5 2>&1 | tee ~/snmp-demo/debug.log
```

**6. Turn hits into `snmpwalk` queries.** Parse the log into `(ip, community)`
pairs, then walk the system subtree with each pair.

```bash
onesixtyone -c ~/snmp-demo/communities.txt -i ~/snmp-demo/hosts.txt \
  -o ~/snmp-demo/hits.log
awk -F'[][]' '/\[/ && !/Host responded with error/ {gsub(/ /,"",$1); print $1, $2}' \
  ~/snmp-demo/hits.log > ~/snmp-demo/hits.tsv
while read -r ip community; do
  printf '== %s [%s]\n' "$ip" "$community"
  snmpwalk -v2c -c "$community" -Oq "$ip" 1.3.6.1.2.1.1
done < ~/snmp-demo/hits.tsv
```

## Output & pipelines

Default stdout is one hit per line (illustrative record):

```text
10.10.10.5 [public] Hardware: x86 Family 15 Model 0 - Software: Windows 2000
```

The community shown is the one echoed in the response. A valid community that cannot
read `sysDescr.0` may log `Host responded with error ...` instead of a
description, so parse for `[` as well as for text. Startup lines
(`Scanning N hosts, M communities`, `Logging to file ...`) also reach stdout;
filter with `grep -E '^[0-9]'` when piping. There is no JSON or CSV mode.

Handoffs: the `-o` log to `awk` for an IP/community/banner inventory; the `-s`
list to `sort -u`; `(ip, community)` pairs to `snmpwalk`/`snmpbulkwalk` in
[`snmp/`](../snmp/README.md), or to `snmp-check` where installed. The tool exits
0 after a completed scan even with no hits, so detect hits with `grep`, not `$?`.

## Gotchas & operational security

- **No response is ambiguous:** host down, SNMP off, wrong community, late
  reply, or firewall drop all look identical. There are no retries.
- **`-w` is a per-packet receive window, not just spacing.** In the upstream
  source the scanner waits up to `-w` ms after each send, so a run costs about
  `hosts x communities x w` milliseconds plus a fixed five-second tail. `-w 0`
  sends flat out and loses replies.
- **CIDR ranges include network and broadcast addresses.** `10.10.10.0/24`
  probes all 256 addresses; the 65536-host cap limits one entry to `/16`.
- **Only SNMPv1 requests are built.** v3-only or v1-refusing agents look
  silent; use Net-SNMP with `-v3` for those.
- **A restricted view causes false negatives.** An agent may accept the
  community but deny `sysDescr.0`, producing only an error line.
- **Argument forms conflict.** `-c` plus a positional community is a usage
  error, as is `-i` plus a positional host. Pick one input form.
- **`-q` without `-o` discards hit records** unless `-s` is set. `-s` is
  additive in the upstream source: it adds a bare IP line but does not
  suppress the normal hit line unless `-q` is also set. Test this build.
- **`-o` truncates**; reuse of a path overwrites the previous sweep.
- **Communities on the command line appear in `ps` and shell history.** Put
  recurring strings in a mode-600 file and use `-c`.
- **The sweep is loud.** A subnet with a large dictionary is a UDP/161 burst
  that IDS and NetFlow can flag; redirect `-d` output before parsing.

## Legal & safety

Use onesixtyone only on devices you own or have **explicit written
authorization** to test. Sweeping a subnet, guessing community strings, and
reading banners are active operations and unauthorized access when scope is
absent, even though the tool never writes to the device.

Do not probe public IP space, third-party infrastructure, or out-of-scope
internal ranges. If a community grants write access, stop and report it rather
than changing anything. Store results only in the engagement directory.

## Related tools

- [`snmp/`](../snmp/README.md) — Net-SNMP `snmpget`, `snmpwalk`,
  `snmpbulkwalk`, and `snmpset` for follow-up enumeration.
- [`wireshark/`](../wireshark/README.md) — decode SNMP and prove cleartext exposure.
- [`enum4linux-ng/`](../enum4linux-ng/README.md) — companion SMB/NetBIOS enumeration.
- `snmp-check` — external one-shot SNMP report tool; not in this knowledge base.

## References

- Installed help snapshot: [`help.txt`](help.txt)
- Upstream source and README: https://github.com/trailofbits/onesixtyone
- Man page (0.3.3):
  https://manpages.ubuntu.com/manpages/jammy/man1/onesixtyone.1.html
- Kali tool page: https://www.kali.org/tools/onesixtyone/
- Net-SNMP documentation: http://www.net-snmp.org/docs/man/
