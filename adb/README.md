# adb — Android Debug Bridge

- **Version:** 1.0.41 (Android SDK platform-tools `34.0.5-debian`)
- **Binary:** `/usr/bin/adb` → `/usr/lib/android-sdk/platform-tools/adb`
- **Help snapshot:** [`help.txt`](help.txt) — `adb --help`
- **Upstream:** https://android.googlesource.com/platform/packages/modules/adb/
- **Manual:** https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/master/docs/user/adb.1.md
- **Licence:** Apache-2.0

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

`adb` is the host-side client and server for Android devices. The client sends
commands to a background server, which multiplexes USB and TCP connections to
one or more devices. It is the transport for app installation, file transfer,
shell access, log capture, port forwarding, and reboot/bootloader workflows.

For authorised mobile assessments it is the first foothold: enumerate the
device, pull the APK, read logs, forward a proxy port, and drive the app. For
forensic work it can image userdata only on an unlocked or rooted device;
`adb backup` is deprecated and app-dependent.

## Architecture

```text
adb client  ──TCP 5037──▶  adb server  ──USB / TCP──▶  adbd on device
```

- The **client** is the command you run.
- The **server** starts on first use and listens on `localhost:5037`.
- `adbd` runs on the device and exposes the services.
- `-H`, `-P`, and `-L` relocate the server host, port, or listening socket.
- `start-server` / `kill-server` control the server explicitly.

## Global options

| Option | Meaning |
|--------|---------|
| `-a` | Listen on all interfaces, not just localhost |
| `-d` | Use the single USB device; error if several |
| `-e` | Use the single TCP/IP device; error if several |
| `-s SERIAL` | Target a device by serial (overrides `$ANDROID_SERIAL`) |
| `-t ID` | Target a transport id |
| `-H HOST` | Server host (default `localhost`) |
| `-P PORT` | Server port (default `5037`) |
| `-L SOCKET` | Server listen socket (default `tcp:localhost:5037`) |
| `--one-device SERIAL\|USB` | Restrict the server to one USB device |
| `--exit-on-write-error` | Exit when stdout is closed |

## Command groups

### Device discovery

| Command | Meaning |
|---------|---------|
| `devices [-l]` | List devices; `-l` adds model, product, device, transport id |
| `get-state` | Print `offline`, `bootloader`, or `device` |
| `get-serialno` | Print the serial number |
| `get-devpath` | Print the device path |
| `wait-for[-TRANSPORT]-STATE` | Block until a device reaches a state |
| `mdns check` | Report whether mDNS discovery is available |
| `mdns services` | List discovered mDNS services |
| `connect HOST[:PORT]` | Connect over TCP (default port 5555) |
| `disconnect [HOST[:PORT]]` | Drop one TCP device, or all |
| `pair HOST[:PORT] [CODE]` | Pair for secure TCP/IP debugging |

`wait-for` states are `device`, `recovery`, `rescue`, `sideload`,
`bootloader`, and `disconnect`. Transports are `usb`, `local`, and `any`.

### File transfer

| Command | Meaning |
|---------|---------|
| `push [--sync] [-z ALG] [-Z] LOCAL... REMOTE` | Copy to the device |
| `push -n ...` | Dry run: report without writing |
| `pull [-a] [-z ALG] [-Z] REMOTE... LOCAL` | Copy from the device |
| `sync [-l] [-n] [-z ALG] [-Z] [all\|data\|...]` | Sync `$ANDROID_PRODUCT_OUT` |

Compression algorithms are `any`, `none`, `brotli`, `lz4`, and `zstd`.
`-Z` disables compression; `-a` preserves timestamp and mode on `pull`.

### Shell and app management

| Command | Meaning |
|---------|---------|
| `shell [-e ESC] [-n] [-Tt] [-x] [CMD...]` | Remote shell (interactive if no command) |
| `emu COMMAND` | Emulator console command |
| `install [-lrtsdg] [--instant] APK` | Install a package |
| `install-multiple ... APK...` | Install split APKs |
| `install-multi-package ... APK...` | Install several packages atomically |
| `uninstall [-k] PACKAGE` | Remove a package; `-k` keeps data |
| `bugreport [PATH]` | Write a bug report (zip or directory) |
| `jdwp` | List PIDs hosting a JDWP transport |
| `logcat` | Device log (`logcat --help` for filters) |

`install` flags: `-r` replace, `-t` test packages, `-d` allow downgrade,
`-g` grant runtime permissions, `-p` partial install, `--abi ABI`,
`--instant`, `--no-streaming`/`--streaming`, and fast-deploy controls.

### Port forwarding

| Command | Meaning |
|---------|---------|
| `forward --list` | List host→device forwards |
| `forward [--no-rebind] LOCAL REMOTE` | Forward a host socket to the device |
| `forward --remove LOCAL` | Remove one forward |
| `forward --remove-all` | Remove all forwards |
| `reverse --list` | List device→host reverses |
| `reverse [--no-rebind] REMOTE LOCAL` | Forward a device socket to the host |
| `reverse --remove REMOTE` | Remove one reverse |
| `reverse --remove-all` | Remove all reverses |

Socket specifications: `tcp:<port>`, `localabstract:<name>`,
`localreserved:<name>`, `localfilesystem:<name>`, `dev:<node>`,
`jdwp:<pid>` (remote only), `vsock:<CID>:<port>` (remote only), and
`acceptfd:<fd>` (listen only).

