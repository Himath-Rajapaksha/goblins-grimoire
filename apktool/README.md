# apktool — Android APK decoding and rebuilding

- **Version:** 2.7.0-dirty (smali/baksmali 2.5.2.git2771eae-debian)
- **Binary:** `/usr/bin/apktool` (wrapper for `/usr/share/apktool/apktool-2.7.0.jar`)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/iBotPeaches/Apktool
- **Docs:** https://ibotpeaches.github.io/Apktool/ (linked from the installed help)

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Apktool is a reverse-engineering tool for Android APK files. It decodes an
APK's compiled resources (`resources.arsc`, binary XML such as
`AndroidManifest.xml`) back into readable XML and decodes DEX bytecode into
smali assembly. The result is a directory tree that can be inspected, patched,
and rebuilt into a new APK.

It also installs "framework" resource packages. Stock Android apps decode with
the built-in framework, but OEM or modified ROM apps often reference resources
that only exist in a vendor `framework-res.apk`; installing that framework lets
apktool resolve those resource IDs.

This is a **local, static** analysis tool. It does not install, run, or
communicate with the app under test.

## Use cases

| Situation | Why apktool |
|-----------|-------------|
| Triage an untrusted APK | Read the manifest, permissions, services, and receivers without running it |
| Inspect native/resource-only changes | Decode resources while skipping smali with `-s` |
| Review DEX logic | Produce smali trees that can be grepped, diffed, or patched |
| Compare two builds | Decode both and diff the smali/resource trees |
| Patch an app for lab testing | Edit smali/resources, rebuild, then sign separately |
| Resolve OEM resource IDs | Install the vendor framework and decode with `-t` |

## Requirements & install layout

- **Java runtime required.** The wrapper runs
  `java -Xmx512M -Dfile.encoding=utf-8 -cp "/usr/share/apktool/*" brut.apktool.Main`.
- Debian package layout: wrapper `/usr/bin/apktool`, JARs under
  `/usr/share/apktool/`, including `apktool-2.7.0.jar`, `smali.jar`, and
  `baksmali.jar`.
- Framework directory: the wrapper creates `~/.local/share/apktool/framework/`
  and seeds `1.apk` from
  `/usr/share/android-framework-res/framework-res.apk` on first run.
- No root is needed for normal decode/build work. Write output inside your home
  directory or a case directory.
- Decoding large APKs can be memory-hungry; the wrapper fixes the Java heap at
  512 MB. Very large apps may need a manual `java -Xmx...` invocation of the
  JAR.

## Basic syntax

```bash
apktool                      # print the built-in usage banner
apktool --version            # print the version and exit
apktool d target.apk         # decode APK into ./target
apktool b target/            # rebuild decoded tree into dist/
apktool if framework-res.apk # install a framework resource package
```

Note: the installed build prints `Unrecognized option: --help` and then the
usage banner. Use `apktool` with no arguments, not `apktool --help`.

## Command reference

The captured help snapshot is short and covers only the four entry points and
their options. Flags not shown below are not documented here; consult the
upstream 2.7.0 documentation before relying on them.

### Top level

| Flag | Meaning |
|------|---------|
| `-advance, --advanced` | Print advanced information |
| `-version, --version` | Print the version, then exit |

### `if` / `install-framework`

Usage: `apktool if|install-framework [options] <framework.apk>`

| Flag | Meaning |
|------|---------|
| `-p, --frame-path <dir>` | Store framework files in `<dir>` |
| `-t, --tag <tag>` | Tag the installed framework with `<tag>` |

### `d` / `decode`

Usage: `apktool d[ecode] [options] <file_apk>`

| Flag | Meaning |
|------|---------|
| `-f, --force` | Force-delete the destination directory before decoding |
| `-o, --output <dir>` | Output folder; default is `apk.out` |
| `-p, --frame-path <dir>` | Use framework files located in `<dir>` |
| `-r, --no-res` | Do not decode resources |
| `-s, --no-src` | Do not decode sources |
| `-t, --frame-tag <tag>` | Use framework files tagged with `<tag>` |

### `b` / `build`

Usage: `apktool b[uild] [options] <app_path>`

