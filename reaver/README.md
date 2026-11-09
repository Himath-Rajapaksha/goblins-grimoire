# reaver — WPS PIN and Pixie-Dust attack tool

- **Version:** Reaver v1.6.6 (Debian package `reaver` `1.6.6-2+b1`)
- **Binary:** `/usr/bin/reaver`
- **Help snapshot:** [`help.txt`](help.txt) — `reaver -h` (44 lines)
- **Upstream:** https://github.com/t6x/reaver-wps-fork-t6x ·
  original by Tactical Network Solutions (Craig Heffner)
- **Docs:** `man reaver`, upstream README and wiki
- **Ships with:** `wash` (the WPS survey tool) — same Debian package

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Reaver attacks Wi-Fi Protected Setup. WPS was designed to let a user join a
network with an 8-digit PIN, and that PIN is verified in two halves — so a
brute force is 10⁴ + 10³ attempts, not 10⁸. Reaver implements that online PIN
attack and also drives the offline **Pixie-Dust** attack against APs with weak
Diffie-Hellman nonce generation.

Two attack modes matter:

| Mode | Flag | What happens |
|------|------|--------------|
| Online PIN | default | Reaver tries PINs against the live AP; slow, and may trigger lockout |
| Pixie-Dust | `-K` / `-Z` | Reaver collects the WPS handshake and calls `pixiewps` offline |

Reaver also performs the WPS registration protocol itself, so it needs a
monitor-mode interface, a target BSSID, and — unless `-A` is used — an
association with the AP.

```text
reaver -i <monitor-iface> -b <bssid> [options]
```

## Required arguments

| Option | Meaning |
|--------|---------|
| `-i, --interface=<wlan>` | Monitor-mode interface |
| `-b, --bssid=<mac>` | Target AP BSSID |

## Optional arguments

| Option | Meaning |
|--------|---------|
| `-m, --mac=<mac>` | MAC of the host system |
| `-e, --essid=<ssid>` | Target ESSID |
| `-c, --channel=<channel>` | Set the channel (implies `-f`) |
| `-s, --session=<file>` | Restore a previous session |
| `-C, --exec=<command>` | Run a command after PIN recovery |
| `-f, --fixed` | Disable channel hopping |
| `-5, --5ghz` | Use 5 GHz channels |
| `-v, --verbose` | Non-critical warnings (`-vv`, `-vvv` for more) |
| `-q, --quiet` | Critical messages only |
| `-h, --help` | Help |

## Advanced options

| Option | Default | Meaning |
|--------|---------|---------|
| `-p, --pin=<pin>` | — | Use a specified PIN (arbitrary string or 4/8 digits) |
| `-d, --delay=<seconds>` | `1` | Delay between PIN attempts |
| `-l, --lock-delay=<seconds>` | `60` | Wait after the AP locks WPS |
| `-g, --max-attempts=<num>` | — | Quit after this many PIN attempts |
| `-x, --fail-wait=<seconds>` | `0` | Sleep after 10 unexpected failures |
| `-r, --recurring-delay=<x:y>` | — | Sleep `y` seconds every `x` PIN attempts |
| `-t, --timeout=<seconds>` | `10` | Receive timeout |
| `-T, --m57-timeout=<seconds>` | `0.40` | M5/M7 timeout |
| `-A, --no-associate` | off | Do not associate; another program must |
| `-N, --no-nacks` | off | Do not NACK out-of-order packets |
| `-S, --dh-small` | off | Small DH keys to speed up the crack |
| `-L, --ignore-locks` | off | Ignore the AP's locked state |
| `-E, --eap-terminate` | off | End each WPS session with EAP FAIL |
| `-J, --timeout-is-nack` | off | Treat timeout as NACK (DIR-300/320) |
| `-F, --ignore-fcs` | off | Ignore frame checksum errors |
| `-w, --win7` | off | Mimic a Windows 7 registrar |
| `-K, --pixie-dust` | off | Run the Pixie-Dust attack |
| `-Z` | off | Run the Pixie-Dust attack (alias) |
| `-O, --output-file=<file>` | — | Write packets of interest to a pcap |

