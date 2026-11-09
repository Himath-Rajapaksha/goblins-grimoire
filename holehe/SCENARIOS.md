# holehe — command scenarios for deep research

Installed build: holehe v1.61. Flags come from `help.txt`.

## How to invoke

```text
@deep-research For holehe v1.61 in 2026, which of its site checks still return
reliable positive/negative results, which are broken by bot protection or
markup changes, and what timeout/retry/no-password-recovery settings minimise
false results and side effects? Cite the holehe repo, issue tracker, and
observed site behaviour; return a ranked site reliability table and a safe
batch command.
```

---

## Scenario 1 — single-address footprint

| # | Command | How to use |
|---|---------|------------|
| 1 | `holehe target@example.com` | Full check |
| 2 | `holehe --only-used target@example.com` | Positives only |
| 3 | `holehe --no-color --no-clear target@example.com` | Log-friendly |
| 4 | `holehe -T 20 target@example.com` | Longer timeout |
| 5 | `holehe -v target@example.com` | Verbose/debug where available |

**Research questions**
- Which sites need JavaScript or cookies and always fail in CLI mode?
- How does the tool distinguish "exists" from "rate-limited"?

**Sources:** holehe source, site responses, issue tracker.
**Output:** a reliable positive set for one address.

---

## Scenario 2 — batch and export

| # | Command | How to use |
|---|---------|------------|
| 6 | `holehe -C target@example.com` | CSV export |
| 7 | `holehe -C -NP a@example.com b@example.com` | Batch, no recovery |
| 8 | `holehe --only-used --no-color a@example.com b@example.com` | Multiple positives |
| 9 | `ls -la holehe_*.csv` | Locate CSV output |
| 10 | `cut -d, -f1 holehe_*.csv \| sort -u` | Extract sites |

**Research questions**
- Is the CSV one row per site per address, and what are the columns?
- Does batching reuse connections or trigger more rate limits?

**Sources:** CSV output, source code.
**Output:** a normalised service list per address.

---

## Scenario 3 — reducing side effects

| # | Command | How to use |
|---|---------|------------|
| 11 | `holehe -NP target@example.com` | No password recovery |
| 12 | `holehe -NP -T 5 target@example.com` | Fast fail |
| 13 | `holehe -NP --only-used --no-clear target@example.com` | Minimal interaction |
| 14 | `holehe -NP --no-color target@example.com 2>&1 \| grep -E '^\[\+\]'` | Parse hits |
| 15 | `whois $(dig +short example.com) ` | Identify the target's mail provider before recovery tests |

**Research questions**
- Which sites send a recovery email even when the address exists, and does `-NP` fully suppress it?
- Does `-T 5` cause more false negatives on slow sites?

**Sources:** holehe source, site policies, mail logs in a lab.
**Output:** a low-impact check that still answers the question.

---

## Scenario 4 — correlation and verification

| # | Command | How to use |
|---|---------|------------|
| 16 | `holehe --only-used --no-color target@example.com > used.txt` | Capture hits |
| 17 | `sherlock --print-found target` | Cross-check the username |
| 18 | `ghunt email target@example.com` | Google-side details |
| 19 | `theHarvester -d example.com -b all -q` | Domain-wide addresses |
| 20 | `comm -12 <(sort used.txt) <(sort known-services.txt)` | Intersect lists |

**Research questions**
- Which services appear in both holehe and sherlock results, and which are false positives?
- How should conflicting evidence be weighted?

**Sources:** cross-tool output, manual verification.
**Output:** a verified service footprint.

---

## Scenario 5 — evidence and retention

| # | Command | How to use |
|---|---------|------------|
| 21 | `holehe -C -NP target@example.com` | Structured evidence |
| 22 | `sha256sum holehe_*.csv` | Hash evidence |
| 23 | `wc -l holehe_*.csv` | Count sites checked |
| 24 | `grep -c ',' holehe_*.csv` | Sanity-check rows |
| 25 | `shred -u holehe_*.csv` | Destroy per retention policy |

**Research questions**
- What is the retention period for account-existence data?
- Does the CSV include error rows that should be excluded from evidence?

**Sources:** engagement policy, CSV inspection.
**Output:** a hashed, minimal evidence set.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Output | `--only-used --no-color --no-clear -C` | 1, 2, 5 |
| Behaviour | `-NP -T` | 3 |
| Targets | `EMAIL [EMAIL ...]` | 1–3 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — single address | 5 |
| 2 — batch/export | 5 |
| 3 — side effects | 5 |
| 4 — correlation | 5 |
| 5 — evidence | 5 |
