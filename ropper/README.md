# ropper — ROP gadget finder and chain builder

- **Version:** 1.13.13
- **Binary:** `/usr/local/bin/ropper`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/sashs/Ropper
- **Package:** https://pypi.org/project/ropper/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Ropper loads an executable (ELF, PE, Mach-O, or a raw blob), disassembles its
executable sections, and indexes instruction sequences that end in a control
transfer — `ret`, `jmp reg`, `call reg` — as ROP, JOP, or SYS gadgets. It can
filter by bad bytes, instruction count, and regular expression, inspect file
headers and symbols, and generate ready-made chains with the built-in `execve`,
`mprotect`, and `virtualprotect` generators.

It is an offline static analyzer. It does not execute the target and does not
resolve a runtime libc base; you supply the load address with `-I` when the
binary is PIE or when you are working against a mapped module.

## Use cases

| Situation | Why ropper |
|-----------|------------|
| CTF or lab pwn challenge | Fast gadget search without writing a parser |
| Building a ret2libc / ret2syscall chain | `execve` generator and bad-byte filtering |
| Finding a stack pivot | `--stack-pivot` enumerates `leave; ret`, `xchg rsp, ...`, and similar |
| PE / Windows exploit development | PE headers, `DllCharacteristics`, CFG-aware filtering |
| Auditing a binary's gadget surface | Section, symbol, import, and entry-point inspection |
| Cross-architecture work | x86, x86-64, ARM/Thumb, ARM64, MIPS, PPC, SPARC |
| Reproducible gadget inventories | Text output that can be diffed between builds |

## Requirements & install layout

- Installed with **pipx**; the launcher is `/usr/local/bin/ropper`.
- Pure Python; no root privileges and no special hardware are required.
- Analysis is read-only against the target file.
- Colored output by default; use `--nocolor` for files and pipelines.
- No config file is required. A gadget cache is maintained internally and can
  be cleared with `--clear-cache`.
- The target must be readable. Stripped binaries still work; symbols and
  imports are simply unavailable.

## Basic syntax

```bash
ropper -f ./vuln
ropper -f ./vuln --search "pop rdi; ret"
ropper -f ./vuln --chain "execve=/bin/sh"
ropper -f ./vuln --stack-pivot
ropper --console -f ./vuln
```

## Flag reference

### File loading and architecture

| Flag | Meaning |
|------|---------|
| `-f, --file <file> [<file> ...]` | File(s) to load |
| `-r, --raw` | Treat the input as a raw binary, not a recognized container |
| `-a, --arch <arch>` | Architecture: `x86`, `x86_64`, `MIPS`, `MIPS64`, `ARM`, `ARMTHUMB`, `ARM64`, `PPC`, `PPC64`, `SPARC`, `SPARC64` |
| `-I <imagebase>` | Image base used when printing gadget addresses |

### Binary inspection

| Flag | Meaning |
|------|---------|
| `-i, --info` | Show the file header (ELF/PE/Mach-O) |
| `-e` | Show the entry point |
| `--imagebase` | Show the image base (ELF/PE/Mach-O) |
| `-c, --dllcharacteristics` | Show PE `DllCharacteristics` |
| `-s, --sections` | Show file sections |
| `-S, --segments` | Show file segments (ELF/Mach-O) |
| `--imports` | Show imports (ELF/PE) |
| `--symbols` | Show symbols (ELF) |
| `--section <section>` | Print the contents of one section |
| `--string [<string>]` | Search all data sections for a string |
| `--hex` | Print selected sections in hex |

### Assemble and disassemble

| Flag | Meaning |
|------|---------|
| `--asm [<asm> [H\|S\|R] ...]` | Assemble a string; output as hex (`H`, default), string (`S`), or raw (`R`) |
| `--disasm <opcode>` | Disassemble an opcode such as `ffe4` or `89c8c3` |
| `--disassemble-address <addr:length>` | Disassemble at an address; `L<n>` limits instruction count |

### Gadget search

| Flag | Meaning |
|------|---------|
| `--search <regex>` | Search gadgets by regular expression |
| `--quality <quality>` | Quality threshold for search results (`1` = best) |
| `--opcode <opcode>` | Search opcodes; `?` is a wildcard (`ffe?`, `ff??`) |
| `--instructions <instructions>` | Search instruction text, e.g. `"jmp esp"` |
| `--type <type>` | Gadget type: `rop`, `jop`, `sys`, or `all` (default `all`) |
| `--inst-count <n>` | Maximum instructions per gadget (default `6`) |
| `-b, --badbytes <badbytes>` | Reject gadgets containing the given bytes |
| `-p, --ppr` | Find `pop reg; pop reg; ret` sequences (x86/x86-64) |
| `-j, --jmp <reg>` | Find `jmp reg`; accepts a comma-separated register list (x86/x86-64) |
| `--stack-pivot` | Print stack-pivot gadgets |
| `--detailed` | Print more detail per gadget |
| `--all` | Do not remove duplicate gadgets |
| `--cfg-only` | Drop gadgets that fail the Microsoft CFG check (PE) |

