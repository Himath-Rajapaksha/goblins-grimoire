# dirsearch — web path and file brute forcer

- **Version:** packaged Debian build (dirsearch.py, pkg_resources deprecation warning on 3.13)
- **Binary:** `/usr/bin/dirsearch`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/maurosoria/dirsearch

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

dirsearch brute-forces web paths and files using a wordlist, then filters the
responses by status code, size, text, regex, or similarity to a known 404 page.
It supports multiple targets, recursion, custom extensions, raw requests,
authentication, proxies, and structured reports.

## Use cases

| Situation | Why dirsearch |
|-----------|---------------|
| Directory discovery | Fast path enumeration with recursion |
| Backup/config hunting | Extension lists surface `.bak`, `.old`, `.env` |
| Recursive mapping | Follow discovered directories to a depth |
| Authenticated scans | Cookies, headers, basic/digest/bearer/NTLM |
| Raw request replay | Reuse a Burp-style request file |
| Multi-target sweeps | URL file, stdin, or CIDR |
| Evidence | JSON/XML/HTML/SQLite reports |

## Requirements & install layout

- Debian package; binary `/usr/bin/dirsearch`.
- Default wordlists ship under `/usr/share/dirsearch/db/` (varies by package).
- Config at `config.ini` in the working directory or `--config`.
- Python 3; on 3.13 the `pkg_resources` warning is cosmetic.

## Basic syntax

```bash
# default scan
dirsearch -u https://target/

# extensions and a custom wordlist
dirsearch -u https://target/ -e php,asp,aspx,jsp -w /path/wordlist.txt

# recursive scan with a depth cap
dirsearch -u https://target/ -r -R 2

# authenticated scan
dirsearch -u https://target/ --cookie "session=abc" -e php

# multiple targets from a file
dirsearch -l targets.txt -e php,html --threads 20

# raw request from Burp
dirsearch --raw request.txt --scheme https
```

## Flag reference

### Mandatory / targets
| Flag | Meaning |
|------|---------|
| `-u, --url <url>` | Target URL (repeatable) |
| `-l, --url-file <path>` | File of URLs |
| `--stdin` | Read URLs from stdin |
| `--cidr <cidr>` | Target a CIDR range |
| `--raw <path>` | Load a raw HTTP request |
| `-s, --session <file>` | Session file |
| `--config <path>` | Config file |

### Dictionary
| Flag | Meaning |
|------|---------|
| `-w, --wordlists <list>` | Comma-separated wordlists |
| `-e, --extensions <list>` | Extensions (`php,asp,...`) |
| `-f, --force-extensions` | Append extensions to every entry |
| `-O, --overwrite-extensions` | Replace existing extensions |
| `--exclude-extensions <list>` | Skip these extensions |
| `--remove-extensions` | Strip extensions from entries |
| `--prefixes <list>` / `--suffixes <list>` | Add affixes |
| `-U`, `-L`, `-C` | Uppercase / lowercase / capitalise |

### Scan behaviour
| Flag | Meaning |
|------|---------|
| `-t, --threads <n>` | Threads |
| `-r, --recursive` | Recursive brute force |
| `--deep-recursive` | Recurse at every depth |
| `--force-recursive` | Recurse into all found paths |
| `-R, --max-recursion-depth <n>` | Depth cap |
| `--recursion-status <codes>` | Codes that trigger recursion |
| `--subdirs <list>` / `--exclude-subdirs <list>` | Scope recursion |
| `-i, --include-status <codes>` | Include only these codes |
| `-x, --exclude-status <codes>` | Exclude these codes |
| `--exclude-sizes <sizes>` | Exclude by size (`0B,4KB`) |
| `--exclude-text <text>` | Exclude by body text |
| `--exclude-regex <regex>` | Exclude by body regex |
| `--exclude-redirect <str>` | Exclude by redirect target |
| `--exclude-response <path>` | Exclude responses like this page |
| `--skip-on-status <codes>` | Skip the target on these codes |
| `--min-response-size <n>` / `--max-response-size <n>` | Size bounds |
| `--max-time <sec>` | Runtime cap |
| `--exit-on-error` | Abort on error |

