# httpx — command scenarios for deep research

Every httpx command form, grouped by scenario. Flags from v1.12.0 (`help.txt`).

## How to invoke

```text
@deep-research For httpx v1.12, which probes are reliable enough to triage
10k hosts, and what rate limits avoid bans on shared bug-bounty infrastructure?
Cite docs, GitHub issues, and independent benchmarks; return exact flags.
```

---

## Scenario 1 — triage a host list

| # | Command | How to use |
|---|---------|------------|
| 1 | `httpx -u https://example.com` | Single target |
| 2 | `httpx -l subs.txt` | File of hosts |
| 3 | `cat subs.txt \| httpx` | STDIN pipeline |
| 4 | `httpx -l subs.txt -sc -title -td -silent -o live.txt` | Standard triage: status, title, tech |
| 5 | `httpx -l subs.txt -sc -cl -ct -location` | Headers-level summary |
| 6 | `httpx -l subs.txt -sc -title -td -fr` | Follow redirects, show final |

**Research questions**
- Is default HTTPS-first with HTTP fallback still the behavior in 1.12?
- What does `-td` detect that `-cpe`/`-server` miss, and how accurate is it?

**Sources**: docs usage, ProjectDiscovery blog on httpx, GitHub issues on tech
detection.

---

## Scenario 2 — filter to what matters

| # | Command | How to use |
|---|---------|------------|
| 7 | `httpx -l subs.txt -mc 200` | Only 200 OK |
| 8 | `httpx -l subs.txt -mc 200,401,403 -title` | Auth-gated surfaces |
| 9 | `httpx -l subs.txt -ms admin -ms login -title` | Body substring match |
| 10 | `httpx -l subs.txt -mr 'admin\|dashboard' -title` | Regex match |
| 11 | `httpx -l subs.txt -fc 404,502` | Filter codes |
| 12 | `httpx -l subs.txt -fpt login,captcha,parked` | Filter page types |
| 13 | `httpx -l subs.txt -fd` | Drop near-duplicate pages |
| 14 | `httpx -l subs.txt -e cdn` | Exclude CDN-fronted hosts |
| 15 | `httpx -l subs.txt -mdc 'status_code == 200 && contains(title, "admin")'` | DSL match |
| 16 | `httpx -l subs.txt -fdc 'status_code == 404'` | DSL filter |
| 17 | `httpx -l subs.txt -strip` | Strip tags before matching |

**Research questions**
- False-positive rate of `-ms`/`-mr` versus `-fpt` on modern SPAs?
- Does `-fd` dedupe by body hash or fuzzy similarity?

**Sources**: docs matchers/filters, source code, community writeups on httpx
filtering.

---

## Scenario 3 — evidence and fingerprints

| # | Command | How to use |
|---|---------|------------|
| 18 | `httpx -l live.txt -favicon -jarm` | Favicon mmh3 + JARM |
| 19 | `httpx -l live.txt -hash sha256` | Body hash (md5/mmh3/simhash/sha1/sha256/sha512) |
| 20 | `httpx -l live.txt -tls-grab -json` | Full TLS metadata |
| 21 | `httpx -l live.txt -cpe -server -td` | CPE + server + tech |
| 22 | `httpx -l live.txt -wp` | WordPress plugins/themes |
| 23 | `httpx -l live.txt -ip -cname -asn` | Network attribution |
| 24 | `httpx -l live.txt -cdn -probe` | CDN/WAF + probe status |
| 25 | `httpx -l live.txt -efqdn` | Extract FQDNs from responses |
| 26 | `httpx -l live.txt -er 'api[_-]?key' -ep url,mail` | Regex/preset extraction |
| 27 | `httpx -l live.txt -kb` | Knowledge-base classification |

**Research questions**
- Which fingerprints are stable enough for asset attribution?
- Does `-jarm` add meaningful signal over favicon + TLS?

**Sources**: docs probes, Salesforce JARM paper, favicon-hash writeups.

---

## Scenario 4 — screenshots and headless

