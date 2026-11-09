# wifite — automated wireless auditing front end

- **Version:** wifite2 2.7.0 (Debian package `wifite` `2.7.0-3`)
- **Binary:** `/usr/bin/wifite`
- **Help snapshot:** [`help.txt`](help.txt) — `wifite -h` (51 lines, with
  `-h -v` giving the same settings plus verbose mode)
- **Upstream:** https://github.com/kimocoder/wifite2 ·
  original by derv82
- **Docs:** `wifite -h`, `wifite -h -v`, upstream README and wiki
- **Default wordlist:** `/usr/share/dict/wordlist-probable.txt`
- **Backends used:** `aircrack-ng`, `aireplay-ng`, `airodump-ng`, `airmon-ng`,
  `reaver`, `bully`, `hcxdumptool`, `hcxtools`, `hashcat`, `john`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Wifite is a wrapper. It puts the adapter into monitor mode, surveys access
points and clients, ranks targets, and then runs the right backend attack for
each encryption type:

| Target type | Attack chain | Backend |
|-------------|--------------|---------|
| WEP | IV generation, ARP replay, chopchop, fragmentation | `aireplay-ng` + `aircrack-ng` |
| WPA/WPA2-PSK | Handshake capture, optional deauth, dictionary crack | `airodump-ng`, `aireplay-ng`, `aircrack-ng`/`hashcat` |
| WPA/WPA2-PSK | PMKID capture (clientless) | `hcxdumptool` + `hashcat` |
| WPS | PIN attack, Pixie-Dust | `reaver` (default) or `bully` |

It is interactive by default: it shows a target list, asks which targets to
attack, then reports per-target progress. `-p SECONDS` ("pillage") and
`-first N` make it non-interactive. Everything it runs is printed with `-v`,
which is the main reason to prefer it over hand-run commands during a
repeatable test.

```text
wifite [options]
```

## Settings

| Option | Meaning |
|--------|---------|
| `-h, --help` | Show help |
| `-v, --verbose` | Show more options and print every command and its output |
| `-i INTERFACE` | Wireless interface (default: auto-detect) |
| `-c CHANNEL` | Channels to scan, e.g. `1,3-6` (default: all 2.4 GHz) |
| `-inf, --infinite` | Infinite attack mode; scanning time controlled by `-p` |
| `-mac, --random-mac` | Randomise the card MAC |
| `-p SECONDS` | Pillage: attack all targets after this scan time |
| `--kill` | Kill conflicting processes (NetworkManager, wpa_supplicant) |
| `-pow, --power MIN_POWER` | Attack targets at or above this signal strength |
| `--skip-crack` | Capture only; do not crack |
| `-first N, --first N` | Attack the first N targets |
| `-ic, --ignore-cracked` | Hide previously cracked targets |
| `--clients-only` | Only show targets with associated clients |
| `--nodeauths` | Passive mode: never deauthenticate |
| `--daemon` | Return the device to managed mode after quitting |

## WEP

| Option | Meaning |
|--------|---------|
| `--wep` | Show only WEP networks |
| `--require-fakeauth` | Fail the attack if fake authentication fails |
| `--keep-ivs` | Keep `.IVS` files and reuse them when cracking |

## WPA

| Option | Meaning |
|--------|---------|
| `--wpa` | Show only WPA networks (includes WPS) |
| `--new-hs` | Capture new handshakes; ignore existing ones in `hs/` |
| `--dict FILE` | Wordlist (default `/usr/share/dict/wordlist-probable.txt`) |

## WPS

| Option | Meaning |
|--------|---------|
| `--wps` | Show only WPS-enabled networks |
| `--wps-only` | Use only WPS PIN and Pixie-Dust attacks |
| `--bully` | Use `bully` instead of `reaver` |
| `--reaver` | Use `reaver` (default) |
| `--ignore-locks` | Keep attacking after the AP locks WPS |

## PMKID

| Option | Meaning |
|--------|---------|
| `--pmkid` | Use only PMKID capture; skip WPS and WPA handshake attacks |
| `--no-pmkid` | Disable PMKID capture |
| `--pmkid-timeout SECONDS` | Wait time per target (default 300) |

## Commands

| Option | Meaning |
|--------|---------|
| `--cracked` | Print previously cracked access points |
| `--check [FILE]` | Check a `.cap` file, or all `hs/*.cap`, for handshakes |
| `--crack` | Show the commands to crack a captured handshake |

## Recipes

**1. Interactive survey and attack.**

```bash
sudo wifite
```

**2. Verbose mode — print every backend command.**

```bash
sudo wifite -v
```

