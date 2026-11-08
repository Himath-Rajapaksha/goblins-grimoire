# AFL++ — command scenarios for deep research

Every AFL++ 4.21c command form, grouped by fuzzing workflow. `afl-fuzz` flags
come from [`help.txt`](help.txt); the companion tools were captured from their
own `-h` output during the scenario pass.

## How to invoke

```text
@deep-research For AFL++ 4.21c, what is the fastest reliable path to fuzz a
network daemon that reads length-prefixed protobuf, when source is available
but the parser forks per request? Compare a persistent harness, a forkserver
harness, and afl-network-server; cite AFL++ docs and real write-ups; return
exact build and run commands.
```

---

## Scenario 1 — build an instrumented target

| # | Command | How to use |
|---|---------|------------|
| 1 | `afl-clang-fast -O2 -g target.c -o target` | Default LLVM PCGUARD build |
| 2 | `afl-clang-fast++ -O2 -g target.cc -o target` | C++ equivalent |
| 3 | `afl-clang-lto -O2 -g target.c -o target` | Best coverage/speed, slower link |
| 4 | `afl-clang-lto++ -O2 -g target.cc -o target` | C++ LTO build |
| 5 | `afl-gcc-fast -O2 -g target.c -o target` | GCC plugin build |
| 6 | `afl-g++-fast -O2 -g target.cc -o target` | C++ GCC plugin build |
| 7 | `afl-cc -O2 -g target.c -o target` | Auto-select best backend |
| 8 | `afl-c++ -O2 -g target.cc -o target` | C++ auto-select |
| 9 | `afl-cc --afl-llvm -O2 -g target.c -o target` | Force LLVM mode |
| 10 | `afl-cc --afl-lto -O2 -g target.c -o target` | Force LTO mode |
| 11 | `afl-cc --afl-gcc -O2 -g target.c -o target` | Force GCC plugin mode |
| 12 | `CC=afl-cc CXX=afl-c++ ./configure --disable-shared` | Autoconf project |
| 13 | `cmake -DCMAKE_C_COMPILER=afl-cc -DCMAKE_CXX_COMPILER=afl-c++ .` | CMake project |
| 14 | `AFL_USE_ASAN=1 afl-clang-fast -O1 -g target.c -o target-asan` | ASAN build |
| 15 | `AFL_USE_UBSAN=1 afl-clang-fast -O1 -g target.c -o target-ubsan` | UBSan build |
| 16 | `AFL_USE_MSAN=1 afl-clang-fast -O1 -g target.c -o target-msan` | MSan build |
| 17 | `AFL_USE_TSAN=1 afl-clang-fast -O1 -g target.c -o target-tsan` | TSan build |
| 18 | `AFL_USE_LSAN=1 afl-clang-fast -O1 -g target.c -o target-lsan` | Leak checker |
| 19 | `AFL_USE_CFISAN=1 afl-clang-fast -O1 -g target.c -o target-cfi` | CFI sanitizer |
| 20 | `AFL_HARDEN=1 afl-clang-fast -O2 -g target.c -o target-hardened` | Extra hardening |
| 21 | `AFL_LLVM_CMPLOG=1 afl-clang-fast -O2 -g target.c -o target.cmplog` | CmpLog build |
| 22 | `AFL_LLVM_DICT2FILE=dict.txt afl-clang-fast -O2 target.c -o target` | Auto-extract dictionary |
| 23 | `AFL_INST_RATIO=50 afl-clang-fast -O2 target.c -o target` | Instrument half the branches |
| 24 | `AFL_LLVM_INSTRUMENT=CTX afl-clang-fast -O2 target.c -o target` | Context-sensitive coverage |
| 25 | `AFL_LLVM_INSTRUMENT=NGRAM-4 afl-clang-fast -O2 target.c -o target` | N-gram coverage |
| 26 | `afl-cc --version` | Confirm backend and LLVM version |

**Research questions**
- For a fork-per-request daemon, does a persistent harness change the bug class
  you can find?