| # | Command | How to use |
|---|---------|------------|
| 28 | `httpx -l live.txt -ss -system-chrome` | Screenshot with local Chrome |
| 29 | `httpx -l live.txt -ss -ho '--no-sandbox,--disable-gpu'` | Extra Chrome flags |
| 30 | `httpx -l live.txt -ss -esb -json` | Exclude screenshot bytes from JSON |
| 31 | `httpx -l live.txt -ss -ehb` | Exclude headless body |
| 32 | `httpx -l live.txt -ss -no-screenshot-full-page` | Viewport-only |
| 33 | `httpx -l live.txt -ss -st 20 -sid 2` | Screenshot + idle timeouts |
| 34 | `httpx -l live.txt -ss -jsc 'document.title'` | Run JS after navigation |

**Research questions**
- Headless Chrome vs system Chrome: reliability, memory, and version drift?
- Which timeout pair avoids blank screenshots on slow apps?

**Sources**: docs headless, GitHub issues tagged screenshot, ProjectDiscovery
blog on httpx screenshots.

---

## Scenario 5 — input breadth and protocol probing

| # | Command | How to use |
|---|---------|------------|
| 35 | `httpx -rr request.txt` | Raw HTTP request file |
| 36 | `httpx -im burp -l burp.xml` | Burp XML export |
| 37 | `httpx -u 10.0.0.0/24 -p http:80,443,8000-8080` | Port sweep on a CIDR |
| 38 | `httpx -l subs.txt -path /admin,/api` | Probe extra paths |
| 39 | `httpx -l subs.txt -pa` | Probe every A record |
| 40 | `httpx -l subs.txt -tls-probe` | TLS SAN discovery |
| 41 | `httpx -l subs.txt -csp-probe` | CSP domain discovery |
| 42 | `httpx -l subs.txt -vhost` | Vhost discovery |
| 43 | `httpx -l subs.txt -vhost-input` | Treat input as vhost names |
| 44 | `httpx -l subs.txt -x POST -body 'a=1'` | Custom method + body |
| 45 | `httpx -l subs.txt -http2` | Force HTTP/2 |
| 46 | `httpx -l subs.txt -pipeline` | HTTP/1.1 pipelining |
| 47 | `httpx -l subs.txt -ws` | WebSocket support check |

**Research questions**
- Which protocol features (HTTP/2, pipelining) are safe against WAFs?
- How does vhost probing differ from Host-header fuzzing?

**Sources**: docs, GitHub issues on vhost/TLS probes, RFC 9113 notes.

---

## Scenario 6 — output and storage

| # | Command | How to use |
|---|---------|------------|
| 48 | `httpx -l live.txt -o out.txt` | Plain output |
| 49 | `httpx -l live.txt -oa` | All formats |
| 50 | `httpx -l live.txt -sr -srd responses/` | Store responses / to dir |
| 51 | `httpx -l live.txt -ob` | Omit body |
| 52 | `httpx -l live.txt -csv` / `-j` / `-md` | CSV / JSONL / Markdown |
| 53 | `httpx -l live.txt -irh` / `-irr` / `-irrb` | Include headers / req-resp / base64 |
| 54 | `httpx -l live.txt -include-chain -store-chain` | Redirect chain |
| 55 | `httpx -l live.txt -pr http11` | Force protocol for output |
| 56 | `httpx -l live.txt -rdb postgres -rdb-connection '...'` | Store to a database |

**Research questions**
- Which output fields are stable across versions for downstream parsers?
- Is `-irr` enough evidence for a report, or is `-sr` needed?

**Sources**: docs output, JSON schema in source, issue threads on field changes.

---

## Scenario 7 — rate, retries, optimization

| # | Command | How to use |
|---|---------|------------|
| 57 | `httpx -l live.txt -t 20 -rl 50` | Concurrency + req/s |
| 58 | `httpx -l live.txt -rlm 600` | Requests per minute |
| 59 | `httpx -l live.txt -delay 500ms` | Fixed delay |
| 60 | `httpx -l live.txt -retries 2 -timeout 15` | Retry budget |
| 61 | `httpx -l live.txt -nf` | No HTTP fallback |
| 62 | `httpx -l live.txt -nfs` | No fallback scheme |
| 63 | `httpx -l live.txt -maxhr 5` | Max errors per host |
| 64 | `httpx -l live.txt -e private-ips` | Exclude private ranges |
| 65 | `httpx -l live.txt -resume resume.cfg` | Resume long runs |
| 66 | `httpx -l live.txt -stream` | Stream input (no sort) |
| 67 | `httpx -l live.txt -sd` | Skip dedupe |

