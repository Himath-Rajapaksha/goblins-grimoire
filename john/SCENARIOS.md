# John the Ripper — command scenarios for deep research

Installed build: John the Ripper 1.9.0 (core, Debian package `john`). Every
flag below appears in [`help.txt`](help.txt). The binary lives in
`/usr/sbin/john`, which is not on a normal user's `PATH`; the examples call it
by full path. `unshadow`, `unafs`, and `unique` are separate helpers.

> **Sensitive:** password cracking processes real credential material. Use only
> on hashes you are authorized to test, keep derived files encrypted, and
> delete them at engagement close.

## How to invoke

```text
@deep-research For a Linux host whose /etc/shadow I am authorized to audit,
compare John the Ripper 1.9.0 core with hashcat for md5crypt, bcrypt, and LM
hashes. Which modes and rule sets give the best chance per CPU-hour, what do
the shipped john.conf incremental modes actually cover, and where does core
John stop and jumbo/hashcat begin? Cite the Openwall documentation, the
installed john.conf, and independent benchmarks; return an ordered cracking
plan with stop conditions.
```

Create the demo workspace and inputs, replacing them with in-scope material:

```bash
mkdir -p ~/john-demo/evidence
cp /etc/passwd /etc/shadow ~/john-demo/
printf 'acme\nsummer\nwinter\n' > ~/john-demo/words.txt
```

## Scenario 1 — normalize the input

John wants a passwd-format file. `unshadow` merges the passwd and shadow
entries; `unique` removes duplicate lines. Record the hash format before
spending CPU on the wrong mode.

| # | Command | How to use |
|---|---------|------------|
| 1 | `unshadow ~/john-demo/passwd ~/john-demo/shadow > ~/john-demo/combined.txt` | Merge passwd and shadow |
| 2 | `unique ~/john-demo/unique.txt < ~/john-demo/combined.txt` | Deduplicate entries |
| 3 | `cut -d: -f2 ~/john-demo/combined.txt \| grep -v '^[!*]' \| head` | Inspect non-empty hash prefixes |
| 4 | `wc -l ~/john-demo/combined.txt ~/john-demo/unique.txt` | Count accounts and unique entries |
| 5 | `/usr/sbin/john --test=2` | Confirm which formats this build benchmarks |

**Research questions**
- Which hash prefixes (`$1$`, `$2a$`, `$5$`, `$6$`, `$y$`) does this core build
  actually support, and which require hashcat?
- Does `unique` preserve the first or last duplicate, and why does that matter
  for `--show`?
- Which accounts are locked (`!`, `*`) and should be excluded before cracking?

**Sources:** `man john`, `man unshadow`, `man unique`, `/etc/john/john.conf`,
Openwall John documentation.
**Output:** `combined.txt`, `unique.txt`, and a format inventory table.

## Scenario 2 — single mode and wordlist with rules

Single mode derives candidates from account metadata and is the cheapest first
pass. Wordlist mode with `--rules` applies the shipped mangling rules.

| # | Command | How to use |
|---|---------|------------|
| 6 | `/usr/sbin/john --single ~/john-demo/combined.txt` | Metadata-derived candidates |
| 7 | `/usr/sbin/john --wordlist=/usr/share/john/password.lst ~/john-demo/combined.txt` | Shipped wordlist, no mangling |
| 8 | `/usr/sbin/john --wordlist=/usr/share/john/password.lst --rules ~/john-demo/combined.txt` | Shipped wordlist plus rules |
| 9 | `/usr/sbin/john --wordlist=~/john-demo/words.txt --rules ~/john-demo/combined.txt` | Engagement-specific wordlist |
| 10 | `/usr/sbin/john --wordlist=~/john-demo/words.txt --rules --format=md5crypt ~/john-demo/combined.txt` | Force one format |
| 11 | `/usr/sbin/john --stdin --rules ~/john-demo/combined.txt < ~/john-demo/words.txt` | Candidates from stdin |

**Research questions**
- Which fields does `--single` use, and what happens when GECOS is empty?
- How many candidates per word does the default `[List.Rules:Wordlist]` set
  generate?
- When does forcing `--format` help, and when does it hide accounts?
- Why is `--stdin` slower or faster than `--wordlist` for the same file?

**Sources:** `/etc/john/john.conf` (`[List.Rules:Single]`,
`[List.Rules:Wordlist]`), Openwall rules documentation.
**Output:** a per-mode hit count and a list of cracked accounts with the mode
that recovered each one.

## Scenario 3 — incremental and candidate generation

Incremental mode walks a charset. The shipped `.chr` files and
`[Incremental:*]` sections define the keyspace. `--stdout` turns John into a
candidate generator for another cracker.

| # | Command | How to use |
|---|---------|------------|
| 12 | `/usr/sbin/john --incremental=Digits ~/john-demo/combined.txt` | Numeric keyspace |
| 13 | `/usr/sbin/john --incremental=Alnum ~/john-demo/combined.txt` | Alphanumeric keyspace |
| 14 | `/usr/sbin/john --incremental=Lower ~/john-demo/combined.txt` | Lowercase keyspace |
| 15 | `/usr/sbin/john --incremental=ASCII ~/john-demo/combined.txt` | Full ASCII keyspace (large) |
| 16 | `/usr/sbin/john --wordlist=~/john-demo/words.txt --rules --stdout > ~/john-demo/evidence/candidates.txt` | Emit mangled candidates |
| 17 | `/usr/sbin/john --wordlist=~/john-demo/words.txt --rules --stdout=8 > ~/john-demo/evidence/candidates8.txt` | Cap candidate length at 8 |
| 18 | `/usr/sbin/john --make-charset=~/john-demo/evidence/custom.chr` | Build a custom charset file |

