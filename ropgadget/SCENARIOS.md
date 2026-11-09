# ROPgadget — command scenarios for deep research

Installed build: ROPgadget via pipx at `/usr/local/bin/ROPgadget`. Flags come
from `help.txt`.

## How to invoke

```text
@deep-research For ROPgadget (current pipx build), how do --only, --filter,
and --callPreceded change gadget quality on modern PIE/NX binaries, how does
--ropchain compare with pwntools ROP and ropper chain generation, and what
validation proves a gadget is reachable? Cite the ROPgadget repo, Capstone
docs, and exploit-dev references; return a validated search and chain workflow.
```

---

## Scenario 1 — inventory and narrowing

| # | Command | How to use |
|---|---------|------------|
| 1 | `ROPgadget --binary ./vuln --silent > gadgets.txt` | Full inventory |
| 2 | `ROPgadget --binary ./vuln --only "pop\|ret"` | Argument gadgets |
| 3 | `ROPgadget --binary ./vuln --filter "xchg\|add\|sub"` | Remove noise |
| 4 | `ROPgadget --binary ./vuln --depth 5` | Shallow search |
| 5 | `ROPgadget --binary ./vuln --re "pop rdi"` | Regex match |

**Research questions**
- How does `--depth` trade coverage against runtime and false positives?
- Which mnemonics most often signal unwanted side effects?

**Sources:** ROPgadget source, Capstone, disassembly.
**Output:** a narrowed gadget inventory.

---

## Scenario 2 — bad bytes and ranges

| # | Command | How to use |
|---|---------|------------|
| 6 | `ROPgadget --binary ./vuln --badbytes "00\|0a\|0d"` | Reject address bytes |
| 7 | `ROPgadget --binary ./vuln --range 0x08041000-0x08042000` | Narrow range |
| 8 | `ROPgadget --binary ./vuln --offset 0x555555554000` | PIE base |
| 9 | `ROPgadget --binary ./vuln --align 8` | Aligned gadgets |
| 10 | `ROPgadget --binary ./vuln --dump --only "ret"` | Bytes included |

**Research questions**
- Does `--badbytes` filter the address or the instruction bytes?
- How does `--align` interact with multi-byte instruction encoding?

**Sources:** help, source, output comparison.
**Output:** a target-specific usable gadget set.

---

## Scenario 3 — raw binaries and other architectures

| # | Command | How to use |
|---|---------|------------|
| 11 | `ROPgadget --binary ./fw.bin --rawArch arm --rawMode thumb` | Thumb firmware |
| 12 | `ROPgadget --binary ./fw.bin --rawArch mips --rawMode 32 --rawEndian big` | MIPS big-endian |
| 13 | `ROPgadget --binary ./vuln --thumb` | ARM Thumb mode |
| 14 | `ROPgadget --binary ./vuln --mipsrop stackfinder` | MIPS stack pivot |
| 15 | `ROPgadget --binary ./vuln --mipsrop system` | MIPS system gadgets |

**Research questions**
- Which raw settings are required for each architecture, and how are they
  verified?
- What do the `--mipsrop` helper modes actually search for?

**Sources:** help, source, firmware docs.
**Output:** architecture-correct gadget lists.

---

## Scenario 4 — chain generation and comparison

| # | Command | How to use |
|---|---------|------------|
| 16 | `ROPgadget --binary ./vuln --ropchain` | Generated chain |
| 17 | `ropper -f ./vuln --chain "execve=/bin/sh"` | Cross-check with ropper |
| 18 | `python3 -c 'from pwn import *; ...'` | Port into pwntools |
| 19 | `ROPgadget --binary ./vuln --ropchain > chain.txt` | Save draft |
| 20 | `diff <(grep '0x' chain.txt) <(ropper ... )` | Compare outputs |

**Research questions**
- What target assumptions does `--ropchain` encode?
- Where do ROPgadget and ropper disagree, and why?

**Sources:** generator source, pwntools docs, cross-tool output.
**Output:** a chain draft with documented assumptions.

---

## Scenario 5 — verification and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `gdb -q ./vuln -ex 'start' -ex 'x/3i 0x...' -ex 'quit'` | Verify gadget bytes |
| 22 | `ROPgadget --binary ./vuln --silent > run1.txt` | Baseline |
| 23 | `ROPgadget --binary ./vuln --silent > run2.txt && diff run1.txt run2.txt` | Reproducibility |
| 24 | `sha256sum ./vuln run1.txt` | Hash evidence |
| 25 | `wc -l run1.txt` | Inventory size |

**Research questions**
- Which gadgets survive debugger verification, and which are data artefacts?
- How stable is the inventory across rebuilds?

**Sources:** GDB, hashes, diff output.
**Output:** verified gadgets and a reproducible evidence set.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Core | `-v -c --binary` | 4 |
| Search | `--opcode --string --memstr --depth --only --filter --range --badbytes --re --offset --align` | 1–3 |
| Raw/arch | `--rawArch --rawMode --rawEndian --thumb --mipsrop` | 3 |
| Engine | `--norop --nojop --nosys --callPreceded --multibr --all` | 1, 3 |
| Output | `--noinstr --dump --silent --console --ropchain` | 2, 4, 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — inventory | 5 |
| 2 — bad bytes | 5 |
| 3 — architectures | 5 |
| 4 — chains | 5 |
| 5 — verification | 5 |
