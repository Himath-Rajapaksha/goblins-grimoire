# katana — web crawler & endpoint extractor

- **Version:** v1.8.0
- **Binary:** `/usr/local/bin/katana`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/projectdiscovery/katana
- **Docs:** https://docs.projectdiscovery.io/tools/katana/overview

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Katana crawls web applications and extracts URLs, endpoints, forms, JS-derived
paths, and optionally XHR requests. It supports classic HTTP crawling and
headless-browser crawling for JavaScript-heavy SPAs. Output feeds `nuclei`,
`dalfox`, `arjun`, and manual testing.

## Use cases

| Situation | Flags |
|-----------|-------|
| Classic site crawl | `-d 3` |
| SPA/React/Angular app | `-hl` or `-hh` |
| Find endpoints inside JS | `-jc -jsl` |
| Extract forms/inputs | `-fx` |
| Extract XHR calls (headless) | `-hl -xhr` |
| Scope control | `-cs`, `-cos`, `-fs` |
| Known files (robots/sitemap) | `-kf all` |
| Authenticated crawl | `-H "Cookie: ..."` or `-al user:pass` |
| Replay a recorded flow | `-rf flow.json` |

## Basic syntax

```bash
katana -u https://example.com
katana -list live.txt -d 3 -silent -o urls.txt
```

## Flag reference

### Input
| Flag | Meaning |
|------|---------|
| `-u, -list string[]` | Target URL or file of URLs |
| `-resume string` | Resume from state file |
| `-e, -exclude string[]` | Exclude `cdn`, `private-ips`, CIDR, IP, regex |

### Crawl configuration
| Flag | Meaning |
|------|---------|
| `-d, -depth int` | Max depth (default 3) |
| `-jc, -js-crawl` | Parse/crawl endpoints in JS files |
| `-jsl, -jsluice` | jsluice parsing (memory-heavy) |
| `-ct, -crawl-duration` | Max crawl time (`30s`, `10m`, `1h`) |
| `-kf, -known-files` | Crawl `all`, `robotstxt`, `sitemapxml` |
| `-mrs, -max-response-size` | Max response bytes (default 4 MB) |
| `-timeout` | Request timeout seconds (default 10) |
| `-time-stable` | Wait for page stability |
| `-aff` | Automatic form filling (experimental) |
| `-fx, -form-extraction` | Emit form/input/textarea/select JSONL |
| `-retry` | Retries per request |
| `-proxy` | HTTP/SOCKS5 proxy |
| `-td, -tech-detect` | Tech detection |
| `-H, -headers` | Custom headers/cookies |
| `-config` | Config file |
| `-fc`, `-flc` | Form / field config files |
| `-s, -strategy` | `depth-first` (default) or `breadth-first` |
| `-iqp` | Ignore query-param variants of same path |
| `-fsu`, `-fst` | Filter similar URLs / threshold |
| `-tlsi` | TLS client-hello randomization |
| `-dr` | Disable redirects |
| `-pc, -path-climb` | Crawl parent paths |
| `-kb`, `-kb-secrets`, `-kb-validate-secrets`, `-kb-endpoints` | Knowledge-base classifiers |
| `-mdp` | Max pages per domain |

### Headless
| Flag | Meaning |
|------|---------|
| `-hl, -headless` | Headless crawling |
| `-hh, -hybrid` | Standard + headless |
| `-sc, -system-chrome` | Use installed Chrome |
| `-sb` | Show browser window |
| `-ho` | Extra Chrome options |
| `-nos` | `--no-sandbox` |
| `-cdd` | Chrome data dir |
| `-scp` | Chrome binary path |
| `-noi` | Disable incognito |
| `-cwu` | Attach to remote Chrome WS |
| `-xhr` | Extract XHR method/URL in JSONL |
| `-mfc` | Max consecutive action failures |
| `-ed` | Diagnostics |
| `-pls` | Page-load strategy: heuristic/load/domcontentloaded/networkidle/none |
| `-dwt` | DOM wait after load |
| `-csp`, `-csk` | Captcha-solver provider/key |
| `-al` | Auto-login `user:pass` |
| `-rf` | Replay DevTools Recorder JSON |

