# arjun — HTTP parameter discovery

- **Version:** 2.2.1 (Arjun)
- **Binary:** `/usr/bin/arjun` (restricted: `root:sectools 750` via dpkg-statoverride)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/s0md3v/Arjun
- **Docs:** https://github.com/s0md3v/Arjun/wiki

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Arjun brute-forces hidden HTTP parameters by sending chunks of candidate names
and comparing responses against a baseline. Hidden parameters often expose
undocumented API behavior, debug flags, mass-assignment fields, redirects, and
injection points.

## Use cases

| Situation | Why |
|-----------|-----|
| API endpoint looks too simple | Find undocumented query/body fields |
| Before XSS/SQLi/SSRF testing | Expand the attack surface |
| Bug bounty | Discover `?debug=`, `?admin=`, `?redirect=` |
| POST/JSON endpoints | Test body parameters, not just query |
| Passive recon | Pull parameter names from Wayback/CommonCrawl/OTX |

## Basic syntax

```bash
arjun -u https://example.com/page
arjun -u https://example.com/api -m POST
arjun -i urls.txt -oT params.txt
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-u URL` | Target URL |
| `-i [IMPORT_FILE]` | Import URLs from file |
| `-o, -oJ JSON_FILE` | JSON output |
| `-oT TEXT_FILE` | Text output |
| `-oB [BURP_PROXY]` | Send discovered requests to Burp (default `127.0.0.1:8080`) |
| `-m METHOD` | `GET`, `POST`, `XML`, `JSON` (default GET) |
| `-d DELAY` | Delay between requests (seconds) |
| `-t THREADS` | Concurrent threads (default 5) |
| `-w WORDLIST` | Candidate wordlist (default `db/large.txt`) |
| `-T TIMEOUT` | Request timeout seconds (default 15) |
| `-c CHUNKS` | Parameters sent per request (chunk size) |
| `-q` | Quiet |
| `--rate-limit` | Requests/second (default 9999) |
| `--headers` | Extra headers, newline-separated |
| `--passive` | Collect names from Wayback/CommonCrawl/OTX |
| `--stable` | Favor stability over speed |
| `--include` | Data included in every request |
| `--disable-redirects` | Don't follow redirects |
| `--casing` | Param naming style: `like_this`, `likeThis`, `likethis` |

## Recipes

```bash
# GET parameter discovery
arjun -u https://example.com/search

# JSON API body parameters
arjun -u https://api.example.com/v1/user -m JSON

# POST form parameters
arjun -u https://example.com/login -m POST

# Many URLs, text output
arjun -i live_urls.txt -oT params.txt -t 10

# Passive + active
arjun -u https://example.com --passive

# Route through Burp for manual review
arjun -u https://example.com -oB 127.0.0.1:8080

# Custom wordlist
arjun -u https://example.com -w ~/wordlists/params.txt

# Rate-limited against a sensitive target
arjun -u https://example.com --rate-limit 5 -t 2 --stable
```

## How it works (practical view)

1. Sends a baseline request.
2. Sends batches (chunks) of candidate parameter names.
3. Compares status, length, body hash, and reflection against baseline.
4. Narrows down which names caused a change.
5. Reports the parameter names (and can replay them through Burp).

Because it relies on response differences, dynamic pages, caching, and WAFs can
produce false positives/negatives. `--stable` and smaller chunks help.

## Gotchas

- Large default wordlist + high threads = loud; reduce on production.
- JSON/XML modes need correct `Content-Type`; add with `--headers`.
- Redirects can mask parameter effects; try `--disable-redirects` when results look odd.
- Parameter names found are not automatically vulnerable — test them.
- The binary is `750 root:sectools`; you must be in `sectools` (or root) to run it.
- `-oB` opens a Burp proxy connection; Burp must be listening.

## Output

- Default: colored console list of parameters.
- `-oT`: plain text.
- `-oJ`: JSON with URL → parameter list (good for pipelines).

## References

- Source: https://github.com/s0md3v/Arjun
- Wiki: https://github.com/s0md3v/Arjun/wiki
