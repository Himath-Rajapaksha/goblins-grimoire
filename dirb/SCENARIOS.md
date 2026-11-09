# dirb — command scenarios for deep research

Installed build: DIRB v2.22 (Debian package `dirb`). Every flag below appears
in [`help.txt`](help.txt). The default wordlist is compiled in as
`/usr/share/dirb/wordlists/common.txt`. There is no help flag: running `dirb`
with no arguments prints the usage block.

> **Intrusive:** every wordlist entry becomes an HTTP request. Use only on
> authorized targets, and agree on a request rate before scanning a production
> site.

## How to invoke

```text
@deep-research For an authorized web application, compare DIRB 2.22, ffuf
2.1.0, gobuster 3.6, and dirsearch for content discovery. Which tool handles
soft-404 and wildcard responses most reliably, how do their recursion and
extension models differ, and what request rate keeps a production WAF from
blocking the scan? Cite each tool's documentation, the Dirb README, and field
reports; return a wordlist and pacing plan with a stop condition.
```

Create the demo inputs, replacing the URL with an in-scope target:

```bash
mkdir -p ~/dirb-demo/evidence
printf '.php\n.bak\n.old\n' > ~/dirb-demo/extensions.txt
printf 'admin\nbackup\nconfig\n' > ~/dirb-demo/words.txt
```

## Scenario 1 — baseline and wordlist choice

The default run uses `common.txt`. Record which wordlist was used before
comparing results between runs.

| # | Command | How to use |
|---|---------|------------|
| 1 | `dirb https://target.example/` | Built-in `common.txt` |
| 2 | `dirb https://target.example/ /usr/share/dirb/wordlists/common.txt` | Explicit default wordlist |
| 3 | `dirb https://target.example/ /usr/share/dirb/wordlists/big.txt` | Larger wordlist |
| 4 | `dirb https://target.example/ /usr/share/dirb/wordlists/common.txt,/usr/share/dirb/wordlists/vulns/apache.txt` | Two wordlists in one run |
| 5 | `dirb https://target.example/ ~/dirb-demo/words.txt` | Engagement-specific words |

**Research questions**
- How many entries are in each shipped wordlist, and what is the request cost?
- Does dirb deduplicate words across multiple wordlists?
- Which wordlist is appropriate for a quick sweep versus a deep audit?
- How does dirb's default wordlist compare with SecLists `common.txt`?

**Sources:** `/usr/share/dirb/wordlists/`; Dirb README; SecLists documentation.
**Output:** a wordlist inventory with line counts and a per-run request count.

## Scenario 2 — extensions, case, and location

Extension sweeps find backup and source files that a plain directory scan
misses. `-i` and `-l` change how results are matched and displayed.

| # | Command | How to use |
|---|---------|------------|
| 6 | `dirb https://target.example/ -X .php,.bak,.old` | Append three extensions |
| 7 | `dirb https://target.example/ -x ~/dirb-demo/extensions.txt` | Extensions from a file |
| 8 | `dirb https://target.example/ -X .php -i` | Case-insensitive matching |
| 9 | `dirb https://target.example/ -X .php -l` | Print `Location` headers |
| 10 | `dirb https://target.example/ -b` | Use the path as-is |
| 11 | `dirb https://target.example/ -t` | Do not force a trailing slash |

**Research questions**
- Does `-X` apply to directory words as well as file words?
- When does `-i` create duplicate results?
- What does `-b` change for a URL that already contains encoded characters?
- Why can `-t` break recursion?

**Sources:** Dirb README; `man dirb`; HTTP URI normalization (RFC 3986).
**Output:** an extension-hit table and a normalized-URL comparison.

## Scenario 3 — recursion and filtering

Recursion is on by default and is dirb's main differentiator. Filtering decides
which responses count as "found".

| # | Command | How to use |
|---|---------|------------|
| 12 | `dirb https://target.example/ -r` | Disable recursion |
| 13 | `dirb https://target.example/ -R` | Ask before each recursion |
| 14 | `dirb https://target.example/ -N 403` | Ignore 403 responses |
| 15 | `dirb https://target.example/ -N 429 -N 503` | Ignore rate-limit and unavailable |
| 16 | `dirb https://target.example/ -f` | Fine-tune 404 detection |
| 17 | `dirb https://target.example/ -v` | Show NOT_FOUND pages too |
| 18 | `dirb https://target.example/ -w` | Continue past warnings |

**Research questions**
- How does dirb decide a response is "not found" by default?
- When is `-f` necessary, and how do you verify its classification?
- Does `-N` filter by exact code or class?
- What warnings does `-w` suppress, and which are safety-relevant?

**Sources:** Dirb source (`src/dirb.c`); `man dirb`; OWASP WSTG content
discovery guidance.
**Output:** a before/after result set for each filter and a false-positive list.

## Scenario 4 — authentication, cookies, and headers

