# scapy — command scenarios for deep research

Installed build: Scapy 2.6.1+dfsg-1 (`python3-scapy`), CLI at `/usr/bin/scapy`.
Every option and API below was checked on this host; raw-socket tests ran as
root on `lo` and the default interface.

> **Scope:** sending crafted packets, ARP sweeps, wireless injection, and
> spoofing can disrupt networks and other people's devices. Only run them on
> links and hosts you own or are explicitly authorised to test. Prefer `lo` or
> a lab VLAN for send/receive experiments.

> **Privileges:** raw sockets need root or `CAP_NET_RAW`. Without them,
> `send`, `sr`, `sniff`, `arping`, and `traceroute` raise `PermissionError`.
> Offline pcap work and packet crafting do not need privileges.

## How to invoke

```text
@deep-research For an authorised network assessment with Scapy 2.6.1, explain
the packet model (layer stacking, lazy field computation, `show`/`summary`/
`sprintf`/`command`), the difference between `send`/`sendp` and `sr`/`sr1`,
how `sniff`, `AsyncSniffer`, `PcapReader`, and `PcapWriter` handle live and
offline traffic, and which protocol layers require an explicit import. Show
which operations need root, how `ls()` and `lsc()` differ, and how the CLI
`-s`/`-c`/`-p`/`-C`/`-P`/`-H` options load session and startup files. Cite the
installed help text, the upstream API docs, and observed output. Separate
confirmed behaviour from inference and state confidence.
```

Create a working directory and record the environment:

```bash
mkdir -p ~/scapy-demo/evidence
cd ~/scapy-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
python3 -c 'import scapy; print(scapy.__version__)' > evidence/version.txt
python3 -c 'from scapy.all import *; wrpcap("t.pcap", [Ether()/IP(dst="192.0.2.1")/TCP(dport=80), Ether()/IP(dst="192.0.2.2")/UDP(dport=53)])'
```

The `t.pcap` fixture is used by the offline capture and pcap scenarios; the
pcap scenario rewrites it with three packets.

## Scenario 1 — version, package, CLI, session files

| # | Command | Purpose |
|---|---------|---------|
| 1 | `scapy -h` | CLI usage |
| 2 | `scapy -h \| head -4` | Option summary |
| 3 | `python3 -c 'import scapy; print(scapy.__version__)'` | Library version |
| 4 | `python3 -c 'import scapy; print(scapy.__file__)'` | Package location |
| 5 | `dpkg -s python3-scapy \| grep -E 'Version\|Depends'` | Package provenance |
| 6 | `dpkg -L python3-scapy \| grep '/usr/bin/'` | Shipped executables |
| 7 | `command -v scapy` | Resolve the CLI |
| 8 | `file "$(command -v scapy)"` | Wrapper type |
| 9 | `head -1 "$(command -v scapy)"` | Interpreter line |
| 10 | `timeout 5 scapy --nonsense >/dev/null 2>&1; echo "badopt=$?"` | Unknown option exits 1 |
| 11 | `printf 'print("startup ran")\n' > start.py` | Startup file |
| 12 | `timeout 5 scapy -c start.py -H 2>&1 \| grep -c 'startup ran'` | `-c` runs it |
| 13 | `printf 'print("prestart ran")\n' > pre.py` | Pre-startup file |
| 14 | `timeout 5 scapy -p pre.py -c start.py -H 2>&1 \| grep -c 'prestart ran'` | `-p` runs first |
| 15 | `timeout 5 scapy -C -H 2>&1 \| grep -c 'startup ran'` | `-C` skips startup |
| 16 | `timeout 5 scapy -P -H 2>&1 \| grep -c 'prestart ran'` | `-P` skips prestart |
| 17 | `printf 'save_session("sess.pkl")\nexit()\n' \| timeout 10 scapy -H >/dev/null 2>&1` | Write a session pickle |
| 18 | `ls -l sess.pkl` | Session file exists |
| 19 | `timeout 5 scapy -s sess.pkl -c start.py -H 2>&1 \| grep -c 'startup ran'` | Load a session |
| 20 | `timeout 5 scapy -s start.py -H 2>&1 \| tail -2` | A script is not a session |
| 21 | `timeout 5 scapy -H 2>&1 \| head -2` | Header-less banner |
| 22 | `timeout 5 scapy -H 2>&1 \| grep -c 'aSPY'` | No ASCII art with `-H` |

