# wifiphisher — rogue access point and credential-phishing framework

- **Version:** 1.4 (upstream git, locally patched for Python 3.13)
- **Binary:** `/usr/local/bin/wifiphisher` → `/opt/wifiphisher/.venv/bin/wifiphisher`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/wifiphisher/wifiphisher
- **Docs:** https://wifiphisher.readthedocs.io/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Wifiphisher creates a rogue access point, optionally deauthenticates clients
from their real AP, and serves a phishing page that captures credentials or
other user input. It supports KARMA, known-beacon, WPS-PBC, handshake-capture,
and MAC-randomisation modes, with extensions that add further Wi-Fi attacks.

The upstream project had been Python 2-only in practice because its
`roguehostapd` dependency used `SafeConfigParser`, removed in Python 3.12.
This install contains a local compatibility patch: `roguehostapd` now imports
`ConfigParser as SafeConfigParser`, and `pyric`, `six`, `scapy`, `pbkdf2`, and
`tornado` are installed in `/opt/wifiphisher/.venv`.

## Use cases

| Situation | Why wifiphisher |
|-----------|-----------------|
| Authorized wireless assessment | Stand up a convincing captive portal and measure credential exposure |
| Client deauth resilience test | See whether clients auto-connect to a look-alike SSID |
| Phishing-awareness exercise | Capture submitted credentials in a controlled lab |
| WPS-PBC assessment | Detect push-button WPS registrar activity |
| Handshake verification | Check a captured WPA handshake against a passphrase |

## Requirements & install layout

- **Root** is required for AP mode, monitor mode, and deauthentication.
- Two wireless adapters are recommended: one AP-capable, one monitor-capable.
- `dnsmasq` and `iw` must be installed (`/usr/sbin/dnsmasq` is present).
- `roguehostapd` is the default AP implementation; `-fH` forces system
  `hostapd` instead.
- Source: `/opt/wifiphisher`; virtualenv: `/opt/wifiphisher/.venv`.
- The patched dependency source is `/opt/roguehostapd`.

## Basic syntax

```bash
# pick interfaces interactively
sudo wifiphisher

# explicit AP + deauth interfaces
sudo wifiphisher -aI wlan0 -eI wlan1

# skip AP selection and run one phishing scenario
sudo wifiphisher -aI wlan0 -eI wlan1 -p firmware_upgrade

# capture credentials to a known file and stop on first hit
sudo wifiphisher -aI wlan0 -eI wlan1 -p oauth_login -qS -cP /tmp/creds.log
```

## Flag reference

### Interface selection
| Flag | Meaning |
|------|---------|
| `-i, --interface` | One interface supporting both AP and monitor mode |
| `-aI, --apinterface` | Interface used to spawn the rogue AP |
| `-eI, --extensionsinterface` | Monitor-mode interface for deauth/extensions |
| `-iI, --internetinterface` | Interface connected to the internet |
| `-pI, --protectinterface` | Interface(s) NetworkManager must not touch |
| `-mI, --mitminterface` | Internet-connected interface used for MITM |
| `-iAM, --mac-ap-interface` | MAC address for the AP interface |
| `-iEM, --mac-extensions-interface` | MAC address for the extensions interface |
| `-iNM, --no-mac-randomization` | Do not change any MAC address |
| `-kN, --keepnetworkmanager` | Do not kill NetworkManager |

### Rogue AP and attack behaviour
| Flag | Meaning |
|------|---------|
| `-e, --essid` | Rogue AP ESSID; skips AP selection |
| `-dE, --deauth-essid` | Deauth every BSSID with this ESSID |
| `-dC, --deauth-channels` | Channels to deauthenticate |
| `-nD, --nodeauth` | Skip the deauthentication phase |
| `-nE, --noextensions` | Do not load any extensions |
| `-dK, --disable-karma` | Disable the KARMA attack |
| `-kB, --known-beacons` | Broadcast popular WLAN beacon frames |
| `-wP, --wps-pbc` | Monitor for WPS-PBC registrar button presses |
| `-wAI, --wpspbc-assoc-interface` | Interface used to associate to the WPS AP |
| `-cM, --channel-monitor` | Follow the target AP if it changes channel |
| `-fH, --force-hostapd` | Use system `hostapd` instead of roguehostapd |
| `--dnsmasq-conf` | Custom `dnsmasq.conf` path |

