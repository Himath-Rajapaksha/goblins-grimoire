# apktool — command scenarios for deep research

Scenarios for the installed apktool 2.7.0-dirty build. Every flag below is
from the captured `help.txt`; commands assume an authorized case file at
`/home/user/cases/target.apk`. Substitute your own case path, but keep all
output inside the case directory.

## How to invoke

```text
@deep-research For apktool 2.7.0, which decode failures are caused by a missing
OEM framework versus a genuinely malformed APK, and what is the minimal safe
sequence to install a vendor framework and rebuild without corrupting the
original? Cite upstream docs and issue threads; return exact flags and a
verification checklist.
```

---

## Scenario 1 — decode and inventory an APK

Start with a full decode so the manifest, resources, and smali are all
available. Inventory the tree before deciding which parts matter.

| # | Command | How to use |
|---|---------|------------|
| 1 | `mkdir -p /home/user/cases` | Create the case workspace |
| 2 | `apktool d /home/user/cases/target.apk -o /home/user/cases/app` | Full decode into a named directory |
| 3 | `ls /home/user/cases/app` | Confirm manifest, `res/`, `smali/`, `apktool.yml` |
| 4 | `grep -E 'package=\|android:permission\|exported=' /home/user/cases/app/AndroidManifest.xml` | First-pass manifest review |
| 5 | `du -sh /home/user/cases/app` | Budget disk before copying or diffing |

**Research questions**
- What does a "successful" full decode guarantee, and what can still be packed
  or obfuscated?
- Which manifest attributes are most useful for attack-surface triage?

**Sources**: upstream decode docs, Android manifest reference, apktool issue
tracker for decode failures.
**Output**: a decoded tree and a one-page manifest inventory.

---

## Scenario 2 — split resource and source decoding

When one side fails or is irrelevant, decode only the other. This is faster and
isolates resource-decoding errors from smali errors.

| # | Command | How to use |
|---|---------|------------|
| 6 | `apktool d -s /home/user/cases/target.apk -o /home/user/cases/app-res` | Resources only, no smali |
| 7 | `apktool d -r /home/user/cases/target.apk -o /home/user/cases/app-src` | Sources only, no resources |
| 8 | `apktool d -f -s /home/user/cases/target.apk -o /home/user/cases/app-res` | Re-decode resources, replacing the old tree |
| 9 | `find /home/user/cases/app-res/res -maxdepth 2 -type d \| head` | Check which resource directories decoded |

**Research questions**
- Which apktool failures are resource-only and disappear with `-r`?
- Can a `-s` tree still be rebuilt, or are resources required by the build?
- Does `-f` delete only the output directory, or anything else?

**Sources**: `help.txt` decode options, upstream build documentation, issue
threads about partial decodes.
**Output**: two partial trees plus a note on which decode path is viable.

---

## Scenario 3 — install and select an OEM framework

Vendor and modified-ROM APKs often reference resource IDs absent from the
stock framework. Install the matching `framework-res.apk` under a tag, then
decode with that tag.

| # | Command | How to use |
|---|---------|------------|
| 10 | `apktool if /home/user/cases/framework-res.apk` | Install as the default framework |
| 11 | `apktool if -t vendor /home/user/cases/framework-res.apk` | Install with a named tag |
| 12 | `apktool d -t vendor /home/user/cases/target.apk -o /home/user/cases/app-vendor` | Decode using the tagged framework |
| 13 | `apktool d -p /home/user/cases/frameworks -t vendor /home/user/cases/target.apk -o /home/user/cases/app-vendor2` | Use a custom framework directory |
| 14 | `ls ~/.local/share/apktool/framework/` | Inspect the default framework store |

**Research questions**
- How does apktool map a `-t` tag to a stored framework file?
- When is `-p` preferable to the default `~/.local/share/apktool/framework/`?
- What happens when the wrong framework is selected: hard error or wrong values?

**Sources**: upstream framework docs, apktool issues on resource ID `0x01...`
errors, vendor ROM extraction notes.
**Output**: a working framework selection and the exact decode command that
reproduces it.

