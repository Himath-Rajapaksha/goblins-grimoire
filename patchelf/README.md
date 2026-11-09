# patchelf — modify ELF interpreter, RPATH, NEEDED, and related fields

- **Version:** 0.18.0 (Debian `patchelf` `0.18.0-1.4`)
- **Binary:** `/usr/bin/patchelf`
- **Help snapshot:** [`help.txt`](help.txt) — `patchelf` (no arguments)
- **Man page:** `man patchelf`
- **Upstream:** https://github.com/NixOS/patchelf
- **Licence:** GPL-3.0-or-later (see `/usr/share/doc/patchelf/copyright`)

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

> **No-validation warning:** patchelf does not check that the interpreter,
> NEEDED library, or page size you give it exists or makes sense. A typo in
> `--set-interpreter` produces an ELF that the kernel refuses to execute, and
> the command still exits 0. Keep a copy and test the result.

## What it does

patchelf rewrites selected fields of an ELF file:

- the **program interpreter** (`PT_INTERP`, the dynamic loader path);
- the **RPATH / RUNPATH** search path (`DT_RPATH`, `DT_RUNPATH`);
- the **NEEDED** list of shared libraries (`DT_NEEDED`);
- the **SONAME** (`DT_SONAME`);
- the **OS ABI** byte in the ELF header (`EI_OSABI`);
- the **executable-stack** request (`PT_GNU_STACK` flags);
- **symbol versions** and **dynamic symbol names**.

It is the standard tool for making a binary built for one prefix run from
another — Nix, Guix, AppImage, and many packaging systems use it. It is also
used to patch challenge binaries, which is why the recipes below include
verification steps.

patchelf rewrites the file in place by default. `--output FILE` writes a
patched copy and leaves the input alone. Because it moves sections and can
grow the file, the result is not byte-identical and any signature or hash
covering the original is invalidated.

## Basic syntax

```bash
patchelf --print-interpreter ./app
patchelf --set-interpreter /lib64/ld-linux-x86-64.so.2 ./app
patchelf --print-rpath ./app
patchelf --set-rpath '$ORIGIN/lib' ./app
patchelf --print-needed ./app
patchelf --replace-needed libfoo.so.1 libfoo.so.2 ./app
patchelf --output ./app.patched --set-rpath '$ORIGIN/lib' ./app
```

Multiple actions can appear in one invocation. They are processed left to
right, so `--print-rpath --set-rpath /opt/lib` prints the old value and then
sets it; `--set-rpath /opt/lib --print-rpath` prints the new value.

## Flag reference

### Interpreter and OS ABI

| Flag | Meaning |
|------|---------|
| `--print-interpreter` | Print `PT_INTERP` |
| `--set-interpreter FILE` | Set the dynamic loader path |
| `--print-os-abi` | Print `EI_OSABI` |
| `--set-os-abi ABI` | Set `EI_OSABI`; accepts `sysv`, `linux`, `freebsd`, `solaris` |
| `--page-size SIZE` | Page size used when growing the file |

### RPATH / RUNPATH

| Flag | Meaning |
|------|---------|
| `--print-rpath` | Print the current RPATH/RUNPATH (empty line if none) |
| `--set-rpath PATH` | Replace the search path |
| `--add-rpath PATH` | Append to the search path |
| `--remove-rpath` | Remove RPATH and RUNPATH |
| `--shrink-rpath` | Drop entries that contain no needed library |
| `--allowed-rpath-prefixes PREFIXES` | With `--shrink-rpath`, keep only entries under these prefixes |
| `--force-rpath` | Write `DT_RPATH` instead of `DT_RUNPATH` |

### NEEDED and SONAME

| Flag | Meaning |
|------|---------|
| `--print-needed` | Print the `DT_NEEDED` list |
| `--add-needed LIB` | Add a NEEDED entry |
| `--remove-needed LIB` | Remove a NEEDED entry |
| `--replace-needed OLD NEW` | Replace one NEEDED entry |
| `--no-default-lib` | Set `DF_1_NODEFLIB` so the loader ignores default library search paths |
| `--print-soname` | Print `DT_SONAME` (empty + exit 0 if absent) |
| `--set-soname NAME` | Set `DT_SONAME` |

