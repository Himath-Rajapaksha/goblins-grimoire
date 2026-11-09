# corsy — command scenarios for deep research

Every corsy command form available in the installed 1.0-beta build, grouped by
task. Flags come from [`help.txt`](help.txt); the check names and JSON fields
were verified against `/opt/Corsy/core/tests.py` and `/opt/Corsy/db/details.json`.

## How to invoke

```text
@deep-research For corsy 1.0-beta, which of its ten CORS checks are reliable on
CDN-fronted and multi-tenant APIs, and how do real programs rate reflected-origin
versus null-origin findings? Cite upstream code/issues and program policies;
return a triage rule and exact corsy commands.
```

Corsy only sends GET requests. Everything below is still active testing and
requires authorization for the target.

---

## Scenario 1 — single-endpoint triage

Start with one known API endpoint. The goal is a fast yes/no decision: does the
endpoint reflect an attacker origin, and does it allow credentials?

| # | Command | How to use |
|---|---------|------------|
| 1 | `corsy -u https://app.example.com/api/profile` | Full output for one endpoint |
| 2 | `corsy -u https://app.example.com/api/profile -q` | Class + header values only |
| 3 | `corsy -u https://app.example.com/api/profile -o /home/user/findings/corsy-one.json` | Save JSON while printing |
| 4 | `corsy -u http://staging.example.com/api/profile -q` | Plain-HTTP staging target |
| 5 | `corsy -h` | Confirm installed flags before scripting |

**Research questions**
- Does this build test preflight (`OPTIONS`) at all, or only simple GETs?
- What does an empty result mean when the baseline response has no ACAO header?

**Sources**: `help.txt`, `/opt/Corsy/core/tests.py`, upstream README.
**Output**: one finding block or "No misconfigurations found", plus optional JSON.

---

## Scenario 2 — sweep a scope list

Batch input is the normal mode. `-i` reads a file; non-TTY stdin is also read,
so httpx or katana output can be piped directly.

| # | Command | How to use |
|---|---------|------------|
| 6 | `corsy -i /home/user/scope/urls.txt -o /home/user/findings/corsy.json` | File of URLs |
| 7 | `cat /home/user/scope/live.txt \| corsy -t 10` | Pipe live hosts from httpx |
| 8 | `corsy -u https://api.example.com -i /home/user/scope/more.txt` | Combine one URL and a file |
| 9 | `corsy -i /home/user/scope/urls.txt -t 5 -d 0.5 -q` | Moderate concurrency with spacing |

**Research questions**
- How much does the default two-thread setting underuse a wide scope list?
- Does the script open one connection per URL, and does that avoid keep-alive
  fingerprints that WAFs key on?

**Sources**: upstream code, issue threads about threading and timeouts.
**Output**: a single merged JSON keyed by URL.

---

## Scenario 3 — authenticated APIs and custom headers

Most interesting CORS bugs are on authenticated endpoints. Corsy can carry a
session cookie or bearer token. Headers are one string separated by a literal
`\n`.

| # | Command | How to use |
|---|---------|------------|
| 10 | `corsy -u https://app.example.com/api/me -q --headers 'Cookie: session=REDACTED'` | Session cookie |
| 11 | `corsy -u https://app.example.com/api/me -q --headers 'Authorization: Bearer REDACTED'` | Bearer token API |
| 12 | `corsy -u https://app.example.com/api/me --headers 'Cookie: a=b\nX-Tenant: acme' -q` | Multiple headers, literal `\n` |
| 13 | `corsy -i /home/user/scope/api.txt --headers 'Cookie: session=REDACTED' -o /home/user/findings/corsy-auth.json` | Auth sweep |
| 14 | `corsy -u https://app.example.com/api/me --headers` | Interactive header entry (opens `nano`) |

**Research questions**
- Does `Access-Control-Allow-Credentials: true` change severity only when ACAO
  is a specific reflected origin, or also for wildcard?
- Do programs accept "reflected origin without credentials" as a valid finding?

**Sources**: MDN CORS credential rules, program policy pages, PortSwigger CORS
research.
**Output**: findings proving whether credentialed cross-origin reads are possible.

Warning: tokens on the command line land in shell history and process listings.
Prefer a restricted environment or a headers file you delete after the run.

---

## Scenario 4 — tune for fragile targets