## Recipes

**1. Confirm the target with `wash`.**

```bash
sudo wash -i wlan0mon
```

**2. Basic online PIN attack.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -vv
```

**3. Pin the channel and disable hopping.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -c 6 -vv
```

**4. Pixie-Dust attack.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -K -vv
```

**5. Pixie-Dust with small DH keys.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -K -S -vv
```

**6. Resume a session.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -s /tmp/target.session -vv
```

**7. Try a known PIN.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -p 12345670 -vv
```

**8. Slow down and respect lockouts.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -d 5 -l 300 -vv
```

**9. Bound the attempt count.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -g 50 -vv
```

**10. Sleep periodically to avoid lockout.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -r 10:60 -vv
```

**11. Use 5 GHz channels.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -5 -vv
```

**12. Do not associate — pair with an external supplicant.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -A -vv
```

**13. Capture WPS packets for offline analysis.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -O /tmp/wps.pcap -vv
```

**14. Run a command after recovery.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -C 'echo recovered >> /tmp/wps.log' -vv
```

## Gotchas

- **WPS must be enabled.** Reaver cannot attack an AP that does not advertise
  WPS. Confirm with `wash` first.
- **Online PIN attacks are slow.** At the default 1-second delay, a full PIN
  search is hours to days even before lockouts.
- **Many APs lock WPS after a handful of failures.** `-L` ignores the lock
  report, but the AP may still refuse; that is a denial-of-service against
  the owner, not a speed-up.
- **`-K` and `-Z` are the same thing.** Both run Pixie-Dust; the help lists
  them separately.
- **Pixie-Dust only works on vulnerable chipsets.** It exploits weak nonce
  generation. On a patched AP it fails quickly and there is nothing to retry.
- **`-S` must match `pixiewps`.** If you use small DH keys in reaver, pass
  `-S` to `pixiewps` too, or the computed keys will not match.
- **`-c` implies `-f`.** Setting a channel disables hopping; make sure it is
  the AP's actual channel.
- **`-A` shifts association to another program.** Without association, most
  APs will not run the WPS registration exchange.
- **`-p` accepts 4- or 8-digit PINs.** A 4-digit value is treated as the
  first half; the checksum and second half are still needed.
- **`-w` is a compatibility hack.** It makes reaver look like a Windows 7
  registrar; some APs require it, most do not.
- **`-J` is for specific D-Link models.** It treats timeouts as NACKs because
  those APs never send a proper NACK.
- **`-F` hides checksum errors.** That can make a bad capture look usable;
  use it only when you know the frames are valid.
- **Session files are state, not evidence.** They let you resume; they do not
  prove what happened. Keep the pcap and logs.
- **Root and monitor mode are required.** Reaver needs raw 802.11 injection.
- **The recovered WPS PIN is not the Wi-Fi passphrase.** Reaver reports the
  PIN; use it to retrieve the PSK, or use the PIN to enrol a client.
- **This is an offensive tool with legal limits.** WPS attacks are detectable
  and disruptive; use them only with authorization.

## Related docs

- [`wash`](../wash/README.md) — WPS survey, same package
- [`pixiewps`](../pixiewps/README.md) — offline Pixie-Dust computation
- [`wifite`](../wifite/README.md) — automated front end that drives reaver
- [`aircrack-ng`](../aircrack-ng/README.md) — capture and monitor-mode suite
- [`bettercap`](../bettercap/README.md) — alternative Wi-Fi attacks
- [`hcxdumptool`](../hcxdumptool/README.md) — PMKID and handshake capture
- [`iw`](../iw/README.md) — radio inspection
- [`macchanger`](../macchanger/README.md) — MAC changes
