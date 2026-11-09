# dsniff — password sniffer and layer-2 attack suite

- **Version:** dsniff 2.4 (`dsniff -h` prints `Version: 2.4`); Debian package
  `dsniff` version `2.4b1+debian-34`
- **Binaries:** `/usr/sbin/dsniff`, `arpspoof`, `dnsspoof`, `macof`, `tcpkill`,
  `tcpnice`, `urlsnarf`, `filesnarf`, `mailsnarf`, `msgsnarf`, `sshmitm`,
  `webmitm`, `webspy`, `sshow`
- **Help snapshots:** [`help.txt`](help.txt) — `dsniff -h`;
  [`help-tools.txt`](help-tools.txt) — usage of the other 13 tools
- **Upstream:** Dug Song's dsniff (historical:
  https://www.monkey.org/~dugsong/dsniff/); Debian source package `dsniff`
- **Docs:** `man dsniff`, `man arpspoof`, `man dnsspoof`, `man macof`,
  `man tcpkill`, `man tcpnice`, `man urlsnarf`, `man filesnarf`,
  `man mailsnarf`, `man msgsnarf`, `man sshmitm`, `man webmitm`,
  `man webspy`, `man sshow`
- **Data files:** `/usr/share/dsniff/dsniff.services`,
  `/usr/share/dsniff/dsniff.magic`, `/usr/share/dsniff/dnsspoof.hosts`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

`dsniff` is a passive password sniffer. It reassembles TCP streams with
libnids, detects application protocols, extracts only the authentication
material, and stores unique attempts in a Berkeley DB file. It understands
FTP, Telnet, SMTP, HTTP, POP, poppass, NNTP, IMAP, SNMP, LDAP, Rlogin, RIP,
OSPF, PPTP MS-CHAP, NFS, VRRP, YP/NIS, SOCKS, X11, CVS, IRC, AIM, ICQ,
Napster, PostgreSQL, Meeting Maker, Citrix ICA, pcAnywhere, Microsoft SMB,
Oracle SQL*Net, Sybase, and Microsoft SQL.

The package is really a **suite**: one sniffer plus tools that redirect,
forge, throttle, or extract specific traffic.

| Tool | Purpose |
|------|---------|
| `dsniff` | Passive password sniffer with protocol detection |
| `arpspoof` | Forge ARP replies to intercept traffic on a switched LAN |
| `dnsspoof` | Forge DNS address/PTR replies |
| `macof` | Flood a switch with random MAC addresses |
| `tcpkill` | Kill selected in-progress TCP connections |
| `tcpnice` | Slow selected TCP connections with active shaping |
| `urlsnarf` | Print sniffed HTTP URLs in Common Log Format |
| `filesnarf` | Save files sniffed from NFS traffic |
| `mailsnarf` | Save SMTP/POP messages in Berkeley mbox format |
| `msgsnarf` | Record AIM, ICQ, IRC, MSN, and Yahoo chat messages |
| `sshmitm` | SSHv1 monkey-in-the-middle |
| `webmitm` | HTTP/HTTPS monkey-in-the-middle |
| `webspy` | Send sniffed URLs to a local Netscape browser |
| `sshow` | Analyze encrypted SSH traffic (lengths, not contents) |

## `dsniff`

```text
dsniff [-c] [-d] [-m] [-n] [-i interface | -p pcapfile] [-s snaplen]
       [-f services] [-t trigger[,...]] [-r|-w savefile] [expression]
```

| Flag | Meaning |
|------|---------|
| `-c` | Half-duplex TCP stream reassembly for asymmetric routes |
| `-d` | Debugging mode |
| `-m` | Automatic protocol detection |
| `-n` | Do not resolve IP addresses to hostnames |
| `-i IFACE` | Listen on this interface |
| `-p PCAP` | Process a pcap file instead of the network |
| `-s SNAPLEN` | Analyze at most this many bytes per connection (default 1024) |
| `-f SERVICES` | Load triggers from a services file |
| `-t TRIGGER[,...]` | Triggers as `port/proto=service`, e.g. `80/tcp=http` |
| `-r SAVEFILE` | Read sessions from a `-w` savefile |
| `-w FILE` | Write sessions to a savefile instead of printing |
| `expression` | tcpdump-style BPF filter |

On `SIGHUP`, dsniff dumps its current trigger table to `dsniff.services`.

## `arpspoof`

```text
arpspoof [-i interface] [-c own|host|both] [-t target] [-r] host
```

| Flag | Meaning |
|------|---------|
| `-i IFACE` | Interface to use |
| `-c own\|host\|both` | Hardware address used while restoring the ARP cache |
| `-t TARGET` | Poison only this host; repeat for multiple targets |
| `-r` | Poison both host and target (requires `-t`) |
| `host` | The host whose traffic you want to intercept (usually the gateway) |

