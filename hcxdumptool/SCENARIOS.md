# hcxdumptool — command scenarios for deep research

Installed build: hcxdumptool 6.3.5. All flags below come from `help-usage.txt`;
channel/band rules are from the same snapshot.

## How to invoke

```text
@deep-research Which hcxdumptool 6.3.5 options and BPF filters maximise PMKID
and EAPOL capture on modern WPA2/WPA3-transition APs while minimising
deauthentication, and how do --attemptapmax / --attemptclientmax interact with
the passive modes? Cite the hcxdumptool docs, release notes, and hashcat wiki;
return an exact capture command and conversion pipeline.
```

---

## Scenario 1 — interface preparation

| # | Command | How to use |
|---|---------|------------|
| 1 | `hcxdumptool -L` | List interfaces that support monitor mode |
| 2 | `hcxdumptool -l` | Greppable tab-separated interface list |
| 3 | `hcxdumptool -I wlan1` | Detailed capability report for one adapter |
| 4 | `sudo hcxdumptool -m wlan1` | Set monitor mode and exit |
| 5 | `sudo systemctl stop NetworkManager wpa_supplicant` | Release the interface |

**Research questions**
- Which chipsets support active monitor mode (needed for `-A`)?
- Does monitor mode survive a NetworkManager restart on this distro?

**Sources:** hcxdumptool docs, `iw list`, driver documentation.
**Output:** a clean adapter in monitor mode.

---

## Scenario 2 — survey and broad capture

| # | Command | How to use |
|---|---------|------------|
| 6 | `sudo hcxdumptool -i wlan1 -F --rcascan=active` | Active channel scan, no dump |
| 7 | `sudo hcxdumptool -i wlan1 -F --rcascan=passive` | Passive scan only |
| 8 | `sudo hcxdumptool -i wlan1 -w survey.pcapng -F` | Capture on all channels |
| 9 | `sudo hcxdumptool -i wlan1 -w survey.pcapng -F --rds=1` | Sort display by hash status |
| 10 | `sudo hcxdumptool -i wlan1 -w survey.pcapng --tot=30` | Stop after 30 minutes |
| 11 | `sudo hcxdumptool -i wlan1 -w survey.pcapng --watchdogmax=300` | Exit after 5 min silence |

**Research questions**
- Does `-F` cover 6 GHz on this adapter, or must channels be listed explicitly?
- What packet loss appears at high channel-hop rates?

**Sources:** hcxdumptool docs, adapter datasheet, capture statistics.
**Output:** a channel map or a broad pcapng.

---

## Scenario 3 — targeted capture with filters

| # | Command | How to use |
|---|---------|------------|
| 12 | `hcxdumptool --bpfc="wlan addr3 aabbccddeeff" > ap.bpf` | Compile an AP filter |
| 13 | `sudo hcxdumptool -i wlan1 -w ap.pcapng --bpf=ap.bpf` | Capture only that AP |
| 14 | `printf 'CorpNet\n' > essids.txt` | Build an ESSID list |
| 15 | `sudo hcxdumptool -i wlan1 -w corp.pcapng --essidlist=essids.txt` | Seed probe/response ring buffer |
| 16 | `sudo hcxdumptool -i wlan1 -w corp.pcapng -c 6a,11a,36b` | Pin known channels |
| 17 | `sudo hcxdumptool -i wlan1 -w hash.pcapng --exitoneapol=3` | Exit at first PMKID/M2M3 |

**Research questions**
- Which `wlan addr` field selects the BSSID versus the station?
- Does a narrow BPF filter reduce CPU enough to sustain all-channel hopping?

**Sources:** `man pcap-filter`, hcxdumptool examples, CPU measurements.
**Output:** a small, targeted capture with one hash.

---

## Scenario 4 — reduce or disable transmission

| # | Command | How to use |
|---|---------|------------|
| 18 | `sudo hcxdumptool -i wlan1 -w passive.pcapng --attemptapmax=0` | Fully passive AP behaviour |
| 19 | `sudo hcxdumptool -i wlan1 -w passive.pcapng --attemptclientmax=0` | No client attacks |
| 20 | `sudo hcxdumptool -i wlan1 -w passive.pcapng --disable_deauthentication` | No deauth frames |
| 21 | `sudo hcxdumptool -i wlan1 -w passive.pcapng --disable_proberequest --disable_association` | Beacon-listen only |
| 22 | `sudo hcxdumptool -i wlan1 -w passive.pcapng --attemptapmax=2` | Limit requests per AP |

**Research questions**
- Is a PMKID ever returned without any association attempt?
- Which disable combination still yields EAPOL on WPA3-transition APs?

**Sources:** hcxdumptool docs, AP behaviour captures.
**Output:** a low-noise capture suitable for sensitive environments.

---

## Scenario 5 — GPS, automation, and conversion

| # | Command | How to use |
|---|---------|------------|
| 23 | `sudo hcxdumptool -i wlan1 -w wardrive.pcapng --gpsd --nmea_out=track.nmea` | GPS-tag a wardrive |
| 24 | `sudo hcxdumptool -i wlan1 -w wardrive.pcapng --nmea_dev=/dev/ttyACM0` | Serial GPS receiver |
| 25 | `sudo hcxdumptool -i wlan1 -w wardrive.pcapng --gpio_button=17 --gpio_statusled=27` | Pi button/LED control |
| 26 | `sudo hcxdumptool -i wlan1 -w wardrive.pcapng --ontot=poweroff --tot=60` | Power off at timeout |
| 27 | `hcxpcapngtool -o wardrive.22000 -E essids.txt wardrive.pcapng` | Convert to hashcat input |
| 28 | `gpsbabel -w -t -i nmea -f track.nmea -o gpx -F track.gpx` | Convert NMEA to GPX |

**Research questions**
- Which `--on*` actions are safe on a headless field device?
- Does `--nmea_pcapng` survive `hcxpcapngtool` conversion?

**Sources:** hcxdumptool docs, gpsbabel docs, field-test logs.
**Output:** a geotagged capture plus GPX track.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Core | `-i -w -c -f -F -t -A -m -L -l -I` | 1, 2 |
| Transmission | `--disable_* --proberesponsetx --essidlist --attempt*` | 4 |
| BPF | `--bpfc --bpf` | 3 |
| Termination | `--tot --exitoneapol --errormax --watchdogmax --rcascan --rds` | 2, 3, 5 |
| GPS/GPIO | `--gpsd --nmea_* --gpio_* --on*` | 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — interface prep | 5 |
| 2 — survey/capture | 6 |
| 3 — targeted capture | 6 |
| 4 — disable transmission | 5 |
| 5 — GPS/automation | 6 |
