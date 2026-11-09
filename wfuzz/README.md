# wfuzz — web fuzzer

- **Version:** wfuzz 3.1.0
- **Binary:** `/usr/bin/wfuzz`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/xmendez/wfuzz
- **Docs:** https://wfuzz.readthedocs.io/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

wfuzz replaces the `FUZZ` keyword (and `FUZ2Z`, `FUZ3Z`, …) in a URL, header,
cookie, or POST body with values from one or more payloads. It sends the
resulting requests concurrently, applies matchers and filters to the responses,
and prints the survivors. It handles directory and file discovery, parameter
fuzzing, credential brute force, virtual-host discovery, and injection
payload sweeps.

## Use cases

| Situation | Why wfuzz |
|-----------|-----------|
| Content discovery | Directory and file brute force |
| Parameter fuzzing | GET/POST/header values |
| Virtual hosts | `Host: FUZZ.example.com` |
| Credential testing | Basic, digest, NTLM brute force |
| Injection sweeps | Feed payload lists into parameters |
| Encoder chains | `md5@sha1`, URL/base64 transforms |

## Requirements & install layout

- Debian package 3.1.0; binary `/usr/bin/wfuzz`.
- Python 3 with pycurl. The packaged build may warn that pycurl is not linked
  against OpenSSL, which can break HTTPS fuzzing — test an HTTPS target first.
- Wordlists live under `/usr/share/wordlists/` or SecLists (install separately).
- No API keys.

## Basic syntax

```bash
# directory discovery
wfuzz -w /usr/share/wordlists/dirb/common.txt https://target/FUZZ

# multiple payloads
wfuzz -w users.txt -w pass.txt -d "user=FUZZ&pass=FUZ2Z" https://target/login

# filter out 404s and show only 200/301/302
wfuzz -w list.txt --hc 404 -w ... https://target/FUZZ

# virtual host discovery
wfuzz -w hosts.txt -H "Host: FUZZ.example.com" https://example.com/

# POST parameter fuzzing
wfuzz -w payloads.txt -d "q=FUZZ" https://target/search
```

## Flag reference

### Payloads and iterators
| Flag | Meaning |
|------|---------|
| `-z <payload[,param][,encoder]>` | Define a payload for a FUZZ keyword |
| `-w <wordlist>` | Alias for `-z file,<wordlist>` |
| `-m <iterator>` | Combine payloads: `product` (default), `zip`, `chain` |
| `--zP <params>` | Arguments for the preceding payload |
| `--zD <default>` | Default parameter for the preceding payload |
| `--zE <encoder>` | Encoder for the preceding payload |
| `--slice <filter>` | Slice payload elements |
| `-V alltype` | Brute force all GET/POST parameters without FUZZ |
| `-e <type>` | List payloads/encoders/iterators/printers/scripts |
| `-X <method>` | HTTP method (`FUZZ` allowed) |

### Request
| Flag | Meaning |
|------|---------|
| `-u <url>` | Request URL |
| `-d <data>` | POST body |
| `-H <header>` | Header (repeatable) |
| `-b <cookie>` | Cookie (repeatable) |
| `--basic/--digest/--ntlm <cred>` | Authentication (`user:pass`) |
| `-p <ip:port:type>` | Proxy (repeatable) |
| `-t <n>` | Concurrent connections (default 10) |
| `-s <n>` | Delay between requests |
| `-R <depth>` | Recursive path discovery |
| `-D <depth>` | Maximum link depth |
| `-L, --follow` | Follow redirects |
| `--ip <host:port>` | Connect to a fixed IP |
| `--req-delay <n>` / `--conn-delay <n>` | Request/connect timeouts |
| `-Z` | Scan mode: ignore connection errors |

### Matching and filtering
| Flag | Meaning |
|------|---------|
| `--hc/--hl/--hw/--hh <codes>` | Hide by code/lines/words/chars |
| `--sc/--sl/--sw/--sh <codes>` | Show by code/lines/words/chars |
| `--ss <regex>` / `--hs <regex>` | Show/hide by body regex |
| `--filter <expr>` | Filter language expression |
| `--prefilter <expr>` | Filter payload items before fuzzing |
| `--efield <expr>` / `--field <expr>` | Show extra fields / only fields |

