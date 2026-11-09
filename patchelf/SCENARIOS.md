# patchelf — command scenarios for deep research

Installed build: patchelf 0.18.0 (Debian `patchelf` `0.18.0-1.4`). Every flag
below appears in [`help.txt`](help.txt); field semantics come from
`man patchelf` and observed ELF output.

> **Scope:** the recipes build and patch throwaway binaries in a demo tree.
> Patching changes file layout, invalidates hashes and signatures, and can
> produce a binary the kernel refuses to run. Keep the original and test the
> result. Do not patch binaries you are not authorised to modify.

> **No-validation warning:** `--set-interpreter`, `--add-needed`, and
> `--page-size` accept nonsense values and exit 0. The recipes therefore
> re-read the field and execute the patched binary.

## How to invoke

```text
@deep-research For an authorised ELF-patching task with patchelf 0.18.0,
explain how `--set-interpreter`, `--set-rpath`/`--force-rpath`,
`--add-needed`/`--replace-needed`, `--set-soname`, `--set-os-abi`,
`--set-execstack`, `--no-default-lib`, and `--rename-dynamic-symbols` change
the ELF headers, dynamic section, and symbol tables. Show which flags write
`DT_RPATH` versus `DT_RUNPATH`, how `DF_1_NODEFLIB` differs from removing a
`DT_NEEDED` entry, and which invalid inputs still exit 0. Cite the installed
help text, `man patchelf`, `readelf` output, and the runtime behaviour of the
patched binaries. Separate confirmed behaviour from inference and state
confidence.
```

Create the demo tree and the test programs:

```bash
mkdir -p ~/patchelf-demo/evidence
cd ~/patchelf-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
cat > app.c <<'EOF'
#include <stdio.h>
int main(void) { puts("hi"); return 0; }
EOF
gcc -o app app.c
gcc -shared -fPIC -o libfoo.so app.c
sha256sum app libfoo.so > evidence/before.sha256
```

## Scenario 1 — version, help, install layout

