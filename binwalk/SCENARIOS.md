# binwalk — command scenarios for deep research

Installed build: binwalk 2.4.3 (Debian `2.4.3+dfsg1-2+deb13u1`). Every flag
below appears in [`help.txt`](help.txt); extraction behaviour is defined by
`/usr/lib/python3/dist-packages/binwalk/config/extract.conf`. The binary
prints an end-of-life notice for v2 on 2025-12-12.

> **Scope:** Firmware images may contain third-party code, personal data, or
> material you are not licensed to extract. Work on images you own or are
> authorised to assess, and extract untrusted input in a disposable VM or
> container with no host mounts and no network.

> **Root note:** when binwalk runs as UID 0, `-e` refuses to execute external
> extractors unless `--run-as=<user>` is given. The recipes below assume a
> normal user shell; the root variants are shown explicitly.

## How to invoke

```text
@deep-research For an authorised firmware assessment with binwalk 2.4.3,
explain how signature scanning, extraction rules, recursive matryoshka
extraction, and entropy analysis interact. Compare the built-in extractor
plugins with the external utilities in extract.conf, and explain what
`-e`, `-z`, `-M`, `-V`, `-r`, `-j`, and `-n` change on disk. Cite the
binwalk man page, the shipped extract.conf, the upstream v2 documentation,
and the v3 migration notes. Separate confirmed behaviour from inference and
state confidence.
```

Create the baseline and the sample image:

```bash
mkdir -p ~/binwalk-demo/evidence
cd ~/binwalk-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
printf 'HELLO' > plain.txt
gzip -c plain.txt > plain.gz
cat plain.txt plain.gz > sample.bin
sha256sum sample.bin > evidence/sample.sha256
```

## Scenario 1 — version, layout, magic inventory, extractor availability

binwalk is a Python package plus a set of external programs. Record both
before trusting an extraction result.

| # | Command | How to use |
|---|---------|------------|
| 1 | `binwalk --help \| head -4` | Version banner and upstream URL |
| 2 | `binwalk --help \| tail -2` | End-of-life notice |
| 3 | `dpkg-query -W binwalk` | Debian package revision |
| 4 | `python3 -c 'import binwalk; print(binwalk.__version__, binwalk.__file__)'` | Module version and path |
| 5 | `ls /usr/lib/python3/dist-packages/binwalk/magic` | Signature files shipped |
| 6 | `ls /usr/lib/python3/dist-packages/binwalk/magic \| wc -l` | Signature-file count |
| 7 | `grep -c '^\^' /usr/lib/python3/dist-packages/binwalk/config/extract.conf` | Extraction-rule count |
| 8 | `sed -n '1,25p' /usr/lib/python3/dist-packages/binwalk/config/extract.conf` | Rule syntax and placeholders |
| 9 | `for b in 7z unsquashfs sasquatch jefferson ubireader_extract_files cramfsck cramfsswap dtc tsk_recover unrar cabextract unzip jar; do printf '%-26s %s\n' "$b" "$(command -v "$b" \|\| echo missing)"; done` | Extractor inventory |
| 10 | `man binwalk \| col -b \| head -40` | Installed man page |
| 11 | `dpkg -L binwalk \| head -30` | Package file layout |
| 12 | `binwalk --help \| grep -cE '^    -'` | Count of help option lines |

**Research questions**
- Which extraction rules depend on utilities that are not installed here?
- Does a missing utility fail the whole extraction or only that rule?
- Which formats are handled internally by plugins rather than external tools?
- Is the installed man page generated from the same version as the binary?
- Which magic file would you customise to add a proprietary header?

**Sources:** `binwalk --help`; `dpkg -L binwalk`;
`binwalk/config/extract.conf`; `man binwalk`.
**Output:** an install record with the extractor gap list.

## Scenario 2 — signature scan basics

The default action is a signature scan. These commands establish the output
shape before any extraction.

