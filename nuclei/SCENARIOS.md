# nuclei — command scenarios for deep research

Every nuclei command form, grouped by scenario. Flags from v3.11.1 (`help.txt`).

## How to invoke

```text
@deep-research For nuclei 3.11, which template tags give the best signal on a
mature bug-bounty target without generating destructive traffic, and how do
-ss strategies change coverage? Cite ProjectDiscovery docs, template repo, and
independent comparisons; return exact flags.
```

---

## Scenario 1 — target intake

| # | Command | How to use |
|---|---------|------------|
| 1 | `nuclei -u https://example.com` | Single target |
| 2 | `nuclei -l live.txt` | One target per line |
| 3 | `nuclei -targets-inline $'https://a\nhttps://b'` | Inline multiline list |
| 4 | `nuclei -l live.txt -eh 10.0.0.0/8,staging.example.com` | Exclude hosts/CIDRs |
| 5 | `nuclei -l live.txt -sa` | Scan every IP behind a DNS name |
| 6 | `nuclei -l live.txt -iv 4` | IPv4 only |
| 7 | `nuclei -l live.txt -iv 6` | IPv6 only |
| 8 | `nuclei -l live.txt -resume scan.cfg` | Resume a scan (clustering disabled) |
| 9 | `nuclei -l live.txt -mt 2h` | Hard max runtime |

**Research questions**
- Does `-sa` multiply requests per hostname, and by how much on CDNs?
- What state does `-resume` preserve — findings, request dedupe, or both?

**Sources**: docs running/targets, issues on resume, CDN behavior writeups.

**Output**: a deduplicated target set with runtime caps.

---

## Scenario 2 — template selection

| # | Command | How to use |
|---|---------|------------|
| 10 | `nuclei -u https://example.com -as` | Automatic scan by detected tech |
| 11 | `nuclei -u https://example.com -t http/cves/` | Template directory |
| 12 | `nuclei -u https://example.com -t http/cves/ -t ssl` | Multiple template paths |
| 13 | `nuclei -u https://example.com -nt` | Only templates new in latest release |
| 14 | `nuclei -u https://example.com -ntv v9.9.0` | New in a specific version |
| 15 | `nuclei -u https://example.com -w workflows/` | Run a workflow |
| 16 | `nuclei -u https://example.com -turl https://example.com/t.yaml` | Remote template |
| 17 | `nuclei -u https://example.com -wurl https://example.com/w.yaml` | Remote workflow |
| 18 | `nuclei -u https://example.com -ai 'check for exposed .git'` | Generate and run a template from a prompt |
| 19 | `nuclei -u https://example.com -tp profile.yaml` | Template profile |
| 20 | `nuclei -u https://example.com -tpl` | List community profiles |

**Research questions**
- How current is the template repo, and how often do templates false-positive?
- Is `-as` safe on production (which tags can it trigger)?

**Sources**: nuclei-templates repo, profile docs, issues tagged false-positive.

**Output**: a template set mapped to the target's stack and the program's rules.

---

## Scenario 3 — filtering templates

| # | Command | How to use |
|---|---------|------------|
| 21 | `nuclei -u https://example.com -tags cve,rce` | Include by tag |
| 22 | `nuclei -u https://example.com -etags dos,fuzz` | Exclude by tag |
| 23 | `nuclei -u https://example.com -itags cve` | Force-include excluded tags |
| 24 | `nuclei -u https://example.com -a pdteam` | By author |
| 25 | `nuclei -u https://example.com -id CVE-2024-*` | By template ID, wildcard allowed |
| 26 | `nuclei -u https://example.com -eid CVE-2019-*` | Exclude IDs |
| 27 | `nuclei -u https://example.com -s high,critical` | By severity |
| 28 | `nuclei -u https://example.com -es info,unknown` | Exclude severity |
| 29 | `nuclei -u https://example.com -pt http,dns` | By protocol type |
| 30 | `nuclei -u https://example.com -ept headless,code` | Exclude protocol type |
| 31 | `nuclei -u https://example.com -tc 'contains(tags,"cve")'` | Condition expression |
| 32 | `nuclei -u https://example.com -it path/custom.yaml` | Force-include a template path |
| 33 | `nuclei -u https://example.com -et path/noisy/` | Exclude a template path |
| 34 | `nuclei -u https://example.com -em matcher-name` | Drop specific matchers from results |

