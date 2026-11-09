# strace — command scenarios for deep research

Installed build: strace 6.13 (Debian `6.13+ds-1`), with
`stack-trace=libunwind stack-demangle m32-mpers mx32-mpers`. Every option
below appears in [`help.txt`](help.txt); semantics come from `man strace` and
observed output.

> **Scope:** the recipes trace throwaway programs and a few harmless system
> binaries. `strace` observes the kernel boundary; it can reveal credentials,
> tokens, file contents, and network endpoints. Treat a trace as sensitive
> data, and do not attach to processes you are not authorised to inspect.

> **Filter gotcha:** `-e signal=` filters signals, not syscalls. Use
> `-e trace=none -e signal=SIG` for signal-only output. Multiple
> `-e trace=` filters are unioned, so a negation followed by a group re-adds
> the excluded call.

## How to invoke

```text
@deep-research For an authorised troubleshooting task with strace 6.13,
explain how the `-e trace=` grammar (syscalls, groups, regexes, negation,
personality suffixes), `-e status=`, `-e trace-fds=`, `-P`, `-z`/`-Z`,
`-f`/`-ff`, `-c`/`-C`, and `-e inject=`/`-e fault=` interact. Show how
`--seccomp-bpf`, `--syscall-limit`, and `-D`/`-DD`/`-DDD` change the tracer's
process model, and how `-y`/`-yy`/`--decode-fds` and `-k` derive their
annotations. Cite the installed help text, `man strace`, and observed output.
Separate confirmed behaviour from inference and state confidence.
```

Create the demo tree and the test programs:

```bash
mkdir -p ~/strace-demo/evidence
cd ~/strace-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
cat > w.c <<'EOF'
#include <stdio.h>
#include <unistd.h>
int main(void) { write(1, "hello\n", 6); return 0; }
EOF
cat > f.c <<'EOF'
#include <stdio.h>
#include <unistd.h>
int main(void) {
    if (fork() == 0) { write(1, "child\n", 6); _exit(0); }
    write(1, "parent\n", 7);
    return 0;
}
EOF
gcc -o w w.c
gcc -o f f.c
```

## Scenario 1 — version, help, build features

