# tcpdump — command scenarios for deep research

Installed build: tcpdump 4.99.5, libpcap 1.10.5. Flags come from
[`help.txt`](help.txt). Live capture needs root or `CAP_NET_RAW`; `-r` does not.

> **Sensitive:** captures can contain credentials and personal data. Use only
> on authorized networks and minimize scope.

## How to invoke

```text
@deep-research Which BPF filter and snaplen give the smallest capture that
still proves a specific network behavior (scan, retransmit, TLS handshake,
cleartext credential), and how should captures be rotated and verified for
loss? Cite the pcap-filter man page, tcpdump README, and packet-capture field
practice; return a capture plan with exact filters and a verification step.
```

Prepare the demo directory:

```bash
mkdir -p ~/tcpdump-demo/captures
```

## Scenario 1 — prove what a scanner sent

Capture on the scanning host or a span port, then compare the packets with the
scanner's own output. This is the fastest way to settle "did the probe leave?"

| # | Command | How to use |
|---|---------|------------|
| 1 | `sudo tcpdump -i any -nn 'host 10.10.10.5 and tcp[tcpflags] & tcp-syn != 0' -c 20` | First 20 SYNs to one host |
| 2 | `sudo tcpdump -i eth0 -nn -s 0 'host 10.10.10.5' -w ~/tcpdump-demo/captures/host.pcap` | Full-packet capture |
| 3 | `sudo tcpdump -i any -nn 'tcp port 80 or tcp port 443' -c 100` | Web traffic only |
| 4 | `tcpdump -nr ~/tcpdump-demo/captures/host.pcap 'tcp[tcpflags] & tcp-rst != 0'` | Find resets |
| 5 | `tcpdump -nr ~/tcpdump-demo/captures/host.pcap -tttt -nn` | Human-readable timestamps |

**Research questions**
- Do the captured SYNs match nmap/masscan's reported probe count?
- Which destinations answer with RST versus silence?
- Does the interface see both directions, or only egress?

**Sources:** pcap-filter man page; scanner output; `ip -s link`.
**Output:** a pcap plus a packet-count reconciliation.

## Scenario 2 — connectivity and retransmission diagnosis

Look at the TCP handshake before blaming the application. SYN retransmits,
RSTs, and zero-window advertisements each point at a different layer.

| # | Command | How to use |
|---|---------|------------|
| 6 | `sudo tcpdump -i any -nn 'host 10.10.10.5 and port 443'` | Live handshake view |
| 7 | `sudo tcpdump -i any -nn -v 'tcp port 443'` | Verbose flags/window |
| 8 | `tcpdump -nr capture.pcap 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack == 0'` | Initial SYNs only |
| 9 | `tcpdump -nr capture.pcap -S -nn 'tcp port 443'` | Absolute sequence numbers |
| 10 | `tcpdump -nr capture.pcap 'tcp[13] & 0x12 == 0x12'` | SYN-ACK packets |

**Research questions**
- How many SYN retransmits occur before a connection succeeds or fails?
- Is the RST from the target, a middlebox, or the local stack?
- Does the window size suggest a bottleneck or a receiver problem?

**Sources:** RFC 9293; tcpdump TCP flag decoding; `ss -ti`.
**Output:** a handshake timeline with retransmit count and RST source.

## Scenario 3 — cleartext protocol audit

On an authorized segment, capture only the protocols that should never carry
credentials in clear text. This is evidence for a finding, not a fishing trip.

| # | Command | How to use |
|---|---------|------------|
| 11 | `sudo tcpdump -i any -nn -A 'tcp port 21'` | FTP commands and replies |
| 12 | `sudo tcpdump -i any -nn -A 'tcp port 23'` | Telnet |
| 13 | `sudo tcpdump -i any -nn -A 'tcp port 80'` | HTTP headers/body |
| 14 | `sudo tcpdump -i any -nn -A 'udp port 161'` | SNMP community strings |
| 15 | `sudo tcpdump -i any -nn -X 'tcp port 389'` | LDAP simple bind (hex/ASCII) |

**Research questions**
- Which protocols expose credentials or session tokens in the clear?
- Is the traffic internal-only or crossing a trust boundary?
- Does TLS termination happen before or after the capture point?

**Sources:** protocol RFCs; the capture itself; TLS termination diagram.
**Output:** redacted evidence lines and a protocol/credential table.

## Scenario 4 — durable, rotated captures

Long captures need rotation and a loss check. A full disk or a dropped-packet
counter invalidates the evidence.

| # | Command | How to use |
|---|---------|------------|
| 16 | `sudo tcpdump -i eth0 -s 0 -C 100 -W 10 -w ~/tcpdump-demo/captures/ring.pcap` | 10 × 100 MB ring |
| 17 | `sudo tcpdump -i eth0 -s 0 -G 600 -W 24 -w ~/tcpdump-demo/captures/time-%Y%m%d-%H%M%S.pcap` | Hourly rotation for a day |
| 18 | `sudo tcpdump -i eth0 -s 0 -B 4096 -w ~/tcpdump-demo/captures/big.pcap` | Larger capture buffer |
| 19 | `sudo tcpdump -i eth0 -Z tcpdump -w ~/tcpdump-demo/captures/drop.pcap` | Drop privileges after open |
| 20 | `tcpdump -nr ~/tcpdump-demo/captures/big.pcap -c 1 >/dev/null; echo $?` | Validate the file opens |

**Research questions**
- How many packets are dropped at the default buffer size?
- Does `-C`/`-G`/`-W` rotate before the disk fills?
- Is `-Z` effective on this build, and which user does it use?

**Sources:** tcpdump man page; `df -h`; exit statistics.
**Output:** a rotation schedule and a zero-drop verification.

## Scenario 5 — remote capture and handoff

Capture on a remote host and analyze locally. `-U` and `-w -` are the key flags
for streaming.

| # | Command | How to use |
|---|---------|------------|
| 21 | `ssh user@host 'sudo tcpdump -i eth0 -U -s 0 -w - "tcp port 443"' > ~/tcpdump-demo/captures/remote.pcap` | Stream to a local file |
| 22 | `ssh user@host 'sudo tcpdump -i eth0 -U -s 0 -w - "tcp port 443"' \| wireshark -k -i -` | Stream to Wireshark |
| 23 | `tcpdump -nr ~/tcpdump-demo/captures/remote.pcap -nn -A 'tcp port 80'` | Local ASCII review |
| 24 | `tshark -r ~/tcpdump-demo/captures/remote.pcap -z follow,tcp,ascii,0` | Reassemble a stream |
| 25 | `tcpdump -nr ~/tcpdump-demo/captures/remote.pcap -w - 'tcp port 443' \| tcpdump -nn -r -` | Re-filter a capture |

**Research questions**
- Does `-U` prevent missing packets at the start of a stream?
- Is the remote capture seeing both directions?
- Which tool preserves the most evidence for the report?

**Sources:** tcpdump man page; Wireshark/tshark docs.
**Output:** a local pcap and a stream-reassembly note.
