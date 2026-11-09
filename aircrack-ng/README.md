# aircrack-ng — 802.11 capture, injection, and key-cracking suite

- **Version:** aircrack-ng 1.7 (Debian package
  `aircrack-ng` `1:1.7+git20230807.4bf83f1a-2`)
- **Binaries:** `/usr/bin/aircrack-ng`, `/usr/bin/airdecap-ng`,
  `/usr/bin/airdecloak-ng`, `/usr/bin/airolib-ng`, `/usr/bin/airscan-discover`,
  `/usr/bin/ivstools`, `/usr/bin/kstats`, `/usr/bin/makeivs-ng`,
  `/usr/bin/packetforge-ng`; `/usr/sbin/airbase-ng`,
  `/usr/sbin/aireplay-ng`, `/usr/sbin/airmon-ng`, `/usr/sbin/airodump-ng`,
  `/usr/sbin/airodump-ng-oui-update`, `/usr/sbin/airserv-ng`,
  `/usr/sbin/airtun-ng`, `/usr/sbin/airventriloquist-ng`
- **Help snapshots:** [`help.txt`](help.txt) — `aircrack-ng --help`;
  [`help-airmon-ng.txt`](help-airmon-ng.txt);
  [`help-airodump-ng.txt`](help-airodump-ng.txt);
  [`help-aireplay-ng.txt`](help-aireplay-ng.txt);
  [`help-airbase-ng.txt`](help-airbase-ng.txt);
  [`help-airdecap-ng.txt`](help-airdecap-ng.txt);
  [`help-airolib-ng.txt`](help-airolib-ng.txt);
  [`help-packetforge-ng.txt`](help-packetforge-ng.txt);
  [`help-airserv-ng.txt`](help-airserv-ng.txt)
- **Upstream:** https://www.aircrack-ng.org/ ·
  https://github.com/aircrack-ng/aircrack-ng
- **Docs:** https://www.aircrack-ng.org/doku.php

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Aircrack-ng is a suite, not a single program. The pieces split into four
stages:

| Stage | Tool | Role |
|-------|------|------|
| Adapter setup | `airmon-ng` | Create and tear down monitor-mode interfaces |
| Discovery | `airodump-ng`, `airscan-discover` | Find APs, clients, and WPS data |
| Injection | `aireplay-ng`, `airbase-ng`, `packetforge-ng` | Generate and replay frames |
| Cracking | `aircrack-ng`, `airolib-ng`, `airdecap-ng` | Recover WEP/WPA keys, decrypt captures |

Supporting tools fill gaps: `airdecloak-ng` removes WEP cloaking,
`ivstools` merges IVS files, `makeivs-ng` synthesises IVS files, `kstats`
reports on aircrack-ng's key-rank progress, `airserv-ng` shares a radio over
TCP, `airtun-ng` builds a tunnel over 802.11, and `airventriloquist-ng`
injects at layer 2.

The suite is built for **WEP and WPA/WPA2-PSK**. It is not a WPA3/SAE cracker
and does not perform PMKID cracking itself — it can export material for
hashcat (`-I` PMKID string, `-j` HCCAPX, `-J` HCCAP) and read an airolib-ng
database (`-r`).

## `airmon-ng`

```text
airmon-ng <start|stop|check> <interface> [channel or frequency]
```

This build prints only the usage line for `--help`, so the operations are
documented from the tool's behaviour and the upstream wiki:

| Command | Meaning |
|---------|---------|
| `airmon-ng` | List wireless interfaces and their drivers/chipsets |
| `airmon-ng check` | Report processes that can interfere (NetworkManager, wpa_supplicant, dhclient) |
| `airmon-ng check kill` | Kill those processes |
| `airmon-ng start IFACE [CHANNEL]` | Create a monitor interface, optionally pinned to a channel |
| `airmon-ng stop IFACE` | Remove the monitor interface |

Modern kernels and drivers often allow monitor mode directly on the managed
interface, in which case `airmon-ng start` may report that no additional
interface is needed.

## `airodump-ng`

```text
airodump-ng <options> <interface>[,<interface>,...]
```