**Research questions**
- Does `-H` change anything besides the banner?
- What does `-s` actually unpickle?
- Can `-c` and `-p` be combined, and in what order do they run?
- Which file does `-C` suppress: startup or pre-startup?

**Sources:** `scapy -h`; observed output; upstream `scapy/main.py`.
**Output:** a CLI option table and a session-file trace.

## Scenario 2 — packet model, layers, and formatting

| # | Command | Purpose |
|---|---------|---------|
| 23 | `python3 -c 'from scapy.all import IP; print(len(IP.fields_desc))'` | Field count |
| 24 | `python3 -c 'from scapy.all import IP, TCP; p=IP(dst="192.0.2.1")/TCP(dport=443, flags="S"); print(p.summary())'` | Stack and summarise |
| 25 | `python3 -c 'from scapy.all import IP, TCP; p=IP(dst="192.0.2.1")/TCP(dport=443); print(p.command())'` | Reproduce as code |
| 26 | `python3 -c 'from scapy.all import IP, TCP; p=IP(dst="192.0.2.1")/TCP(dport=443); print(p.sprintf("%IP.dst%:%TCP.dport%"))'` | Field formatting |
| 27 | `python3 -c 'from scapy.all import IP, TCP; p=IP(dst="192.0.2.1")/TCP(); print(p.haslayer(TCP), p.getlayer(TCP).dport)'` | Layer access |
| 28 | `python3 -c 'from scapy.all import IP, TCP; p=IP()/TCP(); print([l.__name__ for l in p.layers()])'` | Layer walk |
| 29 | `python3 -c 'from scapy.all import IP, TCP; p=IP()/TCP(); print(type(p.payload).__name__, type(p.payload.payload).__name__)'` | Payload chain |
| 30 | `python3 -c 'from scapy.all import IP; p=IP(dst="192.0.2.1"); print(p.len, p.chksum, p.src)'` | Lazy fields are None |
| 31 | `python3 -c 'from scapy.all import IP, TCP; p=IP(dst="192.0.2.1")/TCP(); print(len(bytes(p)))'` | Serialise and size |
| 32 | `python3 -c 'from scapy.all import IP, TCP; p=IP(dst="192.0.2.1")/TCP(dport=443); q=IP(bytes(p)); print(q.summary(), q[TCP].dport)'` | Dissect round trip |
| 33 | `python3 -c 'from scapy.all import IP, TCP; p=IP()/TCP(); print(p.copy()==p)'` | Copy equality |
| 34 | `python3 -c 'from scapy.all import IP, TCP; p=IP()/TCP(); print(p.direction)'` | Direction hint |
| 35 | `python3 -c 'from scapy.all import IP, TCP; p=IP()/TCP(); p.show()'` | Field display |
| 36 | `python3 -c 'from scapy.all import IP, TCP; p=IP()/TCP(); p.show2()'` | Computed field display |
| 37 | `python3 -c 'from scapy.all import IP, TCP; p=IP()/TCP(); print(p.wirelen)'` | Wire length |
| 38 | `python3 -c 'from scapy.all import IP; p=IP(); print(p.__class__.__mro__[1].__name__)'` | Base class |
| 39 | `python3 -c 'from scapy.all import IP, Raw; p=IP()/Raw(b"abc"); print(p[Raw].load)'` | Raw payload |
| 40 | `python3 -c 'from scapy.all import Ether, IP; p=Ether()/IP(); print(p.summary())'` | Ethernet stack |
| 41 | `python3 -c 'from scapy.all import IP, UDP; print((IP()/UDP()).sprintf("%IP.proto%"))'` | Protocol number |
| 42 | `python3 -c 'from scapy.all import IP, TCP; print((IP()/TCP()).sprintf("%TCP.flags%"))'` | Flags formatting |

