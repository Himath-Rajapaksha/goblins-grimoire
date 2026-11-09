# bettercap — modular MITM, network recon, and Wi-Fi attack framework

- **Version:** bettercap v2.33.0 (built for linux amd64 with go1.24.4);
  Debian package `bettercap` version `2.33.0-1+deb13u2`
- **Binary:** `/usr/bin/bettercap`
- **Help snapshots:** [`help.txt`](help.txt) — CLI flags;
  [`help-modules.txt`](help-modules.txt) — interactive `help`, plus
  `help arp.spoof`, `dns.spoof`, `wifi`, `net.probe`, `net.sniff`, `syn.scan`,
  `mac.changer`, `http.proxy`, `https.proxy`, `any.proxy`, `events.stream`
- **Upstream:** https://www.bettercap.org/ ·
  https://github.com/bettercap/bettercap
- **Caplets:** https://github.com/bettercap/caplets
- **Docs:** https://www.bettercap.org/usage/ ·
  https://www.bettercap.org/modules/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Bettercap is a single Go binary that turns a Linux host into a network
attack platform. It starts an **interactive session** (or runs a **caplet**
script), loads a set of **modules**, and exposes each module's state as
`module.parameter` variables you set with `set` and act on with
`module action`.

The architecture matters more than any single feature:

| Layer | What it gives you |
|-------|-------------------|
| Session | `set`, `get`, `help`, `include`, `!`, `alias`, event triggers |
| Modules | 31 modules: spoofing, sniffing, proxying, Wi-Fi, BLE, scanning |
| Events | A stream of parsed events you can filter, log, rotate, or trigger on |
| Caplets | Plain-text command scripts that reproduce a whole session |

By default only `events.stream` autostarts (`-autostart` default). Every other
module must be started explicitly with `MODULE on` or listed in `-autostart`.

```text
bettercap [options]
```

There is no positional target argument. Targets are set through module
parameters such as `arp.spoof.targets` or passed to module actions such as
`syn.scan`.

## Use cases

| Situation | Module / approach |
|-----------|-------------------|
| Map hosts on the local subnet | `net.probe` + `net.recon` |
| ARP-poison a client and gateway | `arp.spoof` |
| Spoof DNS answers | `dns.spoof` |
| Capture credentials and URLs | `net.sniff` |
| Inject JavaScript into HTTP pages | `http.proxy` with `http.proxy.injectjs` |
| Terminate TLS with a generated CA | `https.proxy` |
| Redirect arbitrary TCP to a custom proxy | `any.proxy` |
| Quick SYN port scan | `syn.scan` |
| Change the interface MAC | `mac.changer` |
| Wi-Fi AP and client discovery | `wifi.recon` |
| Capture WPA handshakes / PMKIDs | `wifi` + `wifi.assoc` |
| Deauthenticate clients | `wifi.deauth` |
| Rogue access point | `wifi.ap` |
| Wake a host on the LAN | `wol` |
| Remote-control the session over HTTP | `api.rest` + `ui` |

## CLI flags

| Flag | Meaning |
|------|---------|
| `-autostart LIST` | Comma-separated modules to start (default `events.stream`) |
| `-caplet FILE` | Read commands from this file and execute them in the session |
| `-caplets-path PATH` | Alternative base path for caplets |
| `-cpu-profile FILE` | Write a CPU profile |
| `-debug` | Print debug messages |
| `-env-file FILE` | Load environment variables from this file; empty disables persistence |
| `-eval CMD;CMD` | Run interactive commands from the command line |
| `-gateway-override IP` | Use this address instead of the detected default gateway |
| `-iface IFACE` | Interface to bind; auto-selected when empty |
| `-mem-profile FILE` | Write a memory profile |
| `-no-colors` | Disable colour output |
| `-no-history` | Disable the interactive history file |
| `-pcap-buf-size N` | PCAP buffer size; `0` for default (default `-1`) |
| `-script FILE` | Load a session script |
| `-silent` | Suppress all logs that are not errors |
| `-version` | Print the version and exit |

## Interactive commands

| Command | Meaning |
|---------|---------|
| `help [MODULE]` | List commands, or show module-specific help |
| `active` | Show active modules |
| `quit` | Close the session and exit |
| `sleep SECONDS` | Pause the session |
| `get NAME` | Read a variable; `*` for all, `NAME*` as a wildcard |
| `set NAME VALUE` | Set a variable |
| `read VARIABLE PROMPT` | Prompt for input and store it in a variable |
| `clear` | Clear the screen |
| `include CAPLET` | Load and run a caplet in the current session |
| `! COMMAND` | Run a shell command and print its output |
| `alias MAC NAME` | Assign a friendly name to an endpoint |

## Modules