| Option | Meaning |
|--------|---------|
| `--ivs` | Save only IVs |
| `--gpsd` | Use GPSd |
| `--write PREFIX` | Dump file prefix |
| `--beacons` | Record all beacons in the dump |
| `--update SECS` | Display update delay |
| `--showack` | Print ACK/CTS/RTS statistics |
| `-h` | Hide known stations for `--showack` |
| `-f MSECS` | Channel-hop interval |
| `--berlin SECS` | Time before removing idle AP/client (default 120) |
| `-r FILE` | Read packets from a file |
| `--real-time` | Replay a file at live arrival rate |
| `-x MSECS` | Active scanning simulation |
| `--manufacturer` | Resolve IEEE OUI vendors |
| `--uptime` | AP uptime from beacon timestamp |
| `--wps` | Display WPS information |
| `--output-format FORMATS` | `pcap`, `ivs`, `csv`, `gps`, `kismet`, `netxml`, `logcsv` |
| `--ignore-negative-one` | Suppress the fixed-channel `-1` message |
| `--write-interval SECS` | Output write interval |
| `--background ENABLE` | Override background detection |

**Filters:** `--encrypt SUITE`, `--netmask`, `--bssid`, `--essid`,
`--essid-regex`, `--min-packets`, `--min-power`, `--min-rxq`, `-a`
(unassociated stations only), `-z` (associated stations only).

**Channel control:** `--channel`, `-C FREQS`, `--band abg`, `--ht20`,
`--ht40-`, `--ht40+`, `--ignore-other-chans`, `--cswitch 0|1|2`. By default
airodump-ng hops 2.4 GHz channels.

## `aireplay-ng`

```text
aireplay-ng <options> <replay interface>
```

| Mode | Flag | Number | Purpose |
|------|------|--------|---------|
| Deauth | `--deauth COUNT` | `-0` | Deauthenticate one or all stations |
| Fake auth | `--fakeauth DELAY` | `-1` | Associate with an AP |
| Interactive | `--interactive` | `-2` | Choose frames by hand |
| ARP replay | `--arpreplay` | `-3` | Standard ARP-request replay |
| Chopchop | `--chopchop` | `-4` | Decrypt a WEP packet |
| Fragment | `--fragment` | `-5` | Generate valid keystream |
| Caffe-Latte | `--caffe-latte` | `-6` | Query a client for new IVs |
| Cfrag | `--cfrag` | `-7` | Fragment against a client |
| Migration mode | `--migmode` | `-8` | Attack WPA migration mode |
| Test | `--test` | `-9` | Test injection and quality |

Key options by group:

| Group | Options |
|-------|---------|
| Filters | `-b bssid`, `-d dmac`, `-s smac`, `-m minlen`, `-n maxlen`, `-u type`, `-v subtype`, `-t tods`, `-f fromds`, `-w iswep`, `-D` |
| Replay | `-x nbpps`, `-p fctrl`, `-a bssid`, `-c dmac`, `-h smac`, `-g value`, `-F` |
| Fake auth | `-e essid`, `-o npckts`, `-q sec`, `-Q`, `-y prga`, `-T n` |
| ARP replay | `-j` (inject FromDS packets) |
| Fragmentation | `-k IP`, `-l IP` |
| Source | `-i iface`, `-r file` |
| Misc | `-R`, `--ignore-negative-one`, `--deauth-rc 0-254` (default 7) |

## `airbase-ng`

```text
airbase-ng <options> <replay interface>
```

| Option | Meaning |
|--------|---------|
| `-a bssid` | Access point MAC |
| `-i iface` | Capture interface |
| `-w WEP key` | Encrypt/decrypt with this WEP key |
| `-h MAC` | Source MAC for MITM mode |
| `-f disallow` | Disallow these client MACs |
| `-W 0\|1` | Set or clear the WEP flag in beacons |
| `-q`, `-v` | Quiet / verbose |
| `-A` | Ad-hoc mode |
| `-Y in\|out\|both` | External packet processing |
| `-c channel` | AP channel |
| `-X` | Hidden ESSID |
| `-s` | Force shared-key authentication |
| `-S` | Shared-key challenge length (default 128) |
| `-L` | Caffe-Latte attack |
| `-N` | Cfrag attack (recommended) |
| `-x nbpps` | Packets per second (default 100) |
| `-y` | Do not answer broadcast probes |
| `-0` | Set all WPA/WEP/open tags |
| `-z type` | WPA1 tag type (1 WEP40, 2 TKIP, 3 WRAP, 4 CCMP, 5 WEP104) |
| `-Z type` | Same for WPA2 |
| `-V type` | Fake EAPOL (1 MD5, 2 SHA1, 3 auto) |
| `-F prefix` | Write sent/received frames to a pcap |
| `-P` | Answer all probes |
| `-I interval` | Beacon interval in ms |
| `-C seconds` | Beacon probed ESSIDs (requires `-P`) |
| `-n hex` | User ANonce for the 4-way handshake |

