# wash — WPS survey tool

- **Version:** Wash v1.6.6 (Debian package `reaver` `1.6.6-2+b1`; `wash` is
  shipped by the `reaver` package, not a separate one)
- **Binary:** `/usr/bin/wash`
- **Help snapshot:** [`help.txt`](help.txt) — `wash -h` (26 lines)
- **Upstream:** https://github.com/t6x/reaver-wps-fork-t6x ·
  original by Tactical Network Solutions (Craig Heffner)
- **Docs:** `man wash`, upstream README

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Wash is the discovery half of the WPS toolchain. It puts the adapter in
monitor mode (or reads capture files), listens for WPS information elements in
beacons and probe responses, and prints a table of access points with their
WPS state, version, and lock status.

It is the correct first step before `reaver` or `wifite`: an AP that does not
advertise WPS cannot be attacked with WPS PIN or Pixie-Dust.

```text
wash -i <interface> [options]
wash -f <capture> [options]
```

## Required arguments

One of:

| Option | Meaning |
|--------|---------|
| `-i, --interface=<iface>` | Interface to capture packets on |
| `-f, --file [FILE1 FILE2 ...]` | Read packets from capture files |

## Optional arguments

| Option | Default | Meaning |
|--------|---------|---------|
| `-c, --channel=<num>` | auto | Channel to listen on |
| `-n, --probes=<num>` | 15 | Maximum probes per AP in scan mode |
| `-O, --output-file=<file>` | — | Write packets of interest to a pcap |
| `-F, --ignore-fcs` | off | Ignore frame checksum errors |
| `-2, --2ghz` | off | Use 2.4 GHz channels |
| `-5, --5ghz` | off | Use 5 GHz channels |
| `-s, --scan` | off | Use scan mode |
| `-u, --survey` | **default** | Use survey mode |
| `-a, --all` | off | Show all APs, even those without WPS |
| `-j, --json` | off | Print extended WPS info as JSON |
| `-U, --utf8` | off | Show UTF8 ESSID without sanitising (dangerous) |
| `-p, --progress` | off | Show percentage of crack progress |
| `-h, --help` | — | Show help |

## Survey vs scan mode

| Mode | Flag | Behaviour |
|------|------|-----------|
| Survey | `-u` (default) | Passive; waits for beacons and probe responses |
| Scan | `-s` | Active; sends up to `-n` probes per AP |

Survey is the safe default. Scan mode transmits, which is faster on quiet
networks but is visible to the AP and to any wireless IDS.

## Recipes

**1. Basic survey.**

```bash
sudo wash -i wlan0mon
```

**2. Survey including non-WPS APs.**

```bash
sudo wash -i wlan0mon -a
```

**3. JSON output for parsing.**

```bash
sudo wash -i wlan0mon -j
```

**4. One channel only.**

```bash
sudo wash -i wlan0mon -c 6
```

**5. 5 GHz survey.**

```bash
sudo wash -i wlan0mon -5
```

**6. 2.4 GHz survey.**

```bash
sudo wash -i wlan0mon -2
```

**7. Active scan mode.**

```bash
sudo wash -i wlan0mon -s -n 10
```

**8. Read an existing capture.**

```bash
wash -f capture.pcap
```

**9. Merge several captures.**

```bash
wash -f capture1.pcap capture2.pcap capture3.pcap
```

**10. Save interesting packets.**

```bash
sudo wash -i wlan0mon -O /tmp/wps-scan.pcap
```

**11. Show crack progress.**

```bash
sudo wash -i wlan0mon -p
```

**12. Ignore checksum errors.**

```bash
sudo wash -i wlan0mon -F
```

**13. Show UTF-8 ESSIDs.**

```bash
sudo wash -i wlan0mon -U
```

## Gotchas

- **`wash` is part of the `reaver` package.** Do not install a separate
  `wash` package; `dpkg -S /usr/bin/wash` reports `reaver`.
- **Monitor mode is required for `-i`.** Use `airmon-ng start` first, or a
  driver that supports monitor mode on the managed interface.
- **Survey mode is passive; scan mode transmits.** `-s` sends probes and is
  visible. Prefer `-u` unless you need speed.
- **`-n` only applies to scan mode.** It caps probes per AP.
- **`-a` shows non-WPS APs too.** That is useful for a full survey but makes
  the table much larger.
- **`-j` changes the output format.** It prints extended WPS info as JSON;
  combine it with a parser rather than reading it by eye.
- **`-U` does not sanitise.** The help calls it dangerous: an ESSID with
  terminal escape sequences can affect your terminal. Use it only on trusted
  captures.
- **`-p` is a progress display, not a crack.** Wash does not attack WPS; it
  reports the percentage shown by a running crack.
- **`-F` hides checksum errors.** That can make a bad capture look usable.
- **`-f` accepts multiple files.** That is the easiest way to survey a set of
  captures offline.
- **Channel hopping is automatic by default.** Pin `-c` when you need to stay
  on a target, or when the AP is on a 5 GHz channel.
- **`-2` and `-5` are band selectors, not channel selectors.** They restrict
  the hop set; `-c` pins a single channel.
- **WPS lock state is reported, not enforced.** Wash shows that an AP is
  locked; it does not wait or retry.
- **No root, no live capture.** Reading `-f` captures can be done unprivileged.

## Related docs

- [`reaver`](../reaver/README.md) — WPS PIN and Pixie-Dust attacks
- [`pixiewps`](../pixiewps/README.md) — offline Pixie-Dust computation
- [`wifite`](../wifite/README.md) — automated front end
- [`aircrack-ng`](../aircrack-ng/README.md) — monitor mode and capture
- [`bettercap`](../bettercap/README.md) — alternative Wi-Fi framework
- [`hcxdumptool`](../hcxdumptool/README.md) — PMKID and handshake capture
- [`iw`](../iw/README.md) — radio inspection
- [`macchanger`](../macchanger/README.md) — MAC changes