| Flag | Meaning |
|------|---------|
| `-f, --force-all` | Skip change detection and build all files |
| `-o, --output <dir>` | Output APK path; default is `dist/name.apk` |
| `-p, --frame-path <dir>` | Use framework files located in `<dir>` |

## Recipes

```bash
# Full decode into a named case directory
mkdir -p /home/user/cases/app && \
apktool d /home/user/cases/target.apk -o /home/user/cases/app

# Decode resources only -- fast manifest/resource review, no smali
apktool d /home/user/cases/target.apk -s -o /home/user/cases/app-res

# Decode sources only -- smali review when resource decoding fails
apktool d /home/user/cases/target.apk -r -o /home/user/cases/app-src

# Re-decode over an existing directory (force removes it first)
apktool d -f /home/user/cases/target.apk -o /home/user/cases/app

# Install an OEM framework and decode with its tag
apktool if -t vendor /home/user/cases/framework-res.apk
apktool d -t vendor /home/user/cases/target.apk -o /home/user/cases/app-vendor

# Rebuild an edited tree into an explicitly named APK
apktool b /home/user/cases/app -o /home/user/cases/app-rebuilt.apk
```

The rebuilt APK is unsigned. Signing is a separate step with your Android
tooling; apktool's captured help exposes no signing option.

## Output & pipelines

A full decode produces, at minimum:

- `AndroidManifest.xml` — decoded binary manifest
- `res/` — decoded resources
- `smali/` — disassembled DEX (plus `smali_classes2/`, etc. for multidex)
- `apktool.yml` — metadata used by the rebuild step
- `assets/`, `lib/`, `original/` — copied assets, native libraries, and the
  original manifest/META-INF when present

Useful follow-on steps:

```bash
# Manifest quick review
grep -E 'package=|android:permission|exported=|android:name' \
  /home/user/cases/app/AndroidManifest.xml

# Find URLs and interesting strings in smali
grep -RhoE 'https?://[^" ]+' /home/user/cases/app/smali* | sort -u

# Compare two decoded builds
diff -ru /home/user/cases/app-v1 /home/user/cases/app-v2 | less
```

Decoded trees also feed `grep`/`rg` workflows, secret scanners, and manual
review. Native libraries under `lib/` should be analyzed with the ELF/RE tools
in this KB rather than apktool.

## Gotchas & operational security

- `apktool --help` is not recognized by this build; use bare `apktool`.
- `-f` deletes the destination directory. Never point `-o` at a directory that
  contains uncommitted work.
- `-r` and `-s` produce partial trees. A build from a partial tree may fail or
  produce a broken APK; keep the full decode for rebuild work.
- OEM apps frequently need their matching framework. A resource-ID error during
  decode usually means the wrong framework is installed.
- Resource and smali output can be huge. Decode into a case directory, not
  `/tmp`, and budget disk space.
- The installed version reports `2.7.0-dirty`; behavior can differ from a clean
  upstream 2.7.0 build. Record the version in findings.
- Apktool output is untrusted input. Treat decoded XML/smali as data; do not
  execute scripts or install the APK on a production device.
- Static decoding does not reveal runtime behavior, native packing, or
  anti-tampering checks. Pair it with dynamic analysis on an isolated device.

## Legal & safety

Use apktool only on APKs you own, have written authorization to assess, or have
obtained lawfully for research. Decoding and rebuilding may violate license
terms, app-store rules, or local law when used to modify or redistribute
third-party software. Do not install rebuilt APKs outside an isolated lab, and
never handle suspected malware on a production workstation without a sandbox.

## Related tools

- [`../semgrep/`](../semgrep/README.md) — static rules over extracted source and
  configuration
- [`../secretfinder/`](../secretfinder/README.md) — secret/URL extraction from
  JavaScript, useful for hybrid apps
- [`../linkfinder/`](../linkfinder/README.md) — endpoint extraction from web
  assets bundled in an APK
- [`../pwndbg/`](../pwndbg/README.md) and [`../gef/`](../gef/README.md) — native
  library debugging when an APK ships `.so` files

## References

- Installed usage banner: [`help.txt`](help.txt)
- Upstream repository: https://github.com/iBotPeaches/Apktool
- Upstream documentation: https://ibotpeaches.github.io/Apktool/
- smali/baksmali: https://github.com/JesusFreke/smali
