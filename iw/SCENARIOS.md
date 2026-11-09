# iw — command scenarios for deep research

Installed build: iw 6.9 (Debian package `iw` `6.9-1`). Every command and flag
below appears in [`help.txt`](help.txt), the 482-line `iw help` dump. `iw` is
read-only until a command changes state, so the scenarios separate inspection
from configuration.

> **Scope:** Changing interface type, channel, transmit power, or regulatory
> domain affects every client on the radio and may violate local spectrum
> rules. Work only on adapters you are authorized to reconfigure, and record
> the original state (`iw dev`, `iw phy`, `iw reg get`) before the first
> change.

## How to invoke

```text
@deep-research For an authorized wireless assessment on Linux, explain how iw
6.9 exposes nl80211 capabilities that aircrack-ng 1.7, wifite 2.7.0, and
bettercap 2.33.0 depend on. Which interface types and monitor flags does a
typical adapter support, how do `iw phy` capabilities predict whether
injection and monitor mode will work, and where does iw stop and a supplicant
begin? Cite the iw source and manual, the nl80211/cfg80211 kernel
documentation, IEEE 802.11, and the upstream wikis for the three tools;
separate confirmed fact from inference and state confidence.
```

Record the original state before any change:

```bash
mkdir -p ~/iw-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/iw-demo/evidence/run-start.txt
iw dev > ~/iw-demo/evidence/iw-dev-before.txt
iw phy > ~/iw-demo/evidence/iw-phy-before.txt
iw reg get > ~/iw-demo/evidence/iw-reg-before.txt
```

## Scenario 1 — inventory interfaces and capabilities

Capability discovery is the step that decides which later tools can work.

| # | Command | How to use |
|---|---------|------------|
| 1 | `iw --version` | Version banner (6.9) |
| 2 | `iw dev` | List wireless interfaces and their types |
| 3 | `iw phy` | List phys |
| 4 | `iw list` | All devices with full capabilities |
| 5 | `iw phy phy0 info` | One phy in detail |
| 6 | `iw dev wlan0 info` | One interface in detail |
| 7 | `iw phy phy0 info \| sed -n '/Supported interface modes/,/^$/p'` | Which types are supported |
| 8 | `iw commands` | Known commands with decimal and hex values |
| 9 | `iw features` | Supported top-level feature bits |
| 10 | `iw help dev scan` | Usage for one command |

**Research questions**
- Which interface types does this adapter support, and which are missing?
- What does `iw list` show that `iw phy phy0 info` does not?
- Why does `iw commands` expose numeric command IDs?
- Which capabilities are phy-wide and which are per-interface?
- What does the absence of `monitor` in supported modes predict about
  injection tests?

**Sources:** `iw help`; `iw list`; nl80211/cfg80211 documentation; IEEE
802.11.
**Output:** a capability record with a monitor/injection verdict.

## Scenario 2 — create, convert, and remove interfaces

Interface creation is how monitor, mesh, and IBSS testing begins. Every
change is reversible with `del` or `set type`.

| # | Command | How to use |
|---|---------|------------|
| 11 | `sudo iw phy phy0 interface add mon0 type monitor` | Create a monitor interface |
| 12 | `sudo ip link set mon0 up` | Bring it up |
| 13 | `iw dev mon0 info` | Verify type, channel, address |
| 14 | `sudo iw dev mon0 set monitor control otherbss` | Add monitor flags |
| 15 | `sudo iw dev mon0 set monitor fcsfail cook` | Different flags |
| 16 | `sudo ip link set wlan0 down && sudo iw dev wlan0 set type monitor && sudo ip link set wlan0 up` | Convert an existing interface |
| 17 | `sudo ip link set wlan0 down && sudo iw dev wlan0 set type managed && sudo ip link set wlan0 up` | Convert it back |
| 18 | `sudo iw dev mon0 del` | Remove the virtual interface |
| 19 | `sudo iw dev wlan0 set 4addr on` | Enable WDS/4-address mode |
| 20 | `sudo iw dev wlan0 set power_save off` | Disable power save for a test |

