# wireless-tools — legacy Wireless Extensions CLI suite

- **Version:** 30~pre9-18+b1
- **Binaries:** `/sbin/iwconfig`, `/sbin/iwlist`, `/sbin/iwpriv`, `/sbin/iwspy`, `/sbin/iwgetid`, `/sbin/iwevent`
- **Help snapshots:** [`help-iwconfig.txt`](help-iwconfig.txt), [`help-iwlist.txt`](help-iwlist.txt), [`help-iwpriv.txt`](help-iwpriv.txt), [`help-iwspy.txt`](help-iwspy.txt), [`help-iwgetid.txt`](help-iwgetid.txt), [`help-iwevent.txt`](help-iwevent.txt)
- **Upstream:** https://hewlettpackard.github.io/wireless-tools/
- **Licence:** GPL-2.0

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

`wireless-tools` is the original Linux Wi-Fi configuration suite, built on the
Wireless Extensions (WEXT) kernel interface. It predates nl80211 and cfg80211.
`iwconfig` reads and sets interface parameters, `iwlist` enumerates channels,
frequencies, encryption, and scan results, `iwpriv` reaches driver-private
ioctls, `iwspy` collects per-address link quality, `iwgetid` prints one field
for scripts, and `iwevent` streams wireless events.

For assessments it is still useful because:

- Many embedded, older, and vendor kernels expose WEXT but not nl80211.
- `iwlist` output is a stable, greppable format that older tooling parses.
- `iwconfig` shows the WEXT view of a link, which can differ from `iw`.
- `iwpriv` is the only path to some vendor ioctls.
- `iwevent` is a lightweight way to watch association and scan events.

On a modern Debian host with a nl80211 driver, most of these tools work in
read-only mode and report `no wireless extensions` for non-Wi-Fi interfaces.
`iw` is the supported replacement for configuration; `iwconfig` is deprecated
and cannot express HE/EHT, PMF, or most modern features.

## WEXT versus nl80211

| Task | wireless-tools (WEXT) | `iw` (nl80211) |
|------|----------------------|----------------|
| Show link | `iwconfig wlan0` | `iw dev wlan0 link` |
| Scan | `iwlist wlan0 scanning` | `iw dev wlan0 scan` |
| Set channel | `iwconfig wlan0 channel 6` | `iw dev wlan0 set channel 6` |
| Set ESSID | `iwconfig wlan0 essid lab` | `iw dev wlan0 connect lab` |
| Set mode | `iwconfig wlan0 mode monitor` | `iw dev wlan0 set type monitor` |
| TX power | `iwconfig wlan0 txpower 15` | `iw dev wlan0 set txpower fixed 1500` |
| List interfaces | `iwconfig` | `iw dev` |
| Events | `iwevent` | `iw event` |

WEXT has no representation for SAE, OWE, 802.11w/PMF, 802.11ax/be, or
multi-link operation. If a feature only exists in nl80211, `iwconfig` cannot
configure it and `iwlist` may not report it.

## `iwconfig`

```text
iwconfig [interface]
iwconfig interface essid {NNN|any|on|off}
iwconfig interface mode {managed|ad-hoc|master|...}
iwconfig interface freq N.NNN[k|M|G]
iwconfig interface channel N
iwconfig interface bit {N[k|M|G]|auto|fixed}
iwconfig interface rate {N[k|M|G]|auto|fixed}
iwconfig interface enc {NNNN-NNNN|off}
iwconfig interface key {NNNN-NNNN|off}
iwconfig interface power {period N|timeout N|saving N|off}
iwconfig interface nickname NNN
iwconfig interface nwid {NN|on|off}
iwconfig interface ap {N|off|auto}
iwconfig interface txpower {NmW|NdBm|off|auto}
iwconfig interface sens N
iwconfig interface retry {limit N|lifetime N}
iwconfig interface rts {N|auto|fixed|off}
iwconfig interface frag {N|auto|fixed|off}
iwconfig interface modulation {11g|11a|CCK|OFDMg|...}
iwconfig interface commit
```

