# Goblin's Grimoire

![Animated banner: an engraved brass seal with an eye at its centre, ringed by icons for recon, web, credentials, network, wireless, exploitation, reversing, and privacy.](assets/goblins-grimoire-banner.svg)

A field manual for the pentest and bug-bounty toolbox: every command, every flag, and a research workflow for deciding when to use each one.
Each tool has its own directory containing:

| File | Contents |
|------|----------|
| `README.md` | What the tool does, use cases, full command/flag reference, recipes, gotchas, links |
| `SCENARIOS.md` | Command-wise scenarios grouped by task, each with the `@deep-research` question that decides when to use it, plus complete flag coverage |
| `help.txt` (or `help-*.txt`) | Verbatim `--help` output captured from the installed version |

Full help snapshots are authoritative for the exact installed build. The `README.md`
adds context, workflows, and copy-paste recipes that `--help` does not explain.

> **Start here for research-driven use:** [`DEEP-RESEARCH.md`](DEEP-RESEARCH.md) —
> the `@deep-research` workflow, prompt template, evidence rules, research-log
> format, and the index of all 119 `SCENARIOS.md` files (118 tools;
> `metasploit-msfvenom` is documented alongside Metasploit).
>
> **External references:** [`RELATED-DOCS.md`](RELATED-DOCS.md) — a cited
> comparison of public per-tool/per-command documentation projects (WADComs,
> GTFOBins, LOLBAS, Arsenal, Kali Tools, HackTricks, and others), what each
> does better, and what this repo should borrow.

---

## Tool index

### Recon & asset discovery
| Tool | Directory | One-liner |
|------|-----------|-----------|
| nmap | [`nmap/`](nmap/README.md) | Port scanning, service/OS fingerprinting, NSE scripting |
| masscan | [`masscan/`](masscan/README.md) | Internet-scale asynchronous port scanner |
| tcpdump | [`tcpdump/`](tcpdump/README.md) | Packet capture and BPF-filtered traffic analysis |
| whois | [`whois/`](whois/README.md) | Domain/IP/ASN registration lookups (RFC 3912) |
| traceroute | [`traceroute/`](traceroute/README.md) | Hop-by-hop path discovery with UDP/ICMP/TCP probes |
| dnsrecon | [`dnsrecon/`](dnsrecon/README.md) | DNS enumeration, zone transfer, brute force, cache snooping |
| subfinder | [`subfinder/`](subfinder/README.md) | Passive subdomain enumeration from dozens of sources |
| httpx | [`httpx/`](httpx/README.md) | Fast HTTP probing, tech detection, screenshots, filtering |
| katana | [`katana/`](katana/README.md) | Web crawler (standard + headless), JS endpoint extraction |
| arjun | [`arjun/`](arjun/README.md) | HTTP parameter discovery |

### OSINT & people
| Tool | Directory | One-liner |
|------|-----------|-----------|
| theHarvester | [`theharvester/`](theharvester/README.md) | Emails, hosts, and employee names from public sources |
| recon-ng | [`recon-ng/`](recon-ng/README.md) | Modular OSINT/recon framework |
| spiderfoot | [`spiderfoot/`](spiderfoot/README.md) | Automated OSINT correlation and scan automation |
| ghunt | [`ghunt/`](ghunt/README.md) | Google account OSINT |
| holehe | [`holehe/`](holehe/README.md) | Email registration checks across services |
| sherlock | [`sherlock/`](sherlock/README.md) | Username search across social networks |
| metagoofil | [`metagoofil/`](metagoofil/README.md) | Metadata extraction from public documents |