- LTO vs PCGUARD: when is the slower build worth it?
- Which sanitizer combination is stable enough for a long campaign?

**Sources**: AFL++ `docs/fuzzing_in_depth.md`, `instrumentation/README.*`,
LLVM sanitizer docs, project write-ups.

**Output**: a build recipe plus the harness type, recorded before fuzzing.

---

## Scenario 2 — prepare and minimize the corpus

| # | Command | How to use |
|---|---------|------------|
| 27 | `mkdir -p corpus && cp samples/* corpus/` | Seed directory |
| 28 | `afl-cmin -i corpus -o corpus-min -- ./target @@` | Coverage-minimize a corpus |
| 29 | `afl-cmin -T all -i corpus -o corpus-min -- ./target @@` | Parallel minimization |
| 30 | `afl-cmin -e -i corpus -o corpus-min -- ./target @@` | Edge coverage only |
| 31 | `afl-cmin -A -i corpus -o corpus-min -- ./target @@` | Allow crashes/timeouts |
| 32 | `afl-cmin -C -i corpus -o corpus-crash -- ./target @@` | Keep only crashing inputs |
| 33 | `afl-cmin -m 2048 -t 2000 -i corpus -o corpus-min -- ./target @@` | Memory/time limits |
| 34 | `afl-cmin -O -i corpus -o corpus-min -- ./target @@` | FRIDA binary-only |
| 35 | `afl-cmin -Q -i corpus -o corpus-min -- ./target @@` | QEMU binary-only |
| 36 | `afl-tmin -i crash -o crash-min -- ./target @@` | Minimize one input |
| 37 | `afl-tmin -e -i crash -o crash-min -- ./target @@` | Edge coverage only |
| 38 | `afl-tmin -x -i crash -o crash-min -- ./target @@` | Non-zero exit = crash |
| 39 | `afl-tmin -H -i hang -o hang-min -- ./target @@` | Minimize a hang |
| 40 | `afl-tmin -l 4 -i crash -o crash-min -- ./target @@` | Minimum deletion block |
| 41 | `afl-tmin -t 5000 -m 2048 -i crash -o crash-min -- ./target @@` | Limits for slow targets |
| 42 | `afl-tmin -Q -i crash -o crash-min -- ./target @@` | QEMU binary-only |
| 43 | `afl-addseeds -o findings -i new-seeds` | Add seeds to a live campaign |
| 44 | `afl-addseeds -o findings seed1.bin seed2.bin` | Add individual files |
| 45 | `afl-addseeds -o findings -i seeds/ extra/` | Mixed dirs and files |

**Research questions**
- Does `afl-cmin` preserve the exact edge set, and what does `-A` sacrifice?
- When does `afl-tmin` change crash reproducibility, and how do you verify?
- Can `afl-addseeds` race with a running fuzzer?

**Sources**: `docs/afl-cmin`, `docs/afl-tmin`, corpus minimization write-ups,
issues on unstable minimization.

**Output**: a minimized corpus with the original seed count and coverage count
recorded.

---

## Scenario 3 — baseline fuzzing run