| # | Command | How to use |
|---|---------|------------|
| 13 | `cd ~/binwalk-demo` | Work in the demo directory |
| 14 | `binwalk sample.bin` | Default signature scan |
| 15 | `binwalk -B sample.bin` | Explicit signature scan |
| 16 | `binwalk -v sample.bin` | Verbose header with MD5 and signature count |
| 17 | `binwalk -t sample.bin` | Fit output to the terminal width |
| 18 | `binwalk -q sample.bin` | Suppress stdout |
| 19 | `binwalk -f scan.txt sample.bin` | Write the text log |
| 20 | `cat scan.txt` | Read the log |
| 21 | `binwalk -c -f scan.csv sample.bin` | Write CSV |
| 22 | `cat scan.csv` | Inspect the CSV columns |
| 23 | `binwalk sample.bin plain.gz` | Scan two files in one run |
| 24 | `binwalk plain.txt` | A file with no known signature |
| 25 | `binwalk /bin/ls \| head -12` | A normal ELF binary |
| 26 | `binwalk -v /bin/ls \| head -10` | Verbose ELF scan header |

**Research questions**
- Which line of the default output is the machine-parseable result?
- Does `-q` suppress the log file as well as stdout?
- Is the CSV schema stable across binwalk versions?
- How does a file with no signature differ from one with an invalid signature?
- Does `-v` change the scan or only the header?

**Sources:** `man binwalk`; `binwalk --help`; observed output.
**Output:** a scan record with text and CSV artefacts.

## Scenario 3 — filters, raw signatures, custom magic, opcodes

Filters operate on the description text, not on the file contents. Raw and
custom-magic scans cover signatures that are not in the default database.

| # | Command | How to use |
|---|---------|------------|
| 27 | `binwalk -y gzip sample.bin` | Keep only gzip hits |
| 28 | `binwalk -x gzip sample.bin` | Drop gzip hits |
| 29 | `binwalk -y 'compressed' sample.bin` | Keep compression hits |
| 30 | `binwalk -x 'compressed' sample.bin` | Drop compression hits |
| 31 | `binwalk -R '\x1f\x8b\x08' sample.bin` | Raw gzip magic |
| 32 | `binwalk -R '\x48\x45\x4c\x4c\x4f' sample.bin` | Raw `HELLO` sequence |
| 33 | `binwalk -A /bin/ls` | Opcode signature scan |
| 34 | `binwalk -A -v /bin/ls` | Opcode scan with header |
| 35 | `binwalk -I sample.bin` | Include invalid results |
| 36 | `binwalk -b sample.bin` | Disable smart keywords |
| 37 | `binwalk -m /usr/lib/python3/dist-packages/binwalk/magic/compressed sample.bin` | Scan with only the compression magic file |
| 38 | `binwalk -m /usr/lib/python3/dist-packages/binwalk/magic/filesystems sample.bin` | Scan with only the filesystem magic file |
| 39 | `binwalk -y 'gzip' -y 'deflate' sample.bin` | Repeat a filter (last one wins?) |
| 40 | `binwalk -R '\x00\x00\x00\x00' sample.bin` | A common but weak sequence |

**Research questions**
- Are `-x` and `-y` regular expressions or literal substrings?
- Does repeating `-y` combine filters or replace the previous one?
- Why does `-A` return nothing on a normal ELF binary?
- Does `-m` replace the default magic set or add to it?
- What makes a raw signature too short to be useful?

**Sources:** `man binwalk`; `binwalk/magic/*`; observed output.
**Output:** a filter table with the hit count per filter.

## Scenario 4 — offsets, base addresses, block size, byte swap

These flags change which bytes are scanned and how offsets are printed. They
do not change the file itself.

