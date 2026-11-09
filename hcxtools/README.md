# hcxtools — capture conversion and hash preparation suites

- **Version:** hcxtools 6.3.5 (C) 2024 ZeroBeat
- **Binaries:** `/usr/bin/hcxpcapngtool`, `/usr/bin/hcxhashtool`,
  `/usr/bin/hcxpsktool`, `/usr/bin/hcxeiutool`, `/usr/bin/hcxwltool`,
  `/usr/bin/hcxpmktool`
- **Help snapshots:** [`help-hcxpcapngtool.txt`](help-hcxpcapngtool.txt),
  [`help-hcxhashtool.txt`](help-hcxhashtool.txt),
  [`help-hcxpsktool.txt`](help-hcxpsktool.txt),
  [`help-hcxeiutool.txt`](help-hcxeiutool.txt),
  [`help-hcxwltool.txt`](help-hcxwltool.txt),
  [`help-hcxpmktool.txt`](help-hcxpmktool.txt)
- **Upstream:** https://github.com/ZerBea/hcxtools

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

hcxtools converts wireless captures into password-cracking input and
manipulates the resulting hashes and wordlists. `hcxpcapngtool` is the entry
point: it reads pcapng/pcap/cap and writes hashcat `-m 22000` lines, ESSID and
identity lists, device inventories, GPS tracks, and access-point CSVs. The
other tools filter hashes, derive vendor-specific PSK candidates, build
targeted wordlists, format wordlists, and verify a candidate PSK/PMK offline.

## Use cases

| Situation | Binary |
|-----------|--------|
| Convert a capture for hashcat | `hcxpcapngtool` |
| Keep only PMKID or only EAPOL | `hcxhashtool --type=` |
| Filter by ESSID, MAC, OUI, vendor | `hcxhashtool` |
| Deduplicate / group hashes by ESSID | `hcxhashtool --essid-group` |
| Generate default-PSK candidates | `hcxpsktool` |
| Build rule-based wordlists from ESSIDs | `hcxeiutool` |
| Normalise a wordlist to a charset | `hcxwltool` |
| Verify a candidate PSK against a hash | `hcxpmktool` |
| Export AP inventory as CSV | `hcxpcapngtool --csv=` |
| Export GPS track | `hcxpcapngtool --nmea=` |

## Requirements & install layout

- Installed from Debian package 6.3.5; all binaries under `/usr/bin`.
- Conversion is CPU/IO bound but needs no radio or root.
- Uses the **original** pcapng. Do not clean, merge, or re-save it with
  third-party tools: optional comment fields carry the metadata hcxtools needs.
- Output files are appended, not truncated — delete old outputs first when
  re-running.

## Basic syntax

```bash
# capture -> hashcat input + ESSID list
hcxpcapngtool -o capture.22000 -E essids.txt capture.pcapng

# inspect what a hash file contains
hcxhashtool -i capture.22000 --info=stdout

# keep only PMKID hashes for one ESSID
hcxhashtool -i capture.22000 -o pmkid.22000 --type=1 --essid="CorpNet"

# generate vendor-default PSK candidates
hcxpsktool -c capture.22000 -o candidates.txt --netgear --weakpass

# verify a candidate PSK offline
hcxpmktool -l '<hash line>' -e 'CorpNet' -p 'candidate-password'
```

## Flag reference

### hcxpcapngtool — conversion

