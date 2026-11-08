# subfinder — command scenarios for deep research

Every subfinder command form, grouped by scenario, with the research question
that decides when to use it. Flags are from v2.17.0 (`help.txt`).

## How to invoke

```text
@deep-research Which subfinder sources still return data without API keys in
v2.17, and what does -all cost versus a pinned source set? Cite docs and
GitHub issues; return a source shortlist and exact flags.
```

---

## Scenario 1 — seed a program's scope

| # | Command | How to use |
|---|---------|------------|
| 1 | `subfinder -d example.com` | One root domain, default sources |
| 2 | `subfinder -d example.com,example.org` | Comma-separated roots |
| 3 | `subfinder -dL domains.txt` | One domain per line from a scope file |
| 4 | `subfinder -dL domains.txt -oD out/` | One output directory per domain |
| 5 | `subfinder -d example.com -silent -o subs.txt` | Clean pipeline seed |

**Research questions**
- Which sources are passive-only and safe for a "no active testing" phase?
- How do programs define scope: `*.example.com` only, or apex + subs?

**Sources**: docs usage/sources, `subfinder -ls`, program policy page.
**Output**: a scope file and a seed command that cannot touch out-of-scope
infrastructure.

---

## Scenario 2 — maximize coverage

| # | Command | How to use |
|---|---------|------------|
| 6 | `subfinder -d example.com -all -silent` | Every source (slow, quota-heavy) |
| 7 | `subfinder -d example.com -s crtsh,github` | Pin specific sources |
| 8 | `subfinder -d example.com -recursive` | Only recursive-capable sources |
| 9 | `subfinder -d example.com -es alienvault,zoomeyeapi` | Exclude noisy/broken sources |
| 10 | `subfinder -d example.com -ls` | List available sources |
| 11 | `subfinder -d example.com -rls hackertarget=10/m` | Per-provider rate limits |

**Research questions**
- Which sources are deprecated or key-gated in 2.17?
- What quotas do GitHub/Shodan/VirusTotal keys impose?
- Does `-all` meaningfully beat `crtsh,github,securitytrails`?

**Sources**: https://docs.projectdiscovery.io/tools/subfinder/usage,
https://github.com/projectdiscovery/subfinder/issues, provider API docs.

---

## Scenario 3 — filter noise before scanning

| # | Command | How to use |
|---|---------|------------|
| 12 | `subfinder -d example.com -m dev,staging` | Substring match filter |
| 13 | `subfinder -d example.com -f test,old` | Drop matches |
| 14 | `subfinder -d example.com -match-regex '^api\.'` | Regex include |
| 15 | `subfinder -d example.com -filter-regex 'cdn|akamai'` | Regex exclude |
| 16 | `subfinder -d example.com -ei` | Drop bare IPs |
| 17 | `subfinder -d example.com -mr 50` | Cap results per source |

**Research questions**
- Does `-m` test the full hostname or only the subdomain label?
- Which filters risk dropping in-scope assets?

**Sources**: docs flags, source code (`runner.go`), issue threads on filtering.

---

## Scenario 4 — validate and hand off

| # | Command | How to use |
|---|---------|------------|
| 18 | `subfinder -d example.com -nW -silent` | Only resolving hosts |
| 19 | `subfinder -d example.com -nW -oI` | Include resolved IP |
| 20 | `subfinder -d example.com -r 1.1.1.1,8.8.8.8` | Custom resolvers |
| 21 | `subfinder -d example.com -rL resolvers.txt` | Resolver list file |
| 22 | `subfinder -d example.com -oJ -cs -o subs.jsonl` | JSONL + source attribution |
| 23 | `subfinder -d example.com -proxy http://127.0.0.1:8080` | Route source queries via proxy |

**Research questions**
- Does `-nW` use system resolvers or `-r`?
- How often do passive results include wildcard DNS noise, and does `dnsx`
  filter it better than `-nW`?

**Sources**: docs, projectdiscovery/dnsx docs, wildcard-DNS writeups.

---

## Scenario 5 — operate and tune

| # | Command | How to use |
|---|---------|------------|
| 24 | `subfinder -dL domains.txt -rl 20 -t 5` | Global rate + concurrency |
| 25 | `subfinder -d example.com -timeout 20 -max-time 5` | Request and total caps |
| 26 | `subfinder -d example.com -rsr 2000000` | Cap source response size |
| 27 | `subfinder -config cfg.yaml -pc provider.yaml` | Custom config/keys |
| 28 | `subfinder -up` | Self-update |
| 29 | `subfinder -d example.com -duc` | Skip update check |
| 30 | `subfinder -d example.com -stats -v` | Statistics + verbose |
| 31 | `subfinder -d example.com -nc` | No color |
| 32 | `subfinder -version` | Version |

**Research questions**
- Does a mid-run `-up` break long scans?
- What are sane `-rl`/`-t` values for a shared VPS IP?

**Sources**: docs, release notes, issue threads on rate limiting.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| Input | `-d`, `-dL` | 1 |
| Sources | `-s`, `-recursive`, `-all`, `-es`, `-ls`, `-rls` | 2 |
| Filtering | `-m`, `-f`, `-match-regex`, `-filter-regex`, `-ei`, `-mr` | 3 |
| Resolution | `-nW`/`-active`, `-oI`/`-ip`, `-r`, `-rL`, `-proxy` | 4 |
| Output | `-o`, `-oJ`/`-json`, `-oD`, `-cs`, `-silent`, `-nc` | 1, 4 |
| Performance | `-rl`, `-rls`, `-t`, `-timeout`, `-max-time`, `-rsr` | 5 |
| Config/maint | `-config`, `-pc`, `-up`, `-duc`, `-stats`, `-v`, `-version` | 5 |

Full per-flag text: [`help.txt`](help.txt).
