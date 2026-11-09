# CeWL — custom wordlist generator

- **Version:** CeWL 6.2.1 (More Fixes)
- **Binary:** `/usr/bin/cewl`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/digininja/CeWL
- **Docs:** https://digi.ninja/projects/cewl.php

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

CeWL spiders a website to a chosen depth and extracts words, producing a
wordlist tailored to the target's vocabulary, product names, and terminology.
With `-a` it also pulls document metadata (authors, emails), and `-e` collects
email addresses. The output is commonly used for password guessing, username
generation, and content analysis.

## Use cases

| Situation | Why CeWL |
|-----------|----------|
| Password wordlists | Target-specific vocabulary beats generic lists |
| Username seeds | Author names from metadata |
| Email harvesting | `-e` collects addresses |
| Brand vocabulary | Understand naming conventions |
| CTF/lab prep | Fast custom lists |
| Content review | Frequency counts with `-c` |

## Requirements & install layout

- Debian package 6.2.1; binary `/usr/bin/cewl`.
- Ruby-based; uses `exiftool` for `--meta`.
- No API keys.
- Output goes to stdout unless `-w` is used.

## Basic syntax

```bash
# spider two levels and print words
cewl https://target/

# write a wordlist
cewl -w words.txt -d 3 -m 5 https://target/

# include emails and metadata
cewl -e --email_file emails.txt -a --meta_file meta.txt https://target/

# count occurrences
cewl -c https://target/
```

## Flag reference

### Spidering and output
| Flag | Meaning |
|------|---------|
| `<url>` | Site to spider |
| `-d <x>, --depth <x>` | Spider depth (default 2) |
| `-m, --min_word_length <n>` | Minimum word length (default 3) |
| `-x, --max_word_length <n>` | Maximum word length |
| `-k, --keep` | Keep the downloaded pages |
| `-o, --offsite` | Follow offsite links |
| `--exclude <file>` | Paths to exclude |
| `--allowed <regex>` | Only follow paths matching the regex |
| `-w, --write <file>` | Write words to a file |
| `-n, --no-words` | Suppress word output |
| `-g <x>, --groups <x>` | Also emit groups of words |
| `--lowercase` | Lowercase all words |
| `--with-numbers` | Keep words containing digits |
| `--convert-umlauts` | Convert Latin-1 umlauts |
| `-c, --count` | Show counts per word |
| `-v, --verbose` / `--debug` | Verbosity |

### Metadata and email
| Flag | Meaning |
|------|---------|
| `-a, --meta` | Include document metadata |
| `--meta_file <file>` | Metadata output file |
| `-e, --email` | Include email addresses |
| `--email_file <file>` | Email output file |
| `--meta-temp-dir <dir>` | exiftool temp dir (default `/tmp`) |

### Request settings
| Flag | Meaning |
|------|---------|
| `-u, --ua <agent>` | User-Agent |
| `--auth_type <basic\\|digest>` | Auth type |
| `--auth_user <user>` / `--auth_pass <pass>` | Credentials |
| `--proxy_host <host>` / `--proxy_port <port>` | Proxy |
| `--proxy_username` / `--proxy_password` | Proxy auth |
| `-H, --header <name:value>` | Extra header (repeatable) |

## Recipes

```bash
# 1. Simple target wordlist
cewl -w words.txt https://target/

# 2. Deeper spider, longer words only
cewl -w words.txt -d 4 -m 6 -x 20 https://target/

# 3. Lowercase, keep numbers, count occurrences
cewl -w words.txt --lowercase --with-numbers -c https://target/

# 4. Emails and metadata
cewl -e --email_file emails.txt -a --meta_file meta.txt https://target/

# 5. Stay on the main site and skip logout paths
cewl -w words.txt --allowed '/blog/.*' --exclude excludes.txt https://target/

# 6. Authenticated area
cewl -w words.txt --auth_type basic --auth_user alice --auth_pass 'secret' https://target/

# 7. Proxy through Burp
cewl -w words.txt --proxy_host 127.0.0.1 --proxy_port 8080 https://target/

# 8. Word groups for passphrase candidates
cewl -g 2 -w groups.txt https://target/

# 9. Merge with a general list and deduplicate
cat words.txt /usr/share/wordlists/rockyou.txt | sort -u > combined.txt

# 10. Inspect top words
cewl -c --no-words https://target/ | sort -rn | head -30
```

## Output & pipelines

- Wordlist to stdout or `-w`.
- `-e` writes emails; `-a` writes metadata (author, producer, etc.).
- `-c` adds counts, useful for ranking.
- Output feeds `wfuzz`, `hydra`, `hashcat`, `kerbrute` (username seeds), and
  password-spraying tools.

## Gotchas & operational security

- Spidering is visible and can be slow on large sites; depth 4+ explodes.
- The extracted vocabulary is only useful if the target reuses its own words
  in passwords — combine with rules rather than using it raw.
- `-a` requires `exiftool`; without it metadata is skipped.
- Documents downloaded for metadata may contain malware — isolate them.
- `--offsite` leaks the crawl to third-party sites; prefer `--allowed`.
- Login forms and JS-rendered content are not handled; results can be thin.
- Email/metadata are personal data; scope and retain per policy.

## Legal & safety

Use only on authorized targets. Spidering and metadata harvesting are active
interactions; respect robots, rate limits, and privacy law.

## Related tools

- [`metagoofil/`](../metagoofil/README.md) — document metadata at scale
- [`wfuzz/`](../wfuzz/README.md), [`dirsearch/`](../dirsearch/README.md) — wordlist consumers
- [`theharvester/`](../theharvester/README.md) — email discovery
- [`crunch/`](../crunch/README.md) — pattern-based list generation

## References

- https://digi.ninja/projects/cewl.php
- https://github.com/digininja/CeWL
