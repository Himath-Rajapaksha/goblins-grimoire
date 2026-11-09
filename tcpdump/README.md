# tcpdump — command-line packet capture and BPF filtering

- **Version:** tcpdump 4.99.5, libpcap 1.10.5 (TPACKET_V3)
- **Binary:** `/usr/bin/tcpdump` (Debian package `tcpdump`)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/the-tcpdump-group/tcpdump
- **Man page:** https://www.tcpdump.org/manpages/tcpdump.1.html
- **Filter syntax:** https://www.tcpdump.org/manpages/pcap-filter.7.html

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

tcpdump captures packets from an interface (or a file) and prints a decoded
summary. It is the ground truth for "what actually went over the wire": proving
a scan, checking whether a payload arrived, measuring retransmissions, or
capturing traffic for Wireshark. Its filter language is libpcap BPF, which is
also accepted by Wireshark, tshark, and many other tools.

tcpdump is deliberately small. It decodes common protocols and writes pcap; it
does not reassemble streams, extract files, or provide a GUI. Use Wireshark or
tshark for deep dissection.

## Use cases

| Situation | Why tcpdump |
|-----------|-------------|
| Prove a scan/probe | Capture the exact packets nmap or masscan sent |
| Troubleshoot connectivity | See whether SYN, SYN-ACK, or RST arrives |
| Measure retransmits | Count duplicate SYNs and resets |
| Capture for Wireshark | `-w file.pcap` then open the GUI |
| Lightweight remote capture | `ssh host tcpdump -U -w -` piped to local Wireshark |
| Credential/plaintext audit | Filter HTTP, FTP, Telnet, SNMP on an authorized segment |
| Ring-buffer long captures | `-G`/`-W`/`-C` rotate files by time or size |

## Requirements & install layout

- Debian package `tcpdump`; binary `/usr/bin/tcpdump`. Version 4.99.5.
- **Root or `CAP_NET_RAW`/`CAP_NET_ADMIN`** is required to open a capture
  interface. `-r` (read a file) does not need privilege.
- `-Z <user>` drops privileges after opening the interface (default may already
  be `tcpdump` on Debian).
- No config or state files; filters are command-line expressions.
- Interface names come from `ip link` / `tcpdump -D`.

## Basic syntax

```bash
tcpdump [options] [expression]

tcpdump -i eth0
tcpdump -i any -nn 'tcp port 443'
tcpdump -i eth0 -w capture.pcap 'host 10.10.10.5'
tcpdump -nr capture.pcap 'tcp[tcpflags] & tcp-syn != 0'
```

If no interface is given, tcpdump picks one (often the first non-loopback).
`-i any` captures on all interfaces on Linux (cooked capture; link-layer headers
may be unavailable).

## Flag reference

### Capture source and volume

| Flag | Meaning |
|------|---------|
| `-i <iface>` | Interface to capture (`-i any` for all) |
| `-D` | List available interfaces |
| `-c <count>` | Stop after `count` packets |
| `-s <snaplen>` | Capture only the first `snaplen` bytes (0 = whole packet) |
| `-B <size>` | Set the capture buffer size in KiB |
| `--immediate-mode` | Deliver packets as soon as they arrive |
| `-p` | Do not put the interface into promiscuous mode |
| `-Q in\|out\|inout` | Direction filter (Linux) |
| `-y <datalink>` | Set the link-layer type |
| `-I` | Put a Wi-Fi interface into monitor mode |

### Output and files

| Flag | Meaning |
|------|---------|
| `-w <file>` | Write raw packets to a pcap file |
| `-r <file>` | Read packets from a pcap file |
| `-C <size>` | Rotate the output file when it exceeds `size` MB |
| `-G <seconds>` | Rotate the output file every `seconds` seconds |
| `-W <count>` | Limit the number of rotated files |
| `-z <command>` | Run a command on each rotated file |
| `-Z <user>` | Drop privileges to `user` after opening the interface |
| `-U` | Write packets as they arrive (unbuffered) |
| `-E <algo:secret>` | Decrypt IPsec ESP with a known key |
| `-T <type>` | Interpret packets as a specific type |
| `-V <file>` | Read a list of file names |
| `-F <file>` | Read the filter expression from a file |
| `--count` | Print only the matched-packet count when reading a file |
| `--version` | Print the tcpdump and libpcap versions and exit |
| `--time-stamp-precision <p>` | `micro` or `nano` |
| `--micro` / `--nano` | Shorthand timestamp precision |

### Display

| Flag | Meaning |
|------|---------|
| `-n` | Do not resolve hostnames |
| `-nn` | Do not resolve hostnames or ports |
| `-N` | Do not print domain names |
| `-f` | Do not resolve foreign IPv4 addresses |
| `-e` | Print link-layer headers |
| `-A` | Print each packet in ASCII |
| `-X` | Print packet data in hex and ASCII |
| `-x` | Print packet data in hex |
| `-XX` | Hex and ASCII, including link-layer header |
| `-v` / `-vv` / `-vvv` | Increasing verbosity |
| `-q` | Quick output (less protocol detail) |
| `-t` | Do not print timestamps |
| `-tt` | Print Unix epoch timestamps |
| `-tttt` | Print a date and timestamp |
| `-u` | Print undecoded NFS handles |
| `-l` | Line-buffer stdout (useful for pipes) |
| `-#` / `--number` | Print a packet number |
| `-S` | Print absolute TCP sequence numbers |
| `-K` | Do not attempt to verify TCP checksums |
| `-L` | List known data-link types |
| `-H` | Attempt to detect 802.11s draft mesh headers |
| `-J` | List supported time-stamp types |
| `-j <type>` | Set the time-stamp type |
| `-M <secret>` | Use a secret for TCP-MD5 |
| `-b` | Print BGP AS numbers in ASDOT notation |
| `-O` | Do not run the BPF optimizer |
| `--print` | Print parsed packets even when writing raw packets with `-w` |
| `-d` / `-dd` / `-ddd` | Dump the compiled BPF program |

