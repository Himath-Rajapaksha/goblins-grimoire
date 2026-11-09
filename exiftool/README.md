# exiftool — read and write metadata in media and document files

- **Version:** 13.25 (Debian `libimage-exiftool-perl` `13.25+dfsg-1`)
- **Binary:** `/usr/bin/exiftool` (Perl)
- **Perl library:** `Image::ExifTool` under `/usr/share/perl5/Image/ExifTool/`
- **Help snapshot:** [`help.txt`](help.txt) — `exiftool -help` (this build prints `No file specified`)
- **File-type list:** [`help-listf.txt`](help-listf.txt) — `exiftool -listf`
- **Man page:** `man exiftool` (**authoritative** — it is ~2,900 lines)
- **Upstream:** https://exiftool.org/
- **Source:** https://github.com/exiftool/exiftool
- **Licence:** Perl Artistic License / GPL (see `/usr/share/doc/libimage-exiftool-perl/copyright`)

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

> **Help-text warning:** `exiftool -help` does **not** print the manual on this
> build; it treats `-help` as a tag name and prints `No file specified`.
> `exiftool --help` prints the short syntax block, and `exiftool` with no
> arguments prints the same. Use `man exiftool` for the full option reference.

## What it does

ExifTool reads, writes, and copies metadata across an unusually wide range of
formats. It understands EXIF, IPTC, XMP, ICC profiles, MakerNotes, GPS,
QuickTime atoms, PDF objects, Office XML parts, HTML, PNG chunks, and dozens
of vendor-specific structures. It is the reference implementation for
metadata inspection because it normalises tag names across containers while
still exposing the raw group and tag ID.

The two things to understand before using it:

1. **A tag name is a lookup key, not a location.** `-Artist` resolves to the
   right container for the file type (EXIF `Artist` in JPEG, XMP `Artist` in
   PNG, PDF `Author`, etc.). `-G1 -a -s` shows where each value actually came
   from.
2. **Writing is a file rewrite.** ExifTool writes a new file and renames it
   over the original. The default keeps a `<file>_original` backup. A crash
   or a full disk can leave the original untouched but a partial temp file
   behind.

## Supported formats

`exiftool -listf` prints the 19-line extension inventory; the short list
includes JPEG, TIFF, PNG, HEIC/HEIF, AVIF, WebP, GIF, SVG, PDF, PostScript,
Office (DOCX/XLSX/PPTX and legacy DOC/XLS/PPT), OpenDocument, EPUB, MOBI,
MP3, MP4, MOV, MKV, WebM, FLAC, OGG, WAV, AVI, ZIP, 7z, RAR, and many camera
raw formats. `exiftool -listwf` prints only the writable extensions.

```bash
exiftool -listf          # all recognised extensions (19 lines)
exiftool -listwf         # writable extensions
exiftool -listg0         # tag groups, family 0
exiftool -listg1         # tag groups, family 1
exiftool -listd          # deletable groups
exiftool -listx          # full XML tag database (496,771 lines)
```

`-listx` is enormous; never dump it to a terminal. Redirect it or grep it.

## Tag model

| Concept | Meaning |
|---------|---------|
| Tag | A named metadata field, e.g. `Artist`, `CreateDate`, `GPSLatitude` |
| Group family 0 | High-level container: `EXIF`, `XMP`, `IPTC`, `PDF`, `File`, `Composite` |
| Group family 1 | More specific location: `IFD0`, `ExifIFD`, `XMP-dc`, `MakerNotes` |
| Composite tag | Derived value computed from other tags (e.g. `ImageSize`, `GPSPosition`) |
| Writable tag | A tag ExifTool can set for that file type (`-listw`) |
| Deletable group | A group that `-all=` can remove (`-listd`) |

`-a` allows duplicate tags (same name in different groups). `-G1` prefixes each
line with its family-1 group. `-s` uses the tag name instead of the description.

## Basic syntax

```bash
exiftool image.jpg                    # all readable tags
exiftool -Artist image.jpg            # one tag
exiftool -Artist -Comment image.jpg   # several tags
exiftool -Artist=Alice image.jpg      # write a tag
exiftool -all= image.jpg              # remove all writable metadata
exiftool -json image.jpg              # JSON output
```

