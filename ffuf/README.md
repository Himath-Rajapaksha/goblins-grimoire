# ffuf — fast web fuzzer

- **Version:** 2.1.0-dev (`ffuf -V`)
- **Binary:** `/usr/bin/ffuf` (Debian package `ffuf`)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/ffuf/ffuf
- **Docs:** https://github.com/ffuf/ffuf#usage

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

ffuf replaces the `FUZZ` keyword in a URL, header, POST body, or raw request
with each line of a wordlist and sends the request. It then decides which
responses are interesting using matchers (status, size, words, lines, regex,
time) and filters (the same dimensions, inverted). It is the general-purpose
web fuzzer in this collection: directory/file discovery, parameter discovery,
vhost discovery, value fuzzing, and authenticated API probing all use the same
engine.

Its differentiator is calibration. `-ac` sends a few random probes first and
learns what a "not found" response looks like, so `-fs`/`-fw`/`-fl` filters can
be set automatically instead of by hand. That matters on soft-404 sites, where
every path returns `200 OK`.

## Use cases

| Situation | Why ffuf |
|-----------|----------|
| Content discovery | Find directories, files, and backups from a wordlist |
| Parameter discovery | Fuzz names and values in query strings and JSON bodies |
| Vhost discovery | Fuzz the `Host` header against one IP |
| Authenticated testing | Replay cookies and headers while fuzzing |
| Recursive discovery | `-recursion` walks into discovered directories |
| Multi-wordlist logic | `clusterbomb`, `pitchfork`, and `sniper` modes |
| Reproducible evidence | `-of json` / `-od` writes machine-readable results |

## Requirements & install layout

- Debian package `ffuf`; binary `/usr/bin/ffuf`. Version 2.1.0-dev.
- No root, no config file required. `-config` loads a saved config.
- Wordlists are external; common locations on this system are
  `/usr/share/wordlists/` and `/usr/share/seclists/` (check what is installed).
- Defaults that matter: `-t 40` threads, `-timeout 10`, match codes
  `200-299,301,302,307,401,403,405,500`, `-mode clusterbomb`.
- History is written under the user's config directory; `-search` finds a
  previous `FFUFHASH` payload.

## Basic syntax

```bash
ffuf -w wordlist.txt -u https://target/FUZZ
ffuf -w wordlist.txt -u https://target/FUZZ -mc all -fs 42 -c -v
ffuf -w hosts.txt -u https://target/ -H 'Host: FUZZ' -mc 200
ffuf -w entries.txt -u https://target/api -X POST \
  -H 'Content-Type: application/json' -d '{"name":"FUZZ"}' -fr error
```

The keyword is `FUZZ` by default; `-w file:KEYWORD` renames it so multiple
wordlists can be used in one request.

## Flag reference

### HTTP

| Flag | Meaning |
|------|---------|
| `-u <url>` | Target URL containing `FUZZ` |
| `-X <method>` | HTTP method (default GET) |
| `-d <data>` | POST body |
| `-H <header>` | Header, repeatable (`-H 'Name: Value'`) |
| `-b <cookie>` | Cookie header |
| `-r` | Follow redirects |
| `-raw` | Do not encode the URI |
| `-http2` | Use HTTP/2 |
| `-ignore-body` | Do not download response bodies |
| `-timeout <s>` | Request timeout (default 10) |
| `-x <proxy>` | HTTP or SOCKS5 proxy |
| `-replay-proxy <proxy>` | Replay matched requests through a proxy |
| `-sni <name>` | TLS SNI value |
| `-cc <file>` / `-ck <file>` | Client certificate / key |
| `-recursion` | Recurse into matches (URL must end in `FUZZ`) |
| `-recursion-depth <n>` | Maximum depth |
| `-recursion-strategy <s>` | `default` (redirect-based) or `greedy` |

### Input

| Flag | Meaning |
|------|---------|
| `-w <file[:KEYWORD]>` | Wordlist and optional keyword, repeatable |
| `-mode <mode>` | `clusterbomb`, `pitchfork`, `sniper` |
| `-e <exts>` | Extensions appended to `FUZZ` (e.g. `php,html`) |
| `-D` | DirSearch-compatible wordlist mode (with `-e`) |
| `-ic` | Ignore wordlist comments |
| `-enc <spec>` | Encoders, e.g. `FUZZ:urlencode b64encode` |
| `-input-cmd <cmd>` | Generate inputs from a command |
| `-input-num <n>` | Number of generated inputs (default 100) |
| `-input-shell <shell>` | Shell for `-input-cmd` |
| `-request <file>` | Raw HTTP request file |
| `-request-proto <p>` | Protocol for the raw request (default https) |

### Matchers

| Flag | Meaning |
|------|---------|
| `-mc <codes>` | Match status codes (default `200-299,301,302,307,401,403,405,500`; `all` for everything) |
| `-ms <sizes>` | Match response size |
| `-mw <words>` | Match word count |
| `-ml <lines>` | Match line count |
| `-mr <regex>` | Match response body by regex |
| `-mt <cond>` | Match time to first byte (`>100`, `<100`) |
| `-mmode <and\|or>` | Matcher operator (default `or`) |

### Filters

| Flag | Meaning |
|------|---------|
| `-fc <codes>` | Filter status codes |
| `-fs <sizes>` | Filter response size |
| `-fw <words>` | Filter word count |
| `-fl <lines>` | Filter line count |
| `-fr <regex>` | Filter response body by regex |
| `-ft <cond>` | Filter time to first byte |
| `-fmode <and\|or>` | Filter operator (default `or`) |

### General and output

