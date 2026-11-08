# dalfox — XSS scanner

- **Version:** v3.3.0
- **Binary:** `/usr/local/bin/dalfox`
- **Full help snapshots:** [`help.txt`](help.txt), [`help-scan.txt`](help-scan.txt), [`help-payload.txt`](help-payload.txt)
- **Upstream:** https://github.com/hahwul/dalfox
- **Docs:** https://dalfox.hahwul.com/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Dalfox scans for reflected, stored, and DOM-based XSS. It parses parameters from
URLs, bodies, JSON, headers, cookies, and raw requests; discovers hidden
parameters; injects payloads with encoders; verifies execution where possible;
performs AST-based DOM source→sink analysis; and can use interactsh for blind
XSS.

## Use cases

| Situation | Flags |
|-----------|-------|
| Single URL scan | `dalfox scan https://target/?q=1` |
| Bulk URLs | `dalfox file urls.txt` |
| Raw HTTP request from Burp | `-i raw-http --data @req.txt` |
| JSON API body | `-m JSON -d '{"q":"1"}'` |
| Parameter discovery only | `--only-discovery` |
| Blind XSS | `-b https://callback` or `--blind-oob` |
| Stored XSS | `--sxss --sxss-url https://target/list` |
| WAF evasion | `--waf-bypass auto --waf-evasion` |
| CI/JSON output | `-f jsonl -o out.jsonl` |
| Resume a mass scan | `--state-file scan.state` |

## Basic syntax

```bash
dalfox scan https://example.com/search?q=test
dalfox https://example.com/search?q=test      # scan is default
dalfox file urls.txt
cat urls.txt | dalfox pipe
```

## Commands

| Command | Purpose |
|---------|---------|
| `scan` | Scan targets for XSS (default) |
| `server` | REST API / server mode |
| `payload` | List or manage built-in payloads |
| `mcp` | MCP stdio server exposing dalfox tools |
| `completion` | Shell completions |
| `help` | Help |

## Scan flag reference

### Input
| Flag | Meaning |
|------|---------|
| `-i, --input-type` | `auto,url,file,pipe,raw-http,har,openapi,postman` |
| `--base-url` | Base URL for OpenAPI/Postman specs |
| `--dedup-urls` | `exact`, `signature`, `off` |
| `--state-file` | Resume file for interrupted scans |
| `[TARGET]...` | URLs or files |

### Output
| Flag | Meaning |
|------|---------|
| `-f, --format` | `plain,json,jsonl,markdown,sarif,toml` |
| `-o, --output` | Output file |
| `--include-request/response/all` | Attach HTTP evidence |
| `-S, --silence` | Only POC output |
| `--poc-type` | `plain,curl,httpie,http-request` |
| `--limit`, `--limit-result-type` | Cap/filter results |
| `--only-poc` | Show only `v,r,a,i` finding types |
| `--baseline` | Diff against previous JSON report |
| `--baseline-mode` | `filter` or `annotate` |
| `--min-confidence` | `low` or `high` |
| `--stream-findings` | Emit findings as verified |
| `--dry-run` | Parse/discover only, no payloads |
| `--no-color` | Disable colors |

### Targets / request shaping
| Flag | Meaning |
|------|---------|
| `-p, --param` | Parameter + type: `id:query`, `q:body`, `token:header`, ... |
| `-d, --data` | Request body |
| `-H, --headers` | Header (repeatable) |
| `--cookies` | Cookie (repeatable) |
| `-X, --method` | HTTP method (default GET) |
| `--user-agent` | Custom UA |
| `--cookie-from-raw` | Load cookies from raw request |

### Scope
`--include-url` / `--exclude-url` regex, `--ignore-param`,
`--out-of-scope` / `--out-of-scope-file` (wildcards).

### Parameter discovery & mining
| Flag | Meaning |
|------|---------|
| `--only-discovery` | Discovery only |
| `--skip-discovery` | No discovery |
| `--skip-reflection-header/cookie/path` | Skip those reflection checks |
| `-W, --mining-dict-word` | Wordlist-based mining |
| `--remote-wordlists` | `burp`, `assetnote` |
| `--skip-mining`, `--skip-mining-dict`, `--skip-mining-dom` | Skip stages |

### Session handling
`--session-check` regex, `--session-check-url`, `--on-session-loss abort|continue`.

