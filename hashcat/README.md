# hashcat — GPU/CPU password hash cracker

- **Version:** hashcat v6.2.6 (`hashcat --version`)
- **Binary:** `/usr/bin/hashcat` (Debian package `hashcat`)
- **Data:** `/usr/share/hashcat/` (`rules/`, `masks/`, `charsets/`, `modules/`,
  `OpenCL/`, `hashcat.hcstat2`)
- **Full help snapshot:** [`help.txt`](help.txt) (includes every hash mode)
- **Upstream:** https://github.com/hashcat/hashcat
- **Wiki:** https://hashcat.net/wiki/
- **Hash-mode reference:** https://hashcat.net/wiki/doku.php?id=example_hashes

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

hashcat recovers plaintext from password hashes by generating candidate
passwords and testing them against the target hash. It supports hundreds of
hash modes (raw hashes, salted/iterated hashes, network captures, disk
encryption, archives, and application formats) and several attack modes
(dictionary, combination, mask/brute-force, hybrid, association). It runs on
GPUs through CUDA/OpenCL/HIP/Metal and on CPUs through OpenCL.

The installed build on this host reports a single **CPU** OpenCL device through
PoCL (`hashcat -I`), so benchmarks and speeds here reflect CPU cracking, not a
GPU rig. That changes strategy: short dictionaries and targeted masks are
practical; full 8-character `?a` keyspaces are not.

## Use cases

| Situation | Why hashcat |
|-----------|-------------|
| Recovered NTLM/LM hashes | Mode `1000` / `3000` with a dictionary and rules |
| NetNTLMv2 captures | Mode `5600` after `Responder`/`ntlmrelayx` |
| Kerberoast/AS-REP | Modes `13100` / `18200` |
| Unix password files | `500` (md5crypt), `1800` (sha512crypt), `3200` (bcrypt) |
| Database dumps | `300` (MySQL), `1731` (MSSQL), `12` (PostgreSQL) |
| WPA handshakes | `22000` (PMKID + EAPOL) |
| Archive/document hashes | ZIP, RAR, 7-Zip, Office, PDF modes from the help table |
| Rule development | `--stdout` shows candidates without cracking |
| Hash identification | `--identify` names the likely mode |

## Requirements & install layout

- Debian package `hashcat`; binary `/usr/bin/hashcat`. Version 6.2.6.
- No root needed for normal cracking; GPU drivers and permissions matter for
  GPU backends.
- OpenCL runtime required. This host uses PoCL CPU-only:
  `Platform #1: The pocl project`, `Device #1: CPU ... Intel Core i3-1005G1`.
- Rules, masks, and charsets ship under `/usr/share/hashcat/`.
- Potfile and session files are written to the current directory by default
  (`hashcat.potfile`, `<session>.restore`); `--potfile-path` and
  `--restore-file-path` relocate them.
- `-O` enables optimized kernels but caps password length (commonly 31
  characters); leave it off when long candidates matter.

## Basic syntax

```bash
hashcat [options]... hash|hashfile|hccapxfile [dictionary|mask|directory]...

hashcat -a 0 -m 1000 ntlm.txt rockyou.txt
hashcat -a 0 -m 0 md5.txt rockyou.txt -r rules/best64.rule
hashcat -a 3 -m 0 md5.txt ?a?a?a?a?a?a
hashcat -a 1 -m 0 md5.txt left.txt right.txt
```

`-m` selects the hash mode; `-a` selects the attack mode. If `-m` is omitted,
hashcat tries to autodetect, which is convenient but can guess wrong on short
or ambiguous hashes.

## Attack modes

| # | Mode | Input | Use |
|---|------|-------|-----|
| 0 | Straight | hash + wordlist | Dictionary attack; add `-r` rules |
| 1 | Combination | hash + two wordlists | Concatenate one word from each |
| 3 | Brute-force / mask | hash + mask | Exhaustive or targeted keyspace |
| 6 | Hybrid wordlist + mask | hash + wordlist + mask | Word plus appended mask |
| 7 | Hybrid mask + wordlist | hash + mask + wordlist | Mask plus appended word |
| 9 | Association | hash + wordlist + rules | Rule-associated candidates |

## Hash modes used most in engagements

The full list is in [`help.txt`](help.txt). These are the common ones:

| Mode | Hash |
|------|------|
| `0` | MD5 |
| `100` | SHA1 |
| `1400` | SHA2-256 |
| `1700` | SHA2-512 |
| `1000` | NTLM |
| `3000` | LM |
| `5500` | NetNTLMv1 / NetNTLMv1+ESS |
| `5600` | NetNTLMv2 |
| `7500` | Kerberos 5, etype 23, AS-REQ Pre-Auth |
| `13100` | Kerberos 5, etype 23, TGS-REP (Kerberoast) |
| `18200` | Kerberos 5, etype 23, AS-REP |
| `22000` | WPA-PBKDF2-PMKID+EAPOL |
| `500` | md5crypt (Unix), Cisco IOS `$1$` |
| `1800` | sha512crypt (Unix) |
| `7400` | sha256crypt (Unix) |
| `3200` | bcrypt `$2*$` |
| `300` | MySQL4.1/MySQL5 |
| `200` | MySQL323 |
| `131` / `132` / `1731` | MSSQL 2000 / 2005 / 2012-2014 |
| `12` | PostgreSQL |
| `16500` | JWT |
| `13600` | WinZip |
| `12500` | RAR3-hp |
| `11600` | 7-Zip |