Options may appear after file names. Tag names are case-insensitive, but
single-character options are case-sensitive when an upper-case variant exists
(`-s` vs `-S`).

## Reading options

| Option | Meaning |
|--------|---------|
| `-TAG` | Extract the named tag |
| `-x TAG`, `-exclude` | Exclude a tag |
| `-a`, `-duplicates` | Allow duplicate tags from different groups |
| `-e`, `--composite` | Do not generate composite tags |
| `-ee[NUM]`, `-extractEmbedded` | Extract metadata from embedded files |
| `-u`, `-unknown` | Extract unknown tags |
| `-U`, `-unknown2` | Extract unknown binary tags too |
| `-f`, `-forcePrint` | Print all requested tags even if missing |
| `-fast[NUM]` | Faster, less thorough extraction (`-fast2`, `-fast3`) |
| `-scanForXMP` | Brute-force scan for XMP |
| `-validate` | Validate metadata and report warnings/errors |
| `-warning`, `-error` | Print only warnings or errors |
| `-diff FILE2` | Compare metadata with another file |

## Writing options

| Option | Meaning |
|--------|---------|
| `-TAG=VALUE` | Set a tag |
| `-TAG+=VALUE` | Add to a list tag |
| `-TAG-=VALUE` | Remove from a list tag |
| `-TAG^=VALUE` | Set and delete the original tag (XMP) |
| `-TAG<=DATFILE` | Write a tag value from a file's contents (`DATFILE` may use `%d`/`%f`/`%e`) |
| `-all=` | Delete all writable metadata |
| `-tagsFromFile SRCFILE` | Copy tags from another file |
| `-wm MODE`, `-writeMode` | Control tag creation (`w`, `c`, `g`, `d`) |
| `-P`, `-preserve` | Preserve file modification date/time |
| `-overwrite_original` | Suppress the `_original` backup |
| `-overwrite_original_in_place` | Overwrite in place, preserving attributes |
| `-delete_original[!]` | Delete `_original` backups (`!` without prompting) |
| `-restore_original` | Restore from `_original` backups |
| `-o OUTFILE`, `-out` | Write output to a new file or directory |
| `-progress[NUM][:[TITLE]]` | Show a progress count |

## Copying options

```bash
exiftool -TagsFromFile src.jpg -Artist -Comment dst.jpg
exiftool -TagsFromFile src.jpg -all:all dst.jpg
exiftool -TagsFromFile @ -Artist -Comment dst.jpg
exiftool -all= -tagsFromFile @ -Artist dst.jpg
```

`@` means "the current file", which is how you copy between groups within one
file. `-all:all` copies every tag; without it you must name the tags. Copying
is a write, so the backup rules apply.

## Output format options

| Option | Meaning |
|--------|---------|
| `-j`, `-json` | JSON output (one object per file) |
| `-csv` | CSV output (one row per file) |
| `-X`, `-xmlFormat` | RDF/XML output |
| `-php` | PHP array output |
| `-args`, `-argFormat` | `-Tag=value` lines, reusable as arguments |
| `-h`, `-htmlFormat` | HTML table output |
| `-htmldump[OFFSET]` | Binary HTML dump to stdout |
| `-p STR`, `-printFormat` | Custom print format using `$TAG` |
| `-t`, `-tab` | Tab-delimited list |
| `-T`, `-table` | Tabular output |
| `-S`, `-veryShort` | Values only |
| `-s[NUM]`, `-short` | Tag names instead of descriptions |
| `-D`, `-decimal` | Decimal tag IDs |
| `-H`, `-hex` | Hexadecimal tag IDs |
| `-g[NUM]`, `-groupHeadings` | Group output by tag group |
| `-G[NUM]`, `-groupNames` | Prefix each line with its group |
| `-l`, `-long` | Two-line long output |
| `-v[NUM]`, `-verbose` | Verbose structural output |
| `-n`, `--printConv` | Raw values, no print conversion |
| `-d FMT`, `-dateFormat` | Date/time format |
| `-c FMT`, `-coordFormat` | GPS coordinate format |
| `-sep STR`, `-separator` | List-item separator |
| `-struct` | Structured output for JSON/XML |
| `-sort` | Sort output alphabetically |
| `-L`, `-latin` | Windows Latin-1 encoding |
| `-charset [[TYPE=]CHARSET]` | Character-set handling |
| `-lang [LANG]` | Output language |
| `-E`, `-ex`, `-ec` | Escape for HTML, XML, or C |
| `-b`, `-binary` | Output binary tag values |