### Web application testing
| Tool | Directory | One-liner |
|------|-----------|-----------|
| burp | [`burp/`](burp/README.md) | Burp Suite Community — intercepting proxy & manual testing |
| zap | [`zap/`](zap/README.md) | OWASP ZAP — proxying, spidering, active scanning, automation |
| mitmproxy | [`mitmproxy/`](mitmproxy/README.md) | Scriptable TLS-intercepting proxy (CLI/TUI/web) |
| nikto | [`nikto/`](nikto/README.md) | Classic web server misconfiguration/vulnerability scanner |
| nuclei | [`nuclei/`](nuclei/README.md) | Template-driven vulnerability scanner |
| dalfox | [`dalfox/`](dalfox/README.md) | XSS scanner with DOM analysis, mining, WAF bypass |
| sqlmap | [`sqlmap/`](sqlmap/README.md) | SQL injection detection, exploitation, and takeover |
| ffuf | [`ffuf/`](ffuf/README.md) | Fast content, vhost, and parameter fuzzing |
| gobuster | [`gobuster/`](gobuster/README.md) | Directory, DNS, vhost, and fuzz brute force |
| wfuzz | [`wfuzz/`](wfuzz/README.md) | General-purpose web fuzzer |
| dirsearch | [`dirsearch/`](dirsearch/README.md) | Web path and file brute force |
| dirb | [`dirb/`](dirb/README.md) | Classic directory brute force with recursive scanning |
| whatweb | [`whatweb/`](whatweb/README.md) | Web technology fingerprinting with ~1,800 plugins |
| cewl | [`cewl/`](cewl/README.md) | Wordlist generator from website content |

### JavaScript, CMS & secrets
| Tool | Directory | One-liner |
|------|-----------|-----------|
| linkfinder | [`linkfinder/`](linkfinder/README.md) | Extract endpoints from JavaScript |
| secretfinder | [`secretfinder/`](secretfinder/README.md) | Find secrets and API keys in JavaScript |
| cmseek | [`cmseek/`](cmseek/README.md) | CMS detection and deep scanning |
| joomscan | [`joomscan/`](joomscan/README.md) | Joomla enumeration and vulnerability checks |
| paramspider | [`paramspider/`](paramspider/README.md) | Harvest parameterized URLs from archives |
| x8 | [`x8/`](x8/README.md) | Hidden parameter discovery |

### API & protocol testing
| Tool | Directory | One-liner |
|------|-----------|-----------|
| corsy | [`corsy/`](corsy/README.md) | CORS misconfiguration scanner |
| smuggler | [`smuggler/`](smuggler/README.md) | HTTP request smuggling detector |
| websocat | [`websocat/`](websocat/README.md) | WebSocket CLI client and server |
| grpcurl | [`grpcurl/`](grpcurl/README.md) | `curl` for gRPC services |
| subjs | [`subjs/`](subjs/README.md) | Fetch JavaScript files from URLs |
| sourcemapper | [`sourcemapper/`](sourcemapper/README.md) | Recover original source from source maps |

### Credentials & password attacks
| Tool | Directory | One-liner |
|------|-----------|-----------|
| hashcat | [`hashcat/`](hashcat/README.md) | GPU/CPU hash cracking across 300+ modes |
| john | [`john/`](john/README.md) | John the Ripper password cracker (core 1.9.0 build) |
| hydra | [`hydra/`](hydra/README.md) | Network login brute force across 50+ protocols |
| medusa | [`medusa/`](medusa/README.md) | Parallel network login brute force (25 modules) |

