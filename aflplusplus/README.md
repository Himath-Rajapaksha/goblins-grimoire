# AFL++ — coverage-guided fuzzing

- **Version:** afl-fuzz++ 4.21c
- **Binaries:** `/usr/bin/afl-*` (afl-fuzz, afl-cc, afl-clang-fast, afl-cmin,
  afl-tmin, afl-showmap, afl-analyze, afl-whatsup, ...)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Docs:** https://github.com/AFLplusplus/AFLplusplus/tree/stable/docs
- **Source:** https://github.com/AFLplusplus/AFLplusplus

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

AFL++ is a fork of American Fuzzy Lop with modern instrumentation (LLVM LTO,
CmpLog, persistent mode), multiple mutation schedules, QEMU/FRIDA/Unicorn modes
for binary-only targets, and parallel fuzzing. It finds crashes and hangs by
maximizing code coverage from a seed corpus.

## Use cases

| Situation | Mode |
|-----------|------|
| Source available | `afl-clang-fast` / `afl-cc` instrumentation |
| Binary only | `-Q` (QEMU) or `-O` (FRIDA) |
| Network service | harness or `afl-network-server` |
| Library/API | persistent-mode harness |
| Structured input | dictionary + custom mutator |
| Multi-core | `-M`/`-S` parallel instances |
| Minimize corpus | `afl-cmin`, `afl-tmin` |

## Workflow

```
1. Build target with instrumentation
2. Prepare seed corpus
3. (Optional) create dictionary
4. Dry-run: afl-fuzz -i corpus -o out -- ./target @@
5. Run / parallelize
6. Triage crashes with gdb/ASAN
7. Minimize inputs with afl-tmin
```

## Building instrumented targets

```bash
# LLVM fast instrumentation (recommended)
afl-clang-fast -O2 -g target.c -o target

# LTO mode (best coverage/performance, slower build)
afl-clang-lto -O2 -g target.c -o target

# GCC plugin
afl-gcc-fast -O2 -g target.c -o target

# C++ equivalents: afl-clang-fast++, afl-g++-fast, afl-clang-lto++

# Compile-time check
afl-cc --version
```

For projects using `./configure`:

```bash
CC=afl-clang-fast CXX=afl-clang-fast++ ./configure --disable-shared
make -j$(nproc)
```

Persistent mode (much faster, requires modifying the harness):

```c
#include "afl-fuzz.h"  // or use __AFL_LOOP macros

int main(void) {
    __AFL_INIT();
    while (__AFL_LOOP(1000)) {
        // read input, run target logic
    }
    return 0;
}
```

Deferred forkserver:

```c
__AFL_INIT();
```

## afl-fuzz reference

### Required
| Flag | Meaning |
|------|---------|
| `-i <dir>` | Seed corpus directory (`-` resumes) |
| `-o <dir>` | Output directory |
| `-- <cmd>` | Target command; `@@` is replaced by input file |

### Execution control
| Flag | Meaning |
|------|---------|
| `-P <strategy>` | `explore` / `exploit` / seconds |
| `-p <schedule>` | `explore, fast, exploit, seek, rare, mmopt, coe, lin, quad` |
| `-f <file>` | Input file path (default stdin or `@@`) |
| `-t <ms>` | Per-run timeout (auto-scaled) |
| `-m <MB>` | Memory limit (`0` = none) |
| `-O` | FRIDA mode (binary-only) |
| `-Q` | QEMU mode (binary-only) |
| `-U` | Unicorn mode |
| `-W` | QEMU + Wine |
| `-X` / `-Y` | Nyx VM modes |

### Mutators
| Flag | Meaning |
|------|---------|
| `-a <text\|binary>` | Input format hint |
| `-g <min>` / `-G <max>` | Min/max generated input length |
| `-L <min>` | MOpt mode |
| `-c <binary>` | CmpLog binary (`-c 0` if target is CmpLog) |
| `-l <opts>` | CmpLog options (`2ATR`) |

