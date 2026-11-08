# dalfox — command scenarios for deep research

Every dalfox command form, grouped by scenario. Flags from v3.3.0
(`help.txt`, `help-scan-full.txt`, `help-server.txt`, `help-payload-full.txt`,
`help-completion.txt`).

## How to invoke

```text
@deep-research For dalfox 3.3, how reliable are the AST DOM-XSS findings and
the WAF bypass modes against Cloudflare, and which input modes (HAR/OpenAPI/
Postman) preserve auth? Cite dalfox docs/source, release notes, and current
WAF writeups; return exact flags.
```

---

## Scenario 1 — one target, many input shapes

| # | Command | How to use |
|---|---------|------------|
| 1 | `dalfox scan https://example.com/?q=1` | URL target, default scan |
| 2 | `dalfox https://example.com/?q=1` | Target without subcommand defaults to scan |
| 3 | `dalfox scan -i url 'https://example.com/?q=1'` | Explicit URL input |
| 4 | `dalfox scan -i file targets.txt` | File of URLs |
| 5 | `cat urls.txt \| dalfox scan -i pipe` | STDIN pipe |
| 6 | `dalfox scan -i raw-http request.txt` | Raw HTTP request file |
| 7 | `dalfox scan -i har traffic.har` | HAR export |
| 8 | `dalfox scan -i openapi api.yaml --base-url https://staging.example.com` | OpenAPI with base URL |
| 9 | `dalfox scan -i postman collection.json --base-url https://staging.example.com` | Postman collection |
| 10 | `dalfox scan https://a https://b https://c` | Multiple targets |

**Research questions**
- Which input modes carry cookies/auth automatically, and which need `-H`?
- Does `--base-url` override or prefix the spec's server URL?

**Sources**: dalfox input docs, source parsers, issues on OpenAPI/Postman.

**Output**: one target list per input type with auth handled explicitly.

---

## Scenario 2 — request shaping and auth

| # | Command | How to use |
|---|---------|------------|
| 11 | `dalfox scan https://example.com -p q -p page` | Analyze named parameters |
| 12 | `dalfox scan https://example.com -p id:query -p token:header` | Typed parameters |
| 13 | `dalfox scan https://example.com -d 'q=1&page=2'` | Body data |
| 14 | `dalfox scan https://example.com -X POST -d 'q=1'` | Method override |
| 15 | `dalfox scan https://example.com -H 'Cookie: session=...'` | Cookie header |
| 16 | `dalfox scan https://example.com -H 'Authorization: Bearer ...'` | Bearer auth |
| 17 | `dalfox scan https://example.com --cookies 'session=...'` | Cookie jar form |
| 18 | `dalfox scan https://example.com --cookie-from-raw request.txt` | Cookies from a raw request |
| 19 | `dalfox scan https://example.com --user-agent 'Mozilla/5.0'` | Custom UA |
| 20 | `dalfox scan https://example.com -F` | Follow redirects |
| 21 | `dalfox scan https://example.com --ignore-return 302,403,404` | Ignore status codes |
| 22 | `dalfox scan https://example.com --insecure` | Skip TLS verification (default on) |
| 23 | `dalfox scan https://example.com --insecure=false` | Enforce TLS validation |

**Research questions**
- Parameter type syntax (`id:query`, `token:header`) — full list and defaults?
- Does `--insecure` default really apply, and what does it change?

**Sources**: dalfox scanning docs, source flag parser, issues on auth.

**Output**: a request template that reproduces the authenticated session
without storing the secret in docs.

---

## Scenario 3 — scope, dedupe, resume

| # | Command | How to use |
|---|---------|------------|
| 24 | `dalfox scan -i file targets.txt --include-url '/api/'` | Only matching URLs |
| 25 | `dalfox scan -i file targets.txt --exclude-url '/logout'` | Exclude URLs |
| 26 | `dalfox scan https://example.com --ignore-param csrf` | Skip a parameter |
| 27 | `dalfox scan https://example.com --out-of-scope '*.dev.example.com'` | Wildcard out-of-scope |
| 28 | `dalfox scan -i file targets.txt --out-of-scope-file oos.txt` | Out-of-scope file |
| 29 | `dalfox scan -i file targets.txt --dedup-urls exact` | Drop byte-identical requests |
| 30 | `dalfox scan -i file targets.txt --dedup-urls signature` | Collapse value-only variants |
| 31 | `dalfox scan -i file targets.txt --dedup-urls off` | Scan every line |
| 32 | `dalfox scan -i file targets.txt --state-file scan.state` | Resume interrupted mass scans |
| 33 | `dalfox scan https://example.com --dry-run` | Parse + discover, send no payloads |