| Flag | Meaning |
|------|---------|
| `-t <n>` | Concurrent threads (default 40) |
| `-rate <n>` | Requests per second (default 0 = unlimited) |
| `-p <delay>` | Delay between requests, e.g. `0.1` or `0.1-2.0` |
| `-ac` | Auto-calibrate filters |
| `-acc <string>` | Custom calibration string |
| `-ach` | Per-host calibration |
| `-ack <keyword>` | Calibration keyword (default `FUZZ`) |
| `-acs <strategy>` | Custom calibration strategies |
| `-c` | Colorized output |
| `-v` | Verbose (full URL and redirect target) |
| `-s` | Silent |
| `-json` | Newline-delimited JSON to stdout |
| `-o <file>` | Output file |
| `-od <dir>` | Directory for matched responses |
| `-of <fmt>` | `json`, `ejson`, `html`, `md`, `csv`, `ecsv`, `all` |
| `-or` | Do not create an output file when there are no results |
| `-maxtime <s>` | Overall time limit |
| `-maxtime-job <s>` | Per-job time limit |
| `-sf` | Stop if >95% of responses are 403 |
| `-se` | Stop on spurious errors |
| `-sa` | Stop on all errors (implies `-sf` and `-se`) |
| `-noninteractive` | Disable the interactive console |
| `-config <file>` | Load a configuration file |
| `-debug-log <file>` | Internal debug log |
| `-search <hash>` | Find a previous FFUFHASH payload |

## Recipes

Authorized targets only.

**1. Directory discovery with auto-calibration.** On soft-404 sites, plain
status matching produces thousands of false positives.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -u https://target/FUZZ -ac -c -t 40 -o evidence/dirs.json -of json
```

**2. Extension sweep.**

```bash
ffuf -w words.txt -u https://target/FUZZ -e .php,.bak,.old,.zip -mc 200,301,302
```

**3. Vhost discovery against one IP.** The `Host` header changes; the URL does
not.

```bash
ffuf -w vhosts.txt -u https://10.10.10.5/ -H 'Host: FUZZ.example.com' -mc 200 -fs 0
```

**4. POST JSON parameter/value fuzzing.**

```bash
ffuf -w values.txt -u https://target/api/login -X POST \
  -H 'Content-Type: application/json' -d '{"user":"admin","pass":"FUZZ"}' \
  -mc all -fr 'invalid' -c
```

**5. Authenticated fuzzing.** Copy the session cookie from the browser.

```bash
ffuf -w words.txt -u https://target/admin/FUZZ \
  -b 'SESSION=abc123' -H 'X-Requested-With: XMLHttpRequest' -ac
```

**6. Two wordlists (clusterbomb).** Every username × every password.

```bash
ffuf -w users.txt:USER -w passes.txt:PASS \
  -u https://target/login -X POST -d 'user=USER&pass=PASS' \
  -mc all -fr 'Login failed' -mode clusterbomb
```

**7. Recursive discovery.** Only `FUZZ` is supported for recursion, and the URL
must end in it.

```bash
ffuf -w dirs.txt -u https://target/FUZZ -recursion -recursion-depth 2 -ac
```

**8. Throttled run for a fragile or monitored target.**

```bash
ffuf -w words.txt -u https://target/FUZZ -rate 20 -p 0.2 -t 5 -timeout 20
```

## Output & pipelines

- Default stdout is a table: status, size, words, lines, and URL. `-v` adds the
  full URL and redirect location.
- `-json` emits one JSON object per line; `-of json` writes a single JSON
  document; `-of ejson` is the extended form with request/response data.
- `-od <dir>` stores the matched response bodies, one file per result — useful
  for reviewing what a `200` actually returned.
- `-or` avoids an empty output file, which keeps CI artifacts clean.
- `-search <FFUFHASH>` replays a specific result's input; the hash is printed
  in the interactive console.

## Gotchas & operational security

- **Soft 404s are the main source of noise.** A site that returns `200` for
  every path defeats status matching; use `-ac`, `-fs`, or `-fr`.
- **`-mc all` is rarely what you want.** It matches everything; pair it with a
  filter.
- **Thread count and rate matter.** `-t 40` is fast and loud; on a monitored or
  fragile target use `-rate`/`-p` and a lower `-t`.
- **Recursion can explode.** A wildcard that matches every directory turns
  `-recursion` into an unbounded crawl; set `-recursion-depth` and `-maxtime`.
- **Auto-calibration is a heuristic.** Review the first results before trusting
  the filter; `-acc` lets you calibrate against a known-bad path.
- **Cookies and tokens expire.** A long run may silently start getting login
  pages; watch for a change in response size.
- **`-replay-proxy` sends matched traffic to a proxy.** That is useful for Burp
  review but duplicates the load on the target.
- **Output can contain sensitive responses.** Store `-od` bodies in the
  engagement directory, not in a public repo.

## Legal & safety

Fuzz only applications you own or have **explicit written authorization** to
test. Content discovery and credential fuzzing generate real load and real
authentication failures; they can lock accounts, trigger alerts, and violate
terms of service. Respect rate limits and stop conditions in the rules of
engagement.

## Related tools

- [`gobuster/`](../gobuster/README.md) — simpler directory/DNS/vhost enumeration.
- [`dirsearch/`](../dirsearch/README.md) — Python content discovery with reports.
- [`sqlmap/`](../sqlmap/README.md) — take discovered parameters into SQLi testing.
- [`nuclei/`](../nuclei/README.md) — template checks on discovered endpoints.
- [`burp/`](../burp/README.md) — capture the request ffuf should replay.
- [`httpx/`](../httpx/README.md) — probe and fingerprint discovered hosts.

## References

- Installed help snapshot: [`help.txt`](help.txt)
- Upstream README: https://github.com/ffuf/ffuf
- Kali tool page: https://www.kali.org/tools/ffuf/
