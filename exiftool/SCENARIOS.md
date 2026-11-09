# exiftool — command scenarios for deep research

Installed build: ExifTool 13.25 (Debian `libimage-exiftool-perl`
`13.25+dfsg-1`). Every option below appears in `man exiftool`; the short
[`help.txt`](help.txt) snapshot is only the no-argument syntax block because
`-help` is treated as a tag name on this build.

> **Scope:** the recipes read and write metadata in a throwaway demo tree.
> Metadata can contain personal data, GPS traces, device serial numbers, and
> authorship history. Do not process third-party media without authorisation,
> and do not publish a metadata dump without reviewing it first.

> **Backup warning:** every write creates a `<file>_original` backup unless
> `-overwrite_original` is given. The recipes that strip metadata use
> `-overwrite_original` only after a copy has been made. `-overwrite_original`
> is irreversible.

## How to invoke

```text
@deep-research For an authorised metadata review with ExifTool 13.25, explain
how tag names resolve to different containers across JPEG, PNG, PDF, and
Office files, how the group families (G0/G1) and composite tags are derived,
and how writes, backups, and `-overwrite_original` change files on disk.
Compare `-json`, `-csv`, `-X`, `-args`, and `-p` output, and explain the
exit-status contract and the missing-path exception. Cite the exiftool man
page, the installed tag database (`-listx`), and observed output. Separate
confirmed behaviour from inference and state confidence.
```

Create the demo tree and a tagged image:

```bash
mkdir -p ~/exiftool-demo/evidence ~/exiftool-demo/photos
cd ~/exiftool-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
python3 - <<'PY'
from PIL import Image
im = Image.new('RGB', (32, 24), (20, 120, 200))
im.save('photos/blue.png')
im.save('photos/blue.jpg', quality=90)
PY
exiftool -Artist='Alice' -Comment='demo note' -Copyright='ACME' photos/blue.jpg >/dev/null
exiftool -Artist='Alice' -Comment='demo note' photos/blue.png >/dev/null
sha256sum photos/* > evidence/before.sha256
```

## Scenario 1 — version, help, format inventory

| # | Command | Purpose |
|---|---------|---------|
| 1 | `exiftool -ver` | Print the version |
| 2 | `exiftool` | No arguments: short syntax block |
| 3 | `exiftool --help` | Long-form help flag: same syntax block |
| 4 | `exiftool -help` | Show that `-help` is treated as a tag |
| 5 | `man exiftool \| col -b \| head -30` | Read the authoritative manual |
| 6 | `man exiftool \| col -b \| grep -n '^EXIT STATUS'` | Locate the exit-status section |
| 7 | `exiftool -listf` | All recognised extensions (19 lines) |
| 8 | `exiftool -listwf` | Writable extensions |
| 9 | `exiftool -listg0` | Tag groups, family 0 |
| 10 | `exiftool -listg1 \| head -10` | Tag groups, family 1 |
| 11 | `exiftool -listd` | Deletable groups |
| 12 | `exiftool -listx \| head -5` | XML tag database header |
| 13 | `exiftool -listx \| wc -l` | Size of the tag database |
| 14 | `dpkg -s libimage-exiftool-perl \| grep -E 'Version\|Depends'` | Package provenance |

**Research questions**
- Why does `-help` print `No file specified` while `--help` prints syntax?
- Which extensions are writable but not readable, or vice versa?
- How large is the tag database, and why does `-listx` matter?
- Which group family should a report cite?

**Sources:** `man exiftool`; `-list*` output; `dpkg -s`.
**Output:** a version/format inventory and the authoritative help pointer.

## Scenario 2 — reading tags, groups, duplicates