**Filters:** `--bssid`, `--bssids FILE`, `--client`, `--clients FILE`,
`--essid` (default `default`), `--essids FILE`.

## `aircrack-ng`

```text
aircrack-ng [options] <input file(s)>
```

### Common

| Option | Meaning |
|--------|---------|
| `-a amode` | Force mode: 1 = WEP, 2 = WPA-PSK |
| `-e essid` | Target network name |
| `-b bssid` | Target AP MAC |
| `-p nbcpu` | CPUs to use (default all) |
| `-q` | Quiet |
| `-C macs` | Merge APs into a virtual one |
| `-l file` | Write the key to a file (overwrites) |

### Static WEP

| Option | Meaning |
|--------|---------|
| `-c` | Alphanumeric key characters only |
| `-t` | Binary-coded decimal characters only |
| `-h` | Fritz!Box numeric key search |
| `-d mask` | Key mask, e.g. `A1:XX:CF:YY` |
| `-m maddr` | Filter usable packets by MAC |
| `-n nbits` | Key length: 64/128/152/256/512 |
| `-i index` | Key index 1–4 |
| `-f fudge` | Brute-force fudge factor (default 2) |
| `-k korek` | Disable one attack method (1–17) |
| `-x`, `-x0`, `-x1`, `-x2` | Last-keybyte brute-force control |
| `-X` | Disable brute-force multithreading |
| `-y` | Experimental single brute-force mode |
| `-K` | Only old KoreK attacks (pre-PTW) |
| `-s` | Show the key in ASCII |
| `-M num` | Maximum IVs to use |
| `-D` | WEP decloak, skip broken keystreams |
| `-P num` | PTW debug (1 disables Klein, 2 PTW) |
| `-1` | Single PTW attempt |
| `-V` | Visual inspection mode |

### WEP and WPA-PSK

| Option | Meaning |
|--------|---------|
| `-w words` | Wordlist path(s) |
| `-N file` | New session file |
| `-R file` | Resume session file |

### WPA-PSK

| Option | Meaning |
|--------|---------|
| `-E file` | EWSA Project v3 |
| `-I str` | PMKID string (hashcat `-m 16800`) |
| `-j file` | Hashcat v3.6+ HCCAPX |
| `-J file` | Hashcat HCCAP |
| `-S` | Speed test |
| `-Z sec` | Speed-test duration |
| `-r DB` | airolib-ng database (cannot combine with `-w`) |

### SIMD

`--simd-list` prints available architectures; `--simd=<option>` forces one:
`generic`, `avx512`, `avx2`, `avx`, `sse2`, `altivec`, `power8`, `asimd`,
`neon`. `-u` prints CPU count and SIMD support.

## `airdecap-ng`

```text
airdecap-ng [options] <pcap file>
```

| Option | Meaning |
|--------|---------|
| `-l` | Keep the 802.11 header |
| `-b bssid` | AP MAC filter |
| `-e essid` | Target SSID |
| `-o fname` | Output file (default `<src>-dec`) |
| `-w key` | WEP key in hex |
| `-c fname` | Corrupted-WEP output (default `<src>-bad`) |
| `-p pass` | WPA passphrase |
| `-k pmk` | WPA PMK in hex |

If the capture contains WDS packets, `-b` is mandatory; otherwise only
packets destined to the AP are decrypted.

## `airolib-ng`

```text
airolib-ng <database> <operation> [options]
```

| Operation | Meaning |
|-----------|---------|
| `--stats` | Database information |
| `--sql SQL` | Execute SQL |
| `--clean [all]` | Remove junk; `all` also shrinks and integrity-checks |
| `--batch` | Precompute all ESSID/password combinations |
| `--verify [all]` | Verify random PMKs; `all` deletes invalid ones |
| `--import essid\|passwd FILE` | Import a text list |
| `--import cowpatty FILE` | Import cowpatty data |
| `--export cowpatty ESSID FILE` | Export cowpatty data |