### Network / Active Directory
| Tool | Directory | One-liner |
|------|-----------|-----------|
| netexec | [`netexec/`](netexec/README.md) | Multi-protocol network exploitation (SMB/LDAP/WinRM/...) |
| impacket | [`impacket/`](impacket/README.md) | Python SMB/MS-RPC/Kerberos toolkit and 62 example scripts |
| smbclient | [`smbclient/`](smbclient/README.md) | Samba SMB/CIFS client and mini shell |
| smbget | [`smbget/`](smbget/README.md) | `wget`-style SMB downloader with resume and recursion |
| samba-common-bin | [`samba-common-bin/`](samba-common-bin/README.md) | `net`, `smbpasswd`, `testparm`, `smbcontrol`, `nmblookup`, `winexe` |
| rpcclient | [`rpcclient/`](rpcclient/README.md) | Samba MS-RPC client for SAMR/LSA/SPOOLSS |
| ldapsearch | [`ldapsearch/`](ldapsearch/README.md) | OpenLDAP query tool for directory enumeration |
| ldapdomaindump | [`ldapdomaindump/`](ldapdomaindump/README.md) | Bulk AD LDAP dump to HTML/JSON/greppable text |
| enum4linux-ng | [`enum4linux-ng/`](enum4linux-ng/README.md) | SMB/NetBIOS/AD enumeration |
| wireshark | [`wireshark/`](wireshark/README.md) | Packet capture & analysis (`tshark`, `dumpcap`, GUI) |
| scapy | [`scapy/`](scapy/README.md) | Python packet crafting, sniffing, and protocol layers |
| snmp | [`snmp/`](snmp/README.md) | Net-SNMP CLI suite (`snmpwalk`, `snmpget`, `snmpset`, ...) |
| onesixtyone | [`onesixtyone/`](onesixtyone/README.md) | Fast SNMP community-string scanner |
| nbtscan | [`nbtscan/`](nbtscan/README.md) | NetBIOS name and address scanner |
| sshuttle | [`sshuttle/`](sshuttle/README.md) | VPN-over-SSH for pivoting into internal networks |
| openvpn | [`openvpn/`](openvpn/README.md) | OpenVPN 2.x client/server: config, TLS, routing, management |
| openvpn3-client | [`openvpn3-client/`](openvpn3-client/README.md) | OpenVPN 3 Linux client (`openvpn3` / `openvpn2`) |
| socat | [`socat/`](socat/README.md) | Multipurpose relay: TCP/UDP/TLS/UNIX/PTY bridging |
| nc | [`nc/`](nc/README.md) | netcat-traditional ad-hoc TCP/UDP client and listener |
| proxychains4 | [`proxychains4/`](proxychains4/README.md) | Force TCP through SOCKS/HTTP proxy chains via LD_PRELOAD |
| kerbrute | [`kerbrute/`](kerbrute/README.md) | AD username enumeration and password spraying |

### Sniffing & MITM
| Tool | Directory | One-liner |
|------|-----------|-----------|
| bettercap | [`bettercap/`](bettercap/README.md) | Modular MITM framework: ARP/DNS spoofing, Wi-Fi, BLE, HTTP(S) proxy |
| ettercap | [`ettercap/`](ettercap/README.md) | Classic ARP/MITM sniffer with plugins and protocol dissectors |
| dsniff | [`dsniff/`](dsniff/README.md) | Suite of ARP/DNS spoof, snarf, and connection-kill tools |

### Wireless & RF
| Tool | Directory | One-liner |
|------|-----------|-----------|
| aircrack-ng | [`aircrack-ng/`](aircrack-ng/README.md) | 802.11 capture, injection, and WEP/WPA-PSK cracking suite |
| wifite | [`wifite/`](wifite/README.md) | Automated wireless auditing front end |
| reaver | [`reaver/`](reaver/README.md) | WPS PIN and Pixie-Dust attack tool |
| pixiewps | [`pixiewps/`](pixiewps/README.md) | Offline WPS Pixie-Dust computation |
| wash | [`wash/`](wash/README.md) | WPS survey and lock-state discovery |
| macchanger | [`macchanger/`](macchanger/README.md) | Inspect and change interface MAC addresses |
| iw | [`iw/`](iw/README.md) | nl80211 wireless configuration and inspection |
| hcxdumptool | [`hcxdumptool/`](hcxdumptool/README.md) | Monitor-mode capture of PMKID/EAPOL for WPA/WPA2 cracking |
| hcxtools | [`hcxtools/`](hcxtools/README.md) | Convert, clean, and inspect captures; build hashcat inputs |
| kismet | [`kismet/`](kismet/README.md) | Wireless detector, sniffer, and IDS with web UI |
| bully | [`bully/`](bully/README.md) | WPS PIN brute force |
| eaphammer | [`eaphammer/`](eaphammer/README.md) | 802.1X/EAP Evil Twin and credential capture |
| wifiphisher | [`wifiphisher/`](wifiphisher/README.md) | Rogue AP and credential-phishing framework (Python 3 patched) |
| wpa_supplicant | [`wpa_supplicant/`](wpa_supplicant/README.md) | WPA/WPA2/WPA3 supplicant plus `wpa_cli` and `wpa_passphrase` |
| wireless-tools | [`wireless-tools/`](wireless-tools/README.md) | Legacy `iwconfig`/`iwlist`/`iwpriv`/`iwspy` interface utilities |

### Bluetooth
| Tool | Directory | One-liner |
|------|-----------|-----------|
| bluez | [`bluez/`](bluez/README.md) | `bluetoothctl`, `hcitool`, `btmgmt`, `btmon`, `sdptool`, `rfcomm` |

