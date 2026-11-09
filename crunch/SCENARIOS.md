# crunch — command scenarios for deep research

Installed build: crunch 3.6 (`/usr/bin/crunch`). Flags come from `man crunch`;
`help.txt` is only the short banner.

## How to invoke

```text
@deep-research For crunch 3.6, how should a keyspace be estimated and bounded
before generation, which mask and duplicate-limit combinations match common
password policies without exploding in size, and how do chunked output,
compression, and resume behave across filesystems and failures? Cite the crunch
man page, hashcat rule/mask documentation, and field reports; return a sizing
worksheet and a safe generation workflow.
```

---

## Scenario 1 — keyspace sizing before generation

| # | Command | How to use |
|---|---------|------------|
| 1 | `crunch 1 4 abc \| wc -l` | Exact count for a tiny set |
| 2 | `python3 -c 'print(sum(62**n for n in range(1,9)))'` | Formula check |
| 3 | `df -h .` | Free space before writing |
| 4 | `timeout 4 crunch 8 8 0123456789 \| head -1` | Read the estimate, then abort |
| 5 | `crunch 2 2 12 --stdout` | Smoke test |

**Research questions**
- How does the tool's estimate compare with the sum-of-powers calculation?
- At what candidate count does disk, not CPU, become the constraint?

**Sources:** man page, `df`, measured output.
**Output:** a written keyspace and storage budget.

---

## Scenario 2 — masks and literals

| # | Command | How to use |
|---|---------|------------|
| 6 | `crunch 4 4 -t @@%% -o mask4.txt` | Two letters, two digits |
| 7 | `crunch 7 7 -t 'p@ss,%^' -l 'a@aaaaa'` | Keep `@` literal |
| 8 | `crunch 8 8 -t @@dog@@@ -s cbdogaaa \| head` | Fixed word inside mask |
| 9 | `crunch 5 5 -d 2@ -t @@@%% \| head` | Limit repeated letters |
| 10 | `crunch 6 6 -t 'admin@' \| wc -l` | One-variable suffix |

**Research questions**
- Which positions in a real password policy are fixed, and which vary?
- How much does `-d` reduce the space for repeated-character policies?

**Sources:** policy documents, mask output counts, man page examples.
**Output:** a policy-faithful mask and its exact size.

---

## Scenario 3 — charsets and permutations

| # | Command | How to use |
|---|---------|------------|
| 11 | `crunch 4 4 -f /usr/share/crunch/charset.lst mixalpha-numeric \| wc -l` | Named charset |
| 12 | `crunch 4 5 -p abc` | Word permutations |
| 13 | `crunch 4 5 -p dog cat bird` | Multi-word permutations |
| 14 | `crunch 2 2 -q seeds.txt` | File-based permutations |
| 15 | `crunch 3 3 abc + 123 '!@#' -t '@%^' \| head` | Per-position charsets |

**Research questions**
- When is a permutation smaller and more realistic than exhaustive generation?
- How are per-position charsets ordered, and what does `+` substitute?

**Sources:** `charset.lst`, man page examples 11–15, output.
**Output:** a candidate set matched to the known structure.

---

## Scenario 4 — chunking, compression, and resume

| # | Command | How to use |
|---|---------|------------|
| 16 | `crunch 1 5 -o START -c 6000 -z gzip` | Line-bounded compressed chunks |
| 17 | `crunch 4 5 -b 20mib -o START` | Size-bounded chunks |
| 18 | `crunch 8 8 0123456789 -o digits8.txt` | Start a resumable run |
| 19 | `crunch 8 8 0123456789 -r -o digits8.txt` | Resume after interruption |
| 20 | `ls -lh digits8.txt *.gz 2>/dev/null` | Inspect artefacts |

**Research questions**
- How are chunk boundaries named, and can they be recombined unambiguously?
- What is lost or duplicated when resuming with `-r`?

**Sources:** output filenames, checksums, man page notes.
**Output:** a chunked, resumable generation plan.

---

## Scenario 5 — validation and cracking handoff

| # | Command | How to use |
|---|---------|------------|
| 21 | `head -5 mask4.txt && wc -l mask4.txt` | Spot-check and count |
| 22 | `sha256sum mask4.txt` | Hash the wordlist |
| 23 | `crunch 8 8 0123456789 --stdout \| hashcat -m 0 -a 0 hashes.txt` | Hashcat stream |
| 24 | `crunch 2 4 abcdefghijklmnopqrstuvwxyz \| aircrack-ng capture.cap -e LABSSID -w -` | Lab handshake |
| 25 | `grep -cE '^[a-z]{2}[0-9]{2}$' mask4.txt` | Validate mask shape |

**Research questions**
- Does the downstream cracker consume stdin correctly, or does it need a file?
- What evidence proves the wordlist matches the intended mask?

**Sources:** hashcat docs, aircrack-ng docs, wordlist checks.
**Output:** a validated wordlist and a reproducible cracking handoff.

## Flag coverage checklist

| Flag group | Flags | Scenario |
|------------|-------|----------|
| Length/charset | positional `min max charset` | 1, 3 |
| Output | `-o -b -c -z -u --stdout` | 4, 5 |
| Masks | `-t -l -d -s -e -i` | 2, 4 |
| Charset file | `-f` | 3 |
| Permutations | `-p -q -m` | 3 |
| Resume | `-r` | 4 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — sizing | 5 |
| 2 — masks | 5 |
| 3 — charsets/permutations | 5 |
| 4 — chunk/resume | 5 |
| 5 — validation/handoff | 5 |