The `help` output on this build lists 31 modules. The one-line purposes below
come from the module help captured in [`help-modules.txt`](help-modules.txt)
and the bettercap module documentation.

| Module | Purpose |
|--------|---------|
| `any.proxy` | Firewall redirection to any custom proxy |
| `api.rest` | REST API for controlling the session |
| `arp.spoof` | Keep selected hosts ARP-spoofed |
| `ble.recon` | Bluetooth Low Energy reconnaissance |
| `c2` | Command-and-control channel |
| `caplets` | Caplet management |
| `dhcp6.spoof` | DHCPv6 spoofing |
| `dns.spoof` | Reply to DNS queries with spoofed answers |
| `events.stream` | Continuous event stream (default) |
| `gps` | GPS integration |
| `graph` | Network graph |
| `hid` | HID device interaction |
| `http.proxy` | HTTP proxy with content injection |
| `http.server` | HTTP server |
| `https.proxy` | HTTPS proxy with a generated CA |
| `https.server` | HTTPS server |
| `mac.changer` | Change the active interface MAC |
| `mdns.server` | mDNS responder |
| `mysql.server` | MySQL server emulation |
| `ndp.spoof` | IPv6 neighbour-discovery spoofing |
| `net.probe` | Probe for new hosts on the subnet |
| `net.recon` | Network reconnaissance |
| `net.sniff` | Packet sniffer |
| `packet.proxy` | Packet-level proxy |
| `syn.scan` | SYN port scanner |
| `tcp.proxy` | TCP proxy |
| `ticker` | Run commands on a timer |
| `ui` | Web UI |
| `update` | Self-update |
| `wifi` | 802.11 monitoring and attacks |
| `wol` | Wake-on-LAN |

## Module parameter reference

### `arp.spoof`

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `arp.spoof.targets` | entire subnet | IPs, MACs, aliases, or nmap-style ranges |
| `arp.spoof.whitelist` | empty | Addresses to skip |
| `arp.spoof.fullduplex` | `false` | Spoof target and gateway; needed when the router has ARP protections |
| `arp.spoof.internal` | `false` | Also spoof host-to-host local traffic |
| `arp.spoof.skip_restore` | `false` | Do not restore target ARP caches on stop |

Actions: `arp.spoof on|off`, `arp.ban on|off` (cut connectivity instead of
forwarding).

### `dns.spoof`

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `dns.spoof.address` | interface address | Address to map spoofed domains to |
| `dns.spoof.all` | `false` | Answer every query, not just queries aimed at this host |
| `dns.spoof.domains` | empty | Comma-separated domains to spoof |
| `dns.spoof.hosts` | empty | Hosts-file mapping |
| `dns.spoof.ttl` | `1024` | TTL of spoofed replies |

### `wifi`

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `wifi.interface` | auto | Interface to use instead of `-iface` |
| `wifi.recon.channel` | empty (hop) | Channels to monitor, or `clear` to hop |
| `wifi.hop.period` | `250` | Channel-hop interval in ms |
| `wifi.region` | empty | Regulatory region set before activation |
| `wifi.rssi.min` | `-200` | Minimum signal strength in dBm |
| `wifi.ap.ssid` | `FreeWiFi` | Rogue AP SSID |
| `wifi.ap.bssid` | random | Rogue AP BSSID |
| `wifi.ap.channel` | `1` | Rogue AP channel |
| `wifi.ap.encryption` | `true` | WPA2 if true, open if false |
| `wifi.deauth.open` | `true` | Deauth clients of open networks too |
| `wifi.deauth.acquired` | `false` | Deauth APs whose key material you already have |
| `wifi.assoc.open` | `false` | Send association requests to open networks |
| `wifi.assoc.acquired` | `false` | Re-associate APs already acquired |
| `wifi.handshakes.file` | `~/bettercap-wifi-handshakes.pcap` | Handshake output |
| `wifi.handshakes.aggregate` | `true` | One pcap instead of one file per network |
| `wifi.source.file` | empty | Read 802.11 from a pcap instead of hardware |
| `wifi.txpower` | `30` | TX power set before activation |
| `wifi.skip-broken` | `true` | Skip frames with invalid checksums |

Actions include `wifi.recon on|off`, `wifi.show`, `wifi.show.wps BSSID`,
`wifi.deauth BSSID`, `wifi.assoc BSSID`, `wifi.ap`, `wifi.probe BSSID ESSID`,
`wifi.fake_auth BSSID CLIENT`, and `wifi.channel_switch_announce BSSID CHANNEL`.

