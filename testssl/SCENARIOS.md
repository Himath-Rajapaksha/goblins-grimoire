# testssl — command scenarios for deep research

Installed build: testssl 3.2.1 (`/usr/bin/testssl`). Flags come from `help.txt`;
recipes and caveats are in [`README.md`](README.md).

## How to invoke

```text
@deep-research For testssl 3.2.1 on Debian 13, which checks still produce
authoritative results against TLS 1.3 and modern CDNs, how do its findings
compare with SSLyze 6.3.1 and raw OpenSSL, and what output/exit-code strategy
supports CI gating? Cite testssl docs, SSLyze docs, and cross-tool comparisons;
return a disagreement-triage table and a JSON query set.
```

---

## Scenario 1 — baseline audit

| # | Command | How to use |
|---|---------|------------|
| 1 | `testssl example.com` | Default scan |
| 2 | `testssl --quiet --color 0 example.com` | Log-safe output |
| 3 | `testssl https://example.com:8443` | Non-default port |
| 4 | `testssl -p -s -f -h example.com` | Protocols, ciphers, FS, headers |
| 5 | `testssl -b` | Banner and version |

**Research questions**
- Which checks does the default run skip (`-E`, `-g`)?
- How does the OpenSSL build affect TLS 1.3 results?

**Sources:** help, banner, docs.
**Output:** a TLS baseline with ratings.

---

## Scenario 2 — protocols and ciphers

| # | Command | How to use |
|---|---------|------------|
| 6 | `testssl -p example.com` | Protocol support |
| 7 | `testssl -s example.com` | Cipher categories |
| 8 | `testssl -e example.com` | Each local cipher |
| 9 | `testssl -E example.com` | Ciphers per protocol |
| 10 | `testssl -P example.com` | Server preference |

**Research questions**
- Which cipher categories map to which strength ratings?
- How expensive is `-e` versus `-E`?

**Sources:** docs, output, run timing.
**Output:** a cipher/protocol matrix.

---

## Scenario 3 — vulnerabilities

| # | Command | How to use |
|---|---------|------------|
| 11 | `testssl -U example.com` | All vulnerability checks |
| 12 | `testssl -H example.com` | Heartbleed |
| 13 | `testssl --BB example.com` | ROBOT |
| 14 | `testssl -I -R example.com` | CCS and renegotiation |
| 15 | `testssl -C -B -O -W example.com` | CRIME, BREACH, POODLE, SWEET32 |

**Research questions**
- Which vulnerability checks require `--phone-out` or external services?
- Which are definitive versus heuristic?

**Sources:** CVE docs, help, cross-tool runs.
**Output:** a confirmed vulnerability list.

---

## Scenario 4 — STARTTLS, mass scans, and tuning

| # | Command | How to use |
|---|---------|------------|
| 16 | `testssl -t smtp mail.example.com:25` | SMTP STARTTLS |
| 17 | `testssl -t postgres db.example.com:5432` | Postgres STARTTLS |
| 18 | `testssl --file scan.gnmap --mode parallel --warnings batch` | Mass scan |
| 19 | `testssl --ids-friendly --sneaky --connect-timeout 10 example.com` | IDS-aware |
| 20 | `testssl --proxy 127.0.0.1:8080 example.com` | Proxy |

**Research questions**
- Which STARTTLS protocols are supported, and how is `auto` resolved?
- What does `--mode parallel` do to reliability and rate limits?

**Sources:** help, nmap output, run logs.
**Output:** a scalable, protocol-aware scan.

---

## Scenario 5 — output and CI gating

| # | Command | How to use |
|---|---------|------------|
| 21 | `testssl --quiet --jsonfile-pretty tls.json example.com` | Structured JSON |
| 22 | `testssl -oA tls-audit --severity MEDIUM example.com` | All formats |
| 23 | `testssl --csvfile tls.csv example.com` | CSV |
| 24 | `testssl --htmlfile tls.html example.com` | HTML |
| 25 | `jq '.[] \| select(.severity == "CRITICAL" or .severity == "HIGH")' tls.json` | Critical findings |

**Research questions**
- Which output format preserves severity and finding IDs?
- Why is exit status unreliable for CI gating, and what replaces it?

**Sources:** docs, JSON output, CI logs.
**Output:** a gated, hashed TLS report.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Target/mass | URI, `-t --mx --file --mode --warnings --connect-timeout` | 1, 4 |
| Single checks | `-e -E -s -f -p -g -S -P -x -c -h` | 1, 2 |
| Vulnerabilities | `-U -H -I -T --BB --SI -R -C -B -O -Z -W -A -L -WS -F -J -D -4` | 3 |
| Tuning | `--ssl-native --openssl --proxy --ip -n --sneaky --ids-friendly --phone-out` | 3, 4 |
| Output | `--json --jsonfile --csv --html -oA --severity` | 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — baseline | 5 |
| 2 — protocols | 5 |
| 3 — vulnerabilities | 5 |
| 4 — STARTTLS/mass | 5 |
| 5 — output/CI | 5 |