**Research questions**
- How do `-rl`, `-rlm`, and `-t` interact in 1.12?
- What retry policy triggers WAF blocks?

**Sources**: docs rate limiting, GitHub issues on throttling, program rate rules.

---

## Scenario 8 — configuration and debugging

| # | Command | How to use |
|---|---------|------------|
| 68 | `httpx -l live.txt -config cfg.yaml` | Config file |
| 69 | `httpx -l live.txt -r resolvers.txt -allow 10.0.0.0/8 -deny 169.254.0.0/16` | Resolver + network allow/deny |
| 70 | `httpx -l live.txt -H 'Authorization: Bearer ...'` | Custom headers |
| 71 | `httpx -l live.txt -proxy http://127.0.0.1:8080` | HTTP proxy |
| 72 | `httpx -l live.txt -sni custom.example.com` | SNI override |
| 73 | `httpx -l live.txt -random-agent -auto-referer` | UA/referer behavior |
| 74 | `httpx -l live.txt -unsafe` | Raw/unsafe requests |
| 75 | `httpx -l live.txt -no-decode` | Don't decode bodies |
| 76 | `httpx -l live.txt -tlsi` | TLS client-hello randomization |
| 77 | `httpx -l live.txt -ldv` | List DSL variables |
| 78 | `httpx -l live.txt -debug -debug-req -debug-resp` | Full debug |
| 79 | `httpx -l live.txt -stats -silent -nc -v -tr` | Runtime telemetry |
| 80 | `httpx -health-check` | Self-check |

**Research questions**
- Does `-tlsi` defeat specific TLS fingerprint blocks, and at what cost?
- Which `-debug*` flag leaks credentials into logs?

**Sources**: docs, source, security writeups on TLS fingerprinting.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| Input | `-u`, `-l`, `-rr`, `-im` | 1, 5 |
| Probes | `-sc`, `-cl`, `-ct`, `-location`, `-favicon`, `-hash`, `-jarm`, `-rt`, `-lc`, `-wc`, `-title`, `-bp`, `-server`, `-td`, `-cff`, `-cpe`, `-wp`, `-kb`, `-method`, `-ws`, `-ip`, `-cname`, `-asn`, `-cdn`, `-probe`, `-efqdn` | 1, 3 |
| Headless | `-ss`, `-system-chrome`, `-ho`, `-esb`, `-ehb`, `-no-screenshot-full-page`, `-st`, `-sid`, `-jsc` | 4 |
| Matchers | `-mc`, `-ml`, `-mlc`, `-mwc`, `-mfc`, `-ms`, `-mr`, `-mcdn`, `-mrt`, `-mdc` | 2 |
| Extractors | `-er`, `-ep` | 3 |
| Filters | `-fc`, `-fpt`, `-fd`, `-fl`, `-flc`, `-fwc`, `-ffc`, `-fs`, `-fe`, `-fcdn`, `-frt`, `-fdc`, `-strip` | 2 |
| Rate | `-t`, `-rl`, `-rlm`, `-delay` | 7 |
| Protocol/ports | `-pa`, `-p`, `-path`, `-tls-probe`, `-csp-probe`, `-tls-grab`, `-pipeline`, `-http2`, `-vhost`, `-vhost-input`, `-x`, `-body`, `-ldv` | 5 |
| Output | `-o`, `-oa`, `-sr`, `-srd`, `-ob`, `-csv`, `-j`, `-md`, `-irh`, `-irr`, `-irrb`, `-include-chain`, `-store-chain`, `-pr`, `-rdb*` | 6 |
| Config | `-config`, `-r`, `-allow`, `-deny`, `-sni`, `-random-agent`, `-auto-referer`, `-H`, `-proxy`, `-unsafe`, `-resume`, `-fr`, `-maxr`, `-fhr`, `-rhsts`, `-s`, `-sd`, `-ldp`, `-no-decode`, `-tlsi`, `-no-stdin` | 8 |
| Optimization | `-nf`, `-nfs`, `-maxhr`, `-e`, `-retries`, `-timeout`, `-rsts`, `-rstr` | 7 |
| Debug | `-debug`, `-debug-req`, `-debug-resp`, `-stats`, `-silent`, `-v`, `-nc`, `-tr`, `-health-check` | 8 |

Full per-flag text: [`help.txt`](help.txt).