**Research questions**
- Which fields are computed only at build time?
- Does `summary()` mutate the packet by filling `src`?
- Does `command()` round-trip a packet exactly?
- What is the difference between `show()` and `show2()`?

**Sources:** observed output; upstream API docs.
**Output:** a packet-model cheat sheet with field states.

## Scenario 3 — sending and receiving on loopback

| # | Command | Purpose |
|---|---------|---------|
| 43 | `python3 -c 'from scapy.all import *; r=sr1(IP(dst="127.0.0.1")/ICMP(), timeout=2, verbose=0); print(r.summary() if r else "none")'` | `sr1` reply |
| 44 | `python3 -c 'from scapy.all import *; r=sr1(IP(dst="127.0.0.1")/TCP(dport=1, flags="S"), timeout=1, verbose=0); print(r.summary() if r else "none")'` | TCP RST or none |
| 45 | `python3 -c 'from scapy.all import *; print(send(IP(dst="127.0.0.1")/ICMP(), verbose=0))'` | `send` returns None |
| 46 | `python3 -c 'from scapy.all import *; a,u=sr(IP(dst="127.0.0.1")/ICMP(), timeout=2, verbose=0); print(len(a), len(u))'` | `sr` answer counts |
| 47 | `python3 -c 'from scapy.all import *; r=sr1(IP(dst="127.0.0.1")/UDP(dport=9), timeout=1, verbose=0); print(r.summary() if r else "none")'` | UDP probe |
| 48 | `python3 -c 'from scapy.all import *; r=sr1(IP(dst="127.0.0.1")/ICMP(), timeout=0.01, verbose=0); print(r)'` | Short timeout returns None |
| 49 | `python3 -c 'from scapy.all import *; r=sr1(IP(dst="192.0.2.1")/ICMP(), timeout=1, verbose=0); print(r)'` | Unreachable target |
| 50 | `python3 -c 'from scapy.all import *; print(conf.route.route("127.0.0.1"))'` | Loopback route |
| 51 | `python3 -c 'from scapy.all import *; print(conf.route.route("192.0.2.1"))'` | Default route |
| 52 | `python3 -c 'from scapy.all import *; print(conf.iface)'` | Default interface |
| 53 | `python3 -c 'from scapy.all import *; print(len(conf.ifaces))'` | Interface count |
| 54 | `python3 -c 'from scapy.all import *; print(conf.route6.route("::1"))'` | IPv6 route |
| 55 | `python3 -c 'from scapy.all import *; print(arpcache())'` | ARP cache |
| 56 | `python3 -c 'from scapy.all import *; print(sendp.__doc__.splitlines()[0])'` | `sendp` contract |
| 57 | `python3 -c 'from scapy.all import *; print(srp1.__doc__.splitlines()[0])'` | `srp1` contract |
| 58 | `python3 -c 'from scapy.all import *; print(bridge_and_sniff.__doc__.splitlines()[0])'` | Bridge helper |
| 59 | `python3 -c 'from scapy.all import *; import inspect; print(inspect.signature(sendp))'` | `sendp` signature |
| 60 | `python3 -c 'from scapy.all import *; import inspect; print(inspect.signature(sr))'` | `sr` signature |

**Research questions**
- Does `sr1` return `None` or raise on timeout?
- Does `send` return a count or `None`?
- Which loopback probes produce replies?
- Does `conf.route.route()` fill `IP.src`?

**Sources:** observed output; upstream API docs.
**Output:** a send/receive result table with timing.

## Scenario 4 — live capture and background sniffing

