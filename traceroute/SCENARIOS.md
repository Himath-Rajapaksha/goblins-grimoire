# traceroute — command scenarios for deep research

Installed build: Modern traceroute for Linux 2.1.6 (Debian package
`traceroute` version `1:2.1.6-1`). Every flag below appears in
[`help.txt`](help.txt) or `man traceroute`. Built-in modules: `default`,
`icmp`, `tcp`, `udp`, `udplite`, `dccp`.

> **Scope:** Traceroute sends packets to the target and elicits ICMP from
> every router on the path. Run it only against hosts you are authorized to
> probe, and remember that `-A` sends the target to third-party routing
> registries.

## How to invoke

```text
@deep-research For an authorized network path analysis, compare traceroute
2.1.6 with mtr, Paris-traceroute, and `nmap --traceroute` for hop discovery,
load-balanced paths, and path-MTU detection. Which method (UDP, ICMP, TCP SYN)
is most reliable across common firewall policies, what do stars actually prove,
and how do per-hop RTTs mislead? Cite the traceroute manual, RFC 792, RFC 1812,
and independent measurement papers; return a method-selection matrix with
confidence notes.
```

Create an evidence workspace, replacing targets with in-scope hosts:

```bash
mkdir -p ~/traceroute-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/traceroute-demo/evidence/run-start.txt
```

## Scenario 1 — identify the build, privileges, and modules

Method availability depends on the binary and on privileges. Confirm both
before interpreting a failed trace.

| # | Command | How to use |
|---|---------|------------|
| 1 | `traceroute -V` | Version banner |
| 2 | `traceroute --help \| sed -n '1,20p'` | Synopsis and first options |
| 3 | `readlink -f /usr/sbin/traceroute /usr/bin/traceroute6` | Resolve alternatives |
| 4 | `dpkg-query -W -f='${Version}\n' traceroute` | Package version |
| 5 | `traceroute -O help 127.0.0.1` | Options for the default module |
| 6 | `traceroute -M icmp -O help 127.0.0.1` | ICMP module options |
| 7 | `traceroute -M tcp -O help 127.0.0.1` | TCP module options |
| 8 | `id -u; cat /proc/sys/net/ipv4/ping_group_range` | Privilege and dgram-ICMP availability |

**Research questions**
- Which modules are built in, and which shortcuts map to them?
- Which methods work unprivileged on this host, and why?
- What does `ping_group_range` permit?
- What is the difference between the `traceroute` and `traceroute6` binaries?
- Why does `-O help` still require a host argument?

**Sources:** `traceroute -V`; `man traceroute`; `ping_group_range` kernel
documentation; `getcap(8)`.
**Output:** a method/privilege matrix for this install.

## Scenario 2 — method selection

The probe method decides whether a firewall lets the packet through and what
the destination does with it. Always record which method produced a result.

| # | Command | How to use |
|---|---------|------------|
| 9 | `traceroute -n -m 5 target.example` | Default UDP, incrementing port |
| 10 | `traceroute -I -n -m 5 target.example` | ICMP ECHO |
| 11 | `traceroute -T -n -m 5 -p 80 target.example` | TCP SYN to 80 |
| 12 | `traceroute -T -n -m 5 -p 443 target.example` | TCP SYN to 443 |
| 13 | `traceroute -U -n -m 5 -p 53 target.example` | Fixed UDP port 53 |
| 14 | `traceroute -UL -n -m 5 target.example` | UDPLITE |
| 15 | `traceroute -D -n -m 5 target.example` | DCCP |
| 16 | `traceroute -M tcp -O syn,ack -n -m 5 -p 443 target.example` | Module-specific TCP flags |

**Research questions**
- Which method reaches furthest through the target's filtering?
- Does the final hop answer differently per method, and why?
- What does a TCP RST at the destination prove that a star does not?
- When is `-U -p 53` more informative than the default UDP method?
- Which methods require privileges, and how does that limit a non-root run?