| # | Command | How to use |
|---|---------|------------|
| 46 | `afl-fuzz -i corpus -o findings -- ./target @@` | Minimal run |
| 47 | `afl-fuzz -i corpus -o findings -- ./target` | stdin-based target |
| 48 | `afl-fuzz -i - -o findings -- ./target @@` | Resume an existing campaign |
| 49 | `afl-fuzz -V 3600 -i corpus -o findings -- ./target @@` | Fixed one-hour run |
| 50 | `afl-fuzz -E 1000000 -i corpus -o findings -- ./target @@` | Stop after ~1M execs |
| 51 | `afl-fuzz -s 1234 -i corpus -o findings -- ./target @@` | Reproducible RNG |
| 52 | `afl-fuzz -p fast -i corpus -o findings -- ./target @@` | Fast power schedule |
| 53 | `afl-fuzz -p rare -i corpus -o findings -- ./target @@` | Rare-edge schedule |
| 54 | `afl-fuzz -p explore -i corpus -o findings -- ./target @@` | Coverage exploration |
| 55 | `afl-fuzz -p exploit -i corpus -o findings -- ./target @@` | Exploit known paths |
| 56 | `afl-fuzz -p seek -i corpus -o findings -- ./target @@` | Seek new coverage |
| 57 | `afl-fuzz -p mmopt -i corpus -o findings -- ./target @@` | MOpt schedule |
| 58 | `afl-fuzz -p coe -i corpus -o findings -- ./target @@` | Cut-off exponential |
| 59 | `afl-fuzz -p lin -i corpus -o findings -- ./target @@` | Linear schedule |
| 60 | `afl-fuzz -p quad -i corpus -o findings -- ./target @@` | Quadratic schedule |
| 61 | `afl-fuzz -P explore -i corpus -o findings -- ./target @@` | Fix mutation strategy |
| 62 | `afl-fuzz -P 300 -i corpus -o findings -- ./target @@` | Switch to exploit after 300s |
| 63 | `afl-fuzz -t 500 -i corpus -o findings -- ./target @@` | 500 ms per run |
| 64 | `afl-fuzz -t 1000+ -i corpus -o findings -- ./target @@` | Auto-scale timeout |
| 65 | `afl-fuzz -m 2048 -i corpus -o findings -- ./target @@` | 2 GB memory cap |
| 66 | `afl-fuzz -m 0 -i corpus -o findings -- ./target @@` | No memory cap |
| 67 | `afl-fuzz -b 0 -i corpus -o findings -- ./target @@` | Bind to CPU 0 |
| 68 | `afl-fuzz -T "parser-1" -i corpus -o findings -- ./target @@` | Custom screen banner |
| 69 | `afl-fuzz -e bin -i corpus -o findings -- ./target @@` | Input file extension |
| 70 | `AFL_NO_UI=1 AFL_SKIP_CPUFREQ=1 afl-fuzz -V 300 -i corpus -o findings -- ./target @@` | CI-friendly run |
| 71 | `afl-fuzz -hh` | Show environment variables |

**Research questions**
- Which power schedule finds parser bugs fastest on structured input?
- What does `-t 1000+` actually auto-scale from, and when does it hide hangs?
- Is `-s` enough for reproducibility, or must the corpus and binary be pinned too?

**Sources**: `docs/FAQ.md`, schedule descriptions in `docs/`, benchmark posts,
AFL++ issues on timeouts.

**Output**: one canonical run command with schedule, timeouts, and stop
conditions recorded.

---

## Scenario 4 — input mode and binary-only targets

| # | Command | How to use |
|---|---------|------------|
| 72 | `afl-fuzz -f input.bin -i corpus -o findings -- ./target` | Fixed input file, no `@@` |
| 73 | `afl-fuzz -Q -i corpus -o findings -- ./target @@` | QEMU mode, closed source |
| 74 | `afl-fuzz -O -i corpus -o findings -- ./target @@` | FRIDA mode, closed source |
| 75 | `afl-fuzz -U -i corpus -o findings -- ./target @@` | Unicorn mode |
| 76 | `afl-fuzz -W -i corpus -o findings -- ./target.exe @@` | QEMU + Wine |
| 77 | `afl-fuzz -X -i corpus -o findings -- ./target @@` | Nyx standalone VM mode |
| 78 | `afl-fuzz -Y -i corpus -o findings -- ./target @@` | Nyx multi-instance mode |
| 79 | `afl-fuzz -n -i corpus -o findings -- ./target @@` | Dumb fuzzing, no instrumentation |
| 80 | `afl-fuzz -N -i corpus -o findings -- ./target /dev/input` | Don't unlink input file |

**Research questions**
- QEMU vs FRIDA vs Unicorn for a stripped Linux binary: which is fastest and
  which handles threads?
- Does `-n` still find crashes worth reporting, and how do you dedupe them?
- What are Nyx's host requirements and snapshot limitations?

