# nmap — command scenarios for deep research

Installed build: Nmap 7.95. Every flag below appears in [`help.txt`](help.txt)
or the installed NSE script set. Commands run as `user`; raw-socket scans
(`-sS`, `-sU`, `-O`) need root or `CAP_NET_RAW`.

> **Intrusive:** port scans and NSE scripts send real traffic. Use only on
> authorized, in-scope hosts.

## How to invoke

```text
@deep-research Which nmap host-discovery, port, and NSE combinations give the
most reliable service inventory on a modern filtered network without tipping
into intrusive checks, and when should masscan or a protocol-specific client
replace nmap? Cite the Nmap book, NSE docs, and field reports; return an
ordered scan plan with exact flags and a stop condition for each phase.
```

Prepare the demo scope used below, replacing addresses with in-scope targets:

```bash
mkdir -p ~/nmap-demo/evidence
printf '10.10.10.5\n10.10.10.6\n' > ~/nmap-demo/scope.txt
```

## Scenario 1 — confirm scope and live hosts

Start with discovery only. A host that does not answer may be down, filtered,
or ICMP-blocked; decide which before port scanning.

| # | Command | How to use |
|---|---------|------------|
| 1 | `nmap -sL 10.10.10.0/24` | List/resolve targets without packets |
| 2 | `nmap -sn 10.10.10.0/24` | ICMP/TCP/UDP discovery only |
| 3 | `nmap -sn -PS22,80,443 10.10.10.0/24` | TCP SYN discovery on selected ports |
| 4 | `nmap -sn -Pn 10.10.10.5` | Skip discovery for a known-live host |
| 5 | `nmap -sn 10.10.10.0/24 -oG - \| awk '/Up$/{print $2}'` | Emit a clean live-host list |

**Research questions**
- Which discovery probes survive a firewall that drops ICMP echo?
- How many hosts answer only on TCP 80/443 or UDP 53?
- Does `-Pn` change the port results on this segment, and why?

**Sources:** Nmap host-discovery chapter; RFC 792/1122; packet capture.
**Output:** `live.txt` with one confirmed address per line.

## Scenario 2 — port inventory

Choose the port set from the engagement goal, not habit. Top-1000 is a
compromise; full TCP is exhaustive but loud.

| # | Command | How to use |
|---|---------|------------|
| 6 | `nmap -F 10.10.10.5` | 100 most common ports |
| 7 | `nmap --top-ports 1000 -T4 10.10.10.5` | Explicit top-1000 |
| 8 | `nmap -p 22,80,443,445,3389 10.10.10.5` | Named ports |
| 9 | `nmap -p- -T4 --min-rate 2000 10.10.10.5` | Full TCP range, bounded rate |
| 10 | `nmap -sU --top-ports 100 10.10.10.5` | UDP top-100 (root) |
| 11 | `nmap -sS -p- --max-retries 1 10.10.10.5` | SYN scan with fewer retries |

**Research questions**
- What is the measured time difference between `-F`, top-1000, and `-p-`?
- Which UDP services answer only after a retransmit?
- How does `--min-rate` change loss and IDS alerts versus `-T5`?

**Sources:** Nmap port-scanning and performance chapters; local timing runs.
**Output:** a port/state table with the exact command and timestamp.

## Scenario 3 — service, version, and OS fingerprinting

Version detection turns ports into services; OS detection is probabilistic and
needs at least one open and one closed port.

| # | Command | How to use |
|---|---------|------------|
| 12 | `nmap -sV 10.10.10.5` | Default version intensity |
| 13 | `nmap -sV --version-all 10.10.10.5` | Try every probe (slow) |
| 14 | `nmap -O --osscan-guess 10.10.10.5` | OS guess |
| 15 | `nmap -A 10.10.10.5` | OS + version + default scripts + traceroute |
| 16 | `nmap -sV --version-trace 10.10.10.5` | Debug version probes |

**Research questions**
- Which banner is authoritative when `-sV` and an application banner disagree?
- How often does `-O` guess wrong on virtualized or containerized hosts?
- Does `-A` add enough value to justify its noise on this scope?

