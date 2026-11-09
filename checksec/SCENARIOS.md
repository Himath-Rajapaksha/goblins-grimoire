# checksec — command scenarios for deep research

Installed build: pwntools 4.15.0 (`/usr/local/bin/checksec` →
`/opt/pipx/venvs/pwntools/bin/checksec`). Every flag below appears in
[`help.txt`](help.txt); field semantics come from `pwnlib/elf/elf.py`.

> **Scope:** these recipes inspect binaries on the local host and build small
> test programs with the system compiler. They do not execute the inspected
> binaries. Do not point the batch recipes at paths you are not authorised to
> read.

> **Interpreter note:** the system `python3` cannot import pwntools in this
> install. The Python-API scenario uses
> `/opt/pipx/venvs/pwntools/bin/python` explicitly.

## How to invoke

```text
@deep-research Using pwntools 4.15.0 checksec on Debian 13, explain how each
reported field (RELRO, canary, NX, PIE, RPATH/RUNPATH, FORTIFY, Stripped,
Debuginfo, Packer, ASAN/MSAN/UBSAN, SHSTK/IBT) is derived from ELF headers,
sections, symbols, and GNU property notes. Compare this pwntools build with
the classic checksec.sh interface, and show which compiler/linker flags move
each field. Cite the pwntools source in pwnlib/elf/elf.py, the shipped
help text, and observed compiler output. Separate confirmed behaviour from
inference and state confidence.
```

Create the workspace and the test program:

```bash
mkdir -p ~/checksec-demo/evidence
cd ~/checksec-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
cat > vuln.c <<'EOF'
#include <stdio.h>
#include <string.h>
int main(int argc, char **argv) {
    char buf[64];
    if (argc > 1) strcpy(buf, argv[1]);
    puts("ok");
    return 0;
}
EOF
```

## Scenario 1 — version, install layout, CLI surface

