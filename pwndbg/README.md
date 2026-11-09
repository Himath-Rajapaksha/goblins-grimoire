# pwndbg — GDB extension for exploit development

- **Version:** 2026.10.28 build a403136
- **Binary:** loaded by GDB via `~/.gdbinit`; commands are GDB commands
- **Help snapshot:** [`help.txt`](help.txt) — the installed snapshot contains
  only the version banner, so this page is grounded in upstream pwndbg
  documentation and the commands listed there, not in a local `help` capture.
- **Upstream:** https://github.com/pwndbg/pwndbg
- **Docs:** https://pwndbg.re/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

pwndbg is a GDB plugin that replaces the default stop prompt with a structured
context view: registers, stack, disassembly, backtrace, and thread state. It
adds heap analysis (`heap`, `bins`, `vis_heap_chunks`, `malloc_chunk`), memory
inspection (`telescope`, `vmmap`, `search`), binary introspection (`checksec`,
`got`, `plt`, `elfinfo`), exploit helpers (`cyclic`, `rop`, `shellcode`,
`asm`), and remote/pwntools integration (`attachp`, `decomp`).

## Use cases

| Situation | Why pwndbg |
|-----------|------------|
| Crash triage | Context view shows registers, stack, and source at once |
| Heap exploitation | Bins, chunk visualization, tcache inspection |
| ROP validation | `rop` search and stack inspection |
| Mitigation check | `checksec` on the loaded binary |
| Pattern offsets | `cyclic` create/find |
| Remote debugging | `attachp`, gdbserver, pwntools |
| Binary introspection | GOT/PLT, symbols, sections |

## Requirements & install layout

- Installed for GDB 16.3 and auto-loads from the GDB init chain configured for
  this account.
- `gdb -q ./vuln` should print the pwndbg banner; `pwndbg` or `context` redraws.
- No special privileges; ptrace scope must allow debugging your own processes.
- `decomp` needs a decompiler backend (IDA/Ghidra/radare2/angr) installed and
  configured.

## Basic syntax

```bash
# start a session
gdb -q ./vuln

# inside GDB
pwndbg
context
checksec
vmmap
telescope $rsp 20
heap
bins
cyclic 200
cyclic -l 0x6161616b
rop --grep "pop rdi"
```

## Command reference

### Session and context

| Command | Meaning |
|---------|---------|
| `pwndbg` | Redraw the pwndbg context |
| `context` | Registers, stack, disassembly, backtrace |
| `reload` | Reload pwndbg |
| `theme` / `config` | Appearance and settings |
| `version` | pwndbg version |

### Execution

| Command | Meaning |
|---------|---------|
| `start` / `run` | Standard GDB execution |
| `nextcall` | Step to the next call |
| `nextjmp` | Step to the next jump |
| `nextret` | Step to the next return |
| `nextsyscall` | Step to the next syscall |
| `stepover` | Step over the current instruction |
| `stepuntil <addr>` | Run until an address |
| `xuntil <addr>` | Execute until an address |
| `breakrva <offset>` | Break at PIE-relative offset |

### Memory and stack

| Command | Meaning |
|---------|---------|
| `telescope <addr> [n]` | Recursively dereference memory |
| `vmmap` | Memory map with permissions |
| `xinfo <addr>` | Page and section info for an address |
| `search <value>` | Search memory |
| `stack [n]` | Stack view |
| `regs` | Register view |
| `dps <addr>` | Dump pointer-sized values |
| `hexdump <addr>` | Hex dump |
| `x/…` | Native GDB examine |

### Heap

| Command | Meaning |
|---------|---------|
| `heap` | Heap overview (chunks, arenas, tcache) |
| `bins` | Fastbin/smallbin/largebin/unsorted contents |
| `vis_heap_chunks` | Visual chunk map |
| `malloc_chunk <addr>` | Decode one chunk |
| `try_free <addr>` | Simulate `free()` checks |
| `arena` | Arena details |
| `heap_config` | Heap parsing settings |

