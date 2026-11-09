# ettercap — multipurpose sniffer and man-in-the-middle suite

- **Version:** ettercap 0.8.3.1 (binary from Debian package
  `ettercap-text-only` version `1:0.8.3.1-14`; shared code from
  `ettercap-common` same version)
- **Binary:** `/usr/bin/ettercap`
- **Help snapshots:** [`help.txt`](help.txt) — `ettercap --help` (70 lines);
  [`help-plugins.txt`](help-plugins.txt) — `ettercap -P list` (34 plugins)
- **Upstream:** https://www.ettercap-project.org/ ·
  https://github.com/Ettercap/ettercap
- **Docs:** `man ettercap`, `man etterfilter`, `man ettersh`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Ettercap is a switched-LAN sniffer and MITM framework. It has two sniffing
models and a separate MITM engine:

| Model | Flag | Behaviour |
|-------|------|-----------|
| Unified | default | Sniffs one interface and forwards packets with layer-3 routing |
| Bridged | `-B IFACE` | Uses two interfaces and forwards at layer 1; stealthy but invasive |

MITM is independent of sniffing. `-M METHOD:ARGS` hijacks traffic and redirects
it to ettercap; the sniffing engine forwards it if necessary. You can combine
methods, or use `-o` to perform only the MITM and let the kernel forward.

The build installed here is **text-only**. `-T` (text) and `-C` (curses) are
available; `-G` (GTK) is not, because the GTK front end is a separate package.

```text
ettercap [OPTIONS] [TARGET1] [TARGET2]
```

## Target specification

Targets are **filters**, not source/destination pairs. Traffic between the two
targets, in either direction, is matched.

```text
MAC/IPs/PORTs
```

With IPv6 enabled, the form is `MAC/IPs/IPv6/PORTs`. Any omitted part means
"any".

| Target | Meaning |
|--------|---------|
| `//` | All hosts in the subnet |
| `//80` | Any host, port 80 only |
| `/192.0.2.10/` | Only that IP, any port |
| `/192.0.2.1-5;192.0.2.33/` | IP range plus a separate address |
| `/192.0.2.10/20-25,80,110/` | Port range plus individual ports |
| `00:11:22:33:44:55//` | A specific MAC |
| `-R /192.0.2.10/` | Everything **except** that target |

Notes from the manual:

- IP ranges use `-`, single addresses use `,`, and `;` separates different
  address groups.
- Port ranges use `-`, single ports use `,`.
- Specifying no target means "no target"; specifying `//` means "all hosts in
  the subnet".
- Targets also drive the initial LAN scan, so they restrict what is discovered.

## MITM methods

| Method | Arguments | Notes |
|--------|-----------|-------|
| `arp` | `[remote],[oneway]` | ARP cache poisoning; the standard bidirectional LAN MITM |
| `icmp` | `MAC/IP` | ICMP redirect; half-duplex, client side only |
| `dhcp` | `ip_pool/netmask/dns` | DHCP spoofing; half-duplex, races the real server |
| `port` | `[remote],[tree]` | Port stealing on switches; very noisy, dangerous |
| `ndp` | `[remote],[oneway]` | IPv6 neighbour-discovery poisoning |

`arp` and `ndp` accept `remote` when the victim's traffic must pass through a
gateway, and `oneway` to poison only TARGET1 → TARGET2. In `-z` silent mode
only the first target is selected unless you load a host list with `-j`.

`-M help` is **not** a valid method on this build; it prints
"MiTM method 'help' not supported". Use `man ettercap` for the method list.

## Flag reference

### Sniffing and attack

| Flag | Meaning |
|------|---------|
| `-M, --mitm METHOD:ARGS` | Perform a MITM attack |
| `-o, --only-mitm` | Do not sniff; only perform the MITM |
| `-b, --broadcast` | Sniff packets destined to broadcast |
| `-B, --bridge IFACE` | Bridged sniffing (needs two interfaces) |
| `-p, --nopromisc` | Do not put the interface in promiscuous mode |
| `-S, --nosslmitm` | Do not forge SSL certificates |
| `-u, --unoffensive` | Do not forward packets |
| `-r, --read FILE` | Read from a pcap file |
| `-f, --pcapfilter STRING` | Set the pcap filter |
| `-R, --reversed` | Reverse target matching |
| `-t, --proto PROTO` | Sniff only this protocol |
| `--certificate FILE` | Certificate for SSL MITM |
| `--private-key FILE` | Private key for SSL MITM |

### User interface

