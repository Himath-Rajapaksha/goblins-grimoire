# theHarvester — command scenarios for deep research

Installed build: theHarvester 4.11.1. Flags come from `help.txt`; source
behaviour is documented in the project wiki.

## How to invoke

```text
@deep-research For theHarvester 4.11.1 in 2026, which -b sources still return
useful results without paid API keys, which are key-gated or dead, and what
result quality and rate limits should be expected per source? Cite the wiki,
source code, and provider documentation; return a ranked source list and an
evidence-producing command.
```

---

## Scenario 1 — passive footprint

| # | Command | How to use |
|---|---------|------------|
| 1 | `theHarvester -d example.com -b crtsh,hackertarget,rapiddns -l 500` | Fast, keyless |
| 2 | `theHarvester -d example.com -b all -l 500` | Full source set |
| 3 | `theHarvester -d example.com -b all -q` | Suppress key warnings |
| 4 | `theHarvester -d example.com -b urlscan,otx,threatcrowd` | Threat-intel sources |
| 5 | `theHarvester -d example.com -b duckduckgo,brave,mojeek` | Search-engine sources |

**Research questions**
- Which sources return results without keys, and which silently skip?
- What per-source result limits apply?

**Sources:** wiki, provider docs, run logs.
**Output:** a baseline host/email/URL list.

---

## Scenario 2 — resolution and screenshots

| # | Command | How to use |
|---|---------|------------|
| 6 | `theHarvester -d example.com -b crtsh -r` | Resolve subdomains |
| 7 | `theHarvester -d example.com -b crtsh -n` | DNS server lookup |
| 8 | `theHarvester -d example.com -b crtsh -r -n` | Both |
| 9 | `theHarvester -d example.com -b crtsh -r --screenshot /evidence/shots` | Screenshot hosts |
| 10 | `theHarvester -d example.com -b crtsh -e 1.1.1.1 -r` | Custom resolver |

**Research questions**
- Which resolver list does `-r` use by default?
- How does screenshot capture handle TLS errors and timeouts?

**Sources:** source code, resolver docs, screenshot output.
**Output:** live hosts with visual evidence.

---

## Scenario 3 — brute force and API discovery

| # | Command | How to use |
|---|---------|------------|
| 11 | `theHarvester -d example.com -b crtsh -c -w subs.txt` | DNS brute force |
| 12 | `theHarvester -d example.com -b crtsh -a` | API endpoint scan |
| 13 | `theHarvester -d example.com -b crtsh -a -w api.txt` | Custom API wordlist |
| 14 | `theHarvester -d example.com -b all -t` | Takeover check |
| 15 | `theHarvester -d example.com -b all -s` | Shodan enrichment |

**Research questions**
- How loud is `-c` on authoritative DNS, and can it be rate-limited?
- Which takeover fingerprints are current in 4.11.1?

**Sources:** source code, DNS logs, takeover datasets.
**Output:** additional subdomains, APIs, or takeover candidates.

---

## Scenario 4 — identity and breach sources

| # | Command | How to use |
|---|---------|------------|
| 16 | `theHarvester -d example.com -b dehashed,haveibeenpwned -q` | Breach exposure |
| 17 | `theHarvester -d example.com -b hunter,rocketreach,tomba -q` | Contact enrichment |
| 18 | `theHarvester -d example.com -b github-code,gitlab -q` | Code leaks |
| 19 | `theHarvester -d example.com -b linkedin` (where supported) | Employee names |
| 20 | `jq -r '.emails[]' report.json \| sort -u` | Extract emails |

**Research questions**
- Which identity sources comply with GDPR-style minimisation?
- How fresh are breach records, and how are false matches filtered?

**Sources:** provider docs, privacy law, output inspection.
**Output:** a scoped contact/exposure dataset.

---

## Scenario 5 — evidence and reproducibility

| # | Command | How to use |
|---|---------|------------|
| 21 | `theHarvester -d example.com -b all -f /evidence/acme` | XML + JSON output |
| 22 | `sha256sum /evidence/acme.xml /evidence/acme.json` | Hash evidence |
| 23 | `jq '.hosts \| length' /evidence/acme.json` | Count hosts |
| 24 | `theHarvester -d example.com -b all -p` | Route through proxies |
| 25 | `theHarvester -d example.com -b all -S 100 -l 100` | Paginate results |

**Research questions**
- Does `-S` paginate per source or globally?
- How should proxy rotation be configured in `proxies.yaml`?

**Sources:** source code, proxies.yaml, run logs.
**Output:** a hashed, reproducible evidence set.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Target | `-d -l -S -b` | 1, 5 |
| Enrichment | `-e -r -n -c -w -a -s -t --screenshot` | 2, 3 |
| Output | `-f -p -q` | 1, 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — passive footprint | 5 |
| 2 — resolution | 5 |
| 3 — brute/API | 5 |
| 4 — identity/breach | 5 |
| 5 — evidence | 5 |