**Sources**: `qemu_mode/README.md`, `frida_mode/README.md`, `unicorn_mode/`,
Nyx docs, comparative benchmarks.

**Output**: the cheapest input mode that still gives coverage feedback, with
its speed penalty measured.

---

## Scenario 5 — mutation, dictionaries and CmpLog

| # | Command | How to use |
|---|---------|------------|
| 81 | `afl-fuzz -a text -i corpus -o findings -- ./target @@` | Text input hint |
| 82 | `afl-fuzz -a binary -i corpus -o findings -- ./target @@` | Binary input hint |
| 83 | `afl-fuzz -g 1 -G 4096 -i corpus -o findings -- ./target @@` | Generated length bounds |
| 84 | `afl-fuzz -x dict.txt -i corpus -o findings -- ./target @@` | Dictionary |
| 85 | `afl-fuzz -x http.dict -x tls.dict -i corpus -o findings -- ./target @@` | Up to four dictionaries |
| 86 | `afl-fuzz -c ./target.cmplog -i corpus -o findings -- ./target @@` | CmpLog binary |
| 87 | `afl-fuzz -c 0 -i corpus -o findings -- ./target @@` | Target is already CmpLog |
| 88 | `afl-fuzz -c - -i corpus -o findings -- ./target @@` | Disable CmpLog |
| 89 | `afl-fuzz -c ./target.cmplog -l 2ATR -i corpus -o findings -- ./target @@` | CmpLog options |
| 90 | `afl-fuzz -c ./target.cmplog -l 3X -i corpus -o findings -- ./target @@` | Extreme transforms |
| 91 | `afl-fuzz -L 0 -i corpus -o findings -- ./target @@` | MOpt immediately |
| 92 | `afl-fuzz -L -1 -i corpus -o findings -- ./target @@` | MOpt + normal mutation |
| 93 | `afl-fuzz -L 60 -i corpus -o findings -- ./target @@` | MOpt after 60 min dry spell |
| 94 | `afl-fuzz -z -i corpus -o findings -- ./target @@` | Skip deterministic stage |

**Research questions**
- What does each CmpLog option (`A`, `T`, `X`, `R`) cost in execs, and which
  helps on magic-byte checks?
- Does a dictionary beat CmpLog for text protocols?
- Is MOpt still effective in 4.21 or superseded by schedules?

**Sources**: `docs/FAQ.md`, `instrumentation/README.cmplog.md`,
`docs/custom_mutators.md`, benchmark write-ups.

**Output**: a mutation configuration tied to the input format, with expected
speed cost.

---

## Scenario 6 — parallel and distributed fuzzing

| # | Command | How to use |
|---|---------|------------|
| 95 | `afl-fuzz -i corpus -o findings -M main -- ./target @@` | Master instance |
| 96 | `afl-fuzz -i corpus -o findings -S s1 -- ./target @@` | Secondary instance |
| 97 | `afl-fuzz -i corpus -o findings -S s2 -- ./target @@` | More secondaries |
| 98 | `afl-fuzz -i corpus -o findings -M main -b 0 -- ./target @@` | Master pinned to CPU 0 |
| 99 | `afl-fuzz -i corpus -o findings -S s1 -b 1 -- ./target @@` | Secondary on CPU 1 |
| 100 | `afl-fuzz -i corpus -o findings -M main -F /shared/other -F /shared/third -- ./target @@` | Sync from foreign queues |
| 101 | `afl-fuzz -Z -i corpus -o findings -S seq -- ./target @@` | Sequential queue selection |
| 102 | `afl-fuzz -C -i corpus -o findings -- ./target @@` | Crash exploration mode |
| 103 | `afl-fuzz -I 'notify-send crash' -i corpus -o findings -- ./target @@` | Hook on new crash |
| 104 | `AFL_IMPORT_FIRST=1 afl-fuzz -i corpus -o findings -S s1 -- ./target @@` | Sync early |