Authenticated scans need the session material on every request. Keep the
credential out of shell history where possible.

| # | Command | How to use |
|---|---------|------------|
| 19 | `dirb https://target.example/ -u 'alice:Summer2026!'` | HTTP basic auth |
| 20 | `dirb https://target.example/ -c 'session=abc123'` | Session cookie |
| 21 | `dirb https://target.example/ -H 'X-Forwarded-For: 127.0.0.1'` | Custom header |
| 22 | `dirb https://target.example/ -a 'Mozilla/5.0 (X11; Linux x86_64)'` | Custom user agent |
| 23 | `dirb https://target.example/ -E ~/dirb-demo/client.pem` | Client certificate |
| 24 | `dirb https://target.example/ -c 'session=abc123' -u 'alice:Summer2026!' -H 'Accept: application/json'` | Combined session |

**Research questions**
- Does dirb send the cookie on recursion and extension requests?
- Which header order or default headers reveal dirb to the target?
- When does client-certificate auth require `-E`, and what format does it
  expect?
- How do you confirm the session is still valid mid-scan?

**Sources:** RFC 7617; RFC 6265; Dirb README; target authentication docs.
**Output:** a request transcript showing the auth material on each request.

## Scenario 5 — proxy, pacing, and noise control

Production targets need slower scans. `-z` is the only built-in delay; a proxy
adds another control point.

| # | Command | How to use |
|---|---------|------------|
| 25 | `dirb https://target.example/ -z 250` | 250 ms between requests |
| 26 | `dirb https://target.example/ -p 127.0.0.1:8080` | HTTP proxy |
| 27 | `dirb https://target.example/ -p 127.0.0.1:8080 -P 'proxyuser:proxypass'` | Authenticated proxy |
| 28 | `dirb https://target.example/ -S` | Silent; hide tested words |
| 29 | `dirb https://target.example/ -S -z 500 -N 429` | Quiet, slow, rate-limit aware |
| 30 | `dirb https://target.example/ -p 127.0.0.1:8080 -z 250 -S -o ~/dirb-demo/evidence/scan.txt` | Proxy, delay, output |

**Research questions**
- What request rate triggers the target's WAF or rate limiter?
- Does `-z` delay before or after each request, and does it apply to recursion?
- How do you verify traffic is actually going through the proxy?
- Which is less noisy: `-S` or `-v`?

**Sources:** Dirb README; target rate-limit documentation; proxy logs.
**Output:** a request-rate measurement and a WAF-block timeline.

## Scenario 6 — output, resume, and reporting

Dirb can save output and resume after the `q` hotkey. Preserve both the output
and the wordlist so the run is reproducible.

| # | Command | How to use |
|---|---------|------------|
| 31 | `dirb https://target.example/ -o ~/dirb-demo/evidence/scan.txt` | Save output |
| 32 | `dirb https://target.example/ -S -o ~/dirb-demo/evidence/scan-silent.txt` | Save without tested words |
| 33 | `dirb -resume` | Resume the saved session |
| 34 | `wc -l ~/dirb-demo/evidence/scan.txt` | Count result lines |
| 35 | `curl -sS -o /dev/null -w '%{http_code} %{size_download}\n' https://target.example/admin` | Verify one hit |
| 36 | `shred -u ~/dirb-demo/extensions.txt ~/dirb-demo/words.txt` | Remove demo inputs |

**Research questions**
- Where does dirb store resume state, and what happens if the wordlist changed?
- Does `-o` capture the same lines as stdout, including warnings?
- How do you separate real hits from soft-404 noise in the output?
- What metadata is needed to reproduce the scan later?

**Sources:** Dirb README; `man dirb`; engagement evidence policy.
**Output:** a reproducible scan record with URL, wordlist, flags, timestamp,
and a verified hit list.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| `-a`, `-b`, `-c`, `-E`, `-H`, `-u` | 2, 4 |
| `-p`, `-P` | 5 |
| `-X`, `-x` | 2 |
| `-i`, `-l`, `-N`, `-r`, `-R`, `-t` | 2, 3 |
| `-f`, `-v`, `-S`, `-w` | 3, 5 |
| `-o`, `-z` | 5, 6 |
| `-resume` (pseudo-flag) | 6 |

Default wordlist: `/usr/share/dirb/wordlists/common.txt`. Default proxy port:
1080. Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Baseline and wordlist choice | 5 |
| 2 | Extensions, case, and location | 6 |
| 3 | Recursion and filtering | 7 |
| 4 | Authentication and headers | 6 |
| 5 | Proxy and pacing | 6 |
| 6 | Output, resume, reporting | 6 |
| **Total** | | **36** |

## Legal and safety

Directory scanning is intrusive and can trigger WAF blocks, alerts, and
account lockouts. Use only on authorized targets, respect the agreed rate, and
stop if the target shows signs of distress. Do not use `-X` to hunt for
credentials or backups outside the engagement scope.
