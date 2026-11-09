# metagoofil — command scenarios for deep research

Installed build: metagoofil 1.5.0 (opsdisk Python 3 fork). Flags come from
`help.txt`.

## How to invoke

```text
@deep-research For metagoofil 1.5.0 in 2026, which search backends still return
results without CAPTCHAs, which document types yield the most reliable author
and path metadata, and how should downloads be isolated and parsed safely?
Cite the opsdisk fork, exiftool docs, and field reports; return a polite
harvesting workflow with a metadata extraction table.
```

---

## Scenario 1 — link discovery only

| # | Command | How to use |
|---|---------|------------|
| 1 | `metagoofil -d example.com -t pdf,doc,xls,ppt` | List links |
| 2 | `metagoofil -d example.com -t pdf -l 50` | Cap results |
| 3 | `metagoofil -d example.com -t pdf,docx -f links.txt` | Save links |
| 4 | `metagoofil -d example.com -t pdf -u "Mozilla/5.0"` | Custom UA |
| 5 | `wc -l links.txt` | Count discovered links |

**Research questions**
- Which search backend does the fork use, and when does it get blocked?
- Does `-l` limit per type or overall?

**Sources:** source code, run output, Google behaviour.
**Output:** a document link inventory.

---

## Scenario 2 — download and extract

| # | Command | How to use |
|---|---------|------------|
| 6 | `metagoofil -d example.com -t pdf,docx,xlsx,pptx -w` | Download and parse |
| 7 | `metagoofil -d example.com -t pdf -w -o evidence` | Custom directory |
| 8 | `metagoofil -d example.com -t pdf -w -n 10` | Cap downloads |
| 9 | `metagoofil -d example.com -t pdf -w -r 4` | Fewer threads |
| 10 | `ls -la evidence/` | Inspect downloads |

**Research questions**
- Which metadata fields are extracted per file type, and which are missing?
- What does the tool do with corrupt or password-protected files?

**Sources:** parser source, output, file inspection.
**Output:** downloaded documents and a metadata report.

---

## Scenario 3 — rate limiting and stealth

| # | Command | How to use |
|---|---------|------------|
| 11 | `metagoofil -d example.com -t pdf -w -e 60` | Slow search pace |
| 12 | `metagoofil -d example.com -t pdf -w -i 30` | Longer URL timeout |
| 13 | `metagoofil -d example.com -t pdf -w -r 1` | Single-thread downloads |
| 14 | `metagoofil -d example.com -t pdf -w -l 20 -n 5` | Minimal footprint |
| 15 | `date; metagoofil ...; date` | Measure runtime |

**Research questions**
- How visible is the run to the target's web logs?
- Does the random UA actually rotate per request?

**Sources:** target logs (authorized lab), source code.
**Output:** a low-and-slow harvesting profile.

---

## Scenario 4 — metadata analysis

| # | Command | How to use |
|---|---------|------------|
| 16 | `exiftool -r evidence/ > metadata.txt` | Full metadata dump |
| 17 | `exiftool -r -Author -Creator -Producer -CompanyName evidence/` | Key fields |
| 18 | `grep -Eio '[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}' metadata.txt \| sort -u` | Emails |
| 19 | `grep -Ei 'author\|creator' metadata.txt \| sort -u` | Usernames |
| 20 | `grep -Ei 'producer\|application' metadata.txt \| sort -u` | Software versions |

**Research questions**
- Which fields survive PDF sanitisation versus Office "inspect document"?
- How reliable are author names as username seeds?

**Sources:** exiftool docs, metadata standards, output.
**Output:** username/email/software intelligence.

---

## Scenario 5 — isolation and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `mkdir -p /evidence/metagoofil && chmod 700 /evidence/metagoofil` | Secure storage |
| 22 | `sha256sum evidence/* > hashes.txt` | Hash downloads |
| 23 | `tar czf metagoofil-evidence.tgz evidence hashes.txt` | Bundle evidence |
| 24 | `clamscan -r evidence/` | Scan before handling |
| 25 | `rm -rf evidence/` | Destroy per policy after reporting |

**Research questions**
- What malware-handling policy applies to downloaded documents?
- Which artefacts must be retained for reproducibility?

**Sources:** engagement policy, AV output, evidence rules.
**Output:** a scanned, hashed, policy-compliant evidence bundle.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Target/search | `-d -t -l -f -u` | 1 |
| Download | `-w -o -n -r -i` | 2, 3 |
| Pace | `-e` | 3 |
| Analysis | exiftool, grep | 4 |
| Evidence | hashing, scanning, retention | 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — discovery | 5 |
| 2 — download | 5 |
| 3 — rate limiting | 5 |
| 4 — analysis | 5 |
| 5 — evidence | 5 |