## BPF filter reference

| Expression | Matches |
|------------|---------|
| `host 10.10.10.5` | Traffic to or from one host |
| `src host 10.10.10.5` / `dst host 10.10.10.5` | One direction |
| `net 10.10.10.0/24` | A subnet |
| `port 443` / `portrange 8000-8100` | TCP or UDP port(s) |
| `src port 53` / `dst port 53` | One direction |
| `tcp` / `udp` / `icmp` / `arp` / `ip6` | Protocol |
| `tcp[tcpflags] & tcp-syn != 0` | TCP SYN set |
| `tcp[tcpflags] & (tcp-syn\|tcp-ack) == tcp-syn` | SYN without ACK |
| `icmp[icmptype] == icmp-echo` | ICMP echo request |
| `vlan 100` | VLAN-tagged traffic |
| `less 100` / `greater 1000` | Packet length |
| `not port 22` | Exclusion |
| `and` / `or` / `not` | Boolean composition |

Parentheses group expressions; quote the whole filter in the shell.

## Recipes

Authorized, in-scope traffic only.

**1. Capture all traffic on one interface to a file.**

```bash
sudo tcpdump -i eth0 -s 0 -w ~/captures/all.pcap
```

**2. Capture one host and port, no DNS noise.**

```bash
sudo tcpdump -i any -nn -s 0 'host 10.10.10.5 and tcp port 443' -w ~/captures/tls.pcap
```

**3. Read a capture and show only TCP SYNs.**

```bash
tcpdump -nr ~/captures/all.pcap 'tcp[tcpflags] & tcp-syn != 0'
```

**4. Print HTTP requests in ASCII.**

```bash
sudo tcpdump -i any -nn -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'
```

**5. Ring buffer: 10 files of 100 MB, rotating every 10 minutes.**

```bash
sudo tcpdump -i eth0 -s 0 -C 100 -G 600 -W 10 -w ~/captures/ring-%Y%m%d-%H%M%S.pcap
```

**6. Remote capture piped to local Wireshark.**

```bash
ssh user@host 'sudo tcpdump -i eth0 -U -s 0 -w - "tcp port 443"' | wireshark -k -i -
```

**7. Capture only DNS queries and responses.**

```bash
sudo tcpdump -i any -nn 'udp port 53'
```

**8. Check whether a scan actually left the host.**

```bash
sudo tcpdump -i any -nn 'host 10.10.10.5 and tcp[tcpflags] & tcp-syn != 0' -c 20
```

## Output & pipelines

- A typical line is `HH:MM:SS.ffffff IP src.port > dst.port: Flags [S], seq ..., win ..., length 0`.
- `-w` writes binary pcap; `-r` reads it. Pipe `-w -` to another process for
  live remote capture.
- `-l` is required for line-buffered output through `grep`/`awk`; otherwise
  stdout is block-buffered.
- `-A`/`-X` expose payloads but not reassembled streams. For stream reassembly,
  use `tshark -z follow,tcp,ascii,0` or Wireshark's *Follow TCP Stream*.
- Timestamps: `-tttt` for human reports, `-tt` for epoch math, `--nano` for
  high-resolution timing.

## Gotchas & operational security

- **Promiscuous mode is the default.** `-p` keeps the interface non-promiscuous;
  some virtual interfaces ignore the distinction.
- **`-s 0` captures full packets and grows files fast.** Use a snaplen (e.g.
  `-s 128`) when only headers matter.
- **`-i any` uses cooked capture.** Link-layer headers may be missing or
  different; use a specific interface when you need Ethernet headers.
- **Dropped packets are silent.** Check the "packets dropped by kernel" line at
  exit; raise `-B` or reduce the filter scope if it is non-zero.
- **BPF is compiled before capture.** A syntax error stops the capture; test
  with `-d` or a short `-c`.
- **Privilege drop matters.** `-Z` should be set on shared hosts; capturing as
  root for a long ring buffer is unnecessary risk.
- **Payload capture can record credentials and personal data.** Treat pcaps as
  sensitive evidence; encrypt at rest and delete when the engagement ends.
- **Time zones and clock skew** make cross-host correlation wrong. Record the
  capture host's clock and use `-tt`/`--nano` consistently.

## Legal & safety

Capture only traffic on networks you own or have **explicit written
authorization** to monitor. Packet capture can record credentials, personal
data, and third-party communications; unauthorized interception is illegal in
most jurisdictions. Minimize scope, secure the files, and do not capture more
than the engagement requires.

## Related tools

- [`wireshark/`](../wireshark/README.md) — GUI/tshark dissection of tcpdump
  captures; `dumpcap` is the capture engine.
- [`nmap/`](../nmap/README.md) — generate traffic worth capturing.
- [`masscan/`](../masscan/README.md) — high-rate scans; verify with tcpdump.
- [`mitmproxy/`](../mitmproxy/README.md) — scriptable HTTP interception and replay.
- [`sshuttle/`](../sshuttle/README.md) — VPN-style pivoting when a capture point is remote.

## References

- Installed help snapshot: [`help.txt`](help.txt)
- tcpdump man page: https://www.tcpdump.org/manpages/tcpdump.1.html
- pcap-filter man page: https://www.tcpdump.org/manpages/pcap-filter.7.html
- Upstream source: https://github.com/the-tcpdump-group/tcpdump