| # | Command | Purpose |
|---|---------|---------|
| 61 | `python3 -c 'from scapy.all import *; p=sniff(iface="lo", count=1, timeout=3); print(len(p))'` | Capture on `lo` |
| 62 | `python3 -c 'from scapy.all import *; p=sniff(iface="lo", count=1, timeout=3, filter="icmp"); print(len(p))'` | BPF filter |
| 63 | `python3 -c 'from scapy.all import *; p=sniff(iface="lo", count=1, timeout=3, lfilter=lambda x: x.haslayer(ICMP)); print(len(p))'` | Python filter |
| 64 | `python3 -c 'from scapy.all import *; p=sniff(iface="lo", count=1, timeout=3, prn=lambda x: x.summary())'` | Per-packet callback |
| 65 | `python3 -c 'from scapy.all import *; p=sniff(iface="lo", count=1, timeout=3, store=False); print(p)'` | Do not store |
| 66 | `python3 -c 'from scapy.all import *; s=AsyncSniffer(iface="lo", count=1, timeout=3); s.start(); s.join(); print(len(s.results))'` | Background sniffer |
| 67 | `python3 -c 'from scapy.all import *; s=AsyncSniffer(iface="lo", filter="icmp"); s.start(); send(IP(dst="127.0.0.1")/ICMP(), verbose=0); s.stop(); print(len(s.results))'` | Capture around a send |
| 68 | `python3 -c 'from scapy.all import *; s=AsyncSniffer(offline="t.pcap"); s.start(); s.join(); print(len(s.results))'` | Offline async |
| 69 | `python3 -c 'from scapy.all import *; p=sniff(offline="t.pcap", count=2); print(len(p))'` | Offline `sniff` |
| 70 | `python3 -c 'from scapy.all import *; p=sniff(iface="lo", count=1, timeout=3, promisc=False); print(len(p))'` | Promiscuous off |
| 71 | `python3 -c 'from scapy.all import *; p=sniff(iface="lo", count=1, timeout=3, monitor=True); print(len(p))'` | Monitor flag on `lo` |
| 72 | `python3 -c 'from scapy.all import *; import inspect; print(inspect.signature(AsyncSniffer))'` | Sniffer signature |
| 73 | `python3 -c 'from scapy.all import *; print(conf.use_pcap)'` | libpcap backend |
| 74 | `python3 -c 'from scapy.all import *; print(conf.sniff_promisc)'` | Default promisc |
| 75 | `python3 -c 'from scapy.all import *; print(conf.L3socket)'` | L3 socket class |
| 76 | `python3 -c 'from scapy.all import *; print(conf.L2socket)'` | L2 socket class |
| 77 | `python3 -c 'from scapy.all import *; p=sniff(iface="lo", count=1, timeout=1, started_callback=lambda: print("started"))'` | Start callback |

**Research questions**
- Does `sniff` need root when reading a pcap?
- Does `AsyncSniffer.stop()` return results or require `join()`?
- Which filter runs in the kernel and which in Python?
- Does `monitor=True` work on a loopback interface?

**Sources:** observed output; upstream API docs.
**Output:** a capture-mode matrix with privilege notes.

## Scenario 5 — pcap and pcapng I/O