### Binary introspection

| Command | Meaning |
|---------|---------|
| `checksec` | Mitigations (NX, PIE, canary, RELRO) |
| `got` / `plt` | GOT and PLT entries |
| `elfinfo` | ELF header details |
| `pie` | PIE base and breakpoints |
| `libcinfo` / `glibc` | libc version and offsets |
| `main` | Break at main |
| `entry` | Break at entry point |

### Exploit helpers

| Command | Meaning |
|---------|---------|
| `cyclic <n>` | Generate a De Bruijn pattern |
| `cyclic -l <value>` | Find the offset of a value |
| `rop [--grep <re>]` | Search ROP gadgets |
| `asm <insn>` | Assemble an instruction |
| `shellcode` | Shellcode helpers |
| `attachp <pid\|name>` | Attach to a process |
| `decomp <func>` | Decompile with a configured backend |

## Recipes

```bash
# 1. Start and confirm pwndbg
gdb -q ./vuln -ex 'checksec' -ex 'vmmap' -ex 'quit'

# 2. Find a crash offset
gdb -q ./vuln -ex 'run < <(python3 -c "print(\"A\"*300)")'
# then: cyclic 300, feed it, cyclic -l $rsp-value

# 3. Heap triage after a crash
gdb -q ./vuln -ex 'run' -ex 'heap' -ex 'bins' -ex 'vis_heap_chunks'

# 4. Inspect a suspected overflowed return address
gdb -q ./vuln -ex 'run' -ex 'telescope $rsp 40'

# 5. Search for a gadget
gdb -q ./vuln -ex 'rop --grep "pop rdi"'

# 6. Break at a PIE-relative offset
gdb -q ./vuln -ex 'start' -ex 'breakrva 0x1234' -ex 'continue'

# 7. Attach to a running process
gdb -q -ex 'attachp myapp' -ex 'context'

# 8. Use a pattern to locate the offset
python3 -c 'from pwn import cyclic; print(cyclic(300).decode())'
gdb -q ./vuln -ex 'run' -ex 'cyclic -l 0x6161616b'
```

## Output & pipelines

- The stop context is human-oriented; `gdb -batch -ex ...` output is scriptable.
- `checksec` output is useful in reports and CI logs.
- `vmmap`, `telescope`, and `heap` output feed exploit notes.
- pwntools `gdb.debug()` / `gdb.attach()` integrate with pwndbg automatically.

## Gotchas & operational security

- Auto-loading conflicts with GEF: only one GDB extension should be active.
  Check `~/.gdbinit` and `/etc/gdb/gdbinit.d/`.
- `decomp` silently fails without a backend.
- Heap parsing can misread corrupted heaps; treat `heap` output on a corrupted
  state as approximate.
- ASLR changes addresses between runs; use `breakrva` and re-check.
- ptrace may be restricted (`kernel.yama.ptrace_scope`); root or
  `CAP_SYS_PTRACE` may be needed for other users' processes.
- Attaching to production processes pauses them; never do so without approval.
- GDB can execute arbitrary code and files; do not run untrusted `.gdbinit`.

## Legal & safety

Debug only binaries and processes you own or are authorized to analyze. Live
attachment can pause or crash a service, and memory dumps may contain secrets
and personal data. Keep exploit work in an isolated lab.

## Related tools

- [`gef/`](../gef/README.md) — alternative GDB extension
- [`ropper/`](../ropper/README.md), [`ropgadget/`](../ropgadget/README.md) —
  offline gadget search
- [`one-gadget/`](../one-gadget/README.md) — glibc one-shot gadgets
- [`pwntools/`](../pwntools/README.md) — exploit automation

## References

- https://pwndbg.re/
- https://github.com/pwndbg/pwndbg
- https://browserpwndbg.readthedocs.io/ (older docs mirror)
