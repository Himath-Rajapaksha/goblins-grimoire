# binwalk — firmware signature scanner and extractor

- **Version:** 2.4.3 (Debian `binwalk` `2.4.3+dfsg1-2+deb13u1`)
- **Binary:** `/usr/bin/binwalk`
- **Python module:** `/usr/lib/python3/dist-packages/binwalk`
- **Magic signatures:** `/usr/lib/python3/dist-packages/binwalk/magic/` (27 files)
- **Extraction rules:** `/usr/lib/python3/dist-packages/binwalk/config/extract.conf`
- **Help snapshot:** [`help.txt`](help.txt) — `binwalk --help`
- **Man page:** `man binwalk`
- **Upstream (banner):** https://github.com/OSPG/binwalk
- **v3 rewrite:** https://github.com/ReFirmLabs/binwalk (Rust; different internals)
- **Licence:** MIT (see `/usr/share/doc/binwalk/copyright`)

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

> **EOL notice:** the installed v2.4.3 prints `Binwalk v2.x will reach EOL in
> 12/12/2025. Please migrate to binwalk v3.x`. The v3 command line is similar
> but not identical; treat v2 recipes as version-specific.

## What it does

binwalk scans a file for known byte signatures — filesystem headers,
compression streams, executable headers, bootloaders, archive formats — and
reports each hit as an offset plus a description. It is the standard first
pass on an unknown firmware image because it answers three questions at once:

1. **What is inside?** — gzip, LZMA, Squashfs, JFFS2, UBIFS, Cramfs, ext,
   cpio, tar, ELF, uImage, DTB, and hundreds of other signatures.
2. **Where does it start?** — decimal and hexadecimal offsets, optionally
   rebased to a firmware load address with `-O`.
3. **Can it be extracted?** — `-e` runs the matching external utility (or an
   internal plugin) on each carved region.

The scan is signature-based, not semantic. A compressed blob with no recognisable
header is invisible to it, and a false positive is possible; entropy analysis
(`-E`) and the binary-diff mode (`-W`) exist to cover those gaps.

## Install layout and external tools

binwalk is a Python package. Extraction is delegated to programs in `PATH`,
so the useful question is not "is binwalk installed" but "which extractors are
installed".

Present on this host:

```text
7z 7za unsquashfs lzop gzip xz bzip2 zstd cpio ar
```

Missing on this host (the corresponding filesystems cannot be auto-extracted):

```text
sasquatch jefferson ubireader_extract_images ubireader_extract_files
cramfsck cramfsswap dtc tsk_recover unrar cabextract unzip jar
```

The default rules in `extract.conf` cover gzip, LZMA, xz, bzip2, compress,
tar, rar, arj, lha, ISO, CAB, DMG, LZO, Intel HEX, Motorola S-record, ZIP,
Squashfs, Cramfs, ext, ROMFS, JFFS2, UBIFS, and YAFFS. Some rules try several
utilities in order; a missing one simply fails that rule.

## Basic syntax

```bash
binwalk firmware.bin                 # signature scan (default action)
binwalk -B firmware.bin              # explicit signature scan
binwalk -e firmware.bin              # scan and extract
binwalk -e -M firmware.bin           # recursive extraction
binwalk -E firmware.bin              # entropy analysis
binwalk -W old.bin new.bin           # hexdump diff
```

`binwalk` with no action flag performs `-B`. Multiple input files are
accepted. Extraction writes to `_<basename>.extracted/` in the current
directory unless `-C` says otherwise.

## Flag reference

### Signature scanning

| Flag | Meaning |
|------|---------|
| `-B`, `--signature` | Scan for common file signatures (default) |
| `-R <str>`, `--raw=<str>` | Scan for a literal byte sequence, e.g. `'\x1f\x8b\x08'` |
| `-A`, `--opcodes` | Scan for common executable opcode signatures |
| `-m <file>`, `--magic=<file>` | Use a custom libmagic-style signature file |
| `-b`, `--dumb` | Disable smart signature keywords |
| `-I`, `--invalid` | Show results normally marked invalid |
| `-x <str>`, `--exclude=<str>` | Drop results whose description matches the regex |
| `-y <str>`, `--include=<str>` | Keep only results whose description matches the regex |

### Extraction and carving

