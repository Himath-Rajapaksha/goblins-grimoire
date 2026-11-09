# SecretFinder — secret discovery in JavaScript

- **Version:** packaged SecretFinder (Python 3 port)
- **Binary:** `/usr/local/bin/secretfinder` (wrapper)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/m4ll0k/SecretFinder

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

SecretFinder scans JavaScript files for API keys, tokens, passwords, cloud
credentials, and other sensitive strings using a bank of regular expressions
(Google, AWS, Stripe, Slack, GitHub, JWT, private keys, and more). It can take
a single JS URL, a page (with `-e` to extract and process all linked scripts),
a local file, or a folder, and it can filter by URL substring or regex.

## Use cases

| Situation | Why SecretFinder |
|-----------|------------------|
| Client-side secret leaks | Find keys shipped in bundles |
| Bug bounty recon | High-signal JS analysis |
| Cloud credential exposure | Detect AWS/Google/Azure keys |
| API token discovery | Stripe, Slack, GitHub, JWT |
| Authenticated bundles | Cookies and custom headers supported |
| Evidence | HTML report with matched patterns |

## Requirements & install layout

- Python 3 script with wrapper at `/usr/local/bin/secretfinder`.
- No API keys.
- JS-heavy targets may need cookies/headers to fetch real bundles.
- Works on URLs, local files, and folders.

## Basic syntax

```bash
# scan one JS file
secretfinder -i https://target/app.js -o results.html

# extract and scan every JS on a page
secretfinder -e -i https://target/ -o page.html

# local bundle
secretfinder -i bundle.js

# authenticated bundle
secretfinder -i https://target/app.js -c 'session=abc'

# custom headers
secretfinder -i https://target/app.js -H $'Authorization: Bearer x\nX-Env: prod'

# filter URLs processed
secretfinder -e -i https://target/ -n 'app;vendor' -g 'analytics;tracking'
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-i, --input <url\\|file\\|folder>` | Input target |
| `-e, --extract` | Extract all JS links from the page and scan them |
| `-o, --output <file>` | Output file (default `output.html`) |
| `-r, --regex <regex>` | Filter processing by URL regex |
| `-b, --burp` | Support a Burp-exported file |
| `-c, --cookie <cookie>` | Cookies for authenticated JS |
| `-g, --ignore <strings>` | Skip JS URLs containing these `;`-separated strings |
| `-n, --only <strings>` | Only process JS URLs containing these strings |
| `-H, --headers <headers>` | Headers as `Name:Value\nName:Value` |
| `-p, --proxy <host:port>` | Proxy |
| `-h, --help` | Help |

## Recipes

```bash
# 1. Single bundle scan
secretfinder -i https://target/static/app.js -o app.html

# 2. Whole page, all scripts
secretfinder -e -i https://target/ -o page.html

# 3. Authenticated scan
secretfinder -i https://target/app.js -c 'session=abc' -H $'X-Env: prod'

# 4. Focus on first-party bundles
secretfinder -e -i https://target/ -n 'app;main;bundle' -g 'vendor;polyfill'

# 5. Local folder of downloaded JS
secretfinder -i '/evidence/js/*.js' -o local.html

# 6. Proxy through Burp for inspection
secretfinder -i https://target/app.js -p 127.0.0.1:8080

# 7. Extract candidate secrets from the report
grep -Eo '(AKIA[0-9A-Z]{16}|AIza[0-9A-Za-z_-]{35}|ghp_[A-Za-z0-9]{36}|sk_live_[A-Za-z0-9]+)' results.html | sort -u

# 8. Verify a Google API key (read-only, authorized key)
curl 'https://maps.googleapis.com/maps/api/geocode/json?address=test&key=AIza...'

# 9. Triage by provider
grep -Eio 'aws|google|stripe|slack|github|jwt|private key' results.html | sort | uniq -c
```

## Output & pipelines

- HTML report with each secret, its pattern name, and the source file.
- Console summary of matched types.
- Candidates feed manual verification, `trufflehog`, `gitleaks`, and cloud
  permission checks.
- Always verify before reporting — many matches are public/sample keys.

## Gotchas & operational security

- Regex matching produces **false positives**: documentation keys, test keys,
  placeholder values, and public client IDs.
- Minified code can split secrets across strings; static regexes miss them.
- Source maps expose original variable names and more literals — check for
  them separately.
- Authenticated scans need a live session; expired cookies return login pages.
- Never use discovered keys beyond the authorized scope; testing a key may
  create billable or destructive actions.
- Store reports securely — they contain credentials.
- `-p` proxy output includes the secrets; protect the proxy history too.

## Legal & safety

Extracting secrets from public JavaScript is a common authorized activity, but
**using** them is not. Report exposure to the owner, do not pivot with found
credentials unless the rules of engagement explicitly allow it, and redact
secrets in reports.

## Related tools

- [`linkfinder/`](../linkfinder/README.md) — endpoint discovery
- [`subjs/`](../subjs/README.md) — collect JS URLs
- [`sourcemapper/`](../sourcemapper/README.md) — reconstruct sources
- [`trufflehog`](https://github.com/trufflesecurity/trufflehog),
  [`gitleaks`](https://github.com/gitleaks/gitleaks) — secret scanning

## References

- https://github.com/m4ll0k/SecretFinder
