# hcxdumptool — WPA/WPA2 handshake and PMKID capture

- **Version:** hcxdumptool 6.3.5 (C) 2024 ZeroBeat
- **Binary:** `/usr/bin/hcxdumptool`
- **Help snapshots:** [`help.txt`](help.txt) · [`help-usage.txt`](help-usage.txt)
- **Upstream:** https://github.com/ZerBea/hcxdumptool
- **Docs:** https://github.com/ZerBea/hcxdumptool/tree/master/docs

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

hcxdumptool puts a wireless adapter into monitor mode and actively requests
PMKIDs and EAPOL message pairs from access points and clients. It writes a
pcapng capture that `hcxpcapngtool` converts into hashcat mode 22000 input.
Unlike passive `airodump-ng`, it transmits association and probe frames to
solicit handshakes, so it is noisy and must be used only in scope.

## Use cases

| Situation | Why hcxdumptool |
|-----------|-----------------|
| WPA/WPA2-PSK assessment | Capture PMKID without waiting for a client |
| Client-connected AP | Capture a full 4-way handshake |
| Survey before cracking | Build a pcapng of all PMKIDs/EAPOL in range |
| Channel audit | Use `--rcascan` to map APs per channel |
| GPS-tagged wardrive | Pair with `--gpsd` or `--nmea_dev` |

## Requirements & install layout

- **Root**; the adapter must support monitor mode and frame injection.
- Installed from Debian package, version 6.3.5.
- Default output: `yyyyddmmhhmmss-<interface>.pcapng` in the current directory.
- Stop NetworkManager and `wpa_supplicant` on the target interface first.
- Channel syntax requires a band suffix: `1a` = 2.4 GHz, `36b` = 5 GHz,
  `5c` = 6 GHz.

## Basic syntax

```bash
# list usable interfaces
hcxdumptool -L

# passive/active channel scan, nothing written to disk
sudo hcxdumptool -i wlan1 -F --rcascan=active

# capture everything, all channels, sorted by latest PMKID/EAPOL
sudo hcxdumptool -i wlan1 -w capture.pcapng -F --rds=1

# target one BSSID via a BPF filter, stop at first hash
sudo hcxdumptool -i wlan1 -w target.pcapng -c 6a \
  --bpf="wlan addr3 aabbccddeeff" --exitoneapol=3
```

## Flag reference

### Core
| Flag | Meaning |
|------|---------|
| `-i <iface>` | Interface to use (mode, MAC, channel are changed) |
| `-w <file>` | Write pcapng; existing file is overwritten |
| `-c <chan>` | Fixed channel(s), band suffix mandatory (`1a`, `36b`) |
| `-f <freq>` | Fixed frequency in MHz (`2412`, `5180`) |
| `-F` | Use all frequencies supported by the interface |
| `-t <sec>` | Minimum stay time per channel (default 1) |
| `-A` | ACK incoming frames (needs active monitor mode) |
| `-m <iface>` | Set monitor mode and exit |
| `-L` / `-l` / `-I <iface>` | List interfaces (pretty / greppable / detailed) |

### Transmission control
| Flag | Meaning |
|------|---------|
| `--disable_deauthentication` | Never send deauth/disassoc frames |
| `--disable_proberequest` | Never send probe requests |
| `--disable_association` | Never authenticate/associate |
| `--disable_reassociation` | Never reassociate a client |
| `--disable_beacon` | Disable the internal wildcard beacon |
| `--proberesponsetx=<n>` | Number of probe responses to send (default 5) |
| `--essidlist=<file>` | Seed the ESSID ring buffer |
| `--attemptclientmax=<n>` | Max M2 requests per client (0 disables client attacks) |
| `--attemptapmax=<n>` | Max beacons before PMKID/handshake request (0 = passive) |

### BPF filters
| Flag | Meaning |
|------|---------|
| `--bpfc=<expr>` | Compile a pcap-filter expression and print BPF bytecode |
| `--bpf=<file>` | Load compiled BPF bytecode |