| # | Command | Purpose |
|---|---------|---------|
| 78 | `python3 -c 'from scapy.all import *; pk=[Ether()/IP(dst="192.0.2.%d"%i)/TCP(dport=80) for i in (1,2,3)]; wrpcap("t.pcap", pk); print(len(rdpcap("t.pcap")))'` | Write and read pcap |
| 79 | `python3 -c 'from scapy.all import *; print(len(rdpcap("t.pcap")))'` | Read all |
| 80 | `python3 -c 'from scapy.all import *; print(rdpcap("t.pcap", count=1)[0].summary())'` | Read one |
| 81 | `python3 -c 'from scapy.all import *; print(sum(1 for _ in PcapReader("t.pcap")))'` | Stream read |
| 82 | `python3 -c 'from scapy.all import *; w=PcapWriter("w.pcap"); [w.write(p) for p in rdpcap("t.pcap")]; w.close(); print(len(rdpcap("w.pcap")))'` | Stream write |
| 83 | `python3 -c 'from scapy.all import *; w=PcapWriter("a.pcap", append=True); w.write(IP()/ICMP()); w.close(); print(len(rdpcap("a.pcap")))'` | Append mode |
| 84 | `python3 -c 'from scapy.all import *; wrpcap("t.pcapng", [IP()/ICMP()]); print(len(rdpcap("t.pcapng")))'` | pcapng round trip |
| 85 | `python3 -c 'from scapy.all import *; w=PcapWriter("z.pcap", gz=True); w.write(IP()/ICMP()); w.close(); print(len(rdpcap("z.pcap")))'` | Gzip writer |
| 86 | `python3 -c 'from scapy.all import *; w=PcapWriter("n.pcap", nano=True); w.write(IP()/ICMP()); w.close(); print(len(rdpcap("n.pcap")))'` | Nanosecond timestamps |
| 87 | `python3 -c 'from scapy.all import *; print(PcapWriter("x.pcap").__class__.__name__)'` | Writer class |
| 88 | `python3 -c 'from scapy.all import *; print(PcapReader("t.pcap").__class__.__name__)'` | Reader class |
| 89 | `python3 -c 'from scapy.all import *; p=rdpcap("t.pcap")[0]; print(p.time)'` | Packet timestamp |
| 90 | `python3 -c 'from scapy.all import *; p=rdpcap("t.pcap")[0]; print(p.wirelen)'` | Original wire length |
| 91 | `python3 -c 'from scapy.all import *; p=rdpcap("t.pcap")[0]; print(p.sniffed_on)'` | Capture interface metadata |
| 92 | `python3 -c 'from scapy.all import *; p=rdpcap("t.pcap"); wrpcap("filtered.pcap", [x for x in p if x.haslayer(TCP)]); print(len(rdpcap("filtered.pcap")))'` | Filter and rewrite |
| 93 | `python3 -c 'from scapy.all import *; print(len(rdpcap("t.pcap")[0].payload))'` | Nested payload |
| 94 | `python3 -c 'from scapy.all import *; print(rdpcap("t.pcap")[0].original)'` | Original bytes |
| 95 | `python3 -c 'from scapy.all import *; print(rdpcap("t.pcap")[0].getlayer(IP).dst)'` | Layer lookup |
| 96 | `python3 -c 'from scapy.all import *; print(len(rdpcap("t.pcap")[0].summary()))'` | Summary length |
| 97 | `python3 -c 'from scapy.all import *; print(rdpcap("/nonexistent.pcap"))'` | Missing file raises |

**Research questions**
- Which pcap variants does this build read and write?
- Does `append=True` rewrite the global header?
- Does `gz=True` write a `.gz` or a gzip stream in `.pcap`?
- Which metadata survives a round trip?

**Sources:** observed output; upstream API docs.
**Output:** a pcap I/O matrix with file sizes.

## Scenario 6 — protocol layers and crafting

