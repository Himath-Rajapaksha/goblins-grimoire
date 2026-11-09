# wifite — command scenarios for deep research

Installed build: wifite2 2.7.0 (Debian package `wifite` `2.7.0-3`). Every
option below appears in [`help.txt`](help.txt). Wifite is a wrapper, so each
scenario also names the backend it drives; behaviour questions are only
answerable by reading the backend output, which is why `-v` matters.

> **Scope:** Wifite deauthenticates clients, captures handshakes, attacks WPS,
> and can lock an AP's WPS function. Run it only on networks you are
> authorized to test. Prefer `--nodeauths` and `--skip-crack` when the
> engagement is passive, and record every backend command with `-v`.

## How to invoke

```text
@deep-research For an authorized wireless assessment, compare wifite 2.7.0
with the aircrack-ng 1.7 suite and bettercap 2.33.0 for WPA2-PSK and WPS
testing. Which tool gives the most reproducible evidence trail, how do they
differ in handshake and PMKID capture reliability, and what does wifite hide
that the underlying tools expose? Cite the wifite2 repository, the
aircrack-ng wiki, the bettercap Wi-Fi module docs, and the reaver and bully
documentation; return a wrapper-vs-native comparison with confidence notes.
```

Create an evidence workspace and record the radio state:

```bash
mkdir -p ~/wifite-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/wifite-demo/evidence/run-start.txt
iw dev > ~/wifite-demo/evidence/iw-dev-before.txt
iw list > ~/wifite-demo/evidence/iw-list.txt
ls -l /usr/share/dict/wordlist-probable.txt > ~/wifite-demo/evidence/wordlist.txt
```

## Scenario 1 — inventory, dependencies, and safe defaults

Wifite's behaviour is the sum of its backends. Confirm which are installed and
which mode the defaults select before running an attack.

| # | Command | How to use |
|---|---------|------------|
| 1 | `wifite -h` | Show all settings |
| 2 | `wifite -h -v` | Verbose settings variant |
| 3 | `dpkg-query -W -f='${Version}\n' wifite` | Installed version |
| 4 | `for b in aircrack-ng aireplay-ng airodump-ng airmon-ng reaver bully hcxdumptool hcxtools hashcat john; do command -v "$b" \|\| echo "missing: $b"; done` | Backend inventory |
| 5 | `ls -l /usr/share/dict/wordlist-probable.txt` | Default wordlist exists |
| 6 | `head -5 /usr/share/dict/wordlist-probable.txt` | What the default list looks like |
| 7 | `man wifite \| col -b \| sed -n '1,60p'` | Packaged manual |
| 8 | `python3 -c 'import wifite; print(wifite.__file__)'` | Locate the installed module |

**Research questions**
- Which backends does wifite prefer, and which are optional?
- What does the default wordlist contain, and how large is it?
- Which options are only visible with `-v`?
- How does wifite decide the interface when `-i` is absent?
- What happens when a backend is missing — a warning or a silent skip?

**Sources:** `wifite -h`; `wifite -h -v`; Debian package contents; upstream
wifite2 README.
**Output:** a backend inventory and a defaults record.

## Scenario 2 — survey, target selection, and passive mode

The survey is passive and safe. Target selection is where scope is enforced.

| # | Command | How to use |
|---|---------|------------|
| 9 | `sudo wifite` | Interactive survey and menu |
| 10 | `sudo wifite -i wlan0 -c 1,6,11` | Interface and channel restriction |
| 11 | `sudo wifite --clients-only` | Only APs with associated clients |
| 12 | `sudo wifite --nodeauths` | Passive mode; never deauth |
| 13 | `sudo wifite -pow -70` | Minimum signal strength |
| 14 | `sudo wifite --wpa` | WPA/WPA2 (and WPS) only |
| 15 | `sudo wifite --wep` | WEP only |
| 16 | `sudo wifite --wps` | WPS-enabled only |
| 17 | `sudo wifite -ic --cracked` | Show previously cracked targets |

