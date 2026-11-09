# fastboot — command scenarios for deep research

Installed build: fastboot 34.0.5-debian, binary at
`/usr/lib/android-sdk/platform-tools/fastboot`. Every option below appears in
[`help.txt`](help.txt); semantics come from the upstream `fastboot` manual and
observed behaviour.

> **Scope and risk:** `fastboot` writes raw images to named partitions and can
> unlock the bootloader. A wrong image, a wrong partition name, or `--force`
> can brick the device. Every scenario assumes a device you own, a known-good
> factory image, and a recovery path. Do not run flashing commands against
> hardware you do not control.

> **Blocking behaviour:** with no device attached, `fastboot devices` exits 0
> with empty output, but `getvar`, `flash`, `reboot`, and even an unknown
> command print `< waiting for any device >` and block. Always wrap them in
> `timeout` when scripting or researching.

## How to invoke

```text
@deep-research For an authorised Android device recovery with fastboot
34.0.5-debian, explain the difference between bootloader fastboot and
fastbootd, how `flash`, `flashall`, `update`, and `flash:raw` choose their
images, how A/B slots and `--slot`/`--set-active` interact, and which commands
need `fastbootd`. Show which commands block with no device attached, which exit
0 with an empty list, and how `-S` sparse splitting and `--fs-options` change
the write. Cite the installed help text, the upstream platform-tools docs, and
observed output. Separate confirmed behaviour from inference and state
confidence.
```

Create a working directory and record the environment:

```bash
mkdir -p ~/fastboot-demo/evidence
cd ~/fastboot-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
fastboot --version > evidence/version.txt
```

## Scenario 1 — version, help, binary, and provenance

