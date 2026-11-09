# ropper — command scenarios for deep research

Installed build: ropper 1.13.13. Flags come from `help.txt`; the README covers
the full surface and caveats.

## How to invoke

```text
@deep-research For ropper 1.13.13, which gadget-search strategies produce
reachable, side-effect-free gadgets on modern PIE/ASLR Linux binaries, how do
its execve/mprotect chain generators compare with pwntools ROP and ROPgadget
--ropchain, and what validation catches gadgets that exist in bytes but are
unreachable in practice? Cite ropper docs, pwntools docs, and exploit-dev field
reports; return a validated gadget-selection workflow.
```

---

## Scenario 1 — fingerprint and inspect the target

| # | Command | How to use |
|---|---------|------------|
| 1 | `ropper -f ./vuln -i` | File header |
| 2 | `ropper -f ./vuln -s` | Sections |
| 3 | `ropper -f ./vuln --symbols` | Symbols |
| 4 | `ropper -f ./vuln --imports` | Imports |
| 5 | `ropper -f ./vuln -e --imagebase` | Entry point and image base |

**Research questions**
- Which sections are executable, and does the binary have writable+executable
  segments?
- Do symbols/imports survive stripping, and what does their absence imply?

**Sources:** ELF spec, ropper output, `readelf -S`.
**Output:** a binary profile that constrains gadget choice.

---

## Scenario 2 — targeted gadget search

| # | Command | How to use |
|---|---------|------------|
| 6 | `ropper -f ./vuln --search "pop rdi; ret" --nocolor` | First argument gadget |
| 7 | `ropper -f ./vuln --search "pop rsi" -b 00 --nocolor` | Null-byte-free |
| 8 | `ropper -f ./vuln --search "syscall" --type sys --nocolor` | Syscall gadgets |
| 9 | `ropper -f ./vuln --instructions "jmp rsp" --nocolor` | Instruction match |
| 10 | `ropper -f ./vuln --opcode c3 --nocolor` | Opcode search |

**Research questions**
- Which gadgets have unwanted side effects (memory writes, register clobber)?
- How does `--quality` rank search results, and is the ranking trustworthy?

**Sources:** ropper source, disassembly, debugger validation.
**Output:** a shortlist of candidate gadgets.

---

## Scenario 3 — pivots and bad bytes

| # | Command | How to use |
|---|---------|------------|
| 11 | `ropper -f ./vuln --stack-pivot --nocolor` | All pivots |
| 12 | `ropper -f ./vuln -p --nocolor` | `pop; pop; ret` |
| 13 | `ropper -f ./vuln -j rsp --nocolor` | `jmp rsp` |
| 14 | `ropper -f ./vuln --search "leave; ret" --nocolor` | Classic leave pivot |
| 15 | `ropper -f ./vuln --search "xchg" -b 00,0a,0d --nocolor` | Filtered pivots |

**Research questions**
- Which pivots land in attacker-controlled memory given the overflow layout?
- How do bad-byte filters interact with multi-byte addresses?

**Sources:** exploit layout, debugger, ropper output.
**Output:** a viable pivot and a filtered gadget set.

---

## Scenario 4 — chain generation

| # | Command | How to use |
|---|---------|------------|
| 16 | `ropper -f ./vuln --chain "execve=/bin/sh" --nocolor` | execve chain |
| 17 | `ropper -f ./vuln --chain "mprotect address=0xdeadbeef size=0x10000" --nocolor` | mprotect chain |
| 18 | `ropper -f ./vuln --chain "virtualprotect address=0xdeadbeef"` | PE chain |
| 19 | `ropper -f ./vuln --set aslr --chain "execve=/bin/sh"` | ASLR assumption |
| 20 | `ropper -f ./vuln --set nx --chain "execve=/bin/sh"` | NX assumption |

**Research questions**
- What assumptions do the generators make about registers and stack layout?
- When is a hand-built chain preferable to a generated one?

**Sources:** generator source, pwntools ROP, lab validation.
**Output:** a draft chain to port into pwntools.

---

## Scenario 5 — validation and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `ropper -f ./vuln --search "pop rdi; ret" --nocolor > gadgets.txt` | Capture |
| 22 | `grep -E '^0x' gadgets.txt \| sort -u` | Unique addresses |
| 23 | `ropper -f ./vuln -I 0x555555554000 --search "pop rdi; ret"` | PIE base |
| 24 | `gdb -q ./vuln -ex 'start' -ex 'x/2i 0x...' -ex 'quit'` | Verify bytes |
| 25 | `sha256sum ./vuln gadgets.txt` | Hash evidence |

**Research questions**
- Which gadgets are reachable at runtime, and how is that proven?
- What must be recorded for another analyst to reproduce the chain?

**Sources:** GDB/pwndbg output, binary hash, notes.
**Output:** a validated, reproducible gadget inventory.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| File/arch | `-f -r -a -I` | 1, 5 |
| Inspect | `-i -e --imagebase -c -s -S --imports --symbols --section --string --hex` | 1 |
| Assembler | `--asm --disasm --disassemble-address` | 2 |
| Search | `--search --quality --opcode --instructions --type --inst-count -b -p -j --stack-pivot --detailed --all --cfg-only` | 2, 3 |
| Chains | `--chain --set --unset` | 4 |
| Semantic | `--semantic --analyse --count-of-findings` | 2 |
| Console | `--console --no-load --clear-cache --single --nocolor` | 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — fingerprint | 5 |
| 2 — search | 5 |
| 3 — pivots | 5 |
| 4 — chains | 5 |
| 5 — validation | 5 |
