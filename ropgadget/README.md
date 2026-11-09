# ROPgadget — ROP/JOP gadget finder and chain generator

- **Version:** ROPgadget (pipx install; version from `ROPgadget --version`)
- **Binary:** `/usr/local/bin/ROPgadget`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/JonathanSalwan/ROPgadget

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

ROPgadget scans an executable (ELF, PE, Mach-O, or raw blob) for instruction
sequences ending in a control transfer: ROP (`ret`), JOP (`jmp`/`call`), and
SYS (`syscall`, `int 0x80`) gadgets. It uses Capstone for disassembly, can
filter by mnemonic, address range, and bad bytes, and can generate a full ROP
chain for `execve`-style or Windows API goals with `--ropchain`.

## Use cases

| Situation | Why ROPgadget |
|-----------|---------------|
| Ret2libc/ret2syscall prep | Fast gadget inventory across architectures |
| Cross-checking ropper | Different search engine, different dedupe |
| Chain generation | `--ropchain` emits a ready chain |
| PE exploit development | `--only`, `--filter`, and `--callPreceded` for CFG |
| Raw firmware blobs | `--rawArch/--rawMode/--rawEndian` |
| MIPS/ARM work | `--mipsrop` and `--thumb` helpers |

## Requirements & install layout

- Installed with pipx; launcher `/usr/local/bin/ROPgadget`.
- Pure Python + Capstone; no root, no network.
- Analysis is read-only against the target file.
- Case-sensitive binary name: `ROPgadget` (not `ropgadget`).
- Use `--silent` when the analysis progress output would pollute a pipeline.

## Basic syntax

```bash
# default search
ROPgadget --binary ./vuln

# find a specific instruction sequence
ROPgadget --binary ./vuln --only "pop|ret"

# suppress noisy mnemonics
ROPgadget --binary ./vuln --filter "xchg|add|sub"

# generate a chain
ROPgadget --binary ./vuln --ropchain

# raw firmware blob
ROPgadget --binary ./fw.bin --rawArch arm --rawMode thumb --rawEndian little
```

## Flag reference

### Core

| Flag | Meaning |
|------|---------|
| `-h, --help` | Help |
| `-v, --version` | Version |
| `-c, --checkUpdate` | Check for a newer release |
| `--binary <file>` | Target binary |

### Search modes

| Flag | Meaning |
|------|---------|
| `--opcode <opcodes>` | Search raw opcodes in the executable segment |
| `--string <string>` | Search for a string (`.` wildcards allowed) in readable segments |
| `--memstr <string>` | Match each byte anywhere in readable segments |
| `--depth <nbyte>` | Search depth (default 10) |
| `--only <key>` | Only include gadgets matching these mnemonics (`mov\|ret`) |
| `--filter <key>` | Suppress gadgets containing these mnemonics |
| `--range <start-end>` | Restrict to an address range |
| `--badbytes <byte>` | Reject gadgets whose address contains these bytes |
| `--re <re>` | Regular-expression gadget match |
| `--offset <hexaddr>` | Add an offset to printed addresses |
| `--align <bytes>` | Only show gadgets aligned to this boundary |

### Raw files and architectures

| Flag | Meaning |
|------|---------|
| `--rawArch <arch>` | `x86, arm, arm64, sparc, mips, ppc, riscv` |
| `--rawMode <mode>` | `32, 64, arm, thumb` |
| `--rawEndian <endian>` | `little, big` |
| `--thumb` | Thumb mode for ARM |

### Engine selection and output

| Flag | Meaning |
|------|---------|
| `--norop` | Disable ROP search |
| `--nojop` | Disable JOP search |
| `--nosys` | Disable SYS search |
| `--callPreceded` | Only gadgets preceded by a call |
| `--multibr` | Allow multiple branch gadgets |
| `--all` | Keep duplicate gadgets |
| `--noinstr` | Do not print instructions |
| `--dump` | Print gadget bytes |
| `--silent` | Suppress progress output |
| `--console` | Interactive console |
| `--ropchain` | Generate a ROP chain |
| `--mipsrop <rtype>` | MIPS helper: `stackfinder, system, tails, lia0, registers` |

## Recipes

```bash
# 1. Full inventory, quiet
ROPgadget --binary ./vuln --silent > gadgets.txt

# 2. Classic argument gadgets only
ROPgadget --binary ./vuln --only "pop|ret" --silent

# 3. Suppress side-effect-prone mnemonics
ROPgadget --binary ./vuln --filter "xchg|add|sub|cmov" --silent

# 4. Bad-byte-aware search on a 32-bit target
ROPgadget --binary ./vuln --badbytes "00|0a|0d" --only "pop|ret"

# 5. Search a narrow code range
ROPgadget --binary ./vuln --range 0x08041000-0x08042000 --only "ret"

# 6. Find a writable string
ROPgadget --binary ./vuln --string "/bin/sh"

# 7. Generate a chain
ROPgadget --binary ./vuln --ropchain

# 8. PIE-aware addresses
ROPgadget --binary ./vuln --offset 0x555555554000 --only "pop|ret"

# 9. Raw ARM firmware
ROPgadget --binary ./fw.bin --rawArch arm --rawMode thumb --rawEndian little --silent

# 10. PE with call-preceded gadgets only
ROPgadget --binary ./app.exe --callPreceded --only "pop|ret"
```

## Output & pipelines

- Default: address, bytes, and instruction sequence per line.
- `--silent` removes progress noise, making output `grep`-friendly.
- `--dump` adds the gadget byte string.
- Feed into pwntools `ROP()` or a hand-built `flat()`.
- Diff two runs to detect gadget drift after recompilation.

## Gotchas & operational security

- Gadgets are matched in executable bytes and may be unreachable at runtime;
  validate in a debugger.
- `--depth` above 10 explodes runtime and output size.
- Deduplication is on by default; `--all` produces very large output.
- `--badbytes` inspects the address, not the gadget bytes — different from
  ropper's `-b`.
- `--ropchain` makes fixed assumptions about the target and should be treated
  as a draft.
- `--offset` only changes printed values; it does not relocate the binary.
- The `-c/--checkUpdate` flag contacts the network; avoid it in offline labs.
- Very large binaries can exhaust memory; scope with `--range`.

## Legal & safety

Use only on binaries you own or are authorized to assess in an isolated lab.
Exploit development can crash targets and trigger monitoring; never run
generated chains against production or third-party systems.

## Related tools

- [`ropper/`](../ropper/README.md) — alternative finder with chain generators
- [`one-gadget/`](../one-gadget/README.md) — single-address glibc gadgets
- [`pwndbg/`](../pwndbg/README.md), [`gef/`](../gef/README.md) — debugger
  verification
- [`pwntools/`](../pwntools/README.md) — chain assembly

## References

- https://github.com/JonathanSalwan/ROPgadget
- https://pypi.org/project/ROPgadget/