**3. Pick the interface and restrict channels.**

```bash
sudo wifite -i wlan0 -c 1,6,11
```

**4. Attack all targets after a 60-second scan.**

```bash
sudo wifite -p 60
```

**5. Attack only WPS networks with Pixie-Dust/PIN.**

```bash
sudo wifite --wps --wps-only
```

**6. PMKID-only capture with a 120-second wait.**

```bash
sudo wifite --pmkid --pmkid-timeout 120
```

**7. Passive mode — no deauthentication.**

```bash
sudo wifite --nodeauths
```

**8. Capture only, crack later.**

```bash
sudo wifite --skip-crack
```

**9. Use a real wordlist.**

```bash
sudo wifite --dict /usr/share/wordlists/rockyou.txt
```

**10. Use bully instead of reaver.**

```bash
sudo wifite --wps --bully
```

**11. Attack the first three targets only.**

```bash
sudo wifite -first 3
```

**12. Hide targets you already cracked.**

```bash
sudo wifite -ic --cracked
```

**13. Check existing captures for handshakes.**

```bash
wifite --check
wifite --check hs/handshake.cap
```

**14. Print cracking commands for a capture.**

```bash
wifite --crack
```

**15. Kill interfering processes, then restore managed mode on exit.**

```bash
sudo wifite --kill --daemon
```

## Gotchas

- **`wifite` is a wrapper, not a cracker.** Every result comes from
  `aircrack-ng`, `hashcat`, `reaver`, `bully`, or `hcxdumptool`; when
  something fails, read the backend output with `-v`.
- **The default wordlist is tiny.** `/usr/share/dict/wordlist-probable.txt`
  is a probable-password list, not a comprehensive one. Most real cracks need
  an explicit `--dict`.
- **PMKID is on by default and costs time.** Each target can take up to
  `--pmkid-timeout` seconds (default 300). Use `--no-pmkid` when you want
  handshake attacks to run first.
- **`--pmkid` disables the other WPA paths.** It is an exclusive mode, not an
  addition.
- **`--wps-only` still needs a WPS-enabled AP.** It does not discover WPS on
  non-WPS networks.
- **`--ignore-locks` can burn the AP's WPS lockout.** Reaver and bully will
  keep trying after the AP locks; that is noisy and can make the AP
  unavailable to its owner for a while.
- **`--nodeauths` gives up handshake speed.** Passive capture only works when
  a client naturally reconnects; it is the right choice when deauth is out of
  scope.
- **`-p` implies attacking everything it finds.** "Pillage" is not a survey
  mode. Confirm the target list before using it.
- **`--kill` kills NetworkManager and wpa_supplicant.** Your own connectivity
  drops. `--daemon` only restores managed mode; it does not restart services.
- **`--random-mac` can break some drivers.** Some adapters fail to associate
  or inject after a MAC change; verify before relying on it.
- **`--daemon` restores managed mode, not the original MAC.** Combine it with
  a manual MAC restore if you changed it.
- **`-c` is 2.4 GHz by default.** 5 GHz channels need to be named explicitly
  and the adapter must support them.
- **Handshakes are stored under `hs/`.** Wifite reuses them unless `--new-hs`
  is set, which can make a "new" run look successful when it only replayed an
  old capture.
- **`--check` is a capture validator, not a cracker.** It reports whether a
  handshake is present; `--crack` prints the commands.
- **Root is required.** Monitor mode, injection, and raw sockets all need
  privileges.
- **WPA3/SAE is not supported.** Wifite targets WEP, WPA/WPA2-PSK, WPS, and
  PMKID. It will not crack SAE.
- **Results depend on the adapter.** Injection quality and monitor-mode
  stability vary by chipset; test with `aireplay-ng -9` first.

## Related docs

- [`aircrack-ng`](../aircrack-ng/README.md) — the capture, injection, and
  cracking suite wifite drives
- [`reaver`](../reaver/README.md) — default WPS backend
- [`pixiewps`](../pixiewps/README.md) — offline Pixie-Dust computation
- [`wash`](../wash/README.md) — WPS survey
- [`hcxdumptool`](../hcxdumptool/README.md) — PMKID and handshake capture
- [`hcxtools`](../hcxtools/README.md) — capture conversion
- [`hashcat`](../hashcat/README.md) — GPU cracking backend
- [`bettercap`](../bettercap/README.md) — alternative Wi-Fi framework
- [`kismet`](../kismet/README.md) — passive detection
- [`iw`](../iw/README.md) — radio inspection
- [`macchanger`](../macchanger/README.md) — MAC changes