| # | Command | Purpose |
|---|---------|---------|
| 15 | `exiftool photos/blue.jpg` | Dump all readable tags |
| 16 | `exiftool -Artist photos/blue.jpg` | One tag |
| 17 | `exiftool -Artist -Comment -Copyright photos/blue.jpg` | Several tags |
| 18 | `exiftool -G1 -a -s photos/blue.jpg` | Group, duplicates, raw tag names |
| 19 | `exiftool -G0 -a -s photos/blue.jpg` | Family-0 groups only |
| 20 | `exiftool -s -S -Artist photos/blue.jpg` | Value only |
| 21 | `exiftool -a -Artist photos/blue.jpg` | Allow duplicate `Artist` tags |
| 22 | `exiftool -e -Artist photos/blue.jpg` | Suppress composite tags |
| 23 | `exiftool -u -U photos/blue.jpg \| head -20` | Include unknown tags |
| 24 | `exiftool -f -Artist -NoSuchTag photos/blue.jpg` | Force printing of missing tags |
| 25 | `exiftool -fast2 -Artist photos/blue.jpg` | Faster, shallower extraction |
| 26 | `exiftool -ee -Artist photos/blue.jpg` | Extract embedded-file metadata |
| 27 | `exiftool -validate -warning -error photos/blue.jpg` | Validate and report problems |
| 28 | `exiftool -json -G1 -a photos/blue.jpg` | Grouped JSON record |

**Research questions**
- Which tags appear in more than one group?
- What do `-G0` and `-G1` add to a report?
- Does `-a` change values or only duplicate visibility?
- Which tags are composite rather than stored?

**Sources:** `man exiftool`; observed output.
**Output:** a tag inventory with group provenance.

## Scenario 3 — output formats

| # | Command | Purpose |
|---|---------|---------|
| 29 | `exiftool -json photos/blue.jpg` | JSON object |
| 30 | `exiftool -json -r photos/ > evidence/photos.json` | JSON for a tree |
| 31 | `exiftool -csv photos/blue.jpg` | CSV row |
| 32 | `exiftool -csv -r photos/ > evidence/photos.csv` | CSV manifest |
| 33 | `exiftool -X photos/blue.jpg \| head -20` | RDF/XML output |
| 34 | `exiftool -php photos/blue.jpg \| head -10` | PHP array output |
| 35 | `exiftool -args photos/blue.jpg \| head -10` | `-Tag=value` argument lines |
| 36 | `exiftool -h photos/blue.jpg \| head -8` | HTML table output |
| 37 | `exiftool -htmldump photos/blue.jpg > evidence/dump.html` | Binary HTML dump to stdout |
| 38 | `exiftool -t photos/blue.jpg \| head -8` | Tab-delimited list |
| 39 | `exiftool -T -Artist -Comment photos/blue.jpg` | Tabular values |
| 40 | `exiftool -p '$FileName: $Artist / $Comment' photos/blue.jpg` | Custom print format |
| 41 | `exiftool -struct -j photos/blue.jpg \| head -12` | Structured JSON |
| 42 | `exiftool -sort photos/blue.jpg \| head -8` | Alphabetical output |

**Research questions**
- Which formats are lossless enough for a report pipeline?
- Does `-args` output round-trip as arguments?
- What does `-struct` change compared with plain `-j`?
- Why is `-htmldump` stdout-only?

**Sources:** `man exiftool`; observed output.
**Output:** the same record in seven machine-readable shapes.

## Scenario 4 — filtering, formatting, and date/GPS presentation

| # | Command | Purpose |
|---|---------|---------|
| 43 | `exiftool -if '$Artist' -Artist photos/blue.jpg` | Process only files with an Artist |
| 44 | `exiftool -if '$Artist eq "Alice"' -Artist photos/blue.jpg` | Conditional expression |
| 45 | `exiftool -if 'not $Artist' -FileName photos/blue.png` | Negative condition |
| 46 | `exiftool -x FileName -x Directory photos/blue.jpg \| head -8` | Exclude tags |
| 47 | `exiftool -n -ImageWidth -ImageHeight photos/blue.jpg` | Raw numeric values |
| 48 | `exiftool -d '%Y-%m-%d' -FileModifyDate photos/blue.jpg` | Custom date format |
| 49 | `exiftool -coordFormat '%d %d %.6f' -GPSLatitude photos/blue.jpg` | GPS formatting |
| 50 | `exiftool -sep ', ' -Keywords photos/blue.jpg` | List separator |
| 51 | `exiftool -L -Artist photos/blue.jpg` | Latin-1 output |
| 52 | `exiftool -charset utf8 -Artist photos/blue.jpg` | UTF-8 handling |
| 53 | `exiftool -lang de -Artist photos/blue.jpg` | German tag descriptions |
| 54 | `exiftool -E -Artist photos/blue.jpg` | HTML escaping |
| 55 | `exiftool -b -ThumbnailImage photos/blue.jpg \| head -c 16 \| xxd` | Binary tag output |
| 56 | `exiftool -fileOrder FileName -Artist photos/` | Deterministic file order |