| # | Command | Purpose |
|---|---------|---------|
| 1 | `strace -V` | Version and optional features |
| 2 | `strace -V 2>&1 \| tail -1` | The feature line only |
| 3 | `strace -h \| head -20` | Usage and general options |
| 4 | `strace -h \| grep -A20 'Filtering:'` | Filtering section |
| 5 | `strace -h \| grep -A20 'Tampering:'` | Tampering section |
| 6 | `man strace \| col -b \| head -40` | Authoritative manual |
| 7 | `man strace \| col -b \| grep -n '^EXIT STATUS'` | Exit-status section |
| 8 | `command -v strace` | Resolve the binary |
| 9 | `file "$(command -v strace)"` | Confirm a native ELF |
| 10 | `dpkg -s strace \| grep -E 'Version\|Depends'` | Package provenance |
| 11 | `cat /proc/sys/kernel/yama/ptrace_scope` | Attach policy |
| 12 | `id -u` | Whether privileged tracing is available |
| 13 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt` | Timestamp the run |

**Research questions**
- Which optional features are compiled in?
- Does this build support `-k`?
- What does `ptrace_scope` restrict?
- Where is the authoritative option list?

**Sources:** `strace -V`; `man strace`; `/proc/sys/kernel/yama/ptrace_scope`.
**Output:** a version/feature record and the attach policy.

## Scenario 2 — basic invocation, streams, exit propagation

| # | Command | Purpose |
|---|---------|---------|
| 14 | `strace ./w` | Trace to stderr |
| 15 | `strace ./w 2>/dev/null` | Confirm program stdout is separate |
| 16 | `strace ./w 2>&1 \| head -5` | Merge trace and program output |
| 17 | `strace -o trace.log ./w && head -5 trace.log` | Write the trace to a file |
| 18 | `strace -o trace.log ./w > stdout.log` | Separate all three streams |
| 19 | `cat stdout.log` | Program stdout only |
| 20 | `strace /bin/true >/dev/null 2>&1; echo "true=$?"` | Child exit 0 |
| 21 | `strace /bin/false >/dev/null 2>&1; echo "false=$?"` | Child exit 1 |
| 22 | `strace sh -c 'exit 42' >/dev/null 2>&1; echo "exit42=$?"` | Arbitrary child status |
| 23 | `strace sh -c 'kill -TERM $$' >/dev/null 2>&1; echo "sig=$?"` | Signal death status |
| 24 | `strace -e trace=write ./w 2>&1 \| head -3` | One syscall only |
| 25 | `strace -e trace=none ./w 2>&1` | No syscall lines, exit line only |
| 26 | `strace -q -e trace=none ./w 2>&1` | Suppress the exit line |
| 27 | `strace -qq -e trace=none ./w 2>&1` | Suppress more messages |

**Research questions**
- Which stream carries the trace by default?
- Does strace propagate the child's exit status?
- What does `-e trace=none` leave behind?
- Which quiet level removes the exit line?

**Sources:** observed output; `man strace`.
**Output:** a stream/exit-status table.

## Scenario 3 — syscall filtering by name, group, regex, status

| # | Command | Purpose |
|---|---------|---------|
| 28 | `strace -e trace=write ./w 2>&1 \| head -3` | Single syscall |
| 29 | `strace -e trace=write,read ./w 2>&1 \| head -4` | Comma-separated set |
| 30 | `strace -e trace='!write' ./w 2>&1 \| grep -c '^write'` | Negation excludes write |
| 31 | `strace -e trace=/open /bin/true 2>&1 \| head -4` | Regex matches `openat` too |
| 32 | `strace -e trace=%file /bin/true 2>&1 \| wc -l` | File-operation group |
| 33 | `strace -e trace=%desc /bin/true 2>&1 \| wc -l` | Descriptor group |
| 34 | `strace -e trace=%process /bin/true 2>&1 \| head -4` | Process group |
| 35 | `strace -e trace=%memory /bin/true 2>&1 \| head -4` | Memory group |
| 36 | `strace -e trace=%net /bin/true 2>&1 \| head -3` | Network group |
| 37 | `strace -e trace=%pure /bin/true 2>&1 \| head -3` | Pure calls only |
| 38 | `strace -e trace='!openat' -e trace=%file /bin/true 2>&1 \| grep -c openat` | Union re-adds openat |
| 39 | `strace -e trace=write@64 ./w 2>&1 \| head -3` | 64-bit personality |
| 40 | `strace -e trace=write@32 ./w 2>&1 \| head -3` | 32-bit filter does not match |
| 41 | `strace -e status=failed -e trace=%file /bin/true 2>&1 \| head -4` | Failed calls only |
| 42 | `strace -e status=successful -e trace=%file /bin/true 2>&1 \| head -4` | Successful calls only |
| 43 | `strace -z -e trace=openat /bin/true 2>&1 \| head -4` | `-z` shorthand |
| 44 | `strace -Z -e trace=openat /bin/true 2>&1 \| head -4` | `-Z` shorthand |

**Research questions**
- Does a regex filter match by substring or by full name?
- Are multiple `-e trace=` filters intersected or unioned?
- What does a personality suffix change?
- Do `-z`/`-Z` and `-e status=` agree?

**Sources:** `man strace`; observed counts.
**Output:** a filter-semantics table with counts.

## Scenario 4 — path and descriptor filtering

| # | Command | Purpose |
|---|---------|---------|
| 45 | `strace -P /etc/ld.so.cache /bin/true 2>&1 \| head -5` | Only accesses to one path |
| 46 | `strace -P /etc/ld.so.cache /bin/true 2>&1 \| grep -c 'ld.so.cache'` | Confirm the restriction |
| 47 | `strace -P /nonexistent /bin/true 2>&1 \| head -3` | No matching path |
| 48 | `strace -e trace-fds=1 -e trace=write ./w 2>&1 \| head -3` | Operations on fd 1 |
| 49 | `strace -e trace-fds=0,1,2 -e trace=read,write ./w 2>&1 \| head -5` | Standard descriptors |
| 50 | `strace -e trace-fds=999 -e trace=write ./w 2>&1 \| head -3` | Unused descriptor |
| 51 | `strace -y -e trace=openat /bin/true 2>&1 \| head -3` | Decode fd paths |
| 52 | `strace -yy -e trace=openat /bin/true 2>&1 \| head -3` | Decode all fd details |
| 53 | `strace -y -e trace=openat /bin/true 2>&1 \| grep -o '<[^>]*>' \| head -3` | Extract the annotations |
| 54 | `strace --decode-fds=path -e trace=openat /bin/true 2>&1 \| head -2` | Explicit path decoding |
| 55 | `strace --decode-fds=dev -e trace=openat /bin/true 2>&1 \| head -2` | Device decoding |
| 56 | `strace --decode-fds=socket -e trace=openat /bin/true 2>&1 \| head -2` | Socket decoding |
| 57 | `strace --decode-fds=pid -e trace=openat /bin/true >/dev/null 2>&1; echo "pid=$?"` | Invalid detail name |
| 58 | `strace -e trace=openat -P /etc/hostname /bin/cat /etc/hostname 2>&1 \| head -5` | Path plus syscall filter |

**Research questions**
- Does `-P` filter by path argument or by resolved inode?
- Which descriptor details does `-yy` add over `-y`?
- Which `--decode-fds` names are accepted?
- Does a descriptor filter require a matching syscall filter?

**Sources:** `man strace`; observed output.
**Output:** a path/fd filter matrix with decoded annotations.

## Scenario 5 — output formatting, timestamps, strings, stack traces

| # | Command | Purpose |
|---|---------|---------|
| 59 | `strace -a 60 -e trace=write ./w 2>&1 \| head -3` | Wider alignment |
| 60 | `strace -i -e trace=write ./w 2>&1 \| head -2` | Instruction pointer |
| 61 | `strace -n -e trace=write ./w 2>&1 \| head -2` | Syscall number |
| 62 | `strace -t -e trace=write ./w 2>&1 \| head -2` | Absolute seconds |
| 63 | `strace -tt -e trace=write ./w 2>&1 \| head -2` | Microsecond timestamps |
| 64 | `strace -ttt -e trace=write ./w 2>&1 \| head -2` | UNIX timestamps |
| 65 | `strace -r -e trace=write ./w 2>&1 \| head -2` | Relative timestamps |
| 66 | `strace --absolute-timestamps=format:unix,precision:us -e trace=write ./w 2>&1 \| head -2` | Custom timestamp form |
| 67 | `strace -T -e trace=write ./w 2>&1 \| head -2` | Per-call duration |
| 68 | `strace -s 3 -e trace=write ./w 2>&1 \| head -2` | Truncated string |
| 69 | `strace -s 256 -e trace=write ./w 2>&1 \| head -2` | Full string |
| 70 | `strace -v -e trace=write ./w 2>&1 \| head -2` | Unabbreviated structures |
| 71 | `strace -x -e trace=write ./w 2>&1 \| head -2` | Non-ASCII as hex |
| 72 | `strace -xx -e trace=write ./w 2>&1 \| head -2` | All strings as hex |
| 73 | `strace --strings-in-hex=non-ascii-chars -e trace=write ./w 2>&1 \| head -2` | Hex escape policy |
| 74 | `strace -X raw -e trace=openat /bin/true 2>&1 \| head -2` | Raw constants |
| 75 | `strace -X abbrev -e trace=openat /bin/true 2>&1 \| head -2` | Abbreviated constants |
| 76 | `strace -X verbose -e trace=openat /bin/true 2>&1 \| head -2` | Verbose constants |
| 77 | `strace -X abbreviated /bin/true >/dev/null 2>&1; echo "badX=$?"` | Invalid spelling exits 1 |
| 78 | `strace -k -e trace=write ./w 2>&1 \| head -8` | Stack trace per call |
| 79 | `strace -k --stack-trace-frame-limit=2 -e trace=write ./w 2>&1 \| head -6` | Frame cap |
| 80 | `strace -Y -e trace=write ./w 2>&1 \| head -2` | Decode PIDs to names |
| 81 | `strace --always-show-pid -e trace=write ./w 2>&1 \| head -2` | PID on the first process |
| 82 | `strace --decode-pids=pidns -e trace=write ./w 2>&1 \| head -2` | Namespace PID view |

**Research questions**
- Which timestamp options are cumulative (`-tt` vs `-ttt`)?
- Does `-s` affect non-string arguments?
- What does `-X verbose` add to a constant?
- Does `-k` require a separate debug package?
- Is the PID prefix stable across `-f`?

**Sources:** `man strace`; observed output.
**Output:** the same call rendered in a dozen formats.

## Scenario 6 — forks, daemonization, output files, syscall limit

| # | Command | Purpose |
|---|---------|---------|
| 83 | `strace ./f 2>&1 \| head -6` | Without `-f`, only the parent |
| 84 | `strace -f ./f 2>&1 \| head -10` | Follow the child |
| 85 | `strace -f -o fork.log ./f && wc -l fork.log` | One merged file |
| 86 | `rm -f ff.* && strace -ff -o ff ./f && ls ff.*` | One file per PID |
| 87 | `rm -f sep.* && strace --output-separately -o sep ./f && ls sep.*` | Explicit separate output |
| 88 | `strace -f -e trace=clone,fork,vfork,execve ./f 2>&1 \| head -6` | Process-creation calls |
| 89 | `rm -f app.log && strace -A -o app.log -e trace=none ./w && strace -A -o app.log -e trace=none ./w && wc -l app.log` | Append mode |
| 90 | `strace -D -o d.log ./w && head -2 d.log` | Tracer as a grandchild |
| 91 | `strace -DD -o dd.log ./w` | Separate process group |
| 92 | `strace -DDD -o ddd.log ./w` | Separate session |
| 93 | `strace -b execve -e trace=execve /bin/true 2>&1 \| head -3` | Detach on execve |
| 94 | `strace --syscall-limit=2 -o lim.log /bin/true; echo "exit=$?"; cat lim.log` | Detach after two calls |
| 95 | `strace --kill-on-exit -e trace=none /bin/true >/dev/null 2>&1; echo "koe=$?"` | Kill-on-exit option accepted |
| 96 | `strace -I 1 -e trace=none /bin/true 2>&1 \| head -2` | Signal-blocking mode |
| 97 | `strace -I 4 -e trace=none /bin/true 2>&1 \| head -2` | `never_tstp` mode |

**Research questions**
- Does `-f` merge or separate output by default?
- How does `-ff` name its files?
- Does `--syscall-limit` kill or detach?
- What changes when the tracer is daemonized?
- Does `-b execve` stop tracing at the first exec?

**Sources:** `man strace`; observed file layout.
**Output:** a fork/output-mode matrix.

## Scenario 7 — statistics and cost accounting

| # | Command | Purpose |
|---|---------|---------|
| 98 | `strace -c /bin/true 2>&1 \| head -8` | Default summary |
| 99 | `strace -c -S calls /bin/true 2>&1 \| head -8` | Sort by call count |
| 100 | `strace -c -S name /bin/true 2>&1 \| head -8` | Sort by name |
| 101 | `strace -c -S min-time /bin/true 2>&1 \| head -8` | Sort by minimum time |
| 102 | `strace -c -S max-time /bin/true 2>&1 \| head -8` | Sort by maximum time |
| 103 | `strace -c -S avg-time /bin/true 2>&1 \| head -8` | Sort by average time |
| 104 | `strace -c -S errors /bin/true 2>&1 \| head -8` | Sort by error count |
| 105 | `strace -c -S nothing /bin/true 2>&1 \| head -8` | Keep insertion order |
| 106 | `strace -c -U name,calls,errors /bin/true 2>&1 \| head -6` | Custom columns |
| 107 | `strace -c -U time-percent,total-time,min-time,max-time,avg-time,calls,errors,name /bin/true 2>&1 \| head -5` | All columns |
| 108 | `strace -c -U bogus /bin/true >/dev/null 2>&1; echo "badU=$?"` | Unknown column exits 1 |
| 109 | `strace -c -w /bin/true 2>&1 \| head -6` | Wall-clock latency |
| 110 | `strace -c -O 10us /bin/true 2>&1 \| head -6` | Subtract per-call overhead |
| 111 | `strace -C -e trace=write ./w 2>&1 \| head -12` | Summary plus trace |
| 112 | `strace -c -e trace=%file /bin/true 2>&1 \| head -8` | Summary of a filtered set |

**Research questions**
- Does `-c` report system time or wall time by default?
- How does `-O` change the totals?
- Are filtered-out calls included in the summary?
- Which column is the best proxy for cost?
- Does `-C` duplicate the trace lines?

**Sources:** `man strace`; observed tables.
**Output:** a sort/column matrix and a cost table.

## Scenario 8 — fault injection and tampering

| # | Command | Purpose |
|---|---------|---------|
| 113 | `strace -e inject=write:retval=0 -e trace=write ./w 2>&1 \| head -3` | Short-circuit a write |
| 114 | `strace -e inject=write:error=EPIPE -e trace=write ./w 2>&1 \| head -3` | Return an errno |
| 115 | `strace -e inject=write:signal=SIGPIPE -e trace=write ./w 2>&1 \| head -4` | Deliver a signal |
| 116 | `strace -e inject=write:delay_enter=10ms -e trace=write ./w 2>&1 \| head -3` | Delay on entry |
| 117 | `strace -e inject=write:delay_exit=10ms -e trace=write ./w 2>&1 \| head -3` | Delay on exit |
| 118 | `strace -e inject=write:retval=0:when=1 -e trace=write ./w 2>&1 \| head -3` | First occurrence only |
| 119 | `strace -e inject=write:retval=0:when=1+ -e trace=write ./w 2>&1 \| head -3` | First and later |
| 120 | `strace -e inject=write:retval=0:when=1+2 -e trace=write ./w 2>&1 \| head -3` | Calls 1, 3, 5, … |
| 121 | `strace -e inject=write:retval=0:when=..3 -e trace=write ./w >/dev/null 2>&1; echo "range=$?"` | Invalid range syntax |
| 122 | `strace -e fault=openat:error=EACCES -e trace=openat /bin/true 2>&1 \| head -4` | `-e fault` synonym |
| 123 | `strace -e inject=openat:error=EACCES:when=1 -e trace=openat /bin/true 2>&1 \| head -4` | Fail the first open |
| 124 | `strace -e inject=read:retval=0 -e trace=read /bin/cat /etc/hostname 2>&1 \| head -4` | Simulate EOF |
| 125 | `strace -e inject=mmap:error=ENOMEM -e trace=mmap /bin/true 2>&1 \| head -4` | Fail an allocation |
| 126 | `strace -e inject=write:poke_enter=@arg2=6869 -e trace=write ./w 2>&1 \| head -3` | Overwrite memory pointed to by arg2 (hex) |
| 127 | `strace -e inject=write:syscall=write -e trace=write ./w 2>&1 \| head -3` | Redirect to the same syscall |

**Research questions**
- What is the difference between `:error=` and `:retval=`?
- Is the injected call still printed?
- Does `when=1+2` mean “calls 1, 3, 5, …” or “calls 1 and 2”?
- Can injection make a program hang?
- Does `-e fault` default to `ENOSYS`?

**Sources:** `man strace`; observed `(INJECTED)` markers.
**Output:** an injection matrix with observed effects.

## Scenario 9 — attach, signals, seccomp, cleanup

| # | Command | Purpose |
|---|---------|---------|
| 128 | `strace -p 999999 >/dev/null 2>&1; echo "attach=$?"` | Attach failure exits 1 |
| 129 | `sleep 30 & pid=$!; strace -p "$pid" -e trace=none & tracer=$!; sleep 1; kill "$pid" "$tracer" 2>/dev/null` | Attach to a live process |
| 130 | `sleep 30 & pid=$!; strace -p "$pid" -f -tt -T -o attach.log & tracer=$!; sleep 1; kill "$pid" "$tracer" 2>/dev/null; wc -l attach.log` | Attach with timestamps |
| 131 | `strace -e signal=SIGUSR1 /bin/true 2>&1 \| head -3` | Signal filter does not hide syscalls |
| 132 | `strace -e trace=none -e signal=SIGUSR1 /bin/true 2>&1 \| head -3` | Signal-only view |
| 133 | `strace -e trace=none sh -c 'kill -USR1 $$' 2>&1 \| head -4` | Observe a delivered signal |
| 134 | `strace -e quiet=attach,exit -e trace=none /bin/true 2>&1 \| head -3` | Selective quieting |
| 135 | `strace -qqq -e trace=none /bin/true 2>&1 \| head -3` | Fully quiet |
| 136 | `strace --seccomp-bpf -e trace=openat /bin/true 2>&1 \| head -3` | Warning without `-f` |
| 137 | `strace -f --seccomp-bpf -e trace=openat /bin/true 2>&1 \| head -3` | Seccomp filtering with `-f` |
| 138 | `strace -d -e trace=none /bin/true 2>&1 \| head -4` | Debug diagnostics |
| 139 | `strace -e bogus=1 /bin/true >/dev/null 2>&1; echo "bade=$?"` | Invalid expression exits 1 |
| 140 | `strace --tips /bin/true 2>&1 \| tail -3` | Tips output |
| 141 | `strace --tips=id:1,format:none /bin/true 2>&1 \| tail -3` | Specific tip, no formatting |
| 142 | `strace --tips=1:none /bin/true >/dev/null 2>&1; echo "badtips=$?"` | Colon form is rejected |
| 143 | `strace -E FOO=bar -e trace=execve /bin/true 2>&1 \| head -2` | Set an environment variable |
| 144 | `strace --argv0=custom -e trace=execve /bin/true 2>&1 \| head -2` | Override `argv[0]` |
| 145 | `rm -rf ~/strace-demo` | Remove the demo tree |
| 146 | `test ! -e ~/strace-demo; echo "removed=$?"` | Verify cleanup |

**Research questions**
- Why does `-e signal=` alone still print syscalls?
- Does `--seccomp-bpf` fail or degrade without `-f`?
- What does `-qqq` suppress that `-q` does not?
- Which attach failures are visible to `$?`?
- Does `--tips` pollute a scripted log?

**Sources:** `man strace`; observed output.
**Output:** an attach/signal/seccomp table and a cleaned workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `-V`, `-h`, no arguments | 1 |
| `-o`, `-A`, `--output-separately`, `-q`/`-qq`/`-qqq`, `-e trace=none` | 2 |
| `-e trace=` names/groups/regex/negation, `@64`/`@32`, `-e status=`, `-z`, `-Z` | 3 |
| `-P`, `-e trace-fds=`, `-y`, `-yy`, `--decode-fds=` | 4 |
| `-a`, `-i`, `-n`, `-t`/`-tt`/`-ttt`, `-r`, `--absolute-timestamps`, `-T`, `-s`, `-v`, `-x`/`-xx`, `--strings-in-hex`, `-X`, `-k`, `--stack-trace-frame-limit`, `-Y`, `--always-show-pid`, `--decode-pids` | 5 |
| `-f`, `-ff`, `--output-separately`, `-D`/`-DD`/`-DDD`, `-b`, `--syscall-limit`, `--kill-on-exit`, `-I` | 6 |
| `-c`, `-C`, `-O`, `-S`, `-U`, `-w` | 7 |
| `-e inject=`, `-e fault=`, `:error=`, `:retval=`, `:signal=`, `:delay_enter=`, `:delay_exit=`, `:poke_enter=`, `:syscall=`, `:when=` | 8 |
| `-p`, `-e signal=`, `-e quiet=`, `--seccomp-bpf`, `-d`, `--tips`, `-E`, `--argv0` | 9 |

Full text: [`help.txt`](help.txt); authoritative description in `man strace`.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, help, build features | 13 |
| 2 | Basic invocation, streams, exit propagation | 14 |
| 3 | Syscall filtering by name, group, regex, status | 17 |
| 4 | Path and descriptor filtering | 14 |
| 5 | Output formatting, timestamps, strings, stack traces | 24 |
| 6 | Forks, daemonization, output files, syscall limit | 15 |
| 7 | Statistics and cost accounting | 15 |
| 8 | Fault injection and tampering | 15 |
| 9 | Attach, signals, seccomp, cleanup | 19 |
| **Total** | | **146** |

## Legal and safety

A trace is a detailed record of what a process read, wrote, opened, connected
to, and which credentials it handled. It can contain passwords, tokens, file
contents, and personal data. Do not attach to processes you are not authorised
to inspect, do not leave traces on shared hosts, and redact them before
sharing. Fault injection changes program behaviour; never use `-e inject=` on
a production process unless the failure is the experiment. Delete the demo
tree when finished.