**Research questions**
- How many instances per core, and does hyperthreading help or hurt?
- What does `-M` disable that `-S` keeps, and why does it matter for trimming?
- What are the bandwidth and locking requirements for a shared `-F` directory?

**Sources**: `docs/fuzzing_in_depth.md` section C, parallel fuzzing posts,
issues on sync performance.

**Output**: a launcher script with core pinning and a sync layout.

---

## Scenario 7 — observe, triage and replay

| # | Command | How to use |
|---|---------|------------|
| 105 | `afl-whatsup findings/` | Full status of all instances |
| 106 | `afl-whatsup -s findings/` | Summary only |
| 107 | `afl-whatsup -d findings/` | Include dead fuzzers |
| 108 | `afl-whatsup -m findings/` | Minimal stats |
| 109 | `afl-whatsup -n findings/` | No color |
| 110 | `afl-showmap -o map.txt -- ./target input` | Coverage map for one input |
| 111 | `afl-showmap -e -o map.txt -- ./target input` | Edge coverage only |
| 112 | `afl-showmap -r -o map.txt -- ./target input` | Real tuple values |
| 113 | `afl-showmap -s -o map.txt -- ./target input` | Unclassified map |
| 114 | `afl-showmap -q -o map.txt -- ./target input` | Sink target output |
| 115 | `afl-showmap -i corpus -o maps/ -- ./target @@` | One map per corpus file |
| 116 | `afl-showmap -I filelist -o maps/ -- ./target @@` | File list input |
| 117 | `afl-showmap -C -i corpus -o crashes.map -- ./target @@` | Collect crashing edges |
| 118 | `afl-showmap -c -o map.txt -- ./target input` | Allow core dumps |
| 119 | `afl-showmap -Q -o map.txt -- ./target input` | QEMU mode |
| 120 | `afl-analyze -i input.bin -- ./target @@` | Per-byte input influence |
| 121 | `afl-analyze -e -i input.bin -- ./target @@` | Edge coverage only |
| 122 | `afl-analyze -f input.bin -i input.bin -- ./target` | Fixed input file |
| 123 | `afl-plot findings/default plots/` | Generate coverage/crash plots |
| 124 | `afl-plot -g findings/default plots/` | Also open a plot window |
| 125 | `ls findings/default/crashes/` | List crashes |
| 126 | `gdb --args ./target findings/default/crashes/id:000000*` | Reproduce under GDB |

**Research questions**
- Which `afl-whatsup` counter predicts a stalled campaign earliest?
- How do you deduplicate crashes by root cause rather than by input hash?
- What does `afl-analyze` tell you that coverage maps do not?

**Sources**: `docs/` tool pages, exploit-dev triage guides, AFL++ issues.

**Output**: a triage log with crash dedupe key, repro command, and severity.

---

## Scenario 8 — network targets

| # | Command | How to use |
|---|---------|------------|
| 127 | `afl-network-client 127.0.0.1 1337` | Connect to a proxy server |
| 128 | `afl-network-client ::1 1337 65536` | IPv6, custom max input |
| 129 | `afl-network-client 127.0.0.1%eth0 1337` | Bind to an interface |
| 130 | `afl-network-server -i 1337 -- ./target` | Proxy stdin to TCP |
| 131 | `afl-network-server -i 1337 -t 2000 -m 2048 -- ./target` | Limits |
| 132 | `afl-network-server -i 1337 -f input.bin -- ./target` | Fixed input file |
| 133 | `afl-network-server -i 1337 -Q -- ./target` | QEMU mode |

**Research questions**
- Is `afl-network-server` a substitute for an in-process harness, and what
  latency does it add?
- How do you handle protocols that need a handshake before the fuzzed bytes?
- What is the `AFL_MAP_SIZE` requirement for large targets?

**Sources**: `utils/afl_network_proxy/README.md`, network fuzzing write-ups,
issues on map size.

**Output**: measured exec/s for proxy vs harness, and the chosen approach.

---

## Scenario 9 — host tuning and CPU state

