# whois — command scenarios for deep research

Installed build: whois 5.6.3 (Debian package `whois`). Every flag below appears
in [`help.txt`](help.txt) or `man whois`. There is **no** `/etc/whois.conf` on
this install, so server selection uses the built-in table.

> **Query disclosure:** WHOIS is plaintext TCP/43. Every query is visible to the
> network path and to each registry in the referral chain. Do not put sensitive
> identifiers into a public registry query, and use only in-scope domains.

## How to invoke

```text
@deep-research For an authorized external reconnaissance engagement, compare
whois 5.6.3 with RDAP and `dig` for domain and IP registration data. Which
fields are authoritative at the registry versus the registrar, how do thin and
thick registries differ, what does --no-recursion change, and what data has
been removed by redaction policy? Cite RFC 3912, the ICANN RDAP documentation,
the RIPE Database Query Reference Manual, and the whois(1) manual; return a
field-provenance table with confidence notes.
```

Create an evidence workspace, replacing example objects with in-scope ones:

```bash
mkdir -p ~/whois-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/whois-demo/evidence/run-start.txt
```

## Scenario 1 — identify the build and its configuration

Server selection is the client's main job, and it depends on compile-time
tables plus an optional config file. Confirm both before interpreting output.

| # | Command | How to use |
|---|---------|------------|
| 1 | `whois --version` | Version and bug address |
| 2 | `whois --help` | Full option list |
| 3 | `dpkg-query -W -f='${Version}\n' whois` | Installed package version |
| 4 | `ls -l /etc/whois.conf` | Is a local server table present? |
| 5 | `man 5 whois.conf \| col -b \| sed -n '1,40p'` | Config format if you add one |
| 6 | `env \| grep -E '^WHOIS_'` | `WHOIS_OPTIONS` / `WHOIS_SERVER` overrides |
| 7 | `whois --verbose example.com 2>&1 \| head -20` | Show the client's decisions |

**Research questions**
- Which server does the client choose for a `.com` domain, and why?
- What changes when `WHOIS_SERVER` is set?
- What is the precedence between `WHOIS_OPTIONS` and command-line flags?
- Why does the manual warn that this parser may have buffer overflows?
- What does `--verbose` reveal that the default output hides?

**Sources:** `whois --version`; `man whois` (`FILES`, `ENVIRONMENT`, `BUGS`);
`man 5 whois.conf`.
**Output:** a resolved server table and a note on local overrides.

## Scenario 2 — domain registration and the referral chain

A domain query can touch two servers. The registry record says who the registrar
is; the registrar record says who the registrant is (subject to redaction).

| # | Command | How to use |
|---|---------|------------|
| 8 | `whois example.com` | Registry plus registrar |
| 9 | `whois --no-recursion example.com` | Registry record only |
| 10 | `whois -I example.com` | Ask IANA first, then follow |
| 11 | `whois -H example.com` | Hide legal disclaimers |
| 12 | `whois --verbose example.com 2>&1 \| tee ~/whois-demo/evidence/domain-verbose.txt` | Record the referral path |
| 13 | `whois -h whois.verisign-grs.com example.com` | Query the registry directly |
| 14 | `timeout 20 whois example.com > ~/whois-demo/evidence/domain.txt` | Bound a slow server |

**Research questions**
- Which server produced each section of the combined output?
- What does `--no-recursion` remove, and what remains?
- Does `-I` produce the same answer as the default path?
- Which fields are registry-authoritative and which are registrar-authoritative?
- What happens when the registrar's server is down or rate-limits you?

**Sources:** `man whois` (`-I`, `--no-recursion`, `NOTES`); RFC 3912; ICANN
gTLD registry agreements on thick/thin registration data.
**Output:** a two-server provenance map for one domain.

## Scenario 3 — IP addresses, netblocks, and AS numbers

IP queries go to the regional internet registry (RIR) responsible for the
address. ARIN strips the CIDR suffix from the query; other RIRs may not.

| # | Command | How to use |
|---|---------|------------|
| 15 | `whois 203.0.113.10` | Single address |
| 16 | `whois 203.0.113.0/24` | Netblock |
| 17 | `whois -b 203.0.113.0/24` | Brief range plus abuse contact |
| 18 | `whois AS64500` | AS number |
| 19 | `whois -h whois.arin.net 203.0.113.10` | Force the ARIN server |
| 20 | `whois -h whois.apnic.net 203.0.113.10` | Compare another RIR |
| 21 | `whois -i origin AS64500` | RIPE inverse lookup by origin |

**Research questions**
- Which RIR is authoritative for the address, and how did the client know?
- Does the netblock answer match the single-address answer?
- What does `-b` add or omit compared with the full record?
- Why is `-i origin` RIPE-specific, and what happens elsewhere?
- How do you tell a direct allocation from a sub-allocation?

**Sources:** `man whois` (`NOTES` on ARIN); RIR allocation policy documents;
RFC 3912.
**Output:** an address-to-RIR map with allocation type and abuse contact.

## Scenario 4 — RIPE-specific object queries

RIPE's database supports a richer query language than the base protocol. These
flags are ignored by non-RIPE servers, so verify the server first.

