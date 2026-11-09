# GEF — command scenarios for deep research

Installed build: GEF for Linux in GDB 16.3 / Python 3.13, loaded from
`/opt/gef/gef.py`. The `help.txt` snapshot lists 90 loaded commands; per-command
help comes from `help gef <command>`.

## How to invoke

```text
@deep-research For GEF on GDB 16.3 (2026), which commands remain reliable for
glibc 2.36+ heap analysis and PIE binaries, how do GEF and pwndbg differ in
checksec/rop/heap behaviour, and what is the safest way to run both without
conflict? Cite GEF docs, gef-extras, and issue reports; return a comparison
matrix and a canonical debugging recipe.
```

---

## Scenario 1 — load and configure

| # | Command | How to use |
|---|---------|------------|
| 1 | `gdb -q -ex 'source /opt/gef/gef.py' ./vuln` | Start with GEF |
| 2 | `gef help` | Command list |
| 3 | `gef missing` | Missing dependencies |
| 4 | `gef config` | Current settings |
| 5 | `gef save` | Persist configuration |

**Research questions**
- Which optional dependencies are missing, and what breaks without them?
- Where is the GEF config stored, and how is it restored?

**Sources:** GEF docs, `gef missing`, config file.
**Output:** a working, reproducible GEF setup.

---

## Scenario 2 — crash context

| # | Command | How to use |
|---|---------|------------|
| 6 | `gdb -q -ex 'source /opt/gef/gef.py' ./vuln -ex run` | Reproduce crash |
| 7 | `context` | Full dashboard |
| 8 | `telescope $rsp 30` | Stack dereference |
| 9 | `registers` | Registers |
| 10 | `xinfo $pc` | Code mapping |

**Research questions**
- Which register controls the return address?
- Does the fault come from a read, write, or jump?

**Sources:** context output, backtrace.
**Output:** crash classification and control point.

---

## Scenario 3 — mitigations and introspection

| # | Command | How to use |
|---|---------|------------|
| 11 | `checksec` | Mitigation summary |
| 12 | `elf-info` | ELF header |
| 13 | `got` | GOT entries |
| 14 | `vmmap` | Memory map |
| 15 | `functions` | Function list |

**Research questions**
- Which mitigations are reported, and which are inferred?
- Which GOT entry is a viable leak?

**Sources:** GEF docs, `readelf`, output.
**Output:** a mitigation and leak-target profile.

---

## Scenario 4 — heap and patterns

| # | Command | How to use |
|---|---------|------------|
| 16 | `heap` | Heap overview |
| 17 | `bins` | Bin contents |
| 18 | `vis` | Visual heap |
| 19 | `pattern create 300` | De Bruijn pattern |
| 20 | `pattern search 0x6161616b` | Offset lookup |

**Research questions**
- How does GEF's heap model handle tcache and safe-linking?
- Does `pattern search` accept decimal, hex, and byte order consistently?

**Sources:** glibc source, GEF docs, output.
**Output:** heap state and exact crash offset.

---

## Scenario 5 — ROP, remote, and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `rop --grep "pop rdi"` | Gadget search |
| 22 | `gdb -q -ex 'source /opt/gef/gef.py' -ex 'gef-remote 127.0.0.1:1234'` | Remote |
| 23 | `gdb -q -batch -ex 'source /opt/gef/gef.py' -ex checksec ./vuln` | Batch |
| 24 | `set logging file gef.log` / `set logging on` | Evidence log |
| 25 | `sha256sum gef.log ./vuln` | Hash evidence |

**Research questions**
- How should `gef-remote` be tunnelled, and what authentication does it lack?
- Which commands produce stable output in `-batch` mode?

**Sources:** GDB manual, GEF docs, logs.
**Output:** validated gadgets, a remote session, and hashed evidence.

## Command coverage checklist

| Group | Commands | Scenario |
|-------|----------|----------|
| Meta | `gef help config save restore install missing run set` | 1 |
| Context | `context vmmap xinfo telescope registers memory` | 2, 3 |
| Binary | `checksec elf-info got functions pie libc` | 3 |
| Heap | `heap bins arena vis try_free` | 4 |
| Exploit | `pattern rop patch nop set-permission shellcode` | 4, 5 |
| Remote | `gef-remote` | 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — load/configure | 5 |
| 2 — crash context | 5 |
| 3 — mitigations | 5 |
| 4 — heap/patterns | 5 |
| 5 — ROP/remote | 5 |
