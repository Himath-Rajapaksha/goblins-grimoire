# crunch — deterministic wordlist generator

- **Version:** 3.6
- **Binary:** `/usr/bin/crunch` (Debian package `crunch`)
- **Help snapshot:** [`help.txt`](help.txt) (short banner; the man page is the
  authoritative flag reference on this host)
- **Charset file:** `/usr/share/crunch/charset.lst`
- **Upstream:** https://sourceforge.net/projects/crunch-wordlist/
- **Manual:** `man crunch`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

crunch generates candidate strings from a character set, a length range, or a
mask pattern, and writes them to stdout or to files. Unlike rule-based
generators, it does not mutate a base wordlist: it enumerates a keyspace
deterministically, which makes it suitable for PINs, fixed-format identifiers,
known-pattern passwords, and exhaustive short-length searches.

It can also permute a small set of words (`-p`, `-q`), limit repeated
characters (`-d`), split output into size- or line-bounded files (`-b`, `-c`),
compress those files (`-z`), and resume an interrupted run (`-r`). Output size
is the operator's responsibility: the tool will happily generate an
astronomically large keyspace.

## Use cases

| Situation | Why crunch |
|-----------|-----------|
| Numeric PIN or OTP space | Exhaustive digits-only enumeration |
| Fixed password policy | Mask exactly the known positions |
| Short keyspace recovery | Full enumeration where rules would miss |
| Wordlist for a lab hash | Deterministic, reproducible candidates |
| Chunked large generation | `-b`/`-c` split files for parallel cracking |
| Permutations of known words | `-p` and `-q` produce orderings |
| Wireless lab handshake | Pipe candidates to `aircrack-ng` |

## Requirements & install layout

- Installed from Debian (`apt`), binary `/usr/bin/crunch`.
- No root required; write access to the output directory is enough.
- `charset.lst` lives at `/usr/share/crunch/charset.lst` and defines named
  sets such as `numeric`, `lalpha`, `ualpha`, `mixalpha-numeric`,
  `mixalpha-numeric-symbol14`, and their `-space` variants.
- Generation is CPU-light per candidate but I/O- and disk-heavy at scale.
- crunch prints an estimate and waits about three seconds before generating, so
  an accidental huge run can still be cancelled.
- No configuration file and no network access.

## Basic syntax

```bash
# all 4-digit numeric strings
crunch 4 4 0123456789 -o pins.txt

# named charset from charset.lst
crunch 6 8 -f /usr/share/crunch/charset.lst mixalpha-numeric -o words.txt

# mask: two lowercase letters then two digits
crunch 4 4 -t @@%% -o mask4.txt

# permutations of a short word
crunch 4 5 -p abc

# chunked, compressed output
crunch 1 5 -o START -c 6000 -z gzip

# write to stdout for a downstream tool
crunch 2 4 abcdefghijklmnopqrstuvwxyz --stdout | head
```

The synopsis is `crunch <min-len> <max-len> [<charset string>] [options]`.
Both length arguments are mandatory even when an option ignores them.

## Flag reference

Flags below are from `man crunch` for version 3.6.

### Output and sizing

| Flag | Meaning |
|------|---------|
| `-o <file>` | Write output to a file; `START` generates names from the first and last word |
| `-b <number><type>` | Split output by size; types `kb`, `mb`, `gb`, `kib`, `mib`, `gib`; requires `-o START` |
| `-c <number>` | Split output by line count; requires `-o START` |
| `-z <gzip\|bzip2\|lzma\|7z>` | Compress split output; requires `-o` |
| `-u` | Disable the percentage thread; place last |
| `--stdout` | Write to stdout when a script expects a stream (documented in the man page examples) |

### Keyspace control

