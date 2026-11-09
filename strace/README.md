# strace — trace system calls and signals

- **Version:** 6.13 (Debian `strace` `6.13+ds-1`)
- **Binary:** `/usr/bin/strace`
- **Help snapshot:** [`help.txt`](help.txt) — `strace -h`
- **Man page:** `man strace`
- **Upstream:** https://strace.io/
- **Source:** https://github.com/strace/strace
- **Licence:** LGPL-2.1-or-later (see `/usr/share/doc/strace/copyright`)
- **Optional features:** `stack-trace=libunwind`, `stack-demangle`, `m32-mpers`, `mx32-mpers`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

> **Filter gotcha:** `-e signal=SIG` filters *signals*, not syscalls. If you
> want signal-only output, use `-e trace=none -e signal=SIG`. Likewise,
> multiple `-e trace=` options are **unioned**, so `-e trace='!openat' -e
> trace=%file` re-includes `openat`.

## What it does

strace runs a program under `ptrace(2)` and prints every system call it makes,
with decoded arguments and the return value, plus signals delivered to the
process. It is the first tool to reach for when a program fails for a reason
the source does not explain: missing file, wrong permissions, unexpected
`ENOSYS`, a library loaded from the wrong path, a timeout, a signal that
killed it.

It is not a debugger. It cannot read or change program variables, and it does
not understand application-level protocols. It observes the kernel boundary,
which is exactly why it works on stripped binaries, shell scripts, and
programs whose source you do not have.

## Invocation forms

```bash
strace ./prog arg1 arg2          # run and trace
strace -p 1234                   # attach to a running process
strace -f -o trace.log ./prog    # follow forks, write to a file
strace -c ./prog                 # summary table only
strace -C ./prog                 # summary plus the normal trace
```

`strace` prints to **stderr** by default. Redirecting stdout does not capture
the trace; use `-o FILE` or `2> trace.log`.

## Filtering

| Option | Meaning |
|--------|---------|
| `-e trace=SET` | Trace only these syscalls |
| `-e trace=!SET` | Trace everything except these |
| `-e trace=/REGEX` | Match syscall names by regular expression |
| `-e trace=all` | Trace all syscalls (default) |
| `-e trace=none` | Trace no syscalls (useful for signal-only runs) |
| `-e signal=SET` | Print only these signals |
| `-e status=SET` | Keep calls by result: `successful`, `failed`, `unfinished`, `unavailable`, `detached` |
| `-e trace-fds=SET` | Trace operations on the listed file descriptors |
| `-P PATH` | Trace accesses to a path |
| `-z` | Successful calls only |
| `-Z` | Failed calls only |

### Syscall groups

| Group | Covers |
|-------|--------|
| `%clock` | time, clock_* |
| `%creds` | uid/gid, capabilities |
| `%desc` | file descriptors: open, read, write, close, mmap, fstat, … |
| `%file` | path-based file operations |
| `%fstat` | fstat family |
| `%ipc` | System V IPC |
| `%lstat` | lstat family |
| `%memory` | brk, mmap, mprotect, mremap, … |
| `%net` | socket, connect, send/recv, … |
| `%process` | clone, execve, exit, wait, … |
| `%pure` | calls that do not change state |
| `%signal` | signal handling |
| `%stat` | stat family |
| `%statfs` | statfs family |

`%%stat`, `%%statfs`, `%fstatfs` are the “excluding” variants: `%%stat` means
`%stat` minus the calls already in `%fstat`/`%lstat`.

A syscall can also be restricted to a personality: `write@64`, `write@32`,
`write@x32`.

## Output format

