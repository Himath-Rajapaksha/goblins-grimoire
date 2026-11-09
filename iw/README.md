# iw — nl80211 wireless configuration and inspection

- **Version:** iw 6.9 (Debian package `iw` `6.9-1`)
- **Binary:** `/usr/sbin/iw`
- **Help snapshot:** [`help.txt`](help.txt) — `iw help` (482 lines: every
  command and subcommand the build knows)
- **Upstream:** https://git.kernel.org/pub/scm/linux/kernel/git/jberg/iw.git
- **Docs:** `man iw`, `iw help`, `iw help <command>`
- **Kernel interface:** nl80211 (cfg80211)

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

`iw` is the reference userspace tool for the Linux wireless stack. It talks
to the kernel over **nl80211** and covers what the older `iwconfig`/`iwlist`
pair did — and far more: interface creation, monitor mode, scan control,
mesh, P2P, NAN, FTM, WoWLAN, channel availability checks, and per-station
tuning.

It is a **configuration and inspection** tool, not a capture or attack tool.
It creates the interfaces that `tcpdump`, `wireshark`, `airodump-ng`, and
`bettercap` then use, and it reads the capabilities that decide whether those
tools will work at all.

```text
iw [options] command
```

| Global option | Meaning |
|---------------|---------|
| `--debug` | Enable netlink debugging |
| `--version` | Print the version (6.9) |

## Command groups

| Group | Purpose |
|-------|---------|
| `dev <devname> …` | Per-interface operations (the largest group) |
| `phy <phyname> …` | Per-hardware operations |
| `wdev <idx> …` | Per-wdev operations (NAN, P2P) |
| `reg …` | Regulatory domain |
| `event` | Kernel event monitor |
| `list` | List all wireless devices and capabilities |
| `phy` | List phys |
| `dev` | List wireless interfaces |
| `features` | Supported top-level feature bits |
| `commands` | All known commands with decimal and hex values |
| `help [command]` | Usage for all or one command |

Two conveniences matter when typing by hand: `dev`/`phy` can be omitted when
the target is unambiguous (`iw wlan0 info`, `iw phy0 info`), and a netdev
command can be given a `wdev` index instead of a device name. **Do not rely on
either in scripts** — the help explicitly warns not to screenscrape output.

## `dev` subcommands

### Interface lifecycle and identity

| Command | Meaning |
|---------|---------|
| `dev` | List all wireless interfaces |
| `dev <devname> info` | Interface details: type, channel, address, txpower |
| `dev <devname> del` | Remove a virtual interface |
| `dev <devname> interface add <name> type <type> [mesh_id <id>] [4addr on\|off] [flags <flag>*] [addr <mac>]` | Create a virtual interface |
| `dev <devname> set type <type>` | Change interface type |
| `dev <devname> set monitor <flag>*` | Set monitor flags |
| `dev <devname> set 4addr <on\|off>` | WDS/4-address mode |
| `dev <devname> set peer <MAC>` | WDS peer |
| `dev <devname> set power_save <on\|off>` | Power-save state |
| `dev <devname> get power_save` | Read power-save state |

Valid interface types: `managed`, `ibss`, `monitor`, `mesh`, `wds`.

Monitor flags: `none`, `fcsfail`, `control`, `otherbss`, `cook`, `active`,
`mumimo-groupid <id>`, `mumimo-follow-mac <mac>`.

### Association

| Command | Meaning |
|---------|---------|
| `dev <devname> connect [-w] <SSID> [<freq>] [<bssid>] [auth open\|shared] [key …] [mfp:req/opt/no]` | Join a network |
| `dev <devname> auth <SSID> <bssid> <open\|shared> <freq> [key …]` | Authenticate only |
| `dev <devname> disconnect` | Leave the network |
| `dev <devname> link` | Current connection details |
| `dev <devname> cqm rssi <threshold\|off> [<hysteresis>]` | Connection-quality monitor |

`connect -w` waits for the connection to finish or fail; without it, the
command returns after the request is accepted. `mfp` controls management-frame
protection: `req` (require), `opt` (optional), `no` (disable).

### Scanning

