# bluez — Bluetooth stack CLI suite

- **Version:** 5.82-1.1
- **Binaries:** `/usr/bin/bluetoothctl`, `/usr/bin/hciconfig`, `/usr/bin/hcitool`, `/usr/bin/btmon`, `/usr/bin/btmgmt`, `/usr/bin/l2ping`, `/usr/bin/sdptool`, `/usr/bin/rfcomm`, `/usr/bin/btattach`, `/usr/bin/hciattach`
- **Help snapshots:** [`help-bluetoothctl.txt`](help-bluetoothctl.txt), [`help-hciconfig.txt`](help-hciconfig.txt), [`help-hcitool.txt`](help-hcitool.txt), [`help-btmon.txt`](help-btmon.txt), [`help-btmgmt.txt`](help-btmgmt.txt), [`help-l2ping.txt`](help-l2ping.txt), [`help-sdptool.txt`](help-sdptool.txt), [`help-rfcomm.txt`](help-rfcomm.txt), [`help-btattach.txt`](help-btattach.txt), [`help-hciattach.txt`](help-hciattach.txt)
- **Upstream:** http://www.bluez.org/
- **Licence:** GPL-2.0-or-later

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

BlueZ is the Linux Bluetooth stack. The package ships both the `bluetoothd`
daemon and the command-line tools that drive it. For an assessment it covers
the whole Bluetooth surface:

- Classic BR/EDR discovery, name resolution, and pairing.
- BLE advertising, scanning, GATT attribute read/write, and notification.
- SDP service enumeration on remote devices.
- L2CAP reachability checks with `l2ping`.
- RFCOMM serial-port binding and listening.
- HCI-level monitoring and arbitrary command injection.
- Serial HCI controller attachment for embedded targets.

The stack is split into layers, and knowing which tool talks to which layer
prevents a lot of confusion:

```text
  bluetoothctl ──D-Bus──▶ bluetoothd ──mgmt/HCI──▶ kernel ──▶ controller
  btmgmt ─────────────────mgmt socket──────────────▶ kernel
  hcitool / hciconfig ─────────raw HCI ioctl────────▶ kernel
  btmon ──────────────────HCI/mgmt monitor──────────▶ kernel
  sdptool / l2ping / rfcomm ──L2CAP sockets─────────▶ kernel
```

`bluetoothctl` is the supported interface. `hciconfig` and `hcitool` are
deprecated and increasingly incomplete, but they remain useful because they
expose raw HCI operations that `bluetoothctl` does not. `btmgmt` uses the
kernel management API directly and is the closest replacement for the legacy
pair.

## Binaries

| Binary | Layer | Role |
|--------|-------|------|
| `bluetoothctl` | D-Bus → `bluetoothd` | Controller, device, agent, GATT, advertising, media control |
| `btmgmt` | Kernel mgmt socket | Low-level controller configuration and discovery |
| `hciconfig` | Raw HCI ioctl | Legacy controller configuration |
| `hcitool` | Raw HCI ioctl | Legacy discovery, LE scan, arbitrary HCI commands |
| `btmon` | HCI/mgmt monitor | Capture, replay, and analyse btsnoop traces |
| `l2ping` | L2CAP | Echo request to a remote BD_ADDR |
| `sdptool` | SDP | Browse and manage Service Discovery Protocol records |
| `rfcomm` | RFCOMM | Bind RFCOMM channels to TTYs, listen for connections |
| `btattach` | Serial | Attach a serial-attached Bluetooth controller |
| `hciattach` | Serial UART | Legacy UART HCI initialisation |

## `bluetoothctl`

```text
bluetoothctl [--options] [commands]
```

| Option | Meaning |
|--------|---------|
| `--agent CAP` | Register an agent handler with the given capability |
| `--endpoints` | Register media endpoints |
| `--monitor` | Enable monitor output |
| `--timeout SECONDS` | Timeout for non-interactive mode |
| `--version` | Print version |
| `--init-script FILE` | Run an init script |
| `--help` | Print help |

With no command it opens an interactive prompt. With one command it runs that
command and exits. Multiple commands require an interactive session or
`script <file>`; a submenu cannot be entered and used in a single non-interactive
invocation.