| Flag | Meaning |
|------|---------|
| `-e`, `--extract` | Automatically extract known file types |
| `-D <type[:ext[:cmd]]>` | Custom rule: match `<type>`, give extension `<ext>`, run `<cmd>` (`%e` = carved file) |
| `-M`, `--matryoshka` | Recursively scan extracted files |
| `-d <int>`, `--depth=<int>` | Recursion depth for `-M` (default 8) |
| `-C <dir>`, `--directory=<dir>` | Extraction directory (default: current directory) |
| `-j <int>`, `--size=<int>` | Maximum size of each extracted file |
| `-n <int>`, `--count=<int>` | Maximum number of extracted files |
| `-0 <user>`, `--run-as=<user>` | Run external extractors as `<user>` (root only when switching users) |
| `-1`, `--preserve-symlinks` | Keep extracted symlinks that point outside the extraction directory (**dangerous**) |
| `-r`, `--rm` | Delete carved files after extraction |
| `-z`, `--carve` | Carve but do not execute extraction utilities; use together with `-e` |
| `-V`, `--subdirs` | Put each result in a subdirectory named by its offset |

### Entropy

| Flag | Meaning |
|------|---------|
| `-E`, `--entropy` | Calculate Shannon entropy per block |
| `-F`, `--fast` | Faster, less detailed entropy analysis |
| `-J`, `--save` | Save the entropy plot as a PNG |
| `-Q`, `--nlegend` | Omit the plot legend |
| `-N`, `--nplot` | Do not generate a plot (headless-safe) |
| `-H <float>`, `--high=<float>` | Rising-edge threshold (default 0.95) |
| `-L <float>`, `--low=<float>` | Falling-edge threshold (default 0.85) |

### Binary diffing

| Flag | Meaning |
|------|---------|
| `-W`, `--hexdump` | Hexdump or diff one or more files |
| `-G`, `--green` | Show only bytes identical in all files |
| `-i`, `--red` | Show only bytes different in all files |
| `-U`, `--blue` | Show only bytes different in some files |
| `-u`, `--similar` | Show only lines identical between all files |
| `-w`, `--terse` | Diff all files but hexdump only the first |

### Raw compression streams

| Flag | Meaning |
|------|---------|
| `-X`, `--deflate` | Scan for raw deflate streams |
| `-Z`, `--lzma` | Scan for raw LZMA streams |
| `-P`, `--partial` | Superficial but faster scan |
| `-S`, `--stop` | Stop after the first result |

### General

| Flag | Meaning |
|------|---------|
| `-l <int>`, `--length=<int>` | Number of bytes to scan |
| `-o <int>`, `--offset=<int>` | Start scanning at this file offset |
| `-O <int>`, `--base=<int>` | Add a base address to printed offsets |
| `-K <int>`, `--block=<int>` | File block size |
| `-g <int>`, `--swap=<int>` | Reverse every `n` bytes before scanning |
| `-f <file>`, `--log=<file>` | Log results to a file |
| `-c`, `--csv` | Log results as CSV |
| `-t`, `--term` | Format output to the terminal width |
| `-q`, `--quiet` | Suppress stdout |
| `-v`, `--verbose` | Verbose output |
| `-a <regex>`, `--finclude=<regex>` | Only scan files whose names match |
| `-p <regex>`, `--fexclude=<regex>` | Skip files whose names match |
| `-s <port>`, `--status=<port>` | Start the status server on a TCP port |

## Recipes

**1. Identify an unknown firmware image.**

```bash
binwalk firmware.bin
```

**2. Extract everything it recognises, into a named directory.**

```bash
binwalk -e -C /tmp/firmware-extract firmware.bin
```

**3. Recurse into extracted archives and filesystems.**

```bash
binwalk -e -M -d 3 firmware.bin
```

**4. Carve without running third-party tools.**

```bash
binwalk -e -z firmware.bin
```

**5. Keep each result in its own offset-named directory.**

```bash
binwalk -e -V firmware.bin
```

**6. Scan only a region, rebased to the load address.**

```bash
binwalk -o 0x1000 -l 0x20000 -O 0x8000000 firmware.bin
```

**7. Look for a raw gzip header.**

```bash
binwalk -R '\x1f\x8b\x08' firmware.bin
```

**8. Keep only filesystem hits.**

```bash
binwalk -y 'filesystem' firmware.bin
```

**9. Drop compression hits to reduce noise.**

```bash
binwalk -x 'compressed data' firmware.bin
```

**10. Entropy pass without a plot (safe over SSH/CI).**

