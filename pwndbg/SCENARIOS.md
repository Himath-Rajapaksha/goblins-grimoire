# pwndbg — command scenarios for deep research

Installed build: pwndbg 2026.10.28 (a403136). The local `help.txt` snapshot only
records the version banner; commands below are from upstream pwndbg
documentation for this release line.

## How to invoke

```text
@deep-research For pwndbg 2026.10.28 on Debian 13 with GDB 16.3, which heap
commands remain accurate for glibc 2.36+ tcache/safe-linking, how should
`checksec`/`vmmap`/`telescope` be scripted in batch mode, and what conflicts
arise when GEF is also installed? Cite pwndbg docs, glibc malloc internals, and
issue reports; return a batch-debugging recipe and a heap-triage decision tree.
```

---

## Scenario 1 — session and mitigations

| # | Command | How to use |
|---|---------|------------|
| 1 | `gdb -q ./vuln` | Start session |
| 2 | `gdb -q ./vuln -ex checksec -ex quit` | Mitigations |
| 3 | `gdb -q ./vuln -ex vmmap -ex quit` | Memory map |
| 4 | `gdb -q ./vuln -ex elfinfo -ex quit` | ELF header |
| 5 | `gdb -q ./vuln -ex 'version' -ex quit` | pwndbg version |

**Research questions**
- Which mitigations does `checksec` report, and which are inferred rather than
  read from the binary?
- How do PIE and RELRO change the addresses shown by `vmmap`?

**Sources:** pwndbg docs, `readelf`, output.
**Output:** a target mitigation profile.

---

## Scenario 2 — crash context and offset discovery

| # | Command | How to use |
|---|---------|------------|
| 6 | `gdb -q ./vuln -ex run` | Reproduce crash |
| 7 | `context` | Full context |
| 8 | `telescope $rsp 40` | Stack dereference |
| 9 | `cyclic 300` | Generate pattern |
| 10 | `cyclic -l 0x6161616b` | Find offset |

**Research questions**
- Which register controls the return address, and what is the exact offset?
- Does the crash occur on a read, write, or control transfer?

**Sources:** context output, pattern math, backtrace.
**Output:** offset and control point.

---

## Scenario 3 — heap triage

| # | Command | How to use |
|---|---------|------------|
| 11 | `heap` | Heap overview |
| 12 | `bins` | Bin contents |
| 13 | `vis_heap_chunks` | Visual chunks |
| 14 | `malloc_chunk <addr>` | Decode a chunk |
| 15 | `try_free <addr>` | Simulate free checks |

**Research questions**
- Which tcache/smallbin state is present at crash time?
- How do safe-linking and tcache keying change what `vis_heap_chunks` shows?

**Sources:** glibc malloc source, pwndbg docs, output.
**Output:** a heap state description and exploitation hypothesis.

---

## Scenario 4 — ROP and binary introspection

| # | Command | How to use |
|---|---------|------------|
| 16 | `rop --grep "pop rdi"` | Gadget search |
| 17 | `got` | GOT entries |
| 18 | `plt` | PLT entries |
| 19 | `breakrva 0x1234` | PIE-relative break |
| 20 | `search "/bin/sh"` | String search |

**Research questions**
- Does `rop` use the same gadget rules as ropper/ROPgadget?
- Which GOT entry is a viable leak target?

**Sources:** pwndbg docs, binary, cross-tool output.
**Output:** validated gadgets and leak targets.

---

## Scenario 5 — automation and remote

| # | Command | How to use |
|---|---------|------------|
| 21 | `gdb -q ./vuln -batch -ex checksec -ex 'info functions'` | Batch script |
| 22 | `gdb -q -ex 'attachp myapp' -ex context` | Attach |
| 23 | `gdb -q ./vuln -ex 'target remote :1234'` | gdbserver |
| 24 | `python3 -c 'from pwn import *; ...'` | pwntools gdb.debug |
| 25 | `gdb -q ./vuln -ex 'set logging file gdb.log' -ex 'set logging on'` | Log evidence |

**Research questions**
- Which commands behave differently in `-batch` mode?
- How should remote sessions be secured and scoped?

**Sources:** GDB manual, pwntools docs, pwndbg issues.
**Output:** reproducible batch logs and remote sessions.

## Command coverage checklist

| Group | Commands | Scenario |
|-------|----------|----------|
| Context | `pwndbg context vmmap` | 1, 2 |
| Execution | `start run nextcall nextret stepuntil breakrva` | 2, 4 |
| Memory | `telescope search stack regs xinfo` | 2, 4 |
| Heap | `heap bins vis_heap_chunks malloc_chunk try_free` | 3 |
| Binary | `checksec got plt elfinfo pie libcinfo` | 1, 4 |
| Exploit | `cyclic rop asm shellcode attachp decomp` | 2, 4, 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — session | 5 |
| 2 — crash | 5 |
| 3 — heap | 5 |
| 4 — ROP/introspection | 5 |
| 5 — automation | 5 |