| # | Command | How to use |
|---|---------|------------|
| 22 | `whois -h whois.ripe.net -- '-i admin-c ABCD-RIPE'` | Inverse lookup |
| 23 | `whois -h whois.ripe.net -l 203.0.113.0/24` | One level less specific |
| 24 | `whois -h whois.ripe.net -L 203.0.113.0/24` | All less specific |
| 25 | `whois -h whois.ripe.net -m 203.0.113.0/24` | One level more specific |
| 26 | `whois -h whois.ripe.net -M 203.0.113.0/24` | All more specific |
| 27 | `whois -h whois.ripe.net -x 203.0.113.0/24` | Exact match only |
| 28 | `whois -h whois.ripe.net -t person` | Object template |
| 29 | `whois -h whois.ripe.net -v person` | Verbose template |
| 30 | `whois -h whois.ripe.net -q sources` | List mirrored sources |

**Research questions**
- What is the difference between `-l`/`-L` and `-m`/`-M`?
- Why must server-specific options follow `--`?
- Which attributes are valid for `-i`, and where is that documented?
- Does `-x` change the query sent or only the client's filtering?
- What does `-q sources` tell you that a normal query does not?

**Sources:** `man whois` (`OPTIONS`); RIPE Database Query Reference Manual;
RIPE database object templates.
**Output:** a RIPE query cheat sheet with verified server behavior.

## Scenario 5 — redaction, privacy, and data quality

Modern registration data is incomplete by policy. Distinguish "not present"
from "removed", and never present a redacted record as a full one.

| # | Command | How to use |
|---|---------|------------|
| 31 | `whois example.com \| grep -iE 'registrant\|admin\|tech\|email'` | See what contact data remains |
| 32 | `whois -B example.com \| grep -iE 'email\|registrant'` | Disable client-side filtering |
| 33 | `whois -H example.com \| sed -n '1,40p'` | Read the record without disclaimers |
| 34 | `whois example.com \| grep -iE 'status\|updated\|created\|expires'` | Lifecycle fields |
| 35 | `whois example.com \| grep -i 'redacted'` | Explicit redaction markers |
| 36 | `whois example.com \| wc -l` | Compare record size across registries |

**Research questions**
- Which fields are missing because of redaction rather than non-response?
- Does `-B` ever restore a redacted e-mail address?
- Which status codes indicate client-hold, server-hold, or transfer lock?
- How do created/updated/expiry fields differ between registry and registrar?
- What can you legitimately infer from a redacted record, and what can you not?

**Sources:** `man whois` (`-B`, `-H`); ICANN Temporary Specification and
Registration Data Policy; GDPR Art. 6; registry redaction notices.
**Output:** a data-quality note that separates absent, redacted, and
not-authoritative fields.

## Scenario 6 — automation, evidence, and rate limits

WHOIS output is unstructured, so automation must record the exact query, the
server, and the time. Cache aggressively and back off on throttling.

| # | Command | How to use |
|---|---------|------------|
| 37 | `for d in example.com example.net; do timeout 20 whois "$d"; sleep 2; done > ~/whois-demo/evidence/batch.txt` | Spaced batch with a bound |
| 38 | `whois example.com >/dev/null 2>&1; echo "exit=$?"` | Capture the exit code |
| 39 | `whois --no-recursion 'not a domain' >/dev/null 2>&1; echo "exit=$?"` | Failure exit code |
| 40 | `whois example.com \| sha256sum` | Fingerprint a record |
| 41 | `date -u +'%Y-%m-%dT%H:%M:%SZ'; whois example.com \| head -5` | Timestamp the evidence |
| 42 | `whois -h whois.ripe.net -q version` | Server version metadata |
| 43 | `rm -rf ~/whois-demo/evidence` | Remove collected records at close |

**Research questions**
- What exit code does the client return for a valid, an invalid, and a
  rate-limited query?
- How do you distinguish a registry error page from a valid empty answer?
- Which output fields are stable enough to parse, and which are not?
- How long should records be cached before re-querying?
- What must be redacted before a whois transcript is shared?

**Sources:** `man whois`; registry rate-limit policies; RFC 3912; shell exit
code conventions.
**Output:** a reproducible query log with timestamps, hashes, and exit codes.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| `-h`, `--host`, `-p`, `--port` | 1, 2, 3, 4 |
| `-I` | 2 |
| `-H` | 2, 5 |
| `--verbose` | 1, 2 |
| `--no-recursion` | 2, 6 |
| `--help`, `--version` | 1 |
| `-l`, `-L`, `-m`, `-M`, `-x` | 4 |
| `-b`, `-B`, `-G`, `-d` | 3, 5 |
| `-i`, `-T`, `-K`, `-r`, `-R`, `-a`, `-s`, `-g` | 3, 4 |
| `-t`, `-v`, `-q` | 4, 6 |

Config and environment: `/etc/whois.conf` (absent), `WHOIS_OPTIONS`,
`WHOIS_SERVER`, `LANG`/`LC_MESSAGES`. Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Identify build and configuration | 7 |
| 2 | Domain registration and referral chain | 7 |
| 3 | IP, netblock, and ASN queries | 7 |
| 4 | RIPE-specific object queries | 9 |
| 5 | Redaction and data quality | 6 |
| 6 | Automation, evidence, rate limits | 7 |
| **Total** | | **43** |

## Legal and safety

WHOIS queries are public, logged, and attributable. Query only domains and
addresses that are in scope for the engagement, respect registry rate limits,
and do not use registration data to contact individuals outside the agreed
rules of engagement. Registration records can contain personal data; store
them as engagement evidence, minimize what you keep, and delete them at close.