**Research questions**
- What exactly does `signature` dedupe key on, and can it drop real variants?
- Does `--state-file` mark a target complete before or after verification?

**Sources**: dalfox docs, release notes for state/dry-run, source.

**Output**: a scope file plus a resumable scan command.

---

## Scenario 4 — parameter discovery and mining

| # | Command | How to use |
|---|---------|------------|
| 34 | `dalfox scan https://example.com --only-discovery` | Discovery only, no payloads |
| 35 | `dalfox scan https://example.com --skip-discovery` | Skip discovery |
| 36 | `dalfox scan https://example.com --skip-reflection-header` | No header reflection checks |
| 37 | `dalfox scan https://example.com --skip-reflection-cookie` | No cookie reflection checks |
| 38 | `dalfox scan https://example.com --skip-reflection-path` | No path reflection checks |
| 39 | `dalfox scan https://example.com -W words.txt` | Dictionary mining |
| 40 | `dalfox scan https://example.com --remote-wordlists burp,assetnote` | Remote wordlists |
| 41 | `dalfox scan https://example.com --skip-mining` | Skip all mining |
| 42 | `dalfox scan https://example.com --skip-mining-dict` | Skip dictionary mining |
| 43 | `dalfox scan https://example.com --skip-mining-dom` | Skip DOM attribute mining |

**Research questions**
- Which discovery checks send requests that could change server state?
- How do burp/assetnote wordlists differ in size and hit rate?

**Sources**: dalfox mining docs, source, issues on discovery false positives.

**Output**: a discovery profile matched to the request budget.

---

## Scenario 5 — payload engineering

| # | Command | How to use |
|---|---------|------------|
| 44 | `dalfox scan https://example.com -e url,html` | Default encoder set |
| 45 | `dalfox scan https://example.com -e none` | Raw payloads |
| 46 | `dalfox scan https://example.com -e url,2url,3url,4url` | Layered URL encoding |
| 47 | `dalfox scan https://example.com -e html,htmlpad,base64,unicode,zwsp` | Mixed encoders |
| 48 | `dalfox scan https://example.com --custom-payload payloads.txt` | Custom payload file |
| 49 | `dalfox scan https://example.com --custom-payload p.txt --only-custom-payload` | Only custom payloads |
| 50 | `dalfox scan https://example.com --remote-payloads portswigger,payloadbox` | Remote payload sets |
| 51 | `dalfox scan https://example.com --custom-alert-value 'document.domain'` | Change alert payload value |
| 52 | `dalfox scan https://example.com --custom-alert-type str` | Quote the alert value |
| 53 | `dalfox scan 'https://example.com/?q=FUZZ' --inject-marker FUZZ` | Explicit injection marker |
| 54 | `dalfox scan https://example.com --max-payloads-per-param 500` | Cap payloads per parameter |
| 55 | `dalfox scan https://example.com --deep-scan` | Test all payloads after a hit |
| 56 | `dalfox scan https://example.com --skip-xss-scanning` | Only discovery/mining |

**Research questions**
- How are encoders combined, and when do layered encodings break a payload?
- What does `--deep-scan` cost on reflection-heavy endpoints?

**Sources**: payload/encoder docs, source payload builder, issues on WAF.

**Output**: a payload budget and an encoder order justified per target.

---

## Scenario 6 — blind, stored, and DOM XSS

| # | Command | How to use |
|---|---------|------------|
| 57 | `dalfox scan https://example.com -b https://callback.example.com` | External blind callback |
| 58 | `dalfox scan https://example.com --blind-oob` | interactsh OOB, public servers |
| 59 | `dalfox scan https://example.com --blind-oob=oast.fun,oast.me` | Pin OAST servers |
| 60 | `dalfox scan https://example.com --blind-oob --blind-oob-secret TOKEN` | Self-hosted interactsh auth |
| 61 | `dalfox scan https://example.com --blind-oob --blind-oob-wait 60` | Longer callback poll |
| 62 | `dalfox scan https://example.com --custom-blind-xss-payload blind.txt` | Custom blind payloads |
| 63 | `dalfox scan https://example.com --sxss` | Stored XSS mode |
| 64 | `dalfox scan https://example.com --sxss-url https://example.com/list` | Check a specific page |
| 65 | `dalfox scan https://example.com --sxss-method POST` | Stored XSS check method |
| 66 | `dalfox scan https://example.com --sxss-retries 5` | Retry count for propagation |
| 67 | `dalfox scan https://example.com --skip-ast-analysis` | Disable `[A]` DOM findings |
| 68 | `dalfox scan https://example.com --analyze-external-js` | Analyze external bundles |
| 69 | `dalfox scan https://example.com --hpp` | HTTP parameter pollution |
| 70 | `dalfox scan https://example.com --detect-outdated-libs` | Informational JS library findings |