**Research questions**
- Which filter expressions are safe to use in batch jobs?
- Does `-if` change the exit status when nothing matches?
- Which formatting options affect values versus only labels?
- Is `-b` safe to pipe into a decoder?

**Sources:** `man exiftool`; observed output.
**Output:** a filtered, formatted, deterministic report.

## Scenario 5 — writing tags and backup semantics

| # | Command | Purpose |
|---|---------|---------|
| 57 | `cp photos/blue.jpg evidence/write-test.jpg` | Work on a copy |
| 58 | `exiftool -Artist='Bob' evidence/write-test.jpg` | Write one tag, keep a backup |
| 59 | `ls -l evidence/write-test.jpg*` | Confirm `_original` exists |
| 60 | `exiftool -Artist evidence/write-test.jpg` | Read the new value |
| 61 | `exiftool -Comment+= 'extra' evidence/write-test.jpg` | Append to a list tag |
| 62 | `exiftool -Comment-= 'extra' evidence/write-test.jpg` | Remove from a list tag |
| 63 | `exiftool -overwrite_original -Artist='Carol' evidence/write-test.jpg` | No backup |
| 64 | `ls -l evidence/write-test.jpg*` | Confirm no new backup |
| 65 | `exiftool -overwrite_original_in_place -Artist='Dave' evidence/write-test.jpg` | In-place write |
| 66 | `stat -c '%i %a %U' evidence/write-test.jpg` | Check inode and attributes |
| 67 | `exiftool -P -Comment='kept mtime' evidence/write-test.jpg` | Preserve file mtime |
| 68 | `printf 'Eve\n' > evidence/artist.txt` | Prepare a value file |
| 69 | `exiftool '-Artist<=evidence/artist.txt' evidence/write-test.jpg` | Write from file contents |
| 70 | `exiftool -all= evidence/write-test.jpg` | Strip all writable metadata |
| 71 | `exiftool -Artist -Comment evidence/write-test.jpg` | Confirm the strip |
| 72 | `exiftool -delete_original! evidence/write-test.jpg` | Remove the backup non-interactively |

**Research questions**
- Which write path preserves inode, owner, and permissions?
- Does `-all=` remove filesystem timestamps?
- What happens if the process dies mid-write?
- Is `-overwrite_original` ever the safe default?

**Sources:** `man exiftool`; observed file metadata.
**Output:** a write/backup matrix with inode evidence.

## Scenario 6 — copying tags and batch trees

| # | Command | Purpose |
|---|---------|---------|
| 73 | `exiftool -TagsFromFile photos/blue.jpg -Artist -Copyright photos/blue.png` | Copy named tags |
| 74 | `exiftool -TagsFromFile photos/blue.jpg -all:all photos/blue.png` | Copy every tag |
| 75 | `exiftool -TagsFromFile @ -Artist photos/blue.png` | Copy within one file |
| 76 | `exiftool -all= -tagsFromFile @ -Artist photos/blue.png` | Keep only Artist |
| 77 | `exiftool -r -ext jpg -Artist photos/` | Recurse, filter by extension |
| 78 | `exiftool -r -ext+ png -Artist photos/` | Add PNG to the default set |
| 79 | `exiftool -i evidence -Artist photos/` | Ignore a directory name |
| 80 | `exiftool -srcfile %d%f.xmp -Artist photos/blue.jpg` | Process a sidecar file |
| 81 | `exiftool -o evidence/out -ext jpg photos/` | Copy to an output directory |
| 82 | `exiftool -w evidence/%f.txt -Artist photos/blue.jpg` | Write one text file per input |
| 83 | `exiftool -W evidence/%f-%t.txt -Artist -Comment photos/blue.jpg` | One file per tag |
| 84 | `exiftool -progress -Artist photos/` | Progress count |
| 85 | `exiftool -echo '--- done ---' -Artist photos/blue.jpg` | Echo a marker |
| 86 | `exiftool -efile evidence/errors.txt -Artist photos/` | Record files with errors |