### Android & mobile
| Tool | Directory | One-liner |
|------|-----------|-----------|
| adb | [`adb/`](adb/README.md) | Android Debug Bridge: devices, shell, files, packages, port forwarding |
| fastboot | [`fastboot/`](fastboot/README.md) | Bootloader flashing, unlocking, slots, and partition management |
| scrcpy | [`scrcpy/`](scrcpy/README.md) | USB/TCP screen mirroring and control with recording and audio |

### Phishing & social engineering
| Tool | Directory | One-liner |
|------|-----------|-----------|
| gophish | [`gophish/`](gophish/README.md) | Phishing campaign platform with tracking and templates |
| evilginx | [`evilginx/`](evilginx/README.md) | Adversary-in-the-middle phishing proxy |
| set | [`set/`](set/README.md) | Social-Engineer Toolkit menu-driven attack framework |
| crunch | [`crunch/`](crunch/README.md) | Deterministic wordlist generator |

### Exploitation & fuzzing
| Tool | Directory | One-liner |
|------|-----------|-----------|
| metasploit | [`metasploit/`](metasploit/README.md) | Metasploit Framework console, modules, sessions |
| msfvenom | [`metasploit-msfvenom/`](metasploit-msfvenom/README.md) | Payload generation and encoding |
| pwntools | [`pwntools/`](pwntools/README.md) | Python exploit-development framework/CLI |
| aflplusplus | [`aflplusplus/`](aflplusplus/README.md) | Coverage-guided fuzzing |

### Reverse engineering, pwn & forensics
| Tool | Directory | One-liner |
|------|-----------|-----------|
| apktool | [`apktool/`](apktool/README.md) | Android APK decode/rebuild |
| ropper | [`ropper/`](ropper/README.md) | ROP gadget finder and chain builder |
| ropgadget | [`ropgadget/`](ropgadget/README.md) | ROP gadget finder |
| one-gadget | [`one-gadget/`](one-gadget/README.md) | glibc one-shot `execve` gadget finder |
| pwndbg | [`pwndbg/`](pwndbg/README.md) | GDB plugin for exploit development |
| gef | [`gef/`](gef/README.md) | GDB Enhanced Features |
| volatility3 | [`volatility3/`](volatility3/README.md) | Memory forensics framework |
| ghidra | [`ghidra/`](ghidra/README.md) | NSA reverse-engineering suite: headless analysis, scripts, server |
| binwalk | [`binwalk/`](binwalk/README.md) | Firmware signature scanning, entropy analysis, extraction |
| checksec | [`checksec/`](checksec/README.md) | ELF mitigation report (pwntools wrapper) |
| exiftool | [`exiftool/`](exiftool/README.md) | Read/write metadata across image, document, and media formats |
| mat2 | [`mat2/`](mat2/README.md) | Metadata anonymisation for documents, images, audio, archives |
| patchelf | [`patchelf/`](patchelf/README.md) | Modify ELF interpreter, RPATH/RUNPATH, NEEDED, SONAME |
| strace | [`strace/`](strace/README.md) | Trace system calls and signals |

### TLS & service scanning
| Tool | Directory | One-liner |
|------|-----------|-----------|
| testssl | [`testssl/`](testssl/README.md) | TLS/SSL configuration and vulnerability scanner |
| sslyze | [`sslyze/`](sslyze/README.md) | Fast Python TLS scanner with JSON output |

### Code / container / supply chain
| Tool | Directory | One-liner |
|------|-----------|-----------|
| semgrep | [`semgrep/`](semgrep/README.md) | Static analysis (SAST) with community & custom rules |
| trivy | [`trivy/`](trivy/README.md) | Vulnerability/misconfig/secret scanning for images, FS, IaC |

### Privacy / anonymization
| Tool | Directory | One-liner |
|------|-----------|-----------|
| torsocks | [`torsocks/`](torsocks/README.md) | Route a single app's TCP through Tor (SOCKS5) |
| tor | [`tor/`](tor/README.md) | Tor daemon: torrc, bridges, onion services, control port |
| tor-browser | [`tor-browser/`](tor-browser/README.md) | Hardened Firefox preconfigured for Tor |

---

## Recommended end-to-end bug-bounty pipeline