| Command | Meaning |
|---------|---------|
| `dev <devname> scan [-u] [freq <freq>*] [duration <dur>] [ies <hex>] [meshid <id>] [lowpri,flush,ap-force,duration-mandatory] [randomise[=<addr>/<mask>]] [ssid <ssid>*\|passive]` | Scan now |
| `dev <devname> scan trigger …` | Trigger a scan with the same options |
| `dev <devname> scan dump [-u]` | Dump cached results |
| `dev <devname> scan abort` | Abort an ongoing scan |
| `dev <devname> scan sched_start …` | Start a scheduled scan |
| `dev <devname> scan sched_stop` | Stop a scheduled scan |

`-u` prints unknown/unparsed information elements. `passive` scans without
probing. `randomise` uses a randomised source address, optionally constrained
by an address/mask pair.

### Channel and radio control

| Command | Meaning |
|---------|---------|
| `dev <devname> set channel <ch> [width]` | Set channel |
| `dev <devname> set freq <freq> [width] [<center1> [<center2>]]` | Set frequency and HT/VHT config |
| `dev <devname> set freq_khz <freq> [width]` | Set frequency in kHz |
| `dev <devname> switch channel <ch> …` | Channel switch announcement (CSA) |
| `dev <devname> switch freq <freq> …` | CSA by frequency |
| `dev <devname> set txpower <auto\|fixed\|limit> [<mBm>]` | Transmit power |
| `dev <devname> set bitrates …` | Legacy/HT/VHT/HE rate masks |
| `dev <devname> set mcast_rate <Mbps>` | Multicast bitrate |
| `dev <devname> set noack_map <map>` | NoAck map per TID |
| `dev <devname> set tidconf …` | Per-peer TID configuration |
| `dev <devname> offchannel <freq> <duration>` | Leave the operating channel briefly |
| `dev <devname> roc start <freq> <ms>` | Remain-on-channel |

Widths accepted by `set freq`: `NOHT`, `HT20`, `HT40+`, `HT40-`, `5MHz`,
`10MHz`, `80MHz`, `160MHz`, `320MHz`, or the
`<control> [5|10|20|40|80|80+80|160] [<center1> [<center2>]]` form.

### Channel availability (DFS)

| Command | Meaning |
|---------|---------|
| `dev <devname> cac channel <ch> [width]` | Start a CAC |
| `dev <devname> cac freq <freq> [width]` | CAC by frequency |
| `dev <devname> cac background channel\|freq …` | Background CAC |
| `dev <devname> cac trigger channel\|freq …` | Trigger a CAC |

CAC (Channel Availability Check) is the radar-detection wait required before
transmitting on DFS channels. `background` and `trigger` let the check run
without blocking the current channel.

### Mesh, IBSS, OCB

| Command | Meaning |
|---------|---------|
| `dev <devname> ibss join <SSID> <freq> [width] [fixed-freq] [<bssid>] [beacon-interval <TU>] [basic-rates …] [mcast-rate …] [key …]` | Join/create an IBSS |
| `dev <devname> ibss leave` | Leave the IBSS |
| `dev <devname> mesh join <id> [[freq <freq> <width>] [basic-rates …]] [mcast-rate …] [beacon-interval …] [dtim-period …] [vendor_sync on\|off] [<param>=<value>]*` | Join a mesh |
| `dev <devname> mesh leave` | Leave the mesh |
| `dev <devname> mesh_param dump` | List mesh parameters |
| `dev <devname> set mesh_param <param>=<value> …` | Set mesh parameters |
| `dev <devname> get mesh_param [<param>]` | Read mesh parameters |
| `dev <devname> mpath dump` | Mesh paths |
| `dev <devname> mpath set <dst> next_hop <hop>` | Change a path |
| `dev <devname> mpath new <dst> next_hop <hop>` | Create a path |
| `dev <devname> mpath del <MAC>` | Delete a path |
| `dev <devname> mpath get <MAC>` | Read a path |
| `dev <devname> mpath probe <dst> frame <frame>` | Inject via a chosen next hop |
| `dev <devname> mpp dump` | Mesh proxy paths |
| `dev <devname> mpp get <MAC>` | Proxy path to a node |
| `dev <devname> ocb join <freq> <5MHz\|10MHz>` | Join an OCB network |
| `dev <devname> ocb leave` | Leave OCB |

### Stations, survey, keys, vendor, management