---

## Scenario 4 — rebuild a modified tree

After editing smali or resources, rebuild into an explicitly named APK. Keep
the original APK untouched and verify the output before any signing step.

| # | Command | How to use |
|---|---------|------------|
| 15 | `apktool b /home/user/cases/app -o /home/user/cases/app-rebuilt.apk` | Build to a named output file |
| 16 | `apktool b -f /home/user/cases/app -o /home/user/cases/app-rebuilt.apk` | Force a full rebuild after broad edits |
| 17 | `ls -l /home/user/cases/app-rebuilt.apk` | Confirm the artifact exists and has non-zero size |
| 18 | `unzip -l /home/user/cases/app-rebuilt.apk \| head -30` | Inspect the rebuilt archive layout |

**Research questions**
- Which edits require `-f`, and which are detected incrementally?
- What does apktool change in the manifest or resources during a rebuild?
- Why is the rebuilt APK unsigned, and which signing tool should follow?
- How can a rebuild be verified without installing it?

**Sources**: upstream build docs, `help.txt` build flags, apktool issues on
incremental builds and `dist/` output.
**Output**: a rebuilt APK, archive listing, and a signing/verification TODO.

---

## Scenario 5 — resource and asset review without smali

For manifest/permission/native-library questions, skip smali entirely and work
from the resource-only tree.

| # | Command | How to use |
|---|---------|------------|
| 19 | `find /home/user/cases/app-res -maxdepth 3 -type f \| sort \| head -50` | Enumerate decoded files |
| 20 | `find /home/user/cases/app-res/lib -type f 2>/dev/null` | List packaged native libraries |
| 21 | `grep -RhoE 'https?://[^" <]+' /home/user/cases/app-res/res 2>/dev/null \| sort -u` | Extract URLs from resources |
| 22 | `file /home/user/cases/app-res/lib/*/* 2>/dev/null` | Identify native library architectures |

**Research questions**
- Which resource files commonly hide endpoints or configuration?
- How should native libraries be handed to ELF analysis tools?
- Does `-s` preserve enough structure for manifest and asset review?

**Sources**: Android resource format docs, upstream apktool resource docs,
mobile assessment methodology.
**Output**: an asset inventory and a handoff list for native/ELF analysis.

---

## Scenario 6 — batch triage and pipeline handoff

For multiple APKs, decode each into its own directory and keep a manifest
index. This scenario exercises the remaining output/overwrite flags.

| # | Command | How to use |
|---|---------|------------|
| 23 | `for apk in /home/user/cases/apks/*.apk; do apktool d -s "$apk" -o "/home/user/cases/decoded/$(basename "$apk" .apk)"; done` | Batch resource-only decode |
| 24 | `apktool d -f -o /home/user/cases/app2 /home/user/cases/target.apk` | Force decode when the destination already exists |
| 25 | `apktool --version` | Record the exact build in the case notes |
| 26 | `apktool` | Show the usage banner when a flag is rejected |

**Research questions**
- What is a safe batch layout that avoids output collisions?
- Which failures should stop the batch versus be logged and skipped?
- How should version and framework state be recorded for reproducibility?

**Sources**: upstream CLI docs, shell loop patterns, apktool issue threads on
batch decoding.
**Output**: one decoded directory per APK, a manifest index, and a reproducible
version record.

---

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Top level | `-advance`, `-version` | 6 |
| `if` / `install-framework` | `-p`, `-t` | 3 |
| `d` / `decode` | `-f`, `-o`, `-p`, `-r`, `-s`, `-t` | 1, 2, 3, 6 |
| `b` / `build` | `-f`, `-o`, `-p` | 4 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — decode and inventory | 5 |
| 2 — split resource/source decoding | 4 |
| 3 — OEM framework selection | 5 |
| 4 — rebuild a modified tree | 4 |
| 5 — resource and asset review | 4 |
| 6 — batch triage and pipeline | 4 |
| **Total** | **26** |
