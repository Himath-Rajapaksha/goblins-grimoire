# wash — command scenarios for deep research

Installed build: Wash v1.6.6, shipped by the Debian `reaver` package
`1.6.6-2+b1`. Every option below appears in [`help.txt`](help.txt). Wash is
passive by default; only `-s` transmits.

> **Scope:** Survey mode only listens, but monitor mode still changes the
> adapter's state, and scan mode transmits probes. Use wash only on networks
> you are authorized to assess, and restore the adapter when you are done.

## How to invoke

```text
@deep-research For an authorized WPS survey, compare wash 1.6.6 with the
WPS discovery paths in wifite 2.7.0 and bettercap 2.33.0. Which fields does
wash report that the others do not, how do survey and scan modes differ in
reliability and detectability, and how should a WPS lock state be interpreted?
Cite the wash and reaver repositories, the WPS specification, and IEEE
802.11 beacon/probe formats; return a field-by-field comparison with
confidence notes.
```

Create an evidence workspace:

```bash
mkdir -p ~/wash-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/wash-demo/evidence/run-start.txt
iw dev > ~/wash-demo/evidence/iw-dev-before.txt
dpkg -S /usr/bin/wash > ~/wash-demo/evidence/package.txt
```

## Scenario 1 — inventory and passive survey

The default survey is the safe path. Confirm the package, the adapter, and the
output format before touching a target.

| # | Command | How to use |
|---|---------|------------|
| 1 | `wash -h` | All options |
| 2 | `dpkg -S /usr/bin/wash` | Which package ships wash |
| 3 | `dpkg-query -W -f='${Version}\n' reaver` | Package version |
| 4 | `iw dev` | Interface state |
| 5 | `sudo wash -i wlan0mon` | Default passive survey |
| 6 | `sudo wash -i wlan0mon -a` | Include non-WPS APs |
| 7 | `sudo wash -i wlan0mon -p` | Show crack progress |
| 8 | `sudo wash -i wlan0mon -2` | 2.4 GHz only |

**Research questions**
- Which columns come from beacons and which from probe responses?
- What does the WPS version field tell you about attack options?
- Why does `-a` produce a larger table, and what does it add?
- What does the lock field mean, and when is it set?
- Why is the default mode called "survey" rather than "scan"?

**Sources:** `wash -h`; WPS specification; IEEE 802.11 beacon and probe
response formats; Debian package metadata.
**Output:** a WPS inventory with version, lock state, and band.

## Scenario 2 — band, channel, and scan mode

Scan mode transmits. Treat it as an active step with a scope decision.

| # | Command | How to use |
|---|---------|------------|
| 9 | `sudo wash -i wlan0mon -c 6` | Pin channel 6 |
| 10 | `sudo wash -i wlan0mon -5` | 5 GHz survey |
| 11 | `sudo wash -i wlan0mon -2 -c 1` | 2.4 GHz, channel 1 |
| 12 | `sudo wash -i wlan0mon -s` | Active scan mode |
| 13 | `sudo wash -i wlan0mon -s -n 5` | Cap probes per AP |
| 14 | `sudo wash -i wlan0mon -s -n 30` | More probes per AP |
| 15 | `sudo wash -i wlan0mon -O /tmp/wps-scan.pcap` | Save interesting packets |
| 16 | `sudo wash -i wlan0mon -F` | Ignore FCS errors |

**Research questions**
- What changes in the frame stream between survey and scan mode?
- How does `-n` trade time for discovery reliability?
- Why would a 5 GHz AP be missed by a default survey?
- What does `-O` capture that the table does not show?
- When is `-F` legitimate, and when does it hide a real problem?

**Sources:** `wash -h`; IEEE 802.11 probe request/response; upstream wash
documentation.
**Output:** a band/channel coverage record with probe counts.

## Scenario 3 — offline capture analysis

Wash can read pcaps, which makes it usable without a radio and reproducible
after the fact.

| # | Command | How to use |
|---|---------|------------|
| 17 | `wash -f capture.pcap` | Survey one capture |
| 18 | `wash -f capture1.pcap capture2.pcap capture3.pcap` | Merge several |
| 19 | `wash -f capture.pcap -j` | JSON output |
| 20 | `wash -f capture.pcap -a` | Include non-WPS APs |
| 21 | `wash -f capture.pcap -p` | Show crack progress |
| 22 | `tcpdump -r capture.pcap -nn 2>/dev/null \| head` | Confirm capture readability |
| 23 | `tshark -r capture.pcap -Y wps 2>/dev/null \| head` | Locate WPS frames |
| 24 | `sha256sum capture.pcap` | Hash the source |

**Research questions**
- Which fields are available offline that a live survey cannot show?
- How does wash handle captures from multiple channels?
- What is the difference between `-f` and `-i` in output?
- Why hash the capture before analysis?
- Which WPS messages must be present for a complete result?

**Sources:** `wash -h`; pcap file format; `tshark` display filters.
**Output:** an offline survey table tied to a hashed capture.