### Main-menu commands

| Group | Commands |
|-------|----------|
| Controllers | `list`, `show [ctrl]`, `select <ctrl>`, `system-alias`, `reset-alias` |
| Power and modes | `power <on/off>`, `pairable <on/off>`, `discoverable <on/off>`, `discoverable-timeout [value]` |
| Devices | `devices [Paired/Bonded/Trusted/Connected]`, `info [dev/set]`, `set-alias`, `remove` |
| Discovery | `scan <on/off/bredr/le>`, `pair [dev]`, `cancel-pairing [dev]` |
| Trust and access | `trust`, `untrust`, `block`, `unblock` |
| Connections | `connect <dev> [uuid]`, `disconnect [dev] [uuid]`, `wake`, `bearer` |
| Agents | `agent <on/off/auto/capability>`, `default-agent` |
| Advertising | `advertise <on/off/type>` |
| Scripting | `menu <name>`, `script <filename>`, `export`, `version`, `quit`, `help` |

### Submenus

`bluetoothctl help` lists the submenus: `advertise`, `monitor` (advertisement
monitor), `scan`, `gatt`, `admin`, `player`, `endpoint`, `transport`, `mgmt`,
`assistant`, and `hci`. Each has its own command set; the full text is in
[`help-bluetoothctl.txt`](help-bluetoothctl.txt).

The GATT submenu is the one most used in assessments:

| GATT command | Purpose |
|--------------|---------|
| `list-attributes [dev/local]` | List services, characteristics, descriptors |
| `select-attribute <attribute/UUID>` | Select an attribute |
| `attribute-info [attribute/UUID]` | Show attribute metadata |
| `read [offset]` | Read the selected value |
| `write <data=xx xx> [offset] [type]` | Write the selected value |
| `acquire-write` / `release-write` | Acquire or release a write file descriptor |
| `acquire-notify` / `release-notify` | Acquire or release a notify file descriptor |
| `notify` | Toggle notifications |
| `register-application` / `unregister-application` | Register a profile |
| `register-service` / `register-characteristic` / `register-descriptor` | Build a local GATT server |

Submenus are entered interactively or through a script:

```bash
printf 'menu gatt\nlist-attributes\nquit\n' | bluetoothctl --timeout 10
```

## `hciconfig`

```text
hciconfig
hciconfig [-a] hciX [command ...]
```

With no argument it lists every HCI device. `-a` adds features, packet types,
link policy, name, and class. Commands include `up`, `down`, `reset`, `rstat`,
`auth`, `noauth`, `encrypt`, `noencrypt`, `piscan`, `noscan`, `iscan`, `pscan`,
`ptype`, `lm`, `lp`, `name`, `class`, `voice`, `iac`, `inqtpl`, `inqmode`,
`inqdata`, `inqtype`, `inqparms`, `pageparms`, `pageto`, `afhmode`, `sspmode`,
`aclmtu`, `scomtu`, `delkey`, `oobdata`, `commands`, `features`, `version`,
`revision`, `block`, `unblock`, `lerandaddr`, `leadv`, `noleadv`, and
`lestates`.

`hciconfig` is deprecated. Prefer `bluetoothctl` for state changes and
`btmgmt` for controller features that `bluetoothctl` does not expose.

## `hcitool`

```text
hcitool [options] <command> [command parameters]
```

| Option | Meaning |
|--------|---------|
| `--help` | Help |
| `-i dev` | HCI device |

| Command group | Commands |
|---------------|----------|
| Local | `dev` |
| Classic discovery | `inq`, `scan`, `name`, `info`, `spinq`, `epinq` |
| Connections | `con`, `cc`, `dc`, `sr`, `cpt`, `rssi`, `lq`, `tpl`, `afh`, `lp`, `lst`, `auth`, `enc`, `key`, `clkoff`, `clock` |
| LE | `lescan`, `leinfo`, `lealadd`, `lealrm`, `lealsz`, `lealclr`, `lerladd`, `lerlrm`, `lerlclr`, `lerlsz`, `lerlon`, `lerloff`, `lecc`, `ledc`, `lecup` |
| Raw | `cmd <ogf> <ocf> [parameters]` |