| # | Command | Purpose |
|---|---------|---------|
| 1 | `checksec --help` | Show the argparse surface |
| 2 | `checksec` | No arguments: usage on stdout, exit 0 |
| 3 | `checksec >/dev/null 2>&1; echo "exit=$?"` | Confirm the no-argument exit code |
| 4 | `command -v checksec` | Resolve the entry point |
| 5 | `readlink -f "$(command -v checksec)"` | Follow the pipx shim |
| 6 | `head -5 /opt/pipx/venvs/pwntools/bin/checksec` | Show the generated wrapper |
| 7 | `/opt/pipx/venvs/pwntools/bin/python -c 'import pwn; print(pwn.__version__)'` | Confirm the pwntools version |
| 8 | `pipx list \| grep -i pwntools` | Confirm the pipx environment |
| 9 | `sed -n '1,60p' /opt/pipx/venvs/pwntools/lib/python3.13/site-packages/pwnlib/commandline/checksec.py` | Read the CLI definition |
| 10 | `grep -n 'def checksec' /opt/pipx/venvs/pwntools/lib/python3.13/site-packages/pwnlib/elf/elf.py` | Locate the formatter |
| 11 | `grep -n 'def relro\|def canary\|def nx\|def pie' /opt/pipx/venvs/pwntools/lib/python3.13/site-packages/pwnlib/elf/elf.py` | Locate field implementations |
| 12 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt` | Timestamp the run |

**Research questions**
- Is this the shell `checksec.sh` or the pwntools Python command?
- Which interface compatibility flags does it actually implement?
- Where is the field logic, and which file is authoritative?

**Sources:** `pwnlib/commandline/checksec.py`; `pwnlib/elf/elf.py`; observed output.
**Output:** a version record and the exact CLI surface.

## Scenario 2 — baseline system binaries

| # | Command | Purpose |
|---|---------|---------|
| 13 | `checksec /bin/ls` | Dynamic PIE executable |
| 14 | `checksec /bin/cat` | Second dynamic executable for comparison |
| 15 | `checksec /bin/bash` | Large shell binary |
| 16 | `checksec /bin/busybox` | Static multi-call binary |
| 17 | `checksec /usr/bin/ssh` | Network-facing setuid-adjacent binary |
| 18 | `checksec /usr/bin/sudo` | Setuid binary |
| 19 | `checksec /usr/bin/passwd` | Setuid binary |
| 20 | `checksec /lib/x86_64-linux-gnu/libc.so.6` | Shared library: PIE line is not meaningful |
| 21 | `checksec --file /bin/ls` | `--file` compatibility form |
| 22 | `checksec --file /bin/ls /bin/cat` | `--file` accepts multiple paths |
| 23 | `checksec /bin/ls /bin/cat /bin/bash` | Positional multiple paths |
| 24 | `checksec /bin/ls \| cat -v \| head -3` | Piped output has no ANSI colour |
| 25 | `checksec /bin/ls > evidence/ls.txt; cat evidence/ls.txt` | Save a plain-text record |
| 26 | `checksec /bin/ls \| wc -l` | Count the fields emitted for a hardened binary |

**Research questions**
- Which fields appear for a typical Debian binary?
- Why does a shared library report `PIE enabled`?
- Does `--file` differ from positional invocation?
- Is colour suppressed automatically when stdout is not a TTY?

**Sources:** observed output on Debian 13; `pwnlib/elf/elf.py`.
**Output:** a baseline field matrix for system binaries.

## Scenario 3 — compile a mitigation matrix

| # | Command | Purpose |
|---|---------|---------|
| 27 | `gcc -o default vuln.c` | Distro default flags |
| 28 | `checksec default` | Baseline posture |
| 29 | `gcc -O2 -D_FORTIFY_SOURCE=2 -fstack-protector-strong -o fortified vuln.c` | Enable FORTIFY and canary |
| 30 | `checksec fortified` | Expect FORTIFY and canary |
| 31 | `gcc -O2 -U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=0 -o nofortify vuln.c` | Explicitly disable FORTIFY |
| 32 | `checksec nofortify` | Expect no FORTIFY line |
| 33 | `gcc -fno-stack-protector -o nocanary vuln.c` | Disable the canary |
| 34 | `checksec nocanary` | Expect `No canary found` |
| 35 | `gcc -z execstack -o execstack vuln.c` | Request an executable stack |
| 36 | `checksec execstack` | Look for `Stack: Executable` / `RWX` |
| 37 | `gcc -no-pie -o nopie vuln.c` | Fixed-address executable |
| 38 | `checksec nopie` | Expect `No PIE (0x400000)` |
| 39 | `gcc -Wl,-z,relro,-z,now -o fullrelro vuln.c` | Force Full RELRO |
| 40 | `checksec fullrelro` | Expect `Full RELRO` |
| 41 | `gcc -Wl,-z,norelro -o norelro vuln.c` | Disable RELRO |
| 42 | `checksec norelro` | Expect `No RELRO` |
| 43 | `gcc -static -o static-test vuln.c` | Static link |
| 44 | `checksec static-test` | Static binary field set |

**Research questions**
- Which single flag changes each field?
- Does `-z execstack` still produce a `GNU_STACK` segment on this toolchain?
- Does a static binary lose the `NX` determination?
- Which fields are independent of optimisation level?

**Sources:** `gcc --help=common`; observed `readelf`/`checksec` output.
**Output:** a before/after mitigation matrix.

## Scenario 4 — reading the ELF evidence behind the labels

| # | Command | Purpose |
|---|---------|---------|
| 45 | `readelf -lW fullrelro \| grep GNU_RELRO` | Confirm the RELRO segment |
| 46 | `readelf -dW fullrelro \| grep -E 'BIND_NOW\|FLAGS'` | Confirm eager binding |
| 47 | `readelf -lW execstack \| grep GNU_STACK` | Inspect the stack flags |
| 48 | `readelf -hW nopie \| grep Type` | Confirm `ET_EXEC` |
| 49 | `readelf -hW fullrelro \| grep Type` | Confirm `ET_DYN` |
| 50 | `file fullrelro` | Cross-check the file type |
| 51 | `readelf -lW static-test \| grep -c GNU_STACK` | Does a static binary have the segment? |
| 52 | `gcc -shared -fPIC -o libdemo.so vuln.c` | Build a shared object |
| 53 | `checksec libdemo.so` | Shared-object field set |
| 54 | `readelf -hW libdemo.so \| grep Type` | Confirm `ET_DYN` |
| 55 | `ldd static-test` | Show that a static binary has no loader dependencies |
| 56 | `objdump -p fullrelro \| grep -E 'RELRO\|BIND_NOW'` | Second view of the same facts |

**Research questions**
- Which ELF structure drives each label?
- Is `PIE enabled` on a `.so` a real mitigation claim?
- Does a missing `GNU_STACK` mean an executable stack?
- Which tool is authoritative when `checksec` and `readelf` disagree?

**Sources:** `readelf(1)`; `elf(5)`; observed output.
**Output:** a label-to-structure mapping with raw evidence.

## Scenario 5 — FORTIFY, symbols, debuginfo, sanitizers

| # | Command | Purpose |
|---|---------|---------|
| 57 | `nm -D fortified \| grep _chk` | Show checked libc symbols |
| 58 | `nm -D nofortify \| grep _chk` | Show their absence |
| 59 | `checksec fortified \| grep FORTIFY` | Confirm the derived line |
| 60 | `checksec nofortify \| grep FORTIFY; echo "grep=$?"` | Confirm the line is omitted |
| 61 | `objdump -d fortified \| grep -c stack_chk` | Count canary references |
| 62 | `gcc -g -o debuginfo vuln.c` | Build with DWARF |
| 63 | `checksec debuginfo` | Expect `Debuginfo: Yes` |
| 64 | `readelf -SW debuginfo \| grep debug_info` | Confirm the section |
| 65 | `strip -o stripped debuginfo` | Remove symbols and DWARF |
| 66 | `checksec stripped` | Expect no `Stripped` line and no `Debuginfo` |
| 67 | `readelf -SW stripped \| grep -c SYMTAB` | Confirm the symbol table is gone |
| 68 | `gcc -fsanitize=address -o asan vuln.c` | Link the ASAN runtime |
| 69 | `checksec asan` | Expect `ASAN: Enabled` |
| 70 | `ldd asan \| grep -i asan` | Confirm the runtime dependency |

**Research questions**
- Is FORTIFY detected from symbols, sections, or compile flags?
- Can one `_chk` symbol enable the label for an otherwise unchecked binary?
- Does stripping remove the `Debuginfo` line as well as `Stripped`?
- Does ASAN change any other field?

**Sources:** `pwnlib/elf/elf.py` (`fortify`, `stripped`, `debuginfo`);
observed `nm`/`readelf` output.
**Output:** a symbol-level explanation of the derived labels.

## Scenario 6 — CET, RPATH, RWX, and hardening extras

| # | Command | Purpose |
|---|---------|---------|
| 71 | `gcc -Wl,-z,ibt,-z,shstk -o cet vuln.c` | Emit x86 CET property bits |
| 72 | `checksec cet` | Expect `SHSTK` and `IBT` lines |
| 73 | `readelf -n cet \| grep -A3 'x86 feature'` | Inspect the property note |
| 74 | `gcc -fcf-protection=full -o fcf vuln.c` | Compiler CET flag only |
| 75 | `checksec fcf` | Does it set the same property bits? |
| 76 | `readelf -n fcf \| head -12` | Compare the notes |
| 77 | `gcc -Wl,-rpath,/opt/lib -o rpath vuln.c` | Add a RUNPATH |
| 78 | `checksec rpath` | Expect `RUNPATH: b'/opt/lib'` |
| 79 | `gcc -Wl,-rpath,/opt/lib,--disable-new-dtags -o rpath-old vuln.c` | Force legacy RPATH |
| 80 | `checksec rpath-old` | Expect `RPATH` instead of `RUNPATH` |
| 81 | `readelf -dW rpath \| grep -E 'RPATH\|RUNPATH'` | Confirm the dynamic tag |
| 82 | `gcc -Wl,-z,noexecstack -o noexec vuln.c` | Explicit non-executable stack |
| 83 | `checksec noexec` | Confirm no `Stack: Executable` line |
| 84 | `readelf -lW noexec \| grep GNU_STACK` | Confirm the flags |

**Research questions**
- Which linker flag produces SHSTK/IBT notes that pwntools recognises?
- Is `-fcf-protection=full` sufficient on this toolchain?
- How do RPATH and RUNPATH differ in the output?
- Does `-z noexecstack` change any other field?

**Sources:** `ld(1)`; `readelf(1)`; observed output.
**Output:** a CET/RPATH field demonstration with raw notes.

## Scenario 7 — error handling and exit codes

| # | Command | Purpose |
|---|---------|---------|
| 85 | `checksec /etc/hostname` | Non-ELF: error text, exit 0 |
| 86 | `checksec /etc/hostname >/dev/null 2>&1; echo "exit=$?"` | Confirm the zero exit |
| 87 | `checksec /tmp` | Directory: argparse error |
| 88 | `checksec /tmp >/dev/null 2>&1; echo "exit=$?"` | Confirm exit 2 |
| 89 | `checksec /tmp/definitely-not-here` | Missing path |
| 90 | `checksec /tmp/definitely-not-here >/dev/null 2>&1; echo "exit=$?"` | Confirm exit 2 |
| 91 | `checksec >/dev/null 2>&1; echo "exit=$?"` | No arguments exits 0 |
| 92 | `checksec --file >/dev/null 2>&1; echo "exit=$?"` | `--file` with no value exits 0 |
| 93 | `checksec --output=json /bin/ls` | Unsupported classic-checksec flag |
| 94 | `checksec --output=json /bin/ls >/dev/null 2>&1; echo "exit=$?"` | Confirm exit 2 |
| 95 | `checksec '/bin/l?'` | Shell glob expansion before invocation |
| 96 | `checksec -h \| head -3` | Short help path |

**Research questions**
- Which failures are invisible to `$?`?
- Why does a non-ELF exit 0?
- Which classic `checksec.sh` flags are rejected?
- Does a glob of mixed files stop at the first error?

**Sources:** `pwnlib/commandline/checksec.py`; argparse behaviour; observed output.
**Output:** an exit-code table and a scripting-safe wrapper.

## Scenario 8 — Python API, batch reporting, cleanup

| # | Command | Purpose |
|---|---------|---------|
| 97 | `/opt/pipx/venvs/pwntools/bin/python -c "import pwn; print(pwn.__version__)"` | Confirm the venv interpreter |
| 98 | `/opt/pipx/venvs/pwntools/bin/python -c "from pwn import ELF; e=ELF('/bin/ls'); print(e.relro, e.canary, e.nx, e.pie)"` | Read four fields as values |
| 99 | `/opt/pipx/venvs/pwntools/bin/python -c "from pwn import ELF; e=ELF('/bin/ls'); print(e.fortify, e.stripped, e.debuginfo, e.execstack, e.packed)"` | Read the boolean fields |
| 100 | `/opt/pipx/venvs/pwntools/bin/python -c "from pwn import ELF; e=ELF('/bin/ls'); print(e.shstk, e.ibt, e.rwx_segments, e.rpath, e.runpath)"` | Read CET/path fields |
| 101 | `/opt/pipx/venvs/pwntools/bin/python -c "from pwn import ELF; e=ELF('/bin/ls'); print(e.arch, e.bits, e.endian, e.path)"` | Read identity fields |
| 102 | `/opt/pipx/venvs/pwntools/bin/python -c "from pwn import ELF; print(ELF('/bin/ls').checksec(color=False))"` | Return the formatted block as a string |
| 103 | `/opt/pipx/venvs/pwntools/bin/python - <<'PY'` … `PY` | Loop over candidates and print TSV |
| 104 | `checksec default fortified nopie fullrelro > evidence/matrix.txt` | Save the build matrix |
| 105 | `diff -u <(checksec default) <(checksec fortified)` | Compare two postures |
| 106 | `grep -c 'Full RELRO' evidence/matrix.txt` | Count hardened builds |
| 107 | `rm -rf ~/checksec-demo` | Remove the demo tree |
| 108 | `test ! -e ~/checksec-demo; echo "removed=$?"` | Verify cleanup |
| 109 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > /tmp/checksec-end.txt; cat /tmp/checksec-end.txt` | Record the end time |

