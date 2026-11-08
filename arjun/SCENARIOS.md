# arjun — command scenarios for deep research

Every arjun command form, grouped by scenario. Flags from the installed build
(`help.txt`).

## How to invoke

```text
@deep-research For arjun, how do passive sources (wayback, commoncrawl, otx)
compare with the active wordlist for parameter discovery on modern SPAs, and
what chunk size avoids WAF rate limits? Cite docs, GitHub issues, and
independent benchmarks; return exact flags.
```

---

## Scenario 1 — discover parameters on one URL

| # | Command | How to use |
|---|---------|------------|
| 1 | `arjun -u https://example.com/app` | Default GET discovery, built-in large wordlist |
| 2 | `arjun -u https://example.com/app -q` | Quiet: hits only, no progress noise |
| 3 | `arjun -u https://example.com/app -oJ params.json` | JSON output for downstream tooling |
| 4 | `arjun -u https://example.com/app -oT params.txt` | Plain text output for pipelines |
| 5 | `arjun -u https://example.com/app -oJ params.json -oT params.txt` | Both formats in one run |

**Research questions**
- How does arjun decide a parameter is valid — status, length, reflection, or
  a mix — and what is its false-positive rate?
- Do modern SPA backends return stable responses for unknown params, defeating
  the comparison?

**Sources**: https://github.com/s0md3v/Arjun docs, source (`core/`),
issues tagged false-positive.

**Output**: a parameter list plus the evidence rule arjun used, so findings can
be re-verified manually.

---

## Scenario 2 — method matrix

| # | Command | How to use |
|---|---------|------------|
| 6 | `arjun -u https://example.com/api -m GET` | Query-string parameters |
| 7 | `arjun -u https://example.com/api -m POST` | Form-encoded body parameters |
| 8 | `arjun -u https://example.com/api -m JSON` | JSON body keys |
| 9 | `arjun -u https://example.com/api -m XML` | XML element names |
| 10 | `arjun -u https://example.com/api -m POST --include 'csrf=token'` | Keep a required field in every request |
| 11 | `arjun -u https://example.com/api --casing like_this` | Force snake_case probing |
| 12 | `arjun -u https://example.com/api --casing likeThis` | Force camelCase probing |
| 13 | `arjun -u https://example.com/api --casing likethis` | Force lowercase probing |

**Research questions**
- Does `--casing` transform the wordlist before sending, or send all variants?
- Which content-type does arjun set for `-m JSON`, and does it matter for
  strict APIs?

**Sources**: docs usage, source request builders, issues on JSON/XML support.

**Output**: one method/casing combination per endpoint, not all four blindly —
pick based on how the app accepts input.

---

## Scenario 3 — bulk targets and imports

| # | Command | How to use |
|---|---------|------------|
| 14 | `arjun -i targets.txt` | One target URL per line |
| 15 | `arjun -i targets.txt -oJ all-params.json` | Bulk run, combined JSON output |
| 16 | `arjun -u https://example.com -i extra.txt` | Combine a primary URL with a file |

**Research questions**
- Does `-i` with no argument prompt interactively, and is that usable in
  scripts?
- Are duplicate hosts deduplicated before scanning?

**Sources**: docs, source CLI parser, issues on bulk mode.

**Output**: a deduplicated target list and one output file per batch.

---

## Scenario 4 — passive collection

| # | Command | How to use |
|---|---------|------------|
| 17 | `arjun -u https://example.com --passive wayback` | Wayback Machine parameter names |
| 18 | `arjun -u https://example.com --passive commoncrawl` | Common Crawl index |
| 19 | `arjun -u https://example.com --passive otx` | AlienVault OTX |
| 20 | `arjun -u https://example.com --passive wayback,commoncrawl,otx` | All three at once |

**Research questions**
- Does `--passive` replace or augment active probing?
- How stale are wayback/OTX parameter names for a fast-moving app?

**Sources**: docs passive mode, source providers, issues on passive coverage.

**Output**: passive names merged into the active wordlist, with provenance
noted for each.

---

## Scenario 5 — request shaping and auth

| # | Command | How to use |
|---|---------|------------|
| 21 | `arjun -u https://example.com --headers 'Cookie: session=...'` | Authenticated discovery |
| 22 | `arjun -u https://example.com --headers 'Authorization: Bearer ...'` | Token auth |
| 23 | `arjun -u https://example.com --headers $'X-One: a\nX-Two: b'` | Multiple headers, newline-separated |
| 24 | `arjun -u https://example.com --disable-redirects` | Do not follow 3xx |
| 25 | `arjun -u https://example.com -T 30` | Longer per-request timeout |

**Research questions**
- Are headers applied to every probe and to the baseline comparison request?
- How do redirects distort the baseline when `--disable-redirects` is off?

**Sources**: docs headers, source HTTP layer, issues on auth.

**Output**: an authenticated parameter map, with the exact header set recorded
outside the doc (never store live tokens here).

---

## Scenario 6 — stealth, speed, and stability

| # | Command | How to use |
|---|---------|------------|
| 26 | `arjun -u https://example.com -t 2 -d 1` | Low concurrency, 1s delay |
| 27 | `arjun -u https://example.com --rate-limit 10` | Hard cap of 10 req/s |
| 28 | `arjun -u https://example.com --stable` | Favor reliability over speed |
| 29 | `arjun -u https://example.com -c 50` | 50 parameters per request |
| 30 | `arjun -u https://example.com -w custom.txt` | Custom wordlist |
| 31 | `arjun -u https://example.com -w custom.txt -c 200` | Big chunks with a focused list |

**Research questions**
- How do `-t`, `-d`, and `--rate-limit` interact — is the delay per thread or
  global?
- What chunk size triggers URL-length or body-size rejections?

**Sources**: docs performance, source scheduler, program rate rules, issues on
429/403 handling.

**Output**: a rate profile that stays inside the program's rules and does not
trip WAFs.

---

## Scenario 7 — Burp handoff

| # | Command | How to use |
|---|---------|------------|
| 32 | `arjun -u https://example.com -oB` | Send discovered requests to Burp at 127.0.0.1:8080 |
| 33 | `arjun -u https://example.com -oB 127.0.0.1:8081` | Custom Burp listener |

**Research questions**
- What exactly does `-oB` export — requests with found params, or all probes?
- Does `-oB` require Burp's proxy listener to be running first?

**Sources**: docs Burp integration, source exporter, Burp proxy docs.

**Output**: a Burp history populated with parameter-bearing requests ready for
Repeater/Intruder.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| Target | `-u`, `-i` | 1, 3 |
| Method | `-m` | 2 |
| Wordlist | `-w` | 6 |
| Request | `--headers`, `--include`, `--disable-redirects`, `--casing` | 2, 5 |
| Passive | `--passive` | 4 |
| Performance | `-t`, `-d`, `-T`, `-c`, `--rate-limit`, `--stable` | 6 |
| Output | `-o`/`-oJ`, `-oT`, `-oB` | 1, 7 |
| Misc | `-q`, `-h` | 1 |

Full per-flag text: [`help.txt`](help.txt).