```bash
MPLBACKEND=Agg binwalk -E -N firmware.bin
```

**11. Save the entropy plot.**

```bash
MPLBACKEND=Agg binwalk -E -J firmware.bin
```

**12. Diff two firmware versions.**

```bash
binwalk -W v1.bin v2.bin
```

**13. Show only bytes that differ in every file.**

```bash
binwalk -W -i v1.bin v2.bin v3.bin
```

**14. Custom extraction rule for a format binwalk does not know.**

```bash
binwalk -D 'myformat:bin:my-extractor %e' firmware.bin
```

**15. Run extraction utilities as a non-root user when binwalk itself is root.**

```bash
sudo binwalk -e --run-as=anorak firmware.bin
```

**16. Log a scan as CSV.**

```bash
binwalk -c -f scan.csv firmware.bin
```

## Gotchas

- **v2 is EOL.** The binary itself prints the 2025-12-12 end-of-life notice.
  v3 is a Rust rewrite; scripts that parse v2 output may need changes.
- **`-z` alone does nothing.** `--carve` only changes extraction behaviour; it
  must be combined with `-e`. `binwalk -z file` still performs a plain scan.
- **Root cannot extract without `--run-as`.** When binwalk runs as UID 0 and
  `--run-as` is absent, it refuses to execute external extractors. Use
  `--run-as=<user>` or run binwalk as the unprivileged user.
- **`--run-as` to another user needs root.** Switching UID requires privilege;
  the same flag run unprivileged can only name the current user.
- **`-1` is dangerous.** Preserving symlinks lets a malicious archive create a
  link that points outside the extraction directory; only use it on trusted
  images in a disposable VM.
- **Extraction executes third-party parsers on untrusted input.** A crafted
  firmware can exploit `unsquashfs`, `7z`, `tar`, or any other extractor.
  Extract in a container or VM with no network and no host mounts.
- **The output directory is `_<basename>.extracted`.** It is created in the
  current working directory, not next to the input file. Use `-C` in scripts.
- **`-M` defaults to depth 8.** Deeply nested images can explode into many
  files; pair `-M` with `-j` and `-n`.
- **`-r` deletes the carved intermediate files, not the extracted output.**
  It is a disk-space measure, not a cleanup of the results.
- **`-V` changes the layout.** Each hit goes into an offset-named subdirectory,
  which is easier to audit but breaks rules that expect a flat output tree.
- **`-O` only changes printed offsets.** It does not relocate the data or
  change extraction; it is a presentation aid for load-address arithmetic.
- **`-l` and `-o` are byte counts/offsets.** A signature that straddles the
  boundary is missed; leave slack around the region you care about.
- **`-g` reverses bytes before scanning.** Useful for endian-swapped images,
  but it changes every offset interpretation; record the swap width.
- **Entropy plotting can hang headless.** Set `MPLBACKEND=Agg` or pass `-N`.
  The scan itself is fast; the matplotlib backend is what blocks.
- **`-E` reports edges, not a verdict.** A high-entropy block may be
  compressed, encrypted, or simply random; a low-entropy block may still be
  compressed with a weak ratio.
- **`-W` only renders on a TTY.** The diff module filters lines by their ANSI
  colour codes, and colour is disabled when stdout is a pipe or file. Piped
  output collapses to `*`; run it interactively or wrap it in
  `script -qec "binwalk -W a b" /dev/null`.
- **`-s` opens a TCP status server.** It is a debugging aid, not a hardened
  service; never expose it beyond loopback.
- **Signature false positives are normal.** A short magic can match inside
  unrelated data; confirm with `-I`, a hexdump, and the extracted result.
- **`-a`/`-p` filter file names, not contents.** They are useful when scanning
  a directory of samples, not when scanning one image.
- **CSV output is not a stable schema.** Column order and quoting have changed
  across versions; parse defensively.

## Related docs

- [`ghidra`](../ghidra/README.md) — static analysis of extracted executables
- [`checksec`](../checksec/README.md) — hardening of extracted ELF files
- [`strace`](../strace/README.md) — runtime behaviour of extracted binaries
- [`exiftool`](../exiftool/README.md) — metadata in extracted images and documents
- [`volatility3`](../volatility3/README.md) — memory images rather than firmware
- [`trivy`](../trivy/README.md) — vulnerability scan of extracted filesystems
- [`mat2`](../mat2/README.md) — metadata cleanup of extracted documents