Kernel IP forwarding (or a userland forwarder such as fragrouter) must be
enabled **before** you start. `-c host` restores with the host's address; the
manual notes this can disrupt some switch/AP/bridge configurations, while the
default `own` address is more compatible but sometimes less reliable.

## `dnsspoof`

```text
dnsspoof [-i interface] [-f hostsfile] [expression]
```

| Flag | Meaning |
|------|---------|
| `-i IFACE` | Interface to use |
| `-f HOSTSFILE` | hosts(5)-format file; one hostname per line, wildcards allowed |
| `expression` | BPF filter |

With no hosts file, dnsspoof forges an answer for **every** address query on
the LAN, pointing at the local machine. Hostnames may contain wildcards such
as `*.example.com`; aliases are not supported.

## `macof`

```text
macof [-s src] [-d dst] [-e tha] [-x sport] [-y dport] [-i interface] [-n times]
```

| Flag | Meaning |
|------|---------|
| `-i IFACE` | Interface to send on |
| `-s SRC` | Source IP address |
| `-d DST` | Destination IP address |
| `-e THA` | Target hardware address |
| `-x SPORT` | TCP source port |
| `-y DPORT` | TCP destination port |
| `-n TIMES` | Number of packets to send |

Unspecified values are generated randomly. The goal is to overflow a switch's
MAC table so it fails open in repeating mode.

## `tcpkill` and `tcpnice`

```text
tcpkill [-i interface] [-1..9] expression
tcpnice [-A] [-I] [-M] [-i interface] expression
```

| Tool | Flag | Meaning |
|------|------|---------|
| `tcpkill` | `-i IFACE` | Interface to listen on |
| `tcpkill` | `-1..9` | Brute-force degree for landing a RST (default 3) |
| `tcpkill` | `expression` | BPF filter selecting connections to kill |
| `tcpnice` | `-A` | Inject tiny TCP window advertisements |
| `tcpnice` | `-I` | Inject ICMP source-quench replies |
| `tcpnice` | `-M` | Inject ICMP fragmentation-needed with tiny MTUs |
| `tcpnice` | `-i IFACE` | Interface to listen on |
| `tcpnice` | `expression` | BPF filter selecting connections to slow |

If no `tcpnice` attack type is given, all three are enabled.

## `urlsnarf`, `filesnarf`, `mailsnarf`, `msgsnarf`

```text
urlsnarf   [-n] [-i interface | -p pcapfile] [[-v] pattern [expression]]
filesnarf  [-i interface | -p pcapfile] [[-v] pattern [expression]]
mailsnarf  [-i interface | -p pcapfile] [[-v] pattern [expression]]
msgsnarf   [-i interface | -p pcapfile] [[-v] pattern [expression]]
```

| Tool | Output | Notes |
|------|--------|-------|
| `urlsnarf` | Common Log Format | `-n` skips reverse DNS |
| `filesnarf` | Files written to the **current directory** | NFS traffic only |
| `mailsnarf` | Berkeley mbox | SMTP and POP |
| `msgsnarf` | Chat transcript | AIM, ICQ 2000, IRC, MSN, Yahoo |

All four accept `-v` to invert the pattern match and a BPF `expression`.

## `sshmitm`, `webmitm`, `webspy`, `sshow`

```text
sshmitm [-d] [-I] [-p port] host [port]
webmitm [-d] [host]
webspy  [-i interface | -p pcapfile] host
sshow   [-d] [-i interface | -p pcapfile] [expression]
```

| Tool | Flag | Meaning |
|------|------|---------|
| `sshmitm` | `-d` | Verbose debugging |
| `sshmitm` | `-I` | Monitor or hijack an interactive session |
| `sshmitm` | `-p PORT` | Local listen port |
| `sshmitm` | `host [port]` | Remote host and port to relay to |
| `webmitm` | `-d` | Debugging (repeatable) |
| `webmitm` | `host` | Host to proxy to |
| `webspy` | `-i`, `-p` | Live interface or pcap |
| `webspy` | `host` | Web client to spy on |
| `sshow` | `-d` | Verbose debugging |
| `sshow` | `-i`, `-p` | Live interface or pcap |

`sshmitm` supports **SSH protocol version 1 only**. `webmitm` uses
`webmitm.crt` in the working directory as its certificate. `webspy` expects a
Netscape browser on the local X display and is effectively historical.

## Recipes

**1. Sniff credentials on an interface.**

```bash
sudo dsniff -i eth0
```

**2. Sniff a pcap offline.**

```bash
dsniff -p capture.pcap
```

**3. Restrict to HTTP and FTP.**

```bash
sudo dsniff -i eth0 'tcp port 80 or tcp port 21'
```

**4. Save sessions to a Berkeley DB instead of printing.**

```bash
sudo dsniff -i eth0 -w sessions.db
dsniff -r sessions.db
```

**5. Add a custom trigger.**

