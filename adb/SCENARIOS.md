# adb — command scenarios for deep research

Installed build: Android Debug Bridge 1.0.41 (`34.0.5-debian`), binary at
`/usr/lib/android-sdk/platform-tools/adb`. Every option below appears in
[`help.txt`](help.txt); semantics come from the upstream `adb.1.md` manual and
observed behaviour.

> **Scope:** the recipes assume a device or emulator you own or are authorised
> to test. Many commands (`root`, `remount`, `install`, `pull`) change or read
> user data. Do not run them against a device you do not control.

> **No-device behaviour:** `adb devices` exits 0 with an empty list;
> `get-state`, `shell`, and most device commands exit 1. `wait-for-device`
> blocks, so wrap it in `timeout`.

## How to invoke

```text
@deep-research For an authorised Android assessment with adb 1.0.41, explain
how the client/server model, device selection (`-d`, `-e`, `-s`, `-t`,
`ANDROID_SERIAL`), `wait-for-*`, `forward`/`reverse`, and `install` flags
interact. Show which commands return exit 1 with no device, how `root`,
`remount`, `disable-verity`, and `tcpip` change device state, and how
`push`/`pull` compression and `--sync` differ. Cite the installed help text,
the upstream `adb.1.md`, and observed output. Separate confirmed behaviour
from inference and state confidence.
```

Create a working directory and record the environment:

```bash
mkdir -p ~/adb-demo/evidence
cd ~/adb-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
adb version > evidence/version.txt
```

## Scenario 1 — version, help, binary, server lifecycle