With no arguments it lists every interface and whether it has wireless
extensions. With only an interface it prints the current WEXT state.

| Parameter | Meaning |
|-----------|---------|
| `essid` | Network name; `any` clears the preference, `on`/`off` toggles |
| `mode` | `managed` (station), `ad-hoc`, `master` (AP), `monitor`, `repeater` |
| `freq` / `channel` | Frequency in Hz/kHz/MHz/GHz or channel number |
| `bit` / `rate` | Bit rate, `auto`, or `fixed` |
| `enc` / `key` | WEP key or `off`; open/restricted encoding |
| `power` | Power management: period, timeout, saving, off |
| `ap` | Lock to a specific access-point MAC |
| `txpower` | Transmit power in mW or dBm |
| `sens` | Sensitivity threshold |
| `retry` | Retry limit or lifetime |
| `rts` / `frag` | RTS and fragmentation thresholds |
| `modulation` | Legacy modulation selector |
| `commit` | Apply all pending changes at once |

`iwconfig` cannot set WPA keys. Use `wpa_supplicant` for WPA/WPA2/WPA3.

## `iwlist`

```text
iwlist [interface] scanning [essid NNN] [last]
iwlist [interface] frequency
iwlist [interface] channel
iwlist [interface] bitrate
iwlist [interface] rate
iwlist [interface] encryption
iwlist [interface] keys
iwlist [interface] power
iwlist [interface] txpower
iwlist [interface] retry
iwlist [interface] ap
iwlist [interface] accesspoints
iwlist [interface] peers
iwlist [interface] event
iwlist [interface] auth
iwlist [interface] wpakeys
iwlist [interface] genie
iwlist [interface] modulation
```

| Query | Returns |
|-------|---------|
| `scanning` | BSS list with ESSID, BSSID, channel, quality, encryption |
| `frequency` | Supported channels and frequencies |
| `channel` | Current channel |
| `bitrate` / `rate` | Supported and current rates |
| `encryption` / `keys` / `wpakeys` | Encryption capability and key state |
| `power` / `txpower` / `retry` | Power and retry parameters |
| `ap` / `accesspoints` / `peers` | Peer/AP table where the driver exposes one |
| `event` | Supported event capability |
| `auth` | Authentication parameters |
| `genie` | Raw information elements |
| `modulation` | Modulation capabilities |

`scanning` triggers an active scan and takes several seconds. `last` reuses
the previous results where the driver supports it.

## `iwpriv`

```text
iwpriv interface [private-command [private-arguments]]
```

With no private command it lists the driver's private ioctls. With a command
it sets or gets a driver-specific value. Names and semantics are entirely
driver-defined — there is no portable vocabulary. Output is not stable across
drivers or kernel versions.

## `iwspy`

```text
iwspy interface [+] [MAC address] [IP address]
```

Collects per-address link quality for up to eight addresses. `+` adds an
address, `-` removes one, and no argument lists the current set. Many modern
drivers do not implement it and answer
`Interface doesn't support wireless statistic collection`.

## `iwgetid`

```text
iwgetid [OPTIONS] [ifname]
```

| Option | Prints |
|--------|--------|
| `-a`, `--ap` | Access-point address |
| `-c`, `--channel` | Current channel |
| `-f`, `--freq` | Current frequency |
| `-m`, `--mode` | Current mode |
| `-p`, `--protocol` | Protocol name |
| `-r`, `--raw` | Raw value for scripts |
| `-s`, `--scheme` | PCMCIA scheme identifier |
| `-h`, `--help` | Help |

With no interface it uses the first wireless interface. `-r` and `-s` are the
script-friendly forms; the other options include a human-readable label.

## `iwevent`

```text
iwevent [-h|--help] [-v|--version]
```

Streams wireless events (association, disassociation, scan completion, mode
changes) to stdout until interrupted. It needs no arguments and blocks.