An airolib-ng database stores precomputed PMKs, so cracking the same ESSID
against a large password list is much faster on the second run. `aircrack-ng
-r DB` reads it, and `-w` cannot be combined with `-r`.

## `packetforge-ng`

```text
packetforge-ng <mode> <options>
```

| Mode | Flag | Number |
|------|------|--------|
| ARP | `--arp` | `-0` |
| UDP | `--udp` | `-1` |
| ICMP | `--icmp` | `-2` |
| Null | `--null` | `-3` |
| Custom | `--custom` | `-9` |

Forge options: `-p fctrl`, `-a bssid`, `-c dmac`, `-h smac`, `-j` (FromDS),
`-o` (clear ToDS), `-e` (disable WEP), `-k ip[:port]`, `-l ip[:port]`,
`-t ttl`, `-w file`, `-s size`, `-n packets`. Source options: `-r file`
(raw packet) and `-y file` (PRGA).

## `airserv-ng`

```text
airserv-ng <options>
```

| Option | Meaning |
|--------|---------|
| `-h` | Help |
| `-p port` | TCP port (default 666) |
| `-d iface` | Wireless interface |
| `-c chan` | Channel |
| `-v level` | Debug level 1–3 (default 1) |

`airserv-ng` exposes a radio over TCP so aircrack-ng tools can run on another
host. It is unauthenticated and unencrypted — bind it only to a trusted
network.

## Recipes

**1. Check the adapter and interfering processes.**

```bash
sudo airmon-ng
sudo airmon-ng check
sudo airmon-ng check kill
```

**2. Create and remove a monitor interface.**

```bash
sudo airmon-ng start wlan0
sudo airmon-ng stop wlan0mon
```

**3. Pin a monitor interface to channel 6.**

```bash
sudo airmon-ng start wlan0 6
```

**4. Discover APs and clients, writing a pcap.**

```bash
sudo airodump-ng --write /tmp/survey wlan0mon
```

**5. Capture one AP on one channel.**

```bash
sudo airodump-ng --bssid AA:BB:CC:DD:EE:FF -c 6 --write /tmp/target wlan0mon
```

**6. Show WPS data during a survey.**

```bash
sudo airodump-ng --wps --manufacturer wlan0mon
```

**7. Capture with a filter and a CSV output.**

```bash
sudo airodump-ng --band abg --output-format csv,pcap --write /tmp/survey wlan0mon
```

**8. Replay a capture at live rate.**

```bash
airodump-ng --real-time -r /tmp/survey-01.cap
```

**9. Test injection quality.**

```bash
sudo aireplay-ng -9 -i wlan0mon wlan0mon
```

**10. Deauthenticate one client.**

```bash
sudo aireplay-ng -0 5 -a AA:BB:CC:DD:EE:FF -c 11:22:33:44:55:66 wlan0mon
```

**11. Fake-authenticate to an AP.**

```bash
sudo aireplay-ng -1 0 -e TargetSSID -a AA:BB:CC:DD:EE:FF wlan0mon
```

**12. ARP-request replay to generate IVs.**

```bash
sudo aireplay-ng -3 -b AA:BB:CC:DD:EE:FF -h 11:22:33:44:55:66 wlan0mon
```

**13. Crack WEP from a capture.**

```bash
aircrack-ng -b AA:BB:CC:DD:EE:FF -n 64 /tmp/target-01.cap
```

**14. Crack WPA-PSK from a handshake.**

```bash
aircrack-ng -w /usr/share/wordlists/rockyou.txt -e TargetSSID /tmp/target-01.cap
```

**15. Export a handshake for hashcat.**

```bash
aircrack-ng -j /tmp/target.hccapx /tmp/target-01.cap
aircrack-ng -I 'PMKID_STRING' -E /tmp/target.ewsa /tmp/target-01.cap
```

**16. Precompute PMKs with airolib-ng, then crack from the database.**

```bash
airolib-ng /tmp/pmk.db --import essid /tmp/essids.txt
airolib-ng /tmp/pmk.db --import passwd /usr/share/wordlists/rockyou.txt
airolib-ng /tmp/pmk.db --batch
aircrack-ng -r /tmp/pmk.db /tmp/target-01.cap
```

**17. Decrypt a WEP capture.**