**Research questions**
- `-itags` versus `-it` — which overrides default exclusions?
- Which severity bands are worth running on a mature target?

**Sources**: template filtering docs, default exclude config, issues.

**Output**: a filter chain that is explainable line by line.

---

## Scenario 4 — template management and authoring

| # | Command | How to use |
|---|---------|------------|
| 35 | `nuclei -validate -t custom.yaml` | Validate templates |
| 36 | `nuclei -validate -t custom.yaml -nss` | Skip strict syntax checks |
| 37 | `nuclei -td -t custom.yaml` | Display template content |
| 38 | `nuclei -tl -tags cve` | List templates matching filters |
| 39 | `nuclei -tgl` | List all available tags |
| 40 | `nuclei -sign -t custom.yaml` | Sign with `NUCLEI_SIGNATURE_PRIVATE_KEY` |
| 41 | `nuclei -u https://example.com -code` | Enable code-protocol templates |
| 42 | `nuclei -u https://example.com -file` | Enable file templates |
| 43 | `nuclei -u https://example.com -esc` | Enable self-contained templates |
| 44 | `nuclei -u https://example.com -egm` | Enable global matchers |
| 45 | `nuclei -u https://example.com -dut` | Refuse unsigned/mismatched templates |

**Research questions**
- What does `-code` permit a template to execute, and what is the sandbox?
- Is signing enforced in the default template repo?

**Sources**: template syntax docs, code protocol docs, security advisories.

**Output**: a validated, signed custom template and a safe loading policy.

---

## Scenario 5 — input formats and variables

| # | Command | How to use |
|---|---------|------------|
| 46 | `nuclei -l burp.xml -im burp` | Burp XML export |
| 47 | `nuclei -l requests.jsonl -im jsonl` | JSONL input |
| 48 | `nuclei -l input.yaml -im yaml` | YAML input |
| 49 | `nuclei -l openapi.yaml -im openapi` | OpenAPI spec |
| 50 | `nuclei -l swagger.json -im swagger` | Swagger spec |
| 51 | `nuclei -l input.yaml -im yaml -ro` | Only required fields when building requests |
| 52 | `nuclei -l input.yaml -im yaml -sfv` | Skip format validation |
| 53 | `nuclei -l input.yaml -im yaml -vtt` | Text-templating for vars |
| 54 | `nuclei -l input.yaml -im yaml -vfp vars/` | Inject vars from YAML files |
| 55 | `nuclei -u https://example.com -V token=abc -V env=staging` | Custom template vars |
| 56 | `nuclei -u https://example.com -ev` | Allow env vars in templates |

**Research questions**
- Which input modes preserve auth headers and which require `-H`?
- How are vars scoped when multiple input records define the same key?

**Sources**: input-format docs, template variables docs, issues.

**Output**: a reproducible input file plus the exact var set, secrets excluded.

---

## Scenario 6 — output and reporting

| # | Command | How to use |
|---|---------|------------|
| 57 | `nuclei -l live.txt -o findings.txt` | Plain findings |
| 58 | `nuclei -l live.txt -j -o findings.jsonl` | JSONL |
| 59 | `nuclei -l live.txt -je findings.json` | JSON export |
| 60 | `nuclei -l live.txt -jle findings.jsonl` | JSONL export |
| 61 | `nuclei -l live.txt -se findings.sarif` | SARIF |
| 62 | `nuclei -l live.txt -me reports/` | Markdown export |
| 63 | `nuclei -l live.txt -pe findings.pdf` | PDF export |
| 64 | `nuclei -l live.txt -sresp` | Store all request/response pairs |
| 65 | `nuclei -l live.txt -srd evidence/` | Custom evidence directory |
| 66 | `nuclei -l live.txt -or` | Omit request/response from structured output |
| 67 | `nuclei -l live.txt -ot` | Omit encoded template |
| 68 | `nuclei -l live.txt -rd 'authorization,cookie'` | Redact keys from output |
| 69 | `nuclei -l live.txt -silent` | Findings only |
| 70 | `nuclei -l live.txt -nc` | No ANSI color |
| 71 | `nuclei -l live.txt -nm` | No result metadata |
| 72 | `nuclei -l live.txt -ts` | Timestamps in CLI output |
| 73 | `nuclei -l live.txt -ms` | Show matcher failure status |
| 74 | `nuclei -l live.txt -rdb report.db` | Persist to reporting DB |