### Chain generation

| Flag | Meaning |
|------|---------|
| `--chain <generator>` | Generate a chain: `execve[=/bin/sh]`, `mprotect address=... size=...`, `virtualprotect address=...` |
| `--set <option>` | Set a chain-generation option: `aslr` or `nx` |
| `--unset <option>` | Unset a chain-generation option: `aslr` or `nx` |

### Semantic search

| Flag | Meaning |
|------|---------|
| `--semantic <constraint>` | Search gadgets that satisfy a semantic constraint |
| `--analyse <quality>` | Analysis quality used by the semantic-search implementation |
| `--count-of-findings <n>` | Maximum semantic results printed (`0` = unlimited, default `5`) |

### Console and maintenance

| Flag | Meaning |
|------|---------|
| `--console` | Start the interactive ropper console |
| `--no-load` | Do not preload gadgets when the console starts |
| `--clear-cache` | Clear ropper's cache |
| `--single` | Disable multiprocess gadget scanning |
| `--nocolor` | Disable ANSI color |
| `--help-examples` | Print usage examples |
| `-v, --version` | Print the version |
| `-h, --help` | Show help |

## Recipes

```bash
# Fingerprint the target before searching
ropper -f ./vuln -i -s --symbols --imports

# Canonical pop rdi; ret lookup
ropper -f ./vuln --search "pop rdi; ret" --nocolor

# Null-byte-free pop rsi (single bad byte is unambiguous)
ropper -f ./vuln --search "pop rsi" -b 00 --nocolor

# Build a Linux execve("/bin/sh", 0, 0) chain
ropper -f ./vuln --chain "execve=/bin/sh" --nocolor

# Build an mprotect chain for a staged shellcode page
ropper -f ./vuln --chain "mprotect address=0xdeadbeef size=0x10000" --nocolor

# Enumerate pivots that move rsp into controlled memory
ropper -f ./vuln --stack-pivot --nocolor

# Address gadgets for a PIE binary loaded at a known base
ropper -f ./vuln -I 0x555555554000 --search "pop rdi; ret" --nocolor

# PE target compiled with CFG
ropper -f ./app.exe -c --cfg-only --search "pop rcx"

# Raw shellcode blob
ropper -r -a x86_64 -f ./stage.bin --search "jmp rsp"

# Interactive exploration
ropper --console -f ./vuln
```

## Output & pipelines

Default output is colored text: gadget address, raw bytes, and disassembly.
`--nocolor` makes it stable for `grep`, `awk`, and file capture:

```bash
ropper -f ./vuln --search "pop rdi; ret" --nocolor | tee gadgets.txt
ropper -f ./vuln --search "pop rdi; ret" --nocolor | awk '{print $1}'
```

Chain output is a textual chain listing. Copy the addresses into a pwntools
`ROP()` object or a `flat()` call; ropper does not emit a pwntools script.
Gadget lists diff cleanly between two builds of the same target, which is
useful for regression-checking compiler or mitigation changes.

## Gotchas & operational security

- The default instruction cap is 6. A useful gadget may be longer; raise
  `--inst-count` deliberately, because output grows quickly.
- Duplicate removal is on by default. `--all` can produce very large output and
  is rarely what you want in a pipeline.
- Bad-byte filtering only inspects the gadget bytes. It does not model stack
  alignment, side effects, or values the gadget writes.
- `-I` changes printed addresses only; it does not relocate or load the file.
- `--set aslr` / `--set nx` are chain-generation assumptions, not switches that
  modify the target or prove a bypass.
- A gadget can be decoded from data embedded in an executable section and may
  never be reachable in practice. Validate in a debugger (`pwndbg`, `gef`) or
  with an emulator before relying on it.
- Semantic search is the least documented surface in the snapshot. Read
  `ropper --help-examples` on the host before scripting it.
- Very large binaries and `--all` are CPU- and memory-heavy; `--single` trades
  speed for lower parallelism.
- No network access is needed; do not upload target binaries to third-party
  gadget services.

## Legal & safety

Use ropper only on binaries you own or are explicitly authorized to assess, in
an isolated lab. Developing or validating an exploit against a production
system, a third party's service, or a shared host is not authorized by this
documentation. Exploit development can crash the target, corrupt memory, and
trigger monitoring; contain it in a VM or dedicated test host.

## Related tools

- [`ropgadget/`](../ropgadget/README.md) — alternative gadget finder with a
  different search engine and `--ropchain` output.
- [`one-gadget/`](../one-gadget/README.md) — single-address glibc gadgets.
- [`pwndbg/`](../pwndbg/README.md) — interactive gadget and memory inspection
  during a debug session.
- [`gef/`](../gef/README.md) — alternative GDB extension with `rop` search.
- [`pwntools/`](../pwntools/README.md) — chain assembly and process interaction.

## References

- Upstream repository and README: https://github.com/sashs/Ropper
- PyPI package: https://pypi.org/project/ropper/
- Installed help snapshot: [`help.txt`](help.txt)
