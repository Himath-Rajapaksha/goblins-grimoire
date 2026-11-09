# GEF — GDB Enhanced Features for exploit development

- **Version:** GEF for linux, loaded in GDB 16.3 with Python 3.13 (from the
  installed `help.txt` banner)
- **Binary:** loaded by GDB; commands are GDB commands
- **Help snapshot:** [`help.txt`](help.txt) — captures the startup banner and
  the `gef` command list (90 commands loaded); per-command help is available
  interactively with `help gef <command>`.
- **Upstream:** https://github.com/hugsy/gef
- **Docs:** https://hugsy.github.io/gef/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

GEF is a GDB extension aimed at exploit developers and reverse engineers. It
adds a context dashboard, memory and process introspection, heap analysis,
ROP gadget search, pattern generation, checksec-style mitigation checks,
shellcode and assembler helpers, and remote-debugging conveniences. It is
lighter than pwndbg and is commonly used where a minimal, scriptable extension
is preferred.

## Use cases

| Situation | Why GEF |
|-----------|---------|
| Crash triage | Context view plus registers/stack/disassembly |
| Mitigation review | `checksec` |
| Heap work | `heap`, `bins`, `vis`, `arena` |
| ROP | `rop` search with filters |
| Pattern offsets | `pattern create/search` |
| Memory patching | `patch`, `set-permission`, `nop` |
| Remote debugging | `gef-remote` |
| Scripting | GEF Python API |

## Requirements & install layout

- Installed at `/opt/gef/gef.py` for GDB 16.3. It is not auto-loaded: the
  account `.gdbinit` sources pwndbg only, so GEF is started explicitly with
  `gdb -q -ex 'source /opt/gef/gef.py' ./vuln`.
- Do not source GEF and allow pwndbg to auto-load in the same session; the two
  extensions conflict.
- `gef install` fetches scripts from `gef-extras` over the network.
- `gef save`/`gef restore` persist configuration.

## Basic syntax

```bash
# start GDB with GEF
gdb -q -ex 'source /opt/gef/gef.py' ./vuln

# inside GDB
gef help
gef config
context
checksec
vmmap
telescope $rsp 20
heap
bins
pattern create 200
pattern search 0x6161616b
rop --grep "pop rdi"
```

## Command reference

### GEF meta-commands

| Command | Meaning |
|---------|---------|
| `gef help` | List commands |
| `gef config` | Show/change settings |
| `gef save` / `gef restore` | Persist/restore settings |
| `gef install <script>` | Install a gef-extras script |
| `gef missing` | List missing optional dependencies |
| `gef run` | Run with GEF context |
| `gef set` | Set GDB variables through GEF |

### Context and process

| Command | Meaning |
|---------|---------|
| `context` | Registers, stack, code, threads |
| `vmmap` | Memory mappings |
| `xinfo <addr>` | Address details |
| `process-status` / `process-search` | Process state and search |
| `registers` | Register view |
| `telescope <addr> [n]` | Dereference chain |
| `memory <addr> <size>` | Read memory |
| `hexdump <addr>` | Hex dump |

### Binary and mitigations

| Command | Meaning |
|---------|---------|
| `checksec` | Mitigation summary |
| `elf-info` | ELF header |
| `got` | GOT entries |
| `functions` | Function list |
| `ksymaddr <symbol>` | Kernel symbol address |
| `pie` | PIE information |
| `libc` / `glibc` | libc identification |

### Heap

| Command | Meaning |
|---------|---------|
| `heap` | Heap overview |
| `bins` | Bin contents |
| `arena` | Arena details |
| `vis` | Visual heap view |
| `try_free <addr>` | Free simulation |

### Exploit helpers

| Command | Meaning |
|---------|---------|
| `pattern create <n>` | De Bruijn pattern |
| `pattern search <value>` | Offset lookup |
| `rop [--grep <re>]` | Gadget search |
| `shellcode` | Shellcode helper |
| `nop` | Patch with NOPs |
| `patch <what> <value>` | Patch memory/registers |
| `set-permission <addr> <perm>` | Change page permissions |
| `gef-remote <host:port>` | Remote debugging |
| `format-string-helper` | Detect format-string bugs |
| `highlight` / `theme` | Appearance |
| `gef install` | Install extras |

## Recipes

```bash
# 1. Start GEF and confirm
gdb -q -ex 'source /opt/gef/gef.py' ./vuln -ex 'gef help' -ex quit

# 2. Mitigation profile
gdb -q -ex 'source /opt/gef/gef.py' ./vuln -ex checksec -ex quit

# 3. Crash context
gdb -q -ex 'source /opt/gef/gef.py' ./vuln -ex run -ex context

# 4. Pattern offset
gdb -q -ex 'source /opt/gef/gef.py' ./vuln -ex 'pattern create 300' -ex quit

# 5. Heap triage
gdb -q -ex 'source /opt/gef/gef.py' ./vuln -ex run -ex heap -ex bins

# 6. ROP search
gdb -q -ex 'source /opt/gef/gef.py' ./vuln -ex 'rop --grep "pop rdi"'

# 7. Remote session
gdb -q -ex 'source /opt/gef/gef.py' -ex 'gef-remote 127.0.0.1:1234'

# 8. Persist settings
gdb -q -ex 'source /opt/gef/gef.py' -ex 'gef config context.nb_lines_stack 12' -ex 'gef save'
```

## Output & pipelines

- `context` is interactive; `gdb -batch -ex ...` captures individual command
  output for logs.
- `checksec`, `vmmap`, and `ropper` output are report-friendly.
- `pattern create` output feeds the target; `pattern search` consumes the
  crash value.
- GEF's Python API can drive custom commands.

## Gotchas & operational security

- GEF and pwndbg must not both auto-load; conflicts produce confusing errors.
- `gef install` downloads from GitHub — do it deliberately and pin versions.
- Heap parsing on corrupted heaps is approximate.
- `set-permission` changes the debuggee's memory protections; never use it on
  a process you do not own.
- `gef-remote` connects in plaintext unless tunnelled; use SSH port forwards.
- ASLR invalidates addresses between runs.
- Attaching pauses the process; avoid production.
- `pattern search` needs the exact byte order of the crash value.

## Legal & safety

Use only on binaries and processes you own or are authorized to analyze.
Remote debugging and memory patching can crash services and expose secrets;
contain all work in an isolated lab.

## Related tools

- [`pwndbg/`](../pwndbg/README.md) — the other major GDB extension
- [`ropper/`](../ropper/README.md), [`ropgadget/`](../ropgadget/README.md)
- [`one-gadget/`](../one-gadget/README.md)
- [`pwntools/`](../pwntools/README.md)

## References

- https://hugsy.github.io/gef/
- https://github.com/hugsy/gef
- https://github.com/hugsy/gef-extras
