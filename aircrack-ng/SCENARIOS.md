# aircrack-ng — command scenarios for deep research

Installed build: aircrack-ng 1.7 (Debian package
`1:1.7+git20230807.4bf83f1a-2`). Every flag below appears in one of the nine
captured help files. `airmon-ng` prints only its usage line, so its operations
are documented from behaviour and the upstream wiki.

> **Scope:** Monitor mode, deauthentication, injection, and fake access points
> all affect other users' radios. Run them only on networks you are authorized
> to test, on channels you are allowed to transmit on, and with a stop plan
> that restores managed mode and any services killed by `airmon-ng check kill`.

## How to invoke

```text
@deep-research For an authorized 802.11 assessment, compare the aircrack-ng
1.7 suite with wifite 2.7.0 and bettercap 2.33.0 for WPA2-PSK capture and
cracking. How do the tools differ in handshake capture reliability, PMKID
support, channel handling, and export formats for hashcat? What does each do
when the adapter cannot inject, and which steps fail silently? Cite the
aircrack-ng wiki, the wifite documentation, the bettercap Wi-Fi module docs,
IEEE 802.11, and the hashcat example-hashes page; return a stage-by-stage
comparison with confidence notes.
```

Create an evidence workspace and record the radio state:

```bash
mkdir -p ~/aircrack-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/aircrack-demo/evidence/run-start.txt
iw dev > ~/aircrack-demo/evidence/iw-dev-before.txt
iw list > ~/aircrack-demo/evidence/iw-list.txt
rfkill list > ~/aircrack-demo/evidence/rfkill.txt
```

## Scenario 1 — inventory the suite and the adapter

The suite is 17 binaries in two directories, and the adapter's capabilities
decide which of them will work.

| # | Command | How to use |
|---|---------|------------|
| 1 | `aircrack-ng --help` | Main cracking help |
| 2 | `aircrack-ng -u` | CPU count and SIMD support |
| 3 | `aircrack-ng --simd-list` | Available SIMD architectures |
| 4 | `dpkg-query -W -f='${Version}\n' aircrack-ng` | Installed package version |
| 5 | `dpkg -L aircrack-ng \| grep -E '/(s?bin)/'` | All shipped binaries |
| 6 | `iw list \| sed -n '/Supported interface modes/,/^$/p'` | Does the driver support monitor mode? |
| 7 | `iw list \| grep -A4 'Supported interface modes'` | Check for `monitor` |
| 8 | `sudo airmon-ng` | Interfaces, drivers, chipsets |

**Research questions**
- Which suite binaries are in `/usr/sbin` and why?
- Which SIMD path does this CPU use, and how much does it change cracking
  speed?
- What does `iw list` report that `airmon-ng` does not?
- Which binaries are not covered by the captured help files?
- What is the difference between monitor mode and promiscuous mode?

**Sources:** `aircrack-ng --help`; `aircrack-ng -u`; `iw list`; Debian package
contents; IEEE 802.11.
**Output:** a binary inventory and an adapter capability record.

## Scenario 2 — monitor mode and interference control

`airmon-ng` creates the monitor interface, but killing NetworkManager and
wpa_supplicant is a separate, reversible step.

| # | Command | How to use |
|---|---------|------------|
| 9 | `sudo airmon-ng check` | List interfering processes |
| 10 | `sudo airmon-ng check kill` | Kill them (drops your own Wi-Fi) |
| 11 | `sudo airmon-ng start wlan0` | Create a monitor interface |
| 12 | `ip -brief link` | See the new interface name |
| 13 | `sudo airmon-ng start wlan0 6` | Create it pinned to channel 6 |
| 14 | `iw dev wlan0mon info` | Verify mode, channel, type |
| 15 | `sudo airmon-ng stop wlan0mon` | Remove it |
| 16 | `sudo systemctl restart NetworkManager` | Restore connectivity |

**Research questions**
- Which processes does `check` flag, and why does each interfere?
- Does this driver need a separate monitor interface, or does it support
  monitor mode on the managed interface?
- What changes when the interface is created on a fixed channel?
- What is lost by killing NetworkManager, and how do you restore it?
- How would you verify that monitor mode is actually receiving frames?

**Sources:** `airmon-ng` usage; `iw dev`; `iw list`; upstream airmon-ng wiki;
NetworkManager and wpa_supplicant documentation.
**Output:** a monitor-mode runbook with a verified restore step.

