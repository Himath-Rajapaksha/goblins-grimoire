# onesixtyone — command scenarios for deep research

The installed build prints `onesixtyone 0.3.3` in [`help.txt`](help.txt); every
flag below appears there. The snapshot shows `-p` without its port value, which
the man page and upstream source confirm. Commands run as `user` on Debian.

> **Intrusive:** every scenario sends SNMP requests; use only on authorized, in-scope hosts.

## How to invoke

```text
@deep-research Which SNMP community strings still ship as vendor defaults in
2026, and how do SNMPv1-only probes behave against v2c- and v3-only agents?
Cite vendor docs, Net-SNMP behavior, and onesixtyone source; return a
shortlist and exact flags for an authorized /24 sweep.
```

Create the demo inputs used below, replacing addresses with in-scope targets:

```bash
mkdir -p ~/snmp-demo
printf '10.10.10.5\n10.10.10.6\n' > ~/snmp-demo/hosts.txt
printf 'public\nprivate\nsnmp\ncisco\nadmin\n' > ~/snmp-demo/communities.txt
```

## Scenario 1 — verify SNMP exposure on one host

Start with one address to learn whether the agent answers, which community it accepts, and what the banner reveals.

| # | Command | How to use |
|---|---------|------------|
| 1 | `onesixtyone 10.10.10.5` | Built-in `public`/`private` list |
| 2 | `onesixtyone 10.10.10.5 public` | One named community |
| 3 | `onesixtyone -p 1161 10.10.10.5 public` | Non-standard agent port |
| 4 | `onesixtyone -d 10.10.10.5 public` | Configuration-level debug |
| 5 | `onesixtyone -dd 10.10.10.5 public` | Per-packet debug |
| 6 | `snmpget -v2c -c public -Oqv 10.10.10.5 1.3.6.1.2.1.1.1.0` | Independent `sysDescr.0` read |

**Research questions**
- Do v2c-only agents answer onesixtyone's SNMPv1 GET, and which refuse it?
- Which device classes omit `sysDescr.0` or answer `NO ACCESS`?
- Does `-dd` expose retransmission or parsing failures worth tuning?

**Sources**: upstream README and `onesixtyone.c`; RFC 1157; Net-SNMP `snmpget` docs.
**Output**: one confirmed `(ip, community, banner)` record or a documented
no-response result.

## Scenario 2 — sweep an in-scope subnet

CIDR input expands to every address, including network and broadcast, and is the loudest routine use.

| # | Command | How to use |
|---|---------|------------|
| 7 | `onesixtyone 10.10.10.0/24 public` | One community across 256 addresses (noisy) |
| 8 | `onesixtyone -c ~/snmp-demo/communities.txt 10.10.10.0/24 -o ~/snmp-demo/class-c.log -w 100` | Dictionary sweep with log and pacing |
| 9 | `onesixtyone -q -o ~/snmp-demo/class-c-quiet.log 10.10.10.0/24` | Log only, no stdout hit lines |

**Research questions**
- How long does a /24 take at default `-w 10`, and how much does `-w 100` add?
- Do network/broadcast probes produce replies or ICMP noise on this segment?
- What is the largest CIDR within the 65536-host cap, and why is `/16` the
  practical maximum?

**Sources**: `help.txt` limits; upstream performance notes; IDS/NetFlow
documentation for UDP scan detection.
**Output**: a timestamped hit log plus a measured runtime for the range.

## Scenario 3 — use host lists and mixed entries

`-i` takes addresses and CIDRs, so scope files feed it directly. Append a range
with `printf '10.10.10.0/28\n' >> ~/snmp-demo/hosts.txt` before the next run.

| # | Command | How to use |
|---|---------|------------|
| 10 | `onesixtyone -i ~/snmp-demo/hosts.txt` | Defaults against a two-host list |
| 11 | `onesixtyone -i ~/snmp-demo/hosts.txt -c ~/snmp-demo/communities.txt -o ~/snmp-demo/hosts.log` | Full dictionary with log |
| 12 | `onesixtyone -i ~/snmp-demo/hosts.txt public` | Positional community with `-i` |

**Research questions**
- Are entries whitespace- or line-delimited, and are blank lines skipped?
- How does the 65536-host cap interact with several CIDRs in one file?
- Why is `-i` plus a positional *host* a usage error while a positional
  *community* is accepted?

**Sources**: upstream `read_hosts()` and `add_host()`; `help.txt`.
**Output**: a host list reusable by every later sweep without re-parsing scope.

## Scenario 4 — bound the community dictionary

Every string is tried with no early stop or lockout handling, so dictionary size
controls both runtime and noise.

| # | Command | How to use |
|---|---------|------------|
| 13 | `onesixtyone -c ~/snmp-demo/communities.txt 10.10.10.5` | Custom list against one host |
| 14 | `onesixtyone -c ~/snmp-demo/communities.txt -i ~/snmp-demo/hosts.txt` | List against a host file |
| 15 | `onesixtyone -c ~/snmp-demo/communities.txt 10.10.10.0/24` | List against a range (loud) |

