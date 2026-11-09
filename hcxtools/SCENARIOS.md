# hcxtools — command scenarios for deep research

Installed build: hcxtools 6.3.5. Flags come from the six `help-hcx*.txt`
snapshots in this directory.

## How to invoke

```text
@deep-research For hcxtools 6.3.5, which hcxpcapngtool conversion options and
hcxhashtool filters produce the smallest crackable hash set for a mixed
WPA2/WPA3-transition capture, and when do --nonce-error-corrections or --all
help versus create uncrackable hashes? Cite the hcxtools docs, hashcat wiki,
and project issues; return an exact conversion/filter chain.
```

---

## Scenario 1 — convert a capture

| # | Command | How to use |
|---|---------|------------|
| 1 | `hcxpcapngtool -o cap.22000 cap.pcapng` | Basic hashcat conversion |
| 2 | `hcxpcapngtool -o cap.22000 -E essids.txt cap.pcapng` | Also export ESSIDs |
| 3 | `hcxpcapngtool -o cap.22000 -D devices.txt cap.pcapng` | Device inventory |
| 4 | `hcxpcapngtool --csv=aps.csv cap.pcapng` | AP inventory CSV |
| 5 | `hcxpcapngtool --nmea=track.nmea cap.pcapng` | GPS track export |
| 6 | `hcxpcapngtool --prefix=out cap.pcapng` | Every list type at once |

**Research questions**
- Which fields does `--csv` emit, and which are empty without GPS?
- Does `--prefix` overwrite or append on a second run?

**Sources:** hcxpcapngtool help, docs, sample captures.
**Output:** hash file plus inventory artefacts.

---

## Scenario 2 — inspect and filter hashes

| # | Command | How to use |
|---|---------|------------|
| 7 | `hcxhashtool -i cap.22000 --info=stdout` | Composition report |
| 8 | `hcxhashtool -i cap.22000 -o pmkid.22000 --type=1` | PMKID only |
| 9 | `hcxhashtool -i cap.22000 -o eapol.22000 --type=2` | EAPOL only |
| 10 | `hcxhashtool -i cap.22000 -o corp.22000 --essid="CorpNet"` | ESSID filter |
| 11 | `hcxhashtool -i cap.22000 -o recent.22000 --essid-list=targets.txt` | List filter |
| 12 | `hcxhashtool -i cap.22000 -o ap.22000 --mac-ap=aabbccddeeff` | AP MAC filter |
| 13 | `hcxhashtool -i cap.22000 -o vendor.22000 --vendor=Netgear` | Vendor filter |
| 14 | `hcxhashtool -i cap.22000 -o auth.22000 --authorized` | Authorized pairs only |
| 15 | `hcxhashtool -i cap.22000 -o dedup.22000 --hcx-max=1` | Drop repeated ESSIDs |

**Research questions**
- Which EAPOL status produces the highest crack rate after packet loss?
- Does `--authorized` exclude the only usable rogue-challenge pairs?

**Sources:** hcxhashtool help, hashcat 22000 format notes.
**Output:** a minimal, targeted hash set.

---

## Scenario 3 — vendor PSK candidates

| # | Command | How to use |
|---|---------|------------|
| 16 | `hcxpsktool -c cap.22000 -e "CorpNet" -o psk.txt` | ESSID/MAC candidates |
| 17 | `hcxpsktool -c cap.22000 --netgear -o netgear.txt` | NETGEAR defaults |
| 18 | `hcxpsktool -c cap.22000 --spectrum -o spectrum.txt` | Spectrum defaults (large) |
| 19 | `hcxpsktool -c cap.22000 --weakpass --simple -o weak.txt` | Weak patterns |
| 20 | `hcxpsktool -c cap.22000 --maconly -o mac.txt` | MAC-derived only |
| 21 | `hcxpsktool -c cap.22000 --usdate --eudate -o dates.txt` | Date patterns |

**Research questions**
- Which vendor sets still match current ISP firmware defaults?
- How large is each set, and what is the crack-time tradeoff?

**Sources:** hcxpsktool help, ISP default-password research, hashcat benchmarks.
**Output:** a ranked candidate list.

---

## Scenario 4 — ESSID-derived wordlists

| # | Command | How to use |
|---|---------|------------|
| 22 | `hcxeiutool -i essids.txt -d digits.txt -x xdigits.txt -c chars.txt -s sc.txt` | Split ESSIDs by charset |
| 23 | `cat essids.txt digits.txt xdigits.txt chars.txt sc.txt \| sort -u > wl.txt` | Merge seed lists |
| 24 | `hashcat --stdout -r rules/best64.rule chars.txt >> wl.txt` | Rule expansion |
| 25 | `hcxwltool -i wl.txt --lower --length=10 -o wl10.txt` | Normalise length/case |
| 26 | `hcxwltool -i wl.txt --digit -o digits-only.txt` | Digits only |
| 27 | `sort -u wl10.txt -o wl10.txt` | Deduplicate |

**Research questions**
- Which rule set gives the best return on ESSID-derived seeds?
- Does `-s` (0x0a replacement) improve rule compatibility in practice?

**Sources:** hcxeiutool example, hashcat rule docs.
**Output:** a compact, targeted wordlist.

---

## Scenario 5 — verify and report

| # | Command | How to use |
|---|---------|------------|
| 28 | `hcxpmktool -l "$(head -1 cap.22000)" -e "CorpNet" -p 'candidate'` | Verify one PSK |
| 29 | `hcxpmktool -l "$(head -1 cap.22000)" -e "CorpNet" -p - < list.txt` | Verify a small list |
| 30 | `hcxhashtool -i cap.22000 --info-vendor=stdout` | Vendor breakdown |
| 31 | `hcxhashtool -i cap.22000 -E essids.txt` | Extract ESSIDs for reporting |
| 32 | `hcxhashtool -d` | Refresh the OUI database |

**Research questions**
- What do hcxpmktool exit codes 0/1/2 mean operationally?
- How often does the IEEE OUI list need refreshing?

**Sources:** hcxpmktool help, IEEE OUI registry.
**Output:** a confirmed key or a documented non-match.

## Flag coverage checklist

| Binary | Major flags | Scenario |
|--------|-------------|----------|
| hcxpcapngtool | `-o -E -R -I -U -D --all --eapoltimeout --nonce-error-corrections --ignore-ie --max-essids --eapmd5 --eapleap --tacacs-plus --nmea --csv --log --raw-* --lts --pmkid* --hccap* --john --prefix --add-timestamp` | 1 |
| hcxhashtool | `-i -o -E -L -d --*-group --type --essid* --mac* --oui* --vendor* --authorized --challenge --rc* --apless --hcx-* --info* --psk --pmk --hccap* --john --vendorlist` | 2, 5 |
| hcxpsktool | `-c -i -j -z -e -b -o --maconly --noessidcombination --<vendor> --weakpass --[eu/us]date --wpskeys --egn --simple` | 3 |
| hcxeiutool | `-i -d -x -c -s` | 4 |
| hcxwltool | `-i -o --straight --digit --xdigit --lower --upper --capital --length` | 4 |
| hcxpmktool | `-l -e -p` | 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — convert | 6 |
| 2 — filter | 9 |
| 3 — vendor PSKs | 6 |
| 4 — wordlists | 6 |
| 5 — verify/report | 5 |