## Scenario 3 — discovery with `airodump-ng`

Discovery produces the BSSID, channel, client, and encryption facts every later
step depends on.

| # | Command | How to use |
|---|---------|------------|
| 17 | `sudo airodump-ng wlan0mon` | Default 2.4 GHz survey |
| 18 | `sudo airodump-ng --band abg wlan0mon` | 2.4, 5, and 6 GHz bands |
| 19 | `sudo airodump-ng --wps --manufacturer wlan0mon` | WPS and vendor data |
| 20 | `sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF wlan0mon` | One AP, one channel |
| 21 | `sudo airodump-ng --essid-regex '^Corp' wlan0mon` | Regex ESSID filter |
| 22 | `sudo airodump-ng --encrypt WPA2 --min-power -70 wlan0mon` | Cipher and signal filters |
| 23 | `sudo airodump-ng -a wlan0mon` | Unassociated stations only |
| 24 | `sudo airodump-ng --output-format csv,pcap --write /tmp/survey wlan0mon` | CSV plus pcap |
| 25 | `airodump-ng --real-time -r /tmp/survey-01.cap` | Replay at live rate |

**Research questions**
- What does the default channel hop cover, and what does `--band abg` add?
- Which filter flags compose, and which conflict?
- What is in `--output-format logcsv` that plain `csv` lacks?
- How does `--real-time` change file replay for tool testing?
- Which columns come from beacons and which from data frames?

**Sources:** `airodump-ng --help`; IEEE 802.11 beacon and probe frames;
upstream airodump-ng documentation.
**Output:** a survey dataset with an AP/client inventory and channel map.

## Scenario 4 — injection, deauth, and handshake capture

Injection is the dividing line between passive and active testing. Test it
before attacking.

| # | Command | How to use |
|---|---------|------------|
| 26 | `sudo aireplay-ng -9 -i wlan0mon wlan0mon` | Injection and quality test |
| 27 | `sudo aireplay-ng -9 -e TargetSSID -a AA:BB:CC:DD:EE:FF wlan0mon` | Test against one AP |
| 28 | `sudo aireplay-ng -0 5 -a AA:BB:CC:DD:EE:FF wlan0mon` | Deauth all clients, 5 bursts |
| 29 | `sudo aireplay-ng -0 1 -a AA:BB:CC:DD:EE:FF -c 11:22:33:44:55:66 wlan0mon` | Deauth one client |
| 30 | `sudo aireplay-ng -0 5 --deauth-rc 7 -a AA:BB:CC:DD:EE:FF wlan0mon` | Explicit reason code |
| 31 | `sudo aireplay-ng -1 0 -e TargetSSID -a AA:BB:CC:DD:EE:FF wlan0mon` | Fake authentication |
| 32 | `sudo aireplay-ng -3 -b AA:BB:CC:DD:EE:FF -h 11:22:33:44:55:66 wlan0mon` | ARP replay |
| 33 | `sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w /tmp/hs wlan0mon` | Capture the handshake |
| 34 | `aircrack-ng /tmp/hs-01.cap` | Did we get a handshake? |

**Research questions**
- What does `-9` measure, and what does a partial result mean?
- Why is a targeted deauth preferable to a broadcast deauth?
- What does the reason code change, and which codes are commonly filtered?
- Why does fake authentication matter for some attacks and not others?
- How do you confirm a complete 4-way handshake from the cap file?

**Sources:** `aireplay-ng --help`; IEEE 802.11 authentication and deauth
frames; upstream injection test documentation.
**Output:** an injection-quality record and a verified handshake capture.

## Scenario 5 — WEP attacks and packet forging

WEP is obsolete but still appears in legacy equipment. The attack chain is
statistical and needs IVs, not a wordlist.

