# CMSeeK — CMS detection and deep scan

- **Version:** CMSeeK 1.1.3
- **Binary:** `/usr/local/bin/cmseek` (wrapper to `cmseek.py`)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/Tuhinshubhra/CMSeeK

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

CMSeeK identifies the content management system behind a site and, in deep
scan mode, enumerates its version, components, plugins/themes, users, and
known vulnerabilities. It ships detection modules for WordPress, Joomla,
Drupal, and dozens of smaller CMSs, and can scan a list of targets in batch.

## Use cases

| Situation | Why CMSeeK |
|-----------|------------|
| Technology fingerprinting | Identify CMS and version quickly |
| Component enumeration | List plugins/themes for vuln mapping |
| Version-based triage | Match against known CVEs |
| Bulk assessment | Scan a list of sites |
| Lightweight recon | Fast CMS-only mode |
| Evidence | Saved scan results per target |

## Requirements & install layout

- Installed at `/opt/cmseek` with a wrapper in `/usr/local/bin`.
- Python 3; results stored under the install's `Result/` directory.
- No API keys; uses the target's own responses.
- `--update` requires git and network access.

## Basic syntax

```bash
# guided scan
cmseek

# scan one target
cmseek -u https://target/

# CMS detection only
cmseek -u https://target/ --only-cms

# light scan (CMS + version, no deep scan)
cmseek -u https://target/ --light-scan

# batch scan with no prompts
cmseek -l targets.txt --batch

# random user agent
cmseek -u https://target/ -r

# verbose
cmseek -u https://target/ -v
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-u, --url <url>` | Target URL |
| `-l, --list <file>` | File of targets (comma-separated or one per line) |
| `-i, --ignore-cms <ids>` | Skip these CMS IDs (avoid false positives) |
| `--strict-cms <ids>` | Only check against these CMS IDs |
| `--skip-scanned` | Skip targets already scanned |
| `--light-scan` | Skip deep scan |
| `-o, --only-cms` | CMS detection only |
| `--follow-redirect` / `--no-redirect` | Redirect handling |
| `-r, --random-agent` | Random User-Agent |
| `--googlebot` | Googlebot User-Agent |
| `--user-agent <ua>` | Custom User-Agent |
| `-v, --verbose` | Verbose output |
| `--update` | Update CMSeeK |
| `--version` | Show version |
| `--clear-result` | Delete stored results |
| `--batch` | Never prompt between batch targets |
| `-h, --help` | Help |

## Recipes

```bash
# 1. Quick CMS-only check
cmseek -u https://target/ --only-cms

# 2. Identify CMS and version without deep scan
cmseek -u https://target/ --light-scan

# 3. Full scan
cmseek -u https://target/ -v

# 4. Batch, non-interactive
cmseek -l targets.txt --batch -v

# 5. Avoid WordPress false positives on a custom app
cmseek -u https://target/ -i 1

# 6. Force a WordPress-only check
cmseek -u https://target/ --strict-cms 1

# 7. Stay on the exact target
cmseek -u https://target/ --no-redirect

# 8. Randomise the UA
cmseek -u https://target/ -r

# 9. Inspect stored results
ls -la /opt/cmseek/Result/
```

## Output & pipelines

- Console report with CMS name, version, components, and users where found.
- Per-target results under `/opt/cmseek/Result/`.
- Findings feed `wpscan`, `joomscan`, `droopescan`, `nuclei` CMS templates, and
  CVE lookup.
- Version strings are the key pivot to vulnerability research.

## Gotchas & operational security

- Detection relies on signatures (headers, paths, HTML); customised or
  headless CMSs produce false negatives.
- Deep scan can be noisy and may trigger WAF blocks.
- Version strings are often hidden or spoofed; verify manually.
- Bulk scans (`-l`) accelerate blocking; pace them.
- `--clear-result` deletes evidence — do not run it mid-engagement.
- WordPress/Joomla user enumeration modules can trigger lockouts or alerts.
- Results directory may contain personal data (user lists).

## Legal & safety

Only scan systems you are authorized to assess. CMS enumeration is active and
visible; respect rate limits and the target's acceptable-use policy.

## Related tools

- [`joomscan/`](../joomscan/README.md) — Joomla-specific
- [`wpscan`](https://wpscan.com/) — WordPress-specific
- [`dirsearch/`](../dirsearch/README.md) — path discovery
- [`nuclei`](https://github.com/projectdiscovery/nuclei) — CVE templates

## References

- https://github.com/Tuhinshubhra/CMSeeK