## Flag reference

### Core

| Flag | Meaning |
|------|---------|
| `-m, --hash-type <n>` | Hash mode |
| `-a, --attack-mode <n>` | Attack mode |
| `-V, --version` | Print version |
| `-h, --help` | Print help (includes mode tables) |
| `--quiet` | Suppress output |
| `--identify` | Identify the hash mode of an input file |
| `--example-hashes` | Show example hashes for a mode |
| `--hash-info` | Show information for each hash mode |
| `-b, --benchmark` | Benchmark selected modes |
| `--benchmark-all` | Benchmark every mode (with `-b`) |
| `-I, --backend-info` | Show backend/device info |

### Wordlist and rules

| Flag | Meaning |
|------|---------|
| `-r, --rules-file <file>` | Apply a rule file |
| `-j, --rule-left <rule>` | Single rule on the left wordlist |
| `-k, --rule-right <rule>` | Single rule on the right wordlist |
| `-g, --generate-rules <n>` | Generate `n` random rules |
| `--generate-rules-func-min/max <n>` | Rule function bounds |
| `--generate-rules-seed <n>` | Reproducible rule generation |
| `-c, --segment-size <MB>` | Wordlist cache size |
| `-s, --skip <n>` / `-l, --limit <n>` | Skip/limit candidates |
| `--keyspace` | Print keyspace base:mod values |
| `-S, --slow-candidates` | Advanced candidate generators |

### Masks and charsets

| Flag | Meaning |
|------|---------|
| `-1 <charset>` ... `-4 <charset>` | Custom charsets `?1`...`?4` |
| `-i, --increment` | Increment mask length |
| `--increment-min <n>` / `--increment-max <n>` | Bounds for increment |
| `--hex-charset` | Charset is given in hex |
| `--hex-salt` | Salt is given in hex |
| `--hex-wordlist` | Wordlist words are hex |

Built-in charsets:

| Token | Charset |
|-------|---------|
| `?l` | `a-z` |
| `?u` | `A-Z` |
| `?d` | `0-9` |
| `?h` | `0-9a-f` |
| `?H` | `0-9A-F` |
| `?s` | special characters |
| `?a` | `?l?u?d?s` |
| `?b` | `0x00`-`0xff` |

### Output and sessions

| Flag | Meaning |
|------|---------|
| `-o, --outfile <file>` | Write recovered hashes |
| `--outfile-format <list>` | Fields: `1` hash[:salt], `2` plain, `3` hex plain, `4` crack pos, `5`/`6` timestamps |
| `--show` | Show cracked hashes from the potfile |
| `--left` | Show uncracked hashes |
| `--username` | Ignore `user:` prefixes in the hash file |
| `--remove` | Remove cracked hashes from the input file |
| `--potfile-disable` / `--potfile-path <file>` | Disable or relocate the potfile |
| `--session <name>` | Named session |
| `--restore` | Restore a session |
| `--restore-disable` | Do not write a restore file |
| `--runtime <s>` | Stop after this many seconds |
| `--status` / `--status-json` | Live status updates |
| `--status-timer <s>` | Status interval |
| `--machine-readable` | Machine-readable status |
| `--stdout` | Print candidates without cracking |
| `--keep-guessing` | Keep going after a crack |
| `--loopback` | Add recovered plains to the candidate pool |
| `--logfile-disable` | Disable the logfile |

### Performance and devices

| Flag | Meaning |
|------|---------|
| `-O, --optimized-kernel-enable` | Faster kernels; limits password length |
| `-w, --workload-profile <1-4>` | Low, default, high, nightmare |
| `-n, --kernel-accel <n>` | Outer loop step size |
| `-u, --kernel-loops <n>` | Inner loop step size |
| `-T, --kernel-threads <n>` | Thread count |
| `-d, --backend-devices <list>` | Select devices |
| `-D, --opencl-device-types <list>` | `1` CPU, `2` GPU, `3` FPGA/DSP |
| `--backend-ignore-opencl` | Skip the OpenCL backend |
| `--hwmon-disable` | Disable temperature/fan reads |
| `--hwmon-temp-abort <C>` | Abort above a temperature |
| `--spin-damp <percent>` | CPU-based device sync |
| `--force` | Ignore warnings |

### Rules and debugging

| Flag | Meaning |
|------|---------|
| `--debug-mode <1-5>` | Rule debugging format |
| `--debug-file <file>` | Debug output file |
| `--markov-disable` / `--markov-classic` | Markov-chain behavior |
| `--markov-threshold <n>` | Markov threshold |
| `--encoding-from <code>` / `--encoding-to <code>` | Wordlist encoding |
| `--brain-server` / `-z, --brain-client` | Distributed brain |

