# nuclei — template-driven vulnerability scanner

- **Version:** v3.11.1
- **Binary:** `/usr/local/bin/nuclei`
- **Templates:** `~/nuclei-templates` (install/update with `nuclei -ut`)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/projectdiscovery/nuclei
- **Docs:** https://docs.nuclei.sh/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Nuclei sends YAML-defined requests and evaluates matchers to detect CVEs,
misconfigurations, exposed panels, default credentials, takeovers, and more.
Templates can speak HTTP, DNS, TCP, SSL, WebSocket, WHOIS, headless browser,
JavaScript, and code. It is the de-facto fast triage scanner for large asset
lists.

## Use cases

| Situation | Flags |
|-----------|-------|
| Broad CVE/misconfig triage | `-severity low,medium,high,critical` |
| Only critical/high | `-s critical,high` |
| Tech-aware automatic scan | `-as` |
| Specific template family | `-t http/cves/ -t ssl` |
| CVE by ID | `-id CVE-2024-...` |
| Tag-based (e.g. WordPress) | `-tags wordpress,wp-plugin` |
| DAST/fuzzing | `-dast` |
| Blind/OAST findings | interactsh enabled by default |
| CI gating | `-jsonl -severity high,critical` + exit code |
| Re-scan only new templates | `-nt` |
| Bug-bounty safe mode | lower `-rl`, `-c`, `-bs`, `-timeout` |

## Basic syntax

```bash
nuclei -u https://example.com
nuclei -l live.txt -severity high,critical
nuclei -l live.txt -t http/exposures/ -silent
```

## Flag reference

### Target input
| Flag | Meaning |
|------|---------|
| `-u, -target string[]` | Target URL/host |
| `-l, -list string` | Target list file |
| `-eh, -exclude-hosts` | Exclude IP/CIDR/hostname |
| `-resume string` | Resume scan from state file |
| `-sa, -scan-all-ips` | Scan all A records |
| `-iv, -ip-version` | `4` or `6` |
| `-im, -input-mode` | `list`, `burp`, `jsonl`, `yaml`, `openapi`, `swagger` |
| `-ro` | Required fields only |
| `-sfv` | Skip input format validation |

### Template selection
| Flag | Meaning |
|------|---------|
| `-t, -templates` | Template/dir (repeatable/comma) |
| `-turl` | Remote template URL |
| `-w, -workflows` | Workflow file/dir |
| `-nt` / `-ntv` | New templates / by version |
| `-as, -automatic-scan` | Wappalyzer-driven tag mapping |
| `-tl` | List matching templates |
| `-tgl` | List tags |
| `-validate` | Validate templates |
| `-td` | Display template content |
| `-code` | Enable code-protocol templates |
| `-dut` | Disable unsigned templates |
| `-esc`, `-egm`, `-file` | Self-contained / global matchers / file templates |

### Filtering
| Flag | Meaning |
|------|---------|
| `-a, -author` | By author |
| `-tags` / `-etags` / `-itags` | Include/exclude/force-include tags |
| `-id` / `-eid` | Template ID include/exclude (wildcards ok) |
| `-it` / `-et` | Include/exclude template path |
| `-em` | Exclude matchers |
| `-s, -severity` | info, low, medium, high, critical, unknown |
| `-es, -exclude-severity` | Exclude severities |
| `-pt, -type` | dns, file, http, headless, tcp, workflow, ssl, websocket, whois, code, javascript |
| `-ept` | Exclude protocol types |
| `-tc` | Template-condition expression |

### Output
| Flag | Meaning |
|------|---------|
| `-o, -output` | Findings file |
| `-silent` | Findings only |
| `-nc` | No color |
| `-j, -jsonl` | JSONL stdout |
| `-or, -omit-raw` | Omit request/response pairs |
| `-ot, -omit-template` | Omit encoded template |
| `-nm` | No metadata |
| `-ts` | Timestamps |
| `-rdb` | Reporting DB |
| `-ms` | Matcher failure status |
| `-me` | Markdown export dir |
| `-se` | SARIF export |
| `-je`, `-jle` | JSON / JSONL export file |
| `-pe` | PDF export |
| `-rd` | Redact keys from output |
| `-sresp` / `-srd` | Store responses / to dir |

### Configuration & request options
| Flag | Meaning |
|------|---------|
| `-config` | Config file |
| `-tp` / `-tpl` | Template profile / list profiles |
| `-fr` / `-fhr` / `-mr` / `-dr` | Follow redirects / same-host / max / disable |
| `-H` | Custom headers |
| `-V` | Custom vars `key=value` |
| `-r` / `-sr` | Resolvers / system fallback |
| `-dc` | Disable request clustering |
| `-passive` | Passive HTTP response processing |
| `-fh2` | Force HTTP/2 |
| `-ev` | Use env vars in templates |
| `-cc`, `-ck`, `-ca` | Client cert/key/CA |
| `-sni` | TLS SNI override |
| `-lfa` | Allow local file access |
| `-lna` | Restrict local/private network access |
| `-i` | Interface |
| `-at` | Attack type: batteringram, pitchfork, clusterbomb |
| `-sip` | Source IP |
| `-rsr` / `-rss` | Response read/save size |
| `-tlsi` | TLS hello randomization |