**Research questions**
- Which vendor default communities are still current, and which lists are
  licensed for engagement use?
- How are entries near the 32-byte cap and lists over 16384 entries truncated?
- What coverage gap remains because onesixtyone cannot speak SNMPv3?

**Sources**: upstream `dict.txt`; vendor default-credential advisories;
SecLists; `read_communities()` in upstream source.
**Output**: a ranked dictionary with a measured hit rate and runtime estimate.

## Scenario 5 — tune pacing and estimate runtime

The wait trades speed for reply capture; the default suits a LAN, while VPN, NAT, or lossy wireless needs more.

| # | Command | How to use |
|---|---------|------------|
| 16 | `time onesixtyone -c ~/snmp-demo/communities.txt -i ~/snmp-demo/hosts.txt -o ~/snmp-demo/w10.log -w 10` | Baseline timing at the default |
| 17 | `time onesixtyone -c ~/snmp-demo/communities.txt -i ~/snmp-demo/hosts.txt -o ~/snmp-demo/w100.log -w 100` | Ten times the per-packet window |
| 18 | `onesixtyone -c ~/snmp-demo/communities.txt -i ~/snmp-demo/hosts.txt -o ~/snmp-demo/w0.log -w 0` | Flood mode; expect dropped replies (noisy) |
| 19 | `split -l 128 ~/snmp-demo/hosts.txt ~/snmp-demo/chunk-` | Chunk a large target list |
| 20 | `for f in ~/snmp-demo/chunk-*; do onesixtyone -c ~/snmp-demo/communities.txt -i "$f" -o "$f.log" -w 50; done` | Run chunks sequentially |

**Research questions**
- Does measured runtime match `hosts x communities x w`, and how large is the
  fixed five-second tail?
- At what `-w` does a lossy link stop producing false negatives?
- How do chunking and pacing change IDS/NetFlow visibility?

**Sources**: upstream `receive_snmp()` and `main()`; RFC 1157 datagram
semantics; local `time` measurements.
**Output**: a pacing table and a chunked run plan with expected duration.

## Scenario 6 — parse hits and hand off to Net-SNMP

The log is a text stream, not structured data. Parsing it into `(ip, community)`
pairs is what turns a sweep into enumerable targets.

| # | Command | How to use |
|---|---------|------------|
| 21 | `onesixtyone -c ~/snmp-demo/communities.txt -i ~/snmp-demo/hosts.txt -o ~/snmp-demo/hits.log` | Produce the authoritative log |
| 22 | `awk -F'[][]' '/\[/ && !/Host responded with error/ {gsub(/ /,"",$1); print $1, $2}' ~/snmp-demo/hits.log > ~/snmp-demo/hits.tsv` | Extract non-error pairs |
| 23 | `while read -r ip community; do snmpwalk -v2c -c "$community" -Oq "$ip" 1.3.6.1.2.1.1; done < ~/snmp-demo/hits.tsv` | Walk the system subtree per pair |
| 24 | `onesixtyone -q -s -c ~/snmp-demo/communities.txt -i ~/snmp-demo/hosts.txt -o ~/snmp-demo/short.log > ~/snmp-demo/short-raw.txt && awk '/^[0-9]/{print $1}' ~/snmp-demo/short-raw.txt > ~/snmp-demo/short-ips.txt && sort -u -o ~/snmp-demo/short-ips.txt ~/snmp-demo/short-ips.txt` | Quiet short mode to a deduplicated IP list |

**Research questions**
- What is the exact grammar of a hit line, and how are error lines formatted?
- Which follow-up OIDs give inventory value with the least additional noise?
- When is `snmp-check` a better report than a hand-built `snmpwalk` loop, and
  is it installed here?

**Sources**: upstream `parse_snmp_response()` and `logr()`; Net-SNMP
`snmpwalk` docs; `snmp-check` project documentation.
**Output**: a TSV inventory and a validated follow-up walk per host.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| Positional `<host>` and `<community>` | 1, 2 |
| `-c <communityfile>` | 2, 4, 6 |
| `-i <inputfile>` | 3, 5, 6 |
| `-p` (port value) | 1 |
| `-w n` | 2, 5 |
| `-o <outputfile>` | 2, 5, 6 |
| `-q` | 2, 6 |
| `-s` | 6 |
| `-d` / `-dd` | 1 |
| Compiled limits (hosts, community length, community count) | 3, 4, 5 |

Defaults: `public private`, port 161, 10 ms. Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Single-host verification | 6 |
| 2 | Subnet sweep | 3 |
| 3 | Host-list input | 3 |
| 4 | Community dictionary | 3 |
| 5 | Pacing and runtime | 5 |
| 6 | Parsing and handoff | 4 |
| **Total** | | **24** |