```bash
sudo dsniff -i eth0 -t 8080/tcp=http -t 3306/tcp=mysql
```

**6. ARP-poison a client through the gateway.**

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo arpspoof -i eth0 -t 192.0.2.50 192.0.2.1
```

**7. Bidirectional ARP poisoning.**

```bash
sudo arpspoof -i eth0 -t 192.0.2.50 -r 192.0.2.1
```

**8. Forge DNS answers from a hosts file.**

```bash
sudo dnsspoof -i eth0 -f /tmp/spoof.hosts
```

**9. Forge every DNS answer to the local host.**

```bash
sudo dnsspoof -i eth0
```

**10. Flood the switch MAC table.**

```bash
sudo macof -i eth0 -n 100000
```

**11. Kill a specific TCP connection.**

```bash
sudo tcpkill -i eth0 -9 'host 192.0.2.50 and port 443'
```

**12. Slow a connection instead of killing it.**

```bash
sudo tcpnice -i eth0 -A 'host 192.0.2.50 and port 80'
```

**13. Snarf URLs and mail.**

```bash
sudo urlsnarf -i eth0
sudo mailsnarf -i eth0
```

**14. Replay a capture through the snarfers.**

```bash
urlsnarf -p capture.pcap
filesnarf -p capture.pcap
mailsnarf -p capture.pcap
msgsnarf -p capture.pcap
```

**15. SSHv1 MITM relay.**

```bash
sudo sshmitm -d -p 2222 192.0.2.50 22
```

**16. HTTP/HTTPS MITM relay.**

```bash
sudo webmitm -d 192.0.2.50
```

**17. SSH traffic analysis.**

```bash
sudo sshow -i eth0 'tcp port 22'
```

## Gotchas

- **The suite is old and unmaintained.** dsniff 2.4b1 predates TLS 1.3, modern
  HTTP/2, and most current protocols. It will not parse what it does not know.
- **`dsniff` only sees cleartext.** It recovers credentials from protocols that
  send them in the clear. TLS, SSHv2, and modern auth flows are opaque to it.
- **It logs unique authentication attempts only.** Berkeley DB deduplication
  means repeated identical logins may not appear twice.
- **Root or `CAP_NET_RAW` is required.** Every tool opens raw or monitor-mode
  sockets.
- **`arpspoof` needs forwarding already enabled.** It poisons caches; it does
  not forward. Without `net.ipv4.ip_forward=1`, victims lose connectivity.
- **`arpspoof -r` requires `-t`.** The manual says it is only valid with a
  target.
- **`arpspoof` cleanup can disrupt the network.** `-c host` is more reliable in
  some topologies but can break switch/AP/bridge behaviour; the default `own`
  is safer but sometimes fails to restore.
- **`dnsspoof` without `-f` answers everything.** That is a blunt instrument;
  use a hosts file for targeted spoofing.
- **`dnsspoof` hosts files allow wildcards but not aliases.** One hostname per
  line.
- **`macof` is a denial-of-service primitive.** Flooding a switch can disrupt
  the entire segment; use it only in an authorized resilience test.
- **`tcpkill` must see the handshake.** It relies on libnids TCB creation, so
  it cannot kill a connection whose setup it did not observe.
- **`tcpkill -9` is not subtle.** Higher brute force sends more RSTs and is
  more likely to be logged.
- **`tcpnice` changes throughput, not content.** It is traffic shaping, not
  interception.
- **`filesnarf` writes into the current directory.** Run it in a dedicated
  evidence directory or you will scatter captured files through your home.
- **`mailsnarf` produces mbox.** Treat the output as sensitive mail content.
- **`sshmitm` is SSHv1-only.** It cannot intercept modern SSHv2 sessions.
- **`webmitm` uses a static certificate file.** Clients must accept
  `webmitm.crt`; certificate warnings are the expected signal.
- **`webspy` needs Netscape.** It is included for completeness, not for modern
  use.
- **`sshow` does not decrypt SSH.** It reports authentication attempts,
  password lengths, and command-line lengths from traffic analysis.
- **`-p` and `-i` are alternatives.** Do not pass both to the snarfers.
- **BPF expressions are your friend.** Without a filter, dsniff processes
  everything it can reassemble.

## Related docs

- [`bettercap`](../bettercap/README.md) — modern replacement for most of this
  suite
- [`ettercap`](../ettercap/README.md) — MITM with plugins and content filters
- [`wireshark`](../wireshark/README.md) — decode the captures these tools make
- [`tcpdump`](../tcpdump/README.md) — capture with the same BPF syntax
- [`mitmproxy`](../mitmproxy/README.md) — modern HTTP/TLS interception
- [`nmap`](../nmap/README.md) — discover targets before poisoning
- [`nbtscan`](../nbtscan/README.md) — NetBIOS discovery for SMB targets
- [`smbclient`](../smbclient/README.md) — use captured SMB credentials
