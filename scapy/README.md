# scapy — interactive packet manipulation

- **Version:** 2.6.1+dfsg-1 (Python 3)
- **Binary:** `/usr/bin/scapy` (CLI wrapper) and the `scapy` Python package
- **Help snapshot:** [`help.txt`](help.txt) — `scapy -h`
- **Upstream:** https://scapy.net/ · https://github.com/secdev/scapy
- **API docs:** https://scapy.readthedocs.io/
- **Licence:** GPL-2.0

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Scapy is a Python library and interactive shell for building, sending,
sniffing, dissecting, and rewriting network packets. Unlike `tcpdump` or
`tshark`, it does not have a fixed protocol vocabulary: you compose layers in
Python, and Scapy fills in fields, computes checksums, and decodes replies.

For assessments it is the glue between capture and custom logic — ARP scans,
DHCP starvation, 802.11 beacon crafting, DNS fuzzing, TLS record inspection,
pcap rewriting, and protocol-specific probes that no packaged tool covers.

## CLI versus API

The `scapy` command is an interactive Python shell with Scapy pre-imported.
Almost all real work happens in a script:

```python
from scapy.all import *
```

The CLI is useful for one-off `ls()`, `show()`, and `sniff()` experiments; a
script is reproducible and reviewable.

## CLI options

| Option | Meaning |
|--------|---------|
| `-h` | Help |
| `-H` | Header-less start (skip the banner) |
| `-C` | Do not read the startup file |
| `-P` | Do not read the pre-startup file |
| `-c FILE` | Use FILE as the startup file |
| `-p FILE` | Use FILE as the pre-startup file |
| `-s FILE` | Load a pickled session |

`-s` expects a pickle written by `save_session()` **inside** a Scapy session,
not a Python script. A plain `.py` file fails to unpickle.

## Core packet model

| Operation | Example |
|-----------|---------|
| Stack layers | `IP(dst="192.0.2.1")/TCP(dport=443, flags="S")` |
| Inspect fields | `pkt.show()` |
| One-line summary | `pkt.summary()` |
| Format fields | `pkt.sprintf("%IP.src% -> %IP.dst%")` |
| Reproduce as code | `pkt.command()` |
| Serialise | `bytes(pkt)` |
| Dissect | `IP(bytes(pkt))` |
| Layer test | `pkt.haslayer(TCP)`, `pkt.getlayer(TCP)` |
| Walk layers | `pkt.layers()`, `pkt.payload` |
| Copy | `pkt.copy()` |

Scapy computes checksums and lengths at serialisation time, so fields such as
`len`, `chksum`, and `src` often show `None` until the packet is built or
routed.

## Sending and receiving

| Function | Use |
|----------|-----|
| `send(pkt)` | Send at layer 3 (IP) |
| `sendp(pkt, iface=...)` | Send at layer 2 (Ethernet) |
| `sr(pkt, timeout=...)` | Send and receive; returns answered, unanswered |
| `sr1(pkt, timeout=...)` | Send and return the first reply |
| `srp(pkt, iface=...)` | Layer-2 send/receive |
| `srp1(pkt, iface=...)` | Layer-2 first reply |
| `sniff(...)` | Capture packets |
| `AsyncSniffer(...)` | Background capture with `start()` / `join()` |
| `bridge_and_sniff(...)` | Forward between two interfaces while capturing |

Raw sockets need root or `CAP_NET_RAW`. Layer-2 operations also need an
interface and often a wired or monitor-capable link.

## Capture filters and offline pcap

| Operation | Example |
|-----------|---------|
| Live capture | `sniff(iface="eth0", count=10, filter="tcp port 443")` |
| Offline read | `sniff(offline="capture.pcap")` |
| Read all | `rdpcap("capture.pcap")` |
| Stream read | `PcapReader("capture.pcap")` |
| Write list | `wrpcap("out.pcap", packets)` |
| Stream write | `PcapWriter("out.pcap", append=False)` |
| pcapng | `rdpcap` / `wrpcap` handle `.pcapng` in this build |