```bash
# build and use a filter for one AP MAC
hcxdumptool --bpfc="wlan addr3 aabbccddeeff" > target.bpf
sudo hcxdumptool -i wlan1 -w target.pcapng --bpf=target.bpf
```

### Termination and scan behaviour
| Flag | Meaning |
|------|---------|
| `--tot=<min>` | Timeout timer in minutes |
| `--exitoneapol=<mask>` | Exit on first matching hash: 1 PMKID, 2 M2M3, 4 M1M2, 8 M1 |
| `--errormax=<n>` | Max errors before exit (default 100) |
| `--watchdogmax=<sec>` | Exit when no packets arrive (default 600) |
| `--rcascan=a\|p` | Channel assignment scan, active or passive (no dump) |
| `--rds=<0\|1>` | Real-time display sort order |

### GPS and hardware
| Flag | Meaning |
|------|---------|
| `--gpsd` | Read position from gpsd (NMEA0183 mode) |
| `--nmea_dev=<dev>` | Read NMEA from a serial device at 9600 baud |
| `--nmea_out=<file>` | Write NMEA track (GPRMC/GPGGA/GPWPL/GPTXT) |
| `--nmea_pcapng` | Embed GPS data in the pcapng |
| `--gpio_button=<pin>` | Raspberry Pi GPIO terminate button |
| `--gpio_statusled=<pin>` | Raspberry Pi GPIO status LED |
| `--on{term,gpiobutton,tot,watchdog,error}=<action>` | `poweroff` or `reboot` on that event |

## Recipes

```bash
# 1. Identify the adapter and confirm monitor-mode support
hcxdumptool -I wlan1

# 2. Stop interfering services
sudo systemctl stop NetworkManager wpa_supplicant

# 3. Broad capture for 30 minutes
sudo timeout 1800 hcxdumptool -i wlan1 -w survey.pcapng -F --tot=30

# 4. Target one ESSID via a filter list
printf 'CorpNet\n' > essids.txt
sudo hcxdumptool -i wlan1 -w corp.pcapng -c 6a,11a --essidlist=essids.txt

# 5. Clientless PMKID only, exit at first usable hash
sudo hcxdumptool -i wlan1 -w pmkid.pcapng --attemptclientmax=0 --exitoneapol=1

# 6. Convert to hashcat input
hcxpcapngtool -o capture.22000 -E essids.txt capture.pcapng
```

## Output & pipelines

- **pcapng** capture → `hcxpcapngtool -o hashes.22000` → `hashcat -m 22000`.
- `-E` extracts ESSID wordlists for rule-based attacks.
- `--nmea_out` → `gpsbabel` → GPX/KML.
- Do **not** clean or merge the pcapng with third-party tools; hcxtools needs
  the original radiotap/comment metadata.

## Gotchas & operational security

- **Running `hcxdumptool` with no arguments starts an active scan.** Always pass
  `-i` and a mode.
- Deauthentication and association frames are radio transmissions and are
  illegal without authorization in most jurisdictions.
- The tool changes the interface MAC and channel; restore both afterwards.
- `--exitoneapol` values are a bitmask: 3 = PMKID + authorized M2M3.
- PMKID capture is clientless but not universal; some APs never emit one.
- Large `-F` runs generate big pcapng files and constant radio noise.

## Legal & safety

Use only on networks you own or have **written permission** to test, in a
controlled RF environment. Never deauth a production network. Captured
handshakes are credentials-adjacent data — encrypt evidence at rest.

## Related tools

- [`hcxtools/`](../hcxtools/README.md) — conversion and hash filtering
- [`kismet/`](../kismet/README.md) — passive discovery
- [`wifiphisher/`](../wifiphisher/README.md) — rogue AP
- [`bully/`](../bully/README.md) — WPS
- [`eaphammer/`](../eaphammer/README.md) — EAP attacks

## References

- https://github.com/ZerBea/hcxdumptool
- https://github.com/ZerBea/hcxdumptool/tree/master/docs
- https://hashcat.net/wiki/doku.php?id=cracking_wpawpa2