| Flag | Meaning |
|------|---------|
| `-o <file>` | WPA-PBKDF2-PMKID+EAPOL hash file (hashcat `-m 22000`) |
| `-E <file>` | ESSID wordlist from all frames |
| `-R <file>` | ESSID wordlist from probe requests only |
| `-I <file>` | Identity list |
| `-U <file>` | Username list |
| `-D <file>` | Device information list |
| `--all` | Convert every possible hash, not just the best pair |
| `--eapoltimeout=<ms>` | EAPOL pair timeout (default 5000) |
| `--nonce-error-corrections=<n>` | Nonce correction (default 0) |
| `--ignore-ie` | Ignore cipher/AKM info (can create uncrackable hashes) |
| `--max-essids=<n>` | Max ESSIDs per hash (default 1) |
| `--eapmd5=<file>` | EAP-MD5 challenge (hashcat 4800) |
| `--eapleap=<file>` | EAP-LEAP/MSCHAPv2 (hashcat 5500) |
| `--tacacs-plus=<file>` | TACACS+ v1 (hashcat 16100) |
| `--nmea=<file>` | GPS track in NMEA 0183 |
| `--csv=<file>` | AP inventory: time, MAC, ESSID, encryption, channel, RSSI, GPS |
| `--log=<file>` | Conversion log |
| `--raw-out=<file>` / `--raw-in=<file>` | Hex-ASCII frame export/import |
| `--lts=<file>` | BSSID list with Linux timestamps and RSSI |
| `--pmkid=<file>` | Deprecated PMKID-only format |
| `--pmkid-client=<file>` | Mesh/repeater PMKID format |
| `--hccapx=<file>` / `--hccap=<file>` / `--john=<file>` | Legacy formats |
| `--prefix=<file>` | Write every list type with one prefix |
| `--add-timestamp` | Prefix hash lines with time and EAPOL gap |

### hcxhashtool — filtering and inspection

| Flag | Meaning |
|------|---------|
| `-i <file>` / `-o <file>` | Input / output hash file |
| `-E <file>` | Filtered ESSID list |
| `-L <file>` | Unfiltered, unsorted ESSID list (hashcat `-a 9`) |
| `-d` | Download the IEEE OUI list to `~/.hcxtools/oui.txt` |
| `--essid-group` | Split hashes into ESSID groups (PBKDF2 reuse) |
| `--oui-group`, `--mac-group-ap`, `--mac-group-client` | Group by OUI/MAC |
| `--type=<mask>` | 1 PMKID, 2 EAPOL, 3 both |
| `--essid=<s>`, `--essid-part=<s>`, `--essid-partx=<s>` | ESSID filters |
| `--essid-list=<file>`, `--essid-regex=<re>` | ESSID list/regex filters |
| `--essid-len`, `--essid-min`, `--essid-max` | Length filters |
| `--mac-ap=<mac>`, `--mac-client=<mac>` | MAC filters |
| `--mac-list=<file>`, `--mac-skiplist=<file>` | MAC allow/exclude lists |
| `--oui-ap=<oui>`, `--oui-client=<oui>` | OUI filters |
| `--vendor=<s>`, `--vendor-ap=<s>`, `--vendor-client=<s>` | Vendor filters |
| `--authorized` / `--challenge` / `--apless` | EAPOL status filters |
| `--rc` / `--rc-not` | Replay-count checked / unchecked |
| `--hcx-min=<n>`, `--hcx-max=<n>` | Occurrence thresholds |
| `--info=<file>`, `--info=stdout` | Detailed hash-file report |
| `--info-vendor[-ap\|-client]=<file>` | Vendor report |
| `--psk=<psk>`, `--pmk=<pmk>` | Test a key (no nonce correction) |
| `--hccapx-in/out`, `--hccap-in/out`, `--john=` | Legacy conversion |
| `--vendorlist` | Print the whole OUI list |

### hcxpsktool — PSK candidate generation

| Flag | Meaning |
|------|---------|
| `-c <file>` | Input hashcat 22000/22001 hash file |
| `-i <file>` | Input EAPOL hash file (2500/2501) |
| `-j <file>` | Input John EAPOL file |
| `-z <file>` | Input PMKID file |
| `-e <essid>` | ESSID |
| `-b <mac>` | AP MAC |
| `-o <file>` | Output candidate list (default stdout) |
| `--maconly` | MAC-derived candidates only |
| `--noessidcombination` | Skip ESSID combinations |
| `--netgear`, `--spectrum`, `--digit10`, `--phome`, `--tenda`, `--ee`, `--eeupper`, `--alticeoptimum`, `--asus` | Vendor-default candidate sets (multi-GB) |
| `--weakpass`, `--eudate`, `--usdate`, `--wpskeys`, `--egn`, `--simple` | Weak-pattern candidate sets |