`hcitool cmd` is the escape hatch for HCI commands that no other tool
implements. It takes the OGF and OCF as hex and passes the remaining bytes as
parameters.

## `btmon`

```text
btmon [options]
```

| Option | Meaning |
|--------|---------|
| `-r, --read FILE` | Read a btsnoop trace |
| `-w, --write FILE` | Write a btsnoop trace |
| `-a, --analyze FILE` | Analyse a btsnoop trace; plots latency if gnuplot is installed |
| `-s, --server SOCKET` | Start a monitor server socket |
| `-p, --priority LEVEL` | Show only this priority or lower |
| `-i, --index NUM` | Filter by controller index |
| `-d, --tty TTY` | Read from a TTY |
| `-B, --tty-speed RATE` | TTY speed (default 115200) |
| `-V, --vendor COMPID` | Default company identifier |
| `-M, --mgmt` | Open the mgmt channel |
| `-t, --time` | Show absolute time instead of offset |
| `-T, --date` | Show date and time |
| `-S, --sco` | Dump SCO traffic |
| `-A, --a2dp` | Dump A2DP stream traffic |
| `-I, --iso` | Dump ISO traffic |
| `-E, --ellisys [ip]` | Send Ellisys HCI injection |
| `-P, --no-pager` | Disable the pager |
| `-J, --jlink ...` | Read from RTT |
| `-R, --rtt ...` | RTT control-block parameters |
| `-C, --columns [width]` | Output width when not a terminal |
| `-c, --color [mode]` | `auto`, `always`, `never` |
| `-h, --help` | Help |

`btmon` with no source attaches to the live HCI/mgmt channel and blocks.
`-w` also blocks until interrupted, then writes the trace.

## `btmgmt`

```text
btmgmt [--options] [commands]
```

| Option | Meaning |
|--------|---------|
| `--index N` | Adapter index |
| `--monitor` | Enable monitor output |
| `--timeout SECONDS` | Timeout for non-interactive mode |
| `--version` | Version |
| `--init-script FILE` | Init script |
| `--help` | Help |

`btmgmt` commands cover controller information, power, discoverability,
pairing, LE configuration, advertising instances, PHY configuration, and
system configuration. On this host every command except `--version` blocked
until killed, even with `--timeout`; see the quirks section.

## `l2ping`

```text
l2ping [-i device] [-s size] [-c count] [-t timeout] [-d delay] [-f] [-r] [-v] <bdaddr>
```

| Option | Meaning |
|--------|---------|
| `-i device` | Local HCI device |
| `-s size` | Payload size |
| `-c count` | Number of pings |
| `-t timeout` | Per-ping timeout in seconds |
| `-d delay` | Delay between pings |
| `-f` | Flood ping (delay 0) |
| `-r` | Reverse ping |
| `-v` | Verify request and response payload |

`l2ping` confirms L2CAP reachability. It needs a routable, connectable
BD_ADDR; pinging the local adapter fails with `No route to host`.

## `sdptool`

```text
sdptool [options] <command> [command parameters]
```

| Command | Purpose |
|---------|---------|
| `search [--bdaddr BDADDR] [--tree] [--raw] [--xml] SERVICE` | Search for a service by name or UUID |
| `browse [--tree] [--raw] [--xml] BDADDR` | Browse every service |
| `records [--tree] [--raw] [--xml] BDADDR` | Request all records |
| `add [--handle=N --channel=N] SERVICE` | Add a local service record |
| `del RECORD_HANDLE` | Delete a local record |
| `get [--tree] [--raw] [--xml] [--bdaddr BDADDR] HANDLE` | Get a local record |
| `setattr RECORD_HANDLE ATTRIB_ID VALUE` | Set an attribute |
| `setseq RECORD_HANDLE ATTRIB_ID VALUES` | Set an attribute sequence |

Service names include `DID`, `DUN`, `LAN`, `FAX`, `OPUSH`, `FTP`, `PRINT`,
`HS`, `HSAG`, `HF`, `HFAG`, `SAP`, `PBAP`, `MAP`, `NAP`, `GN`, `PANU`, `HCRP`,
`HID`, `KEYB`, `WIIMOTE`, `A2SRC`, `A2SNK`, `AVRCT`, `AVRTG`, `GATT`, and
others listed in the help text.

