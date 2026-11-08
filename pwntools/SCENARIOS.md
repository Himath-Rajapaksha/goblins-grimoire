# pwntools — command scenarios for deep research

Every `pwn` subcommand and flag from [`help.txt`](help.txt) and the per-command
`-h` output, grouped by the work they support. pwntools is primarily a Python
library; this file covers the CLI surface that wraps the same `pwnlib` calls.

## How to invoke

```text
@deep-research For pwntools 4.15, what is the most reliable way to identify a
remote libc from three leaked function offsets when the target distro is
unknown, and how do I verify the match before building a ROP chain? Cite
docs.pwntools.com, libc.rip/libc-database, and write-ups; return exact `pwn
libcdb` and Python commands.
```

---

## Scenario 1 — triage a challenge binary

| # | Command | How to use |
|---|---------|------------|
| 1 | `pwn checksec ./vuln` | All mitigations for one binary |
| 2 | `pwn checksec ./vuln ./libc.so.6` | Multiple files in one pass |
| 3 | `pwn checksec --file ./vuln` | `checksec.sh`-compatible form |
| 4 | `pwn checksec --file ./vuln ./loader` | Multiple files after `--file` |
| 5 | `pwn elfdiff ./vuln ./vuln.patched` | Compare two ELFs |
| 6 | `pwn libcdb file ./libc.so.6` | Dump libc version and symbol offsets |
| 7 | `pwn libcdb file -s read,write,execve ./libc.so.6` | Add symbols to the dump |
| 8 | `pwn libcdb file -o system ./libc.so.6` | Offsets relative to `system` |
| 9 | `pwn libcdb file --unstrip ./libc.so.6` | Unstrip with debuginfod symbols |
| 10 | `pwn pwnstrip ./vuln` | Strip symbols for a cleaner CTF binary |
| 11 | `pwn pwnstrip -b ./vuln` | Also strip the build ID |
| 12 | `pwn pwnstrip -p main -o vuln.stripped ./vuln` | Patch/strip a specific function |
| 13 | `pwn version` | Toolchain version for the write-up |

**Research questions**
- Which `checksec` results actually change the exploit strategy (Full RELRO,
  CET/IBT, SHSTK) on Debian 13 binaries?
- Does `pwnstrip -p` remove or rename the function, and can it break `main`?
- When is `elfdiff` more useful than `cmp`/`radiff2`?

**Sources**: docs.pwntools.com `checksec`/`ELF`, checksec.sh, Debian hardening
docs, CTF write-ups using `elfdiff`.

**Output**: a one-line mitigation summary per binary and a decision on whether
the challenge is ret2win, ret2libc, or heap.

---

## Scenario 2 — find the offset to RIP

| # | Command | How to use |
|---|---------|------------|
| 14 | `pwn cyclic 200` | Generate a 200-byte De Bruijn pattern |
| 15 | `pwn cyclic 200 > pattern.txt` | Save a pattern for a crash run |
| 16 | `pwn cyclic -n 8 400` | 8-byte unique subsequences (64-bit) |
| 17 | `pwn cyclic -a ABC 100` | Custom alphabet (no bad chars) |
| 18 | `pwn cyclic -l 0x6161616c` | Find offset from a 32-bit crash value |
| 19 | `pwn cyclic -l 0x6161616c6161616a` | Find offset from a 64-bit crash value |
| 20 | `pwn cyclic -c amd64 -l 0x6161616c6161616a` | Endianness-aware lookup |
| 21 | `pwn cyclic -o 0x6161616c` | `--offset` alias for lookup |
| 22 | `pwn cyclic --lookup 0x6161616c` | Long-form lookup |
| 23 | `pwn constgrep -c amd64 ^PROT_` | Find `PROT_*` constants |
| 24 | `pwn constgrep -e -c linux O_RDONLY` | Exact constant match |
| 25 | `pwn constgrep -i -c linux 'sock_'` | Case-insensitive regex |
| 26 | `pwn constgrep -m -c amd64 '^PROT_' 3 + 4` | Mask-mode value search |

**Research questions**
- For a 64-bit crash with a non-canonical address, how do you recover the
  offset when the full 8-byte value is truncated?