| # | Command | How to use |
|---|---------|------------|
| 41 | `binwalk -o 5 sample.bin` | Start at the gzip magic |
| 42 | `binwalk -o 6 sample.bin` | Start one byte late and miss it |
| 43 | `binwalk -O 0x8000000 sample.bin` | Rebase printed offsets |
| 44 | `binwalk -O 0x08048000 /bin/ls \| head -8` | Rebase an ELF scan |
| 45 | `binwalk -K 16 sample.bin` | 16-byte scan blocks |
| 46 | `binwalk -K 4096 /bin/ls \| head -8` | 4 KiB scan blocks |
| 47 | `binwalk -g 2 sample.bin` | Reverse every 2 bytes before scanning |
| 48 | `binwalk -g 4 sample.bin` | Reverse every 4 bytes |
| 49 | `binwalk -l 0x100 sample.bin` | Scan 256 bytes |
| 50 | `binwalk -l 4 -o 5 sample.bin` | Signature starts at the boundary |
| 51 | `binwalk -O 0 -o 5 sample.bin` | Explicit zero base |
| 52 | `binwalk -K 1 sample.bin` | One-byte blocks (slow, noisy) |

**Research questions**
- Does `-O` affect extraction offsets or only the display?
- Is `-l` a decimal byte count or does it accept `0x` prefixes?
- What happens to a signature that straddles the `-l` boundary?
- Does `-g` change the reported offset or only the compared bytes?
- Why can a smaller `-K` change which signatures are found?

**Sources:** `man binwalk`; observed output; upstream scan internals.
**Output:** an offset table with the exact byte ranges scanned.

## Scenario 5 — extraction and carving

Extraction writes into `_<basename>.extracted/`. The `-z` flag only changes
behaviour when `-e` is also present.

| # | Command | How to use |
|---|---------|------------|
| 53 | `binwalk -e sample.bin` | Scan and extract |
| 54 | `find _sample.bin.extracted -maxdepth 2` | Inspect the output tree |
| 55 | `binwalk -e -C /tmp/binwalk-out sample.bin` | Extract into a named directory |
| 56 | `find /tmp/binwalk-out -maxdepth 2` | Inspect the custom directory |
| 57 | `binwalk -e -V sample.bin` | Offset-named subdirectories |
| 58 | `find _sample.bin.extracted -maxdepth 2` | Inspect the subdirectory layout |
| 59 | `binwalk -e -z sample.bin` | Carve but do not run extractors |
| 60 | `find _sample.bin.extracted -maxdepth 2` | Compare the carve-only tree |
| 61 | `binwalk -e -r sample.bin` | Delete carved files after extraction |
| 62 | `find _sample.bin.extracted -maxdepth 2` | Confirm the carved files are gone |
| 63 | `binwalk -e -j 16 sample.bin` | Cap each extracted file at 16 bytes |
| 64 | `binwalk -e -n 1 sample.bin` | Extract at most one file |
| 65 | `binwalk -D 'gzip:gz:gunzip -k %e' sample.bin` | Custom extraction rule |
| 66 | `find _sample.bin.extracted -maxdepth 1` | Inspect the custom-rule output |
| 67 | `sudo binwalk -e --run-as="$SUDO_USER" sample.bin` | Root shell, drop to the login user |
| 68 | `binwalk -e -1 sample.bin` | Preserve symlinks (dangerous; trusted input only) |

**Research questions**
- Does `-z` alone create any files, and why?
- What is the difference between the carved file and the extractor output?
- Does `-j` truncate the carved file or the extracted file?
- Does `-n` count hits or extracted files?
- What does `%e` expand to in a custom rule, and what does `%%` do?
- Why does `-1` create a path-traversal risk?

**Sources:** `binwalk/config/extract.conf`; `man binwalk`; observed output.
**Output:** an extraction record with before/after file lists and sizes.

## Scenario 6 — recursive extraction and resource limits

`-M` re-scans extracted files. It is powerful and easy to make expensive.