**Research questions**
- Which output format is stable enough for a CI gate?
- Does `-rd` redact values before or after writing request/response pairs?

**Sources**: output docs, SARIF schema, issues on redaction.

**Output**: one machine format plus one human format, with evidence retained.

---

## Scenario 7 — network, auth, and TLS

| # | Command | How to use |
|---|---------|------------|
| 75 | `nuclei -l live.txt -H 'Cookie: session=...'` | Custom header |
| 76 | `nuclei -l live.txt -H headers.txt` | Headers from file |
| 77 | `nuclei -l live.txt -r resolvers.txt` | Custom resolvers |
| 78 | `nuclei -l live.txt -sr` | System resolver fallback |
| 79 | `nuclei -l live.txt -cc client.crt -ck client.key -ca ca.pem` | mTLS |
| 80 | `nuclei -l live.txt -sni custom.example.com` | SNI override |
| 81 | `nuclei -l live.txt -i eth0` | Bind to interface |
| 82 | `nuclei -l live.txt -sip 10.0.0.5` | Source IP |
| 83 | `nuclei -l live.txt -lna` | Block local/private network access |
| 84 | `nuclei -l live.txt -lfa` | Allow local file access in payloads |
| 85 | `nuclei -l live.txt -tlsi` | TLS client-hello randomization |
| 86 | `nuclei -l live.txt -ztls` | ztls autofallback (deprecated) |
| 87 | `nuclei -l live.txt -fh2` | Force HTTP/2 |
| 88 | `nuclei -l live.txt -dka 30s` | Dialer keep-alive |
| 89 | `nuclei -l live.txt -dc` | Disable request clustering |
| 90 | `nuclei -l live.txt -passive` | Passive response processing |
| 91 | `nuclei -l live.txt -p http://127.0.0.1:8080` | Proxy |
| 92 | `nuclei -l live.txt -pi` | Proxy internal requests too |
| 93 | `nuclei -l live.txt -config cfg.yaml` | Config file |
| 94 | `nuclei -l live.txt -fr` | Follow redirects |
| 95 | `nuclei -l live.txt -fhr` | Follow same-host redirects |
| 96 | `nuclei -l live.txt -mr 5` | Max redirects |
| 97 | `nuclei -l live.txt -dr` | Disable redirects |
| 98 | `nuclei -l live.txt -sf secrets.yaml` | Authenticated-scan secrets file |
| 99 | `nuclei -l live.txt -ps` | Prefetch secrets |

**Research questions**
- Does `-lna` block only template traffic or also DNS resolution?
- Safe way to keep the API key/secret file out of process listings?

**Sources**: docs network/auth flags, secrets docs, issues on proxy internals.

**Output**: a network profile that cannot reach internal ranges by accident.

---

## Scenario 8 — rate, concurrency, optimization

