# x8 — hidden parameter discovery

- **Version:** x8 4.3.2
- **Binary:** `/usr/local/bin/x8` (Rust)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/Sh1Yo/x8

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

x8 discovers hidden HTTP parameters by sending requests with candidate
parameters and comparing responses against a learned baseline. It works on GET
query strings, POST bodies (urlencoded or JSON), and headers, and it can test
custom non-random values such as `admin=true` or `debug=1`. It supports raw
request files, recursion, replay proxies, and multiple output formats.

## Use cases

| Situation | Why x8 |
|-----------|--------|
| Hidden parameter discovery | Find undocumented GET/POST/header params |
| Access-control testing | `admin=true`, `debug=1`, `env=prod` |
| API fuzzing | JSON body parameter discovery |
| Burp handoff | Replay found parameters through a proxy |
| Batch URLs | Workers and per-host sequencing |
| Evidence | JSON/url/request output formats |

## Requirements & install layout

- Rust binary at `/usr/local/bin/x8`.
- No API keys.
- Learns a response baseline with several requests (`--learn-requests`).
- Works best on dynamic pages with measurable differences; static pages yield
  little signal.

## Basic syntax

```bash
# GET parameter discovery
x8 -u https://target/page -w params.txt

# POST urlencoded
x8 -u https://target/login -X POST -w params.txt

# JSON body
x8 -u https://target/api -X POST -b '{"%s":{{random}}}' -t json -w params.txt

# header discovery
x8 -u https://target/ --headers -w headers.txt

# raw request
x8 -r request.txt -w params.txt

# batch URLs
x8 -u urls.txt -w params.txt -W 10
```

## Flag reference

### Target and input
| Flag | Meaning |
|------|---------|
| `-u, --url <url\\|file>` | URL(s), file of URLs, or `%s` injection point |
| `-r, --request <file>` | Raw HTTP request file |
| `-w, --wordlist <file>` | Parameter wordlist (stdin if empty) |
| `-X, --method <methods>` | Methods (`GET POST ...`) |
| `-b, --body <body>` | Body template, `%s` placeholder, `{{random}}` |
| `-t, --data-type <type>` | `urlencode` or `json` |
| `-H <headers>` | Headers |
| `--headers` | Header discovery mode |
| `-p, --port <port>` / `--proto <proto>` | For raw requests |

### Detection
| Flag | Meaning |
|------|---------|
| `--learn-requests <n>` | Baseline requests (default 9) |
| `-m, --max <n>` | Max parameters per request |
| `-c <n>` | Concurrent requests per URL |
| `-W, --workers <n>` | Concurrent URL checks (`-W0` = all parallel) |
| `--one-worker-per-host` | Serialise same-host URLs |
| `--strict` | Only report parameters that change different page parts |
| `--reflected-only` | Skip comparison, look for reflection |
| `--verify` | Verify found parameters |
| `--custom-parameters <list>` | Non-random parameter names |
| `--custom-values <list>` | Values for those names |
| `--disable-custom-parameters` | Skip the built-in set |
| `--check-binary` | Compare binary response bodies |
| `--force` | Allow pages > 25MB |
| `--recursion-depth <n>` | Recurse with found parameters |

### Request behaviour
| Flag | Meaning |
|------|---------|
| `-L, --follow-redirects` | Follow redirects |
| `--encode` | Encode special characters |
| `--mimic-browser` | Add browser-like headers |
| `--http <1.1\\|2>` | HTTP version |
| `-d, --delay <ms>` | Delay between requests |
| `--timeout <sec>` | Request timeout |
| `-x, --proxy <url>` / `-B` | Proxy (`-B` = localhost:8080) |
| `--replay-proxy <url>` / `--replay-once` | Replay findings |
| `--disable-trustdns` | DNS workaround |
| `-j, --joiner <str>` | Parameter joiner |
| `-P, --param-template <t>` | Custom parameter template |

### Output
| Flag | Meaning |
|------|---------|
| `-o, --output <file>` | Output file |
| `-O, --output-format <fmt>` | `standart`, `json`, `url`, `request` |
| `--append` | Append to output |
| `--remove-empty` | Skip empty results |
| `--save-responses <dir>` | Save request/response on findings |
| `--disable-colors`, `--disable-progress-bar` | Cosmetic |
| `--test` | Print request and response |
| `-v, --verbose <0\\|1\\|2>` | Verbosity |

## Recipes

```bash
# 1. GET discovery with a good wordlist
x8 -u https://target/page -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt

# 2. POST urlencoded
x8 -u https://target/login -X POST -w params.txt -t urlencode

# 3. JSON body discovery
x8 -u https://target/api -X POST -b '{"%s":{{random}}}' -t json -w params.txt

# 4. Header discovery
x8 -u https://target/ --headers -w headers.txt

# 5. Raw request from Burp
x8 -r request.txt -w params.txt

# 6. Batch with per-host sequencing
x8 -u urls.txt -w params.txt -W 10 --one-worker-per-host

# 7. Strict, verified findings only
x8 -u https://target/page -w params.txt --strict --verify

# 8. Replay findings through Burp
x8 -u https://target/page -w params.txt --replay-proxy http://127.0.0.1:8080 --replay-once

# 9. JSON output for parsing
x8 -u https://target/page -w params.txt -O json -o findings.json

# 10. Recursive discovery
x8 -u https://target/page -w params.txt --recursion-depth 2
```

## Output & pipelines

- Default table lists URL, method, parameter, and detection reason.
- `-O json` for machine parsing; `-O url` for direct re-testing; `-O request`
  for raw requests.
- Findings feed `ffuf`, `nuclei`, Burp Repeater, and manual access-control
  testing.
- `--save-responses` keeps evidence for each hit.

## Gotchas & operational security

- Dynamic pages (timestamps, ads, CSRF tokens) produce false positives; use
  `--strict` and `--verify`.
- Too few `--learn-requests` on noisy pages causes bad baselines.
- `--custom-parameters` sends meaningful values and can change state — review
  before running against production.
- Recursion multiplies requests; cap depth.
- High `-c`/`-W` values can overload small apps.
- Pages over 25MB are skipped unless `--force`.
- `--replay-once` combines findings into one request, which may behave
  differently from individual requests.

## Legal & safety

Parameter discovery is active testing and can trigger state changes. Use only
on authorized targets, avoid destructive parameter names, and coordinate rate
limits with the target's operators.

## Related tools

- [`paramspider/`](../paramspider/README.md) — passive parameter mining
- [`wfuzz/`](../wfuzz/README.md), [`dirsearch/`](../dirsearch/README.md)
- [`corsy/`](../corsy/README.md), [`smuggler/`](../smuggler/README.md)
- [`linkfinder/`](../linkfinder/README.md) — parameters in JS

## References

- https://github.com/Sh1Yo/x8