| Flag | Meaning |
|------|---------|
| `-t <pattern>` | Mask using `@` lower, `,` upper, `%` digit, `^` symbol; other characters are literal |
| `-l <string>` | Same length as `-t`; mark pattern positions to treat as literals |
| `-d <n><symbol>` | Limit consecutive duplicates, e.g. `-d 2@` or `-d 3%` |
| `-s <string>` | Start generating at this block |
| `-e <string>` | Stop early at this string |
| `-i` | Invert output order |
| `-f <file> <name>` | Load a named charset from a charset list file |
| `-p <word...>` | Permute the given words; must be the last option; ignores min/max but both are still required |
| `-q <file>` | Read words from a file and permute them, like `-p` |
| `-r` | Resume a previous `-o` run; repeat the original command without `-s` |
| `-m` | Deprecated alias merged into `-p`; use `-p` |

## Recipes

```bash
# 1. Four-digit numeric space
crunch 4 4 0123456789 -o pins.txt

# 2. Eight-character lowercase-numeric space, split into 100 MiB files
crunch 8 8 -f /usr/share/crunch/charset.lst lalpha-numeric \
       -o START -b 100mib

# 3. Mask with a literal domain-like prefix
crunch 7 7 -t 'p@ss,%^' -l 'a@aaaaa' | head

# 4. Permute a small word set
crunch 4 5 -p dog cat bird

# 5. Permute lines from a file
crunch 2 2 -q seeds.txt

# 6. Chunk and compress
crunch 1 5 -o START -c 6000 -z gzip

# 7. Resume an interrupted numeric run
crunch 8 8 0123456789 -o digits8.txt
crunch 8 8 0123456789 -r -o digits8.txt

# 8. Stream into a cracker
crunch 8 8 0123456789 --stdout | hashcat -m 0 -a 0 hashes.txt
```

## Output & pipelines

- Default output is one candidate per line on stdout.
- `-o` writes a file; `-o START` names chunk files by their first and last
  candidate, e.g. `aaaa-gvfed.txt`.
- `-z` appends `.gz`, `.bz2`, `.lzma`, or `.7z`.
- `--stdout` is the safe form when a downstream tool expects a stream and the
  estimate output would otherwise mix in.
- Pipes: `hashcat`, `john`, `aircrack-ng`, `airolib-ng`, `ffuf`, and custom
  scripts.
- Progress and the initial keyspace estimate go to stderr, so stderr can be
  discarded for clean data.

## Gotchas & operational security

- **Estimate before generating.** For a charset of size `c` and lengths `m`
  through `M`, the count is the sum of `c^L`. A 62-character set at length 8 is
  about 2.18e14 candidates; no disk can hold it.
- `-b` and `-c` only take effect with `-o START`.
- `-z` only applies to file output, not stdout.
- `-p` must be the last option, and min/max are ignored but still required.
- `-r` requires the same command as the original run, minus `-s`.
- `-s` and `-e` values must exist in the generated sequence.
- `-l` must be the same length as `-t`; only the marked positions become
  literal.
- A space in a charset must be escaped or quoted, and symbols such as `!`,
  `*`, `?`, and `#` are shell metacharacters.
- Charset order is significant: lowercase, uppercase, digits, symbols.
- crunch waits roughly three seconds before generating; do not automate around
  that delay as if it were a hang.
- The percentage thread writes to the terminal; `-u` suppresses it and must be
  the last option.
- Generated files inherit the current umask; they may contain candidate
  credentials, so restrict permissions and delete them when finished.

## Legal & safety

Use crunch only against password hashes, captures, and systems you own or are
explicitly authorized to test. Generating and using candidate passwords against
third-party accounts or networks is unlawful and usually detectable. Keep
generated wordlists access-controlled, and never publish them with engagement
evidence.

## Related tools

- [`cewl/`](../cewl/README.md) — target-derived wordlists
- [`hashcat/`](https://hashcat.net/hashcat/) — GPU cracking
- [`john/`](https://www.openwall.com/john/) — CPU cracking
- `aircrack-ng` — wireless handshake testing in a lab
- [`ffuf/`](https://github.com/ffuf/ffuf) — fuzzing with generated candidates

## References

- `man crunch` (version 3.6) and `/usr/share/crunch/charset.lst`
- https://sourceforge.net/projects/crunch-wordlist/
- https://www.openwall.com/john/
- https://hashcat.net/wiki/