| # | Command | How to use |
|---|---------|------------|
| 100 | `nuclei -l live.txt -rl 150` | Requests per second |
| 101 | `nuclei -l live.txt -rld 1s` | Rate window |
| 102 | `nuclei -l live.txt -per-host-rate-limit -rl 10` | Per-host limiting |
| 103 | `nuclei -l live.txt -rlm 600` | Per minute (deprecated) |
| 104 | `nuclei -l live.txt -bs 25` | Hosts per template |
| 105 | `nuclei -l live.txt -c 25` | Templates in parallel |
| 106 | `nuclei -l live.txt -hbs 10` | Headless hosts per template |
| 107 | `nuclei -l live.txt -headc 10` | Headless templates in parallel |
| 108 | `nuclei -l live.txt -jsc 120` | JS runtimes |
| 109 | `nuclei -l live.txt -pc 25` | Payload concurrency |
| 110 | `nuclei -l live.txt -prc 50` | Probe concurrency |
| 111 | `nuclei -l live.txt -tlc 50` | Template loading concurrency |
| 112 | `nuclei -l live.txt -timeout 10` | Request timeout |
| 113 | `nuclei -l live.txt -retries 1` | Retry count |
| 114 | `nuclei -l live.txt -ldp` | Keep default ports in URLs |
| 115 | `nuclei -l live.txt -mhe 30` | Max host errors |
| 116 | `nuclei -l live.txt -te timeout,refused` | Track extra error types |
| 117 | `nuclei -l live.txt -nmhe` | Never skip a host on errors |
| 118 | `nuclei -l live.txt -project -project-path /tmp/proj` | Deduplicate requests across runs |
| 119 | `nuclei -l live.txt -spm` | Stop after first match per request |
| 120 | `nuclei -l live.txt -stream` | Stream input, no sorting |
| 121 | `nuclei -l live.txt -ss host-spray` | Scan strategy |
| 122 | `nuclei -l live.txt -irt 3m` | Input read timeout |
| 123 | `nuclei -l live.txt -nh` | Skip httpx probing |
| 124 | `nuclei -l live.txt -preflight-portscan` | Resolve + TCP prefilter |
| 125 | `nuclei -l live.txt -no-stdin` | Ignore stdin |

**Research questions**
- Interaction of `-per-host-rate-limit` with global `-rl` in 3.11?
- When does `-spm` silently reduce coverage?

**Sources**: docs rate limiting, issues on concurrency/429, scan-strategy docs.

**Output**: a rate profile validated against the program's published limits.

---

## Scenario 9 — OAST / interactsh

| # | Command | How to use |
|---|---------|------------|
| 126 | `nuclei -l live.txt -iserver oast.example.com` | Self-hosted interactsh |
| 127 | `nuclei -l live.txt -itoken TOKEN` | Server auth token |
| 128 | `nuclei -l live.txt -interactions-cache-size 5000` | Cache size |
| 129 | `nuclei -l live.txt -interactions-eviction 60` | Eviction window |
| 130 | `nuclei -l live.txt -interactions-poll-duration 5` | Poll interval |
| 131 | `nuclei -l live.txt -interactions-cooldown-period 5` | Extra polling before exit |
| 132 | `nuclei -l live.txt -ni` | Disable OAST templates entirely |

**Research questions**
- Reliability of the public interactsh servers versus self-hosting?
- What data do OAST callbacks reveal to the server operator?

**Sources**: interactsh docs, public-server status, issues on missed callbacks.

**Output**: an OAST setup that does not leak target data.

---

## Scenario 10 — DAST / fuzzing

| # | Command | How to use |
|---|---------|------------|
| 133 | `nuclei -l live.txt -dast` | Run DAST/fuzz templates |
| 134 | `nuclei -l live.txt -ft replace` | Override fuzzing type |
| 135 | `nuclei -l live.txt -fm multiple` | Override fuzzing mode |
| 136 | `nuclei -l live.txt -dast-server -dtsa localhost:9055` | DAST server mode |
| 137 | `nuclei -l live.txt -dast-server -dtst TOKEN` | DAST server token |
| 138 | `nuclei -l live.txt -dast-report report.json` | DAST report |
| 139 | `nuclei -l live.txt -dfp` | Show fuzz points |
| 140 | `nuclei -l live.txt -fuzz-param-frequency 10` | Skip uninteresting params |
| 141 | `nuclei -l live.txt -fa low` | Fuzz aggression |
| 142 | `nuclei -l live.txt -cs '/api/'` | In-scope fuzz regex |
| 143 | `nuclei -l live.txt -cos '/logout/'` | Out-of-scope fuzz regex |
| 144 | `nuclei -l live.txt -fuzz` | Legacy fuzzing flag (deprecated) |