**Sources:** `man traceroute`; RFC 792 (ICMP); RFC 793/9293 (TCP); RFC 3828
(UDPLITE); RFC 4340 (DCCP).
**Output:** a per-method reachability table for the same target.

## Scenario 3 — hop range, probe count, and timing

Default settings are tuned for a quick look, not for measurement. Adjust the
hop window, probes per hop, concurrency, and wait policy explicitly.

| # | Command | How to use |
|---|---------|------------|
| 17 | `traceroute -n -f 5 -m 15 target.example` | Start at hop 5, stop at 15 |
| 18 | `traceroute -n -q 1 target.example` | One probe per hop |
| 19 | `traceroute -n -q 5 target.example` | Five probes per hop |
| 20 | `traceroute -n -N 1 target.example` | Serialize probes |
| 21 | `traceroute -n -N 32 target.example` | More probes in flight |
| 22 | `traceroute -n -w 2 target.example` | Max wait 2 s |
| 23 | `traceroute -n -w 5,3,10 target.example` | Explicit MAX,HERE,NEAR |
| 24 | `traceroute -n -z 0.5 target.example` | Half-second gap between probes |
| 25 | `traceroute -n --sport=40000 -p 443 -T target.example` | Fixed source port; implies `-N 1` |

**Research questions**
- What do `HERE` and `NEAR` mean in the wait policy?
- How does `-N` change the apparent path when load balancers are present?
- Why does `-q 1` produce a cleaner but less representative trace?
- What does `-z` protect against, and what does it cost?
- Why does `--sport` force `-N 1`?

**Sources:** `man traceroute` (`-w`, `-q`, `-N`, `-z`, `--sport`); RFC 1812
on ICMP rate limiting.
**Output:** a timing-parameter table with the trade-off for each.

## Scenario 4 — path analysis: ASN, MPLS, MTU, reverse path

These options turn a hop list into routing evidence, but each has limits.

| # | Command | How to use |
|---|---------|------------|
| 26 | `traceroute -A -n target.example` | AS-path lookups |
| 27 | `traceroute -e -n target.example` | ICMP extensions and MPLS labels |
| 28 | `traceroute --mtu -n target.example` | Path MTU discovery |
| 29 | `traceroute --back -n target.example` | Compare reverse path |
| 30 | `traceroute -F -n -m 10 target.example` | Don't Fragment |
| 31 | `traceroute -n -m 10 -q 3 target.example > ~/traceroute-demo/evidence/forward.txt` | Save a baseline |
| 32 | `traceroute6 -n target.example` | IPv6 path |

**Research questions**
- Which registry does `-A` query, and what does it leak?
- What do MPLS labels in `-e` output tell you about the provider network?
- How does `--mtu` find the MTU, and why is it serialized?
- How reliable is `--back`'s reverse-path guess?
- Why can a hop appear in the forward trace but not the reverse trace?

**Sources:** `man traceroute` (`-A`, `-e`, `--mtu`, `--back`, `-F`); RFC 4884
(ICMP extensions); RFC 4950 (MPLS in ICMP); RFC 1191 (PMTUD).
**Output:** a path annotation with ASN, label, MTU, and asymmetry notes.

## Scenario 5 — source, interface, and packet controls

Multi-homed hosts and policy routing make the source address part of the
measurement. Pin it when you need a reproducible path.

| # | Command | How to use |
|---|---------|------------|
| 33 | `ip route get target.example` | See the chosen source and interface |
| 34 | `traceroute -n -i eth0 target.example` | Pin the outgoing interface |
| 35 | `traceroute -n -s 10.0.0.10 target.example` | Pin the source address |
| 36 | `traceroute -n -t 0x10 target.example` | Set TOS / traffic class |
| 37 | `traceroute -n -l 0x12345 -6 target.example` | IPv6 flow label |
| 38 | `traceroute -n --fwmark=0x1 target.example` | Firewall mark |
| 39 | `traceroute -n -g 10.0.0.1 target.example` | Source route (usually dropped) |