**Research questions**
- Which blind callback service is appropriate for client data, and what leaks?
- Does stored-XSS mode write payloads that persist on the target?

**Sources**: blind XSS docs, interactsh docs, issues on stored mode cleanup.

**Output**: a callback plan and a written note on persistence/cleanup.

---

## Scenario 7 — WAF handling

| # | Command | How to use |
|---|---------|------------|
| 71 | `dalfox scan https://example.com --waf-bypass auto` | Detect + bypass (default) |
| 72 | `dalfox scan https://example.com --waf-bypass force --force-waf cloudflare` | Force a WAF profile |
| 73 | `dalfox scan https://example.com --waf-bypass off` | Detect only, no mutations |
| 74 | `dalfox scan https://example.com --skip-waf-probe` | No provocation probe |
| 75 | `dalfox scan https://example.com --waf-evasion` | Stealth pacing |
| 76 | `dalfox scan https://example.com --waf-min-confidence 0.0` | Keep weak fingerprints |

**Research questions**
- Which WAF names does `--force-waf` accept, and how are strategies chosen?
- Does `--waf-evasion` meaningfully change block rates?

**Sources**: WAF docs, source fingerprints, independent WAF comparisons.

**Output**: a WAF mode justified by the target's edge and program rules.

---

## Scenario 8 — session health

| # | Command | How to use |
|---|---------|------------|
| 77 | `dalfox scan https://example.com --cookies 'session=...' --session-check 'Sign out'` | Authoritative session marker |
| 78 | `dalfox scan https://example.com --session-check-url https://example.com/me` | Probe a cheap auth endpoint |
| 79 | `dalfox scan https://example.com --on-session-loss abort` | Stop on lost session |
| 80 | `dalfox scan https://example.com --on-session-loss continue` | Keep scanning |

**Research questions**
- Default heuristics for session loss (401/403, login redirect, login form) —
  how do they false-positive?
- Is a session-lost target reported as incomplete in JSON output?

**Sources**: session docs, source heuristics, issues.

**Output**: a session policy that cannot silently produce a "clean" result.

---

## Scenario 9 — network, rate, and engine

| # | Command | How to use |
|---|---------|------------|
| 81 | `dalfox scan https://example.com --timeout 20` | Per-request timeout |
| 82 | `dalfox scan https://example.com --scan-timeout 600` | Injection-stage wall clock |
| 83 | `dalfox scan https://example.com --delay 200` | Per-worker delay (ms) |
| 84 | `dalfox scan https://example.com -r 20` | Global req/s cap |
| 85 | `dalfox scan https://example.com --retries 2` | Retry 5xx/transient |
| 86 | `dalfox scan https://example.com --retry-delay 500` | Backoff base |
| 87 | `dalfox scan https://example.com --proxy http://127.0.0.1:8080` | HTTP proxy |
| 88 | `dalfox scan https://example.com --proxy socks5://127.0.0.1:9050` | SOCKS proxy |
| 89 | `dalfox scan https://example.com --workers 10` | Worker count |
| 90 | `dalfox scan https://example.com --max-concurrent-targets 5` | Target concurrency |
| 91 | `dalfox scan https://example.com --max-targets-per-host 50` | Per-host cap |

**Research questions**
- Why `--rate-limit` is safer than `--delay` on shared IPs?
- Interaction of `--scan-timeout` with multi-phase scans?

**Sources**: rate docs, source scheduler, issues on 429.

**Output**: a rate profile inside the program's published limits.

---

## Scenario 10 — output and reporting