### Scope & filters
| Flag | Meaning |
|------|---------|
| `-cs` / `-cos` | In-scope / out-of-scope URL regex |
| `-fs` | Scope field: `dn`, `rdn`, `fqdn`, or regex (default `rdn`) |
| `-ns` | Disable host-based scope |
| `-do` | Display out-of-scope endpoints |
| `-mr` / `-fr` | Match / filter output URL regex |
| `-f` | Output field (deprecated; use `-ot`) |
| `-sf` | Store field per host |
| `-em` / `-ef` | Extension match / filter |
| `-ndef` | Keep default extension filter |
| `-mdc` / `-fdc` | DSL match / filter |
| `-duf` | Disable duplicate-content filter |
| `-pcs` | Page-content similarity filter |
| `-pcsm` | `simhash`, `tfidf`, `bm25` |
| `-pcsd`, `-pcst`, `-pcsn` | Similarity tuning |
| `-fpt` | Filter page types (`error,captcha,parked`) |

### Rate limits
| Flag | Meaning |
|------|---------|
| `-c, -concurrency` | Concurrent fetchers (default 10) |
| `-p, -parallelism` | Concurrent inputs (default 10) |
| `-rd, -delay` | Delay between requests (seconds) |
| `-rl` / `-rlm` | Global rate/minute |
| `-hrl` / `-hrlm` | Per-host rate/minute |

### Output
| Flag | Meaning |
|------|---------|
| `-o, -output` | Output file |
| `-ot, -output-template` | Custom output template |
| `-sr` / `-srd` | Store request/response / to dir |
| `-ncb` | No clobber |
| `-sfd` | Per-host field dir |
| `-or`, `-ob` | Omit raw / body from JSONL |
| `-lof`, `-eof` | List / exclude output fields |
| `-j, -jsonl` | JSONL output |
| `-nc` | No color |
| `-silent`, `-v`, `-debug`, `-version` | Logging/version |

## Recipes

```bash
# Basic crawl
katana -u https://example.com -d 3 -silent -o urls.txt

# Crawl a live-host list from httpx
httpx -l subs.txt -mc 200 -silent | katana -silent -o urls.txt

# JS-aware crawl, include known files
katana -u https://app.example.com -jc -jsl -kf all -d 4 -silent

# Headless crawl of an SPA + XHR extraction
katana -u https://app.example.com -hl -xhr -timeout 15 -silent -jsonl -o app.jsonl

# Form discovery
katana -u https://example.com -fx -jsonl -o forms.jsonl

# Authenticated crawl
katana -u https://app.example.com -H "Cookie: session=ABC" -hl -silent

# Scoped crawl with out-of-scope reporting
katana -u https://example.com -fs rdn -cos 'logout|delete' -do -silent

# Crawl from Burp-exported URLs
katana -list burp_urls.txt -jc -d 2 -silent
```

## Output

Default: one URL per line. `-jsonl` adds request/response metadata, form fields,
XHR details, and source. `-store-response` writes replayable request/response
pairs for Burp.

## Gotchas

- Headless crawling is much slower and heavier; scope it.
- `-jsl` can blow memory on huge JS bundles.
- Depth 3 is usually enough; deeper explodes output size.
- Default scope is registrable domain (`rdn`), so subdomains may be excluded.
- Always rate-limit production targets; use `-rl`/`-hrl`.
- `-kf` needs `-d 3+` to reach all known files.

## References

- Overview: https://docs.projectdiscovery.io/tools/katana/overview
- Usage: https://docs.projectdiscovery.io/tools/katana/usage
- Field extraction deep dive: https://blog.projectdiscovery.io/a-deep-dive-on-katana-field-extraction/
- Source: https://github.com/projectdiscovery/katana
