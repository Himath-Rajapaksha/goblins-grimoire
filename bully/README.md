# bully — WPS PIN brute force

- **Version:** bully v1.4 (kimocoder fork)
- **Binary:** `/usr/bin/bully`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/kimocoder/bully

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

bully attacks the Wi-Fi Protected Setup PIN exchange. It can brute-force the
PIN (with optional checksum-digit brute force and sequential or randomised
ordering), run Pixie Dust against vulnerable chipsets, and report WPS
configuration exposed by an access point. It needs a monitor-mode interface
and root, and it transmits management and EAP frames to the AP.

## Use cases

| Situation | Why bully |
|-----------|-----------|
| WPS enabled on an in-scope AP | Recover the WPS PIN and derive the PSK |
| Pixie Dust–vulnerable chipset | Offline PIN recovery in seconds |
| WPS posture audit | Confirm whether WPS is reachable at all |
| Lockout testing | Measure AP lockout behaviour with `-D` |

## Requirements & install layout

- **Root** and a monitor-mode, injection-capable adapter.
- Debian package 1.4; binary at `/usr/bin/bully`.
- Session files default to `~/.bully/`.
- The AP must have WPS enabled; many modern APs rate-limit or disable it.
- `pixiewps` is invoked through `-d` and must be installed.

## Basic syntax

```bash
# brute force by BSSID on a fixed channel
sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6

# target by ESSID
sudo bully wlan1 -e "CorpNet" -c 6

# Pixie Dust attempt
sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -d

# passive test mode: no packet injection
sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -T
```

## Flag reference

### Required
| Flag | Meaning |
|------|---------|
| `interface` | Monitor-mode wireless interface (root required) |
| `-b, --bssid <mac>` | Target AP MAC address |
| `-e, --essid <string>` | Target AP by ESSID (alternative to `-b`) |

### Optional
| Flag | Meaning |
|------|---------|
| `-c, --channel <n[,n...]>` | Channel or channel hop list |
| `-i, --index <n>` | Starting PIN index (7 or 8 digits, auto) |
| `-l, --lockwait <sec>` | Wait after an AP lockout (default 43) |
| `-o, --outfile <file>` | Write messages to a file instead of stdout |
| `-p, --pin <n>` | Starting PIN (7 or 8 digits, auto) |
| `-s, --source <mac>` | Source/hardware MAC address |
| `-u, --lua <file>` | Lua script file |
| `-v, --verbosity <1-4>` | Verbosity (default 3) |
| `-w, --workdir <path>` | PIN/session file directory (default `~/.bully/`) |
| `-5, --5ghz` | Hop the 5 GHz channel list |
| `-B, --bruteforce` | Brute-force the PIN checksum digit |
| `-F, --force` | Continue despite warnings |
| `-S, --sequential` | Sequential PINs (do not randomise) |
| `-T, --test` | Test mode: inject no packets |

### Advanced
| Flag | Meaning |
|------|---------|
| `-d, --pixiewps` | Attempt Pixie Dust |
| `-a, --acktime <n>` | Deprecated/ignored |
| `-r, --retries <n>` | Retransmit unacked packets (default 2) |
| `-m, --m13time <n>` | Deprecated/ignored |
| `-t, --timeout <n>` | Deprecated/ignored |
| `-1, --pin1delay M,N` | Delay M seconds every Nth NACK at M5 |
| `-2, --pin2delay M,N` | Delay M seconds every Nth NACK at M7 |
| `-A, --noacks` | Disable ACK checking |
| `-C, --nocheck` | Skip CRC/FCS validation (faster) |
| `-D, --detectlock` | Detect lockouts the AP does not report |
| `-E, --eapfail` | Terminate every exchange on EAP failure |
| `-L, --lockignore` | Ignore reported WPS locks |
| `-M, --m57nack` | Treat M5/M7 timeouts as WSC_NACK |
| `-N, --nofcs` | Packets have no FCS field |
| `-P, --probe` | Probe request for a non-beaconing AP |
| `-Q, --wpsinfo` | Probe for WPS information |
| `-R, --radiotap` | Assume radiotap headers |
| `-W, --windows7` | Masquerade as a Windows 7 registrar |
| `-Z, --suppress` | Suppress packet throttling |
| `-V, --version` | Print version |
| `-h, --help` | Print help |

## Recipes

```bash
# 1. Put the adapter in monitor mode
sudo ip link set wlan1 down
sudo iw dev wlan1 set type monitor
sudo ip link set wlan1 up

# 2. Read WPS information without injecting
sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -Q -T

# 3. Fastest path: Pixie Dust
sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -d

# 4. Classic PIN brute force with a work directory
sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -w /evidence/bully

# 5. Resume a session
sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -w /evidence/bully -S

# 6. Lockout-tolerant run
sudo bully wlan1 -b AA:BB:CC:DD:EE:FF -c 6 -l 120 -D
```

## Output & pipelines

- Prints the WPS PIN and, when the AP exposes it, the PSK.
- Session/PIN state in `~/.bully/` (or `-w`) allows resuming.
- Use the recovered PSK with `wpa_supplicant` or confirm it against a
  handshake via `hcxtools`.

## Gotchas & operational security

- WPS brute force is **loud** and usually locks the AP after a few attempts.
- `-T` is the safe way to confirm reachability; it injects nothing.
- Pixie Dust only works on specific chipsets; a failed `-d` is not a verdict
  on WPS generally.
- `-L` ignores lockouts and can permanently wedge weak APs.
- Some APs require `-W` (Windows 7 registrar) emulation to progress.
- Restore the interface to managed mode afterwards.

## Legal & safety

Only test WPS on APs you own or have **written authorization** to assess.
Brute-forcing a neighbour's AP is unlawful in most jurisdictions and can
deauthenticate legitimate clients.

## Related tools

- [`hcxdumptool/`](../hcxdumptool/README.md), [`hcxtools/`](../hcxtools/README.md)
- [`wifiphisher/`](../wifiphisher/README.md) — WPS-PBC mode
- [`kismet/`](../kismet/README.md) — find WPS-enabled APs
- [`eaphammer/`](../eaphammer/README.md)

## References

- https://github.com/kimocoder/bully
- https://github.com/wiire-a/pixiewps
