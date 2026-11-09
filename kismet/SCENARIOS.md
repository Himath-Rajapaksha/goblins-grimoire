# kismet — command scenarios for deep research

Installed build: Kismet 2026.11.0-76367fa from source at `/opt/kismet`.
Flags below come from `help.txt`; datasource options are documented upstream
rather than in `--help`.

## How to invoke

```text
@deep-research Which Kismet datasource options and capture helpers are reliable
on kernel 6.12 with common Atheros/MediaTek/RTL8812AU adapters, and which
kismetdb-to-* export best preserves 802.11 management frames for later
hcxtools processing? Cite Kismet docs, release notes, and adapter compatibility
lists; return exact -c syntax and export commands.
```

---

## Scenario 1 — start a survey

| # | Command | How to use |
|---|---------|------------|
| 1 | `sudo kismet -c wlan1` | Auto-detect the source and open the web UI |
| 2 | `sudo kismet -c wlan1:name=survey1` | Label the source for the UI |
| 3 | `sudo kismet -c wlan1:channel=6` | Pin one channel |
| 4 | `sudo kismet -c wlan1:hop=true` | Explicit channel hopping |
| 5 | `sudo kismet -c wlan1 -t site-survey` | Set the log title |

**Research questions**
- Which capture helper does Kismet pick for this adapter, and can it hop?
- Does pinning `channel=` disable hopping on all helper types?

**Sources:** Kismet datasource docs, `iw list`, `dmesg` driver messages.
**Output:** a running server and a `.kismet` log.

---

## Scenario 2 — headless and long-running capture

| # | Command | How to use |
|---|---------|------------|
| 6 | `sudo kismet -c wlan1 --no-ncurses -s` | No console, quiet stdout |
| 7 | `sudo kismet -c wlan1 --daemonize` | Detach into the background |
| 8 | `sudo kismet -c wlan1 -p /evidence/kismet` | Logs to an evidence directory |
| 9 | `sudo kismet -c wlan1 -n` | Disable logging entirely |
| 10 | `sudo kismet -c wlan1 --device-timeout=600` | Expire stale devices |

**Research questions**
- Does `--daemonize` survive the parent shell, and where does it log?
- Which log types are enabled by default (`-T` overrides them)?

**Sources:** `kismet_logging.conf`, upstream docs, process inspection.
**Output:** a background capture with bounded log growth.

---

## Scenario 3 — datasources beyond Wi-Fi

| # | Command | How to use |
|---|---------|------------|
| 11 | `sudo kismet -c wlan1 -c hci0` | Wi-Fi plus Bluetooth |
| 12 | `kismet -c /evidence/capture.pcap` | Replay a pcap |
| 13 | `sudo kismet -c rtl433` | RTL-SDR sensor source |
| 14 | `sudo kismet -c wlan1:type=linux_wifi` | Force a helper type |
| 15 | `ls /opt/kismet/bin/kismet_cap_*` | List installed capture helpers |

**Research questions**
- Which helpers need extra hardware or udev rules?
- Does pcap replay preserve radiotap metadata?

**Sources:** Kismet datasource docs, hardware notes.
**Output:** a multi-source database or a verified replay.

---

## Scenario 4 — export and convert

| # | Command | How to use |
|---|---------|------------|
| 16 | `kismetdb_statistics -i survey.kismet` | Summarise packet/device counts |
| 17 | `kismetdb_dump_devices -i survey.kismet --json -o devices.json` | Export devices |
| 18 | `kismetdb_to_pcap -i survey.kismet -o survey.pcap` | Convert for Wireshark |
| 19 | `kismetdb_to_wiglecsv -i survey.kismet -o survey.csv` | WiGLE CSV |
| 20 | `kismetdb_to_kml -i survey.kismet -o survey.kml` | Map output |
| 21 | `kismetdb_clean -i survey.kismet -o clean.kismet` | Vacuum/clean a database |

**Research questions**
- Does `kismetdb_to_pcap` preserve all management frames needed for EAPOL?
- Which export is acceptable to WiGLE without leaking client identifiers?

**Sources:** kismetdb tool docs, Wireshark validation.
**Output:** an analysable pcap, CSV, or JSON artefact.

---

## Scenario 5 — configuration and hardening

| # | Command | How to use |
|---|---------|------------|
| 22 | `sudo kismet -f /opt/kismet/etc/kismet.conf` | Explicit config file |
| 23 | `sudo kismet --override httpd` | Load `kismet_httpd.conf` overrides |
| 24 | `sudo kismet --confdir /opt/kismet/etc` | Alternate config directory |
| 25 | `sudo kismet --no-plugins` | Start without plugins |
| 26 | `grep -E 'httpd_\|log_prefix' /opt/kismet/etc/kismet_httpd.conf` | Inspect UI binding |

**Research questions**
- What is the minimum config to bind the UI to localhost only?
- Which plugins are safe to disable for a pure capture role?

**Sources:** shipped `.conf` files, upstream hardening notes.
**Output:** a minimal, localhost-only capture configuration.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Generic | `-v -h -c -f --override --homedir --confdir --datadir --daemonize --debug -s --no-line-wrap --no-*-wrapper --enable-ncurses --no-plugins` | 1–5 |
| Logging | `-T -t -p -n` | 2 |
| Device tracking | `--device-timeout` | 2 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — start a survey | 5 |
| 2 — headless capture | 5 |
| 3 — datasources | 5 |
| 4 — export/convert | 6 |
| 5 — config/hardening | 5 |