**Research questions**
- What does the target list show that a raw `airodump-ng` does not?
- How does `--clients-only` change the ranking?
- What is the difference between `--wpa` and `--wps` filters?
- Where does wifite store cracked-target state?
- What happens to the adapter when you quit?

**Sources:** `wifite -h`; `airodump-ng --help`; upstream wifite2
documentation.
**Output:** a scoped target list with a signal and client record.

## Scenario 3 — WPA handshake capture and cracking

This is the main path: capture a 4-way handshake, then crack it offline. The
quality of the capture decides whether the crack is even possible.

| # | Command | How to use |
|---|---------|------------|
| 18 | `sudo wifite --wpa --dict /usr/share/wordlists/rockyou.txt` | Interactive WPA path |
| 19 | `sudo wifite --wpa --new-hs` | Ignore existing `hs/` captures |
| 20 | `sudo wifite --wpa --skip-crack` | Capture only |
| 21 | `sudo wifite -v --wpa -first 1` | One target, full command trace |
| 22 | `ls -l hs/` | Inspect captured handshakes |
| 23 | `wifite --check` | Validate all `hs/*.cap` files |
| 24 | `wifite --check hs/handshake.cap` | Validate one capture |
| 25 | `wifite --crack` | Print the cracking commands |
| 26 | `aircrack-ng -w /usr/share/wordlists/rockyou.txt hs/handshake.cap` | Verify with the backend directly |

**Research questions**
- When does wifite choose to deauth, and what changes with `--nodeauths`?
- What is the difference between a captured handshake and a valid one?
- Why does `--new-hs` matter for repeatability?
- How does `--check` decide a handshake is usable?
- What does `--crack` print that `--check` does not?

**Sources:** `wifite -h`; `airodump-ng --help`; `aireplay-ng --help`;
`aircrack-ng --help`; upstream handshake-validation notes.
**Output:** a validated capture plus the exact crack command and runtime.

## Scenario 4 — PMKID capture

PMKID is clientless and fast when it works, but not every AP returns one.
Wifite makes it the default path, which changes timing.

| # | Command | How to use |
|---|---------|------------|
| 27 | `sudo wifite --pmkid` | PMKID only |
| 28 | `sudo wifite --pmkid --pmkid-timeout 60` | Shorter wait per target |
| 29 | `sudo wifite --no-pmkid` | Disable PMKID, handshake only |
| 30 | `sudo wifite -v --pmkid -first 1` | Trace one PMKID attempt |
| 31 | `ls -l hs/` | Inspect PMKID output |
| 32 | `hcxdumptool --help 2>&1 \| head -20` | Backend usage |
| 33 | `hashcat -m 22000 --help 2>&1 \| head -5` | Hash mode used downstream |

**Research questions**
- Which APs return a PMKID, and what do they have in common?
- Why does `--pmkid` exclude WPS and handshake attacks?
- What does the timeout control, and what happens on expiry?
- How does a PMKID capture differ from a handshake in size and crackability?
- Which hashcat mode consumes the output?

**Sources:** `wifite -h`; `hcxdumptool` documentation; hashcat example-hashes;
the original PMKID research by Jens Steube.
**Output:** a PMKID capture record with a per-AP success table.

## Scenario 5 — WPS PIN and Pixie-Dust

WPS is the highest-yield path when it is enabled, and the most likely to leave
the AP locked. Treat lockout as a scope issue.

| # | Command | How to use |
|---|---------|------------|
| 34 | `sudo wifite --wps --wps-only` | WPS-only attacks |
| 35 | `sudo wifite --wps --reaver` | Force reaver backend |
| 36 | `sudo wifite --wps --bully` | Force bully backend |
| 37 | `sudo wifite --wps --ignore-locks` | Continue after lockout |
| 38 | `sudo wifite -v --wps -first 1` | Trace one WPS attempt |
| 39 | `wash -i wlan0mon` | Independent WPS survey |
| 40 | `reaver -i wlan0mon -b AA:BB:CC:DD:EE:FF -vv` | Run reaver directly |
| 41 | `pixiewps --help 2>&1 \| head -20` | Offline Pixie-Dust usage |

