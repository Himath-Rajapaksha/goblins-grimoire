# john — John the Ripper password cracker (core build)

- **Version:** John the Ripper 1.9.0 (`john` with no arguments)
- **Binary:** `/usr/sbin/john` (Debian package `john`; `/usr/sbin` is not on a
  normal user's `PATH`)
- **Helpers:** `/usr/sbin/unshadow`, `/usr/sbin/unafs`, `/usr/sbin/unique`
- **Config:** `/etc/john/john.conf` (rules, incremental modes, `$JOHN` paths)
- **Data:** `/usr/share/john/` (`password.lst`, `*.chr` charset files)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://www.openwall.com/john/
- **Docs:** https://www.openwall.com/john/doc/ ; `man john`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

John the Ripper recovers plaintext passwords from password hashes. It loads
one or more password files, applies a candidate-generation mode (single,
wordlist, incremental, external), hashes each candidate in the detected format,
and compares the result with the stored hash. Cracked credentials are written
to `john.pot` and can be printed with `--show`.

This is the **core** John build, not `-jumbo`. The installed binary supports
only the formats listed in [`help.txt`](help.txt): descrypt, bsdicrypt,
md5crypt, bcrypt, LM, AFS, tripcode, `dummy`, and the generic `crypt`. It does
**not** include NTLM, raw MD5/SHA-1, SHA-512 crypt, Kerberos, or archive
formats. Use [`hashcat`](../hashcat/README.md) for those.

## Use cases

| Situation | Why john |
|-----------|----------|
| `/etc/passwd` + `/etc/shadow` from a Linux host | `unshadow` then `--single`/`--wordlist` |
| Old LM or DES crypt hashes | Native, very fast on CPU |
| md5crypt (`$1$`) or bcrypt (`$2a$`) | Supported formats |
| Rules-based mangling of a small wordlist | `--rules` with the shipped rule sets |
| Resumable long run | `--session` / `--restore` / `--status` |
| Benchmarking CPU throughput | `--test=seconds` |
| Generating candidate lists for another tool | `--stdout` with `--rules` |

## Modes

| Mode | Flag | How it builds candidates |
|------|------|--------------------------|
| Single | `--single` | Login name, GECOS fields, home directory; fastest first pass |
| Wordlist | `--wordlist=FILE` | Each line of the file, optionally mangled |
| Rules | `--rules` | Apply `[List.Rules:Wordlist]` to each word |
| Incremental | `--incremental[=MODE]` | Brute force over a charset; `MODE` names a `[Incremental:*]` section |
| External | `--external=MODE` | Custom candidate generator from `john.conf` |
| stdin | `--stdin` | Read candidates from standard input |
| stdout | `--stdout[=LENGTH]` | Print candidates instead of cracking |

Shipped incremental modes: `ASCII`, `LM_ASCII`, `Alnum`, `Alpha`, `LowerNum`,
`UpperNum`, `LowerSpace`, `Lower`, `Upper`, `Digits`.

## Flag reference

### Cracking modes and rules

| Flag | Meaning |
|------|---------|
| `--single` | "Single crack" mode |
| `--wordlist=FILE` | Wordlist mode |
| `--stdin` | Read words from stdin |
| `--rules` | Enable word-mangling rules for wordlist mode |
| `--incremental[=MODE]` | Incremental (brute-force) mode |
| `--external=MODE` | External mode or word filter |
| `--stdout[=LENGTH]` | Output candidates instead of cracking |
| `--format=NAME` | Force hash type: `descrypt`, `bsdicrypt`, `md5crypt`, `bcrypt`, `LM`, `AFS`, `tripcode`, `dummy`, `crypt` |

### Sessions and parallel work

| Flag | Meaning |
|------|---------|
| `--session=NAME` | Name a new session |
| `--restore[=NAME]` | Restore an interrupted session |
| `--status[=NAME]` | Print session status |
| `--fork=N` | Fork `N` processes |
| `--node=MIN[-MAX]/TOTAL` | This node's share of the work |
| `--save-memory=LEVEL` | Memory saving, level 1–3 |

### Input selection and output

| Flag | Meaning |
|------|---------|
| `--show` | Show cracked passwords for the loaded files |
| `--users=[-]LOGIN\|UID[,…]` | Load or exclude users |
| `--groups=[-]GID[,…]` | Load or exclude groups |
| `--shells=[-]SHELL[,…]` | Load or exclude shells |
| `--salts=[-]N` | Load salts with or without at least `N` passwords |
| `--make-charset=FILE` | Build a charset file (overwrites it) |
| `--test[=TIME]` | Benchmark each format for `TIME` seconds |

## Recipes

**1. Build a crackable file from passwd and shadow.**

```bash
sudo cp /etc/passwd /etc/shadow ~/john-demo/
unshadow ~/john-demo/passwd ~/john-demo/shadow > ~/john-demo/combined.txt
```

**2. Single mode first — it is cheap and often wins.**

```bash
/usr/sbin/john --single ~/john-demo/combined.txt
```

**3. Wordlist with rules.**

```bash
/usr/sbin/john --wordlist=/usr/share/john/password.lst --rules \
  ~/john-demo/combined.txt
```

**4. Incremental brute force with a named charset.**

```bash
/usr/sbin/john --incremental=Alnum ~/john-demo/combined.txt
```

**5. Named, resumable session.**

```bash
/usr/sbin/john --session=acme --wordlist=words.txt ~/john-demo/combined.txt
/usr/sbin/john --status=acme
/usr/sbin/john --restore=acme
```

**6. Split work across four processes.**

```bash
/usr/sbin/john --fork=4 --wordlist=words.txt ~/john-demo/combined.txt
```

**7. Show what was cracked.**

```bash
/usr/sbin/john --show ~/john-demo/combined.txt
```

**8. Generate mangled candidates for another tool.**

```bash
/usr/sbin/john --wordlist=words.txt --rules --stdout > candidates.txt
```

**9. Benchmark the installed formats.**

```bash
/usr/sbin/john --test=5
```

## Gotchas

- **Core, not jumbo.** No NTLM, no raw hashes, no SHA-512 crypt, no 7-Zip,
  no Office. Check `--format` support before promising a result.
- **`/usr/sbin` is not on a normal user's `PATH`.** Call `/usr/sbin/john` or
  use `sudo`.
- **`--show` needs the same input file and format.** It reads `john.pot`, not
  the hash file alone.
- **`--rules` without `--wordlist` does nothing useful.** Rules mangle wordlist
  entries; they are not a mode by themselves.
- **`--incremental` is exponential.** Start with a short custom charset or
  `--incremental=Digits`, not `ASCII`, unless the keyspace is understood.
- **`--fork` is per machine, not per cluster.** For distributed runs use
  `--node` with the same total on each host.
- **`unshadow` output contains real hashes.** Treat it as a secret, keep it
  encrypted, and delete it at engagement close.
- **Do not crack hashes you do not own.** Password recovery requires written
  authorization and a defined scope.

## Related docs

- [`hashcat`](../hashcat/README.md) — GPU/CPU cracking for the formats core
  John lacks
- [`hydra`](../hydra/README.md) — online guessing against a live service
- [`medusa`](../medusa/README.md) — alternative online brute forcer
- [`netexec`](../netexec/README.md) — AD credential validation and spraying
