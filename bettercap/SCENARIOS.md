# bettercap — command scenarios for deep research

Installed build: bettercap v2.33.0 (Debian package `bettercap`
`2.33.0-1+deb13u2`). Every CLI flag below appears in [`help.txt`](help.txt);
every interactive command and module parameter appears in
[`help-modules.txt`](help-modules.txt). Default autostart is `events.stream`.

> **Scope:** Bettercap is an active attack platform. ARP poisoning, DNS
> spoofing, deauthentication, rogue APs, and proxy redirection all change
> other hosts' traffic. Run it only on networks and clients you are authorized
> to test, with a defined stop-and-restore procedure.

## How to invoke

```text
@deep-research For an authorized layer-2 assessment, compare bettercap 2.33.0
with ettercap, dsniff, and arpspoof for ARP poisoning and DNS spoofing on a
switched LAN. Which tool gives the most reliable bidirectional MITM, how do
they differ in packet forwarding, cache restoration, and detection footprint,
and what does each leave behind in iptables and neighbour caches? Cite the
bettercap module documentation, the ettercap manual, RFC 826, and RFC 1035;
return a capability matrix with confidence notes.
```

Create an evidence workspace and record the environment before any active step:

```bash
mkdir -p ~/bettercap-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/bettercap-demo/evidence/run-start.txt
ip -brief link > ~/bettercap-demo/evidence/links.txt
ip route > ~/bettercap-demo/evidence/routes.txt
```

## Scenario 1 — build, interface, and session basics

Confirm the version, the interface, and how the session exposes state before
starting a module.

| # | Command | How to use |
|---|---------|------------|
| 1 | `bettercap -version` | Version banner and Go build |
| 2 | `bettercap --help` | All 17 CLI flags |
| 3 | `dpkg-query -W -f='${Version}\n' bettercap` | Installed package version |
| 4 | `ip -brief link` | Pick the interface deliberately |
| 5 | `sudo bettercap -iface eth0` | Start the interactive session |
| 6 | `help` | List session commands and all modules |
| 7 | `active` | Show which modules are running |
| 8 | `get *` | Dump every variable and its current value |

**Research questions**
- Which modules autostart, and where is that configured?
- What is the difference between a CLI flag and a module parameter?
- Which variables are global and which are namespaced per module?
- What does `get NAME*` return that `get NAME` does not?
- Why does `-iface` matter when a host has several interfaces?

**Sources:** `bettercap --help`; `help`; bettercap usage documentation.
**Output:** an interface and module inventory for the session.

## Scenario 2 — host discovery and inventory

`net.probe` sends discovery traffic; `net.recon` maintains the host table.
Neither is passive, so record what you send.

| # | Command | How to use |
|---|---------|------------|
| 9 | `net.probe on` | Start probing the subnet |
| 10 | `sleep 15` | Give probes time to complete |
| 11 | `net.show` | Print the discovered hosts |
| 12 | `set net.probe.throttle 50` | Slow probes to 50 ms apart |
| 13 | `set net.probe.mdns false` | Disable mDNS probes |
| 14 | `net.probe off` | Stop probing |
| 15 | `! ip neigh` | Compare with the kernel neighbour table |

**Research questions**
- Which probe types does `net.probe` send, and which one found each host?
- How does the bettercap host table differ from `ip neigh`?
- What does throttling change about reliability and detectability?
- Why can a host appear in one table but not the other?
- Which discovered attributes are passive observations and which are inferred?

**Sources:** `help net.probe`; `help net.recon`; bettercap module documentation;
RFC 6762 (mDNS); RFC 1001/1002 (NetBIOS).
**Output:** a host inventory with the probe type that produced each entry.

## Scenario 3 — ARP poisoning, DNS spoofing, and sniffing

This is the core MITM chain. Poison first, then redirect name resolution, then
capture. Stop in reverse order and verify restoration.

| # | Command | How to use |
|---|---------|------------|
| 16 | `set arp.spoof.targets 192.0.2.50` | Narrow the victim set |
| 17 | `set arp.spoof.fullduplex true` | Poison both victim and gateway |
| 18 | `arp.spoof on` | Start poisoning |
| 19 | `arp.ban on` | Cut connectivity instead of forwarding |
| 20 | `arp.spoof off` | Stop and restore caches |
| 21 | `set dns.spoof.domains example.com` | Domains to answer |
| 22 | `set dns.spoof.address 192.0.2.10` | Answer address |
| 23 | `dns.spoof on` | Start the DNS spoofer |
| 24 | `net.sniff on` | Capture forwarded traffic |

**Research questions**
- What changes when `arp.spoof.fullduplex` is true?
- Does `arp.ban` forward or drop, and how would a victim observe the difference?
- Which ARP cache entries are restored on `arp.spoof off`, and which are not?
- Why does `dns.spoof.all` default to false, and when is true required?
- What does `net.sniff.filter` exclude by default, and why does that matter
  while debugging poisoning?

**Sources:** `help arp.spoof`; `help dns.spoof`; `help net.sniff`; RFC 826;
RFC 1035; bettercap module documentation.
**Output:** a before/after neighbour-cache record plus captured evidence.

## Scenario 4 — HTTP, HTTPS, and arbitrary TCP proxying

Proxying turns interception into content control. Each module installs a
redirect, so record the firewall state before and after.

| # | Command | How to use |
|---|---------|------------|
| 25 | `set http.proxy.injectjs /tmp/inject.js` | Inject a local script into HTML |
| 26 | `http.proxy on` | Start the HTTP proxy and redirect |
| 27 | `https.proxy on` | Start the HTTPS proxy with a generated CA |
| 28 | `set https.proxy.sslstrip true` | Strip HTTPS instead of terminating TLS |
| 29 | `set any.proxy.src_port 3389` | Redirect RDP to a custom proxy |
| 30 | `set any.proxy.dst_port 8080` | Point at the proxy listener |
| 31 | `any.proxy on` | Install the generic redirect |
| 32 | `! iptables -t nat -L -n` | Verify the redirect rules exist |