```bash
# 1. Passive subdomain enumeration
subfinder -d example.com -all -silent -o subs.txt

# 2. Probe live HTTP services, capture status/title/tech
httpx -l subs.txt -sc -title -td -tech-detect -follow-redirects -silent -o live.txt

# 3. Crawl live hosts for URLs/endpoints (JS-aware)
katana -list live.txt -jc -jsl -d 3 -silent -o urls.txt

# 4. Discover hidden parameters
arjun -i live.txt -oT params.txt

# 5. Scan for known CVEs/misconfigs with templates
nuclei -l live.txt -severity low,medium,high,critical -o nuclei.txt

# 6. XSS-focused pass on crawled URLs
dalfox file urls.txt --silence --format jsonl -o xss.jsonl

# 7. Manual deep-dive with Burp/ZAP/mitmproxy on anything interesting
```

## Wireless assessment pipeline

```bash
# 1. Discover networks and clients passively
sudo kismet -c wlan1

# 2. Capture PMKID/EAPOL for a target BSSID
sudo hcxdumptool -i wlan1 -w capture.pcapng --filterlist_ap=target.txt --filtermode=2

# 3. Convert to hashcat format
hcxpcapngtool -o hashes.22000 -E essids.txt capture.pcapng

# 4. Crack offline
hashcat -m 22000 hashes.22000 wordlist.txt

# 5. WPS path, only if WPS is exposed and in scope
sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6
```

## OSINT pipeline

```bash
# 1. Domain-centric harvesting
theHarvester -d example.com -b duckduckgo,bing,crtsh -l 500 -f harvest

# 2. Correlate discovered entities
spiderfoot -s example.com -t DOMAIN_NAME,IP_ADDRESS,EMAILADDR -q

# 3. Build usernames/emails into presence checks
sherlock --output results.txt username
holehe --only-used user@example.com
```

## Environment notes

- Everything here is installed system-wide and available to members of `sectools`.
- `afl-fuzz` and `arjun` are `root:sectools 750` via `dpkg-statoverride`.
- `wireshark` group membership is required for live packet capture.
- Wireless and MITM tools (`aircrack-ng`, `wifite`, `reaver`, `wash`, `iw`,
  `macchanger`, `bettercap`, `ettercap`, `dsniff`, `hcxdumptool`, `kismet`,
  `bully`, `eaphammer`, `wifiphisher`, `wpa_supplicant`, `wireless-tools`)
  require root for capture, injection, or interface changes. A limited
  NOPASSWD sudoers grant covers these and the other root-only tools (sshuttle,
  gophish, evilginx, setoolkit).
- Bluetooth tools (`bluez`: `bluetoothctl`, `hcitool`, `btmgmt`, `btmon`,
  `sdptool`, `rfcomm`) need a running `bluetoothd`, a `bluetooth`-group user or
  root, and an adapter that is not blocked by `rfkill`.
- Android tools (`adb`, `fastboot`) need a udev rule granting the logged-in
  user access to the device; without one, `adb devices` shows `no permissions`
  and `fastboot` waits for a device. `scrcpy` is a local build at
  `/usr/local/bin/scrcpy` that embeds `/usr/local/bin/adb`; export
  `ADB=/usr/bin/adb` so it uses the packaged client.
- Tor-related tools: use `torsocks <cmd>` for per-app Tor; the root-only
  `tor-mode` toggle provides transparent system-wide routing. A standalone
  `tor` instance needs a writable `DataDirectory` and free `SocksPort`.
- VPN clients (`openvpn`, `openvpn3-client`) need root or `CAP_NET_ADMIN` to
  create tun/tap devices and change routes.
- `strace -p` is limited by `kernel.yama.ptrace_scope`; attach to processes
  you own, or run the target under `strace` directly.
- Most tools write config/state under `$HOME` (`/home/user`). SET is the
  exception: it uses `/opt/set/src` and `/etc/setoolkit`.

## Safety & legality

Use only against systems you own or have **written authorization** to test.
Passive recon (subfinder, public data) is generally safe; active scanning
(nuclei, nikto, dalfox, ZAP active scan, netexec, Metasploit), wireless
transmission (deauth, rogue AP), and phishing simulations are not. Respect
scope, rate limits, and program rules.

---

*Generated for a Debian 13 pentest workstation. Help snapshots taken 2026-11-09 to 2026-11-10.*