## Recipes

Authorized hashes only.

**1. NTLM with a dictionary and rules.**

```bash
hashcat -a 0 -m 1000 ntlm.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule -O -w 3
```

**2. NetNTLMv2 capture.**

```bash
hashcat -a 0 -m 5600 netntlmv2.txt rockyou.txt -r best64.rule
```

**3. Kerberoast TGS-REP.**

```bash
hashcat -a 0 -m 13100 kerberoast.txt rockyou.txt -r best64.rule
```

**4. Mask attack with a known pattern.** Eight digits, then increment down.

```bash
hashcat -a 3 -m 0 md5.txt ?d?d?d?d?d?d?d?d --increment --increment-min 6
```

**5. Hybrid: dictionary word plus four digits.**

```bash
hashcat -a 6 -m 1000 ntlm.txt rockyou.txt ?d?d?d?d
```

**6. Custom charset for a policy-compliant mask.** `?1` is lowercase+digits.

```bash
hashcat -a 3 -m 1000 ntlm.txt -1 ?l?d ?1?1?1?1?1?1?1?1
```

**7. Identify an unknown hash.**

```bash
hashcat --identify unknown.hash
```

**8. Show what is already cracked, then attack only the remainder.**

```bash
hashcat -m 1000 ntlm.txt --show
hashcat -a 0 -m 1000 ntlm.txt --left rockyou.txt
```

**9. Preview rule output without cracking.**

```bash
hashcat -a 0 -m 0 --stdout wordlist.txt -r best64.rule | head -20
```

**10. Benchmark the CPU backend before choosing a strategy.**

```bash
hashcat -b -m 1000
hashcat -I
```

## Output & pipelines

- Recovered plains are appended to `hashcat.potfile` and printed as
  `hash:plain`.
- `--show` reads the potfile and prints all known cracks; `--left` prints only
  the uncracked hashes.
- `--outfile-format` chooses fields; `2` (plain) is the usual machine-readable
  output.
- `--stdout` is the clean way to see the candidate stream, including rule
  expansions.
- Exit codes: `0` = exhausted, `1` = cracked, `2` = error, `3` = aborted,
  `4` = checkpoint abort.
- `--status-json` and `--machine-readable` make progress parseable by scripts.

## Gotchas & operational security

- **CPU-only here.** `hashcat -I` shows a PoCL CPU device. Rates are orders of
  magnitude below a GPU; benchmark before committing to a keyspace.
- **`-O` silently caps length.** Optimized kernels reject candidates longer
  than the kernel limit (often 31). Drop `-O` for long passwords or use a mode
  that supports them.
- **Autodetect can be wrong.** A 32-hex string could be MD5, NTLM, or something
  else; set `-m` explicitly when you know the source.
- **Potfile reuse hides work.** A previous run's cracks are not retried; use
  `--potfile-disable` or a separate `--potfile-path` for a clean test.
- **Rules can explode the keyspace.** `best64` is mild; `dive` and generated
  rules can multiply candidates by thousands.
- **Mask length is exponential.** `?a?a?a?a?a?a?a?a` is ~6.6e15 candidates;
  on this CPU that is not a practical run.
- **Sessions and restore files are state.** Use `--session` for long runs and
  keep the restore file until the run finishes.
- **Temperature and power limits apply.** `--hwmon-temp-abort` protects
  hardware; `-w 4` can make a desktop unresponsive.
- **Cracked plains are sensitive.** They are credentials; store potfiles in the
  engagement directory and delete at close.

## Legal & safety

Crack only hashes you own or are **explicitly authorized** to test. Password
recovery can expose personal accounts and is regulated in many jurisdictions.
Do not use hashcat against third-party services, and do not retain recovered
plaintext longer than the engagement requires.

## Related tools

- [`crunch/`](../crunch/README.md) — generate targeted wordlists and masks.
- [`cewl/`](../cewl/README.md) — build wordlists from target web content.
- [`hydra/`](../hydra/README.md) — online password guessing against services.
- [`hcxtools/`](../hcxtools/README.md) — convert captures to hashcat `22000`.
- [`hcxdumptool/`](../hcxdumptool/README.md) — capture PMKID/EAPOL for mode 22000.
- [`kerbrute/`](../kerbrute/README.md) — Kerberos pre-auth and AS-REP discovery.
- [`netexec/`](../netexec/README.md) — collect NTLM hashes and spray credentials.

## References

- Installed help snapshot: [`help.txt`](help.txt)
- Upstream wiki: https://hashcat.net/wiki/
- Example hashes: https://hashcat.net/wiki/doku.php?id=example_hashes
- Rule-based attack: https://hashcat.net/wiki/doku.php?id=rule_based_attack
- Mask attack: https://hashcat.net/wiki/doku.php?id=mask_attack