**Research questions**
- How does the chosen source address change the forward path?
- When does `-i` fail even though the interface exists?
- What does `-t` change upstream, and how can it alter queueing?
- Why is `-g` usually ineffective on modern networks?
- What does `--fwmark` let you test that a plain trace cannot?

**Sources:** `man traceroute`; `ip-route(8)`; RFC 791 (TOS); RFC 6437 (IPv6
flow label); Linux policy-routing documentation.
**Output:** a reproducible source/interface configuration for the trace.

## Scenario 6 — evidence, automation, and interpretation limits

A trace is a snapshot. Record the exact invocation, the time, and the method,
and avoid over-claiming from stars or RTTs.

| # | Command | How to use |
|---|---------|------------|
| 40 | `for i in 1 2 3; do traceroute -n -q 1 target.example; sleep 5; done > ~/traceroute-demo/evidence/repeat.txt` | Repeat for stability |
| 41 | `traceroute -n -q 1 target.example \| tee ~/traceroute-demo/evidence/one-run.txt` | Save one run verbatim |
| 42 | `traceroute -n -q 1 target.example >/dev/null 2>&1; echo "exit=$?"` | Capture the exit code |
| 43 | `traceroute -n -q 1 unreachable.invalid >/dev/null 2>&1; echo "exit=$?"` | Failure exit code |
| 44 | `tcpdump -i any -n 'icmp or udp portrange 33434-33534' -c 20` | Independent packet evidence |
| 45 | `date -u +'%Y-%m-%dT%H:%M:%SZ'; traceroute -n -q 1 target.example \| tail -3` | Timestamp the result |
| 46 | `rm -rf ~/traceroute-demo/evidence` | Remove collected path data at close |

**Research questions**
- What exit code does traceroute return when the target is unreachable?
- How do repeated runs differ, and what does that say about path stability?
- Which is stronger evidence: traceroute output or a packet capture?
- What can a star at hop N prove, and what can it not prove?
- What path information is sensitive enough to redact before sharing?

**Sources:** `man traceroute`; `tcpdump(8)`; RFC 792; ICMP rate-limiting
behavior in RFC 1812.
**Output:** a repeatable trace log with timestamps, exit codes, and a capture.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| `-4`, `-6` | 4, 5 |
| `-d`, `-V`, `--help` | 1 |
| `-F`, `--mtu` | 4 |
| `-f`, `-m` | 3 |
| `-g` | 5 |
| `-I`, `-T`, `-U`, `-UL`, `-D`, `-P`, `-M`, `-O` | 1, 2 |
| `-i`, `-s` | 5 |
| `-N`, `-q`, `-w`, `-z`, `--sport` | 3 |
| `-n`, `-e`, `-A`, `--back` | 2, 4, 6 |
| `-p`, `-t`, `-l`, `--fwmark` | 2, 5 |
| `-r` | 5 (discussion) |
| `packetlen` | 4 (MTU context) |

Defaults: max TTL 30, 3 probes per hop, 16 in flight, wait `5.0,3,10`, UDP port
starting at 33434. Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Build, privileges, modules | 8 |
| 2 | Method selection | 8 |
| 3 | Hop range and timing | 9 |
| 4 | ASN, MPLS, MTU, reverse path | 7 |
| 5 | Source and packet controls | 7 |
| 6 | Evidence and interpretation | 7 |
| **Total** | | **46** |

## Legal and safety

Traceroute is an active probe. Every router on the path sees your packets, and
`-A` additionally discloses the target to routing registries. Probe only
in-scope hosts, keep probe rates low enough not to trip ICMP rate limits or
abuse controls, and treat hop addresses, ASNs, and MPLS labels as engagement
data. Delete collected traces at close unless the rules of engagement require
retention.