| Command | Meaning |
|---------|---------|
| `dev <devname> station dump [-v]` | All known stations |
| `dev <devname> station get <MAC>` | One station |
| `dev <devname> station set <MAC> txpwr <auto\|limit> [<dBm>]` | Per-station txpower |
| `dev <devname> station set <MAC> airtime_weight <weight>` | Airtime scheduling weight |
| `dev <devname> station set <MAC> mesh_power_mode <active\|light\|deep>` | Mesh power mode |
| `dev <devname> station set <MAC> vlan <ifindex>` | AP VLAN |
| `dev <devname> station set <MAC> plink_action <open\|block>` | Mesh peer-link action |
| `dev <devname> station del <MAC> [subtype <s>] [reason-code <c>]` | Remove a station entry |
| `dev <devname> survey dump [--radio]` | Channel survey counters |
| `dev <devname> key get <index> <MAC>` | Retrieve a key and sequence |
| `dev <devname> mgmt dump frame <type> <pattern> [frame …] [count <n>]` | Capture selected management frames |
| `dev <devname> vendor send <oui> <subcmd> <data>` | Vendor command |
| `dev <devname> vendor recv <oui> <subcmd> <data>` | Vendor command with reply |
| `dev <devname> vendor recvbin <oui> <subcmd> <data>` | Vendor reply, binary |
| `dev <devname> measurement ftm_request <config-file> [timeout=<s>] [randomise[=…]]` | FTM ranging request |
| `dev <devname> ftm start_responder [lci=…] [civic=…]` | Start an FTM responder (needs AP mode) |
| `dev <devname> ftm get_stats` | FTM responder statistics |
| `dev <devname> ap start …` | Start AP mode |
| `dev <devname> ap stop` | Stop AP mode |

`station del` sends a disassociation (`subtype 0xA`) or deauthentication
(`0xC`) frame — the help marks it "use with caution", because it is a real
deauth, not just a table cleanup.

## `phy` subcommands

| Command | Meaning |
|---------|---------|
| `phy` | List phys |
| `phy <phyname> info` | Capabilities of one phy |
| `phy <phyname> channels` | Available channels with flags |
| `phy <phyname> interface add …` | Create an interface on this phy |
| `phy <phyname> reg get` | Regulatory domain for this phy |
| `phy <phyname> set txq limit <n> \| memory_limit <bytes> \| quantum <bytes>` | TX queue parameters |
| `phy <phyname> get txq` | Read TX queue parameters |
| `phy <phyname> set antenna <bitmap> \| all \| <tx> <rx>` | Antenna selection |
| `phy <phyname> set txpower <auto\|fixed\|limit> [<mBm>]` | Transmit power |
| `phy <phyname> set distance <auto\|m>` | Dynack or coverage class for a link distance |
| `phy <phyname> set coverage <class>` | Coverage class (1 per 3 µs) |
| `phy <phyname> set netns { <pid> \| name <nsname> }` | Move the phy to a netns |
| `phy <phyname> set retry [short <n>] [long <n>]` | Retry limits |
| `phy <phyname> set rts <threshold\|off>` | RTS threshold |
| `phy <phyname> set frag <threshold\|off>` | Fragmentation threshold |
| `phy <phyname> set channel <ch> [width]` | Channel |
| `phy <phyname> set freq <freq> [width] [<center1> [<center2>]]` | Frequency |
| `phy <phyname> set freq_khz <freq> [width]` | Frequency in kHz |
| `phy <phyname> set name <new name>` | Rename the phy |
| `phy <phyname> set sar_specs <type> <index:sar power>*` | SAR limits |
| `phy <phyname> coalesce show\|disable\|enable <config-file>` | Packet coalescing |
| `phy <phyname> wowlan show\|disable\|enable …` | Wake-on-WLAN |
| `phy <phyname> hwsim wakequeues\|stopqueues\|setps <v>\|getps` | mac80211_hwsim test controls |

### WoWLAN

`wowlan enable` accepts trigger flags `any`, `disconnect`, `magic-packet`,
`gtk-rekey-failure`, `eap-identity-request`, `4way-handshake`,
`rfkill-release`, and the compound `net-detect …` and `tcp <config-file>`
forms. `net-detect` takes scan plans, delay, frequencies, match sets, and
active/passive probing; `tcp` takes a file of `source=`, `dest=`, `data=`,
`data.interval=`, and optional `wake=`, `data.seq=`, `data.tok=` lines.

### Coalesce

The coalesce config file is a list of rules:

```text
delay=<msec>
condition=<1|0>
patterns=<[offset+]<pattern>,…>
```

Patterns are hex bytes with `-` as a wildcard, and an optional `offset+`
prefix, e.g. `18+43:34:00:12` matches those bytes 18 bytes into the frame.