**Research questions**
- Which monitor flags change which frames reach userspace?
- Does the driver accept `set type` while the interface is up?
- What is the difference between deleting a virtual interface and converting
  it?
- What does `4addr on` change in the frame header?
- Why can power save break timing-sensitive captures?

**Sources:** `iw help`; nl80211 documentation; driver documentation for the
adapter; IEEE 802.11 monitor-mode behaviour.
**Output:** an interface runbook with before/after `iw dev` output.

## Scenario 3 — scanning and association

Scanning is the bridge between capability checks and real targets. `iw`'s scan
is the raw nl80211 view; `airodump-ng` and `bettercap` build on the same
results.

| # | Command | How to use |
|---|---------|------------|
| 21 | `sudo iw dev wlan0 scan` | Full active scan |
| 22 | `sudo iw dev wlan0 scan freq 2412 2437 2462` | Specific frequencies |
| 23 | `sudo iw dev wlan0 scan passive` | Passive scan, no probes |
| 24 | `sudo iw dev wlan0 scan ssid CorpSSID` | Probe one SSID |
| 25 | `sudo iw dev wlan0 scan -u` | Include unknown IEs |
| 26 | `sudo iw dev wlan0 scan randomise` | Randomised source address |
| 27 | `iw dev wlan0 scan dump` | Read the cached results |
| 28 | `sudo iw dev wlan0 scan trigger freq 5180` | Trigger on one channel |
| 29 | `sudo iw dev wlan0 scan abort` | Abort an ongoing scan |
| 30 | `sudo iw dev wlan0 scan sched_start interval 30000 freqs 2412 2437` | Scheduled scan |
| 31 | `sudo iw dev wlan0 scan sched_stop` | Stop the schedule |
| 32 | `sudo iw dev wlan0 connect -w OpenSSID` | Join an open network |
| 33 | `iw dev wlan0 link` | Confirm the connection |
| 34 | `sudo iw dev wlan0 disconnect` | Leave the network |

**Research questions**
- Which scan results come from beacons, probe responses, and probe requests?
- What does `passive` change in the frame stream and in discovery time?
- When is `randomise` useful, and what does it break?
- What does `-u` add to the output?
- Why is `iw connect` not a substitute for `wpa_supplicant`?

**Sources:** `iw help`; nl80211 scan documentation; IEEE 802.11 scan
procedure; `wpa_supplicant` documentation.
**Output:** a scan dataset with a per-target channel and security record.

## Scenario 4 — channel, power, and radio tuning

These settings change what the radio hears and how far it reaches. They are
also the ones most likely to be regulated.

| # | Command | How to use |
|---|---------|------------|
| 35 | `iw phy phy0 channels` | Available channels and flags |
| 36 | `sudo iw dev mon0 set channel 6` | Set a 20 MHz channel |
| 37 | `sudo iw dev mon0 set channel 36 80MHz` | Set an 80 MHz channel |
| 38 | `sudo iw dev mon0 set freq 5180 80 5210` | Set frequency with centre |
| 39 | `sudo iw dev wlan0 set txpower fixed 1500` | Fixed 15 dBm |
| 40 | `sudo iw dev wlan0 set txpower limit 2000` | Cap at 20 dBm |
| 41 | `sudo iw dev wlan0 set txpower auto` | Return to automatic |
| 42 | `iw dev wlan0 survey dump` | Channel survey counters |
| 43 | `sudo iw phy phy0 set antenna all` | Use all antennas |
| 44 | `sudo iw phy phy0 set retry short 7 long 4` | Retry limits |
| 45 | `sudo iw phy phy0 set rts 500` | RTS threshold |
| 46 | `sudo iw phy phy0 set frag off` | Disable fragmentation |
| 47 | `sudo iw dev wlan0 set bitrates legacy-2.4 1 2 5.5 11` | Legacy rate mask |
| 48 | `sudo iw dev wlan0 set mcast_rate 6` | Multicast rate |

