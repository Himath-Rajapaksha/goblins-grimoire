# theHarvester — email, subdomain, and host OSINT aggregation

- **Version:** theHarvester 4.11.1
- **Binary:** `/usr/local/bin/theHarvester` (pipx)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/laramies/theHarvester
- **Docs:** https://github.com/laramies/theHarvester/wiki

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

theHarvester queries many public sources — search engines, certificate
transparency logs, code repositories, breach databases, DNS datasets — for a
domain or company name, and aggregates emails, subdomains, hosts, employee
names, open ports, and banners. One command fans out across dozens of sources
and normalises the results into a report.

## Use cases

| Situation | Why theHarvester |
|-----------|------------------|
| External footprint | Collect emails and subdomains in one pass |
| Credential exposure | Check breach sources for domain accounts |
| Subdomain discovery | Certificate transparency plus passive DNS |
| Pre-engagement scope | Build an asset list before touching targets |
| Threat intel | Find leaked documents and mentions |
| API discovery | Scan for exposed API endpoints |

## Requirements & install layout

- Installed via pipx at `/opt/pipx`; entry point `/usr/local/bin/theHarvester`.
- Most sources work without keys; several require API keys stored in
  `api-keys.yaml` (created on first run in `~/.theHarvester/`).
- `proxies.yaml` in the same directory configures `-p`.
- Python 3.13 environment managed by pipx.
- Network access to many providers; rate limits apply per source.

## Basic syntax

```bash
# default source set for a domain
theHarvester -d example.com -b all

# a focused source list
theHarvester -d example.com -b crtsh,dnsdumpster,hackertarget

# limit results, save XML and JSON
theHarvester -d example.com -b all -l 200 -f evidence/acme

# resolve discovered subdomains and take screenshots
theHarvester -d example.com -b crtsh -r -n --screenshot shots/

# brute force subdomains with a wordlist
theHarvester -d example.com -b all -c -w subdomains.txt

# check for subdomain takeovers
theHarvester -d example.com -b all -t
```

## Flag reference

### Target and scope
| Flag | Meaning |
|------|---------|
| `-d, --domain <domain>` | Company name or domain to search (required) |
| `-l, --limit <n>` | Max results per source (default 500) |
| `-S, --start <n>` | Start at result number X (default 0) |
| `-b, --source <list>` | Comma-separated sources (see below) |

### Network enrichment
| Flag | Meaning |
|------|---------|
| `-e, --dns-server <ip>` | DNS server for lookups |
| `-r, --dns-resolve [list]` | Resolve subdomains (resolver list or defaults) |
| `-n, --dns-lookup` | Enable DNS server lookup |
| `-c, --dns-brute` | Brute force subdomains |
| `-w, --wordlist <file>` | Wordlist for DNS brute force / API scanning |
| `-a, --api-scan` | Scan for API endpoints |
| `-s, --shodan` | Enrich discovered hosts with Shodan |
| `-t, --take-over` | Check subdomain takeover |
| `--screenshot <dir>` | Screenshot resolved domains |

### Output and request control
| Flag | Meaning |
|------|---------|
| `-f, --filename <base>` | Write `<base>.xml` and `<base>.json` |
| `-p, --proxies` | Use `proxies.yaml` |
| `-q, --quiet` | Suppress missing-key warnings |
| `-h, --help` | Help |

### Sources (`-b`)
`baidu, bevigil, bitbucket, brave, bufferoverun, builtwith, censys, certspotter,
chaos, commoncrawl, criminalip, crtsh, dehashed, dnsdumpster, duckduckgo, dymo,
fofa, fullhunt, github-code, gitlab, hackertarget, haveibeenpwned, hudsonrock,
hunter, hunterhow, intelx, leakix, leaklookup, mojeek, netlas, onyphe, otx,
pentesttools, projectdiscovery, rapiddns, robtex, rocketreach, securityscorecard,
securityTrails, sherlockeye, shodan, shodanInternetDB, subdomaincenter,
subdomainfinderc99, thc, threatcrowd, tomba, urlscan, venacus, virustotal,
waybackarchive, whoisxml, windvane, yahoo, zoomeye`

`-b all` runs the full set; many sources silently skip without a key.

## Recipes

```bash
# 1. Fast passive footprint
theHarvester -d example.com -b crtsh,hackertarget,rapiddns,urlscan -l 500

# 2. Full run with evidence files
theHarvester -d example.com -b all -l 500 -f /evidence/example

# 3. Resolve and screenshot every discovered host
theHarvester -d example.com -b crtsh -r -n --screenshot /evidence/shots

# 4. Brute force with a custom list
theHarvester -d example.com -b crtsh -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# 5. Takeover check
theHarvester -d example.com -b all -t

# 6. API endpoint discovery
theHarvester -d example.com -b all -a -w api-wordlist.txt

# 7. Breach exposure check
theHarvester -d example.com -b dehashed,haveibeenpwned,leakix -q

# 8. Extract unique emails from JSON output
jq -r '.emails[]' /evidence/example.json | sort -u
```

## Output & pipelines

- Console table plus optional XML and JSON (`-f`).
- JSON keys include `emails`, `hosts`, `ips`, `urls`, `linkedin_people`,
  `linkedin_links`, `interesting_urls`, `trello_urls`, `twitter_people`.
- Feed hosts into `httpx`, `nuclei`, `gobuster`, or `netexec`.
- Emails feed phishing simulations (authorized) and breach analysis.
- Screenshots land in the target directory named by host.

## Gotchas & operational security

- Many sources need API keys; without them results look thin and warnings are
  noisy (use `-q`).
- Search engines block aggressive use; `-p` and `-l` reduce friction.
- `-b all` is slow and likely to hit rate limits.
- Results are public-source only — stale records and false positives are
  normal; verify before acting.
- Breach sources return personal data; handle under privacy law.
- The tool contacts dozens of third parties, leaking the target domain to each.
- DNS brute force (`-c`) is active and visible to authoritative servers.

## Legal & safety

Passive OSINT is generally lower risk than scanning, but still regulated by
privacy law and provider terms. Collect only what the engagement needs, store
it securely, and never use breach data to access accounts.

## Related tools

- [`spiderfoot/`](../spiderfoot/README.md) — automated correlation
- [`recon-ng/`](../recon-ng/README.md) — modular framework
- [`sherlock/`](../sherlock/README.md), [`holehe/`](../holehe/README.md) — account discovery
- [`metagoofil/`](../metagoofil/README.md) — document metadata
- [`ghunt/`](../ghunt/README.md) — Google account OSINT

## References

- https://github.com/laramies/theHarvester
- https://github.com/laramies/theHarvester/wiki