| # | Command | Purpose |
|---|---------|---------|
| 1 | `patchelf --version` | Print the version |
| 2 | `patchelf` | No arguments: syntax block, exit 0 |
| 3 | `patchelf >/dev/null 2>&1; echo "exit=$?"` | Confirm the no-argument exit |
| 4 | `patchelf --help 2>&1 \| head -3` | `--help` is not a real flag |
| 5 | `command -v patchelf` | Resolve the binary |
| 6 | `file "$(command -v patchelf)"` | Confirm a native ELF |
| 7 | `dpkg -s patchelf \| grep -E 'Version\|Depends'` | Package provenance |
| 8 | `man patchelf \| col -b \| head -40` | Read the manual |
| 9 | `man patchelf \| col -b \| grep -n '^ *--no-default-lib'` | Locate a specific flag |
| 10 | `ldd "$(command -v patchelf)"` | Confirm the tool's own loader deps |
| 11 | `sha256sum "$(command -v patchelf)"` | Record the tool hash |
| 12 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt` | Timestamp the run |

**Research questions**
- Which flags are documented in help but behave differently in practice?
- Is `--help` a real option or an unrecognised argument?
- Which libc and loader does the tool itself use?
- Does the Debian build differ from upstream?

**Sources:** `patchelf` output; `man patchelf`; `dpkg -s`.
**Output:** a version record and the exact flag surface.

## Scenario 2 — baseline inspection of an executable and a library

| # | Command | Purpose |
|---|---------|---------|
| 13 | `patchelf --print-interpreter app` | Loader path |
| 14 | `patchelf --print-rpath app \| od -c \| head -2` | Empty RPATH is a newline |
| 15 | `patchelf --print-needed app` | Shared-library dependencies |
| 16 | `patchelf --print-execstack app` | Executable-stack request |
| 17 | `patchelf --print-os-abi app` | ELF OS ABI byte |
| 18 | `patchelf --print-soname app; echo "exit=$?"` | Executables have no SONAME |
| 19 | `patchelf --print-interpreter libfoo.so; echo "exit=$?"` | Libraries have no interpreter |
| 20 | `patchelf --print-soname libfoo.so; echo "exit=$?"` | Library built without a SONAME |
| 21 | `patchelf --print-needed libfoo.so` | Library dependencies |
| 22 | `readelf -hW app \| grep -E 'Type\|OS/ABI'` | Header cross-check |
| 23 | `readelf -lW app \| grep -E 'INTERP\|GNU_STACK'` | Segment cross-check |
| 24 | `readelf -dW app \| grep -E 'NEEDED\|RPATH\|RUNPATH'` | Dynamic-section cross-check |
| 25 | `readelf -sW app \| grep -w puts` | Symbol version baseline |
| 26 | `objdump -p app \| grep -E 'NEEDED\|RPATH\|RUNPATH'` | Second view |

**Research questions**
- Which fields exist on an executable but not on a shared library?
- Does `--print-rpath` distinguish "no path" from "empty path"?
- Where does `--print-soname` read from?
- Which tool is authoritative when the views disagree?

**Sources:** `readelf(1)`; `objdump(1)`; observed output.
**Output:** a baseline field matrix for the two ELF types.

## Scenario 3 — interpreter patching

| # | Command | Purpose |
|---|---------|---------|
| 27 | `cp app interp` | Work on a copy |
| 28 | `patchelf --set-interpreter /lib64/ld-linux-x86-64.so.2 interp` | Set the same loader explicitly |
| 29 | `patchelf --print-interpreter interp` | Re-read the field |
| 30 | `./interp` | Confirm it still runs |
| 31 | `readelf -lW interp \| grep INTERP` | Confirm the segment |
| 32 | `readelf -p .interp interp` | Dump the string table entry |
| 33 | `cp app badinterp` | Second copy |
| 34 | `patchelf --set-interpreter /nonexistent/ld.so badinterp` | Set a bogus loader |
| 35 | `patchelf --set-interpreter /nonexistent/ld.so badinterp >/dev/null 2>&1; echo "exit=$?"` | Confirm exit 0 |
| 36 | `./badinterp 2>&1 \| head -2` | Confirm the runtime failure |
| 37 | `patchelf --print-interpreter badinterp` | The field was written anyway |
| 38 | `readelf -lW badinterp \| grep INTERP` | Segment points at the bogus path |
| 39 | `cp /bin/true staticish && patchelf --print-interpreter staticish` | A system binary's loader |
| 40 | `ldd interp` | Confirm the loader resolves |

**Research questions**
- Does patchelf validate the interpreter path?
- Does the kernel reject a missing interpreter before `main`?
- Is the `.interp` string length constrained?
- Can the same loader path be restored?

**Sources:** `man patchelf`; `readelf`; observed runtime error.
**Output:** a working and a deliberately broken interpreter patch.

## Scenario 4 — RPATH and RUNPATH manipulation

| # | Command | Purpose |
|---|---------|---------|
| 41 | `cp app rp-run` | RUNPATH test copy |
| 42 | `patchelf --set-rpath /opt/lib rp-run` | Default writes RUNPATH |
| 43 | `patchelf --print-rpath rp-run` | Read it back |
| 44 | `readelf -dW rp-run \| grep RUNPATH` | Confirm the tag |
| 45 | `cp app rp-old` | RPATH test copy |
| 46 | `patchelf --force-rpath --set-rpath /opt/lib rp-old` | Force the legacy tag |
| 47 | `readelf -dW rp-old \| grep RPATH` | Confirm `DT_RPATH` |
| 48 | `readelf -dW rp-old \| grep -c RUNPATH` | Confirm no RUNPATH |
| 49 | `cp app rp-add` | Add-path copy |
| 50 | `patchelf --set-rpath /opt/lib rp-add && patchelf --add-rpath /usr/local/lib rp-add` | Append an entry |
| 51 | `patchelf --print-rpath rp-add` | Confirm the order |
| 52 | `cp app rp-origin` | `$ORIGIN` copy |
| 53 | `patchelf --set-rpath '$ORIGIN/lib' rp-origin` | Relative search path |
| 54 | `patchelf --print-rpath rp-origin` | Confirm literal `$ORIGIN` |
| 55 | `cp app rp-rm` | Removal copy |
| 56 | `patchelf --remove-rpath rp-rm` | Remove the path |
| 57 | `patchelf --print-rpath rp-rm \| od -c \| head -2` | Confirm empty |
| 58 | `cp app rp-shrink` | Shrink copy |
| 59 | `patchelf --set-rpath '/lib/x86_64-linux-gnu:/tmp/nowhere' rp-shrink` | One real, one fake entry |
| 60 | `patchelf --shrink-rpath rp-shrink` | Drop entries with no needed library |
| 61 | `patchelf --print-rpath rp-shrink` | Confirm the surviving entry |
| 62 | `cp app rp-prefix` | Prefix-restricted copy |
| 63 | `patchelf --set-rpath '/opt/lib:/usr/lib' rp-prefix` | Two entries |
| 64 | `patchelf --shrink-rpath --allowed-rpath-prefixes /opt rp-prefix` | Keep only `/opt` |
| 65 | `patchelf --print-rpath rp-prefix` | Inspect the result |
| 66 | `LD_DEBUG=libs ./rp-run 2>&1 \| grep -i 'search path' \| head -3` | Loader search diagnostics |

**Research questions**
- Which flag writes RUNPATH, and which writes RPATH?
- How does RPATH precedence differ from RUNPATH at load time?
- Does `--add-rpath` deduplicate entries?
- Can `--shrink-rpath` remove every entry?
- Is `$ORIGIN` expanded by patchelf or by the loader?

**Sources:** `ld.so(8)`; `readelf -d`; `LD_DEBUG` output.
**Output:** a RUNPATH/RPATH matrix with loader evidence.

## Scenario 5 — NEEDED, SONAME, and symbol tables

| # | Command | Purpose |
|---|---------|---------|
| 67 | `cp app need-add` | NEEDED test copy |
| 68 | `patchelf --add-needed libm.so.6 need-add` | Add a real library |
| 69 | `patchelf --print-needed need-add` | Confirm the new entry |
| 70 | `cp app need-bogus` | Bogus NEEDED copy |
| 71 | `patchelf --add-needed libnope.so need-bogus` | Add a non-existent library |
| 72 | `patchelf --add-needed libnope.so need-bogus >/dev/null 2>&1; echo "exit=$?"` | Confirm exit 0 |
| 73 | `./need-bogus 2>&1 \| head -2` | Confirm the loader failure |
| 74 | `cp app need-replace` | Replacement copy |
| 75 | `patchelf --replace-needed libc.so.6 libc.so.6 need-replace` | Replace with the same name |
| 76 | `patchelf --print-needed need-replace` | Confirm the list |
| 77 | `cp app need-absent` | Absent-library copy |
| 78 | `patchelf --replace-needed libnope.so libother.so need-absent` | Replace a missing entry |
| 79 | `patchelf --replace-needed libnope.so libother.so need-absent >/dev/null 2>&1; echo "exit=$?"` | Confirm exit 0 |
| 80 | `patchelf --print-needed need-absent` | Confirm the list is unchanged |
| 81 | `cp app need-remove` | Removal copy |
| 82 | `patchelf --remove-needed libnope.so need-remove` | Remove a missing entry |
| 83 | `patchelf --remove-needed libnope.so need-remove >/dev/null 2>&1; echo "exit=$?"` | Confirm exit 0 |
| 84 | `cp app need-nodefault` | `DF_1_NODEFLIB` copy |
| 85 | `patchelf --no-default-lib need-nodefault` | Set the flag |
| 86 | `readelf -dW need-nodefault \| grep FLAGS_1` | Expect `NODEFLIB` |
| 87 | `patchelf --print-needed need-nodefault` | libc is still listed |
| 88 | `cp libfoo.so soname` | SONAME test copy |
| 89 | `patchelf --set-soname libfoo.so.1 soname` | Set the SONAME |
| 90 | `patchelf --print-soname soname` | Read it back |
| 91 | `cp app soname-exe && patchelf --set-soname foo soname-exe` | Set a SONAME on an executable |
| 92 | `patchelf --print-soname soname-exe` | Confirm it is accepted |
| 93 | `cp app symver` | Symbol-version copy |
| 94 | `patchelf --clear-symbol-version puts symver` | Clear the dynsym version |
| 95 | `readelf -sW symver \| grep -w puts` | Compare dynsym and symtab |
| 96 | `printf 'puts puts_alias\n' > rename.map` | Space-separated map |
| 97 | `cp app rename && patchelf --rename-dynamic-symbols rename.map rename` | Rename a dynamic symbol |
| 98 | `readelf -sW rename \| grep -E 'puts\|puts_alias'` | Confirm the rename |
| 99 | `printf 'puts:puts_alias\n' > bad.map && cp app rename-bad` | Colon-separated map |
| 100 | `patchelf --rename-dynamic-symbols bad.map rename-bad >/dev/null 2>&1; echo "exit=$?"` | Confirm exit 1 |
| 101 | `cp app rename-nomap && patchelf --rename-dynamic-symbols /nonexistent rename-nomap >/dev/null 2>&1; echo "exit=$?"` | Missing map exits 0 |

**Research questions**
- Which NEEDED operations validate that the library exists?
- Does `--no-default-lib` remove `libc.so.6` or set a flag?
- Can an executable carry a SONAME?
- Why does `--clear-symbol-version` leave one `puts@GLIBC_2.2.5` behind?
- Is the rename map format space- or colon-separated?

**Sources:** `readelf -d`; `readelf -s`; observed loader errors.
**Output:** a NEEDED/SONAME/symbol table with runtime consequences.

## Scenario 6 — OS ABI, execstack, debug tag, and output copies

| # | Command | Purpose |
|---|---------|---------|
| 102 | `for abi in sysv linux freebsd solaris; do cp app abi-$abi; patchelf --set-os-abi $abi abi-$abi; printf '%-8s ' $abi; patchelf --print-os-abi abi-$abi; done` | Accepted ABI names |
| 103 | `cp app abi-bad && patchelf --set-os-abi hpux abi-bad >/dev/null 2>&1; echo "exit=$?"` | `hpux` is rejected |
| 104 | `patchelf --print-os-abi abi-bad` | The old value survives |
| 105 | `cp app abi-none && patchelf --set-os-abi none abi-none >/dev/null 2>&1; echo "exit=$?"` | `none` is rejected |
| 106 | `cp app stack-on && patchelf --print-execstack stack-on` | Baseline is non-executable |
| 107 | `patchelf --set-execstack stack-on && patchelf --print-execstack stack-on` | Request an executable stack |
| 108 | `readelf -lW stack-on \| grep GNU_STACK` | Expect `RWE` |
| 109 | `patchelf --clear-execstack stack-on && patchelf --print-execstack stack-on` | Clear the request |
| 110 | `readelf -lW stack-on \| grep GNU_STACK` | Expect `RW` |
| 111 | `cp app debugtag && readelf -dW debugtag \| grep -c DEBUG` | Executables usually have `DT_DEBUG` |
| 112 | `patchelf --add-debug-tag debugtag && readelf -dW debugtag \| grep -c DEBUG` | Confirm the no-op |
| 113 | `cp libfoo.so debugtag-so && readelf -dW debugtag-so \| grep -c DEBUG` | Libraries usually do not |
| 114 | `patchelf --add-debug-tag debugtag-so && readelf -dW debugtag-so \| grep -c DEBUG` | Confirm the added tag |
| 115 | `rm -f copy-noaction && patchelf --output copy-noaction app && ls -l copy-noaction` | `--output` alone still copies |
| 116 | `rm -f copy-action && patchelf --output copy-action --set-rpath /opt/lib app` | Copy plus patch |
| 117 | `patchelf --print-rpath copy-action; patchelf --print-rpath app` | Only the copy changed |
| 118 | `cmp -s app copy-action; echo "cmp=$?"` | The copy differs from the input |
| 119 | `cp app debug-run && patchelf --debug --set-rpath /opt/lib debug-run 2>&1 \| head -12` | Rewrite diagnostics |
| 120 | `cp app no-sort && patchelf --no-sort --set-rpath /opt/lib no-sort` | Skip header sorting |
| 121 | `cp app page-size && patchelf --page-size 123 --set-rpath /opt/lib page-size` | Nonsense page size accepted |

**Research questions**
- Which OS ABI spellings are accepted, and what does `hpux` do?
- Does `--set-execstack` change the `GNU_STACK` flags or only a request bit?
- When does `--add-debug-tag` actually add `DT_DEBUG`?
- Does `--output` write a copy when no action is requested?
- Does `--debug` change the output or only the log?

**Sources:** `readelf -l`; `readelf -d`; observed output.
**Output:** an ABI/stack/debug-tag matrix and a copy-mode comparison.

## Scenario 7 — failure modes and exit codes

| # | Command | Purpose |
|---|---------|---------|
| 122 | `patchelf --print-rpath /nonexistent >/dev/null 2>&1; echo "exit=$?"` | Missing path exits 1 |
| 123 | `patchelf --print-rpath /tmp >/dev/null 2>&1; echo "exit=$?"` | Directory exits 1 |
| 124 | `patchelf --print-rpath app.c >/dev/null 2>&1; echo "exit=$?"` | Non-ELF exits 1 |
| 125 | `patchelf --print-rpath app.c 2>&1 \| head -2` | Read the error text |
| 126 | `cp app badabi && patchelf --set-os-abi nonsense badabi >/dev/null 2>&1; echo "exit=$?"` | Invalid ABI exits 1 |
| 127 | `cp app badpage && patchelf --page-size 123 --set-rpath /opt/lib badpage >/dev/null 2>&1; echo "exit=$?"` | Invalid page size exits 0 |
| 128 | `cp app badneed && patchelf --add-needed /nonexistent/lib.so badneed >/dev/null 2>&1; echo "exit=$?"` | Bad NEEDED exits 0 |
| 129 | `patchelf --print-needed badneed` | The entry was added |
| 130 | `cp app badsoname && patchelf --print-soname badsoname >/dev/null 2>&1; echo "exit=$?"` | Missing SONAME exits 0 |
| 131 | `patchelf --print-soname badsoname 2>&1 \| head -2` | Confirm no error text |
| 132 | `cp app badrename && printf 'onlyone\n' > one.map && patchelf --rename-dynamic-symbols one.map badrename >/dev/null 2>&1; echo "exit=$?"` | One-name map exits 1 |
| 133 | `patchelf --rename-dynamic-symbols one.map badrename 2>&1 \| head -2` | Read the map error |
| 134 | `cp app noop && patchelf --output noop-copy noop >/dev/null 2>&1; echo "exit=$?"` | `--output` without action exits 0 |
| 135 | `patchelf --print-rpath noop-copy \| od -c \| head -2` | The copy has no path |

**Research questions**
- Which semantic errors are invisible to `$?`?
- Which operations fail before touching the file?
- Does a failed patch leave a partial file?
- What is a safe wrapper for a patch pipeline?

**Sources:** observed exit codes; `man patchelf`.
**Output:** an exit-code table with runtime consequences.

## Scenario 8 — runtime verification, hardening cross-check, cleanup

| # | Command | Purpose |
|---|---------|---------|
| 136 | `patchelf --set-rpath '$ORIGIN' app && ./app` | Confirm the patched binary still runs |
| 137 | `ldd app` | Confirm dependency resolution |
| 138 | `checksec app` | Hardening after patching |
| 139 | `checksec rp-run \| grep -E 'RPATH\|RUNPATH'` | The patch is visible to checksec |
| 140 | `readelf -dW app \| grep -E 'RPATH\|RUNPATH'` | Raw dynamic tag |
| 141 | `LD_LIBRARY_PATH=/tmp ./app 2>&1 \| head -2` | Environment override still works |
| 142 | `patchelf --print-interpreter app` | Final interpreter |
| 143 | `patchelf --print-needed app` | Final dependency list |
| 144 | `patchelf --print-execstack app` | Final stack request |
| 145 | `sha256sum app \| tee evidence/after.sha256` | Record the patched hash |
| 146 | `diff -u evidence/before.sha256 evidence/after.sha256` | Confirm the hash changed |
| 147 | `cp app final && patchelf --output final.patched --set-rpath '$ORIGIN/lib' final` | Keep a clean original |
| 148 | `cmp -s final final.patched; echo "cmp=$?"` | Confirm the copy differs |
| 149 | `rm -rf ~/patchelf-demo` | Remove the demo tree |
| 150 | `test ! -e ~/patchelf-demo; echo "removed=$?"` | Verify cleanup |

**Research questions**
- Does a patched binary still satisfy its original dependencies?
- Does `checksec` reflect the RPATH/RUNPATH change?
- Does `LD_LIBRARY_PATH` override RUNPATH, RPATH, or neither?
- What evidence proves the original was preserved?

**Sources:** `checksec`; `ldd`; observed runtime output.
**Output:** a runtime verification record and a cleaned workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `--version`, `--help` (rejected), no arguments | 1 |
| `--print-interpreter`, `--print-rpath`, `--print-needed`, `--print-execstack`, `--print-os-abi`, `--print-soname` | 2 |
| `--set-interpreter` | 3 |
| `--set-rpath`, `--force-rpath`, `--add-rpath`, `--remove-rpath`, `--shrink-rpath`, `--allowed-rpath-prefixes` | 4 |
| `--add-needed`, `--replace-needed`, `--remove-needed`, `--no-default-lib`, `--set-soname`, `--clear-symbol-version`, `--rename-dynamic-symbols` | 5 |
| `--set-os-abi`, `--set-execstack`, `--clear-execstack`, `--add-debug-tag`, `--output`, `--debug`, `--no-sort`, `--page-size` | 6 |
| error paths for all of the above | 7 |

Full text: [`help.txt`](help.txt); authoritative description in
`man patchelf`.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, help, install layout | 12 |
| 2 | Baseline inspection of an executable and a library | 14 |
| 3 | Interpreter patching | 14 |
| 4 | RPATH and RUNPATH manipulation | 26 |
| 5 | NEEDED, SONAME, and symbol tables | 35 |
| 6 | OS ABI, execstack, debug tag, and output copies | 20 |
| 7 | Failure modes and exit codes | 14 |
| 8 | Runtime verification, hardening cross-check, cleanup | 15 |
| **Total** | | **150** |

## Legal and safety

Patching an ELF changes its bytes, invalidates hashes and code signatures, and
can produce a binary that fails before `main`. Keep an untouched original,
record the before/after hashes, and run the result in a controlled environment.
A changed interpreter or RPATH is a code-loading change: a writable search
path is a privilege-escalation vector on setuid binaries, and patching a
setuid binary can remove or weaken a distribution's hardening. Only patch
files you own or are authorised to modify, and delete the demo tree when
finished.