**Research questions**
- What do the channel flags in `iw phy channels` mean (disabled, no IR, DFS,
  radar detection)?
- Why does 80 MHz need a centre frequency, and how is it chosen?
- What is the difference between `fixed` and `limit` txpower?
- What does `survey dump` count, and how can it detect interference?
- Which of these settings survive a driver reload?

**Sources:** `iw help`; nl80211 channel and power documentation; IEEE 802.11
HT/VHT; local regulatory rules.
**Output:** a radio configuration record with a regulatory check.

## Scenario 5 — DFS, CAC, and channel availability

DFS channels require a radar check before use. `iw` exposes the check
directly, including background and triggered variants.

| # | Command | How to use |
|---|---------|------------|
| 49 | `sudo iw dev wlan0 cac freq 5260` | Start a CAC on a DFS channel |
| 50 | `sudo iw dev wlan0 cac channel 52 80MHz` | CAC by channel and width |
| 51 | `sudo iw dev wlan0 cac background freq 5260` | Background CAC |
| 52 | `sudo iw dev wlan0 cac trigger freq 5260` | Trigger a CAC |
| 53 | `iw dev wlan0 info` | Confirm the operating channel |
| 54 | `dmesg \| grep -i -E 'dfs\|radar\|cac'` | Kernel radar/CAC messages |

**Research questions**
- Which channels are DFS in the current regulatory domain?
- What is the difference between `cac`, `cac background`, and `cac trigger`?
- What happens when radar is detected during a CAC?
- Why is transmitting before CAC completion a regulatory problem?
- How do you confirm the CAC result from userspace?

**Sources:** `iw help`; nl80211 DFS documentation; ETSI/ FCC radar rules;
kernel messages.
**Output:** a DFS channel plan with CAC evidence.

## Scenario 6 — mesh, IBSS, OCB, and station control

These are the multi-node and per-client modes. They are the least common and
the easiest to leave misconfigured.

| # | Command | How to use |
|---|---------|------------|
| 55 | `sudo iw dev wlan0 ibss join TestCell 2412 HT20` | Join or create an IBSS |
| 56 | `sudo iw dev wlan0 ibss join TestCell 2412 fixed-freq 02:00:00:00:00:01` | Fixed BSSID |
| 57 | `sudo iw dev wlan0 ibss leave` | Leave the IBSS |
| 58 | `sudo iw dev wlan0 mesh join meshnet freq 2412 HT20` | Join a mesh |
| 59 | `iw dev wlan0 mesh_param dump` | List mesh parameters |
| 60 | `sudo iw dev wlan0 set mesh_param mesh_ttl=31` | Set a mesh parameter |
| 61 | `iw dev wlan0 get mesh_param mesh_ttl` | Read it back |
| 62 | `iw dev wlan0 mpath dump` | Mesh paths |
| 63 | `iw dev wlan0 mpp dump` | Mesh proxy paths |
| 64 | `sudo iw dev wlan0 mpath new 02:00:00:00:00:02 next_hop 02:00:00:00:00:03` | Create a path |
| 65 | `sudo iw dev wlan0 ocb join 5890 10MHz` | Join OCB |
| 66 | `sudo iw dev wlan0 ocb leave` | Leave OCB |
| 67 | `iw dev wlan0 station dump -v` | Stations known to the interface |
| 68 | `iw dev wlan0 station get 02:00:00:00:00:02` | One station |
| 69 | `sudo iw dev wlan0 station set 02:00:00:00:00:02 airtime_weight 4` | Airtime weight |
| 70 | `sudo iw dev wlan0 station del 02:00:00:00:00:02 subtype 0xC reason-code 3` | Send a deauth |
| 71 | `sudo iw dev wlan0 mesh leave` | Leave the mesh |

**Research questions**
- What does a fixed BSSID change in IBSS cell formation?
- Which mesh parameters affect path selection?
- What is the difference between an mpath and an mpp?
- Why does `station del` need a subtype and reason code?
- What does OCB mode target that IBSS does not?

