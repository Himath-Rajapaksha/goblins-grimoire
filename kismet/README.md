# kismet — wireless detector, sniffer, and IDS

- **Version:** Kismet 2026.11.0-76367fa (built from source)
- **Binary:** `/usr/local/bin/kismet` → `/opt/kismet/bin/kismet`
- **Support binaries:** `kismetdb_*` tools in `/usr/local/bin`
- **Help snapshot:** [`help.txt`](help.txt) · **Version:** [`version.txt`](version.txt)
- **Upstream:** https://github.com/kismetwireless/kismet
- **Docs:** https://www.kismetwireless.net/docs/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Kismet is a passive wireless framework: it puts supported adapters into
monitor mode, captures 802.11 (and Bluetooth, RF sensors, and other datasources),
tracks devices over time, detects anomalies, and exposes everything through a
web UI and a REST API. Captures are stored in the Kismet database format and can
be converted to pcap, KML, WiGLE CSV, GPX, and other formats.

This host has no Debian package for the current release, so Kismet was compiled
from source into `/opt/kismet`. All capture helpers (`kismet_cap_*`) live beside
the main binary.

## Use cases

| Situation | Why Kismet |
|-----------|-----------|
| Wireless site survey | Map APs, channels, encryption, and client associations |
| Rogue-AP detection | Alert on unexpected BSSIDs and SSID impersonation |
| Client profiling | Track probe requests and device movement over time |
| Capture for later cracking | Produce pcap/pcapng for `hcxtools` and `hashcat` |
| Bluetooth/BLE survey | Use `linux_bluetooth` datasources |
| RF sensor logging | RTL433, ADS-B, AIS, and LoRa datasources |

## Requirements & install layout

- **Root** for monitor-mode capture and raw sockets.
- A monitor-mode-capable adapter; internal cards often cannot switch modes.
- Source tree built with prefix `/opt/kismet`; config in `/opt/kismet/etc`.
- Default web UI: `http://127.0.0.1:2501` (login `kismet` / `kismet`;
  change it in `kismet_httpd.conf`).
- Default logs: `~/.kismet/logs/` unless `-p` or `log_prefix` changes it.
- Plugins and capture helpers are loaded from the install prefix.
- `LD_LIBRARY_PATH=/opt/kismet/lib` is only needed if you call the binary
  directly; the packaged rpath makes the symlinked `kismet` work.

## Basic syntax

```bash
# start with one Wi-Fi adapter, web UI on 2501
sudo kismet -c wlan1

# name the source and pin a channel
sudo kismet -c wlan1:name=wifi1,channel=6

# headless server, no console wrapper, logs to a chosen directory
sudo kismet -c wlan1 --no-ncurses -p /evidence/kismet -t engagement-01

# replay a pcap as a datasource
kismet -c /evidence/capture.pcap

# inspect a stored kismetdb
kismetdb_statistics -i /evidence/kismet/engagement-01.kismet
```

## Flag reference

### Generic
| Flag | Meaning |
|------|---------|
| `-v, --version` | Print version and exit |
| `-h, --help` | Print help |
| `-c <datasource>` | Add a capture datasource (repeatable) |
| `-f, --config-file <file>` | Use an alternate `kismet.conf` |
| `--override <flavor>` | Load `kismet_<flavor>.conf` or a specific override file |
| `--homedir <path>` | Alternate home directory for the Kismet user |
| `--confdir <path>` | Alternate config directory |
| `--datadir <path>` | Alternate data directory |
| `--no-plugins` | Do not load plugins |
| `--daemonize` | Detach and run in the background |
| `--debug` | Disable the console wrapper and crash handler |
| `-s, --silent` | Suppress stdout after setup |
| `--no-line-wrap` | Disable line wrapping (grep-friendly) |
| `--no-console-wrapper`, `--no-ncurses-wrapper`, `--no-ncurses` | Disable the ncurses console |
| `--enable-ncurses` | Force the ncurses console even if config disables it |

### Logging
| Flag | Meaning |
|------|---------|
| `-T, --log-types <types>` | Override which log types are written |
| `-t, --log-title <title>` | Set the log title / file stem |
| `-p, --log-prefix <prefix>` | Directory for log files |
| `-n, --no-logging` | Disable logging entirely |

### Device tracking
| Flag | Meaning |
|------|---------|
| `--device-timeout=n` | Expire devices after N seconds without packets |

## Datasource syntax

```text
-c <interface>                     # auto-detect source type
-c <interface>:name=<label>        # label shown in the UI
-c <interface>:channel=<n>         # lock a Wi-Fi channel
-c <interface>:hop=true            # channel hopping
-c <file.pcap>                     # pcap/pcapng replay
-c <interface>:type=linux_wifi     # force a capture helper
-c rtl433                          # RTL-SDR sensor source
```

List compatible helpers with `ls /opt/kismet/bin/kismet_cap_*`.

## Recipes

```bash
# 1. Passive Wi-Fi survey with a labelled source
sudo kismet -c wlan1:name=survey1 -t site-survey

# 2. Headless capture for a fixed engagement window
sudo timeout 3600 kismet -c wlan1 --no-ncurses -p /evidence/kismet -t wardrive

# 3. Serve only the web UI, no console clutter
sudo kismet -c wlan1 --no-ncurses -s

# 4. Replay an existing capture for analysis
kismet -c /evidence/handshake.pcap -p /tmp/kismet-replay -t replay

# 5. Export devices from a stored database
kismetdb_dump_devices -i /evidence/kismet/wardrive.kismet --json -o devices.json

# 6. Convert to pcap for Wireshark
kismetdb_to_pcap -i /evidence/kismet/wardrive.kismet -o wardrive.pcap

# 7. WiGLE CSV for upload/analysis
kismetdb_to_wiglecsv -i /evidence/kismet/wardrive.kismet -o wardrive.csv

# 8. Compress/clean a large database
kismetdb_clean -i big.kismet -o clean.kismet
```

## Output & pipelines

- `.kismet` SQLite database: devices, packets, alerts, GPS, messages.
- `kismetdb_to_pcap` → Wireshark/tshark, `hcxtools`.
- `kismetdb_to_wiglecsv` → WiGLE, spreadsheets.
- `kismetdb_to_kml` / `kismetdb_to_gpx` → maps.
- `kismetdb_dump_devices --json` → jq, reporting scripts.
- REST API on port 2501 (`/system/status`, `/devices/all_devices.json`, ...).

## Gotchas & operational security

- Monitor mode requires root and a supported adapter; `iw list` shows support.
- Kismet holds the interface in monitor mode — stop it before running
  `hcxdumptool` or `wifiphisher` on the same adapter.
- The default web credentials are public knowledge; change them before
  exposing the UI beyond localhost.
- Large surveys produce multi-GB databases; use `-p` on a roomy filesystem.
- Passive capture still records real client MAC addresses and probe SSIDs —
  treat the database as personal data.
- The source build has no package manager tracking; re-run `make install`
  after pulling a new release.

## Legal & safety

Passive capture of radio traffic is regulated differently by jurisdiction.
Capture only networks and devices you own or have **written authorization**
to assess. Never use captured client identifiers for tracking people.

## Related tools

- [`hcxdumptool/`](../hcxdumptool/README.md), [`hcxtools/`](../hcxtools/README.md)
- [`wifiphisher/`](../wifiphisher/README.md), [`eaphammer/`](../eaphammer/README.md)
- [`bully/`](../bully/README.md)
- [`wireshark/`](../wireshark/README.md)

## References

- https://www.kismetwireless.net/docs/
- https://github.com/kismetwireless/kismet
- https://www.kismetwireless.net/docs/readme/datasources/
