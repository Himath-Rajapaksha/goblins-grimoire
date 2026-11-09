# x8 — command scenarios for deep research

Installed build: x8 4.3.2. Flags come from `help.txt`.

## How to invoke

```text
@deep-research For x8 4.3.2, how do --learn-requests, --strict, and --verify
interact on dynamic pages with CSRF tokens and timestamps, which parameter
wordlists give the best discovery rate, and how should custom parameters like
admin=true be tested without changing state? Cite x8 docs, SecLists parameter
lists, and field reports; return a false-positive control matrix and a safe
command profile.
```

---

## Scenario 1 — GET discovery

| # | Command | How to use |
|---|---------|------------|
| 1 | `x8 -u https://target/page -w params.txt` | Basic GET |
| 2 | `x8 -u https://target/page -w burp-parameter-names.txt` | Larger list |
| 3 | `x8 -u https://target/page -w params.txt --strict` | Strict comparison |
| 4 | `x8 -u https://target/page -w params.txt --verify` | Verify hits |
| 5 | `x8 -u https://target/page -w params.txt -O json -o out.json` | JSON output |

**Research questions**
- Which detection heuristic flags a parameter by default?
- How does `--strict` reduce false positives on dynamic pages?

**Sources:** docs, output, source.
**Output:** a verified hidden-parameter list.

---

## Scenario 2 — POST and JSON bodies

| # | Command | How to use |
|---|---------|------------|
| 6 | `x8 -u https://target/login -X POST -w params.txt` | urlencoded default |
| 7 | `x8 -u https://target/login -X POST -t urlencode -w params.txt` | Explicit type |
| 8 | `x8 -u https://target/api -X POST -b '{"%s":{{random}}}' -t json -w params.txt` | JSON body |
| 9 | `x8 -u https://target/api -X POST -b '{"user":{"%s":{{random}}}}' -t json -w params.txt` | Nested JSON |
| 10 | `x8 -r request.txt -w params.txt` | Raw request |

**Research questions**
- How is the JSON template expanded for arrays and nested objects?
- Does `-r` preserve the original method and headers?

**Sources:** docs, `--test`, source.
**Output:** body parameters the API accepts.

---

## Scenario 3 — headers and custom values

| # | Command | How to use |
|---|---------|------------|
| 11 | `x8 -u https://target/ --headers -w headers.txt` | Header discovery |
| 12 | `x8 -u https://target/ --headers --custom-parameters x-forwarded-for -w h.txt` | Custom header name |
| 13 | `x8 -u https://target/page --custom-parameters admin -w p.txt` | admin param |
| 14 | `x8 -u https://target/page --custom-parameters debug --custom-values true -w p.txt` | debug=true |
| 15 | `x8 -u https://target/page --disable-custom-parameters -w p.txt` | Disable built-ins |

**Research questions**
- Which built-in custom parameters are tested by default?
- How should state-changing parameters be scoped?

**Sources:** help, source, engagement RoE.
**Output:** header/parameter behaviour differences.

---

## Scenario 4 — scale, proxies, recursion

| # | Command | How to use |
|---|---------|------------|
| 16 | `x8 -u urls.txt -w params.txt -W 10` | Batch URLs |
| 17 | `x8 -u urls.txt -w params.txt -W 10 --one-worker-per-host` | Per-host serial |
| 18 | `x8 -u https://target/page -w params.txt -x http://127.0.0.1:8080` | Proxy |
| 19 | `x8 -u https://target/page -w params.txt --replay-proxy http://127.0.0.1:8080 --replay-once` | Replay findings |
| 20 | `x8 -u https://target/page -w params.txt --recursion-depth 2` | Recursive |

**Research questions**
- How do `-W` and `-c` interact for a single URL?
- Does recursion re-test found parameters with new ones?

**Sources:** docs, proxy logs, run timing.
**Output:** a scalable, auditable scan.

---

## Scenario 5 — evidence and validation

| # | Command | How to use |
|---|---------|------------|
| 21 | `x8 -u https://target/page -w params.txt --save-responses evidence/` | Save evidence |
| 22 | `x8 -u https://target/page -w params.txt -O url -o urls.txt` | Retest URLs |
| 23 | `httpx -l urls.txt -silent -mc 200` | Validate |
| 24 | `jq '.results \| length' out.json` | Count findings |
| 25 | `sha256sum out.json urls.txt` | Hash evidence |

**Research questions**
- What does `--save-responses` store, and how large can it get?
- Which findings need manual confirmation before reporting?

**Sources:** output, evidence policy, manual tests.
**Output:** a hashed, validated finding set.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Input | `-u -r -w -X -b -t -H --headers` | 1–3 |
| Detection | `--learn-requests -m --strict --reflected-only --verify --custom-* --recursion-depth` | 1–4 |
| Request | `-L --encode --mimic-browser -d --timeout -x -B --replay-proxy` | 4 |
| Output | `-o -O --append --remove-empty --save-responses` | 1, 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — GET | 5 |
| 2 — POST/JSON | 5 |
| 3 — headers | 5 |
| 4 — scale | 5 |
| 5 — evidence | 5 |