| # | Command | How to use |
|---|---------|------------|
| 69 | `mkdir -p nested && cp sample.bin nested/` | Nested sample tree |
| 70 | `binwalk -e -M nested/sample.bin` | Recursive extraction |
| 71 | `find _sample.bin.extracted -maxdepth 4` | Inspect the recursion result |
| 72 | `binwalk -e -M -d 1 nested/sample.bin` | Limit recursion to one level |
| 73 | `find _sample.bin.extracted -maxdepth 3` | Compare the shallower tree |
| 74 | `binwalk -e -M -d 8 nested/sample.bin` | Default depth explicitly |
| 75 | `binwalk -e -M -j 1024 nested/sample.bin` | Size cap during recursion |
| 76 | `binwalk -e -M -n 5 nested/sample.bin` | Count cap during recursion |
| 77 | `binwalk -e -M -V nested/sample.bin` | Offset subdirectories at every level |
| 78 | `find _sample.bin.extracted -maxdepth 3` | Inspect the layout |
| 79 | `du -sh _sample.bin.extracted` | Disk consumed |
| 80 | `binwalk -e -M -C /tmp/binwalk-rec nested/sample.bin` | Recursion into a named directory |

**Research questions**
- Does `-d 0` mean unlimited or no recursion?
- Is `-j` applied per file or per extraction round?
- Can `-n` stop a recursion before the interesting layer?
- Does `-V` compose with `-M`, or does the layout collide?
- How much disk can a small image consume at depth 8?

**Sources:** `man binwalk`; observed output; upstream matryoshka notes.
**Output:** a recursion record with depth, file count, and disk use.

## Scenario 7 — entropy and raw compression streams

Entropy shows where the image stops looking like code or text and starts
looking compressed or encrypted. Raw stream scans are heuristic.

| # | Command | How to use |
|---|---------|------------|
| 81 | `MPLBACKEND=Agg binwalk -E -N sample.bin` | Entropy without a plot |
| 82 | `MPLBACKEND=Agg binwalk -E -F -N sample.bin` | Fast entropy |
| 83 | `MPLBACKEND=Agg binwalk -E -N -H 0.9 -L 0.8 sample.bin` | Custom edge thresholds |
| 84 | `MPLBACKEND=Agg binwalk -E -J sample.bin` | Save the entropy plot |
| 85 | `ls -l sample.bin.png` | Confirm the PNG |
| 86 | `MPLBACKEND=Agg binwalk -E -J -Q sample.bin` | Plot without a legend |
| 87 | `binwalk -X sample.bin` | Raw deflate scan |
| 88 | `binwalk -Z sample.bin` | Raw LZMA scan |
| 89 | `binwalk -X -P sample.bin` | Superficial deflate scan |
| 90 | `binwalk -X -S sample.bin` | Stop after the first deflate hit |
| 91 | `binwalk -Z -P -S sample.bin` | Fast LZMA scan, first hit |
| 92 | `head -c 4096 /dev/urandom > random.bin` | High-entropy input |
| 93 | `MPLBACKEND=Agg binwalk -E -N random.bin` | Entropy of random data |
| 94 | `printf 'A%.0s' {1..4096} > zeros.bin` | Low-entropy input |
| 95 | `MPLBACKEND=Agg binwalk -E -N zeros.bin` | Entropy of zeros |
| 96 | `binwalk -X random.bin \| head -8` | False positives on random data |

**Research questions**
- Why does `-E` report edges rather than a single entropy value?
- How do `-H` and `-L` change which edges are printed?
- Why does `-X` find deflate streams inside a gzip file?
- Does `-S` stop the whole scan or only the current signature type?
- What does a high-entropy block tell you, and what does it not tell you?

**Sources:** `man binwalk`; Shannon entropy definition; observed output.
**Output:** an entropy profile and a raw-stream hit list with a false-positive
note.

## Scenario 8 — binary diffing, logging, status server, cleanup

The diff mode is colour-driven and only renders correctly on a TTY. The status
server is a debugging aid, not a service to expose.

