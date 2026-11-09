# SpiderFoot — command scenarios for deep research

Installed build: SpiderFoot 4.0.0. Flags come from `help.txt`; module and
profile behaviour is documented upstream.

## How to invoke

```text
@deep-research For SpiderFoot 4.0.0, which module profiles (passive,
footprint, investigate, all) and event-type selections give the best
coverage-to-noise ratio for an external domain assessment without API keys,
and how does strict mode change chaining? Cite SpiderFoot docs, module source,
and field reports; return a profile choice matrix and a CLI command.
```

---

## Scenario 1 — discovery of modules and types

| # | Command | How to use |
|---|---------|------------|
| 1 | `spiderfoot -M` | List all modules |
| 2 | `spiderfoot -T` | List all event types |
| 3 | `spiderfoot -l 127.0.0.1:5001` | Start the web UI |
| 4 | `spiderfoot -V` | Record version |
| 5 | `curl -s http://127.0.0.1:5001/api/v1/...` | Probe the API |

**Research questions**
- Which modules are key-gated versus keyless in this build?
- Which event types are consumed by the most modules?

**Sources:** `-M`, `-T`, module source, API docs.
**Output:** a shortlist of modules and event types.

---

## Scenario 2 — passive scan

| # | Command | How to use |
|---|---------|------------|
| 6 | `spiderfoot -s example.com -u passive -o json > passive.json` | Passive profile |
| 7 | `spiderfoot -s example.com -m sfp_dnsresolve,sfp_crt -o tab` | Two modules only |
| 8 | `spiderfoot -s example.com -t IP_ADDRESS,EMAILADDR -o tab` | Event-type driven |
| 9 | `spiderfoot -s example.com -u passive -max-threads 4 -o csv` | Throttled |
| 10 | `spiderfoot -s example.com -u passive -q -o json` | Quiet logging |

**Research questions**
- Which passive modules still work without keys in 2026?
- Does `-t` override `-m`, or combine with it?

**Sources:** docs, module source, output comparison.
**Output:** a passive footprint with no target contact.

---

## Scenario 3 — footprint and investigate profiles

| # | Command | How to use |
|---|---------|------------|
| 11 | `spiderfoot -s example.com -u footprint -o csv > footprint.csv` | Deeper scan |
| 12 | `spiderfoot -s example.com -u investigate -o json` | Investigative profile |
| 13 | `spiderfoot -s example.com -u all -o json` | Everything (slow) |
| 14 | `spiderfoot -s example.com -u footprint -o csv -D ';'` | Semicolon CSV |
| 15 | `spiderfoot -s example.com -u footprint -r -o tab` | Include source data |

**Research questions**
- Which active modules does `footprint` enable, and are they scope-safe?
- How long does `all` take on a median domain?

**Sources:** profile definitions, run timing, module docs.
**Output:** a deeper asset map with active findings.

---

## Scenario 4 — strict mode and correlation

| # | Command | How to use |
|---|---------|------------|
| 16 | `spiderfoot -s example.com -x -o json` | Strict mode |
| 17 | `spiderfoot -s example.com -x -t IP_ADDRESS -o tab` | Strict plus type |
| 18 | `spiderfoot -s example.com -u footprint -f -o json` | Filter to requested types |
| 19 | `spiderfoot -s example.com -u footprint -F IP_ADDRESS,EMAILADDR` | Show only these |
| 20 | `spiderfoot -C 1A2B3C4D5E6F` | Correlate a stored scan |

**Research questions**
- What chaining is lost under `-x`, and when is that acceptable?
- Which correlation rules fire most often, and how noisy are they?

**Sources:** strict-mode docs, correlation rule source, scan output.
**Output:** a precise, low-noise result set.

---

## Scenario 5 — output, API, and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `spiderfoot -s example.com -u footprint -o csv -H -n > clean.csv` | Headerless, single-line |
| 22 | `spiderfoot -s example.com -u footprint -o csv -S 200` | Truncate long fields |
| 23 | `sqlite3 spiderfoot.db '.tables'` | Inspect the database |
| 24 | `sqlite3 spiderfoot.db 'SELECT * FROM tbl_scan_results LIMIT 10;'` | Query results |
| 25 | `sha256sum scan.json` | Hash evidence |

**Research questions**
- Which table stores scan metadata versus events?
- Does truncation (`-S`) affect the stored data or only display?

**Sources:** DB schema, output comparison, docs.
**Output:** a clean CSV/JSON bundle and a verifiable evidence hash.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Discovery | `-M -T -V -l` | 1 |
| Targeting | `-s -m -t -u -x` | 2–4 |
| Output | `-o -H -n -r -S -D -f -F -q` | 3–5 |
| Ops | `-d -max-threads -C` | 4–5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — discovery | 5 |
| 2 — passive | 5 |
| 3 — footprint | 5 |
| 4 — strict/correlation | 5 |
| 5 — output/evidence | 5 |