### Hardening and symbols

| Flag | Meaning |
|------|---------|
| `--print-execstack` | Print whether an executable stack is requested |
| `--set-execstack` | Request an executable stack |
| `--clear-execstack` | Request a non-executable stack |
| `--clear-symbol-version SYMBOL` | Remove the version from a dynamic symbol |
| `--rename-dynamic-symbols MAPFILE` | Rename dynamic symbols from a map file |
| `--add-debug-tag` | Add `DT_DEBUG` if missing |
| `--no-sort` | Do not sort program/section headers (debugging) |
| `--debug` | Print rewrite diagnostics |
| `--output FILE` | Write a patched copy |
| `--version` | Print the version |

## RPATH vs RUNPATH

`--set-rpath` writes **RUNPATH** by default on this build. `--force-rpath`
writes the older **RPATH**. The difference matters at load time:

| Tag | Search order |
|-----|--------------|
| `DT_RPATH` | RPATH → `LD_LIBRARY_PATH` → RUNPATH → cache → default |
| `DT_RUNPATH` | `LD_LIBRARY_PATH` → RUNPATH → cache → default |

RPATH therefore takes precedence over `LD_LIBRARY_PATH` and is harder to
override; RUNPATH is the modern default. Use `readelf -d` to confirm which tag
you actually produced. `--print-rpath` prints the value without telling you
which tag it came from.

## Recipes

**1. Inspect a binary before changing it.**

```bash
patchelf --print-interpreter ./app
patchelf --print-rpath ./app
patchelf --print-needed ./app
patchelf --print-execstack ./app
```

**2. Point a binary at a different loader.**

```bash
cp app app.bak
patchelf --set-interpreter /lib64/ld-linux-x86-64.so.2 app
patchelf --print-interpreter app
./app
```

**3. Make a binary find libraries next to itself.**

```bash
patchelf --set-rpath '$ORIGIN/lib' app
patchelf --print-rpath app
readelf -d app | grep -E 'RPATH|RUNPATH'
```

**4. Prefer RUNPATH explicitly (the default) or force RPATH.**

```bash
patchelf --set-rpath /opt/lib app          # writes RUNPATH
patchelf --force-rpath --set-rpath /opt/lib app   # writes RPATH
```

**5. Add a fallback directory.**

```bash
patchelf --add-rpath /usr/local/lib app
patchelf --print-rpath app
```

**6. Remove entries that do not contain a needed library.**

```bash
patchelf --set-rpath '/lib/x86_64-linux-gnu:/tmp/nowhere' app
patchelf --shrink-rpath app
patchelf --print-rpath app
```

**7. Restrict shrinking to a prefix.**

```bash
patchelf --set-rpath '/opt/lib:/usr/lib' app
patchelf --shrink-rpath --allowed-rpath-prefixes /opt app
patchelf --print-rpath app
```

**8. Replace a library dependency.**

```bash
patchelf --replace-needed libfoo.so.1 libfoo.so.2 app
patchelf --print-needed app
```

**9. Add a dependency (patchelf does not check that it exists).**

```bash
patchelf --add-needed libm.so.6 app
patchelf --print-needed app
```

**10. Mark the object to ignore default library search paths.**

```bash
patchelf --no-default-lib app
readelf -dW app | grep FLAGS_1     # expect NODEFLIB
```

`--no-default-lib` does **not** remove `libc.so.6` from `DT_NEEDED`; it sets
the `DF_1_NODEFLIB` flag so the loader skips the default paths.

**11. Set a SONAME on a shared library.**

```bash
patchelf --set-soname libfoo.so.1 libfoo.so
patchelf --print-soname libfoo.so
```

**12. Clear the executable-stack request.**

```bash
patchelf --clear-execstack app
patchelf --print-execstack app
readelf -lW app | grep GNU_STACK
```

