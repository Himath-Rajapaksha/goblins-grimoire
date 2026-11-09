# fastboot — Android bootloader flash tool

- **Version:** 34.0.5-debian (Android SDK platform-tools)
- **Binary:** `/usr/bin/fastboot` → `/usr/lib/android-sdk/platform-tools/fastboot`
- **Help snapshot:** [`help.txt`](help.txt) — `fastboot --help`
- **Upstream:** https://android.googlesource.com/platform/packages/modules/adb/
- **Licence:** Apache-2.0

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

`fastboot` talks to a device's bootloader over USB (or TCP) and writes
partitions, changes the active slot, unlocks the bootloader, and reboots.
It is the recovery path when Android will not boot, and the tool that turns a
factory image into a working device.

It is **not** a general file-transfer tool. It writes raw images to named
partitions; a wrong partition name or image can brick the device. Every
recipe here assumes an unlocked bootloader, a known-good factory image, and a
device you own.

## Device states

| State | What it means |
|-------|---------------|
| `fastboot` / bootloader | Device is in the bootloader; `fastboot devices` lists it |
| `fastbootd` | Userspace fastboot in recovery; needed for dynamic partitions |
| Android | Normal boot; use `adb reboot bootloader` to reach fastboot |

`fastboot devices` only lists devices already in a fastboot state. It does not
switch a booted device into the bootloader.

## Command groups

### Discovery and variables

| Command | Meaning |
|---------|---------|
| `devices [-l]` | List fastboot devices; `-l` adds device paths |
| `getvar NAME` | Read a bootloader variable |
| `reboot [bootloader]` | Reboot to system or back to the bootloader |

`getvar all` dumps every variable on most bootloaders, but it is not listed in
the help text.

### Flashing

| Command | Meaning |
|---------|---------|
| `update ZIP` | Flash all partitions from an `update.zip` |
| `flashall` | Flash from `$ANDROID_PRODUCT_OUT` |
| `flash PARTITION [FILE]` | Flash one partition; file defaults to the product out |
| `flash:raw PARTITION KERNEL [RAMDISK [SECOND]]` | Build and flash a boot image |
| `boot KERNEL [RAMDISK [SECOND]]` | Download and boot a kernel from RAM |
| `erase PARTITION` | Erase a partition |
| `format[:FS_TYPE[:SIZE]] PARTITION` | Format a partition |
| `fetch PARTITION OUT_FILE` | Pull a partition image from the device |

`flashall` sets the flashed slot active on A/B devices and may write secondary
images to the inactive slot. `update` expects a zip with an `android-info.txt`
and images.

### Slots and dynamic partitions

| Command | Meaning |
|---------|---------|
| `set_active SLOT` | Set the active A/B slot |
| `--slot SLOT` | Use `SLOT`, `all`, or `other` |
| `--set-active[=SLOT]` | Set the slot before rebooting |
| `create-logical-partition NAME SIZE` | Create a logical partition |
| `delete-logical-partition NAME` | Delete a logical partition |
| `resize-logical-partition NAME SIZE` | Resize a logical partition |
| `wipe-super [SUPER_EMPTY]` | Reset the super partition |
| `snapshot-update cancel` | Cancel an in-progress snapshot update |
| `snapshot-update merge` | Finish a merging snapshot update |

Logical-partition and snapshot commands generally require `fastbootd`.

### Locking and unlocking

| Command | Meaning |
|---------|---------|
| `flashing lock` | Lock the bootloader |
| `flashing unlock` | Unlock the bootloader (wipes user data) |
| `flashing lock_critical` | Lock critical partitions |
| `flashing unlock_critical` | Unlock critical partitions |
| `flashing get_unlock_ability` | Report whether unlocking is allowed |

Unlocking is device-specific and may require an OEM unlock toggle and a
vendor code. `flashing unlock` is destructive.

### Boot image options

| Option | Meaning |
|--------|---------|
| `--dtb DTB` | Device tree blob for header v2 |
| `--cmdline CMDLINE` | Kernel command line |
| `--base ADDRESS` | Kernel base (default `0x10000000`) |
| `--kernel-offset` | Kernel offset (default `0x00008000`) |
| `--ramdisk-offset` | Ramdisk offset (default `0x01000000`) |
| `--tags-offset` | Tags offset (default `0x00000100`) |
| `--dtb-offset` | DTB offset (default `0x01100000`) |
| `--page-size BYTES` | Flash page size (default 2048) |
| `--header-version VERSION` | Boot image header version |
| `--os-version MAJOR[.MINOR[.PATCH]]` | OS version |
| `--os-patch-level YYYY-MM-DD` | Security patch level |

### Global options

