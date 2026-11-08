# Wireshark / tshark / dumpcap — packet capture & analysis

- **Version:** Wireshark 4.4.19
- **Binaries:** `wireshark` (GUI), `tshark` (CLI), `dumpcap` (capture engine)
- **Full help snapshots:** [`help-dumpcap.txt`](help-dumpcap.txt), [`help-tshark.txt`](help-tshark.txt)
- **Docs:** https://www.wireshark.org/docs/
- **Display filter reference:** https://www.wireshark.org/docs/dfref/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Wireshark captures and dissects network traffic. `dumpcap` captures packets
efficiently; `tshark` gives the same dissectors a scriptable CLI; the GUI adds
visual analysis, follow-stream, and statistics.

On this machine `/usr/bin/dumpcap` has `cap_net_admin,cap_net_raw=eip` and is
group-readable by `wireshark`; your user must be a member.

## Use cases

| Situation | Tool |
|-----------|------|
| Live interactive analysis | `wireshark` |
| Scripted capture | `dumpcap -i ... -w file.pcapng` |
| Extract HTTP objects/files | `tshark --export-objects` |
| Follow a TCP stream | GUI *Follow → TCP Stream* or `tshark -z follow,tcp,ascii,0` |
| Debug TLS handshakes | `tshark -Y 'tls.handshake'` |
| Check DNS | `tshark -Y dns` |
| Find retransmits/latency | `tshark -z expert` / `-z io,stat` |
| Credential/plaintext audit | filter `http.authorization`, `ftp`, `telnet` |
| Capture from remote host | `ssh host tcpdump -U -w - ... \| wireshark -k -i -` |

## Permissions

```bash
id | grep wireshark          # must be a member
getcap /usr/bin/dumpcap      # cap_net_admin,cap_net_raw=eip
```

If capture is denied, log out/in after `usermod -aG wireshark $USER`, or use
`newgrp wireshark`.

## dumpcap basics

```bash
dumpcap -D                          # list interfaces
dumpcap -i wlp2s0 -w cap.pcapng     # capture all traffic
dumpcap -i wlp2s0 -f "tcp port 443" -a duration:60 -w tls.pcapng
dumpcap -i any -b filesize:102400 -b files:10 -w ring.pcapng
```

| Flag | Meaning |
|------|---------|
| `-i <iface>` | Interface (`any` for all, `-D` to list) |
| `-f <bpf>` | **Capture** filter (libpcap syntax) |
| `-s <len>` | Snaplen |
| `-p` | No promiscuous mode |
| `-I` | Monitor mode (Wi-Fi) |
| `-B <MiB>` | Kernel buffer |
| `-y <linktype>` | Link-layer type |
| `-c <n>` | Stop after N packets |
| `-a duration:N`, `-a filesize:N`, `-a files:N`, `-a packets:N` | Autostop |
| `-b duration/filesize/files/packets:N` | Ring buffer |
| `-w <file>` | Output file |
| `-g` | Group-readable output |
| `-S` | Per-interface stats each second |
| `-M` | Machine-readable `-D/-L/-S` |

## tshark basics

```bash
tshark -D                                   # interfaces
tshark -i wlp2s0                            # live summary
tshark -i wlp2s0 -f "tcp port 80" -Y "http.request" 
tshark -r cap.pcapng -Y "http.request" -T fields -e http.host -e http.request.uri
tshark -r cap.pcapng -q -z io,stat,1
tshark -r cap.pcapng -q -z follow,tcp,ascii,0
tshark -r cap.pcapng --export-objects http,outdir
```

### Key options

| Flag | Meaning |
|------|---------|
| `-i <iface>` | Capture interface |
| `-f <bpf>` | Capture filter |
| `-Y <filter>` | **Display** filter |
| `-r <file>` | Read pcap |
| `-w <file>` | Write pcap |
| `-c <n>` | Packet count |
| `-a <cond>` | Autostop |
| `-b <cond>` | Ring buffer |
| `-s <len>` | Snaplen |
| `-p` | No promiscuous |
| `-I` | Monitor mode |
| `-n` | Disable name resolution |
| `-N <flags>` | Name resolution flags (`mNd`) |
| `-2` | Two-pass analysis |
| `-R <filter>` | Read filter (needs `-2`) |
| `-Y <filter>` | Display filter |
| `-V` | Full packet detail |
| `-x` | Hex dump |
| `-T fields -e <field>` | Extract fields |
| `-E separator=,` / `header=y` | Field output formatting |
| `-q` | Suppress packet output (for `-z`) |
| `-z <stat>` | Statistics (`io,stat`, `expert`, `follow,...`, `http,tree`, `dns,tree`) |
| `--export-objects <proto>,<dir>` | Extract files (http, smb, tftp, imf) |
| `-o <pref:value>` | Override preference, e.g. `tls.keylog_file:...` |
| `-Y` filters | See below |

## Capture vs display filters