**Research questions**
- What iptables rules does each proxy module install, and when are they removed?
- How does the generated HTTPS CA differ from `sslstrip` in capability and in
  detectability?
- What breaks when a client uses HSTS or certificate pinning?
- Why is `any.proxy` a redirect rather than a proxy?
- What must be true for `http.proxy.injectjs` to affect a page?

**Sources:** `help http.proxy`; `help https.proxy`; `help any.proxy`;
RFC 9110 (HTTP); RFC 8446 (TLS); bettercap proxy documentation.
**Output:** a firewall-rule diff and an injection test record.

## Scenario 5 — Wi-Fi reconnaissance and attacks

The `wifi` module is the largest surface in bettercap. Discovery, PMKID
capture, deauth, and rogue APs are separate actions with separate risks.

| # | Command | How to use |
|---|---------|------------|
| 33 | `set wifi.interface wlan0` | Pin the radio |
| 34 | `wifi.recon on` | Start discovery and channel hopping |
| 35 | `wifi.show` | List access points and clients |
| 36 | `wifi.show.wps all` | Show WPS information for every AP |
| 37 | `wifi.assoc all` | Request PMKID material from every AP |
| 38 | `wifi.deauth all` | Deauth clients of every AP with clients |
| 39 | `set wifi.ap.ssid FreeWiFi` | Rogue AP name |
| 40 | `set wifi.ap.encryption false` | Open rogue AP |
| 41 | `wifi.ap` | Inject rogue AP beacons |

**Research questions**
- Which channels does bettercap hop by default, and how does `wifi.hop.period`
  change coverage?
- What is the difference between `wifi.assoc` and a normal client association?
- Which APs answer `wifi.show.wps`, and what does that imply about their
  configuration?
- What does `wifi.deauth.open` enable by default, and why is that risky?
- What does a beacon-only rogue AP lack compared with a real AP?

**Sources:** `help wifi`; IEEE 802.11; Wi-Fi Alliance WPS specification;
bettercap Wi-Fi documentation.
**Output:** an AP/client inventory plus a per-action risk note.

## Scenario 6 — automation, scanning, and evidence

Caplets and `-eval` make runs reproducible. Events and rotating logs make them
auditable.

| # | Command | How to use |
|---|---------|------------|
| 42 | `set events.stream.output /tmp/bettercap.log` | Write events to a file |
| 43 | `events.stream on` | Ensure the stream is running |
| 44 | `events.on endpoint.new "set arp.spoof.targets $MAC"` | Trigger on a new endpoint |
| 45 | `events.triggers` | Verify the trigger exists |
| 46 | `syn.scan 192.0.2.0/24 1 1024` | Scan a subnet's low ports |
| 47 | `syn.scan.progress` | Print scan progress |
| 48 | `set mac.changer.address 02:00:00:00:00:01` | Set a known MAC |
| 49 | `mac.changer on` | Apply it |
| 50 | `quit` | End the session cleanly |

**Research questions**
- What event tags exist, and which one fires for a new host?
- How do event-trigger variables such as `$MAC` get populated?
- What does `syn.scan` report that `net.probe` does not?
- Why should the MAC be restored before leaving the engagement?
- Which artifacts prove what the session actually did?

**Sources:** `help events.stream`; `help syn.scan`; `help mac.changer`;
bettercap events documentation; `bettercap -caplet`.
**Output:** a caplet, an event log, and a scan report tied to timestamps.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| `-version`, `--help` | 1 |
| `-iface` | 1, 2, 3 |
| `-eval` | 1, 6 |
| `-caplet`, `-caplets-path`, `include` | 6 |
| `-autostart` | 1, 6 |
| `-env-file`, `-no-history` | 1 |
| `-gateway-override` | 3 |
| `-debug`, `-silent`, `-no-colors` | 1 |
| `-pcap-buf-size`, `-cpu-profile`, `-mem-profile`, `-script` | 1 |
| `help`, `active`, `get`, `set`, `alias`, `!`, `sleep`, `quit` | 1–6 |
| `arp.spoof.*` | 3 |
| `dns.spoof.*` | 3 |
| `net.probe.*`, `net.recon` | 2 |
| `net.sniff.*` | 3 |
| `http.proxy.*`, `https.proxy.*`, `any.proxy.*` | 4 |
| `wifi.*` | 5 |
| `events.stream.*`, `events.on` | 6 |
| `syn.scan*` | 6 |
| `mac.changer.*` | 6 |

Full text: [`help.txt`](help.txt) and [`help-modules.txt`](help-modules.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Build, interface, session basics | 8 |
| 2 | Host discovery and inventory | 7 |
| 3 | ARP/DNS spoofing and sniffing | 9 |
| 4 | HTTP/HTTPS/any proxying | 8 |
| 5 | Wi-Fi reconnaissance and attacks | 9 |
| 6 | Automation, scanning, evidence | 9 |
| **Total** | | **50** |

## Legal and safety

ARP poisoning, DNS spoofing, deauthentication, and rogue APs disrupt other
users' connectivity and can expose their credentials. Use them only with
written authorization, on the agreed SSIDs and subnets, and with an explicit
stop procedure: stop the proxy modules, stop spoofing, restore MAC addresses,
verify neighbour caches, and remove any iptables rules. Captured credentials,
handshakes, and PCAPs are engagement evidence — encrypt them, minimize
retention, and delete them at close.