**Research questions**
- Which attributes map one-to-one to the printed fields?
- Does `checksec(color=False)` return the block or print it?
- Which fields are `None` rather than `False` when absent?
- Can the Python API replace text scraping in a report pipeline?

**Sources:** `pwnlib/elf/elf.py`; observed Python output.
**Output:** a programmatic field extractor and a cleaned workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `-h`, `--help` | 1, 7 |
| `--file` | 2, 7 |
| positional ELF arguments | 2 |
| multiple paths | 2 |
| glob expansion | 7 |
| `--output=json` (rejected) | 7 |
| Python API (`ELF.*`) | 8 |

Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, install layout, CLI surface | 12 |
| 2 | Baseline system binaries | 14 |
| 3 | Compile a mitigation matrix | 18 |
| 4 | Reading the ELF evidence behind the labels | 12 |
| 5 | FORTIFY, symbols, debuginfo, sanitizers | 14 |
| 6 | CET, RPATH, RWX, and hardening extras | 14 |
| 7 | Error handling and exit codes | 12 |
| 8 | Python API, batch reporting, cleanup | 13 |
| **Total** | | **109** |

## Legal and safety

`checksec` only reads files, but the batch recipes walk system directories and
can reveal which setuid or network-facing binaries lack mitigations. Treat the
resulting inventory as sensitive: it is a target list. Do not run the Python
API against untrusted files without the same parsing caution you would apply
to any ELF parser, and delete the demo tree when finished.
