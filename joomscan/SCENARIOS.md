# JoomScan — command scenarios for deep research

Installed build: OWASP JoomScan 0.0.7 (2018). Flags come from `help.txt`.

## How to invoke

```text
@deep-research For JoomScan 0.0.7 (2018 signatures) against Joomla 4/5 in 2026,
which checks still produce accurate version and component results, which are
obsolete, and what manual verification replaces them? Cite the JoomScan repo,
Joomla release notes, and CVE data; return a gap list and a modern verification
workflow using nuclei and manual manifest checks.
```

---

## Scenario 1 — version and fingerprint

| # | Command | How to use |
|---|---------|------------|
| 1 | `joomscan -u https://target/ -jv` | Version only |
| 2 | `joomscan -u https://target/ -nr` | Full scan, no report file |
| 3 | `curl -s https://target/administrator/manifests/files/joomla.xml` | Manual manifest |
| 4 | `curl -sI https://target/ \| head` | Headers and cookies |
| 5 | `joomscan --version` | Record scanner version |

**Research questions**
- How does JoomScan derive the version, and how is that spoofed?
- Which Joomla 4/5 paths expose the real version?

**Sources:** source, Joomla docs, manifest output.
**Output:** a verified version string.

---

## Scenario 2 — component enumeration

| # | Command | How to use |
|---|---------|------------|
| 6 | `joomscan -u https://target/ -ec -nr` | Enumerate components |
| 7 | `joomscan -u https://target/ -ec -nr > comps.log` | Save output |
| 8 | `grep -Ei 'com_' comps.log` | Extract components |
| 9 | `searchsploit joomla com_<name>` | CVE lookup |
| 10 | `nuclei -u https://target/ -tags joomla` | Template validation |

**Research questions**
- Which generated paths reveal installed components?
- How reliable is component-to-CVE mapping for current extensions?

**Sources:** output, CVE databases, nuclei templates.
**Output:** a component inventory with vulnerability candidates.

---

## Scenario 3 — authenticated and proxied scans

| # | Command | How to use |
|---|---------|------------|
| 11 | `joomscan -u https://target/ --cookie "session=abc" -nr` | Cookie auth |
| 12 | `joomscan -u https://target/ --proxy http://127.0.0.1:8080 -nr` | HTTP proxy |
| 13 | `joomscan -u https://target/ --proxy socks://127.0.0.1:9050 -nr` | SOCKS proxy |
| 14 | `joomscan -u https://target/ -a "Mozilla/5.0" -nr` | Custom UA |
| 15 | `joomscan -u https://target/ -r --timeout 20 -nr` | Random UA, timeout |

**Research questions**
- Which checks require an authenticated session?
- Does the proxy handle TLS interception certificates?

**Sources:** docs, lab proxy, output.
**Output:** coverage of protected paths.

---

## Scenario 4 — batch scanning and evasion

| # | Command | How to use |
|---|---------|------------|
| 16 | `joomscan -m targets.txt -nr` | Bulk scan |
| 17 | `joomscan -m targets.txt -nr > batch.log` | Log batch |
| 18 | `joomscan -u https://target/ -r -nr` | Random UA |
| 19 | `joomscan -u https://target/ --timeout 10 -nr` | Fast fail |
| 20 | `grep -Ei 'vulnerab\|found' batch.log` | Extract findings |

**Research questions**
- How does `--mass` handle timeouts and errors?
- What pacing avoids WAF blocks?

**Sources:** source, logs, target behaviour.
**Output:** a prioritised list of Joomla findings.

---

## Scenario 5 — verification and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `joomscan -u https://target/ -nr > scan.log` | Capture evidence |
| 22 | `grep -Ei 'version\|component\|vulnerab' scan.log` | Extract |
| 23 | `searchsploit joomla <version>` | Exploit lookup |
| 24 | `sha256sum scan.log` | Hash |
| 25 | `rm -f scan.log` (after reporting) | Retention cleanup |

**Research questions**
- Which findings need manual proof before reporting?
- What retention applies to scanner logs?

**Sources:** engagement policy, CVE data, output.
**Output:** verified findings and hashed evidence.

## Flag coverage checklist

| Flag | Scenario |
|------|----------|
| `-u -m` | 1–5 |
| `-jv` | 1 |
| `-ec` | 2 |
| `-nr` | 1–5 |
| `--cookie` | 3 |
| `--proxy` | 3 |
| `-a -r --timeout` | 3, 4 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — version | 5 |
| 2 — components | 5 |
| 3 — auth/proxy | 5 |
| 4 — batch | 5 |
| 5 — verification | 5 |
