# bluez — command scenarios for deep research

Installed build: bluez 5.82-1.1, with `bluetoothctl`, `hciconfig`, `hcitool`,
`btmon`, `btmgmt`, `l2ping`, `sdptool`, `rfcomm`, `btattach`, and `hciattach`.
Every option below appears in one of the ten help snapshots; behaviour was
observed on this host against adapter `hci0` (`5C:BA:EF:EA:C6:78`) and its
paired devices.

> **Scope:** Bluetooth scanning, pairing, advertising, and service enumeration
> are active radio operations. Only run them against devices you own or are
> explicitly authorised to test, and expect nearby personal devices to appear
> in scans. Do not pair with, read, or write to third-party devices.

> **State changes persist.** `power`, `pairable`, `discoverable`, `trust`,
> `block`, and `advertise` modify the host adapter or pairing database. Record
> the original values first and restore them at the end.

> **Privileges:** `bluetoothctl` normally works as a normal user through
> D-Bus and the `bluetooth` group. `btmon`, raw HCI commands, `hciconfig`
> writes, `rfcomm bind`, and `btattach` need root or `CAP_NET_ADMIN`.

## How to invoke

```text
@deep-research For an authorised Bluetooth assessment with BlueZ 5.82,
explain the split between the D-Bus `bluetoothd` interface (`bluetoothctl`),
the kernel mgmt API (`btmgmt`), and raw HCI (`hcitool`, `hciconfig`), how
Classic discovery, BLE scanning, SDP browsing, and GATT attribute access
differ, and which commands need root. Show which invocations block, which
return 0 despite a `Failed` reply, how non-interactive submenus and scripts
work, and how btsnoop traces are captured and replayed. Cite the installed
help text, the BlueZ documentation, and observed output. Separate confirmed
behaviour from inference and state confidence.
```

Create a working directory and record the environment:

```bash
mkdir -p ~/bluez-demo/evidence
cd ~/bluez-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
bluetoothctl --version > evidence/version.txt
rfkill list bluetooth > evidence/rfkill.txt
systemctl is-active bluetooth > evidence/bluetoothd.txt
```

## Scenario 1 — version, help, binaries, package provenance