`filter=` uses libpcap BPF syntax and is applied in the kernel. `lfilter=` is a
Python predicate applied after capture.

## Discovery helpers

| Function | Use |
|----------|-----|
| `arping("192.0.2.0/24")` | ARP sweep; returns answered and unanswered |
| `traceroute("192.0.2.1")` | UDP/ICMP traceroute |
| `arpcache()` | Show the local ARP cache |
| `conf.route.route("192.0.2.1")` | Interface, source, gateway for a target |
| `conf.route6` | IPv6 routing table |
| `conf.iface`, `conf.ifaces` | Default and available interfaces |
| `conf.layers` | Registered layer classes |
| `conf.commands` | Registered Scapy commands |

## Protocol layers

Scapy ships over 1,500 layer classes. Commonly used ones:

| Area | Modules / classes |
|------|-------------------|
| L2 | `Ether`, `ARP`, `Dot1Q`, `LLC`, `STP` |
| L3 | `IP`, `IPv6`, `ICMP`, `ICMPv6`, `IPOption` |
| L4 | `TCP`, `UDP`, `SCTP` |
| Naming | `DNS`, `DNSQR`, `DNSRR`, `LLMNR`, `NBNS`, `mDNS` |
| Addresses | `DHCP`, `BOOTP`, `DHCPv6` |
| Web | `scapy.layers.http.HTTPRequest`, `HTTPResponse` |
| TLS | `scapy.layers.tls.all.TLS` and record/handshake sublayers |
| Wireless | `RadioTap`, `Dot11`, `Dot11Beacon`, `Dot11ProbeReq`, `EAPOL` |
| Auth | `ISAKMP`, `IKEv2`, `RADIUS` |
| File sharing | `SMB`, `NMB`, `DCERPC` |
| Voice | `RTP`, `MGCP`, `Skinny` |
| Industrial | `Modbus`, `DNP3`, `BACnet` |
| Tunnels | `GRE`, `VXLAN`, `Geneve`, `IPsec` |

Import a specific layer module when `from scapy.all import *` does not pull it
in:

```python
from scapy.layers.http import HTTPRequest
from scapy.layers.dot11 import RadioTap, Dot11, Dot11Beacon
from scapy.layers.tls.all import TLS
```

## Utilities

| Utility | Use |
|---------|-----|
| `ls()` | List all layer classes |
| `ls(IP)` | Show IP fields |
| `lsc()` | List Scapy commands |
| `explore()` | Browse layers interactively |
| `fuzz(pkt)` | Randomise fields for fuzzing |
| `show()` / `show2()` | Show fields / show fields with computed values |
| `hexdump(bytes(pkt))` | Hex dump |
| `wireshark(pkt)` | Open a packet in Wireshark (needs a display) |

`ls(IP)` prints and returns `None`; capture stdout if you need to parse it.

## Recipes

**1. Build and inspect a TCP SYN.**

```python
from scapy.all import *
p = IP(dst="192.0.2.1")/TCP(dport=443, flags="S")
p.show()
print(p.summary())
print(p.command())
```

**2. Send a probe and wait for a reply.**

```python
reply = sr1(IP(dst="192.0.2.1")/ICMP(), timeout=2, verbose=0)
print(reply.summary() if reply else "no reply")
```

**3. Capture 10 packets with a BPF filter.**

```python
pkts = sniff(iface="eth0", count=10, filter="tcp port 443")
print(len(pkts), pkts[0].summary())
```

**4. Background capture around an action.**

```python
s = AsyncSniffer(iface="eth0", filter="arp", store=True)
s.start()
sendp(Ether(dst="ff:ff:ff:ff:ff:ff")/ARP(pdst="192.0.2.1"), iface="eth0")
s.join()
print(len(s.results))
```

**5. Read, filter, and rewrite a pcap.**

```python
pkts = rdpcap("in.pcap")
syn = [p for p in pkts if p.haslayer(TCP) and p[TCP].flags == "S"]
wrpcap("syn.pcap", syn)
```