### Network
| Flag | Meaning |
|------|---------|
| `--timeout` | Per-request seconds (default 10) |
| `--scan-timeout` | Wall-clock cap for injection stage (0 = off) |
| `--delay` | Milliseconds between requests |
| `-r, --rate-limit` | Global req/s |
| `--retries`, `--retry-delay` | Retry tuning (429 always retried) |
| `--proxy` | HTTP/SOCKS proxy |
| `-F, --follow-redirects` | Follow redirects |
| `--ignore-return` | Ignore status codes |
| `--insecure` | Skip TLS verification (default on) |

### Engine
`--workers` (default 50), `--max-concurrent-targets` (50),
`--max-targets-per-host` (100).

### XSS engine
| Flag | Meaning |
|------|---------|
| `-e, --encoders` | `none,url,2url,3url,4url,html,htmlpad,base64,unicode,zwsp` |
| `--remote-payloads` | `portswigger`, `payloadbox` |
| `--custom-payload` | Custom payload file |
| `--only-custom-payload` | Only custom payloads |
| `-b, --blind` | Blind XSS callback URL |
| `--blind-oob` | interact.sh blind XSS |
| `--custom-blind-xss-payload` | Custom blind payloads |
| `--inject-marker` | Marker string replaced with payloads (e.g. `FUZZ`) |
| `--deep-scan` | Test all payloads even after a hit |
| `--sxss`, `--sxss-url`, `--sxss-method`, `--sxss-retries` | Stored XSS |
| `--skip-ast-analysis` | Skip DOM XSS AST checks |
| `--analyze-external-js` | Analyze same-origin external JS |
| `--hpp` | HTTP parameter pollution bypass |
| `--detect-outdated-libs` | Retire.js-style informational check |
| `--skip-xss-scanning` | Skip XSS entirely (discovery only) |

### WAF
| Flag | Meaning |
|------|---------|
| `--waf-bypass` | `auto` (default), `force`, `off` |
| `--skip-waf-probe` | Detect only from preflight |
| `--force-waf` | e.g. `cloudflare`, `akamai`, `modsecurity` |
| `--waf-evasion` | Stealthy pacing + cooldowns |
| `--waf-min-confidence` | Fingerprint threshold 0.0–1.0 (default 0.3) |

## Finding types

| Code | Meaning |
|------|---------|
| `V` | Verified/executable XSS |
| `R` | Reflected (payload appears, execution not proven) |
| `A` | AST DOM XSS |
| `I` | Informational |

## Recipes

```bash
# Single URL
dalfox scan 'https://target/search?q=test'

# Bulk URLs from katana/httpx
dalfox file urls.txt --silence --format jsonl -o xss.jsonl

# POST JSON API
dalfox scan https://api.target/v1/search -m JSON -d '{"q":"test"}' -H 'Content-Type: application/json'

# Raw request captured in Burp
dalfox scan -i raw-http --data @request.txt --silence

# Blind XSS with your own callback
dalfox file urls.txt -b https://your-callback.example --silence

# Blind XSS via interactsh
dalfox file urls.txt --blind-oob --blind-oob-wait 60 --silence

# Deep scan, high confidence only
dalfox file urls.txt --deep-scan --min-confidence high -f json -o xss.json

# WAF evasion
dalfox file urls.txt --waf-bypass auto --waf-evasion --rate-limit 10

# Parameter discovery only (no payloads)
dalfox file urls.txt --only-discovery --format jsonl

# Through Burp for manual verification
dalfox file urls.txt --proxy http://127.0.0.1:8080

# Resume a mass scan
dalfox file urls.txt --state-file scan.state --silence
```

## Payload command

```bash
dalfox payload list                 # list payload categories
dalfox payload search alert         # search payloads
dalfox payload build              # show payload build info
```

(Exact subcommands may vary; run `dalfox payload -h`.)

## Server mode

```bash
dalfox server --port 6664
# then POST scan jobs to the REST API; see docs/server-mode
```

## Gotchas

- `R` findings are reflections, not proof of execution — verify manually.
- Discovery/mining can be noisy; use `--skip-discovery` when you already have params.
- High concurrency + WAF = bans; use `--waf-evasion` and low `--rate-limit`.
- Stored XSS needs an authenticated/browser flow and a check URL.
- `--deep-scan` can multiply request volume substantially.
- `--insecure` is on by default; disable with `--insecure=false`.
- Some targets reflect everything (canonical/self links); `--min-confidence high` helps.

## References

- Docs: https://dalfox.hahwul.com/
- Source: https://github.com/hahwul/dalfox
- Releases/changelog: https://github.com/hahwul/dalfox/releases
