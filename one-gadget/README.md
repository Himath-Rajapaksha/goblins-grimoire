# one_gadget — single-address glibc shell gadgets

- **Version:** current Ruby gem build (`one_gadget --version`)
- **Binary:** `/usr/local/bin/one_gadget`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/david942j/one_gadget

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

one_gadget scans a glibc shared object for addresses that, when jumped to
directly, execute `execve("/bin/sh", ...)`. Each candidate carries the register
and stack constraints required for it to work — for example `rsp+0x40 == NULL`
or `[rsp+0x30] == NULL`. It can order gadgets by distance to a symbol or GOT
entry, emit raw or JSON output, run an exploit script against every candidate,
and look up a libc by BuildID.

## Use cases

| Situation | Why one_gadget |
|-----------|----------------|
| Fast shell in a CTF/lab exploit | Skip a full ROP chain |
| Constraint discovery | See exactly which registers must be NULL |
| Libc-specific lookup | BuildID lookup without a local file |
| Chain shortcuts | Gadget near a known function/GOT entry |
| Automation | JSON output for exploit scripts |
| Triage | Count and rank candidates per libc build |

## Requirements & install layout

- Installed as a Ruby gem; launcher `/usr/local/bin/one_gadget`.
- No root, no network for local files.
- `--build-id` performs an online lookup; keep `--force-file` for offline work.
- Works on glibc `libc.so.6` and some other libc builds.

## Basic syntax

```bash
# list gadgets for a libc
one_gadget /lib/x86_64-linux-gnu/libc.so.6

# raw offsets only
one_gadget -r /lib/x86_64-linux-gnu/libc.so.6

# JSON for automation
one_gadget -o json /lib/x86_64-linux-gnu/libc.so.6

# order by distance to system
one_gadget -n system /lib/x86_64-linux-gnu/libc.so.6

# include lower-probability gadgets
one_gadget -l 1 /lib/x86_64-linux-gnu/libc.so.6

# apply a base address
one_gadget --base 0x7ffff7c00000 /lib/x86_64-linux-gnu/libc.so.6
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-b, --build-id <id>` | Look up a libc by BuildID |
| `-f, --force-file` | Search the file even when a BuildID is present |
| `-l, --level <n>` | Output level; higher = more (less reliable) gadgets (default 0) |
| `-n, --near <func\|file>` | Order gadgets by distance to a function or GOT entry |
| `-o, --output-format <fmt>` | `pretty`, `raw`, or `json` (default `pretty`) |
| `-r, --raw` | Alias for raw output, offsets separated by spaces |
| `-s, --script <file>` | Run a script once per gadget as `script <offset>` |
| `--info <build-id>` | Show version information for a BuildID |
| `--base <addr>` | Base address added to offsets (default 0) |
| `--version` | Gem version |
| `-h, --help` | Help |

## Recipes

```bash
# 1. Standard listing
one_gadget /lib/x86_64-linux-gnu/libc.so.6

# 2. Machine-readable
one_gadget -o json /lib/x86_64-linux-gnu/libc.so.6 > gadgets.json

# 3. Raw offsets for scripts
one_gadget -r /lib/x86_64-linux-gnu/libc.so.6

# 4. Rank by proximity to system
one_gadget -n system /lib/x86_64-linux-gnu/libc.so.6

# 5. More candidates
one_gadget -l 1 /lib/x86_64-linux-gnu/libc.so.6

# 6. Absolute addresses for a known base
one_gadget --base 0x7ffff7c00000 /lib/x86_64-linux-gnu/libc.so.6

# 7. Try every candidate with a script
one_gadget -s ./try_gadget.sh /lib/x86_64-linux-gnu/libc.so.6

# 8. BuildID lookup (network)
one_gadget -b 1f8d6e... 

# 9. Force local file with BuildID present
one_gadget -f /path/to/libc.so.6
```

## Output & pipelines

- Pretty output: offset, instructions, and constraints.
- `-o json` for exploit automation; `-r` for quick scripting.
- Constraints must be satisfied before use; a gadget that violates them crashes.
- Feed offsets into pwntools (`libc.address + offset`) or a manual exploit.

## Gotchas & operational security

- Gadgets are **highly** build-specific; never reuse an offset across libc
  versions.
- Constraints are mandatory. Check `rsp+N`, register, and memory conditions.
- Higher `-l` levels include gadgets with lower success probability.
- BuildID lookup sends the ID to an online service; prefer local files in
  sensitive environments.
- The gadget may still `execve` with unexpected argv/envp; verify in a lab.
- ASLR/PIE means the base must be leaked first.
- JSON schema changes across gem versions; pin the gem when automating.

## Legal & safety

Use only on systems and binaries you are authorized to test, in an isolated
lab. A one-gadget jump is still code execution; never aim it at production or
third-party systems.

## Related tools

- [`ropper/`](../ropper/README.md), [`ropgadget/`](../ropgadget/README.md) —
  full gadget search
- [`pwndbg/`](../pwndbg/README.md), [`gef/`](../gef/README.md) — verify
  constraints at runtime
- [`pwntools/`](../pwntools/README.md) — exploit assembly

## References

- https://github.com/david942j/one_gadget
- https://rubygems.org/gems/one_gadget
