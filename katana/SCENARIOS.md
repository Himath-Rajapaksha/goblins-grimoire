# katana — command scenarios for deep research

Every katana command form, grouped by scenario. Flags from v1.8.0 (`help.txt`).

## How to invoke

```text
@deep-research For katana v1.8, when is -jsl worth its memory cost, and what
scope flags prevent crawling outside a program's wildcard? Cite docs, the field
extraction blog, and GitHub issues; return exact flags.
```

---

## Scenario 1 — standard crawl

| # | Command | How to use |
|---|---------|------------|
| 1 | `katana -u https://example.com` | Single URL |
| 2 | `katana -list live.txt -d 3 -silent -o urls.txt` | Host list, depth 3 |
| 3 | `cat live.txt \| katana -silent` | STDIN pipeline |
| 4 | `katana -u https://example.com -d 5` | Deeper crawl |
| 5 | `katana -u https://example.com -ct 10m` | Time-bounded crawl |
| 6 | `katana -u https://example.com -kf all` | robots.txt + sitemap.xml |
| 7 | `katana -u https://example.com -mrs 8000000` | Raise response-size cap |
| 8 | `katana -u https://example.com -timeout 20 -retry 2` | Timeout + retries |
| 9 | `katana -u https://example.com -time-stable` | Wait for page stability |
| 10 | `katana -u https://example.com -dr` | Disable redirects |
| 11 | `katana -u https://example.com -pc` | Crawl parent paths |
| 12 | `katana -u https://example.com -resume state.cfg` | Resume crawl |

**Research questions**
- What depth actually reaches hidden endpoints without exploding output?
- Does `-ct` stop cleanly or leave goroutines hanging?

**Sources**: docs usage, ProjectDiscovery blog, issues tagged depth/timeout.

---

## Scenario 2 — JavaScript-aware crawl

| # | Command | How to use |
|---|---------|------------|
| 13 | `katana -u https://app.example.com -jc` | Parse JS for endpoints |
| 14 | `katana -u https://app.example.com -jsl` | jsluice parsing (memory-heavy) |
| 15 | `katana -u https://app.example.com -jc -jsl -d 4 -silent` | Both, deeper |
| 16 | `katana -u https://app.example.com -kb-secrets` | JS secret classifiers |
| 17 | `katana -u https://app.example.com -kb-validate-secrets` | Validate found secrets |
| 18 | `katana -u https://app.example.com -kb-endpoints` | Endpoint classifier |
| 19 | `katana -u https://app.example.com -mdp 20` | Max pages per domain |

**Research questions**
- Memory ceiling of `-jsl` on 5 MB+ bundles?
- Do `-kb-*` classifiers leak values into logs?

**Sources**: field-extraction blog, docs, issues on jsluice/OOM.

---

## Scenario 3 — headless and hybrid

| # | Command | How to use |
|---|---------|------------|
| 20 | `katana -u https://app.example.com -hl` | Headless crawl |
| 21 | `katana -u https://app.example.com -hh` | Hybrid standard + headless |
| 22 | `katana -u https://app.example.com -hl -sc` | Use installed Chrome |
| 23 | `katana -u https://app.example.com -hl -sb` | Show browser window |
| 24 | `katana -u https://app.example.com -hl -ho '--proxy-server=...'` | Extra Chrome flags |
| 25 | `katana -u https://app.example.com -hl -nos` | `--no-sandbox` |
| 26 | `katana -u https://app.example.com -hl -cdd /tmp/chrome` | Chrome data dir |
| 27 | `katana -u https://app.example.com -hl -scp /usr/bin/chromium` | Chrome binary path |
| 28 | `katana -u https://app.example.com -hl -noi` | Disable incognito |
| 29 | `katana -u https://app.example.com -hl -cwu ws://127.0.0.1:9222` | Attach remote Chrome |
| 30 | `katana -u https://app.example.com -hl -xhr` | Extract XHR requests |
| 31 | `katana -u https://app.example.com -hl -mfc 5` | Max action failures |
| 32 | `katana -u https://app.example.com -hl -ed` | Browser diagnostics |
| 33 | `katana -u https://app.example.com -hl -pls networkidle` | Page-load strategy |
| 34 | `katana -u https://app.example.com -hl -dwt 3s` | DOM wait after load |
| 35 | `katana -u https://app.example.com -hl -csp capsolver -csk KEY` | Captcha solving |
| 36 | `katana -u https://app.example.com -hl -al user:pass` | Auto-login |
| 37 | `katana -u https://app.example.com -hl -rf flow.json` | Replay DevTools flow |

**Research questions**
- Headless vs hybrid coverage and cost on SPAs?
- Does `-al` store credentials in the state file?

**Sources**: docs headless, DevTools recorder docs, issues on Chrome attach.

---

## Scenario 4 — scope and output filtering

| # | Command | How to use |
|---|---------|------------|
| 38 | `katana -u https://example.com -fs rdn` | Registrable-domain scope |
| 39 | `katana -u https://example.com -fs fqdn` | Exact host scope |
| 40 | `katana -u https://example.com -fs '.*\.example\.com'` | Regex scope |
| 41 | `katana -u https://example.com -cs '/api/'` | In-scope regex |
| 42 | `katana -u https://example.com -cos 'logout\|delete'` | Out-of-scope regex |
| 43 | `katana -u https://example.com -ns` | Disable host scope |
| 44 | `katana -u https://example.com -do` | Display out-of-scope |
| 45 | `katana -u https://example.com -mr '/api/'` | Match output URL |
| 46 | `katana -u https://example.com -fr '\.png$'` | Filter output URL |
| 47 | `katana -u https://example.com -em js,json` | Extension match |
| 48 | `katana -u https://example.com -ef jpg,css` | Extension filter |
| 49 | `katana -u https://example.com -ndef` | Keep default ext filter |
| 50 | `katana -u https://example.com -mdc 'len(body) > 0'` | DSL match |
| 51 | `katana -u https://example.com -fdc 'status_code == 404'` | DSL filter |
| 52 | `katana -u https://example.com -duf` | Disable duplicate filter |
| 53 | `katana -u https://example.com -pcs -pcsm simhash` | Similarity filter |
| 54 | `katana -u https://example.com -pcsd 0.9 -pcst 0.8 -pcsn 10` | Similarity tuning |
| 55 | `katana -u https://example.com -fpt error,captcha,parked` | Filter page types |