| # | Command | Purpose |
|---|---------|---------|
| 98 | `python3 -c 'from scapy.all import *; print((IP()/TCP()).summary())'` | TCP |
| 99 | `python3 -c 'from scapy.all import *; print((IP()/UDP()).summary())'` | UDP |
| 100 | `python3 -c 'from scapy.all import *; print((IP()/ICMP()).summary())'` | ICMP |
| 101 | `python3 -c 'from scapy.all import *; print((Ether()/ARP()).summary())'` | ARP |
| 102 | `python3 -c 'from scapy.all import *; print((Ether()/Dot1Q(vlan=10)/IP()).summary())'` | VLAN |
| 103 | `python3 -c 'from scapy.all import *; print((IP()/IPv6()).summary())'` | IP-in-IP |
| 104 | `python3 -c 'from scapy.all import *; print((IPv6()/ICMPv6EchoRequest()).summary())'` | ICMPv6 |
| 105 | `python3 -c 'from scapy.all import *; print((IP()/GRE()/IP()).summary())'` | GRE tunnel |
| 106 | `python3 -c 'from scapy.all import *; print((IP()/UDP()/VXLAN()).summary())'` | VXLAN |
| 107 | `python3 -c 'from scapy.layers.dns import *; print((IP()/UDP()/DNS(qd=DNSQR(qname="example.com"))).summary())'` | DNS query |
| 108 | `python3 -c 'from scapy.layers.dns import *; print((DNS(qd=DNSQR(qname="example.com"))/DNSRR(rrname="example.com", rdata="192.0.2.1")).summary())'` | DNS answer |
| 109 | `python3 -c 'from scapy.layers.dhcp import *; print((BOOTP()/DHCP()).summary())'` | DHCP |
| 110 | `python3 -c 'from scapy.layers.dhcp import *; print((DHCP(options=[("message-type","discover"),"end"])).summary())'` | DHCP discover |
| 111 | `python3 -c 'from scapy.layers.llmnr import *; print((LLMNRQuery(qd=DNSQR(qname="host"))).summary())'` | LLMNR |
| 112 | `python3 -c 'from scapy.layers.netbios import *; print((NBNSQueryRequest(QUESTION_NAME="HOST")).summary())'` | NBNS |
| 113 | `python3 -c 'from scapy.layers.http import *; print((HTTPRequest(Method="GET", Path="/")).summary())'` | HTTP request |
| 114 | `python3 -c 'from scapy.layers.http import *; print((HTTPResponse(Status_Code=200)).summary())'` | HTTP response |
| 115 | `python3 -c 'from scapy.layers.tls.all import *; print(TLS.__name__)'` | TLS classes |
| 116 | `python3 -c 'from scapy.layers.isakmp import *; print(ISAKMP.__name__)'` | ISAKMP |
| 117 | `python3 -c 'from scapy.layers.radius import *; print(RADIUS.__name__)'` | RADIUS |
| 118 | `python3 -c 'from scapy.layers.smb import *; print(SMB.__name__)'` | SMB |
| 119 | `python3 -c 'from scapy.layers.dot11 import *; print((RadioTap()/Dot11()/Dot11Beacon()).summary())'` | 802.11 beacon |
| 120 | `python3 -c 'from scapy.layers.dot11 import *; print((RadioTap()/Dot11()/Dot11ProbeReq()).summary())'` | Probe request |
| 121 | `python3 -c 'from scapy.layers.eap import *; print(EAPOL.__name__)'` | EAPOL |
| 122 | `python3 -c 'from scapy.layers.mgcp import *; print(MGCP.__name__)'` | MGCP |
| 123 | `python3 -c 'from scapy.layers.rtp import *; print(RTP.__name__)'` | RTP |
| 124 | `python3 -c 'from scapy.contrib.modbus import *; print(ModbusADURequest.__name__)'` | Modbus |
| 125 | `python3 -c 'from scapy.layers.ntp import *; print(NTP.__name__)'` | NTP |
| 126 | `python3 -c 'from scapy.layers.snmp import *; print(SNMP.__name__)'` | SNMP |

**Research questions**
- Which layers are pulled in by `from scapy.all import *`?
- Which modules must be imported explicitly?
- Does `scapy.contrib` hold protocol codecs that `scapy.layers` lacks?
- Which layer classes need a device or a live socket to build?

**Sources:** observed output; upstream API docs.
**Output:** a layer inventory with import paths.

## Scenario 7 — discovery, utilities, and fuzzing