### `net.probe` and `net.sniff`

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `net.probe.mdns` | `true` | mDNS discovery probes |
| `net.probe.nbns` | `true` | NetBIOS name-service probes |
| `net.probe.upnp` | `true` | UPnP discovery probes |
| `net.probe.wsd` | `true` | WSD discovery probes |
| `net.probe.throttle` | `10` | Milliseconds between probes |
| `net.sniff.filter` | `not arp` | BPF filter |
| `net.sniff.local` | `false` | Include packets to/from this host |
| `net.sniff.output` | empty | Write captured packets to a pcap |
| `net.sniff.regexp` | empty | Only report packets matching this regex |
| `net.sniff.source` | empty | Read from a pcap instead of the interface |
| `net.sniff.verbose` | `false` | Report every parsed packet, not just application-layer events |

### `http.proxy`, `https.proxy`, `any.proxy`

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `http.proxy.port` | `8080` | Local HTTP proxy port |
| `http.port` | `80` | Port redirected to the proxy |
| `http.proxy.redirect` | `true` | Install iptables redirect |
| `http.proxy.injectjs` | empty | URL, file, or JavaScript to inject into HTML |
| `http.proxy.sslstrip` | `false` | Enable SSL stripping |
| `http.proxy.script` | empty | Proxy JS script |
| `https.proxy.port` | `8083` | Local HTTPS proxy port |
| `https.port` | `443` | Port redirected to the proxy |
| `https.proxy.certificate` | `~/.bettercap-ca.cert.pem` | CA certificate |
| `https.proxy.key` | `~/.bettercap-ca.key.pem` | CA key |
| `https.proxy.certificate.commonname` | `Go Daddy Secure Certificate Authority - G2` | Spoofed CA common name |
| `any.proxy.src_port` | `80` | Remote port(s) to redirect |
| `any.proxy.dst_port` | `8080` | Proxy listening port |
| `any.proxy.protocol` | `TCP` | Proxy protocol |

### `syn.scan` and `mac.changer`

| Command / parameter | Meaning |
|---------------------|---------|
| `syn.scan IP-RANGE START-PORT END-PORT` | Scan a range |
| `syn.scan stop` | Stop the scan |
| `syn.scan.progress` | Print progress |
| `syn.scan.show-progress-every` | Progress interval in seconds (default `1`) |
| `mac.changer on\|off` | Apply / restore the MAC |
| `mac.changer.address` | Address to apply (default random) |
| `mac.changer.iface` | Interface to change |

### `events.stream`

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `events.stream.output` | stdout | Write events to this file |
| `events.stream.output.rotate` | `true` | Enable rotation |
| `events.stream.output.rotate.how` | `size` | Rotate by `size` or `time` |
| `events.stream.output.rotate.when` | `10` | Size in MB or time in seconds |
| `events.stream.output.rotate.compress` | `true` | Compress rotated logs |
| `events.stream.http.request.dump` | `false` | Dump full HTTP requests |
| `events.stream.http.response.dump` | `false` | Dump full HTTP responses |

Commands: `events.show`, `events.on TAG COMMANDS`, `events.triggers`,
`events.waitfor TAG TIMEOUT`, `events.ignore FILTER`, `events.include FILTER`,
`events.clear`.

## Recipes

**1. Version and CLI help.**

```bash
bettercap -version
bettercap --help
```

**2. Interactive session on an interface.**

```bash
sudo bettercap -iface eth0
```

**3. Run commands non-interactively.**

```bash
sudo bettercap -iface eth0 -eval 'net.probe on; sleep 10; net.show; quit'
```

**4. Run a caplet.**

```bash
sudo bettercap -iface eth0 -caplet /path/to/session.cap
```

**5. Discover hosts, then inspect the inventory.**

```text
net.probe on
sleep 15
net.show
```

**6. ARP spoof one client and the gateway, then sniff.**

```text
set arp.spoof.targets 192.0.2.50
set arp.spoof.fullduplex true
arp.spoof on
net.sniff on
```

**7. Spoof DNS for selected domains.**

```text
set dns.spoof.domains example.com,www.example.com
set dns.spoof.address 192.0.2.10
dns.spoof on
```

**8. HTTP proxy with JavaScript injection.**

```text
set http.proxy.injectjs /tmp/inject.js
http.proxy on
```

**9. HTTPS proxy with a generated CA.**

```text
https.proxy on
# Distribute ~/.bettercap-ca.cert.pem to the client, or use sslstrip.
```

**10. Quick SYN scan.**

```text
syn.scan 192.0.2.0/24 1 1024
syn.scan.progress
```

**11. Wi-Fi discovery.**

```text
set wifi.interface wlan0
wifi.recon on
wifi.show
```

**12. Capture a PMKID from every AP with clients.**

```text
wifi.assoc all
```

**13. Deauthenticate clients of one BSSID.**

```text
wifi.deauth AA:BB:CC:DD:EE:FF
```

**14. Rogue AP.**