### Phishing and capture
| Flag | Meaning |
|------|---------|
| `-p, --phishingscenario` | Phishing scenario name (e.g. `firmware_upgrade`) |
| `-pE, --phishing-essid` | ESSID shown on the phishing page |
| `-pK, --presharedkey` | Add WPA/WPA2 protection to the rogue AP |
| `-pPD, --phishing-pages-directory` | Directory of custom phishing pages |
| `--payload-path` | Path served by payload scenarios |
| `-hC, --handshake-capture` | Verify a captured handshake (requires `cowpatty`) |
| `-lC, --lure10-capture` | Capture BSSIDs for the Lure10 attack |
| `-lE, --lure10-exploit` | Replay captured BSSIDs for Lure10 |
| `-qS, --quitonsuccess` | Exit after the first credential pair |

### Logging
| Flag | Meaning |
|------|---------|
| `--logging` | Log activity to a file |
| `-lP, --logpath` | Full path of the log file |
| `-cP, --credential-log-path` | File that stores captured credentials |

## Recipes

```bash
# 1. See what the tool would do without touching the radio
sudo wifiphisher --help

# 2. Interactive interface + AP selection (most reliable first run)
sudo wifiphisher -i wlan0

# 3. Single-adapter rogue AP + phishing page
sudo wifiphisher -i wlan0 -p firmware_upgrade

# 4. Two-adapter setup with deauth on selected channels
sudo wifiphisher -aI wlan0 -eI wlan1 -dC 1,6,11 -p oauth_login

# 5. Keep NetworkManager alive on a lab machine
sudo wifiphisher -aI wlan0 -eI wlan1 -kN -p firmware_upgrade

# 6. Verify an existing handshake against a passphrase
sudo wifiphisher -hC /evidence/handshake.pcap
```

## Output & pipelines

Captured credentials go to the path given by `-cP`; activity logs go to
`-lP`. Handshake verification consumes a pcap produced by `hcxdumptool`,
`airodump-ng`, or `tshark`. Combine with `hcxtools` for hash extraction and
`hashcat` for offline cracking.

## Gotchas & operational security

- **Radio transmission:** deauth and rogue-AP frames are illegal without
  explicit authorization. Use a Faraday-shielded lab or an isolated channel.
- The tool normally kills NetworkManager and randomises MAC addresses; use
  `-kN` / `-iNM` only when you understand the consequences.
- AP mode requires a driver that supports it; many internal cards do not.
- Captive-portal scenarios need DNS and DHCP to work; a broken `dnsmasq`
  config produces clients that associate but never see the page.
- Python 3 support here is a local patch. Re-check it after any `git pull`.

## Legal & safety

Use only against networks and devices you own or have **written permission**
to test, in a controlled RF environment. Never capture credentials belonging
to real users without a signed engagement scope.

## Related tools

- [`hcxdumptool/`](../hcxdumptool/README.md) — handshake/PMKID capture
- [`hcxtools/`](../hcxtools/README.md) — capture conversion and hashing
- [`eaphammer/`](../eaphammer/README.md) — 802.1X/EAP Evil Twin alternative
- [`bully/`](../bully/README.md) — WPS PIN attacks
- [`kismet/`](../kismet/README.md) — passive wireless discovery

## References

- https://github.com/wifiphisher/wifiphisher
- https://wifiphisher.readthedocs.io/
- https://github.com/wifiphisher/roguehostapd