| # | Command | Purpose |
|---|---------|---------|
| 1 | `adb version` | Client and server version |
| 2 | `adb --help \| head -20` | Global options |
| 3 | `adb help \| head -5` | `help` is an alias for `--help` |
| 4 | `adb --help \| grep -A10 'general commands'` | Discovery commands |
| 5 | `adb --help \| grep -A12 'networking'` | Networking commands |
| 6 | `adb --help \| grep -A12 'file transfer'` | Transfer commands |
| 7 | `adb --help \| grep -A10 'scripting'` | Wait/state commands |
| 8 | `command -v adb` | Resolve the binary |
| 9 | `readlink -f "$(command -v adb)"` | Real path in the SDK |
| 10 | `dpkg -s adb \| grep -E 'Version\|Depends'` | Package provenance |
| 11 | `file "$(readlink -f "$(command -v adb)")"` | Confirm a native ELF |
| 12 | `adb start-server; echo "start=$?"` | Start the server |
| 13 | `pgrep -af 'adb.*server' \| head -2` | Confirm the server process |
| 14 | `adb kill-server; echo "kill=$?"` | Stop the server |
| 15 | `pgrep -af 'adb.*server' \| head -2` | Confirm it exited |
| 16 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt` | Timestamp the run |

**Research questions**
- Which process listens on port 5037?
- Does `kill-server` affect a device session?
- Where is the Debian binary installed?
- Which help section lists `wait-for`?

**Sources:** `adb --help`; `adb version`; `dpkg -s adb`; `pgrep`.
**Output:** a version record and a server lifecycle trace.

## Scenario 2 — device discovery and state

| # | Command | Purpose |
|---|---------|---------|
| 17 | `adb devices` | Short device list |
| 18 | `adb devices -l` | Long list with model/product/transport |
| 19 | `adb devices -l \| tail -n +2` | Strip the header |
| 20 | `adb get-state; echo "state=$?"` | Device state, exit 1 if absent |
| 21 | `adb get-serialno; echo "serial=$?"` | Serial, exit 1 if absent |
| 22 | `adb get-devpath; echo "devpath=$?"` | USB device path |
| 23 | `timeout 2 adb wait-for-device; echo "wait=$?"` | Block until a device appears |
| 24 | `timeout 2 adb wait-for-usb-device; echo "waitusb=$?"` | USB transport only |
| 25 | `timeout 2 adb wait-for-local-device; echo "waitlocal=$?"` | Local transport only |
| 26 | `timeout 2 adb wait-for-disconnect; echo "waitdisc=$?"` | Wait for a disconnect |
| 27 | `adb -d devices 2>&1 \| head -3` | USB selection |
| 28 | `adb -e devices 2>&1 \| head -3` | TCP selection |
| 29 | `adb -s emulator-5554 devices 2>&1 \| head -3` | Serial selection |
| 30 | `adb -t 1 devices 2>&1 \| head -3` | Transport id selection |
| 31 | `ANDROID_SERIAL=emulator-5554 adb devices 2>&1 \| head -3` | Environment selection |
| 32 | `adb -H 127.0.0.1 -P 5037 devices` | Explicit server endpoint |
| 33 | `adb mdns check; echo "mdns=$?"` | mDNS availability |
| 34 | `adb mdns services; echo "services=$?"` | Discovered services |

**Research questions**
- Which selection option wins when several are given?
- Does `devices` exit non-zero with no device?
- Which wait states are accepted?
- Does mDNS require the server to be running?

**Sources:** `adb --help`; upstream `adb.1.md`; observed output.
**Output:** a selection matrix and an exit-code table.

## Scenario 3 — shell, install, and app inspection

| # | Command | Purpose |
|---|---------|---------|
| 35 | `adb shell getprop ro.build.version.release` | Android version |
| 36 | `adb shell getprop ro.product.model` | Device model |
| 37 | `adb shell id` | Remote uid |
| 38 | `adb shell pm list packages \| head -10` | Installed packages |
| 39 | `adb shell pm path com.android.settings` | APK path for a package |
| 40 | `adb shell dumpsys package com.android.settings \| head -20` | Package details |
| 41 | `adb shell cmd package help \| head -10` | Package-manager commands |
| 42 | `adb shell settings list global \| head -10` | Global settings |
| 43 | `adb shell -T getprop ro.build.type` | Disable pty allocation |
| 44 | `adb shell -t getprop ro.build.type` | Allocate a pty |
| 45 | `adb shell -n getprop ro.build.type` | Do not read stdin |
| 46 | `adb shell -x false; echo "x=$?"` | Disable exit-code propagation |
| 47 | `adb install --help 2>&1 \| head -10` | Install flags |
| 48 | `adb install -r -g app.apk` | Replace and grant permissions |
| 49 | `adb install -t app-debug.apk` | Allow a test package |
| 50 | `adb install -d app.apk` | Allow a version downgrade |
| 51 | `adb install-multiple base.apk split_config.arm64_v8a.apk` | Split APK |
| 52 | `adb uninstall com.example.app` | Remove an app |
| 53 | `adb uninstall -k com.example.app` | Remove but keep data |
| 54 | `adb jdwp` | PIDs exposing JDWP |
| 55 | `adb logcat -d -v threadtime \| tail -20` | Dump the log buffer |

**Research questions**
- Does `shell -x` change `$?` for a failing remote command?
- Which install flags survive a reinstall?
- Does `pm path` require root?
- Is `logcat -d` a snapshot or a follow?

**Sources:** `adb --help`; `adb shell pm help`; observed output.
**Output:** an install/inspection transcript with exit codes.

## Scenario 4 — file transfer and sync

| # | Command | Purpose |
|---|---------|---------|
| 56 | `printf 'hello\n' > local.txt` | Create a test file |
| 57 | `adb push local.txt /sdcard/Download/local.txt` | Plain push |
| 58 | `adb push -z zstd local.txt /sdcard/Download/local.txt` | Push with zstd |
| 59 | `adb push -z lz4 local.txt /sdcard/Download/local.txt` | Push with lz4 |
| 60 | `adb push -Z local.txt /sdcard/Download/local.txt` | Push without compression |
| 61 | `adb push -n local.txt /sdcard/Download/local.txt` | Dry run |
| 62 | `adb push --sync local.txt /sdcard/Download/local.txt` | Only if newer |
| 63 | `adb pull /sdcard/Download/local.txt ./pulled.txt` | Plain pull |
| 64 | `adb pull -a /sdcard/Download/local.txt ./pulled-a.txt` | Preserve mode/time |
| 65 | `adb pull -z zstd /sdcard/Download/local.txt ./pulled-z.txt` | Pull with zstd |
| 66 | `cmp -s local.txt pulled.txt; echo "cmp=$?"` | Verify content |
| 67 | `stat -c '%a %y' pulled-a.txt` | Verify preserved metadata |
| 68 | `adb sync -l all` | List files that would sync |
| 69 | `adb sync -n all` | Dry-run sync |
| 70 | `adb sync -z zstd data` | Sync one partition |
| 71 | `adb sync system` | Sync the system partition |
| 72 | `adb shell ls -l /sdcard/Download/local.txt` | Confirm on device |
| 73 | `adb shell rm /sdcard/Download/local.txt` | Clean up on device |

**Research questions**
- Which compression algorithm is fastest for small files?
- Does `--sync` compare mtime, size, or both?
- Does `pull -a` preserve SELinux context?
- What does `sync` read from `$ANDROID_PRODUCT_OUT`?

**Sources:** `adb --help`; observed output.
**Output:** a transfer/compression matrix with hashes.

## Scenario 5 — forwarding and reverse forwarding

| # | Command | Purpose |
|---|---------|---------|
| 74 | `adb forward --list` | Baseline host→device forwards |
| 75 | `adb forward tcp:8080 tcp:8080` | Forward a TCP port |
| 76 | `adb forward --list` | Confirm the entry |
| 77 | `adb forward --no-rebind tcp:8080 tcp:8080` | Fail if already bound |
| 78 | `adb forward tcp:0 tcp:8080` | Pick any free host port |
| 79 | `adb forward tcp:9000 localabstract:scrcpy` | Forward to an abstract socket |
| 80 | `adb forward tcp:9001 localfilesystem:/data/local/tmp/sock` | Forward to a filesystem socket |
| 81 | `adb forward tcp:9002 dev:/dev/input/event0` | Forward to a character device |
| 82 | `adb forward --remove tcp:8080` | Remove one forward |
| 83 | `adb forward --remove-all` | Remove all forwards |
| 84 | `adb forward --list` | Confirm the table is empty |
| 85 | `adb reverse --list` | Baseline device→host reverses |
| 86 | `adb reverse tcp:8080 tcp:8080` | Device reaches host port |
| 87 | `adb reverse --no-rebind tcp:8080 tcp:8080` | Fail if already bound |
| 88 | `adb reverse tcp:0 tcp:8080` | Pick any free device port |
| 89 | `adb reverse --remove tcp:8080` | Remove one reverse |
| 90 | `adb reverse --remove-all` | Remove all reverses |
| 91 | `adb reverse --list` | Confirm the table is empty |
| 92 | `adb forward --remove tcp:9999 2>&1 \| head -3` | Removing an absent entry |

**Research questions**
- Are forwards and reverses stored in the same table?
- Does `tcp:0` print the chosen port?
- Which socket types are host-only and which are device-only?
- Does removing an absent entry exit non-zero?

**Sources:** `adb --help`; observed output.
**Output:** a forward/reverse table with socket types.

## Scenario 6 — reboot, root, verity, and network mode

| # | Command | Purpose |
|---|---------|---------|
| 93 | `adb reboot --help 2>&1 \| head -5` | Reboot targets |
| 94 | `adb reboot` | Reboot to system |
| 95 | `adb reboot bootloader` | Reboot to bootloader |
| 96 | `adb reboot recovery` | Reboot to recovery |
| 97 | `adb reboot sideload` | Reboot to sideload mode |
| 98 | `adb reboot sideload-auto-reboot` | Sideload then reboot |
| 99 | `adb root; echo "root=$?"` | Restart adbd as root |
| 100 | `adb unroot; echo "unroot=$?"` | Restart adbd as shell |
| 101 | `adb remount; echo "remount=$?"` | Remount read-write |
| 102 | `adb remount -R; echo "remountR=$?"` | Remount and reboot if needed |
| 103 | `adb disable-verity; echo "dv=$?"` | Disable dm-verity |
| 104 | `adb enable-verity; echo "ev=$?"` | Re-enable dm-verity |
| 105 | `adb tcpip 5555; echo "tcpip=$?"` | Restart adbd on TCP |
| 106 | `adb usb; echo "usb=$?"` | Restart adbd on USB |
| 107 | `adb connect 192.0.2.10:5555` | Connect over TCP |
| 108 | `adb disconnect 192.0.2.10:5555` | Disconnect one device |
| 109 | `adb disconnect` | Disconnect all TCP devices |
| 110 | `adb pair 192.0.2.10:37000 123456` | Pair for secure TCP |
| 111 | `adb sideload ota.zip` | Sideload an OTA |

**Research questions**
- Which of these commands require an unlocked bootloader?
- Does `root` persist across a reboot?
- Does `tcpip` open an authenticated or unauthenticated port?
- What is the default pairing port?

**Sources:** `adb --help`; upstream manual; observed output.
**Output:** a state-change matrix with preconditions.

## Scenario 7 — keys, mDNS, server internals, USB control

| # | Command | Purpose |
|---|---------|---------|
| 112 | `mkdir -p ~/.android && adb keygen ~/.android/adbkey_audit` | Generate a key pair |
| 113 | `ls -l ~/.android/adbkey_audit*` | Check permissions |
| 114 | `stat -c '%a %n' ~/.android/adbkey_audit*` | Private key mode |
| 115 | `ADB_VENDOR_KEYS=~/.android/adbkey_audit adb devices` | Use a specific key |
| 116 | `adb mdns check` | mDNS availability |
| 117 | `adb mdns services` | Discovered services |
| 118 | `ADB_MDNS_AUTO_CONNECT=adb-tls-connect adb devices` | Auto-connect policy |
| 119 | `adb start-server` | Ensure the server runs |
| 120 | `adb -L tcp:localhost:5038 start-server` | Alternate listen socket |
| 121 | `adb -P 5038 devices` | Talk to the alternate server |
| 122 | `adb -H 127.0.0.1 -P 5038 kill-server` | Stop the alternate server |
| 123 | `adb reconnect` | Host-side reconnect |
| 124 | `adb reconnect device` | Device-side reconnect |
| 125 | `adb reconnect offline` | Reset offline devices |
| 126 | `adb attach; echo "attach=$?"` | Attach a detached USB device |
| 127 | `adb detach; echo "detach=$?"` | Detach from USB |
| 128 | `ADB_TRACE=adb adb devices 2>&1 \| head -10` | Debug the client |
| 129 | `ADB_TRACE=usb,transport adb devices 2>&1 \| head -10` | Debug USB/transport |
| 130 | `ADB_LOCAL_TRANSPORT_MAX_PORT=5590 adb devices` | Change emulator scan ceiling |

**Research questions**
- Which key does adb use if `ADB_VENDOR_KEYS` is unset?
- Does `-L` change the client's default port?
- What does `reconnect offline` do to an unauthorised device?
- Which trace categories are most useful for a failed connection?

**Sources:** `adb --help`; observed output.
**Output:** a key/permissions record and a server-endpoint matrix.

## Scenario 8 — failure modes, environment, cleanup

| # | Command | Purpose |
|---|---------|---------|
| 131 | `adb frobnicate >/dev/null 2>&1; echo "badcmd=$?"` | Unknown subcommand exits 1 |
| 132 | `adb shell echo hi; echo "shell=$?"` | No device exits 1 |
| 133 | `adb get-state; echo "state=$?"` | No device exits 1 |
| 134 | `adb install /nonexistent.apk 2>&1 \| head -3` | Missing APK |
| 135 | `adb push /nonexistent /sdcard/x 2>&1 \| head -3` | Missing local file |
| 136 | `adb pull /nonexistent ./x 2>&1 \| head -3` | Missing remote file |
| 137 | `adb forward tcp:8080 tcp:8080 2>&1 \| head -3` | Forward with no device |
| 138 | `adb -s does-not-exist get-state 2>&1 \| head -3` | Bad serial |
| 139 | `adb -P 59999 devices 2>&1 \| head -3` | Wrong server port |
| 140 | `adb -H 192.0.2.1 devices 2>&1 \| head -3` | Unreachable server host |
| 141 | `timeout 2 adb wait-for-device; echo "timeout=$?"` | Wait timeout is 124 |
| 142 | `adb kill-server; echo "kill=$?"` | Clean shutdown |
| 143 | `rm -rf ~/adb-demo` | Remove the demo tree |
| 144 | `test ! -e ~/adb-demo; echo "removed=$?"` | Verify cleanup |

**Research questions**
- Which errors are client-side and which come from the server?
- Does `install` clean up a partially pushed APK?
- Does `-P` fail fast when nothing listens?
- What evidence proves the server was stopped?

**Sources:** observed exit codes; `adb --help`.
**Output:** an error matrix and a clean workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `version`, `--help`, `help`, `start-server`, `kill-server` | 1 |
| `devices`, `get-state`, `get-serialno`, `get-devpath`, `wait-for-*`, `-d`, `-e`, `-s`, `-t`, `-H`, `-P`, `mdns` | 2 |
| `shell`, `install`, `install-multiple`, `uninstall`, `jdwp`, `logcat`, `pm`, `dumpsys`, `settings` | 3 |
| `push`, `pull`, `sync`, `-z`, `-Z`, `-n`, `--sync`, `-a` | 4 |
| `forward`, `reverse`, `--list`, `--remove`, `--remove-all`, `--no-rebind`, `tcp:0`, socket types | 5 |
| `reboot`, `root`, `unroot`, `remount`, `disable-verity`, `enable-verity`, `tcpip`, `usb`, `connect`, `disconnect`, `pair`, `sideload` | 6 |
| `keygen`, `mdns`, `-L`, `reconnect`, `attach`, `detach`, `ADB_TRACE`, `ADB_VENDOR_KEYS` | 7 |
| error paths and cleanup | 8 |

Full text: [`help.txt`](help.txt); authoritative description in the upstream
`adb.1.md`.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, help, binary, server lifecycle | 16 |
| 2 | Device discovery and state | 18 |
| 3 | Shell, install, and app inspection | 21 |
| 4 | File transfer and sync | 18 |
| 5 | Forwarding and reverse forwarding | 19 |
| 6 | Reboot, root, verity, and network mode | 19 |
| 7 | Keys, mDNS, server internals, USB control | 19 |
| 8 | Failure modes, environment, cleanup | 14 |
| **Total** | | **144** |

## Legal and safety

`adb` can install software, read logs, pull private data, change system
settings, and reboot a device. Only target devices you own or are explicitly
authorised to test. `adb root`, `remount`, `disable-verity`, and `sideload`
can brick a device or weaken its verified boot; confirm the recovery path
first. `adb tcpip` and `-a` expose a debug interface on the network — pair it,
tunnel it, or keep it on a trusted USB link. Delete the demo tree and any
pulled user data when finished.