| # | Command | Purpose |
|---|---------|---------|
| 1 | `bluetoothctl --version` | Control CLI version |
| 2 | `bluetoothctl version` | Same value from the REPL command |
| 3 | `bluetoothctl --help` | Options and top-level command list |
| 4 | `bluetoothctl help` | Main menu with submenus |
| 5 | `btmgmt --version` | Mgmt tool version |
| 6 | `hciconfig` | Legacy tool presence and usage |
| 7 | `hcitool --help` | Legacy HCI command list |
| 8 | `btmon --help` | Monitor options |
| 9 | `l2ping` | Usage when BD_ADDR is missing |
| 10 | `sdptool -h` | SDP command list |
| 11 | `rfcomm` | Usage and commands |
| 12 | `btattach --help` | Serial attach options |
| 13 | `hciattach -l` | Supported UART device types |
| 14 | `command -v bluetoothctl hciconfig hcitool btmon btmgmt l2ping sdptool rfcomm btattach hciattach` | Resolve all ten |
| 15 | `readlink -f "$(command -v bluetoothctl)"` | Real path |
| 16 | `file "$(command -v bluetoothctl)"` | Confirm a native ELF |
| 17 | `dpkg -S /usr/bin/bluetoothctl` | Owning package |
| 18 | `dpkg -s bluez \| grep -E 'Version\|Depends'` | Package provenance |
| 19 | `dpkg -L bluez \| grep -E 'bin/'` | Shipped executables |
| 20 | `systemctl is-active bluetooth` | Daemon state |
| 21 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt` | Timestamp the run |

**Research questions**
- Which package owns all ten binaries?
- Do `--version` and the `version` command agree?
- Which tools are deprecated but still shipped?
- Is `bluetoothd` running, and does it matter for each tool?

**Sources:** per-tool help; `dpkg -S`; `systemctl`; observed output.
**Output:** a provenance record, a tool inventory, and a daemon-state check.

## Scenario 2 — controller inventory and state

| # | Command | Purpose |
|---|---------|---------|
| 22 | `bluetoothctl list` | Controllers known to BlueZ |
| 23 | `bluetoothctl show` | Default controller details |
| 24 | `bluetoothctl show 5C:BA:EF:EA:C6:78` | Explicit controller |
| 25 | `bluetoothctl show nosuch` | Unknown controller |
| 26 | `bluetoothctl select 5C:BA:EF:EA:C6:78` | Set the default controller |
| 27 | `bluetoothctl select nosuch` | Unknown controller on select |
| 28 | `bluetoothctl system-alias lab-adapter` | Set a temporary alias |
| 29 | `bluetoothctl reset-alias` | Restore the alias |
| 30 | `hciconfig` | Legacy device list |
| 31 | `hciconfig -a` | Full legacy state |
| 32 | `hciconfig hci0` | One adapter |
| 33 | `hciconfig nosuch` | Unknown name falls back |
| 34 | `hciconfig hci0 version` | HCI and LMP versions |
| 35 | `hciconfig hci0 features` | Supported features |
| 36 | `hciconfig hci0 commands` | Supported HCI commands |
| 37 | `hciconfig hci0 ptype` | Packet types |
| 38 | `hciconfig hci0 lm` | Link mode |
| 39 | `hciconfig hci0 lp` | Link policy |
| 40 | `hcitool dev` | Local HCI devices |
| 41 | `rfkill list bluetooth` | Soft and hard block state |
| 42 | `bluetoothctl show > evidence/controller-before.txt` | Save the baseline |
| 43 | `diff <(bluetoothctl show) evidence/controller-before.txt` | Stability check |

**Research questions**
- Which fields come from the controller and which from `bluetoothd`?
- Does `hciconfig nosuch` really fall back to `hci0`?
- Which HCI version and manufacturer does this adapter report?
- Does `system-alias` change the controller name or only the BlueZ alias?

**Sources:** `bluetoothctl help`; `hciconfig` help; observed output.
**Output:** a controller field map and a saved baseline.

## Scenario 3 — device inventory and device information

| # | Command | Purpose |
|---|---------|---------|
| 44 | `bluetoothctl devices` | All known devices |
| 45 | `bluetoothctl devices Paired` | Paired devices only |
| 46 | `bluetoothctl devices Bonded` | Bonded devices only |
| 47 | `bluetoothctl devices Trusted` | Trusted devices only |
| 48 | `bluetoothctl devices Connected` | Currently connected devices |
| 49 | `bluetoothctl info` | First device's details |
| 50 | `bluetoothctl info FC:58:FA:D4:55:D9` | One device by BD_ADDR |
| 51 | `bluetoothctl info nosuch` | Unknown device |
| 52 | `bluetoothctl set-alias lab-headset FC:58:FA:D4:55:D9` | Set a local alias |
| 53 | `hcitool name 5C:BA:EF:EA:C6:78` | Remote name request |
| 54 | `hcitool info 5C:BA:EF:EA:C6:78` | Remote information |
| 55 | `hcitool con` | Active ACL connections |
| 56 | `hcitool rssi 78:75:67:32:D1:F7` | Connection RSSI |
| 57 | `hcitool lq 78:75:67:32:D1:F7` | Link quality |
| 58 | `hcitool tpl 78:75:67:32:D1:F7` | Transmit power level |
| 59 | `hcitool afh 78:75:67:32:D1:F7` | AFH channel map |
| 60 | `hcitool lst 78:75:67:32:D1:F7` | Link supervision timeout |
| 61 | `hcitool clock 78:75:67:32:D1:F7` | Remote clock |
| 62 | `bluetoothctl devices > evidence/devices.txt` | Save the inventory |
| 63 | `wc -l evidence/devices.txt` | Device count |
| 64 | `bluetoothctl info 78:75:67:32:D1:F7 > evidence/device-info.txt` | Save details |

**Research questions**
- What do `Paired`, `Bonded`, `Trusted`, and `Connected` mean independently?
- Which fields require the device to be connected?
- Does `hcitool name` work without pairing?
- Which commands return data for an unknown BD_ADDR?

**Sources:** `bluetoothctl help`; `hcitool` help; observed output.
**Output:** a device-property reference and a saved inventory.

## Scenario 4 — Classic and BLE discovery

| # | Command | Purpose |
|---|---------|---------|
| 65 | `timeout 20 hcitool scan` | Classic inquiry |
| 66 | `timeout 20 hcitool inq` | Inquiry with RSSI |
| 67 | `hcitool scan --help` | Scan options |
| 68 | `hcitool inq --help` | Inquiry options |
| 69 | `timeout 20 hcitool lescan` | BLE scan |
| 70 | `hcitool lescan --help` | LE scan options |
| 71 | `timeout 20 hcitool lescan --passive` | Passive LE scan |
| 72 | `timeout 20 hcitool lescan --duplicates` | Do not filter duplicates |
| 73 | `timeout 20 hcitool lescan --acceptlist` | Scan the accept list only |
| 74 | `timeout 20 hcitool lescan --privacy` | Enable privacy |
| 75 | `bluetoothctl --timeout 15 scan on` | BlueZ discovery |
| 76 | `bluetoothctl --timeout 15 scan bredr` | Classic only |
| 77 | `bluetoothctl --timeout 15 scan le` | LE only |
| 78 | `bluetoothctl scan off` | Stop discovery |
| 79 | `bluetoothctl --timeout 15 scan on > evidence/scan.txt` | Save discovery output |
| 80 | `grep -c 'Device' evidence/scan.txt` | Device count |
| 81 | `grep -i 'audio\|headset\|phone' evidence/scan.txt` | Classify by name |
| 82 | `hcitool spinq` | Start periodic inquiry |
| 83 | `hcitool epinq` | Stop periodic inquiry |
| 84 | `btmgmt find` | Mgmt discovery (blocks) |
| 85 | `iw dev` | Wi-Fi view for cross-reference |

**Research questions**
- How long does each scan run before it is killed?
- Does `hcitool lescan` show addresses, names, or both?
- Does `bluetoothctl --timeout` stop discovery automatically?
- Which scan type sees the most devices in this environment?

**Sources:** `hcitool` help; `bluetoothctl` help; observed output.
**Output:** a scan comparison table and a device-count baseline.

## Scenario 5 — SDP, GATT, and L2CAP enumeration

| # | Command | Purpose |
|---|---------|---------|
| 86 | `sdptool browse 78:75:67:32:D1:F7` | Browse all SDP services |
| 87 | `sdptool browse --tree 78:75:67:32:D1:F7` | Tree output |
| 88 | `sdptool browse --raw 78:75:67:32:D1:F7` | Raw output |
| 89 | `sdptool browse --xml 78:75:67:32:D1:F7` | XML output |
| 90 | `sdptool records 78:75:67:32:D1:F7` | All records |
| 91 | `sdptool records --tree 78:75:67:32:D1:F7` | Tree records |
| 92 | `sdptool search --bdaddr 78:75:67:32:D1:F7 HID` | Search by service name |
| 93 | `sdptool search --bdaddr 78:75:67:32:D1:F7 0x1124` | Search by UUID |
| 94 | `sdptool search --tree --bdaddr 78:75:67:32:D1:F7 A2SRC` | Tree search |
| 95 | `sdptool get 0x10000` | Local record by handle |
| 96 | `sdptool get --tree 0x10000` | Tree form |
| 97 | `sdptool get --xml 0x10000` | XML form |
| 98 | `sdptool add --channel=3 SP` | Add a local record |
| 99 | `sdptool del 0x10000` | Delete a local record |
| 100 | `sdptool setattr 0x10000 0x0100 lab` | Set an attribute |
| 101 | `sdptool setseq 0x10000 0x0100 0x0001` | Set a sequence |
| 102 | `printf 'menu gatt\nlist-attributes\nquit\n' \| bluetoothctl --timeout 10` | GATT attributes |
| 103 | `printf 'menu gatt\nlist-attributes local\nquit\n' \| bluetoothctl --timeout 10` | Local attributes |
| 104 | `printf 'menu gatt\nselect-attribute /org/bluez/hci0/dev_78_75_67_32_D1_F7/service0001/char0002\nread\nquit\n' \| bluetoothctl --timeout 10` | Read a characteristic |
| 105 | `printf 'menu gatt\nselect-attribute /org/bluez/hci0/dev_78_75_67_32_D1_F7/service0001/char0002\nnotify\nquit\n' \| bluetoothctl --timeout 10` | Enable notifications |
| 106 | `hcitool leinfo 78:75:67:32:D1:F7` | LE remote information |
| 107 | `hcitool leinfo --static 78:75:67:32:D1:F7` | Static address form |
| 108 | `l2ping -c 3 78:75:67:32:D1:F7` | L2CAP echo |
| 109 | `l2ping -c 3 -s 64 -t 2 78:75:67:32:D1:F7` | Sized and timed echo |
| 110 | `l2ping -c 3 -f 78:75:67:32:D1:F7` | Flood ping |
| 111 | `l2ping -c 3 -r 78:75:67:32:D1:F7` | Reverse ping |
| 112 | `l2ping -c 3 -v 78:75:67:32:D1:F7` | Verify payload |
| 113 | `sdptool browse 78:75:67:32:D1:F7 > evidence/sdp.txt` | Save SDP output |

**Research questions**
- Which SDP services does this headset advertise?
- Does GATT access require an active connection?
- Which L2CAP options change behaviour on a headset?
- Does `sdptool get` address the local database or a remote device?

**Sources:** `sdptool` help; `bluetoothctl` help; observed output.
**Output:** an SDP/GATT service map and an L2CAP reachability record.

## Scenario 6 — advertising, pairing, and state changes

Record the original state first and restore it afterwards.

| # | Command | Purpose |
|---|---------|---------|
| 114 | `bluetoothctl show > evidence/state-before.txt` | Baseline controller state |
| 115 | `bluetoothctl --timeout 2 power off` | Power the adapter down |
| 116 | `bluetoothctl --timeout 2 power on` | Power it back up |
| 117 | `bluetoothctl --timeout 2 pairable on` | Enable pairing |
| 118 | `bluetoothctl --timeout 2 pairable off` | Disable pairing |
| 119 | `bluetoothctl --timeout 2 discoverable on` | Enable discoverability |
| 120 | `bluetoothctl --timeout 2 discoverable off` | Disable discoverability |
| 121 | `bluetoothctl --timeout 2 discoverable-timeout 180` | Set the timeout |
| 122 | `bluetoothctl --timeout 2 agent on` | Register the default agent |
| 123 | `bluetoothctl --timeout 2 default-agent` | Mark it default |
| 124 | `bluetoothctl --timeout 2 advertise on` | Start LE advertising |
| 125 | `bluetoothctl --timeout 2 advertise off` | Stop LE advertising |
| 126 | `printf 'menu advertise\nuuids\nquit\n' \| bluetoothctl --timeout 10` | Show advertised UUIDs |
| 127 | `printf 'menu advertise\nname lab\nquit\n' \| bluetoothctl --timeout 10` | Set advertised name |
| 128 | `printf 'menu advertise\nclear\nquit\n' \| bluetoothctl --timeout 10` | Clear advertising config |
| 129 | `bluetoothctl pair 78:75:67:32:D1:F7` | Pair with a device |
| 130 | `bluetoothctl cancel-pairing 78:75:67:32:D1:F7` | Cancel pairing |
| 131 | `bluetoothctl trust 78:75:67:32:D1:F7` | Trust a device |
| 132 | `bluetoothctl untrust 78:75:67:32:D1:F7` | Untrust a device |
| 133 | `bluetoothctl block 78:75:67:32:D1:F7` | Block a device |
| 134 | `bluetoothctl unblock 78:75:67:32:D1:F7` | Unblock a device |
| 135 | `bluetoothctl connect 78:75:67:32:D1:F7` | Connect all profiles |
| 136 | `bluetoothctl disconnect 78:75:67:32:D1:F7` | Disconnect |
| 137 | `bluetoothctl wake 78:75:67:32:D1:F7` | Read wake support |
| 138 | `bluetoothctl bearer 78:75:67:32:D1:F7` | Read preferred bearer |
| 139 | `bluetoothctl remove 78:75:67:32:D1:F7` | Remove the pairing |
| 140 | `bluetoothctl show > evidence/state-after.txt` | Compare afterwards |
| 141 | `diff evidence/state-before.txt evidence/state-after.txt` | Show drift |

**Research questions**
- Which changes survive a `bluetoothctl` process exit?
- Does `pair` need an agent and an active scan?
- What happens to audio routing when `power off` is issued?
- Does `remove` delete the link key from the controller?

**Sources:** `bluetoothctl help`; observed output.
**Output:** a state-change matrix and a restore checklist.

## Scenario 7 — monitoring, traces, and raw HCI

| # | Command | Purpose |
|---|---------|---------|
| 142 | `sudo timeout 20 btmon -w trace.btsnoop` | Capture live HCI traffic |
| 143 | `ls -l trace.btsnoop` | Confirm the trace was written |
| 144 | `btmon -r trace.btsnoop` | Replay the trace |
| 145 | `btmon -a trace.btsnoop` | Analyse the trace |
| 146 | `btmon -r trace.btsnoop -t` | Absolute time |
| 147 | `btmon -r trace.btsnoop -T` | Date and time |
| 148 | `btmon -r trace.btsnoop -S` | Include SCO traffic |
| 149 | `btmon -r trace.btsnoop -A` | Include A2DP traffic |
| 150 | `btmon -r trace.btsnoop -I` | Include ISO traffic |
| 151 | `btmon -M` | Open the mgmt channel |
| 152 | `btmon -i 0` | Filter controller 0 |
| 153 | `btmon -P` | Disable the pager |
| 154 | `btmon -c never` | Disable colour |
| 155 | `hcitool cmd 0x03 0x0000` | Read the local name via raw HCI |
| 156 | `hcitool cmd 0x04 0x0001` | Read local version information |
| 157 | `hcitool cmd 0x01 0x0001` | Read the buffer size |
| 158 | `hcitool cmd 0x03 0x0013 0x41 0x42` | Send arbitrary parameters |
| 159 | `hciconfig hci0 commands` | Supported command bitmask |
| 160 | `hciconfig hci0 features` | Supported feature bitmask |
| 161 | `dmesg \| grep -i bluetooth \| tail -20` | Kernel-side messages |
| 162 | `journalctl -u bluetooth -n 30 --no-pager` | Daemon log |
| 163 | `wc -c trace.btsnoop` | Trace size |

**Research questions**
- Which HCI commands does this adapter actually accept?
- Does `btmon -a` need gnuplot for the latency plot?
- Which events appear during scan, pair, and connect?
- Are raw HCI commands subject to the same permissions as `bluetoothctl`?

**Sources:** `btmon --help`; `hcitool` help; observed output.
**Output:** a trace file, an event timeline, and a raw-HCI command log.

## Scenario 8 — failure modes, serial attach, and cleanup

| # | Command | Purpose |
|---|---------|---------|
| 164 | `bluetoothctl connect` | Missing device argument |
| 165 | `bluetoothctl remove` | Missing device argument |
| 166 | `bluetoothctl wake` | Missing device argument |
| 167 | `bluetoothctl bearer` | Missing device argument |
| 168 | `bluetoothctl show nosuch` | Unknown controller |
| 169 | `bluetoothctl select nosuch` | Unknown controller on select |
| 170 | `bluetoothctl --bogus` | Unrecognised option |
| 171 | `bluetoothctl script /tmp/nosuch.script` | Missing script |
| 172 | `bluetoothctl info nosuch` | Unknown device |
| 173 | `hcitool name` | Missing BD_ADDR |
| 174 | `hcitool info` | Missing BD_ADDR |
| 175 | `hcitool rssi` | Missing BD_ADDR |
| 176 | `hcitool cmd` | Missing OGF/OCF |
| 177 | `hcitool scan --help` | Scan option reference |
| 178 | `sdptool search` | Missing service |
| 179 | `sdptool records` | Missing BD_ADDR |
| 180 | `sdptool get` | Missing record handle |
| 181 | `sdptool add` | Missing service |
| 182 | `rfcomm show` | Missing device |
| 183 | `l2ping` | Missing BD_ADDR |
| 184 | `l2ping 5C:BA:EF:EA:C6:78` | Local adapter is unroutable |
| 185 | `btattach` | No controller |
| 186 | `btattach -B /dev/ttyS0` | Plain TTY is not a controller |
| 187 | `hciattach` | Missing arguments |
| 188 | `hciattach /dev/ttyS0 bogus` | Unknown device type |
| 189 | `btmon -r /tmp/nosuch.btsnoop` | Missing trace exits 0 |
| 190 | `btmon -a /tmp/nosuch.btsnoop` | Missing trace on analyse |
| 191 | `timeout 3 btmgmt info` | Mgmt tool blocks |
| 192 | `pgrep -af 'btmon\|bluetoothctl'` | No leftover processes |
| 193 | `rm -rf ~/bluez-demo` | Remove the demo tree |
| 194 | `test ! -e ~/bluez-demo; echo "removed=$?"` | Verify cleanup |

**Research questions**
- Which missing-argument paths exit 0 and which exit non-zero?
- Which failures are D-Bus errors and which are kernel or HCI errors?
- Does a failed `btmon -r` differ from an empty trace?
- Which tools leave state or processes behind?

**Sources:** per-tool help; observed output.
**Output:** an error matrix, a serial-attach record, and a clean workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| versions, help, ten binaries, package provenance, daemon state | 1 |
| `list`, `show`, `select`, `system-alias`, `reset-alias`, `hciconfig` state, `hcitool dev`, `rfkill` | 2 |
| `devices` filters, `info`, `set-alias`, `hcitool name/info/con/rssi/lq/tpl/afh/lst/clock` | 3 |
| `hcitool scan/inq/lescan/spinq/epinq`, `bluetoothctl scan`, `btmgmt find` | 4 |
| `sdptool browse/records/search/get/add/del/setattr/setseq`, GATT submenu, `leinfo`, `l2ping` | 5 |
| `power`, `pairable`, `discoverable`, `discoverable-timeout`, `agent`, `default-agent`, `advertise`, `pair`, `trust`, `block`, `connect`, `remove` | 6 |
| `btmon` `-w`/`-r`/`-a`/`-t`/`-T`/`-S`/`-A`/`-I`/`-M`/`-i`/`-P`/`-c`, `hcitool cmd`, `hciconfig commands/features`, logs | 7 |
| missing arguments, bad options, serial attach, missing traces, blocking tools, cleanup | 8 |

Full text: [`help-bluetoothctl.txt`](help-bluetoothctl.txt),
[`help-hciconfig.txt`](help-hciconfig.txt), [`help-hcitool.txt`](help-hcitool.txt),
[`help-btmon.txt`](help-btmon.txt), [`help-btmgmt.txt`](help-btmgmt.txt),
[`help-l2ping.txt`](help-l2ping.txt), [`help-sdptool.txt`](help-sdptool.txt),
[`help-rfcomm.txt`](help-rfcomm.txt), [`help-btattach.txt`](help-btattach.txt),
[`help-hciattach.txt`](help-hciattach.txt); authoritative protocol detail in
the Bluetooth Core Specification and the BlueZ documentation.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, help, binaries, package provenance | 21 |
| 2 | Controller inventory and state | 22 |
| 3 | Device inventory and device information | 21 |
| 4 | Classic and BLE discovery | 21 |
| 5 | SDP, GATT, and L2CAP enumeration | 28 |
| 6 | Advertising, pairing, and state changes | 28 |
| 7 | Monitoring, traces, and raw HCI | 22 |
| 8 | Failure modes, serial attach, and cleanup | 31 |
| **Total** | | **194** |

## Legal and safety

Bluetooth scanning, pairing, service enumeration, and advertising are active
radio operations against devices that may belong to other people. Only test
devices and adapters you own or are explicitly authorised to assess. Do not
pair with, read, write, or block third-party devices. State changes such as
`power off`, `block`, and `remove` can disconnect the user's keyboard, mouse,
or headset; record the original state and restore it. HCI traces and device
lists contain BD_ADDRs, device names, and sometimes service data; treat them
as personal data and delete the demo tree when finished. Raw HCI commands can
brick or destabilise a controller; keep them to a lab adapter.
