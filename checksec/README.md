# checksec — ELF mitigation inspector (pwntools build)

- **Version:** pwntools 4.15.0 (the `checksec` entry point ships with pwntools)
- **Binary:** `/usr/local/bin/checksec` → `/opt/pipx/venvs/pwntools/bin/checksec`
- **Python package:** `/opt/pipx/venvs/pwntools/lib/python3.13/site-packages/pwnlib`
- **CLI source:** `pwnlib/commandline/checksec.py`
- **Implementation:** `pwnlib/elf/elf.py` (`ELF.checksec`, `ELF.relro`, …)
- **Help snapshot:** [`help.txt`](help.txt) — `checksec --help`
- **Upstream:** https://github.com/Gallopsled/pwntools
- **Licence:** MIT (see `/usr/share/doc/pwntools/copyright` if packaged; pipx install carries the upstream licence)

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

> **This is not `checksec.sh`.** The classic `slimm609/checksec.sh` script
> accepts `--file=`, `--output=json`, `--fortify-file`, `--kernel`, and
> `--proc-all`. This build is pwntools' Python re-implementation: it accepts
> positional paths and `--file [elf ...]` only, prints a fixed six-to-ten-line
> block per binary, and has **no JSON, XML, or kernel mode**. `--file` exists
> solely for command-line compatibility with the shell script.

## What it does

`checksec` opens an ELF file, reads its headers, program headers, dynamic
section, symbol tables, and GNU property notes, and reports the mitigation
posture that matters when you are deciding whether a crash is exploitable:

- **RELRO** — is the GOT made read-only before user code runs?
- **Stack canary** — is there a `__stack_chk_fail` reference / canary setup?
- **NX** — is the stack marked non-executable (`GNU_STACK`)?
- **PIE** — is the main executable position-independent (`ET_DYN`)?
- **RPATH / RUNPATH** — does the loader search attacker-writable paths?
- **FORTIFY** — were checked libc variants (`*_chk`) linked?
- **Stripped / Debuginfo** — are symbol and DWARF sections present?
- **Packer** — does the file start with a UPX marker?
- **Sanitizers** — ASAN, MSAN, UBSAN instrumentation.
- **CET** — SHSTK (shadow stack) and IBT (indirect branch tracking) notes.

It is a triage tool, not a proof. A "Full RELRO" binary can still be
vulnerable, and "No canary" only means the compiler did not emit one — a
hand-written assembly routine or a JIT page can still be exploitable.

## Output fields

| Field | Values | Meaning |
|-------|--------|---------|
| `Arch` | `amd64-64-little`, `i386-32-little`, `aarch64-64-little`, … | pwntools architecture string |
| `RELRO` | `No RELRO`, `Partial RELRO`, `Full RELRO` | `PT_GNU_RELRO` present, plus `DF_BIND_NOW`/`DF_1_NOW` for Full |
| `Stack` | `Canary found`, `No canary found` | `__stack_chk_fail` / canary references |
| `NX` | `NX enabled`, `NX disabled`, `NX unknown - GNU_STACK missing` | `PT_GNU_STACK` execute bit |
| `PIE` | `PIE enabled`, `No PIE (0x400000)` | `ET_DYN` vs `ET_EXEC`; the address is the load base |
| `RPATH` | `b'/opt/lib'` | `DT_RPATH` (deprecated search path) |
| `RUNPATH` | `b'/opt/lib'` | `DT_RUNPATH` (modern search path) |
| `Stripped` | `No` (omitted when stripped) | absence of `SHT_SYMTAB` |
| `FORTIFY` | `Enabled` (omitted when absent) | any dynamic symbol ending `_chk` |
| `Stack` (second line) | `Executable` | `GNU_STACK` is `RWE` |
| `RWX` | `Has RWX segments` | any `PT_LOAD` with write+execute |
| `Packer` | `Packed with UPX` | `UPX!` in the first `0xFF` bytes |
| `ASAN` / `MSAN` / `UBSAN` | `Enabled` | sanitizer runtime symbols |
| `SHSTK` / `IBT` | `Enabled` | x86 GNU property note bits |
| `Debuginfo` | `Yes` | `.debug_info` section present |

Fields that do not apply are simply absent. A stripped, non-fortified,
non-packed binary prints a shorter block — there is no `Stripped: Yes` line.

## Usage

```bash
checksec /bin/ls
checksec --file /bin/ls
checksec /bin/ls /bin/cat /usr/bin/ssh
checksec '/bin/l?'
```

- Positional arguments and `--file` are equivalent; both accept multiple paths.
- Shell globs are expanded by the shell before `checksec` sees them.
- Output goes to stdout; pwntools colour codes are emitted only when stdout is
  a TTY. Piping to `cat` or a file produces plain text.
- The block is prefixed with `[*] '<path>'`; that prefix is pwntools logging,
  not part of the field data.

### Exit codes

| Situation | Exit |
|-----------|------|
| ELF file checked | 0 |
| Non-ELF file (`Magic number does not match`) | 0 |
| Directory argument | 2 |
| Missing path | 2 |
| No arguments (prints usage) | 0 |
| `--output=json` (unrecognised) | 2 |
| `--file` with no value | 0 |

The non-ELF case is the dangerous one for scripting: `checksec /etc/hostname`
prints an error-like line but **exits 0**, so `set -e` will not catch it.
Check for the `Magic number does not match` string or pre-filter with `file`.

## Recipes

**1. Triage a single binary.**

```bash
checksec ./target
```

**2. Triage a directory of candidates without stopping on non-ELFs.**