## File handling options

| Option | Meaning |
|--------|---------|
| `-r[.]`, `-recurse` | Recurse into subdirectories |
| `-ext[+] EXT`, `-extension` | Process only these extensions (`+` adds to defaults) |
| `-i DIR`, `-ignore` | Ignore a directory name |
| `-fileOrder[NUM] [-]TAG` | Sort file processing order |
| `-srcfile FMT` | Process a different source file |
| `-fileNUM ALTFILE` | Load tags from an alternate file |
| `-o OUTFILE`, `-out` | Output file or directory |
| `-w[+|!] EXT`, `-textOut` | Write text output per input file |
| `-W[+|!] FMT`, `-tagOut` | Write one output file per tag |
| `-Wext EXT`, `-tagOutExt` | Restrict `-W` to certain extensions |
| `-password PASSWD` | Password for protected files |
| `-z`, `-zip` | Read/write compressed information |
| `-list_dir` | List directories, not their contents |
| `-q`, `-quiet` | Suppress normal output |
| `-m`, `-ignoreMinorErrors` | Ignore minor errors |
| `-k`, `-pause` | Pause before terminating |
| `-progress` | Progress count |
| `-echo[NUM] TEXT` | Echo text to stdout/stderr |
| `-efile[NUM][!] TXTFILE` | Save names of files with errors |

## Batch and integration options

| Option | Meaning |
|--------|---------|
| `-@ ARGFILE` | Read arguments from a file |
| `-stay_open FLAG` | Keep reading the arg file after EOF |
| `-execute[NUM]` | Execute multiple commands on one line |
| `-common_args` | Arguments that apply to every `-execute` block |
| `-config CFGFILE` | Use a specific config file |
| `-api OPT[[^]=[VAL]]` | Set an API option |
| `-userParam PARAM[[^]=[VAL]]` | Set a user parameter |
| `-use MODULE` | Load a plug-in module (e.g. `MWG`) |
| `-globalTimeShift SHIFT` | Shift all formatted date/time values |
| `-geotag TRKFILE` | Geotag images from a GPS log |
| `-plot` | Output an SVG plot of tags |

## Backup and overwrite semantics

| Invocation | Original file | Backup left behind |
|------------|---------------|--------------------|
| `exiftool -Artist=X f.jpg` | Replaced | `f.jpg_original` |
| `exiftool -overwrite_original -Artist=X f.jpg` | Replaced | none |
| `exiftool -overwrite_original_in_place -Artist=X f.jpg` | Replaced in place | none; inode/attributes preserved |
| `exiftool -o out.jpg -Artist=X f.jpg` | Unchanged | none; `out.jpg` created |
| `exiftool -delete_original f.jpg` | Unchanged | backup deleted (interactive prompt) |
| `exiftool -delete_original! f.jpg` | Unchanged | backup deleted, no prompt |
| `exiftool -restore_original f.jpg` | Restored from backup | backup consumed |

`-overwrite_original` is the usual batch choice. It removes the safety net,
so validate the command on a copy first. `-overwrite_original_in_place`
preserves hard links, ownership, and permissions, at the cost of a riskier
write path.

## Exit status

The man page defines `0` for success, `1` for a write error, and `2` for an
`-if` condition that rejected every file. Observed behaviour on this build:

| Situation | Exit |
|-----------|------|
| Read succeeds | 0 |
| Write succeeds | 0 |
| Write error (`/etc/hostname`) | 1 |
| `-if` false for all files | 2 |
| Missing path | **0** |
| Non-existent file passed with `-json=` import | 0 (`No file specified`) |
| Unsupported option | 0 (treated as a tag name) |