## Recipes

**1. List wireless interfaces.**

```bash
iwconfig
```

**2. Show the WEXT view of a link.**

```bash
iwconfig wlan0
```

**3. Print just the ESSID for a script.**

```bash
iwgetid -r wlan0
```

**4. Print the current channel and frequency.**

```bash
iwgetid -c wlan0
iwgetid -f wlan0
```

**5. Scan and list visible BSSes.**

```bash
sudo iwlist wlan0 scanning
```

**6. Filter scan output to ESSID and signal.**

```bash
sudo iwlist wlan0 scanning | grep -E 'ESSID|Quality|Signal'
```

**7. List supported channels.**

```bash
iwlist wlan0 frequency
```

**8. Check encryption capability.**

```bash
iwlist wlan0 encryption
```

**9. Inspect driver-private ioctls.**

```bash
iwpriv wlan0
```

**10. Watch wireless events.**

```bash
sudo iwevent
```

**11. Set a legacy ad-hoc link.**

```bash
sudo iwconfig wlan0 mode ad-hoc essid lab-adhoc channel 6
```

**12. Put an interface into monitor mode (legacy path).**

```bash
sudo iwconfig wlan0 mode monitor
```

Prefer `iw dev wlan0 set type monitor` where nl80211 is available.

## Exit status

| Tool | Code | Meaning |
|------|------|---------|
| `iwconfig` | 0 | Query succeeded, even for a non-wireless interface |
| `iwconfig` | non-zero | Invalid parameter or interface |
| `iwlist` | 0 | Query succeeded |
| `iwlist` | 1 | No interface or missing query |
| `iwpriv` | 0 | Query or set succeeded |
| `iwspy` | 0 | Query succeeded, including "not supported" |
| `iwgetid` | 0 | Value printed |
| `iwgetid` | 255 | Interface has no wireless extensions |
| `iwevent` | 124 under `timeout` | Normal: it blocks waiting for events |

The tools often report "not supported" on stdout with exit 0. Check the text,
not only `$?`.

## Quirks and limits

- **WEXT is deprecated.** Modern features are nl80211-only; use `iw` for
  configuration where possible.
- **"No wireless extensions" is not an error.** `iwconfig` prints it and exits
  0 for Ethernet, loopback, and tunnel interfaces.
- **`iwlist` with no arguments exits 1.** It prints usage rather than listing
  interfaces.
- **`iwlist` calls the scan query `scanning`.** `scan` may be accepted as an
  abbreviation but is not documented.
- **`iwlist ... ap` can report no peer table.** Many drivers do not expose one;
  the exit code is still 0.
- **`iwspy` is widely unimplemented.** Expect "doesn't support wireless
  statistic collection".
- **`iwpriv` output is driver-specific.** Do not parse it across machines.
- **`iwgetid -r` drops labels.** The other modes prefix the interface name.
- **`iwgetid --version` is unrecognised.** Use `iwconfig --version` or the
  package version.
- **`iwevent` blocks forever.** Always wrap it in `timeout` or run it in the
  background.
- **Monitor mode needs driver support.** `iwconfig wlan0 mode monitor` fails on
  adapters that only implement nl80211 monitor mode.
- **WEP-era encryption syntax only.** `enc`/`key` cannot configure WPA.
- **The suite is root-only for writes.** Reads mostly work as a normal user;
  scans and interface changes need `CAP_NET_ADMIN`.

## Related tools

- [`iw`](../iw/) — nl80211 configuration and inspection; the modern replacement.
- [`wpa_supplicant`](../wpa_supplicant/) — WPA/WPA2/WPA3 supplicant.
- [`aircrack-ng`](../aircrack-ng/) — capture and injection suite.
- [`kismet`](../kismet/) — passive wireless detection.
- [`macchanger`](../macchanger/) — MAC address changes.
- [`wireshark`](../wireshark/) — 802.11 frame dissection.
