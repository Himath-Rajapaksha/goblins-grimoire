# JoomScan — Joomla vulnerability scanner

- **Version:** OWASP JoomScan 0.0.7 (2018/09/23)
- **Binary:** `/usr/local/bin/joomscan` → `/opt/joomscan/joomscan.pl`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/OWASP/joomscan

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

JoomScan fingerprints Joomla installations, reports the exact version, lists
components, and probes for known vulnerable extensions, misconfigurations,
backup files, and exposed admin paths. It can scan a single URL or a list,
use a cookie or custom User-Agent, and write a report.

## Use cases

| Situation | Why JoomScan |
|-----------|--------------|
| Joomla fingerprinting | Version and component discovery |
| Vulnerability triage | Known extension issues |
| Misconfiguration checks | Backup, config, and admin exposure |
| Component enumeration | Installed extensions |
| Bulk Joomla hunt | `--mass` over a URL list |
| Evidence | Report file per scan |

## Requirements & install layout

- Installed at `/opt/joomscan`; wrapper in `/usr/local/bin`.
- Perl script; requires Perl modules shipped with the install.
- No API keys.
- The banner is noisy; use `--no-report` for console-only work.
- Version 0.0.7 is old — signatures reflect 2018-era Joomla.

## Basic syntax

```bash
# full scan
joomscan -u https://target/

# version only
joomscan -u https://target/ -jv

# enumerate components
joomscan -u https://target/ -ec

# bulk scan
joomscan -m targets.txt

# authenticated
joomscan -u https://target/ --cookie "session=abc"

# no report file
joomscan -u https://target/ -nr

# proxy and random UA
joomscan -u https://target/ --proxy http://127.0.0.1:8080 -r
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-u, --url <url>` | Joomla URL to scan |
| `-m, --mass <file>` | File of URLs |
| `-ec, --enumerate-components` | Enumerate components |
| `-jv, --joomla-version` | Print version and exit |
| `-nr, --no-report` | Do not write a report |
| `--cookie <string>` | Cookie |
| `-a, --user-agent <ua>` | User-Agent |
| `-r, --random-agent` | Random User-Agent |
| `--timeout <sec>` | Timeout |
| `--proxy <url>` | Proxy (`http://`, `https://`, `socks://`) |
| `--about` | Author info |
| `--update` | Update |
| `--version` | Version |
| `-h, --help` | Help |

## Recipes

```bash
# 1. Version check only
joomscan -u https://target/ -jv

# 2. Full scan without a report file
joomscan -u https://target/ -nr

# 3. Component enumeration
joomscan -u https://target/ -ec -nr

# 4. Bulk scan a list
joomscan -m targets.txt -nr

# 5. Authenticated scan
joomscan -u https://target/ --cookie "session=abc" -nr

# 6. Through Burp
joomscan -u https://target/ --proxy http://127.0.0.1:8080 -nr

# 7. Random UA with a timeout
joomscan -u https://target/ -r --timeout 20 -nr

# 8. Cross-check the version manually
curl -s https://target/administrator/manifests/files/joomla.xml | grep -i version

# 9. Map components to CVEs
searchsploit joomla <component>
```

## Output & pipelines

- Console report lists version, components, and findings.
- Report files are written to the working directory unless `-nr`.
- Version and component names feed `searchsploit`, `nuclei`, and CVE databases.
- Findings should be manually verified — the scanner's signatures are dated.

## Gotchas & operational security

- **0.0.7 is from 2018**: newer Joomla versions and extensions are not in its
  signature set; treat clean results as inconclusive.
- The version check can be fooled by custom `joomla.xml` files or caching.
- Component enumeration is noisy and can trigger WAF rules.
- `--mass` serialises scans; pace them to avoid blocks.
- Report files may include full response bodies; protect them.
- The banner clears the screen; capture output to a file for logs.
- False positives occur on non-Joomla sites with Joomla-like paths.

## Legal & safety

Only scan authorized Joomla installations. Active probing is visible to the
target and may affect availability on fragile hosts.

## Related tools

- [`cmseek/`](../cmseek/README.md) — general CMS detection
- [`dirsearch/`](../dirsearch/README.md) — path discovery
- [`nuclei`](https://github.com/projectdiscovery/nuclei) — CVE templates
- `searchsploit` — local exploit database

## References

- https://github.com/OWASP/joomscan
- https://owasp.org/www-project-joomscan/