**Research questions**
- Which DAST templates can mutate server state?
- How does aggression map to payload counts?

**Sources**: DAST docs, fuzzing template repo, issues.

**Output**: a fuzz scope boundary and a payload budget.

---

## Scenario 11 — headless

| # | Command | How to use |
|---|---------|------------|
| 145 | `nuclei -l live.txt -headless` | Enable headless templates |
| 146 | `nuclei -l live.txt -headless -page-timeout 30` | Page wait |
| 147 | `nuclei -l live.txt -headless -sb` | Show browser |
| 148 | `nuclei -l live.txt -headless -ho '--no-sandbox'` | Extra Chrome flags |
| 149 | `nuclei -l live.txt -headless -sc` | Use system Chrome |
| 150 | `nuclei -l live.txt -headless -cdpe http://127.0.0.1:9222` | Remote CDP |
| 151 | `nuclei -l live.txt -lha` | List headless actions |

**Research questions**
- Headless Chrome resource cost per concurrent host?
- Does root-run Chrome still disable the sandbox in this build?

**Sources**: headless docs, issues tagged headless, Chrome sandbox docs.

**Output**: a headless budget that will not OOM the host.

---

## Scenario 12 — cover, stats, debug, updates, cloud

| # | Command | How to use |
|---|---------|------------|
| 152 | `nuclei -uc -uq 'ssl:example.com'` | Uncover query |
| 153 | `nuclei -uc -ue shodan -uf ip:port -ul 100` | Engine, fields, limit |
| 154 | `nuclei -uc -ur 60` | Uncover engine rate |
| 155 | `nuclei -l live.txt -stats` | Live statistics |
| 156 | `nuclei -l live.txt -sj` | JSONL statistics |
| 157 | `nuclei -l live.txt -si 10` | Stats interval |
| 158 | `nuclei -l live.txt -mp 9092` | Metrics port |
| 159 | `nuclei -l live.txt -hps` | HTTP status stats |
| 160 | `nuclei -l live.txt -debug` | Full request/response debug |
| 161 | `nuclei -l live.txt -dreq` | Requests only |
| 162 | `nuclei -l live.txt -dresp` | Responses only |
| 163 | `nuclei -l live.txt -ldf` | List DSL functions |
| 164 | `nuclei -l live.txt -tlog trace.log` | Request trace log |
| 165 | `nuclei -l live.txt -elog error.log` | Error log |
| 166 | `nuclei -l live.txt -hm` | Hang monitor |
| 167 | `nuclei -l live.txt -v` | Verbose |
| 168 | `nuclei -l live.txt -vv` | Templates loaded |
| 169 | `nuclei -l live.txt -svd` | Variable dump |
| 170 | `nuclei -l live.txt -vdl 255` | Var dump length limit |
| 171 | `nuclei -l live.txt -ep` | pprof server |
| 172 | `nuclei -l live.txt -profile-mem mem.prof` | Heap profile |
| 173 | `nuclei -tv` | Installed template version |
| 174 | `nuclei -hc` | Health check |
| 175 | `nuclei -version` | Version |
| 176 | `nuclei -up` | Update engine |
| 177 | `nuclei -ut` | Update templates |
| 178 | `nuclei -ud /path/templates` | Custom template dir |
| 179 | `nuclei -duc` | Disable update check |
| 180 | `nuclei -hpd` | Honeypot detection |
| 181 | `nuclei -hpt 15` | Honeypot threshold |
| 182 | `nuclei -shp` | Suppress honeypot hosts |
| 183 | `nuclei -auth` | Configure cloud API key |
| 184 | `nuclei -tid TEAM` | Team ID |
| 185 | `nuclei -cup` | Cloud upload (deprecated) |
| 186 | `nuclei -sid SCAN` | Existing scan ID |
| 187 | `nuclei -sname NAME` | Scan name |
| 188 | `nuclei -pd` | Dashboard upload/view |
| 189 | `nuclei -pdu findings.jsonl` | Upload an existing results file |

