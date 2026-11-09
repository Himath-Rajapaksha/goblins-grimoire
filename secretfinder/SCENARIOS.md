# SecretFinder — command scenarios for deep research

Installed build: packaged SecretFinder. Flags come from `help.txt`.

## How to invoke

```text
@deep-research For SecretFinder in 2026, which regex patterns produce the
highest false-positive rates, which provider key formats have changed, and how
should discovered secrets be verified and reported without using them? Cite
the SecretFinder repo, provider key-format docs, and secret-scanning guidance;
return a triage table and a safe verification checklist.
```

---

## Scenario 1 — single bundle

| # | Command | How to use |
|---|---------|------------|
| 1 | `secretfinder -i https://target/app.js -o app.html` | Scan one file |
| 2 | `secretfinder -i bundle.js` | Local file |
| 3 | `secretfinder -i https://target/app.js -r 'app'` | URL regex filter |
| 4 | `secretfinder -i https://target/app.js -p 127.0.0.1:8080` | Proxy |
| 5 | `grep -c 'Secret' app.html` | Count matches |

**Research questions**
- Which patterns fire most often on minified bundles?
- Does `-r` filter the input URL or matched content?

**Sources:** source, output, docs.
**Output:** a candidate list from one bundle.

---

## Scenario 2 — page-wide extraction

| # | Command | How to use |
|---|---------|------------|
| 6 | `secretfinder -e -i https://target/ -o page.html` | All scripts |
| 7 | `secretfinder -e -i https://target/ -n 'app;main'` | First-party only |
| 8 | `secretfinder -e -i https://target/ -g 'vendor;polyfill'` | Skip third-party |
| 9 | `secretfinder -e -i https://target/ -o page.html -c 'session=abc'` | Auth page |
| 10 | `grep -Eo 'AKIA[0-9A-Z]{16}' page.html \| sort -u` | AWS candidates |

**Research questions**
- How does `-e` resolve relative script URLs?
- Do `-n` and `-g` apply before or after fetching?

**Sources:** source, page HTML, output.
**Output:** page-wide secret candidates.

---

## Scenario 3 — authenticated and custom headers

| # | Command | How to use |
|---|---------|------------|
| 11 | `secretfinder -i https://target/app.js -c 'session=abc'` | Cookie |
| 12 | `secretfinder -i https://target/app.js -H $'Authorization: Bearer x'` | Bearer |
| 13 | `secretfinder -i https://target/app.js -H $'X-Env: prod\nX-Tenant: acme'` | Multiple headers |
| 14 | `secretfinder -e -i https://target/ -c 'session=abc' -H $'X-Env: prod'` | Combined |
| 15 | `curl -s -H 'Cookie: session=abc' https://target/app.js \| head` | Verify fetch |

**Research questions**
- How are newlines in `-H` parsed across shells?
- Which headers change the served bundle (env, tenant, locale)?

**Sources:** shell quoting, source, response diff.
**Output:** secrets from protected bundles.

---

## Scenario 4 — triage and verification

| # | Command | How to use |
|---|---------|------------|
| 16 | `grep -Eio 'aws\|google\|stripe\|slack\|github\|jwt\|private key' results.html \| sort \| uniq -c` | Classify |
| 17 | `grep -Eo 'AIza[0-9A-Za-z_-]{35}' results.html \| sort -u` | Google keys |
| 18 | `grep -Eo 'ghp_[A-Za-z0-9]{36}' results.html \| sort -u` | GitHub tokens |
| 19 | `grep -Eo 'sk_live_[A-Za-z0-9]+' results.html \| sort -u` | Stripe live keys |
| 20 | `grep -Eo 'eyJ[A-Za-z0-9_-]+\\.[A-Za-z0-9_-]+\\.[A-Za-z0-9_-]+' results.html` | JWTs |

**Research questions**
- Which candidates are public by design (Maps client keys, Stripe publishable)?
- What is the minimum safe verification for each provider?

**Sources:** provider docs, key-format references, triage notes.
**Output:** a prioritised, de-duplicated candidate list.

---

## Scenario 5 — reporting and hygiene

| # | Command | How to use |
|---|---------|------------|
| 21 | `secretfinder -e -i https://target/ -o /evidence/secrets.html` | Evidence |
| 22 | `sha256sum /evidence/secrets.html` | Hash |
| 23 | `sed -E 's/(AKIA[0-9A-Z]{4})[0-9A-Z]{12}/\1REDACTED/g' secrets.html > redacted.html` | Redact |
| 24 | `chmod 600 /evidence/secrets.html` | Restrict access |
| 25 | `shred -u /evidence/secrets.html` | Destroy per policy |

**Research questions**
- What redaction standard applies to credential evidence?
- Who must be notified when a live secret is found?

**Sources:** engagement policy, disclosure rules.
**Output:** a redacted, access-controlled evidence bundle.

## Flag coverage checklist

| Flag | Scenario |
|------|----------|
| `-i` | 1–5 |
| `-e` | 2, 5 |
| `-o` | 1, 5 |
| `-r` | 1 |
| `-c -H` | 3 |
| `-n -g` | 2 |
| `-p` | 1 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — single bundle | 5 |
| 2 — page-wide | 5 |
| 3 — auth | 5 |
| 4 — triage | 5 |
| 5 — reporting | 5 |