| # | Command | How to use |
|---|---------|------------|
| 92 | `dalfox scan https://example.com -f plain` | Plain text |
| 93 | `dalfox scan https://example.com -f json -o out.json` | JSON |
| 94 | `dalfox scan https://example.com -f jsonl -o out.jsonl` | JSONL |
| 95 | `dalfox scan https://example.com -f markdown -o out.md` | Markdown |
| 96 | `dalfox scan https://example.com -f sarif -o out.sarif` | SARIF |
| 97 | `dalfox scan https://example.com -f toml -o out.toml` | TOML |
| 98 | `dalfox scan https://example.com --include-request` | Include request in output |
| 99 | `dalfox scan https://example.com --include-response` | Include response |
| 100 | `dalfox scan https://example.com --include-all` | Both |
| 101 | `dalfox scan https://example.com --poc-type curl` | Curl PoC |
| 102 | `dalfox scan https://example.com --poc-type httpie` | HTTPie PoC |
| 103 | `dalfox scan https://example.com --poc-type http-request` | Raw HTTP PoC |
| 104 | `dalfox scan https://example.com --limit 10` | Limit findings |
| 105 | `dalfox scan https://example.com --limit-result-type v` | Limit counts verified only |
| 106 | `dalfox scan https://example.com --only-poc 'v,r'` | Show only chosen types |
| 107 | `dalfox scan https://example.com --min-confidence high` | Drop low-confidence findings |
| 108 | `dalfox scan https://example.com --baseline baseline.json` | Only new findings |
| 109 | `dalfox scan https://example.com --baseline baseline.json --baseline-mode annotate` | Keep + mark new |
| 110 | `dalfox scan https://example.com --stream-findings` | Emit as found |
| 111 | `dalfox scan https://example.com -S` | Silence logs, POCs only |
| 112 | `dalfox scan https://example.com --no-color` | No ANSI |
| 113 | `dalfox scan https://example.com --debug` | Debug logging |

**Research questions**
- Which formats include enough evidence for a report?
- How does `--baseline` compute finding identity?

**Sources**: output docs, source serializers, issues on SARIF.

**Output**: a report format chosen for its consumer, plus a baseline for
re-testing.

---

## Scenario 11 — server mode

| # | Command | How to use |
|---|---------|------------|
| 114 | `dalfox server` | API on 127.0.0.1:6664 |
| 115 | `dalfox server -p 7777 -H 0.0.0.0` | Custom port/host |
| 116 | `dalfox server --api-key SECRET` | Require `X-API-KEY` |
| 117 | `DALFOX_API_KEY=SECRET dalfox server` | Key from environment |
| 118 | `dalfox server --log-file server.log` | Log to a 0600 file |
| 119 | `dalfox server --rate-limit 20` | Server-wide per-scan rate cap |
| 120 | `dalfox server --scan-timeout 600` | Server-wide scan time cap |
| 121 | `dalfox server --max-concurrent-scans 10` | Concurrency cap |
| 122 | `dalfox server --allowed-hosts api.example.com` | DNS-rebinding guard |
| 123 | `dalfox server --max-retained-scans 100` | Bound retained results |
| 124 | `dalfox server --max-body-bytes 1048576` | Request body cap |
| 125 | `dalfox server --allowed-origins 'https://app.example.com'` | CORS allowlist |
| 126 | `dalfox server --allowed-origins '*'` | Wildcard CORS (opens the gate) |
| 127 | `dalfox server --cors-allow-methods GET,POST` | CORS methods |
| 128 | `dalfox server --cors-allow-headers Content-Type,X-API-KEY'` | CORS headers |
| 129 | `dalfox server --jsonp` | Allow JSONP |
| 130 | `dalfox server --callback-param-name cb` | JSONP callback name |

**Research questions**
- What is the API contract for POST /scan and /preflight?
- Why is `--allowed-origins '*'` dangerous on a reachable host?

**Sources**: server docs, source routes, issues on CORS/API.

**Output**: a locked-down server config for shared/remote use.

---

## Scenario 12 — payload enumeration, MCP, completion