**Research questions**
- How does wifite decide between PIN and Pixie-Dust?
- Which APs are vulnerable to Pixie-Dust, and how would you know in advance?
- What does an AP lockout look like, and how long does it last?
- What does `--ignore-locks` risk for the AP's owner?
- How do reaver and bully differ in speed and reliability?

**Sources:** `wifite -h`; `reaver --help`; `wash --help`; `pixiewps --help`;
the Pixie-Dust research by Dominique Bongard.
**Output:** a WPS result table with lockout status and backend used.

## Scenario 6 — automation, evidence, and cleanup

Wifite can run unattended, which makes it easy to leave the radio or the
network in a changed state. Build the stop procedure first.

| # | Command | How to use |
|---|---------|------------|
| 42 | `sudo wifite -p 60` | Scan 60 s, then attack all targets |
| 43 | `sudo wifite -inf -p 30` | Infinite mode with 30 s scans |
| 44 | `sudo wifite --kill --daemon` | Kill conflicts, restore managed mode |
| 45 | `sudo wifite -mac --daemon` | Random MAC, restore managed mode |
| 46 | `sudo wifite -v -first 2 2>&1 \| tee ~/wifite-demo/evidence/run.log` | Full evidence log |
| 47 | `ip -brief link` | Verify the interface state |
| 48 | `sudo systemctl restart NetworkManager` | Restore connectivity |
| 49 | `iw dev` | Confirm managed mode and MAC |
| 50 | `ls -l hs/` | Collect captures for the report |

**Research questions**
- What does `-p` do that `-inf` does not?
- Why is `-v` essential for a reproducible run?
- What does `--daemon` restore, and what does it leave changed?
- How do you prove the adapter was returned to managed mode?
- Which artifacts belong in the report, and which should be deleted?

**Sources:** `wifite -h`; `iw dev`; `systemctl`; upstream wifite2
documentation.
**Output:** a run log, a capture inventory, and a signed-off cleanup record.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| `-h`, `-v` | 1 |
| `-i`, `-c` | 2 |
| `-inf`, `-p` | 6 |
| `-mac` | 6 |
| `--kill`, `--daemon` | 6 |
| `-pow`, `--power` | 2 |
| `--skip-crack` | 3 |
| `-first`, `--first` | 3, 4, 5, 6 |
| `-ic`, `--ignore-cracked` | 2 |
| `--clients-only` | 2 |
| `--nodeauths` | 2 |
| `--wep`, `--require-fakeauth`, `--keep-ivs` | 2 (WEP filter) |
| `--wpa`, `--new-hs`, `--dict` | 3 |
| `--wps`, `--wps-only`, `--bully`, `--reaver`, `--ignore-locks` | 5 |
| `--pmkid`, `--no-pmkid`, `--pmkid-timeout` | 4 |
| `--cracked` | 2 |
| `--check [FILE]` | 3 |
| `--crack` | 3 |

Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Inventory, dependencies, defaults | 8 |
| 2 | Survey, target selection, passive mode | 9 |
| 3 | WPA handshake capture and cracking | 9 |
| 4 | PMKID capture | 7 |
| 5 | WPS PIN and Pixie-Dust | 8 |
| 6 | Automation, evidence, cleanup | 9 |
| **Total** | | **50** |

## Legal and safety

Wifite deauthenticates clients and attacks WPS by default. That disrupts
other users and can lock an AP's WPS function for its owner. Use it only with
written authorization, prefer `--nodeauths` and `--skip-crack` for passive
work, and never use `--ignore-locks` without explicit permission. Before
leaving, stop the run, confirm the adapter is back in managed mode with
`iw dev`, restore the original MAC, restart NetworkManager/wpa_supplicant, and
delete captures that contain third-party traffic. Handshakes, PMKIDs, and
recovered keys are sensitive evidence: encrypt them and keep only what the
report requires.