**Sources:** `iw help`; IEEE 802.11 IBSS, mesh, and OCB clauses; nl80211 mesh
documentation.
**Output:** a multi-node topology record with path and station tables.

## Scenario 7 — events, regulatory domain, and WoWLAN

Observability and legal constraints. These commands explain why a connection
failed and what the radio is allowed to do.

| # | Command | How to use |
|---|---------|------------|
| 72 | `sudo iw event -t` | Timestamped kernel events |
| 73 | `sudo iw event -T -f` | Absolute time and full frames |
| 74 | `sudo iw event -r` | Relative timestamps |
| 75 | `iw reg get` | Kernel regulatory domain |
| 76 | `iw phy phy0 reg get` | Phy-specific domain |
| 77 | `sudo iw reg set GB` | Set the domain |
| 78 | `sudo iw reg reload` | Reload the regulatory database |
| 79 | `iw phy phy0 wowlan show` | Wake-on-WLAN status |
| 80 | `sudo iw phy phy0 wowlan enable magic-packet disconnect` | Enable triggers |
| 81 | `sudo iw phy phy0 wowlan enable net-detect interval 5000 delay 30 freqs 2412 2422 matches ssid foo ssid bar` | Net-detect |
| 82 | `sudo iw phy phy0 wowlan disable` | Disable WoWLAN |
| 83 | `iw phy phy0 get txq` | TX queue parameters |
| 84 | `sudo iw phy phy0 set txq limit 1024 quantum 1514` | Set TX queue limits |

**Research questions**
- Which events does `iw event` surface that `dmesg` does not?
- Why can `iw reg get` and `phy reg get` disagree?
- What constrains the effective regulatory domain besides `reg set`?
- Which WoWLAN triggers does this adapter actually support?
- What does the TXQ quantum control?

**Sources:** `iw help`; nl80211 regulatory and WoWLAN documentation; kernel
`cfg80211` documentation; local spectrum rules.
**Output:** an event log, a regulatory record, and a WoWLAN capability check.

## Scenario 8 — namespaces, NAN, P2P, and cleanup

The remaining groups are specialist: network namespaces for isolation, NAN
and P2P for proximity services, and the cleanup that restores the radio.

| # | Command | How to use |
|---|---------|------------|
| 85 | `sudo iw phy phy0 set netns name testns` | Move the phy to a netns |
| 86 | `sudo ip netns exec testns iw dev` | Inspect it there |
| 87 | `sudo iw phy phy0 set name phy9` | Rename the phy |
| 88 | `iw dev` | List wdevs and their indices |
| 89 | `sudo iw wdev 0x1 p2p start` | Start P2P |
| 90 | `sudo iw wdev 0x1 p2p stop` | Stop P2P |
| 91 | `sudo iw wdev 0x1 nan start pref 5 bands 2GHz 5GHz` | Start NAN |
| 92 | `sudo iw wdev 0x1 nan config pref 3 bands 5GHz` | Reconfigure NAN |
| 93 | `sudo iw wdev 0x1 nan stop` | Stop NAN |
| 94 | `sudo iw phy phy0 coalesce show` | Coalescing status |
| 95 | `sudo iw phy phy0 coalesce disable` | Disable coalescing |
| 96 | `iw dev mon0 info` | Final interface check |
| 97 | `sudo iw dev mon0 del` | Remove test interfaces |
| 98 | `sudo iw reg get` | Confirm the domain is unchanged |
| 99 | `diff ~/iw-demo/evidence/iw-dev-before.txt <(iw dev)` | Prove the restore |

**Research questions**
- What disappears from the current namespace when a phy moves to a netns?
- What does the wdev index identify that the interface name does not?
- Which NAN parameters control discovery range and filtering?
- Why is `coalesce` a power-saving feature rather than a capture feature?
- How do you prove the radio is back to its original state?