| # | Command | How to use |
|---|---------|------------|
| 35 | `sudo aireplay-ng -4 -b AA:BB:CC:DD:EE:FF -h 11:22:33:44:55:66 wlan0mon` | Chopchop |
| 36 | `sudo aireplay-ng -5 -b AA:BB:CC:DD:EE:FF -h 11:22:33:44:55:66 wlan0mon` | Fragmentation |
| 37 | `sudo aireplay-ng -6 -b AA:BB:CC:DD:EE:FF -h 11:22:33:44:55:66 wlan0mon` | Caffe-Latte |
| 38 | `sudo aireplay-ng -7 -b AA:BB:CC:DD:EE:FF -h 11:22:33:44:55:66 wlan0mon` | Cfrag |
| 39 | `sudo aireplay-ng -8 -b AA:BB:CC:DD:EE:FF -h 11:22:33:44:55:66 wlan0mon` | WPA migration mode |
| 40 | `packetforge-ng --arp -a AA:BB:CC:DD:EE:FF -h 11:22:33:44:55:66 -k 192.0.2.1 -l 192.0.2.50 -y /tmp/keystream.xor -w /tmp/arp.cap` | Forge an ARP packet |
| 41 | `packetforge-ng --udp -a AA:BB:CC:DD:EE:FF -h 11:22:33:44:55:66 -k 192.0.2.1:53 -l 192.0.2.50:5353 -y /tmp/keystream.xor -w /tmp/udp.cap` | Forge a UDP packet |
| 42 | `aircrack-ng -b AA:BB:CC:DD:EE:FF -n 64 -K /tmp/wep-01.cap` | Old KoreK-only crack |
| 43 | `aircrack-ng -b AA:BB:CC:DD:EE:FF -n 128 -x2 /tmp/wep-01.cap` | Two-keybyte brute force |

**Research questions**
- Which WEP attack needs a client, and which works with only the AP?
- What does chopchop produce, and how is it used downstream?
- Why is `-K` (old KoreK) sometimes slower but more reliable?
- What does `-x2` change in the search space?
- Why is WEP cracking a statistical attack rather than a dictionary attack?

**Sources:** `aireplay-ng --help`; `aircrack-ng --help`; `packetforge-ng
--help`; the original WEP attacks (FMS, KoreK, PTW); IEEE 802.11.
**Output:** an IV/keystream record and a recovered WEP key with its source.

## Scenario 6 — cracking, exporting, and decrypting

The cracking stage is offline. Export formats matter because hashcat is
usually faster than aircrack-ng for WPA.

| # | Command | How to use |
|---|---------|------------|
| 44 | `aircrack-ng -w /usr/share/wordlists/rockyou.txt -e TargetSSID /tmp/hs-01.cap` | Dictionary attack |
| 45 | `aircrack-ng -w list1.txt,list2.txt /tmp/hs-01.cap` | Multiple wordlists |
| 46 | `aircrack-ng -S -Z 10 /tmp/hs-01.cap` | Speed test for 10 seconds |
| 47 | `aircrack-ng -j /tmp/hs.hccapx /tmp/hs-01.cap` | Export HCCAPX for hashcat |
| 48 | `aircrack-ng -J /tmp/hs.hccap /tmp/hs-01.cap` | Export legacy HCCAP |
| 49 | `aircrack-ng -I 'PMKID' -E /tmp/hs.ewsa /tmp/hs-01.cap` | Export EWSA/PMKID |
| 50 | `airdecap-ng -p 'passphrase' -e TargetSSID /tmp/hs-01.cap` | Decrypt the capture |
| 51 | `airolib-ng /tmp/pmk.db --import essid /tmp/essids.txt` | Build a PMK database |
| 52 | `airolib-ng /tmp/pmk.db --batch` | Precompute PMKs |
| 53 | `aircrack-ng -r /tmp/pmk.db /tmp/hs-01.cap` | Crack from the database |
| 54 | `airdecap-ng -w 1A2B3C4D5E -b AA:BB:CC:DD:EE:FF /tmp/wep-01.cap` | Decrypt WEP |

**Research questions**
- When is hashcat faster than aircrack-ng, and what does the export lose?
- What is the difference between HCCAP and HCCAPX?
- Why can `-r DB` not be combined with `-w`?
- What does `--batch` cost in time and disk for a given ESSID × password set?
- What does `airdecap-ng` need that `aircrack-ng` does not?

**Sources:** `aircrack-ng --help`; `airdecap-ng --help`; `airolib-ng --help`;
hashcat example-hashes; upstream airolib-ng documentation.
**Output:** a crack report with the export format, runtime, and key source.

## Scenario 7 — rogue AP, tunnelling, and remote radio

These are the specialist edges of the suite. Each changes the RF environment
or exposes hardware.