```text
set wifi.ap.ssid FreeWiFi
set wifi.ap.encryption false
wifi.ap
```

**15. Change the MAC and restore it.**

```text
set mac.changer.address 02:00:00:00:00:01
mac.changer on
mac.changer off
```

**16. Log events to a rotating file.**

```text
set events.stream.output /var/log/bettercap.log
set events.stream.output.rotate.how time
set events.stream.output.rotate.when 3600
events.stream on
```

**17. Minimal caplet file.**

```text
# /tmp/session.cap
set net.sniff.verbose true
net.probe on
net.sniff on
```

**18. Caplet with a shell escape.**

```text
! ip -brief address
include /tmp/session.cap
```

## Gotchas

- **Most modules need root or capabilities.** Sniffing, spoofing, Wi-Fi monitor
  mode, and iptables redirection need `CAP_NET_RAW` / `CAP_NET_ADMIN`; run as
  root unless you have deliberately granted capabilities.
- **`events.stream` is already running.** `-autostart` defaults to it. Adding
  it again is harmless but unnecessary.
- **`-eval` uses `;` as the separator.** Quote the whole string so the shell
  does not split it.
- **`arp.spoof.fullduplex` defaults to false.** With ARP protections on the
  gateway, half-duplex poisoning silently fails; set it to true when you are
  authorized to poison both sides.
- **`arp.spoof.skip_restore` leaves the victim broken.** The default restores
  ARP caches on stop; do not disable that unless you intend to leave the
  network in a poisoned state.
- **`dns.spoof.all` defaults to false.** Without it, only queries aimed at the
  bettercap host are answered. Set it to true only for an authorized
  man-in-the-middle position.
- **`wifi.deauth.open` defaults to true.** Bettercap will deauth clients of
  open networks by default. Narrow `wifi.deauth.skip` or use a targeted BSSID.
- **Wi-Fi attacks need monitor mode.** Bettercap can set the interface up, but
  the driver must support monitor mode and packet injection. Check with
  `iw list` before blaming bettercap.
- **Channel hopping is the default.** An empty `wifi.recon.channel` means hop;
  set explicit channels when you need to stay on a target.
- **`wifi.assoc` is not passive.** It sends association requests to obtain
  PMKID material. That is active traffic and is visible to the AP.
- **`wifi.ap` injects beacons, not a real BSS.** Clients may associate, but
  there is no DHCP or upstream unless you provide it.
- **The HTTPS proxy CA is generated locally.** Clients must trust
  `~/.bettercap-ca.cert.pem`, or you must use `sslstrip`, which breaks on HSTS.
- **`http.proxy.redirect` edits iptables.** If bettercap exits uncleanly,
  check `iptables -t nat -L` for leftover redirects.
- **`any.proxy` is a redirect, not a proxy.** It sends traffic to a proxy you
  run separately; it does not implement the proxy protocol itself.
- **`syn.scan` is a raw-socket scanner.** It needs privileges and produces
  different results from a full TCP connect scan.
- **`net.sniff.filter` defaults to `not arp`.** Add `arp` back if you are
  debugging the poisoning itself.
- **No caplets are installed here.** `/usr/share/bettercap/caplets` is absent;
  fetch the caplets repository or point `-caplets-path` at your own scripts.
- **The `ui` module is not the CLI.** It serves a web interface; treat its
  exposure as a remote-control surface and bind it deliberately.
- **`-env-file` persists variables.** If you do not want state carried between
  sessions, pass `-env-file ''`.
- **`-pcap-buf-size -1` means "use the default".** Do not pass `-1` expecting a
  one-byte buffer.

## Related docs

- [`ettercap`](../ettercap/README.md) — classic MITM suite with plugins and
  content filters
- [`dsniff`](../dsniff/README.md) — the original ARP/DNS spoofing and snarfing
  toolkit
- [`aircrack-ng`](../aircrack-ng/README.md) — 802.11 capture and cracking suite
- [`wifite`](../wifite/README.md) — automated wireless auditing front end
- [`hcxdumptool`](../hcxdumptool/README.md) — PMKID and handshake capture
- [`hcxtools`](../hcxtools/README.md) — convert captures to hashcat formats
- [`kismet`](../kismet/README.md) — passive wireless detection
- [`eaphammer`](../eaphammer/README.md) — rogue AP and EAP attacks
- [`wifiphisher`](../wifiphisher/README.md) — captive-portal phishing
- [`mitmproxy`](../mitmproxy/README.md) — interactive HTTP/TLS interception
- [`wireshark`](../wireshark/README.md) — packet-level verification
- [`macchanger`](../macchanger/README.md) — standalone MAC changes
- [`iw`](../iw/README.md) — inspect and configure wireless interfaces