- Do different pattern lengths or alphabets change `cyclic_find` results?
- Which constants matter for a `mprotect`/`execve` shellcode stage?

**Sources**: pwntools cyclic docs, GDB crash examples, kernel syscall tables,
CTF write-ups.

**Output**: the exact offset used in `flat({offset: ...})`, plus the crash
register evidence that produced it.

---

## Scenario 3 — assemble and disassemble

| # | Command | How to use |
|---|---------|------------|
| 27 | `pwn asm 'mov rax, 60; xor rdi, rdi; syscall'` | Assemble a snippet |
| 28 | `pwn asm -c amd64 'nop; ret'` | Set architecture/OS |
| 29 | `pwn asm -f hex 'int3'` | Hex output |
| 30 | `pwn asm -f raw -o sc.bin 'int3'` | Raw bytes to a file |
| 31 | `pwn asm -f string 'int3'` | Python string literal |
| 32 | `pwn asm -f elf -o sc.elf 'int3'` | ELF-wrapped shellcode |
| 33 | `pwn asm -i snippet.asm` | Assemble from a file |
| 34 | `pwn asm -v 000a0d 'mov rax, 60'` | Avoid specific bytes |
| 35 | `pwn asm -n 'mov rax, 60'` | Avoid newlines |
| 36 | `pwn asm -z 'mov rax, 60'` | Avoid NUL bytes |
| 37 | `pwn asm -e xor 'mov rax, 60'` | Force an encoder |
| 38 | `pwn asm -r 'int3'` | Assemble and execute |
| 39 | `pwn asm -d 'int3'` | Assemble and debug in GDB |
| 40 | `pwn disasm '4831c0c3'` | Disassemble hex |
| 41 | `pwn disasm -c amd64 '4831c0c3'` | Set architecture |
| 42 | `pwn disasm -a 0x400000 'e8fbffffff'` | Resolve relative addresses |
| 43 | `pwn disasm --color '90c3'` | Force colored output |
| 44 | `pwn disasm --no-color '90c3'` | Plain output for pipes |
| 45 | `pwn disasm < sc.bin` | Disassemble stdin |

**Research questions**
- When should shellcode be assembled with `-f elf` instead of raw bytes?
- Which encoders exist, and which avoid the byte set that breaks a specific
  parser?
- Does `-a` affect only display or also symbol resolution?

**Sources**: asm/disasm docs, shellcode encoder docs, pwntools source,
write-ups with bad-character constraints.

**Output**: reproducible bytes plus the exact context (`arch`, `os`) used.

---

## Scenario 4 — generate shellcode with shellcraft

| # | Command | How to use |
|---|---------|------------|
| 46 | `pwn shellcraft amd64.linux.sh` | `execve('/bin/sh')` |
| 47 | `pwn shellcraft -l` | List all shellcodes |
| 48 | `pwn shellcraft -l sh` | Filter the list |
| 49 | `pwn shellcraft -? amd64.linux.sh` | Show shellcode documentation |
| 50 | `pwn shellcraft -f a amd64.linux.sh` | Assembly output |
| 51 | `pwn shellcraft -f c amd64.linux.sh` | C-style array |
| 52 | `pwn shellcraft -f p amd64.linux.sh` | Preprocessed code |
| 53 | `pwn shellcraft -f r -o sc.bin amd64.linux.sh` | Raw bytes to a file |
| 54 | `pwn shellcraft -f i amd64.linux.sh` | Hex-escaped string |
| 55 | `pwn shellcraft -f d amd64.linux.sh` | Escape-d hex string |
| 56 | `pwn shellcraft amd64.linux.execve '/bin/sh'` | Parameterized shellcode |
| 57 | `pwn shellcraft amd64.linux.sh + amd64.linux.cat flag.txt` | Concatenate shellcodes |
| 58 | `pwn shellcraft --delim '\n' a + b` | Custom delimiter |
| 59 | `pwn shellcraft -b amd64.linux.sh` | `int3` before code |
| 60 | `pwn shellcraft -a amd64.linux.sh` | `int3` after code |
| 61 | `pwn shellcraft -v 000a0d amd64.linux.sh` | Avoid bytes |
| 62 | `pwn shellcraft -n amd64.linux.sh` | Avoid newlines |
| 63 | `pwn shellcraft -z amd64.linux.sh` | Avoid NULs |
| 64 | `pwn shellcraft -r amd64.linux.sh` | Run it |
| 65 | `pwn shellcraft -d amd64.linux.sh` | Debug it |
| 66 | `pwn shellcraft --syscalls` | List syscall names/numbers |
| 67 | `pwn shellcraft --address 0x400000 amd64.linux.sh` | Load address |
| 68 | `pwn shellcraft -s -f elf amd64.linux.sh` | Shared-library ELF |
| 69 | `pwn shellcraft --color -f a amd64.linux.sh` | Colorized assembly |
| 70 | `pwn shellcraft --no-color -f a amd64.linux.sh` | Plain assembly |