## `wdev` subcommands

| Command | Meaning |
|---------|---------|
| `wdev <idx> nan add_func type <publish\|subscribe\|followup> [active] [solicited] [unsolicited] [bcast] [close_range] name <name> [info <info>] …` | Add a NAN function |
| `wdev <idx> nan rm_func cookie <cookie>` | Remove a NAN function |
| `wdev <idx> nan config [pref <pref>] [bands [2GHz] [5GHz]]` | Configure NAN |
| `wdev <idx> nan start pref <pref> [bands …]` | Start NAN |
| `wdev <idx> nan stop` | Stop NAN |
| `wdev <idx> p2p start` | Start P2P |
| `wdev <idx> p2p stop` | Stop P2P |

## `reg` subcommands

| Command | Meaning |
|---------|---------|
| `reg get` | Kernel regulatory domain |
| `reg set <ISO 3166-1 alpha2>` | Set the regulatory domain |
| `reg reload` | Reload the kernel regulatory database |

The regulatory domain is the legal constraint on channels and power. A phy
inherits it, but `phy <phyname> reg get` can differ from `reg get` when a
driver applies its own limits.

## `event`

```text
event [-t|-T|-r] [-f]
```

Monitors kernel wireless events. `-t` prints a timestamp, `-T` an absolute
human-readable timestamp, `-r` a relative timestamp, and `-f` prints the full
frame for auth/association events. This is the tool to run in one terminal
while reproducing an association or deauth problem in another.

## Recipes

**1. List interfaces and their current state.**

```bash
iw dev
```

**2. Show one interface in detail.**

```bash
iw dev wlan0 info
```

**3. Show hardware capabilities — the file to read before any wireless test.**

```bash
iw phy
iw phy phy0 info
```

**4. Check whether the driver supports monitor mode.**

```bash
iw phy phy0 info | sed -n '/Supported interface modes/,/^$/p'
```

**5. List available channels with their flags.**

```bash
iw phy phy0 channels
```

**6. Create a monitor interface.**

```bash
sudo iw phy phy0 interface add mon0 type monitor
sudo ip link set mon0 up
```

**7. Create a monitor interface on a specific channel.**

```bash
sudo iw phy phy0 interface add mon0 type monitor
sudo ip link set mon0 up
sudo iw dev mon0 set channel 6
```

**8. Set monitor flags to include control frames and other BSSes.**

```bash
sudo iw dev mon0 set monitor control otherbss
```

**9. Change an interface to managed mode.**

```bash
sudo ip link set wlan0 down
sudo iw dev wlan0 set type managed
sudo ip link set wlan0 up
```

**10. Scan and read the cached results.**

```bash
sudo iw dev wlan0 scan
iw dev wlan0 scan dump
```

**11. Passive scan on one frequency.**

```bash
sudo iw dev wlan0 scan freq 2412 passive
```

**12. Scan with a randomised source address.**

```bash
sudo iw dev wlan0 scan randomise
```

**13. Connect to an open network and wait.**

```bash
sudo iw dev wlan0 connect -w OpenSSID
```

**14. Connect with a raw WEP key (WPA2-PSK passphrases belong to
wpa_supplicant, not `iw`).**

```bash
sudo iw dev wlan0 connect -w CorpSSID auth open key 0:abcde d:1:6162636465
```

**15. Show the current link.**

```bash
iw dev wlan0 link
```

**16. Dump stations seen by an AP interface.**

```bash
iw dev wlan0 station dump -v
```

**17. Show channel survey counters.**

```bash
iw dev wlan0 survey dump
```

**18. Monitor kernel wireless events.**

```bash
sudo iw event -t -f
```

**19. Read the regulatory domain.**

```bash
iw reg get
iw phy phy0 reg get
```

**20. Set the regulatory domain.**

```bash
sudo iw reg set GB
```

**21. Check whether the phy supports AP mode.**

```bash
iw phy phy0 info | sed -n '/Supported interface modes/,/^$/p' | grep -i ap
```

**22. Join an IBSS cell.**

```bash
sudo iw dev wlan0 ibss join TestCell 2412 HT20
```

**23. Join a mesh.**

```bash
sudo iw dev wlan0 mesh join meshnet freq 2412 HT20
```

**24. List mesh parameters.**

```bash
iw dev wlan0 mesh_param dump
```

**25. Read mesh paths.**

```bash
iw dev wlan0 mpath dump
iw dev wlan0 mpp dump
```