| # | Command | How to use |
|---|---------|------------|
| 55 | `sudo airbase-ng -e FreeWiFi -c 6 wlan0mon` | Beacon a rogue AP |
| 56 | `sudo airbase-ng -e Corp -c 6 -P -C 60 wlan0mon` | Answer probes and beacon probed ESSIDs |
| 57 | `sudo airbase-ng -e Corp -c 6 -Z 4 wlan0mon` | Advertise WPA2-CCMP tags |
| 58 | `sudo airbase-ng -e Corp -c 6 -F /tmp/rogue wlan0mon` | Record all frames |
| 59 | `sudo airserv-ng -d wlan0mon -p 666` | Share the radio over TCP |
| 60 | `sudo airtun-ng -a AA:BB:CC:DD:EE:FF wlan0mon` | Build an 802.11 tunnel |
| 61 | `sudo airventriloquist-ng -d wlan0mon` | Layer-2 injection |
| 62 | `sudo airdecloak-ng -i /tmp/wep-01.cap --ssid TargetSSID` | Remove WEP cloaking |
| 63 | `ivstools --merge /tmp/a.ivs /tmp/b.ivs /tmp/merged.ivs` | Merge IVS files |

**Research questions**
- What does a beacon-only rogue AP lack compared with a real AP?
- Why does `-P -C` change the probe-response behaviour?
- What does `-Z 4` advertise, and how would a client verify it?
- What are the risks of exposing `airserv-ng` on a network?
- What does WEP cloaking do, and why does `airdecloak-ng` exist?

**Sources:** `airbase-ng --help`; `airserv-ng --help`; upstream wiki for
`airtun-ng`, `airventriloquist-ng`, `airdecloak-ng`, and `ivstools`; IEEE
802.11; WEP cloaking research.
**Output:** a rogue-AP and tooling record with an exposure assessment.

## Flag coverage checklist

| Help file / flag | Scenario |
|------------------|----------|
| `aircrack-ng` common `-a`, `-e`, `-b`, `-p`, `-q`, `-C`, `-l` | 6 |
| `aircrack-ng` WEP `-c`, `-t`, `-h`, `-d`, `-m`, `-n`, `-i`, `-f`, `-k`, `-x*`, `-X`, `-y`, `-K`, `-s`, `-M`, `-D`, `-P`, `-1`, `-V` | 5, 6 |
| `aircrack-ng` `-w`, `-N`, `-R` | 6 |
| `aircrack-ng` `-E`, `-I`, `-j`, `-J`, `-S`, `-Z`, `-r` | 6 |
| `aircrack-ng` `--simd-list`, `--simd`, `-u` | 1 |
| `airmon-ng` `start`, `stop`, `check` | 2 |
| `airodump-ng` output and display options | 3 |
| `airodump-ng` filters and channel options | 3 |
| `aireplay-ng` filters and replay options | 4 |
| `aireplay-ng` attack modes `-0`…`-9` | 4, 5 |
| `airbase-ng` options and filters | 7 |
| `airdecap-ng` `-l`, `-b`, `-e`, `-o`, `-w`, `-c`, `-p`, `-k` | 6 |
| `airolib-ng` `--stats`, `--sql`, `--clean`, `--batch`, `--verify`, `--import`, `--export` | 6 |
| `packetforge-ng` modes and forge/source options | 5 |
| `airserv-ng` `-p`, `-d`, `-c`, `-v` | 7 |

Full text: [`help.txt`](help.txt) and the eight `help-*.txt` files.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Suite and adapter inventory | 8 |
| 2 | Monitor mode and interference control | 8 |
| 3 | Discovery with `airodump-ng` | 9 |
| 4 | Injection, deauth, handshake capture | 9 |
| 5 | WEP attacks and packet forging | 9 |
| 6 | Cracking, exporting, decrypting | 11 |
| 7 | Rogue AP, tunnelling, remote radio | 9 |
| **Total** | | **63** |

## Legal and safety

Monitor mode and injection transmit on shared spectrum. Deauthentication,
rogue APs, and WEP injection disrupt other users and may be illegal without
authorization even where the network is your own. Work only on channels and
networks named in the engagement scope, keep transmit power low, and record
every deauth burst. Before leaving, stop injection, remove the monitor
interface, restart NetworkManager/wpa_supplicant, and delete captures that
contain third-party traffic. Handshakes and recovered keys are sensitive
evidence: encrypt them and keep only what the report requires.
