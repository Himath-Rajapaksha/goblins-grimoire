# wfuzz — command scenarios for deep research

Installed build: wfuzz 3.1.0. Flags come from `help.txt` and the advanced help.

## How to invoke

```text
@deep-research For wfuzz 3.1.0 on Debian 13 with a pycurl build that may lack
OpenSSL, which invocation patterns give reliable HTTPS fuzzing, how should
wildcard responses be filtered with --hh/--hc/--hs, and which payload/encoder
combinations are safe at scale? Cite wfuzz docs, pycurl packaging, and field
reports; return a tested HTTPS recipe and a filter decision table.
```

---

## Scenario 1 — content discovery

| # | Command | How to use |
|---|---------|------------|
| 1 | `wfuzz -w common.txt https://target/FUZZ` | Basic directory scan |
| 2 | `wfuzz -w common.txt --hc 404 https://target/FUZZ` | Hide 404 |
| 3 | `wfuzz -w files.txt --hc 404,403 https://target/FUZZ` | Hide 404/403 |
| 4 | `wfuzz -w list.txt --hh 0,1234 https://target/FUZZ` | Filter by size |
| 5 | `wfuzz -w list.txt -R 2 --hc 404 https://target/FUZZ` | Recursive |

**Research questions**
- How do you establish a wildcard baseline before choosing `--hh` values?
- Does `-R` follow discovered directories only?

**Sources:** wfuzz docs, response samples, run logs.
**Output:** a clean list of existing paths.

---

## Scenario 2 — parameters and methods

| # | Command | How to use |
|---|---------|------------|
| 6 | `wfuzz -w payloads.txt -d "q=FUZZ" https://target/search` | POST body |
| 7 | `wfuzz -w payloads.txt "https://target/?id=FUZZ"` | GET parameter |
| 8 | `wfuzz -w payloads.txt -H "X-Forwarded-For: FUZZ" https://target/` | Header fuzz |
| 9 | `wfuzz -w payloads.txt -X FUZZ https://target/` | Method fuzz |
| 10 | `wfuzz -V alltype https://target/page` | All params |

**Research questions**
- How does `-V alltype` enumerate parameter names, and how loud is it?
- Does `-d` default to POST, and how is JSON body fuzzing expressed?

**Sources:** docs, `--dry-run`, request logs.
**Output:** a set of parameters and methods that behave differently.

---

## Scenario 3 — authentication and multi-payload

| # | Command | How to use |
|---|---------|------------|
| 11 | `wfuzz -w users.txt -w pass.txt -d "u=FUZZ&p=FUZ2Z" --hc 200 https://target/login` | Credential brute force |
| 12 | `wfuzz -w users.txt -w pass.txt -m zip --hc 200 ...` | Zip payloads |
| 13 | `wfuzz -w list.txt --basic user:pass https://target/FUZZ` | Basic auth |
| 14 | `wfuzz -w list.txt --ntlm DOMAIN\\user:pass https://target/FUZZ` | NTLM auth |
| 15 | `wfuzz -w list.txt -b "session=FUZZ" https://target/FUZZ` | Cookie fuzz |

**Research questions**
- What does the `product` iterator do to request count versus `zip`?
- How are auth options combined with payload placeholders?

**Sources:** docs, dry-run, lab app.
**Output:** valid credentials or a tested auth surface.

---

## Scenario 4 — virtual hosts and encoders

| # | Command | How to use |
|---|---------|------------|
| 16 | `wfuzz -w hosts.txt -H "Host: FUZZ.example.com" --hc 404 https://example.com/` | Vhost discovery |
| 17 | `wfuzz -w list.txt -z file,list.txt,base64 https://target/FUZZ` | Base64 encoder |
| 18 | `wfuzz -w list.txt -z file,list.txt,url https://target/FUZZ` | URL encoder |
| 19 | `wfuzz -w list.txt -z file,list.txt,md5@sha1 https://target/FUZZ` | Chained encoders |
| 20 | `wfuzz -w list.txt --dry-run https://target/FUZZ` | Verify wire format |

**Research questions**
- How do encoded payloads interact with `--hh` baselines?
- Which vhost status codes indicate a real virtual host behind a default page?

**Sources:** docs, encoder list, dry-run output.
**Output:** discovered vhosts or correctly encoded payloads.

---

## Scenario 5 — output, recipes, and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `wfuzz -w list.txt -o json --hc 404 https://target/FUZZ > results.json` | JSON output |
| 22 | `jq '.results \| length' results.json` | Count findings |
| 23 | `wfuzz -w list.txt --oF run.json https://target/FUZZ` | Save for reuse |
| 24 | `wfuzz --dump-recipe run.recipe ...` | Persist options |
| 25 | `sha256sum results.json run.json` | Hash evidence |

**Research questions**
- What is the JSON schema for a result row?
- Does `--oF` preserve request/response bodies or only payloads?

**Sources:** output inspection, docs.
**Output:** a reproducible recipe and hashed JSON evidence.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Payloads | `-z -w -m -V -X -e --slice` | 1–4 |
| Request | `-u -d -H -b --basic --ntlm -p -t -s -R -L` | 1–4 |
| Filters | `--hc --hl --hw --hh --sc --ss --hs --filter` | 1, 5 |
| Output | `-o -f --oF --recipe --dry-run -c -v` | 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — discovery | 5 |
| 2 — parameters | 5 |
| 3 — auth | 5 |
| 4 — vhost/encoders | 5 |
| 5 — evidence | 5 |