| # | Command | Purpose |
|---|---------|---------|
| 1 | `fastboot --version` | Version and install path |
| 2 | `fastboot --help \| head -20` | Flashing commands |
| 3 | `fastboot --help \| grep -A12 'basics'` | Discovery and variables |
| 4 | `fastboot --help \| grep -A10 'locking'` | Lock/unlock commands |
| 5 | `fastboot --help \| grep -A20 'advanced'` | Slots, logical partitions |
| 6 | `fastboot --help \| grep -A20 'boot image'` | Boot image options |
| 7 | `fastboot --help \| grep -A20 'flash image'` | Sparse and filesystem options |
| 8 | `fastboot --help \| grep -A10 'Android Things'` | Staging commands |
| 9 | `fastboot -h \| head -3` | Short help flag |
| 10 | `command -v fastboot` | Resolve the binary |
| 11 | `readlink -f "$(command -v fastboot)"` | Real path in the SDK |
| 12 | `dpkg -s fastboot \| grep -E 'Version\|Depends'` | Package provenance |
| 13 | `file "$(readlink -f "$(command -v fastboot)")"` | Confirm a native ELF |
| 14 | `fastboot --help \| wc -l` | Help surface size |
| 15 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt` | Timestamp the run |

**Research questions**
- Which Debian package provides both `adb` and `fastboot`?
- Where does the SDK place the binary?
- Which help section documents `--slot`?
- Is `fastboot -h` identical to `--help`?

**Sources:** `fastboot --help`; `fastboot --version`; `dpkg -s fastboot`.
**Output:** a version record and a help-section map.

## Scenario 2 — discovery, selection, and exit codes

| # | Command | Purpose |
|---|---------|---------|
| 16 | `timeout 2 fastboot devices; echo "devices=$?"` | Short list |
| 17 | `timeout 2 fastboot devices -l; echo "devicesl=$?"` | List with device paths |
| 18 | `timeout 2 fastboot devices \| wc -l` | Count attached devices |
| 19 | `timeout 2 fastboot -s SERIAL devices; echo "serial=$?"` | Select by serial |
| 20 | `timeout 2 fastboot -s tcp:192.0.2.10:5555 devices; echo "tcp=$?"` | Select by TCP |
| 21 | `timeout 2 fastboot -s udp:192.0.2.10:5555 devices; echo "udp=$?"` | Select by UDP |
| 22 | `timeout 2 fastboot --unbuffered devices; echo "unbuffered=$?"` | Unbuffered output |
| 23 | `timeout 2 fastboot -v devices; echo "verbose=$?"` | Verbose output |
| 24 | `timeout 2 fastboot >/dev/null 2>&1; echo "noargs=$?"` | No command exits 1 |
| 25 | `timeout 2 fastboot frobnicate >/dev/null 2>&1; echo "badcmd=$?"` | Unknown command blocks |
| 26 | `timeout 2 fastboot getvar product; echo "getvar=$?"` | Device wait is 124 |
| 27 | `timeout 2 fastboot reboot; echo "reboot=$?"` | Reboot blocks too |
| 28 | `timeout 2 fastboot --slot nonsense devices; echo "badslot=$?"` | Bad slot is not validated |
| 29 | `timeout 2 fastboot --slot all devices; echo "allslot=$?"` | `all` is accepted |
| 30 | `timeout 2 fastboot --slot other devices; echo "otherslot=$?"` | `other` is accepted |
| 31 | `timeout 2 fastboot -S 1M devices; echo "sparse=$?"` | Sparse option with discovery |
| 32 | `timeout 2 fastboot --force devices; echo "force=$?"` | Force with discovery |
| 33 | `timeout 2 fastboot -w devices; echo "wipe=$?"` | Wipe flag is not validated early |

**Research questions**
- Does `devices` exit non-zero when nothing is attached?
- Why does a misspelled command block instead of failing?
- Which `--slot` values are accepted?
- Does `-s tcp:` need a reachable device before validation?

**Sources:** `fastboot --help`; observed exit codes.
**Output:** an exit-code and selection matrix.

## Scenario 3 — variables and device state

| # | Command | Purpose |
|---|---------|---------|
| 34 | `timeout 2 fastboot getvar product` | Product name |
| 35 | `timeout 2 fastboot getvar current-slot` | Active A/B slot |
| 36 | `timeout 2 fastboot getvar slot-count` | Number of slots |
| 37 | `timeout 2 fastboot getvar unlocked` | Lock state |
| 38 | `timeout 2 fastboot getvar secure` | Secure boot state |
| 39 | `timeout 2 fastboot getvar version-bootloader` | Bootloader version |
| 40 | `timeout 2 fastboot getvar version-baseband` | Baseband version |
| 41 | `timeout 2 fastboot getvar serialno` | Serial number |
| 42 | `timeout 2 fastboot getvar max-download-size` | Transfer chunk ceiling |
| 43 | `timeout 2 fastboot getvar all 2>&1 \| head -40` | Dump all variables |
| 44 | `timeout 2 fastboot getvar all 2>&1 \| wc -l` | Count variables |
| 45 | `timeout 2 fastboot getvar product 2>&1 \| grep -c '(bootloader)'` | Response prefix |
| 46 | `timeout 2 fastboot getvar nonexistent 2>&1 \| head -3` | Unknown variable |
| 47 | `timeout 2 fastboot flashing get_unlock_ability` | Unlock permission |
| 48 | `timeout 2 fastboot reboot-bootloader` | Reboot to bootloader |
| 49 | `timeout 2 fastboot reboot fastboot` | Reboot to fastbootd |
| 50 | `timeout 2 fastboot reboot recovery` | Reboot to recovery |
| 51 | `timeout 2 fastboot reboot` | Reboot to system |

**Research questions**
- Which variables are guaranteed on every bootloader?
- Does `getvar all` work when it is not in the help text?
- What does `flashing get_unlock_ability` return when unlocking is disabled?
- Does `reboot fastboot` require a device already in fastbootd?

**Sources:** `fastboot --help`; observed output.
**Output:** a variable table with example values.

## Scenario 4 — single-partition flashing

| # | Command | Purpose |
|---|---------|---------|
| 52 | `timeout 30 fastboot flash boot boot.img` | Flash boot |
| 53 | `timeout 30 fastboot flash --slot=all boot boot.img` | Flash both slots |
| 54 | `timeout 30 fastboot --slot=other flash boot boot.img` | Flash the inactive slot |
| 55 | `timeout 30 fastboot flash system system.img` | Flash system |
| 56 | `timeout 30 fastboot flash vendor vendor.img` | Flash vendor |
| 57 | `timeout 30 fastboot flash vbmeta vbmeta.img` | Flash vbmeta |
| 58 | `timeout 30 fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img` | Patch vbmeta flags |
| 59 | `timeout 30 fastboot flash dtbo dtbo.img` | Flash dtbo |
| 60 | `timeout 30 fastboot flash --force boot boot.img` | Force an unsafe write |
| 61 | `timeout 30 fastboot -S 256M flash system system.img` | Sparse split |
| 62 | `timeout 30 fastboot flash:raw boot kernel.img` | Build and flash a boot image |
| 63 | `timeout 30 fastboot flash:raw boot kernel.img ramdisk.img` | With ramdisk |
| 64 | `timeout 30 fastboot flash:raw boot kernel.img ramdisk.img second.img` | With second stage |
| 65 | `timeout 30 fastboot erase cache` | Erase a partition |
| 66 | `timeout 30 fastboot erase userdata` | Erase userdata |
| 67 | `timeout 30 fastboot format cache` | Format a partition |
| 68 | `timeout 30 fastboot format:ext4 userdata` | Format with a filesystem |
| 69 | `timeout 30 fastboot format:ext4:64M userdata` | Format with size |
| 70 | `timeout 30 fastboot --fs-options=casefold,projid format userdata` | Format features |
| 71 | `timeout 30 fastboot fetch boot /tmp/boot-dump.img` | Pull a partition image |

**Research questions**
- Does `flash --slot=all` write the same image to both slots?
- What does `--disable-verity` change in vbmeta?
- Is `flash:raw` still supported on modern boot header v4 devices?
- Does `format` need a filesystem type on a raw partition?

**Sources:** `fastboot --help`; upstream platform-tools docs; observed output.
**Output:** a flashing matrix with preconditions and warnings.

## Scenario 5 — bulk flashing and boot image options

| # | Command | Purpose |
|---|---------|---------|
| 72 | `export ANDROID_PRODUCT_OUT=/path/to/out` | Point at a product tree |
| 73 | `timeout 300 fastboot flashall` | Flash the whole product tree |
| 74 | `timeout 300 fastboot --skip-secondary flashall` | Keep the inactive slot |
| 75 | `timeout 300 fastboot --skip-reboot flashall` | Do not reboot afterwards |
| 76 | `timeout 300 fastboot --disable-super-optimization flashall` | Disable super optimisation |
| 77 | `timeout 300 fastboot --disable-fastboot-info flashall` | Use the image list |
| 78 | `timeout 300 fastboot -w flashall` | Wipe userdata first |
| 79 | `timeout 300 fastboot update update.zip` | Flash an update zip |
| 80 | `unzip -l update.zip \| head -20` | Inspect the zip first |
| 81 | `unzip -p update.zip android-info.txt` | Check the device match |
| 82 | `timeout 30 fastboot --base 0x10000000 boot kernel.img` | Kernel base |
| 83 | `timeout 30 fastboot --kernel-offset 0x8000 boot kernel.img` | Kernel offset |
| 84 | `timeout 30 fastboot --ramdisk-offset 0x1000000 boot kernel.img` | Ramdisk offset |
| 85 | `timeout 30 fastboot --tags-offset 0x100 boot kernel.img` | Tags offset |
| 86 | `timeout 30 fastboot --dtb dtb.img --dtb-offset 0x1100000 boot kernel.img` | DTB and offset |
| 87 | `timeout 30 fastboot --page-size 2048 boot kernel.img` | Page size |
| 88 | `timeout 30 fastboot --header-version 3 boot kernel.img` | Header version |
| 89 | `timeout 30 fastboot --os-version 14.0.0 boot kernel.img` | OS version |
| 90 | `timeout 30 fastboot --os-patch-level 2024-01-05 boot kernel.img` | Patch level |
| 91 | `timeout 30 fastboot --cmdline 'console=ttyMSM0' boot kernel.img` | Kernel cmdline |
| 92 | `timeout 30 fastboot boot kernel.img` | Boot without flashing |

**Research questions**
- Does `flashall` prefer `fastboot-info.txt` over the image list?
- Which boot-image options are ignored on header v4?
- Does `update` verify `android-info.txt` before writing?
- Can `boot` run without unlocking?

**Sources:** `fastboot --help`; observed output.
**Output:** a bulk-flash runbook and a boot-image option table.

## Scenario 6 — slots, logical partitions, and snapshots

| # | Command | Purpose |
|---|---------|---------|
| 93 | `timeout 2 fastboot getvar current-slot` | Current slot |
| 94 | `timeout 2 fastboot set_active a` | Activate slot a |
| 95 | `timeout 2 fastboot set_active b` | Activate slot b |
| 96 | `timeout 2 fastboot --set-active=a reboot` | Activate then reboot |
| 97 | `timeout 2 fastboot --set-active=b reboot bootloader` | Activate then boot bootloader |
| 98 | `timeout 2 fastboot --set-active reboot` | Activate the other slot |
| 99 | `timeout 2 fastboot create-logical-partition product_a 1073741824` | Create a logical partition |
| 100 | `timeout 2 fastboot delete-logical-partition product_a` | Delete a logical partition |
| 101 | `timeout 2 fastboot resize-logical-partition product_a 2147483648` | Resize a logical partition |
| 102 | `timeout 2 fastboot wipe-super` | Reset the super partition |
| 103 | `timeout 2 fastboot wipe-super super_empty.img` | Reset from an empty image |
| 104 | `timeout 2 fastboot snapshot-update cancel` | Cancel a snapshot merge |
| 105 | `timeout 2 fastboot snapshot-update merge` | Finish a snapshot merge |
| 106 | `timeout 2 fastboot reboot fastboot` | Enter fastbootd |
| 107 | `timeout 2 fastboot getvar is-userspace` | Confirm fastbootd |
| 108 | `timeout 2 fastboot getvar is-logical:product` | Logical-partition check |
| 109 | `timeout 2 fastboot getvar super-partition-name` | Super partition name |
| 110 | `timeout 2 fastboot getvar partition-size:system` | Partition size |

**Research questions**
- Which of these commands fail in bootloader fastboot?
- Does `set_active` validate the slot exists?
- Does `wipe-super` erase user data in logical partitions?
- How does `is-userspace` distinguish fastbootd from the bootloader?

**Sources:** `fastboot --help`; upstream docs; observed output.
**Output:** a slot/dynamic-partition state matrix.

## Scenario 7 — locking, unlocking, and staging

| # | Command | Purpose |
|---|---------|---------|
| 111 | `timeout 2 fastboot flashing get_unlock_ability` | Is unlocking allowed? |
| 112 | `timeout 2 fastboot flashing unlock` | Unlock (wipes data) |
| 113 | `timeout 2 fastboot flashing lock` | Lock again |
| 114 | `timeout 2 fastboot flashing unlock_critical` | Unlock critical partitions |
| 115 | `timeout 2 fastboot flashing lock_critical` | Lock critical partitions |
| 116 | `timeout 2 fastboot getvar unlocked` | Verify the lock state |
| 117 | `timeout 2 fastboot getvar secure` | Verify secure boot |
| 118 | `timeout 2 fastboot oem help` | OEM command surface |
| 119 | `timeout 2 fastboot oem device-info` | OEM device info |
| 120 | `timeout 2 fastboot oem unlock` | Legacy OEM unlock |
| 121 | `timeout 2 fastboot oem lock` | Legacy OEM lock |
| 122 | `timeout 2 fastboot stage staged.img` | Stage a file |
| 123 | `timeout 2 fastboot get_staged /tmp/staged.out` | Read staged data |
| 124 | `cmp -s staged.img /tmp/staged.out; echo "cmp=$?"` | Verify staging |
| 125 | `sha256sum staged.img /tmp/staged.out` | Compare hashes |
| 126 | `timeout 2 fastboot --version` | Version after state changes |
| 127 | `timeout 2 fastboot devices -l` | Confirm the device is still listed |
| 128 | `timeout 2 fastboot reboot` | Return to Android |

**Research questions**
- Does `flashing unlock` require an OEM toggle first?
- Is `get_unlock_ability` reliable across vendors?
- Does locking again restore verified boot?
- Is `stage`/`get_staged` Android Things only?

**Sources:** `fastboot --help`; observed output.
**Output:** a lock-state runbook with data-loss warnings.

## Scenario 8 — failure modes, transport, and cleanup

| # | Command | Purpose |
|---|---------|---------|
| 129 | `timeout 2 fastboot flash boot /nonexistent.img 2>&1 \| head -3` | Missing image |
| 130 | `timeout 2 fastboot flash nosuchpartition boot.img 2>&1 \| head -3` | Unknown partition |
| 131 | `timeout 2 fastboot -s does-not-exist devices` | Bad serial |
| 132 | `timeout 2 fastboot -s tcp:192.0.2.1:5555 devices` | Unreachable TCP device |
| 133 | `timeout 2 fastboot -s udp:192.0.2.1:5555 devices` | Unreachable UDP device |
| 134 | `timeout 2 fastboot getvar product 2>&1 \| head -3` | Device wait message |
| 135 | `timeout 2 fastboot --force flash boot boot.img 2>&1 \| head -3` | Force path |
| 136 | `timeout 2 fastboot -S notanumber flash boot boot.img 2>&1 \| head -3` | Bad sparse size |
| 137 | `timeout 2 fastboot --fs-options=bogus format userdata 2>&1 \| head -3` | Bad filesystem option |
| 138 | `timeout 2 fastboot --header-version 9 boot kernel.img 2>&1 \| head -3` | Bad header version |
| 139 | `timeout 2 fastboot --os-patch-level 2024-13-40 boot kernel.img 2>&1 \| head -3` | Bad date |
| 140 | `timeout 2 fastboot update not-a-zip.zip 2>&1 \| head -3` | Bad update zip |
| 141 | `lsusb \| grep -i -E 'google\|android\|fastboot'` | USB visibility |
| 142 | `ls -l /dev/bus/usb/*/* 2>/dev/null \| head -5` | USB permissions |
| 143 | `timeout 2 fastboot devices` | Final device check |
| 144 | `rm -rf ~/fastboot-demo` | Remove the demo tree |
| 145 | `test ! -e ~/fastboot-demo; echo "removed=$?"` | Verify cleanup |

**Research questions**
- Which errors are client-side and which need a device?
- Does a missing image fail before or after the device wait?
- How do USB permissions look when `fastboot devices` is empty?
- Does `timeout` leave the device in a partial state?

**Sources:** observed exit codes; `fastboot --help`.
**Output:** an error matrix and a clean workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `--version`, `--help`, `-h`, binary and package provenance | 1 |
| `devices`, `-l`, `-s`, `tcp:`, `udp:`, `--unbuffered`, `-v`, `--slot`, `-S`, `--force`, `-w` | 2 |
| `getvar`, `flashing get_unlock_ability`, `reboot`, `reboot-bootloader`, `reboot fastboot` | 3 |
| `flash`, `--slot=all`, `--disable-verity`, `--disable-verification`, `flash:raw`, `erase`, `format`, `--fs-options`, `fetch` | 4 |
| `flashall`, `update`, `--skip-secondary`, `--skip-reboot`, `--base`, `--kernel-offset`, `--ramdisk-offset`, `--tags-offset`, `--dtb`, `--page-size`, `--header-version`, `--os-version`, `--os-patch-level`, `--cmdline`, `boot` | 5 |
| `set_active`, `--set-active`, logical-partition commands, `wipe-super`, `snapshot-update` | 6 |
| `flashing lock/unlock`, `lock_critical`, `unlock_critical`, `oem`, `stage`, `get_staged` | 7 |
| error paths, transport, USB permissions, cleanup | 8 |

Full text: [`help.txt`](help.txt); authoritative description in the upstream
platform-tools documentation.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, help, binary, and provenance | 15 |
| 2 | Discovery, selection, and exit codes | 18 |
| 3 | Variables and device state | 18 |
| 4 | Single-partition flashing | 20 |
| 5 | Bulk flashing and boot image options | 21 |
| 6 | Slots, logical partitions, and snapshots | 18 |
| 7 | Locking, unlocking, and staging | 18 |
| 8 | Failure modes, transport, and cleanup | 17 |
| **Total** | | **145** |

## Legal and safety

`fastboot` can unlock verified boot, wipe userdata, overwrite the bootloader,
and permanently brick a device. Only use it on hardware you own or are
explicitly authorised to service. `flashing unlock`, `-w`, `erase userdata`,
`wipe-super`, and `--force` are destructive — confirm the recovery path and
back up first. Unlocking may void warranties and disable device encryption
guarantees. `fastboot` over TCP/UDP is unauthenticated; keep it on a trusted
link. Delete the demo tree and any fetched partition images when finished.