**Sources:** Nmap version-detection and OS-detection chapters; service banners.
**Output:** `service/version` and `OS` fields copied into the findings table.

## Scenario 4 — NSE for findings, not just banners

NSE categories range from safe (`default`, `discovery`) to intrusive (`brute`,
`exploit`, `vuln`). Read `--script-help` before running anything in the latter.

| # | Command | How to use |
|---|---------|------------|
| 17 | `nmap -sC 10.10.10.5` | Default script category |
| 18 | `nmap -p445 --script smb-os-discovery,smb-enum-shares,smb-enum-users 10.10.10.5` | SMB surface |
| 19 | `nmap -p389 --script ldap-rootdse,ldap-search 10.10.10.5` | LDAP surface |
| 20 | `nmap -p443 --script ssl-enum-ciphers,ssl-cert 10.10.10.5` | TLS posture |
| 21 | `nmap -p80,443 --script http-title,http-headers,http-methods 10.10.10.5` | Web metadata |
| 22 | `nmap --script-help smb-enum-shares` | Read before running |
| 23 | `nmap -p445 --script smb-brute --script-args smbtype=v1 10.10.10.5` | Credential brute (intrusive; authorized only) |

**Research questions**
- Which NSE scripts are safe versus state-changing, and how is that encoded?
- Do `smb-*` results agree with `enum4linux-ng` and `netexec`?
- Which `ssl-*` findings are real weaknesses versus policy differences?

**Sources:** NSE documentation; script source under `/usr/share/nmap/scripts`.
**Output:** one finding per script hit, with the script name and evidence line.

## Scenario 5 — evasion, timing, and fragile links

Evasion options change packet shape, not authorization. Test them only when the
rules of engagement explicitly permit firewall/IDS testing.

| # | Command | How to use |
|---|---------|------------|
| 24 | `nmap -T2 --max-retries 1 10.10.10.5` | Slow, loss-tolerant scan |
| 25 | `nmap -sS -f --mtu 8 10.10.10.5` | Fragment packets |
| 26 | `nmap -sS -g 53 10.10.10.5` | Source port 53 |
| 27 | `nmap -sS -D RND:10 10.10.10.5` | Random decoys |
| 28 | `nmap --spoof-mac 0 10.10.10.5` | Random MAC (local segment) |
| 29 | `nmap --badsum 10.10.10.5` | Bogus checksums |

**Research questions**
- Which options are blocked or normalized by the target's firewall?
- Does fragmentation change the result or only the timing?
- What is the detection signature of each option in the local IDS?

**Sources:** Nmap evasion chapter; firewall/IDS documentation; packet capture.
**Output:** a before/after result table and a detection note.

## Scenario 6 — durable output and tool handoff

Record the command, version, timestamp, and target with every scan. XML is the
most portable evidence format.

| # | Command | How to use |
|---|---------|------------|
| 30 | `nmap -sV -sC -oA ~/nmap-demo/evidence/top1000 10.10.10.5` | All three formats |
| 31 | `nmap -sV -oX ~/nmap-demo/evidence/scan.xml 10.10.10.0/24` | XML for import |
| 32 | `nmap -sV -oG ~/nmap-demo/evidence/scan.gnmap 10.10.10.0/24` | Grepable for `awk` |
| 33 | `nmap --resume ~/nmap-demo/evidence/top1000.xml` | Resume an interrupted scan |
| 34 | `nmap --iflist` | Debug interface/route selection |
| 35 | `nmap --packet-trace -p80 10.10.10.5` | Prove what was sent |

**Research questions**
- Which output format preserves NSE evidence best for a report?
- Does `--resume` preserve the original options and target list?
- What does `--iflist` reveal when the wrong interface is chosen?

**Sources:** Nmap output chapter; Metasploit `db_import` docs.
**Output:** an evidence directory with `.nmap`, `.xml`, `.gnmap`, and a
`command.txt` recording the exact invocation.