| Option | Meaning |
|--------|---------|
| `-w` | Wipe userdata |
| `-s SERIAL` | Target a USB device |
| `-s tcp\|udp:HOST[:PORT]` | Target a network device |
| `-S SIZE[K\|M\|G]` | Split into sparse files of at most SIZE |
| `--force` | Force an operation that may be unsafe |
| `--slot SLOT` | Slot selection |
| `--skip-secondary` | Do not flash secondary slots |
| `--skip-reboot` | Do not reboot after flashing |
| `--disable-verity` | Set disable-verity when flashing vbmeta |
| `--disable-verification` | Set disable-verification when flashing vbmeta |
| `--disable-super-optimization` | Disable super-partition optimisations |
| `--disable-fastboot-info` | Use the image list instead of `fastboot-info.txt` |
| `--fs-options=OPT[,OPT]` | Filesystem features: `casefold`, `projid`, `compress` |
| `--unbuffered` | Do not buffer input/output |
| `-v`, `--verbose` | Verbose output |
| `--version` | Version |
| `-h`, `--help` | Help |

### Android Things

| Command | Meaning |
|---------|---------|
| `stage IN_FILE` | Stage a file for the next command |
| `get_staged OUT_FILE` | Write the staged data to a file |

## Recipes

**1. Confirm a device is in fastboot.**

```bash
fastboot devices -l
```

**2. Read the product and current slot.**

```bash
fastboot getvar product
fastboot getvar current-slot
fastboot getvar all 2>&1 | head -40
```

**3. Flash one partition from a factory image.**

```bash
fastboot flash boot boot.img
```

**4. Flash every partition from a product-out tree.**

```bash
ANDROID_PRODUCT_OUT=/path/to/out fastboot flashall
```

**5. Flash an OTA-style update zip.**

```bash
fastboot update update.zip
```

**6. Boot a kernel without flashing it.**

```bash
fastboot boot kernel.img
```

**7. Switch the active slot.**

```bash
fastboot --set-active=b
fastboot reboot
```

**8. Wipe userdata and reboot.**

```bash
fastboot -w reboot
```

**9. Unlock, then check the unlock ability.**

```bash
fastboot flashing get_unlock_ability
fastboot flashing unlock
```

**10. Split a large image into sparse files.**

```bash
fastboot -S 256M flash system system.img
```

**11. Create and resize a logical partition.**

```bash
fastboot create-logical-partition product_a 1073741824
fastboot resize-logical-partition product_a 2147483648
```

**12. Fetch a partition image for analysis.**

```bash
fastboot fetch boot /tmp/boot.img
```

## Exit status

| Situation | Exit |
|-----------|------|
| Command succeeded | 0 |
| No command given | 1 |
| `devices` with no devices | 0 (empty output) |
| Command needing a device, none attached | blocks (`< waiting for any device >`) |
| Unknown command, no device | blocks before validation |
| `--slot nonsense` with no device | 0 |

**The blocking behaviour is the important one.** `fastboot getvar`, `flash`,
`reboot`, and even a misspelled command wait for a device instead of failing.
Wrap them in `timeout` when scripting.

## Quirks and limits

- **Most commands block until a device appears.** Use `timeout` in scripts.
- **`devices` exits 0 even with no devices.** Parse the output.
- **A misspelled command blocks too.** Validation happens after the device
  wait, so a typo looks like a hang.
- **`flashing unlock` wipes user data.** Back up first; some devices also
  require an OEM unlock toggle and a network check.
- **`-w` is destructive.** It wipes userdata as part of the command.
- **Slot names are device-specific.** `a`/`b` are common; some devices use
  `_a`/`_b` suffixes on partition names.
- **Dynamic partition commands need `fastbootd`.** Classic bootloader
  fastboot cannot create logical partitions.
- **`flashall` depends on `$ANDROID_PRODUCT_OUT`.** Without it, the command
  has nothing to flash.
- **Sparse images are not universal.** `-S` splits the image; the bootloader
  must understand the sparse format.
- **`--force` can brick a device.** It bypasses safety checks.
- **`getvar all` is not in the help text.** It works on most bootloaders but
  is not guaranteed.
- **USB access needs udev rules or root.** A permission error looks like “no
  devices”.
- **Fastboot over TCP is not encrypted.** Use it only on a trusted link.

## Related tools

- [`adb`](../adb/) — Android Debug Bridge; reboot into the bootloader.
- [`scrcpy`](../scrcpy/) — screen mirroring over adb.
- [`apktool`](../apktool/) — decode APKs pulled from the device.
- [`binwalk`](../binwalk/) — inspect boot images and firmware.
- [`ghidra`](../ghidra/) — analyse native binaries from a dumped partition.
- `payload-dumper-go`, `avbtool` — OTA payload and vbmeta tooling (not
  installed here).