### Bootloader, root, and security

| Command | Meaning |
|---------|---------|
| `reboot [bootloader\|recovery\|sideload\|sideload-auto-reboot]` | Reboot |
| `sideload OTAPACKAGE` | Sideload a full OTA package |
| `root` / `unroot` | Restart `adbd` with or without root |
| `usb` | Restart `adbd` on USB |
| `tcpip PORT` | Restart `adbd` on TCP |
| `remount [-R]` | Remount partitions read-write |
| `disable-verity` / `enable-verity` | Toggle dm-verity on userdebug builds |
| `keygen FILE` | Generate an adb public/private key pair |

### Server and USB control

| Command | Meaning |
|---------|---------|
| `start-server` | Start the server if needed |
| `kill-server` | Stop the server |
| `reconnect` | Reconnect host-side |
| `reconnect device` | Reconnect device-side |
| `reconnect offline` | Reset offline/unauthorised devices |
| `attach` / `detach` | Attach or detach a USB device |

## Environment variables

| Variable | Meaning |
|----------|---------|
| `ADB_TRACE` | Debug categories: `all`, `adb`, `sockets`, `packets`, `rwx`, `usb`, `sync`, `sysdeps`, `transport`, `jdwp` |
| `ADB_VENDOR_KEYS` | Colon-separated key files or directories |
| `ANDROID_SERIAL` | Default serial (same as `-s`) |
| `ANDROID_LOG_TAGS` | Default logcat tags |
| `ADB_LOCAL_TRANSPORT_MAX_PORT` | Emulator scan ceiling (default 5585) |
| `ADB_MDNS_AUTO_CONNECT` | mDNS services allowed to auto-connect |

## Recipes

**1. Confirm the server and list devices.**

```bash
adb start-server
adb devices -l
```

**2. Target one device explicitly.**

```bash
adb -s emulator-5554 shell getprop ro.build.version.release
```

**3. Install and launch an app.**

```bash
adb install -r -g app.apk
adb shell monkey -p com.example.app -c android.intent.category.LAUNCHER 1
```

**4. Pull an installed APK for static analysis.**

```bash
adb shell pm path com.example.app
adb pull /data/app/.../base.apk ./base.apk
```

**5. Capture logs with a tag filter.**

```bash
adb logcat -v threadtime -s ActivityManager:E AndroidRuntime:E
```

**6. Forward a host port to a device service.**

```bash
adb forward tcp:8080 tcp:8080
curl http://localhost:8080/
adb forward --remove tcp:8080
```

**7. Reverse-forward so the device reaches a host proxy.**

```bash
adb reverse tcp:8080 tcp:8080
adb shell settings put global http_proxy 127.0.0.1:8080
```

**8. Copy a directory with compression.**

```bash
adb push -z zstd ./corpus /sdcard/Download/corpus
```

**9. Wait for a device, then reboot to bootloader.**

```bash
adb wait-for-device
adb reboot bootloader
```

**10. Generate a dedicated key pair.**

```bash
adb keygen ~/.android/adbkey_audit
```

**11. Check mDNS discovery.**

```bash
adb mdns check
adb mdns services
```

**12. Reset a stuck device connection.**

```bash
adb reconnect offline
adb kill-server && adb start-server
```

## Exit status

| Situation | Exit |
|-----------|------|
| Command succeeded | 0 |
| No device / emulator | 1 |
| Unknown subcommand | 1 |
| `wait-for-*` with no matching device | blocks until timeout or device |
| `devices` with no devices | 0 (empty list) |
| `start-server` / `kill-server` | 0 |

`adb` reports protocol errors on stderr. A zero exit from `devices` does not
mean a device is attached; parse the list or use `get-state`.

## Quirks and limits

- **The server outlives the client.** It keeps running after the command; use
  `adb kill-server` when you need a clean state.
- **`devices` exits 0 even with no devices.** Check the output, not `$?`.
- **`wait-for-device` can hang forever.** Wrap it in `timeout`.
- **USB authorisation is per-host.** The device shows an RSA prompt; an
  unauthorised device appears as `unauthorized` in `devices`.
- **`root` only works on userdebug/eng builds.** Production builds reject it.
- **`remount` needs an unlocked or rooted device.** It fails on stock
  production builds.
- **`adb backup` is deprecated.** Modern apps can opt out, and the output is
  not a full filesystem image.
- **`install -g` grants only runtime permissions.** Special permissions still
  need `appops` or a system app.
- **`forward` and `reverse` are separate tables.** `forward --remove-all` does
  not clear reverses.
- **TCP mode is unauthenticated at the transport layer.** `adb tcpip` exposes
  `adbd` on the network; pair or tunnel it.
- **`-a` exposes the server to the network.** Do not use it on a shared host
  without a firewall.
- **`keygen` writes a private key with mode 0600.** Protect it like any other
  credential.

## Related tools

- [`fastboot`](../fastboot/) — bootloader flashing and unlocking.
- [`scrcpy`](../scrcpy/) — screen mirroring and control over adb.
- [`apktool`](../apktool/) — decode a pulled APK.
- [`ghidra`](../ghidra/) — analyse native libraries from an APK.
- [`exiftool`](../exiftool/) — inspect images pulled from the device.
- `frida`, `objection`, `jadx` — dynamic and static app analysis (not
  installed here).