```bash
airdecap-ng -w 1A2B3C4D5E -e TargetSSID -b AA:BB:CC:DD:EE:FF /tmp/target-01.cap
```

**18. Decrypt a WPA capture.**

```bash
airdecap-ng -p 'passphrase' -e TargetSSID /tmp/target-01.cap
```

**19. Forge an ARP packet from a PRGA file.**

```bash
packetforge-ng --arp -a AA:BB:CC:DD:EE:FF -h 11:22:33:44:55:66 \
  -k 192.0.2.1 -l 192.0.2.50 -y /tmp/keystream.xor -w /tmp/arp.cap
```

**20. Share a radio over TCP.**

```bash
sudo airserv-ng -d wlan0mon -p 666
```

## Gotchas

- **`airmon-ng --help` prints only the usage line.** It is not a flag parser;
  read the upstream wiki for operations.
- **`airmon-ng check kill` kills NetworkManager and wpa_supplicant.** That
  drops your own connectivity; restore the services when you are done.
- **Monitor mode is driver-dependent.** Check `iw list` for `monitor` under
  supported interface modes before blaming the suite.
- **Channel pinning matters.** Hopping while capturing a handshake or
  injecting loses frames; use `-c`/`--channel` on both capture and injection.
- **`airodump-ng` defaults to 2.4 GHz.** Use `--band abg` or `-C` for 5 GHz.
- **`--wps` is passive.** It reads WPS information elements from beacons and
  probe responses; it does not attack WPS.
- **`-h` means two different things.** In `airodump-ng` it hides known
  stations; in `aireplay-ng` and `airbase-ng` it sets the source MAC. Read
  the context.
- **`-w` means two different things.** In `aircrack-ng` it is a wordlist; in
  `airdecap-ng` and `airbase-ng` it is a WEP key.
- **`aireplay-ng -0` is disruptive.** Deauth drops clients; use the smallest
  count that produces the handshake and record it.
- **`-9` is the safe first injection test.** It measures injection quality
  without attacking a client.
- **WEP is broken, WPA2-PSK is not.** WEP cracking is statistical; WPA
  cracking is a dictionary/brute-force problem and succeeds only if the
  passphrase is weak.
- **PMKID export is not cracking.** `-I` writes the hashcat string; you still
  need hashcat mode 22000/16800 to attack it.
- **`-r DB` and `-w` are mutually exclusive.** airolib-ng databases already
  contain the PMKs.
- **`airolib-ng --batch` is expensive.** Precomputation cost grows with
  ESSID × password count; size the list deliberately.
- **`airdecap-ng` needs the key, not the handshake.** It decrypts with a known
  WEP key or WPA passphrase/PMK.
- **WDS captures need `-b`.** Without the AP MAC filter, only packets
  destined to the AP are decrypted.
- **`packetforge-ng -y` needs a valid PRGA.** A keystream from the wrong
  network or length produces unusable frames.
- **`airserv-ng` has no authentication.** Anyone who can reach the port gets
  your radio. Bind to localhost or a trusted tunnel.
- **`airtun-ng` and `airventriloquist-ng` are specialist tools.** They are
  installed but not covered by the captured help; read the upstream wiki
  before use.
- **`airodump-ng-oui-update` needs network access.** It refreshes the OUI
  database used by `--manufacturer`.
- **Captures contain handshakes and traffic.** Treat `.cap` files as sensitive
  engagement evidence and delete them at close.

## Related docs

- [`wifite`](../wifite/README.md) — automated front end over this suite
- [`reaver`](../reaver/README.md) — WPS PIN attacks
- [`pixiewps`](../pixiewps/README.md) — offline WPS Pixie-Dust
- [`wash`](../wash/README.md) — WPS survey
- [`hcxdumptool`](../hcxdumptool/README.md) — PMKID and handshake capture
- [`hcxtools`](../hcxtools/README.md) — convert captures for hashcat
- [`hashcat`](../hashcat/README.md) — GPU cracking of exported hashes
- [`john`](../john/README.md) — CPU cracking alternative
- [`bettercap`](../bettercap/README.md) — modern Wi-Fi recon and attacks
- [`kismet`](../kismet/README.md) — passive wireless detection
- [`iw`](../iw/README.md) — inspect and configure the radio
- [`macchanger`](../macchanger/README.md) — change the interface MAC
- [`wireshark`](../wireshark/README.md) — inspect 802.11 frames