**Research questions**
- Which debug flags write credentials into logs?
- What leaves the host when cloud/dashboard upload is enabled?

**Sources**: docs debug/cloud, privacy notes, issues on log leakage.

**Output**: a debug bundle that is safe to share, and cloud use only by
explicit choice.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| Common | `-mt` | 1 |
| Target | `-u`/`-target`, `-l`/`-list`, `-targets-inline`, `-eh`/`-exclude-hosts`, `-resume`, `-sa`/`-scan-all-ips`, `-iv`/`-ip-version` | 1 |
| Target format | `-im`, `-ro`, `-sfv`, `-vtt`, `-vfp` | 5 |
| Templates | `-nt`, `-ntv`, `-as`, `-t`, `-turl`, `-ai`, `-w`, `-wurl`, `-validate`, `-nss`, `-td`, `-tl`, `-tgl`, `-sign`, `-code`, `-dut`, `-esc`, `-egm`, `-file` | 2, 4 |
| Filtering | `-a`, `-tags`, `-etags`, `-itags`, `-id`, `-eid`, `-it`, `-et`, `-em`, `-s`/`-severity`, `-es`, `-pt`, `-ept`, `-tc` | 3 |
| Output | `-o`, `-sresp`, `-srd`, `-silent`, `-nc`, `-j`, `-irr`, `-or`, `-ot`, `-nm`, `-ts`, `-rdb`, `-ms`, `-me`, `-se`, `-je`, `-jle`, `-pe`, `-rd` | 6 |
| Config | `-config`, `-tp`, `-tpl`, `-fr`, `-fhr`, `-mr`, `-dr`, `-rc`, `-H`, `-V`, `-r`, `-sr`, `-dc`, `-passive`, `-fh2`, `-ev`, `-cc`, `-ck`, `-ca`, `-sml`, `-ztls`, `-sni`, `-dka`, `-lfa`, `-lna`, `-i`, `-at`, `-sip`, `-rsr`, `-rss`, `-reset`, `-tlsi`, `-hae` | 7 |
| Interactsh | `-iserver`, `-itoken`, `-interactions-cache-size`, `-interactions-eviction`, `-interactions-poll-duration`, `-interactions-cooldown-period`, `-ni` | 9 |
| Fuzzing | `-ft`, `-fm`, `-fuzz`, `-dast`, `-dts`, `-dtr`, `-dtst`, `-dtsa`, `-dfp`, `-fuzz-param-frequency`, `-fa`, `-cs`, `-cos` | 10 |
| Uncover | `-uc`, `-uq`, `-ue`, `-uf`, `-ul`, `-ur` | 12 |
| Rate | `-rl`, `-rld`, `-per-host-rate-limit`, `-rlm`, `-bs`, `-c`, `-hbs`, `-headc`, `-jsc`, `-pc`, `-prc`, `-tlc` | 8 |
| Optimization | `-timeout`, `-retries`, `-ldp`, `-mhe`, `-te`, `-nmhe`, `-project`, `-project-path`, `-spm`, `-stream`, `-ss`, `-irt`, `-nh`, `-preflight-portscan`, `-no-stdin` | 8 |
| Headless | `-headless`, `-page-timeout`, `-sb`, `-ho`, `-sc`, `-cdpe`, `-lha` | 11 |
| Debug | `-debug`, `-dreq`, `-dresp`, `-p`, `-pi`, `-ldf`, `-tlog`, `-elog`, `-version`, `-hm`, `-v`, `-profile-mem`, `-vv`, `-svd`, `-vdl`, `-ep`, `-tv`, `-hc` | 12 |
| Update | `-up`, `-ut`, `-ud`, `-duc` | 12 |
| Honeypot | `-hpd`, `-hpt`, `-shp` | 12 |
| Statistics | `-stats`, `-sj`, `-si`, `-mp`, `-hps` | 12 |
| Cloud | `-auth`, `-tid`, `-cup`, `-sid`, `-sname`, `-pd`, `-pdu` | 12 |
| Auth | `-sf`, `-ps` | 7 |

Full per-flag text: [`help.txt`](help.txt).
