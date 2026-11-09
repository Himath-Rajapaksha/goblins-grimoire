# hashcat — command scenarios for deep research

Installed build: hashcat v6.2.6. Flags and mode tables come from
[`help.txt`](help.txt). This host exposes one **CPU** OpenCL device through
PoCL (`hashcat -I`), so speed expectations differ from a GPU rig.

> **Sensitive:** cracking recovers credentials. Use only on hashes you own or
> are explicitly authorized to test, and protect the potfile.

## How to invoke

```text
@deep-research Given a hash type and a CPU-only OpenCL backend, which attack
mode, rule set, and mask strategy maximize recovery per hour without
overrunning the keyspace, and how should the choice change for NTLM versus
bcrypt versus WPA? Cite the hashcat wiki, mode-specific benchmark data, and
field reports; return a decision table with a benchmark-first procedure.
```

Prepare the demo directory:

```bash
mkdir -p ~/hashcat-demo/evidence
cd ~/hashcat-demo
printf '8846f7eaee8fb117ad06bdd830b7586c\n' > md5.txt   # "password"
printf 'aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c\n' > ntlm.txt
```

Use only hashes generated for the lab; do not paste production hashes into a
shared shell history.

## Scenario 1 — identify the hash and benchmark

Never choose a mode or a strategy before identifying the hash and measuring the
device. `--identify` and `-b` are the two cheapest steps.

| # | Command | How to use |
|---|---------|------------|
| 1 | `hashcat --identify ~/hashcat-demo/md5.txt` | Guess the mode |
| 2 | `hashcat --example-hashes -m 1000 \| head -40` | See the expected format |
| 3 | `hashcat -I` | List backends and devices |
| 4 | `hashcat -b -m 0` | Benchmark MD5 |
| 5 | `hashcat -b -m 1000` | Benchmark NTLM |
| 6 | `hashcat -b -m 3200` | Benchmark bcrypt (slow by design) |

**Research questions**
- Does `--identify` agree with the source system's hash format?
- What is the measured hash/s rate for each candidate mode?
- Which mode is so slow that a dictionary is the only practical attack?

**Sources:** hashcat wiki example hashes; local benchmarks; source-system docs.
**Output:** a mode/rate table and a shortlist of viable attacks.

## Scenario 2 — dictionary and rule attacks

A dictionary plus rules is the highest-yield attack for human-chosen passwords.
Start with the smallest rule set that fits the target's password policy.

| # | Command | How to use |
|---|---------|------------|
| 7 | `hashcat -a 0 -m 1000 ntlm.txt rockyou.txt` | Plain dictionary |
| 8 | `hashcat -a 0 -m 1000 ntlm.txt rockyou.txt -r best64.rule` | Light rules |
| 9 | `hashcat -a 0 -m 1000 ntlm.txt rockyou.txt -r dive.rule` | Heavier rules |
| 10 | `hashcat -a 0 -m 1000 ntlm.txt rockyou.txt -j 'c' -k '^!'` | Single left/right rules |
| 11 | `hashcat -a 1 -m 1000 ntlm.txt left.txt right.txt` | Combination |
| 12 | `hashcat -a 0 -m 1000 --stdout wordlist.txt -r best64.rule \| head` | Preview candidates |

**Research questions**
- How much does each rule set increase the candidate count?
- Which rule set matches the observed password policy?
- Does combination mode find patterns a single dictionary misses?

**Sources:** hashcat rule wiki; `--stdout` counts; policy documentation.
**Output:** a rule/candidate-count table and the best recovery rate.

## Scenario 3 — mask and hybrid attacks

Masks are for known patterns: fixed prefixes, digit suffixes, dates, and policy
shapes. They are exponential, so bound them with `--increment` and a keyspace
check.

| # | Command | How to use |
|---|---------|------------|
| 13 | `hashcat -a 3 -m 0 md5.txt ?d?d?d?d?d?d` | Six digits |
| 14 | `hashcat -a 3 -m 0 md5.txt ?d?d?d?d?d?d?d?d --increment --increment-min 4` | Digits, growing length |
| 15 | `hashcat -a 3 -m 1000 ntlm.txt -1 ?l?d ?1?1?1?1?1?1?1?1` | Lowercase+digits, 8 chars |
| 16 | `hashcat -a 6 -m 1000 ntlm.txt rockyou.txt ?d?d?d?d` | Word + 4 digits |
| 17 | `hashcat -a 7 -m 1000 ntlm.txt ?d?d?d?d rockyou.txt` | 4 digits + word |
| 18 | `hashcat -a 3 -m 0 md5.txt ?a?a?a?a?a?a --keyspace` | Keyspace estimate |