| # | Command | How to use |
|---|---------|------------|
| 134 | `afl-gotcpu` | Show available/busy CPUs |
| 135 | `sudo afl-system-config` | Runtime tuning (reduces security) |
| 136 | `sudo afl-persistent-config` | Permanent tuning (reduces security) |
| 137 | `cat /proc/sys/kernel/core_pattern` | Check crash handling |
| 138 | `cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor` | Check governor |
| 139 | `echo performance \| sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor` | Manual governor fix |

**Research questions**
- Exactly which sysctls do `afl-system-config` and `afl-persistent-config`
  change, and how do you revert them?
- Why does `core_pattern` break crash detection, and is a pipe to `apport`
  always fatal?
- Are these tuning scripts safe on a shared workstation?

**Sources**: script source, kernel docs, `docs/FAQ.md`, issues on core_pattern.

**Output**: a documented tuning diff and a revert procedure.

---

## Complete flag coverage

Every flag from the installed help output appears at least once.

| Tool | Flags / forms | Scenario |
|------|---------------|----------|
| `afl-fuzz` | `-i`, `-o`, `-P`, `-p`, `-f`, `-t`, `-m`, `-O`, `-Q`, `-U`, `-W`, `-X`, `-Y`, `-a`, `-g`, `-G`, `-L`, `-c`, `-l`, `-Z`, `-N`, `-n`, `-x`, `-s`, `-V`, `-E`, `-M`, `-S`, `-F`, `-z`, `-T`, `-I`, `-C`, `-b`, `-e`, `-hh` | 3–6 |
| `afl-cmin` | `-i`, `-o`, `-T`, `-f`, `-m`, `-t`, `-O`, `-Q`, `-U`, `-X`, `-A`, `-C`, `-e` | 2 |
| `afl-tmin` | `-i`, `-o`, `-f`, `-t`, `-m`, `-O`, `-Q`, `-U`, `-W`, `-X`, `-e`, `-l`, `-x`, `-H` | 2 |
| `afl-analyze` | `-i`, `-f`, `-t`, `-m`, `-O`, `-Q`, `-U`, `-W`, `-X`, `-e` | 7 |
| `afl-showmap` | `-o`, `-t`, `-m`, `-O`, `-Q`, `-U`, `-W`, `-X`, `-i`, `-I`, `-C`, `-q`, `-e`, `-r`, `-s`, `-c` | 7 |
| `afl-whatsup` | `-d`, `-m`, `-n`, `-s` | 7 |
| `afl-gotcpu` | none | 9 |
| `afl-addseeds` | `-o`, `-i` | 2 |
| `afl-plot` | `-g/--graphical` | 7 |
| `afl-system-config` | none | 9 |
| `afl-persistent-config` | none | 9 |
| `afl-network-client` | `host port [max-input-size]`, `host%iface port` | 8 |
| `afl-network-server` | `-i` (port), `-f`, `-t`, `-m`, `-Q`, `-U`, `-W` | 8 |
| `afl-cc` / `afl-c++` | `--afl-llvm`, `--afl-lto`, `--afl-gcc`; `-h`, `-hh`; normal compiler flags | 1 |
| Wrapper symlinks | `afl-clang`, `afl-clang++`, `afl-clang-fast`, `afl-clang-fast++`, `afl-clang-lto`, `afl-clang-lto++`, `afl-gcc`, `afl-g++`, `afl-gcc-fast`, `afl-g++-fast`, `afl-lto`, `afl-lto++`, `afl-ld-lto` | 1 |

Environment variables that change behavior are documented by `afl-fuzz -hh`
and `afl-cc -hh`; the ones used above are `AFL_USE_*SAN`, `AFL_HARDEN`,
`AFL_LLVM_CMPLOG`, `AFL_LLVM_DICT2FILE`, `AFL_INST_RATIO`,
`AFL_LLVM_INSTRUMENT`, `AFL_NO_UI`, `AFL_SKIP_CPUFREQ`, `AFL_IMPORT_FIRST`.

Full command text: [`help.txt`](help.txt); re-capture companion help with
`<tool> -h`.
