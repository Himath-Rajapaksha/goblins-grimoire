# mat2 — metadata anonymisation toolkit

- **Version:** 0.13.5 (Debian `mat2` `0.13.5-1.2`)
- **Binary:** `/usr/bin/mat2` (Python 3)
- **Library:** `/usr/lib/python3/dist-packages/libmat2/`
- **Supported formats:** 36 MIME types (`mat2 --list`)
- **Help snapshot:** [`help.txt`](help.txt) — `mat2 --help`
- **Upstream:** https://0xacab.org/jvoisin/mat2
- **Documentation:** `/usr/share/doc/mat2/` (`README.md`, `implementation_notes.md`, `threat_model.md`)
- **Licence:** LGPL-3.0-or-later (see `/usr/share/doc/mat2/copyright`)

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

> **Output-name warning:** the default output is a **new file** named
> `<stem>.cleaned<ext>` next to the input; the input is not modified.
> `--inplace` renames the cleaned file over the input **without a backup**.
> The output name is predictable, which creates a symlink race — see
> "Security model" below.

## What it does

mat2 removes metadata from files by parsing each supported format, dropping
the metadata structures it knows about, and writing a cleaned copy. It is
deliberately narrower than ExifTool: it targets the metadata that commonly
identifies a person or device, not every tag in every container.

It handles images (JPEG, PNG, GIF, TIFF, BMP, PPM, HEIC), audio (MP3, FLAC,
OGG/Opus, WAV, AIFF), video (MP4, AVI, WMV), documents (PDF, EPUB, Office
Open XML, OpenDocument), archives (ZIP, TAR), HTML/CSS, SVG, and plain text.

The design goal is *anonymisation*, not *forensic preservation*. That means:

- PDFs are rasterised, so **text is lost** and the page becomes an image.
- Images are **re-encoded**, so quality changes and pixel data is not
  bit-identical.
- Archives are rebuilt, so member order, timestamps, and unknown members can
  change.
- Unsupported formats are skipped with a message, not converted.

## Supported formats

```bash
mat2 --list
```

The list includes 36 MIME types. The important groups:

| Group | Formats |
|-------|---------|
| Images | JPEG, PNG, GIF, TIFF, BMP, PPM, HEIC |
| Audio | MP3, FLAC, OGG/Opus, WAV, AIFF |
| Video | MP4, AVI, WMV |
| Documents | PDF, EPUB, DOCX, XLSX, PPTX, ODT/ODS/ODP/ODG/ODF/ODC/ODI |
| Archives | ZIP, TAR |
| Web/text | HTML, XHTML, CSS, SVG, plain text |
| Other | Torrent, NCX |

`mat2` uses libmagic-style detection, not the file extension alone. A `.png`
that is really a JPEG is handled as a JPEG; an unsupported container is
reported and skipped.

## Basic syntax

```bash
mat2 file.png                  # writes file.cleaned.png
mat2 --inplace file.png        # replaces file.png, no backup
mat2 --show file.png           # list metadata without removing it
mat2 --lightweight file.png    # remove only superficial metadata
mat2 --unknown-members omit a.zip
mat2 --no-sandbox file.pdf
mat2 --check-dependencies
```

Multiple files and directories are accepted. Directories are walked
recursively; every supported file gets its own `.cleaned` output.

## Options

| Option | Meaning |
|--------|---------|
| `-h`, `--help` | Show help |
| `-v`, `--version` | Show version |
| `-V`, `--verbose` | Verbose status output |
| `-l`, `--list` | List supported formats |
| `--check-dependencies` | Report optional dependency availability |
| `-s`, `--show` | List harmful metadata without removing it |
| `-L`, `--lightweight` | Remove only some metadata |
| `--inplace` | Clean in place, **without backup** |
| `--unknown-members policy` | Archive policy: `abort` (default), `omit`, `keep` |
| `--no-sandbox` | Disable the bubblewrap sandbox |

## Output naming

| Input | Default output | `--inplace` result |
|-------|----------------|--------------------|
| `photo.png` | `photo.cleaned.png` | `photo.png` replaced |
| `report.pdf` | `report.cleaned.pdf` | `report.pdf` replaced |
| `archive.tar` | `archive.cleaned.tar` | `archive.tar` replaced |
| `dir/a.png` | `dir/a.cleaned.png` | `dir/a.png` replaced |
| `dir/sub/b.png` | `dir/sub/b.cleaned.png` | `dir/sub/b.png` replaced |

The cleaned file inherits the input's mode (`shutil.copymode`). The original
keeps its mode and timestamps in the default path. With `--inplace`, the
rename preserves the cleaned file's inode, not the original's.

## What is removed

`mat2 --show` lists the harmful metadata it can detect. The exact set depends
on the format:

| Format | Typical removals |
|--------|------------------|
| JPEG/PNG/TIFF | EXIF, XMP, IPTC, ICC profile, comments, GPS, thumbnails |
| PDF | Author, creator, producer, dates, XMP, embedded thumbnails; page rasterised |
| Office | core/app properties, author, last-modified-by, revision, custom XML |
| Audio | ID3 tags, Vorbis comments, cover art, encoder strings |
| Video | container metadata, creation time, device model, GPS |
| Archives | member timestamps, owner/group names, comments, extra fields |
| HTML/SVG | meta tags, comments, editor markers |