**Sources:** `iw help`; `ip-netns(8)`; nl80211 NAN/P2P and coalesce
documentation; kernel `mac80211_hwsim` documentation.
**Output:** a namespace/proximity record and a diff proving restoration.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `--debug`, `--version` | 1 |
| `dev`, `phy`, `list`, `features`, `commands`, `help` | 1 |
| `dev … info`, `dev … del` | 1, 2 |
| `dev … interface add type …` | 2 |
| `dev … set type`, `set monitor`, `set 4addr` | 2 |
| `dev … set peer` | reference (WDS) |
| `dev … set power_save`, `get power_save` | 2 |
| `dev … scan` (`-u`, `freq`, `duration`, `ies`, `meshid`, `lowpri`, `flush`, `ap-force`, `duration-mandatory`, `randomise`, `ssid`, `passive`) | 3 |
| `dev … scan trigger`, `dump`, `abort`, `sched_start`, `sched_stop` | 3 |
| `dev … connect`, `disconnect`, `link` | 3 |
| `dev … auth`, `cqm rssi` | reference (association variants) |
| `dev … set channel`, `set freq`, `set freq_khz`, `switch` | 4 |
| `dev … set txpower`, `set bitrates`, `set mcast_rate` | 4 |
| `dev … set noack_map`, `set tidconf` | reference (per-TID tuning) |
| `dev … offchannel`, `roc start` | reference (timing-specific) |
| `dev … cac channel\|freq\|background\|trigger` | 5 |
| `dev … ibss join\|leave` | 6 |
| `dev … mesh join\|leave`, `mesh_param`, `mpath`, `mpp` | 6 |
| `dev … ocb join\|leave` | 6 |
| `dev … station dump\|get\|set\|del` | 6 |
| `dev … survey dump` | 4 |
| `dev … key get` | reference (needs a key) |
| `dev … mgmt dump` | reference (management-frame capture) |
| `dev … vendor send\|recv\|recvbin` | reference (driver-specific) |
| `dev … measurement ftm_request`, `ftm start_responder`, `ftm get_stats` | reference (needs an FTM peer) |
| `dev … ap start\|stop` | 1 (capability check only) |
| `phy … info`, `channels`, `reg get` | 1, 4, 7 |
| `phy … interface add` | 2 |
| `phy … set txq`, `get txq` | 7 |
| `phy … set antenna`, `txpower`, `retry`, `rts`, `frag`, `channel`, `freq` | 4 |
| `phy … set netns`, `name` | 8 |
| `phy … set distance`, `coverage`, `freq_khz`, `sar_specs` | reference (link/regulatory tuning) |
| `phy … coalesce show\|disable\|enable` | 8 |
| `phy … wowlan show\|disable\|enable` | 7 |
| `phy … hwsim wakequeues\|stopqueues\|setps\|getps` | not exercised (test radio only) |
| `wdev … nan add_func\|rm_func\|config\|start\|stop` | 8 |
| `wdev … p2p start\|stop` | 8 |
| `reg get\|set\|reload` | 7 |
| `event -t\|-T\|-r\|-f` | 7 |

Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Inventory interfaces and capabilities | 10 |
| 2 | Create, convert, and remove interfaces | 10 |
| 3 | Scanning and association | 14 |
| 4 | Channel, power, and radio tuning | 14 |
| 5 | DFS, CAC, and channel availability | 6 |
| 6 | Mesh, IBSS, OCB, station control | 17 |
| 7 | Events, regulatory domain, WoWLAN | 13 |
| 8 | Namespaces, NAN, P2P, cleanup | 15 |
| **Total** | | **99** |

## Legal and safety

`iw` changes the radio's type, channel, power, and regulatory domain. Monitor
mode, IBSS, mesh, and AP interfaces transmit; a channel or power setting that
exceeds the local domain is a regulatory violation, and `station del` sends a
real deauthentication frame. Work only on adapters and channels you are
authorized to use, record `iw dev`, `iw phy`, and `iw reg get` before the
first change, and restore the original type, channel, power, and regulatory
domain before leaving. Confirm the restore with a diff against the baseline,
not from memory.