### hcxeiutool — ESSID-derived wordlists

| Flag | Meaning |
|------|---------|
| `-i <file>` | Input ESSID list |
| `-d <file>` | Digit-only wordlist |
| `-x <file>` | Hex-digit wordlist |
| `-c <file>` | Character-only wordlist (others removed) |
| `-s <file>` | Character wordlist with others replaced by `0x0a` (rule-friendly) |

### hcxwltool — wordlist formatting

| Flag | Meaning |
|------|---------|
| `-i <file>` / `-o <file>` | Input / output wordlist |
| `--straight` | Pass through unchanged |
| `--digit` / `--xdigit` | Digits / hex digits only |
| `--lower` / `--upper` / `--capital` | Case transforms |
| `--length=<n>` | Password length 8–32 |

### hcxpmktool — verification

| Flag | Meaning |
|------|---------|
| `-l <hash line>` | Hashcat 22000 line to test |
| `-e <essid>` | Network name |
| `-p <psk>` | PSK or PMK; `-p -` reads from stdin |
| Exit codes | 0 confirmed, 1 error, 2 unconfirmed |

## Recipes

```bash
# 1. Convert and inventory
hcxpcapngtool -o cap.22000 -E essids.txt -D devices.txt --csv=aps.csv cap.pcapng

# 2. See hash composition
hcxhashtool -i cap.22000 --info=stdout

# 3. PMKID-only corpus
hcxhashtool -i cap.22000 -o pmkid.22000 --type=1

# 4. Group by ESSID to exploit PBKDF2 reuse
cd workdir && hcxhashtool -i ../cap.22000 --essid-group

# 5. Vendor candidates plus weak patterns
hcxpsktool -c cap.22000 -o psk.txt --netgear --weakpass --simple

# 6. ESSID-derived rule wordlist
hcxeiutool -i essids.txt -d digits.txt -x xdigits.txt -c chars.txt -s sc.txt
cat essids.txt digits.txt xdigits.txt chars.txt sc.txt | sort -u > wl.txt
hashcat --stdout -r rules/best64.rule chars.txt >> wl.txt

# 7. Normalise and crack
hcxwltool -i wl.txt --lower --length=12 | sort -u > wl12.txt
hashcat -m 22000 cap.22000 wl12.txt

# 8. Verify one candidate without hashcat
hcxpmktool -l "$(head -1 cap.22000)" -e "CorpNet" -p "Summer2026!"
```

## Output & pipelines

- `*.22000` → hashcat `-m 22000` or John `wpapsk-*`.
- ESSID/identity/user lists feed `hcxeiutool`, `hcxwltool`, and hashcat rules.
- CSV/GPS exports feed mapping and reporting.
- `hcxpcapngtool` also emits EAP-MD5/LEAP/TACACS+ formats for legacy audits.

## Gotchas & operational security

- Output is **appended**; stale files silently mix runs.
- `--ignore-ie` and nonce error corrections can manufacture uncrackable hashes.
- `--all` multiplies output size; filter afterwards with `hcxhashtool`.
- Vendor wordlists from `hcxpsktool` can exceed several GB.
- Never clean or re-save the pcapng outside Wireshark/tshark.
- Hash files and ESSIDs are sensitive: they identify networks and clients.

## Legal & safety

Cracking captures is only lawful for networks you own or are authorized to
test. Captured handshakes and derived PSKs are credentials — store them
encrypted and delete them when the engagement ends.

## Related tools

- [`hcxdumptool/`](../hcxdumptool/README.md) — capture
- [`kismet/`](../kismet/README.md) — alternate capture source
- [`wifiphisher/`](../wifiphisher/README.md) — handshake verification path
- [`hashcat`] — external cracker (not documented here)

## References

- https://github.com/ZerBea/hcxtools
- https://hashcat.net/wiki/doku.php?id=cracking_wpawpa2
- https://pcapng.com/