## `rfcomm`

```text
rfcomm [options] <command> <dev>
```

| Option | Meaning |
|--------|---------|
| `-i, --device [hciX\|bdaddr]` | Local HCI device or BD_ADDR |
| `-r, --raw` | Raw TTY mode |
| `-A, --auth` | Enable authentication |
| `-E, --encrypt` | Enable encryption |
| `-S, --secure` | Secure connection |
| `-C, --central` | Become piconet central |
| `-L, --linger SECONDS` | Linger timeout |
| `-a` | Show all devices (default) |

| Command | Purpose |
|---------|---------|
| `bind <dev> <bdaddr> [channel]` | Bind an RFCOMM channel to a TTY |
| `release <dev>` | Release the TTY |
| `show <dev>` | Show the binding |
| `connect <dev> <bdaddr> [channel]` | Connect and attach a TTY |
| `listen <dev> [channel [cmd]]` | Listen for an incoming connection |
| `watch <dev> [channel [cmd]]` | Watch for connections |

## `btattach` and `hciattach`

`btattach` attaches a serial-attached controller:

```text
btattach [-B bredr-device] [-A amp-device] [-P protocol] [-S baudrate] [-N] [-h]
```

`hciattach` is the legacy UART initialiser:

```text
hciattach [-n] [-p] [-b] [-r] [-t timeout] [-s initial_speed] <tty> <type|id> [speed] [flow|noflow] [sleep|nosleep] [bdaddr]
hciattach -l
```

`hciattach -l` lists supported device types. `btattach` is the modern
replacement for UART controllers that the kernel does not bind automatically.

## Recipes

**1. Inventory controllers and devices.**

```bash
bluetoothctl list
bluetoothctl show
bluetoothctl devices
bluetoothctl devices Connected
```

**2. Legacy controller state.**

```bash
hciconfig
hciconfig -a
hciconfig hci0 version
hcitool dev
```

**3. Discover nearby Classic devices.**

```bash
timeout 20 hcitool scan
timeout 20 hcitool inq
```

**4. Discover BLE devices.**

```bash
timeout 20 hcitool lescan
timeout 20 bluetoothctl --timeout 15 scan le
```

**5. Read a remote device's name and details.**

```bash
hcitool name AA:BB:CC:DD:EE:FF
hcitool info AA:BB:CC:DD:EE:FF
bluetoothctl info AA:BB:CC:DD:EE:FF
```

**6. Enumerate SDP services.**

```bash
sdptool browse AA:BB:CC:DD:EE:FF
sdptool search --bdaddr AA:BB:CC:DD:EE:FF HID
sdptool records AA:BB:CC:DD:EE:FF
```

**7. Enumerate GATT attributes.**

```bash
printf 'menu gatt\nlist-attributes\nquit\n' | bluetoothctl --timeout 10
```

**8. Read a GATT characteristic.**

```bash
printf 'menu gatt\nselect-attribute /org/bluez/hci0/dev_AA_BB_CC_DD_EE_FF/service0001/char0002\nread\nquit\n' \
  | bluetoothctl --timeout 10
```

**9. Check L2CAP reachability.**

```bash
l2ping -c 3 AA:BB:CC:DD:EE:FF
l2ping -c 3 -s 64 -t 2 AA:BB:CC:DD:EE:FF
```

**10. Capture an HCI trace.**

```bash
sudo timeout 60 btmon -w trace.btsnoop
btmon -r trace.btsnoop
btmon -a trace.btsnoop
```

**11. Bind an RFCOMM channel to a TTY.**

```bash
sudo rfcomm bind 0 AA:BB:CC:DD:EE:FF 1
ls -l /dev/rfcomm0
sudo rfcomm release 0
```

**12. Attach a serial controller.**

```bash
sudo btattach -B /dev/ttyUSB0 -S 115200
```

## Exit status