**Research questions**
- Which shellcodes are staged vs single-stage, and which require a writable
  segment?
- What changes when targeting musl, static binaries, or seccomp filters?
- Are `-b`/`-a` traps safe to leave in a final payload?

**Sources**: shellcraft docs, pwntools shellcraft source, seccomp write-ups,
CTF challenge archives.

**Output**: final shellcode bytes plus a note on size, bad characters, and
required memory permissions.

---

## Scenario 5 — patch and prepare binaries

| # | Command | How to use |
|---|---------|------------|
| 71 | `pwn elfpatch ./vuln 0x1234 90 90` | Patch virtual address with NOPs |
| 72 | `pwn elfpatch ./vuln 0x1234 c3` | Insert `ret` |
| 73 | `pwn disablenx ./vuln` | Clear NX for a CTF binary |
| 74 | `pwn disablenx ./vuln ./vuln2` | Multiple binaries |
| 75 | `pwn elfdiff ./vuln ./vuln.patched` | Verify the patch |
| 76 | `pwn pwnstrip -o vuln.clean ./vuln` | Write a stripped copy |
| 77 | `pwn pwnstrip -b -o vuln.nobuildid ./vuln` | Remove build ID too |
| 78 | `pwn pwnstrip -p check -o vuln.nocheck ./vuln` | Patch out a check function |

**Research questions**
- Is `elfpatch` writing to the virtual address or the file offset — and does
  it handle PIE load bias?
- Does `disablenx` modify the program header, and will the kernel still honor
  it on modern Debian?
- What does `pwnstrip -p` actually do to the function body?

**Sources**: ELF docs, pwntools `ELF.patch` docs, issues on `disablenx` with
PIE, CTF write-ups.

**Output**: patched binary with a diff proving only intended bytes changed.

---

## Scenario 6 — debug a target

| # | Command | How to use |
|---|---------|------------|
| 79 | `pwn debug ./vuln` | Launch in GDB |
| 80 | `pwn debug -x init.gdb ./vuln` | Run a GDB script |
| 81 | `pwn debug --pid 1234` | Attach to a PID |
| 82 | `pwn debug --process bash` | Attach by process name |
| 83 | `pwn debug --exec ./vuln` | Explicit executable |
| 84 | `pwn debug -c amd64 ./vuln` | Set context for GDB helpers |
| 85 | `pwn debug --sysroot /chroot ./vuln` | Debug against a sysroot |
| 86 | `pwn template ./vuln` | Generate an exploit skeleton |
| 87 | `pwn template ./vuln --host target.ctf --port 1337 > exploit.py` | Remote-ready skeleton |
| 88 | `pwn template --host ssh.ctf --port 22 --user ctf --pass pw --path /challenge/vuln` | SSH skeleton |
| 89 | `pwn template --libc ./libc.so.6 ./vuln` | Pin libc in the template |
| 90 | `pwn template --template ./custom.mako ./vuln` | Custom template |
| 91 | `pwn template --quiet ./vuln` | Fewer comments |
| 92 | `pwn template --color always ./vuln` | Colored output |
| 93 | `pwn template --no-auto ./vuln` | No binary auto-detection |

**Research questions**
- Which `pwn debug` paths require a TTY, and how do you use it over SSH?
- What does `pwn template` infer from the binary, and what must be filled in
  manually?
- How does the generated template switch between local and remote (`args.REMOTE`)?

**Sources**: GDB docs, pwntools debug/template docs, example exploit repos.