### Interactsh (OAST)
| Flag | Meaning |
|------|---------|
| `-iserver` | Self-hosted interactsh server |
| `-itoken` | Auth token |
| `-interactions-cache-size`, `-interactions-eviction`, `-interactions-poll-duration`, `-interactions-cooldown-period` | Polling tuning |
| `-ni, -no-interactsh` | Disable OAST templates |

### DAST / fuzzing
| Flag | Meaning |
|------|---------|
| `-dast` | DAST/fuzz templates |
| `-ft` | Fuzz type: replace/prefix/postfix/infix |
| `-fm` | Fuzz mode: multiple/single |
| `-fa` | Aggression: low/medium/high |
| `-cs` / `-cos` | In-scope / out-of-scope regex |
| `-dfp` | Display fuzz points |
| `-dts`, `-dtr`, `-dtst`, `-dtsa` | DAST server/report/token/address |

### Rate limits & performance
| Flag | Meaning |
|------|---------|
| `-rl` | Requests/second (default 150) |
| `-rld` | Rate window (default 1s) |
| `-per-host-rate-limit` | Per-host limiting |
| `-bs` | Bulk size/hosts per template (default 25) |
| `-c` | Templates in parallel (default 25) |
| `-hbs` / `-headc` | Headless bulk / concurrency |
| `-jsc` | JS runtimes |
| `-pc` / `-prc` | Payload / probe concurrency |
| `-timeout` | Request timeout (default 10s) |
| `-retries` | Retries (default 1) |
| `-mhe` | Max errors per host (default 30) |
| `-project` / `-project-path` | Deduplicate requests across templates |
| `-spm` | Stop at first match |
| `-stream` | Stream input without sorting |
| `-ss` | Scan strategy auto/host-spray/template-spray |
| `-nh` | Disable httpx probing |
| `-preflight-portscan` | Resolve + TCP portscan prefilter |

### Headless
`-headless`, `-page-timeout` (default 20), `-sb` show browser, `-ho` options,
`-sc` system Chrome, `-cdpe` CDP endpoint, `-lha` list headless actions.

### Debug / stats / maintenance
`-debug`, `-dreq`, `-dresp`, `-p` proxy (comma/file), `-pi` proxy internal,
`-ldf` list DSL functions, `-tlog`, `-elog`, `-hm` hang monitor, `-v`, `-vv`,
`-svd`, `-ep` pprof, `-tv` templates version, `-hc` health check,
`-stats`, `-sj`, `-si`, `-mp`, `-up`, `-ut`, `-ud`, `-duc`,
`-hpd` honeypot detect, `-hpt`, `-shp` suppress.

## Writing a minimal template

```yaml
id: example-detect

info:
  name: Example Panel Detect
  author: your-handle
  severity: info
  tags: panel,example

http:
  - method: GET
    path:
      - "{{BaseURL}}/admin"
    matchers-condition: and
    matchers:
      - type: status
        status: [200]
      - type: word
        words: ["Example Admin"]
        part: body
```

Validate and run:

```bash
nuclei -validate -t my-template.yaml
nuclei -u https://target -t my-template.yaml -debug
```

## Recipes

```bash
# Update engine + templates
nuclei -up && nuclei -ut

# Safe broad triage
nuclei -l live.txt -severity low,medium,high,critical -rl 50 -c 10 -timeout 15 -o findings.txt

# Critical/high only, JSONL for tooling
nuclei -l live.txt -s critical,high -jsonl -o crit.jsonl -silent

# Specific tech tags
nuclei -l live.txt -tags wordpress,joomla,drupal -silent

# CVE family
nuclei -l live.txt -id 'CVE-2023-*' -silent

# Exposures + misconfigurations only
nuclei -l live.txt -t http/exposures/ -t http/misconfiguration/ -silent

# Headless checks (DOM XSS, panels)
nuclei -l live.txt -headless -t http/headless/ -silent

# Blind SSRF/XSS with OAST (default interactsh)
nuclei -l live.txt -t http/vulnerabilities/ -ni=false -silent

# CI gate: fail on high/critical (nuclei exits non-zero on findings)
nuclei -l live.txt -s high,critical -silent -jsonl -o out.jsonl

# DAST pass on one parameterized URL
nuclei -u 'https://target/search?q=test' -dast -fa low -silent
```

## Template directories worth knowing

```
http/cves/                 # CVE checks
http/exposures/            # exposed files, panels, keys
http/misconfiguration/     # headers, debug modes, defaults
http/vulnerabilities/      # generic classes (SSRF, LFI, XSS helpers)
http/takeovers/            # subdomain takeover
http/technologies/         # fingerprinting
http/headless/             # browser-based checks
dns/ tcp/ ssl/ javascript/ # protocol-specific
workflows/                 # multi-template chains
```

## Gotchas

- Default rate is 150 rps; shared IPs and bug-bounty programs often require lower.
- `-as` needs Wappalyzer data and is noisier/slower than targeted templates.
- Interactsh needs outbound DNS/HTTP; corporate networks may block it.
- `-dast` sends fuzz payloads — confirm scope first.
- Template updates can add aggressive checks; pin/limit with `-t` and severity.
- Exit code is non-zero on findings (useful for CI, surprising in scripts).
- Old/signed-template changes can break custom workflows; run `-validate`.

## References

- Docs: https://docs.nuclei.sh/
- Templates: https://github.com/projectdiscovery/nuclei-templates
- Template syntax: https://docs.projectdiscovery.io/templates/
- Source: https://github.com/projectdiscovery/nuclei