### Output and control
| Flag | Meaning |
|------|---------|
| `-f <file,printer>` | Write results with a printer |
| `-o <printer>` | Output printer (e.g. `json`, `html`, `magictree`) |
| `--oF <file>` | Save results for later use as a payload |
| `--recipe <file>` / `--dump-recipe <file>` | Load/save option recipes |
| `--dry-run` | Print requests without sending |
| `-c` | Coloured output |
| `-v` | Verbose |
| `--script=<plugins>` / `--script-args <k=v>` | Run plugins |
| `-A`, `--AA`, `--AAA` | Verbose + colour + default/verbose/discover scripts |

### Payload names worth knowing
`file`, `wordlist`, `range`, `hexrange`, `hexrand`, `names`, `burpstate`,
`ipnet`, `permutation`, `buffer_overflow`, `dirwalk`, `list`, `stdin`,
`iprange`, `guitab`, `wfuzzp`, `ipnet`.

### Encoders worth knowing
`url`, `double url`, `base64`, `md5`, `sha1`, `sha256`, `hexlify`, `html`,
`rot13`, `none`, `uri_hex`, `json`, `mysql`, `mssql`.

## Recipes

```bash
# 1. Basic directory scan, hide 404 and 403
wfuzz -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
      --hc 404,403 https://target/FUZZ

# 2. File extensions from a wordlist
wfuzz -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt \
      --hc 404 https://target/FUZZ

# 3. POST login brute force with two payloads
wfuzz -w users.txt -w passwords.txt -d "username=FUZZ&password=FUZ2Z" \
      --hc 200 https://target/login

# 4. Virtual host discovery
wfuzz -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
      -H "Host: FUZZ.example.com" --hc 404 https://example.com/

# 5. Filter by response size (strip wildcard responses)
wfuzz -w list.txt --hh 0,1234 https://target/FUZZ

# 6. Filter by body regex
wfuzz -w list.txt --hs "not found" https://target/FUZZ

# 7. Recursive discovery with a depth cap
wfuzz -w list.txt -R 2 --hc 404 https://target/FUZZ

# 8. Save and resume results
wfuzz -w list.txt --oF run.json https://target/FUZZ
wfuzz -z wfuzzp,run.json https://target/FUZZ

# 9. JSON output for parsing
wfuzz -w list.txt -o json --hc 404 https://target/FUZZ > results.json

# 10. Dry run to verify request shaping
wfuzz -w list.txt --dry-run https://target/FUZZ
```

## Output & pipelines

- Default printer shows code, lines, words, chars, and the payload.
- `-o json` / `-o html` produce structured output for parsing and reports.
- `--oF` stores results for reuse with the `wfuzzp` payload.
- Combine with `jq` when JSON output is selected.

## Gotchas & operational security

- HTTPS may fail if the packaged pycurl lacks OpenSSL — verify early.
- Wildcard responses (every path returns 200) produce noise; use `--hh`,
  `--hc`, or `--hs` after sampling a baseline.
- High `-t` values can overwhelm small apps or trip WAFs.
- `-V alltype` sends a large number of parameter mutations; scope it.
- Encoders are easy to get wrong; use `--dry-run` to inspect the wire format.
- Some plugins make active requests (discover); review `--script-help` first.
- `-R` recursion can explode request counts; cap with `-D`.

## Legal & safety

Fuzzing generates high request volumes and can degrade or crash services. Use
only against systems you are authorized to test, respect rate limits, and stop
if availability is affected.

## Related tools

- [`dirsearch/`](../dirsearch/README.md) — directory-focused scanner
- [`x8/`](../x8/README.md) — parameter discovery
- [`paramspider/`](../paramspider/README.md) — passive parameter collection
- [`ffuf`](https://github.com/ffuf/ffuf) — alternative fuzzer

## References

- https://wfuzz.readthedocs.io/
- https://github.com/xmendez/wfuzz