**Research questions**
- Which copy form creates tags in the preferred group?
- Does `-o` preserve the relative directory structure?
- What does `%d%f%e` expand to?
- When is `-srcfile` preferable to copying?

**Sources:** `man exiftool`; observed output.
**Output:** a batch-copy recipe with per-file artifacts.

## Scenario 7 — JSON/CSV export and import

| # | Command | Purpose |
|---|---------|---------|
| 87 | `exiftool -json -r photos/ > evidence/manifest.json` | Export JSON |
| 88 | `python3 -m json.tool evidence/manifest.json \| head -20` | Validate the JSON |
| 89 | `exiftool -csv -r photos/ > evidence/manifest.csv` | Export CSV |
| 90 | `head -3 evidence/manifest.csv` | Inspect the CSV header |
| 91 | `cp photos/blue.jpg evidence/import.jpg` | Prepare an import target |
| 92 | `printf '[{"SourceFile":"%s","Artist":"JSON Artist"}]\n' "$PWD/evidence/import.jpg" > evidence/import.json` | Build a JSON import |
| 93 | `exiftool -json=evidence/import.json evidence/import.jpg` | Import with explicit file args |
| 94 | `exiftool -Artist evidence/import.jpg` | Confirm the import |
| 95 | `exiftool -json=evidence/import.json` | Show the missing-file error |
| 96 | `printf 'SourceFile,Artist\n%s,CSV Artist\n' "$PWD/evidence/import.jpg" > evidence/import.csv` | Build a CSV import |
| 97 | `exiftool -csv=evidence/import.csv evidence/import.jpg` | Import CSV |
| 98 | `exiftool -Artist evidence/import.jpg` | Confirm the CSV value |
| 99 | `exiftool -csvDelim ';' -csv -r photos/ \| head -3` | Custom CSV delimiter |
| 100 | `jq '.[0] \| keys \| length' evidence/manifest.json` | Count keys in the first record |

**Research questions**
- Why do `-json=` and `-csv=` need explicit file arguments?
- Which fields are required in an import record?
- Does import merge or replace existing tags?
- Which export format is safest for a lossless round trip?

**Sources:** `man exiftool`; observed import/export output.
**Output:** a validated export/import pair.

## Scenario 8 — dates, GPS, language, config, API

| # | Command | Purpose |
|---|---------|---------|
| 101 | `cp photos/blue.jpg evidence/date.jpg` | Work on a copy |
| 102 | `exiftool -DateTimeOriginal='2024:05:01 12:00:00' evidence/date.jpg` | Write a capture date |
| 103 | `exiftool -d '%Y-%m-%d %H:%M' -DateTimeOriginal evidence/date.jpg` | Reformat on read |
| 104 | `exiftool -globalTimeShift '+1:00' -DateTimeOriginal evidence/date.jpg` | Shift displayed times |
| 105 | `exiftool -GPSLatitude=51.5 -GPSLatitudeRef=N evidence/date.jpg` | Write a latitude |
| 106 | `exiftool -GPSLongitude=0.12 -GPSLongitudeRef=W evidence/date.jpg` | Write a longitude |
| 107 | `exiftool -GPSPosition evidence/date.jpg` | Read the composite position |
| 108 | `exiftool -n -GPSLatitude -GPSLongitude evidence/date.jpg` | Raw coordinate values |
| 109 | `exiftool -use MWG -Artist evidence/date.jpg` | MWG module tag resolution |
| 110 | `exiftool -config /dev/null -Artist evidence/date.jpg` | Show the config warning |
| 111 | `exiftool -api QuickTimeUTC=1 -Artist photos/blue.jpg` | Set an API option |
| 112 | `exiftool -lang de -Artist evidence/date.jpg` | Localised descriptions |

**Research questions**
- Are GPS values stored as rationals or decimals?
- Does `-globalTimeShift` change stored data or only output?
- Which MWG tags differ from their EXIF counterparts?
- Is `-config /dev/null` safe in a batch job?

**Sources:** `man exiftool`; observed output.
**Output:** a date/GPS record with raw and formatted views.

