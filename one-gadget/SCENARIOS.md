# one_gadget — command scenarios for deep research

Installed build: one_gadget Ruby gem. Flags come from `help.txt`.

## How to invoke

```text
@deep-research For one_gadget against current glibc 2.36+ on Debian 13, which
constraint patterns (rsp offsets, register NULLs, environ) most often block the
standard gadgets, how do gadget counts and reliability differ across libc
builds, and what verification step should precede use in an exploit? Cite the
one_gadget repo, glibc source, and exploit-dev writeups; return a constraint
checklist and a verification script.
```

---

## Scenario 1 — listing and formats

| # | Command | How to use |
|---|---------|------------|
| 1 | `one_gadget /lib/x86_64-linux-gnu/libc.so.6` | Pretty listing |
| 2 | `one_gadget -r /lib/x86_64-linux-gnu/libc.so.6` | Raw offsets |
| 3 | `one_gadget -o json /lib/x86_64-linux-gnu/libc.so.6` | JSON |
| 4 | `one_gadget --version` | Gem version |
| 5 | `one_gadget -l 1 /lib/x86_64-linux-gnu/libc.so.6` | More candidates |

**Research questions**
- How many gadgets does level 0 versus level 1 return for this libc?
- Which output format preserves constraint text for automation?

**Sources:** output, gem docs, source.
**Output:** a candidate list in the chosen format.

---

## Scenario 2 — constraints

| # | Command | How to use |
|---|---------|------------|
| 6 | `one_gadget libc.so.6 \| grep -A3 'constraints'` | Show constraints |
| 7 | `one_gadget -o json libc.so.6 \| jq '.[].constraints'` | Machine-readable |
| 8 | `one_gadget -n system libc.so.6` | Rank by proximity |
| 9 | `one_gadget --base 0x7ffff7c00000 libc.so.6` | Absolute addresses |
| 10 | `one_gadget -f libc.so.6` | Force local file |

**Research questions**
- Which constraints can be satisfied by the overflow layout, and which cannot?
- Does `-n` change reliability or only ordering?

**Sources:** glibc source, exploit layout, output.
**Output:** a shortlist of satisfiable gadgets.

---

## Scenario 3 — BuildID lookups

| # | Command | How to use |
|---|---------|------------|
| 11 | `file libc.so.6` | Identify build |
| 12 | `readelf -n libc.so.6 \| grep 'Build ID'` | Extract BuildID |
| 13 | `one_gadget -b <build-id>` | Online lookup |
| 14 | `one_gadget --info <build-id>` | Build information |
| 15 | `one_gadget -f libc.so.6` | Offline fallback |

**Research questions**
- When is a BuildID lookup appropriate, and what data leaves the host?
- How are libc builds without public IDs handled?

**Sources:** `readelf`, gem docs, output.
**Output:** gadget offsets for an unknown local libc.

---

## Scenario 4 — automated testing

| # | Command | How to use |
|---|---------|------------|
| 16 | `cat try_gadget.sh` | Inspect test script |
| 17 | `one_gadget -s ./try_gadget.sh libc.so.6` | Try each gadget |
| 18 | `./try_gadget.sh 0xebc81` | Single offset |
| 19 | `one_gadget -r libc.so.6 > offsets.txt` | Offsets for a loop |
| 20 | `for o in $(cat offsets.txt); do ./try_gadget.sh $o; done` | Manual loop |

**Research questions**
- What contract does `-s` use (`script <offset>`), and how are failures
  reported?
- Which gadgets crash silently versus fail cleanly?

**Sources:** help, script output, debugger.
**Output:** the most reliable gadget for the target.

---

## Scenario 5 — verification and reporting

| # | Command | How to use |
|---|---------|------------|
| 21 | `gdb -q ./vuln -ex 'b *$base+0xebc81' -ex run` | Break at gadget |
| 22 | `x/8i $pc` | Confirm instructions |
| 23 | `p/x $rsp` / `x/4gx $rsp` | Check constraints |
| 24 | `sha256sum libc.so.6 offsets.txt` | Hash evidence |
| 25 | `grep -c '^0x' offsets.txt` | Count gadgets |

**Research questions**
- How can constraints be checked without executing the gadget?
- What provenance is needed for a libc-specific offset?

**Sources:** GDB, glibc source, hashes.
**Output:** a verified gadget and documented evidence.

## Flag coverage checklist

| Flag | Scenario |
|------|----------|
| `-b --info` | 3 |
| `-f` | 2, 3 |
| `-l` | 1 |
| `-n` | 2 |
| `-o -r` | 1, 4 |
| `-s` | 4 |
| `--base` | 2 |
| `--version` | 1 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — listing | 5 |
| 2 — constraints | 5 |
| 3 — BuildID | 5 |
| 4 — testing | 5 |
| 5 — verification | 5 |
