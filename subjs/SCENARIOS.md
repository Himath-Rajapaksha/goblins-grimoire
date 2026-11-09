# subjs — command scenarios for deep research

Installed build: subjs 1.0.1. Flags come from `help.txt`; the tool has a small
surface and no configuration file.

## How to invoke

```text
@deep-research For subjs 1.0.1, how complete is HTML-only JavaScript discovery
compared with browser network capture and crawler-based collection, what
concurrency and timeout settings balance coverage against WAF blocking, and how
should JavaScript URL inventories be deduplicated and validated before source
analysis? Cite the subjs repo, gau/LinkFinder workflows, and bug-bounty field
reports; return a coverage comparison and a hardened recon pipeline.
```

---

## Scenario 1 — baseline discovery

| # | Command | How to use |
|---|---------|------------|
| 1 | `cat urls.txt \| subjs` | Stdin input |
| 2 | `subjs -i urls.txt` | File input |
| 3 | `subjs -version` | Confirm build |
| 4 | `subjs -i urls.txt \| sort -u > js.txt` | Deduplicate |
| 5 | `wc -l urls.txt js.txt` | Compare input and output counts |

**Research questions**
- What fraction of supplied pages yield at least one script URL?
- Which pages return no scripts because of redirects, login walls, or empty
  bodies?

**Sources:** subjs output, HTTP status from `httpx`, page HTML.
**Output:** a first-pass JavaScript inventory with coverage counts.

---

## Scenario 2 — concurrency and timeout tuning

| # | Command | How to use |
|---|---------|------------|
| 6 | `subjs -i urls.txt -c 10 -t 15` | Defaults |
| 7 | `subjs -i urls.txt -c 40 -t 20` | Faster, more parallel |
| 8 | `subjs -i urls.txt -c 5 -t 30` | Gentle on slow hosts |
| 9 | `time subjs -i urls.txt -c 40 -t 10 > /dev/null` | Measure runtime |
| 10 | `subjs -i urls.txt -c 40 -t 3` | Deliberately short timeout |

**Research questions**
- At what concurrency do responses start failing or slowing?
- Which timeout best distinguishes slow-but-valid pages from dead hosts?

**Sources:** runtime measurements, server logs where available, error output.
**Output:** a tuned, reproducible concurrency profile.

---

## Scenario 3 — user agent and blocking behaviour

| # | Command | How to use |
|---|---------|------------|
| 11 | `subjs -i urls.txt -ua 'Mozilla/5.0 (X11; Linux x86_64)'` | Browser-like UA |
| 12 | `subjs -i urls.txt -ua 'curl/8.5.0'` | Obvious tool UA |
| 13 | `diff <(subjs -i urls.txt) <(subjs -i urls.txt -ua 'Mozilla/5.0')` | Compare results |
| 14 | `subjs -i urls.txt -ua ''` | Empty UA behaviour |
| 15 | `httpx -l urls.txt -mc 200,403 -o status.txt` | Status context |

**Research questions**
- Which user agent produces the most complete script list?
- Do 403 or challenge pages still return script tags that mislead the count?

**Sources:** comparative output, `httpx` status, WAF documentation.
**Output:** a UA choice justified by measured coverage.

---

## Scenario 4 — pipeline integration

| # | Command | How to use |
|---|---------|------------|
| 16 | `cat hosts.txt \| gau \| subjs` | Historical URLs to JS |
| 17 | `cat hosts.txt \| gau \| subjs \| sort -u > js.txt` | Stable pipeline |
| 18 | `subjs -i urls.txt \| tee js-raw.txt \| wc -l` | Preserve raw output |
| 19 | `subjs -i urls.txt \| grep -E '\.js(\?\|$)'` | Filter obvious scripts |
| 20 | `subjs -i urls.txt \| xargs -n1 -P8 curl -sI` | Check headers at scale |

**Research questions**
- How much does `gau` history add over live-page extraction?
- Which pipeline stages silently drop URLs, and how is that detected?

**Sources:** gau docs, pipeline counts, curl headers.
**Output:** an auditable, multi-source JavaScript URL list.

---

## Scenario 5 — validation and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `httpx -l js.txt -mc 200 -o js-live.txt` | Validate live scripts |
| 22 | `sort -u js.txt \| comm -23 - js-live.txt` | Show dead entries |
| 23 | `sha256sum urls.txt js-live.txt` | Hash inputs and outputs |
| 24 | `head -20 js-live.txt` | Spot-check paths |
| 25 | `grep -c '\.map' js-live.txt` | Count source-map references |

**Research questions**
- Which discovered URLs are stale or blocked, and why?
- Do any JavaScript URLs reveal source maps or internal hostnames?

**Sources:** `httpx`, hashes, URL structure.
**Output:** a validated inventory ready for LinkFinder or sourcemapper.

## Flag coverage checklist

| Flag | Scenario |
|------|----------|
| `-c` | 2 |
| `-i` | 1, 2, 3, 4, 5 |
| `-t` | 2 |
| `-ua` | 3 |
| `-version` | 1 |
| stdin (no flag) | 1, 4 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — baseline | 5 |
| 2 — concurrency | 5 |
| 3 — user agent | 5 |
| 4 — pipelines | 5 |
| 5 — validation | 5 |
