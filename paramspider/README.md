# ParamSpider — URL and parameter mining from web archives

- **Version:** packaged Parameter Spider (0xasm0d3us fork)
- **Binary:** `/usr/local/bin/paramspider`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/devanshbatham/ParamSpider

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

ParamSpider queries web archives (Wayback Machine, Common Crawl, and similar)
for URLs belonging to a domain, keeps the ones that carry query parameters,
and rewrites parameter values to a placeholder such as `FUZZ`. The output is a
ready-made list of parameterised endpoints for fuzzing, without touching the
target directly.

## Use cases

| Situation | Why ParamSpider |
|-----------|-----------------|
| Passive parameter discovery | Mine archives without target contact |
| Fuzzing prep | Produce `?id=FUZZ` style URLs |
| Forgotten endpoints | Surface old parameters no longer linked |
| Bug bounty recon | Fast, low-noise surface expansion |
| Scope validation | See what the domain historically exposed |

## Requirements & install layout

- Installed via pipx; entry point `/usr/local/bin/paramspider`.
- Python 3; no API keys.
- Writes results under `results/<domain>.txt` in the working directory.
- Archive APIs are rate-limited and occasionally unavailable.

## Basic syntax

```bash
# mine one domain
paramspider -d example.com

# keep URLs streaming to the terminal
paramspider -d example.com -s

# placeholder for values
paramspider -d example.com -p FUZZ

# multiple domains
paramspider -l domains.txt

# route through a proxy
paramspider -d example.com --proxy http://127.0.0.1:8080
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-d, --domain <domain>` | Domain to mine |
| `-l, --list <file>` | File of domains |
| `-s, --stream` | Stream URLs to the terminal |
| `--proxy <url>` | Proxy for archive requests |
| `-p, --placeholder <value>` | Value placeholder (default `FUZZ`) |
| `-h, --help` | Help |

## Recipes

```bash
# 1. Mine and stream
paramspider -d example.com -s

# 2. Save to the default results file
paramspider -d example.com
cat results/example.com.txt

# 3. Custom placeholder for ffuf/wfuzz
paramspider -d example.com -p FUZZ

# 4. Batch domains
paramspider -l domains.txt

# 5. Proxy through Burp
paramspider -d example.com --proxy http://127.0.0.1:8080 -s

# 6. Deduplicate and count
paramspider -d example.com -s | sort -u | tee params.txt | wc -l

# 7. Feed into a fuzzer
paramspider -d example.com -s | sort -u > params.txt
ffuf -w params.txt -u 'https://example.com/FUZZ' -mc 200,301,302

# 8. Filter to a path prefix
grep '/api/' params.txt
```

## Output & pipelines

- Default file: `results/<domain>.txt`.
- Each line is a full URL with the parameter value replaced by the placeholder.
- Feed `ffuf`, `wfuzz`, `x8`, `nuclei`, or manual testing.
- URLs are historical; many are dead — validate before fuzzing.

## Gotchas & operational security

- Archive data is stale; endpoints may 404 or behave differently now.
- The placeholder replacement can mangle multi-parameter URLs if values
  contain encoded separators.
- Duplicate and near-duplicate URLs are common; deduplicate before fuzzing.
- Archive APIs rate-limit; retries and long runs are normal.
- Streaming (`-s`) plus a pipe can hide progress; use `tee`.
- Results may include third-party hosts if the archive indexed them; scope-check.
- No active requests to the target — but fuzzing the output is active.

## Legal & safety

Passive archive mining is low-impact, but the endpoints belong to someone.
Only fuzz targets you are authorized to test, and respect scope boundaries even
for historical URLs.

## Related tools

- [`x8/`](../x8/README.md) — active parameter discovery
- [`wfuzz/`](../wfuzz/README.md), [`dirsearch/`](../dirsearch/README.md)
- [`linkfinder/`](../linkfinder/README.md) — parameters in JS
- [`subjs/`](../subjs/README.md) — JS collection

## References

- https://github.com/devanshbatham/ParamSpider