**Research questions**
- Exact semantics of `-fs dn` vs `-fs rdn` vs `-fs fqdn`?
- Does `-cos` block the request or only the output?

**Sources**: docs scope section, source, issues on out-of-scope crawling.

---

## Scenario 5 — forms, auth, and replay

| # | Command | How to use |
|---|---------|------------|
| 56 | `katana -u https://example.com -fx` | Emit form/input JSONL |
| 57 | `katana -u https://example.com -aff` | Automatic form filling |
| 58 | `katana -u https://example.com -H 'Cookie: session=...'` | Authenticated crawl |
| 59 | `katana -u https://example.com -H 'Authorization: Bearer ...'` | Header auth |
| 60 | `katana -u https://example.com -config cfg.yaml` | Config file |
| 61 | `katana -u https://example.com -fc forms.yaml -flc fields.yaml` | Form/field configs |
| 62 | `katana -u https://example.com -proxy http://127.0.0.1:8080` | Through Burp |
| 63 | `katana -u https://example.com -td` | Tech detection per host |

**Research questions**
- Does `-aff` submit destructive forms, and how to constrain it?
- Is `-rf` compatible with the installed Chrome's recorder format?

**Sources**: docs form extraction, DevTools recorder docs, issues.

---

## Scenario 6 — output and storage

| # | Command | How to use |
|---|---------|------------|
| 64 | `katana -u ... -o urls.txt` | Plain URLs |
| 65 | `katana -u ... -ot '{{host}}/{{path}}'` | Output template |
| 66 | `katana -u ... -j` / `-jsonl` | JSONL |
| 67 | `katana -u ... -sr -srd resp/` | Store responses |
| 68 | `katana -u ... -ncb` | No clobber |
| 69 | `katana -u ... -sfd out/` | Per-host field dir |
| 70 | `katana -u ... -or -ob` | Omit raw / body |
| 71 | `katana -u ... -lof url,status -eof body` | Field selection |
| 72 | `katana -u ... -sf url` | Store field per host |
| 73 | `katana -u ... -f url` | Output field (deprecated) |

**Research questions**
- Which JSONL fields are stable for a parser?
- Storage cost of `-sr` on a 100k-URL crawl?

**Sources**: docs output, JSONL schema in source.

---

## Scenario 7 — rate, concurrency, debug

| # | Command | How to use |
|---|---------|------------|
| 74 | `katana -list live.txt -c 10 -p 10` | Fetcher/input concurrency |
| 75 | `katana -list live.txt -rd 1s` | Delay between requests |
| 76 | `katana -list live.txt -rl 50 -rlm 300` | Global rate |
| 77 | `katana -list live.txt -hrl 10 -hrlm 60` | Per-host rate |
| 78 | `katana -u ... -s depth-first` | Traversal strategy |
| 79 | `katana -u ... -iqp` | Ignore query variants |
| 80 | `katana -u ... -fsu -fst 0.8` | Filter similar URLs |
| 81 | `katana -u ... -tlsi` | TLS hello randomization |
| 82 | `katana -u ... -e cdn,private-ips` | Exclude ranges |
| 83 | `katana -u ... -j -silent -nc -v -debug -version` | Logging/version |

**Research questions**
- Correct `-hrl` unit (per second or per minute)?
- Does `-rd` apply per host or globally?

**Sources**: docs rate flags, source, issues.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| Input | `-u`/`-list`, `-resume`, `-e` | 1, 7 |
| Crawl | `-d`, `-jc`, `-jsl`, `-ct`, `-kf`, `-mrs`, `-timeout`, `-time-stable`, `-aff`, `-fx`, `-retry`, `-proxy`, `-td`, `-H`, `-config`, `-fc`, `-flc`, `-s`, `-iqp`, `-fsu`, `-fst`, `-tlsi`, `-dr`, `-pc`, `-kb`, `-kb-secrets`, `-kb-validate-secrets`, `-kb-endpoints`, `-mdp` | 1, 2, 5 |
| Headless | `-hl`, `-hh`, `-sc`, `-sb`, `-ho`, `-nos`, `-cdd`, `-scp`, `-noi`, `-cwu`, `-xhr`, `-mfc`, `-ed`, `-pls`, `-dwt`, `-csp`, `-csk`, `-al`, `-rf` | 3 |
| Scope | `-cs`, `-cos`, `-fs`, `-ns`, `-do`, `-mr`, `-fr`, `-em`, `-ef`, `-ndef`, `-mdc`, `-fdc`, `-duf`, `-pcs`, `-pcsm`, `-pcsd`, `-pcst`, `-pcsn`, `-fpt` | 4 |
| Rate | `-c`, `-p`, `-rd`, `-rl`, `-rlm`, `-hrl`, `-hrlm` | 7 |
| Output | `-o`, `-ot`, `-sr`, `-srd`, `-ncb`, `-sfd`, `-or`, `-ob`, `-lof`, `-eof`, `-sf`, `-f`, `-j`/`-jsonl`, `-nc`, `-silent`, `-v`, `-debug`, `-version` | 6, 7 |

Full per-flag text: [`help.txt`](help.txt).