| # | Command | How to use |
|---|---------|------------|
| 131 | `dalfox payload javascript` | JavaScript payloads |
| 132 | `dalfox payload event-handlers` | Event-handler payloads |
| 133 | `dalfox payload useful-tags` | Useful HTML tags |
| 134 | `dalfox payload payloadbox` | Payloads from payloadbox |
| 135 | `dalfox payload portswigger` | PortSwigger payloads |
| 136 | `dalfox payload uri-scheme` | URI-scheme payloads |
| 137 | `dalfox payload special-chars` | Special characters |
| 138 | `dalfox payload functions` | Function payloads |
| 139 | `dalfox payload awesome-alert` | Alert payloads |
| 140 | `dalfox payload dom-clobbering` | DOM clobbering |
| 141 | `dalfox payload mxss` | Mutation XSS |
| 142 | `dalfox payload blind` | Blind payloads |
| 143 | `dalfox payload all` | Everything |
| 144 | `dalfox payload all --json` | JSON array output |
| 145 | `dalfox mcp` | MCP stdio server for agent use |
| 146 | `dalfox completion bash` | Bash completion |
| 147 | `dalfox completion zsh` | Zsh completion |
| 148 | `dalfox completion fish` | Fish completion |
| 149 | `dalfox --version` | Version |
| 150 | `dalfox --config dalfox.toml` | Config file |
| 151 | `dalfox --debug` | Debug logging |
| 152 | `dalfox --no-color` | No color |
| 153 | `dalfox --help` | Top-level help |

**Research questions**
- Which payload selectors are generated versus vendored?
- What tools does the MCP server expose, and what auth does it need?

**Sources**: payload docs, source payload DB, MCP docs.

**Output**: a payload shortlist per context and a working completion setup.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| Global | `--config`, `--debug`, `--no-color`, `-S`/`--silence`, `-V`/`--version`, `-h` | 10, 12 |
| Input | `-i`/`--input-type`, `--base-url`, `--dedup-urls`, `--state-file` | 1, 3 |
| Output | `-f`/`--format`, `-o`/`--output`, `--include-request`, `--include-response`, `--include-all`, `--dry-run`, `--stream-findings`, `--poc-type`, `--limit`, `--limit-result-type`, `--only-poc`, `--baseline`, `--baseline-mode`, `--min-confidence` | 10 |
| Targets | `-p`/`--param`, `-d`/`--data`, `-H`/`--headers`, `--cookies`, `-X`/`--method`, `--user-agent`, `--cookie-from-raw` | 2 |
| Session | `--session-check`, `--session-check-url`, `--on-session-loss` | 8 |
| Scope | `--include-url`, `--exclude-url`, `--ignore-param`, `--out-of-scope`, `--out-of-scope-file` | 3 |
| Discovery | `--only-discovery`, `--skip-discovery`, `--skip-reflection-header`, `--skip-reflection-cookie`, `--skip-reflection-path` | 4 |
| Mining | `-W`/`--mining-dict-word`, `--remote-wordlists`, `--skip-mining`, `--skip-mining-dict`, `--skip-mining-dom` | 4 |
| Network | `--timeout`, `--scan-timeout`, `--delay`, `-r`/`--rate-limit`, `--retries`, `--retry-delay`, `--proxy`, `--insecure`, `-F`/`--follow-redirects`, `--ignore-return` | 9 |
| Engine | `--workers`, `--max-concurrent-targets`, `--max-targets-per-host` | 9 |
| Scanning | `-e`/`--encoders`, `--remote-payloads`, `--custom-blind-xss-payload`, `-b`/`--blind`, `--blind-oob`, `--blind-oob-secret`, `--blind-oob-wait`, `--custom-payload`, `--only-custom-payload`, `--custom-alert-value`, `--custom-alert-type`, `--inject-marker`, `--skip-xss-scanning`, `--max-payloads-per-param`, `--deep-scan`, `--sxss`, `--sxss-url`, `--sxss-method`, `--sxss-retries`, `--skip-ast-analysis`, `--analyze-external-js`, `--hpp`, `--detect-outdated-libs` | 5, 6 |
| WAF | `--waf-bypass`, `--skip-waf-probe`, `--force-waf`, `--waf-evasion`, `--waf-min-confidence` | 7 |
| Server | `-p`/`--port`, `-H`/`--host`, `--api-key`, `--log-file`, `--rate-limit`, `--scan-timeout`, `--max-concurrent-scans`, `--allowed-hosts`, `--max-retained-scans`, `--max-body-bytes`, `--allowed-origins`, `--cors-allow-methods`, `--cors-allow-headers`, `--jsonp`, `--callback-param-name` | 11 |
| Payload | selectors + `--json` | 12 |
| Commands | `scan`, `server`, `payload`, `mcp`, `completion`, `help` | all |

Full per-flag text: [`help.txt`](help.txt), [`help-scan-full.txt`](help-scan-full.txt),
[`help-server.txt`](help-server.txt), [`help-payload-full.txt`](help-payload-full.txt).