**Capture filters** (BPF, applied by kernel, faster, less expressive):

```
host 10.0.0.5
net 10.0.0.0/24
port 443
tcp port 80 or tcp port 443
udp and not port 53
src host 10.0.0.5 and dst port 8080
```

**Display filters** (Wireshark syntax, applied after capture, very expressive):

```
ip.addr == 10.0.0.5
tcp.port == 443
http.request.method == "POST"
http.host contains "example"
http.response.code >= 400
dns.qry.name contains "example.com"
tls.handshake.type == 1
tcp.flags.syn == 1 && tcp.flags.ack == 0
tcp.analysis.retransmission
tcp.stream eq 3
frame contains "password"
http.authorization
ftp.request.command == "USER"
!(arp or icmp)
```

Common field names:

| Layer | Fields |
|-------|--------|
| IP | `ip.src`, `ip.dst`, `ip.addr`, `ip.ttl` |
| TCP | `tcp.srcport`, `tcp.dstport`, `tcp.flags`, `tcp.stream`, `tcp.analysis.*` |
| UDP | `udp.srcport`, `udp.dstport`, `udp.length` |
| HTTP | `http.request.method`, `http.host`, `http.request.uri`, `http.response.code`, `http.user_agent`, `http.cookie`, `http.authorization` |
| DNS | `dns.qry.name`, `dns.a`, `dns.flags.response` |
| TLS | `tls.handshake.type`, `tls.handshake.extensions_server_name`, `tls.record.version` |
| ARP | `arp.opcode`, `arp.src.proto_ipv4` |

## Recipes

```bash
# Capture 60s of all traffic on Wi-Fi
dumpcap -i wlp2s0 -a duration:60 -w /tmp/cap.pcapng

# Capture only HTTP/HTTPS
dumpcap -i wlp2s0 -f "tcp port 80 or tcp port 443" -w /tmp/web.pcapng

# Ring buffer: 10 files x 100 MB
dumpcap -i any -b filesize:102400 -b files:10 -w /tmp/ring.pcapng

# Live HTTP requests with hosts and URIs
tshark -i wlp2s0 -Y "http.request" -T fields -e http.host -e http.request.uri

# Extract all HTTP objects from a capture
mkdir -p /tmp/http-objects
tshark -r cap.pcapng --export-objects http,/tmp/http-objects

# Follow TCP stream 0 as ASCII
tshark -r cap.pcapng -q -z follow,tcp,ascii,0

# Top talkers
tshark -r cap.pcapng -q -z endpoints,ip

# Protocol hierarchy
tshark -r cap.pcapng -q -z io,phs

# Expert info (retransmits, resets, errors)
tshark -r cap.pcapng -q -z expert

# DNS queries
tshark -r cap.pcapng -Y "dns.flags.response == 0" -T fields -e dns.qry.name | sort -u

# TLS SNI values
tshark -r cap.pcapng -Y "tls.handshake.extensions_server_name" \
  -T fields -e tls.handshake.extensions_server_name | sort -u

# HTTP credentials (plaintext/basic)
tshark -r cap.pcapng -Y "http.authorization" -T fields -e http.host -e http.authorization

# Decrypt TLS with a keylog file
tshark -r cap.pcapng -o tls.keylog_file:/tmp/sslkeys.log -Y "http.request"

# Remote capture piped into local Wireshark
ssh user@host 'sudo tcpdump -U -i any -w - not port 22' | wireshark -k -i -
```

## TLS decryption

Set `SSLKEYLOGFILE` in the browser before launching:

```bash
export SSLKEYLOGFILE=/tmp/sslkeys.log
firefox &
tshark -r cap.pcapng -o tls.keylog_file:/tmp/sslkeys.log -Y "http.request"
```

Wireshark GUI: *Preferences → Protocols → TLS → (Pre)-Master-Secret log filename*.
This only works when the client supports key logging (Firefox/Chromium do) and
you captured the handshake.

## Gotchas

- Capture filters use **BPF** syntax; display filters use **Wireshark** syntax.
  Mixing them is the most common error.
- Promiscuous mode sees other hosts only on a hub/mirror/AP you control.
- Monitor mode requires supported Wi-Fi hardware and `-I`.
- Without the TLS keylog or a MITM proxy, HTTPS payloads are opaque.
- `dumpcap` is the safe capture engine; `tshark` also captures but with more overhead.
- Large captures: use ring buffers and snaplen limits.
- On Wi-Fi, captures may include 802.11 headers; set the link type accordingly.

## References

- Wireshark docs: https://www.wireshark.org/docs/
- tshark man page: https://www.wireshark.org/docs/man-pages/tshark.html
- dumpcap man page: https://www.wireshark.org/docs/man-pages/dumpcap.html
- Display filter reference: https://www.wireshark.org/docs/dfref/
- Capture filter syntax: https://www.tcpdump.org/manpages/pcap-filter.7.html