| Option | Meaning |
|--------|---------|
| `-a COLUMN` | Align return values at column (default 40) |
| `-i` | Print the instruction pointer |
| `-n` | Print the syscall number |
| `-k` | Print a stack trace for each syscall |
| `--stack-trace-frame-limit=N` | Cap stack frames (default 256) |
| `-o FILE` | Write to a file |
| `-A` | Append to the `-o` file |
| `--output-separately` | One file per PID |
| `-q`, `-qq`, `-qqq` | Suppress attach/exit messages |
| `-r` | Relative timestamps |
| `-t`, `-tt`, `-ttt` | Absolute timestamps (seconds, µs, UNIX µs) |
| `--absolute-timestamps=…` | Custom timestamp format/precision |
| `-T` | Time spent in each syscall |
| `-v` | Do not abbreviate structures |
| `-s STRSIZE` | String limit (default 32) |
| `-x`, `-xx` | Hex for non-ASCII, or all strings |
| `--strings-in-hex=…` | Hex escape policy |
| `-X FORMAT` | Constant style: `raw`, `abbrev`, `verbose` |
| `-y` | Decode file-descriptor paths |
| `-yy` | Decode all available fd information |
| `--decode-fds=SET` | `dev`, `eventfd`, `path`, `pidfd`, `socket`, `signalfd` |
| `-Y` | Decode PIDs to command names |
| `--decode-pids=pidns` | Also print PIDs in strace's namespace |
| `--always-show-pid` | Prefix the initial process too |

`-X abbreviated` is **not** accepted; the valid spellings are `raw`,
`abbrev`, and `verbose`.

## Statistics

| Option | Meaning |
|--------|---------|
| `-c` | Summary only |
| `-C` | Summary plus normal trace |
| `-O OVERHEAD` | Per-syscall overhead to subtract (default µs; accepts `s`, `ms`, `us`, `ns`) |
| `-S SORTBY` | Sort by `time`, `min-time`, `max-time`, `avg-time`, `calls`, `errors`, `name`, `nothing` |
| `-U COLUMNS` | Choose columns: `time-percent`, `total-time`, `min-time`, `max-time`, `avg-time`, `calls`, `errors`, `name` |
| `-w` | Wall-clock latency instead of system time |

The default columns are
`time-percent,total-time,avg-time,calls,errors,name`. `-c -p PID` works on
this build: attach, let the target exit, and the summary is printed.

## Tampering

`-e inject=` and `-e fault=` modify syscall behaviour without changing the
program. They are useful for fault-injection tests: make `open` return
`EACCES`, make `write` return a short count, delay a call, or deliver a
signal.

```bash
strace -e inject=openat:error=EACCES ./prog
strace -e inject=write:retval=0 ./prog
strace -e inject=read:delay_enter=100ms ./prog
strace -e fault=mmap:error=ENOMEM ./prog
```

| Clause | Meaning |
|--------|---------|
| `:error=ERRNO` | Return `-1` with that errno |
| `:retval=VALUE` | Return a specific value |
| `:signal=SIG` | Deliver a signal |
| `:syscall=SYSCALL` | Inject a different syscall |
| `:delay_enter=DELAY` | Sleep before the call |
| `:delay_exit=DELAY` | Sleep after the call |
| `:poke_enter=@argN=HEXDATA` | Write hex bytes to memory pointed to by an argument on entry |
| `:poke_exit=@argN=HEXDATA` | Write hex bytes on exit |
| `:when=WHEN` | Apply on selected occurrences: `FIRST`, `FIRST..LAST`, `FIRST+`, `FIRST+STEP` |

`when=..3` and `when=3..` are rejected; use `1..3` or `3+`. `-e fault=` is a
synonym for `-e inject=` with `ENOSYS` as the default error.

## Startup and attach

| Option | Meaning |
|--------|---------|
| `-p PID` | Attach; may be repeated |
| `-E VAR=VAL` | Set an environment variable for the command |
| `-E VAR` | Remove a variable |
| `-u USER` | Run as a user (`UID:GID` avoids name lookups) |
| `--argv0=NAME` | Override `argv[0]` |
| `-b execve` | Detach on `execve` |
| `-D` | Run the tracer as a grandchild |
| `-DD` | Separate process group |
| `-DDD` | Separate session |
| `-I N` | Signal-blocking mode: `1` anywhere, `2` waiting (default), `3` never, `4` never including `SIGTSTP` |
| `--kill-on-exit` | Kill tracees when strace is killed |
| `--syscall-limit=N` | Detach after N syscalls |
| `--seccomp-bpf` | Use seccomp-bpf filtering (requires `-f` and a `-e trace` filter) |

## Exit status