**13. Patch a copy and keep the original.**

```bash
patchelf --set-rpath '$ORIGIN/lib' --output app.patched app
patchelf --print-rpath app.patched
```

**14. Remove a symbol version.**

```bash
patchelf --clear-symbol-version puts app
readelf -sW app | grep puts
```

`--clear-symbol-version` patches `.dynsym`; the same symbol in `.symtab` keeps
its `@GLIBC_x.y` suffix until you strip or re-link the binary.

**15. Rename dynamic symbols from a map file.**

```bash
printf 'puts puts_alias\n' > rename.map
patchelf --rename-dynamic-symbols rename.map app
readelf -sW app | grep -E 'puts'
```

The map format is `old new`, one pair per line, **space-separated**. A colon
(`puts:puts_alias`) fails with `Map file line is missing the second element`.

## Exit status

| Situation | Exit |
|-----------|------|
| Successful read or write | 0 |
| Missing file | 1 |
| Directory argument | 1 |
| Not an ELF (`missing ELF header`) | 1 |
| Invalid `--set-os-abi` | 1 |
| `--print-soname` with no SONAME | **0** (despite the help text saying it raises an error) |
| `--remove-needed` for an absent library | 0 |
| Invalid `--page-size` | 0 |
| Bad `--set-interpreter` path | 0 (the binary is now broken) |
| Bad `--add-needed` path | 0 (the entry is added anyway) |
| `--output` with no action | 0 (a copy is still written) |

**The zero exits on bad input are the important ones.** patchelf reports only
structural problems (not ELF, missing file, invalid ABI), not semantic ones.
Always run the patched binary and re-read the field afterwards.

## Quirks and limits

- **No validation of paths.** `--set-interpreter /nonexistent/ld.so` succeeds
  and produces a non-running binary. Test with `./app` or `ldd ./app`.
- **`--print-soname` does not error on a missing SONAME.** The help text is
  wrong; it prints nothing and exits 0.
- **`--add-debug-tag` adds `DT_DEBUG` only when absent.** Executables usually
  already have it (no-op); a shared library usually does not, so the tag is
  added there.
- **`--rename-dynamic-symbols` does not check the map file exists.** A missing
  path exits 0 with no change; a malformed line exits 1.
- **`--set-os-abi` rejects unknown names.** `hpux` and `none` exit 1 and leave
  the previous value in place; the accepted spellings are `sysv`, `linux`,
  `freebsd`, and `solaris`.
- **`--page-size` accepts nonsense.** A value like `123` is accepted silently;
  use a real page size (often `4096` or `65536`) only when the default fails.
- **`--shrink-rpath` can empty the path.** If no entry contains a needed
  library, the result is an empty RUNPATH. That may be correct, but check it.
- **`--force-rpath` changes loader semantics.** RPATH overrides
  `LD_LIBRARY_PATH`; that is a security-relevant difference.
- **Patching changes the file layout.** Sections move, the file grows, and
  hashes/signatures break. Keep the original.
- **Patching a running binary is unsafe.** Stop the process first.
- **Static binaries have no interpreter.** `--set-interpreter` on a static
  binary may add a `PT_INTERP` that the kernel then rejects; do not use it as
  a general-purpose "make static dynamic" tool.
- **Cross-architecture patching is not supported.** patchelf handles the ELF
  class it is given, but a wrong interpreter or library architecture produces
  a non-running binary.
- **`--debug` is verbose and slow.** It prints every section rewrite; use it
  when a patch fails unexpectedly.

## Related tools

- [`checksec`](../checksec/) — see how a patch changed RELRO/NX/PIE/RPATH.
- [`pwntools`](../pwntools/) — `ELF` parsing, patching helpers, and ROP.
- [`ghidra`](../ghidra/) — disassemble the patched binary.
- [`strace`](../strace/) — see which loader and libraries are actually used.
- [`binwalk`](../binwalk/) — inspect firmware images before patching.
- `readelf`, `objdump`, `ldd`, `ld.so` — the authoritative ELF views.