### Behavior
| Flag | Meaning |
|------|---------|
| `-Z` | Sequential queue |
| `-N` | Don't unlink input file |
| `-n` | Non-instrumented mode (dumb fuzzing) |
| `-x <dict>` | Dictionary (up to 4) |
| `-s <seed>` | Fixed RNG seed |
| `-V <sec>` | Fuzz for N seconds |
| `-E <execs>` | Stop after ~N executions |
| `-M <id>` / `-S <id>` | Master / secondary parallel instance |
| `-F <dir>` | Sync from foreign fuzzer |
| `-z` | Skip deterministic stage |
| `-T <text>` | Custom banner |
| `-I <cmd>` | Run command on new crash |
| `-C` | Crash exploration mode |
| `-b <cpu>` | Bind to CPU core |
| `-e <ext>` | Input file extension |

`afl-fuzz -hh` shows environment variables.

## Stages / environment

| Variable | Effect |
|----------|--------|
| `AFL_SKIP_CPUFREQ=1` | Skip CPU governor check |
| `AFL_NO_UI=1` | Plain log output |
| `AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1` | Bypass core_pattern check |
| `AFL_FAST_CAL=1` | Faster calibration |
| `AFL_HARDEN=1` | Compile-time hardening |
| `AFL_USE_ASAN=1` | Build with AddressSanitizer |
| `AFL_USE_UBSAN=1` | Build with UBSan |
| `AFL_LLVM_CMPLOG=1` | Build CmpLog binary |
| `AFL_PERSISTENT_RECORD` | Persistent record (if supported) |

Fix host settings once:

```bash
sudo afl-system-config
```

## Corpus and dictionaries

```bash
# Seed corpus
mkdir -p corpus
cp samples/* corpus/

# Minimize corpus by coverage
afl-cmin -i corpus -o corpus-min -- ./target @@

# Minimize a single crashing input
afl-tmin -i crash -o crash-min -- ./target @@

# Dictionary example (magic values, keywords)
cat > dict.txt <<'EOF'
"GET"
"POST"
"HTTP/1.1"
"\x00\x00\x00\x00"
EOF
```

## Recipes

```bash
# Basic fuzz
afl-fuzz -i corpus -o findings -- ./target @@

# With dictionary and timeout
afl-fuzz -i corpus -o findings -x dict.txt -t 500 -- ./target @@

# Parallel: 1 master + 3 secondaries
afl-fuzz -i corpus -o findings -M main -- ./target @@
afl-fuzz -i corpus -o findings -S s1 -- ./target @@
afl-fuzz -i corpus -o findings -S s2 -- ./target @@
afl-fuzz -i corpus -o findings -S s3 -- ./target @@

# Binary-only via QEMU
afl-fuzz -Q -i corpus -o findings -- ./target @@

# Binary-only via FRIDA
afl-fuzz -O -i corpus -o findings -- ./target @@

# CmpLog-assisted
afl-fuzz -i corpus -o findings -c ./target.cmplog -- ./target @@

# Fixed duration
afl-fuzz -V 3600 -i corpus -o findings -- ./target @@

# Non-interactive CI run
AFL_NO_UI=1 AFL_SKIP_CPUFREQ=1 afl-fuzz -V 300 -i corpus -o findings -- ./target @@

# Monitor parallel run
afl-whatsup findings/
```

## Triage

```bash
# List crashes
ls findings/default/crashes/

# Reproduce
./target findings/default/crashes/id:000000,...

# Debug
gdb --args ./target findings/default/crashes/id:000000,...

# ASAN build for better reports
AFL_USE_ASAN=1 afl-clang-fast -g -O1 target.c -o target-asan
```

## Gotchas

- Instrumentation is everything: an uninstrumented target gives near-useless
  coverage (use `-Q`/`-O` only when source is unavailable).
- `core_pattern` must pipe crashes to a file, not `apport`/`systemd-coredump`;
  `afl-system-config` fixes common issues.
- CPU frequency scaling and ASLR checks can stop the run; set the env vars above.
- Huge seeds slow calibration; minimize with `afl-cmin`.
- `-m` default in older AFL was 50 MB; here `0` means unlimited — read help.
- Persistent mode requires a correct harness; a bad loop causes false crashes.
- QEMU mode is much slower than compiler instrumentation.
- Fuzzing is CPU-bound; use parallel instances and check `afl-whatsup`.

## References

- Docs: https://github.com/AFLplusplus/AFLplusplus/tree/stable/docs
- Fuzzing in depth: https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/fuzzing_in_depth.md
- FAQ: https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/FAQ.md
- custom mutators: https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/custom_mutators.md
