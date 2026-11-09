# masscan — command scenarios for deep research

Installed build: masscan 1.9.6#stable. Flags come from [`help.txt`](help.txt)
and `masscan --echo` ([`help-echo.txt`](help-echo.txt)). Commands need root.

> **Intrusive:** masscan sends raw TCP SYNs at high rate. Use only on
> authorized, in-scope address space, and keep the rate low enough not to
> degrade the target or your own uplink.

## How to invoke

```text
@deep-research What packet rate, retry count, and wait time make a masscan
sweep reliable on a given link without causing loss or collateral load, and
when should the result be re-scanned with nmap before being trusted? Cite the
masscan README, TCP SYN behavior, and field measurements; return a rate-tuning
procedure with a stop condition.
```

Prepare the demo scope and output directory:

```bash
mkdir -p ~/masscan-demo/evidence
printf '10.10.10.5\n10.10.10.6\n' > ~/masscan-demo/scope.txt
```

## Scenario 1 — small, bounded sweep

Start with a handful of ports and a rate you can measure. The goal is a clean
baseline, not maximum throughput.

| # | Command | How to use |
|---|---------|------------|
| 1 | `masscan 10.10.10.5 -p80` | One host, one port |
| 2 | `masscan 10.10.10.0/24 -p22,80,443 --rate 1000` | /24, three ports |
| 3 | `masscan 10.10.10.0/24 -p22,80,443 --rate 5000 -oL ~/masscan-demo/evidence/scan.txt` | List output |
| 4 | `masscan -iL ~/masscan-demo/scope.txt -p0-65535 --rate 10000` | Full TCP range on a host list |
| 5 | `masscan 10.10.10.5 -p80 --offline` | Generate packets without sending |

**Research questions**
- At what rate does the local link start dropping replies?
- How many ports are missed at `--retries 1` versus `--retries 3`?
- Does `--offline` produce the same packet shape as a live run?

**Sources:** masscan README; local interface counters; `tcpdump` on the uplink.
**Output:** a scan list plus measured packets-per-second and loss estimate.

## Scenario 2 — rate, retries, and wait tuning

The three timing controls interact. `--rate` sets send speed, `--retries` sets
how many SYNs per port, and `--wait` sets how long to listen after the last
packet.

| # | Command | How to use |
|---|---------|------------|
| 6 | `masscan 10.10.10.5 -p1-1000 --rate 1000 --retries 1` | Low rate, one try |
| 7 | `masscan 10.10.10.5 -p1-1000 --rate 10000 --retries 3 --wait 5` | Higher rate, retries |
| 8 | `masscan 10.10.10.0/24 -p443 --rate 50000 --wait 10` | High rate, long tail |
| 9 | `masscan 10.10.10.0/24 -p443 --rate 100000 --retries 0` | Stress case; expect loss |
| 10 | `masscan 10.10.10.0/24 -p443 --seed 1234 --rate 5000` | Reproducible order |

**Research questions**
- Which combination minimizes false negatives on a lossy link?
- Does `--wait` recover late replies that `--rate` missed?
- Is the probe order stable with the same `--seed`?

**Sources:** masscan source and README; repeated runs; pcap timing.
**Output:** a table of rate/retries/wait versus observed open ports.

## Scenario 3 — banners and first-pass service triage

`--banners` sends protocol-specific probes for a subset of ports. It is a
triage aid, not a version scan.

| # | Command | How to use |
|---|---------|------------|
| 11 | `masscan 10.10.10.5 -p80,443 --banners --rate 1000` | HTTP/HTTPS banners |
| 12 | `masscan 10.10.10.0/24 -p21,22,25,80,443,8080 --banners --rate 5000 -oJ ~/masscan-demo/evidence/banners.json` | Multi-protocol banner grab |
| 13 | `masscan 10.10.10.5 -p80 --banners --http-user-agent 'Mozilla/5.0'` | Custom User-Agent |
| 14 | `masscan 10.10.10.5 -p80 --banners --hello 'GET / HTTP/1.0\r\n\r\n'` | Custom probe payload |

**Research questions**
- Which ports actually produce a useful banner with `--banners`?
- How does the banner compare with `nmap -sV` and `httpx`?
- Does a custom `--hello` change the response or only the request?

**Sources:** masscan banner code; nmap/httpx comparison runs.
**Output:** a banner table with confidence notes.

## Scenario 4 — sharding and large ranges

Sharding divides one logical scan across machines. Every shard must agree on
targets, seed, and total.

| # | Command | How to use |
|---|---------|------------|
| 15 | `masscan 10.10.0.0/16 -p443 --rate 25000 --seed 1234 --shard 1/4 -oL ~/masscan-demo/evidence/s1.txt` | Shard 1 of 4 |
| 16 | `masscan 10.10.0.0/16 -p443 --rate 25000 --seed 1234 --shard 2/4 -oL ~/masscan-demo/evidence/s2.txt` | Shard 2 of 4 |
| 17 | `cat ~/masscan-demo/evidence/s*.txt \| sort -u > ~/masscan-demo/evidence/all.txt` | Merge results |
| 18 | `masscan 10.10.0.0/16 -p443 --excludefile ~/masscan-demo/exclude.txt --rate 10000` | Exclude sensitive ranges |
| 19 | `masscan --range 10.10.10.1-10.10.10.254 -p22 --rate 1000` | Explicit range |

**Research questions**
- Do shards cover the range exactly once, with no overlap or gap?
- What happens if one shard uses a different seed or total?
- How should exclusions be distributed across shards?

**Sources:** masscan sharding docs; merge counts; range arithmetic.
**Output:** a merged list whose unique-host count matches the expected range.

## Scenario 5 — evidence capture and handoff

Record the probe traffic and the exact command. masscan is a discovery tool;
the finding is confirmed by a second tool.

| # | Command | How to use |
|---|---------|------------|
| 20 | `masscan 10.10.10.5 -p80,443 --rate 1000 --pcap ~/masscan-demo/evidence/probe.pcap -oL ~/masscan-demo/evidence/scan.txt` | Capture probes |
| 21 | `tcpdump -nr ~/masscan-demo/evidence/probe.pcap 'tcp[tcpflags] & tcp-syn != 0'` | Inspect SYNs |
| 22 | `awk '/open/{print $4}' ~/masscan-demo/evidence/scan.txt \| sort -u > ~/masscan-demo/evidence/ips.txt` | Extract hosts |
| 23 | `nmap -iL ~/masscan-demo/evidence/ips.txt -sV -sC -oA ~/masscan-demo/evidence/nmap` | Confirm services |
| 24 | `masscan -c ~/masscan-demo/masscan.conf` | Re-run from config |

**Research questions**
- Does the pcap contain exactly the probes masscan reported?
- Which masscan "open" results fail nmap confirmation, and why?
- Is the config file equivalent to the original command line?

**Sources:** pcap analysis; nmap comparison; `--echo` output.
**Output:** an evidence directory with pcap, list, nmap confirmation, and a
`command.txt`.