Production APIs behind a WAF may rate-limit or block a threaded scan. Serialize
and add delay, then raise concurrency only after a clean pass.

| # | Command | How to use |
|---|---------|------------|
| 15 | `corsy -i /home/user/scope/fragile.txt -t 1 -d 1` | Fully serial, one second apart |
| 16 | `corsy -i /home/user/scope/fragile.txt -t 2 -d 0.5 -q` | Default threads plus delay |
| 17 | `corsy -i /home/user/scope/wide.txt -t 20 -d 0` | Fast pass on a tolerant target |
| 18 | `corsy -u https://app.example.com/api -t 4 -d 2 -q` | Single endpoint under throttling |

**Research questions**
- What request rate triggers common CDN/WAF CORS rules, and does the tool retry?
- Does `-d` sleep between checks within one URL or only between URLs?

**Sources**: upstream source (`time.sleep(delay)` placement), WAF vendor docs.
**Output**: a scan profile that completes without 429/403 noise.

---

## Scenario 5 — manual confirmation and severity triage

Treat corsy output as a lead. Re-request the endpoint with the same forged
origin, then decide whether ACAC, authentication, and data sensitivity make it
reportable.

| # | Command | How to use |
|---|---------|------------|
| 19 | `curl -sS -D - -o /dev/null -H 'Origin: https://example.com' https://app.example.com/api/profile` | Inspect ACAO/ACAC headers |
| 20 | `curl -sS -D - -o /dev/null -H 'Origin: null' https://app.example.com/api/profile` | Verify null-origin acceptance |
| 21 | `curl -sS -D - -o /dev/null -H 'Origin: https://app.example.com_.example.com' https://app.example.com/api/profile` | Verify the underscore bypass |
| 22 | `curl -sS -D - -o /dev/null -H 'Origin: http://app.example.com' https://app.example.com/api/profile` | Verify HTTP-origin acceptance |

**Research questions**
- Which checks produce the most false positives in the field, and why?
- Does the browser actually send credentials for each reported origin scenario?

**Sources**: browser CORS specifications, PortSwigger labs, disclosed reports.
**Output**: a confirmed request/response pair for the report appendix.

---

## Scenario 6 — reporting and regression tracking

Corsy writes JSON only when it finds something, keyed by URL. Snapshot results
per run and diff them after a fix.

| # | Command | How to use |
|---|---------|------------|
| 23 | `mkdir -p /home/user/findings && corsy -i /home/user/scope/api.txt -q -o /home/user/findings/corsy-2026-11-09.json` | Timestamped snapshot |
| 24 | `python3 -m json.tool /home/user/findings/corsy-2026-11-09.json` | Pretty-print/validate |
| 25 | `python3 -c "import json;d=json.load(open('/home/user/findings/corsy-2026-11-09.json'));print(len(d));[print(k,v['class'],v['severity']) for k,v in d.items()]"` | Summarize URL, class, severity |
| 26 | `find /home/user/findings -name 'corsy-*.json' -printf '%TY-%Tm-%Td %p\n' \| sort` | List snapshots for a diff |

**Research questions**
- How should a JSON snapshot be normalized so two runs can be diffed reliably?
- What evidence must accompany a CORS finding for it to be accepted?

**Sources**: program report templates, OWASP WSTG CORS testing guidance.
**Output**: report-ready JSON plus a severity summary table.

---

## Flag coverage checklist

| Help group / flag | Scenario that exercises it |
|-------------------|----------------------------|
| `-u TARGET` | 1, 3, 4 |
| `-i INP_FILE` | 2, 3, 4, 6 |
| stdin (implicit input) | 2 |
| `-o JSON_FILE` | 1, 2, 3, 6 |
| `-q` | 1, 2, 3, 4, 6 |
| `-t THREADS` | 2, 4 |
| `-d DELAY` | 2, 4 |
| `--headers [HEADER_DICT]` | 3 |
| `-h, --help` | 1 |

## Scenario index

| Scenario | Focus | Commands |
|----------|-------|----------|
| 1 | Single-endpoint triage | 5 |
| 2 | Scope-list sweep | 4 |
| 3 | Authenticated APIs | 5 |
| 4 | Rate and concurrency tuning | 4 |
| 5 | Manual confirmation | 4 |
| 6 | Reporting and regression | 4 |
