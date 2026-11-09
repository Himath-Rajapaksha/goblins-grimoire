# LinkFinder — endpoint discovery in JavaScript

- **Version:** packaged LinkFinder (Python 3 port)
- **Binary:** `/usr/local/bin/linkfinder` (wrapper)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/GerbenJavado/LinkFinder

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

LinkFinder parses JavaScript files for URLs, relative paths, and API endpoints.
It accepts a single JS URL, an HTML page (with `-d` it recursively parses all
scripts on the page), a local file, or a folder of files. Results can be
filtered by regex and written as HTML or plain text.

## Use cases

| Situation | Why LinkFinder |
|-----------|----------------|
| API discovery | Extract hidden endpoints from SPA bundles |
| Attack-surface mapping | Find routes not linked in the UI |
| JS review | Collect URLs before manual analysis |
| Recursive crawling | Parse all scripts on a page |
| Bug bounty recon | Turn minified bundles into endpoint lists |

## Requirements & install layout

- Python 3 script installed with a wrapper at `/usr/local/bin/linkfinder`.
- No API keys.
- Works on remote URLs, local files, and folders (wildcards supported).
- `-d` requires an HTML page to discover script sources.

## Basic syntax

```bash
# parse one JS file
linkfinder -i https://target/app.js

# recursively parse all JS on a page
linkfinder -d -i https://target/

# local file or folder
linkfinder -i bundle.js
linkfinder -i '/path/js/*.js'

# filter to API-ish paths and write HTML
linkfinder -i https://target/app.js -r '^/api/' -o endpoints.html
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-i, --input <url\\|file\\|folder>` | Input (URL, file, or folder/wildcard) |
| `-d, --domain` | Treat input as a page and recursively parse its JS |
| `-o, --output <file>` | Output file (default `output.html`) |
| `-r, --regex <regex>` | Filter results by regex (e.g. `^/api/`) |
| `-b, --burp` | Burp-compatible output |
| `-c, --cookies <cookie>` | Cookies for authenticated JS |
| `-t, --timeout <sec>` | Request timeout (default 10) |
| `-h, --help` | Help |

## Recipes

```bash
# 1. Single bundle
linkfinder -i https://target/static/app.js

# 2. All scripts on a page
linkfinder -d -i https://target/

# 3. API paths only
linkfinder -d -i https://target/ -r '^/api/' -o api.html

# 4. Authenticated bundle
linkfinder -i https://target/app.js -c 'session=abc' -o app.html

# 5. Local folder of downloaded JS
linkfinder -i '/evidence/js/*.js' -o local.html

# 6. Plain-text endpoint extraction from HTML output
grep -Eo 'https?://[^"<> ]+|/[a-zA-Z0-9_./-]+' output.html | sort -u > endpoints.txt

# 7. Feed endpoints into a scanner
cat endpoints.txt | httpx -silent -mc 200,301,302,401,403

# 8. Burp-friendly output
linkfinder -d -i https://target/ -b -o burp.html
```

## Output & pipelines

- Default HTML report groups findings by source file.
- Regex filtering (`-r`) keeps only matching endpoints.
- Endpoint lists feed `httpx`, `nuclei`, `ffuf`, `x8`, and manual review.
- Local-folder mode is useful after downloading bundles with `wget -r` or a
  browser's save-all.

## Gotchas & operational security

- Minified/obfuscated bundles hide endpoints in string concatenation; static
  parsing misses dynamically built URLs.
- Source maps (`.map`) expose far more; check for them separately.
- `-d` only finds `<script src>` references; inline JS needs a direct input.
- Authenticated bundles require a valid cookie; expired sessions yield login
  pages.
- Output HTML can be large; prefer `-r` filters and text extraction.
- Regex-only filtering can drop relative endpoints without a leading slash.
- The tool is passive against the target but reveals your IP to the CDN.

## Legal & safety

Parsing publicly served JavaScript is low-impact, but extracting and using
endpoints still requires authorization. Respect terms of service and do not
retrieve files you are not permitted to access.

## Related tools

- [`secretfinder/`](../secretfinder/README.md) — secrets in JS
- [`subjs/`](../subjs/README.md) — collect JS URLs at scale
- [`sourcemapper/`](../sourcemapper/README.md) — source-map reconstruction
- [`x8/`](../x8/README.md), [`paramspider/`](../paramspider/README.md) — parameters
- [`dirsearch/`](../dirsearch/README.md) — path discovery

## References

- https://github.com/GerbenJavado/LinkFinder