### Request / connection
| Flag | Meaning |
|------|---------|
| `-m, --http-method <method>` | HTTP method |
| `-d, --data <data>` / `--data-file <path>` | Request body |
| `-H, --header <h>` / `--header-file <path>` | Headers |
| `-F, --follow-redirects` | Follow redirects |
| `--random-agent` / `--user-agent <ua>` | User-Agent |
| `--auth <cred>` / `--auth-type <type>` | Authentication |
| `--cert-file` / `--key-file` | Client certificate |
| `--cookie <cookie>` | Cookies |
| `--timeout <sec>` / `--delay <sec>` | Timing |
| `--proxy <url>` / `--proxy-file <path>` / `--proxy-auth` | Proxies |
| `--replay-proxy <url>` | Replay findings through a proxy |
| `--tor` | Use Tor |
| `--scheme <scheme>` | Scheme for raw requests |
| `--max-rate <rps>` | Rate limit |
| `--retries <n>` | Retries |
| `--ip <ip>` | Connect to a fixed IP |

### Advanced / output
| Flag | Meaning |
|------|---------|
| `--crawl` | Crawl responses for new paths |
| `--full-url` | Print full URLs |
| `--redirects-history` | Show redirect chains |
| `--no-color` | No colour |
| `-q, --quiet-mode` | Quiet |
| `-o, --output <path>` | Output file |
| `--format <fmt>` | `simple, plain, json, xml, md, csv, html, sqlite` |
| `--log <path>` | Log file |

## Recipes

```bash
# 1. Quick default scan with common extensions
dirsearch -u https://target/ -e php,html,js,txt --exclude-status 404

# 2. Thorough scan with recursion
dirsearch -u https://target/ -e php,asp,aspx,jsp -r -R 3 --threads 20

# 3. Filter wildcard responses by size and text
dirsearch -u https://target/ --exclude-sizes 0B,1234B --exclude-text "Not Found"

# 4. Exclude responses identical to the 404 page
dirsearch -u https://target/ --exclude-response 404.html

# 5. Authenticated scan with a bearer token
dirsearch -u https://target/api/ -H "Authorization: Bearer <token>" -e json

# 6. Raw request replay
dirsearch --raw burp-request.txt --scheme https -e php

# 7. Multi-target sweep at a safe rate
dirsearch -l targets.txt --max-rate 20 --threads 10 --format json -o scan.json

# 8. Backup-file hunt
dirsearch -u https://target/ -e bak,old,orig,save,swp,zip,tar,gz \
      --exclude-status 404

# 9. HTML report for evidence
dirsearch -u https://target/ -e php --format html -o report.html

# 10. Crawl plus brute force
dirsearch -u https://target/ --crawl -e php -r -R 2
```

## Output & pipelines

- Console table: status, size, redirect, path.
- `--format json|xml|csv|html|sqlite` for machine-readable evidence.
- `--replay-proxy` sends found URLs to Burp/ZAP for manual review.
- Feed found paths into `httpx`, `nuclei`, `x8`, `paramspider`.

## Gotchas & operational security

- Wildcard 200 responses can swamp results; always sample a random path and
  use `--exclude-sizes`, `--exclude-text`, or `--exclude-response`.
- High thread counts trip WAFs and can take down fragile apps.
- `--force-extensions` multiplies request count by the extension list.
- Recursion plus `--force-recursive` can explode; cap with `-R`.
- Packaged wordlists may be small; point `-w` at SecLists for coverage.
- On Python 3.13 the `pkg_resources` warning is harmless but noisy.
- Raw request files may contain stale Host headers; verify with `--scheme`.

## Legal & safety

Brute forcing is noisy and can affect availability. Test only systems you are
authorized to assess, honour rate limits, and coordinate timing with the
target's operators.

## Related tools

- [`wfuzz/`](../wfuzz/README.md) — general fuzzer
- [`x8/`](../x8/README.md), [`paramspider/`](../paramspider/README.md) — parameters
- [`cmseek/`](../cmseek/README.md), [`joomscan/`](../joomscan/README.md) — CMS-specific
- [`linkfinder/`](../linkfinder/README.md), [`secretfinder/`](../secretfinder/README.md) — JS analysis

## References

- https://github.com/maurosoria/dirsearch
- https://github.com/maurosoria/dirsearch/wiki
