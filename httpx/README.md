# httpx — HTTP probing toolkit

- **Version:** v1.12.0
- **Binary:** `/usr/local/bin/httpx`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/projectdiscovery/httpx
- **Docs:** https://docs.projectdiscovery.io/tools/httpx/overview

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

httpx takes hosts, URLs, CIDRs, or raw requests and probes them over HTTP/S.
It reports status codes, titles, tech stack, TLS/JARM fingerprints, response
hashes, CDN/WAF, screenshots, and more. It is the standard "which of these
subdomains are actually alive and interesting?" stage.

## Use cases

| Situation | Flags to reach for |
|-----------|--------------------|
| Filter dead subdomains | `-sc -title -td` |
| Technology fingerprinting | `-td -cpe -server` |
| Find admin/login panels | `-mc 200 -ms admin -title` |
| Screenshot a large asset list | `-ss` |
| Detect CDN/WAF to deprioritize | `-cdn` (default true) |
| Feed crawlers/scanners | `-silent -o live.txt` |
| Parse Burp exports | `-im burp` |
| Vhost discovery | `-vhost` |
| TLS certificate recon | `-tls-grab -json` |

## Basic syntax

```bash
httpx -u https://example.com
httpx -l hosts.txt
cat hosts.txt | httpx
```

## Flag reference

### Input
| Flag | Meaning |
|------|---------|
| `-u, -target string[]` | Target host(s)/URL(s) |
| `-l, -list string` | Input file |
| `-rr, -request string` | Raw HTTP request file |
| `-im, -input-mode string` | Input mode: `burp` for Burp XML export |

### Probes (what to display/collect)
| Flag | Meaning |
|------|---------|
| `-sc, -status-code` | HTTP status code |
| `-cl, -content-length` | Body length |
| `-ct, -content-type` | Content-Type |
| `-location` | Redirect target |
| `-favicon` | mmh3 favicon hash |
| `-hash string` | Body hash: md5, mmh3, simhash, sha1, sha256, sha512 |
| `-jarm` | JARM TLS fingerprint |
| `-rt, -response-time` | Response time |
| `-lc / -wc` | Line / word count |
| `-title` | HTML title |
| `-bp, -body-preview` | First N body chars (default 100) |
| `-server, -web-server` | Server header |
| `-td, -tech-detect` | Wappalyzer-style tech detection |
| `-cff string` | Custom fingerprint file |
| `-cpe` | CPE product/version |
| `-wp, -wordpress` | WordPress plugins/themes |
| `-kb, -knowledge-base` | Knowledge-base classification |
| `-method` | HTTP method used |
| `-ws, -websocket` | WebSocket support |
| `-ip`, `-cname`, `-asn` | Network metadata |
| `-cdn` | CDN/WAF detection (default true) |
| `-probe` | Probe status |
| `-efqdn` | Extract FQDNs from response |

### Headless / screenshots
| Flag | Meaning |
|------|---------|
| `-ss, -screenshot` | Save page screenshots |
| `-system-chrome` | Use local Chrome |
| `-ho string[]` | Extra Chrome flags |
| `-esb` | Exclude screenshot bytes from JSON |
| `-ehb` | Exclude headless body from JSON |
| `-no-screenshot-full-page` | Viewport-only screenshot |
| `-st value` | Screenshot timeout (default 10s) |
| `-sid value` | Idle time before shot (default 1s) |
| `-jsc string[]` | JavaScript to execute after navigation |

### Matchers (keep only matches)
`-mc` status codes, `-ml` length, `-mlc` line count, `-mwc` word count,
`-mfc` favicon hash, `-ms` substring, `-mr` regex, `-mcdn` CDN provider,
`-mrt` response-time expression (`'< 1'`), `-mdc` DSL condition.

### Extractors
| Flag | Meaning |
|------|---------|
| `-er, -extract-regex string[]` | Print regex matches from response |
| `-ep, -extract-preset string[]` | Presets: `url`, `ipv4`, `mail` |

### Filters (drop matches)
`-fc` codes, `-fpt` page type (`login,captcha,parked`), `-fd` near-duplicates,
`-fl` length, `-flc` lines, `-fwc` words, `-ffc` favicon, `-fs` substring,
`-fe` regex, `-fcdn` CDN, `-frt` response time, `-fdc` DSL, `-strip` tags,
`-lof` list output fields, `-eof` exclude output fields.