| Flag | Meaning |
|------|---------|
| `-T, --text` | Text-only UI |
| `-q, --quiet` | Do not display packet contents |
| `-s, --script CMD` | Issue these commands to the UI |
| `-C, --curses` | Curses UI |
| `-D, --daemon` | Daemonize, no UI |
| `-G, --gtk` | GTK UI (not available in the text-only build) |

### Logging

| Flag | Meaning |
|------|---------|
| `-w, --write FILE` | Write sniffed data to a pcap |
| `-L, --log FILE` | Log all traffic |
| `-l, --log-info FILE` | Log only passive information |
| `-m, --log-msg FILE` | Log all messages |
| `-c, --compress` | Gzip-compress log files |

### Visualization

| Flag | Meaning |
|------|---------|
| `-d, --dns` | Resolve IP addresses to hostnames |
| `-V, --visual FORMAT` | Set the visualization format |
| `-e, --regex REGEX` | Visualize only packets matching a regex |
| `-E, --ext-headers` | Print extended headers for every packet |
| `-Q, --superquiet` | Do not display usernames and passwords |

### Lua

| Flag | Meaning |
|------|---------|
| `--lua-script S1[,S2,...]` | Comma-separated Lua scripts |
| `--lua-args n1=v1[,n2=v2,...]` | Arguments to the Lua scripts |

### General

| Flag | Meaning |
|------|---------|
| `-i, --iface IFACE` | Interface to use |
| `-I, --liface` | List all interfaces |
| `-Y, --secondary IFACES` | Secondary interfaces |
| `-n, --netmask NETMASK` | Force a netmask on the interface |
| `-A, --address ADDRESS` | Force a local address on the interface |
| `-P, --plugin PLUGIN` | Launch a plugin (repeatable) |
| `--plugin-list P1[,P2,...]` | Comma-separated plugin list |
| `-F, --filter FILE` | Load an etterfilter content filter |
| `-z, --silent` | Do not perform the initial ARP scan |
| `-6, --ip6scan` | ICMPv6 probe for IPv6 nodes |
| `-j, --load-hosts FILE` | Load the hosts list |
| `-k, --save-hosts FILE` | Save the hosts list |
| `-W, --wifi-key WKEY` | WEP/WPA key for Wi-Fi decryption |
| `-a, --config FILE` | Alternative config file |
| `-v, --version` | Version and exit |
| `-h, --help` | Help and exit |

## Plugins

`ettercap -P list` works without a UI and prints 34 plugins. The most useful
for a MITM workflow:

| Plugin | Purpose |
|--------|---------|
| `dns_spoof` | Send spoofed DNS replies |
| `mdns_spoof` | Send spoofed mDNS replies |
| `nbns_spoof` | Spoof NBNS and send SMB challenges |
| `arp_cop` | Report suspicious ARP activity |
| `repoison_arp` | Re-poison after broadcast ARP |
| `chk_poison` | Check whether poisoning succeeded |
| `find_ettercap` | Detect other ettercap activity |
| `scan_poisoner` | Actively search for other poisoners |
| `gw_discover` | Find the LAN gateway |
| `find_ip` | Find an unused IP in the subnet |
| `isolate` | Isolate a host from the LAN |
| `autoadd` | Add new victims inside the target range |
| `sslstrip` | SSL stripping |
| `smb_clear`, `smb_down` | Force weaker SMB authentication |
| `krb5_downgrade` | Downgrade Kerberos V5 AS-REQ security |
| `stp_mangler` | Become root of the spanning tree |
| `remote_browser` | Send visited URLs to a browser |
| `dos_attack`, `smurf_attack`, `fraggle_attack` | Denial-of-service tests |
| `pptp_*` | PPTP authentication downgrades |

Full list: [`help-plugins.txt`](help-plugins.txt).

## Recipes

**1. Text UI, no promiscuous mode.**

```bash
sudo ettercap -Tp
```

**2. Quiet text UI, no initial ARP scan, no packet contents.**

```bash
sudo ettercap -Tzq
```

**3. ARP poison all hosts in the LAN (dangerous).**

```bash
sudo ettercap -T -M arp // //
```

**4. ARP poison a gateway and a client range, with remote sniffing.**

```bash
sudo ettercap -T -M arp:remote /192.0.2.1/ /192.0.2.2-10/
```

**5. Load a victim list and poison two ranges.**

```bash
sudo ettercap -T -j /tmp/victims -M arp /192.0.2.1-7/ /192.0.2.10-20/
```

**6. Sniff only POP3 from every host.**