| Tool | Code | Meaning |
|------|------|---------|
| `bluetoothctl` | 0 | Command completed, including some `Failed` replies |
| `bluetoothctl` | 1 | Missing argument or unavailable controller/device |
| `bluetoothctl` | 124 under `timeout` | Blocking scan, monitor, or submenu |
| `hciconfig` | 0 | Query completed, even for an unknown device name |
| `hcitool` | 0 | Query or usage message; many missing-argument paths still exit 0 |
| `hcitool` | 124 under `timeout` | Blocking `scan`, `lescan`, `inq` |
| `btmon` | 0 | Help, or read of a missing trace file |
| `btmon` | 124 under `timeout` | Live capture or `-w` |
| `btmgmt` | 0 | `--version` only on this host |
| `btmgmt` | 124 under `timeout` | Every other invocation |
| `l2ping` | 0 | At least one reply |
| `l2ping` | 1 | Usage error or no route |
| `sdptool` | 0 | `records` even when the connection failed |
| `sdptool` | 255 | Usage error or failed browse/search |
| `rfcomm` | 0 | Bind, release, or listing |
| `rfcomm` | 1 | Missing command argument |
| `btattach` | 1 | No controller attached |
| `hciattach` | 0 | `-l` list |
| `hciattach` | 1 | Usage error or unknown device type |

## Quirks and limits

- **`bluetoothctl --bogus` exits 0.** It prints `unrecognized option` and the
  usage text, but the exit code is success. Check the output, not only `$?`.
- **Non-interactive submenus do not work.** `bluetoothctl menu gatt
  list-attributes` is parsed as one invalid command. Pipe a script or use
  `script <file>`.
- **`bluetoothctl pair` with no argument uses a device.** On this host it
  attempted the first known device and failed with `AlreadyExists`. Always
  pass an explicit BD_ADDR.
- **`bluetoothctl scan off` when idle exits 0.** It prints
  `Failed to stop discovery: org.bluez.Error.Failed`; that is not a transport
  error.
- **`bluetoothctl export` blocks.** It prints pointer values and stays in the
  REPL; wrap it in `timeout`.
- **`btmgmt` blocks on this host.** `--version` works, but `--help`, `info`,
  `commands`, and `find` all printed their output and then hung until killed.
  Run it under `timeout` and treat 124 as expected.
- **`hciconfig nosuch` falls back to the first adapter.** It printed `hci0`
  state and exited 0. Do not use a wrong device name to test for absence.
- **`hcitool` missing arguments often exit 0.** `name`, `info`, `rssi`, and
  `cmd` print usage and return success. Parse the text.
- **`hcitool scan`/`lescan`/`inq` block.** They do not stop on their own;
  `timeout` is required for scripted use.
- **`sdptool browse` with no address blocks.** It appears to browse the local
  adapter. `sdptool records` on an unreachable address exits 0 after printing
  a failure, while `browse` and `search` exit 255 for the same failure.
- **`sdptool` subcommands print usage and exit 255 when arguments are
  missing.** `browse` is the exception and blocks.
- **`rfcomm` with no arguments exits 0 and prints nothing.** `rfcomm show`
  without a device prints usage and exits 1.
- **`btmon -r` on a missing file exits 0.** It prints only the version banner.
  Confirm the trace exists before trusting an empty analysis.
- **`btmon -w` writes while blocking.** The file grows during capture; kill it
  cleanly with `SIGINT` or `timeout` so the trace is flushed.
- **`l2ping` cannot ping the local adapter.** Expect `No route to host`.
- **`btattach` needs a real serial controller.** A plain `/dev/ttyS0` fails
  with `Failed to flush serial port` and `No controller attached`.
- **`bluetoothd` must be running.** Most D-Bus commands fail or hang without
  it; check `systemctl status bluetooth` first.
- **State changes persist.** `power off`, `block`, `trust`, `pairable`, and
  `discoverable` alter the host's adapter or pairing database. Record the
  original state and restore it.
- **BD_ADDRs are not secrets but they are identifiers.** Treat captured
  device lists and traces as personal data.

## Related tools

- [`bettercap`](../bettercap/) — BLE and Wi-Fi reconnaissance and MITM.
- [`wireshark`](../wireshark/) — dissect btsnoop and Bluetooth traffic.
- [`kismet`](../kismet/) — passive wireless and Bluetooth detection.
- `ubertooth` — dedicated Bluetooth sniffing hardware and host tools (not
  installed here).