**Research questions**
- What keyspace does each shipped incremental mode cover, and how long is it
  on this CPU?
- When is `--stdout` preferable to cracking in John?
- What does `--make-charset` sample, and how do you use the result?
- Which mode is a reasonable stop condition before switching to hashcat?

**Sources:** `/usr/share/john/*.chr`, `/etc/john/john.conf`
(`[Incremental:*]`), Openwall documentation.
**Output:** keyspace estimates, candidate files, and a stop-condition table.

## Scenario 4 — sessions, parallelism, and resumption

Long runs must survive interruption. Name the session, check status, restore,
and split work with `--fork` or `--node`.

| # | Command | How to use |
|---|---------|------------|
| 19 | `/usr/sbin/john --session=acme --wordlist=~/john-demo/words.txt ~/john-demo/combined.txt` | Named session |
| 20 | `/usr/sbin/john --status=acme` | Progress of a running session |
| 21 | `/usr/sbin/john --restore=acme` | Resume after interruption |
| 22 | `/usr/sbin/john --fork=4 --wordlist=~/john-demo/words.txt ~/john-demo/combined.txt` | Four local processes |
| 23 | `/usr/sbin/john --node=1-4/16 --wordlist=~/john-demo/words.txt ~/john-demo/combined.txt` | Node 1 of 16 |
| 24 | `/usr/sbin/john --save-memory=2 --incremental=Alnum ~/john-demo/combined.txt` | Reduce memory use |
| 25 | `/usr/sbin/john --users=alice,bob ~/john-demo/combined.txt` | Restrict to named users |
| 26 | `/usr/sbin/john --groups=1000 ~/john-demo/combined.txt` | Restrict to one GID |
| 27 | `/usr/sbin/john --shells=/bin/bash ~/john-demo/combined.txt` | Restrict to one shell |
| 28 | `/usr/sbin/john --salts=2 ~/john-demo/combined.txt` | Only salts with at least two hashes |

**Research questions**
- Where does John store session state, and what must be preserved to restore?
- Does `--fork` share the same `john.pot`, and what happens on a race?
- How do `--node` and `--fork` compose across a cluster?
- When do `--users`, `--groups`, and `--salts` reduce work without hiding a
  target account?

**Sources:** `man john`, Openwall session documentation, `/etc/john/john.conf`.
**Output:** a resumable session, a status log, and a parallel-run worksheet.

## Scenario 5 — verify, report, and clean up

`--show` reads `john.pot` and prints recovered passwords for the loaded file.
Report only the accounts in scope, then remove the working material.

| # | Command | How to use |
|---|---------|------------|
| 29 | `/usr/sbin/john --show ~/john-demo/combined.txt` | Print cracked credentials |
| 30 | `/usr/sbin/john --show --format=md5crypt ~/john-demo/combined.txt` | Same, one format |
| 31 | `/usr/sbin/john --test=5` | Benchmark all supported formats |
| 32 | `shred -u ~/john-demo/combined.txt ~/john-demo/shadow ~/john-demo/unique.txt` | Destroy working copies |
| 33 | `rm -f ~/john.pot` | Remove the pot file after reporting |

**Research questions**
- What exactly is stored in `john.pot`, and is it encrypted?
- How do you reconcile `--show` output with the original account list?
- Which benchmark result predicts throughput for the engagement's hash type?
- What does secure deletion guarantee on this filesystem?

**Sources:** `man john`, Openwall `john.pot` documentation, Debian package
file list.
**Output:** a redacted credential report, a benchmark table, and a deletion
record.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| `--single` | 2 |
| `--wordlist`, `--stdin` | 2 |
| `--rules` | 2, 3 |
| `--incremental[=MODE]` | 3, 4 |
| `--external` | 3 (discussion) |
| `--stdout[=LENGTH]` | 3 |
| `--restore`, `--session`, `--status` | 4 |
| `--make-charset` | 3 |
| `--show` | 5 |
| `--test[=TIME]` | 1, 5 |
| `--users`, `--groups`, `--shells`, `--salts` | 4 |
| `--save-memory` | 4 |
| `--node`, `--fork` | 4 |
| `--format` | 2, 5 |

Default wordlist: `/usr/share/john/password.lst` (`Wordlist = $JOHN/password.lst`
in `/etc/john/john.conf`). Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Normalize and classify hashes | 5 |
| 2 | Single and wordlist cracking | 6 |
| 3 | Incremental and candidate generation | 7 |
| 4 | Sessions and parallelism | 10 |
| 5 | Verify, report, clean up | 5 |
| **Total** | | **33** |

## Legal and safety

Cracking is only lawful on hashes you own or are explicitly authorized to
test. Keep `combined.txt`, `john.pot`, and candidate files encrypted, never
commit them, and record deletion. A recovered password is a credential: store
it in the engagement's secret manager, not in the report body.