### Rate limiting
| Flag | Meaning |
|------|---------|
| `-t, -threads int` | Workers (default 50) |
| `-rl, -rate-limit int` | Requests/second (default 150) |
| `-rlm int` | Requests/minute |
| `-delay value` | Fixed delay between requests |

### Miscellaneous
| Flag | Meaning |
|------|---------|
| `-pa, -probe-all-ips` | Probe every IP for a hostname |
| `-p string[]` | Ports, nmap syntax, e.g. `http:80,8080-8090,https:443` |
| `-path string` | Paths to probe (comma/file) |
| `-tls-probe` | Probe TLS SAN domains |
| `-csp-probe` | Probe CSP domains |
| `-tls-grab` | Dump TLS data |
| `-pipeline` | HTTP/1.1 pipelining support |
| `-http2` | HTTP/2 support |
| `-vhost` | Vhost support |
| `-ldv` | List DSL variables |

### Output
| Flag | Meaning |
|------|---------|
| `-o, -output string` | Output file |
| `-oa, -output-all` | Write all formats |
| `-sr` / `-srd` | Store responses / to dir |
| `-ob, -omit-body` | Omit body |
| `-csv`, `-j` (JSONL), `-md` (Markdown) | Formats |
| `-irh`, `-irr`, `-irrb` | Include headers / full req-resp / base64 |
| `-include-chain`, `-store-chain` | Redirect chain in JSON / stored |
| `-pr, -protocol string` | `http11`, `http2`, `http3` |
| `-rdb*` | Store in MongoDB/Postgres/MySQL |

### Configuration / debugging
`-config`, `-r/-resolvers`, `-allow`, `-deny`, `-sni`, `-random-agent` (default),
`-auto-referer`, `-H` headers, `-proxy`/`-http-proxy`, `-unsafe`, `-resume`,
`-fr` follow redirects, `-maxr`, `-fhr`, `-rhsts`, `-vhost-input`, `-x` methods,
`-body`, `-s` stream, `-sd` skip dedupe, `-ldp`, `-no-decode`, `-tlsi` TLS
impersonation, `-no-stdin`, `-debug`, `-debug-req`, `-debug-resp`, `-stats`,
`-silent`, `-v`, `-nc`, `-tr`, `-health-check`.

### Optimization
`-nf` no fallback, `-nfs` no fallback scheme, `-maxhr` max errors/host,
`-e` exclude (`cdn`, `private-ips`, CIDR, IP, regex), `-retries`, `-timeout`
(default 10), `-rsts`, `-rstr`.

## Recipes

```bash
# Live hosts with status, title, tech
httpx -l subs.txt -sc -title -td -silent -o live.txt

# Only 200s, show server + content type
httpx -l subs.txt -mc 200 -server -ct

# Find login/admin panels
httpx -l subs.txt -mc 200,401,403 -ms login -ms admin -title

# Screenshot every live host
httpx -l subs.txt -ss -system-chrome -silent -o shots.txt

# Probe common web ports on a CIDR
httpx -u 10.0.0.0/24 -p http:80,443,8000,8080,8443 -sc -title

# Follow redirects and show final URL
httpx -l subs.txt -fr -location -sc -title

# JSONL with full request/response for evidence
httpx -l subs.txt -json -irr -o results.jsonl

# Filter CDN-fronted hosts out
httpx -l subs.txt -sc -title -e cdn

# Feed URLs straight into katana
httpx -l subs.txt -mc 200 -silent | katana -silent
```

## Output formats

- Default: `url [status] [title] [tech]` style text.
- `-json` / `-jsonl`: one JSON object per line (best for automation).
- `-csv`, `-md`: tabular.
- Stored responses (`-sr`) keep raw request/response files for replay in Burp.

## Gotchas

- Default is HTTPS-first with HTTP fallback; use `-nf` to see both.
- `-ss` needs a Chrome/Chromium; `-system-chrome` uses the installed browser.
- `-td` fingerprints are heuristics, verify manually.
- Rate limits matter on shared IPs; use `-rl` and `-t` conservatively.
- `-mc`/`-fc` match on the probed response, not post-redirect unless `-fr`.
- `-e private-ips` is useful when scanning from a VPS to avoid metadata endpoints.

## References

- Overview: https://docs.projectdiscovery.io/tools/httpx/overview
- Usage: https://docs.projectdiscovery.io/tools/httpx/usage
- Source: https://github.com/projectdiscovery/httpx