**Missing paths exit 0.** A typo in a batch script therefore looks like
success. Check the `N image files read` line, use `-if`, or pre-validate with
`test -e`.

## Recipes

**1. Dump every tag with its group and raw name.**

```bash
exiftool -G1 -a -s image.jpg
```

**2. Machine-readable record for one file.**

```bash
exiftool -json -G1 -a image.jpg
```

**3. Machine-readable record for a tree.**

```bash
exiftool -json -r -ext jpg -ext png photos/ > photos.json
```

**4. Find files that carry GPS coordinates.**

```bash
exiftool -r -if '$GPSLatitude' -p '$Directory/$FileName' photos/
```

**5. Strip all metadata, keeping a backup.**

```bash
exiftool -all= image.jpg
```

**6. Strip all metadata without a backup, after testing on a copy.**

```bash
exiftool -all= -overwrite_original image.jpg
```

**7. Copy the artist and copyright from a template.**

```bash
exiftool -TagsFromFile template.jpg -Artist -Copyright -overwrite_original image.jpg
```

**8. Set a capture date without changing the file mtime.**

```bash
exiftool -P -DateTimeOriginal='2024:05:01 12:00:00' image.jpg
```

**9. Write a tag from a text file.**

```bash
printf 'Alice\n' > artist.txt
exiftool '-Artist<=artist.txt' image.jpg
```

The quotes around the whole argument are required: the shell would otherwise
treat `<` as a redirection operator.

**10. Validate and show only problems.**

```bash
exiftool -validate -warning -error image.jpg
```

**11. Compare two files' metadata.**

```bash
exiftool -diff a.jpg b.jpg
```

**12. Export a CSV manifest and re-import it.**

```bash
exiftool -csv -r photos/ > manifest.csv
exiftool -csv=manifest.csv -overwrite_original photos/
```

**13. Persistent process for many operations.**

```bash
exiftool -stay_open True -@ - <<'EOF'
-Artist
image1.jpg
-execute
-Comment
image2.jpg
-execute
-stay_open
False
EOF
```

**14. Reuse a long argument list.**

```bash
printf '%s\n' -Artist -Comment > args.txt
exiftool -@ args.txt image.jpg
```

## Quirks and limits

- **`-help` is not help.** It prints `No file specified`. Use `man exiftool`.
- **Unrecognised options become tag names.** `--output=json` is read as a tag
  request, not an error.
- **Missing files exit 0.** See the exit-status table.
- **`-htmldump` writes to stdout.** `-o file.html` with `-htmldump` fails with
  `Can't write HTML files`; redirect stdout instead.
- **`-json=` and `-csv=` imports need explicit file arguments.** Passing only
  the option prints `No file specified`; add the target files.
- **`-config /dev/null` warns** (`Config file is not found`); use a real empty
  file or omit it.
- **`-delete_original` prompts** unless `!` is appended. In a non-interactive
  shell it can fail or hang.
- **`-all=` does not remove everything.** It removes writable groups listed by
  `-listd`; filesystem timestamps, some MakerNotes, and embedded previews can
  survive.
- **`-fast3` can miss tags.** It stops after the first block; use it only when
  speed matters more than completeness.
- **Writing is not atomic across a crash.** A full disk can leave a temp file
  and an untouched original; check the exit status and the `_original` file.
- **`-overwrite_original` is irreversible.** Test on a copy first.
- **PDF writing is limited.** ExifTool can update some PDF metadata, but it
  does not rewrite arbitrary PDF structure; encrypted PDFs need `-password`.
- **`-listx` is ~497k lines.** Redirect it.

## Related tools

- [`mat2`](../mat2/) — metadata anonymisation with a narrower, safer surface.
- [`binwalk`](../binwalk/) — signature scanning and extraction, including
  embedded files that may carry metadata.
- [`ghidra`](../ghidra/) — reverse engineering of executables and firmware.
- `file`, `mediainfo`, `pdfinfo`, `identify` — format-specific inspectors.
- `jq` — process `exiftool -json` output.