| # | Command | How to use |
|---|---------|------------|
| 97 | `cp /bin/ls ls-a.bin && cp /bin/ls ls-b.bin` | Two identical binaries |
| 98 | `printf 'PATCH' \| dd of=ls-b.bin bs=1 seek=200 conv=notrunc status=none` | Patch one byte range |
| 99 | `script -qec "binwalk -W ls-a.bin ls-b.bin" /dev/null \| head -6` | Full hexdump diff under a PTY |
| 100 | `script -qec "binwalk -W -i ls-a.bin ls-b.bin" /dev/null \| head -8` | Only bytes different in all files |
| 101 | `script -qec "binwalk -W -G ls-a.bin ls-b.bin" /dev/null \| head -8` | Only bytes the same in all files |
| 102 | `script -qec "binwalk -W -U ls-a.bin ls-b.bin" /dev/null \| head -8` | Bytes different in some files |
| 103 | `script -qec "binwalk -W -u ls-a.bin ls-b.bin" /dev/null \| head -8` | Lines identical between all files |
| 104 | `script -qec "binwalk -W -w ls-a.bin ls-b.bin" /dev/null \| head -8` | Terse: dump only the first file |
| 105 | `binwalk -f full.log sample.bin` | Text log |
| 106 | `binwalk -c -f full.csv sample.bin` | CSV log |
| 107 | `head -c 20000000 /dev/urandom > big.bin && binwalk -s 8099 -K 1 big.bin & sleep 1` | Start the status server on a slow scan |
| 108 | `ss -ltnp \| grep 8099` | Confirm the loopback listener |
| 109 | `binwalk -a '.*\.bin' sample.bin ls-a.bin` | Scan only names matching `.bin` |
| 110 | `binwalk -p '.*\.bin' sample.bin ls-a.bin` | Skip names matching `.bin` |
| 111 | `rm -rf _sample.bin.extracted /tmp/binwalk-out /tmp/binwalk-rec` | Remove extraction output |
| 112 | `rm -rf ~/binwalk-demo` | Remove the demo tree |

**Research questions**
- Why does piped `binwalk -W` output collapse to `*`?
- Which diff filter is useful for finding a single patched byte?
- Does the status server expose file contents or only progress?
- Are `-a` and `-p` applied before or after the file is opened?
- What evidence proves the demo files and listeners are gone?

**Sources:** `binwalk/modules/hexdiff.py`; `man binwalk`; observed output.
**Output:** a diff record, a log pair, and a cleanup check.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `-B`, `-R`, `-A`, `-m`, `-b`, `-I`, `-x`, `-y` | 3 |
| `-e`, `-D`, `-M`, `-d`, `-C`, `-j`, `-n`, `-0`, `-1`, `-r`, `-z`, `-V` | 5, 6 |
| `-E`, `-F`, `-J`, `-Q`, `-N`, `-H`, `-L` | 7 |
| `-W`, `-G`, `-i`, `-U`, `-u`, `-w` | 8 |
| `-X`, `-Z`, `-P`, `-S` | 7 |
| `-l`, `-o`, `-O`, `-K`, `-g` | 4 |
| `-f`, `-c`, `-t`, `-q`, `-v`, `-a`, `-p`, `-s` | 2, 8 |
| `-h`, `--help` | 1 |

Full text: [`help.txt`](help.txt) and `man binwalk`.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, layout, magic inventory, extractor availability | 12 |
| 2 | Signature scan basics | 14 |
| 3 | Filters, raw signatures, custom magic, opcodes | 14 |
| 4 | Offsets, base addresses, block size, byte swap | 12 |
| 5 | Extraction and carving | 16 |
| 6 | Recursive extraction and resource limits | 12 |
| 7 | Entropy and raw compression streams | 16 |
| 8 | Binary diffing, logging, status server, cleanup | 16 |
| **Total** | | **112** |

## Legal and safety

Firmware frequently contains third-party binaries, credentials, personal data,
and code under licences that do not permit redistribution. Extracted files are
derived works; keep them inside the engagement boundary and do not publish
them. Extraction runs third-party parsers on untrusted input, so use a
disposable VM or container with no host mounts and no network. `-1` and
`-s` are both dangerous in their own way: the first can write outside the
extraction directory through a crafted symlink, the second opens a TCP
listener. Delete the demo tree and any extracted output when finished.