## Scenario 4 — JSON output and parsing

`-j` is the machine-readable path. Use it when the result feeds a report or a
pipeline.

| # | Command | How to use |
|---|---------|------------|
| 25 | `sudo wash -i wlan0mon -j > /tmp/wps.json` | Save JSON |
| 26 | `python3 -m json.tool /tmp/wps.json \| head -40` | Validate and pretty-print |
| 27 | `jq '.[0]' /tmp/wps.json` | Inspect one record |
| 28 | `jq '.[] \| select(.wps_locked == true)' /tmp/wps.json` | Find locked APs |
| 29 | `jq '.[] \| {bssid, essid, wps_version}' /tmp/wps.json` | Extract key fields |
| 30 | `wc -l /tmp/wps.json` | Record output size |
| 31 | `sha256sum /tmp/wps.json` | Hash the result |

**Research questions**
- What extended WPS information does `-j` add over the table?
- Which JSON fields map to the table columns?
- How are locked APs represented?
- What does an empty result look like?
- How stable is the schema across versions?

**Sources:** `wash -h`; upstream wash source; `jq` manual.
**Output:** a validated JSON dataset with a documented schema.

## Scenario 5 — UTF-8, checksums, and edge cases

These flags change what you see, and two of them are explicitly risky.

| # | Command | How to use |
|---|---------|------------|
| 32 | `sudo wash -i wlan0mon -U` | Show raw UTF-8 ESSIDs |
| 33 | `sudo wash -i wlan0mon -U -j` | UTF-8 plus JSON |
| 34 | `sudo wash -i wlan0mon -F -p` | Ignore FCS, show progress |
| 35 | `sudo wash -i wlan0mon -a -j \| jq 'length'` | Count all APs |
| 36 | `sudo wash -i wlan0mon -c 0` | Behaviour on an invalid channel |
| 37 | `wash -f /nonexistent.pcap` | Behaviour on a missing file |
| 38 | `sudo wash -i nonexistent0` | Behaviour on a missing interface |

**Research questions**
- Why is `-U` described as dangerous?
- What does `-F` change in the parsed frame stream?
- How does wash report an invalid channel or interface?
- Which errors are fatal and which are warnings?
- What should a wrapper script check before parsing output?

**Sources:** `wash -h`; upstream wash source; terminal escape-sequence
handling.
**Output:** an edge-case log with exit statuses.

## Scenario 6 — handoff to reaver and cleanup

Wash's job ends where reaver's begins. Record the handoff explicitly.

| # | Command | How to use |
|---|---------|------------|
| 39 | `sudo wash -i wlan0mon -j > ~/wash-demo/evidence/wps.json` | Freeze the target list |
| 40 | `jq -r '.[] \| select(.wps_locked == false) \| .bssid' ~/wash-demo/evidence/wps.json` | Unlocked targets |
| 41 | `sudo reaver -i wlan0mon -b <bssid> -K -vv` | Hand off to Pixie-Dust |
| 42 | `sudo wash -i wlan0mon` | Re-survey after the attack |
| 43 | `iw dev` | Confirm interface state |
| 44 | `sudo airmon-ng stop wlan0mon` | Remove monitor interface |
| 45 | `sudo systemctl restart NetworkManager` | Restore connectivity |
| 46 | `ls -l ~/wash-demo/evidence/` | Collect artifacts |

**Research questions**
- Which unlocked APs are good Pixie-Dust candidates?
- What does a post-attack survey show about lock state?
- How do you prove the adapter was restored?
- Which artifacts belong in the report?
- What must be deleted at close?

**Sources:** `wash -h`; `reaver -h`; `iw dev`; `airmon-ng` usage.
**Output:** a target handoff record and a cleanup checklist.

## Flag coverage checklist

| Flag | Scenario |
|------|----------|
| `-i`, `-f` | 1, 3 |
| `-c` | 2 |
| `-n` | 2 |
| `-O` | 2 |
| `-F` | 2, 5 |
| `-2`, `-5` | 2 |
| `-s`, `-u` | 2 |
| `-a` | 1, 3, 5 |
| `-j` | 3, 4, 5 |
| `-U` | 5 |
| `-p` | 1, 3, 5 |
| `-h` | 1 |

Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Inventory and passive survey | 8 |
| 2 | Band, channel, and scan mode | 8 |
| 3 | Offline capture analysis | 8 |
| 4 | JSON output and parsing | 7 |
| 5 | UTF-8, checksums, edge cases | 7 |
| 6 | Handoff to reaver and cleanup | 8 |
| **Total** | | **46** |

## Legal and safety

Wash is mostly passive, but monitor mode changes the adapter and scan mode
transmits probes. Use it only on networks you are authorized to assess. Before
leaving, stop the survey, remove the monitor interface, restart
NetworkManager/wpa_supplicant, and delete captures that contain third-party
traffic. The JSON target list and any pcaps are engagement evidence: hash
them, encrypt them, and keep only what the report requires.