```bash
find . -maxdepth 1 -type f -exec sh -c '
  for f; do checksec "$f" 2>/dev/null | grep -q "Magic number" || checksec "$f"; done
' sh {} +
```

**3. Build a comparison matrix of a hardening change.**

```bash
for f in build/before/app build/after/app; do
  printf '%s\n' "== $f"
  checksec "$f" | sed 's/^/  /'
done
```

**4. Find every non-PIE executable in a root filesystem.**

```bash
find /usr/bin /usr/sbin -type f -perm -111 -exec sh -c '
  for f; do checksec "$f" 2>/dev/null | grep -q "No PIE" && echo "$f"; done
' sh {} +
```

**5. Find every binary with an RPATH or RUNPATH.**

```bash
find /opt /usr/local -type f -perm -111 -exec sh -c '
  for f; do checksec "$f" 2>/dev/null | grep -qE "RPATH|RUNPATH" && echo "$f"; done
' sh {} +
```

**6. Confirm that a build really is Full RELRO.**

```bash
checksec ./app | grep -q 'Full RELRO' && echo hardened || echo not-hardened
readelf -lW ./app | grep GNU_RELRO
readelf -dW ./app | grep -E 'BIND_NOW|FLAGS'
```

**7. Check a shared library (PIE is not meaningful there).**

```bash
checksec /lib/x86_64-linux-gnu/libc.so.6
```

`libc.so.6` is `ET_DYN`, so pwntools prints `PIE enabled`. For a `.so` that
line says "loadable at an arbitrary base", not "built as a PIE executable".

**8. Get the same data from Python for a report.**

```bash
/opt/pipx/venvs/pwntools/bin/python - <<'PY'
from pwn import ELF, context
context.log_level = 'error'
e = ELF('/bin/ls')
print(e.path, e.arch, e.bits, e.endian)
print('RELRO', e.relro, 'canary', e.canary, 'NX', e.nx, 'PIE', e.pie)
print('FORTIFY', e.fortify, 'stripped', e.stripped, 'debuginfo', e.debuginfo)
print('execstack', e.execstack, 'rwx', e.rwx_segments, 'packed', e.packed)
print('SHSTK', e.shstk, 'IBT', e.ibt, 'rpath', e.rpath, 'runpath', e.runpath)
PY
```

The system `python3` cannot import pwntools in this install; use the pipx venv
interpreter shown above.

## How each field is computed

The definitions are worth knowing because they explain the false positives.

| Field | Source of truth in `pwnlib/elf/elf.py` |
|-------|----------------------------------------|
| RELRO | `PT_GNU_RELRO` presence; Full requires `DF_BIND_NOW` or `DF_1_NOW` |
| Canary | `__stack_chk_fail` in dynamic symbols / relocations |
| NX | `PT_GNU_STACK` `p_flags & PF_X`; missing segment reports "unknown" |
| PIE | `e_type == ET_DYN` |
| FORTIFY | any dynamic symbol whose name ends in `_chk` |
| Stripped | no `SHT_SYMTAB` section |
| Debuginfo | `.debug_info` section present |
| Packer | `UPX!` in the first `0xFF` bytes |
| SHSTK / IBT | `GNU_PROPERTY_X86_FEATURE_1_AND` note bits |
| RWX | any `PT_LOAD` with `PF_W | PF_X` |

Two consequences:

- **FORTIFY is symbol-based.** A binary that uses `strcpy` but was compiled
  without `_FORTIFY_SOURCE` has no `__strcpy_chk` and reports no FORTIFY,
  even if other checked calls are present. The reverse is also possible:
  a single `_chk` symbol enables the line.
- **Canary is reference-based.** A statically linked binary with its own
  `__stack_chk_fail` may report a canary even if no function uses one.

## Limits and quirks

- **No machine-readable output.** There is no `--output=json`. Parse the text
  or use the Python API. `checksec --output=json` exits 2.
- **Non-ELF exits 0.** Scripts must inspect the text, not only `$?`.
- **Directories exit 2.** argparse's `FileType('rb')` rejects them before the
  ELF parser runs.
- **Static binaries** can report `NX unknown - GNU_STACK missing`; the absence
  of a `PT_GNU_STACK` segment is not proof of an executable stack.
- **`-z execstack` on modern GCC** may still emit a `GNU_STACK` segment with
  `RWE`; the extra `Stack: Executable` and `RWX` lines appear only when the
  linker/loader combination produces them.
- **CET notes are toolchain-dependent.** `-fcf-protection=full` alone did not
  produce SHSTK/IBT in this build; `-Wl,-z,ibt,-z,shstk` did.
- **Sanitizer lines** indicate the runtime is linked, not that the binary was
  built with useful coverage.
- **Version drift.** pwntools changes field names and detection heuristics
  between releases. Record the version with every report.

## Related tools

- [`pwntools`](../pwntools/) — the library this command belongs to; `ELF`,
  `ROP`, `pwn checksec`.
- [`gef`](../gef/) — GDB enhancement with its own `checksec` command at
  runtime.
- [`pwndbg`](../pwndbg/) — GDB enhancement with `checksec`/`got`/`vmmap`.
- [`one-gadget`](../one-gadget/) — find `execve("/bin/sh")` one-gadget offsets.
- [`ropgadget`](../ropgadget/) — gadget search once mitigations are known.
- [`ropper`](../ropper/) — alternative gadget finder and ELF inspector.
- `readelf`, `objdump`, `file`, `ldd` — the underlying binutils views.