`--lightweight` skips the deeper structural cleanups. For images, the file is
still re-encoded, so it is not a true "metadata-only" mode.

## Security model

mat2's threat model is documented in
`/usr/share/doc/mat2/threat_model.md`. The important operational points:

- **The output name is predictable.** `file.cleaned.png` is created with
  ordinary `open()`. If an attacker can create a symlink at that path, mat2
  writes through it. Demonstrated on this host: a symlink
  `p.cleaned.png -> target.txt` caused mat2 to overwrite `target.txt`.
  Clean untrusted directories in a private directory, or pre-create the output
  with `O_NOFOLLOW`/`mktemp`, or use `--inplace` only after reviewing the
  directory.
- **The sandbox is bubblewrap.** `--no-sandbox` removes the isolation around
  parsers, which matters when processing untrusted files.
- **Parsers are the attack surface.** PDF, Office, and archive parsers process
  hostile input. Run mat2 on untrusted files inside a disposable VM or
  container with no host mounts and no network.
- **It is not an anonymiser for content.** It removes metadata; it does not
  remove watermarks, steganography, visible identifiers, or stylometric
  traces. It also does not change filesystem timestamps (atime/mtime/ctime)
  or the file owner.
- **It does not guarantee a fixed output.** Re-encoding can vary with library
  versions; do not treat the hash as a stable identifier.

## Exit status

| Situation | Exit |
|-----------|------|
| Cleaned successfully | 0 |
| Unsupported file | 0 (message, no output) |
| No arguments | 0 (usage) |
| Missing path | 255 |
| Directory tree | 0 (walked recursively) |
| Fatal top-level error | 1 (observed on `/tmp`, which contains entries the walker cannot handle) |

A directory argument is not an error in normal use: mat2 walks it. The `1`
above is for a path that resolves to a directory when a single file was
expected by the parser layer.

## Recipes

**1. Inspect before cleaning.**

```bash
mat2 --show photo.jpg
```

**2. Clean one file, keeping the original.**

```bash
mat2 photo.jpg
# photo.cleaned.jpg
```

**3. Clean in place after making your own backup.**

```bash
cp photo.jpg photo.jpg.bak
mat2 --inplace photo.jpg
```

**4. Clean a tree, preserving relative paths.**

```bash
mat2 --inplace ~/Documents/reports
find ~/Documents/reports -name '*.cleaned.*' -print
```

**5. Strip a PDF's author and producer (accepting rasterisation).**

```bash
mat2 report.pdf
pdfinfo report.cleaned.pdf | head
```

**6. Clean an archive, aborting on unknown members.**

```bash
mat2 archive.zip
```

**7. Clean an archive, dropping unknown members.**

```bash
mat2 --unknown-members omit archive.zip
```

**8. Keep unknown members but accept the leakage warning.**

```bash
mat2 --unknown-members keep archive.zip
```

**9. Check optional dependencies before relying on a format.**

```bash
mat2 --check-dependencies
```

**10. Disable the sandbox only for a trusted file.**

```bash
mat2 --no-sandbox trusted.pdf
```

**11. Verify the metadata is gone.**

```bash
mat2 photo.jpg
exiftool photo.cleaned.jpg
```

**12. Compare before/after hashes and metadata.**

```bash
sha256sum photo.jpg photo.cleaned.jpg
exiftool -G1 -a -s photo.jpg photo.cleaned.jpg
```

## Quirks and limits

- **No backup with `--inplace`.** The help text says "without backup" and the
  source confirms `os.rename(p.output_filename, filename)`. Copy first.
- **PDFs lose text.** Rasterisation is intentional; if you need searchable
  text, use a different tool and accept that some metadata may remain.
- **Images are re-encoded.** Expect quality/size changes even with
  `--lightweight`.
- **Archives abort on unknown members by default.** `abort` is safe but
  inconvenient; `omit` drops data; `keep` can leak metadata.
- **Unsupported files are silent successes.** Exit 0 with no output; check
  for the `.cleaned` file.
- **Directories are recursive.** There is no `--recursive` flag because it is
  the default for directory arguments.
- **`.cleaned` files are re-processed on a second run.** A directory walk will
  clean `photo.cleaned.png` into `photo.cleaned.cleaned.png` unless you filter.
- **It does not remove all metadata.** Vendor-specific MakerNotes, some XMP
  sidecars, and non-standard chunks may survive. Validate with `exiftool`.
- **It does not anonymise content.** Watermarks, steganography, visible text,
  and writing style remain.
- **It does not change timestamps.** Use `touch`/`touch -d` deliberately if
  that is part of your threat model.
- **Optional dependencies matter.** `--check-dependencies` reports Cairo,
  ExifTool, FFmpeg, GLib, GdkPixbuf, Mutagen, Poppler, and PyGObject; a
  missing one disables a format or a cleanup path.

## Related tools

- [`exiftool`](../exiftool/) — read/write every tag, with backups and
  format-specific control.
- [`binwalk`](../binwalk/) — find embedded files that may carry metadata.
- [`ghidra`](../ghidra/) — inspect executables and firmware that embed
  documents.
- `pdfinfo`, `qpdf`, `ocrmypdf` — PDF-specific inspection and rewriting.
- `ffmpeg` — re-encode audio/video with explicit metadata control.