**26. Disable power save for a test.**

```bash
sudo iw dev wlan0 set power_save off
iw dev wlan0 get power_save
```

**27. Set a fixed transmit power.**

```bash
sudo iw dev wlan0 set txpower fixed 1500
```

**28. Show TX queue settings.**

```bash
iw phy phy0 get txq
```

**29. Move a phy into a network namespace.**

```bash
sudo iw phy phy0 set netns name testns
```

**30. Remove a virtual interface.**

```bash
sudo iw dev mon0 del
```

## Gotchas

- **`iw` is not a capture tool.** It creates and configures interfaces; use
  `tcpdump`, `wireshark`, `airodump-ng`, or `bettercap` to capture.
- **Output is explicitly not stable.** The help says "Do NOT screenscrape
  this tool." Parse netlink or use `-j`-capable tools when you need machine
  output.
- **`iw dev` vs `iw phy` is a real distinction.** Interface settings live on
  the netdev; capabilities, channels, antenna, and retry limits live on the
  phy. Some settings exist in both and mean different scopes.
- **Omitting `dev`/`phy` works only when unambiguous.** The help recommends
  against it in scripts.
- **Monitor mode is driver-dependent.** If `Supported interface modes` does
  not list `monitor`, no amount of `iw` will create a working monitor
  interface.
- **AP mode is normally driven by `hostapd`.** `iw dev … ap start` exists but
  requires a crafted beacon head/tail; the help lists only `managed`, `ibss`,
  `monitor`, `mesh`, and `wds` as settable types, so do not expect
  `set type ap` to work.
- **Bring the interface down before changing type.** Most drivers reject
  `set type` on an up interface.
- **`set channel` does not create monitor mode.** It sets the channel of an
  existing interface; the type must already be `monitor` for capture.
- **`scan` needs the interface up and not associated** on many drivers.
  Associated scans are possible but restricted.
- **`scan dump` shows the cache, not a new scan.** If the list is stale,
  trigger a fresh `scan`.
- **`connect` is not a supplicant.** It handles open/shared auth and simple
  keys; WPA2-Enterprise, 802.1X, and most roaming belong to
  `wpa_supplicant`.
- **`station del` sends a real deauth/disassoc frame.** It is disruptive and
  is not a harmless cache cleanup.
- **`cac` is a DFS requirement.** Transmitting on a DFS channel before the CAC
  completes is a regulatory violation, not just a failed command.
- **`reg set` may be overridden.** The kernel, the driver, and the country IE
  from the AP can all constrain the effective domain; check `reg get` after
  setting.
- **`phy ... set netns` moves the whole radio.** The interface disappears from
  the current namespace; that is the point, but it surprises people.
- **`wowlan` is hardware-specific.** Most laptop cards support only a subset
  of triggers; unsupported flags fail at enable time.
- **`hwsim` commands only work with `mac80211_hwsim`.** They are test hooks,
  not general radio controls.
- **`event` is verbose.** Run it in its own terminal and filter with `grep`
  when reproducing a specific association.
- **Root is required for anything that changes state.** Read-only `iw dev`,
  `iw phy`, `iw list`, and `iw reg get` work unprivileged.
- **`iwconfig` and `iwlist` are deprecated.** If a guide uses them, translate
  to `iw` rather than installing the old wireless-tools.

## Related docs

- [`aircrack-ng`](../aircrack-ng/README.md) — monitor mode, capture, injection
- [`wifite`](../wifite/README.md) — automated wireless auditing
- [`bettercap`](../bettercap/README.md) — Wi-Fi recon and attack framework
- [`reaver`](../reaver/README.md) — WPS PIN and Pixie-Dust
- [`wash`](../wash/README.md) — WPS survey
- [`macchanger`](../macchanger/README.md) — change the interface MAC
- [`hcxdumptool`](../hcxdumptool/README.md) — PMKID/EAPOL capture
- [`hcxtools`](../hcxtools/README.md) — capture conversion
- [`kismet`](../kismet/README.md) — passive wireless detection
- [`eaphammer`](../eaphammer/README.md) — 802.1X/EAP attacks
- [`wifiphisher`](../wifiphisher/README.md) — rogue AP phishing
- [`bully`](../bully/README.md) — WPS PIN brute force
- [`wireshark`](../wireshark/README.md) — frame analysis
- [`tcpdump`](../tcpdump/README.md) — packet capture