## Scenario 9 — validation, diff, error handling, cleanup

| # | Command | Purpose |
|---|---------|---------|
| 113 | `exiftool -validate -warning -error photos/blue.jpg` | Validation pass |
| 114 | `exiftool -diff photos/blue.jpg evidence/date.jpg` | Metadata diff |
| 115 | `exiftool /nonexistent` | Missing path: note the exit status |
| 116 | `exiftool /nonexistent >/dev/null 2>&1; echo "exit=$?"` | Confirm the zero exit |
| 117 | `exiftool -if '0' photos/blue.jpg >/dev/null 2>&1; echo "exit=$?"` | All files fail the condition |
| 118 | `exiftool -Artist=X /etc/hostname >/dev/null 2>&1; echo "exit=$?"` | Write error exit |
| 119 | `exiftool --output=json photos/blue.jpg \| head -3` | Unrecognised option becomes a tag |
| 120 | `exiftool -htmldump -o evidence/dump2.html photos/blue.jpg` | Show the `-o` incompatibility |
| 121 | `timeout 2 exiftool -delete_original evidence/write-test.jpg` | Interactive delete behaviour |
| 122 | `cp evidence/write-test.jpg evidence/restore.jpg && exiftool -Comment='tmp' evidence/restore.jpg >/dev/null` | Prepare a restore test |
| 123 | `exiftool -restore_original evidence/restore.jpg` | Restore from backup |
| 124 | `exiftool -Comment evidence/restore.jpg` | Confirm the restored value |
| 125 | `rm -rf ~/exiftool-demo` | Remove the demo tree |
| 126 | `test ! -e ~/exiftool-demo; echo "removed=$?"` | Verify cleanup |

**Research questions**
- Which failures are invisible to `$?`?
- Does `-restore_original` consume the backup?
- Can `-validate` catch a semantically valid but privacy-leaking tag?
- What is the safe wrapper for a batch write?

**Sources:** `man exiftool`; observed exit codes.
**Output:** an exit-code table and a cleaned workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `-ver`, `-listf`, `-listwf`, `-listg0`, `-listg1`, `-listd`, `-listx` | 1 |
| `-TAG`, `-x`, `-a`, `-e`, `-ee`, `-u`, `-U`, `-f`, `-fast`, `-validate` | 2 |
| `-j`, `-csv`, `-X`, `-php`, `-args`, `-h`, `-htmldump`, `-t`, `-T`, `-p`, `-struct`, `-sort` | 3 |
| `-if`, `-n`, `-d`, `-c`, `-sep`, `-L`, `-charset`, `-lang`, `-E`, `-b`, `-fileOrder` | 4 |
| `-TAG=`, `+=`, `-=`, `-all=`, `-P`, `-overwrite_original`, `-overwrite_original_in_place`, `-delete_original` | 5 |
| `-tagsFromFile`, `-r`, `-ext`, `-i`, `-srcfile`, `-o`, `-w`, `-W`, `-progress`, `-echo`, `-efile` | 6 |
| `-json=`, `-csv=`, `-csvDelim` | 7 |
| `-globalTimeShift`, `-use`, `-config`, `-api` | 8 |
| `-diff`, `-restore_original` | 9 |

Full text: `man exiftool`; short syntax in [`help.txt`](help.txt); extensions
in [`help-listf.txt`](help-listf.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, help, format inventory | 14 |
| 2 | Reading tags, groups, duplicates | 14 |
| 3 | Output formats | 14 |
| 4 | Filtering, formatting, and date/GPS presentation | 14 |
| 5 | Writing tags and backup semantics | 16 |
| 6 | Copying tags and batch trees | 14 |
| 7 | JSON/CSV export and import | 14 |
| 8 | Dates, GPS, language, config, API | 12 |
| 9 | Validation, diff, error handling, cleanup | 14 |
| **Total** | | **126** |

## Legal and safety

Metadata is often the most sensitive part of a file: GPS traces, device
serials, author names, edit history, and embedded thumbnails that survive a
crop. Treat every dump as personal data. Do not strip metadata from material
you do not own or lack authorisation to modify, and do not use
`-overwrite_original` on evidence — a backup is the only rollback path.
Delete the demo tree when finished.