| Situation | Exit |
|-----------|------|
| Traced program exits 0 | 0 |
| Traced program exits non-zero | that status |
| Traced program killed by signal | 128 + signal |
| `-p` attach failure | 1 |
| Invalid `-e` expression | 1 |
| Invalid `-X` value | 1 |
| Invalid `-U` column | 1 |
| `--seccomp-bpf` without `-f` | warning, then runs without it, exit 0 |

strace propagates the child's exit status, so it is safe in a pipeline that
checks `$?`.

## Recipes

**1. Find out why a program cannot start.**

```bash
strace -f -e trace=%file,execve ./prog 2>&1 | tail -40
```

**2. See which configuration file is actually opened.**

```bash
strace -e trace=openat,open -y ./prog 2>&1 | grep -i conf
```

**3. Follow a daemon's children into separate files.**

```bash
strace -ff -o /tmp/trace ./daemon
ls /tmp/trace.*
```

**4. Profile syscall time.**

```bash
strace -c -S total-time ./prog
```

**5. Show only failed calls.**

```bash
strace -Z -e trace=%file ./prog
```

**6. Show only calls touching one path.**

```bash
strace -P /etc/ld.so.cache ./prog
```

**7. Make a file open fail, to test error handling.**

```bash
strace -e inject=openat:error=EACCES:when=1 ./prog
```

**8. Add latency to one syscall.**

```bash
strace -e inject=read:delay_enter=250ms ./prog
```

**9. Trace a running process.**

```bash
sudo strace -p "$(pgrep -n prog)" -f -tt -T
```

**10. Capture the trace without mixing it with program output.**

```bash
strace -o /tmp/trace.log -f ./prog > /tmp/stdout.log 2> /tmp/stderr.log
```

**11. Show stack traces for each syscall.**

```bash
strace -k -e trace=openat ./prog
```

**12. Decode descriptor paths and socket details.**

```bash
strace -yy -e trace=network ./prog
```

## Quirks and limits

- **`-e signal=` does not restrict syscalls.** Use `-e trace=none` as well for
  a signal-only view.
- **Multiple `-e trace=` filters are unioned.** A negation followed by a group
  re-adds the excluded call.
- **`-X abbreviated` is invalid.** Use `-X abbrev`.
- **`-U` with an unknown column exits 1** and prints `unknown column name`.
- **`--seccomp-bpf` needs `-f` and a filter.** Without them it warns
  (`cannot be used without -f/--follow-forks, disabling`) and continues.
- **`--syscall-limit` detaches, it does not kill.** The tracee keeps running.
- **`-p` needs ptrace permission.** `ptrace_scope=1` restricts attaching to
  descendants; `sudo` or a `prctl(PR_SET_PTRACER, …)` is required otherwise.
- **`-f` changes the process tree.** A traced multithreaded program can run
  slower and schedule differently, which can hide races.
- **`-k` needs libunwind.** The Debian build has it; a build without it
  rejects `-k`.
- **`-y` only annotates descriptors that are open.** A descriptor closed before
  the print has no path.
- **`-s` truncates strings.** The default 32 bytes is often too short for
  paths; use `-s 256` or `-v`.
- **`-c` and `-C` do not show argument values.** Use them for cost, not for
  content.
- **Timestamps are not a security boundary.** They come from the tracer's
  clock and are not forensic-grade.
- **`-e inject=` changes behaviour.** Do not use it on a production process
  unless the fault is the point.
- **`--tips` output is decorative.** It prints on exit and can pollute a log;
  use `--tips=none` or omit it in scripts.
- **`strace` is not a sandbox.** A traced program still has all its normal
  privileges.

## Related tools

- [`gef`](../gef/) — GDB enhancement with syscall and memory views.
- [`pwndbg`](../pwndbg/) — GDB enhancement with syscall context.
- [`pwntools`](../pwntools/) — process control and ELF inspection for exploit work.
- [`checksec`](../checksec/) — mitigation posture of the binary being traced.
- [`patchelf`](../patchelf/) — change the interpreter or RPATH that `strace`
  shows the loader using.
- [`tcpdump`](../tcpdump/) — network capture when the issue is on the wire.
- [`binwalk`](../binwalk/) — inspect firmware that contains the traced binary.
- `perf trace`, `bpftrace`, `auditd` — alternative observation layers.
