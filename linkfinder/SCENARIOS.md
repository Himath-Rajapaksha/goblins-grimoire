# LinkFinder — command scenarios for deep research

Installed build: packaged LinkFinder. Flags come from `help.txt`.

## How to invoke

```text
@deep-research For LinkFinder in 2026, which JavaScript patterns hide endpoints
from static extraction (string concatenation, webpack chunks, source maps), how
does -d discovery interact with SPAs and authenticated bundles, and what regex
set gives the best precision for API discovery? Cite LinkFinder issues, JS
bundling docs, and bug-bounty field reports; return a high-precision extraction
workflow.
```

---

## Scenario 1 — single-file extraction

| # | Command | How to use |
|---|---------|------------|
| 1 | `linkfinder -i https://target/app.js` | Parse one bundle |
| 2 | `linkfinder -i https://target/app.js -o out.html` | Save report |
| 3 | `linkfinder -i bundle.js` | Local file |
| 4 | `linkfinder -i bundle.js -r '^/api/'` | Filter API paths |
| 5 | `linkfinder -i bundle.js -t 30` | Raise timeout |

**Research questions**
- Which URL patterns does the default regex capture?
- How are relative paths normalised in the HTML report?

**Sources:** source, output, docs.
**Output:** an endpoint list per bundle.

---

## Scenario 2 — recursive page parsing

| # | Command | How to use |
|---|---------|------------|
| 6 | `linkfinder -d -i https://target/` | All scripts on a page |
| 7 | `linkfinder -d -i https://target/ -r '^/api/' -o api.html` | Filtered |
| 8 | `linkfinder -d -i https://target/ -c 'session=abc'` | Authenticated |
| 9 | `linkfinder -d -i https://target/ -b -o burp.html` | Burp output |
| 10 | `grep -c 'http' api.html` | Count findings |

**Research questions**
- Does `-d` follow scripts loaded dynamically or only `<script src>`?
- How are cross-origin scripts handled?

**Sources:** docs, page source, output.
**Output:** a full-page script endpoint map.

---

## Scenario 3 — folders, source maps, and chunks

| # | Command | How to use |
|---|---------|------------|
| 11 | `linkfinder -i '/evidence/js/*.js' -o local.html` | Folder of files |
| 12 | `wget -r -np -A '*.js' https://target/static/` | Download bundles |
| 13 | `linkfinder -i '/evidence/static/**/*.js'` | Recursive wildcard |
| 14 | `curl -s https://target/app.js.map -o app.js.map` | Fetch a source map |
| 15 | `jq '.sources' app.js.map` | Inspect original sources |

**Research questions**
- Which wildcard syntax works for nested folders?
- Are source maps commonly exposed, and how should they be stored?

**Sources:** shell globbing, source-map spec, target responses.
**Output:** reconstructed source and deeper endpoint coverage.

---

## Scenario 4 — filtering and normalisation

| # | Command | How to use |
|---|---------|------------|
| 16 | `linkfinder -i app.js -r '^/api/'` | API prefix |
| 17 | `linkfinder -i app.js -r 'v[0-9]+'` | Versioned paths |
| 18 | `grep -Eo 'https?://[^"<> ]+' output.html \| sort -u` | Absolute URLs |
| 19 | `grep -Eo '/[a-zA-Z0-9_./-]+' output.html \| sort -u` | Relative paths |
| 20 | `awk -F/ '{print $2}' endpoints.txt \| sort \| uniq -c` | Group by root |

**Research questions**
- Which regex balances recall versus noise for minified code?
- How should duplicate endpoints across bundles be merged?

**Sources:** output, regex tests.
**Output:** a deduplicated, prioritised endpoint list.

---

## Scenario 5 — handoff and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `linkfinder -d -i https://target/ -o report.html` | Evidence report |
| 22 | `grep -Eo '/[a-zA-Z0-9_./-]+' report.html \| sort -u > endpoints.txt` | Extract list |
| 23 | `httpx -l endpoints.txt -silent -mc 200,301,302,401,403` | Validate live paths |
| 24 | `wc -l endpoints.txt` | Count |
| 25 | `sha256sum report.html endpoints.txt` | Hash evidence |

**Research questions**
- Which endpoints are worth manual testing first?
- What provenance should accompany extracted endpoints?

**Sources:** validation output, engagement policy.
**Output:** validated endpoints and a hashed evidence bundle.

## Flag coverage checklist

| Flag | Scenario |
|------|----------|
| `-i` | 1–5 |
| `-d` | 2, 5 |
| `-o` | 1, 5 |
| `-r` | 1, 4 |
| `-b` | 2 |
| `-c` | 2 |
| `-t` | 1 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — single file | 5 |
| 2 — recursive | 5 |
| 3 — folders/maps | 5 |
| 4 — filtering | 5 |
| 5 — handoff | 5 |