**Output**: an `exploit.py` that runs locally and remotely with one flag.

---

## Scenario 7 — identify a remote libc

| # | Command | How to use |
|---|---------|------------|
| 94 | `pwn libcdb lookup 'read 3e0 write 520'` | Match libc by symbol offsets |
| 95 | `pwn libcdb lookup 'puts 80e50 system 52290'` | Two-symbol match |
| 96 | `pwn libcdb lookup -d 'read 3e0'` | Download the matching libc |
| 97 | `pwn libcdb lookup --offline-only 'read 3e0'` | Use the local database only |
| 98 | `pwn libcdb lookup --no-unstrip 'read 3e0'` | Keep the stripped binary |
| 99 | `pwn libcdb hash -t buildid abc123` | Lookup by build ID |
| 100 | `pwn libcdb hash -t md5 d41d8cd98f00b204e9800998ecf8427e` | Lookup by MD5 |
| 101 | `pwn libcdb hash -t sha1 <sha1>` | Lookup by SHA-1 |
| 102 | `pwn libcdb hash -t sha256 <sha256>` | Lookup by SHA-256 |
| 103 | `pwn libcdb hash -d -t buildid abc123` | Download the match |
| 104 | `pwn libcdb hash --offline-only -t id 123` | Offline hash lookup |
| 105 | `pwn libcdb fetch` | Init/upgrade the libc database |
| 106 | `pwn libcdb fetch -u libc6_2.39 libc6_2.40` | Fetch specific categories |
| 107 | `pwn libcdb fetch /srv/libc-database` | Use a custom DB path |

**Research questions**
- How many leaked offsets are needed to disambiguate a libc uniquely?
- How do you verify a downloaded libc against the leak before using it?
- What is the trust model for the libc database and debuginfod unstrip?

**Sources**: libc.rip, libc-database, pwntools libcdb docs, distro package
archives.

**Output**: a libc file whose symbol offsets reproduce all leaks, plus its
provenance (hash and source).

---

## Scenario 8 — encode, convert, inspect bytes

| # | Command | How to use |
|---|---------|------------|
| 108 | `pwn hex 'AAAA'` | Hex-encode text |
| 109 | `pwn hex -p '\x' 'AAAA'` | Prefix each byte |
| 110 | `pwn hex -s ' ' 'AAAA'` | Space-separated bytes |
| 111 | `pwn hex < payload.bin` | Encode binary from stdin |
| 112 | `pwn unhex 41414141` | Decode hex text |
| 113 | `pwn unhex 41 41 41 41` | Decode multiple hex tokens |
| 114 | `pwn unhex < hex.txt` | Decode from stdin |
| 115 | `pwn phd payload.bin` | Pretty hex dump |
| 116 | `pwn phd -w 32 payload.bin` | Wider lines |
| 117 | `pwn phd -l 41 42 payload.bin` | Highlight specific bytes |
| 118 | `pwn phd -s 16 payload.bin` | Skip the first 16 bytes |
| 119 | `pwn phd -c 64 payload.bin` | Show only 64 bytes |
| 120 | `pwn phd -o 0x400000 payload.bin` | Start the address column at base |
| 121 | `pwn phd --color always payload.bin` | Force color |
| 122 | `pwn phd --color never payload.bin` | No color |
| 123 | `pwn scramble -f hex 'AAAA'` | Encode bytes |
| 124 | `pwn scramble -p -f hex 'AAAA'` | Alphanumeric encoder |
| 125 | `pwn scramble -v 000a0d -f raw -o enc.bin sc.bin` | Avoid bytes |
| 126 | `pwn scramble -n -f hex 'AAAA'` | Avoid newlines |
| 127 | `pwn scramble -z -f hex 'AAAA'` | Avoid NULs |
| 128 | `pwn scramble -c amd64 -d -f raw sc.bin` | Debug the encoder |
| 129 | `pwn errno 2` | Message for errno 2 |
| 130 | `pwn errno ENOENT` | Message for a name |
| 131 | `pwn errno -1` | Unknown/negative lookup |

**Research questions**
- When is `scramble` preferable to `asm -e`?
- Does the alphanumeric encoder preserve length, and what decoder is needed?
- How do you keep hex dumps stable for diffing between runs?

