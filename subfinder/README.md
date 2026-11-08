# subfinder — passive subdomain discovery

- **Version:** v2.17.0
- **Binary:** `/usr/local/bin/subfinder`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/projectdiscovery/subfinder
- **Docs:** https://docs.projectdiscovery.io/tools/subfinder/overview

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Subfinder enumerates subdomains using **passive** sources only — certificate
transparency logs, search engines, DNS datasets, threat-intel APIs, and similar.
It never brute-forces names, so it is fast and quiet, and it does not touch the
target's infrastructure directly.

Use it as the first stage of recon: turn one root domain into an asset list that
`httpx`, `katana`, `nuclei`, and `dalfox` can consume.

## Use cases

| Situation | Why subfinder |
|-----------|---------------|
| Starting a bug-bounty program | Build the full in-scope asset inventory quickly |
| Attack-surface monitoring | Diff new subdomains over time |
| Pentest scoping | Discover forgotten/staging assets before testing |
| OSINT | Map an organization's public footprint without active traffic |
| Pipeline seeding | Feed live-host probing and crawling |

## Basic syntax

```bash
subfinder -d example.com
subfinder -d example.com,example.org
subfinder -dL domains.txt
```

## Flag reference

### Input
| Flag | Meaning |
|------|---------|
| `-d, -domain string[]` | Domain(s) to enumerate (repeat/comma-separated) |
| `-dL, -list string` | File containing one domain per line |

### Source selection
| Flag | Meaning |
|------|---------|
| `-s, -sources string[]` | Only use specific sources, e.g. `-s crtsh,github` |
| `-recursive` | Only sources that can recurse (subdomain.domain.tld vs domain.tld) |
| `-all` | Use every source (slow, most complete) |
| `-es, -exclude-sources string[]` | Exclude sources, e.g. `-es alienvault,zoomeyeapi` |
| `-ls, -list-sources` | Print available sources and exit |

### Filtering
| Flag | Meaning |
|------|---------|
| `-m, -match string[]` | Keep only subdomains matching (substring/file) |
| `-f, -filter string[]` | Drop subdomains matching |
| `-match-regex string[]` | Regex include filter on output |
| `-filter-regex string[]` | Regex exclude filter on output |
| `-ei, -exclude-ip` | Remove entries that are bare IPs |
| `-mr, -max-results int` | Per-source result cap (0 = unlimited) |

### Rate limiting & performance
| Flag | Meaning |
|------|---------|
| `-rl, -rate-limit int` | Global requests/second |
| `-rls, -rate-limits value` | Per-provider limits, e.g. `hackertarget=10/m` |
| `-t int` | Concurrent domains / DNS lookups (default 10) |
| `-timeout int` | Per-request seconds (default 30) |
| `-max-time int` | Total enumeration minutes (default 10) |
| `-rsr, -response-size-read int` | Max passive-source response bytes |

### Resolution & validation
| Flag | Meaning |
|------|---------|
| `-nW, -active` | Only output subdomains that resolve |
| `-oI, -ip` | Include resolved IP (with `-active`) |
| `-r string[]` | Comma-separated resolvers |
| `-rL, -rlist string` | Resolver list file |
| `-proxy string` | HTTP proxy for source requests |

### Output
| Flag | Meaning |
|------|---------|
| `-o, -output string` | Write results to file |
| `-oJ, -json` | JSONL output |
| `-oD, -output-dir string` | Per-domain files (with `-dL`) |
| `-cs, -collect-sources` | Include source attribution (`-json` only) |
| `-silent` | Print subdomains only (no banner/logs) |
| `-nc, -no-color` | Disable ANSI colors |
| `-stats` | Source statistics |
| `-v` | Verbose |
| `-version` | Version |

### Configuration & maintenance
| Flag | Meaning |
|------|---------|
| `-config string` | Main config (`~/.config/subfinder/config.yaml`) |
| `-pc, -provider-config string` | API keys (`~/.config/subfinder/provider-config.yaml`) |
| `-up, -update` | Self-update |
| `-duc, -disable-update-check` | No update check |

## Recipes

```bash
# Fast default pass
subfinder -d example.com -silent

# Maximum coverage, quiet, to file
subfinder -d example.com -all -silent -o subs.txt

# Many domains, one output dir per domain
subfinder -dL domains.txt -oD out/ -silent

# Only resolving hosts, with IPs
subfinder -d example.com -nW -oI -silent

# Certificate-transparency only (fast, no API keys)
subfinder -d example.com -s crtsh -silent

# Filter staging/dev assets
subfinder -d example.com -all -silent -match 'dev|staging|test|uat'

# JSONL with source attribution (for databases)
subfinder -d example.com -all -oJ -cs -o subs.jsonl

# Directly into httpx
subfinder -d example.com -silent | httpx -silent -sc -title -td
```

## API keys (important)

Many sources are rate-limited or disabled without keys. Add them to
`~/.config/subfinder/provider-config.yaml`:

```yaml
github:
  - ghp_xxxxxxxxxxxxxxxxxxxx
shodan:
  - SHODAN_API_KEY
virustotal:
  - VT_API_KEY
securitytrails:
  - ST_API_KEY
censys:
  - CENSYS_API_ID:CENSYS_API_SECRET
```

Check active sources with `subfinder -ls`.

## Output

Default: one subdomain per line on stdout. With `-oJ` each line is a JSON object
including `host`, `source`, `input`, and optionally `ip`.

## Gotchas

- Passive results can include wildcard DNS noise; validate with `httpx` or `-nW`.
- `-all` is much slower and burns API quotas.
- No API keys = fewer sources and tighter limits, not zero results.
- This does not do DNS brute force; use `dnsx`/`shuffledns` for that.
- Respect program scope — subdomains found are not automatically in scope.

## Typical pipeline position

```
subfinder -> httpx -> katana -> nuclei/dalfox
```

## References

- Docs: https://docs.projectdiscovery.io/tools/subfinder/overview
- Usage: https://docs.projectdiscovery.io/tools/subfinder/usage
- Deep-dive blog: https://blog.projectdiscovery.io/do-you-really-know-subfinder-an-in-depth-guide-to-all-features-of-subfinder-beginner-to-advanced/
- Source: https://github.com/projectdiscovery/subfinder