| # | Command | Purpose |
|---|---------|---------|
| 127 | `python3 -c 'from scapy.all import *; print(callable(arping), callable(traceroute))'` | Helpers exist |
| 128 | `python3 -c 'from scapy.all import *; import inspect; print(inspect.signature(arping))'` | ARP sweep signature |
| 129 | `python3 -c 'from scapy.all import *; import inspect; print(inspect.signature(traceroute))'` | Traceroute signature |
| 130 | `python3 -c 'from scapy.all import *; print(arpcache())'` | ARP cache |
| 131 | `python3 -c 'from scapy.all import *; print(conf.route.route("192.0.2.1"))'` | Route lookup |
| 132 | `python3 -c 'from scapy.all import *; print(conf.route6.route("2001:db8::1"))'` | IPv6 route |
| 133 | `python3 -c 'from scapy.all import *; print(conf.iface)'` | Default interface |
| 134 | `python3 -c 'from scapy.all import *; print(len(conf.ifaces))'` | Interface count |
| 135 | `python3 -c 'from scapy.all import *; print(len(conf.layers))'` | Registered layers |
| 136 | `python3 -c 'from scapy.all import *; print(len(conf.commands))'` | Registered commands |
| 137 | `python3 -c 'from scapy.all import *; print(ls(IP))'` | `ls` prints and returns None |
| 138 | `python3 -c 'from scapy.all import *; import io,contextlib; b=io.StringIO(); c=contextlib.redirect_stdout(b); c.__enter__(); ls(IP); c.__exit__(None,None,None); print(len(b.getvalue().splitlines()))'` | Capture `ls` output |
| 139 | `python3 -c 'from scapy.all import *; import io,contextlib; b=io.StringIO(); c=contextlib.redirect_stdout(b); c.__enter__(); lsc(); c.__exit__(None,None,None); print(len(b.getvalue().splitlines()))'` | Capture `lsc` output |
| 140 | `python3 -c 'from scapy.all import *; print(callable(explore))'` | Interactive browser |
| 141 | `python3 -c 'from scapy.all import *; print(fuzz(IP()).summary())'` | Fuzz a header |
| 142 | `python3 -c 'from scapy.all import *; print(fuzz(IP()/UDP()).summary())'` | Fuzz a stack |
| 143 | `python3 -c 'from scapy.all import *; print(hexdump(bytes(IP()/ICMP())))'` | Hex dump |
| 144 | `python3 -c 'from scapy.all import *; print(len(bytes(IP()/ICMP())))'` | Byte length |
| 145 | `python3 -c 'from scapy.all import *; p=IP(); p.show(); print("ok")'` | Show output |
| 146 | `python3 -c 'from scapy.all import *; print(conf.use_pcap)'` | libpcap flag |
| 147 | `python3 -c 'from scapy.all import *; print(conf.sniff_promisc)'` | Default promisc |
| 148 | `python3 -c 'from scapy.all import *; print(conf.verb)'` | Default verbosity |
| 149 | `python3 -c 'from scapy.all import *; conf.verb=0; print(conf.verb)'` | Change verbosity |

**Research questions**
- Does `arping` need root?
- Does `traceroute` use UDP or ICMP by default?
- What does `ls(IP)` return?
- Which `conf` values affect capture and sending?

**Sources:** observed output; upstream API docs.
**Output:** a helper and utility reference with return values.

## Scenario 8 — failure modes, privileges, and cleanup

| # | Command | Purpose |
|---|---------|---------|
| 150 | `python3 -c 'from scapy.all import *; print(rdpcap("/nonexistent.pcap"))' 2>&1 \| tail -1` | Missing pcap |
| 151 | `python3 -c 'from scapy.all import *; print(IP(bytes(b"not a packet")))' 2>&1 \| tail -1` | Bad dissection |
| 152 | `python3 -c 'from scapy.all import *; print(sr1(IP(dst="192.0.2.1")/ICMP(), timeout=0.01, verbose=0))'` | Timeout is None |
| 153 | `python3 -c 'from scapy.all import *; print(sniff(offline="/nonexistent.pcap"))' 2>&1 \| tail -1` | Offline missing file |
| 154 | `python3 -c 'from scapy.all import *; print(conf.route.route("not-an-ip"))' 2>&1 \| tail -1` | Bad route target |
| 155 | `python3 -c 'from scapy.all import *; print(IP(dst="not-an-ip"))' 2>&1 \| tail -1` | Bad IP literal |
| 156 | `python3 -c 'from scapy.all import *; print(IP()/TCP(dport=99999))' 2>&1 \| tail -1` | Out-of-range port |
| 157 | `python3 -c 'from scapy.all import *; print(IP(ttl=999))' 2>&1 \| tail -1` | Out-of-range TTL |
| 158 | `python3 -c 'from scapy.all import *; print(Ether(dst="not-a-mac"))' 2>&1 \| tail -1` | Bad MAC |
| 159 | `python3 -c 'from scapy.all import *; print(sendp(IP()/ICMP()))' 2>&1 \| tail -1` | L3 packet to `sendp` |
| 160 | `python3 -c 'from scapy.all import *; print(send(IP(dst="192.0.2.1")/ICMP(), iface="nosuch0"))' 2>&1 \| tail -1` | Bad interface |
| 161 | `python3 -c 'from scapy.all import *; print(sniff(iface="nosuch0", count=1, timeout=1))' 2>&1 \| tail -1` | Bad capture interface |
| 162 | `python3 -c 'from scapy.all import *; print(fuzz(IP()).command())'` | Fuzzed packet command |
| 163 | `python3 -c 'from scapy.all import *; print(conf.layers[-1].__name__)'` | Last registered layer |
| 164 | `python3 -c 'from scapy.all import *; print(len(conf.layers))'` | Layer count |
| 165 | `python3 -c 'from scapy.all import *; print(len(conf.commands))'` | Command count |
| 166 | `python3 -c 'from scapy.all import *; print(IP().summary())'` | Empty packet summary |
| 167 | `python3 -c 'from scapy.all import *; print(bytes(IP()))'` | Empty packet bytes |
| 168 | `ls -l ~/scapy-demo` | Evidence tree |
| 169 | `rm -rf ~/scapy-demo` | Remove the demo tree |
| 170 | `test ! -e ~/scapy-demo; echo "removed=$?"` | Verify cleanup |