**Research questions**
- What is the keyspace for each mask, and how long would it take at the
  measured rate?
- Does `--increment` find short passwords before the full mask?
- Which hybrid direction matches the observed pattern?

**Sources:** hashcat mask wiki; `--keyspace`; benchmark rates.
**Output:** a keyspace/time table and the chosen mask.

## Scenario 4 — protocol and application hashes

The same engine handles network captures and application formats; only the mode
and input format change. Convert captures with the matching toolkit first.

| # | Command | How to use |
|---|---------|------------|
| 19 | `hashcat -a 0 -m 5600 netntlmv2.txt rockyou.txt -r best64.rule` | NetNTLMv2 |
| 20 | `hashcat -a 0 -m 13100 kerberoast.txt rockyou.txt -r best64.rule` | Kerberoast TGS-REP |
| 21 | `hashcat -a 0 -m 18200 asrep.txt rockyou.txt` | AS-REP |
| 22 | `hashcat -a 0 -m 22000 capture.hc22000 rockyou.txt` | WPA PMKID/EAPOL |
| 23 | `hashcat -a 0 -m 1800 shadow.txt rockyou.txt` | sha512crypt |
| 24 | `hashcat -a 0 -m 3200 bcrypt.txt rockyou.txt` | bcrypt (slow) |

**Research questions**
- Which capture format does each mode expect, and how is it produced?
- How does the per-hash cost change the viable dictionary size?
- Are the Kerberos modes using the right etype for the domain?

**Sources:** hashcat example hashes; hcxtools docs; Kerberos RFC 4120.
**Output:** a per-source mode map and a recovery result.

## Scenario 5 — sessions, potfile hygiene, and resumption

Long runs fail. Sessions, restore files, and a deliberate potfile policy make
the work reproducible and prevent a previous crack from hiding a new test.

| # | Command | How to use |
|---|---------|------------|
| 25 | `hashcat -a 0 -m 1000 ntlm.txt rockyou.txt --session acme --status --status-timer 30` | Named, monitored session |
| 26 | `hashcat --session acme --restore` | Resume after interruption |
| 27 | `hashcat -a 0 -m 1000 ntlm.txt --show` | Show potfile cracks |
| 28 | `hashcat -a 0 -m 1000 ntlm.txt --left rockyou.txt` | Attack only uncracked |
| 29 | `hashcat -a 0 -m 1000 ntlm.txt rockyou.txt --potfile-path evidence/acme.pot` | Separate potfile |
| 30 | `hashcat -a 0 -m 1000 ntlm.txt rockyou.txt --potfile-disable` | Clean-room test |

**Research questions**
- Does the restore file reproduce the exact position after an abort?
- Does `--left` reduce the candidate set correctly?
- Which potfile policy keeps engagements isolated?

**Sources:** hashcat session wiki; `--show` output; restore file inspection.
**Output:** a session log and a potfile-handling note.

## Scenario 6 — evidence and reporting

Record the mode, attack, rules, rate, and result. A crack without the command
and hash source is not reproducible.

| # | Command | How to use |
|---|---------|------------|
| 31 | `hashcat -a 0 -m 1000 ntlm.txt rockyou.txt -r best64.rule -o evidence/cracked.txt --outfile-format 2` | Plain output |
| 32 | `hashcat -a 0 -m 1000 ntlm.txt --show --outfile-format 1,2 > evidence/show.txt` | Hash + plain |
| 33 | `hashcat -b -m 1000 --machine-readable > evidence/bench.txt` | Machine-readable benchmark |
| 34 | `hashcat -a 0 -m 1000 ntlm.txt rockyou.txt --runtime 600 --status-json > evidence/status.ndjson` | Bounded run log |
| 35 | `sha256sum evidence/*.txt` | Evidence integrity |

**Research questions**
- Does the evidence include the exact hash input and mode?
- Is the reported rate consistent with the benchmark?
- Can a second tester reproduce the crack from the saved files?

**Sources:** hashcat output options; report requirements; hash of evidence.
**Output:** an evidence bundle with hash, command, cracked plains, and benchmark.