**6. Stream a large pcap without loading it all.**

```python
with PcapReader("big.pcap") as reader, PcapWriter("out.pcap") as writer:
    for p in reader:
        if p.haslayer(DNS):
            writer.write(p)
```

**7. ARP sweep a subnet.**

```python
answered, unanswered = arping("192.0.2.0/24", timeout=2)
for sent, received in answered:
    print(received.psrc, received.hwsrc)
```

**8. Route lookup for a target.**

```python
iface, src, gateway = conf.route.route("192.0.2.1")
print(iface, src, gateway)
```

**9. Craft a DNS query.**

```python
q = IP(dst="192.0.2.53")/UDP(dport=53)/DNS(qd=DNSQR(qname="example.com"))
reply = sr1(q, timeout=2)
reply[DNS].an.show() if reply and reply.haslayer(DNS) else None
```

**10. Craft an 802.11 beacon.**

```python
beacon = RadioTap()/Dot11(type=0, subtype=8, addr1="ff:ff:ff:ff:ff:ff")/Dot11Beacon()/Dot11Elt(ID="SSID", info="audit")
beacon.show()
```

**11. Fuzz a header.**

```python
from scapy.all import fuzz
p = fuzz(IP(dst="192.0.2.1")/UDP(dport=53))
print(p.summary())
```

**12. Open a packet in Wireshark.**

```python
p = rdpcap("capture.pcap")[0]
wireshark(p)
```

## Exit status

| Situation | Exit |
|-----------|------|
| CLI starts and exits normally | 0 |
| Unknown CLI option | 1 |
| Python exception in a startup file or script | non-zero from Python |
| `send`/`sr` timeout | no exception; `sr1` returns `None` |
| Capture without privileges | `PermissionError` |

The CLI is a Python shell: most failures are Python exceptions, not Scapy exit
codes. Check the traceback, not `$?`.

## Quirks and limits

- **Raw sockets need root or `CAP_NET_RAW`.** Capture and send fail otherwise.
- **Layer-2 sends need an interface.** `sendp` without `iface` may guess wrong.
- **Fields stay `None` until built.** `len`, `chksum`, and `src` are computed
  at serialisation or route time.
- **`ls()` prints and returns `None`.** Redirect stdout to parse it.
- **`-s` is a pickle, not a script.** Only `save_session()` inside Scapy
  produces a loadable file.
- **`save_session()` needs an active Scapy session.** Calling it from plain
  Python raises `KeyError: 'scapy_session'`.
- **`sniff(offline=...)` is supported but `rdpcap` is clearer.** Both read the
  same formats.
- **BPF filters need libpcap.** `filter=` uses kernel BPF; `lfilter=` is pure
  Python and slower.
- **`fuzz()` randomises aggressively.** The result is often not routable; use
  it for parser testing, not for valid traffic.
- **PyX is missing.** `pdfdump()` and `psdump()` are unavailable; the CLI logs
  an INFO line about it.
- **Wireshark integration needs a display.** `wireshark(pkt)` fails headless.
- **TLS dissection is partial.** Encrypted records are shown, not decrypted,
  unless you supply keys.
- **Wireless injection needs the right driver.** Monitor mode and frame
  injection depend on the adapter and `iw`/`aircrack-ng` support.
- **This build is the Debian package.** It may lag upstream Scapy releases.

## Related tools

- [`tcpdump`](../tcpdump/) — fast BPF capture; feed pcaps to Scapy.
- [`wireshark`](../wireshark/) — GUI and `tshark` dissection.
- [`nmap`](../nmap/) — port and service scanning.
- [`masscan`](../masscan/) — high-speed scanning.
- [`bettercap`](../bettercap/) — interactive MITM and wireless framework.
- [`dsniff`](../dsniff/) — ARP spoofing and credential capture.
- [`mitmproxy`](../mitmproxy/) — HTTP(S) interception.
- [`hcxdumptool`](../hcxdumptool/) — raw 802.11 capture for WPA analysis.