**Sources**: pwntools util docs, encoder source, shellcode encoding papers,
write-ups with alphanumeric constraints.

**Output**: encoded payload plus the decoder stub and a byte-class check.

---

## Scenario 9 — keep the toolchain current

| # | Command | How to use |
|---|---------|------------|
| 132 | `pwn update` | Check for updates |
| 133 | `pwn update --install` | Install available update |
| 134 | `pwn update --pre` | Include pre-releases |
| 135 | `pwn --help` | Top-level command list |
| 136 | `pwn asm --help` | Per-command help |

**Research questions**
- Is the pipx/pip install path upgradable in place, or does it need a reinstall?
- How do you pin pwntools for a reproducible CTF environment?

**Sources**: pwntools install docs, pipx docs, release notes.

**Output**: a pinned version and upgrade procedure that does not break the
installed `pwn` entry point.

---

## Complete flag coverage

Every flag from the installed `pwn <subcommand> -h` output appears at least
once. Subcommands without flags are listed at the end.

| Subcommand | Flags | Scenario |
|------------|-------|----------|
| `asm` | `-f/--format`, `-o/--output`, `-c/--context`, `-v/--avoid`, `-n/--newline`, `-z/--zero`, `-d/--debug`, `-e/--encoder`, `-i/--infile`, `-r/--run` | 3 |
| `checksec` | `--file` | 1 |
| `constgrep` | `-e/--exact`, `-i/--case-insensitive`, `-m/--mask-mode`, `-c/--context` | 2 |
| `cyclic` | `-a/--alphabet`, `-n/--length`, `-c/--context`, `-l/-o/--offset/--lookup` | 2 |
| `debug` | `-x`, `--pid`, `-c/--context`, `--exec`, `--process`, `--sysroot` | 6 |
| `disasm` | `-c/--context`, `-a/--address`, `--color`, `--no-color` | 3 |
| `disablenx` | none | 5 |
| `elfdiff` | none | 1, 5 |
| `elfpatch` | none | 5 |
| `errno` | none | 8 |
| `hex` | `-p/--prefix`, `-s/--separator` | 8 |
| `libcdb lookup` | `-d/--download-libc`, `--no-unstrip`, `--offline-only` | 7 |
| `libcdb hash` | `-t/--hash_type`, `-d/--download-libc`, `--no-unstrip`, `--offline-only` | 7 |
| `libcdb file` | `-s/--symbols`, `-o/--offset`, `--unstrip` | 1 |
| `libcdb fetch` | `-u/--update` | 7 |
| `phd` | `-w/--width`, `-l/--highlight`, `-s/--skip`, `-c/--count`, `-o/--offset`, `--color` | 8 |
| `pwnstrip` | `-b/--build-id`, `-p/--patch`, `-o/--output` | 1, 5 |
| `scramble` | `-f/--format`, `-o/--output`, `-c/--context`, `-p/--alphanumeric`, `-v/--avoid`, `-n/--newline`, `-z/--zero`, `-d/--debug` | 8 |
| `shellcraft` | `-?/--show`, `-o/--out`, `-f/--format`, `-d/--debug`, `--delim`, `-b/--before`, `-a/--after`, `-v/--avoid`, `-n/--newline`, `-z/--zero`, `-r/--run`, `--color`, `--no-color`, `--syscalls`, `--address`, `-l/--list`, `-s/--shared` | 4 |
| `template` | `--host`, `--port`, `--user`, `--pass/--password`, `--libc`, `--path`, `--quiet`, `--color`, `--template`, `--no-auto` | 6 |
| `unhex` | none | 8 |
| `update` | `--install`, `--pre` | 9 |
| `version` | none | 1 |

Subcommands: `asm`, `checksec`, `constgrep`, `cyclic`, `debug`, `disasm`,
`disablenx`, `elfdiff`, `elfpatch`, `errno`, `hex`, `libcdb`, `phd`,
`pwnstrip`, `scramble`, `shellcraft`, `template`, `unhex`, `update`,
`version`.

Full per-command text: [`help.txt`](help.txt); re-capture per-command help
with `pwn <subcommand> -h`.
