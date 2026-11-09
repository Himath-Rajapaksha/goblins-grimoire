# sherlock — command scenarios for deep research

Installed build: sherlock 0.16.2. Flags come from `help.txt`; site coverage
comes from the bundled `data.json`.

## How to invoke

```text
@deep-research For sherlock 0.16.2 in 2026, which sites in data.json produce
the most false positives and false negatives, how do Cloudflare and JavaScript
challenges affect detection, and what --site/--timeout/--proxy combination
gives the most reliable footprint? Cite the sherlock repo, issue tracker, and
site behaviour; return a per-site reliability shortlist and a reproducible run.
```

---

## Scenario 1 — basic and batch checks

| # | Command | How to use |
|---|---------|------------|
| 1 | `sherlock targetuser` | Default full check |
| 2 | `sherlock --print-found targetuser` | Hits only |
| 3 | `sherlock -fo evidence alice bob` | Batch to folder |
| 4 | `sherlock -o target.txt targetuser` | Single output file |
| 5 | `sherlock --no-color targetuser \| tee sherlock.log` | Log-friendly |

**Research questions**
- How many sites does the bundled data.json cover in this build?
- Does batch mode reuse connections or serialize per username?

**Sources:** data.json, source code, run timing.
**Output:** a hit list per username.

---

## Scenario 2 — variant expansion

| # | Command | How to use |
|---|---------|------------|
| 6 | `sherlock 'target{?}user'` | Expand `_`, `-`, `.` |
| 7 | `sherlock 'target{?}user' --print-found` | Variants, hits only |
| 8 | `sherlock 'target{?}user' --csv` | Export variants |
| 9 | `ls -la` | Inspect generated files |
| 10 | `grep -h 'http' *.txt \| sort -u` | Merge URLs |

**Research questions**
- Which variant separators are most common on which platforms?
- Does expansion multiply runtime linearly?

**Sources:** source code, output, platform conventions.
**Output:** additional aliases linked to the same person.

---

## Scenario 3 — scoping and reliability

| # | Command | How to use |
|---|---------|------------|
| 11 | `sherlock --site GitHub --site Reddit targetuser` | Narrow check |
| 12 | `sherlock --site GitHub --dump-response targetuser` | Debug a miss |
| 13 | `sherlock --timeout 90 targetuser` | Slow sites |
| 14 | `sherlock --ignore-exclusions --print-all targetuser` | Max coverage |
| 15 | `sherlock --local targetuser` | Force bundled definitions |

**Research questions**
- Which sites return 200 for missing users (false positives)?
- Which require JS/Cloudflare and always miss (false negatives)?

**Sources:** raw responses, issue tracker, site behaviour.
**Output:** a reliability-ranked site list.

---

## Scenario 4 — proxies, privacy, and NSFW

| # | Command | How to use |
|---|---------|------------|
| 16 | `sherlock -p socks5://127.0.0.1:1080 targetuser` | Route via SOCKS |
| 17 | `sherlock -p http://127.0.0.1:8080 targetuser` | HTTP proxy |
| 18 | `sherlock --nsfw targetuser` | Include NSFW sites |
| 19 | `sherlock --nsfw --print-found targetuser` | NSFW hits only |
| 20 | `sherlock --browse targetuser` | Open hits (careful) |

**Research questions**
- Does sherlock honour proxy settings for all request types?
- What privacy risk does `--browse` create (referrers, cookies)?

**Sources:** source code, proxy logs, browser behaviour.
**Output:** a privacy-preserving run and a scoped NSFW result.

---

## Scenario 5 — evidence and handoff

| # | Command | How to use |
|---|---------|------------|
| 21 | `sherlock --csv --txt -fo /evidence targetuser` | Structured evidence |
| 22 | `sha256sum /evidence/*` | Hash evidence |
| 23 | `wc -l /evidence/*.txt` | Count findings |
| 24 | `grep -c 'http' /evidence/*.csv` | Sanity check |
| 25 | `sherlock --version` | Record version |

**Research questions**
- Which export format preserves the most provenance (URL, status, timestamp)?
- What retention applies to identity-linked data?

**Sources:** output files, engagement policy.
**Output:** a hashed evidence bundle ready for reporting.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Targets | usernames, `{?}` | 1, 2 |
| Output | `-o -fo --csv --xlsx --txt --print-all --print-found --no-color` | 1, 5 |
| Control | `--site --timeout --proxy --json --local --ignore-exclusions --dump-response` | 3, 4 |
| Behaviour | `--nsfw --browse -v` | 4 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — basic/batch | 5 |
| 2 — variants | 5 |
| 3 — scoping | 5 |
| 4 — proxy/NSFW | 5 |
| 5 — evidence | 5 |