```bash
sudo ettercap -Tzq //110
```

**7. Sniff telnet, FTP, and SSH to one host.**

```bash
sudo ettercap -Tzq /192.0.2.10/21,22,23
```

**8. List plugins.**

```bash
ettercap -P list
```

**9. Load the DNS spoof plugin.**

```bash
sudo ettercap -T -P dns_spoof -M arp /192.0.2.50/ /192.0.2.1/
```

**10. SSL MITM with an explicit certificate.**

```bash
sudo ettercap -T -M arp --certificate fake.crt --private-key fake.key \
  /192.0.2.50/ /192.0.2.1/
```

**11. Bridged sniffing between two interfaces.**

```bash
sudo ettercap -T -B eth0 -B eth1
```

**12. Write the session to a pcap and a log.**

```bash
sudo ettercap -T -w capture.pcap -L traffic.log -M arp // //
```

**13. Only MITM, no sniffing — let the kernel forward.**

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo ettercap -T -o -M arp /192.0.2.50/ /192.0.2.1/
```

**14. Save and reload the host list.**

```bash
sudo ettercap -T -k /tmp/hosts -M arp // //
sudo ettercap -T -j /tmp/hosts -M arp // //
```

## Gotchas

- **The installed build is text-only.** `-G` cannot work without the
  `ettercap-graphical` package. Use `-T`, `-C`, or `-D`.
- **Ettercap disables kernel IP forwarding.** The manual warns that this is
  invasive on a gateway and cannot be restored after privileges are dropped.
  Re-enable `net.ipv4.ip_forward` yourself when you are done.
- **On a gateway, use `-u`.** Offensive mode plus a gateway can strand traffic
  because ettercap only listens on one interface.
- **`-f` filters before forwarding.** The manual warns that a pcap filter can
  prevent ettercap from forwarding hijacked packets, breaking the MITM.
- **`-o` does not forward.** In only-MITM mode the kernel must have forwarding
  enabled, or victims lose connectivity.
- **`-z` changes target selection.** In silent mode only the first target is
  used for ARP/NDP poisoning unless `-j` loads a host list.
- **`arp:oneway` poisons only one direction.** That is useful to avoid an ARP
  watcher on the router, but it gives you half the conversation.
- **`icmp` and `dhcp` are half-duplex.** Only the client is redirected; filters
  must not change payload length or TCP sequences break.
- **`port` stealing is dangerous.** The manual warns about performance loss and
  general havoc, and says it cannot be used with `-o`.
- **`ndp` needs IPv6 support.** It is a no-op otherwise, and IPv6 ranges are
  not supported — use single addresses.
- **`-q` is not privacy.** It hides packet contents but still shows credentials;
  `-Q` is the flag that suppresses usernames and passwords.
- **`-M help` is not a help command.** It is parsed as a method name and fails.
  Read the manual instead.
- **Plugin names are not validated until launch.** A typo in `-P` fails at
  startup, not at parse time.
- **SSL MITM requires the client to accept your certificate.** Without
  `--certificate`/`--private-key`, ettercap forges one; `-S` disables the
  interception entirely.
- **`-B` needs two interfaces.** Passing one interface is a usage error.
- **`-w` captures everything, not just targets.** The manual notes that dump
  files collect all packets regardless of the target filter.
- **Root is required for link-layer sockets.** Ettercap drops privileges after
  initialization, but it still needs to start as root.
- **Wi-Fi decryption is passive.** `-W` decrypts captured WEP/WPA frames; it
  does not put the card into monitor mode for you.
- **`-a` replaces the config.** If you use an alternative config, plugin and
  UI defaults may differ from the system config.

## Related docs

- [`bettercap`](../bettercap/README.md) — modern modular MITM framework
- [`dsniff`](../dsniff/README.md) — original ARP/DNS spoofing and snarfing suite
- [`wireshark`](../wireshark/README.md) — verify captures at packet level
- [`tcpdump`](../tcpdump/README.md) — lightweight capture and BPF filters
- [`mitmproxy`](../mitmproxy/README.md) — scriptable HTTP/TLS interception
- [`eaphammer`](../eaphammer/README.md) — wireless EAP and rogue-AP attacks
- [`wifiphisher`](../wifiphisher/README.md) — captive-portal phishing
- [`nmap`](../nmap/README.md) — host and service discovery before targeting
- [`nbtscan`](../nbtscan/README.md) — NetBIOS name discovery
- [`smbclient`](../smbclient/README.md) — SMB interaction after credential capture