**Research questions**
- Which failures raise and which return `None`?
- Does Scapy validate field ranges at build or at send time?
- What changes when a non-root user runs the send/receive commands?
- Which artefacts are left after a capture run?

**Sources:** observed output; upstream API docs.
**Output:** an error matrix and a clean workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `scapy -h`, `-H`, `-C`, `-P`, `-c`, `-p`, `-s`, version, package provenance | 1 |
| layer stacking, `show`, `show2`, `summary`, `sprintf`, `command`, `bytes`, dissection, `layers`, `payload`, `Raw` | 2 |
| `send`, `sendp`, `sr`, `sr1`, `srp`, `srp1`, `bridge_and_sniff`, route/interface introspection | 3 |
| `sniff`, `AsyncSniffer`, `filter`, `lfilter`, `prn`, `store`, `promisc`, `monitor`, `started_callback` | 4 |
| `rdpcap`, `wrpcap`, `PcapReader`, `PcapWriter`, `append`, `gz`, `nano`, pcapng, metadata | 5 |
| IP/TCP/UDP/ICMP/ARP/VLAN/IPv6/GRE/VXLAN, DNS, DHCP, LLMNR, NBNS, HTTP, TLS, ISAKMP, RADIUS, SMB, 802.11, EAPOL, MGCP, RTP, Modbus, NTP, SNMP | 6 |
| `arping`, `traceroute`, `arpcache`, `conf.route`, `conf.ifaces`, `ls`, `lsc`, `explore`, `fuzz`, `hexdump`, `conf.verb` | 7 |
| error paths, bad fields, bad interfaces, fuzz, cleanup | 8 |

Full text: [`help.txt`](help.txt); authoritative API reference at
https://scapy.readthedocs.io/.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, package, CLI, session files | 22 |
| 2 | Packet model, layers, and formatting | 20 |
| 3 | Sending and receiving on loopback | 18 |
| 4 | Live capture and background sniffing | 17 |
| 5 | pcap and pcapng I/O | 20 |
| 6 | Protocol layers and crafting | 29 |
| 7 | Discovery, utilities, and fuzzing | 23 |
| 8 | Failure modes, privileges, and cleanup | 21 |
| **Total** | | **170** |

## Legal and safety

Scapy can send arbitrary packets, spoof source addresses, sweep networks, and
inject 802.11 frames. Only use it on links and hosts you own or are explicitly
authorised to test. ARP sweeps and spoofing can disrupt other users; wireless
injection can violate spectrum regulations and neighbour privacy. Capture only
traffic you are entitled to see, and treat pcaps as sensitive evidence. Do not
run denial-of-service or fuzzing campaigns against production systems without
a written scope and a rollback plan. Delete the demo tree and any captured
traffic when finished.
