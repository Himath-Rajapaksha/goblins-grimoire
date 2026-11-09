# ParamSpider — command scenarios for deep research

Installed build: packaged Parameter Spider. Flags come from `help.txt`.

## How to invoke

```text
@deep-research For ParamSpider in 2026, which archive sources still return
usable parameterised URLs, how should placeholder substitution handle encoded
and multi-value parameters, and what deduplication/filtering gives the highest
signal before fuzzing? Cite the ParamSpider repo, archive API docs, and
bug-bounty field reports; return a reliable mining and filtering workflow.
```

---

## Scenario 1 — single-domain mining

| # | Command | How to use |
|---|---------|------------|
| 1 | `paramspider -d example.com` | Default run |
| 2 | `paramspider -d example.com -s` | Stream to terminal |
| 3 | `cat results/example.com.txt` | Inspect saved output |
| 4 | `paramspider -d example.com -p FUZZ` | Custom placeholder |
| 5 | `wc -l results/example.com.txt` | Count URLs |

**Research questions**
- Which archive sources does this build query, and in what order?
- How are URLs with fragments or repeated parameters handled?

**Sources:** source, output, archive docs.
**Output:** a parameterised URL list.

---

## Scenario 2 — batch and scoping

| # | Command | How to use |
|---|---------|------------|
| 6 | `paramspider -l domains.txt` | Multiple domains |
| 7 | `paramspider -l domains.txt -s \| sort -u > all.txt` | Merge and dedupe |
| 8 | `grep '^https://example.com/' all.txt` | Scope to one host |
| 9 | `grep -v 'example.com' all.txt` | Find out-of-scope hosts |
| 10 | `awk -F'?' '{print $1}' all.txt \| sort -u` | Unique paths |

**Research questions**
- Does `-l` write one results file per domain?
- How do subdomains appear in output?

**Sources:** output layout, source.
**Output:** a scoped, deduplicated endpoint map.

---

## Scenario 3 — placeholder and encoding

| # | Command | How to use |
|---|---------|------------|
| 11 | `paramspider -d example.com -p INJECT` | Different placeholder |
| 12 | `paramspider -d example.com -s \| grep '='` | Only parameterised URLs |
| 13 | `paramspider -d example.com -s \| grep -c '%'` | Encoded values |
| 14 | `paramspider -d example.com -s \| grep '?'` | Query strings |
| 15 | `paramspider -d example.com -s \| grep -E '[?&][a-z]+='` | Named params |

**Research questions**
- How does the tool decide what counts as a value to replace?
- Are array parameters (`a[]=1&a[]=2`) preserved?

**Sources:** source, output samples.
**Output:** URLs that a fuzzer will parse correctly.

---

## Scenario 4 — proxying and rate limits

| # | Command | How to use |
|---|---------|------------|
| 16 | `paramspider -d example.com --proxy http://127.0.0.1:8080 -s` | Burp proxy |
| 17 | `paramspider -d example.com --proxy socks5://127.0.0.1:1080 -s` | SOCKS |
| 18 | `date; paramspider -d example.com; date` | Measure runtime |
| 19 | `paramspider -d example.com -s > run.log 2>&1` | Capture errors |
| 20 | `grep -i 'error\|timeout' run.log` | Diagnose failures |

**Research questions**
- Which requests go through the proxy — archive APIs or target?
- What retry behaviour applies when an archive returns 429?

**Sources:** proxy logs, source, run logs.
**Output:** a proxy-audited, rate-aware run.

---

## Scenario 5 — handoff and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `paramspider -d example.com -s \| sort -u > params.txt` | Final list |
| 22 | `ffuf -w params.txt -u 'https://example.com/FUZZ' -mc 200,301,302` | Fuzz |
| 23 | `httpx -l params.txt -silent -mc 200` | Validate live URLs |
| 24 | `sha256sum params.txt` | Hash |
| 25 | `wc -l params.txt` | Report size |

**Research questions**
- Which validation step best removes dead historical URLs?
- What provenance should be recorded for archive-derived endpoints?

**Sources:** validation output, engagement policy.
**Output:** a validated parameter list and hashed evidence.

## Flag coverage checklist

| Flag | Scenario |
|------|----------|
| `-d` | 1–4 |
| `-l` | 2 |
| `-s` | 1–5 |
| `-p` | 1, 3 |
| `--proxy` | 4 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — mining | 5 |
| 2 — batch | 5 |
| 3 — placeholders | 5 |
| 4 — proxy | 5 |
| 5 — handoff | 5 |
